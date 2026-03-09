# 框架库 API 参考

框架库：`github.com/opensourceways/go-ddd-framework`

本文档是框架导出 API 的权威参考，写代码时直接按此签名调用，无需推断。

---

## allerror 包

### 通用构造函数

```go
import "github.com/opensourceways/go-ddd-framework/allerror"

// 创建通用业务错误（HTTP 400）
// code：蛇形字符串，如 "user_not_found"
// msg：用户可读描述，为空时自动将 code 转为可读形式
// err：底层原始错误，仅用于诊断，不暴露给用户
func New(code, msg string, err error) errorImpl
```

### 三种特定错误

```go
// 资源不存在 → HTTP 404
func NewNotFoundError(code, msg string, err error) notFoundError
func IsNotFoundError(err error) bool

// 无权限 → HTTP 403
// 注意：没有 code 参数，错误码固定为 "no_permission"
func NewNoPermission(msg string, err error) noPermissionError
func IsNoPermission(err error) bool

// 超出限制 → HTTP 400
// 注意：没有 code 参数，错误码固定为 "over_limited"
func NewOverLimitError(msg string, err error) overLimitedError
```

### 错误码常量惯例

```go
// 内部错误码（包内使用，小写开头）
const errorCodeXxx = "xxx_yyy"

// 对外错误码（供 App 层引用，大写开头）
const ErrorCodeUserNotFound = "user_not_found"
```

---

## repository 包

### 事务接口

```go
import "github.com/opensourceways/go-ddd-framework/repository"

type Transaction interface {
    Do(ctx context.Context, f func(ctx context.Context) error) error
}
```

### Repository 层底层错误类型

这三种错误**只在 Repository 实现层使用**，App 层负责转换为 allerror：

```go
// 资源不存在（由 gorm.ErrRecordNotFound 转换而来）
type ErrorResourceNotFound struct{ error }
func NewErrorResourceNotFound(err error) ErrorResourceNotFound
func IsErrorResourceNotFound(err error) bool

// 重复创建（由 UNIQUE 约束错误转换而来）
type ErrorDuplicateCreating struct{ error }
func NewErrorDuplicateCreating(err error) ErrorDuplicateCreating
func IsErrorDuplicateCreating(err error) bool

// 并发更新冲突（由乐观锁 RowsAffected == 0 转换而来）
type ErrorConcurrentUpdating struct{ error }
func NewErrorConcurrentUpdating(err error) ErrorConcurrentUpdating
func IsErrorConcurrentUpdating(err error) bool
```

---

## postgresql 包

### CommonModel（嵌入到所有 DO 模型）

