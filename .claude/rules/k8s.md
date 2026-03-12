# K8s 部署规范

---
paths:
  - "**/*.yaml"
  - "**/*.yml"
  - "**/Dockerfile"
  - "**/helm/**"
---

## Dockerfile 规范

### 多阶段构建模板

```dockerfile
# ── 阶段一：编译 ──────────────────────────────────────────────────
# 项目中必须指定具体版本号，禁止省略 tag
FROM openeuler/openeuler:<version> AS BUILDER

RUN dnf update -y && \
    dnf install -y wget tar gcc && \
    wget https://mirrors.aliyun.com/golang/go1.24.1.linux-amd64.tar.gz && \
    tar -C /usr/local -xzf go1.24.1.linux-amd64.tar.gz && \
    export PATH=$PATH:/usr/local/go/bin && \
    go env -w GOPROXY=https://goproxy.cn,direct

COPY . /go/src/github.com/opensourceways/<service>

# 必须启用 PIE + RELRO + 去掉符号表，增强二进制安全性
RUN cd /go/src/github.com/opensourceways/<service> && \
    GO111MODULE=on /usr/local/go/bin/go build \
        -o <binary_name> \
        -buildmode=pie \
        --ldflags "-s -linkmode 'external' -extldflags '-Wl,-z,now'"

# ── 阶段二：运行时镜像 ────────────────────────────────────────────
# 项目中必须指定具体版本号，禁止省略 tag
FROM openeuler/openeuler:<version>

# 创建固定 UID/GID 的非 root 用户（禁止登录 shell）
RUN groupadd -g 1000 app && \
    useradd -u 1000 -g app -s /sbin/nologin -m app

# 清除系统 banner，避免泄露版本信息
RUN echo > /etc/issue && echo > /etc/issue.net && echo > /etc/motd

# 设置家目录权限（仅 owner 可访问）
RUN mkdir -p /home/app && \
    chmod 700 /home/app && \
    chown app:app /home/app

# 禁止 root 历史记录；设置密码最长有效期
RUN echo 'set +o history' >> /root/.bashrc && \
    sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS   90/' /etc/login.defs

# 删除调试工具和无用文件，减小攻击面
RUN rm -rf /tmp/* \
    /usr/share/gdb \
    /usr/share/licenses/glibc

USER app
WORKDIR /home/app

# 拷贝时指定 chown，避免文件属主为 root
COPY --chown=app:app --from=BUILDER \
    /go/src/github.com/opensourceways/<service>/<binary_name> \
    /home/app/<binary_name>

# 可执行文件：owner 可读可执行，group 可读可执行，other 无权限
RUN chmod 550 /home/app/<binary_name>

# shell 配置：禁用历史记录，设置 umask 027（新建文件默认 640/750）
RUN chmod 640 /home/app/.bash* && \
    echo "umask 027"      >> /home/app/.bashrc && \
    echo 'set +o history' >> /home/app/.bashrc

ENTRYPOINT ["/home/app/<binary_name>"]
```

### 强制要求

| 要求 | 说明 |
|------|------|
| 基础镜像 | 统一使用 `openeuler/openeuler`；项目 Dockerfile 中必须指定具体版本 tag，禁止省略或使用 `latest` |
| 多阶段构建 | 最终镜像不得包含编译工具链（go、gcc、wget 等） |
| 非 root 用户 | 必须创建固定 UID/GID（推荐 1000）的专用用户，`-s /sbin/nologin` 禁止登录 |
| 可执行文件权限 | `chmod 550`（owner/group 可读可执行，other 无权限） |
| 其他文件权限 | `.bash*` 等配置文件 `chmod 640`；工作目录 `chmod 700` |
| COPY 指定属主 | `COPY --chown=app:app`，禁止 COPY 后文件属主为 root |
| 删除调试工具 | 必须删除 `/usr/share/gdb` 等调试组件及 `/tmp/*` |
| 清除系统 banner | 清空 `/etc/issue`、`/etc/issue.net`、`/etc/motd` |
| Shell 安全配置 | `.bashrc` 中写入 `set +o history`（禁止历史记录）和 `umask 027` |
| 编译安全选项 | 必须使用 `-buildmode=pie` + `"-s -linkmode 'external' -extldflags '-Wl,-z,now'"` |
| 禁止打包密钥 | 不得将配置文件、证书、密钥打包进镜像，通过 K8s Secret/ConfigMap 挂载 |

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
