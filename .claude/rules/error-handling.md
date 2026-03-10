# 错误处理规范

## 错误包结构

自定义 error 包来自框架库 `github.com/opensourceways/go-ddd-framework/allerror`，核心结构如下：

```go
// 基础实现，不对外暴露
type errorImpl struct {
    code     string // 蛇形字符串错误码，与 API 响应 code 字段对应
    msg      string // 面向用户的描述，为空时自动将 code 转为可读形式
    innerErr error  // 底层原始错误，不对外暴露，仅用于诊断
}

func (e errorImpl) ErrorCode() string { return e.code }
func (e errorImpl) Error() string     { return e.msg } // 或 code 转换结果
```

- 禁止直接使用 `errors.New` 或 `fmt.Errorf` 创建业务错误
- 禁止在业务层直接返回底层错误（数据库错误、第三方 SDK 错误等）

## 错误码命名规范

错误码使用**蛇形字符串**，格式为 `<模块>_<动作/状态>`：

```go
const (
    // 内部使用（小写开头）
    errorCodeOverLimited  = "over_limited"
    errorCodeNoPermission = "no_permission"

    // 对外暴露（大写开头，供其他包引用）
    ErrorCodeUserNotFound         = "user_not_found"
    ErrorCodeUserNoPermission     = "user_no_permission"
    ErrorCodeUserAlreadyExists    = "user_already_exists"
    ErrorCodeUserConcurrentUpdate = "user_concurrent_update"
)
```

- 内部错误码（`errorCode` 前缀）：包内使用，不跨包引用
- 对外错误码（`ErrorCode` 前缀）：供 App 层或 Controller 层引用
- **只为有业务含义的错误定义错误码**，系统错误（如 DB 连接失败）不定义错误码

## 三种特定错误类型

通过**标记接口**区分错误类别，配合 `errors.As()` 进行类型判断：

```go
// 资源不存在 → HTTP 404
type notFoundError struct{ errorImpl }
func (e notFoundError) NotFound() {}
func NewNotFoundError(code, msg string, err error) error
func IsNotFoundError(err error) bool

// 无权限 → HTTP 403
type noPermissionError struct{ errorImpl }
func (e noPermissionError) NoPermission() {}
func NewNoPermission(msg string, err error) error
func IsNoPermission(err error) bool

// 超出限制 → HTTP 400
type overLimitedError struct{ errorImpl }
func (e overLimitedError) OverLimit() {}
func NewOverLimitError(msg string, err error) error
```

其余业务错误使用通用构造函数：

```go
func New(code, msg string, err error) error
```

## Repository 层错误处理

Repository 层返回以下三种"可预期"错误类型（来自 `go-ddd-framework/repository` 包），不使用 allerror：

```go
type ErrorResourceNotFound struct{ error }   // Find 时资源不存在
type ErrorDuplicateCreating struct{ error }   // Create 时唯一约束冲突
type ErrorConcurrentUpdating struct{ error }  // Save/Delete 时乐观锁冲突
```

**App 层只需转换"需要改变 HTTP 语义"的错误**。三种可预期错误若不转换，Controller 会把它们映射为 HTTP 500（系统错误），而实际应返回 404/400，因此必须显式转换。其他所有错误（如 DB 连接失败）本就是系统错误，直接返回即可，Controller 自动映射为 HTTP 500。

### 按操作类型的处理约定

**Find（查询单条）**：检查 `IsErrorResourceNotFound`，其余直接返回

```go
func (s *UserService) GetUser(ctx context.Context, id int64) (*domain.User, error) {
    user, err := s.repo.Find(ctx, id)
    if err != nil {
        if repository.IsErrorResourceNotFound(err) {
            return nil, allerror.NewNotFoundError(ErrorCodeUserNotFound, "", err)
        }
        return nil, err // 系统错误，Controller → HTTP 500
    }
    return user, nil
}
```

**Create（创建）**：检查 `IsErrorDuplicateCreating`，其余直接返回

```go
func (s *UserService) CreateUser(ctx context.Context, v *domain.User) error {
    err := s.repo.Add(ctx, v)
    if err != nil {
        if repository.IsErrorDuplicateCreating(err) {
            return allerror.New(ErrorCodeUserAlreadyExists, "", err) // HTTP 400
        }
        return err // 系统错误，Controller → HTTP 500
    }
    return nil
}
```

**Save/Delete（更新或删除）**：检查 `IsErrorConcurrentUpdating`，其余直接返回

```go
func (s *UserService) UpdateUser(ctx context.Context, v *domain.User) error {
    err := s.repo.Save(ctx, v)
    if err != nil {
        if repository.IsErrorConcurrentUpdating(err) {
            return allerror.New(ErrorCodeUserConcurrentUpdate, "please retry", err) // HTTP 400
        }
        return err // 系统错误，Controller → HTTP 500
    }
    return nil
}
```

**List（列表查询）**：不存在可预期的业务错误，直接返回

```go
func (s *UserService) ListUsers(ctx context.Context, cmd *CmdToListUsers) (UsersDTO, error) {
    items, total, err := s.repo.List(ctx, cmd)
    if err != nil {
        return UsersDTO{}, err // 系统错误，Controller → HTTP 500
    }
    // ... 转换为 DTO
}
```

**三类可预期错误的转换对照：**

| 底层错误 | 触发操作 | 转换为 | HTTP |
|----------|---------|--------|------|
| `ErrorResourceNotFound` | Find | `allerror.NewNotFoundError(code, "", err)` | 404 |
| `ErrorDuplicateCreating` | Create | `allerror.New(code, "", err)` | 400 |
| `ErrorConcurrentUpdating` | Save/Delete | `allerror.New(code, "", err)` | 400 |
| 其他（系统错误） | 任何操作 | 直接 `return err` | 500 |

## Controller 层错误响应

Controller 统一调用 `controller.SendError()`，内部自动映射 HTTP 状态码：

```
NotFound()     → HTTP 404
NoPermission() → HTTP 403
有 ErrorCode() → HTTP 400
其他           → HTTP 500
```

Handler 层统一写法：

```go
func (h *UserHandler) GetUser(c *gin.Context) {
    id, err := controller.GetIndex(c)
    if err != nil {
        return
    }

    traceID := c.GetString("trace_id")
    user, err := h.service.GetUser(c.Request.Context(), id)
    if err != nil {
        logrus.WithFields(logrus.Fields{
            "trace_id": traceID,
            "error":    err.Error(),
        }).Error("get user failed")
        controller.SendError(c, err) // 自动处理状态码和响应格式
        return
    }
    controller.SendRespOfGet(c, user)
}
```

## 日志记录原则

- **只在 Handler 层**记录 Error 日志，底层只包装错误向上传递
- `innerErr`（底层原始错误）通过 `err.Error()` 透传到日志，不单独暴露给用户

## 禁止行为

- 不得 `_ = someFunc()` 忽略错误返回值
- 不得仅打印错误而不返回（`log.Error(err); return nil`）
- 不得在错误信息中包含用户输入的原始内容（防止日志注入）
- 不得为系统错误伪造业务错误码（如 `allerror.New("user_query_failed", ...)` 包装 DB 连接错误）——系统错误直接返回，Controller 映射为 HTTP 500
- 不得遗漏可预期业务错误的转换（Find 时 `IsErrorResourceNotFound`、Create 时 `IsErrorDuplicateCreating`、Save/Delete 时 `IsErrorConcurrentUpdating` 必须检查）