```go
import "github.com/opensourceways/go-ddd-framework/postgresql"

// 禁止使用 gorm.Model，统一使用 CommonModel
type CommonModel struct {
    ID        int64     `gorm:"primarykey"`
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

### DAO 工厂函数

```go
// table：数据库表名
func DAO(table string) *daoImpl
```

### DAO Impl 接口

```go
type Impl interface {
    // 查询条件构造
    EqualQuery(field string) string           // "field = ?"
    NotEqualQuery(field string) string        // "field <> ?"
    BetweenQuery(field string) string         // "field BETWEEN ? AND ?"
    MultiEqualQuery(fields ...string) string  // "f1 = ? AND f2 = ?"
    ORQuery(fields ...string) string          // "f1 = ? OR f2 = ?"
    AndQuery(conditions ...string) string     // 合并多个条件

    // 高级过滤
    InFilter(field string) string                                               // "field IN ?"
    NotIN(field string) string                                                  // "field NOT IN (?)"
    LikeFilter(field, value string) (query, arg string)                        // LIKE 模糊查询
    IntersectionFilter(field string, value []string) (string, pq.StringArray)  // 数组包含

    // 排序
    OrderByDesc(field string) string  // "field desc"
    OrderByAsc(field string) string   // "field asc"

    // CRUD（第一参数必须是 context.Context）
    GetRecord(ctx context.Context, filter, result interface{}) error
    GetByPrimaryKey(ctx context.Context, row interface{}) error
    DeleteByPrimaryKey(ctx context.Context, row interface{}) error

    // 关键：从 context 提取事务对象（两步操作缺一不可）
    New(ctx context.Context) Impl

    // GORM 原生访问
    DB() *gorm.DB
    WithContext(ctx context.Context) *gorm.DB
    TableName() string

    // 检测 UNIQUE 约束冲突（用于识别重复创建错误）
    IsRecordExists(err error) bool
}
```

### 事务创建

```go
func NewTransaction() transactionImpl
```

### 分页工具函数

```go
// 计算 GORM Offset 值
// pageNum 从 1 开始
func Offset(pageNum, countPerPage int) int
```

### 数据库初始化

```go
type Config struct {
    Host    string    `json:"host"`
    User    string    `json:"user"`
    Pwd     string    `json:"pwd"`
    Name    string    `json:"name"`
    Port    int       `json:"port"`
    Dbcert  string    `json:"cert"`         // SSL 证书路径
    Life    int       `json:"life"`         // 连接生命周期（分钟，默认 2）
    MaxConn int       `json:"max_conn"`     // 最大连接数（默认 500）
    MaxIdle int       `json:"max_idle"`     // 最大空闲连接（默认 250）
    Debug   bool      `json:"debug"`        // 是否启用 GORM debug 日志
}

func Init(cfg *Config, removeCfg bool) error
func DB() *gorm.DB
func AutoMigrate(table interface{}) error
```

---

## controller 包

### 统一响应格式

```go
import "github.com/opensourceways/go-ddd-framework/controller"

type ResponseData struct {
    Code    string      `json:"code"`      // 空字符串表示成功
    Msg     string      `json:"msg"`
    Data    interface{} `json:"data"`
    TraceID string      `json:"trace_id"`
}
```

### 成功响应函数

```go
func SendRespOfGet(ctx *gin.Context, data interface{})    // HTTP 200
func SendRespOfPost(ctx *gin.Context, data interface{})   // HTTP 201
func SendRespOfPut(ctx *gin.Context, data interface{})    // HTTP 202
func SendRespOfDelete(ctx *gin.Context)                   // HTTP 204
```

### 错误响应函数

```go
// 自动按错误类型映射 HTTP 状态码：
//   NotFound()     → 404
//   NoPermission() → 403
//   ErrorCode()    → 400
//   其他           → 500
func SendError(ctx *gin.Context, err error)

// 请求体绑定失败（HTTP 400）
func SendBadRequestBody(ctx *gin.Context, err error)

// 请求参数错误（HTTP 400）
func SendBadRequestParam(ctx *gin.Context, err error)
```

### 请求解析辅助

```go
// 解析路径参数 :id（int64 类型）
// 失败时直接调用 SendBadRequestParam，调用方检查 error 后 return 即可
func GetIndex(ctx *gin.Context) (int64, error)
```

### 分页请求绑定

```go
type ReqToPaginate struct {
    Count        bool `form:"count"`
    PageNum      int  `form:"page_num"`
    CountPerPage int  `form:"count_perpage"`
}

type PaginationConfig struct {
    MaxPageNum      int `json:"max_page_num"`       // 默认 10000
    MaxCountPerPage int `json:"max_count_per_page"` // 默认 100
}

// 转换为 dp.Pagination，内部自动校验边界
func (req *ReqToPaginate) ToPagination(cfg *PaginationConfig) dp.Pagination
```

### 排序请求绑定

```go
type ReqToOrder struct {
    Order string `form:"order"`    // "ASC" 或 "DESC"（不区分大小写）
    By    string `form:"orderby"`  // 排序字段名（由调用方负责防 SQL 注入）
}

