# CPA 主流配置学习笔记（网上/官方对照）

- 日期：2026-07-16
- 来源：router-for-me/CLIProxyAPI README、config.example.yaml、help.router-for.me、生态项目与 Issues
- 用途：下次研究对照；非自动改配置清单

## 大佬常见架构

客户端/NewAPI → Nginx HTTPS → CLIProxyAPI:8317 → auths 号池 → 上游模型

本机桌面（Tray）与服务器 Docker 中转是两套路；本机以 127.0.0.1 为主。

## 配置要点

1. host/port：服务器 Docker 端口绑 127.0.0.1，外网只走 443
2. api-keys：强随机，禁测试钥
3. remote-management：默认 allow-remote false；远程要 true + 强 secret（bcrypt）
4. auth-dir：号池目录，永不进 git
5. request-retry / routing / quota-exceeded：多账号标配
6. streaming.keepalive-seconds + bootstrap-retries：中转长流式常开
7. logging-to-file + logs-max-total-size-mb：小盘机建议限制
8. usage：v6.10+ 内置弱，外挂 Usage Keeper / CPA-Manager-Plus
9. proxy-url：需要出海再设
10. 镜像固定 tag，少用 pull always
11. Nginx：buffering off、长超时、client_max_body_size 够大

## 与本机现网

已对齐：端口 127.0.0.1、清测试 key、keepalive 15、retry 3、Nginx 流式/100m、config/auths 不进仓
可改进：pull_policy always、号池压 1G、用量外挂、日志体积
管理 allow-remote true：用户需要远程管，保留 + 强 secret 合理

## 入口

- https://help.router-for.me/cn/
- https://github.com/router-for-me/CLIProxyAPI
- config.example.yaml
