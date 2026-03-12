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

**直接使用框架中间件，不得自行实现：**

```go
// server/gin.go - 中间件注册顺序
engine.Use(
    gin.Recovery(),                            // 1. 必须第一个：捕获 panic 防止崩溃
    traceid.TraceID(),                         // 2. 注入 trace_id（Recovery 之后）
    logRequest(),                              // 3. 请求日志（可访问 trace_id）
    ratelimiter.Handler(),                     // 4. 限流（按需）
    securityheader.SetNormalAPIRespHeader,     // 5. 安全响应头
)
```

中间件行为：
1. 从 `X-Request-Id` 请求头读取 trace_id
2. 缺失时自动生成 UUID
3. 存入 `c.Set("trace_id", traceID)`
4. 写回 `X-Request-Id` 响应头

**Handler 层读取方式：**

```go
traceID := c.GetString("trace_id")
```

## Gin Handler 规范

Handler 层统一使用框架提供的响应函数，不得手动调用 `c.JSON()`：

```go
// ✅ 正确：使用框架响应函数
func (h *UserHandler) GetUser(c *gin.Context) {
    id, err := controller.GetIndex(c) // 解析路径参数 :id
    if err != nil {
        return // GetIndex 内部已调用 SendBadRequestParam
    }

    user, err := h.svc.GetUser(c.Request.Context(), id)
    if err != nil {
        controller.SendError(c, err) // 自动映射 HTTP 状态码
        return
    }
    controller.SendRespOfGet(c, user)
}

// ❌ 错误：手动构造响应
func (h *UserHandler) GetUser(c *gin.Context) {
    c.JSON(http.StatusOK, map[string]interface{}{"data": user})
}
```

**框架响应函数对应关系：**

| 场景 | 函数 | HTTP 状态码 |
|------|------|-----------|
| GET 成功 | `controller.SendRespOfGet(c, data)` | 200 |
| POST 成功 | `controller.SendRespOfPost(c, data)` | 201 |
| PUT 成功 | `controller.SendRespOfPut(c, data)` | 202 |
| DELETE 成功 | `controller.SendRespOfDelete(c)` | 204 |
| 请求体绑定失败 | `controller.SendBadRequestBody(c, err)` | 400 |
| 请求参数错误 | `controller.SendBadRequestParam(c, err)` | 400 |
| 业务/系统错误 | `controller.SendError(c, err)` | 自动映射 |

- Handler 函数体不超过 30 行，超过时拆分为 service 方法
- 不得在 Handler 层直接操作数据库
- 参数绑定必须使用 `ShouldBind` 系列，并处理绑定错误
- 必须调用 `c.Request.Context()` 传递给 App 层（不得传 `context.Background()`）

## 分页与排序接口规范

### 分页参数

使用框架提供的 `controller.ReqToPaginate` 绑定，**参数名不得自行设计**：

```go
// 请求参数：?page_num=1&count_perpage=20&count=true
type reqToListOrders struct {
    controller.ReqToPaginate        // 嵌入分页参数
    Status string `form:"status"`   // 业务过滤参数
}

var paginationCfg = &controller.PaginationConfig{
    MaxPageNum:      10000,
    MaxCountPerPage: 100,
}

func (h *OrderHandler) ListOrders(c *gin.Context) {
    var req reqToListOrders
    if err := c.ShouldBindQuery(&req); err != nil {
        controller.SendBadRequestParam(c, err)
        return
    }
    pagination := req.ToPagination(paginationCfg) // 自动校验边界
    // ...
}
```

**固定参数名：**
- `page_num`：页码（从 1 开始）
- `count_perpage`：每页数量
- `count`：是否返回总数（`true`/`false`）

### 排序参数

使用框架提供的 `controller.ReqToOrder` 绑定：

```go
// 请求参数：?orderby=created_at&order=DESC
type reqToListOrders struct {
    controller.ReqToOrder
    // ...
}

func (h *OrderHandler) ListOrders(c *gin.Context) {
    var req reqToListOrders
    // ...
    orderCmd, err := req.ToOrder() // 返回 "created_at DESC" 或 ""
    if err != nil {
        controller.SendBadRequestParam(c, err)
        return
    }
    // 将 orderCmd 传入 App 层，由 Repository 层防注入
}
```

**固定参数名：**
- `orderby`：排序字段
- `order`：排序方向（`ASC` 或 `DESC`，不区分大小写）

