# Iris 架构

> Iris 是 Maia 的受控智能体服务。自然语言由模型理解，业务副作用只能通过已注册 Skill 和服务 API 发生。

## 1. 定位与信任边界

Iris 运行 DeepAgents/LangGraph，负责会话、澄清、计划、人工确认、Skill 编排、流式进度与报告。它不直接访问业务数据库、Celt、Shell、文件系统或任意 MCP；模型不持有服务/账号凭据，也不是权限裁决者。

## 2. 核心模型

AgentConversation/AgentMessage 只保存智能体交互，不与 Mud 的业务 Conversation/Message 同名或双写；引用业务消息时只保存授权引用与脱敏摘要。AgentConversation、Run、Plan、PlanApproval、Invocation、EventLog 全部显式保存 `tenant_id`、creator/user、唯一 owner 与版本；任何按 ID/cursor/token 的读取、恢复或确认都先校验当前 principal 和有效 Own/User，token 本身不授予访问权。AgentRun 固定模型、提示、Skill 与契约版本；Plan 是待提交的 Action/Workflow/Task Command 列表，不承担分支、失败、子流程或执行状态控制。PlanApproval 只表达用户对会话计划的同意，绑定 Run、计划/Skill/契约版本、规范化参数/目标摘要、确认人、一次性 token 和过期时间，不授予权限或替代 Stem Confirmation；SkillInvocation 保存稳定幂等键、调用状态、授权来源、后端关联 ID 和输出摘要。

## 3. 运行结构

Sage Chat 通过 HTTPS + SSE 调用 Iris API；SSE 是服务端事件的唯一流协议，写命令始终用 HTTPS。Runtime 将 Message/Content/Skill 输出标记为不可信数据，与系统指令隔离，先最小化/脱敏并以 Schema 校验模型输出，再由 Planner 生成结构化 Command。Policy Gateway 不复制业务授权规则：它调用数据所有者的版本化授权/查询端口，校验 tenant、角色、Creator/直接 Owner/User/有效 Own、权限来源以及每个关联实体。Account/Contacts/Content/Actions 查询 Skill 调用 Mud API；业务 Skill 调用 Mint/Tea/Stem。

高风险计划中断等待 PlanApproval；消费其 token、创建 SkillInvocation 与记录待调用状态原子完成。若计划包含外部副作用，Iris 以同一版本/参数/目标摘要请求并只保存 Stem `confirmation_id`，Stem 是 Confirmation 的唯一签发者和入队消费者；换参、换版本、过期或重新授权失败同时作废 PlanApproval 与 Confirmation。SkillInvocation 遵循 `prepared → calling → succeeded/failed/outcome_unknown`；调用前持久化稳定 idempotency key，下游据此去重，崩溃恢复时先查询结果/对账，不无条件重放副作用。最终执行状态以业务服务为准。

LangGraph checkpoint 支持断流恢复和人工中断。每个 Run 只有一个持租约 Worker；权威 EventLog 使用 Run 内单调序号和终态，客户端游标在保留窗口续传，过期后以快照重建并重新授权。通用子智能体、代码执行和动态 MCP 默认关闭，未来启用需独立安全故事。模型供应商的数据保留、地域和训练使用必须由租户策略允许；提示注入检测不能替代 Skill 参数、权限和确认校验。

## 4. 数据、部署与观测

Iris 私有保存 AgentConversation/AgentRun/checkpoint/approval/invocation/event log，不复制公共业务数据。报告保存 Task/Todo/Execution/Proof 授权引用、失败分类和恢复动作，不复制执行事实。Python 服务以 OCI/Helm 部署，模型 Secret 由环境注入；API 与运行 Worker可独立伸缩。审计记录模型/提示/Skill/API 版本，指标覆盖首事件延迟、澄清率、拒绝率、确认等待、outcome_unknown、恢复次数和 token 成本。

### 4.1 业务模块

| 模块 | 职责 | 扩展点 |
|---|---|---|
| Conversation | Agent 会话、消息、附件引用和租户权限 | channel adapter，不复制业务 Conversation |
| Intent & Clarification | 意图、槽位、歧义和澄清状态 | versioned intent schema |
| Planner | 将意图编译成受控 Command DAG | planner strategy；不执行流程 |
| Skill Registry | 查询/命令 Skill descriptor、Schema、风险和授权端口 | 显式发布，不动态发现任意工具 |
| Policy & Approval | 数据授权、提示注入隔离、确认和预算 | owner service policy adapters |
| Run Runtime | checkpoint、单 Run lease、Invocation 状态与对账 | LangGraph storage/worker |
| Report | 组合授权的 Task/Reply/Proof 引用 | report adapter，不复制事实 |

### 4.2 核心流程

`message accepted → intent extracted → missing slots clarified → authorized facts resolved → plan compiled → risk/impact preview → approval if required → invocation prepared → downstream idempotent command → progress projection → final report`。任何阶段都可产生明确拒绝/恢复动作；LLM 文本不能直接改变 Run 或业务状态。

SkillDescriptor 固定 `name/version/input/output/risk/required_role/authorization_targets/idempotency_mode/timeout/result_query`。查询 Skill 只读，命令 Skill 必须提供稳定幂等键和结果查询；缺少对账接口的外部副作用 Skill 不得发布。

### 4.3 非功能和产品演进

- 安全：不可信业务内容与 system prompt 分区；输出 Schema 校验；工具参数再次授权；模型供应商保留/地域策略租户化。
- 高可用：Run lease 防双 Worker；EventLog 可恢复；模型/下游超时进入可对账状态，不重复副作用。
- 性能：首个确认事件与模型生成解耦；长报告分页/流式；会话上下文按 token budget 摘要且保留来源引用。
- 可维护：Planner、Policy、Skill、Model Provider 均为窄端口；Prompt/Skill/模型版本可复现。

路线：I0 Chat/澄清 → I1 只读查询 Skills → I2 低风险 Task Command → I3 高风险确认/崩溃对账 → I4 报告与运营 → I5 经安全评审的领域子 Agent；通用 Shell/文件/MCP 不进入默认路线。

## 5. 扩散与迁移

首个 Sprint 前固定 Sage Chat SSE、Mud 查询/授权端口、Mint/Tea/Stem Command 与幂等契约，并更新总体架构图。`maia-iris-v0.1.0` 使用 Alembic 将旧对话/Agent 实现映射为 AgentConversation/Run/Invocation，无法证明关联的数据只读归档，并切换 Sage client、Skill/确认协议和 checkpoint；Test 验证澄清、Stem Confirmation、崩溃对账、断流恢复和报告。`maia-iris-v0.2.0` 删除旧 API、旧 tool adapter、旧 checkpoint 和旧确认入口，不长期兼容双模型。
