# CCAV Security Notes

> **状态**: 🟡 Baseline Complete — 密钥管理待加强
> **冻结时间**: 2026-07-06

---

## 已实施安全措施

### nginx 层

| 措施 | 配置 |
|------|------|
| TLS 1.2/1.3 | Let's Encrypt 证书，自动续期 |
| HSTS | `max-age=31536000; includeSubDomains` |
| X-Content-Type-Options | `nosniff` |
| X-Frame-Options | `SAMEORIGIN` |
| Referrer-Policy | `strict-origin-when-cross-origin` |
| 敏感路径屏蔽 | `.git`, `.env`, `.aws`, `.ht*` → 444 |
| PHP 探测屏蔽 | `wp-login`, `xmlrpc`, `.php$` → 444 |
| HTTP 方法限制 | 仅 GET/HEAD/POST/OPTIONS |
| 并发限制 | 单 IP 10 连接 |

### 应用层

| 措施 | 配置 |
|------|------|
| 速率限制 (nginx) | `/api/` 5r/s burst=20 |
| 速率限制 (nginx) | `/api/register/` 5r/s burst=10 |
| 速率限制 (Gateway) | 30 req/s per IP, sliding window 1s |
| Admin 认证 | Basic Auth (`/admin/`) |
| GoAccess 认证 | Basic Auth (`/stats`) |
| Gateway Dry-Run 门禁 | Header `X-CCAV-Gateway-Dryrun: phase01j` |
| API v1 未匹配路径 | `return 403` |
| 请求体大小限制 | gateway-api 1MB |

---

## 已知风险 (AR)

### AR-01: 硬编码 API 密钥 (高危)

- **位置**: `/opt/ccav-api/server.mjs`
- **详情**: Kling API 密钥硬编码在代码中
- **影响**: 源码泄露 = 密钥泄露
- **当前状态**: 未修复（系统冻结期间不修改 Node 代码）
- **缓解**: AI 请求已被 nginx 拦截走 Gateway，Node 内硬编码密钥不再对外暴露
- **建议**: 系统解冻后迁移至环境变量或 Vault

### AR-02: 无密钥轮换机制

- **影响**: 密钥泄露后无法快速轮换
- **建议**: 引入 Vault 或至少 `.env` 管理

### AR-03: 无 RBAC

- **影响**: Admin 面板仅靠 Basic Auth，无细粒度权限
- **建议**: 引入 JWT 或 OAuth2

### AR-04: PostgreSQL 单节点

- **影响**: 无 replica，无自动 failover
- **缓解**: SQLite 作为回滚路径
- **建议**: 按需加 streaming replica

### AR-05: 无 WAF

- **影响**: SQL 注入、XSS 等攻击依赖应用层防御
- **建议**: Cloudflare 或 ModSecurity

---

## 不修的原因

系统已冻结为 V1.0。以上风险均已知且已评估，在当前业务规模下风险可控。解冻后按优先级排期。

---

## 运维安全建议

| 操作 | 频率 | 说明 |
|------|------|------|
| nginx 日志审计 | 每周 | 检查异常访问模式 |
| 证书到期检查 | 每月 | `certbot certificates` |
| 磁盘空间检查 | 每周 | `/var/log/ccav/` 日志增长 |
| 密钥轮换 | 按需 | 当前无自动机制 |
| 安全更新 | 每月 | `apt update && unattended-upgrades` |
