# API 设计规范

---
paths:
  - "**/*handler*.go"
  - "**/*controller*.go"
  - "**/api/**/*.go"
  - "**/router*.go"
---

## 统一响应格式

所有接口返回必须使用以下结构：

```json
{
  "code": 0,
  "msg": "success",
  "data": {},
  "trace_id": "abc123"
}
```

- `code`：业务错误码，0 表示成功，非 0 表示失败
- `msg`：人类可读的错误描述
- `data`：业务数据，失败时可为 null
- `trace_id`：必须从请求上下文中取出并返回，不得硬编码或省略

## Gin Handler 规范

```go
// 正确示例
func (h *UserHandler) GetUser(c *gin.Context) {
    traceID := c.GetString("trace_id") // 从中间件注入的上下文获取
    // ...
    c.JSON(http.StatusOK, Response{
        Code:    0,
        Msg:     "success",
        Data:    user,
        TraceID: traceID,
    })
}
```

- Handler 函数体不超过 30 行，超过时拆分为 service 方法
- 不得在 Handler 层直接操作数据库
- 参数绑定必须使用 `ShouldBind` 系列，并处理绑定错误

## HTTP 状态码使用

- 业务逻辑错误统一返回 `200`，通过 `code` 字段区分
- 仅以下情况使用非 200 状态码：
  - `401`：未认证
  - `403`：无权限
  - `500`：服务内部未预期错误
  - `404`：路由不存在（框架层处理）

## 路由命名

- 使用 REST 风格：`/api/v1/users/{id}`
- 版本号必须在路径中体现（`/v1/`）
- 资源名用复数小写，单词间用 `-` 分隔
