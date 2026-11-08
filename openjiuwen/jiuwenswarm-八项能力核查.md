# JiuwenSwarm 八项能力核查报告

> 核查对象：`openJiuwen-ai/jiuwenswarm`，branch `develop`，包名 `workswarm`，版本 **v0.2.8**（`pyproject.toml`：`python >=3.11,<3.15`，Apache-2.0，文件头为 Huawei Technologies 版权）。
> 本地副本：`D:\开源项目源码\jiuwenswarm`（浅克隆）。**三个外部依赖仓也已按 `pyproject.toml` 钉住的 rev 克隆到同一目录**：`D:\开源项目源码\agent-gateway`、`agent-core`、`agent-protocol`（rev 与日期见 §10.1）。
> 方法：只读源码 + 配置模板 + 仓内设计/bugfix 文档。**所有结论后面都带 `文件:行` 指针**；`[外部仓]` 标记的条目已在 §10 逐条闭环，**§10.4 原 6 条也已全部自查闭环**（其中对 §3 凭据隔离与 §1 会话映射两处结论做了更正）。
> **对外交付物 = 同目录 `客户答复-JiuwenSwarm八项能力确认.md`（厂商侧答复口径）；本文件是内部取证底稿，两者结论必须保持一致。**

---

## 0. 先厘清架构边界（读 8 项结论的前提）

| 组件 | 在哪 | 说明 |
|---|---|---|
| **Gateway** | `[外部仓 agent-gateway]` → **已克隆 `D:\开源项目源码\agent-gateway`** | `pyproject.toml` 里以 `agent-gateway @ git+https://gitcode.com/openJiuwen/agent-gateway.git@cff3b1b9…` 依赖。频道接入（Web/IM/E2A）、鉴权（`auth/`）、路由与会话映射（`routing/`）、对象存储（OBS/MinIO，`common/obs_files.py`）、附件安全（`common/attachment_safety.py`）、`tenant_key`（`common/tenant_key.py`）、WS 帧上限（`common/ws_limits.py`）、磁盘保留期（`cleanup.py`）都在那边。 |
| **AgentServer** | 本仓 `jiuwenswarm/` | Agent 运行时、会话、工具、MCP、记忆、上下文引擎。 |
| **Web 前端 / TUI** | 本仓 `jiuwenswarm/channels/web`、`packages/jiuwenswarm-tui` | 前端 5173。 |
| **协议** | `[外部仓 openjiuwen-protocol]` → **已克隆 `D:\开源项目源码\agent-protocol`**（含 `A2A/ A4P/ AgentRegistry/ MCP/ openjiuwen-protocol/`） | `ReqMethod` 枚举定义在外部；本仓只**补齐**缺失成员：`jiuwenswarm/common/schema/message.py` 的 `_ensure_req_method(...)`。所以「有哪些 RPC」的权威清单是 `jiuwenswarm/server/dispatch.py` 的 `HANDLERS` 表。 |
| **Agent 内核** | `[外部仓 openjiuwen-core]` → **已克隆 `D:\开源项目源码\agent-core`** | `openjiuwen.core.*`：`Model`/`ModelRequestConfig`（`core/foundation/llm/schema/config.py:259`）/`InteractiveInput`（`core/session/interaction/interactive_input.py:18`）/`Runner.resource_mgr` → `ToolMgr`（`core/runner/resources_manager/tool_manager.py:39`）/reasoning 能力表（`core/foundation/llm/reasoning.py`）。 |

**两个edition**：`jiuwenswarm/enterprise/edition.py::is_enterprise()`（环境变量 `JIUWENSWARM_EDITION`）。个人版和企业版在「多租户」上差别极大，下面每条都会分开说。

**Front / Runtime 分离**：`jiuwenswarm/server/supervisor/`（`runtime_supervisor.py`、`worker_process.py`、`worker_registry.py`、`affinity.py`、`shutdown.py`）+ `jiuwenswarm/server/worker/`（`main.py`、`runtime_host.py`、`dispatcher.py`、`event_store.py`、`gateway_bridge.py`）+ `jiuwenswarm/server/ipc/`（`server.py`/`client.py`/`loopback.py`/`stream.py`，用 `FileLock` 做 owner lease）。就绪状态机在 `jiuwenswarm/server/lifecycle.py`：`STARTING → TRANSPORT_READY → CONTROL_READY → RUNTIME_WARMING → AGENT_READY / DEGRADED / DRAINING / FAILED`，`wait_agent_ready(timeout)` 供前端等待。

**控制面 / 执行面分离**：`jiuwenswarm/server/control/methods.py` 用 `CONTROL_*_METHODS` frozenset 划分哪些 RPC 走控制面（不唤醒/不占用 agent 运行时），会话创建/切换/删除和项目 git 操作**留在执行面**。

---

## 1. 多用户并发：部署模型、并发机制、资源隔离

### 结论
**有完整的多租户机制，但隔离粒度是「逻辑隔离 + 目录隔离」，不是进程/cgroup 级硬隔离。** 水平扩展靠 Gateway 路由 + 多 AgentServer 实例。

### 1.1 租户身份怎么算
- `jiuwenswarm/enterprise/tenant/tenant_identity.py`
  - `extract_triple(request) -> (group_id, bot_id, user_id)`，三级来源：请求顶层字段 → `metadata.routing`（E2A 权威）→ `metadata.query`（Web HTTP 的 `X-Group-Id/X-Bot-Id/X-User-Id` 兜底）。
  - `extract_tenant_key()`、`bind_control_tenant()`（控制面也要绑租户）。
  - `tenant_key = md5("{group_id}::{bot_id}::{user_id}")`。
- 策略抽象：`tenant_strategy.py`（Protocol：`extract_tenant_key / resolve_manager / adopt_default_manager / bind_request / scoped_manager / cleanup`），实现 `single_tenant_strategy.py`（个人版）、`multi_tenant_strategy.py`（企业版）。

### 1.2 并发机制
- `jiuwenswarm/enterprise/tenant/tenant_agent_pool.py::TenantAgentPool`：每个 `tenant_key` 一个 `AgentManager`，装在 `AsyncLRUCache` 里。API：`get_agent_manager / tenant_scope / touch_manager_if_same / get_manager_nowait / put_nowait / iter_agent_managers_nowait / cleanup / get_agent_count`。
- `jiuwenswarm/enterprise/tenant/async_lru_cache.py::AsyncLRUCache(max_size, ttl_seconds)`：滑动 TTL、`touch_if_same`。
- **关键默认值**：`cache_max_size=None, cache_ttl=None` → **默认不淘汰**。源码注释写明是为了「避免长阻塞（如 `ask_user_question`）期间误淘汰」。
- 请求级绑定：`jiuwenswarm/server/context.py::RequestContext(request, sink, connection_id, services)`；`AgentServerServices.set_scoped_agent_manager()` / `.agent_manager` 属性（取当前请求租户的 Manager，取不到回落默认槽）。
- 分发：`jiuwenswarm/server/dispatch.py::try_dispatch(ctx)` 把每个 handler 包在 `bind_request_tenant` 里（`jiuwenswarm/enterprise/tenant/bind_request.py`）。
- 一个 `AgentManager` 内部再按 `channel_id → {cache_key/mode → JiuWenSwarm}` 二级索引（`jiuwenswarm/server/runtime/agent_manager.py`，2330 行），并有借用/钉住计数（见 §2）。
- **会话身份映射（Gateway 侧，⚠️ 默认按「群」共享上下文）**：`[外部仓] agent_gateway/routing/session_map.py`
  ```python
  class SessionMapScope(str, Enum):
      PER_CHAT_BOT       = "per_chat_bot"        # 默认：一个 (provider, chat, bot) 一个 agent session
      PER_CHAT_BOT_USER  = "per_chat_bot_user"   # 一个 (provider, chat, bot, user) 一个 session
  ```
  `load_session_map_scope()`（19）读 `gateway.session_map_scope`，非法值回落默认；`_make_key`（34）据此生成 `provider::chat::bot[::user]`。注释直说默认档位的语义：**「users in the same chat share context」**。
  → ~~群聊里默认所有成员共用一个 agent session~~ **已收窄（见 §1.2 补充）**：SessionMap 只用于 `feishu` / `feishu_enterprise` 通道（`message_handler.py:449-452`）→ **只有飞书群**默认共享一个 agent session 与上下文；Web 与其他 IM 不受该项影响。
  持久化：个人版写 `get_checkpoint_dir()/session_map.json`；**企业版注入 `SessionMapRepository` 后读写 DB 表 `session_map`，进程内 `_mapping` 只是 write-through 缓存**（49-51、86-121）→ 企业版会话映射是跨实例共享的（这也是 §1.4 多实例能水平扩的关键）。`get_session_id()`（123）已不再自造 id，缺映射直接 `RuntimeError("SessionMap cannot allocate session IDs; use AgentServer session.create")`。
  ⚠️ **作用范围已查实（收窄早期说法）**：`message_handler.py:449-452` 的 `_session_map_channel_types = frozenset({"feishu", "feishu_enterprise"})` → **SessionMap 只用于飞书族通道**（注释解释是为了避免 `feishu:None` 键错配导致每条消息新建会话）。Web 通道与其他 IM 通道不走 SessionMap，所以「群内共享上下文」的风险**仅限飞书群**。
- Gateway → AgentServer 的连接与路由层在 `[外部仓] agent_gateway/routing/`：`agent_client.py`(49 KB)、`base_ws_channel.py`(25 KB)、`e2a_proxy.py`(19 KB)、`agent_http_bridge.py`、`agent_request_timeout.py`、`session_sharing.py`(26 KB)、`session_metadata_remote.py`、`interaction_context.py`、`keys.py`、`mcp_remote.py`、`route_binding.py`（`GatewayRouteBinding`：`forward_methods` / `inbound_interceptor` / `outbound_interceptor` / `cleanup_handler` / `disconnect_handler` / `session_bind_handler` / `ws_channel` 委托）。
- Gateway 鉴权与会话归属：`agent_gateway/auth/`（11 个文件）—— `service.py`(16 KB)、`session_store.py`(18 KB)、`session_owners.py`(5 KB)、`login_credentials.py`(18 KB)、`account_kit.py`(19 KB)、`apig.py`(12 KB，APIG 对接)、`passthrough.py`、`remote_config.py`、`model_catalog.py`(14 KB，企业模型目录 + 发现失败冷却)。单实例互斥 `agent_gateway/gateway_lock.py`（`portalocker`，见 §5.3）。

### 1.3 资源隔离做到了哪一层
| 维度 | 做到 | 证据 |
|---|---|---|
| 工作目录隔离 | ✅ 企业版每租户独立根 `~/.jiuwenswarm/workspace_{tenant_key}`；个人版仍是 `~/.jiuwenswarm` | `enterprise/tenant/tenant_paths.py`；`get_agent_workspace_relative_dir()=agent/workspace`、会话 `agent/sessions`；`seed_tenant_agent_workspace()` 从 `resources/agent/workspace/` 拷 `AGENT/HEARTBEAT/IDENTITY/SOUL/USER.md` + `memory/MEMORY.md`（`_ZH`/`_EN` 后缀） |
| 按用户下发配置 | ✅ 企业版从 DB 读「有效配置」并套用 models / mcp / permissions | `enterprise/config/effective/{loader,apply_models,apply_mcp,apply_permissions,db_queries,schemas}.py` |
| 团队/组织级路由与配额 | ✅ | `enterprise/team/{membership,membership_registry,route_binding_mysql,route_binding_store,push_fanout,quota,member_limits,control_mysql}.py` |
| 工作空间配额 | ✅ | `enterprise/capabilities/workspace/quota.py`：`is_workspace_quota_enabled()`、`resolve_effective_quota(force_refresh=True)`、`check_workspace_write(additional_bytes=…)` |
| 进程/内存硬隔离 | ❌ 同一 AgentServer 进程内多租户共享事件循环与内存；无 cgroup、无 per-tenant 内存/CPU 上限 | 全仓无相关实现 |
| 租户级连接/凭据隔离 | ⚠️ 部分（见 §3，MCP 凭据与连接池是**进程级**） | — |
| 会话上下文按用户隔离 | ⚠️ 仅**飞书族通道**默认按 chat 共享，需配 `gateway.session_map_scope: per_chat_bot_user` | `[外部仓] agent_gateway/routing/session_map.py:10-43`、`message_handler/message_handler.py:449-452` |
| 会话映射跨实例共享 | ✅ 企业版走 DB 表 `session_map`（write-through 缓存） | `[外部仓] agent_gateway/routing/session_map{,_repository,_access}.py` |

### 1.4 部署模型（怎么横向扩）
- `docker/start.sh`：`ROLE=gateway|agentserver`，先跑 `update_conf.py` 再 `exec jiuwenswarm-${ROLE}`；`AGENT_RUNTIME_DEPLOY_MODE=off` 可关掉 runtime 扩展；`EXTENSION_DIRS` 分号分隔。镜像：`docker/Dockerfile.claw`、`Dockerfile.web`、`Dockerfile.sandbox`；OpenShift 变体 `docker/openshift/*`。
- `deploy/yuanrong/*`：k8s 风格部署脚本 + `gateway-config-yuanrong.template.yaml`；`deploy/observability/docker-compose.yml` 观测栈。
- **单机多实例**：`docs/zh/单机多实例运行.md` —— `~/.jiuwenswarm/instances.yaml`；端口 = base + index×1000（agent_server 18092 / web 19000 / gateway 19001 / frontend 5173）；`<workspace>/.instance.lock`（`fcntl.flock`，Windows 独占创建，`STALE_LOCK_TIMEOUT=30s`）+ `.instance.pid`；环境变量 `JIUWENSWARM_DATA_DIR`、`JIUWENSWARM_CONFIG_DIR`；CLI `jiuwenswarm-init --name`、`jiuwenswarm-start --list/--status/--stop/--restart`。实现见 `jiuwenswarm/instance_manager/lock.py`。

### 1.5 缺口 / 风险
1. **租户池默认无上限、无 TTL** → 用户数增长时 `AgentManager` 常驻，内存无上限增长（`tenant_agent_pool.py` 默认 `None/None`）。要么显式配 `cache_max_size/cache_ttl`，要么加外部回收。
2. 多租户同进程 → **MCP 凭据走 `os.environ`（进程全局）**，见 §3，这是最实的一条串用风险。
3. 没有 per-user 的并发请求数 / token 配额（只有 team 级 `quota.py`、`member_limits.py` 和 swarmflow 的 `swarmflow_budget`）。
4. 多实例之间不共享 workspace 目录 → 扩到多节点必须先解决 §5 的共享存储。
5. **`gateway.session_map_scope` 默认 `per_chat_bot`** → **仅作用于 `feishu` / `feishu_enterprise` 通道**（`message_handler.py:449-452`）：飞书群内所有用户共用一个 agent session 与上下文，需要每人独立时显式改 `per_chat_bot_user`（`agent_gateway/routing/session_map.py:10-17`）。Web 与其他 IM 通道不走 SessionMap，不受影响。
6. Gateway 与 AgentServer 的 `tenant_key` 算法同源（`agent_gateway/common/tenant_key.py`，jiuwenswarm 侧 re-export），这点是安全的；但 **`md5(group::bot::user)` 不是 HMAC、无密钥**，可枚举（知道三元组即可推出别人的 tenant_key），若把 `tenant_key` 当作对外不可猜的标识使用会有问题（当前只用于本地目录名与 OBS 前缀的 `sha256(tenant_key)`，风险可控）。

---

## 2. 用户级运行时资源回收：空闲回收、取消、超时、释放接口

### 结论
**取消 / 会话级释放 / 关闭链路很完整；「空闲自动回收」只有零散 TTL 常量，租户级 AgentManager 默认不回收。**

