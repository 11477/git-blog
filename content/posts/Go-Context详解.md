---
title: "Go 学习笔记：Context 详解——goroutine 生命周期管理"
date: 2026-07-01
tags: ["Go", "并发", "面试"]
categories: ["学习笔记"]
summary: "例子驱动的 Go Context 讲解——每个功能从一个有痛点的场景出发，展示问题，再用 Context 解决，最后总结规律"
draft: false
---

> 这篇不讲概念，讲场景。每个例子你都可以 copy 到 `main.go` 里直接跑。

## 先跑起来：context 长什么样

最简单的观察一下 context 是什么：

```go
package main

import (
    "context"
    "fmt"
)

func main() {
    ctx := context.Background()
    fmt.Printf("ctx: %v\n", ctx)       // context.Background
    fmt.Printf("Deadline: %v\n", ctx.Deadline()) // 0001-01-01 false (没有截止时间)
    fmt.Printf("Err: %v\n", ctx.Err())         // nil (没被取消)
    
    // Done() 返回一个 channel，Background 永远不会 Done
    // 所以下面这行会永久阻塞：
    // <-ctx.Done()
}
```

看到 `Background()` 就是一个"空壳"。真正有用的 context 都是从它派生出来的。往下看。

---

## 场景一：goroutine 跑太久，我想叫停它

### 问题

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    go doWork()

    time.Sleep(1 * time.Second) // 主 goroutine 等 1 秒就走了
    fmt.Println("main 退出")
    // 问题：doWork 还在跑，没人让它停！
}

func doWork() {
    for {
        fmt.Println("工作中...")
        time.Sleep(200 * time.Millisecond)
    }
}
```

运行结果：main 退出后 `doWork` 还在打印。如果你有 100 个这样的 goroutine，就泄漏了。

### 解法：WithCancel

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    go doWork(ctx)

    time.Sleep(1 * time.Second)
    cancel() // 👈 叫停
    time.Sleep(100 * time.Millisecond) // 等 doWork 收尾
    fmt.Println("main 退出")
}

func doWork(ctx context.Context) {
    for {
        select {
        case <-ctx.Done(): // 👈 收到取消信号
            fmt.Println("doWork 被通知取消了，退出")
            return
        default:
            fmt.Println("工作中...")
            time.Sleep(200 * time.Millisecond)
        }
    }
}
```

运行结果：

```
工作中...
工作中...
工作中...
工作中...
工作中...
doWork 被通知取消了，退出
main 退出
```

**关键**：`ctx.Done()` 返回一个 channel。你调 `cancel()` 后，这个 channel 被关闭，之前阻塞在 `<-ctx.Done()` 上的代码马上收到通知。

---

## 场景二：能不能自动超时，不用我手动 cancel？

### 问题

你写了个函数调外部 API，API 可能卡死。你不想手动设闹钟，希望"超过 3 秒就自动放弃"。

### 解法：WithTimeout

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    // 3 秒后自动取消
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel() // 👈 养成习惯：拿了 cancel 一定要 defer

    result, err := fetchFromAPI(ctx)
    if err != nil {
        fmt.Println("出错:", err)
        return
    }
    fmt.Println("结果:", result)
}

func fetchFromAPI(ctx context.Context) (string, error) {
    select {
    case <-time.After(5 * time.Second): // 模拟 API 要 5 秒才回
        return "API 返回的数据", nil
    case <-ctx.Done(): // 👈 3 秒到，ctx 自动触发
        return "", fmt.Errorf("请求被取消: %w", ctx.Err())
    }
}
```

输出：

```
出错: 请求被取消: context deadline exceeded
```

**关键**：`ctx.Err()` 告诉你**为什么**结束了——`context deadline exceeded`(超时) 或 `context canceled`(被手动 cancel)。

---

## 场景三：别人给我的 context，我想"加一个更紧的超时"

一个很常见的真实场景：HTTP handler 自带的 ctx 有 60 秒超时，但你的数据库查询不能等那么久，你想给它一个 2 秒的限制。

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    // 模拟 HTTP 请求自带的 ctx（60 秒超时）
    parentCtx, _ := context.WithTimeout(context.Background(), 60*time.Second)

    // 在 handler 里，你想给 DB 查询只 2 秒
    queryDatabase(parentCtx)
}

func queryDatabase(parentCtx context.Context) {
    // 👇 派生一个新的 ctx，2 秒后自动取消（不受 parent 的 60 秒影响）
    ctx, cancel := context.WithTimeout(parentCtx, 2*time.Second)
    defer cancel()

    select {
    case <-time.After(5 * time.Second): // 模拟慢查询
        fmt.Println("查询完成")
    case <-ctx.Done():
        fmt.Println("查询超时:", ctx.Err()) // 2 秒后走到这里
    }
}
```

**关键**：子 ctx 的超时不能超过父 ctx。如果你设了 2 秒，父 ctx 设了 60 秒，那就是 2 秒后取消。反过来，如果父 ctx 只有 1 秒就过期了，你的"2 秒"实际上只有 1 秒——**取最紧的那个**。

