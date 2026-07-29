# Pi Agent Harness 设计决策记录

> 本次设计文档未走烤问流程：用户声明项目背景以仓库代码与文档为准、不另行提问。以下决策均从代码与文档（`packages/*/README.md`、`packages/agent/docs/harness.md`、`packages/coding-agent/docs/*`）中提取并确认，无待确认项。

## D1 · 分层 monorepo：基础库与产品壳分离
- **问题**：LLM 接入、Agent 运行时、终端 UI、CLI 产品四者如何组织？
- **结论**：拆为四个独立发布的 npm 包——`pi-ai`（统一多 provider LLM API）、`pi-agent-core`（Agent 运行时 + Harness + 会话持久化）、`pi-tui`（终端 UI 库）、`pi-coding-agent`（CLI 产品 + SDK），依赖单向自下而上。
- **理由**：每层可独立复用（如 Slack 机器人只用 agent-core，浏览器应用只用 ai），产品策略（内置工具、扩展体系）不污染底层运行时。
- **影响章节**：2.1、第 3 章

## D2 · 扩展优先（extension-first）而非内置功能堆叠
- **问题**：子代理、计划模式、权限门等功能是否内置？
- **结论**：不内置。核心只提供四类扩展机制——Extensions（TypeScript 模块）、Skills（Agent Skills 标准）、Prompt Templates、Themes；功能由用户或第三方 Pi Package 实现。
- **理由**：产品哲学"Adapt pi to your workflows, not the other way around"，避免为所有人的工作流买单、无需 fork 内核。
- **影响章节**：1.1、1.2、3.4

## D3 · 会话即追加日志（append-only log），一库两视图
- **问题**：会话持久化采用什么模型？
- **结论**：会话是 append-only 日志（JSONL 实现即一行一记录）；同一份日志两个视图——Session Entry 构成树（`parentId` 链，支撑分支/导航/压缩），Harness Entry 记录编排事实（operation/tool 开始等），仅用于崩溃恢复，不进树、不进模型上下文。
- **理由**：任意日志前缀都是合法状态，崩溃恢复无需多记录原子写；树结构天然支持 `/tree` 原地分支。
- **影响章节**：3.2

## D4 · 持久化边界先行（durability rule）
- **问题**：运行（run）的崩溃恢复粒度？
- **结论**：副作用发生前先追加 intent 条目（预分配将产生条目的 id），副作用完成后追加结果条目；intent 无结果 = 中断，恢复时按 intent 类型决定重试或放弃。操作（run/compaction/navigation）的 durable outcome 对应日志中一对 `operation_started`…`operation_finished`。
- **理由**：恢复逻辑只读日志即可判断执行进度；provider 流式响应不持久化，中断请求重试而非续传。
- **影响章节**：3.2

## D5 · 上下文只追加不变式与延迟写（deferred writes）
- **问题**：运行中途的写（steer 消息、扩展追加条目、改模型配置）何时生效？
- **结论**：分支上 provider 上下文只能在尾部增长；step 进行中的一切写请求先持久化受理、延迟到下一个 checkpoint 才应用到树尾部。压缩是唯一有意的例外（用全量缓存失效换更短上下文）。
- **理由**：中途插入会破坏 provider KV 缓存前缀，导致 token 成本静默翻倍；同时保证 tool call 与 toolResult 的相邻性。
- **影响章节**：3.2

## D6 · 内建最小权限边界，隔离交给外部
- **问题**：是否内置文件系统/进程/网络权限系统？
- **结论**：不内置。Pi 以启动者权限运行；需要更强边界时由外部容器化/沙箱（Gondolin 微 VM 扩展、Docker、OpenShell）负责。
- **理由**：权限语义因环境而异，内核硬编码会限制嵌入场景；项目信任（project trust）只控制项目本地资源的加载，不做运行时隔离。
- **影响章节**：1.2、3.4

## D7 · 供应链硬化为工程基线
- **问题**：npm 依赖与发布如何管控？
- **结论**：直接外部依赖钉死精确版本；lockfile 为依赖事实源；安装全链路 `--ignore-scripts`；发布包附带 npm-shrinkwrap.json 锁定传递依赖；生命周期脚本依赖需显式 allowlist 评审。
- **理由**：CLI 工具以用户权限执行命令，依赖投毒即 RCE。
- **影响章节**：1.1（背景约束）
