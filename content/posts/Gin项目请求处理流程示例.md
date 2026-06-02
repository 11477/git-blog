---
title: "Gin项目架构图与请求流程-以自制简易全栈博客系统为例"
date: 2026-06-02
tags: ["博客", "技术", "Go"]
categories: ["技术"]
summary: "了解Go的后端架构层级以及请求流转"
draft: false
---

## 架构图

```
                          ┌─────────┐
                          │ Browser │
                          └────┬────┘
                               │ HTTP
                               ▼
                        ┌─────────────┐
                        │    Nginx     │  :80  反向代理
                        │  (生产环境)   │
                        └──────┬──────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────┐
│                     Gin Engine :8080                      │
│                                                          │
│  ┌──────────┐    ┌──────────────────────────────────┐   │
│  │ Static   │    │         Middleware                │   │
│  │ /static  │    │  SetUser (全局)  AuthRequired     │   │
│  └──────────┘    └──────────┬───────────────────────┘   │
│                             │                            │
│  ┌──────────────────────────▼────────────────────────┐   │
│  │                    Router                          │   │
│  │                                                    │   │
│  │  页面: GET /  /posts/new  /posts/:id  /login ...  │   │
│  │  表单: POST /api/login  /api/posts/:id/delete ...  │   │
│  │  JSON: GET/POST/PUT/DELETE /api/posts ...         │   │
│  └────────────────────┬─────────────────────────────┘   │
│                       │                                  │
│         ┌─────────────┼─────────────┐                    │
│         ▼             ▼             ▼                    │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐           │
│  │AuthHandler │ │PostHandler │ │CommentHdlr │  Handler   │
│  │ .go        │ │ .go        │ │ .go        │           │
│  └─────┬──────┘ └─────┬──────┘ └─────┬──────┘           │
│        │              │              │                    │
│        ▼              ▼              ▼                    │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐           │
│  │UserSvc    │ │PostSvc     │ │CommentSvc  │  Service   │
│  │ .go       │ │ .go        │ │ .go        │           │
│  └─────┬──────┘ └─────┬──────┘ └─────┬──────┘           │
│        │              │              │                    │
│        ▼              ▼              ▼                    │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐           │
│  │UserRepo   │ │PostRepo    │ │CommentRepo │  Repository│
│  │ .go       │ │ .go        │ │ .go        │           │
│  └─────┬──────┘ └─────┬──────┘ └─────┬──────┘           │
│        │              │              │                    │
│        └──────────────┼──────────────┘                    │
│                       ▼                                   │
│               ┌──────────────┐                            │
│               │   database   │                            │
│               │   /sql.DB    │                            │
│               └──────┬───────┘                            │
│                      │                                    │
│  ┌────────────┐ ┌────┴─────┐ ┌──────────┐                │
│  │ config/    │ │   db.go  │ │ model/   │                │
│  │ DSN拼接    │ │ 连接+迁移 │ │ 3个结构体 │                │
│  └────────────┘ └──────────┘ └──────────┘                │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
                ┌─────────────┐
                │   MySQL     │  :3306
                │  users      │
                │  posts      │
                │  comments   │
                └─────────────┘
```

## 完整请求流程追踪

### 1. 注册

```
Browser POST /api/register (username, email, password)
    │
    ▼
router.go:       r.POST("/api/register", authHandler.Register)
    │
    ▼
auth.go:         AuthHandler.Register()
    │             c.PostForm("username"/"email"/"password")
    │             h.userService.Register(username, email, password)
    ▼
user.go(svc):    UserService.Register()
    │             bcrypt.GenerateFromPassword()  ← 密码哈希
    │             s.repo.Create(username, email, hash)
    ▼
user.go(repo):   UserRepo.Create()
    │             db.Exec("INSERT INTO users ... VALUES (?, ?, ?)")
    │             result.LastInsertId()          ← MySQL 获取自增ID
    │             db.QueryRow("SELECT ... WHERE id = ?")
    ▼
auth.go:         c.Redirect(302, "/login")      ← 重定向到登录页
```

### 2. 登录

```
Browser POST /api/login (username, password)
    │
    ▼
router.go:       r.POST("/api/login", authHandler.Login)
    ▼
auth.go:         AuthHandler.Login()
    │             h.userService.Login(username, password)
    ▼
user.go(svc):    UserService.Login()
    │             s.repo.FindByUsername(username)
    │             bcrypt.CompareHashAndPassword()  ← 比对密码哈希
    ▼
auth.go:         session.Values["user_id"] = user.ID
    │             session.Save()                    ← 写 Cookie
    │             c.Redirect(302, "/")
    ▼
Browser GET /  (带着 Cookie)
    │
    ▼
router.go:       r.GET("/", postHandler.Index)
    │
    ▼ 先经过全局中间件
middleware/auth.go: SetUser()
    │             store.Get(c.Request, "blog-session")
    │             userService.FindByID(userID)
    │             c.Set("user", user)     ← 注入到 Gin 上下文
    ▼
post.go:         PostHandler.Index()
    │             h.postService.List()
    │             templateData(c, gin.H{"Posts": posts})
    │                └── c.Get("user") → 注入 User 到模板
    │             c.HTML("index.html", ...)
    ▼
Browser:         看到首页 + "写文章"按钮（因为 .User 存在）
```