## 列表接口响应格式

列表接口的 `data` 字段使用**具名结构体**，不使用裸数组：

```go
// ✅ 正确：具名字段，包含总数
type OrdersDTO struct {
    Orders []OrderInfo `json:"orders"`
    Total  int64       `json:"total"`
}

// 响应示例
{
  "code": "",
  "msg": "",
  "data": {
    "orders": [...],
    "total": 100
  },
  "trace_id": "abc123"
}

// ❌ 错误：裸数组（无法携带 total，也无法扩展）
// "data": [...]
```

- 列表字段名使用**复数形式**，与资源名一致（`orders`、`erratas`）
- `total` 字段是否返回由请求参数 `count=true` 控制（使用框架 `ReqToPaginate`）
- 当 `count=false` 时，`total` 返回 `0`，调用方忽略此值

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

## 心跳检查路由

每个服务必须注册心跳路由，**注册在独立路由组**（与业务接口隔离），直接使用框架函数：

```go
import "github.com/opensourceways/go-ddd-framework/controller"

// server/router.go
// 心跳接口注册在 /internal 前缀，与业务接口分组隔离
func setInternalRouter(prefix string, engine *gin.Engine, ...) {
    rg := engine.Group(prefix)
    controller.AddRouterForHeartbeatController(rg) // GET /internal/v1/heartbeat
}
```

K8s 的 `livenessProbe` 和 `readinessProbe` 指向此路由。

### 心跳接口访问频率配置

心跳接口的访问频率通过 `ratelimiter.Config` 单独配置，在 `rate_limit` 列表中为其指定独立限流规则：

```yaml
# config.yaml
common:
  rate_limiter:
    rate_limit:
      - route: "/internal/v1/heartbeat"
        burst_num: 1    # 瞬时最大突发请求数
        request_num: 20 # 每秒最大请求数
```

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `route` | — | 路由前缀，精确匹配请求 URI |
| `request_num` | 500 | 每秒最大请求数（令牌桶补充速率） |
| `burst_num` | 100 | 瞬时突发允许量（令牌桶容量） |

- 不配置时，心跳接口使用 `default` 规则（`request_num: 500`，`burst_num: 100`）
- 心跳接口流量由 K8s 探针产生，`request_num` 建议设为探针频率的 2-3 倍

## API 文档同步

使用 `swaggo/swag` 从代码注释自动生成 Swagger 文档，**每次新增、修改或删除接口后必须重新生成**。

### 注释规范

Handler 函数必须包含完整的 swag 注释：

```go
// @Summary		ListPods
// @Description	list pod records of the current user in the past month
// @Tags			Cloud
// @Param			page_num		query	int	false	"page number (1-based)"
// @Param			count_per_page	query	int	false	"items per page"
// @Accept			json
// @Success		200	{object}	app.UserPodsDTO
// @Failure		400	{object}	responseData	"bad request"
// @Failure		500	{object}	responseData	"system error"
// @Router			/v1/cloud/pod/history [get]
```

### 生成命令

在项目根目录执行：

```bash
swag init
```

生成产物为 `docs/docs.go`、`docs/swagger.json`、`docs/swagger.yaml`，**三个文件必须一并提交**。

### 工具版本

项目使用 `github.com/swaggo/swag` v1.16.x，`go.mod` 中锁定版本。本地安装：

```bash
go install github.com/swaggo/swag/cmd/swag@$(grep swaggo/swag go.mod | awk '{print $2}')
```

> **注意**：v1.16.2 存在解析特定注释时 panic 的 bug，请使用 v1.16.4 及以上版本。

### CI 检查（推荐）

在 CI 流水线中加入文档一致性检查，防止接口改动未同步文档：

```bash
swag init
git diff --exit-code docs/
```

## 安全响应头

**所有接口必须设置安全响应头**，使用框架中间件：

```go
import "github.com/opensourceways/go-ddd-framework/controller/middleware/securityheader"

// 普通 API（绝大多数接口）
c.Use(securityheader.SetNormalAPIRespHeader)

// 文件下载接口
c.Use(securityheader.SetFileAPIRespHeader)
```

设置的头包括：`X-XSS-Protection`、`X-Frame-Options: DENY`、`X-Content-Type-Options: nosniff`、`Strict-Transport-Security`、`Content-Security-Policy`。