---

## 场景四：一个操作要同时调多个下游，任何一个失败就全停

美团下单：查库存、算运费、扣积分，三个可以同时做。但库存没了的话，运费和积分也别算了。

### 不用 context 的写法（有泄漏）

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    result := make(chan string, 3)

    go checkStock(result)   // 3 秒
    go calcShipping(result)  // 200 毫秒
    go deductPoints(result) // 2 秒

    // 取最快的结果
    fmt.Println(<-result)
    // ⚠️ 剩下两个 goroutine 还在跑，没人管
}

func checkStock(ch chan string) {
    time.Sleep(3 * time.Second)
    ch <- "库存不足!" // 这是第一个完成吗？不是，shipping 更快
}

func calcShipping(ch chan string) {
    time.Sleep(200 * time.Millisecond)
    ch <- "运费 5 元"
}

func deductPoints(ch chan string) {
    time.Sleep(2 * time.Second)
    ch <- "积分已扣"
}
```

得到运费结果后，库存和积分 goroutine 还在白白跑。

### 用 context 的写法（errgroup 版本）

```go
package main

import (
    "context"
    "fmt"
    "time"

    "golang.org/x/sync/errgroup"
)

func main() {
    g, ctx := errgroup.WithContext(context.Background())

    g.Go(func() error {
        return checkStock(ctx)
    })
    g.Go(func() error {
        return calcShipping(ctx)
    })
    g.Go(func() error {
        return deductPoints(ctx)
    })

    if err := g.Wait(); err != nil {
        fmt.Println("某个操作出错:", err)
    }
}

func checkStock(ctx context.Context) error {
    time.Sleep(3 * time.Second)
    return fmt.Errorf("库存不足!") // 一 return error，ctx 自动 cancel
}

func calcShipping(ctx context.Context) error {
    select {
    case <-time.After(200 * time.Millisecond):
        fmt.Println("运费计算完成")
        return nil
    case <-ctx.Done(): // checkStock 失败后 ctx 取消，走到这里
        fmt.Println("运费计算被取消:", ctx.Err())
        return ctx.Err()
    }
}

func deductPoints(ctx context.Context) error {
    select {
    case <-time.After(2 * time.Second):
        fmt.Println("积分扣除完成")
        return nil
    case <-ctx.Done(): // 同上
        fmt.Println("积分扣除被取消:", ctx.Err())
        return ctx.Err()
    }
}
```

输出（大概这样）：

```
运费计算完成
积分扣除被取消: context canceled
某个操作出错: 库存不足!
```

**关键**：`errgroup.WithContext` 返回的 ctx 有一个自动机制——**组里任何一个 goroutine return 了 error，ctx 立刻 cancel。** 其他 goroutine 只要在 select 里检测 `ctx.Done()`，就能优雅退出。

---

## 场景五：一个操作要调多个下游，全部完成才算成功（任一失败都要取消）

和上面类似，但这次是"所有都成功才返回"：

```go
package main

import (
    "context"
    "fmt"
    "time"

    "golang.org/x/sync/errgroup"
)

func main() {
    g, ctx := errgroup.WithContext(context.Background())

    urls := []string{"api.example.com/a", "api.example.com/b", "api.example.com/c"}
    results := make([]string, len(urls))

    for i, url := range urls {
        i, url := i, url // 👈 闭包陷阱：必须拷贝变量
        g.Go(func() error {
            return fetch(ctx, i, url, results)
        })
    }

    if err := g.Wait(); err != nil {
        fmt.Println("失败:", err) // 只要一个 URL 失败了，其他也收到 cancel
    } else {
        fmt.Println("全部成功:", results)
    }
}

func fetch(ctx context.Context, index int, url string, results []string) error {
    select {
    case <-time.After(500 * time.Millisecond): // 模拟 HTTP 请求
        results[index] = url + " -> 200 OK"
        fmt.Printf("请求 %s 成功\n", url)
        return nil
    case <-ctx.Done():
        fmt.Printf("请求 %s 被取消\n", url)
        return ctx.Err()
    }
}
```

如果 `api.example.com/b` 的 `fetch` 里 `time.After` 换成一个实际请求、实际请求失败了 return error，其他 2 个请求自动被 cancel。

---

## 场景六：多个下游，拿到最快一个的结果就行，不需要任何失败的反馈

还是下单场景，但这次要求是"不管谁先回来，拿到结果我就走"：

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    resultCh := make(chan string, 3)

    go checkStock2(ctx, resultCh)   // 3 秒
    go calcShipping2(ctx, resultCh)  // 200 毫秒
    go deductPoints2(ctx, resultCh) // 2 秒

    // 只要第一个结果
    first := <-resultCh
    cancel() // 👈 拿到结果后立刻取消其他两个
    fmt.Println("最快结果:", first)
    time.Sleep(100 * time.Millisecond) // 等它们打印取消消息
}

func checkStock2(ctx context.Context, ch chan<- string) {
    select {
    case <-time.After(3 * time.Second):
        ch <- "库存充足"
    case <-ctx.Done():
        fmt.Println("库存检查被取消")
    }
}

func calcShipping2(ctx context.Context, ch chan<- string) {
    select {
    case <-time.After(200 * time.Millisecond):
        ch <- "运费 5 元"
    case <-ctx.Done():
        fmt.Println("运费计算被取消")
    }
}

func deductPoints2(ctx context.Context, ch chan<- string) {
    select {
    case <-time.After(2 * time.Second):
        ch <- "积分已扣"
    case <-ctx.Done():
        fmt.Println("积分扣除被取消")
    }
}
```