### 2.1 释放 / 取消接口清单（AgentManager 层）
`jiuwenswarm/server/runtime/agent_manager.py`：
- 借用与钉住：`_borrow_agent` / `_release_agent_borrower`（task done-callback）、`pin_agent` / `unpin_agent`、`_has_agent_borrowers`；结构 `self.agents: dict[channel_id, dict[cache_key/mode, JiuWenSwarm]]`、`_agent_borrowers: dict[int, set[Task]]`、`_agent_pins: dict[int,int]`、`_reload_lock`。
- 空闲退役：`_schedule_pending_tui_retirement`、`_retire_pending_tui_agent`、`_retire_tui_agent_if_idle(_locked)` → 日志 `[AgentManager] idle TUI root agent removed: cache_key=%s`。
- 取消：`cancel_all_inflight_work(reason="[gateway ws disconnect] ", *, exclude_session_ids)` → 逐 agent `agent.cancel_inflight_work`；`cancel_session_tasks`。
- 会话级清理：`cleanup_session_runtime(*, channel_id, session_id) -> bool`（失败聚合抛 `RuntimeError`，并退役空闲 TUI root agent）；`session_has_live_subagents`、`session_has_pending_interrupt`、`release_subagent_runtime_for_session(session_id, reason=…)`。
- 预热池：`warm_pool = AgentWarmPool(self)`（`jiuwenswarm/server/runtime/agent_warm_pool.py`，719 行）；`claim_prewarmed_session`、`wait_for_session_prewarm`、`begin_foreground_chat`/`end_foreground_chat`、`release_session_prewarm_claim`、`activate_session_prewarm`、`sync_prewarm_channels`。
- LLM 连接回收：`_evict_stale_llm_clients(effective_config)` —— 凭据变了就关掉旧 LLM 连接（diff 基线 `_last_model_conn_configs`）。
- 总关闭：`cleanup()` —— 取消权限重载任务、`await self.warm_pool.close()`、gather 退役任务、逐个 `agent.cleanup()`、清空所有字典。

### 2.2 Runtime 服务层（transport 无关）
`jiuwenswarm/runtime/service.py`（行号）：
`cancel_request` 1329｜`cancel_all_inflight_work` 1554｜`cleanup_session` 3153｜`stop_heartbeat_runs` 3262｜`stop_subagent_runtimes` 3312｜`stop_session_for_archive` 3375｜`stop_session_for_lifecycle` 3439｜`_stop_team_runtime_for_lifecycle` 3456｜`_dispose_agent_session_after_resource_release` 3553｜`close` 3575｜`abort_session_provision` 1213｜`_release_unawaited_interactions` 653｜`release_session_message_interactions` 3833｜`_borrow_paused_agent` 3078｜`_released_after` 3098。

### 2.3 会话生命周期契约（可插拔参与者）
- `jiuwenswarm/runtime/session_lifecycle.py`：`SessionKind(AGENT|TEAM)`、`SessionDescriptor`、`SessionLifecycleTarget`；事件 `SessionInputIntentEvent / SessionForegroundEvent / SessionExecutionEvent / SessionExecutionFinishedEvent / SessionInactiveEvent`；协议 `SessionActivityParticipant`（`session_input_intent / foreground_changed / execution_started / execution_finished / session_inactive`）、`SessionDeleteParticipant`（`before_delete / release_resources / delete_failed / delete_committed`）、`RuntimeResourceLease.release()`、`SessionRuntimeLifecyclePort`（`remove_response_waiter / pause_session / stop_session / publish_pause_completed / is_running`）；注册表 `RuntimeParticipantRegistry`。
- `jiuwenswarm/server/runtime/session/lifecycle.py`（704 行）：`kind, resource_id, action, *, block_execution=True` 闸门 + `generation` 计数 + `phase="block"|"check_sessions"`。
- 配套：`jiuwenswarm/runtime/session/execution_registry.py`(160)、`work_scheduler.py`(233)、`coordinator.py`（`idle_operation` 模式）。
- 离线清理：`jiuwenswarm/server/runtime/offline_session_cleanup.py`（40 行）。

### 2.4 对外 RPC：取消
- `ReqMethod.CHAT_CANCEL = "chat.cancel"` → `jiuwenswarm/server/handlers/chat.py::handle_chat_cancel`（140 行）：
  - `params.intent ∈ {cancel, supplement, pause, resume}`：**只有 cancel/supplement 才掐流式任务**（`session_stream_tasks` 里 `stream_stop_event.set()` + `task.cancel()`）；pause/resume 不掐（任务还在，pause 在 checkpoint 阻塞）。
  - `params.wait_for_stop` → `stop_session_for_archive()` 同步等待停止后再回包。
  - `params.target_team_id / target_request_id` → `cancel_targeted_expert()`（先停 producer 再停响应流，保证晚到的 cron cancel 不误伤同会话更新的流）。
  - 客户端断连：`metadata[E2A_INTERNAL_CANCEL_SOURCE_KEY] == E2A_CANCEL_SOURCE_CLIENT_DISCONNECT` → cancel 后 `cleanup_client_disconnect_session_runtime()`，失败回 `chat.interrupt_result {success:false, error:"session runtime cleanup failed"}`。

### 2.5 超时 / 空闲常量清单（实测抓到的）
| 键 | 默认 | 位置 |
|---|---|---|
| `stream_idle_timeout` | 120s | `jiuwenswarm/resources/config.yaml:369` |
| `stream_first_chunk_timeout` | 300s | `config.yaml:367` |
| `timeout`（HTTP） | 360s（video/audio/vision/image_gen 1800s） | `config.yaml:365,402,414,425,437` |
| `group_routing_timeout_seconds` / `group_stream_first_chunk_timeout_seconds` / `group_stream_idle_timeout_seconds` | 360 / 300 / 120（可用 `JIUWENSWARM_MODEL_GROUP_*` 覆盖） | `config.yaml:480-483` |
| `react.completion_timeout` | `null`（不限时，防长任务被静默截断） | `config.yaml:711` |
| sandbox `idle_ttl_seconds` / `idle_check_interval` | 600 / 60（`config.yaml` 里 `sandbox_idle_timeout_seconds: 600`、`sandbox_idle_check_interval_seconds: 30`） | `config.yaml:976,978`；`jiuwenswarm/common/config.py:3758-3759`，强制转换 `config.py:3871,3875,4442,4446` |
| `SESSION_ADAPTER_IDLE_TTL_SEC` | 2×60×60 | `jiuwenswarm/server/runtime/agent_adapter/interface_deep.py:1996` |
| `turn_idle_timeout_s`（外部 harness / codex） | 300.0 | `jiuwenswarm/server/runtime/agent_adapter/external_harness_helpers.py:351` |
| 观测 sink 会话写手空闲 | 300.0 | `jiuwenswarm/observability/sink.py:45` |
| `max_idle` | 128 | `jiuwenswarm/runtime/kernel.py:120` |
| 分享图上传空闲超时 | 60×60 | `jiuwenswarm/channels/web/app_web.py:374` |
| 事件超时判定 | `idle_for >= event_timeout_sec` | `interface_deep.py:23254` |
| `message_summary_offloader_config.ttl_seconds` | 300（空闲多久才允许 TTL 卸载） | `config.yaml:794` |

### 2.6 进程级回收（worker 生命周期）
`jiuwenswarm/server/supervisor/resident_lifecycle.py`：`startup_lock(socket_path, deadline)`；`worker_process(hello, socket_path, *, allow_legacy, expected_pid)` 校验 `deployment_scope / data_scope / pid / username / exe / argv(-m jiuwenswarm.server.worker.main --socket)`；lease 用 `FileLock(socket_path + ".owner.lock")`；异常 `WorkerShutdownIncomplete`、`LegacyWorkerUnowned`；优雅关闭回执 `{"kind":"shutdown.done","ok":True}`。

### 2.7 磁盘级保留期回收（Gateway 侧 —— 全系统唯一按时间自动回收的机制）
`[外部仓] agent_gateway/cleanup.py`（172 行，`start_background_cleanup()`）：
- 常量：`DEFAULT_CLEANUP_PERIOD_DAYS = 30`、`RECURRING_CLEANUP_INTERVAL_S = 24*60*60`、`FIRST_CLEANUP_DELAY_S = 10*60`、`AGENT_ID = "jiuwenswarm"`。
- 保留期解析 `_get_cleanup_period_days()`：环境变量 `CLEANUP_PERIOD_DAYS` **优先于** 配置 `session_cleanup.retention_days`；非法/负值回落 30；**`0` = 关闭自动清理**。
- `cleanup_old_sessions()`：按目录 `st_mtime` 与 cutoff 比较，`shutil.rmtree(ignore_errors=True)` 删 `get_agent_sessions_dir()` 下的过期会话目录，返回 `{removed, errors}`。
- `cleanup_orphan_file_ops()`：扫 `get_agent_workspace_dir()` 与 `get_user_workspace_dir()` 下的 `.agent_history/file_ops_jiuwenswarm_*.json`，文件名里不含任何现存 session id 的即视为孤儿并删。
- 调度：启动后 10 分钟跑第一次，之后每 24 小时一次；`asyncio` 任务名 `background-cleanup`，重活放 `run_in_executor`。
→ **注意这是「磁盘上会话目录的保留期」，不是内存/运行时回收**：它不会释放 `AgentManager`、不会关 MCP 连接、不会关 LLM client。§2.7 缺口 1 依然成立。
另外 Gateway 侧还有 `agent_gateway/health_check/`、`agent_gateway/heartbeat/` 两个小模块（进程存活与心跳），以及 `agent_gateway/common/updater.py` / `upgrade_executor.py`（升级路径）。

### 2.8 缺口 / 风险
1. **租户级 AgentManager 没有空闲回收**（§1 默认 `None/None`）。`cleanup_session_runtime` 只清会话，不回收租户 Manager；`TenantAgentPool.cleanup()` 只在进程退出路径调。
2. 空闲回收是**各处独立常量**，没有统一的「用户级 runtime idle policy」配置面（sandbox 有 `idle_ttl_seconds`，agent/session 没有对应键）。
3. 没有 per-user 的「最大并发运行数」闸门（`work_scheduler.py` 是会话级调度，不是租户配额）。
4. `cleanup_session_runtime` 失败聚合抛 `RuntimeError`，调用方需要自己兜，否则一次清理失败会冒到 RPC 层。

---

## 3. 按用户 / agent 隔离 MCP：连接池、凭据、作用域、回收、重连

### 结论（本次复核后已上调，早期「隔离最弱」的判断偏严）
**MCP 的「配置与挂载」按 AgentManager（企业版=按租户）扇出；「连接标识」由配置内容 + 已解析凭据哈希得出 → 凭据不同天然是两条连接，连接层隔离成立 ✅。** 残留的隔离弱点只有两处：**(a)** 凭据存储以 MCP **名字**为键、根目录 `get_workspace_dir()/"mcp"` 是进程级而非租户根 → 同名 MCP 跨租户共用一个凭据槽；**(b)** **skill 型** MCP 的 token 同步进 `os.environ`（进程级，BashTool 继承）。连接池 `ToolMgr` 进程级、**无 TTL 无上限**（属资源治理问题，不是隔离问题）。两条弱点均可用「每租户一进程」规避。

### 3.1 挂载作用域（按 agent 扇出）✅
`jiuwenswarm/server/runtime/agent_manager.py:1087 apply_mcp_change(name, action, *, enabled=True, target_channel_id=None)`：
- 只作用于**本 Manager 名下**的 live agent（`self.agents` 遍历，或只 `target_channel_id`），调 `agent.apply_mcp_change(...)`，绕过整份 `mcp.servers` 重载。
- 语义：至少一个 adapter 成功即 `True`；**全失败抛 `RuntimeError(first_error)`**（避免前端显示假「已连接」）；`remove/toggle` 且 `get_mcp_server_config(name) is None`（纯 skill / 纯 CLI 连接器）视为 no-op 成功（1145-1153）。

### 3.2 连接池：进程级 ⚠️
`agent_manager.py:1160 probe_mcp_live_connection(name) -> tuple[bool,str]` → `jiuwenswarm/common/mcp_config.probe_mcp_live_connection`。源码注释直说：

> 「That function talks to the **process-level `Runner.resource_mgr`** directly — no adapter instance needed — so cold-start（还没对话）也能校验 MCP，并把 **spawned stdio subprocess / HTTP connection 缓存**给第一轮对话的 reconcile 复用（不重复 spawn）。」

即：**连接（stdio 子进程 / HTTP client）挂在 openjiuwen-core 的进程级 `Runner.resource_mgr` 上，不按 tenant 分池。**

⚠️ **但「跨租户复用同一条连接」需要更正为条件成立**（本次复核 `jiuwenswarm/common/mcp_config.py:223-224,727-742`）：`server_id` 不是 MCP 名字，而是**配置内容哈希** ——
```python
_stable_mcp_server_id(scope, name, payload) -> f"mcp_{scope}_{name}_{sha256(json({scope,payload}))[:16]}"
# payload 在 223 行之前已经含「已解析的真实凭据」：stdio 的 params.env/args/cwd、HTTP 的 auth_headers/url
```
→ **两个租户配同一个 MCP 但 token 不同 ⇒ payload 不同 ⇒ digest 不同 ⇒ server_id 不同 ⇒ 各自独立连接，不会串用**；只有配置（含凭据）完全相同（例如企业统一下发同一把共享 token）才会复用同一条连接，这是符合预期的复用。`server_id_scope` 是常量 `"jiuwenswarm"`（`mcp_config.py:591`），所以隔离靠的是「凭据差异」而不是「租户维度」。

**已闭环（`agent-core@f3f8f576`）** —— 真正的池在 `openjiuwen/core/runner/resources_manager/tool_manager.py:39 class ToolMgr`：
- `_tools: dict[tool_id → Tool]`、`_mcp_server_resources: dict[server_id → McpServerResource]`、`_mcp_server_name_to_ids`、`_mcp_server_locks: dict[server_id → asyncio.Lock]` —— **键只有 `server_id`，没有 tenant/session/user 维度** → 进程内全局共享，坐实了跨租户复用。
- `add_tool_server(server_config, expiry_time=None)`（104）：同一 `server_id` 已注册时**直接返回缓存的 tool cards**，注释明写「不重复 spawn 第二个 stdio 子进程」；并发注册由 per-server_id `asyncio.Lock` 串行化（107-121）。连接失败抛 `StatusCode.RESOURCE_MCP_SERVER_CONNECTION_ERROR`，并把 anyio 的 `CancelledError`/`BaseExceptionGroup` 兜成 `BaseError` 让真实原因到达前端（136-163）。
- 回收接口：`remove_tool_server(server_id, ignore_not_exist=True)`（214）→ `client.disconnect()` + `_inner_remove_mcp_tools(tool_ids)` + 清 name→ids；`release()`（301）→ 遍历 disconnect 全部。
- 刷新/过期：`refresh_tool_server(server_id, skip_not_exist=False, force=False)`（254）按 `McpServerResource.expiry_time` 与 `last_update_time` 判断是否需要重取工具列表；`_inner_refresh_mcp_tools`（276）重建 `McpServerResource`。
- **没有连接数上限、没有空闲 TTL**：`_mcp_server_resources` 只在显式 `remove_tool_server` / `release()` 时清 → 与 §1.5 的租户池不淘汰是同一类问题。
- 重连策略（`openjiuwen/core/foundation/tool/mcp/client/reconnect.py`）：`@with_reconnect` **只重建一次传输并重试该次调用**；只认传输失败与 session 失效（`session terminated/not found/invalid session/no valid session`、`connection closed/stream closed`、`ClosedResourceError/BrokenResourceError/EndOfStream/RemoteProtocolError/Read/Write/broken pipe`、JSON-RPC `32600`（消息含 session）与 `-32000`）；**业务错误永不重试**；退避 `5s → 10s → … 上限 120s`，±20% 抖动（`_RECONNECT_BASE_DELAY/_MAX_DELAY/_JITTER` 145-147、`_record_reconnect_failure` 159）；per-client 锁（179）；`_lazy_reconnect` 为假时「Not connected」不重试（显式断开不自动复活，133-142）；重试次数由环境变量 `OPENJIUWEN_MCP_RECONNECT_ATTEMPTS` 决定（236-244）；SSE 优先走 transport 原生 `reconnect()`（195-213）。
- 传输实现：`openjiuwen/core/foundation/tool/mcp/client/{stdio_client,sse_client,streamable_http_client,playwright_client,openapi_client}.py`，每个都有 `connect/disconnect(timeout)/close()`。

