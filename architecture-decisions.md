# CCAV 架构决策记录 (ADR)

> 按照 chronologial 顺序列出关键架构决策，含时间、上下文、决策及理由。

---

## ADR-001: Gateway 为主入口（B 方案）

- **日期**: 2026-06
- **上下文**: 存在两套 API: Python FastAPI Gateway 和 Node Express `server.mjs`
- **决策**: 选择 Python FastAPI Gateway 为主入口，Node 降级为遗留数据 API 层
- **理由**:
  - Gateway 为 SSOT 对齐的新代码
  - L3 Provider 架构天然匹配 FastAPI
  - Node 保留向后兼容，不破坏旧业务
- **影响**:
  - nginx 路由 `/api/gateway/*` → Gateway
  - `/api/node/*` → Node (后经 Data Proxy 代理)
  - AI 路径全部走 Gateway

---

## ADR-002: 渐进式数据库迁移

- **日期**: 2026-07
- **上下文**: SQLite 是 Node 唯一数据源，引入 PostgreSQL 必须零中断
- **决策**: 影子写入 → 验证 → 代理读 → 始终保留 SQLite 回滚
- **理由**:
  - 不修改 Node 代码
  - 回滚时间 <30 秒（切换 nginx upstream）
  - 风险零
- **影响**:
  - 引入 dual-write daemon
  - 引入 data proxy sidecar
  - 数据一致性监控上线

---

## ADR-003: 全组件 systemd 守护

- **日期**: 2026-07
- **上下文**: 系统需 7×24 运行，进程退出后必须自动恢复
- **决策**: 所有运行时组件均以 systemd 管理，Restart=always
- **理由**: 生产环境标准做法，Linux 原生支持
- **影响**:
  - 6 个 systemd unit: ccav-api, ccav-gateway, ccav-gateway-2, ccav-data-proxy, ccav-data-proxy-2, ccav-shadow-write
  - 开机自启，异常自动恢复

---

## ADR-004: Gateway HA（主备模式）

- **日期**: 2026-07
- **上下文**: 单点 Gateway 风险
- **决策**: Primary (:8000) + Standby (:8002)，nginx upstream 主备 + proxy_next_upstream 重试
- **理由**:
  - 不引入额外复杂度（无负载均衡器）
  - failover <1s，满足中台要求
- **影响**:
  - nginx upstream `ccav_gateway_ha`
  - 502 retry 2 次
  - 自动恢复无人工干预

---

## ADR-005: Data Proxy HA（主备模式）

- **日期**: 2026-07
- **上下文**: 数据代理层同样存在单点风险
- **决策**: 与 Gateway HA 相同策略——:8001 (primary) + :8003 (standby)
- **理由**: 对称设计，运维一致
- **影响**: nginx upstream `ccav_data_proxy_ha`

---

## ADR-006: 系统冻结

- **日期**: 2026-07-06
- **上下文**: 架构完整、HA 就绪、数据一致、监控闭环
- **决策**: 冻结所有架构变更，仅允许运维操作
- **理由**:
  - 系统已到生产平台基线
  - 避免架构漂移
  - 进入可维护稳定态
- **影响**:
  - 不再新增 CP
  - mode="FROZEN" 写入 status 端点
  - 冻结快照生成

---

## ADR-007: L3 真实调用优先于 SSOT 完整对齐

- **日期**: 2026-07
- **上下文**: SSOT 有 14 个 Provider 文件但使用 fake adapters，真实 AI 调用需立即工作
- **决策**: 先部署 clients.py 确保真实调用路径，再逐步对齐 SSOT adapters
- **理由**: 业务价值优先——空有设计不产生用户价值
- **影响**: `/api/v1/providers` 列出能力但真实调用走 clients.py 而非 SSOT adapters

---

## ADR-008: 速率限制分层（nginx + Gateway middleware）

- **日期**: 2026-07
- **上下文**: 需同时防御 DDoS（nginx 层）和 API 滥用（应用层）
- **决策**:
  - nginx: `limit_req` 5 r/s burst=20（全局 `/api/`）
  - Gateway: 滑动窗口 30 req/s per IP（中间件）
- **理由**: 纵深防御——nginx 挡大流量，Gateway 做精细控制
- **影响**: 双重限流，429 返回 `{"error":"rate_limited"}`