输出：

```
最快结果: 运费 5 元
库存检查被取消
积分扣除被取消
```

和场景四的区别：这里不用 errgroup，手动 `cancel()`，因为**不需要知道"失败原因"**，只知道"最快的人到了，其他人收工"。

---

## 场景七：在 context 里传 trace id，让整个调用链日志都能关联

```go
package main

import (
    "context"
    "fmt"
    "log/slog"
    "os"
)

// 👈 不导出的自定义类型，外面的人猜不到 key
type traceIDKey struct{}

func main() {
    logger := slog.New(slog.NewTextHandler(os.Stdout, nil))
    
    ctx := context.Background()
    ctx = context.WithValue(ctx, traceIDKey{}, "abc-123-x")

    handleRequest(ctx, logger)
}

func handleRequest(ctx context.Context, logger *slog.Logger) {
    traceID, _ := ctx.Value(traceIDKey{}).(string) // 断言成 string
    
    logger.Info("请求开始", "trace_id", traceID)  // trace_id=abc-123-x
    queryUser(ctx, logger)
    logger.Info("请求结束", "trace_id", traceID)
}

func queryUser(ctx context.Context, logger *slog.Logger) {
    traceID, _ := ctx.Value(traceIDKey{}).(string)
    
    logger.Info("查询用户表", "trace_id", traceID) // 同一个 trace_id

    // 只需要 trace id，不需要 override 原来的——直接用父 ctx
    // 不需要 WithValue
}
```

输出：

```
time=... level=INFO msg="请求开始" trace_id=abc-123-x
time=... level=INFO msg="查询用户表" trace_id=abc-123-x
time=... level=INFO msg="请求结束" trace_id=abc-123-x
```

**关键**：`WithValue` 是插入新值，**不会覆盖**已有的值。`queryUser` 直接用 `ctx` 就能拿到 handler 层注入的 trace_id。

---

## 场景八：数据库操作超时（真实库的用法）

之前都是模拟的 `select` + `time.After`。真正调用 `database/sql` 时，context 是这么用的：

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "time"

    _ "github.com/lib/pq" // Postgres 驱动
)

func getUserByID(db *sql.DB, id int) (string, error) {
    // 整个数据库调用的上限是 2 秒
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    var name string
    err := db.QueryRowContext(ctx, "SELECT name FROM users WHERE id = $1", id).Scan(&name)
    if err != nil {
        return "", fmt.Errorf("查询用户 %d 失败: %w", id, err)
    }
    return name, nil
}
```

标准库所有 I/O 操作都有带 `Context` 后缀的版本：

| 无 context | 有 context |
|------------|-----------|
| `db.Query(sql)` | `db.QueryContext(ctx, sql)` |
| `db.QueryRow(sql)` | `db.QueryRowContext(ctx, sql)` |
| `db.Exec(sql)` | `db.ExecContext(ctx, sql)` |
| `http.NewRequest(method, url, body)` | `http.NewRequestWithContext(ctx, method, url, body)` |
| `net.Dial(network, address)` | `net.DialContext(ctx, network, address)` |

**只要做 I/O，就用 Context 版本。** 这是 Go 社区的铁律。

---

## 总结：你怎么记住这些

假设你是公司老板（main goroutine），下属在干活（goroutine），context 就是你给下属的那张**任命书**。任命书上写了三样东西：

> 给你 3 天时间（**超时**），过期我这任命就作废。
> 如果我打电话告诉你停，你就停（**取消**）。
> 上面还贴了一个项目编号，所有报销单上都填同一个编号（**Value**）。

对应到代码就是：

```go
// 创建任命书
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()
// 贴上项目编号
ctx = context.WithValue(ctx, projectIDKey{}, "proj-42")

// 交给下属
go doJob(ctx)
```

下属在工作里，只要在每次"等回复"的地方看一眼任命书还在不在：

```go
func doJob(ctx context.Context) {
    select {
    case result := <-doSlowThing():
        return result
    case <-ctx.Done(): // 👈 "任命书过期了 / 被撤销了"
        return ctx.Err()
    }
}
```

六个核心场景对应的选择：

| 需求 | 用什么 |
|------|--------|
| 手动叫停某个 goroutine | `WithCancel` |
| 超过 N 秒自动放弃 | `WithTimeout` |
| 某个时间点自动放弃 | `WithDeadline` |
| 传递 trace id / 用户 token | `WithValue` |
| 并发跑多个操作，任一失败全停 | `errgroup.WithContext` |
| 并发跑多个操作，拿最快结果就行 | `WithCancel` + channel |
