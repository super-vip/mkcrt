
# mkcrt

`mkcrt` 是一款生产级工具，用于生成完整的自签名 PKI 基础设施，包括根 CA、服务端证书和客户端证书，并自动管理 DNS 和 IP 地址类型的 SAN（Subject Alternative Name）扩展字段。

## 核心特性
- 生成完整的 PKI 层级结构（根 CA + 服务端 + 客户端证书）
- 自动化 SAN 管理：基础条目 + 本机 IP 地址 + 自定义条目（自动去重并排序）
- SAN 排序规则：**IPv4 优先 → IPv6 次之 → 域名最后**，提升可读性
- 支持 RSA（2048/4096 位）和 Ed25519（椭圆曲线算法）
- 根 CA 私钥采用 AES128/AES256 加密保护
- 可选 **PKCS#1 私钥格式**，完全兼容 OpenSSL 工具链
- 跨平台支持（Linux、Windows、macOS）
- 内置密钥-证书配对校验，避免公私钥不匹配问题
- 符合标准 X.509 规范，兼容各类软件系统

## 安装
从 [Releases](https://git.oso.plus/yfc/mkcrt/releases) 页面下载对应操作系统和架构的二进制文件。

## 使用方法

### 基础服务端证书生成
使用默认配置生成完整 PKI 套件（应用名称：nginx，输出目录：./certs）：
```sh
mkcrt -app nginx
```

### 自定义 SAN 配置
生成包含自定义 SAN 条目且使用 Ed25519 算法的证书：
```sh
mkcrt -app k8s -ed25519 -sans "10.96.0.1,kubernetes.default.svc,*.kube-system.svc" -out ./k8s-certs
```

### 兼容 OpenSSL 的 RSA 证书（PKCS#1 格式）
生成传统 PKCS#1 格式的 RSA 私钥，获得最佳 OpenSSL 兼容性：
```sh
mkcrt -app nginx -pkcs1 -out ./nginx-certs
```

### 客户端证书生成
客户端证书会随服务端证书自动生成。如需自定义客户端通用名：
```sh
mkcrt -app mysql -client-cn "mysql-client-prod" -out ./mysql-certs
```

### 完整参数说明
```sh
mkcrt [参数]
```

#### 核心参数
| 参数 | 描述 | 默认值 |
|------|------|--------|
| `-app` | 应用名称（用于生成默认 SAN 条目） | myapp |
| `-out` | 证书输出根目录 | ./certs |
| `-days` | CA 证书有效期（天） | 3650 |
| `-server-days` | 服务端证书有效期（天） | 365 |
| `-client-days` | 客户端证书有效期（天） | 365 |
| `-sans` | 额外 SAN 条目（逗号分隔） | "" |
| `-ed25519` | 使用 Ed25519 替代 RSA | false |
| `-key-bits` | RSA 密钥长度（2048/4096） | 2048 |
| `-pkcs1` | 输出 PKCS#1 格式 RSA 私钥（OpenSSL 原生格式） | false |
| `-ca-cn` | 根 CA 通用名 | Root CA |
| `-server-cn` | 服务端证书通用名（默认使用 app 名称） | "" |
| `-client-cn` | 客户端证书通用名（默认：app-client） | "" |
| `-ca-pass` | 根 CA 私钥加密密码（为空则自动生成） | "" |
| `-cipher` | CA 私钥加密算法（aes128/aes256） | aes256 |
| `-v` | 显示版本号并退出 | false |

## 输出目录结构
工具会创建结构化的目录布局，包含所有所需证书：
```
./certs/
├── ca/
│   ├── root-ca-cert.pem  # 根 CA 证书
│   └── root-ca-key.pem   # 加密的根 CA 私钥
├── server/
│   ├── server-cert.pem   # 服务端证书
│   ├── server-key.pem    # 服务端私钥
│   └── fullchain.pem     # 服务端证书 + CA 证书链（合并文件）
├── client/
│   ├── client-cert.pem   # 客户端证书
│   └── client-key.pem    # 客户端私钥
└── keystore/
    └── ca-cert.pem       # 信任库（CA 证书）
```

## 证书格式转换

### PEM 转 PKCS12（Java / 通用应用）
```sh
openssl pkcs12 -export -in server-cert.pem -inkey server-key.pem -out server.p12 -name server -CAfile ca/root-ca-cert.pem -caname CA
```

### PKCS12 转 JKS（Java 密钥库）
#### 服务端密钥库
```bash
keytool -importkeystore -srckeystore server.p12 -srcstoretype PKCS12 -destkeystore kafka.server.keystore.jks
```

#### 信任库
```bash
keytool -import -file ca/root-ca-cert.pem -alias CA -keystore kafka.server.truststore.jks
```

## 兼容性
本工具生成的标准 X.509 证书兼容以下系统/软件：
- Kubernetes（APIServer、Ingress）
- MySQL
- Kafka
- Redis
- Nginx
- Nacos
- ClickHouse
- Elasticsearch
- MinIO
- GitLab
- Harbor
- Kuboard
- Nexus
- 所有标准 TLS/SSL 应用程序

## 安全注意事项
1. 务必安全存储根 CA 私钥密码
2. 设置合理的文件权限（私钥 0600，证书 0644）
3. 尽可能使用 Ed25519 算法，获得更高的安全性和性能
4. 证书过期前及时轮换（默认：服务端/客户端证书 1 年有效期）
5. 仅添加必要的 SAN 条目，最小化攻击面
6. 自动密钥-证书校验确保无配对错误

## 版本
当前版本：1.0.0
