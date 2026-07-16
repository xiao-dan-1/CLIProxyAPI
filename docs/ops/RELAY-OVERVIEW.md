# NewAPI 中转总览：资料对照 + 现网配置 + 排查结论

- 日期：2026-07-15
- 主机：fresh-cats-1 / 67.230.187.192
- 用途：自学中转架构、配置问题、client_gone 结论、后续改造优先级
- 下次可说：读中转总览 / 打开 RELAY-OVERVIEW.md

## 一、这是什么

NewAPI（QuantumNous/new-api）= 统一 AI 网关：多上游聚合、发 Key、计费、分组、限流。
官方文档：https://docs.newapi.pro/zh/docs
GitHub：https://github.com/QuantumNous/new-api

链路：

用户(Codex/网页) -> 域名(Nginx HTTPS) -> NewAPI -> 渠道(CPA/第三方/官方) -> 模型

本机角色：
- NewAPI：new.xdauv.xyz -> 3000
- CPA：cpa.xdauv.xyz -> 8317
- sub2api：xdauv.xyz -> 8080
- grok2api：8000
- outlook：outlook.xdauv.xyz -> 8765
- x-ui：proxy.xdauv.xyz
- Nginx：80/443

## 二、网上/官方主流经验

官方：Docker Compose；SQLite 或 MySQL/PG；Redis 时注意密钥。
关键环境变量：SESSION_SECRET、CRYPTO_SECRET、SQL_DSN、REDIS_CONN_STRING、STREAMING_TIMEOUT（默认约 300s）。

社区 Nginx 共识（Issues 2974/4364/971）：
- proxy_buffering off
- proxy_cache off
- proxy_read_timeout / proxy_send_timeout 300s~600s
- proxy_http_version 1.1 + Host/XFF
- 默认 60s 反代是 504 重灾区

CPA：streaming.keepalive-seconds 可缓解 SSE 空闲误杀（本机已开 15）。

## 三、现网配置对照

已对齐主流：
1. Nginx newapi+cpa：buffering/cache off + 300s
2. CPA keepalive-seconds=15, bootstrap-retries=1
3. STREAMING_TIMEOUT=180 已进容器
4. HTTPS 域名齐全
5. ops 记录在 /opt/*/docs/ops/

问题/风险：
P0 内存仅 1G，多服务堆叠
P0 端口公网直出 3000/8317/8080/8000/8765
P0 弱密码倾向、codexdebug NOPASSWD ALL
P1 client_gone 高度集中 ch11 CAP-gpt
P1 compose 镜像 ghcr pull denied
P2 一台机功能过重

## 四、client_gone 结论（截至 2026-07-15）

含义：NewAPI 入站 request context 取消，不是 STREAMING_TIMEOUT。
特征：ch11 + gpt-5.6-sol + max + Codex；IP 高度集中 43.201.250.41(codex-tui)；Nginx 有 499；CPA 对齐多为 200；失败常 ct=0。
不是：Nginx 配置回退、CPA 批量 5xx、全站故障。
完整说法：ch11 上游有效输出不足/变慢 + Codex 侧断连/取消 -> client_gone。
复查：/opt/new-api/docs/ops/RECHECK-client-gone.md

## 五、认真做中转的优先级

1. 升配 2G~4G 或拆服务
2. 只公网 80/443(+ssh)，Docker 绑 127.0.0.1
3. 改弱密码、收紧 sudo
4. 多渠道备份，监控 gone 率
5. 分组/Key/告警

## 六、评分（当时）

功能 7；流式反代 8；稳定性 4；安全 3；可维护 6；对外卖中转 3。

## 七、相关路径

- NewAPI：/opt/new-api
- CPA：/opt/CLIProxyAPI
- Nginx：/etc/nginx/sites-available/newapi 与 cpa
- 本文件别名：RELAY-OVERVIEW.md
