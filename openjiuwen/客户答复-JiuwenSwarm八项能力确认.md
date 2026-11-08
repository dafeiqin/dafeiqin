# JiuwenSwarm 八项能力确认答复

**答复方**：华为侧 JiuwenSwarm 团队
**核查对象**：JiuwenSwarm（包名 `workswarm`）v0.2.8，分支 `develop`
**取证基线**（答复中每条结论都对应下列版本，可用源码复核）

| 组件 | 版本基线 |
|---|---|
| JiuwenSwarm（AgentServer + Web/TUI 通道） | `develop` @ `c1b9700`（2026-11-06） |
| Agent Gateway | `agent-gateway` @ `cff3b1b9…`（2026-11-05） |
| Agent 内核 openjiuwen | `agent-core` @ `f3f8f576…`（2026-11-05） |
| 协议 / 注册中心 | `agent-protocol` @ `cbbadde9…`（2026-11-03） |

**答复口径**
- **支持**：开箱可用，无需改造。
- **支持（需配置）**：能力已实现，需要按本文给出的配置项或部署方式打开。
- **需扩展实现**：产品侧未提供，需要集成方在外部补齐（本文给出建议路径）。

**接口阅读约定**：下文代码块只给**接口签名 + 字段含义 + 调用作用**，实现细节省略（`...`）。每节末尾给出源码位置，便于贵方评审时逐条复核。

---

## 一页速览

| # | 客户关注点 | 结论 | 关键条件 |
|---|---|---|---|
| 1 | 多用户并发 | **支持** | 企业版多租户；租户键 = `(group, bot, user)` 三元组；按租户独立工作目录与配置下发 |
| 2 | 运行时资源回收 | **支持** | 取消 / 会话释放 / 关闭链路完整；磁盘侧有 30 天保留期自动清理 |
| 3 | MCP 按用户 / agent 隔离 | **支持（需配置）** | 挂载按 agent 扇出；连接标识含凭据哈希，凭据不同即不同连接；多租户建议每租户一进程 |
| 4 | 用户画像 | **支持（需配置）** | 画像 = Markdown 人设/记忆文件 + 记忆引擎；有编辑与开关接口；多用户必须配记忆隔离标识 |
| 5 | 工作空间跨节点 | **支持（需配置）** | 官方提供 NFS 共享方案 + 分布式 Team（PostgreSQL + pyzmq + 注册中心） |
| 6 | 上下文长度 / 推理等级 | **支持** | 参数名明确、四级优先级、8 档推理 + 按模型能力校验、配置面板可热改 |
| 7 | 模型主动提问 | **支持** | 单选 / 多选 / 自由文本可混排；按 `request_id` 断点恢复，不重跑已完成步骤 |
| 8 | 文件读写 / 上传下载 / 检索 | **支持** | RPC 面完整；分块读 + HTTP Range/206 + 按用户配额 + 对象存储按用户前缀隔离 |

---

## 1. 多用户并发：部署模型、并发机制、资源隔离

### 结论
**支持。** 采用 **Gateway（接入与路由）+ AgentServer（Agent 运行时）** 分离部署。企业版按租户隔离运行时实例与工作目录，并按用户下发模型 / MCP / 权限配置。

### 1.1 部署模型

```
浏览器 / IM(飞书·企微·钉钉·小艺·WhatsApp) / TUI / E2A
        │
   Gateway  ── 鉴权、频道接入、会话路由、对象存储(附件)、附件安全、磁盘保留清理
        │  WebSocket（内部链路）
   AgentServer ── Agent 运行时、会话、工具、MCP、记忆、上下文引擎
```

| 部署形态 | 说明 |
|---|---|
| 容器 | `docker/start.sh` 用 `ROLE=gateway\|agentserver` 区分角色；镜像 `Dockerfile.claw` / `Dockerfile.web` / `Dockerfile.sandbox` |
| 单机多实例 | `~/.jiuwenswarm/instances.yaml`；端口 = base + index×1000；`jiuwenswarm-init --name` / `jiuwenswarm-start --list\|--status\|--stop` |
| 数据目录 | `JIUWENSWARM_DATA_DIR`（默认 `~/.jiuwenswarm`）；多实例各用独立目录 |
| 就绪状态机 | `STARTING → TRANSPORT_READY → CONTROL_READY → RUNTIME_WARMING → AGENT_READY / DEGRADED / DRAINING / FAILED`，前端用 `wait_agent_ready(timeout)` 等待 |

### 1.2 并发机制与租户身份

```python
# ── 租户身份解析（每个请求进入分发前完成）
# 作用：从请求中确定「这是哪个租户的哪次请求」，并绑定到当前执行上下文。
# 取值优先级：请求顶层字段 → E2A 信封 metadata.routing（权威）→ HTTP 头 X-Group-Id/X-Bot-Id/X-User-Id
extract_triple(request) -> tuple[group_id, bot_id, user_id]
extract_tenant_key(request) -> str          # tenant_key = md5(f"{group_id}::{bot_id}::{user_id}")

# ── 租户运行时池（进程内）
# 作用：按 tenant_key 懒创建并缓存该租户的 AgentManager；同一租户的多个会话共享其 Manager。
class TenantAgentPool:
    def get_agent_manager(self, tenant_key) -> AgentManager: ...   # 不存在则创建
    def tenant_scope(self, ...): ...                               # 异步上下文：绑定 TenantContext 后分发
    def touch_manager_if_same(self, ...): ...                      # 刷新 LRU 访问位次（滑动 TTL 用）
    def iter_agent_managers_nowait(self): ...                      # 遍历当前存活租户（监控/巡检用）
    def get_agent_count(self) -> int: ...                          # 当前缓存的租户数
    def cleanup(self) -> None: ...                                 # 释放全部租户运行时

# ── 单个 AgentManager 内部：二级索引
# 作用：同一租户下按 (channel_id, 会话缓存键/模式) 复用 Agent 实例，避免每条消息重建 Agent。
self.agents: dict[channel_id, dict[cache_key|mode, JiuWenSwarm]]
```

> 会话映射（Gateway 侧）：`SessionMap` 决定「一条入站消息属于哪个 agent 会话」。
> - 个人版落盘 `session_map.json`；**企业版注入 `SessionMapRepository` 后读写数据库表 `session_map`（进程内缓存为 write-through）** → 多个 AgentServer 实例共享同一份会话映射，这是水平扩展的基础。
> - 映射粒度由 `gateway.session_map_scope` 决定：`per_chat_bot`（默认）或 `per_chat_bot_user`（每个用户一条独立会话）。

### 1.3 资源隔离做到了哪一层

| 隔离维度 | 支持情况 | 说明 |
|---|---|---|
| 工作目录 | **支持** | 企业版每租户独立根 `~/.jiuwenswarm/workspace_{tenant_key}`；会话目录、记忆文件、MCP 状态均在租户根下 |
| 配置下发 | **支持** | 企业版从库中读取「该用户的有效配置」，套用 models / mcp / permissions（`enterprise/config/effective/`） |
| 团队与成员限额 | **支持** | `enterprise/team/`：成员关系、路由绑定、推送扇出、`quota`、`member_limits` |
| 工作空间配额 | **支持（按用户强制）** | 每个文件/工作空间请求进入前绑定 `(group, bot, user)` 身份，配额检查按该身份计算 |
| 运行时实例 | **支持（逻辑隔离）** | 每租户一个 `AgentManager`，Agent 实例、会话任务、预热池按租户分开 |
| 进程 / CPU / 内存硬隔离 | **需扩展实现** | 同一 AgentServer 进程内多租户共享事件循环；如需强隔离，采用「每租户（或每批租户）一个 AgentServer 进程」的部署口径 |

### 1.4 可配置项

| 配置 / 环境变量 | 默认 | 作用 |
|---|---|---|
| `JIUWENSWARM_EDITION` | 个人版 | 切企业版（多租户、按用户下发配置、DB 会话映射） |
| `JIUWENSWARM_DATA_DIR` | `~/.jiuwenswarm` | 数据根目录，多实例必须分开 |
| `gateway.session_map_scope` | `per_chat_bot` | 会话映射粒度（见注意事项） |
| `ROLE`（容器） | — | `gateway` / `agentserver` |
| `AGENT_RUNTIME_DEPLOY_MODE` | on | `off` 可关闭 runtime 扩展 |
| `EXTENSION_DIRS` | — | 分号分隔的扩展目录 |

### 1.5 注意事项
1. 租户运行时池默认**不做容量与时间淘汰**（设计目的是避免长阻塞交互期间误淘汰租户运行时）。当前版本该池的容量/TTL 未作为配置项暴露，请按「单 AgentServer 进程承载的租户数」做容量规划，超出时用多实例分摊。
2. `gateway.session_map_scope` 默认 `per_chat_bot`，**仅作用于飞书族通道**（`feishu` / `feishu_enterprise`）。若客户在飞书群内要求「每个用户独立上下文」，请显式配置为 `per_chat_bot_user`。Web 通道与其他 IM 通道不受该项影响。
3. 进程级硬隔离（cgroup / 容器 per-tenant）需由部署侧提供，产品侧提供的是逻辑与目录级隔离。

**源码位置**：`jiuwenswarm/enterprise/tenant/{tenant_identity,tenant_agent_pool,async_lru_cache,tenant_paths,multi_tenant_strategy}.py`、`jiuwenswarm/server/runtime/agent_manager.py`、`jiuwenswarm/server/lifecycle.py`、`agent_gateway/routing/session_map.py`、`agent_gateway/common/tenant_key.py`、`docs/zh/单机多实例运行.md`

**回归测试**（客户可自行 `pytest` 核对）：
- `jiuwenswarm/tests/unit_tests/runtime/test_tenant_request_scope_concurrency.py` — 租户请求作用域下的并发隔离
- `jiuwenswarm/tests/unit_tests/server/test_handler_tenant_manager_binding.py` — 请求 → 租户 `AgentManager` 绑定
- `jiuwenswarm/tests/unit_tests/server/test_enterprise_http_tenant_scope.py`、`test_tenant_extract_from_http_query.py` — HTTP 入口的租户作用域与身份提取
- `jiuwenswarm/tests/unit_tests/server/test_front_control_tenant_bind.py`、`test_enterprise_history_scope.py` — 控制面租户绑定、历史作用域
- `jiuwenswarm/tests/unit_tests/common/test_enterprise_protocol_identity.py` — 三元组身份
- `jiuwenswarm/tests/unit_tests/enterprise/test_enterprise_plane.py` — 企业面整体
- `jiuwenswarm/tests/unit_tests/gateway/test_session_map_repository_wiring.py`（同名在 `agent-gateway/tests/unit_tests/gateway/`）— 会话映射仓储接线（企业版走 DB 跨实例共享）
- `jiuwenswarm/tests/unit_tests/agentserver/permissions/test_run_channel_isolation.py` — 通道级隔离
- `jiuwenswarm/tests/unit_tests/common/test_config_concurrent_models.py` — 并发模型配置
- ⚠️ **未检索到租户池淘汰的测试**（与「池默认不启用容量/TTL 淘汰」的结论一致）

