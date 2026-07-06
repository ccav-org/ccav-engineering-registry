# CCAV Failover Design

> **版本**: V1.0 Frozen
> **HA 策略**: Active-Standby（主备模式）
> **Validate date**: 2026-07-06

---

## 架构概览

```
                    nginx
                      │
          ┌───────────┴───────────┐
          │                       │
   ccav_gateway_ha         ccav_data_proxy_ha
   ┌──────┴──────┐         ┌───────┴───────┐
   │             │         │               │
:8000 (primary) :8002    :8001 (primary)  :8003
(standby)                          (standby)
```

- **两层 HA**: Gateway 层 + Data Proxy 层
- **模式**: 主备（非负载均衡）——同一时间只有 primary 服务
- **故障检测**: nginx `proxy_next_upstream`
- **触发条件**: error, timeout, http_502, http_503

---

## Failover 流程

```
1. Primary (:8000/:8001) 故障
2. nginx 检测到 error/timeout/502/503
3. nginx 重试 standby (:8002/:8003)
4. 最多重试 2 次
5. 请求转发至 standby，正常响应
6. systemd 检测 primary 退出 → Restart=always → 自动恢复
7. 后续请求回到已恢复的 primary
```

**Failover 时间**: < 1 秒（单次重试延迟）
**自动恢复**: systemd 重启 primary 后自动回到 active 状态

---

## nginx 配置关键段

```nginx
upstream ccav_gateway_ha {
    server 127.0.0.1:8000;  # primary
    server 127.0.0.1:8002 backup;  # standby
}

upstream ccav_data_proxy_ha {
    server 127.0.0.1:8001;  # primary
    server 127.0.0.1:8003 backup;  # standby
}

# 全局 502 retry
proxy_next_upstream error timeout http_502 http_503;
proxy_next_upstream_tries 2;
```

---

## systemd 恢复

```ini
[Service]
Restart=always
RestartSec=5
```

- 进程无论任何原因退出，5 秒后自动重启
- 配合 `WantedBy=multi-user.target` 实现开机自启

---

## 数据库 HA

| 层 | 状态 | 说明 |
|----|------|------|
| PostgreSQL | 单节点 | 无 replica，风险可接受（有 SQLite 回滚） |
| SQLite | 单文件 | 文件级备份即可 |
| 回滚路径 | 手动 | 切换 nginx upstream 回 Node 直读 SQLite |

---

## 已知限制

| 限制 | 影响 | 优先级 |
|------|------|--------|
| PG 无主从 | 数据库故障需手动恢复 | 低（有 SQLite fallback） |
| 无自动回滚 | 需人工切换 nginx | 低（一致性验证 100%，回滚窗口极小） |
| nginx 单点 | 如 nginx 挂掉，整体不可用 | 中（nginx 极稳定，按需加 keepalived） |
| 无负载均衡 | 所有流量走 primary | 低（当前流量不足以触发瓶颈） |

---

## 测试记录

- ✅ Gateway primary :8000 down → 请求自动路由到 :8002
- ✅ Data Proxy primary :8001 down → 请求自动路由到 :8003
- ✅ systemd 重启 primary → 故障恢复后自动回到 active
- ✅ 502 retry 2 次正常工作
