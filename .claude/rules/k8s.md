# K8s 部署规范

---
paths:
  - "**/*.yaml"
  - "**/*.yml"
  - "**/Dockerfile"
  - "**/helm/**"
---

## Dockerfile 规范

```dockerfile
# 必须使用多阶段构建
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server ./cmd/server

FROM alpine:3.19
# 不得使用 root 用户运行
RUN addgroup -S app && adduser -S app -G app
USER app
WORKDIR /app
COPY --from=builder /app/server .
ENTRYPOINT ["./server"]
```

- 基础镜像必须固定版本 tag，禁止使用 `latest`
- 最终镜像不得包含编译工具链（必须多阶段构建）
- 不得将配置文件或密钥打包进镜像

## K8s 资源规范

### 必须设置的字段

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

# 健康检查
livenessProbe:
  httpGet:
    path: /v1/heartbeat
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /v1/heartbeat
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

- 每个 Deployment 必须设置 `resources.requests` 和 `resources.limits`
- 必须配置 `livenessProbe` 和 `readinessProbe`
- 必须设置 `terminationGracePeriodSeconds`（建议 180s，**必须大于代码端 `GracePeriod` 参数**）

> **优雅关闭说明**：K8s 发送 SIGTERM 后等待 `terminationGracePeriodSeconds` 秒，超时强制 SIGKILL。代码端通过 `interrupts.WaitForGracefulShutdown()` 在 `GracePeriod` 内等待在途请求完成。两者关系：`terminationGracePeriodSeconds` > `GracePeriod` + 预留 buffer（建议 30s）。

### 配置和密钥管理

- 配置项通过 `ConfigMap` 挂载，不得写死在 YAML 中
- 密钥通过 `Secret` 注入，不得明文出现在任何 YAML 文件中
- 不得将 `Secret` 提交到 git 仓库

### 敏感文件启动后清理

K8s Secret 以文件方式挂载后，服务读取完毕必须立即删除，防止敏感内容驻留磁盘。通过 `--rm-config` 参数启用，分两个阶段执行：

| 阶段 | 文件 | 时机 | 实现位置 |
|------|------|------|---------|
| 1 | YAML 配置文件 | 解析完成后立即删除（`defer os.Remove`） | `config.LoadConfig` |
| 2 | TLS 证书和密钥 | 服务启动后等待 `waitServerStart`（3s）再删 | `ServerOptions.clean()` |

**约束：**
- `clean()` 必须在 `interrupts.ListenAndServeTLS()` 之后调用，不得省略
- 删除失败时必须 `logrus.Fatal`，不得静默忽略（`_ = os.Remove(...)` 是错误写法）
- 等待时间 `waitServerStart` 不得缩短，确保 TLS 握手完成后再删证书

### 标签规范

所有资源必须包含以下标签：

```yaml
labels:
  app: <service-name>
  version: <image-tag>
  team: <team-name>
```

## 禁止行为

- 不得在生产 namespace 直接 `kubectl apply`，必须走 CI/CD 流程
- 不得使用 `hostNetwork: true` 或特权容器（`privileged: true`）
- 不得将服务端口直接暴露为 `NodePort`（使用 Ingress）