### 3.3 凭据：加密但按机器/OS 用户，不按租户 ⚠️
`jiuwenswarm/server/runtime/mcp/credential.py`：
- `CredentialStore(workspace_dir=None)`（336）：落盘 `<workspace>/mcp/credentials/<name>.json`，一 MCP 一文件；信封 `{v, salt, nonce, ct}`，密钥 `HKDF-SHA256(machine_id + user_sid, per-file salt)`（`_get_machine_id` 70 / `_read_windows_machine_guid` 56 / `_get_user_sid` 105 / `_derive_key` 123）。旧明文文件下次 `_save` 自动迁移。
- 路径守卫 `_path()`（352）：名字禁止 `/ \ :`、`.`/`..`，拒绝 symlink，且 `resolve().parent` 必须等于存储目录 → 防目录穿越。
- 原子写 `_save()`（385）：同目录 `NamedTemporaryFile` + `os.replace`；Unix `chmod 0600`（Windows NTFS ACL 不走 mode，跳过）。
- 读取 API：`save_token/get_token/get_all/delete_mcp`（422-439）；`delete_mcp` 即「删凭据」接口。
- 占位符解析：`extract_placeholders`（202）、`load_token_schema`（229）、`required_tokens_from_schema`（246）、`build_credentials_prompt`（255）、`detect_credential_kind`（311）、`resolve_placeholders`（466，把 `${VAR}` 替进 env/headers/url/args）。
- **作用域问题（已确认）**：`_mcp_root()` = `get_workspace_dir() / "mcp"`（`credential.py:36 import` + `172-173`、`state_store.py:48,58-60`、`registry.py:17,132-134`）。`get_workspace_dir()` 定义在 `jiuwenswarm/common/utils.py:1768`，返回 `_resolve_paths()` 的进程级 `_workspace_dir`；租户级的是 `get_agent_workspace_dir(tenant_key=None)`（1780，企业版走 `~/.jiuwenswarm/workspace_{tenant_key}/agent/workspace`）。Gateway 侧同名实现 `agent_gateway/common/paths.py:138 get_workspace_dir()` 同样是**进程全局、无 tenant 参数**（`get_agent_workspace_dir()` 174 也无参数）。→ **企业版单进程多租户下，`mcp/state.json` 与 `mcp/credentials/*.json` 是跨租户共享的同一份文件**，落在 `~/.jiuwenswarm/agent/workspace/mcp/`，不在 `workspace_{tenant_key}` 下。所有 `CredentialStore()` 调用点都不传 workspace（`common/mcp_config.py:38`、`server/jiuwenmate/resource_assembly.py:683`、`agent_adapter/interface_deep.py:5590,5623`、`mcp/cli_driver.py:522`、`mcp/oauth_flow.py:182,195,275`、`mcp/registry.py:878,941,991,1078,1408,1573`）。
- ⚠️ **`os.environ` 同步的真实范围已查实（更正早期「MCP 凭据全走 env」的说法）**：`interface_deep.py:5559 _sync_mcp_credentials_environment()` 的 docstring 明写 ——
  > 「**Skill-only MCPs** (ctrip-wendao, netease-mail) run their bundled skill script via **BashTool, which inherits `os.environ`**. **MCP stdio servers get tokens via the McpServerConfig credential_resolver**, but skill scripts have no such hook — so their token env vars must be visible in the agent process's own environment.」

  → **stdio / HTTP 型 MCP 的连接凭据是按连接注入的**（`build_mcp_server_config(..., credential_resolver=...)` 把 `${VAR}` 替进 `params.env`（193）/ `auth_headers`（213）/ `url`（204）/ `args`（180）），**不经过 `os.environ`**；**只有 skill 型 MCP**（能力由 agent 用 BashTool 跑脚本提供）的 token 会写进进程 env（`os.environ[k] = value`，5606）。
  → 因此多租户串用的真实面是：**① `CredentialStore` 按 MCP 名字存、不按租户（同名 MCP 共用一个凭据槽）；② skill 型 MCP 的 token 在进程 env 里对所有租户的 skill 脚本可见**。而不是「所有 MCP 连接都串 token」。

### 3.4 连接状态与回收
`jiuwenswarm/server/runtime/mcp/state_store.py`：`<workspace>/mcp/state.json` 单文件 + `threading.Lock` 串行写（读并发安全，最坏读到 stale）。API：`read_mcp_state / get_mcp_record / list_connected_mcps / list_truly_connected_mcps / list_connecting_mcps / list_tui_enabled_mcps / connected_mcp_skill_dirs / list_registered_mcps / upsert_mcp_record / remove_mcp_record / set_mcp_state / record_to_mcp_entry / set_mcp_enabled`。

`jiuwenswarm/server/runtime/mcp/registry.py`（1480+ 行）关键接口：
- 连接：`connect_mcp(name, *, install_only=False)` 789；`_connect_cli` 1131；`_finalize_cli` 1221；`complete_cli_auth(name, step_index)` 1309；`complete_oauth_callback(query)` 1350；`complete_host_oauth_auth(name)` 1369。
- 取消 / 回滚：`cancel_connect(name)` 746、`was_connect_cancelled` 779、`clear_connect_cancel` 784、`rollback_failed_connect(name)` 1018。
- 断开 + 清凭据：`disconnect_mcp(name)` 1413、`_wipe_stored_credentials(n)` 1397。
- 自定义：`register_custom_mcp(name, config)` 1467、`delete_custom_mcp(name)` 1544、`save_mcp_credentials(name, tokens)` 1584。
- 查询：`list_marketplace_mcps(mcp_filter="builtin")` 457、`get_mcp` 529、`get_mcp_skills` 613、`get_mcp_tools` 632、`build_config_entry` 717、`_mask_mcp_spec`（脱敏）298。
- 错误类型：`CliConnectError(code, message, *, runtime, install_cmd)` 98、`McpRegistryError(code, message)` 111。
- OAuth / CLI 驱动：`mcp/oauth_flow.py`(435)、`mcp/cli_driver.py`(716)、`mcp/call_timeout_patch.py`(188)、`mcp/apps.py`(285)、`mcp/marketplace.py`(355)、`mcp/skill_installer.py`(174)。

### 3.5 对外 RPC
`jiuwenswarm/server/handlers/mcp_hub.py`（866 行）：`handle_mcp_connect / disconnect / cancel_connect / wait_auth / save_credentials / register_custom / delete_custom / install / uninstall / list / show`；`jiuwenswarm/server/handlers/mcp.py`(618)：`handle_command_mcp`（`/mcp`）。
ReqMethod：`MCP_LIST / SHOW / INSTALL / UNINSTALL / REGISTER_CUSTOM / DELETE_CUSTOM / CONNECT / DISCONNECT / CANCEL_CONNECT / SAVE_CREDENTIALS / WAIT_AUTH`、`COMMAND_MCP="/mcp"`。
企业版按用户下发：`jiuwenswarm/enterprise/config/effective/apply_mcp.py`。连接器凭据 schema 示例：`connectors/baidu-netdisk/mcp.json` + `token-schema.json`。
默认配置模板：`config.yaml:1643 mcp.servers: []`（示例含 `transport: stdio|sse`、`command/args/cwd/env`、`headers.Authorization: Bearer ${MCP_TOKEN}`、`timeout_s: 30`）。

### 3.6 原「仍未闭环」条目 → 逐条闭环（含两处对早期结论的更正）
1. ~~`Runner.resource_mgr` 的连接池是否有 TTL、上限、按 key 隔离~~ → **已闭环**：见 §3.2，`ToolMgr` 无 TTL、无上限、key 只有 `server_id`。
2. ~~企业版多租户下 `get_workspace_dir()` 是否已被 patch 成租户根~~ → **已闭环：没有 patch**，MCP 状态与凭据跨租户共享同一份文件（见 §3.3）。
3. ~~`os.environ` 同步 token 的多租户正确性（结构性串用）~~ → **已查实并收窄**：`server_id` 含凭据哈希（见 §3.2），**MCP 连接不会跨租户串 token**；`_sync_mcp_credentials_environment` 的 docstring 也确认 stdio 走 `credential_resolver`、**只有 skill 型 MCP** 的 token 进 `os.environ`。残留的两条真实风险是：**(a)** `CredentialStore` 以 MCP **名字**为键、目录是进程级 → 同名 MCP 跨租户共用一个凭据槽；**(b)** skill 型 MCP 的 token 在进程 env 中，对所有租户的 BashTool 脚本可见。两者都可用「每租户一进程」（`docs/zh/单机多实例运行.md`）规避。
4. ~~刷新路径幂等性存疑，需实测~~ → **已闭环，降级为内核侧潜在缺陷**：全仓 grep 确认 **jiuwenswarm 从不调用 `refresh_tool_server`**（只有两处注释提到 `add_tool_server`），产品路径是「先 `remove_tool_server` 再 `add_tool_server`」；core 还专门写了 `tests/unit_tests/core/runner/test_tool_manager_mcp_dedup.py` 钉住「同 `server_id` 重复 add 必须复用缓存、不得 `already exist tool`」。内核 `_inner_refresh_mcp_tools`（276）确实不先摘旧卡片，若外部调用方直接用 `refresh_tool_server(force=True)` 会撞 `add_tool` 的 `ValueError`（61-63）→ **对外建议：先 remove 再 add**。

---

## 4. 用户画像：字段、来源、更新策略、隐私边界、改删接口

### 结论
**没有「画像 schema」这个东西。** 画像 = 一组 Markdown 人设/记忆文件 + 记忆引擎 + 一个可选的「个人上下文采集」子系统。有编辑/开关接口，**没有删除单条画像、导出、合规遗忘的接口**；隐私边界是 prompt 级约束，不是硬闸门。

### 4.1 画像载体（schema 的事实来源）
每个租户 workspace 下（`enterprise/tenant/tenant_paths.py`）：
| 文件 | 作用 | 模板 |
|---|---|---|
| `AGENT.md` | agent 行为约定 | `jiuwenswarm/resources/agent/workspace/AGENT_{ZH,EN}.md`（1398/1350 B） |
| `IDENTITY.md` | 身份 | `IDENTITY_{ZH,EN}.md`（569/551 B） |
| `SOUL.md` | 人格 | `SOUL_{ZH,EN}.md`（592 B） |
| `USER.md` | **用户画像槽位** | `resources/agent/workspace/USER.md` —— **0 字节空文件**（无字段定义） |
| `HEARTBEAT.md` | 心跳任务 | 同上目录 |
| `memory/MEMORY.md` | 长期记忆真源 | `memory/MEMORY_{ZH,EN}.md` |

`seed_tenant_agent_workspace()` 按语言后缀 `_ZH/_EN` 拷贝。**结论：`USER.md` 是空模板 → 画像字段完全由模型自由发挥，没有 schema、没有类型、没有校验。**

### 4.2 记忆引擎与「画像存储位置」
`config.yaml:142-221`（两个正交开关）：
- `memory.mode: local | cloud`（内置记忆存哪，不影响外接）
- `memory.engine: builtin | external | both | none`
- 内置 jiuwen 变体 `memory.builtin.jiuwen`：`mode: http | sdk`、`base_url`、`api_key`、**`user_id: ${MEMORY_USER_ID:-__default__}`**、**`scope_id`**、`session_id`（空=跨会话共享，显式=按会话隔离）、`agent_id: jiuwenswarm`、`space`、`identity_org/identity_user`、`session_end_extract`。→ **这才是真正的「画像数据隔离标识」，但默认全是 `__default__`，即不隔离。**
- `sdk` 模式装配：`globals.embedder_dim=1024`、`llm_max_tokens: 4096`（抽取输出上限）、`markdown_store.default.params.root`（留空 = `{user_workspace}/agent`）。
- 外接记忆：`memory.external.provider` + `jiuwenswarm/agents/harness/common/memory/external_memory_builder.py`（317 注释：`rerank/graph/enable_thinking/embedder_ssl_verify` 在此钉死，不对外暴露）。
- 其它记忆子系统：`task_memory`（`config.yaml:878`，检索算法 `ACE | ReasoningBank | ReMe`）、TTSE 经验记忆（`react.ttse`，`consult_top_k: 8`、`consult_retrieve_mode: hybrid|embed|bm25`）、`modes.{agent,code}.memory.enabled`（关则 `coding_memory_*` 工具不可用、项目记忆不加载）。
- 文档：`docs/zh/记忆.md`、`自动记忆.md`、`经验记忆.md`、`编码记忆.md`。

### 4.3 隐私边界（唯一成形的机制）
`config.yaml:213-218`：
```yaml
memory.forbidden_memory_definition:
  enabled: false
  patterns: []
  description: {zh: "以下内容禁止记忆：…", en: "…"}
```
实现：`jiuwenswarm/agents/harness/common/memory/forbidden.py`（`_get_memory_forbidden_config` 42、读取 124、生效 202）+ `jiuwenswarm/agents/harness/common/rails/memory_forbidden_rail.py`（把规则注入 system prompt，优先级 `SystemPromptPriority.MEMORY_FORBIDDEN = 113`，`jiuwenswarm/agents/harness/common/prompt/priority_registry.py:82`）+ `jiuwenswarm/agents/harness/common/tools/memory_tools.py:309,380`（写记忆时消费 forbidden）。
**生效范围（本轮实测，别写成全链路强制）**：
- 约束的是**模型写入**：提示词 rail + `write_memory` / `edit_memory` 工具侧检查。
- **管理 RPC 侧不跑 patterns**：`memory_rpc.py:38-41` 的 `_is_forbidden_enabled()` 只读 `enabled` 布尔标志，用途是状态展示（`:370`）与开关（`:469-472`）→ 走 `memory.edit` 手工改画像文件**不受禁记规则约束**。
- **群聊硬闸（本轮新发现）**：`memory_tools.py:24` `_GROUP_CHAT_MODE = ContextVar(bool)`（按请求上下文隔离，不跨会话串）；`:307` `write_memory` 返回 `{"success": False, "error": "群聊模式下禁止写入记忆文件"}`；`:378` `edit_memory` 同理返回「禁止编辑记忆文件」。→ 群聊默认不会沉淀个人记忆，**无需配置**。
**性质：这是「提示词 + 工具侧检查」，不是落盘前的强制脱敏闸门。** 没有 PII 正则库、没有加密落盘、没有分级标签。

### 4.4 改 / 查 / 开关接口（有）—— 注意：**三套记忆，接口各属不同套**

| 套 | 存储 | 模型侧入口 | 管理侧入口 |
|---|---|---|---|
| **内置本地**（`memory.engine` 默认 `builtin`） | 工作空间 md（`USER.md`/`MEMORY.md`/`memory/*.md`）+ `MemoryIndexManager` 本地索引（`memory/manager.py:58`） | 工具 `memory_search`（`tools/memory_tools.py:167` `@tool`）、`write_memory`、`edit_memory`、`memory_get`、`read_memory` | **`memory.*` 五个 RPC** |
| **内核 lite** | KV/Vector/DB（`coding_memory_dir`） | `coding_memory_write/_edit/_read`，由内核 `CodingMemoryRail` 注册（`agent-core/openjiuwen/harness/rails/memory/coding_memory_rail.py:180` → `create_coding_memory_tools`） | 无 |
| **外接** | `openjiuwen` / `mem0` / `openviking` / `lakebase` | **无工具**：`ExternalMemoryRail`（`priority = 75`、`PREFETCH_TIMEOUT = 5.0`、`user_id/scope_id/session_id` 默认 `__default__`）预取注入 | **无 RPC** |