---

## 2. 用户级运行时资源回收：空闲回收、取消、超时、释放

### 结论
**支持。** 取消、会话级释放、进程关闭三条链路完整；磁盘侧有按保留期的自动清理。空闲回收以「事件驱动 + 超时」为主，各层超时常量可由配置调整。

### 2.1 取消（对外 RPC）

```python
# ── 取消 / 暂停 / 补充（前端 → AgentServer）
# 作用：控制当前会话正在执行的一轮工作。intent 决定语义，只有 cancel / supplement 会掐断流式任务。
ReqMethod.CHAT_CANCEL = "chat.cancel"
params = {
  "intent": "cancel | supplement | pause | resume",
      # cancel    : 终止当前执行，掐断流式任务
      # supplement: 追加补充信息，同样掐断当前流后重入
      # pause     : 在 checkpoint 处阻塞，任务保留（不掐流式任务）
      # resume    : 解除 pause 阻塞（不掐流式任务）
  "wait_for_stop": bool,        # true = 同步等待会话停止后再返回（配合 stop_session_for_archive）
  "target_team_id": str,        # 只停指定 Team（先停 producer 再停响应流，避免误伤同会话更新的流）
  "target_request_id": str,
}
# 客户端断连场景：内部标记 E2A_CANCEL_SOURCE_CLIENT_DISCONNECT → 取消后自动清理该会话运行时
```

### 2.2 释放接口（运行时层，transport 无关）

```python
# ── AgentManager 层（每租户）
AgentManager.cleanup_session_runtime(*, channel_id, session_id) -> bool   # 清理一个会话的全部运行时（含空闲 TUI agent 退役）
AgentManager.cancel_all_inflight_work(reason, *, exclude_session_ids=None) # 批量取消（如 WS 断连时）
AgentManager.cancel_session_tasks(session_id)
AgentManager.release_subagent_runtime_for_session(session_id, reason=...) # 释放该会话下的子 Agent 运行时
AgentManager.session_has_pending_interrupt(session_id) -> bool            # 是否有待答中断（回收前判定）
AgentManager.pin_agent(...) / unpin_agent(...)                            # 钉住，避免使用中被回收
AgentManager.cleanup()                                                    # 该租户全部释放

# ── Runtime 服务层
RuntimeService.cancel_request(...)              # 单请求取消
RuntimeService.cleanup_session(...)             # 会话清理
RuntimeService.stop_session_for_archive(...)    # 归档前停止（可同步等待）
RuntimeService.stop_session_for_lifecycle(...)  # 生命周期动作触发的停止（block/check_sessions 两阶段）
RuntimeService.stop_subagent_runtimes(...)      # 子 Agent 停止
RuntimeService.close()                          # 进程级关闭

# ── 会话生命周期契约（可插拔参与者，便于客户接入自己的回收逻辑）
SessionActivityParticipant:   session_input_intent / foreground_changed / execution_started / execution_finished / session_inactive
SessionDeleteParticipant:     before_delete / release_resources / delete_failed / delete_committed
RuntimeResourceLease.release()
```

### 2.3 超时与空闲参数（客户可调）

| 配置键 | 默认 | 作用范围 |
|---|---|---|
| `stream_first_chunk_timeout` | 300 s | 单模型：等待首个流式分片 |
| `stream_idle_timeout` | 120 s | 单模型：流式静默判定 |
| `timeout` | 360 s（多模态类 1800 s） | 单模型 HTTP 请求 |
| `group_routing_timeout_seconds` / `group_stream_*` | 360 / 300 / 120 | 模型组级（可用 `JIUWENSWARM_MODEL_GROUP_*` 覆盖） |
| `react.completion_timeout` | `null`（不限时） | 一轮 ReAct 执行总时长 |
| `sandbox_idle_timeout_seconds` / `sandbox_idle_check_interval_seconds` | 600 / 30 | 沙箱空闲回收 |
| `session_cleanup.retention_days` | 30 | **会话目录与文件操作日志的保留天数，`0` = 不清理** |
| `file_retention_days` | 7 | 导出文件保留天数 |

### 2.4 磁盘侧自动清理（后台任务）

```python
# ── Gateway 后台清理（启动后 10 分钟首跑，之后每 24 小时一次）
# 作用：按保留期删除过期会话目录，并清理已无对应会话的孤儿文件操作日志。
cleanup_old_sessions()      -> {removed: int, errors: list}   # 按目录 mtime 与 cutoff 比较
cleanup_orphan_file_ops()                                     # 清理 .agent_history/file_ops_*_<agent>.json 中的孤儿记录
# 保留期解析：环境变量 CLEANUP_PERIOD_DAYS 优先，其次配置 session_cleanup.retention_days；0 = 关闭
```

### 2.5 注意事项
1. 回收以**事件驱动**为主（断连、会话删除、归档、配置变更、进程关闭）。租户级 `AgentManager` 本身没有空闲超时回收，请配合 §1.5 的容量规划使用。
2. 各层超时常量当前是**分别配置**的，没有统一的「用户级 idle policy」单一开关；上表已列出全部可调项。
3. `cleanup_session_runtime` 内部会聚合子步骤失败并抛错，集成方调用时需要自行兜异常，避免影响上层 RPC。

**源码位置**：`jiuwenswarm/server/handlers/chat.py`、`jiuwenswarm/server/runtime/agent_manager.py`、`jiuwenswarm/runtime/service.py`、`jiuwenswarm/runtime/session_lifecycle.py`、`jiuwenswarm/server/runtime/session/lifecycle.py`、`agent_gateway/cleanup.py`、`jiuwenswarm/resources/config.yaml`

**回归测试**：
- `jiuwenswarm/tests/unit_tests/agentserver/test_agent_manager_session_cleanup.py` — 租户层会话运行时清理
- `jiuwenswarm/tests/unit_tests/agentserver/test_interface_session_runtime_cleanup.py` — adapter 层 `cleanup_session_runtime`
- `jiuwenswarm/tests/unit_tests/runtime/test_session_lifecycle_participants.py` — 会话生命周期参与者（删除 / 归档两阶段）
- `jiuwenswarm/tests/unit_tests/runtime/test_answer_after_idle_eviction.py` — **空闲淘汰后仍可回答**（对应「顺路清扫」语义）
- `jiuwenswarm/tests/unit_tests/agentserver/test_agent_warm_pool.py` — 预热池与空闲 agent 退役
- `jiuwenswarm/tests/unit_tests/agentserver/test_resident_worker_shutdown.py`、`test_attached_worker_final_shutdown.py`、`test_worker_shutdown_deadline.py` — 常驻 worker owner lease 与关停期限
- `jiuwenswarm/tests/unit_tests/server/runtime/test_offline_session_cleanup.py`、`jiuwenswarm/tests/unit_tests/runtime/test_sdk_stream_cleanup.py` — 离线会话清理、流关闭清理
- `jiuwenswarm/tests/unit_tests/common/test_cleanup.py`、`jiuwenswarm/tests/unit_tests/gateway/test_agent_cleanup_trace.py`、`test_lifecycle_handlers.py` — 磁盘保留期清理与 Gateway 侧生命周期
- `jiuwenswarm/tests/unit_tests/observability/test_trajectory_retention.py` — 轨迹保留
- `jiuwenswarm/tests/unit_tests/agentserver/test_symphony_timeout_lifecycle.py` — 超时生命周期
- ⚠️ **未检索到「租户级空闲回收」的测试**（该能力未启用，见注意事项）

---

## 3. 按用户 / agent 隔离 MCP：连接池、凭据、作用域、回收、重连

### 结论
**支持（需配置）。** MCP 的挂载按 agent（企业版按租户）扇出；**连接标识由「配置内容 + 已解析凭据」哈希得出，凭据不同天然是两条连接**；凭据加密落盘、支持 OAuth/CLI 授权流、断线自动重连。多租户强隔离场景建议每租户一个 AgentServer 进程。

### 3.1 挂载作用域（按 agent 扇出）

```python
# ── 配置变更时把 MCP 挂到 / 摘下该租户的 agent
# 作用：新增、删除、启用、停用某个 MCP 时，只影响目标租户（企业版=该 tenant_key 的 AgentManager）
AgentManager.apply_mcp_change(name, action, *, enabled=True, target_channel_id=None)
#   action: 注册 / 卸载 / 启用 / 停用；target_channel_id 可把变更限定到某个频道

# ── 连接状态记录（每租户 workspace 下 mcp/state.json）
list_connected_mcps() / list_truly_connected_mcps() / list_connecting_mcps()
upsert_mcp_record(...) / set_mcp_state(...) / set_mcp_enabled(...)
```

### 3.2 连接标识与连接池（隔离的关键）

```python
# ── 由配置条目构造连接配置，并把 ${VAR} 占位符替换成真实凭据
# 作用：config.yaml 里只存占位符（不落明文 token），实际启动子进程 / 发 HTTP 时才注入真实凭据。
build_mcp_server_config(
    entry,                                  # 一条 mcp.servers 配置
    server_id_scope="jiuwenswarm",          # 用于派生稳定 server_id
    credential_resolver=...,                # Callable[[str], str|None]：按 MCP 名从凭据库取值
    resolve_node_launcher=True,
) -> McpServerConfig
#   stdio          → params.env / args / cwd 中的 ${VAR} 被替换（子进程 argv 不做 shell 展开，必须在此替换）
#   sse/http       → auth_headers（如 Authorization: Bearer ${MCP_TOKEN}）与 url 中的 ${VAR} 被替换

# ── server_id 生成规则（决定「两条配置是否复用同一条连接」）
_stable_mcp_server_id(scope, name, payload) -> "mcp_{scope}_{name}_{sha256(scope+payload)[:16]}"
#   payload 含已解析的 env / headers / url → 凭据不同 ⇒ 哈希不同 ⇒ 各自独立连接，互不共用

# ── 连接池（内核侧）
class ToolMgr:
    async def add_tool_server(self, server_config, expiry_time=None) -> list[McpToolCard]:
        ...   # 同一 server_id 已注册 ⇒ 直接复用现有连接与工具卡片，不会重复 spawn 第二个 stdio 子进程
    async def remove_tool_server(self, server_id, ignore_not_exist=True): ...  # 断开连接 + 摘掉该 server 的工具
    async def refresh_tool_server(self, server_id, skip_not_exist=False, force=False): ...  # 按 expiry_time 判定是否重取工具列表
    def get_mcp_client(server_id) / get_mcp_server_config(server_id) / get_mcp_tool_ids(server_id) / get_mcp_server_ids()
    async def release(self): ...   # 断开全部连接（进程关闭路径）
```

### 3.3 凭据与授权

