---
title: "Go 学习笔记：Context 详解——goroutine 生命周期管理"
date: 2026-07-01
tags: ["Go", "并发", "面试"]
categories: ["学习笔记"]
summary: "Go 语言初学者的 Context 入门：为什么需要 Context、四个构造函数、Value 的惯用法、errgroup 配合模式、以及常见误区"
draft: false
---

> 这是「Go 学习笔记」系列的一篇。面向有基础编程经验、刚开始学 Go 的开发者，目标是讲清楚 Context 的**为什么存在、怎么用、什么时候用**。

## 一、context 是什么，为什么要存在

`context` 是 Go 标准库中的一个包，用于在 **goroutine 之间传递截止时间、取消信号和请求范围的键值对**。

理解它的最简单方式：你启动了一个 HTTP 请求处理，这个请求内部又调数据库、调下游服务、写缓存。如果用户把浏览器关了，你希望这 3 个操作能一起停掉，而不是各自闷头跑到超时——**context 就是用来传这个"停！"信号的那根绳子**。

```
用户请求 → handler(ctx)
                ├─ 查数据库(ctx)     ← 同一根绳子拴着
                ├─ 调下游服务(ctx)    ← 同一条取消信号
                └─ 写 Redis(ctx)      ← 同上
```

Go 标准库（`net/http`、`database/sql` 等）几乎所有涉及 I/O 的函数都接受 `context.Context` 作为第一个参数，这是 Go 的惯例。

## 二、核心接口

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)  // 什么时候到期
    Done() <-chan struct{}                    // 返回一个 channel，context 被取消时会 close
    Err() error                               // 为什么结束了（Canceled 或 DeadlineExceeded）
    Value(key interface{}) interface{}         // 取携带的键值对
}
```

你一般不直接实现这个接口，而是用标准库提供的构造函数来"创建"和"派生"。

## 三、四个构造函数

### 1. `context.Background()` 和 `context.TODO()`

**它们是根 context**，没有超时、没有取消、不携带值，永远不会 Done。

```go
ctx := context.Background()  // 程序主入口用
ctx := context.TODO()         // 还没想好用什么 context 时占位
```

大部分实际场景以它们为根，往下派生。

### 2. `context.WithCancel(parent)` — 手动取消

最常用的一种。返回新 ctx 和 `cancel` 函数：

```go
ctx, cancel := context.WithCancel(context.Background())

go func() {
    select {
    case <-ctx.Done():  // 收到取消信号
        fmt.Println("goroutine 被取消:", ctx.Err())
        return
    }
}()

cancel()  // 任何地方调用它，所有子 goroutine 都会收到 Done() 信号
```

核心约定：**`cancel()` 不 return error，可以多次调用，但只有第一次有效。**

### 3. `context.WithDeadline(parent, time.Time)` — 指定时刻到期

```go
deadline := time.Now().Add(5 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()  // 即使提前结束了也要记得调 cancel，释放资源
```

到点自动取消，你也可以手动提前 cancel。

### 4. `context.WithTimeout(parent, time.Duration)` — 指定时长后到期

就是对 `WithDeadline` 的语法糖：

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
```

## 四、四种类型的关系：context 树

context 是**树状结构**，每个派生操作都是挂到父节点下：

```
Background()
     └─ WithTimeout(5s)           // 客户端请求超时 5s
            ├─ WithValue("traceID")  // 挂了 trace id
            │      └─ WithTimeout(2s)  // 数据库操作只用 2s
            └─ WithCancel()           // 手动取消的下游请求
                   └─ ...
```

取消是**向下传播**的：当父节点被取消，所有子节点都会收到信号。但反向不会——一个子节点 cancel 了，其他兄弟不受影响。

```go
parentCtx, cancel := context.WithCancel(context.Background())

childCtx1, _ := context.WithCancel(parentCtx)
childCtx2, _ := context.WithCancel(parentCtx)

cancel() // 关闭 parentCtx
// childCtx1 和 childCtx2 同时收到 Done() 信号
```

## 五、Value——在 context 中传递请求范围的数据

```go
type contextKey string                              // 自定义类型防止 key 冲突（重要！）
const requestIDKey contextKey = "requestID"

// 写入
ctx := context.WithValue(context.Background(), requestIDKey, "abc-123")

// 读取
id, ok := ctx.Value(requestIDKey).(string)  // 记得用类型断言
if !ok {
    id = "unknown"
}
```

**规则**：
- **只放请求范围的数据**：trace id、user token、请求开始时间。不要放业务参数。
- key 不应该是基础类型（如直接用 `string`），用自定义类型防止不同包的 key 撞车

**为什么不能用 `string` 作为 key**：Value 的底层是沿着父节点一路向上查，靠 `==` 比较 key。如果你和第三方库都用了 `string("id")`，你的值就可能被它的覆盖。

```go
// 错误示范
ctx = context.WithValue(ctx, "id", "foo")  // 别这样，可能被其他包覆盖

// 正确示范
type traceIDKey struct{}                    // 不导出的空 struct，只有本包能用
ctx = context.WithValue(ctx, traceIDKey{}, "foo")
```

## 六、一个完整的实际例子

模拟一个 HTTP handler 查数据库时受超时控制：

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()                           // HTTP 请求自带 context

    // 基于请求 ctx 再加一层超时：整个处理过程 3 秒
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()

    result, err := queryDatabase(ctx, "SELECT ...")  // ctx 传进去
    if err == context.DeadlineExceeded {
        http.Error(w, "数据库查询超时", http.StatusGatewayTimeout)
        return
    }
    // ...
}

