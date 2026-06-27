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

## 29. Memory 系统（一读懂）

> **一句话**：后台 **读写分离**——Phase1 从 rollout **抽**笔记进 DB；Phase2 **选 Top-N 同步文件** + 合并子 Agent **写手册**；运行时 Agent **读文件**（summary 自动注入 + 关键词文本搜），**不查 DB、不用向量**。

### 29.1 总流程

```
session 结束/空闲
  → Phase1（并行，每线程 1 次 LLM）→ memories_1.sqlite / stage1_outputs
  → Phase2 选 Top-256 → 机械写 rollout_summaries/ + raw_memories.md
  → git diff → 合并子 Agent → MEMORY.md / memory_summary.md / skills/
  → 下次 session：注入 summary，Agent 按需搜 MEMORY.md
```

**开关**：`Feature::MemoryTool`；**`use_memories`（读）** 与 **`generate_memories`（写）** 独立。

### 29.2 存哪？两套存储

| 位置 | 是什么 |
|------|--------|
| `~/.codex/memories/` | 运行时 Agent **能读**的文件工作区 |
| `memories_1.sqlite` | 流水线内部：`stage1_outputs`、`jobs`（**不是** `memory.sqlite`） |
| `state_5.sqlite` | 会话元数据；Phase1 认领 rollout job 也靠它 |

运行时 Agent **无 API 读 SQLite**；引用过去 session 靠 **文件 + citation 里的 `rollout_ids`**，必要时沿 `rollout_path` 读原始 jsonl。

### 29.3 Phase1：逐线程抽取

- **触发**：根会话、非 ephemeral、非 sub-agent、DB 可用；后台异步。
- **输入**：rollout JSONL（超长则 **truncate 中间、保留头尾**）；过滤 AGENTS.md / skill 注入块。
- **一次 LLM** → JSON：`raw_memory`、`rollout_summary`、`rollout_slug`。
- **只写 SQLite**，此时 **还不写** `raw_memories.md` / `rollout_summaries/`。
- 同线程 **upsert 一行**，rollout 更新会刷新该行，不堆历史版本。
- Stage1 系统提示 **compile-time 内嵌**，用户不可配（`extract_model` 可配）。

### 29.4 Phase2：合并进文件夹

1. 从 DB **选 Top-N**（默认 **256**，新鲜度 **30 天**）。
2. **代码机械同步**：`rollout_summaries/*.md`、`raw_memories.md`；掉出 Top-N 的 **从磁盘删掉**。
3. `~/.codex/memories/.git` 做 baseline；`phase2_workspace_diff.md` = **Rust 模板 + git diff**（给合并 Agent 看，非 LLM 生成）。
4. 有变更 → **合并子 Agent**（关网、无 MCP、禁递归记忆）改成品；成功后 reset baseline。
5. 无变更 → 不跑 Agent，直接成功。

**`usage_count` 干什么？**  
只影响 **Top-N 排序**（`usage_count DESC` → 时间），**不是入场券**。0 引用的新记忆 **可以进文件夹**——大家同为 0 时按 `source_updated_at` 竞争。

**冷启动陷阱**：若窗口内已有 **≥256 条 `usage_count≥1`** 的记忆占满槽位，新的 0 引用记忆 **本轮进不了 Top-N**（不是永远，旧记忆超 30 天会腾出位）。Citation 是 **事后反馈抬排名**，不是进盘前置条件。

### 29.5 三个成品：薄索引 vs 厚手册

| 文件 | 角色 | 运行时 |
|------|------|--------|
| `memory_summary.md` | 薄路由：画像 + 主题索引；首行必须 `v1` | **自动注入** developer instructions（~2500 tokens） |
| `MEMORY.md` | 厚手册：`# Task Group` → 偏好/知识/踩坑 + `keywords` | Agent **自己搜**（不自动注入） |
| `skills/<name>/` | 可复用流程包（`SKILL.md` + scripts 等） | 按需读；**≠** `~/.codex/skills`，**不自动 promote** |

**为何拆开？** Token 预算 + **渐进式披露**：先 summary 判断要不要查，再 grep `MEMORY.md`，最后才开 rollout summary / skill / 原始 jsonl。

**谁能改成品？** 只有 **Phase2 合并 Agent**。运行时 Agent **不能改** `MEMORY.md`；用户明确要求时只能写 `extensions/ad_hoc/notes/*.md`，等下轮合并吸收。

**`skills/` 会更新吗？** 会。合并 Agent 增量 **改进/合并/新增** skill，不是一次性；但槽位饱和或合并 Agent 认为信号不足时可能长期不动。

### 29.6 运行时检索：无向量

**没有 embedding / 向量 / BM25**（`tool_search` 的 BM25 只搜工具元数据，与 memory 无关）。

| 层 | 机制 |
|----|------|
| L0 | `memory_summary.md` 已注入，不必再打开 |
| L1 | Agent 从 summary 提关键词 → **搜 `MEMORY.md`**（shell `rg`/`grep` 或 `memories/search`） |
| L2 | MEMORY 指向时才开 `rollout_summaries/`、`skills/` |
| L3 | 要精确命令/报错 → 沿 `rollout_path` 搜原始 session jsonl |

`memories/search|read|list` 仅在 **`dedicated_tools=true`** 时作为专用工具；默认走 **shell + 子串匹配**（`contains`，可 normalized）。建议 **≤4–6 步** memory pass。

**Citation**：回复末尾 `<oai-mem-citation>` → `rollout_ids` 写回 SQLite `usage_count++`；读 MEMORY 本身只打遥测，**不写 DB**。

### 29.7 和 `thread/search` 的区别

| | Memory 检索 | `thread/search`（app-server experimental） |
|--|------------|-------------------------------------------|
| 搜什么 | `~/.codex/memories/` 文件 | `~/.codex/sessions/*.jsonl` 原文 |
| 算法 | 子串 / rg | **ripgrep 固定字符串** + `state_5.sqlite` 分页 |
| 向量 | ❌ | ❌ |
| 谁用 | 运行时 Agent | IDE / 客户端，**非** Agent 默认工具 |

### 29.8 其它

**`ad_hoc`**：`extensions/ad_hoc/instructions.md` 给合并 Agent；运行时只写 `notes/`。

**超长线程**：Phase1 输入 truncate + 模型压缩；不靠整卷拷贝。

**护栏**：`guard.rs` 主要 **rate limit** 跳整条流水线；另有 schema strict、Phase2 沙箱、禁记忆递归、提示词卫生规则。

**`phase2_workspace_diff.md`**：代码生成，合并 Agent 的「变更清单」，不是 LLM 写的。

---

## 30. Prompt injection 防御

**Q：Codex 防吗？**  
A：**有， Defense-in-depth**：**沙箱/审批/工具门** 硬减伤；**Guardian「证据不可信」**、**goal/memory 模板 `<untrusted_objective>` / rollout 当 data not instructions** 等软提示；**项目 trust + 默认策略**；**Hooks** 可再加闸。  
**不能**说成「提示词层面彻底免疫」；**LLM 仍可能被带偏**，工程目标是 **限制后果**。
