# CCAV Runtime Topology (Frozen)

> **快照时间**: 2026-07-06 09:35 UTC
> **版本**: CCAV V1.0 Frozen Production Platform

---

## 流量路径

```
Client (Internet)
 │
 ▼
Nginx :443 (TLS 1.2/1.3)
 │
 ├─→ /api/gateway/* ────→ ccav_gateway_ha (8000/8002)  ← FastAPI Gateway (主/备)
 ├─→ /api/v1/gateway/* ──→ ccav_gateway_ha
 ├─→ /api/v1/generate ──→ ccav_gateway_ha              ← L3 Provider 执行
 ├─→ /api/v1/providers ─→ ccav_gateway_ha              ← Provider 列表
 ├─→ /api/kimi/chat ────→ ccav_gateway_ha              ← Kimi AI
 ├─→ /api/optimize ─────→ ccav_gateway_ha              ← Kimi 优化
 ├─→ /api/kling/* ──────→ ccav_gateway_ha              ← 可灵 AI (300s timeout)
 ├─→ /api/replicate/* ──→ ccav_gateway_ha              ← Replicate (300s timeout)
 ├─→ /api/generate ─────→ ccav_gateway_ha              ← 通用 AI 生成 (300s timeout)
 ├─→ /gateway-api/* ────→ ccav_gateway_ha              ← 灰度入口 (dry-run)
 │
 ├─→ /api/node/* ───────→ ccav_data_proxy_ha (8001/8003) ← Data Proxy (读 PG)
 │  │                                                       │
 │  │  (读: experts, courses → PostgreSQL)                   │
 │  │  (读: gallery → Node :3001 via SQLite)                │
 │  │  (写: → Node :3001 → SQLite → shadow → PG)            │
 │  │
 │  └───────────────────────────────────────────────────────┘
 │
 ├─→ /api/* ────────────→ Node :3001 (ccav-api)          ← 通用业务 API (限流 5r/s)
 │
 ├─→ /api/auth/* ───────→ :8080 (报名系统 Ops API)
 ├─→ /api/enrollment/* ─→ :8080
 ├─→ /api/payment/* ────→ :8080
 ├─→ /api/refund/* ─────→ :8080
 ├─→ /api/dashboard/* ──→ :8080
 ├─→ /api/public/* ─────→ :8080
 ├─→ /api/register/* ──→ :9091 (Form Server, 限流 5r/s)
 ├─→ /admin/* ──────────→ :3002 (Admin 前端, Basic Auth)
 ├─→ /board/* ──────────→ :9090
 ├─→ /grafana/* ────────→ :3000 (监控看板)
 ├─→ /stats ────────────→ 静态文件 (GoAccess)
 │
 └─→ / ─────────────────→ 静态文件 (/var/www/ccav)
```

---

## 服务清单

| 服务 | 端口 | systemd Unit | 角色 | 状态 |
|------|------|-------------|------|------|
| Node API | 3001 | ccav-api | 业务核心 (Express) | 🟢 active |
| Gateway Primary | 8000 | ccav-gateway | 执行主节点 (FastAPI) | 🟢 active |
| Gateway Standby | 8002 | ccav-gateway-2 | 执行备节点 | 🟢 active |
| Data Proxy Primary | 8001 | ccav-data-proxy | 数据路由主节点 | 🟢 active |
| Data Proxy Standby | 8003 | ccav-data-proxy-2 | 数据路由备节点 | 🟢 active |
| Shadow Write | — | ccav-shadow-write | SQLite→PG 同步 | 🟢 active |
| Form Server | 9091 | ccav-form-server | 报名表单 | 🟢 active |

---

## 数据库拓扑

```
           ┌──────────────────────────┐
           │      Node :3001          │
           │   (Express server.mjs)   │
           │                          │
           │   读/写 SQLite           │
           └────────┬─────────────────┘
                    │
                    ▼
           ┌──────────────────────────┐
           │   SQLite                 │
           │   /opt/ccav-api/ccav.db  │
           │   18 tables              │
           └────────┬─────────────────┘
                    │ (shadow write, 30s)
                    ▼
           ┌──────────────────────────┐
           │   PostgreSQL 16.14       │
           │   ccav_core@localhost    │
           │   4 tables (synced)       │
           │   + 1 PG-only (logs)     │
           └────────▲─────────────────┘
                    │
           ┌────────┴─────────────────┐
           │   Data Proxy :8001/8003  │
           │   (hybrid routing)       │
           │   experts,courses → PG   │
           │   gallery → Node/SQLite  │
           │   写 → Node → SQLite → PG│
           └──────────────────────────┘
```

---

## nginx 关键配置摘要

| 配置项 | 值 |
|--------|-----|
| TLS | 1.2/1.3, Let's Encrypt |
| 并发限制 | 单 IP 10 连接 |
| 安全头 | nosniff, SAMEORIGIN, HSTS, Referrer-Policy |
| 无效方法 | 仅 GET/HEAD/POST/OPTIONS |
| 敏感路径 | .git/.env/.aws/php 等 → 444 |
| Gateway HA upstream | ccav_gateway_ha → 8000 primary + 8002 backup |
| Data Proxy HA upstream | ccav_data_proxy_ha → 8001 primary + 8003 backup |
| 502 Retry | 2 tries |
| Gateway API 灰度 | 需要 header `X-CCAV-Gateway-Dryrun: phase01j` |
