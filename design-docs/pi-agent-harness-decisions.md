# Pi Agent Harness 设计决策记录

> 本文档为逆向设计文档（基于既有代码与官方文档还原系统设计）的过程决策记录。
> 用户明确表示对项目不熟悉、不进行烤问对齐，以下决策均从项目文档/代码推导，或由文档编写者按合理默认敲定。

## D1 · 文档对象与系统边界
- **问题**：设计文档覆盖整个 monorepo 还是仅 coding-agent CLI？
- **结论**：以整个 Pi agent harness monorepo（packages/ai、agent、coding-agent、tui、storage、server、evals）为系统边界，coding-agent 作为核心交付物重点展开。
- **理由**：README 将项目定位为 "Pi agent harness"，各包是分层依赖关系，单独写 CLI 无法解释其运行时与 LLM 抽象的来源。
- **影响章节**：1.2、2.1、第 3 章全部

## D2 · 信息来源优先级
- **问题**：官网 pi.dev 文档与本地代码不一致时以谁为准？
- **结论**：以本地源码与仓内 docs/（packages/coding-agent/docs、各包 README/CHANGELOG）为准，不访问外部网站。
- **理由**：用户要求从本地项目的文档和代码中获取信息；本地快照是唯一可验证的事实来源。
- **影响章节**：全部

## D3 · 文档语言与产物路径
- **问题**：文档语言与存放路径？
- **结论**：中文写作；产物路径 `D:\02-Dev\pi\design-docs\pi-agent-harness-design.md`，决策记录同目录。
- **理由**：用户使用中文交流；产物放项目内便于随代码版本管理。项目仓库为英文，但本文件是给用户个人理解用的衍生文档，不参与上游提交。
- **影响章节**：无（过程决策）

## D4 · 文档定位：现状还原而非改进方案
- **问题**：写"现有系统的设计还原"还是"改进/新建方案"？
- **结论**：还原既有系统的设计（as-is architecture documentation），不提出改进建议。
- **理由**：用户要求"为这个项目生成相应的方案文档"，项目是成熟开源项目，无新建需求背景。
- **影响章节**：第 1 章（背景从项目定位推导）、全文语气

## D5 · Pi 不内置进程内沙盒
- **问题**：安全边界应放在哪一层？
- **结论**：Pi 在进程内不实现权限沙盒；项目信任（trust.json）仅为"输入加载守卫"，真实隔离交给外部容器/VM/策略沙盒。
- **理由**：`docs/security.md` 明确说明半吊子沙盒会被用户误读为安全边界；`docs/containerization.md` 给出三种可选隔离方案。
- **影响章节**：6 安全与隔离声明

## D6 · Agent loop 的唯一性
- **问题**：pi-server、pi-evals、SDK 宿主是否各自实现 agent loop？
- **结论**：全系统只有 `pi-agent-core` 实现 agent loop；`pi-coding-agent` 的 `AgentSession` 是共享业务核心；pi-server 仅做进程监管与协议桥接；pi-evals 仅做 harness 与断言。
- **理由**：子代理调研确认 server/evals 都不含 agent 逻辑，全部复用 coding-agent 的服务装配 API。
- **影响章节**：2.1、3.4、3.6、3.7

## D7 · 会话树与上下文物化视图
- **问题**：会话上下文如何组织和持久化？
- **结论**：会话以 JSONL 树（id/parentId）落盘；上下文是从当前 leaf 回溯的物化视图；compaction/branch_summary 也是树节点。
- **理由**：`session-format.md`、`compaction.md` 与源码 `session-manager.ts`、`compaction.ts` 一致。
- **影响章节**：3.5.1、3.5.2、3.5.3
