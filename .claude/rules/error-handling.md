# 错误处理规范

## 错误包使用

- 使用项目内自定义 error 包（通常位于 `pkg/errors` 或 `internal/errors`）
- 禁止直接使用 `errors.New` 或 `fmt.Errorf` 创建业务错误
- 跨层传递错误时必须包装，保留调用链信息

## 错误分类

自定义 error 包应支持以下分类（若项目包尚未支持，创建新代码时补充）：

```go
// 参考结构
type AppError struct {
    Code    int    // 业务错误码，与 API 响应 code 对应
    Message string // 面向用户的描述
    Err     error  // 原始错误（不对外暴露）
}
```

## 错误传递规范

```go
// 正确：包装错误，保留上下文
func (s *UserService) GetUser(ctx context.Context, id int64) (*User, error) {
    user, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return nil, errors.Wrap(err, "UserService.GetUser: repo.FindByID failed")
    }
    return user, nil
}

// 禁止：直接返回底层错误，丢失调用链
func (s *UserService) GetUser(ctx context.Context, id int64) (*User, error) {
    return s.repo.FindByID(ctx, id) // 不要这样
}
```

## 日志记录位置

- **只在最顶层**（Handler 层）记录 Error 日志，避免重复记录
- 底层函数只负责包装错误并向上返回

```go
// Handler 层：记录日志 + 返回响应
func (h *UserHandler) GetUser(c *gin.Context) {
    user, err := h.service.GetUser(c.Request.Context(), id)
    if err != nil {
        logrus.WithContext(c.Request.Context()).WithFields(logrus.Fields{
            "trace_id": c.GetString("trace_id"),
            "error":    err.Error(),
        }).Error("get user failed")
        c.JSON(http.StatusOK, errorResponse(err))
        return
    }
    // ...
}
```

## 禁止行为

- 不得 `_ = someFunc()` 忽略错误返回值
- 不得仅打印错误而不返回（`log.Error(err); return nil`）
- 不得在错误信息中包含用户输入的原始内容（防止日志注入）
