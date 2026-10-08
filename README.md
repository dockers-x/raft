# raft-docker — automated images for [botiverse/raft-source](https://github.com/botiverse/raft-source)

> **非官方构建 / Unofficial.** This repository is maintained by the community
> (dockers-x). It is not affiliated with, endorsed by, or supported by
> Botiverse, Inc. The images are built from the source-available
> `botiverse/raft-source` repository using the production Dockerfiles that
> ship with it.

每天检查上游更新，为 Raft 平台的官方 Dockerfile 构建多架构镜像并推送到
Docker Hub 和 GHCR。

## 镜像

| 镜像 | 来源 | 说明 |
| --- | --- | --- |
| `raft-server` | `packages/server/Dockerfile` | API 服务、实时通信、持久化（端口 3001） |
| `raft-web` | `packages/web/Dockerfile` | Web 前端（Vite 静态构建，nginx 托管，端口 80） |
| `raft-trace-upload` | `packages/trace-upload-worker/Dockerfile` | trace 上传 worker（端口 3000） |

支持 `linux/amd64` 与 `linux/arm64`。镜像标签：

| 标签 | 说明 |
| --- | --- |
| `latest` | 最新一次成功构建 |
| `v1.21.2` | 上游应用版本（release 版号） |
| `v1.21.2-source.1` | 上游完整 release 标签（含源码快照序号） |
| `YYYYMMDD` | 按构建日期 |
| `sha-xxxxxxx` | 按上游源码 commit |

版本标签直接对应上游 release：`raft-source` 每发布一个版本就导出一个快照
commit 并打上 `vX.Y.Z-source.N` 标签，镜像标签与之逐字对应。上游 commit
没有精确 tag 时，只保留 `latest` / 日期 / `sha-` 标签。

```bash
# Docker Hub
docker pull czyt/raft-server:v1.21.2
docker pull czyt/raft-server:latest
docker pull czyt/raft-web:latest
docker pull czyt/raft-trace-upload:latest

# GitHub Container Registry
docker pull ghcr.io/dockers-x/raft-server:latest
docker pull ghcr.io/dockers-x/raft-web:latest
docker pull ghcr.io/dockers-x/raft-trace-upload:latest
```

## 自动构建机制

- **每日检查**：每天 UTC 00:00 检查 `botiverse/raft-source` 最新 commit。
- **智能跳过**：三个镜像都已带该 commit 的 `sha-` 标签时跳过构建。
- **多架构**：`linux/amd64` + `linux/arm64`（一次构建同时推送）。
- **手动触发**：Actions → *Build and Push Docker Images* → Run workflow：
  - `push_image`：是否推送到 registry（默认 true）
  - `force_build`：强制重新构建（默认 false）
  - `vite_api_url`：烘焙进 web 镜像的 API 地址（默认空）

需要的仓库 secrets：`DOCKERHUB_USERNAME`、`DOCKERHUB_TOKEN`
（访问 Docker Hub → Account Settings → Security → New Access Token，
权限选 Read & Write）。

## 用 Docker Compose 自部署（示例）

仓库根目录带一份社区维护的 [docker-compose.yml](docker-compose.yml)：

```
浏览器 ──► caddy :8080 ──► web（静态前端）
                    └────► server :3001 ──► postgres
                                        ├──► redis
                                        └──► rustfs（S3 兼容对象存储）
```

```bash
git clone https://github.com/dockers-x/raft
cd raft
cp .env.example .env
# 编辑 .env：至少设置 JWT_SECRET（openssl rand -hex 32）与 POSTGRES_PASSWORD
docker compose up -d
# 打开 http://localhost:8080
```

首次启动会先运行一次性任务：`migrate`（应用数据库 schema）和
`rustfs-bucket-init`（创建附件 bucket），然后才启动 `server`。

注意事项：

- **web 镜像与 API 地址**：预构建的 `raft-web` 镜像在构建时 `VITE_API_URL`
  为空，前端使用同源相对路径（`/api`、`/socket.io`、`/daemon`）。示例中的
  caddy 会把这三个前缀反代到 `server`。如果你想前后端分域部署，需要自行构建
  web 镜像并传入 `VITE_API_URL=https://api.example.com`。
- **SERVER_URL**：daemon / computer（跑在用户机器上的组件）用这个地址回连
  你的服务器，请设置为外部可达地址，而不是容器内地址。
- **外部依赖**：`server` 需要 Postgres、Redis 和 S3 兼容存储
  （示例里是 rustfs，也可换成 MinIO / Cloudflare R2 / AWS S3，只需改 `S3_*`
  变量并保证 bucket 存在）。RisingWave 相关功能是可选的，上游未包含在
  此示例中。
- 以上编排是社区示例而非上游官方部署方式；生产环境请自行配置 TLS、备份和
  密钥管理。

## 许可与商标

上游代码采用
[FSL-1.1-ALv2](https://github.com/botiverse/raft-source/blob/main/LICENSE)
（Functional Source License）：允许阅读、运行、修改及**非竞争性**再分发，
再分发需附带许可链接并保留版权声明。本仓库分发的镜像包含上游软件及其许可；
"Raft" 名称与 logo 是 Botiverse, Inc. 的商标，不在许可范围内。

本仓库自身的 workflow 与编排文件可按 MIT 使用。
