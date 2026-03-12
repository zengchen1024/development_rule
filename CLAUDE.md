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

## Claude Code 工作流约束

**每次生成或修改代码后，必须主动对照以下清单做自检，发现问题直接修复，不等用户指出：**

1. **响应格式**：是否使用项目标准响应结构体（含 `code`/`msg`/`data`/`trace_id`），禁止使用 `gin.H{}` 等裸 map
2. **错误处理**：业务错误必须用 `allerror.New`，禁止将 `fmt.Errorf` / `errors.New` 直接作为函数返回值
3. **响应函数**：Handler 层不得直接调用 `ctx.JSON()`，必须使用框架或项目封装的响应函数
4. **DDD 分层**：Controller 不调用 Repository，App 层不导入 Infrastructure 具体实现包
5. **死代码**：删除逻辑上不可达的分支（如条件已由上游调用保证的冗余错误检查）
6. **日志规范**：使用 `logrus.WithFields`，包含 `trace_id`，不使用字符串拼接
7. **魔法字符串**：禁止将字符串字面量硬编码在逻辑代码中，必须声明为具名常量后再引用；同一字符串在多处使用时尤其不得重复拼写

自检时区分「新增代码的违规」和「沿用既有代码的历史模式」，优先修复前者；历史模式的违规须指出但不强制修改。



1. 不得引入上述清单之外的同类库，如有必要须先与 Tech Lead 确认
2. 所有对外接口必须使用统一响应格式，必须包含 trace_id
3. 日志必须结构化，且包含 trace_id 字段
4. 错误必须在最终返回前完成包装和记录

## 分类规则

详细规则见 `.claude/rules/` 目录：

- [框架库 API 参考](`.claude/rules/framework-api.md`)（**写代码必读**：函数签名、类型定义、初始化方式）
- [项目目录结构与命名](`.claude/rules/project-layout.md`)（**写代码必读**：目录树、包名、文件名、类型命名）
- [DDD 分层架构](`.claude/rules/ddd-layers.md`)
- [技术栈约束](`.claude/rules/tech-stack.md`)
- [API 设计](`.claude/rules/api-design.md`)
- [日志规范](`.claude/rules/logging.md`)
- [错误处理](`.claude/rules/error-handling.md`)
- [GORM 操作](`.claude/rules/gorm.md`)
- [测试规范](`.claude/rules/testing.md`)
- [K8s 部署](`.claude/rules/k8s.md`)
- [安全开发规范](`.claude/rules/security.md`)
- [Pull Request 规范](`.claude/rules/pull-request.md`)