```python
# ── 凭据存储（加密落盘）
CredentialStore().get_all(name) -> dict[str, str]     # 取某个 MCP 的全部凭据（供 ${VAR} 解析与 skill 脚本使用）
CredentialStore().set(...) / delete(...)
save_mcp_credentials(name, tokens)                    # RPC 侧写入
_wipe_stored_credentials(name)                        # 断开时一并清除凭据

# ── 连接与授权流程
connect_mcp(name, *, install_only=False)              # 安装并发起连接
complete_cli_auth(name, step_index)                   # CLI 型 MCP 多步授权
complete_oauth_callback(query)                        # OAuth 回调
complete_host_oauth_auth(name)
cancel_connect(name) / rollback_failed_connect(name)  # 取消与回滚
disconnect_mcp(name)                                  # 断开 + 清凭据
register_custom_mcp(name, config) / delete_custom_mcp(name)
_mask_mcp_spec(...)                                   # 对外展示时脱敏
```

### 3.4 断线重连

```python
# ── 装饰在 MCP 客户端调用上：仅对传输/会话类故障重建连接并重试该次调用
@with_reconnect
# 判定依据：异常消息含 session terminated / not found / invalid session、connection closed / stream closed、
#          broken pipe、not connected；JSON-RPC 错误码 32000（连接关闭）与 32600（需消息含 session）
# 退避：base 5 s，指数到上限 120 s，抖动 ±20%；成功后清零失败计数；每次客户端独立重连锁
# 重试次数：环境变量 OPENJIUWEN_MCP_RECONNECT_ATTEMPTS（未设则按默认策略）
```

### 3.5 对外 RPC

```
mcp.list / mcp.show / mcp.install / mcp.uninstall
mcp.register_custom / mcp.delete_custom
mcp.connect / mcp.disconnect / mcp.cancel_connect
mcp.save_credentials / mcp.wait_auth
命令：/mcp
```

### 3.6 配置示例

```yaml
mcp:
  servers:
    - name: example
      transport: stdio            # stdio | sse | http(streamable-http)
      command: npx
      args: ["-y", "@example/mcp"]
      cwd: /opt/mcp
      env: { EXAMPLE_TOKEN: "${EXAMPLE_TOKEN}" }     # 占位符，运行时替换
      timeout_s: 30
    - name: example-http
      transport: sse
      url: "https://example.com/sse?token=${MCP_TOKEN}"
      headers: { Authorization: "Bearer ${MCP_TOKEN}" }
```

### 3.7 注意事项
1. **凭据槽按 MCP 名称组织**（`CredentialStore` 以 name 为键）。同一进程内若多个用户使用**同名** MCP 且凭据不同，凭据存储层不区分租户 —— 建议同名 MCP 的多租户差异化凭据场景采用「每租户一进程」部署（官方支持单机多实例）。
2. **stdio / HTTP 型 MCP 的凭据是按连接注入的**（见 §3.2），不依赖环境变量。**仅「skill 型 MCP」**（其能力由 agent 通过 Bash 执行脚本提供）会把 token 同步到进程环境变量供脚本读取；该环境变量为进程级，多租户同进程时可见范围较大 —— 同样建议按进程切分租户。
3. 连接池当前**没有空闲 TTL 与连接数上限**，连接由「配置变更 / 断开 / 进程关闭」事件驱动回收。请按每进程 MCP server 数量做资源规划。
4. 内核侧 `refresh_tool_server(force=True)` 的刷新路径在工具卡片同名时存在重复注册风险；产品主链路采用「先摘后挂」，不触发该路径。若贵方直接调用内核 API，建议先 `remove_tool_server` 再 `add_tool_server`。

**源码位置**：`jiuwenswarm/common/mcp_config.py`（`build_mcp_server_config` / `_stable_mcp_server_id`）、`jiuwenswarm/server/runtime/mcp/{state_store,registry,credential,oauth_flow,cli_driver}.py`、`jiuwenswarm/server/handlers/mcp_hub.py`、`jiuwenswarm/server/runtime/agent_manager.py`、`agent-core/openjiuwen/core/runner/resources_manager/tool_manager.py`、`agent-core/openjiuwen/core/foundation/tool/mcp/client/reconnect.py`

**回归测试**：
- `agent-core/tests/unit_tests/core/runner/test_tool_manager_mcp_dedup.py` — **同 `server_id` 重复挂载复用缓存（去重回归，已核对）**
- `agent-core/tests/unit_tests/core/runner/test_resource_manager.py` — 资源管理器生命周期
- `agent-core/tests/unit_tests/core/foundation/tool/mcp/client/test_sse_reconnect_improvements.py`、`test_streamable_http_reconnect.py` — SSE / streamable HTTP 重连
- `agent-core/tests/unit_tests/core/single_agent/test_ability_manager_mcp_allowlist.py` — MCP 工具白名单
- `jiuwenswarm/tests/unit_tests/agentserver/mcp/test_mcp_credential_boundary.py`、`test_mcp_credential.py`、`test_mcp_token_flow.py` — 凭据边界与 token 流转
- `jiuwenswarm/tests/unit_tests/agentserver/mcp/test_mcp_config_placeholder.py` — `${VAR}` 凭据占位解析
- `jiuwenswarm/tests/unit_tests/agentserver/mcp/test_enterprise_session_mcp_reconcile.py` — 企业版按会话 / 租户 reconcile
- `jiuwenswarm/tests/unit_tests/agentserver/mcp/test_mcp_targeted_control.py`、`test_mcp_targeted_control_session_fanout.py` — 按 agent / 会话扇出的挂载控制
- `jiuwenswarm/tests/unit_tests/agentserver/mcp/test_mcp_cancel_connect.py`、`test_disconnect_skill_cleanup.py`、`test_mcp_init_failure_isolation.py` — 取消连接、skill 清理、初始化失败隔离
- `jiuwenswarm/tests/unit_tests/agentserver/mcp/test_mcp_state_store.py`、`test_mcp_servers_merge.py` — 状态存储与配置合并
- `jiuwenswarm/tests/unit_tests/agentserver/mcp/test_register_custom_mcp.py`、`test_delete_custom_mcp.py` — 自定义 MCP 注册 / 删除

---

## 4. 用户画像：字段、来源、更新策略、隐私边界、改删接口

### 结论
**支持（需配置）。** 画像由「人设/记忆 Markdown 文件 + 记忆引擎」承载，提供查看、编辑、开关、隐私规则开关的 RPC；记忆引擎支持内置 / 外接 / 双写，并支持按 `user_id / scope_id / session_id` 做数据隔离。

### 4.1 画像载体

| 文件（每租户 workspace 下） | 作用 |
|---|---|
| `USER.md` | 用户画像槽位（由模型在会话中写入/更新） |
| `AGENT.md` / `IDENTITY.md` / `SOUL.md` | agent 行为约定 / 身份 / 人格 |
| `HEARTBEAT.md` | 定时心跳任务 |
| `memory/MEMORY.md` | 长期记忆真源 |

初始化按语言（`_ZH` / `_EN`）拷贝模板；企业版每租户一份（`seed_tenant_agent_workspace()`）。

### 4.2 记忆引擎与隔离标识

```yaml
memory:
  mode: local | cloud                 # 内置记忆存储位置
  engine: builtin | external | both | none
  builtin:
    jiuwen:
      mode: http | sdk
      base_url: ...
      api_key: ...
      user_id: ${MEMORY_USER_ID:-__default__}   # ← 多用户部署必须设置，画像按此隔离
      scope_id: ...                             # 二级隔离域
      session_id: ""                            # 空 = 跨会话共享；填值 = 按会话隔离
      agent_id: jiuwenswarm
      space: ...
      identity_org: ... / identity_user: ...
      session_end_extract: ...                  # 会话结束时的记忆抽取
  forbidden_memory_definition:
    enabled: false                    # 隐私禁记规则开关（见 §4.4）
    patterns: []
```

### 4.3 画像 / 记忆的读写接口

**先分清三套记忆**，否则会把接口归属认错（三套互不调用，只有存储层可能重叠）：

| 套 | 存储 | 模型侧入口 | 管理侧入口 | 启用方式 |
|---|---|---|---|---|
| **内置本地**（默认） | 工作空间内 Markdown（`USER.md` / `MEMORY.md` / `memory/*.md`）+ 本地索引 | 工具 `memory_search` / `write_memory` / `edit_memory` / `memory_get` / `read_memory` | **`memory.*` 五个 RPC** | `memory.engine` 默认 `builtin` |
| **内核编码记忆** | KV / Vector / DB（`coding_memory_dir`） | 工具 `coding_memory_write` / `_edit` / `_read`（内核 rail 注册） | 无 | 内核 rail |
| **外接记忆** | 外部服务：`openjiuwen` / `mem0` / `openviking` / `lakebase` | **无工具** | **无 RPC** | `memory.engine=external` + `memory.external.provider` |

> **外接记忆没有读写工具、也没有管理接口**：它通过 `ExternalMemoryRail`（`priority = 75`，`PREFETCH_TIMEOUT = 5.0`）在每轮对话前预取相关记忆注入上下文，模型无感。客户若要接自有记忆服务，只有 provider 配置这一条路，检索时机与质量由 rail 决定。

**方法名定义在协议仓，不在产品仓**（按产品仓搜是搜不到的）：

```python
# agent-protocol/openjiuwen-protocol/openjiuwen_protocol/message.py:83  →  class ReqMethod(Enum)
# 同文件 :188-192
MEMORY_LIST   = "memory.list"
MEMORY_EDIT   = "memory.edit"
MEMORY_STATUS = "memory.status"
MEMORY_TOGGLE = "memory.toggle"
MEMORY_OPEN   = "memory.open"
```

**产品侧的注册与实现** —— 这五个是 **API 能力**（供集成层调用），不是界面功能承诺。
⚠️ **关键限定：这套接口管理的是「记忆文件」，不是「画像字段」**。可写范围见下方路径守卫表——**画像文件 `memory/USER.md` 对这套 RPC 是只读的**（由 agent 自动维护），且整套方法**只有改 / 新建，没有删除**。

```python
# 注册：AgentServer 启动时挂进 AdapterRegistry，按 req_method 命中
MemoryAdapter()            # jiuwenswarm/server/agent_ws_server.py:1374 与 :2416
methods = frozenset({ReqMethod.MEMORY_LIST.value, ...})   # memory_adapter.py:35-43

# 实现：真正干活的五个 handler
handle_memory_list(...)    # memory_rpc.py:258   列出可管理的记忆文件（含路径守卫与分类）
handle_memory_edit(...)    # memory_rpc.py:301   编辑 / 新建记忆文件（画像 memory/USER.md 不在可写范围）
handle_memory_status(...)  # memory_rpc.py:352   记忆子系统状态（含禁记规则开关状态）
handle_memory_toggle(...)  # memory_rpc.py:459   按 mode 维度开 / 关记忆能力，写回配置
handle_memory_open(...)    # memory_rpc.py:604   定位到具体记忆文件

# 路径守卫（安全边界）
_validate_edit_path(...)   # memory_rpc.py:115   只允许 _get_allowed_dirs() 内的路径，拦目录越界与符号链接逃逸
_classify_memory_file(...) # memory_rpc.py:154   分类：coding / auto / project / local / user
```

