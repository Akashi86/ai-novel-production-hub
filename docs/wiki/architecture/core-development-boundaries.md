# 二开核心边界

## 背景

本项目的二次开发重点正在从单一 AI 写作助手，扩展为包含 MCP、自动发布、自动导演、Prompt Registry、任务中心和章节生产链的小说生产平台。为了避免新能力绕开现有主链，需要把边界规则放在长期 wiki 中，而不是堆在 `AGENTS.md` 入口文件里。

本文记录自动导演、Prompt Registry、任务中心和章节生产链的核心边界，供新增能力、重构和排障时引用。

## 决策

`AGENTS.md` 只保留高优先级协作规则和入口指引。详细模块边界放入 `docs/wiki/architecture/`，并在涉及跨模块行为时与工作流 wiki、Prompt wiki 和模块 README 互相引用。

二开时必须优先保持四条边界：

- 自动导演是控制面，不是第二套正文生成系统。
- Prompt Registry 是产品级 AI 能力合同，不在业务 service 中散落新 prompt。
- 任务中心是后台任务投影和操作面，不承载具体业务执行。
- 章节生产链是正文生成、验收、修复、保存和时间线封口的唯一主链。

## 自动导演边界

- 自动导演负责把小说从灵感推进到整本可写状态，管理阶段流转、审批点、恢复、重试、接管和自动执行范围。
- 自动导演不能变成第二套章节 writer 或修文系统。导演驱动的章节正文生产必须委托统一章节生产链。
- `continue`、`resume`、`retry`、`approve`、`cancel`、`takeover`、`skip_quality_repair` 等用户动作应先变成命令，不能在 HTTP route 中直接执行长 LLM 链路。
- 新的导演队列任务应使用 `DirectorRunCommand` 路径，通过 `DirectorCommandService`、`DirectorTaskQueue`、`DirectorWorker` 和 `DirectorCommandExecutor` 推进。
- `DirectorRun`、`DirectorStepRun`、`DirectorEvent` 和 `DirectorArtifact` 是 runtime facts 和投影来源。旧 runtime command 表只保留兼容读取，不应作为新后台队列写入点。

相关入口：

- `server/src/routes/novelDirector.ts`
- `server/src/services/novel/director/directorSubsystem.ts`
- `server/src/services/novel/director/README.md`

## Prompt Registry 边界

- `server/src/prompting/` 是产品级 prompt 的治理边界。新增产品 prompt 必须是 `PromptAsset`，放在 `server/src/prompting/prompts/<family>/`，并注册到 `server/src/prompting/registry.ts`。
- 新增 AI 能力时，应扩展 prompt asset、输出 schema、上下文策略、工具合同、repair policy 或 semantic retry policy。
- 不要用关键词路由、本地 JSON 修复、临时 prompt 字符串或业务 service 内的 ad hoc 分支来补产品行为。
- 结构化输出使用统一 prompt runner 和 schema 校验。确定性代码可以在结构化输出后归一化别名、做安全校验，但不能替代 AI 对规划、意图、质量或下一步动作的判断。
- 修改旧的未注册 prompt 路径时，默认先迁入 registry，再扩展能力。

相关入口：

- `server/src/prompting/README.md`
- `server/src/prompting/registry.ts`
- `server/src/prompting/core/promptRunner.ts`

## 任务中心边界

- 任务中心是后台任务的统一投影和操作面，负责列表、筛选、汇总、详情、重试、取消、归档和恢复入口。
- 任务中心不是业务执行器。具体执行逻辑必须留在任务所属模块中。
- 每种任务类型应通过 adapter 把自身持久化模型映射成统一任务 summary/detail shape。
- 新增长任务类型时，先定义或复用 durable task model，再补 task adapter、list/detail/retry/cancel/archive 语义，最后接入任务中心 UI。
- 恢复读取只能说明可恢复位置和原因。普通轮询、预览、详情读取不能顺手写入恢复事件。

相关入口：

