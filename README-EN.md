
# mkcrt

`mkcrt` is a production-grade tool to generate a complete self-signed PKI infrastructure, including Root CA, server certificates, and client certificates with automatically managed SAN (Subject Alternative Name) entries for DNS and IP addresses.

## Key Features
- Generate complete PKI hierarchy (Root CA + server + client certificates)
- Automatic SAN management: base entries + local IP addresses + custom entries (deduplicated and sorted)
- SAN sorted by **IPv4 → IPv6 → domain name** for better readability
- Support for RSA (2048/4096 bits) and Ed25519 (elliptic curve algorithm)
- Encrypted Root CA private key with AES128/AES256
- Optional **PKCS#1 private key format** for full OpenSSL compatibility
- Cross-platform support (Linux, Windows, macOS)
- Built-in key-certificate pair validation to avoid mismatched keys
- Standard X.509 compliance for broad software compatibility

## Install
Download the appropriate binary for your operating system and architecture from the [Releases](https://git.oso.plus/yfc/mkcrt/releases) page.

## Usage

### Basic Server Certificate Generation
Generate a complete PKI set with default configurations (app name: nginx, output: ./certs):
```sh
mkcrt -app nginx
```

### Custom SAN Configuration
Generate certificates with custom SAN entries and Ed25519 algorithm:
```sh
mkcrt -app k8s -ed25519 -sans "10.96.0.1,kubernetes.default.svc,*.kube-system.svc" -out ./k8s-certs
```

### OpenSSL Compatible RSA Certificates (PKCS#1)
Generate RSA private keys in traditional PKCS#1 format for best OpenSSL compatibility:
```sh
mkcrt -app nginx -pkcs1 -out ./nginx-certs
```

### Client Certificate Generation
Client certificates are automatically generated alongside server certificates. To customize the client common name:
```sh
mkcrt -app mysql -client-cn "mysql-client-prod" -out ./mysql-certs
```

### Full Usage Options
```sh
mkcrt [flags]
```

#### Core Flags
| Flag | Description | Default Value |
|------|-------------|---------------|
| `-app` | Application name (used for default SAN entries) | myapp |
| `-out` | Root directory for certificate output | ./certs |
| `-days` | CA certificate validity period (days) | 3650 |
| `-server-days` | Server certificate validity period (days) | 365 |
| `-client-days` | Client certificate validity period (days) | 365 |
| `-sans` | Additional SAN entries (comma-separated) | "" |
| `-ed25519` | Use Ed25519 instead of RSA | false |
| `-key-bits` | RSA key length (2048/4096) | 2048 |
| `-pkcs1` | Output RSA private key in PKCS#1 (OpenSSL native format) | false |
| `-ca-cn` | Root CA common name | Root CA |
| `-server-cn` | Server certificate common name (default: app name) | "" |
| `-client-cn` | Client certificate common name (default: app-client) | "" |
| `-ca-pass` | Root CA private key encryption password (auto-generated if empty) | "" |
| `-cipher` | CA private key encryption algorithm (aes128/aes256) | aes256 |
| `-v` | Show version and exit | false |

## Output Structure
The tool creates a structured directory layout with all required certificates:
```
./certs/
├── ca/
│   ├── root-ca-cert.pem  # Root CA certificate
│   └── root-ca-key.pem   # Encrypted Root CA private key
├── server/
│   ├── server-cert.pem   # Server certificate
│   ├── server-key.pem    # Server private key
│   └── fullchain.pem     # Combined server + CA certificate chain
├── client/
│   ├── client-cert.pem   # Client certificate
│   └── client-key.pem    # Client private key
└── keystore/
    └── ca-cert.pem       # Truststore (CA certificate)
```

## Certificate Conversion

### PEM to PKCS12 (Java / Applications)
```sh
openssl pkcs12 -export -in server-cert.pem -inkey server-key.pem -out server.p12 -name server -CAfile ca/root-ca-cert.pem -caname CA
```

### PKCS12 to JKS (Java Keystore)
#### Server Keystore
```bash
keytool -importkeystore -srckeystore server.p12 -srcstoretype PKCS12 -destkeystore kafka.server.keystore.jks
```

#### Truststore
```bash
keytool -import -file ca/root-ca-cert.pem -alias CA -keystore kafka.server.truststore.jks
```

## Compatibility
This tool generates standard X.509 certificates compatible with:
- Kubernetes (APIServer, Ingress)
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
- All standard TLS/SSL applications

## Security Notes
1. Always store the Root CA private key password securely
2. Set appropriate file permissions (0600 for private keys, 0644 for certificates)
3. Use Ed25519 for enhanced security and performance where supported
4. Rotate certificates before expiration (default: 1 year for server/client)
5. Only add necessary SAN entries to minimize attack surface
6. Automatic key-certificate validation ensures no mismatched pairs

## Version
Current version: 1.0.0

---
