# Iris 用户故事

> 产品演进顺序：I0 会话/澄清 → I1 只读 Skill → I2 低风险命令 → I3 高风险确认/恢复 → I4 报告运营。每个故事必须记录模型、Prompt、Skill、API 和授权版本，并验证拒绝路径。

| ID | 用户故事 | 验收标准 | 来源 | 状态 |
|---|---|---|---|---|
| IRIS-001 | 作为用户，我希望在 Sage Chat 创建和恢复会话。 | Conversation/Run/Approval/Invocation/EventLog 全部租户隔离；SSE 可按游标恢复；断线不重复 Skill。 | IRS-007 | `draft` |
| IRIS-002 | 作为用户，我希望 Iris 澄清缺失信息。 | 账号/对象/内容/权限不足时提问或拒绝；不猜测执行参数。 | IRS-001 | `draft` |
| IRIS-003 | 作为用户，我希望看到结构化执行计划。 | 计划引用已注册能力；展示目标、影响、风险和预计步骤；版本固定。 | IRS-002 | `draft` |
| IRIS-004 | 作为安全负责人，我希望 Skill 调用受统一授权。 | 复用数据所有者判定端口；校验角色、Creator/Owner/User/有效 Own、权限来源及每个关联实体；任意 MCP/shell 不可见。 | IRS-003/008、FND-018 | `draft` |
| IRIS-005 | 作为用户，我希望确认高风险操作。 | PlanApproval 绑定 Run、计划/Skill/契约版本、参数/目标摘要、确认人和过期时间并原子单次消费；外部副作用另创建并关联 Stem `confirmation_id`，由 Stem 在入队单次消费；换参、换版本、过期或重验权失败联动作废两者，直接调用 Stem 也不能绕过。 | IRS-004/009、CFG-006 | `draft` |
| IRIS-006 | 作为用户，我希望委托 Mint/Tea/Stem 创建任务。 | 调用前持久化幂等键；未知结果先对账不重放；返回业务关联 ID；Iris 不直连终端。 | IRS-004 | `draft` |
| IRIS-007 | 作为用户，我希望持续查看进度和最终报告。 | 事件顺序可恢复；状态来自业务服务；失败包含恢复动作和授权证据链接。 | IRS-004/005/007 | `draft` |
| IRIS-008 | 作为管理员，我希望审计智能体决策。 | 保存模型/提示/Skill/契约版本、计划、确认与调用摘要；敏感内容脱敏。 | IRS-006 | `draft` |
| IRIS-009 | 作为发布负责人，我希望在 Test 验证 Agent E2E。 | 覆盖澄清、越权、PlanApproval→Stem Confirmation 关联/单次消费/联动作废/直接 API 绕过拒绝、下游成功但 checkpoint 前崩溃且只产生一次副作用、Skill 失败、SSE 断流恢复和任务回执。 | IRS-010 | `draft` |

## 产品化细化故事

| ID | 用户故事 | 验收标准 | 来源 | 状态 |
|---|---|---|---|---|
| IRIS-010 | 作为用户，我希望查看 Iris 为什么需要澄清。 | 每个问题关联缺失/冲突 slot；保留已确认输入；取消澄清不创建 Command。 | IRS-001 | `draft` |
| IRIS-011 | 作为管理员，我希望发布版本化 Skill。 | descriptor 含 Schema/风险/授权/幂等/对账；契约测试通过后启用；下架阻止新 Run。 | IRS-003/008 | `draft` |
| IRIS-012 | 作为用户，我希望查询联系人、内容和任务而不产生副作用。 | 只读 Skill 逐实体授权；分页/引用可追溯；模型无法将查询升级为命令。 | IRS-003/005 | `draft` |
| IRIS-013 | 作为用户，我希望计划只包含平台可执行能力。 | 每步编译为已发布 Action/Workflow/Task Command；不支持的步骤明确拒绝/建议；不在 LangGraph 隐藏流程。 | IRS-002 | `draft` |
| IRIS-014 | 作为安全负责人，我希望阻止提示注入操纵 Skill。 | 业务内容标不可信；工具参数 Schema+授权+确认；攻击样本不能调用未批准 Skill。 | IRS-008 | `draft` |
| IRIS-015 | 作为平台，我希望一个 Run 只有一个执行 Worker。 | Run lease/CAS；并发恢复仅一方成功；失主可接管；EventLog 无分叉。 | IRS-007 | `draft` |
| IRIS-016 | 作为用户，我希望报告明确区分事实和模型总结。 | Task/Proof 状态来自服务引用；模型总结有来源；未知/过期数据标新鲜度。 | IRS-005 | `draft` |
| IRIS-017 | 作为运维人员，我希望控制模型预算和降级。 | tenant/model/token/time budget；超限转澄清/人工；不因降级绕过授权。 | IRS-006 | `draft` |
| IRIS-018 | 作为发布负责人，我希望回放固定 Run。 | 固定模型/Prompt/Skill/契约和输入摘要；副作用 Skill 使用 dry-run/对账，不在回放中再次执行。 | IRS-006/010 | `draft` |
