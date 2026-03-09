# 项目目录结构与命名规范

---
paths:
  - "**/*.go"
---

## 标准目录树

```
<project>/
├── main.go                          # 入口，仅做启动引导
├── go.mod
├── config/
│   ├── config.go                    # 总配置 Config struct
│   └── config.yaml                  # 默认配置文件
├── server/
│   ├── gin.go                       # Gin 引擎与中间件初始化
│   ├── router.go                    # 路由注册（调用各模块 AddWebRouterForXxx）
│   ├── services.go                  # allServices struct + initServices()
│   └── <module>.go                  # 各业务模块的初始化函数（如 errata.go）
├── common/                          # 跨模块共享（按需创建）
│   ├── domain/
│   │   └── <concept>/               # 共享接口（如 cla/、obs/、emailclient/）
│   │       └── <concept>.go
│   └── infrastructure/
│       └── <concept>impl/           # 共享接口的实现
└── <module>/                        # 业务模块（可有多个，如 errata/、user/）
    ├── config.go                    # 模块级 Config struct
    ├── domain/
    │   ├── <aggregate>.go           # 聚合根
    │   ├── errors.go                # 错误码常量
    │   ├── dp/                      # 值对象（domain primitives）
    │   │   └── <concept>.go         # 如 status.go、email.go
    │   ├── event/                   # 领域事件（有消息发布时才创建）
    │   │   └── event.go
    │   └── repository/
    │       └── repo.go              # Repository 接口定义
    ├── app/
    │   ├── app.go                   # AppService 接口 + 实现（同一文件）
    │   └── dto.go                   # 所有 Cmd、DTO 定义
    ├── controller/
    │   ├── <module>.go              # Controller struct、Handler 方法、路由注册函数
    │   ├── request.go               # 请求体结构体（reqToXxx）
    │   └── config.go                # Controller 配置（如限流阈值，按需创建）
    └── infrastructure/
        ├── repositoryimpl/
        │   ├── <aggregate>.go       # Repository 实现
        │   ├── <aggregate>_do.go    # DO 模型（与实现分离）
        │   └── config.go            # 表名等配置
        └── <concept>impl/           # 其他基础设施实现（如 emailmessageimpl/）
            └── impl.go
```

---

## 包名（package name）约定

| 目录 | package 名 |
|------|-----------|
| `<module>/domain/` | `domain` |
| `<module>/domain/dp/` | `dp` |
| `<module>/domain/repository/` | `repository` |
| `<module>/app/` | `app` |
| `<module>/controller/` | `controller` |
| `<module>/infrastructure/repositoryimpl/` | `repositoryimpl` |
| `<module>/infrastructure/<concept>impl/` | `<concept>impl` |
| `server/` | `server` |
| `config/` | `config` |
| `common/domain/<concept>/` | `<concept>` |

> 多个业务模块都叫 `package domain`、`package app`、`package controller` 是正常的，Go 通过导入路径区分，包名无需加模块前缀。

---

## 类型命名约定

### 各层核心类型

| 层 | 类型 | 命名规则 | 示例 |
|----|------|---------|------|
| Domain | 聚合根 | `<Module>`（公有） | `Errata`、`ErrataReview` |
| Domain | 值对象构造函数 | `New<Concept>(v string) (string, error)` | `NewStatus`、`NewEmail` |
| App | AppService 接口 | `<Module>AppService` | `ErrataAppService` |
| App | AppService 实现 | `<module>App`（私有） | `errataApp` |
| Controller | Controller 结构体 | `<Module>Controller` | `ErrataController` |
| Controller | 路由注册函数 | `AddWebRouterFor<Module>Controller` | `AddWebRouterForErrataController` |
| Infrastructure | Repository 实现 | `<module>Impl`（私有） | `errataImpl` |
| Infrastructure | DO 模型 | `<module>DO`（私有） | `errataDO` |

### Handler 方法命名

Handler 方法使用**动作名**，不加 `Handler` 后缀：

```go
// ✅ 正确
func (ctl *ErrataController) Create(c *gin.Context) {}
func (ctl *ErrataController) List(c *gin.Context) {}
func (ctl *ErrataController) Get(c *gin.Context) {}
func (ctl *ErrataController) Remove(c *gin.Context) {}

// ❌ 错误：不要加 Handler 后缀
func (ctl *ErrataController) CreateHandler(c *gin.Context) {}
```

常用动作名约定：

| 操作 | 方法名 |
|------|-------|
| 创建 | `Create` |
| 查询单个 | `Get` |
| 查询列表 | `List` |
| 更新 | `Update` 或语义化动作名（`Resubmit`、`Revocate`） |
| 删除 | `Remove` |

