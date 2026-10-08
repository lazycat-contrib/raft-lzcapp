# Raft for LazyCat（懒猫微服 LPK）

把 [Raft](https://github.com/botiverse/raft-source) 平台打包为懒猫微服应用（LPK v2），
以 Docker 镜像模式发布到社区商店（喵喵商店）。

> 非官方打包：本仓库由社区维护，与 Botiverse, Inc. 无关。镜像来自本组织自动构建的
> [dockers-x/raft](https://github.com/dockers-x/raft)，上游代码采用
> [FSL-1.1-ALv2](https://github.com/botiverse/raft-source/blob/main/LICENSE)。

## 应用组成

| 服务 | 镜像 | 说明 |
| --- | --- | --- |
| `server` | `czyt/raft-server`（跟踪上游 release 版本） | Raft API / 实时通信 |
| `web` | `czyt/raft-web`（跟踪上游 release 版本） | Web 客户端（nginx 静态服务） |
| `postgres` | `postgres:16-alpine` | 数据库 |
| `redis` | `redis:7-alpine` | 实时通信的跨副本适配（单副本同样可用） |
| `rustfs` | `rustfs/rustfs:latest` | S3 兼容对象存储（附件） |

路由（微服网关）：

- `/` → `web`（前端静态页面）
- `/api`、`/socket.io`、`/daemon`、`/internal`、`/share` → `server`
  （`disable_trim_location: true` 保留完整路径；WebSocket 路径启用
  `fix_websocket_header`）

数据目录：`/lzcapp/var/postgres`、`/lzcapp/var/redis`、`/lzcapp/var/rustfs`。

## 自动发布

`.github/workflows/lazycat.yml` 每 **8 小时**（UTC 00:00 / 08:00 / 16:00）以及
手动触发时运行：

1. 检查 `czyt/raft-server` / `czyt/raft-web` 是否有新的 release 版本镜像（tag
   `vX.Y.Z`，来自上游 release）；
2. 有新版本时更新 `lzc-manifest.yml` 镜像引用与 `package.yml` 版本；
3. 构建 LPK、生成 GitHub Release（`versioned-release-asset`）；
4. 发布到喵喵商店（`stores.private`），已有同版本时自动跳过。

运行机制由 [ca-x/lazycat-github-action](https://github.com/ca-x/lazycat-github-action) 驱动。

### 需要的 Secrets

组织（或本仓库）需要配置：

| Secret | 说明 |
| --- | --- |
| `APPSTORE_URL` | 喵喵商店服务端地址 |
| `APPSTORE_TOKEN` | 发布 Token |
| `APP_ID`（可选） | 商店应用 ID；缺省时按 `stores.private.name` 查找或创建 |
| `PRIVATE_STORE_GROUP_CODES`（可选） | 私有分组码，逗号分隔 |

首次发布时若商店中不存在 `Raft` 应用，Action 会自动创建。

## 手动操作

```bash
# 本地构建 LPK（需要 lzc-cli）
lzc-cli project release -o dist/application.lpk

# 查看包信息
lzc-cli lpk info dist/application.lpk
```

## 维护备注

- `server` 容器启动时通过 `setup_script` 执行数据库迁移
  （`migration-preflight` + `migrate-deploy`）并确保附件 bucket 存在；
  这些步骤是幂等的，容器重启会重新校验。
- 基础镜像（postgres / redis / rustfs）暂未纳入自动版本跟踪，按需手动升级。
- `rustfs` 声明 `user: "0"`（以 root 运行）以写入持久目录。
- 如移动端体验不佳，可在 `package.yml` 中增加 `unsupported_platforms: [ios, android]`。
- web 镜像的 `VITE_API_URL` 为空（同源相对路径），依赖平台网关按路径转发；
  不要给 `/api` 等路由关闭 `disable_trim_location`，否则后端会收到缺失前缀的路径。
