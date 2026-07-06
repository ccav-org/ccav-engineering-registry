# CCAV Engineering Registry

> **冻结时间**: 2026-07-06
> **冻结版本**: CCAV V1.0 Frozen Production Platform
> **冻结机关**: CTO Final Closure (CP-SRE-001)
> **归档人**: 王万里 (云端 AI 助手，运行于腾讯云)

---

## 目的

本仓库是 CCAV 系统从「构建」到「冻结」全生命周期的工程知识归档。
**不修改代码，不新增架构**——仅做知识冻结。

---

## 仓库结构

| 文件/目录 | 内容 |
|-----------|------|
| `README.md` | 本索引（你正在读的） |
| `cp-execution-log.json` | 全部 CP 任务执行记录 (CP-0001 → CP-DATA-007) |
| `data-migration-log.md` | SQLite → PostgreSQL 渐进式迁移完整记录 |
| `architecture-decisions.md` | 架构决策记录 (ADR) |
| `runtime-topology.md` | 当前冻结时点的运行时拓扑 |
| `runtime-topology.json` | 拓扑 JSON（机器可读） |
| `failover-design.md` | HA 与 failover 设计 |
| `security-notes.md` | 安全加固状态与已知风险 |
| `adr/` | 各 ADR 的独立备份 |
| `topology/` | 拓扑图及配置文件快照 |
| `runbooks/` | 运维 runbook（重启、回滚、故障诊断） |
| `logs/` | 冻结时刻的日志快照 |

---

## 系统能力速查

| 能力域 | 状态 |
|--------|------|
| Runtime (systemd + HA) | 🟢 生产级 |
| Execution (L3 provider routing) | 🟢 生产级 |
| Data (dual-write, PG primary, SQLite fallback) | 🟢 100% 一致性，0 drift |
| Traffic (nginx HA routing, rate limiting) | 🟢 生产级 |
| Observability (health, logs, consistency monitor) | 🟢 完整基线 |
| Control Plane (CP execution log) | 🟢 完整 |
| Resilience (HA failover <1s) | 🟢 已验证 |
| Security (rate limit, header hardening, basic auth) | 🟡 基线完整，密钥管理待加强 |

---

## 冻结规则

- ❌ 不再新增 CP
- ❌ 不再修改架构
- ❌ 不再扩展系统
- ✅ 仅允许运维变更（重启、调优阈值、密钥轮换、报警配置）
- ✅ 只运行，只监控，只报警

---

## 关键文件位置（运行时）

| 组件 | 路径 |
|------|------|
| Gateway (FastAPI) | `/ccav-org-os/services/gateway/api_gateway.py` |
| AI Clients | `/ccav-org-os/services/gateway/providers/clients.py` |
| L3 Handler | `/ccav-org-os/services/gateway/l3_handler.py` |
| Node (Express) | `/opt/ccav-api/server.mjs` |
| Nginx 配置 | `/etc/nginx/conf.d/ccav.conf` |
| Systemd Units | `/etc/systemd/system/ccav-*.service` |
| 一致性监控 | `/opt/ccav-monitor/data-consistency-monitor.py` |
| 冻结快照 | `/opt/ccav-monitor/freeze_snapshot.py` |
| 日志 | `/var/log/ccav/` |
| 冻结状态 | `/var/log/ccav/CCAV_V1_FROZEN.json` |