- `server/src/routes/tasks.ts`
- `server/src/services/task/TaskCenterService.ts`
- `server/src/services/task/adapters/`
- `client/src/pages/tasks/TaskCenterPage.tsx`

## 章节生产链边界

- 正文生成与正文修复只有一条业务执行链。手动单章生成、批量 pipeline、Creative Hub 委托和自动导演执行都必须汇入同一套 production/runtime 路径。
- `NovelProductionOrchestrator` 负责 stage 级生产流。`ChapterExecutionStageRunner` 将单章流式执行委托给 `ChapterRuntimeCoordinator`，将批量范围委托给 pipeline job API。
- 不要为特定 route、导演分支、UI 入口或 MCP 工具单独新增 writer、patch repair、heavy repair、full rewrite 或保存正文实现。
- 章节热路径为：轻量可写性检查、整章生成、结构化接收闸门、时间线检测和封口、必要时有限修复、保存正文、stable/degraded checkpoint，然后才能进入下一章。
- 后续章节执行前，上一章必须满足 `final_content -> timeline_finalization -> next_chapter`。
- Timeline finalization 属于 `ChapterTimelineFinalizationService`。导演、route、repair helper、MCP 工具和前端投影都不能复制这套写入逻辑。
- 资产回灌、角色资源账本、RAG 索引和状态快照只能消费稳定章节正文或显式 degraded checkpoint，不应把不稳定草稿拉回热路径。

相关入口：

- `server/src/services/novel/production/NovelProductionOrchestrator.ts`
- `server/src/services/novel/production/ChapterExecutionStageRunner.ts`
- `server/src/services/novel/runtime/ChapterRuntimeCoordinator.ts`
- `server/src/modules/timeline/`

## MCP 与自动发布扩展边界

- MCP Server 应优先作为工具适配层，调用现有 HTTP API 或模块门面，不直接读写数据库。
- MCP 工具不能绕过 Prompt Registry、章节生产链、任务中心投影或自动导演命令队列。
- 长任务型 MCP 工具应返回 task id、command id 或 projection，不同步等待整条生产链完成。
- 自动发布能力应作为独立发布/平台适配模块接入，不能混入章节 writer 或导演 runtime。
- 发布前检查、平台凭证、定时发布状态和人工确认策略应有独立边界；不得把平台 Cookie、token 或会话内容写入日志、仓库或长期记忆库。
- 发布任务进入任务中心时，应以 adapter 暴露状态；任务中心只展示和操作，不承担浏览器自动化或平台 API 执行。

## 边界判断清单

- 新 AI 判断或生成能力：新增或迁移 `PromptAsset`。
- 新导演行为：新增 command/runtime/projection 行为，再把生产工作委托出去。
- 新章节写作或修复能力：扩展统一章节 runtime 或 production stage，不新增入口专属分支。
- 新后台任务：新增 durable task state 和 task-center adapter，执行逻辑留在所属模块。
- 新 UI 状态：读取 projection/detail API，不在前端从无关表拼 runtime truth。
- 新 MCP 工具：调用现有 API 或门面，不重写业务链。
- 新自动发布能力：作为发布适配模块和任务类型接入，不污染正文生产链。
- 跨边界变更：如果形成稳定架构知识，更新相关 wiki 或模块 README。

## 失败模式

- 自动导演中直接生成或修复正文，导致手动单章、批量执行和导演执行出现三套行为。
- MCP 工具为了方便直接查库或调用 LLM，绕过项目已有 Prompt 和 runtime 规则。
- 任务中心 adapter 里混入业务执行，导致列表或详情读取产生副作用。
- 发布流程和正文生成耦合，平台失败反向污染章节生产状态。
- 章节进入下一章前缺少 timeline finalization，后续上下文、hook 和事件承接错乱。

## 相关文档

- `docs/wiki/architecture/module-boundaries.md`
- `docs/wiki/workflows/auto-director-runtime.md`
- `docs/wiki/workflows/chapter-production-chain.md`
- `docs/wiki/prompts/prompt-registry-and-structured-output.md`
- `server/src/prompting/README.md`
- `server/src/services/novel/director/README.md`