func queryDatabase(ctx context.Context, sql string) (string, error) {
    // 模拟一个支持 context 的数据库操作
    done := make(chan string, 1)

    go func() {
        time.Sleep(2 * time.Second)  // 模拟耗时操作
        done <- "查询结果"
    }()

    select {
    case result := <-done:
        return result, nil
    case <-ctx.Done():                    // ctx 超时或者被取消
        return "", ctx.Err()              // 返回 context.DeadlineExceeded
    }
}
```

## 七、context 的传递规则

- **第一个参数永远是 `ctx context.Context`**。官方建议叫 `ctx`，不要叫别的。
- **不要存到 struct 里长久保存**（它代表请求生命周期，请求结束就应该丢弃）。
- `context.Background()` 只在 `main`、`init` 或测试里出现。
- **派生 context 后一定要 defer cancel()**，否则子 goroutine 和资源永远得不到释放（goroutine 泄漏）。

```go
ctx, cancel := context.WithTimeout(parent, 5*time.Second)
defer cancel()  // 不管是正常走完、return、panic，都释放
```

### goroutine 泄漏是什么

```go
// 泄漏的代码
func leakyQuery() {
    ch := make(chan string)
    go func() {
        time.Sleep(30 * time.Second)  // 模拟慢查询
        ch <- "done"
    }()
    select {
    case <-ch:
        return
    case <-time.After(1 * time.Second):
        return  // 超时返回了，但上面的 goroutine 还活着，30 秒后才结束
    }
}
```

加了 context 之后就没有这个问题：

```go
func safeQuery(ctx context.Context) (string, error) {
    ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
    defer cancel()
    // 使用支持 context 的标准库或第三库，goroutine 会在 ctx.Done() 时被回收
    // ...
}
```

## 八、和 errgroup 配合

这是生产上最常见的用法。`golang.org/x/sync/errgroup` 提供了带 context 的 goroutine 组：

```go
import "golang.org/x/sync/errgroup"

g, ctx := errgroup.WithContext(context.Background())

g.Go(func() error {
    return fetchUser(ctx, userID)   // 其中一个失败了，ctx 自动被 cancel
})
g.Go(func() error {
    return fetchOrders(ctx, userID) // 另外一个也会收到取消信号
})

if err := g.Wait(); err != nil {
    log.Println("出错了:", err)
}
```

**这点很重要**：`errgroup.WithContext` 返回的 ctx 会在任何一个 goroutine return error 时自动触发 cancel，你不用手动调。

## 九、两个常见误区

**误区 1："context 没传进去，代码也能跑"**

```go
func badQuery() {
    rows, _ := db.Query("SELECT ...") // 没有 ctx，没有超时控制
}
```

数据库连不上，它永远卡在那。**所有 I/O 操作都应该带 context**。

**误区 2：在 context 里存大对象**

```go
ctx = context.WithValue(ctx, "user", largeUserObject) // 不要
```

`Value` 链是线性查找，会越传越重。只存小粒度的元数据（trace id 等），业务对象直接当参数传。

## 十、一句话总结

context 是 Go 的 **goroutine 生命周期管理机制**，解决的核心需求：**一个操作被取消了，它派生的所有子操作都收到信号并退出来**。核心就三件事：**超时**、**取消**、**传递请求范围的值**。

初学者只需要背下一句：每个 I/O 函数装一个 `ctx` 作为第一个参数，主函数用 `context.WithTimeout` 或 `context.WithCancel` 派生它，记得 `defer cancel()`。先做到这两步，其余的用到了再查。
