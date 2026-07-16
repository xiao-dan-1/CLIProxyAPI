# 运维变更记录：Nginx 流式反代加固

- 日期：2026-07-15
- 操作者：codexdebug（经 Hermes 远程）
- 相关服务：NewAPI（new.xdauv.xyz）、CLIProxyAPI/CPA（cpa.xdauv.xyz）
- 目的：修复反向代理对流式（SSE /v1/responses）不友好的配置，降低缓冲与空闲超时风险

## 背景

NewAPI 日志大量出现：

```text
stream ended: reason=client_gone end_error="context canceled"
```

说明：
- client_gone 表示下游客户端断开，不是 NewAPI 主动超时标签
- 现网 Nginx 对 newapi/cpa 存在流式短板（固定 Connection: upgrade、未关 buffering、默认 60s 读超时）
- 本次修改是反代加固；不能单独解释全部 18–19s client_gone（仍需观察上游/CPA keepalive/客户端）

## 变更内容

### 1. 新增 map 配置

文件：/etc/nginx/conf.d/connection_upgrade.conf

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

作用：
- 请求带 Upgrade 头（WebSocket）→ 变量值为 upgrade → 发送 Connection: upgrade
- 请求不带 Upgrade 头（普通 HTTP / SSE API）→ 变量值为 close → 发送 Connection: close
- 避免以前“所有请求都固定 Connection: upgrade”

为什么需要它：
- WebSocket 握手必须 Connection: upgrade
- SSE / 普通 API 不需要 upgrade；写死 upgrade 不规范，个别后端/keepalive 可能怪异
- map 根据请求头动态生成正确的 Connection 值

### 2. 加固 NewAPI 站点

文件：/etc/nginx/sites-available/newapi
域名：new.xdauv.xyz → http://127.0.0.1:3000

关键项：
- proxy_set_header Connection $connection_upgrade;
- proxy_buffering off;
- proxy_cache off;
- proxy_request_buffering off;
- proxy_socket_keepalive on;
- proxy_connect_timeout 15s;
- proxy_read_timeout 300s;
- proxy_send_timeout 300s;
- send_timeout 300s;

### 3. 加固 CPA 站点

文件：/etc/nginx/sites-available/cpa
域名：cpa.xdauv.xyz → http://127.0.0.1:8317

与 NewAPI 使用相同的流式参数（仅 proxy_pass 端口不同）。

原因：NewAPI 渠道 CAP-gpt 上游为 https://cpa.xdauv.xyz，只改 newapi 不够，NewAPI→CPA 这一跳也要关缓冲/拉长空闲超时。

## 备份

- /etc/nginx/sites-available/newapi.bak.20260715_132506
- /etc/nginx/sites-available/cpa.bak.20260715_132506

## 生效步骤（已执行）

```bash
nginx -t
systemctl reload nginx
```

结果：
- syntax is ok / test is successful
- nginx active
- https://new.xdauv.xyz/api/status → 200
- https://cpa.xdauv.xyz/ → 200

## 回滚

```bash
cp /etc/nginx/sites-available/newapi.bak.20260715_132506 /etc/nginx/sites-available/newapi
cp /etc/nginx/sites-available/cpa.bak.20260715_132506 /etc/nginx/sites-available/cpa
rm -f /etc/nginx/conf.d/connection_upgrade.conf
nginx -t && systemctl reload nginx
```

## 后续观察

```bash
docker logs new-api --since 30m 2>&1 | grep -c reason=client_gone
docker logs new-api --since 30m 2>&1 | grep reason=client_gone | grep -oE "received=[0-9]+" | sort | uniq -c
```

若 18–19s / received=1 的 client_gone 仍高，继续排查：
1. CPA streaming.keepalive-seconds
2. NewAPI STREAMING_TIMEOUT 是否真正进入容器
3. 渠道 ch11（CAP-gpt）上游静默

## 相关路径

- NewAPI 部署：/opt/new-api
- CPA 部署：/opt/CLIProxyAPI
- Nginx 配置：/etc/nginx/sites-available/{newapi,cpa}
- map：/etc/nginx/conf.d/connection_upgrade.conf
