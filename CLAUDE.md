# 团队 Claude Code 规范

本文件为团队共享规范入口，适用于所有 Go 后端服务项目。

## 技术栈约定

- **语言**：Go（版本见各项目 go.mod）
- **Web 框架**：Gin
- **日志**：logrus
- **ORM**：GORM v2
- **错误处理**：`github.com/opensourceways/go-ddd-framework/allerror`
- **配置**：手动解析 YAML/JSON 配置文件
- **部署**：Kubernetes
- **框架库**：`github.com/opensourceways/go-ddd-framework`（Controller 响应、中间件、PostgreSQL DAO、Repository 错误、分页等）

## 核心原则

1. 不得引入上述清单之外的同类库，如有必要须先与 Tech Lead 确认
2. 所有对外接口必须使用统一响应格式，必须包含 trace_id
3. 日志必须结构化，且包含 trace_id 字段
4. 错误必须在最终返回前完成包装和记录

## 分类规则

详细规则见 `.claude/rules/` 目录：

- [框架库 API 参考](`.claude/rules/framework-api.md`)（**写代码必读**：函数签名、类型定义、初始化方式）
- [DDD 分层架构](`.claude/rules/ddd-layers.md`)
- [技术栈约束](`.claude/rules/tech-stack.md`)
- [API 设计](`.claude/rules/api-design.md`)
- [日志规范](`.claude/rules/logging.md`)
- [错误处理](`.claude/rules/error-handling.md`)
- [GORM 操作](`.claude/rules/gorm.md`)
- [测试规范](`.claude/rules/testing.md`)
- [K8s 部署](`.claude/rules/k8s.md`)
