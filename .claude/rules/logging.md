# 日志规范

## 基本要求

- 统一使用 `github.com/sirupsen/logrus`
- 所有日志必须结构化输出（JSON 格式），禁止使用 `logrus.Info("文字 " + variable)` 拼接方式
- 生产环境日志级别为 `Info` 及以上，调试日志使用 `Debug`（默认关闭）

## 必须包含的字段

每条日志必须包含 `trace_id`，从 Gin 上下文中取出（由 TraceID 中间件注入）：

```go
// 正确示例
traceID := c.GetString("trace_id")
logrus.WithFields(logrus.Fields{
    "trace_id": traceID,
    "user_id":  userID,
}).Info("user login success")

// 禁止：缺少 trace_id
logrus.Info("user login success")

// 禁止：字符串拼接
logrus.Info("user " + userID + " login success")
```

## 日志级别使用规范

| 级别 | 使用场景 |
|------|---------|
| `Error` | 错误已影响业务流程，需要人工介入 |
| `Warn` | 异常但不影响主流程（如降级、重试成功） |
| `Info` | 关键业务节点（请求进入、请求完成、关键状态变更） |
| `Debug` | 调试信息，生产环境关闭 |

## 禁止行为

- 不得在循环内大量打印 Info/Error 日志（使用 Debug 或聚合后打印）
- 不得记录密码、Token、完整身份证号等敏感字段
- 不得使用 `fmt.Println` 或 `log.Println` 代替 logrus

## 初始化约定

```go
// 应在 main.go 启动时统一初始化
logrus.SetFormatter(&logrus.JSONFormatter{})
logrus.SetLevel(logrus.InfoLevel) // 从配置读取
```
