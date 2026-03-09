# 测试规范

---
paths:
  - "**/*_test.go"
---

## 基本要求

- 使用 `github.com/stretchr/testify`（`assert` + `require` 包）
- 禁止使用原生 `t.Error` / `t.Fatal` 进行断言
- 测试文件与被测文件同目录，命名为 `xxx_test.go`

## 单元测试

- 关键业务逻辑（service 层、工具函数）必须有单元测试
- 使用 Table-Driven Test 模式编写多场景覆盖：

```go
func TestGetUser(t *testing.T) {
    tests := []struct {
        name    string
        input   int64
        want    *User
        wantErr bool
    }{
        {"正常用户", 1, &User{ID: 1}, false},
        {"用户不存在", 999, nil, true},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := GetUser(tt.input)
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

## Mock 规范

### 使用 testify/mock（手写 Mock 结构体）

统一使用 `github.com/stretchr/testify/mock`，**不引入 mockgen / gomock**。

Mock 文件命名为 `mock_<interface>_test.go`，放在与测试文件相同目录：

```go
// mock_order_repo_test.go
package app_test

import (
    "context"
    "github.com/stretchr/testify/mock"
)

type mockOrderRepo struct {
    mock.Mock
}

func (m *mockOrderRepo) Find(ctx context.Context, id int64) (domain.Order, error) {
    args := m.Called(ctx, id)
    return args.Get(0).(domain.Order), args.Error(1)
}

func (m *mockOrderRepo) Save(ctx context.Context, v *domain.Order) error {
    args := m.Called(ctx, v)
    return args.Error(0)
}
```

### 使用示例

```go
func TestCancelOrder(t *testing.T) {
    repo := new(mockOrderRepo)
    svc := app.NewOrderAppService(repo, ...)

    order := domain.Order{Id: 1, Status: dp.StatusPending}
    repo.On("Find", mock.Anything, int64(1)).Return(order, nil)
    repo.On("Save", mock.Anything, mock.AnythingOfType("*domain.Order")).Return(nil)

    err := svc.Cancel(context.Background(), 1, &testUser)
    require.NoError(t, err)
    repo.AssertExpectations(t)
}
```

### 测试覆盖率要求

- Domain 层状态机/业务规则：**必须有单元测试**
- App 层核心用例（Create、Cancel 等）：**必须有单元测试**
- Controller 层、Repository 实现层：不要求单元测试（集成测试覆盖）



## 集成测试

- 放在 `test/integration/` 或 `_test` 包中，通过 build tag 区分：

```go
//go:build integration
```

- 运行命令：`go test -tags=integration ./...`
- 集成测试可以连接真实数据库，但必须使用独立的测试数据库
- 测试完成后必须清理测试数据

## 禁止行为

- 不得在测试中使用 `time.Sleep` 等待异步结果（使用 channel 或 polling with timeout）
- 不得提交依赖特定本地环境（路径、端口）的测试
- 不得跳过测试（`t.Skip()`）而不添加说明注释
