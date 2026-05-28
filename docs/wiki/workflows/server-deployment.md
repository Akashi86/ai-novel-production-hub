# 服务器部署工作流

## 背景

本项目计划部署到云服务器，并继续扩展 MCP、自动发布、任务中心和长篇小说生产链。服务器部署需要避免两类常见问题：

- 服务器网络访问 GitHub 不稳定，导致拉取、依赖下载或大文件获取失败。
- 服务器目录同时承担运行态数据和临时开发职责，造成 `git status` 长期污染，甚至产生本机、GitHub、服务器三方分叉。

因此服务器应被视为部署工作树，而不是临时代码修改来源。

## 决策

部署默认采用“本机提交并 push，服务器只 fast-forward 或接收已确认构建包”的策略。

标准路径：

```text
本机改代码
-> 本机提交并 push
-> 服务器部署前三边检查
-> 服务器 git pull --ff-only
-> 构建 / 迁移 / 重启
-> 定向验证
```

服务器访问 GitHub 不稳定时，回退到：

```text
本机 git archive
-> scp 到服务器 /tmp
-> 服务器解压到影子目录
-> rsync / compose 验证
-> 通过后切换正式目录
```

## 当前规则

- 部署前先确认三边状态：本机 `HEAD`、GitHub `origin/main`、服务器 `HEAD/origin/main/git status`。
- 服务器只作为部署工作树，不在服务器上随手 `git commit`、`git am` 或生成独立提交。
- 如果服务器已有临时改动，先备份 `git status`、`git diff` 和 untracked 清单，再决定保留、丢弃或移植到本机提交。
- 大版本切换优先使用影子目录验证，例如 `/opt/ai-novel-production-hub-next-<sha>`，验证通过后再切换正式目录。
- 运行态文件不得提交到仓库。`.env`、数据库文件、对象存储、Qdrant 数据、上传产物、日志和本地缓存应放到部署目录外，或写入服务器本地 `.git/info/exclude`。
- 生产环境优先使用 Docker Compose，服务拆为 `web`、`api`、`postgres`、`qdrant` 和反向代理。
- `Dockerfile.api` 按 PostgreSQL 生产模式构建，生产部署必须显式配置 `DATABASE_URL`，并执行 PostgreSQL migration。
- 如果暂时不启用知识库/RAG，可以设置 `RAG_ENABLED=false`；启用 RAG 时需要部署或配置 Qdrant。
- 不读取、不输出 `.env`、API key、Cookie、平台 token 或其他凭证。
- 服务器验证只看关键状态和短日志，默认不 dump 大段日志。

## SSH 与传输规则

- 传文件前先验证 `user + key + host` 是否能登录，不要直接开始 `scp`。
- 如果 `scp` 小文件卡住，优先排查认证组合、known_hosts 交互、DNS、代理和登录用户，不要先归因于带宽。
- 推荐先上传到服务器 `/tmp`，再由服务器上的部署命令移动到目标目录并修正权限。
- `root` 可用于部署、安装和权限修正；运行服务应使用专门的非 root 用户或容器用户。

示例登录验证：

```powershell
C:\Windows\System32\OpenSSH\ssh.exe -i $HOME\.ssh\weibo_ops_server root@<server-ip> "hostname; id"
```

示例上传：

```powershell
git archive --format=tar.gz -o $env:TEMP\ai-novel-production-hub.tgz HEAD
C:\Windows\System32\OpenSSH\scp.exe -i $HOME\.ssh\weibo_ops_server $env:TEMP\ai-novel-production-hub.tgz root@<server-ip>:/tmp/ai-novel-production-hub.tgz
```

## 首次部署建议

首次部署不直接覆盖正式目录，先使用影子目录：

```text
/opt/ai-novel-production-hub-next-<sha>
```

建议步骤：

1. 在本机确认 `main` 已 push 到 GitHub。
2. SSH 验证服务器登录。
3. 检查服务器 Docker、Compose、Git、反向代理和磁盘空间。
4. 检查服务器 GitHub 连通性：`git ls-remote origin main`。
5. 如果连通稳定，服务器 `git clone` 或 `git pull --ff-only`。
6. 如果连通不稳定，本机 `git archive` 后通过 `scp` 上传。
7. 配置生产 `.env`，不要提交。
8. 启动 `postgres`、`qdrant`、`api`、`web`。
9. 执行 `pnpm --filter @ai-novel/server prisma:deploy` 或容器内等价 migration 命令。
10. 验证 `/api/health`、Web 首页、模型配置页和一个非破坏性接口。
11. 验证通过后再把影子目录切到正式服务目录。

## 推荐验证

最小验证优先，不默认跑全量测试：

- `git status --short --branch`
- `docker compose ps`
- `docker compose logs --tail=80 api`
- `curl http://127.0.0.1:3000/api/health`
- `curl http://127.0.0.1:<web-port>/`

如果本次改动涉及数据库 schema、任务恢复、自动导演、章节生产链、Prompt schema 或 RAG，再补对应定向测试或 smoke check。

## 失败模式

- 服务器直接提交代码，导致本机、GitHub、服务器出现同内容不同 hash 的分叉。
- `.env`、数据库、Qdrant 数据或上传文件放在 Git 工作树中，长期污染 `git status`。
- 没有三边检查就 `reset --hard`，覆盖了服务器独有改动。
- 生产 PostgreSQL 没跑 migration，API 容器启动后运行时表结构不匹配。
- `scp` 卡住时盲目重试大包，实际是 SSH 用户或 key 组合错误。

## 相关模块

- `Dockerfile.api`
- `Dockerfile.web`
- `infra/docker-compose.qdrant.yml`
- `infra/nginx/ai-novel-web.conf`
- `server/prisma.config.ts`
- `server/src/config/database.ts`

## 相关文档

- `README.md`
- `docs/wiki/architecture/core-development-boundaries.md`
- `docs/wiki/architecture/module-boundaries.md`
