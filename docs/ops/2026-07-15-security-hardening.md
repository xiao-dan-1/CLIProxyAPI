# 运维变更：第二节安全加固（不升配）

- 日期：2026-07-15
- 主机：67.230.187.192

## 做了什么

1. 端口只绑本机 127.0.0.1
   - new-api 3000 / cpa 多端口 / sub2api 8080 / grok 8000 / outlook 8765
   - sub2api: 宿主机 127.0.0.1:8080，容器内 SERVER_HOST=0.0.0.0（Nginx 可进）
   - 公网保留 22/80/443

2. NewAPI 弱密码轮换
   - redis requirepass + postgres root 已改
   - 密钥文件：/opt/new-api/.secrets-rotated.env（root 600）

3. NewAPI 镜像改为 calciumion/new-api:latest

4. sudo 收紧：仅 nginx/docker/install|cp|tee 等，不再 NOPASSWD ALL
   - 备份：/etc/sudoers.d/99-codexdebug-temp.bak.20260715_192805

## 验证

- new/cpa/sub2api/outlook HTTPS 200
- new-api healthy
- 业务端口仅 127.0.0.1

## 备份

各 compose 有 .bak.20260715_192805