- 外接装配：`interface_deep.py:22131 _build_external_memory_rail()` → `:22278 self._external_memory_rail = ...`；provider 分派见 `memory/external_memory_builder.py:5-8,43-56`（`_BUILTIN_PROVIDERS = {"openjiuwen","mem0","openviking"}` + `lakebase` 分支）；`engine` 读取点 `memory/external_memory_config.py:40`（`or "builtin"`）。
- **结论：外接记忆只有 provider 配置这一条路，没有读写工具、没有管理 RPC、没有 UI 管理面。** 客户要「可管理的外接记忆」必须自建。

**方法名定义在协议仓，不在产品仓**（在产品仓搜 `MEMORY_LIST` 搜不到）：
`agent-protocol/openjiuwen-protocol/openjiuwen_protocol/message.py:83` `class ReqMethod(Enum)`；`:188-192` = `memory.list / memory.edit / memory.status / memory.toggle / memory.open`。

**产品侧注册点**：`jiuwenswarm/server/agent_ws_server.py:1374` 与 `:2416` 两处 `MemoryAdapter()` 挂进 `AdapterRegistry`（`:1381/:2423 register`，`:1382/:2424` 写回 `app_services.adapter_registry`）。`dispatch.py` 的 handler 表里**没有** `MEMORY_*` 条目 → 走 adapter registry 命中。

`jiuwenswarm/agents/harness/common/memory_rpc.py`：`handle_memory_list` 258、`handle_memory_edit` 301、`handle_memory_status` 352、`handle_memory_toggle` 459（写 `update_memory_forbidden_enabled_in_config`）、`handle_memory_open` 604、`_update_mode_memory_config(mode, field, value)` 582。
路径守卫：`_validate_edit_path` 115、`_get_allowed_dirs` 57、`_is_in_allowed_dirs` 70、`_get_coding_memory_dir` 44、`_classify_memory_file` 154、`_is_creatable_memory_file` 170、`_safe_stat` 213、`_count_lines` 221、`_scan_md_files` 229。
RPC 入口：`jiuwenswarm/server/runtime/gateway_adapter/memory_adapter.py::MemoryAdapter`（`methods` frozenset 见 `:35-43`；`handle()` 里 `workspace = str(get_agent_workspace_dir())` 见 `:47`）。

**该文件 docstring 是隐私边界的关键自述（1-7 行，原文照录，别再转述）**：
```python
"""MemoryAdapter: execute TUI memory management in the user AgentServer.

The Gateway must not derive a workspace path or access memory files itself.
This adapter deliberately uses only ``get_agent_workspace_dir()`` from the
current process, whose data directory is supplied when the AgentServer starts.
``user_id`` carried by the E2A envelope is never used for directory selection.
"""
```
⚠️ 原文第一行限定为 **TUI** memory management（Web 侧复用同一 adapter）；「隔离边界 = 进程/数据目录边界」是**我的解读，不是原文承诺**，对外引用时必须标注。

其它 RPC：`MEMORY_FORBIDDEN_GET = "memory.forbidden.get"`（控制面）、`COMMAND_MEMORY = "/memory"`、`COMMAND_RECAP = "/recap"`（后两个是 TUI 命令，**对外答复不必列**，客户不关心界面入口）。

### 4.5 可选的「个人画像」子系统：PersonalContext
`jiuwenswarm/server/personal_context/host_api.py`（1879 行），`class PersonalContextHostAPI`（758），配置文件 `personal_context.yaml`（`_CONFIG_FILENAME` 36，`_MAX_CONFIG_BYTES = 4 MiB`）。
API：`configure` 802、`get_overview` 1074、`get_runtime_config` 1092、`patch_runtime_config` 1100、`select_model` 1132、`set_collection_enabled` 1162、`set_agent_use_enabled` 1210、`set_master_enabled` 1240、`apply_master` 1270、`rollback_master` 1280、`list_fetch_services` 1301、`create_fetch_service` 1312、`delete_fetch_service` 1408、`patch_fetch_service` 1511、`get_fetch_run_status` 1602、`set_fetch_service_enabled` 1628、`run_fetch` 1676、`stop_fetch_run` 1694、`get_graph` 1708、`get_tree` 1718、`search_graph` 1728、`get_graph_page` 1733、`get_source` 1738、`get_authorization_status` 1743、`authorize_provider` 1821、`start/stop/get_status` 1959/2016/2001。
凭据：`_REPOSITORY_PAT_FIELDS = {"github": "token", "gitcode": "pat"}`（42）、`_validate_repository_pat` 654（15s 超时 `_PAT_VALIDATION_TIMEOUT_SECONDS`，响应上限 1 MiB）、`_normalize_provider_credentials` 505。
安全细节：`_reject_symlink_chain` 293、`_publish_yaml` 387（temp+replace）、`_directory_capacity_defaults` 49。
开关：`AgentManager.set_personal_context_runtime_enabled(enabled)`。
→ 这是「从 GitHub/GitCode 等源采集个人数据 → 知识图谱」的画像流水线，**字段是图节点/源，不是固定 schema**；有 `delete_fetch_service`，但**没有「删除某个画像节点 / 清空画像」的接口**。

### 4.6 缺口 / 风险
1. **无画像 schema**：`USER.md` 空文件，字段全靠模型即兴；没有 JSON Schema、没有版本、没有迁移。
2. **无删除接口**：`memory.edit` 能改文件，但没有「遗忘某条画像 / 按主体清除 / 导出画像」的 RPC；GDPR 式「被遗忘权」需要自建。
3. **隐私边界是软的**：`forbidden_memory_definition` 默认 `enabled: false`、`patterns: []`，且靠 prompt 生效。
4. **隔离标识默认不隔离**：`memory.builtin.jiuwen.user_id/scope_id` 默认 `__default__`；多用户部署必须显式配 `MEMORY_USER_ID`，否则记忆库跨用户混。
5. `memory_adapter` 明确拒绝用 E2A `user_id` 选目录（安全上正确），但也意味着**画像隔离完全依赖进程/数据目录隔离**（§1.3），不是请求级隔离。

---

## 5. 工作空间文件跨节点可用：共享、权限隔离、锁、变更通知、快照

### 结论
**默认 workspace 是节点本地目录，跨节点必须显式挂共享存储（官方给了 NFS 脚本）。锁全是本地 advisory 文件锁，跨节点不可靠；没有跨节点变更通知；快照能力靠 git，没有独立快照 API。**

### 5.1 默认与共享路径
- 默认：个人版 `~/.jiuwenswarm`；企业版 `~/.jiuwenswarm/workspace_{tenant_key}`（`enterprise/tenant/tenant_paths.py`）。
- 官方跨节点方案：`scripts/nfs/README.md` + `setup_nfs_server.sh` / `setup_nfs_client.sh` / `teardown_*.sh`。README 原文要点：
  - 一个中心节点做 NFS server，一个或多个 client；
  - **分布式 Team 场景只共享 `team.workspace.root_path` 指向的团队共享目录，不要共享 `.agent_teams`**；
  - 默认共享目录 `${JIUWEN_TEAM_WORKSPACE_ROOT:-/tmp/jiuwenswarm/shared_workspace/jiuwen_team}`；
  - 建议内网 IP、所有节点先各完成一次初始化、**尽量用同一个用户运行**、自定义挂载点要在所有节点设同名 `JIUWEN_TEAM_WORKSPACE_ROOT`。
- 分布式 Team：`docs/zh/分布式Team.md` —— `team.runtime.mode/role/member_name`、`team.transport.type=pyzmq`、`team.transport.params`（`direct_addr`、`bootstrap_direct_addr`、`pubsub_*`）；默认端口 Leader `direct_port=18555/pub_port=18556/sub_port=18557`，Teammate `direct_port=18600`；`team.storage.type=postgresql` + `connection_string` 必须是所有节点可见的同一个库；`react.a2x_registry`（`config.yaml:847`，`base_url/timeout:30.0/api_key/ownership_file/role:teammate|teamleader/dataset:"team_pool"/endpoint`）**注册中心不是内置的**（外部 agent-protocol / Agentregistry）；分布式模式 = 每 channel 单活会话；FAQ 明说「默认 workspace 是各进程工作区下的本地目录，不是共享文件系统；需要显式使用共同可见路径」。
- 对象存储：`jiuwenswarm/common/obs_files.py` 是 **re-export shim** → `[外部仓 agent_gateway.common.obs_files]`；本仓消费点 `jiuwenswarm/server/runtime/attachments/obs_attachments.py`(94)。→ 大文件跨节点走 OBS/MinIO 是**外部 Gateway 能力**，本仓看不到实现。

### 5.2 权限隔离（路径级，不是 POSIX 级）
`jiuwenswarm/enterprise/capabilities/workspace/service.py`（442 行）：
```python
WorkspaceService(tenant_root, max_delete_depth=32, max_delete_entries=5000,
                 max_download_bytes=100*1024*1024)
```
- `resolve()` 用 `os.path.commonpath` 防越界；`list_tree`、`delete_entries`、preview（`_DEFAULT_PREVIEW_MAX_BYTES = 65536`）、download、`du`（优先 subprocess `du`，失败回落 `os.walk`）。
- `class WorkspaceError(code, message, details)`，code ∈ `BAD_REQUEST / FORBIDDEN / NOT_FOUND / INTERNAL_ERROR`。
- 区域分类：`enterprise/capabilities/workspace/zones.py`(86) `classify_zone / is_deletable / normalize_relative_path`。
- 配额：`enterprise/capabilities/workspace/quota.py`(671) `is_workspace_quota_enabled / resolve_effective_quota / check_workspace_write(additional_bytes=…)`。
- 文件适配器同样有 containment：`jiuwenswarm/server/runtime/gateway_adapter/workspace_file_adapter.py::_resolve_within_workspace()`（决策 D3，注入 workspace root 内才允许）。
→ **隔离 = 单租户根 + 路径 containment + zone 白名单。没有 ACL、没有 per-tenant uid、没有跨节点鉴权。** NFS 侧靠 export 的 `--client-ip` 和「同一个用户运行」，README 自己承认这一点。

**但配额确实是按用户强制的**：`jiuwenswarm/server/runtime/gateway_adapter/workspace_file_adapter.py:129-138` 在每个文件/工作空间请求进入前调 `TenantAgentPool.extract_triple(request)` + `bind_quota_identity(group_id, bot_id, user_id)`（`finally` 里 `reset_quota_identity`），使 `quota.py::check_workspace_write()` 拿到的是当前请求的用户身份，而不是进程默认值。

### 5.3 锁（全是本地 advisory 锁）
| 用途 | 机制 | 位置 |
|---|---|---|
| 实例互斥 | `fcntl.flock` / Windows 独占创建，`STALE_LOCK_TIMEOUT=30s` | `jiuwenswarm/instance_manager/lock.py` |
| 项目 store 跨进程互斥 | `fcntl.flock`（锁文件不被 `os.replace` 覆盖） | `jiuwenswarm/server/runtime/session/project_store.py` |
| IM 绑定 | `AsyncFileLock(str(dir/(key+".lock")), timeout=30)` | `jiuwenswarm/common/im_binding_lock.py` |
| Gateway designer adapter | `portalocker.LOCK_EX \| LOCK_NB` | `jiuwenswarm/server/runtime/gateway_adapter/designer_adapter.py:86` |
| a2x ownership | `fcntl.flock(fd, LOCK_EX\|LOCK_NB)` | `jiuwenswarm/agents/harness/team/a2x/client/ownership.py:74` |
| 进程 CLI 会话守卫 | portalocker `LOCK_EX\|LOCK_NB` | `jiuwenswarm/channels/process_cli/session_guard.py:40` |
| 会话记忆证据 | portalocker，`timeout=60` | `jiuwenswarm/agents/harness/common/rails/eternal_conversation/evidence.py` |
| worker owner lease | `FileLock(socket_path + ".owner.lock")` | `jiuwenswarm/server/supervisor/resident_lifecycle.py` |
| Team 文件锁超时 | `from openjiuwen.agent_teams.skill.file_lock import FileLockTimeout` | `jiuwenswarm/server/runtime/agent_adapter/team_helpers.py:363` |
| Gateway 单实例（每 workspace） | `portalocker`，锁文件 `.gateway.lock`，持有整个生命周期，`GATEWAY_LOCK_ACQUIRE_TIMEOUT = 30.0`，Windows 用 `tasklist /FI "PID eq N"` 判活 | `[外部仓] agent_gateway/gateway_lock.py:26-27,38-56` |

→ **没有任何分布式锁（无 Redis/ZK/DB 锁）**，连 Gateway 自己的单实例互斥也是 `portalocker` 文件锁（`agent_gateway/gateway_lock.py`），与 `jiuwenswarm/instance_manager/lock.py` 是两套文件名但同样只在本机有效。NFS 上 `flock` 需要 `lockd`/`nfslock` 且语义弱；多节点并发写同一 workspace 没有正确性保证。
**唯一真正的 lease 语义在 A2X Registry**（`agent-protocol/AgentRegistry`）：README 自述提供「Agent Team 场景下的短时预订锁（reservation lease）」+ 状态声明 `online/busy/offline` + 基于所有权的权限控制 —— 但那是**服务发现/能力预订**的锁，不保护工作区文件，不能拿来当 workspace 互斥用。

### 5.4 变更通知
- 只有**本地** git 变更监听：`jiuwenswarm/server/runtime/session/git_diff_watcher.py`(1344) + `git_diff_status.py`(929)。
- Team 的 pub/sub 是 **agent 消息**（pyzmq `pub_port/sub_port`），不是文件变更事件。
- **没有** inotify → 跨节点广播、没有文件变更 webhook、没有 workspace 事件总线。

### 5.5 快照 / 版本
- 靠 git：`jiuwenswarm/server/runtime/session/project_git.py`(2077) 与 RPC `project.git.{status,probe,init,switch_branch,create_branch,commit,push,diff_status,turn_diff_list,turn_diff,discard_turn_changes,redo_turn_changes}`（`jiuwenswarm/server/runtime/gateway_adapter/project_adapter.py:1265-1291`）。注意：**没有 `git log` / `delete_branch` / `create_worktree` 的 RPC**（worktree 由 `WorktreeRail` 在 agent 侧管，见 `config.yaml:1695`）。
- 会话回滚：`SESSION_REWIND / SESSION_REWIND_AND_RESTORE / SESSION_REWIND_COMPACT / SESSION_REWIND_CONTEXT`（`jiuwenswarm/server/dispatch.py:228-231`）。
- worktree 隔离设计文档：`.agents/agent-docs/designs/archive/team_worktree_isolation_agenttool_cn.md`。
- **没有**「给 workspace 打快照 / 回滚到快照 / 快照列表」的独立 API。

### 5.6 缺口 / 风险
1. 跨节点共享 = 运维自己挂 NFS；**锁语义在 NFS 上不可靠**，官方建议「同一个用户运行」也说明没有真正的多租户文件权限模型。
2. 无跨节点文件变更通知 → 多节点 Team 只能靠 git 提交/拉取或 pyzmq 消息间接同步。
3. 无快照 API；回滚只有 git 与会话 rewind 两条路，粒度不同。
4. `.agent_teams` 明确不能共享（README + 分布式Team FAQ）→ 跨节点时「哪些目录能共享」需要人工切分，容易配错。