---

## 文件命名约定

| 文件 | 位置 | 说明 |
|------|------|------|
| `<aggregate>.go` | `domain/` | 聚合根定义，一个聚合根一个文件 |
| `errors.go` | `domain/` | 该模块所有错误码常量 |
| `<concept>.go` | `domain/dp/` | 值对象，如 `status.go`、`email.go` |
| `repo.go` | `domain/repository/` | Repository 接口定义 |
| `event.go` | `domain/event/` | 领域事件定义（有需要时） |
| `app.go` | `app/` | AppService 接口 + 实现（同文件） |
| `dto.go` | `app/` | 所有 Cmd 和 DTO |
| `<module>.go` | `controller/` | Controller + Handler + 路由注册函数 |
| `request.go` | `controller/` | 所有请求体结构体（reqToXxx） |
| `<aggregate>.go` | `repositoryimpl/` | Repository 实现 |
| `<aggregate>_do.go` | `repositoryimpl/` | DO 模型（与实现分离） |
| `config.go` | 各层 | 本层/本模块的配置 struct |

---

## 聚合根字段可见性

聚合根字段使用**公有字段**（首字母大写）：

```go
// ✅ 团队约定：公有字段
type Errata struct {
    Id        int64
    Email     string
    Status    string
    Version   int
    CreatedAt time.Time
}
```

**trade-off 说明**：
- 公有字段便于 DO 转换和测试，减少 getter 模板代码
- 风险：外部代码可能绕过聚合根方法直接修改字段，须依赖 Code Review 防止

---

## 状态值约定

业务状态使用**字符串常量**，定义在 `domain/dp/<concept>.go`：

```go
// domain/dp/status.go
package dp

const (
    StatusPending    = "pending"
    StatusAccepted   = "accepted"
    StatusRejected   = "rejected"
    StatusCancelled  = "cancelled"
)

// 构造函数：验证合法性
var validStatuses = map[string]bool{
    StatusPending:   true,
    StatusAccepted:  true,
    StatusRejected:  true,
    StatusCancelled: true,
}

func NewStatus(v string) (string, error) {
    if !validStatuses[v] {
        return "", errors.New("invalid status: " + v)
    }
    return v, nil
}
```

- 命名：`Status<State>`（在 dp 包内无需加模块前缀）
- 不使用 `int` 枚举，字符串值与数据库存储值保持一致，直接可读

---

## 错误码约定

定义在 `domain/errors.go`，格式 `ErrorCode<Module><Action/State>`：

```go
// domain/errors.go
package domain

const (
    ErrorCodeOrderNotFound          = "order_not_found"
    ErrorCodeOrderAlreadyExists     = "order_already_exists"
    ErrorCodeOrderCanNotCancel      = "order_can_not_cancel"
    ErrorCodeOrderUpdateConcurrently = "order_update_concurrently"
)
```

- 全部公有（大写开头），供 App 层引用
- 每个业务模块独立定义，不跨模块共享错误码文件

---

## 配置结构

### 总配置（config/config.go）

```go
package config

type Config struct {
    Errata            errata.Config   `json:"errata"`
    User              user.Config     `json:"user"`
    OperationLog      oplog.Config    `json:"operation_log"`
    ReadHeaderTimeout int             `json:"read_header_timeout"`
}
```

### 模块配置（<module>/config.go）

```go
package errata   // 或对应的模块包名

type Config struct {
    DP         dp.Config             `json:"dp"`
    Repo       repositoryimpl.Config `json:"repo"`
    Controller controller.Config     `json:"controller"`
}
```

### Repository 配置（repositoryimpl/config.go）

```go
package repositoryimpl

type Config struct {
    Order string `json:"order"`   // 表名
}
```

---

## Server 层初始化模式

```go
// server/services.go
package server

type allServices struct {
    orderApp  orderapp.OrderAppService
    userApp   userapp.UserAppService
    // ...
}

func initServices(cfg *config.Config) (services allServices, err error) {
    // 按拓扑顺序初始化（被依赖的先初始化）
    if err = initCommon(cfg, &services); err != nil {
        return
    }
    if err = initOrder(cfg, &services); err != nil {
        return
    }
    return
}

// server/order.go
func initOrder(cfg *config.Config, services *allServices) error {
    repo, err := repositoryimpl.NewOrder(postgresql.DAO(cfg.Order.Repo.Order))
    if err != nil {
        return err
    }
    tx := postgresql.NewTransaction()
    services.orderApp = app.NewOrderAppService(repo, tx)
    return nil
}

// server/router.go
func setRouters(rg *gin.RouterGroup, cfg *config.Config, services *allServices) {
    controller.AddWebRouterForOrderController(rg, services.orderApp)
}
```

