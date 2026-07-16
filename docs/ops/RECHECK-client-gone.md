# 复查清单：NewAPI client_gone 修复效果

- 创建：2026-07-15
- 用途：用户下次要求复查时，按本清单执行，勿重复堆 Nginx 参数

## 已完成修复（基线）

1. Nginx newapi + cpa（社区主流最小）
   - proxy_buffering off
   - proxy_cache off
   - proxy_read_timeout 300s
   - proxy_send_timeout 300s
   - 无 map / 无 Connection upgrade 写死
   - 生效约：2026-07-15 14:19 UTC（日志 CST 22:19）

2. CPA streaming keepalive
   - /opt/CLIProxyAPI/config.yaml
   - streaming.keepalive-seconds: 15
   - streaming.bootstrap-retries: 1
   - 容器 recreate 约：2026-07-15 14:36 UTC（日志 CST 22:36）
   - 版本当时：CLIProxyAPI v7.2.78

3. 运维记录目录
   - /opt/new-api/docs/ops/
   - /opt/CLIProxyAPI/docs/ops/

## 复查命令

```bash
# 1) 配置是否仍在
sudo sed -n "1,40p" /etc/nginx/sites-available/newapi | sed -n "/location/,/}/p"
sudo sed -n "148,160p" /opt/CLIProxyAPI/config.yaml
docker exec cli-proxy-api sed -n "148,160p" /CLIProxyAPI/config.yaml

# 2) client_gone 统计
docker logs new-api --since 30m 2>&1 | grep -c reason=client_gone
docker logs new-api --since 30m 2>&1 | grep -oE "reason=(client_gone|eof|done)" | sort | uniq -c
docker logs new-api --since 30m 2>&1 | grep reason=client_gone | grep -oE "received=[0-9]+" | sort | uniq -c

# 3) 是否仍扎 ch11 / 18-19s（consume log）
docker logs new-api --since 1h 2>&1 | grep "record consume log" | grep client_gone | tail
```

## 判断标准

- 好转：gone 率下降；received=1 尖峰消失；18-19s 扎堆减弱；不全是 ch11
- 未好：仍 18-19s + ch11 CAP-gpt 为主 → 查上游/账号/客户端，勿再拧 Nginx
- 配置回退：nginx 无 buffering off/300s，或 CPA 无 keepalive-seconds:15 → 先恢复配置

## 未做（可选）

- NewAPI STREAMING_TIMEOUT=180 可能仍未进入容器（compose 有、printenv 曾无）
- 需要时：cd /opt/new-api && docker compose up -d --force-recreate new-api
