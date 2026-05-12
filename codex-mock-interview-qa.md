# Codex / Agent harness 面经（自练 QA）

> 对照过 codex-rs 行为，背的时候抓主干，别死背名词。

---

## 1. Harness 几块？

**Q：核心子系统怎么分？**  
A：四层够用：**入口**（提交/收事件）→ **交通**（排队+单线程消费）→ **状态**（会话+服务箱）→ **执行**（一轮里模型↔工具循环）。  
别和「对外事件流」混：事件多是观测，不是主控制流。

---

## 2. 会话 vs 回合任务

**Q：区别？**  
A：**会话**活得长，管整条线程的状态和一堆长期服务。**回合任务**包这一轮活，带取消，跑完就完。  
会话里通常**同时最多一档主回合**在跑。

---

## 3. 谁消费 Op / Submission？

**Q：UI 会不会从事件通道拉 Op？**  
A：**不会。** UI 只 **send 提交**。  
**recv、按类型拆开处理**的是会话启动时挂起来的那条**后台单消费者循环**（实现上是模块里一个循环函数 + 任务，不是「一个叫 Consumer 的类」）。

---

## 4. 中断

**Q：用户点中断怎么走？**  
A：提交「中断」→ 会话 **abort 当前回合** → **取消令牌**往下传，停流式、停工具等待。  
**拒绝单次工具**和 **Abort 整轮**不是一回事。

**Q：历史里一定有「user interrupted」吗？**  
A：**不一定。** 会有「回合中止」类信号给客户端；是否再往模型可读历史里钉「中断便签」看配置，形态也不止一种。

---

## 5. 压缩

**Q：自动压缩是不是再排队一条压缩指令？**  
A：**多数不是。** 自动压缩嵌在**同一轮的执行循环**里：超阈值就压，压完继续问模型。  
**手动压缩**才更像单独一条动作进提交流。

---

## 6. 沙箱 / 多会话

**Q：一条对话线程有自己的沙箱吗？**  
A：**有「自己的策略+执行上下文」**，不等于每条线程一个虚拟机。底下还是进程+限制手段。

**Q：不同会话跑命令是不同进程吗？**  
A：**执行层一般是不同子进程**；多个会话可并行，各自记账。

---

## 7. 工具 + 审批

**Q：模型要跑命令，中间卡什么？**  
A：**校验** → **策略（要不要人审、是否直接禁止）** → 可能要 **人审挂起** → **沙箱里真跑** → **结果回灌**。  
**拒绝**= 工具失败结果回灌，不是悄悄跳过。

---

## 8. 落盘 vs 内存（第 5 题收口）

**Q：什么进卷宗？**  
A：按时间追加的证据链：用户/模型材料、工具调用与结果、关键事件、**上下文快照**（每轮基线：目录、模型、策略要点）、用量快照等（记多记少可配）。

**Q：什么算运行时？**  
A：连接、子进程现场、审批挂起、流式拼接、当前主回合占位——重启重建。

**Q：重连最怕啥？**  
A：**卷宗、内存历史、执行现场**对不上；修法靠 **先落稳再发关键信号** + 副作用路径想 **幂等**。

---

## 9. 类图脑补（背关系不背类名）

**Q：枢纽关系？**  
A：门面持会话；会话挂服务箱；后台循环吃提交；回合任务绑「本轮快照」跑工具路由；工具运行时回头读会话/服务。

---

## 10. 一句收口

提交管控制，事件管观测；卷宗管事实，会话管当下；执行现场别当卷宗存。

---

## 11. 提交流循环落点 + `session_loop_termination`

**Q：`submission_loop` 在哪？**  
A：`handlers.rs` 里的 **`submission_loop` 异步函数**；`Codex::spawn` 里 **`tokio::spawn(submission_loop(..., rx_sub))`**，独占消费 **`Receiver<Submission>`**。

**Q：`session_loop_termination` 干啥？**  
A：等 **`submission_loop` 那个 `JoinHandle` 结束** 的共享 future；`shutdown_and_wait` 先发 `Op::Shutdown` 再 **await** 它，保证后台循环真收尾。

---

## 12. `Codex` 和 `CodexThread`