// 返回 "field ASC|DESC"，字段或排序方向为空时返回 ("", nil)
func (req *ReqToOrder) ToOrder() (string, error)
```

### 文件上传

```go
// 验证并上传单文件
// fileParameter：表单参数名
// size：最大文件大小（字节）
// contentTypeMap：允许的 MIME 类型，key 为 MIME，value 为扩展名
// upload：实际上传实现（接收生成的文件名和读取流）
// 返回：生成的文件名（时间戳+MD5）或错误
func UploadSingleFile(
    c *gin.Context,
    fileParameter string,
    size int64,
    contentTypeMap map[string]string,
    upload func(string, io.Reader) error,
) (string, error)

// 检测文件 MIME 类型
func GetFileType(file *multipart.FileHeader) (string, error)
```

### 心跳检查路由

```go
// 注册 GET /v1/heartbeat，响应 {"data":{"status":"good"},...}
func AddRouterForHeartbeatController(r *gin.RouterGroup)
```

---

## controller/middleware 包

### Trace ID 中间件

```go
import "github.com/opensourceways/go-ddd-framework/controller/middleware/traceid"

// 必须作为第一个中间件注册
// 从 X-Request-Id 请求头读取，缺失时生成 UUID
// 存入：c.Set("trace_id", traceID)
// 写回：c.Header("X-Request-Id", traceID)
func TraceID() gin.HandlerFunc
```

### 安全响应头中间件

```go
import "github.com/opensourceways/go-ddd-framework/controller/middleware/securityheader"

// 普通 API 接口：设置 X-XSS-Protection、X-Frame-Options、CSP 等
func SetNormalAPIRespHeader(c *gin.Context)

// 文件下载接口：设置适合文件下载的缓存和安全头
func SetFileAPIRespHeader(c *gin.Context)
```

### 限流中间件

```go
import "github.com/opensourceways/go-ddd-framework/controller/middleware/ratelimiter"

type Config struct {
    MaxCASMultiplier int           `json:"max_cas_multiplier"` // 默认 100
    RateLimit        []routeConfig `json:"rate_limit"`
}

// 必须在路由注册前调用
func Init(client *redis.Client, config *Config) error

// 超限时返回 allerror.NewOverLimitError()（HTTP 400）
func Handler() func(*gin.Context)
```

### 内部服务认证中间件

```go
import "github.com/opensourceways/go-ddd-framework/controller/middleware/internalservice"

type Config struct {
    Salt      string `json:"salt"`       // base64 编码的盐
    TokenHash string `json:"token_hash"` // PBKDF2-SHA256 哈希值
}

// 初始化后 Handler 变量可用
func Init(cfg *Config)
var Handler func(*gin.Context)
// 认证失败返回 allerror.New("token_invalid", ...)（HTTP 400）
```

---

## dp 包

```go
import "github.com/opensourceways/go-ddd-framework/dp"

type Pagination struct {
    Count        bool
    PageNum      int
    CountPerPage int
}

// 验证并构造排序命令
// 返回格式："field ASC" 或 "field DESC"，两者为空时返回 ("", nil)
func NewOrderCmd(orderBy, order string) (string, error)
```

---

## 初始化顺序参考

```go
// server/init.go
func Init() error {
    // 1. 数据库（最底层依赖）
    if err := postgresql.Init(&cfg.DB, true); err != nil {
        return err
    }

    // 2. 限流（依赖 Redis）
    if err := ratelimiter.Init(redisClient, &cfg.RateLimit); err != nil {
        return err
    }

    // 3. 内部服务认证
    internalservice.Init(&cfg.InternalService)

    // 4. 业务模块（依赖数据库）
    initOrder()
    initUser()

    return nil
}

func initOrder() {
    dao := postgresql.DAO("orders")
    repo := repositoryimpl.NewOrderRepo(dao)
    tx := postgresql.NewTransaction()
    svc := app.NewOrderAppService(repo, tx)
    controller.NewOrderHandler(svc)
}
```
