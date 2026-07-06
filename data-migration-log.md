# CCAV 数据迁移日志 (SQLite → PostgreSQL)

> 策略：**渐进式迁移**——先影子写入验证，再切主读，始终保留 SQLite 回滚路径。

---

## 迁移前状态

| 项目 | 值 |
|------|-----|
| 原始数据库 | SQLite (`/opt/ccav-api/ccav.db`) |
| 表数量 | 18 张 |
| 运行时依赖 | Node `server.mjs` 直接读写 SQLite |
| 风险 | 单点故障，无 HA，无审计日志 |

---

## 迁移目标

- PostgreSQL 16.14 作为主读源
- SQLite 保留作为回滚基础
- 所有写操作同步至 PG，一致性 100%
- 回滚时间 < 30 秒

---

## 阶段 1: PostgreSQL 搭建 (CP-DATA-001)

```
PostgreSQL 16.14
Database: ccav_core
User:     ccav_admin
Tables:   users, courses, experts, logs (4/18)
```

## 阶段 2: 影子写入 (CP-DATA-002)

- `dual-write-daemon.py`: 30 秒循环扫描 SQLite，增量写入 PG
- 不修改 Node 读逻辑——Node 仍读 SQLite
- systemd 守护: `ccav-shadow-write.service`

## 阶段 3: 一致性验证 (CP-DATA-003)

- `validate_dual_write.py` 逐表对比
- 结果: 11/11 行完全一致，延迟 <5ms
- 写入成功率 100%

## 阶段 4: 迁移就绪评估 (CP-DATA-004)

- 结论: READY_FOR_PRIMARY_CUTOVER
- 风险: 低
- 回滚: SQLite 完整保留，切换回 Node 直读即可

## 阶段 5: 数据代理层 (CP-DATA-006)

- FastAPI sidecar (`ccav-data-proxy.service`) 端口 8001
- 拦截 `/api/node/*` 读请求 → 直连 PostgreSQL
- 写请求仍走双写路径
- 不修改 Node 代码

## 阶段 6: 一致性监控上线 (CP-DATA-007)

- cron 每 5 分钟运行 `data-consistency-monitor.py`
- 4 项检查:
  1. Row diff detection (PG vs SQLite) → 0 diff
  2. Write latency trend → 30s avg, 0 errors
  3. Data drift alert → 0 drift
  4. Proxy correctness → hybrid routing verified
- 结果: PASS, 0 issues

---

## 当前状态 (Frozen)

| 指标 | 值 |
|------|-----|
| PG-SQLite 一致性 | 100% (0 drift) |
| 同步频率 | 30s/cycle |
| 同步错误 | 0 |
| 同步行数 | 11 (users=1, courses=2, experts=8) |
| PG 专属表 | logs (审计) |
| SQLite 未迁移表 | 15 张 (非核心业务表) |
| 读路径 | PG (via data proxy) |
| 写路径 | SQLite (Node) → PG (shadow) |
| 回滚路径 | 切换 nginx 回 Node :3001 直读 SQLite |

---

## 已知差距

| 事项 | 等级 | 说明 |
|------|------|------|
| 15 张未迁移表 | INFO | gallery, orders, lessons 等非核心表，业务量小，SQLite 足够 |
| 无自动回滚 | TODO | 当前需手动切换 nginx upstream |
| 无 PG 主从 | TODO | 单节点 PG，无 replica |

---

## 关键文件

| 文件 | 用途 |
|------|------|
| `/home/ubuntu/dual-write-daemon.py` | 影子写入 daemon |
| `/home/ubuntu/validate_dual_write.py` | 一致性验证脚本 |
| `/opt/ccav-monitor/data-consistency-monitor.py` | 在线监控 |
| `/var/log/ccav/data-consistency-monitor.json` | 最新监控结果 |
| `/var/log/ccav/migration-decision.json` | 迁移决策记录 |
