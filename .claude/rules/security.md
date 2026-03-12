# Go 安全开发规范

---
paths:
  - "**/*.go"
  - "**/*.yaml"
  - "**/*.yml"
  - "**/Dockerfile"
---

> 本文件覆盖现有规范**未涉及**的安全要点，避免重复。
> 已覆盖的安全项（安全响应头、TLS 配置、Secret 挂载、编译安全选项、限流、日志脱敏）见各自规范文件。

---

## 静态分析与依赖扫描

**以下两个工具必须集成到 CI 流水线**，本地开发前建议先运行：

### gosec（代码安全扫描）

检测命令注入、路径穿越、弱加密、不安全的随机数等：

```bash
# 安装
go install github.com/securego/gosec/v2/cmd/gosec@latest

# 扫描整个项目（排除测试文件）
gosec -exclude-dir=vendor ./...

# CI 中输出 SARIF 格式（供 GitHub Code Scanning 使用）
gosec -fmt sarif -out gosec.sarif ./...
```

### govulncheck（依赖漏洞扫描）

检查依赖包中已知 CVE（基于 Go 官方漏洞数据库）：

```bash
# 安装
go install golang.org/x/vuln/cmd/govulncheck@latest

# 扫描
govulncheck ./...
```

**CI 示例（GitHub Actions）：**

```yaml
- name: Security scan
  run: |
    gosec ./...
    govulncheck ./...
```

- `gosec` 报告的 G101（硬编码凭据）、G201/G202（SQL 注入）、G304（路径穿越）级别的问题**必须修复**，不得用注释屏蔽（`#nosec`）除非有书面说明
- `govulncheck` 发现**高危漏洞**时阻断 CI，中低危须在 sprint 内排期修复

---

## 危险 API 使用规范

### os/exec：禁止命令注入

用户输入**不得**直接拼入命令字符串，必须作为独立参数传递：

```go
// ❌ 危险：shell 注入
cmd := exec.Command("sh", "-c", "echo " + userInput)

// ✅ 正确：参数独立传递，不经过 shell 解析
cmd := exec.Command("echo", userInput)

// ✅ 正确：必须使用 shell 时，对输入严格白名单校验
```

- 生产代码中**禁止使用** `exec.Command("sh", "-c", ...)` 或 `exec.Command("bash", "-c", ...)`
- 若确实需要执行 shell 命令，须经 Tech Lead 审批并说明必要性

### path/filepath：防止路径穿越

用户提供的文件路径必须经过清洗和边界校验：

```go
// ❌ 危险：用户可传入 "../../etc/passwd"
filePath := filepath.Join(baseDir, userInput)
data, _ := os.ReadFile(filePath)

// ✅ 正确：Clean 后再校验是否在允许目录内
func safeOpen(baseDir, userInput string) (*os.File, error) {
    cleaned := filepath.Clean(filepath.Join(baseDir, userInput))
    if !strings.HasPrefix(cleaned, filepath.Clean(baseDir)+string(os.PathSeparator)) {
        return nil, errors.New("path traversal detected")
    }
    return os.Open(cleaned)
}
```

### unsafe 包

**业务代码中禁止使用 `unsafe` 包**。若性能优化确有必要，须满足：
1. 有 benchmark 数据证明瓶颈
2. 经 Tech Lead 审批
3. 注释说明用途和风险

### math/rand vs crypto/rand

| 场景 | 包 |
|------|-----|
| Token、Session ID、验证码、密钥 | `crypto/rand` |
| 测试数据、随机排序、非安全场景 | `math/rand` |

```go
// ❌ 危险：math/rand 可预测
token := fmt.Sprintf("%d", rand.Int63())

// ✅ 正确：crypto/rand 生成不可预测的 Token
func generateToken() (string, error) {
    b := make([]byte, 32)
    if _, err := rand.Read(b); err != nil { // crypto/rand
        return "", err
    }
    return base64.URLEncoding.EncodeToString(b), nil
}
```

---

## 密码学规范

### 禁止使用的算法

| 禁止 | 原因 | 替代 |
|------|------|------|
| MD5 | 碰撞攻击已实用化 | SHA-256 / SHA-3 |
| SHA-1 | 碰撞攻击已实用化 | SHA-256 / SHA-3 |
| DES / 3DES | 密钥长度不足 | AES-256-GCM |
| ECB 模式 | 不加密模式，相同明文产生相同密文 | GCM / CBC+HMAC |
| RSA < 2048 位 | 密钥长度不足 | RSA-2048 / ECDSA P-256 |

### 密码存储

**禁止**将用户密码以明文、MD5、SHA-1、可逆加密方式存储：

```go
// ❌ 禁止
hash := md5.Sum([]byte(password))
hash := sha1.Sum([]byte(password))
encrypted := aesEncrypt(password, key) // 可逆加密

// ✅ 正确：使用 bcrypt（或 argon2id）
import "golang.org/x/crypto/bcrypt"

func hashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(bytes), err
}

func checkPassword(password, hash string) bool {
    return bcrypt.CompareHashAndPassword([]byte(hash), []byte(password)) == nil
}
```

