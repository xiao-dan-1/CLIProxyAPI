# 运维变更记录：CPA streaming keepalive

- 日期：2026-07-15
- 操作者：codexdebug（经 Hermes 远程）
- 服务：CLIProxyAPI / cpa.xdauv.xyz
- 目的：缓解 NewAPI client_gone（上游首包后静默导致下游断开）

## 变更

文件：/opt/CLIProxyAPI/config.yaml

streaming.keepalive-seconds = 15
streaming.bootstrap-retries = 1

说明：
- keepalive-seconds=15：SSE 空闲时每 15 秒发 keep-alive 注释帧
- bootstrap-retries=1：首字节发送前允许 1 次安全重试
- 配置此前已有但缩进不规范；本次规范化缩进并 recreate 容器确保加载

## 操作

- 备份：/opt/CLIProxyAPI/config.yaml.bak.20260715_143615
- docker compose up -d --force-recreate cli-proxy-api
- 镜像：eceasy/cli-proxy-api:latest（v7.2.78）
- 健康检查：8317 与 https://cpa.xdauv.xyz/ 均 200

## 观察

docker logs new-api --since 30m 2>&1 | grep -c reason=client_gone
重点看 ch11 CAP-gpt + gpt-5.6-sol 的 18-19s client_gone 是否下降。
