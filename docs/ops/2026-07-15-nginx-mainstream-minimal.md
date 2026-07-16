# 运维变更记录：Nginx 按 NewAPI 社区主流精简

- 日期：2026-07-15
- 操作者：codexdebug（经 Hermes 远程）
- 相关服务：NewAPI（new.xdauv.xyz）、CLIProxyAPI/CPA（cpa.xdauv.xyz）
- 目的：按 NewAPI Issues 社区主流方案，只保留中转必设项

## 参考来源

1. QuantumNous/new-api issue 2974（维护者 Calcium-Ion）
   - 504 多来自反代默认 60s
   - 建议：proxy_read_timeout 300s; proxy_send_timeout 300s;
2. QuantumNous/new-api issue 4364（宝塔默认反代）
   - 评论：超时和禁缓存，中转必设
   - 参考：proxy_buffering off; proxy_cache off; 超时 600s
3. QuantumNous/new-api issue 1019 / 971
   - 主流最小集：反代头 + 超时 + 关缓冲

## 本次落地配置（最小有效集）

newapi / cpa 的 location 仅保留：

- proxy_pass
- proxy_http_version 1.1
- Host / X-Real-IP / X-Forwarded-For / X-Forwarded-Proto
- proxy_buffering off
- proxy_cache off
- proxy_read_timeout 300s
- proxy_send_timeout 300s

另保留站点级 client_max_body_size 100m。

## 相比上一版（完整加固）删掉了什么

- map http_upgrade / connection_upgrade.conf
- Connection / Upgrade 头
- proxy_socket_keepalive
- proxy_request_buffering
- proxy_connect_timeout / send_timeout
- chunked_transfer_encoding
- X-Accel-Buffering

原因：社区主流不依赖这些；奥卡姆剃刀只留中转必设。

## 备份

- /etc/nginx/sites-available/newapi.bak.20260715_141901
- /etc/nginx/sites-available/cpa.bak.20260715_141901
- /etc/nginx/conf.d/connection_upgrade.conf.bak.*
- 更早完整版备份：*.bak.20260715_132506

## 生效结果

- nginx -t 通过
- reload 成功，active
- https://new.xdauv.xyz/api/status -> 200
- https://cpa.xdauv.xyz/ -> 200

## 说明

本改动修的是反代流式短板（默认 60s / 缓冲）。
client_gone 若仍见 18-19s 主峰，继续查上游静默、CPA keepalive、客户端断开，而不是继续堆 Nginx 参数。