---

## 6. 上下文长度与推理等级：参数名、动态修改、生效范围、token budget

### 结论
**参数名和优先级链非常明确（源码即权威）；推理等级有 8 档 + 按模型能力校验（能力表在 core，profile 命中给精确档位、未命中回落协议默认）；`context_window` 已是 core 正式字段但被 core 自己 exclude、不发厂商，只作本地压缩依据；单模型输出上限可用 `model_config_obj.max_tokens`，但没有跨请求/跨会话的累计 token 预算。**

### 6.1 上下文长度（context window）
`jiuwenswarm/common/context_window.py`：
- `DEFAULT_CONTEXT_WINDOW_TOKENS = 256 * 1024`（11）、`MAX_CONTEXT_WINDOW_TOKENS = 2**53-1`（12）。
- `parse_positive_int()`（31）支持 `262144` / `256k` / `256 K` / `1m` / `1MiB`（单位乘子表 17-28）。
- `resolve_context_window_tokens(model_name, *, context_engine_config, model_config_obj, model_context_window_override)`（95）**优先级**（104-136）：
  1. 全局 `react.context_engine_config.context_window_tokens`
  2. 当前模型 `model_config_obj.context_window`（或运行时 `model_context_window_override`）
  3. 手工映射 `context_engine_config.model_context_window_tokens[model_name]`
  4. 固定默认 256K
- 配置面（`jiuwenswarm/resources/config.yaml`）：
  - 每模型：`models.{defaults,agentos,video,audio,vision,image_gen}[].model_config_obj.context_window`（373-385、451-474 注释；示例 `context_window: 262144`）
  - 全局：`react.context_engine_config.context_window_tokens`（776-777，注释「显式覆盖上下文窗口 token 上限（正整数）；不填则用当前模型配置或 256K 默认值」）
  - 映射：`model_context_window_tokens`（779-780）
  - 已停用自动获取：`enable_openrouter_model_context_window_tokens: false`（756，注释「已停用模型上下文自动获取；上下文窗口由模型配置或固定默认值决定」）
- **`context_window` 的传递语义（已在外部仓闭环，并更正本仓早期判断）**：`jiuwenswarm/common/reasoning_injector.py:15 core_has_context_window_field()` 运行时探测 `"context_window" in ModelRequestConfig.model_fields`，为假才在公共出口 pop 掉（防厂商 `unexpected keyword argument`）。**在钉住的 `agent-core@f3f8f576` 上，`context_window` 已经是正式字段**：`openjiuwen/core/foundation/llm/schema/config.py:259 class ModelRequestConfig` 声明 `context_window: Optional[int] = Field(default=None, description="Maximum context window supported by the model, in tokens; internal context metadata")`，同文件还有 `reasoning: Optional[Union[ReasoningConfig, dict]]` 正式字段，`model_config = {"extra": "allow"}`。→ jiuwenswarm **不再 pop**。
- 但「不发厂商」的结论仍然成立，只是责任移到了 core：`openjiuwen/core/foundation/llm/model_clients/base_model_client.py:623-638` 的 `model_dump(exclude={...})` 明确排除 `context_window`（注释「Context metadata is consumed by the context engine and must never be sent as a provider SDK request parameter」）与 `reasoning`（另走 wire controls）。
- core 侧另有**内置每模型窗口表**：`openjiuwen/core/context_engine/context/context_utils.py:29 MODEL_DEFAULT_CONTEXT_WINDOW_TOKENS`，以及 `fetch_openrouter_model_context_window_tokens(timeout=3.0)`（275）+ `prefetch_*`（240）+ `OPENROUTER_MODEL_CACHE_TTL_SECONDS` 缓存（285-293）。jiuwenswarm 用 `enable_openrouter_model_context_window_tokens: false`（`config.yaml:756`）关掉这条自动发现，优先级链见上。

### 6.2 推理 / 思考等级（reasoning level）
`jiuwenswarm/common/reasoning_config.py`：
- `ReasoningLevel = Literal["off","on","minimal","low","medium","high","xhigh","max"]`（8-18）。
- `normalize_reasoning_level()`（80）别名归一：`none/false/disable(d)→off`；`true/enable(d)→on`；`minimum→minimal`；`med/mid→medium`；`extra_high→xhigh`；`maximum/ultra→max`。**注意 YAML 1.1 会把裸 `off` 读成布尔 False**，代码专门让 `False→"off"`、`True→"on"`（135-137 注释）。
- `reasoning_config_for_level(level)`（106）→ core 的 provider-neutral 配置：`off→{"mode":"disabled"}`；`on→{"mode":"enabled"}`；其它 `{"mode":"enabled","effort":<level>}`。
- `validate_reasoning_level_for_model(...)`（120）：调 `[外部 core] openjiuwen.core.foundation.llm.get_reasoning_capability(provider, model, protocol, api_base, endpoint_profile)` 取该模型支持的档位，不在 `capability.options` 内 → `ValueError("reasoning_level must be one of: …")`；查询异常统一转 `ValueError`（保存路径返回 BAD_REQUEST 而不是 500）。
  **能力表已在外部仓闭环**：`openjiuwen/core/foundation/llm/reasoning.py:150 get_reasoning_capability(...)` → `find_reasoning_profile(provider_key, model)` 命中则 `profile_reasoning_capability(profile, protocol_key)`，**未命中回落 `standard_reasoning_capability(protocol_key)`**（docstring：「The lookup is advisory. Listed provider/model exceptions return exact capabilities; unlisted models inherit the standard selected protocol」）。协议归一：`anthropic|messages|anthropic-messages → anthropic-messages`，`openai|chat|chat-completions → chat-completions`（164-168）。`resolve_reasoning_protocol(client_config)`（134）按 `api_mode` / `auth_mode=OpenAIAccountOAuth` / provider 决定走 `anthropic-messages` / `responses` / `chat-completions`。
  前端可用 `get_reasoning_capability_catalog()`（218，返回 `protocol_defaults` + `model_fallbacks`）与 `get_provider_reasoning_rules()`（180，返回按 provider 的 `patterns` + `openai/anthropic` 两套 capabilities，**顺序即优先级**，注释要求调用方首个命中即用）做自定义模型档位推断。规则源是模块级 `REASONING_PROFILES` 表（同文件）。
- 端点方言 `endpoint_profile`（`config.yaml:357-363`）：`openai / deepseek / openrouter / siliconflow / dashscope / openai_compatible / ollama / lmstudio / vllm / sglang`；`client_provider=Anthropic` 时忽略。自建网关按 host 声明：`endpoint_profile_overrides: {"10.0.0.1": "vllm"}`（`config.yaml:339-347`；`resolve_endpoint_profile_override` 44、`effective_endpoint_profile` 60）。注释明说：这类网关认 `enable_thinking / thinking_budget`、忽略官方 `thinking.type`；**未声明则思考档位对这类网关静默不生效**。
- Kimi 端点/模型强制采样：`SAMPLING_OVERRIDE_RULES`（30）`api.moonshot.cn`/`api.kimi.com` → `temperature=1.0, top_p=0.95`；`resolve_sampling_override`（68）也按 `kimi-*` 模型名前缀命中。
- Team 侧映射：`jiuwenswarm/agents/harness/team/config_loader.py:204 _sanitize_team_member_model()` —— 把 UI hint `reasoning_level` 从 `model_request_config` 里摘掉，经 `build_reasoning_model_request_kwargs()` 转成 provider kwargs，否则 core `extra=allow` 会把 `reasoning_level` 原样透传给 `AsyncCompletions.create()` 报错（`team_manager.py:1835` 同一注释）。

### 6.3 超时（见 §2.5 表）
每模型 `timeout / stream_first_chunk_timeout / stream_idle_timeout`；模型组 `group_routing_timeout_seconds` 等（`config.yaml:480-483`，注释：组可覆盖 `groups[].routing.timeout_seconds / stream_first_chunk_timeout / stream_idle_timeout`，**route 级优先**；并说明「Streaming timeouts are stored per group for future support; IntelliRouter does not consume them yet」→ 组级流超时目前**未被消费**）。

### 6.4 token budget
| 键 | 语义 | 位置 |
|---|---|---|
| `swarmflow_budget` | **Team 级**：该 team 派生的**所有** swarmflow run 的总 token 上限；未设/`null` = 无上限 | `config.yaml:1729-1732` |
| `memory.builtin.jiuwen.sdk.globals.llm_max_tokens` | 记忆抽取单次最大输出 token（4096） | `config.yaml:203` |
| `symphony…max_output_chars: 12000` | 输出**字符**上限（注释：严格低于固定 1% 上下文阈值） | `config.yaml:82-84` |
| `session_token_usage.py` | 会话 token 统计 | `jiuwenswarm/server/runtime/session/session_token_usage.py`(207) |
| `tokenizer_service.py` | token 计数（54 KB 文件）；`enable_tiktoken_counter: false`（关则用字符串长度兜底）、`tokenizer_cache_dir: tokenizers`、`tokenizer_offline`、`tokenizer_registry` | `config.yaml:757-774`；`jiuwenswarm/server/runtime/tokenizer_service.py` |
| Swarmflow 并发闸 | `max_workflows`(L1) / `agents_per_run`(L2) / `max_agents_total`(L3) | `config.yaml:1794-1799` |
→ **没有全局 `max_output_tokens` / `max_tokens` 顶层键**；但**每模型可以限输出**：`ModelRequestConfig.max_tokens` 是 core 的正式字段（`schema/config.py:259` 起），`base_model_client.py:597-600` 明确「优先调用参数，否则取 `self.model_config.max_tokens`，非 None 才塞进 `params["max_tokens"]`」→ 在 `models.*[].model_config_obj.max_tokens` 里写就会真正发给厂商。缺的是**跨请求/跨会话的累计 token 预算**（只有 Team 级 `swarmflow_budget`）。

### 6.5 上下文压缩 / 卸载（上下文引擎）
`config.yaml:753-823`，`react.context_engine_config`：
- `enabled`（全局）、`enable_reload: true`（`[[HANDLE:xxx]]` 把已卸载大消息拉回）
- `compression_recall_config{enabled:false, chunk_size_tokens:3000, chunk_overlap_tokens:500}` → 原文分块归档到会话目录 `compression_recall/`，模型用 `[[COMPRESSION_RECALL: id=…]]` 召回
- `message_summary_offloader_config{protected_tool_names:["read_file","read_pdf"], add_message_threshold_ratio:0.1, ttl_seconds:300, ttl_context_occupancy_ratio:0.5, ttl_message_threshold_ratio:0.05}`
- `session_memory_compressor_config{enabled:false, trigger_context_ratio:0.8, memory.update_trigger_context_ratio:0.1}`
- `dialogue_compressor_config / current_round_compressor_config(keep_recent_messages:4) / round_level_compressor_config(keep_recent_messages:0)`，均 `trigger_context_ratio:0.8`、`min_target_context_ratio:0.1`
- 调试：`enable_context_debug`（落盘最终 messages/tools，注释警告含用户数据，默认关）、`context_debug_dir` / `OPENJIUWEN_CONTEXT_DEBUG_DIR` / `{workspace}/context/{session_id}_context/context_debug/`
- 文档：`docs/zh/上下文压缩.md`

### 6.6 动态修改与生效范围
- 配置面板 RPC（`jiuwenswarm/common/config_panel/models_handlers.py`，`register_models_handlers(...)`）：`models.list` / `models.replace_all` / `config.validate_model` / `models.validate`；TUI 侧同契约 `jiuwenswarm/common/config_panel/tui_models_handlers.py`（`reasoning_level` 读写 962/1045/1142）。
  - 保存时 `validate_reasoning_level_for_model(raw_level=…, …)`（588）+ `_serialize_reasoning_level`（98）/ `reasoning_level_display`（110）；`"reasoning_level" in params` 才写（205-206）→ **未下发即保持原值**。
  - 前端可传 `reasoning_level` 覆盖配置值（155 注释）。
  - 「测试连接」以持久化条目为基线（`resolve_persisted_mcc_for_validate` 210，含 `custom_headers/ssl_cert` 等 write-only 字段）。
- 会话内切换：`ReqMethod.COMMAND_MODEL = "/model"` → `jiuwenswarm/server/handlers/commands.py:1001 handle_command_model`；per-session 选择存 `jiuwenswarm/server/runtime/session/model_selection_store.py`(7)；`jiuwenswarm/server/runtime/{model_config_service,model_selection,model_switch_watcher,model_connection_test,model_routing_registry}.py`。
- 上下文/压缩命令：`handle_command_context` 747（`/context`）、`handle_command_compact` 491（`/compact`）、`handle_command_compact_partial` 698（`/compact-partial`）、`handle_command_status` 1151、`handle_command_resume` 1097。
- 热重载：`jiuwenswarm/server/handlers/ops.py::handle_agent_reload_config`、`handle_config_cache_clear`、`handle_agent_prewarm_sync`；`ReqMethod.MODELS_REFRESH_STATUS = "models.refresh_status"`；KV 亲和 `update_kv_cache_affinity_enabled_in_config` + `kv_cache_affinity_config.enable_kv_cache_affinity`（`config.yaml:700-702`）+ `jiuwenswarm/server/runtime/session/kv_cache/*`。
- **生效范围**：模型配置改动 → `AgentManager._evict_stale_llm_clients(effective_config)` 关掉凭据已变的旧连接（§2.1）；`/model` 是**会话级**；`context_window_tokens` 是**进程/配置级**（改后需 reload）；`reasoning_level` 挂在模型条目上，随模型选中生效，Team 成员经 `_sanitize_team_member_model` 单独映射。
- 企业版按用户下发模型：`jiuwenswarm/enterprise/config/effective/apply_models.py`。

### 6.7 缺口 / 风险
1. `context_window` **不发厂商**（core 缺正式字段，出口 pop）→ 服务端不会因超窗被厂商拒；本仓里 `context_length_exceeded` 只出现在 `jiuwenswarm/extensions/video_duplex/tests/backend/test_video_live.py`，**没有主链路的超窗错误处理**。
2. 无 per-user / per-session 的上下文窗口覆盖（只有全局 + 每模型 + 模型名映射）。
3. 组级流式超时「存了但没消费」（`config.yaml:481` 注释）→ 配了不生效，容易误判。
4. 无输出 token 预算顶层键；`swarmflow_budget` 只覆盖 swarmflow，不覆盖主对话。

---

## 7. 模型主动提问：表单 schema 与恢复流程

### 结论
**单选 / 多选 / 自由文本混合是支持的，schema 明确且有限（最多 4 题、每题 2-4 选项、`Other` 保留）；恢复走 core 的 `InteractiveInput`，按 `request_id` 对齐。缺的是：字段类型只有选项+文本、没有表单版本化、没有超时/离线恢复策略。**

### 7.1 出站事件与 schema
`jiuwenswarm/agents/harness/common/rails/interrupt/interrupt_helpers.py::convert_interactions_to_ask_user_question()`（1176）产出：
```jsonc
{
  "event_type": "chat.ask_user_question",
  "request_id": "<correlation key>",
  "questions": [ /* 见下 */ ],
  "source": "ask_user_interrupt" | "permission_interrupt" | "confirm_interrupt",
  ... /* _interrupt_identity(value_obj) 附加身份字段 */
}
```
- 三种来源（1183-1186 注释）：AskUserRail（value 有 `questions` 或明确 ask_user 的 plain query）→ `ask_user_interrupt`；PermissionRail（无 `questions`）→ `permission_interrupt`；ConfirmInterruptRail（控制类工具确认）→ `confirm_interrupt`。
- 单题投影：`_build_multi_questions`（1240）、`_build_plain_ask_user_question`（1034，`multi_select: False` 固定，1056）、权限卡片 `build_verified_permission_ask_user_question`（1338）。
- 每题字段（1522、1400 注释）：`question`、`options`、`multi_select`；`options` 元素含 `label`（+ `description`/`preview`，见 `AskUserQuestionReference`）。
- 另一条 ask 事件通道：`harness.activate_interaction`（id 键是 `interaction_id`，`jiuwenswarm/runtime/interaction.py:31`）。

