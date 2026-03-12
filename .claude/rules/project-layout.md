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
    │   ├── service/                 # 领域服务（跨聚合操作，按需创建）
    │   │   └── <service>.go
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

### 命令行参数规范

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `--port` | int | 否 | 监听端口，默认 8888 |
| `--config-file` | string | **是** | 配置文件路径，启动时校验非空 |
| `--tls-cert` | string | 否 | TLS 证书文件路径，为空时使用 HTTP |
| `--tls-key` | string | 否 | TLS 私钥文件路径 |
| `--grace-period` | duration | 否 | 优雅关闭等待时长，默认 180s |
| `--rm-config` | bool | 否 | 启动后删除配置文件（K8s Secret 挂载场景） |
| `--enable-debug` | bool | 否 | 开启 Debug 日志级别 |

**约束：**
- 通过 `options` struct 集中管理，提供 `addFlags()` 和 `validate()` 方法，不得散放在 `main()` 中
- `validate()` 必须校验 `--config-file` 非空，缺失则报错退出
- **禁止使用 `os.Getenv()` 读取任何配置**（含敏感信息），所有配置统一通过 `--config-file` 指定的文件传递
- 配置文件在 K8s 场景下通过 Secret 挂载，启动后由 `--rm-config` 控制是否删除

### main.go

```go
package main

import (
    "flag"
    "fmt"
    "os"
    "time"

    "github.com/sirupsen/logrus"
    "your-project/config"
    "your-project/server"
)

const (
    defaultPort        = 8888
    defaultGracePeriod = 180 // 秒
)

type options struct {
    server      server.ServerOptions
    configFile  string
    removeCfg   bool
    enableDebug bool
}

// validate 校验必填参数，并将 removeCfg 同步到 ServerOptions
func (o *options) validate() error {
    if o.configFile == "" {
        return fmt.Errorf("missing --config-file")
    }
    o.server.RemoveCfg = o.removeCfg
    return nil
}

func (o *options) addFlags(fs *flag.FlagSet) {
    fs.IntVar(&o.server.Port, "port", defaultPort, "port to listen on")
    fs.StringVar(&o.server.Cert, "tls-cert", "", "TLS cert file path (empty = HTTP)")
    fs.StringVar(&o.server.Key, "tls-key", "", "TLS key file path")
    fs.DurationVar(
        &o.server.GracePeriod, "grace-period",
        time.Duration(defaultGracePeriod)*time.Second,
        "graceful shutdown wait duration",
    )
    fs.StringVar(&o.configFile, "config-file", "", "path to config file (required)")
    fs.BoolVar(&o.removeCfg, "rm-config", false, "delete config file after loading")
    fs.BoolVar(&o.enableDebug, "enable-debug", false, "enable debug log level")
}

func gatherOptions(fs *flag.FlagSet, args ...string) (options, error) {
    var o options
    o.addFlags(fs)
    if err := fs.Parse(args); err != nil {
        return o, err
    }
    return o, o.validate()
}

func main() {
    o, err := gatherOptions(
        flag.NewFlagSet(os.Args[0], flag.ExitOnError),
        os.Args[1:]...,
    )
    if err != nil {
        logrus.Errorf("invalid options: %s", err.Error())
        return
    }

    logrus.SetFormatter(&logrus.JSONFormatter{})
    if o.enableDebug {
        logrus.SetLevel(logrus.DebugLevel)
    }

    cfg := new(config.Config)
    if err := config.LoadConfig(o.configFile, cfg, o.removeCfg); err != nil {
        logrus.Errorf("load config failed: %s", err.Error())
        return
    }

    server.StartWebServer(&o.server, cfg)
}
```

### config/config.go（配置加载）

Config 加载三步：读文件 → 设默认值 → 校验。

```go
package config

import (
    "os"

    commoncfg "your-project/common/config"
    "gopkg.in/yaml.v3"
)

// LoadConfig 从 YAML 文件加载配置，remove=true 时加载后删除文件（用于 K8s Secret 挂载场景）
func LoadConfig(path string, cfg *Config, remove bool) error {
    if remove {
        defer os.Remove(path)
    }

    b, err := os.ReadFile(path)
    if err != nil {
        return err
    }
    if err := yaml.Unmarshal(b, cfg); err != nil {
        return err
    }

    commoncfg.SetDefault(cfg)   // 递归调用各子配置的 SetDefault()
    return commoncfg.Validate(cfg) // 递归调用各子配置的 Validate()
}

type Config struct {
    Errata            errata.Config   `json:"errata"`
    ReadHeaderTimeout int             `json:"read_header_timeout"`
    // 其他模块配置...
}

// ConfigItems 返回所有子配置指针，供 SetDefault/Validate 递归处理
func (cfg *Config) ConfigItems() []interface{} {
    return []interface{}{
        &cfg.Errata,
        // 其他子配置...
    }
}

// SetDefault 设置顶层配置默认值
func (cfg *Config) SetDefault() {
    if cfg.ReadHeaderTimeout <= 0 {
        cfg.ReadHeaderTimeout = 10
    }
}
```

