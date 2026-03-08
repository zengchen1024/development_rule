# 错误处理规范

## 错误包结构

自定义 error 包位于 `common/domain/allerror`，核心结构如下：

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
    ErrorCodeUserNotFound      = "user_not_found"
    ErrorCodeUserNoPermission  = "user_no_permission"
    ErrorCodeOrderCreateFailed = "order_create_failed"
)
```

- 内部错误码（`errorCode` 前缀）：包内使用，不跨包引用
- 对外错误码（`ErrorCode` 前缀）：供 App 层或 Controller 层引用

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
func NewNoPermissionError(msg string, err error) error
func IsNoPermissionError(err error) bool

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

Repository 层返回以下底层错误类型，不使用 allerror 包：

```go
type ErrorResourceNotFound struct{ error }
type ErrorDuplicateCreating struct{ error }
type ErrorConcurrentUpdating struct{ error }
```

App 层（或 Domain 层）负责将底层错误**显式转换**为业务错误：

```go
// 正确：显式转换，保留调用链
func (s *UserService) GetUser(id int64) (*User, error) {
    user, err := s.repo.Find(id)
    if err != nil {
        if repository.IsErrorResourceNotFound(err) {
            return nil, allerror.NewNotFoundError(
                allerror.ErrorCodeUserNotFound, "", err,
            )
        }
        return nil, allerror.New("user_query_failed", "", err)
    }
    return user, nil
}

// 禁止：直接透传底层错误
func (s *UserService) GetUser(id int64) (*User, error) {
    return s.repo.Find(id) // 不要这样
}
```

## Controller 层错误响应

Controller 统一调用 `SendError()`，内部通过 `errors.As()` 映射 HTTP 状态码：

```go
// HTTP 状态码映射规则
func httpStatusCode(err error) int {
    if errors.As(err, &notFoundError{}) {
        return http.StatusNotFound  // 404
    }
    if errors.As(err, &noPermissionError{}) {
        return http.StatusForbidden // 403
    }
    if errors.As(err, &errorCode(nil)) {
        return http.StatusBadRequest // 400：其他已知业务错误
    }
    return http.StatusInternalServerError // 500：未预期错误
}
```

Handler 层统一写法：

```go
func (h *UserHandler) GetUser(c *gin.Context) {
    traceID := c.GetString("trace_id")
    user, err := h.service.GetUser(id)
    if err != nil {
        logrus.WithFields(logrus.Fields{
            "trace_id": traceID,
            "error":    err.Error(),
        }).Error("get user failed")
        SendError(c, err) // 内部处理状态码和响应格式
        return
    }
    SendRespOfGet(c, user)
}
```

## 日志记录原则

- **只在 Handler 层**记录 Error 日志，底层只包装错误向上传递
- `innerErr`（底层原始错误）通过 `err.Error()` 透传到日志，不单独暴露给用户

## 禁止行为

- 不得 `_ = someFunc()` 忽略错误返回值
- 不得仅打印错误而不返回（`log.Error(err); return nil`）
- 不得在错误信息中包含用户输入的原始内容（防止日志注入）
- 不得跨越层级直接抛出底层错误（如把数据库错误直接返回到 Controller）
