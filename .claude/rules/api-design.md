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
  "code": "",
  "msg": "success",
  "data": {},
  "trace_id": "abc123"
}
```

- `code`：业务错误码，空字符串表示成功，非空字符串表示失败（使用蛇形字符串，如 `"user_not_found"`）
- `msg`：人类可读的错误描述
- `data`：业务数据，失败时可为 null
- `trace_id`：必须从请求上下文中取出并返回，不得硬编码或省略

## Trace ID 注入

trace_id 必须通过中间件统一注入到 Gin 上下文，Handler 层直接读取，不得自行生成。

**中间件实现规范：**

```go
// middleware/traceid/middleware.go
func TraceID() gin.HandlerFunc {
    return func(c *gin.Context) {
        traceID := c.GetHeader("X-Request-Id")
        if traceID == "" {
            traceID = uuid.NewString() // 客户端未传入时自动生成
        }
        c.Set("trace_id", traceID)
        c.Header("X-Request-Id", traceID) // 写回响应头，方便链路追踪
        c.Next()
    }
}
```

**注册顺序**（必须在所有业务中间件之前）：

```go
engine.Use(middleware.TraceID(), gin.Recovery(), ...)
```

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

遵循 REST 语义，使用 HTTP 状态码反映请求结果：

| 状态码 | 含义 | 使用场景 |
|--------|------|---------|
| `200` | 成功 | 请求正常处理完成 |
| `400` | 参数或业务错误 | 参数绑定失败、业务规则不满足 |
| `401` | 未认证 | 未登录或 Token 无效 |
| `403` | 无权限 | 已认证但无操作权限 |
| `404` | 资源不存在 | 指定资源未找到（含路由不存在） |
| `500` | 服务内部错误 | 未预期异常，需人工介入 |

- 业务错误通过 `code` 字段（错误码字符串）进一步区分
- **禁止**将所有错误统一返回 `200` 再靠 `code` 区分，这违反 HTTP 语义

## 路由命名

- 使用 REST 风格：`/api/v1/users/{id}`
- 版本号必须在路径中体现（`/v1/`）
- 资源名用复数小写，单词间用 `-` 分隔
