# 技术栈约束

## 禁止引入的替代库

以下为**禁止**在未经 Tech Lead 审批的情况下引入的库：

| 类别 | 禁止 | 原因 |
|------|------|------|
| Web 框架 | Echo, Fiber, chi | 统一使用 Gin |
| 日志 | zap, slog, zerolog | 统一使用 logrus |
| ORM | ent, sqlx, sqlc, 裸 database/sql | 统一使用 GORM v2 |
| 配置 | viper, envconfig | 统一使用手动解析 |

## 依赖管理

- 新增依赖前，先确认是否有已有库可以满足需求
- 不得 fork 上游库后在内部私用（应提 PR 回上游或使用 replace 指令并说明原因）
- go.sum 必须提交到仓库

## 版本要求

- Go 最低版本：见项目 go.mod 的 `go` 指令
- GORM：使用 v2（`gorm.io/gorm`），不得使用 v1（`github.com/jinzhu/gorm`）