**Q：区别？**  
A：**`Codex`** 是运行时门面（`submit` / `next_event` + `Arc<Session>`）。**`CodexThread`** 再包一层线程产品语义：`session_configured`、`rollout_path`、goal 辅助 API、`validate_turn_context_overrides` 等，多数方法**转发**到内层 `Codex`。

**Q：为啥分开？**  
A：**职责切开**：内核稳定；app-server 线程层杂项不污染所有 `Codex` 调用方。

---

## 13. `apply_goal_*` 要不要经过 `Codex`？

**Q：`CodexThread::apply_goal_*` 绕不绕 `Codex`？**  
A：**绕**：`self.codex.session.goal_runtime_apply(...)`。逻辑在 **`Session::goal_runtime_apply`**（`goals.rs` 里 `impl Session`），读写 **`Session.goal_runtime`**。手里只有 `Arc<Session>` 也可直接调，但 **`CodexThread` 这条 API 路径必经内层 `Codex`**。

---

## 14. `SessionTask` 有几类？都会再起一个 `Codex` 吗？

**Q：实现有哪些？**  
A：产品里常见四种：**`RegularTask`**、**`CompactTask`**、**`ReviewTask`**、**`UserShellCommandTask`**。单测里还有测专用假任务。`TaskKind` 枚举只有 **Regular / Review / Compact**；`UserShellCommandTask` 的 `kind` 仍报 **Regular**，靠 span 与 `run` 体区分。

**Q：都会 `Codex::spawn` 新会话吗？**  
A：**不会。** 多数在同一条 **`Arc<Session>`** 上 **`tokio::spawn` 跑 `run`**。**`ReviewTask::run`** 里会 **`run_codex_thread_one_shot` 起子 `Codex`**。**Guardian** 也起子 `Codex`，但**不走 `SessionTask`**（见下条）。

---

## 15. `ReviewTask` 怎么触发？

**Q：从哪进来？**  
A：提交 **`Op::Review { review_request }`** → **`submission_loop`** `match` → **`handlers::review`** → **`resolve_review_request`** → **`spawn_review_thread`** → **`sess.spawn_task(..., ReviewTask::new())`** → 发 **`EnteredReviewMode`**。之后才是 **`ReviewTask::run`** 里子 `Codex` + 消费事件 + **`exit_review_mode`**。

---

## 16. Guardian 为啥不是 `SessionTask`？和 `Review` 对比

**Q：Guardian 也起子 `Codex`，为啥不像 `Review` 一样用 `Task`？**  
A：**触发与工位语义不同**：`Review` 是显式 **`Op::Review`**，占父会话 **`ActiveTurn`** 一档 **`ReviewTask`**。Guardian 多在**审批链路内部**并行跑**自动预审**，父 **`RegularTask` 还在等人**，不宜再占同一套「独占工位」模型；用 **`GuardianReviewSessionManager`** 管复用、超时、熔断、委托事件更贴切。

**Q：是不是因为不是 `Op` 才不用 `Task`？**  
A：**不成立为充分条件**：关键在**并发与取消边界**，不是「有没有 `Op`」单条件。

---

## 17. `SessionState` 里为啥有「像 config」的字段？

**Q：`latest_rate_limits`、`active_connector_selection` 算啥？**  
A：**运行态**：限流快照来自 API 回包 merge；连接器选择是**会话内当前选中集合**；和 **`SessionConfiguration` 里固化启动快照**不是同一类。`SessionState` 管「跑起来之后世界回了什么、用户选了什么」。

---

## 18. `SessionState` 和 `TurnContext`

**Q：关系？**  
A：**`SessionState`** 全会话一份，锁在 **`Session`** 里，含历史 **`ContextManager`**。**`TurnContext`** 是**某 `sub_id` 一轮**的 **`Arc` 快照**，挂在 **`RunningTask`**，给 **`run_turn` / 工具**稳定读 cwd、权限、模型、工具表等。

**Q：为啥不直接用 `SessionState`？**  
A：**减锁、防回合中途全局变更导致前后语义漂移、收窄依赖面**；少数字段仍可用 `Arc` 内状态在本轮演进（元数据、计时、去重标记）。

---

## 19. `ToolRouter` / `ToolCallRuntime` / `Session` / `TurnContext`

