# 测试规范

---
paths:
  - "**/*.go"
---

## 基本要求

- 使用 `github.com/stretchr/testify`（`assert` + `require` 包）
- 禁止使用原生 `t.Error` / `t.Fatal` 进行断言
- 测试文件与被测文件同目录，命名为 `xxx_test.go`
- **测试包名与源码包名一致**（白盒测试）：Domain 层用 `package domain`，App 层用 `package app`

## 覆盖率要求

| 层 | 要求 | 说明 |
|----|------|------|
| Domain | **所有公开方法必须有测试，覆盖率 ≥ 80%** | 聚合根方法、值对象构造函数全覆盖，包含所有合法/非法输入场景 |
| App | **所有 AppService 方法必须有测试，覆盖率 ≥ 80%** | 覆盖正常路径 + 关键错误路径（资源不存在、并发冲突等） |
| Controller | 不要求单元测试 | 通过集成测试覆盖 |
| Infrastructure | 不要求单元测试 | 通过集成测试覆盖 |

运行覆盖率检查：

```bash
# 查看覆盖率报告
go test -coverprofile=coverage.out ./errata/domain/... ./errata/app/...
go tool cover -func=coverage.out

# CI 中强制 80% 阈值（示例脚本）
COVERAGE=$(go test -coverprofile=coverage.out ./errata/domain/... ./errata/app/... 2>&1 | grep "coverage:" | awk '{print $2}' | tr -d '%')
if (( $(echo "$COVERAGE < 80" | bc -l) )); then
  echo "Coverage $COVERAGE% is below 80%"
  exit 1
fi
```

## Domain 层单元测试

Domain 层是**纯业务逻辑，无 I/O 依赖，不需要任何 Mock**。

### 聚合根方法测试

覆盖所有状态转移和前置条件校验：

```go
// domain/order_test.go
package domain

func TestOrder_Cancel(t *testing.T) {
    tests := []struct {
        name        string
        order       Order
        user        User
        wantErr     bool
        wantErrCode string  // 验证具体错误码，不只是 wantErr bool
        wantStatus  string
    }{
        {
            name:       "pending 订单可取消",
            order:      Order{Id: 1, Status: dp.StatusPending, CreatedBy: 10},
            user:       User{Id: 10},
            wantStatus: dp.StatusCancelled,
        },
        {
            name:        "非 pending 状态不可取消",
            order:       Order{Id: 1, Status: dp.StatusAccepted, CreatedBy: 10},
            user:        User{Id: 10},
            wantErr:     true,
            wantErrCode: "order_not_cancellable",
        },
        {
            name:        "非本人不可取消",
            order:       Order{Id: 1, Status: dp.StatusPending, CreatedBy: 10},
            user:        User{Id: 99},
            wantErr:     true,
            wantErrCode: "no_permission",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := tt.order.Cancel(&tt.user)
            if tt.wantErr {
                require.Error(t, err)
                // 验证错误码（不只验证有无错误）
                var allerr interface{ ErrorCode() string }
                require.ErrorAs(t, err, &allerr)
                assert.Equal(t, tt.wantErrCode, allerr.ErrorCode())
                return
            }
            require.NoError(t, err)
            assert.Equal(t, tt.wantStatus, tt.order.Status)
        })
    }
}
```

### 值对象构造函数测试

覆盖所有合法值和非法边界值：

```go
// domain/dp/status_test.go
package dp

func TestNewStatus(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    string
        wantErr bool
    }{
        {"合法状态 pending",  "pending",  "pending",  false},
        {"合法状态 accepted", "accepted", "accepted", false},
        {"空字符串",          "",         "",         true},
        {"非法值",            "unknown",  "",         true},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := NewStatus(tt.input)
            if tt.wantErr {
                require.Error(t, err)
                return
            }
            require.NoError(t, err)
            assert.Equal(t, tt.want, got)
        })
    }
}
```

## App 层单元测试

App 层需要 Mock 所有外部依赖（Repository、Transaction、外部服务），使用 mockery 自动生成。

### Mockery 配置

在项目根目录创建 `.mockery.yaml`：

