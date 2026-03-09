# DDD 分层架构规范

---
paths:
  - "**/*.go"
---

## 四层结构与依赖方向

```
Controller 层  →  App 层  →  Domain 层  ←  Infrastructure 层
```

依赖规则：外层依赖内层接口，禁止反向依赖。

| 层 | 目录 | 职责 |
|----|------|------|
| Controller | `*/controller/` | HTTP 参数绑定、认证、调用 App Service |
| App | `*/app/` | 用例编排、事务边界、错误转换 |
| Domain | `*/domain/` | 业务规则、聚合根、值对象、领域接口 |
| Infrastructure | `*/infrastructure/` | 接口实现（数据库、外部服务） |

## Domain 层规范

### 聚合根（Aggregate Root）

聚合根是业务规则的唯一执行者，负责维护自身一致性：

- 所有状态变更必须通过聚合根的方法完成，外部不得直接修改字段
- 聚合根方法负责校验前置条件，不满足时返回业务错误
- 通过工厂函数创建新实体，隐藏初始状态细节

```go
// ✅ 正确：状态转移和规则在聚合根方法中
func (v *Order) Cancel(user *User) error {
    if v.Status != StatusPending {
        return allerror.New("order_not_cancellable", "only pending orders can be cancelled", nil)
    }
    if v.CreatedBy != user.Id {
        return allerror.NewNoPermissionError("", nil)
    }
    v.Status = StatusCancelled
    return nil
}

// ❌ 错误：调用方绕过聚合根直接修改状态
func (s *orderApp) Cancel(ctx context.Context, id int64) error {
    order, _ := s.repo.Find(ctx, id)
    order.Status = "cancelled" // 绕过业务规则
    return s.repo.Save(ctx, &order)
}
```

### 值对象（Value Object）

- 放在 `domain/dp/`（domain primitives）目录
- 不可变，通过构造函数验证并返回 `(值, error)`
- 相等性由属性值决定，不依赖 ID

```go
// domain/dp/email.go
func NewEmail(v string) (string, error) {
    if !emailRegex.MatchString(v) {
        return "", errors.New("invalid email format")
    }
    return strings.ToLower(v), nil
}
```

**验证原则**：字段验证在值对象构造时完成（Controller 层的 `req.toCmd()` 中调用），不在 App 层或 Domain 层重复验证。

### 领域服务（Domain Service）

当业务操作涉及多个聚合根、但不归属于任一聚合根时，放入领域服务：

```go
// domain/service/transfer.go
// 转账涉及两个账户聚合根，不归属于任一聚合
type TransferService struct{}

func (s *TransferService) Transfer(from, to *Account, amount int64) error {
    if err := from.Debit(amount); err != nil {
        return err
    }
    return to.Credit(amount)
}
```

### Repository 接口

- Repository 接口定义在 `domain/repository/` 中，属于 Domain 层
- 接口设计面向领域语言（`Find`、`Save`、`Delete`），不暴露 SQL 语义
- 所有方法第一个参数必须是 `context.Context`

```go
// domain/repository/repo.go
type Order interface {
    Find(ctx context.Context, id int64) (domain.Order, error)
    Save(ctx context.Context, v *domain.Order) error
    Delete(ctx context.Context, v *domain.Order) error
    List(ctx context.Context, opt *ListOpt) ([]OrderInfo, int64, error)
}
```

## Infrastructure 层规范

### DO 模型

- 必须内嵌 `postgresql.CommonModel`（提供 ID、CreatedAt、UpdatedAt），**禁止**使用 `gorm.Model`（含软删除）
- 必须包含 `Version int` 字段用于乐观锁
- DO 文件与 Repository 实现文件分离：`order_do.go` + `order.go`

### DO ↔ Domain 转换

双向转换方法必须成对存在，保持私有，不跨包暴露：

```go
// order_do.go

// DO → Domain
func (do *orderDO) toOrder() domain.Order { ... }

// Domain → DO
func toOrderDO(v *domain.Order) orderDO { ... }
```

### 错误转换职责

Repository 实现层负责将 GORM 错误转换为领域通用错误（不含业务含义）：

```go
func (impl *orderImpl) Find(ctx context.Context, id int64) (domain.Order, error) {
    // ...
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return domain.Order{}, repository.NewErrorResourceNotFound(errors.New("order not found"))
    }
    return do.toOrder(), nil
}
```

## App 层规范

### 职责边界

- **编排用例流程**：调用 Repository 取数据 → 调用聚合根方法 → 调用 Repository 持久化
- **事务边界**：跨聚合的原子操作通过 `transaction.Do(ctx, f)` 包装
- **错误转换**：将 `repository.ErrorResourceNotFound` 等领域通用错误转换为带业务码的 `allerror` 错误

```go
// ✅ 正确：App 层负责错误码转换
func (s *orderApp) Cancel(ctx context.Context, id int64, user *User) error {
    order, err := s.repo.Find(ctx, id)
    if err != nil {
        if repository.IsErrorResourceNotFound(err) {
            return allerror.NewNotFoundError(ErrorCodeOrderNotFound, "", err)
        }
        return err
    }
    return order.Cancel(user) // 业务规则在聚合根中
}
```

### 命名规范

