# CPA 安全小改：清测试 api-key + 检查 management secret

- 日期：2026-07-15
- 服务：CLIProxyAPI / cpa.xdauv.xyz

## 做了什么

1. 删除弱调用密钥 sk-testtest（len=11，含 test）
2. 保留其余 api-keys（现剩 1 个正式 key）
3. 检查 remote-management：
   - allow-remote: true（用户需要远程管理，保留）
   - secret-key: bcrypt 哈希（已哈希，非明文弱口令）
4. 备份：config.yaml.bak.api-key-clean.*
5. 已 force-recreate cli-proxy-api

## 验证

- cpa local/nginx 200
- config 中无 sk-testtest
- 启动日志：management routes registered after secret key configuration

## 未改

- allow-remote 仍为 true
- streaming keepalive 15 未动
- 号池 auths 未动