**`memory.edit` 实际可写范围（已逐行核对 `_validate_edit_path` / `_get_allowed_dirs` / `_is_runtime_memory_path`）**：

| 路径 | 能否改 | 依据 |
|---|---|---|
| `<agent_workspace>/memory/**`（含 `memory/USER.md`、`memory/MEMORY.md`、`memory/*.md`） | ❌ **只读** | `memory_rpc.py:130-131` → `"runtime memory is read-only, manual editing is not allowed"` |
| `<agent_workspace>/coding_memory/**` | ❌ **只读** | 同上（`_get_runtime_memory_dirs` 含之） |
| `<agent_workspace>/.jiuwen/**` | ✅ 可写 | `_get_allowed_dirs` |
| `~/.jiuwen/**` | ✅ 可写 | `_get_allowed_dirs` |
| `<project_dir>/**` 与 `<project_dir>/.jiuwen/**` | ✅ 可写 | `_get_allowed_dirs`（`project_dir` 由 TUI 的 `trusted_dirs` / `cwd` 补全） |
| `<agent_workspace>/JIUWENSWARM.md` / `JIUWENSWARM.local.md` | ✅ 可写 / 可新建 | `_validate_edit_path:136-149` 特例 |

> **由此得出对客户问题 4 的准确答案**：这套 RPC 能改的是 **`.jiuwen` / 项目目录 / 用户级目录**；**画像 `memory/USER.md` 与长期记忆 `memory/MEMORY.md` 由 agent 自动维护、对接口只读**。所以「修改画像」**没有面向集成层的字段级接口**，只有模型侧的记忆工具能写（见下表）。

> **边界自述（`memory_adapter.py:1-7` 原文照录）**：
> ```python
> """MemoryAdapter: execute TUI memory management in the user AgentServer.
> The Gateway must not derive a workspace path or access memory files itself.
> This adapter deliberately uses only get_agent_workspace_dir() from the current
> process, whose data directory is supplied when the AgentServer starts.
> user_id carried by the E2A envelope is never used for directory selection."""
> ```
> **解读（非原文）**：目录选择只跟进程启动时的数据目录绑定，请求参数里的 `user_id` 不参与 → 画像目录的隔离边界等于进程 / 数据目录的隔离边界。企业版按租户分根（`get_agent_root_dir(tenant_key)`），因此**按租户隔离成立**；但同一进程内不会靠 `user_id` 再自动分一层目录。

### 4.4 隐私边界

```python
# 禁记规则 forbidden_memory_definition 的生效范围（实测）
memory_forbidden_rail.py:79-80   # 提示词侧注入（优先级 MEMORY_FORBIDDEN = 113）
memory_tools.py:307 / :378       # 写记忆工具侧做 patterns 检查（write_memory / edit_memory）
memory_rpc.py:38-41 与 :370      # 管理 RPC 侧只读 enabled 标志用于状态展示与开关，编辑时不跑 patterns 校验

# 群聊硬闸：群聊场景禁止模型写 / 改记忆文件
memory_tools.py:24               # _GROUP_CHAT_MODE = ContextVar(bool)，按请求上下文隔离，不跨会话串
memory_tools.py:307              # write_memory → {"success": False, "error": "群聊模式下禁止写入记忆文件"}
memory_tools.py:378              # edit_memory  → {"success": False, "error": "群聊模式下禁止编辑记忆文件"}
```

> **生效范围（实测）**：禁记规则约束的是**模型写入**（提示词 + 写记忆工具）；`memory.edit` 管理接口**不跑 patterns 校验**。因此「集成层手工编辑画像文件」这条路径不受禁记规则约束，需要合规级强制时应在集成层前置过滤。
> **现成加分项**：群聊场景下模型写 / 改记忆被硬禁，这是内置的多用户防串味机制，无需额外配置。

### 4.5 可选画像流水线：PersonalContext（从外部源采集 → 知识图谱）

```python
PersonalContextHostAPI:
  configure / get_overview / get_runtime_config / patch_runtime_config / select_model
  set_collection_enabled / set_agent_use_enabled / set_master_enabled / apply_master / rollback_master
  list_fetch_services / create_fetch_service / patch_fetch_service / delete_fetch_service / set_fetch_service_enabled
  run_fetch / stop_fetch_run / get_fetch_run_status
  get_graph / get_tree / search_graph / get_graph_page / get_source
  get_authorization_status / authorize_provider          # GitHub/GitCode 等源授权（PAT 校验，15 s 超时）
```

### 4.6 客户问题四项逐条答复（先把问题对上，再看接口）

| 客户问的 | 我们的答复 | 依据 |
|---|---|---|
| **画像 schema** | **没有**。画像与记忆都是自由文本 Markdown（`memory/USER.md` 模板为 0 字节空文件），无 JSON Schema、无字段类型、无版本号 | `resources/agent/workspace/` 模板 |
| **画像来源** | 三条：① **模型**在对话中调 `write_memory` / `edit_memory` 写入 `memory/USER.md`（**群聊场景被硬禁**）② **会话结束抽取**（`session_end_extract`）③ 可选 **`PersonalContext`** 子系统从外部源采集（GitHub / GitCode 等，需 PAT 授权） | `memory_tools.py:61-84,291,362`、`config.yaml` 的 `memory.builtin.jiuwen.session_end_extract`、`personal_context/host_api.py` |
| **修改接口** | **没有面向集成层的画像字段接口**。能改画像的只有**模型侧工具**（且只允许 `memory/` 目录内）；前端 RPC 的 `memory.edit` 对 `memory/**` 是**只读**。集成层若要程序化改画像 → 只能直接改文件，或走 `PersonalContext` 的 `apply_master` | 见 §4.3 可写范围表 |
| **删除接口** | **没有**。五个 handler 中无 delete；无「删单条画像 / 按主体清除 / 导出画像」RPC。只能删文件本体（运维层面），或删采集源 `delete_fetch_service` | `memory_rpc.py` |
| **隐私边界** | 禁记规则（默认关闭）+ 群聊禁写；但**规则只在模型写入侧生效**，`memory.edit` 不校验 patterns；无 PII 正则、无加密落盘 | §4.4 |

> **一句话结论**：**画像没有 schema、没有字段级读写接口**；它就是一个由模型自动维护的 Markdown 文件（`memory/USER.md`），前端接口对它只读。客户若要求「结构化画像 + 可编程增删改」，需要在集成层自建存储与接口，或基于 `PersonalContext` 二次开发。

### 4.7 注意事项
1. 画像字段是**自由文本 Markdown**，没有 JSON Schema 与字段类型校验；如客户需要结构化画像（字段、类型、版本），建议在集成层定义 schema 并落到 `USER.md` 之外的自有存储。
2. 多用户部署**必须**设置 `MEMORY_USER_ID`（及 `scope_id`），否则内置记忆默认落在 `__default__` 标识下，跨用户共享。
3. 隐私禁记规则默认关闭（`enabled: false`），且属于「提示词 + 工具侧检查」；如需合规级强制脱敏（PII 正则、落盘前过滤、加密分级），建议在集成层前置处理。
4. `memory.edit` 可改 `<agent_workspace>/.jiuwen/**`、`~/.jiuwen/**`、`<project_dir>/**`；**`memory/USER.md` 与 `memory/MEMORY.md` 对接口只读**（agent 自动维护）。因此「改画像」没有集成层接口，只有模型工具；「删除单条画像 / 按主体清除 / 导出画像」的 RPC 也**均未提供**，合规遗忘需求需由集成层基于文件与记忆引擎 API 组合实现。
5. 外接记忆（`mem0` / `openviking` / `lakebase` / `openjiuwen` provider）**只提供 provider 配置，不提供读写工具与管理 RPC**；接入后检索时机由 rail 预取决定（单轮 5 秒超时）。如客户要求「可管理的外接记忆」，需在集成层自建管理面。
6. 内置记忆的模型写入侧在**群聊场景被硬禁**（`write_memory` / `edit_memory` 直接返回错误），因此群聊不会自动沉淀个人记忆；这是默认行为，无需配置。

**源码位置**：方法名定义在协议仓 `agent-protocol/openjiuwen-protocol/openjiuwen_protocol/message.py:83,188-192`；实现与装配在 `jiuwenswarm/agents/harness/common/memory_rpc.py`、`jiuwenswarm/server/runtime/gateway_adapter/memory_adapter.py`、`jiuwenswarm/server/agent_ws_server.py:1374,2416`、`jiuwenswarm/agents/harness/common/tools/memory_tools.py`、`jiuwenswarm/agents/harness/common/memory/forbidden.py`、`jiuwenswarm/agents/harness/common/memory/external_memory_builder.py`、`jiuwenswarm/agents/harness/common/rails/memory_forbidden_rail.py`、`jiuwenswarm/server/personal_context/host_api.py`、`jiuwenswarm/resources/config.yaml`、`docs/zh/{记忆,自动记忆,经验记忆,编码记忆}.md`

**回归测试**：
- `jiuwenswarm/tests/unit_tests/agentserver/memory/test_memory_rpc.py` — 五个 `memory.*` handler（含路径守卫与分类）
- `jiuwenswarm/tests/unit_tests/agentserver/memory/test_external_memory_builder.py`、`test_external_memory_config.py` — 外接 provider 装配与 `engine` 判定
- `jiuwenswarm/tests/unit_tests/agentserver/memory/test_coding_memory_tools.py`、`test_coding_memory_rail.py`、`test_code_adapter_coding_memory.py` — 内核 lite 那套（`coding_memory_*`）
- `jiuwenswarm/tests/system_tests/test_coding_memory.py`、`test_project_memory.py` — 系统级记忆
- `jiuwenswarm/tests/unit_tests/agentserver/rails/test_project_memory_rail.py`、`jiuwenswarm/tests/unit_tests/common/test_coding_memory_paths.py` — 项目记忆 rail、编码记忆路径
- `jiuwenswarm/tests/unit_tests/agentserver/test_personal_context_websocket_api.py`、`test_personal_context_lifecycle.py`、`test_personal_context_rail_registration.py`、`jiuwenswarm/tests/unit_tests/channel/test_web_channel_personal_context_payload.py` — PersonalContext 画像流水线
- `jiuwenswarm/tests/unit_tests/enterprise/config/test_memory_config_decoupling.py`、`jiuwenswarm/tests/integration/test_memory_config_processes.py` — 记忆配置解耦、多进程记忆配置
- ⚠️ **未检索到 `forbidden`（禁记规则）的专门测试文件** → 该规则目前靠提示词 rail + 写记忆工具侧检查，缺独立回归用例；如需合规级保证，建议要求补测试或在集成层自建校验

---

## 5. 工作空间文件跨节点可用：共享、锁、变更通知、快照、权限隔离