### 7.2 契约与上限
`jiuwenswarm/agents/harness/common/rails/ask_user_contract.py`：`MAX_STRUCTURED_QUESTIONS = 4`（5）。
`jiuwenswarm/agents/harness/common/rails/permissions/root_ask_user.py`（274 行，root Host 持有的紧凑续跑）：
- `ASK_USER_TOOL_NAME = "ask_user"`（27）、`ASK_USER_CONTINUATION_METADATA_KEY = "jiuwenswarm_root_ask_user_v1"`（28）、`ASK_USER_RESUME_DTO_KEY = "jiuwenswarm.root_ask_user_resume.v1"`（29）、`_MAX_ANSWERS = 8`（30）。
- `AskUserQuestionReference{question, options, multi_select}`（33）带 `to_mapping/from_mapping`，`from_mapping` 要求键集合**恰好等于** `{question, options, multi_select}`，否则 `ValueError("invalid ask-user question")`（48-53）→ 严格 schema。
- `RootAskUserContinuation{tool_call_id, context, questions}`（70）、`RootAskUserResume{continuation, clarifications, capacity_reason}`（77）。
- `put_ask_user_resume_in_inputs()`（84）把 resume DTO 塞进 `run.context.extra`；`build_ask_user_metadata()`（100）；`prepare_ask_user_resume` / `apply_ask_user_resume` / `ask_user_continuation`。
- 答案校验 `_answer_values(value, *, multi_select)`（252）：非多选必须**恰好 1 个**；最多 8 个；每项非空字符串；选项标签去重且 **`"Other"` 保留字**（249）。

### 7.3 恢复流程（入站）
`jiuwenswarm/agents/harness/common/rails/ask_user_contract.py::build_ask_user_interactive_input(request_id, answers)`（10）：
- 入参 `answers: [{question, selected_options[], custom_input}]`；
- 归并规则：`selected_options` 逐个 strip 去空；有 `custom_input` 时把 `"Other"` 当 UI 占位丢掉、保留真实选项再追加用户文本（`[*normal_options, custom_input]`）；**只选 `Other` 且没文本 = 不完整答案**（40-42 注释引用 issue #2330）→ 值为 `""`；
- 没有 `question` 文本但有答案 → 落到 `answers_dict["__free_text__"]`（63-64）→ **纯文本回答也能恢复**；
- 最终 `interactive_input.update(request_id, {"answers": answers_dict})`（65）→ `[外部 core] InteractiveInput`。
  **core 侧 schema 已闭环**（`openjiuwen/core/session/interaction/interactive_input.py:18`）：
  ```python
  class InteractiveInput(RestoreFieldDefaultsOnUnpickle, BaseModel):
      user_inputs: Dict[str, Any] = {}      # key = interaction id（workflow 节点 id 或暂停的 tool call id）
      raw_inputs: Any = None                # 不绑定 id，用于第一次交互
      interrupt_answers: Dict[str, Any] = {} # key = InterruptRequest.interrupt_id
  ```
  - `update(node_id, value)`（51）按 interaction id 回答；`answer(interrupt_id, value)`（60）按 `InterruptRequest.interrupt_id` 回答，注释明确「**只有 raising 该 interrupt 的那个 rail（或 client tool）会收到这个值**」。jiuwenswarm 的 ask_user 走的是 `update(request_id, …)` 这条，不是 `answer()`。
  - `_check_addressed_answer`（75）：`raw_inputs` 非空时再按 id 回答 → `StatusCode.INTERACTION_INPUT_INVALID`（"raw_inputs existed, update is invalid"）；`interrupt_id` 必须是非空字符串（67-71）。构造时 `raw_inputs=None` 显式传入也会 `INTERACTION_INPUT_INVALID`（36-37）。
  - `RestoreFieldDefaultsOnUnpickle` 基类 + `_omit_empty_interrupt_answers` 序列化器（43-49，空则从 payload 里删掉该键以保持旧格式）→ **这就是 checkpoint 反序列化后恢复待答中断的机制**，与 `runtime/service.py:3765` 的注释对得上。
`jiuwenswarm/runtime/interaction.py`（237 行，transport-neutral 输入）：
- `USER_INTERACTION_EVENTS = {"chat.ask_user_question", "chat.client_tool_request"}`（26）；`_INTERACTION_ID_KEYS`（29-32）。
- `interaction_ids / asked_interaction_ids`（35/42）；`InteractionAnswerError(code="INTERACTION_BAD_REQUEST", retryable=False)`（47）；`validate_answer_items`（69，必须是非空对象序列）。
- `InteractionAnswerInput.to_agent_request()`（约 217-228）：`req_method = CHAT_SEND if resumes_interrupted_turn else CHAT_ANSWER`，`is_stream = resumes_interrupted_turn` → **恢复被中断的一轮时走 `chat.send` 流式，普通回答走 `chat.answer`**。
- 还携带 `trusted_dirs`、`run_permissions`、`evolution_meta`（212-216）。
- 依赖 `[外部仓] agent_gateway.common.interrupt_resume.INTERRUPT_RESUME_SOURCES`（12 行 import）。**已闭环**（`agent-gateway/agent_gateway/common/interrupt_resume.py`，45 行）：
  ```python
  CLIENT_TOOL_INTERRUPT_SOURCE = "client_tool_interrupt"
  INTERRUPT_RESUME_SOURCES = frozenset({
      "permission_interrupt", "confirm_interrupt", "ask_user_interrupt",
      "evolution_interrupt", "client_tool_interrupt",
  })
  ```
  即**恢复来源白名单共 5 个**（本仓 `interrupt_helpers.py` 只产出前三个，`evolution_interrupt` 与 `client_tool_interrupt` 由 core / 前端工具侧产生）。`is_client_tool_resume(params)` 单独判定 client_tool；`answers_paused_calls(params)` 要求 `source ∈ INTERRUPT_RESUME_SOURCES` **且** `answers` 是非空 list **且** `request_id` 非空（`client_tool_interrupt` 例外，不需要 `request_id`）→ 三个条件缺一即不视为恢复请求。
- checkpoint 侧：`jiuwenswarm/runtime/service.py:3765` 注释「checkpoint restores the pending interrupt, and the InteractiveInput …」；`_is_interrupt_resume_request` 3730、`_should_admit_interrupt_resume` 3859、`resumed_interactions` 4252、关联键 4078。
- 会话输入意图：`ReqMethod.SESSION_INPUT_INTENT = "session.input.intent"` → `jiuwenswarm/server/handlers/session.py::handle_session_input_intent`；`jiuwenswarm/runtime/session_input.py`。
- pause/resume 语义在 `chat.cancel` 的 `intent`（§2.4）：pause 在 checkpoint 阻塞、resume 解除阻塞，**不掐流式任务**。
- 已知 bugfix：`jiuwenswarm/server/runtime/agent_adapter/B_20261030_2020_pause-refires-consumed-popups.md`（pause 时重复弹出已消费的问题）。
- 群聊侧提问：`jiuwenswarm/server/runtime/group_questions.py`(7992 B)；语音待中断：`jiuwenswarm/server/runtime/voice_pending_interrupt.py`、`VOICE_TASK_PENDING_INTERRUPT`、`VOICE_SESSION_RECEIPTS`。
- 富 UI 表单：A2UI（`docs/zh/A2UI.md`、`A2UI-IM-channel-analysis.md`、`A2UI-web-only-pr-record.md`；依赖 `a2ui-agent-sdk==0.2.1`；`jiuwenswarm/server/control/a2ui_config.py`；校验脚本 `scripts/verify_a2ui_bundle.py`）。
- Swarmflow HITL：`human` / `human_session` 节点（README、`.agents/agent-docs/designs/archive/SDD-0009-…`）。
- 挂载点：`StructuredAskUserRail` 属于**固定挂载** Rails（`config.yaml:1688` 列表），不需要用户开；权限审批 `ReqMethod.PERMISSIONS_CONFIG="permissions.config"` → `jiuwenswarm/server/handlers/permissions.py::handle_permissions_config`，`JIUWENMATE_PERMISSION_DECIDE`。

### 7.4 缺口 / 风险
1. **字段类型只有「选项 + 多选 + 自由文本」**：没有数字/日期/文件/级联/条件显示等表单原语；复杂表单只能靠 A2UI 富 UI 通道，两者 schema 不统一。
2. **无表单版本化 / 无 schema 协商**：`ASK_USER_RESUME_DTO_KEY` 里带 `v1`，但没有对不兼容 schema 的降级路径。
3. **无提问超时/过期策略**：`AgentManager` 的 LRU 默认不淘汰正是为了「`ask_user_question` 长阻塞不被误淘汰」→ 反过来说，**没人管这个挂多久**；没有 TTL、没有「过期自动按默认答案继续」。
4. ~~跨进程/跨节点恢复依赖 checkpoint + `[外部仓] interrupt_resume`，本仓看不到端到端保证~~ → **两侧都已看到**：core 侧 `InteractiveInput(RestoreFieldDefaultsOnUnpickle)` + `interrupt_answers`（见 §7.3），Gateway 侧 `INTERRUPT_RESUME_SOURCES` 五值白名单 + `answers_paused_calls()` 三条件判定。**仍然缺的是**：跨节点时 checkpoint 文件在本地 `get_checkpoint_dir()`（`agent_gateway/common/paths.py`），没有共享/迁移机制 → 换节点后待答中断找不回来。

---

## 8. 文件读写 / 上传下载 / 搜索：API 完整度、大文件、流式、错误码

### 结论
**RPC 面很全（含分块读、断点续传语义、workspace 树/预览/用量/删除、git 全套）。下载侧 HTTP Range/206 有实现；但「分片上传协议」不存在（OBS 是单次 PUT），三处大小上限互不一致（OBS 50 MiB / 浏览器 WS 150 MiB / workspace 下载 100 MiB），错误码在 HTTP 侧有码表、WS/RPC 侧没有。**

### 8.1 RPC 清单（权威来源 = 各 adapter 的 `methods` frozenset + `dispatch.py`）
**文件/工作空间域**（`jiuwenswarm/server/runtime/gateway_adapter/workspace_file_adapter.py:109-127`，`class WorkspaceFileAdapter(GatewayAdapter)` 106）——15 个方法，逐条来自源码：
```
media.persist / media.discard / document.persist / document.formats
path.select_directory / path.select_files
file.import_url / file.upload_chunk / file.download_verified_chunk / file.read_upload_chunk
im.file_persist
workspace.tree / workspace.preview / workspace.usage / workspace.entries_delete
```
`dispatch.py:271-278` 把 `FILE_*` 与 `WORKSPACE_*` 全部路由到 `handle_gateway_adapter`；注释（`dispatch.py:266`）：**「文件 HTTP 与 WebSocket 复用同一个 WorkspaceFileAdapter」**。
**该 adapter 的 `handle()`（129-138）先做租户配额绑定**：`TenantAgentPool.extract_triple(request)` → `bind_quota_identity(group_id, bot_id, user_id)` → `try: self._handle(request) finally: reset_quota_identity(tokens)`。→ **工作空间配额是按 (group,bot,user) 三元组强制的**，这是 §1.3 / §5.2 里最硬的一条隔离证据。

**项目 / git 域**（`jiuwenswarm/server/runtime/gateway_adapter/project_adapter.py:1265-1291`）：
```
project.{list,info,create,rename,pin,remove,restore,lifecycle,
         pinned_sessions,get_sessions,get_cron_sessions,cron_resolve_binding}
project.git.{status,probe,init,switch_branch,create_branch,commit,push,
             diff_status,turn_diff_list,turn_diff,discard_turn_changes,redo_turn_changes}
```
**会话资产**：`SESSION_ASSETS_LIST` 等由 agent adapter 处理（`jiuwenswarm/server/runtime/agent_adapter/interface.py:1401 → handle_session_assets_list`，实现 `jiuwenswarm/server/runtime/session_assets.py:146`），**不在** gateway adapter 注册表里。
**群文件**：`GROUP_FILE_RENAME / GROUP_FILE_DELETE / GROUP_HISTORY_CLEAR` 是本仓用 `_ensure_req_method` 往外部 `ReqMethod` 上补的成员（`jiuwenswarm/common/schema/message.py:73-75`）。
**HTTP 入口**：`jiuwenswarm/server/handlers/gateway_adapter_http.py::handle_gateway_adapter`（34）从 `ctx.services.adapter_registry` 取 adapter；取不到回 `{"code":"HTTP_METHOD_NOT_HANDLED"}`（49）；查询串里的 `true/false` 会被 `_coerce_bool_params`（21）还原成布尔（`include_hidden`、`pinned`）。

### 8.2 大小限制与流式常量
`workspace_file_adapter.py`：
- `_HIDDEN_PREFIXES = (".", "_")`（列表过滤）
- `_SELECT_FILES_LIMIT = 200`
- `_SELECT_IMAGE_MAX_BYTES = 4 * 1024 * 1024` —— 注释：base64 必须留在全局 **8 MiB WS 帧上限**之内
- `_VERIFIED_DOWNLOAD_CHUNK_MAX_BYTES = 512 * 1024`
- `_resolve_within_workspace()` 强制注入的 workspace root 内（决策 D3）

`jiuwenswarm/server/runtime/attachments/upload_storage.py`（152 行）：
- `UPLOAD_READ_CHUNK_BYTES = 512*1024`
- `read_session_upload_chunk(session_id, raw_path, offset, limit) -> {data_base64, offset, chunk_size, size, eof, name, mime_type}` —— **这就是断点续传的读块语义**（offset/limit + eof）
- 安全：descriptor-anchored `os.open`（`O_NOFOLLOW|O_DIRECTORY`）、要求 `st_nlink == 1` 的 regular file、`MAX_FILE_BYTES` 上限（32 行 import 自 `jiuwenswarm.common.obs_files`）
  **`MAX_FILE_BYTES = 50 * 1024 * 1024`（50 MiB）** —— 权威定义在 `[外部仓] agent_gateway/common/obs_files.py:25`，jiuwenswarm 侧 `jiuwenswarm/common/obs_files.py` 是同名镜像。同文件 `DOWNLOAD_TTL_SECONDS = 7*24*60*60`（26）。校验点：`download_url_bytes` 读 `MAX_FILE_BYTES + 1` 字节，超限 → `ValueError("file must contain 1..50 MiB")`。
- 错误：`ValueError("invalid session_id")`、`ValueError("invalid offset or limit")`、`PermissionError("path outside tenant session uploads")`、`PermissionError("upload must be a single-link regular file")`、`ValueError("invalid upload size or offset")`
- 落盘：`unique_upload_path`（`stem-N`，最多 999，再 `-overflow`）、`atomic_write_unique`（`O_CREAT|O_EXCL|O_WRONLY`，`dir_fd` 下 mode `0o600` 否则 `0o644`，写前配额检查，`FileExistsError("no free upload slot under …")`）

其它上限：`WorkspaceService.max_download_bytes = 100 MiB`、preview `65536 B`（§5.2）；skill zip `50 MiB`（`jiuwenswarm/server/runtime/skill/skill_manager.py:162,5596,5643`）；skill sync `60 MiB`（`skill_sync_http.py:59,405,410`）；`jiuwenswarm/server/wire_truncate.py` 做 wire 载荷截断。