**Q：谁和谁啥关系？**  
A：每次 **`run_sampling_request`**：**`built_tools(sess, turn_context, input_slice, ...)`** 现建 **`Arc<ToolRouter>`**（registry + specs + `model_visible_specs` + 并行 MCP 名）。再 **`ToolCallRuntime::new(router, session, turn_context, tracker)`**，在流式里 **`handle_tool_call`** 调 **`router.dispatch...`**。**`TurnContext`** 定本轮暴露边界；**`Session`** 供执行、审批、事件、服务读口。

---

## 20. `LiveThread` / `ActiveTurn` / `TurnState` / `RunningTask` / `TurnContext`（再收口）

**Q：一句链？**  
A：**`LiveThread`** 写卷宗；**`ActiveTurn`** 挂当前回合 **`RunningTask` 表** + 共享 **`TurnState`**（挂起审批、pending steer 等）；**`RunningTask`** 绑 **`TurnContext`** + **`SessionTask`**；**`RegularTask`** 跑 **`run_turn`**。

---

## 21. 再一句总背（工具链 + 子会话）

**`ToolRouter` 编译本轮可见工具；`ToolCallRuntime` 执行工具调用外壳；`Review` 用 `SessionTask` 占工位再起子 `Codex`；`Guardian` 旁路子 `Codex` 不占 `ReviewTask` 那套工位叙事。**

---

## 22. `ToolOrchestrator` 谁用？

**Q：内置工具哪些走 `ToolOrchestrator`？**  
A：全仓显式 `ToolOrchestrator::new` 基本三类：**shell 族 `run_exec_like`**（`shell` / `local_shell` / `shell_command` / `container.exec`）、**`apply_patch` 委托运行时分支**、**`UnifiedExecProcessManager::exec_command` 内部**。`write_stdin` 只碰已起进程，**不走**编排器。其余内置 handler **未引用** `ToolOrchestrator`。

**Q：`exec_command` 和 `shell`？**  
A：**两个不同工具名**：`exec_command` 一条 `cmd` 字符串 + PTY/`yield`/`write_stdin` 会话语义；`shell` 偏 argv + `timeout_ms` 一次性输出。都可并行工具，但管线不同（unified exec vs `run_exec_like`）。

---

## 23. 权限 profile / 文件系统 / 网络从哪来？

**Q：内置 `:read-only` / `:workspace` 谁写的？**  
A：**代码预制套餐**；你用 `default_permissions` **点名**用哪套。

**Q：自定义 `[permissions.xxx]`？**  
A：**你写 TOML** 的 `filesystem` / `network`，加载时 **compile** 成内部的 `FileSystemSandboxPolicy` + `NetworkSandboxPolicy`。内置套餐也可被 **`sandbox_workspace_write`** 等旋钮叠一层。

**Q：啥都不配？**  
A：仍有 **隐式默认**（信任未定时更偏保守 `:read-only` 等逻辑），不是零策略。

---

## 24. `:read-only` 人话

**Q：只读还能改代码吗？**  
A：**这套 profile 本来就不是为随便写仓库设计的**；日常改代码用 **`:workspace`** 或自定义可写 profile。

**Q：只读整台电脑还是一个文件？**  
A：**都不是字面那样**；是 **Codex 管理沙箱下**对「能读/能写哪」的策略，外加 **元数据路径保护**（`.git` / `.agents` / `.codex` 等）。不是 OS 级把你磁盘全局 chmod。

---

## 25. 用户提交 vs 工具并行锁

**Q：用户侧怎么不乱序？**  
A：**提交队列 + `submission_loop` 串行消费**，不是泛化「事件总线一条一条」。

**Q：工具并行怎么防踩？**  
A：**同回合一把 `Arc<RwLock<()>>`**：`supports_parallel==true` 拿 **读锁**（可叠加）；`false` 拿 **写锁**（独占、与读互斥）。**写锁会等**，不是「申请不到就失败」。**不是按文件 flock** 当主机制。

---

## 26. Rollout 恢复：`TurnContextItem` vs `PreviousTurnSettings`

**Q：啥区别？**  
A：**`TurnContextItem`** 是 **持久化整包回合基线**（cwd、审批/沙箱/权限/网络、模型、指令、截断等一大堆）。**`PreviousTurnSettings`** 是 **从上一回合抽的两字段小条**（`model` + `realtime_active`），给差分/再注入用；反向重建里从 `TurnContextItem` **抄这两样**填进去。