### 结论
**支持（需配置）。** 官方提供 NFS 共享存储方案与分布式 Team（PostgreSQL + pyzmq + 服务注册中心）；权限隔离为「单租户根 + 路径 containment + 区域白名单 + 按用户配额」；版本与回滚走 git 与会话 rewind。

### 5.1 跨节点共享

```yaml
# 分布式 Team（多节点协同）
team:
  runtime: { mode: ..., role: leader|teammate, member_name: ... }
  workspace:
    root_path: /mnt/nfs/jiuwen_team        # ← 跨节点共享的目录（NFS 挂载点）
  transport:
    type: pyzmq
    params: { direct_addr, bootstrap_direct_addr, pubsub_* }   # Leader 默认 18555/18556/18557，Teammate 18600
  storage:
    type: postgresql
    connection_string: ...                 # 所有节点必须可见的同一个库
react:
  a2x_registry:                            # 服务发现（分布式 Team 前置件，独立部署）
    base_url: ... / timeout: 30.0 / api_key: ...
    ownership_file: ... / role: teammate|teamleader
    dataset: "team_pool" / endpoint: ...
```

官方脚本：`scripts/nfs/{setup_nfs_server.sh,setup_nfs_client.sh,teardown_*.sh}` + `scripts/nfs/README.md`。要点：
- 只共享 `team.workspace.root_path` 指向的团队共享目录，**不要共享 `.agent_teams`**（进程本地状态）。
- 所有节点先各完成一次初始化；自定义挂载点时所有节点设同名 `JIUWEN_TEAM_WORKSPACE_ROOT`；建议所有节点用同一运行用户、走内网 IP。
- 附件与大文件跨节点走对象存储（OBS/MinIO），见 §8.7。

### 5.2 权限隔离与配额

```python
# ── 工作空间能力（企业版）
WorkspaceService(
    tenant_root,                       # 该租户的根，所有操作被限制在其内
    max_delete_depth=32,               # 删除递归深度上限
    max_delete_entries=5000,           # 单次删除条目上限
    max_download_bytes=100*1024*1024,  # 下载上限
)
WorkspaceService.resolve(...)          # os.path.commonpath 校验，防路径越界
WorkspaceService.list_tree / delete_entries / preview(≤64 KiB) / download / du
classify_zone(path) / is_deletable(...) / normalize_relative_path(...)   # 区域分类与可删白名单
check_workspace_write(additional_bytes=...)   # 写入前配额检查（按当前请求的 group/bot/user 身份）
```

### 5.3 锁

| 用途 | 机制 | 范围 |
|---|---|---|
| 实例互斥（AgentServer） | `fcntl.flock` / Windows 独占创建，`STALE_LOCK_TIMEOUT=30 s` | 本机 |
| Gateway 单实例（每 workspace） | `portalocker`，锁文件 `.gateway.lock`，获取超时 30 s | 本机 |
| 项目 store 跨进程互斥 | `fcntl.flock` | 本机 |
| IM 绑定 / 权限审批 / a2x ownership / worker owner lease | `AsyncFileLock` / `portalocker` / `FileLock`（带 lease） | 本机 |
| Team 文件锁 | `FileLockTimeout`（超时可捕获） | 本机 |

### 5.4 变更通知与快照

```python
# 本地 git 变更监听（工作区 diff 状态推给前端）
git_diff_watcher.py / git_diff_status.py
# 版本与回滚 RPC
project.git.{status, probe, init, switch_branch, create_branch, commit, push,
             diff_status, turn_diff_list, turn_diff, discard_turn_changes, redo_turn_changes}
# 会话级回滚
session.rewind / session.rewind_and_restore / session.rewind_compact / session.rewind_context
```

### 5.5 注意事项
1. 现有锁均为**本机 advisory 文件锁**，跨节点（NFS）不保证互斥语义。跨节点场景请遵循「**同一 workspace 同一时刻由单节点写入**」的运维口径，或按 Team 角色切分写入目录。
2. 未提供跨节点文件变更广播（inotify → 消息总线）与文件变更 webhook；多节点间同步依赖 git 提交/拉取与 Team 的 pyzmq 消息。
3. 未提供独立的「workspace 快照 / 回滚到快照」API；版本管理走 git（`project.git.*`），会话状态回滚走 `session.rewind*`，两者粒度不同。
4. 服务注册中心（A2X Registry）需独立部署，不是内置组件；它提供 Agent Team 的短时预订锁（reservation lease）与基于所有权的权限控制，用于**能力发现与协同**，不替代文件互斥。

**源码位置**：`scripts/nfs/`、`docs/zh/分布式Team.md`、`jiuwenswarm/enterprise/capabilities/workspace/{service,zones,quota}.py`、`jiuwenswarm/server/runtime/gateway_adapter/workspace_file_adapter.py`、`jiuwenswarm/instance_manager/lock.py`、`agent_gateway/gateway_lock.py`、`jiuwenswarm/server/runtime/session/{project_git,git_diff_watcher,project_store}.py`、`agent-protocol/AgentRegistry/README.md`

**回归测试**：
- `jiuwenswarm/tests/unit_tests/design/test_workspace_safety.py` — 工作空间安全边界
- `jiuwenswarm/tests/unit_tests/agentserver/permissions/test_send_file_path_guard.py` — 路径 containment（拦越界）
- `jiuwenswarm/tests/unit_tests/common/test_team_workspace_read_guard.py` — Team 工作空间读守卫
- `jiuwenswarm/tests/unit_tests/agentserver/test_auto_permission_workspace_owner.py` — 工作空间 owner 判定
- `jiuwenswarm/tests/unit_tests/agentserver/test_workspace_paths.py`、`jiuwenswarm/tests/unit_tests/test_runtime_workspace_binding.py` — 路径解析与运行时绑定
- `jiuwenswarm/tests/unit_tests/runtime/test_workspace_sqlite.py` — 工作空间 SQLite 存储
- `jiuwenswarm/tests/unit_tests/rsi/test_workspace_layout.py`、`test_workspace_recovery.py` — 布局与故障恢复
- `jiuwenswarm/tests/unit_tests/server/test_enterprise_projectless_workspace.py`、`jiuwenswarm/tests/unit_tests/agentserver/test_projectless_agent_workspace.py`、`jiuwenswarm/tests/system_tests/test_init_workspace.py` — 无项目工作空间与初始化
- ⚠️ **未检索到跨节点（NFS / 共享存储）与文件锁的测试**：`scripts/nfs/` 是部署脚本不是测试；锁为本机 advisory，跨节点互斥需客户在集成层保证

---

## 6. 上下文长度与推理等级：参数名、动态修改、生效范围、token 预算

### 结论
**支持。** 上下文窗口有明确的参数名与四级优先级；推理等级 8 档并按模型能力校验；模型配置可通过配置面板热改、会话内可切换；单模型输出上限可配。

### 6.1 上下文长度

```python
# ── 解析与优先级（源码即权威）
DEFAULT_CONTEXT_WINDOW_TOKENS = 256 * 1024
_CONTEXT_WINDOW_UNIT_MULTIPLIERS = {
    "token": 1,
    "tokens": 1,
    "k": 1024,
    "kb": 1024,
    "ki": 1024,
    "kib": 1024,
    "m": 1024 * 1024,
    "mb": 1024 * 1024,
    "mi": 1024 * 1024,
    "mib": 1024 * 1024,
}
parse_positive_int("262144" | "256k" | "256 K" | "1m" | "1MiB") -> int

resolve_context_window_tokens(model_name, *, context_engine_config,
                              model_config_obj, model_context_window_override) -> int
#   优先级：
#   1) 全局覆盖   react.context_engine_config.context_window_tokens
#   2) 当前模型   models.*[].model_config_obj.context_window（或运行时 override）
#   3) 按模型名   react.context_engine_config.model_context_window_tokens[model_name]
#   4) 固定默认   256K
```

| 配置键 | 位置 | 说明 |
|---|---|---|
| `models.{defaults,agentos,video,audio,vision,image_gen}[].model_config_obj.context_window` | 每模型 | 例 `context_window: 262144` |
| `react.context_engine_config.context_window_tokens` | 全局 | 显式覆盖，正整数 |
| `react.context_engine_config.model_context_window_tokens` | 映射 | 按模型名指定 |
| `enable_openrouter_model_context_window_tokens` | `false` | 关闭「自动联网获取模型窗口」，窗口完全由配置决定 |

> 传递语义：`context_window` 是内核 `ModelRequestConfig` 的正式字段，但**内核在构造厂商请求参数时显式排除它**（注释：上下文元数据由上下文引擎消费，不得作为厂商 SDK 请求参数）。→ 该值只用于本地压缩与卸载决策，不会污染厂商请求。

### 6.2 推理 / 思考等级

```python
ReasoningLevel = Literal["off","on","minimal","low","medium","high","xhigh","max"]   # 8 档

normalize_reasoning_level(raw) -> level
#   别名归一：none/false/disabled→off；true/enabled→on；minimum→minimal；med|mid→medium；
#            extra_high→xhigh；maximum/ultra→max
#   注意：YAML 1.1 会把裸 off/on 读成布尔值，代码显式把 False→"off"、True→"on"

reasoning_config_for_level(level) -> dict
#   off → {"mode":"disabled"}；on → {"mode":"enabled"}；其余 → {"mode":"enabled","effort":<level>}

validate_reasoning_level_for_model(raw_level, ...) -> None
#   向内核查询该 provider/model 在当前协议下支持的档位（capability.options）
#   不在支持列表内 → ValueError("reasoning_level must be one of: …")，保存路径返回 BAD_REQUEST（不是 500）
```

```python
# ── 内核能力表（客户自定义模型时可用）
get_reasoning_capability(*, provider, model, protocol, api_base="", endpoint_profile=None) -> ReasoningCapability
#   命中 profile 表 → 返回该模型精确档位；未命中 → 回落所选协议的默认能力（advisory 语义）
get_reasoning_capability_catalog() -> {protocol_defaults, model_fallbacks}   # 前端做档位推断
get_provider_reasoning_rules()     -> 按 provider 的有序 patterns + openai/anthropic 两套能力（首个命中即用）
```

| 相关配置 | 说明 |
|---|---|
| `endpoint_profile` | 端点方言：`openai / deepseek / openrouter / siliconflow / dashscope / openai_compatible / ollama / lmstudio / vllm / sglang` |
| `endpoint_profile_overrides` | 自建网关按 host 声明，例 `{"10.0.0.1": "vllm"}` |
| `models.*[].model_config_obj.reasoning_level` | 每个模型条目的思考档位 |

> 自建 / 私有网关（认 `enable_thinking`、`thinking_budget`，忽略官方 `thinking.type`）**必须**声明 `endpoint_profile`，否则思考档位对该网关不生效。

### 6.3 token 预算