### 8.3 附件与会话资产
`jiuwenswarm/server/runtime/attachments/`：`media_attachments.py`(1064) `normalize_chat_media_attachments`、`discard_session_upload`；`document_attachments.py`(604) `persist_and_parse_documents`、`forbidden_formats`；`steer_images.py`；`obs_attachments.py`(94)（OBS → `[外部仓]`）；`client_attachments/`。
`jiuwenswarm/server/runtime/session_assets.py`(241)：`SESSION_ASSETS_LIST/REMOVE`。
`[外部仓] agent_gateway.common.attachment_safety` 提供 `image_suffix_for_mime / safe_session_dirname / safe_upload_filename`（`steer_images.py:17`、`upload_storage.py:22`）→ **文件名/目录名净化在外部**。
**已闭环**（`agent-gateway/agent_gateway/common/attachment_safety.py`，197 行）：
- `_UNSAFE_FILENAME_CHARS = re.compile(r'[\x00-\x1f\x7f<>:"/\\|?*]+')`；**单独剥离 BIDI 控制字符**（防 `evil‮gnp.md` 这类双向文本伪装扩展名）；`_WINDOWS_RESERVED_NAMES`（`CON/PRN/AUX/NUL/COM1-9/LPT1-9`）。
- `_MAX_FILENAME_BYTES = 180`（**按字节不是字符**，中文文件名会先被截）、`_MAX_SESSION_ID_CHARS = 120`、`_UNSAFE_SESSION_CHARS = re.compile(r"[^A-Za-z0-9._-]+")`；`safe_session_dirname` 额外挡 `.` / `..` / 引号包裹的形式。
- `FORBIDDEN_DOCUMENT_EXTENSIONS` = **30 项**可执行/包扩展名黑名单（`.exe .dll .msi .scr .bat .cmd .ps1 .vbs .wsf .hta .jar .lnk .bin .so .dylib .app .dmg .pkg .command .scpt .scptd .workflow .xpc .bundle .framework .kext .prefpane .saver .component`）。
- 另有 `strip_data_uri_prefix`、`ensure_image_upload_filename`、`image_suffix_for_mime`（只认 `png/jpeg/webp/gif`）。
- **注意**：这是「文档/图片上传」的黑名单，不是全量类型白名单；`.html/.svg/.docm` 之类仍可过。

### 8.4 搜索（**注意：搜索是 agent 工具，不是前端 RPC**）
本仓 `server/` 下**不存在** `PROJECT_FILE_SEARCH` / `PROJECT_SEARCH_IN_FILES` 这类方法名（全仓 grep 无命中）。文件与内容检索能力挂在 **agent 工具**上：
`jiuwenswarm/agents/harness/common/rails/permissions/tool_capabilities.py:67-120` 的工具能力表里含
`read_file / read_pdf / write_file / edit_file / read_text_file / write_text_file / glob_file_search / glob / list_dir / list_files / grep / search_replace / upload_file / upload_photo`；
另有 `tool_search`（渐进式工具发现，110 行）、`recall_compressed_context`（111）、`search_skill`（116）。
权限侧按路径判定写操作：`_is_path_write_tool`（277，`side_effects = {"write_local"}`），写类工具清单 602-606。
→ **前端要「在文件里搜内容」只能靠 agent 调 `grep`/`glob`，没有同步的搜索 RPC**（无搜索结果分页、无超时控制）。
会话历史检索另有 skill：`jiuwenswarm/resources/agent/workspace/skills/cross-channel-history-retrieval/scripts/search_history.py`。

### 8.5 HTTP 通道与大文件
- `jiuwenswarm/server/agent_http_routes.py` / `agent_http_server.py` / `http_chat.py` / `http_invoke.py`：受认证 HTTP bridge（大文件传输走 HTTP 而非 WS 帧）。
- `local_api_server/server.py`（43 KB）：极简 ASGI 风格路由，只看到 `if path == "/health"`（802）与 `if not path.startswith("/v1/")`（806）→ 一个 `/v1/*` 本地 HTTP API（同目录有 `message_log.jsonl`、`proxy_trace_log.jsonl`、`token_usage_log.jsonl`、`usage_summary.json` 运行日志）。
- **Range / 断点续传（下载侧）已在外部仓闭环**：`[外部仓] agent_gateway/channel_manager/web/container_file_http.py:428 _parse_single_byte_range(range_header, file_size)` —— **只接受单个 `bytes=` 区间，带逗号（multipart range）直接拒绝**；`541` 起是共享 HEAD/Range 的流式响应：命中区间 → **HTTP 206 + `Content-Range: bytes start-end/total` + `Accept-Ranges: bytes`**（580-582），无法解析 → **416 + `Content-Range: bytes */total` + `Content-Length: 0`**（553-559），无 Range → 200 + `Content-Length`（574-575），响应体是 `StreamingResponse`（612/902）。本仓的 bugfix 记录：`.agents/agent-docs/bugfix/jiuwenswarm/channels/web/B_20261103_1112_raw-file-range-seek.md`。
- **Gateway 侧真正的文件 HTTP 路由表**（`[外部仓] agent_gateway/channel_manager/web/container_file_http.py`，`FILE_API_PREFIX = "/file-api"`（25），挂载条件「channel 有 AgentOSRouterClient 后端」（621），**与 `/ws` 同端口**）：
  ```
  GET  /file-api/list-files        GET /file-api/list-markdown
  GET  /file-api/raw-file          GET|HEAD /file-api/download
  GET  /file-api/file-content      POST /file-api/file-content
  POST /file-api/upload            POST  /file-api/mkdir
  ```
  上限：`MAX_UPLOAD_COUNT = 20`（26，单请求最多 20 个文件，超出 → `too many files (max 20)`）；`_VERIFIED_DOWNLOAD_CHUNK_MAX_BYTES = 512*1024`（30）。
  **身份绑定**：`_resolve_user_id`（208）的取值链是「显式参数 → `x-user-id` header → token 里的 username」，再与 IAM token 身份比对；不一致 → `_user_mismatch_response()`（182）返回 `code=USER_MISMATCH`、message「user_id 与 token 不匹配」（403）。`_bind_file_api_session`（234）把 session 绑到请求上。
- **下载令牌**（`[外部仓] agent_gateway/common/file_download_token.py`）：`Base64URL(payload_json) + "." + Hex(HMAC-SHA256)`，密钥 `_load_or_create_secret()` 落盘共享（**跨进程：AgentServer / Gateway / app_web.py 共用**，53-83）；`_DEFAULT_EXPIRES_SECONDS = 600`（41）；普通下载 payload `{path, sid, exp?}`（**省略 `exp` 即永不过期**，用于 `send_file_to_user` 交付产物）；skill 正文图片 payload 是 `{purpose:"skill_content_image", name, version, relative_path, exp, sid}`（无 `path`）；verified asset 令牌 kind = `verified_asset_v1`（`container_file_http.py:28`），`exp = asset.expires_at`（`file_download_token.py:119-138`）。单例 `WebFileDownloadManager.get_instance()`（104）。
- **WS 帧上限（更正早期说法）**（`[外部仓] agent_gateway/common/ws_limits.py`，11 行）：
  ```python
  AGENT_WS_MAX_MESSAGE_BYTES = 8 * 2**20    # Gateway ↔ AgentServer 内部链路，注释「no large file base64」
  AGENT_WS_SEND_BUDGET_BYTES = 6 * 2**20
  WEB_WS_MAX_MESSAGE_BYTES   = 150 * 2**20  # 浏览器 ↔ WebChannel（web_connect.py:894 ws_max_size=…）
  ```
  → **浏览器上传不是 8 MiB 而是 150 MiB**（100 MiB 文件 base64 后约 133.3 MiB，正好卡得下）；8 MiB 只约束 Gateway 与 AgentServer 之间的内部链路，所以大文件必须走 `/file-api` HTTP 或 OBS，不能塞进 WS。
- 其它 Gateway 侧尺寸闸（同一仓，便于容量规划）：`web/media/normalizer.py:24-25` 图片 `10 MiB` / 最多 `8` 张；`web/file_picker.py:45` `10 MiB`；`web/task_asr.py:25` 语音 `25 MiB`；`web/personal_share_api.py:32` 并发上传任务 `4`；`web/trajectory_http.py:49,58` 归档 spool `16 MiB`、帧分页 `2000`；`web/diagnosis_http.py:37` 请求体 `64 KiB`；`web_http_routes.py:497` 某 HTTP body `50 MiB`；`im_platforms/xiaoyi/xiaoyi_connect.py:1514` 媒体下载 `30_000_000 B / 60_000 ms`；TUI/HarmonyOS 侧 `1 MiB` / `2 MiB`。

### 8.6 错误码现状（不统一）
| 来源 | 风格 |
|---|---|
| `WorkspaceError(code, message, details)` | `BAD_REQUEST / FORBIDDEN / NOT_FOUND / INTERNAL_ERROR`（企业 workspace） |
| `build_error_response(request, str(exc), code="INTERNAL_ERROR")` | `gateway_adapter/base.py`，把异常字符串直接塞进 message |
| `upload_storage` | 裸 `ValueError` / `PermissionError` / `FileExistsError` + 英文文案 |
| MCP | `CliConnectError(code, message, *, runtime, install_cmd)`、`McpRegistryError(code, message)` |
| 交互 | `InteractionAnswerError(code="INTERACTION_BAD_REQUEST", retryable=False)` |
| 取消 | `chat.interrupt_result {success:false, error:"…"}` |
| **Gateway `/file-api` HTTP** | **有一张 code → HTTP status 映射表**：`_status_for_transfer_code(code)`（`[外部仓] agent_gateway/channel_manager/web/container_file_http.py:95-122`）→ 400 `BAD_REQUEST/INVALID_PATH/MISSING_FILE_PATH/MISSING_SESSION_ID/MISSING_FILE_CONTENT/ONLY_MARKDOWN_SUPPORTED/FORBIDDEN_DIR`；401 `UNAUTHORIZED/AUTH_REQUIRED`；403 `FORBIDDEN/FORBIDDEN_EXTENSION/FORBIDDEN_PATH/USER_MISMATCH`；404 `NOT_FOUND/INSTANCE_NOT_FOUND/FILE_NOT_FOUND`；**413 `PAYLOAD_TOO_LARGE/FILE_TOO_LARGE`**；501 `LIST_NOT_SUPPORTED/NOT_IMPLEMENTED`；其余 **502**。响应体 `{error, code?}`（`_error_json`，125） |
| core（openjiuwen） | `StatusCode` 枚举 + `build_error(StatusCode.X, **kwargs)`，如 `RESOURCE_MCP_SERVER_CONNECTION_ERROR / _REMOVE_ERROR / _REFRESH_ERROR`、`INTERACTION_INPUT_INVALID`（`openjiuwen/core/common/exception/codes.py`） |
→ **HTTP 文件通道有码表（且能映射到状态码），WS/RPC 通道没有**：`WorkspaceError` 的四值、裸异常、`build_error_response` 的字符串塞 message 三种风格并存；**没有全局错误码表、没有 `retryable` 一致性、message 混中英文**。集成方在 WS 侧只能靠字符串匹配。

### 8.7 对象存储（OBS / MinIO）交付路径 —— 已在外部仓闭环
`[外部仓] agent_gateway/common/obs_files.py`（263 行）：
- 常量：`MAX_FILE_BYTES = 50 * 1024 * 1024`（25）、`DOWNLOAD_TTL_SECONDS = 7*24*60*60`（26）。
- 装配：`AttachmentStore.from_env()` 需要 `OBS_URL / OBS_ACCESS_KEY / OBS_SECRET_KEY / OBS_BUCKET`（默认桶 `jiuwenswarm`），`OBS_SECURE` 默认 **false**；`_obs_http_client()` 认 `JIUWENSWARM_SSL_VERIFY`，关闭时打警告「on-path attackers can tamper with OBS uploads/downloads」；`Timeout(connect=5, read=60)`。→ **默认明文 HTTP 到 MinIO，生产必须显式开 `OBS_SECURE`。**
- **对象键与隔离前缀**：
  - 上传：`object_prefix(group_id, bot_id, user_id)` → `uploads/{sha256(json([group_id, bot_id, user_id]))}/`，三者缺一 → `ValueError("user_id, group_id and bot_id are required")`。**按用户三元组隔离，前缀是哈希，猜不到别人的目录**（但桶级策略没做，见缺口）。
  - 下载：`download_object_key(tenant_key, asset_id, filename)` → `downloads/{sha256(tenant_key)}/{asset_id}/{filename}`，`asset_id` 必须匹配 `[0-9a-f]{32}`（防注入）。
- 预签名 URL：**上传 10 分钟、下载 7 天**（`DOWNLOAD_TTL_SECONDS`）。
- **上传是单次 `put_object` / 单个预签名 PUT —— 没有 multipart / 分片续传协议**，所以 50 MiB 就是上传的实际上限。
- 校验：`upload_local_file` 返回 `etag` + metadata `{"sha256", "session-id"}`；`download_to_local` 校验 session 与 digest（见 §8.8 第 3 条）。
- 与 §1 的租户键契约同源：`tenant_key` 由 `[外部仓] agent_gateway/common/tenant_key.py:build_tenant_key()` 生成（`md5("{group_id}::{bot_id}::{user_id}")`，`_TENANT_KEY_SEP = "::"`，`normalize_tenant_key_part(value, default="default")`）；docstring 明写 jiuwenswarm 的 `enterprise/tenant/tenant_paths.py` 是 re-export，「**server/gateway 两侧恒为同一实现**」→ §1.3 的 tenant_key 算法权威定义在 Gateway 仓。

### 8.8 缺口 / 风险
1. **没有分片上传协议**：`FILE_UPLOAD_CHUNK` / `FILE_READ_UPLOAD_CHUNK` 是「服务端从已落盘的会话上传目录按 offset 读块」，不是 `initiate → part → complete` 的 multipart 会话；**真正的断点续传（上传侧）需要客户端自己整文件落盘**，超大文件（>OBS 上限）没有分片合并路径。
2. ~~`MAX_FILE_BYTES` 在外部仓看不到数值~~ → **已闭环：`50 MiB`**（`agent_gateway/common/obs_files.py:25`）。**但由此产生新的硬约束**：OBS 附件通道上限 50 MiB，而浏览器 WS 帧上限 150 MiB、`WorkspaceService.max_download_bytes` 100 MiB —— **三个上限互不一致**，走哪条路决定能传多大文件，文档里没有统一说明。
3. ~~无 checksum / ETag~~ → **OBS 交付路径其实有**：`upload_local_file` 返回 `etag` 并写 metadata `{"sha256": <digest>, "session-id": <sid>}`；`download_to_local` 先校验 `x-amz-meta-session-id`（不符 → `ValueError("download session mismatch")`）再重算 sha256 与 metadata 比对（不符 → `ValueError("downloaded file does not match upload digest")`）。**缺的是**：`/file-api` 的本地文件通道没有条件请求（`ETag`/`If-Range`/`If-None-Match`），也没有把 sha256 暴露给调用方做校验；下载令牌本身不含内容摘要。
4. 无目录级批量下载（zip/tar）接口；`WORKSPACE_ENTRIES_DELETE` 有深度/条目上限（32/5000）但没有异步任务化。
5. 错误码不统一（见 8.6）。
6. **没有搜索 RPC**：文件名/内容检索只有 agent 工具（`grep` / `glob_file_search` / `list_dir`），前端要做「项目内搜索」框只能驱动一次 agent 调用，拿不到同步、可分页、可取消的搜索接口。
7. `SESSION_ASSETS_*` 走 agent adapter、`workspace.*`/`file.*` 走 gateway adapter、`project.git.*` 走 project adapter —— **三套注册路径**，`HTTP_METHOD_NOT_HANDLED`（`gateway_adapter_http.py:49`）只覆盖 HTTP 侧，方法是否可从 HTTP 进来取决于 `adapter_registry` 的装配，集成时容易踩「WS 有、HTTP 没有」。