---

## 27. Goal：实现 +「会不会自己跑到完成」

**Q：Goal 是啥？**  
A：**每线程一条**、**SQLite `state_db`** 持久化；**`get_goal` / `create_goal` / `update_goal`** 给模型；**`Session` + `goals.rs`** 记账发 **`ThreadGoalUpdated`**；**`Feature::Goals`** 关则接口直接报错。

**Q：模型 `update_goal` 能随便改状态吗？**  
A：**不能**；handler 里 **只允许标 `complete`**；暂停/改目标等多走 **app-server `ThreadGoalSet` / `Clear` 等** 或 UI。

**Q：会自动跑到 goal 完成吗？**  
A：**没有强保证**。**空闲时** `MaybeContinueIfIdle` → 可能 **`push_pending_input` 塞一条 `developer` 消息**（`continuation.md` 渲染，含 `<untrusted_objective>` 与预算段）→ **`start_task(..., RegularTask)`** 再推一轮。**完成**仍要 **`update_goal(complete)`** 或外部 Set；另有 **rate limit guard**、**预算触顶 steering**、**plan 模式跳过 continuation** 等。

---

## 28. Multi-agent vs sub-agent；`list_agents`；`agent_type`

**Q：multi-agent 就是 sub-agent 吗？**  
A：**协作 spawn 那条是 `SessionSource::SubAgent(ThreadSpawn…)` + `ThreadSource::Subagent`，`AgentControl` 管树**。但 **sub-agent 不只有 multi-agent**（例：**Guardian**、**Review 子 `Codex`** 也带 subagent 语义，路径不同）。

**Q：`list_agents` 干啥？**  
A：**列当前根线程子树里还活着的 agent**（可选 `path_prefix` 按 `AgentPath` 过滤），给模型 **`send_message` / `wait` / `close`** 前 ** discovery**。

**Q：`spawn_agent` 的 `agent_type` 谁定义？**  
A：**字符串由模型填**，但必须匹配 **配置里解析出的 role 名**（内置 + 用户 role 文件）；未知会 **`unknown agent_type`**。不是主会话运行时「发明新类型对象」。

---

## 29. Memory：Phase1 / Phase2 产物 + 超长线程 + 护栏 + 扩展

**Q：Phase1 产物？**  
A：**SQLite `stage1_outputs` 一行 `Stage1Output` / 线程**（`raw_memory` md + `rollout_summary` + 元数据）；**upsert**，rollout 变新会 **刷新同线程那一行**，不是无限堆历史版本。

**Q：Phase2 产物？**  
A：磁盘 **`memories/` git 工作区**：同步 **`rollout_summaries/*.md`**、**`raw_memories.md`**、**`phase2_workspace_diff.md`**；内部 **consolidation `CodexThread`**（cwd=memory 根、关网、关 MCP/协作/再生成记忆）改 **`memory_summary.md`** 等成品；DB 记 **Phase2 job 成功 + watermark**。

**Q：线程超级长？**  
A：Phase1 输入 JSON **按 token 预算 `truncate_middle_with_token_budget`（头尾保留）**；仍再靠模型 **压缩成高信号笔记**，不是整卷拷贝进库。

**Q：`guard.rs` 护栏是啥？**  
A：**几乎只做 rate limit**：后端额度太紧就 **整条 memory 流水线跳过**。其它「护栏」分散在：**输出 schema strict、脱敏、Phase2 沙箱、禁止记忆递归、Phase1/2 系统提示卫生规则**。

**Q：extensions 人话？**  
A：**第二种资料入口**（例：内置 **`ad_hoc` 随手记**），每个子目录自带 **`instructions.md`** 告诉合并模型 **怎么读、多不可信、删资源时怎么清陈旧记忆**；不必把每种外来笔记硬编码进核心 Rust。

---

## 30. Prompt injection 防御

**Q：Codex 防吗？**  
A：**有， Defense-in-depth**：**沙箱/审批/工具门** 硬减伤；**Guardian「证据不可信」**、**goal/memory 模板 `<untrusted_objective>` / rollout 当 data not instructions** 等软提示；**项目 trust + 默认策略**；**Hooks** 可再加闸。  
**不能**说成「提示词层面彻底免疫」；**LLM 仍可能被带偏**，工程目标是 **限制后果**。