| 键 | 语义 |
|---|---|
| `models.*[].model_config_obj.max_tokens` | **单模型输出上限**，会真正下发给厂商（内核优先用调用参数，否则取该配置，非空才注入） |
| `swarmflow_budget` | **Team 级**：该 Team 派生的所有 swarmflow run 的总 token 上限；`null` = 不限 |
| `memory.builtin.jiuwen.sdk.globals.llm_max_tokens` | 记忆抽取单次输出上限（4096） |
| `max_workflows` / `agents_per_run` / `max_agents_total` | Swarmflow 三级并发闸 |
| `session_token_usage` + `tokenizer_service` | 会话 token 统计与计数（`enable_tiktoken_counter` 关闭时用字符串长度兜底） |

### 6.4 动态修改与生效范围

```
models.list / models.replace_all / models.validate / config.validate_model   # 配置面板 RPC（Web 与 TUI 同契约）
GET /models                                                                  # REST 映射到 models.list
/model                                                                       # 会话内切换模型（会话级）
/context  /compact  /compact-partial                                         # 上下文查看与压缩
agent.reload_config / config.cache_clear / models.refresh_status             # 热重载与状态
```

| 改动 | 生效范围 |
|---|---|
| 模型条目（含 `context_window` / `reasoning_level` / `max_tokens`） | 配置级；保存时校验档位；变更后自动关闭凭据已变的旧 LLM 连接 |
| `/model` | 会话级 |
| `context_window_tokens`（全局） | 配置级，改后需 reload |
| Team 成员模型 | 经 `_sanitize_team_member_model` 单独映射，`reasoning_level` 转成 provider kwargs 后下发 |

### 6.5 上下文压缩 / 卸载（可调）

```yaml
react:
  context_engine_config:
    enabled: true
    enable_reload: true                     # [[HANDLE:xxx]] 把已卸载的大消息拉回
    compression_recall_config:              # 原文分块归档，模型用 [[COMPRESSION_RECALL: id=…]] 召回
      enabled: false
      chunk_size_tokens: 3000
      chunk_overlap_tokens: 500
    message_summary_offloader_config:
      protected_tool_names: ["read_file", "read_pdf"]
      add_message_threshold_ratio: 0.1
      ttl_seconds: 300
      ttl_context_occupancy_ratio: 0.5
    session_memory_compressor_config: { enabled: false, trigger_context_ratio: 0.8 }
    dialogue_compressor_config / current_round_compressor_config / round_level_compressor_config:
      trigger_context_ratio: 0.8
      min_target_context_ratio: 0.1
```

### 6.6 注意事项
1. `context_window` 不下发厂商，因此**服务端不会因窗口超限被厂商拒绝**；超窗处理由本地压缩/卸载完成。若客户希望把窗口值透传给自建推理网关，需要网关侧按 `endpoint_profile` 方言自行处理。
2. 上下文窗口当前支持「全局 / 每模型 / 按模型名映射」三级，**未提供 per-session / per-user 覆盖**。
3. 模型组级流式超时（`groups[].routing.timeout_seconds` 等）当前**已存储但未被路由层消费**，配置后不会改变实际超时行为；请以每模型条目的超时为准。
4. 未提供跨请求 / 跨会话的累计 token 预算顶层键（Team 级 `swarmflow_budget` 覆盖 swarmflow，不覆盖主对话）。成本管控建议在网关或推理服务侧按用户维度计量。

**源码位置**：`jiuwenswarm/common/context_window.py`、`jiuwenswarm/common/reasoning_config.py`、`jiuwenswarm/common/reasoning_injector.py`、`jiuwenswarm/common/config_panel/models_handlers.py`、`jiuwenswarm/server/handlers/commands.py`、`jiuwenswarm/server/runtime/tokenizer_service.py`、`agent-core/openjiuwen/core/foundation/llm/{schema/config.py,reasoning.py,model_clients/base_model_client.py}`、`jiuwenswarm/resources/config.yaml`、`docs/zh/上下文压缩.md`

**回归测试**：
- `jiuwenswarm/tests/unit_tests/agentserver/test_context_window_config.py`、`jiuwenswarm/tests/unit_tests/test_context_window.py` — 上下文窗口配置与四级优先级
- `jiuwenswarm/tests/unit_tests/test_reasoning_config.py` — 推理等级配置
- `jiuwenswarm/tests/unit_tests/common/test_model_config_validation.py`、`test_runtime_model_config.py`、`jiuwenswarm/tests/unit_tests/test_model_config_null_recovery.py` — 模型参数校验与空值恢复
- `jiuwenswarm/tests/unit_tests/gateway/test_resolve_model_config_obj.py` — Gateway 侧模型配置解析
- `jiuwenswarm/tests/unit_tests/test_compact_partial.py`、`test_compact_history_records.py` — 上下文压缩
- `agent-core/tests/unit_tests/core/foundation/llm/test_reasoning_profiles.py`、`test_reasoning_resolver.py`、`test_responses_usage_reasoning.py` — 内核推理档位定义与解析
- `agent-core/tests/unit_tests/core/foundation/llm/test_context_window_metadata.py` — `context_window` 元数据
- `agent-core/tests/unit_tests/core/context_engine/test_compressor_token_budget.py`、`test_full_compact_processor.py`、`test_micro_compact_processor.py`、`test_context_window_diff.py` — 内核压缩与 token 预算
- `agent-core/tests/unit_tests/harness/test_tool_loop_compact.py`、`agent-core/tests/unit_tests/extensions/observability/test_context_window_compaction_correlation.py` — 工具循环压缩、可观测关联
- ⚠️ **未检索到「累计 token 预算闸门」的测试**（与「无跨轮累计预算」的结论一致）

---

## 7. 模型主动提问：表单 schema 与恢复流程

### 结论
**支持。** 单选、多选、自由文本可在同一条提问里混排；提问由固定挂载的交互 rail 产生，无需客户开启；回答按 `request_id` 对齐，Agent 从断点继续，不重跑已完成步骤。

### 7.1 出站：向用户提问（AgentServer → Gateway → 前端）

```jsonc
// 事件类型：chat.ask_user_question
// 作用：把一次「需要用户回答」的中断推给前端；未回答前该会话在 checkpoint 处挂起。
// 来源（source）：ask_user_interrupt（模型主动提问）| permission_interrupt（权限审批）| confirm_interrupt（控制类工具确认）
{
  "event_type": "chat.ask_user_question",
  "request_id": "<关联键：回答时必须原样带回>",
  "source": "ask_user_interrupt",
  "questions": [
    {
      "question": "要部署到哪个环境？",
      "multi_select": false,                 // false=单选，true=多选
      "options": [
        { "label": "测试", "description": "...", "preview": "..." }   // description/preview 可选
      ]
    },
    {
      "question": "还需要补充什么？",
      "multi_select": true,
      "options": [ { "label": "回滚脚本" }, { "label": "Other" } ]     // "Other" 是保留项，用于自由文本
    }
  ]
}
```

**契约与上限（源码常量）**

| 约束 | 值 | 说明 |
|---|---|---|
| 单条提问最多问题数 | `MAX_STRUCTURED_QUESTIONS = 4` | 结构化提问上限 |
| 单次最多答案数 | `_MAX_ANSWERS = 8` | 与问题数独立 |
| 单选答案 | 必须**恰好 1 项** | `_answer_values(multi_select=False)` |
| 多选答案 | 最多 8 项，每项非空字符串 | 选项标签去重 |
| `Other` | 保留字 | 只选 `Other` 且无文本 = 不完整答案（值为空串） |
| 问题对象 | 键集合必须**恰好** `{question, options, multi_select}` | 多一个或少一个键 → `invalid ask-user question`（严格 schema） |

### 7.2 入站：用户回答（前端 → Gateway → AgentServer）

```jsonc
// 回答入口（两个方法，语义由是否恢复被中断的一轮决定）
chat.answer   // 普通回答（不恢复执行流）
chat.send     // 恢复被中断的一轮（is_stream = true，继续流式输出）

// 恢复请求必须满足三个条件（缺一即不视为恢复）
params = {
  "interrupt_source": "ask_user_interrupt",   // 白名单共 5 值：permission_interrupt / confirm_interrupt /
                                              //           ask_user_interrupt / evolution_interrupt / client_tool_interrupt
  "request_id": "<与出站事件一致>",
  "answers": [
    { "question": "要部署到哪个环境？", "selected_options": ["测试"], "custom_input": "" },
    { "question": "还需要补充什么？",   "selected_options": ["回滚脚本", "Other"], "custom_input": "顺带压测" }
  ]
}
// 归并规则：selected_options 逐个 strip 去空；有 custom_input 时丢弃 "Other" 占位、保留真实选项再追加文本
// 无 question 文本但有答案 → 落到 __free_text__ → 纯文本回答同样可以恢复
```

```python
# ── 内核恢复载体（checkpoint 反序列化后重建待答中断）
class InteractiveInput(RestoreFieldDefaultsOnUnpickle, BaseModel):
    user_inputs: Dict[str, Any]         # key = 交互 id（工作流节点 id 或暂停的 tool call id）
    raw_inputs: Any = None              # 不绑定 id，用于首次交互
    interrupt_answers: Dict[str, Any]   # key = InterruptRequest.interrupt_id

    def update(self, node_id, value): ...        # 按交互 id 回答（ask_user 走这条）
    def answer(self, interrupt_id, value): ...   # 按中断 id 回答：只有抛出该中断的 rail/工具会收到该值
    # 约束：raw_inputs 已存在时再按 id 回答 → INTERACTION_INPUT_INVALID；interrupt_id 必须是非空字符串
```

### 7.3 相关能力

| 能力 | 支持情况 |
|---|---|
| 权限审批卡片 | `build_verified_permission_ask_user_question()`，走同一事件与恢复通道 |
| 会话暂停 / 恢复 | `chat.cancel` 的 `intent = pause / resume`（在 checkpoint 阻塞，不掐流式任务） |
| 富 UI 表单 | A2UI 通道（`a2ui-agent-sdk`），Web 侧；与 ask_user 的 schema 各自独立 |
| Swarmflow 人工节点 | `human` / `human_session` 节点 |
| 群聊提问 / 语音待答 | `group_questions.py`、`voice_pending_interrupt.py` |

### 7.4 注意事项
1. 表单原语只有「选项 + 多选 + 自由文本」，**没有**数字 / 日期 / 文件 / 级联 / 条件显示等字段类型；复杂表单请用 A2UI 富 UI 通道（两套 schema 不统一）。
2. 未提供表单版本协商与不兼容降级路径（DTO 键带 `v1`，但无跨版本兼容层）。
3. 未提供提问超时 / 过期策略：挂起的会话会一直等待（租户池不淘汰正是为保障长阻塞提问），需要「超时按默认答案继续」的场景请在集成层用定时器 + `chat.cancel(intent=cancel)` 实现。
4. 待答中断的恢复依赖本地 checkpoint 目录（`<agent root>/.checkpoint`）。跨节点场景需把该目录纳入共享存储，否则切换节点后待答中断无法找回。