---

## App Service 接口 + 实现同文件（app/app.go）

```go
package app

// 接口（公有）
type OrderAppService interface {
    Create(ctx context.Context, cmd *CmdToCreateOrder, user *userdomain.User) (CreateOrderResultDTO, error)
    Cancel(ctx context.Context, id int64, user *userdomain.User) error
    Get(ctx context.Context, id int64, user *userdomain.User) (OrderDTO, error)
    List(ctx context.Context, cmd *CmdToListOrders) (OrdersDTO, error)
}

// 实现（私有）
type orderApp struct {
    repo   repository.Order
    tx     commonrepo.Transaction
}

// 工厂函数（返回接口类型，便于外部 Mock）
func NewOrderAppService(
    repo repository.Order,
    tx   commonrepo.Transaction,
) OrderAppService {
    return &orderApp{repo: repo, tx: tx}
}
```

> 接口与实现定义在同一文件，避免读代码时来回跳转。

---

## App Service 间依赖

允许 App Service 依赖其他 App Service，但必须通过接口：

```go
// ✅ 正确：通过接口依赖
type errataApp struct {
    repo      repository.Errata
    reviewApp ErrataReviewApp   // 接口类型
}

// ❌ 错误：直接依赖具体实现
type errataApp struct {
    reviewApp *errataReviewApp  // 具体类型
}
```

---

## Repository 实现工厂函数

Repository 实现的工厂函数返回**具体类型指针**（不是接口），同时执行数据库表迁移：

```go
// infrastructure/repositoryimpl/order.go
package repositoryimpl

// 返回具体类型，server 层接收后赋值给接口变量
func NewOrder(dao postgresql.Impl) (*orderImpl, error) {
    // AutoMigrate 在初始化时执行，确保表结构存在
    if err := postgresql.AutoMigrate(&orderDO{}); err != nil {
        return nil, err
    }
    return &orderImpl{dao: dao}, nil
}
```

在 `server/<module>.go` 中，具体类型自动满足 `repository.Order` 接口：

```go
repo, err := repositoryimpl.NewOrder(postgresql.DAO(cfg.Order.Repo.TableName))
if err != nil {
    return err
}
// 赋值给接口类型的字段
services.orderApp = app.NewOrderAppService(repo, tx)
```

---

## 启动代码模板

### main.go

```go
package main

import (
    "flag"
    "github.com/sirupsen/logrus"
    "your-project/server"
)

func main() {
    // 1. 解析命令行参数
    var (
        port       = flag.Int("port", 8080, "server port")
        configFile = flag.String("config-file", "config/config.yaml", "config file path")
    )
    flag.Parse()

    // 2. 初始化日志（必须在最早）
    logrus.SetFormatter(&logrus.JSONFormatter{})
    logrus.SetLevel(logrus.InfoLevel)

    // 3. 启动服务（加载配置 + 初始化依赖 + 启动 HTTP）
    if err := server.Start(*port, *configFile); err != nil {
        logrus.WithField("error", err.Error()).Fatal("server start failed")
    }
}
```

### server/gin.go（Gin 引擎与中间件初始化）

```go
package server

import (
    "github.com/gin-gonic/gin"
    "github.com/opensourceways/go-ddd-framework/controller/middleware/traceid"
    "github.com/opensourceways/go-ddd-framework/controller/middleware/ratelimiter"
    "github.com/opensourceways/go-ddd-framework/controller/middleware/securityheader"
)

func newGinEngine(cfg *config.Config) *gin.Engine {
    engine := gin.New()

    // 中间件注册顺序（固定，不得随意调整）
    engine.Use(
        gin.Recovery(),                            // 1. 必须第一：捕获 panic
        traceid.TraceID(),                         // 2. 注入 trace_id
        logRequest(),                              // 3. 请求日志
        ratelimiter.Handler(),                     // 4. 限流（需提前调用 ratelimiter.Init）
        securityheader.SetNormalAPIRespHeader,     // 5. 安全响应头
    )

    return engine
}

// logRequest：记录每个请求的基本信息
func logRequest() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()

        traceID := c.GetString("trace_id")
        logrus.WithFields(logrus.Fields{
            "trace_id":    traceID,
            "method":      c.Request.Method,
            "path":        c.Request.URL.Path,
            "status_code": c.Writer.Status(),
        }).Info("request completed")
    }
}
```

