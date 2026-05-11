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