**源码位置**：`jiuwenswarm/agents/harness/common/rails/interrupt/interrupt_helpers.py`、`jiuwenswarm/agents/harness/common/rails/ask_user_contract.py`、`jiuwenswarm/agents/harness/common/rails/permissions/root_ask_user.py`、`jiuwenswarm/runtime/interaction.py`、`jiuwenswarm/runtime/service.py`、`agent_gateway/common/interrupt_resume.py`、`agent-core/openjiuwen/core/session/interaction/interactive_input.py`、`docs/zh/A2UI.md`

**回归测试**（本节证据最厚）：
- `jiuwenswarm/tests/unit_tests/runtime/test_stream_interaction_answer.py` — **已逐行核对：1292 行 / 29 个测试函数 / 80 个参数化用例**，覆盖：恢复事件必须在流结束前可见、连环提问不断流、错 id 与并发重复回答被拒、客户端提前关流后问题仍可重答、六种畸形 `answers` 不破坏会话状态、跨会话（`SessionCatalogError`）与跨通道（`ProcessCliBoundaryError`）越权拦截、权限卡长续写 300 chunk 逐条流出、MoA 进度身份绑回根执行；并**与前端夹具 `frontend/tests/fixtures/moaControlResume.json` 逐字节对齐**（前后端契约钉死）
- `jiuwenswarm/tests/system_tests/test_structured_ask_user.py` — 结构化提问端到端
- `jiuwenswarm/tests/unit_tests/agentserver/test_ask_user_validation.py` — 提问 schema 校验（问题数、选项数、单选约束）
- `jiuwenswarm/tests/unit_tests/agentserver/permissions/test_interrupt_id_answers.py`、`test_permission_stale_interrupt_replay.py` — 按 interrupt id 回答、过期 interrupt 重放
- `jiuwenswarm/tests/unit_tests/agentserver/test_deep_adapter_interrupt.py`、`test_interface_interrupts.py`、`test_interrupt_helpers.py`、`test_interrupt_history.py`、`test_interrupt_output_handoff.py` — 中断全链路
- `jiuwenswarm/tests/unit_tests/runtime/test_interaction_answer_delivery.py`、`test_client_tool_control_resume.py`、`jiuwenswarm/tests/unit_tests/agentserver/permissions/test_client_tool_resume.py`、`test_client_tool_smart_resume.py` — 恢复投递与智能恢复
- `jiuwenswarm/tests/unit_tests/channel/test_feishu_ask_user_answer.py`（同名在 `agent-gateway/tests/unit_tests/channel/`）— 飞书侧回答
- `jiuwenswarm/tests/unit_tests/gateway/test_web_connect_interrupt_synth.py`（同名在 `agent-gateway/tests/unit_tests/gateway/`）— Web 侧中断合成
- `jiuwenswarm/tests/unit_tests/server/agentserver/test_sensitive_ask_user_answers.py` — 敏感回答处理
- `jiuwenswarm/tests/unit_tests/runtime/test_checkpoint_journal.py`、`jiuwenswarm/tests/unit_tests/agentserver/test_workflow_checkpoint.py`、`agent-gateway/tests/unit_tests/gateway/test_task_checkpoint.py` — 断点续跑
- `jiuwenswarm/tests/unit_tests/agentserver/test_plan_approval_interrupt_rail.py`、`test_code_confirm_interrupt_rail.py`、`test_cron_skip_permission_interrupt_rail.py` — 计划审批 / 代码确认 / 定时任务跳过

---

## 8. 文件读写 / 上传下载 / 检索：API 完整度、大文件、流式、错误码

### 结论
**支持。** 文件与工作空间 RPC 完整；上传/下载支持分块读与 HTTP Range（206）；工作空间配额按 `(group, bot, user)` 三元组强制；对象存储按用户三元组哈希前缀隔离；HTTP 侧有完整的错误码 → 状态码映射。

### 8.1 RPC 清单

```
# 文件 / 工作空间域（WebSocket 与 HTTP 复用同一个 adapter）
media.persist / media.discard / document.persist / document.formats
path.select_directory / path.select_files
file.import_url / file.upload_chunk / file.download_verified_chunk / file.read_upload_chunk
im.file_persist
workspace.tree / workspace.preview / workspace.usage / workspace.entries_delete

# 项目域
project.{list, info, create, rename, pin, remove, restore, lifecycle,
         pinned_sessions, get_sessions, get_cron_sessions, cron_resolve_binding}
project.git.{status, probe, init, switch_branch, create_branch, commit, push,
             diff_status, turn_diff_list, turn_diff, discard_turn_changes, redo_turn_changes}

# 会话资产与群文件
session.assets.list / session.assets.remove
group.file.rename / group.file.delete / group.history.clear
```

### 8.2 HTTP 文件通道（与 `/ws` 同端口）

```
GET  /file-api/list-files        GET  /file-api/list-markdown
GET  /file-api/raw-file          GET|HEAD /file-api/download
GET  /file-api/file-content      POST /file-api/file-content
POST /file-api/upload            POST   /file-api/mkdir
```

```python
# ── 断点续传（下载侧）：HTTP Range
_parse_single_byte_range(range_header, file_size)
#   命中区间 → 206 + Content-Range: bytes start-end/total + Accept-Ranges: bytes
#   区间不可解析 → 416 + Content-Range: bytes */total + Content-Length: 0
#   无 Range → 200 + Content-Length
#   注：只接受单个 bytes= 区间；multipart range（带逗号）不支持

# ── 分块读（上传落盘后按 offset 读块，供 agent 处理大文件）
read_session_upload_chunk(session_id, raw_path, offset, limit)
    -> {data_base64, offset, chunk_size, size, eof, name, mime_type}
```

```python
# ── 身份绑定：请求身份必须与令牌一致
_resolve_user_id(request, explicit=None)
#   取值链：显式参数 → 请求头 x-user-id → 令牌中的 username
#   再与 IAM 令牌身份比对；不一致 → 403 code=USER_MISMATCH（"user_id 与 token 不匹配"）
_bind_file_api_session(request, session_id)   # 把会话绑定到本次请求，后续操作按会话归属校验
```

### 8.3 大小与数量限制

| 限制 | 值 | 适用范围 |
|---|---|---|
| `MAX_FILE_BYTES` | **50 MiB** | 对象存储（OBS/MinIO）附件通道 |
| `WEB_WS_MAX_MESSAGE_BYTES` | 150 MiB | 浏览器 ↔ WebChannel 的 WS 帧（大文件建议走 HTTP，不走 WS） |
| `AGENT_WS_MAX_MESSAGE_BYTES` / `SEND_BUDGET` | 8 MiB / 6 MiB | Gateway ↔ AgentServer 内部链路（禁止大文件 base64） |
| `max_download_bytes` | 100 MiB | 工作空间下载 |
| `_VERIFIED_DOWNLOAD_CHUNK_MAX_BYTES` | 512 KiB | 校验下载分块 |
| `UPLOAD_READ_CHUNK_BYTES` | 512 KiB | 分块读 |
| `MAX_UPLOAD_COUNT` | 20 个/请求 | `/file-api/upload` |
| 图片 | 10 MiB / 最多 8 张 | Web 媒体归一化 |
| 语音（ASR） | 25 MiB | Web 任务语音 |
| 预览 | 64 KiB | `workspace.preview` |
| 选择文件 | 200 项 | `path.select_files` |
| 选择图片 | 4 MiB | 需留在全局 WS 帧内 |
| 下载令牌有效期 | 默认 600 s；交付产物可为长期 | HMAC 签名令牌 |
| OBS 预签名 | 上传 10 分钟 / 下载 7 天 | 对象存储 |

### 8.4 对象存储（附件与大文件跨节点）

```python
# ── 附件存储装配（MinIO/S3）
AttachmentStore.from_env()   # 需要 OBS_URL / OBS_ACCESS_KEY / OBS_SECRET_KEY / OBS_BUCKET（默认桶 jiuwenswarm）
                             # OBS_SECURE 默认 false（生产请置 true）；超时 connect=5 s / read=60 s

# ── 对象键与隔离前缀
object_prefix(group_id, bot_id, user_id)      -> "uploads/{sha256(json([group,bot,user]))}/"
                                               #   三者缺一 → ValueError（按用户三元组隔离，前缀为哈希，不可猜测）
download_object_key(tenant_key, asset_id, fn) -> "downloads/{sha256(tenant_key)}/{asset_id}/{fn}"
                                               #   asset_id 必须匹配 [0-9a-f]{32}（防注入）

# ── 完整性校验
upload_local_file(...)   -> etag，并写对象 metadata {"sha256": <摘要>, "session-id": <会话>}
download_to_local(...)   -> 先校验 x-amz-meta-session-id（不符 → "download session mismatch"）
                           再重算 sha256 与 metadata 比对（不符 → "downloaded file does not match upload digest"）
```

### 8.5 文件名与内容安全

```python
_UNSAFE_FILENAME_CHARS   = [\x00-\x1f\x7f<>:"/\\|?*]+     # 控制字符与 Windows 非法字符
BIDI 控制字符                                          # 单独剥离，防 evil‮gnp.md 这类双向文本伪装扩展名
_WINDOWS_RESERVED_NAMES  = CON/PRN/AUX/NUL/COM1-9/LPT1-9
_MAX_FILENAME_BYTES      = 180（按字节计）
_MAX_SESSION_ID_CHARS    = 120；safe_session_dirname 额外拒绝 "." / ".." / 引号包裹形式
FORBIDDEN_DOCUMENT_EXTENSIONS = 30 项可执行/包扩展名黑名单
    .exe .dll .msi .scr .bat .cmd .ps1 .vbs .wsf .hta .jar .lnk .bin .so .dylib
    .app .dmg .pkg .command .scpt .scptd .workflow .xpc .bundle .framework .kext
    .prefpane .saver .component
image_suffix_for_mime(...)  # 图片仅接受 png / jpeg / webp / gif
```

落盘安全：descriptor-anchored `os.open`（`O_NOFOLLOW | O_DIRECTORY`）、要求 `st_nlink == 1` 的普通文件、`unique_upload_path`（`stem-N`，最多 999）、`atomic_write_unique`（`O_CREAT|O_EXCL`，配额检查前置）。

### 8.6 错误码（HTTP 文件通道）

```python
_status_for_transfer_code(code) -> HTTP status
  400  BAD_REQUEST / INVALID_PATH / MISSING_FILE_PATH / MISSING_SESSION_ID
       / MISSING_FILE_CONTENT / ONLY_MARKDOWN_SUPPORTED / FORBIDDEN_DIR
  401  UNAUTHORIZED / AUTH_REQUIRED
  403  FORBIDDEN / FORBIDDEN_EXTENSION / FORBIDDEN_PATH / USER_MISMATCH
  404  NOT_FOUND / INSTANCE_NOT_FOUND / FILE_NOT_FOUND
  413  PAYLOAD_TOO_LARGE / FILE_TOO_LARGE
  501  LIST_NOT_SUPPORTED / NOT_IMPLEMENTED
  502  其他
响应体：{ "error": "<可读文案>", "code": "<错误码>" }
```

### 8.7 检索能力