---

## 9. 八项总表

| # | 主题 | 有没有 | 隔离/完整度 | 最关键的缺口 |
|---|---|---|---|---|
| 1 | 多用户并发 | ✅ 有完整多租户 | 逻辑 + 目录级；企业版按用户下发配置；会话映射企业版走 DB | 租户池默认不淘汰（且未暴露容量/TTL 配置键）；无进程/cgroup 硬隔离；`session_map_scope` 默认按 chat 共享**仅限飞书族通道** |
| 2 | 运行时资源回收 | ✅ 取消/释放/关闭完整 | 会话级、agent 级都有 API | **租户级 AgentManager 无空闲回收**；idle 常量散落；只有 Gateway 的 30 天磁盘保留期是按时间回收 |
| 3 | MCP 按用户/agent 隔离 | ✅ 挂载按 Manager 扇出 + **连接按凭据哈希隔离** | `server_id = mcp_{scope}_{name}_{sha256(含已解析凭据的配置)[:16]}` → 凭据不同即不同连接 | 连接池进程级（`ToolMgr`，无 TTL/上限）；**凭据槽按 MCP 名（同名跨租户共用）**；**skill 型 MCP 的 token 进 `os.environ`**（stdio/HTTP 走 `credential_resolver`，不经 env） |
| 4 | 用户画像 | ⚠️ 只有文件 + 记忆引擎 | 无 schema | `USER.md` 空模板；**无删除/导出/遗忘接口**；forbidden 是 prompt 级；`user_id` 默认 `__default__` |
| 5 | 工作空间跨节点 | ⚠️ 靠外部共享存储 | 路径 containment + zone | 锁全是本地 advisory（连 Gateway 单实例锁也是 `portalocker`）；**无跨节点变更通知**；无快照 API |
| 6 | 上下文 / 推理等级 | ✅ 参数名与优先级明确 | 8 档 reasoning + 能力校验（core profile 表 + 协议默认） | `context_window` 已是 core 正式字段但被 core exclude、不发厂商；组级流超时未消费；**无跨请求累计 token 预算**（单模型 `max_tokens` 可用） |
| 7 | 模型主动提问 | ✅ 单/多选/文本混合 | schema 严格但窄；恢复白名单 5 值；core 三字段 `InteractiveInput` | 无复杂字段类型；无表单版本化；**无提问超时/过期恢复**；checkpoint 本地不跨节点 |
| 8 | 文件 API | ✅ RPC 面很全 | 分块读 + **Range/206** + 安全守卫 + **按用户绑定的配额** + OBS 前缀按用户哈希隔离 | **无分片上传协议（OBS 单次 PUT，50 MiB 上限）**；**无搜索 RPC**；三个文件大小上限互不一致；WS/RPC 侧错误码不统一 |

---

## 10. 外部仓补录（三个依赖仓已克隆到钉住的 rev）

### 10.1 已克隆的外部仓（按 `jiuwenswarm/pyproject.toml` 钉的 git rev，浅取）
| 仓 | 来源与 rev | 日期 | 规模 | 顶层结构 |
|---|---|---|---|---|
| `jiuwenswarm` | GitHub `develop`，`--depth 1`，HEAD `c1b9700` | 2026-11-06 | 6910 文件 | LFS 对象未取（`docs/assets/videos/*` 不可读） |
| `agent-gateway` | `gitcode.com/openJiuwen/agent-gateway.git@cff3b1b9c2dd02bf29e0cb331dec12be989bc656`（`pyproject.toml:29,247`） | 2026-11-05「fix(gateway): keep legacy partial model reloads from restarting Gateway」 | 811 文件 | `agent_gateway/ docs/ examples/ scripts/ tests/ pyproject.toml uv.lock pytest.ini run_tests.sh README.zh.md SECURITY.md` |
| `agent-core` | `gitcode.com/openJiuwen/agent-core.git@f3f8f576164cc405c046e43079c8ad00ee17f0c4`（`pyproject.toml:22,102,237`） | 2026-11-05「Merge develop into release integration…」 | 7058 文件 | `openjiuwen/ plugins/ deploy/ docs/ examples/ scripts/ tests/ AGENTS.md CLAUDE.md Makefile pyproject.toml` |
| `agent-protocol` | `gitcode.com/openJiuwen/agent-protocol.git@cbbadde9101db6e3d75e4ef2b9bdb6029fd92088#subdirectory=openjiuwen-protocol`（`pyproject.toml:27`；`a4p` 取同仓 tag `agent-protocol-V0.2.0#subdirectory=A4P`，17） | 2026-11-03「feat(protocol): add E2A envelope traceparent」 | 725 文件 | `A2A/ A4P/ AgentRegistry/ MCP/ openjiuwen-protocol/` |
> 复现命令（gitcode 允许按 SHA 直接 fetch，无需鉴权）：
> `git init <dst>; git remote add origin <url>; git fetch --depth 1 origin <rev>; git -c lfs.fetchexclude='*' checkout -f FETCH_HEAD`
> 入口点对应关系：`jiuwenswarm-gateway = "agent_gateway.app_gateway:main"`（`pyproject.toml:164`，实现 `agent_gateway/app_gateway.py`，184 KB）、`jiuwenswarm-acp = "agent_gateway.channel_manager.protocol.acp.acp_connect:main"`（170）。

### 10.2 原「必须问厂商」清单 —— 逐条闭环结果
| # | 原问题 | 结论 |
|---|---|---|
| 1 | `MAX_FILE_BYTES` 数值 | **50 MiB**（`agent_gateway/common/obs_files.py:25`） |
| 2 | OBS/MinIO 是否分片上传 | **没有 multipart**，单次 `put_object` / 单个预签名 PUT；上传预签名 10 分钟、下载预签名 7 天 |
| 3 | `attachment_safety` 到底做了什么 | 见 §8.3：BIDI 剥离、Windows 保留名、180 **字节**文件名上限、30 项可执行扩展名黑名单 |
| 4 | `interrupt_resume` 白名单 | **5 个 source**，见 §7.3 |
| 5 | Gateway 侧鉴权与租户路由 | `auth/`（IAM token + session owners + APIG + 模型目录）与 `routing/`（`session_map` 企业版走 DB、`GatewayRouteBinding`、`e2a_proxy`）；**`tenant_key` 的权威实现在 `agent_gateway/common/tenant_key.py`** |
| 6 | `models.*` 的 E2A 多用户代理 | **确认存在**：`_register_config_proxy("models.list" / "models.replace_all" / "models.validate" / "config.validate_model", …)`（`agent_gateway/channel_manager/web/app_web_handlers.py:5709-5715`），TUI 侧同名注册（`channel_manager/tui/tui_connect.py:1695-1696`），REST 映射 `GET /models → models.list`（`channel_manager/web/http_routes/mapped.py:88`）。企业版 `models.list` 读 Gateway Manager 的库（`app_web_handlers.py:5654,5672` 注释：经目标 AgentServer 执行时看不到 Gateway 连接，登录送的模型列表要特殊处理）；另有 `openai_account.models.list`（5755）与 `config.validate_model` 的「测试模型可用性」复用（2558） |
| 7 | `Runner.resource_mgr` 的 MCP 连接池 | 见 §3.2：`ToolMgr`，key 只有 `server_id`，**无 TTL、无上限**，回收靠 `remove_tool_server` / `release()` |
| 8 | `ModelRequestConfig.context_window` 是否已成正式字段 | **已经是正式字段**（`openjiuwen/core/foundation/llm/schema/config.py:259`），jiuwenswarm 的 pop 逻辑自动停；core 自己在 `base_model_client.py:623-638` 把它 exclude 掉不发厂商（见 §6.1） |
| 9 | `get_reasoning_capability` 能力表 | 见 §6.2：profile 表命中给精确档位，未命中回落协议默认（advisory） |
| 10 | `InteractiveInput` 的 checkpoint 语义 | 见 §7.3：`RestoreFieldDefaultsOnUnpickle` + `user_inputs/raw_inputs/interrupt_answers` 三字段 |
| 11 | AgentRegistry / `react.a2x_registry` | **开源仓里就有**：`agent-protocol/AgentRegistry/`（A2X Registry），见 §10.3 |
| 12 | 企业版 `get_workspace_dir()` 是否 patch 成租户根 | **没有 patch**，MCP 状态与凭据跨租户共享（见 §3.3） |

### 10.3 A2X Registry（`agent-protocol/AgentRegistry/`）—— 分布式 Team 的前置件，开源可得
- 结构：`a2x_registry/{a2x,auth,backend,cluster,common,heartbeat,register,traditional,vector}/`、`client/`、`ui/`、`docs/`、`scripts/`、`tests/`；`README.md` / `README_forDistributed.md` / `README_forAgentTeam.md`；`pyproject.toml` + `VERSION`。
- 自述能力（README）：注册/注销、按字段查询、整卡覆盖与字段级更新、状态声明 `online/busy/offline`、**基于所有权的权限控制**、**Agent Team 场景的短时预订锁（reservation lease）**、数据集隔离、Embedding 可选、服务类型 `generic / A2A / Skill` 并存。
- 特色检索 A2X：LLM 自动构建**层次化能力目录**（分类树）+ 递归语义导航，成本约 `O(log N)`；README 给的对比数据（1839 服务 / 1714 查询）：A2X Hit Rate 92.59%、7,069 tokens/查询，对比「Traditional（MCP 全量列表）」86.00% 但 **66,568 tokens/查询**，并指出加完整 `inputSchema` 后单次约 200k tokens 会撑爆上下文。
- 与 jiuwenswarm 的接缝：`jiuwenswarm` 侧有 `agents/harness/team/a2x/client/ownership.py`（`fcntl.flock` 的本地 ownership，见 §5.3）与 `config.yaml` 的 `react.a2x_registry` 块 —— **Registry 本体不是内置，需要单独部署**；它解决的是「能力发现」，不解决「文件共享」，两者不能混为一谈。

### 10.4 原「仍未闭环」6 条 → **全部自查闭环**（客户答复口径以此为准）
1. **`ToolMgr` 刷新幂等性** → **已闭环，安全**。core 自带回归测试 `tests/unit_tests/core/runner/test_tool_manager_mcp_dedup.py`（docstring 明写「Without dedup the second caller crashes with *already exist tool*」）钉住「同 `server_id` 重复 `add_tool_server` 必须复用缓存连接与工具卡片」；且 jiuwenswarm **从不调用 `refresh_tool_server`**（全仓 grep 只有注释提及），产品路径是先 remove 再 add。→ 对外结论：**重复挂载安全**；若客户直接调内核 API，建议先 `remove_tool_server` 再 `add_tool_server`。
2. **三个文件上限的官方口径** → **已给出答复口径**（客户版 §8.3 表 + 附录 B）：附件/对象存储通道 **50 MiB**（`MAX_FILE_BYTES`）；浏览器 ↔ WebChannel WS 帧 **150 MiB**；工作空间下载 **100 MiB**；Gateway↔AgentServer 内部链路 **8 MiB / 6 MiB 预算（禁止大文件 base64）**。选型规则：**小文件走 WS、大文件走 `/file-api` HTTP（支持 Range/206）、跨节点大文件走 OBS 预签名直传**。上传是单次 PUT，无 multipart 分片会话。
3. **MCP token 串用** → **结论已更正**（见 §3.2/§3.3/§3.6）：`server_id` 含凭据哈希 ⇒ **连接层不跨租户串 token**；`os.environ` 只承载 **skill 型 MCP** 的 token。残留两条真实风险：凭据槽按 MCP 名（同名共用）、skill token 进程 env 可见 → 均可靠「每租户一进程」规避，已在客户版 §3.7 如实列出。
4. **`session_map_scope` 默认是否产品预期** → **已查实，风险大幅收窄**：`message_handler.py:449-452` 表明 SessionMap **只用于 `feishu` / `feishu_enterprise` 两个通道**（注释解释是为了修 `feishu:None` 键错配导致的「每条消息新建会话丢上下文」）。→ 「群内共享上下文」**仅限飞书群**；Web 与其他 IM 不受影响。答复口径：飞书群需要每人独立上下文时显式配 `per_chat_bot_user`。
5. **checkpoint 跨节点** → **已查实**：`get_checkpoint_dir()` = `get_agent_root_dir()/".checkpoint"`（`jiuwenswarm/common/utils.py:2466`、`agent_gateway/common/paths.py:232` 同构），企业版 `get_agent_root_dir(tenant_key)` **按租户分根** → 租户间隔离 ✅，但**仍是本地磁盘、无共享/迁移机制** → 跨节点恢复待答中断需把该目录纳入共享存储（客户版 §7.4 第 4 条已写）。会话映射侧企业版走 DB 表 `session_map` ✅ 跨实例共享。
6. **Gateway 未逐行读的大模块**（`common/config.py` 117 KB 等）→ **判定为超出本次答复范围**：八项能力所需的 Gateway 事实（会话映射、锁、清理、WS 上限、文件 HTTP、附件安全、下载令牌、中断恢复白名单）已逐条取证闭环；剩余模块属于「客户自定义 channel / 日志脱敏对齐」场景，届时再开一轮。

---

## 附：本次核查用到的权威「清单文件」

| 想知道 | 直接看这个 |
|---|---|
| 有哪些 RPC | `jiuwenswarm/server/dispatch.py`（`HANDLERS` 表）+ `jiuwenswarm/common/schema/message.py`（`_ensure_req_method` 补的外部成员）+ `jiuwenswarm/server/control/methods.py`（控制面/执行面划分） |
| 所有可配键与默认值 | `jiuwenswarm/resources/config.yaml`（1900 行，同时是升级时的键白名单：模板里缺席的键会被 `_deep_merge` 从用户配置删掉） |
| 键的类型强制 | `jiuwenswarm/common/config.py`（3771 行） |
| 设计意图 | `.agents/agent-docs/{INDEX.md,designs/REGISTRY.md,designs/{archive,proposals}/,features/,specs/}`、`docs/zh/*.md` |
| 已知缺陷与修复 | `.agents/agent-docs/bugfix/**`、`jiuwenswarm/**/B_YYYYMMDD_*` 命名的 bugfix 文档 |
| **文件大小上限 / OBS 路径** | `agent_gateway/common/obs_files.py`（`MAX_FILE_BYTES`、`DOWNLOAD_TTL_SECONDS`、前缀与预签名）、`agent_gateway/common/ws_limits.py`（WS 帧）、`agent_gateway/channel_manager/web/container_file_http.py`（`/file-api` 路由 + Range + 状态码映射） |
| **tenant_key 权威算法** | `agent_gateway/common/tenant_key.py`（jiuwenswarm 的 `enterprise/tenant/tenant_paths.py` re-export） |
| **MCP 连接池与重连** | `openjiuwen/core/runner/resources_manager/tool_manager.py`（`ToolMgr`）、`openjiuwen/core/foundation/tool/mcp/client/reconnect.py`（退避与可重试判定） |
| **模型请求字段 / 发厂商的参数** | `openjiuwen/core/foundation/llm/schema/config.py`（`ModelRequestConfig`）、`openjiuwen/core/foundation/llm/model_clients/base_model_client.py:552-642`（`_build_request_params` 的 exclude 表） |
| **推理能力档位** | `openjiuwen/core/foundation/llm/reasoning.py`（`REASONING_PROFILES`、`get_reasoning_capability` / `_catalog` / `get_provider_reasoning_rules`） |
| **中断恢复契约** | `agent_gateway/common/interrupt_resume.py`（5 个 source）、`openjiuwen/core/session/interaction/interactive_input.py`（三字段） |
| **服务发现 / 分布式 Team** | `agent-protocol/AgentRegistry/`（`README.md`、`README_forDistributed.md`、`README_forAgentTeam.md`） |