### 3. 创建文章

```
Browser POST /api/posts (title, content)
    │
    ▼
router.go:       api.POST("/posts", AuthRequired, postHandler.APICreate)
    │                              ↑ 中间件拦截未登录
    ▼
middleware/auth.go: AuthRequired()
    │             session.Values["user_id"] → c.Set("user_id", id)
    ▼
post.go:         PostHandler.APICreate()
    │             c.PostForm("title"/"content")  ← 表单取值
    │             h.postService.Create(title, content, userID)
    ▼
post.go(svc):    PostService.Create()
    │             s.repo.Create(title, content, userID)  ← 直接透传
    ▼
post.go(repo):   PostRepo.Create()
    │             db.Exec("INSERT INTO posts ... VALUES (?, ?, ?)")
    │             result.LastInsertId()
    │             db.QueryRow("SELECT ... FROM posts WHERE id = ?")
    ▼
post.go:         c.Redirect(302, "/posts/1")  ← 跳到文章详情
```

### 4. 添加评论

```
Browser POST /api/posts/1/comments (content)
    │
    ▼
router.go:       api.POST("/posts/:id/comments", AuthRequired, commentHandler.APICreate)
    ▼
comment.go:      CommentHandler.APICreate()
    │             c.PostForm("content")
    │             h.commentService.Create(content, postID, userID)
    ▼
comment.go(svc): CommentService.Create()
    │             s.repo.Create(content, postID, userID)
    ▼
comment.go(repo):CommentRepo.Create()
    │             db.Exec("INSERT INTO comments ... VALUES (?, ?, ?)")
    │             result.LastInsertId()
    ▼
comment.go:      c.Redirect(302, "/posts/1")  ← 回到文章页，评论已显示
```

### 5. 编辑文章

```
Browser GET /posts/1/edit
    │
    ▼
router.go:       r.GET("/posts/:id/edit", AuthRequired, postHandler.Edit)
    ▼
post.go:         PostHandler.Edit()
    │             h.postService.FindByID(1)
    │             c.HTML("edit.html", templateData(c, gin.H{"Post": post}))
    ▼
Browser 显示编辑表单

Browser POST /api/posts/1 (_method=PUT, title, content)
    │
    ▼
router.go:       r.POST("/api/posts/:id", AuthRequired, postHandler.Update)
    ▼
post.go:         PostHandler.Update()
    │             h.postService.Update(id, userID, title, content)
    ▼
post.go(svc):    PostService.Update()
    │             s.repo.FindByID(id)          ← 先查文章
    │             if post.UserID != userID      ← 权限校验：只有作者能改
    │               return errors.New("unauthorized")
    │             s.repo.Update(id, title, content)
    ▼
post.go(repo):   PostRepo.Update()
    │             db.Exec("UPDATE posts SET title=?, content=? WHERE id=?")
    ▼
post.go:         c.Redirect(302, "/posts/1")
```

### 6. 删除文章

```
Browser POST /api/posts/1/delete
    │
    ▼
router.go:       r.POST("/api/posts/:id/delete", AuthRequired, postHandler.Delete)
    ▼
post.go:         PostHandler.Delete()
    │             h.postService.Delete(id, userID)
    ▼
post.go(svc):    PostService.Delete()
    │             s.repo.FindByID(id)          ← 先查文章
    │             if post.UserID != userID      ← 权限校验：只有作者能删
    │             s.repo.Delete(id)
    ▼
post.go(repo):   PostRepo.Delete()
    │             db.Exec("DELETE FROM posts WHERE id=?")
    │             → MySQL CASCADE 自动删除该文章的所有评论
    ▼
post.go:         c.Redirect(302, "/")
```

## 各层职责总结

| 层 | 职责 | 不做的事 |
|---|---|---|
| **Router** | 路由分发 + 挂中间件 | 不碰业务逻辑 |
| **Middleware** | 认证拦截 + 上下文注入 | 不碰数据库 |
| **Handler** | 取请求参数 + 选返回格式(HTML/JSON) | 不碰 SQL 和密码 |
| **Service** | 业务规则（密码哈希、权限校验） | 不碰 HTTP 和 SQL |
| **Repository** | SQL 执行 + 结果扫描 | 不碰 HTTP 和业务规则 |