```yaml
# .mockery.yaml
with-expecter: true   # 生成类型安全的 Expecter 方法（推荐，避免字符串方法名）
packages:
  # 为 App 层测试生成 Repository Mock（输出到 app/ 目录）
  github.com/opensourceways/your-project/errata/domain/repository:
    config:
      dir: "errata/app"
      outpkg: "app"
      filename: "mock_{{.InterfaceName | snakecase}}_test.go"
    interfaces:
      Errata:
      ErrataReview:

  # 为 App 层测试生成 Transaction Mock
  github.com/opensourceways/go-ddd-framework/repository:
    config:
      dir: "errata/app"
      outpkg: "app"
      filename: "mock_{{.InterfaceName | snakecase}}_test.go"
    interfaces:
      Transaction:
```

在 `app/app.go` 中添加 go:generate 指令：

```go
// app/app.go
//go:generate mockery --config=../../.mockery.yaml
package app
```

生成 Mock：

```bash
go generate ./...
# 或直接运行
mockery --config=.mockery.yaml
```

### Mock 使用（Expecter 风格，类型安全）

```go
// app/app_test.go
package app

func TestOrderApp_Cancel(t *testing.T) {
    tests := []struct {
        name        string
        setupMock   func(repo *MockErrata)
        userId      int64
        wantErr     bool
        wantErrCode string
    }{
        {
            name: "正常取消",
            setupMock: func(repo *MockErrata) {
                order := domain.Order{Id: 1, Status: dp.StatusPending, CreatedBy: 10}
                // Expecter 风格：类型安全，编译期检查参数类型
                repo.EXPECT().Find(mock.Anything, int64(1)).Return(order, nil)
                repo.EXPECT().Save(mock.Anything, mock.AnythingOfType("*domain.Order")).Return(nil)
            },
            userId:  10,
            wantErr: false,
        },
        {
            name: "订单不存在",
            setupMock: func(repo *MockErrata) {
                repo.EXPECT().Find(mock.Anything, int64(1)).
                    Return(domain.Order{}, repository.NewErrorResourceNotFound(errors.New("not found")))
            },
            userId:      10,
            wantErr:     true,
            wantErrCode: ErrorCodeOrderNotFound,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            repo := NewMockErrata(t)  // t 传入后，AssertExpectations 自动调用
            tt.setupMock(repo)

            tx := NewMockTransaction(t)
            svc := NewOrderAppService(repo, tx)
            user := domain.User{Id: tt.userId}

            err := svc.Cancel(context.Background(), 1, &user)

            if tt.wantErr {
                require.Error(t, err)
                var allerr interface{ ErrorCode() string }
                require.ErrorAs(t, err, &allerr)
                assert.Equal(t, tt.wantErrCode, allerr.ErrorCode())
                return
            }
            require.NoError(t, err)
        })
    }
}
```

> `NewMockXxx(t)` 是 mockery with-expecter 模式生成的构造函数，会自动在测试结束时调用 `AssertExpectations(t)`，无需手动调用。

### 测试辅助函数

重复构建的测试数据抽取为辅助函数，命名用 `newTest<Type>()` 前缀：

```go
// app/testhelper_test.go
package app

func newTestOrder(opts ...func(*domain.Order)) domain.Order {
    o := domain.Order{
        Id:        1,
        Status:    dp.StatusPending,
        CreatedBy: 10,
        Version:   0,
    }
    for _, opt := range opts {
        opt(&o)
    }
    return o
}

func newTestUser(id int64) domain.User {
    return domain.User{Id: id}
}
```

## 集成测试

- 放在 `test/integration/` 目录，通过 build tag 区分：

```go
//go:build integration
```

- 运行命令：`go test -tags=integration ./...`
- 集成测试可以连接真实数据库，但必须使用独立的测试数据库
- 测试完成后必须清理测试数据（使用 `t.Cleanup()` 注册清理函数）

```go
//go:build integration

package repositoryimpl_test

func TestOrderRepo_Save(t *testing.T) {
    repo := setupTestRepo(t)

    order := domain.NewOrder(...)
    err := repo.Save(context.Background(), &order)
    require.NoError(t, err)
    assert.Greater(t, order.Id, int64(0))  // 验证 ID 已回写

    t.Cleanup(func() {
        // 清理测试数据
        _ = repo.Delete(context.Background(), &order)
    })
}
```

## 禁止行为

- 不得在测试中使用 `time.Sleep` 等待异步结果（使用 channel 或 polling with timeout）
- 不得提交依赖特定本地环境（路径、端口）的测试
- 不得跳过测试（`t.Skip()`）而不添加说明注释
- 不得只断言 `wantErr bool`，错误路径必须同时验证错误码（`allerr.ErrorCode()`）
- 不得使用 `mock.On("MethodName", ...)` 字符串风格（使用 `EXPECT()` 类型安全风格）
