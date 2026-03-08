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

- 外部依赖（数据库、HTTP 服务）必须 Mock，不得在单元测试中连接真实服务

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
