# CCAV Runbook

> **适用版本**: CCAV V1.0 Frozen
> **运维权限**: root / sudo

---

## 1. 健康检查（一键）

```bash
curl -s https://ccav.com/api/v1/status | python3 -m json.tool
```

预期输出: `{"node":"ok","gateway":"ok","data_proxy":"ok","db":"ok","mode":"FROZEN","all_ok":true}`

---

## 2. 数据一致性检查

```bash
# 查看最新监控结果
cat /var/log/ccav/data-consistency-monitor.json | python3 -m json.tool

# 手动运行一次
python3 /opt/ccav-monitor/data-consistency-monitor.py
```

---

## 3. 服务管理

```bash
# 查看所有 CCAV 服务状态
systemctl status 'ccav-*'

# 重启单个服务
systemctl restart ccav-gateway

# 查看服务日志
journalctl -u ccav-gateway -n 50 --no-pager
```

---

## 4. Gateway 故障恢复

```
症状: /api/v1/status → gateway: "fail"
操作:
  1. systemctl restart ccav-gateway
  2. 等待 3 秒
  3. curl http://localhost:8000/health
  4. 如仍失败，检查 standby: curl http://localhost:8002/health
  5. nginx 已自动 failover 到 standby，用户无感知
```

---

## 5. Data Proxy 故障恢复

```
症状: /api/node/experts 返回错误
操作:
  1. systemctl restart ccav-data-proxy
  2. curl http://localhost:8001/health
  3. 如仍失败，standby 已接管
```

---

## 6. 数据库回滚（紧急）

```
场景: PostgreSQL 故障，需切回 SQLite
操作:
  1. 编辑 /etc/nginx/conf.d/ccav.conf
  2. 将 /api/node/ 的 proxy_pass 从 ccav_data_proxy_ha 改为 http://127.0.0.1:3001
  3. nginx -t && systemctl reload nginx
  4. 回滚时间 <30 秒
```

---

## 7. Shadow Write 检查

```bash
# 检查 daemon 状态
systemctl status ccav-shadow-write

# 查看同步日志
tail -20 /var/log/ccav/shadow-write.log
```

---

## 8. 冻结快照比对

```bash
# 比较当前状态与冻结点
python3 /opt/ccav-monitor/freeze_snapshot.py
```

---

## 9. 证书续期

```bash
certbot renew --dry-run  # 先试运行
certbot renew             # 正式续期
systemctl reload nginx
```

---

## 10. 服务端口清单

| 端口 | 服务 | systemd Unit |
|------|------|-------------|
| 3001 | Node API | ccav-api |
| 8000 | Gateway Primary | ccav-gateway |
| 8002 | Gateway Standby | ccav-gateway-2 |
| 8001 | Data Proxy Primary | ccav-data-proxy |
| 8003 | Data Proxy Standby | ccav-data-proxy-2 |
| 5432 | PostgreSQL | postgresql |
| 9091 | Form Server | ccav-form-server |
| 3002 | Admin Frontend | — |
