# Pull Request 规范

## PR 标题

沿用 Conventional Commits 格式，与 commit message 保持一致：

```
<type>(<scope>): <简短描述>
```

- `type`：`feat` / `fix` / `refactor` / `docs` / `test` / `chore`
- `scope`：受影响的模块或目录（如 `order`、`auth`、`k8s`）
- 简短描述：祈使句，英文，不加句号，不超过 72 字符

**示例：**
```
feat(order): add cancel endpoint
fix(auth): handle expired token edge case
refactor(user): extract status validation to value object
docs(rules): add pull request convention
```

## PR 描述模板

```markdown
## 背景

<!-- 为什么要做这个改动？解决了什么问题或满足了什么需求。 -->

## 改动内容

<!-- 具体做了什么，使用 bullet list。 -->
-
-

## 注意事项

<!-- 可选。给 reviewer 的特别说明，如：破坏性改动、依赖的配置变更、需要关注的实现细节。无则删除本节。 -->
```

### 填写要求

**背景**：
- 说明改动的上下文和动机，而非重复标题
- 若关联 issue 或需求，附上链接

**改动内容**：
- 每条描述一个独立变更点
- 关注"做了什么"，不需要解释"怎么实现的"（代码本身说明实现）

**注意事项**：
- 有破坏性改动（API 不兼容、配置字段变更）时必须写明
- 无特殊说明时删除本节，不保留空标题

### 示例

```markdown
## 背景

订单模块缺少取消接口，用户创建订单后无法撤回，影响业务流程。

## 改动内容

- 新增 `DELETE /v1/orders/:id` 取消订单接口
- 聚合根 `Order` 增加 `Cancel()` 方法，校验状态前置条件
- App 层处理并发冲突错误，返回 `order_update_concurrently` 错误码

## 注意事项

依赖 `orders` 表新增 `version` 字段，部署前需确认数据库迁移已执行。
```
