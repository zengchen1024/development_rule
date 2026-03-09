# GORM 操作规范

---
paths:
  - "**/infrastructure/repositoryimpl/**/*.go"
  - "**/dao/**/*.go"
---

## Context 传递规范（关键）

### 接口层签名必须包含 ctx

`domain/repository` 接口中的所有方法**必须**将 `context.Context` 作为第一个参数，不得只在实现层传递而接口层省略：

```go
// ❌ 错误：接口层缺少 ctx，导致调用链路在接口边界断裂
type UserRepo interface {
    Find(id int64) (*domain.User, error)
    List(opt *ListOpt) ([]domain.User, int64, error)
}

// ✅ 正确：接口层和实现层统一包含 ctx
type UserRepo interface {
    Find(ctx context.Context, id int64) (*domain.User, error)
    List(ctx context.Context, opt *ListOpt) ([]domain.User, int64, error)
    Save(ctx context.Context, v *domain.User) error
}
```

**接口层缺少 ctx 的后果**：
- 调用方（App 层）无法将 request context 传入，trace_id 和超时在接口边界丢失
- 事务对象无法通过 context 传递，导致事务内操作在事务外执行

### 实现层：两步操作缺一不可

Repository 实现中必须先调用 `dao.New(ctx)` 获取事务感知实例，再调用 `WithContext(ctx)`：

```go
// ❌ 错误：只调用 WithContext，跳过 New(ctx)——事务对象无法注入
func (impl *userImpl) Add(ctx context.Context, v *domain.User) error {
    return impl.dao.WithContext(ctx).Create(&do).Error
}

// ❌ 错误：New(ctx) 后用 context.Background()——链路追踪信息丢失
func (impl *userImpl) Add(ctx context.Context, v *domain.User) error {
    dao := impl.dao.New(ctx)
    return dao.WithContext(context.Background()).Create(&do).Error
}

// ✅ 正确：两步缺一不可
func (impl *userImpl) Add(ctx context.Context, v *domain.User) error {
    dao := impl.dao.New(ctx)               // 步骤1：从 ctx 提取事务对象（如有）
    return dao.WithContext(ctx).Create(&do).Error  // 步骤2：传递 ctx 给 GORM
}
```

**原因**：
- `dao.New(ctx)`：若 ctx 中携带事务对象则返回事务 DB，否则返回普通 DB
- `WithContext(ctx)`：将 ctx 注入 GORM，用于超时控制和链路追踪

## 事务使用规范

### 统一事务包装

使用 `transaction.Do()` 包装事务逻辑，不得手动调用 `db.Begin()` / `Commit()` / `Rollback()`：

```go
// ✅ 正确：使用统一的事务包装
func (s *UserService) CreateUser(ctx context.Context, user *domain.User) error {
    return s.transaction.Do(ctx, func(txCtx context.Context) error {
        // 事务内的所有操作都使用 txCtx（包含事务对象和原始 context 信息）
        if err := s.repo.Add(txCtx, user); err != nil {
            return err
        }
        return s.logRepo.Add(txCtx, &domain.Log{...})
    })
}

// ❌ 禁止：手动管理事务
func (s *UserService) CreateUser(user *domain.User) error {
    tx := db.Begin()
    defer func() {
        if r := recover(); r != nil {
            tx.Rollback()
        }
    }()
    // ...
    tx.Commit()
}
```

### 事务实现参考

```go
// common/infrastructure/postgresql/transaction.go
type contextTxKey struct{}

// ✅ 正确：接收原始 context，保留 trace_id 等信息
func (impl transactionImpl) Do(ctx context.Context, f func(ctx context.Context) error) error {
    return db.Transaction(func(tx *gorm.DB) error {
        // 在原始 context 基础上添加事务对象，不丢失链路信息
        txCtx := context.WithValue(ctx, contextTxKey{}, tx)
        return f(txCtx)
    })
}

// DAO 层从 context 取出事务对象
func (dao *daoImpl) New(ctx context.Context) *gorm.DB {
    if tx, ok := ctx.Value(contextTxKey{}).(*gorm.DB); ok {
        return tx
    }
    return dao.DB
}
```

## 错误处理规范

### 分层错误转换链路

**DAO 层 → Repository 层 → App 层**，每层负责不同的错误转换：