| 类型 | 命名 | 示例 |
|------|------|------|
| 输入命令对象 | `CmdToXxx` | `CmdToCreateOrder` |
| 单实体输出 | `XxxDTO` | `OrderDTO` |
| 操作特定输出 | `XxxResultDTO` | `CreateOrderResultDTO` |
| 列表输出 | `XxxsDTO` | `OrdersDTO` |
| 请求→命令转换 | `req.toCmd()` | `reqToCreateOrder.toCmd()` |
| 实体→DTO 转换 | `toXxxDTO(v)` | `toOrderDTO(v)` |

> `XxxResultDTO` 用于创建/更新操作的响应体与查询响应体结构不同时，相同时可直接复用 `XxxDTO`。

### 构造函数

App Service 依赖通过构造函数注入，参数类型为接口：

```go
func NewOrderAppService(
    repo repository.Order,
    payment PaymentService,
    transaction commonrepo.Transaction,
) *orderApp {
    return &orderApp{repo: repo, payment: payment, transaction: transaction}
}
```

## Controller 层规范

### 命名规范

- 请求体结构体使用 `req` 前缀（包私有）：`reqToCreateOrder`
- 转换方法 `req.toCmd()` 返回 `(Cmd, error)`，负责参数绑定后的值对象验证

### Handler 规范

- 必须调用 `ctx.Request.Context()` 传递给 App 层
- 参数错误：`SendBadRequestBody()` / `SendBadRequestParam()`
- 业务/系统错误：`SendError()`（内部自动映射 HTTP 状态码）
- 成功响应：`SendRespOfGet()` / `SendRespOfPost()` / `SendRespOfPut()` / `SendRespOfDelete()`

### 不得越层

- 不得在 Controller 直接调用 Repository
- 不得在 Controller 直接操作 Domain 对象（聚合根方法）

## 领域事件规范

### 事件定义

- 放在 `domain/event/` 目录
- 命名使用**过去式**，表示已发生的事实（`OrderCancelled`、`UserRegistered`）
- 必须包含事件元信息，便于消费者幂等处理：

```go
// domain/event/order.go
type OrderCancelled struct {
    EventID    string    // 全局唯一 ID，用于消费者幂等去重
    OrderID    int64
    OccurredAt time.Time
}
```

### 发布者接口

发布者接口定义在 Domain 层，实现在 Infrastructure 层：

```go
// domain/event/publisher.go
type Publisher interface {
    Publish(ctx context.Context, topic string, event interface{}) error
}
```

### 发布时机：事务内发布

**团队约定：在事务内发布领域事件。**

```go
// ✅ 正确：事务内发布，确保消息不丢失
func (s *orderApp) Cancel(ctx context.Context, id int64, user *User) error {
    return s.transaction.Do(ctx, func(txCtx context.Context) error {
        order, err := s.repo.Find(txCtx, id)
        if err != nil { ... }

        if err := order.Cancel(user); err != nil {
            return err
        }
        if err := s.repo.Save(txCtx, &order); err != nil {
            return err
        }

        // 在事务内发布：若发布失败则事务回滚，业务数据和消息保持一致
        return s.publisher.Publish(txCtx, topicOrderCancelled, &event.OrderCancelled{
            EventID:    uuid.NewString(),
            OrderID:    order.ID,
            OccurredAt: time.Now(),
        })
    })
}
```

**trade-off 说明：**
- 避免的风险：事务提交后发布失败导致消息丢失，业务中断且链路追踪困难
- 承担的风险：消息发布成功但事务随后回滚，消费者收到"幻象消息"

**因此，消费者必须实现幂等处理**，这是使用此策略的强制前提。

### 消费者幂等规范

**优先利用业务操作的天然幂等性**（推荐），避免引入额外的幂等表：

```go
// 通过业务语义保证幂等：UPDATE WHERE status='pending'，重复执行结果相同
func (h *NotifyHandler) OnOrderCancelled(ctx context.Context, e *event.OrderCancelled) error {
    return h.repo.UpdateStatusIfPending(ctx, e.OrderID, StatusCancelled)
}
```

**当业务操作无法天然幂等时**，用 `EventID` 做显式去重（需持久化存储）：

```go
func (h *AuditHandler) OnOrderCancelled(ctx context.Context, e *event.OrderCancelled) error {
    if h.eventLogRepo.Exists(ctx, e.EventID) {
        return nil
    }
    if err := h.auditRepo.AppendLog(ctx, e); err != nil {
        return err
    }
    return h.eventLogRepo.MarkDone(ctx, e.EventID)
}
```

**多消费者隔离原则**：不同服务消费同一事件时，各自独立维护幂等状态，不共享幂等表。同一 `EventID` 在每个消费者服务中各处理一次，这是正确的行为。

## 服务初始化规范

- 手动依赖注入，不得引入 Wire 等代码生成框架
- 初始化顺序按依赖拓扑排序：基础设施（DB、缓存）→ 通用服务（日志、消息）→ 业务模块
- DAO 通过工厂函数 `postgresql.DAO("table_name")` 创建
- 各业务模块初始化封装为独立函数（`initOrder()`），在 `server/` 目录组装

## 禁止行为

- Domain 层不得导入 `infrastructure`、`app`、`controller` 包
- App 层不得导入 `infrastructure` 具体实现包（只依赖 Domain 层的接口定义）
- 不得在 Controller 或 App 层直接处理 `gorm.ErrRecordNotFound` 等 GORM 错误
- 不得在多处重复实现同一字段的验证逻辑（验证属于值对象构造函数）
- 不得将聚合根内部字段直接暴露为 DTO 的 JSON 字段（通过显式转换方法控制输出）