文件与内容检索作为 **agent 工具**提供（由模型按需调用），工具能力表含：
`read_file / read_pdf / write_file / edit_file / read_text_file / write_text_file / glob_file_search / glob / list_dir / list_files / grep / search_replace / upload_file / upload_photo`，另有 `tool_search`（工具发现）、`search_skill`、`recall_compressed_context`。

### 8.8 注意事项
1. 上传是**单次 PUT**（对象存储预签名或 multipart 表单），未提供 multipart 分片上传会话协议；附件通道实际上限即 `MAX_FILE_BYTES = 50 MiB`。超过该量级的文件建议走共享存储 / 对象存储直传，或按业务侧自行切分后由 agent 合并。
2. 三处上限各自独立（附件 50 MiB / 浏览器 WS 150 MiB / 工作空间下载 100 MiB），**请按通道选择**：小文件走 WS，大文件走 `/file-api` HTTP，跨节点大文件走对象存储。
3. 未提供「项目内搜索」的同步 RPC（无分页 / 无取消）；前端搜索框需驱动一次 agent 调用取结果，或自建索引。
4. 本地文件通道未提供条件请求（`ETag` / `If-None-Match` / `If-Range`）；对象存储通道有 sha256 + etag 校验。
5. 未提供目录级批量打包下载（zip/tar）；`workspace.entries_delete` 有深度 32 / 条目 5000 上限，且为同步调用（未任务化）。
6. WS/RPC 侧错误码风格与 HTTP 侧不完全一致（`WorkspaceError` 四值码 + 部分裸异常），集成方在 WS 侧建议按 `code` 字段 + 文案双通道处理。

**源码位置**：`jiuwenswarm/server/runtime/gateway_adapter/workspace_file_adapter.py`、`jiuwenswarm/server/runtime/gateway_adapter/project_adapter.py`、`jiuwenswarm/server/runtime/attachments/{upload_storage,media_attachments,document_attachments,obs_attachments}.py`、`jiuwenswarm/server/runtime/session_assets.py`、`jiuwenswarm/server/dispatch.py`、`jiuwenswarm/agents/harness/common/rails/permissions/tool_capabilities.py`、`agent_gateway/channel_manager/web/container_file_http.py`、`agent_gateway/common/{obs_files,ws_limits,attachment_safety,file_download_token}.py`

**回归测试**：
- `jiuwenswarm/tests/unit_tests/channels/web/test_web_file_api_errors.py` — **错误码 → HTTP 状态映射**
- `jiuwenswarm/tests/unit_tests/channels/web/test_web_file_download.py`（同名在 `agent-gateway/tests/unit_tests/channels/web/`）— 下载链路
- `jiuwenswarm/tests/unit_tests/agentserver/test_verified_web_file_download.py`、`test_verified_download_assets.py` — 校验下载（etag / sha256）
- `jiuwenswarm/tests/unit_tests/agentserver/test_session_upload_read.py` — 分片上传与分片读取
- `jiuwenswarm/tests/unit_tests/gateway/test_upload_storage.py` — 上传存储
- `jiuwenswarm/tests/unit_tests/agentserver/test_obs_file_store.py`、`test_obs_file_delivery.py`、`test_obs_bucket_initialization.py`、`test_obs_ssl_verify.py` — OBS / S3 存储、投递、桶初始化、SSL
- `jiuwenswarm/tests/unit_tests/agentserver/test_enterprise_file_http.py`、`jiuwenswarm/tests/unit_tests/gateway/test_enterprise_files.py`（同名在 `agent-gateway/tests/unit_tests/gateway/`）— 企业版文件 HTTP 与租户作用域
- `jiuwenswarm/tests/unit_tests/agentserver/test_client_attachments_attach.py`、`test_client_attachments_directory.py`、`test_client_attachments_watch.py`、`test_client_attachments_agentserver.py`、`jiuwenswarm/tests/unit_tests/server/test_media_attachments.py`、`test_media_attachment_mixed_batch.py`、`jiuwenswarm/tests/unit_tests/gateway/test_document_attachments.py`、`jiuwenswarm/tests/unit_tests/channel/test_attachment_persist.py` — 附件与混合批次
- `jiuwenswarm/tests/unit_tests/test_agentos_file_api_iam.py` — 文件 API 鉴权
- `agent-gateway/tests/unit_tests/gateway/test_file_picker.py`、`test_at_file_references.py`、`test_channel_manager_file_fanout.py` — 文件选择器、@ 引用、通道扇出
- ⚠️ **未检索到 HTTP `Range` / 206 断点续传的专门测试**，也未见 multipart 上传测试 → 大文件续传建议在验收阶段实测

---

## 附录 A：组件与协议边界

| 组件 | 职责 | 与本次答复的关系 |
|---|---|---|
| **Gateway**（`agent-gateway`） | 频道接入、鉴权、会话路由与映射、对象存储、附件安全、磁盘保留清理、模型配置代理 | §1 会话映射、§2 磁盘清理、§8 文件 HTTP 与大小上限 |
| **AgentServer**（JiuwenSwarm） | Agent 运行时、会话、工具、MCP、记忆、上下文引擎、RPC 分发 | §1–§8 主体 |
| **openjiuwen 内核**（`agent-core`） | `Model` / `ModelRequestConfig` / `ToolMgr`（MCP 连接池）/ `InteractiveInput` / 推理能力表 | §3 连接池与重连、§6 模型字段、§7 恢复载体 |
| **agent-protocol / A2X Registry** | A2A、A4P、服务发现注册中心（含 reservation lease） | §5 分布式 Team 前置件 |
| **Web / TUI** | 前端（5173）、TUI 通道 | §6 配置面板、§7 提问 UI、§8 上传下载 |

**租户键契约**：`tenant_key` 的权威实现在 Gateway 侧（`md5("{group}::{bot}::{user}"}`，分隔符 `::`），AgentServer 侧为 re-export，**两侧恒为同一实现** → 不存在「两边算法不一致导致租户错位」的风险。

**控制面 / 执行面分离**：`jiuwenswarm/server/control/methods.py` 用 `CONTROL_*_METHODS` 划分哪些 RPC 不唤醒 agent 运行时（配置查询、模型列表等走控制面，会话创建/切换/删除与 git 操作留在执行面）→ 前端读配置不会把已休眠的租户运行时叫醒。

## 附录 B：建议部署口径（容量与隔离）

| 场景 | 建议 |
|---|---|
| 多用户 SaaS（强隔离诉求） | 每租户或每批租户一个 AgentServer 进程（官方支持单机多实例），Gateway 统一路由；企业版启用 DB 会话映射与按用户下发配置 |
| 附件与大文件 | 启用 OBS/MinIO，设 `OBS_SECURE=true`；按 50 MiB 规划附件，更大文件走共享存储或对象存储直传 |
| 跨节点 Team | NFS 只共享 `team.workspace.root_path`；`.agent_teams` 与 checkpoint 目录留本地或按节点独占；PostgreSQL 所有节点可见；独立部署 A2X Registry |
| 自建推理网关 | 必须声明 `endpoint_profile`（或 `endpoint_profile_overrides`），否则思考档位不生效 |
| 记忆隔离 | 必须设 `MEMORY_USER_ID`（及 `scope_id`）；隐私敏感场景开启 `forbidden_memory_definition` 并在集成层前置脱敏 |
| 会话数据保留 | `session_cleanup.retention_days`（默认 30 天，`0` 关闭）或环境变量 `CLEANUP_PERIOD_DAYS` |

## 附录 C：取证索引（供逐条复核）

| 主题 | 关键文件 |
|---|---|
| 租户身份与池 | `jiuwenswarm/enterprise/tenant/{tenant_identity,tenant_agent_pool,async_lru_cache,tenant_paths}.py`、`agent_gateway/common/tenant_key.py` |
| 会话映射 | `agent_gateway/routing/{session_map,session_map_repository,session_sharing}.py` |
| 取消与回收 | `jiuwenswarm/server/handlers/chat.py`、`jiuwenswarm/runtime/service.py`、`jiuwenswarm/server/runtime/agent_manager.py`、`agent_gateway/cleanup.py` |
| MCP | `jiuwenswarm/common/mcp_config.py`、`jiuwenswarm/server/runtime/mcp/*.py`、`agent-core/.../resources_manager/tool_manager.py`、`agent-core/.../mcp/client/reconnect.py` |
| 画像与记忆 | `jiuwenswarm/agents/harness/common/memory_rpc.py`、`jiuwenswarm/server/runtime/gateway_adapter/memory_adapter.py`、`jiuwenswarm/server/personal_context/host_api.py` |
| 工作空间与跨节点 | `jiuwenswarm/enterprise/capabilities/workspace/*.py`、`scripts/nfs/`、`docs/zh/分布式Team.md`、`agent_gateway/gateway_lock.py` |
| 上下文与推理 | `jiuwenswarm/common/{context_window,reasoning_config,reasoning_injector}.py`、`agent-core/.../llm/{schema/config.py,reasoning.py,model_clients/base_model_client.py}` |
| 人机提问 | `jiuwenswarm/agents/harness/common/rails/{ask_user_contract.py,interrupt/interrupt_helpers.py}`、`jiuwenswarm/runtime/interaction.py`、`agent_gateway/common/interrupt_resume.py`、`agent-core/.../session/interaction/interactive_input.py` |
| 文件 API | `jiuwenswarm/server/runtime/gateway_adapter/workspace_file_adapter.py`、`agent_gateway/channel_manager/web/container_file_http.py`、`agent_gateway/common/{obs_files,ws_limits,attachment_safety,file_download_token}.py` |
| 全部配置键与默认值 | `jiuwenswarm/resources/config.yaml`（约 1900 行，同时是升级时的键白名单） |

## 附录 D：回归测试索引与核对方式

§1–§8 每节末尾都有一段「**回归测试**」，列出该能力对应的测试文件（跨 `jiuwenswarm` / `agent-gateway` / `agent-core` 三个仓检索）。客户可按下列方式自行核对：

```bash
# 产品仓根目录（人机提问恢复，证据最厚的一节）
pytest tests/unit_tests/runtime/test_stream_interaction_answer.py -q
# 内核仓（MCP 挂载去重）
pytest tests/unit_tests/core/runner/test_tool_manager_mcp_dedup.py -q
```

三点口径说明：

1. 测试文件按**文件名关键词**检索得到，个别文件同时覆盖多个主题；**具体用例覆盖以文件内容为准**。
2. 本次**已逐行核对**的只有 `tests/unit_tests/runtime/test_stream_interaction_answer.py`（1292 行 / 29 个测试函数 / 80 个参数化用例），其覆盖范围见 §7 末尾的逐条列举。
3. 各节末尾的 ⚠️ 行是**未检索到对应测试的能力**，含义是「能力存在但缺回归保护」或「能力未启用」，验收阶段建议实测补齐。目前共 6 处：租户池淘汰、租户级空闲回收、禁记规则、跨节点共享与文件锁、累计 token 预算、HTTP Range 断点续传。