```go
// DAO 层：直接返回 GORM 原始错误
func (dao *daoImpl) GetRecord(ctx context.Context, filter, result interface{}) error {
    return dao.WithContext(ctx).Where(filter).First(result).Error
}

// Repository 层：转换为 common/domain/repository 定义的错误
func (impl *userImpl) Find(ctx context.Context, id int64) (*domain.User, error) {
    var do userDO
    do.ID = id
    err := impl.dao.GetByPrimaryKey(ctx, &do)

    if errors.Is(err, gorm.ErrRecordNotFound) {
        return nil, repository.NewErrorResourceNotFound(errors.New("user not found"))
    }
    if err != nil {
        return nil, err // 其他数据库错误直接返回
    }
    return do.toDomain(), nil
}

// App 层：转换为 allerror 定义的业务错误
func (s *UserService) GetUser(ctx context.Context, id int64) (*domain.User, error) {
    user, err := s.repo.Find(ctx, id)
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
```

**分层职责**：
- **DAO 层**：只负责数据库操作，返回 GORM 原始错误
- **Repository 层**：转换为领域层通用错误（`ErrorResourceNotFound`、`ErrorDuplicateCreating`、`ErrorConcurrentUpdating`）
- **App 层**：转换为业务错误（`allerror`），添加业务错误码和用户友好的错误信息

## 并发控制规范

### 使用乐观锁（Version 字段）

```go
// DO 模型必须包含 Version 字段
type UserDO struct {
    postgresql.CommonModel
    Version int64  `gorm:"column:version"`
    Name    string `gorm:"column:name"`
}

// 更新时检查 Version 并自增
func (impl *userImpl) Save(ctx context.Context, v *domain.User) error {
    do := toUserDO(v)
    do.Version = v.Version + 1

    dao := impl.dao.New(ctx)
    r := dao.WithContext(ctx).Model(&userDO{}).Where(
        dao.EqualQuery("version"), v.Version,
    ).Updates(&do)

    if r.RowsAffected == 0 {
        return repository.NewErrorConcurrentUpdating(
            errors.New("concurrent updating"),
        )
    }
    return nil
}
```

- 不得使用悲观锁（`FOR UPDATE`），除非有明确的性能瓶颈证明
- Version 字段必须在 WHERE 条件中，通过 `RowsAffected == 0` 判断冲突

## 分页查询规范

```go
func (impl *userImpl) List(ctx context.Context, opt *repository.ListOpt) ([]domain.User, int64, error) {
    // 参数校验
    if opt.PageNum <= 0 {
        opt.PageNum = 1
    }
    if opt.CountPerPage <= 0 || opt.CountPerPage > 100 {
        opt.CountPerPage = 20
    }

    query := impl.dao.WithContext(ctx)

    // 先查总数
    var total int64
    if err := query.Model(&userDO{}).Count(&total).Error; err != nil {
        return nil, 0, err
    }

    // 再查分页数据
    var dos []userDO
    offset := (opt.PageNum - 1) * opt.CountPerPage
    err := query.Offset(offset).Limit(opt.CountPerPage).Find(&dos).Error

    users := make([]domain.User, len(dos))
    for i := range dos {
        users[i] = dos[i].toDomain()
    }
    return users, total, err
}
```

- 必须校验 `PageNum` 和 `CountPerPage`，防止负数或过大值
- 建议设置 `CountPerPage` 上限（如 100）
- `Count()` 和 `Find()` 分开执行，避免 GORM 内部优化失效

## 连接池配置规范

```go
// 初始化时配置连接池
sqlDb, err := dbInstance.DB()
if err != nil {
    return nil, err
}

sqlDb.SetMaxOpenConns(500)           // 最大连接数
sqlDb.SetMaxIdleConns(250)           // 最大空闲连接数
sqlDb.SetConnMaxLifetime(2 * time.Minute) // 连接最大生命周期
```

**推荐值**（根据实际负载调整）：
- `MaxOpenConns`：500（高并发场景）
- `MaxIdleConns`：MaxOpenConns 的 50%
- `ConnMaxLifetime`：2-5 分钟（避免数据库主动断开连接）

## 禁止行为

- 不得在循环内调用 `db.Find()` / `db.Where()`（N+1 问题）
- 不得使用 `db.Raw()` / `db.Exec()` 执行原始 SQL，除非 GORM API 无法满足需求
- 不得忽略 GORM 错误返回值（`_ = db.Create(...)`）
- 不得在 Repository 层使用 `context.Background()`，必须使用传入的 `ctx`
  - **例外**：服务启动初始化阶段（如表初始化、数据预热）无 request context 时可用 `context.Background()`
- 不得在 DO 模型中使用 `gorm.Model`（包含软删除字段），统一使用 `postgresql.CommonModel`
