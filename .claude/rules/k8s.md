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
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /readyz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

- 每个 Deployment 必须设置 `resources.requests` 和 `resources.limits`
- 必须配置 `livenessProbe` 和 `readinessProbe`
- 必须设置 `terminationGracePeriodSeconds`（建议 30s）

### 配置和密钥管理

- 配置项通过 `ConfigMap` 挂载，不得写死在 YAML 中
- 密钥通过 `Secret` 注入，不得明文出现在任何 YAML 文件中
- 不得将 `Secret` 提交到 git 仓库

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