### 对称加密

需要对称加密时，统一使用 **AES-256-GCM**（提供认证加密，防篡改）：

```go
// 使用 crypto/aes + crypto/cipher，nonce 必须随机生成，禁止复用
```

---

## 请求安全

### 请求体大小限制

防止超大请求体导致 OOM，在 Gin 中间件或路由级别限制：

```go
// server/gin.go：全局限制（推荐）
engine.Use(func(c *gin.Context) {
    c.Request.Body = http.MaxBytesReader(c.Writer, c.Request.Body, maxRequestBodySize)
    c.Next()
})

const maxRequestBodySize = 4 << 20 // 4 MB，根据业务实际调整
```

- 文件上传接口使用框架 `controller.UploadSingleFile` 的 `size` 参数控制
- 普通 JSON 接口默认限制 **4 MB**，特殊需求须注释说明

### HTTP Server 超时

`ReadHeaderTimeout` 已在项目模板中配置，还需补充完整超时：

```go
srv := &http.Server{
    Addr:              fmt.Sprintf(":%d", opt.Port),
    Handler:           engine,
    ReadHeaderTimeout: time.Duration(cfg.ReadHeaderTimeout) * time.Second, // 已有
    ReadTimeout:       30 * time.Second,   // 读取完整请求（含 Body）的超时
    WriteTimeout:      60 * time.Second,   // 写响应的超时（含业务处理时间）
    IdleTimeout:       120 * time.Second,  // Keep-Alive 空闲连接超时
}
```

- `WriteTimeout` 必须大于业务处理的最大预期耗时
- 这些值通过配置文件读取，不得硬编码

### SSRF 防护

调用外部 HTTP 服务时，若 URL 来自用户输入或数据库，必须进行校验：

```go
// ❌ 危险：直接使用用户提供的 URL
resp, err := http.Get(userProvidedURL)

// ✅ 正确：白名单校验域名
var allowedHosts = map[string]bool{
    "api.example.com": true,
    "cdn.example.com": true,
}

func safeGet(rawURL string) (*http.Response, error) {
    u, err := url.Parse(rawURL)
    if err != nil || !allowedHosts[u.Hostname()] {
        return nil, allerror.New("invalid_url", "url not allowed", nil)
    }
    return http.Get(rawURL)
}
```

---

## SQL 注入防护

GORM 参数化查询已天然防注入，但**排序字段**是例外。

### 排序字段必须白名单校验

框架 `controller.ReqToOrder` 的 `orderby` 参数由调用方负责防注入（见 `framework-api.md`）：

```go
// domain/repository/repo.go：定义允许排序的字段白名单
var allowedOrderFields = map[string]bool{
    "created_at": true,
    "updated_at": true,
    "status":     true,
}

// repositoryimpl：使用前校验
func (impl *orderImpl) List(ctx context.Context, opt *repository.ListOpt) ([]repository.OrderInfo, int64, error) {
    dao := impl.dao.New(ctx)
    query := dao.WithContext(ctx).Model(&orderDO{})

    if opt.Order != "" {
        field := strings.Split(opt.Order, " ")[0] // "created_at DESC" → "created_at"
        if !allowedOrderFields[field] {
            return nil, 0, allerror.New("invalid_order_field", "unsupported order field", nil)
        }
        query = query.Order(opt.Order)
    }
    // ...
}
```

- 白名单定义在 Repository 实现层，与数据库列名对应
- `opt.Order` 由框架 `ToOrder()` 生成（格式固定为 `"field ASC|DESC"`），Repository 层仍须校验字段名

---

## 竞态条件检测

**所有单元测试和集成测试必须启用 `-race` 标志：**

```bash
# 本地运行
go test -race ./...

# CI 配置
go test -race -coverprofile=coverage.out ./...
```

- `-race` 会使内存消耗增加约 5-10 倍，运行时间增加 2-20 倍，仅用于测试环境
- Race detector 发现的冲突**必须修复**，不得忽略
- 共享状态的访问（全局变量、map、slice 追加）必须通过 `sync.Mutex` / `sync.RWMutex` / `sync/atomic` / channel 保护

---

## 禁止行为汇总

| 禁止 | 说明 |
|------|------|
| `exec.Command("sh", "-c", userInput)` | 命令注入 |
| `filepath.Join(base, userInput)` 不校验边界 | 路径穿越 |
| `import "unsafe"` 在业务代码中 | 内存安全风险 |
| `math/rand` 用于安全敏感场景 | 可预测 |
| MD5/SHA-1/DES 用于密码或加密 | 算法已破解 |
| 密码明文或可逆加密存储 | 数据泄露风险 |
| 用户提供 URL 不校验直接请求 | SSRF |
| `orderby` 参数不做白名单校验 | SQL 注入 |
| 测试不带 `-race` 标志 | 竞态条件漏检 |
| 屏蔽 gosec 报告的高危规则（`#nosec G201`） | 绕过安全检查 |
