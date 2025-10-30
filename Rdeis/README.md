# RediOps

> 言传千行代码，语枢万物智能

## 仓库地址

- API 仓库：`git@github.com:YYC-Cube/yyc3-rediops-api.git`

## 文档

- `docs/NAS-DEPLOY.md` — NAS 部署规划与上线方案
- `docs/API-COMPOSE.md` — API 与 Redis 编排（Compose）使用指南

## 一键编排（可选）

- 准备：复制 `config/.env.example` 为 `config/.env` 并按需修改端口/密码
- 开发联调：`docker compose -f config/docker-compose.yml up -d redis-dev api-dev`
- 生产演练：`docker compose -f config/docker-compose.yml up -d redis-prod api-prod`
- 停止：`docker compose -f config/docker-compose.yml stop api-dev api-prod`
- 日志：`docker compose -f config/docker-compose.yml logs -f api-dev`

## 变量说明（Compose 与容器）

- Compose 插值：读取 `Shell 环境` 与 `config/.env`；不读取 `env_file:`。
- 容器内环境：由 `env_file: ../.env.local` 注入给容器运行时。
- 建议：`REDIS_PROD_PASSWORD` 在 `config/.env` 与 `../.env.local` 保持一致。

## 热重载快速指南（api-dev）

- 前提：`api/package.json` 已配置 `scripts.dev` 为 `nodemon index.js`（已完成）。
- 启动：`api-dev` 在 Compose 中使用 `npm run dev`，代码变更将自动重启。
- 查看日志：`docker compose -f config/docker-compose.yml logs -f api-dev`
- 快速验证：`curl http://localhost:${API_DEV_PORT-默认3000}/status` 期待 `redis: ok`。
- 关闭热重载：将 Compose 中 `api-dev` 的 `command` 改为 `node index.js` 或使用 `npm run start`。
- 忽略变更：`api/nodemon.json` 已忽略 `node_modules/**`、`logs/**`、`.git/**`、`*.log`，可按需扩展。
  本仓库用于统一管理 Redis 开发与生产环境的配置与运维脚本，提供一套标准化的目录结构、启动/停止命令、健康检查与安全规范。

> 说明：`api/` 为独立的 Node API 项目，此次规范化不涉及该目录的代码调整。

## 目录结构

- `config/` Redis 配置与 `docker-compose.yml`
  - `redis-base.conf` 通用基础配置（容器内路径：`/etc/redis/redis-base.conf`）
  - `redis-dev.conf` 开发环境配置
  - `redis-prod.conf` 生产环境配置（强密码、禁用高危命令、持久化、内存策略）
  - `docker-compose.yml` 开发/生产容器编排
- `scripts/` 运维脚本（详见 `docs/SCRIPTS.md`）
  - `redis-manager.sh` 统一管理启动/停止/状态/健康检查
  - `check-redis-prod.sh` 部署前安全校验
  - `start-redis-dev.sh` 本地快速启动
  - `start-redis-docker.sh` Docker 启动入口
  - `sync-redis-config.sh` 配置同步与备份
- `logs/` 日志目录（`dev/`、`prod/`）
- `docs/` 文档
  - `REDIS.md` 配置说明
  - `OPS.md` 运维与部署规范
  - `ENV.md` 环境变量规范
  - `SECURITY.md` 安全策略（极简/强校验切换与 ACL 示例）
  - `NAS-DEPLOY.md` NAS 部署规划与上线方案

## 初始化

- 准备目录（如缺失）：
  - `mkdir -p scripts logs/dev logs/prod`
- 初始化环境变量：
  - `cp .env.example .env.local`
  - （可选）`source .env.local` —— 管理脚本已自动加载 `.env.local`，未提供则回退 `.env.example`

## 前置条件

- macOS（本地调试），已安装：Docker / Docker Compose、Bash、`redis-cli`
- 生产容器镜像：`redis:alpine`（入口 `docker-entrypoint.sh`，命令传入配置路径）

## 快速开始

- 开发（Docker）：
  - `bash scripts/redis-manager.sh start --mode docker --env dev`
  - `bash scripts/redis-manager.sh status`
  - `bash scripts/redis-manager.sh health --env dev`
- 生产（Docker）：
  - 严格校验：`bash scripts/check-redis-prod.sh`
  - 本地调试放宽：`ALLOW_WEAK_PROD=1 bash scripts/check-redis-prod.sh`
  - 启动：`bash scripts/redis-manager.sh start --mode docker --env prod`
  - 健康（带认证）：`REDIS_PROD_PASSWORD=redis_yyc3 bash scripts/redis-manager.sh health --env prod`

## 健康检查与端口

- 生产容器：宿主 `6380` → 容器 `6379`
- 开发容器：宿主 `6381` → 容器 `6380`（示例）
- Compose 健康检查：`CMD-SHELL redis-cli -a "$REDIS_PROD_PASSWORD" -p 6379 ping`
- `redis-manager.sh` 在主机与容器两侧执行 PING，并输出内存信息

## 安全规范（生产）

- 默认极简安全（本地推荐）：`protected-mode yes`、`requirepass`、保留 `CONFIG`，不启用 ACL 与命令禁用，`.env.local` 建议 `ALLOW_WEAK_PROD=1`
- 可切换强校验（更严格）：禁用 `FLUSHALL`、`FLUSHDB`，将 `.env.local` 设为 `ALLOW_WEAK_PROD=0` 后执行严格校验脚本：`bash scripts/check-redis-prod.sh`
- 切换指引：修改 `config/redis-prod.conf` 增加/移除 `rename-command` 行并重启容器；详见 `docs/SECURITY.md`

## 持久化与数据目录

- 目录：`dir /data`（容器内路径）
- RDB 文件名：`dbfilename dump-prod.rdb`
- AOF 文件名：`appendfilename "appendonly-prod.aof"`
- 推荐：`stop-writes-on-bgsave-error yes`、`aof-load-truncated yes`

## 内存策略

- 示例：`maxmemory 8gb`（本地环境）
- 策略：`maxmemory-policy volatile-lru`
- 按机器资源与负载调整，并复核健康输出

## 常用运维命令

- 查看日志：`docker logs --tail 200 redis-prod`
- 检查容器：`docker inspect redis-prod`
- 容器内快速内存测试：`redis-server --test-memory 8192`

## 参考文档

- `docs/REDIS.md` 配置细节与模块兼容性
- `docs/OPS.md` 运维流程与故障排查
- `docs/ENV.md` 环境变量规范
- `docs/SCRIPTS.md` 脚本用法详解

## 范围说明

- 本次标准化不涉及 `api/` 目录的代码与文档调整；其依赖与说明见 `api/README.md`。

> 言启象限，语枢智能