各模块子配置同样实现 `SetDefault()` 和 `Validate()` 方法（按需）。

### server/gin.go（启动与优雅关闭）

```go
package server

import (
    "crypto/tls"
    "fmt"
    "net/http"
    "os"
    "strings"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/opensourceways/server-common-lib/interrupts"
    "github.com/opensourceways/go-ddd-framework/controller/middleware/ratelimiter"
    "github.com/opensourceways/go-ddd-framework/controller/middleware/securityheader"
    "github.com/opensourceways/go-ddd-framework/controller/middleware/traceid"
    "github.com/sirupsen/logrus"
    "your-project/config"
)

const waitServerStart = 3 // 秒，等待服务完成 TLS 握手后再删证书

type ServerOptions struct {
    Port        int
    Cert        string           // TLS 证书路径，空字符串表示使用 HTTP
    Key         string           // TLS 私钥路径
    RemoveCfg   bool             // 启动后删除证书文件（K8s Secret 场景）
    GracePeriod time.Duration    // 优雅关闭等待时长，建议 180s
}

func (opt *ServerOptions) needTLS() bool {
    return opt.Key != "" && opt.Cert != ""
}

// clean 在 TLS 服务启动后删除证书和密钥文件，防止敏感文件驻留磁盘
// 仅当 RemoveCfg=true 时执行（对应 K8s Secret 挂载场景）
func (opt *ServerOptions) clean() {
    if !opt.RemoveCfg {
        return
    }

    time.Sleep(time.Duration(waitServerStart) * time.Second)

    if err := os.Remove(opt.Cert); err != nil {
        logrus.Fatalf("remove cert file: %s", err.Error())
    }

    if err := os.Remove(opt.Key); err != nil {
        logrus.Fatalf("remove key file: %s", err.Error())
    }
}

// StartWebServer 初始化服务并启动，内置优雅关闭
func StartWebServer(opt *ServerOptions, cfg *config.Config) {
    services, err := initServices(cfg)
    if err != nil {
        logrus.Error(err)
        return
    }
    defer exitService() // 关闭数据库连接等资源

    engine := gin.New()
    engine.UseRawPath = true

    // 中间件注册顺序（固定，不得随意调整）
    engine.Use(
        gin.Recovery(),                          // 1. 必须第一：捕获 panic
        traceid.TraceID(),                       // 2. 注入 trace_id
        logRequest(),                            // 3. 请求日志
        ratelimiter.Handler(),                   // 4. 限流
        securityheader.SetNormalAPIRespHeader,   // 5. 安全响应头
    )

    setRouters(engine.Group("/"), cfg, &services)

    srv := &http.Server{
        Addr:              fmt.Sprintf(":%d", opt.Port),
        Handler:           engine,
        ReadHeaderTimeout: time.Duration(cfg.ReadHeaderTimeout) * time.Second,
    }

    // 注册优雅关闭：等待 SIGTERM/SIGINT 后在 GracePeriod 内完成在途请求
    defer interrupts.WaitForGracefulShutdown()

    if !opt.needTLS() {
        interrupts.ListenAndServe(srv, opt.GracePeriod)
        return
    }

    srv.TLSConfig = &tls.Config{
        MinVersion:               tls.VersionTLS12,
        PreferServerCipherSuites: true,
        CipherSuites: []uint16{
            tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
            tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
        },
    }
    interrupts.ListenAndServeTLS(srv, opt.Cert, opt.Key, opt.GracePeriod)

    opt.clean()
}

// logRequest 记录每个请求的基本信息，心跳请求不记录
func logRequest() gin.HandlerFunc {
    return func(c *gin.Context) {
        traceID := c.GetString("trace_id")
        start := time.Now()
        c.Next()

        // 心跳检查请求不记录，避免日志噪音
        if strings.Contains(c.Request.RequestURI, "/v1/heartbeat") {
            return
        }

        logrus.WithFields(logrus.Fields{
            "trace_id": traceID,
            "status":   c.Writer.Status(),
            "duration": time.Since(start).Milliseconds(),
            "method":   c.Request.Method,
            "uri":      c.Request.RequestURI,
        }).Info("request completed")
    }
}
```

> `interrupts.WaitForGracefulShutdown()` 监听 SIGTERM/SIGINT，收到信号后等待 `GracePeriod` 内所有在途请求处理完毕再退出。K8s 的 `terminationGracePeriodSeconds` 应大于此值。

