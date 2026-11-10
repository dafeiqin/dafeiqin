# JiuwenSwarm 追加十一项能力确认答复

> 本文是《JiuwenSwarm 八项能力确认答复》的**续篇**，章节号从 9 接续（9–19），两份合起来覆盖贵方提出的十九项关注点。八项那份未包含的内容全部在此，结论同样逐条回源码取证。

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
| 9 | 私有协议接入（WebSocket / Gateway 与客户端私有协议） | **支持** | 三层私有协议齐备：北向 WS、内部 E2A、REST 组装；方法名与事件名有单一真源（协议仓枚举） |
| 10 | 断线重连、会话恢复、幂等 | **支持**；断线事件补偿**需扩展实现** | 连接层自动换绑 + 轮次层「断连=暂停保账本、主动停=落账」；`request_id` 幂等 |
| 11 | 用户信息透传（tool context 注入 user / token / agent_key）、签名校验与过期 | **支持**（user 透传、签名、过期）；`agent_key` / 用户 token 注入**需扩展实现** | 工具上下文带 `user_id` 路由键；下载令牌 HMAC-SHA256 + 默认 600 秒过期；内部链路 mTLS |
| 12 | 重连或切换节点恢复（会话恢复 / 任务查询 / 取消 / 幂等） | **支持（需配置）**；任务查询**需扩展实现** | 会话级恢复与取消完整、跨节点靠路由层；无 `task_id` 概念，可查询粒度是会话 |
| 13 | 服务端主动推送（订阅 / 推送 / 断线补偿 / 事件序号） | **支持**；事件序号与断线补偿**需扩展实现** | 50+ 事件、按会话定向推；`seq` 字段已在协议里但仅轨迹通道填充 |
| 14 | 大工具结果（大结果 / 二进制 artifact / 分页 / 下载接口） | **支持** | 推送侧 60000 字符截断并带截断标记；产物走受管下载资产 + 签名令牌 + 512 KiB 分片；列表与历史均有分页 |
| 15 | 多模态输入（模态 / 编码 / 限制 / 错误码） | **支持** | 图片 / 文档 / 语音 / 视频；文档单文件 100 MiB、单次 8 个；可执行与脚本类扩展名黑名单拦截 |
| 16 | Langfuse 观测（trace / token / 耗时、OTel 或 Langfuse exporter） | **支持（需配置）** | 内核自带 Langfuse 与 OTLP exporter；需开 `agent_observability.enabled` 并填 Langfuse 公私钥 |
| 17 | 项目管理与目录树（分支创建 / 切换 / 合并 / 删除） | **支持**（项目、目录树、分支创建与切换）；分支合并与删除**需扩展实现** | `project.git.*` 12 个方法；无 merge / delete_branch / 分支列表 / 提交历史 |
| 18 | 会话分支（分支数据模型、创建 / 切换 / 合并 / 删除） | **支持**（分叉、切换、回退）；分支合并**需扩展实现** | `session.fork` 带截断点与两段式提交，血缘记在 `source_session_id`；无会话合并接口 |
| 19 | 文件变更通知（订阅 / 事件序号 / 补偿） | **支持（需配置，限 git 项目）**；通用文件变更订阅、事件序号与补偿**需扩展实现** | `/ws/git` 三档订阅 + 四个 `project.git.diff_*` 事件；非 git 目录无变更事件 |

---

## 9. 私有协议接入：WebSocket、Gateway 与客户端私有协议是否在 JiuwenSwarm 内

### 结论
**支持。** 私有协议不是「一套」而是**三层**，且方法名与事件名有**单一真源**（协议仓枚举），产品仓与 Gateway 都从该真源取值，不存在各自定义一套的情况。

### 9.1 协议真源：方法名与事件名的唯一定义处

```python
# ── 协议数据模型（真源：agent-protocol 仓）
# 文件：openjiuwen-protocol/openjiuwen_protocol/openjiuwen_protocol/message.py
class ReqMethod(Enum):          # :83-624，约 490 个成员；枚举值即线上报文的 method 字符串
    CHAT_SEND   = "chat.send"
    CHAT_RESUME = "chat.resume"        # :88  恢复被暂停的一轮
    CHAT_CANCEL = "chat.interrupt"     # :89  ⚠️ 取消的线上名是 chat.interrupt，不是 chat.cancel
    CHAT_ANSWER = "chat.user_answer"   # :90  ⚠️ 回答的线上名是 chat.user_answer
    SESSION_FORK = "session.fork"      # :156
    WORKSPACE_*  # 见 9.2 的兼容扩展
    ...
class EventType(Enum):          # :625-688，50+ 个事件名（推送面清单见 §13）
    def _missing_(cls, value): ...     # :683 历史名归一：heartbeat.relay → health_check.relay
class Mode(Enum):               # :689 会话模式（对话 / 团队 / 工作流…）
class Message:                  # :759 统一帧模型：type / method / params / request_id / event / payload
```

> 复核提示：贵方若在**产品仓**里搜 `WORKSPACE_TREE =` 会搜不到——它不在协议仓里，而是产品仓与 Gateway 各自在导入期补的兼容成员（见 9.2）。这类「协议里没有」不等于「功能没有」。

### 9.2 兼容扩展：产品仓与 Gateway 在导入期给协议枚举补成员

```python
# ── 产品仓侧：jiuwenswarm/common/schema/message.py:151-153
# 作用：把只在产品侧存在的方法补进协议枚举，使同一份分发代码无需分支即可识别。
_ensure_req_method("WORKSPACE_TREE",    "workspace.tree")
_ensure_req_method("WORKSPACE_PREVIEW", "workspace.preview")
_ensure_req_method("WORKSPACE_USAGE",   "workspace.usage")

# ── Gateway 侧：agent_gateway/common/protocol_compat.py:82
# 同一张兼容表（("WORKSPACE_TREE", "workspace.tree") 等），保证两侧对同一 method 的解析一致。
```

### 9.3 北向私有协议（浏览器 / 客户端 ↔ Gateway）

```python
# ── 请求帧（客户端 → Gateway），实测形状
# 来源：TUI 前端 channels/tui/frontend/src/app-state.ts:2340
{
  "type": "req",
  "method": "chat.interrupt",          # 取值为 ReqMethod 的枚举值
  "request_id": "<客户端生成，用于幂等>",
  "params": { "intent": "cancel" | "supplement" | "pause", "mode": ..., "new_input": ..., "attachments": [...] }
}

# ── Gateway 入口（把 wire 字符串转成协议枚举后分发）
# agent-gateway/agent_gateway/channel_manager/web/web_connect.py:1742
async def dispatch_client_request(ws, method: str, params: dict, ...) -> None: ...
# channel_manager/web/lifecycle_handlers.py:23   req_method = ReqMethod(method)
# channel_manager/web/invoke.py:114              dispatch_web_request(...)

# ── 事件帧（Gateway → 客户端）
# web_connect.py:580
async def send_event(ws, event: str, payload: dict,
                     seq: int | None = None, stream_id: str | None = None) -> None: ...
# web_connect.py:1093  只发给「订阅了该会话」的连接
async def send_session_event(session_id: str, event: str, payload: dict) -> None: ...
```

### 9.4 内部私有协议（Gateway ↔ AgentServer）与 REST 组装

```python
# ── E2A 信封（内部链路，WebSocket）
# jiuwenswarm/common/e2a/models.py
class E2AEnvelope: ...            # method / params / metadata.routing / request_id ...
# jiuwenswarm/common/e2a/constants.py
E2A_CANCEL_SOURCE_CLIENT_DISCONNECT   # 断连来源标记，见 §10
# 帧上限：AGENT_WS_MAX_MESSAGE_BYTES = 8 MiB；单轮发送预算 AGENT_WS_SEND_BUDGET_BYTES = 6 MiB

# ── 同一批 method 的 REST 组装表（便于贵方用 HTTP 接入而非 WS）
# jiuwenswarm/extensions/runtime_management_extension/pod_rest_map.py:29  _ROUTE_ROWS（300+ 行）
API_PREFIX = "/api/v1"
DEFAULT_HTTP_PORT = 8766
#   ("chat.interrupt",        "POST", "/chat/{session_id}/actions/interrupt")
#   ("chat.user_answer",      "POST", "/chat/{session_id}/actions/answer")
#   ("session.fork",          "POST", "/sessions/{session_id}/actions/fork")
#   ("workspace.tree",        "GET",  "/workspace/tree")
#   ("workspace.download",    "GET",  "/workspace/download")
#   ("history.list_turns",    "GET",  "/sessions/{session_id}/turns")
#   ("file.read_upload_chunk","GET",  "/sessions/{session_id}/files/upload-content")

# ── 北向 REST（浏览器侧，与 WS 等价）
# agent_gateway/channel_manager/web/http_routes/mapped.py:819-833
("GET", "/workspace/tree",   "workspace.tree",   "workspace")
("GET", "/workspace/usage",  "workspace.usage",  "workspace")
("GET", "/workspace/preview","workspace.preview","workspace")

# ── 两条旁路通道（同端口不同 path）
# /file-api/*   agent_gateway/channel_manager/web/container_file_http.py:620 attach_container_file_routes()
# /ws/git       agent_gateway/channel_manager/web/git_ws_handler.py（见 §19）
```

### 9.5 握手、连接角色与来源校验

```python
# ── AgentServer 侧接入处理（server/front/connection.py）
def process_handshake(...) -> ...      # :36  Origin 白名单校验（agent_gateway.common.security.ws_origin），不通过 → forbidden_origin_response
def connection_role(ws) -> str         # :57  读 CONNECTION_ROLE_HEADER；rpc 角色不占用推送槽
def connection_ack_frame(readiness_state=..., heartbeat_job_ready=...)  # front/protocol.py:138，连上即回 connection.ack
# 就绪状态：AGENT_READY / DEGRADED → heartbeat_job_ready=True
```

### 9.6 注意事项
- **方法名以协议仓枚举为准**：贵方若自行拼字符串，建议直接引用 `ReqMethod` / `EventType` 枚举值，避免历史别名（如 `chat.cancel` 仅作为个别内部路由列表的遗留写法存在于 `runtime_routed_client.py:43-44`、`agent_ws_server.py:3441`）造成歧义。
- **兼容扩展是双侧的**：新增方法需同时在产品仓 `common/schema/message.py` 与 Gateway `protocol_compat.py` 登记，否则一侧解析失败。
- **REST 与 WS 等价但覆盖面不同**：REST 表覆盖绝大多数方法，若贵方需要的方法只在 WS 上，可提出，我们按 `_ROUTE_ROWS` 的格式补一行即可。

**源码位置**：`agent-protocol/openjiuwen-protocol/openjiuwen_protocol/message.py:83-624, 625-688, 689, 759`；`jiuwenswarm/common/schema/message.py:151-153`；`agent_gateway/common/protocol_compat.py:82`；`agent_gateway/channel_manager/web/{web_connect.py:580,1093,1742,1910,1989,2079,2111, invoke.py:114, lifecycle_handlers.py:23, http_routes/mapped.py:819-833, container_file_http.py:620}`；`jiuwenswarm/extensions/runtime_management_extension/pod_rest_map.py:22,29`；`jiuwenswarm/server/front/{connection.py:36,57,156, protocol.py:138}`；`jiuwenswarm/common/e2a/{models.py, constants.py}`。

---

## 10. 断线重连：客户端重连与会话恢复机制；`request_id` / `session_id` 幂等与恢复语义

### 结论
**支持。** 需要区分两件事：**服务接口重试**与**客户端连接恢复 + 任务续接**是两套机制，产品侧对后者有明确设计——连接断了不丢任务，语义上按「暂停」处理并可续跑；只有用户主动终止才真正结束。幂等目前做在 `request_id` 维度。

### 10.1 连接层：断线与重连的换绑（AgentServer 侧）

```python
# ── 推送连接管理（server/front/connection.py:66 class ConnectionHandler）
# 规则：按到达顺序保存可推送连接，最新一条持有「推送槽」；
#       该连接关闭后，槽位自动升到「仍存活的最新一条」；
#       只有当没有任何可推送连接时，才触发运行时断连清理。
def attach_gateway_connection(ws, send_lock) -> None    # :107 新连接接管推送槽
def _release_push_target(ws, remote) -> None            # :112-140 关闭时升位，日志「push 通道切换到仍存活的连接」
def on_gateway_disconnect(...) -> None                  # :139 仅在无可用推送连接时执行
# 换绑落点：server/front/admission.py:31 / server/supervisor/runtime_supervisor.py:256 / server/worker/dispatcher.py:89
# 事件转发器同步换绑：server/front/event_forwarder.py  EventForwarder.attach(ws, send_lock) / .detach(ws)
```

### 10.2 轮次层：断连 ≠ 终止（恢复语义的核心）

```python
# ── 断连来源标记（内部字段，贵方无需设置，Gateway 断连时自动带上）
# jiuwenswarm/common/e2a/constants.py
E2A_INTERNAL_CANCEL_SOURCE_KEY  = <metadata 键>
E2A_CANCEL_SOURCE_CLIENT_DISCONNECT = <来源取值>

# ── 语义分叉（server/runtime/agent_adapter/interface.py:3351-3360）
# 断连兜底（client_disconnect）→ workflow_disposition = "pause"：保留执行账本，可冷启动续跑
# 用户主动终止              → workflow_disposition = "stop" ：落终态（seal），不再续跑
await team_manager.cancel_session_runtime(session_id, reason=..., workflow_disposition="pause" | "stop")

# ── 断连清理入口（server/handlers/chat.py）
def _is_client_disconnect_cancel_request(request) -> bool   # :24-28 读 metadata 中的来源标记
cleanup_after_cancel = _is_client_disconnect_cancel_request(request)   # :90
await services.cleanup_client_disconnect_session_runtime(request)      # :131
# 会话状态取值：server/runtime/session/model.py:128 CLIENT_DISCONNECTED = "client_disconnected"
```

### 10.3 恢复入口与断点续跑

```python
# ── 恢复被暂停的一轮（客户端重连后调用）
ReqMethod.CHAT_RESUME = "chat.resume"        # message.py:88；实测调用：TUI app-state.ts:2438 sendEventOnly("chat.resume", {})
# 与 chat.send / chat.user_answer 同列为「真实 chat 轮」：server/worker/dispatcher.py:103、server/runtime/session/session_metadata.py:1497

# ── 文件与会话状态恢复
session.rewind_and_restore / session.restore_files     # 见 §18
# 运行时检查点：checkpoint journal 落盘，支持从断点继续（见 §12 注意事项）

# ── 重启后在途消息的保守处理
# server/runtime/session/session_message_store.py:809
def recover_inflight_as_unknown(self) -> int   # 把「在途」消息标为 unknown，不猜测成功或失败
# 调用点：session_message_service.py:160
```

### 10.4 幂等语义（贵方要求的 `request_id` / `session_id`）

```python
# ── request_id 幂等（server/front/admission.py:74 ExecutionAdmission.dispatch）
RequestRegistry.register(request_id, queued=False) -> bool    # :93 已存在 → 返回 :96 DUPLICATE_REQUEST
# 排队闸门：_DEFAULT_QUEUE_LIMIT = 32，_DEFAULT_WAIT_SECONDS = 120.0（超限直接拒绝，不排队）

# ── session_id 侧的幂等约束
# pod_rest_map.py:22 _METHODS_WITHOUT_PARAM_SESSION_ID = {"session.create"}
# 语义：session.create 不接受 params.session_id；恢复已有会话只能 session.switch
#   ⇒ 客户端无法「凭一个 session_id 造出第二个会话」，这是会话维度的幂等保证
```

### 10.5 注意事项
- **事件补偿需扩展实现**：Gateway 不为断线期间的事件建立重放缓冲，也没有 `last_seq` / catch-up 请求（全仓检索无此类入口）。重连后建议按 §13 的方式用 `history.get` 重新拉取，再续订事件。
- **`session_id` 维度不做「重复请求去重」**：同一会话重复发 `chat.send` 会各起一轮，去重请在 `request_id` 上做。
- **断连后是否取消在跑的一轮由 Gateway 决定**：Gateway 检测到无可用推送连接后，会带 `client_disconnect` 来源发起取消；此时运行时按 `pause` 处理，贵方若希望「断连也继续跑完并把结果落库」，可通过接入 §2.2 的会话生命周期参与者自定义该行为。

**源码位置**：`jiuwenswarm/server/front/{connection.py:36,57,66,96-140,156, admission.py:31,34,74,93,96, event_forwarder.py, protocol.py:138}`；`jiuwenswarm/server/supervisor/runtime_supervisor.py:256,262`；`jiuwenswarm/server/worker/dispatcher.py:89,103`；`jiuwenswarm/server/handlers/chat.py:15,24,28,90,131`；`jiuwenswarm/server/runtime/agent_adapter/interface.py:3351-3360,6308`；`interface_deep.py:17153,17166`；`jiuwenswarm/server/runtime/session/{model.py:128, session_message_store.py:809, session_message_service.py:160, session_metadata.py:1497}`；`jiuwenswarm/common/e2a/constants.py:19,56`。

---

## 11. 用户信息透传：tool context 注入 user / token / agent_key；标准 context 字段、签名校验与过期机制

### 结论
**支持**（用户身份透传、签名校验、过期机制三项都有）。需要说明的是：产品侧透传给工具的是**可信的用户路由键**（`user_id` / `session_id` / `request_id`），**不把用户凭证下发到工具**；`agent_key` 这一字段在产品中不存在，若贵方安全要求是「工具侧可校验的调用方身份」，属**需扩展实现**（本文给出建议路径）。

### 11.1 工具上下文里的用户字段（标准 context 字段）

```python
# ── 工具上下文与路由键（agents/harness/common/tools/cron/cron_runtime.py）
class CronToolContext:
    user_id: str          # :104-106 由 deep adapter 从 E2A 请求的 user_id 注入，供 job 归属创建者
    # 企业版群 / 机器人场景：回退读顶层 user_id + metadata.routing（:107-112）
class CronToolRoute:      # :291 构造
    user_id: str; request_id: str; session_id: str; channel_id: str
# 归属校验：:299  if str(job.get("user_id")) != user_id → 拒绝（跨用户操作被挡）
# 注释原文（:46）：保证 AgentOS 下仍携带 user_id / session_id / request_id 路由键

# ── 服务端可信运行时绑定（客户端不可伪造）
# server/runtime/agent_adapter/interface.py:5987-5999
if request.req_method is not ReqMethod.CHAT_SEND:
    request.trusted_runtime_binding = None                       # :5989 / :5993
else:
    runtime_binding = adapter.resolve_runtime_binding(session_id, params)
    request.trusted_runtime_binding = runtime_binding            # :5999 服务端写入
# 消费点：interface_deep.py:18605 / :19384  getattr(request, "trusted_runtime_binding", None)
```

### 11.2 签名校验与过期机制（下载令牌，HMAC-SHA256）

```python
# ── 令牌协议（jiuwenswarm/common/file_download_token.py 文档字符串）
# 格式：Base64URL(payload_json) + "." + Hex(HMAC-SHA256(payload_json, secret))
# 特点：跨进程可验（AgentServer / Gateway / app_web.py 无需共享内存）
_DEFAULT_EXPIRES_SECONDS = 600          # :36 默认 10 分钟
_SECRET_ENV_KEY   = "JIUWENSWARM_FILE_DOWNLOAD_SECRET"   # :37 至少 32 字符
_SECRET_FILE_NAME = ".file_download_secret"              # :38 未配环境变量时自动生成并落盘，os.chmod(0o600)

class WebFileDownloadManager:           # :79 单例 get_instance() / reset_instance()
    def __init__(self, asset_owner: VerifiedDownloadAssetOwner): ...
# payload 绑定的字段随用途不同：
#   普通下载      → path, sid（+ 可选 exp）
#   路由下载      → path, sid, download_http_base（_ROUTED_DOWNLOAD_TOKEN_KEYS）
#   技能配图      → purpose="skill_content_image", name, version, relative_path, exp, sid
#   受管资产      → asset_id, size, digest, name（kind = verified_asset_v1，:41）
# 说明：send_file_to_user 的交付产物可省略 exp（即不过期），因为它是用户主动索取的成品文件

# ── 校验侧（Gateway 文件通道）
# agent_gateway/channel_manager/web/container_file_http.py
def _decode_download_token_payload(token) -> dict | None    # :397
def _decode_download_token_location(token) -> ...           # :408
def _resolve_user_id(...) -> str                            # :208 显式参数 → 请求头 x-user-id → 令牌 username
def _user_id_mismatch_against_token(...) -> bool            # :190 与令牌身份比对
def _user_mismatch_response(...)                            # :182 → 403 code=USER_MISMATCH
```

### 11.3 内部链路的双向认证（mTLS）

```python
# ── Gateway ↔ AgentServer 内部链路 mTLS（agent_gateway/common/security/link_mtls.py）
class LinkMTLSConfig:      # :41
    mode: LinkMTLSMode     # off | observe | enforce（环境变量控制，:53）
    identity: MTLSDeploymentIdentity | None    # :48 部署身份
@classmethod
def from_env(cls, *, role: str = "gateway") -> "LinkMTLSConfig": ...   # :51
# observe 模式：预检证书材料但不改变业务传输（:69-71），便于灰度上线
# 实现来自 agent-runtime 的 openjiuwen_runtime.foundation.security.link_mtls_config（:20-23）
```

### 11.4 注意事项
- **没有 `agent_key` / 用户 token 下发到工具**：工具上下文携带的是路由键，工具无法读取用户凭证——这是有意的最小权限设计。若贵方要求「工具调用链上可校验的调用方身份」，建议路径：由集成方在工具后端（MCP server / HTTP 工具）侧自行校验由 Gateway 注入的 mTLS 客户端证书或自定义签名头，产品侧已具备承载该头的链路（§11.3）。
- **北向（浏览器 ↔ Gateway）不含 JWT/JWKS 校验模块**：`agent_gateway/common/security/` 目前只有 `ws_origin.py`（Origin 白名单）与 `link_mtls.py`。IAM 令牌身份由外部接入层注入后，产品侧只做**一致性比对**（§11.2 的 `USER_MISMATCH`）。若贵方要求 Gateway 直接验签 JWT，属**需扩展实现**。
- **令牌默认 600 秒过期，但交付产物类令牌可不过期**：若贵方安全基线要求「所有令牌必须短时效」，可通过不给 `send_file_to_user` 走免过期路径实现（我们侧可配置）。

**源码位置**：`jiuwenswarm/agents/harness/common/tools/cron/cron_runtime.py:9,46,48-58,64,104-112,125,288,291,299`；`jiuwenswarm/server/runtime/agent_adapter/interface.py:5987-5999`；`interface_deep.py:18605,19384`；`jiuwenswarm/common/file_download_token.py:36,37,38,41,49,57,79`；`jiuwenswarm/common/verified_download_assets.py`；`agent_gateway/common/security/{link_mtls.py:20-23,41-71, ws_origin.py}`；`agent_gateway/channel_manager/web/container_file_http.py:95,146,152,169,182,190,208,397,408`。

---

## 12. 重连或切换节点后的恢复：会话恢复、任务查询、取消与幂等协议

### 结论
**支持（需配置）。** 会话级恢复、取消、幂等三项协议完整；跨节点靠路由层把请求送到会话所在节点。**「任务查询」需要说明粒度**：产品没有独立的 `task_id` 概念，可查询对象是**会话**（含其运行时状态、命令状态与进度事件），若贵方需要「按任务 ID 查询 + 列举」的接口，属**需扩展实现**。

### 12.1 会话恢复与状态查询

```python
# ── 重新接回一个已有会话
ReqMethod.SESSION_SWITCH = "session.switch"
#   REST：POST /sessions/{session_id}/actions/switch
#   注册：server/dispatch.py:224  ReqMethod.SESSION_SWITCH: handle_session_switch
#   实现：server/handlers/session.py:732

# ── 查会话元数据与运行时状态（贵方关心的「这个会话现在在干什么」）
ReqMethod.SESSION_GET_METADATA = "session.get_metadata"
#   REST：GET /sessions/{session_id}
#   实现：server/handlers/session.py（读 server/runtime/session/session_metadata.py）

# ── 命令类任务的状态查询（有明确状态可查的那一类）
#   REST：GET /sessions/{session_id}/commands/status
#   协议：ReqMethod.COMMAND_STATUS = "command.status"

# ── 历史重建（重连后补齐上下文）
ReqMethod.HISTORY_GET = "history.get"          # REST GET /sessions/{session_id}/history，cursor 分页
ReqMethod.HISTORY_LIST_TURNS = "history.list_turns"   # REST GET /sessions/{session_id}/turns
```

### 12.2 取消协议（含来源区分）

```python
# ── 取消 / 暂停 / 补充（同一接口，intent 决定语义）
ReqMethod.CHAT_CANCEL = "chat.interrupt"       # REST POST /chat/{session_id}/actions/interrupt
#   params: { intent: "cancel" | "supplement" | "pause", target_team_id?, target_request_id?, wait_for_stop? }
#   先停 producer 再停响应流；带 target_* 时只停指定 Team
# 断连来源与用户主动终止的语义差异见 §10.2（pause 保账本 / stop 落终态）
```

### 12.3 跨节点：把请求路由到会话所在节点

```python
# ── Gateway 侧按会话定位归属
# agent-gateway/agent_gateway/routing/session_sharing.py:420
async def dispatch_to_session(...) -> ...      # :453 实际 dispatch()

# ── 产品侧按会话路由到目标 AgentServer 实例（pod）
# jiuwenswarm/extensions/runtime_management_extension/runtime_routed_client.py:43-44  可路由方法集合
# jiuwenswarm/extensions/runtime_management_extension/pod_rest_map.py:448  if method in {"chat.send", "chat.resume"} …
```

### 12.4 优雅下线与准入（避免切流期间接新活）

```python
# ── 准入闸门（server/front/admission.py:74）
ReadinessState.DRAINING → 返回错误码 RUNTIME_DRAINING     # 正在下线，不再接新请求
ReadinessState.FAILED   → 返回错误码 RUNTIME_FAILED
队列：_DEFAULT_QUEUE_LIMIT = 32，_DEFAULT_WAIT_SECONDS = 120.0
# 恢复路径：server/supervisor/runtime_supervisor.py（失败后重建运行时）
```

### 12.5 注意事项
- **没有 `task_id` 与「任务列举」接口**：可查询粒度是会话。若贵方运维系统需要「按任务 ID 查状态 / 列出运行中任务」，建议以 `session.get_metadata` + `command.status` + `chat.processing_status` 事件组合实现，或提出由我们补一层任务视图。
- **检查点（checkpoint journal）是节点本地文件**：跨节点续跑需要共享存储（NFS 等，见八项答复 §5.1）；否则切换节点后只能恢复会话与历史，不能从断点继续执行。
- **租户运行时池不做跨节点迁移**：池是进程内缓存，节点故障后由客户端重连 + `session.switch` 在新节点重建，产品侧没有自动故障转移编排。

**源码位置**：`jiuwenswarm/server/dispatch.py:212,224,228-231`；`jiuwenswarm/server/handlers/session.py:732,892,1091,1215-1236`；`jiuwenswarm/server/runtime/session/session_metadata.py:1497`；`jiuwenswarm/extensions/runtime_management_extension/{runtime_routed_client.py:43-44, pod_rest_map.py:22,29,448}`；`agent-gateway/agent_gateway/routing/session_sharing.py:420,453`；`jiuwenswarm/server/front/admission.py:74,93,96`；`jiuwenswarm/server/supervisor/runtime_supervisor.py`。

---

## 13. 服务端主动推送：订阅、推送、断线补偿与事件序号

### 结论
**支持。** 事件推送面完整（50+ 事件），并支持**按会话定向**推送。**事件序号与断线补偿需扩展实现**：协议里已预留 `seq` / `stream_id` 字段，目前只有轨迹（trace）通道填充了序号，普通事件不填充，也没有重放缓冲。

### 13.1 事件清单（协议真源，节选贵方关心的部分）

```python
# agent-protocol/openjiuwen-protocol/openjiuwen_protocol/message.py:625-688  class EventType(Enum)
# 对话流：chat.delta / chat.reasoning / chat.final / chat.retract / chat.error
#         chat.tool_call / chat.tool_update / chat.tool_result
#         chat.media / chat.file / chat.ask_user_question / chat.session_result
# 用量：  chat.usage_metadata / chat.usage_summary / context.usage
# 状态：  chat.processing_status / chat.interrupt_result / runtime.accepted / execution.error
#         chat.symphony_status / chat.evolution_status / chat.subtask_update / chat.subagent_activity
# 会话：  session.archived / session.unarchived / session.deleted / session.lifecycle.updated
# 项目：  project.lifecycle.updated / project.removed / project.restored
# 团队：  team.member / team.task / team.message
# 画布：  designer.run.updated / designer.node.updated / designer.graph.updated / designer.leader.activity
# 其他：  goal.snapshot / goal.updated / todo.updated / plan.approval_required / workflow.updated
#         history.message / proactive_recommendation / health_check.relay / trace.updated
```

### 13.2 订阅模型：连接注册 + 按会话定向

```python
# ── 连接与会话的绑定即「订阅」（Gateway 侧，无独立 subscribe RPC）
# agent_gateway/channel_manager/web/web_connect.py
await self.register_ws(ws, routing_key)     # :1910 / :2079 连接建立时登记（含会话路由键）
await self.unregister_ws(ws)                # :1989 / :2111 断连 finally 清理
async def send_session_event(session_id, event, payload)   # :1093
    for ws in self._clients_for_session(session_id):       # 其他会话的连接收不到
        await self.send_event(ws, event, payload)

# ── 显式订阅只有 /ws/git 一处（见 §19）
#   diff_watch / diff_files_watch / diff_detail_watch / diff_unwatch
```

### 13.3 事件序号：字段已在协议里，目前只有轨迹通道填充

```python
# ── 出站帧签名（字段已预留）
# web_connect.py:580-592 / outbound.py:176-181
async def send_event(ws, event, payload, seq: int | None = None, stream_id: str | None = None)
    if seq is not None: frame["seq"] = seq          # :591
# ⚠️ 普通 chat / session 事件调用时 seq 与 stream_id 传 None（git 推送通道注释两处写明）

# ── 已填充序号的通道：trace.updated（轨迹更新提示）
# web_connect.py:1035-1049  待发更新按 (revision, frame_seq) 双水位合并，新帧可压过同记录的旧提示
# web_connect.py:1066-1077  payload = {session_id, trace_id, revision, frame_seq, store_epoch, lifecycle}
# web_connect.py:1053       _drain_trajectory_updates()：合并窗口 _TRAJECTORY_HINT_COALESCE_SECONDS，避免并发发送积压
```

### 13.4 载荷裁剪（哪些事件保留完整 payload）

```python
# web_connect.py:1127 _should_preserve_full_payload(event_name) -> bool
# 保留全量：team.* / designer.* / org.* / a4p.* / harness.* / personal_context.context.* /
#           cron.run.updated / chat.input_received / chat.output_phase / video.core.result
# 其余事件按 {session_id, content} 精简，减小帧体积
```

### 13.5 注意事项
- **断线补偿需扩展实现**：Gateway 不缓存事件、不重放，也没有 `last_seq` / catch-up 请求（全仓检索无此类入口）。重连后的推荐做法：`history.get`（cursor 分页）重建消息 → 需要 diff 的再重新订阅 `/ws/git` 拿快照。
- **事件序号建议按会话自行补齐**：若贵方客户端要做「漏事件检测」，可在应用层用 `history.get` 的消息序号与本地水位对比；产品侧协议字段 `seq` 已预留，我们可按贵方要求把某类事件的 `seq` 填起来（改动集中在 `send_event` 调用点）。
- **精简载荷的事件不含全量字段**：若贵方二次开发依赖完整结构（例如自研看板），请按 13.4 的白名单确认事件是否在保留列表内，不在的话可提出加入。

**源码位置**：`agent-protocol/.../message.py:625-688`；`agent_gateway/channel_manager/web/{web_connect.py:580-592,746-758,1025-1093,1093-1110,1127,1910,1989,2079,2111, outbound.py:176-181}`；`agent_gateway/channel_manager/web/git_ws_handler.py:13`；`agent_gateway/channel_manager/web/git_diff_watcher.py:194`。

---

## 14. 大工具结果处理：大结果、二进制 artifact、分页；结果大小限制与下载接口

### 结论
**支持。** 三件事都有对应机制：推送侧对工具结果做**硬截断并显式标记**；大产物转成**受管下载资产**（带摘要与签名令牌）+ **分片下载接口**；列表与历史侧都有**分页**。缺的一项：被截断的工具结果没有「分页取全文」的接口（需扩展实现）。

### 14.1 推送侧：工具结果大小限制与截断标记

```python
# ── 内核事件轨上的硬闸门（jiuwenswarm/agents/harness/common/rails/stream_event_rail.py）
_TOOL_RESULT_TEXT_LIMIT = 60000          # :87  单位：字符
"result": str(result)[:_TOOL_RESULT_TEXT_LIMIT]              # :1439
tool_result_payload["rendered_result"] = rendered_result[:_TOOL_RESULT_TEXT_LIMIT]   # :1442
tool_result_payload["raw_output_truncated"] = True           # :1447 显式告知前端「原文被截断」
# 事件名：EventType.CHAT_TOOL_RESULT（chat.tool_result）
```

### 14.2 二进制 artifact：交付工具 → 受管资产 → 下载接口

```python
# ── 模型侧交付工具（产物落盘并生成可下载引用）
# jiuwenswarm/agents/harness/common/rails/permissions/_auto_permission/artifact_authorization.py
_FILE_DELIVERY_TOOLS = frozenset({"send_file_to_user", "upload_file", "upload_photo"})   # :34
# :96 / :118 对 send_file_to_user 单独判定权限（产物交付属于敏感动作）
# 提示词侧明确教模型「要下载/导出/交付文件时调 send_file_to_user，传服务端可访问的绝对路径」
#   （agents/harness/common/prompt/prompt_builder.py:167,238）

# ── 受管下载资产（引用而非裸路径）
# jiuwenswarm/common/verified_download_assets.py
class VerifiedDownloadAsset:  asset_id; size; digest; name
class VerifiedDownloadAssetOwner: ...
# 令牌 kind = verified_asset_v1（common/file_download_token.py:41），payload 绑 asset_id/size/digest/name

# ── 下载接口（两条）
ReqMethod.FILE_DOWNLOAD_VERIFIED_CHUNK = "file.download_verified_chunk"
#   实现：server/runtime/gateway_adapter/workspace_file_adapter.py:187 → :188 _handle_verified_download_chunk(request)
#   单片上限 _VERIFIED_DOWNLOAD_CHUNK_MAX_BYTES = 512 KiB
#   语义：Gateway 不得凭路径自行授权这类资产（Smart Approval 密封资产），只能按资产引用取
HTTP：GET /file-api/download（支持 Range → 206 / 416，见八项答复 §8.2）

# ── 会话资产清单（artifact 的引用面）
ReqMethod.SESSION_ASSETS_LIST / SESSION_ASSETS_REGISTER / SESSION_ASSETS_RENAME   # message.py:386-388
# 训练产物另有：rsi.artifact.download / rsi.files.list / rsi.files.get（message.py:593-595）
```

### 14.3 分页参数（各列表接口实测口径）

```python
# ── 会话列表：limit / offset
# server/handlers/session.py:462 raw_limit = params.get("limit")；:470 raw_offset = params.get("offset")
# ── 历史消息：cursor 协议 + 默认页大小
# server/handlers/session.py:1215 params["limit"] = _coerce_query_int(...)
#                          :1216 空 cursor 归一；:1224 request_cursor = params.get("cursor") if cursor_protocol
#                          :1236 limit=params.get("limit", _HISTORY_PAGE_SIZE)
# ── 轮次列表：history.list_turns（REST GET /sessions/{session_id}/turns）
# ── 定时任务列表：offset / limit（默认 500）      server/handlers/schedule.py:121-122
# ── 团队列表：cursor（兼容 offset）/ limit        server/handlers/team.py:885-891
# ── 检索分页：personal_context.context.search_pages（message.py:450）
# ── 命令列表：offset（0..10_000_000）/ limit      server/handlers/commands.py:250-253
```

### 14.4 大小与数量限制（可直接写进验收用例）

| 场景 | 限制 | 出处 |
|---|---|---|
| 工具结果推送文本 | 60000 字符，超出标 `raw_output_truncated` | `stream_event_rail.py:87` |
| 内部 WS 单帧 / 单轮发送预算 | 8 MiB / 6 MiB | `AGENT_WS_MAX_MESSAGE_BYTES` / `AGENT_WS_SEND_BUDGET_BYTES` |
| 北向 WS 单帧 | 150 MiB | `WEB_WS_MAX_MESSAGE_BYTES` |
| 受管资产分片 | 512 KiB | `_VERIFIED_DOWNLOAD_CHUNK_MAX_BYTES` |
| 工作空间下载 | 100 MiB | `_DEFAULT_DOWNLOAD_MAX_BYTES` |
| 单次上传文件数 | 20 | `MAX_UPLOAD_COUNT` |
| HTTP 流式块 | 64 KiB | `_STREAM_CHUNK` |

### 14.5 注意事项
- **被截断的工具结果没有「取全文」分页**：目前做法是标记 `raw_output_truncated` 后由模型改用 `send_file_to_user` 交付完整文件。若贵方要求「按 offset 分页读工具结果」，属**需扩展实现**（可复用 `file.read_upload_chunk` 的分块语义）。
- **二进制 artifact 没有独立对象引用协议**：走「文件 + 签名令牌 + 资产摘要」组合，没有类似 `artifact_id → 元数据 + 多段 URL` 的对象服务协议。
- **大结果不进事件帧**：超过帧预算的内容不会挤进 `chat.tool_result`，而是走 14.2 的资产路径，这一点在压测时请按资产通道单独计量。

**源码位置**：`jiuwenswarm/agents/harness/common/rails/stream_event_rail.py:87,105,1439,1442,1447`；`jiuwenswarm/agents/harness/common/rails/permissions/_auto_permission/artifact_authorization.py:34,96,118`；`jiuwenswarm/agents/harness/common/prompt/prompt_builder.py:167,169,174,238,244`；`jiuwenswarm/common/{verified_download_assets.py, file_download_token.py:41}`；`jiuwenswarm/server/runtime/gateway_adapter/workspace_file_adapter.py:187-188`；`jiuwenswarm/server/handlers/{session.py:462,470,1215-1236, schedule.py:121-122, team.py:885-891, commands.py:250-253}`；`agent-protocol/.../message.py:386-388,450,593-595`。

---

## 15. 多模态输入：支持的模态、编码、限制与错误码

### 结论
**支持。** 图片、文档、语音、视频四类模态都有接入接口；限制以「尺寸上限 + 扩展名黑名单 + 文件名安全化」三重方式实现；错误通过结构化错误码返回。

### 15.1 模态与接入接口

```python
# ── 图片（浏览器上传 base64 → 服务端解码落盘）
ReqMethod.MEDIA_PERSIST  = "media.persist"     # message.py:256  落盘到 agent/sessions/<session_id>/uploads
ReqMethod.MEDIA_DISCARD  = "media.discard"     # message.py:257  删除未发送的副本（只允许删该目录内普通文件）
# 实现：jiuwenswarm/server/runtime/attachments/media_attachments.py（复用 normalize_chat_media_attachments）
# MIME → 后缀：agent_gateway/common/attachment_safety.py  image_suffix_for_mime（attachments/steer_images.py:17 引用）
# 发送时：chat.send 带 attachments → 事件 chat.media 回推

# ── 文档（base64 或可展开路径 → 落盘并解析）
ReqMethod.DOCUMENT_PERSIST = "document.persist"   # message.py:258
ReqMethod.DOCUMENT_FORMATS = "document.formats"   # message.py:259  返回 forbidden_formats() 供前端提示
# 实现：jiuwenswarm/server/runtime/attachments/document_attachments.py
def persist_and_parse_documents(params, ...) -> ...        # :70 起
def is_supported_document(*, filename) -> bool             # :52  非黑名单即支持

# ── 语音 / 视频
# jiuwenswarm/server/agent_ws_server.py:1369  VoiceTaskServerAdapter(...)
ReqMethod.VOICE_TASK_CHECKPOINT_ACK / VOICE_TASK_FILES     # message.py:280-281
# 能力查询：designer.media.capabilities（message.py:240）

# ── 外部来源（URL / IM 附件）
ReqMethod.FILE_IMPORT_URL   = "file.import_url"    # message.py:262  chat.send 带外部 URL，由 AgentServer 下载落盘（Gateway 不落盘）
ReqMethod.IM_FILE_PERSIST   = "im.file_persist"    # message.py:271  Gateway 取字节后 base64 交 AgentServer 落盘到 <平台>_files/downloads/
```

### 15.2 编码与传输约定

```python
# ── 上行编码：base64（浏览器上传）
# document_attachments.py:9  字段名兼容 base64Data / base64_data
# :32 注释明确：大小上限按「base64 膨胀前的裸字节」计
# ── 转发前剥掉 base64（关键实现约定）
def _strip_document_base64(params) -> None    # :189-207
# :193 注释原文：保留 base64_data 会把内部 WebSocket 帧撑爆
# ── 分块上传（大文件不走单帧）
ReqMethod.FILE_UPLOAD_CHUNK = "file.upload_chunk"
# message.py:262-270 注释说明：用于 AgentOS 多用户场景的大文件，避免单个 E2A WebSocket 帧超过限制
```

### 15.3 限制清单

| 项 | 取值 | 出处 |
|---|---|---|
| 文档单文件（base64 裸字节） | 100 MiB | `document_attachments.py:32 _MAX_DOCUMENT_BYTES` |
| 单次文档条数 | 8（与图片张数对齐） | `document_attachments.py:34 _MAX_DOCUMENT_COUNT` |
| 单次上传文件总数 | 20 | `MAX_UPLOAD_COUNT` |
| 文件名长度 | 180 UTF-8 字节（保留扩展名） | `attachment_safety.py:33 _MAX_FILENAME_BYTES` |
| 会话目录名长度 | 120 字符 | `attachment_safety.py:34 _MAX_SESSION_ID_CHARS` |
| 格式策略 | **黑名单**：可执行 / 脚本 / 包类扩展名拒绝 | `attachment_safety.py:91 FORBIDDEN_DOCUMENT_EXTENSIONS` |
| 文件名清洗 | 控制字符、`<>:"/\|?*`、 bidi 控制字符、Windows 保留名 | `attachment_safety.py:14,19,23,25` |

```python
# ── 拦截判定（Gateway 与产品仓共用同一份实现）
def forbidden_formats() -> list[str]                       # :126  给前端的可展示清单
def is_forbidden_document(*, filename=None, suffix=None) -> bool   # :136
def safe_upload_filename(filename, *, fallback) -> str     # :71   清洗 + 截断
def safe_session_dirname(...) -> str
```

### 15.4 错误码

```python
# ── 工作空间 / 文件域：结构化错误码
class WorkspaceError(Exception):
    code: str          # 由 handler 转成响应错误码
# 落点：server/runtime/gateway_adapter/workspace_file_adapter.py:231  WorkspaceError → build_error_response(request, exc.message, code=exc.code)
# ── HTTP 文件通道：传输错误码 → HTTP 状态
def _status_for_transfer_code(code) -> int     # container_file_http.py:95
# 身份不一致 → 403 code=USER_MISMATCH（见 §11.2）
# Range 不可解析 → 416 + Content-Range: bytes */total
# ── 文档被拒：文案带可展示清单
# document_attachments.py:273 / :321  f"Forbidden: {forbidden_formats()}"
```

### 15.5 注意事项
- **格式策略是黑名单不是白名单**：可执行 / 脚本 / 包类被拦，其余扩展名放行。若贵方安全基线要求「只允许明确列出的格式」，需把策略反转为白名单，属**需扩展实现**（改动点集中在 `is_forbidden_document`）。
- **没有按模态的细粒度限制**：例如图片分辨率 / 像素、音视频时长与采样率、PDF 页数都没有服务端硬限制；目前只有字节上限。若贵方要按模态限流，属**需扩展实现**。
- **没有统一的多模态错误码表**：文件域走 `WorkspaceError.code`，HTTP 通道走 `_status_for_transfer_code`，文档拒绝走文案。若贵方要求「一份多模态错误码对照表」，我们可按贵方网关规范整理并补齐。
- **语音 / 视频链路依赖具体模型能力**：`designer.media.capabilities` 可查当前可用模态，接入前建议先调该接口做能力协商。

**源码位置**：`jiuwenswarm/server/runtime/attachments/{document_attachments.py:9,15,32,34,52,70,137-141,161,189-207,273,321, media_attachments.py, steer_images.py:17, upload_storage.py:22}`；`agent_gateway/common/attachment_safety.py:14,19,23,25,33,34,37,52-68,71,88,91,126,131,136`；`jiuwenswarm/server/runtime/gateway_adapter/workspace_file_adapter.py:11,38,118,183-184,187-188,191-192,200-231,312-331`；`jiuwenswarm/server/agent_ws_server.py:1369`；`agent-protocol/.../message.py:240,256-262,271,280-281`；`agent_gateway/channel_manager/web/container_file_http.py:95`。

---

## 16. Langfuse 观测：trace / token / 耗时；OTel 或 Langfuse exporter、标准 trace API

### 结论
**支持（需配置）。** 内核自带 OpenTelemetry 与 Langfuse exporter，并实现了 Langfuse 的观测语义（observation type / session id / output）；token 与耗时同时以事件形式实时推给前端。需要显式打开开关并填 Langfuse 凭据；产品不自带 collector 与存储。

### 16.1 开关与传输选择

```yaml
# config.yaml（节选）—— 打开即导出，支持热生效
agent_observability:
  enabled: true
  exporter: langfuse          # 或 otlp / file
  langfuse_public_key: ${LANGFUSE_PUBLIC_KEY}
  langfuse_secret_key: ${LANGFUSE_SECRET_KEY}
```

```python
# ── 装配与热更新（jiuwenswarm/agents/harness/agent_observability.py）
def sync_agent_observability(*, force: bool = False) -> None    # :124 配置变更后重新装配
# :128-149 热重载说明：agent_observability.enabled 改动即时生效（开 → 拉起，关 → shutdown）
# :17 / :65 约束说明：OTel 每进程只允许一个全局 TracerProvider
# ── 传输选择说明（jiuwenswarm/agents/harness/observability_runtime.py）
# :39 注释原文：select the transport with 'exporter' (e.g. exporter: langfuse)
# :52-53 配置项：langfuse_public_key / langfuse_secret_key
```

### 16.2 exporter 与语义实现（内核侧）

```python
# ── 内核观测扩展（agent-core/openjiuwen/extensions/observability/）
exporters/langfuse.py            # Langfuse 导出
exporters/transforming.py        # span → Langfuse observation 转换
otlp_codec.py                    # OTLP 编码
file_exporter.py                 # 本地文件导出（离线排障用）
gen_ai_semconv.py / semconv.py   # GenAI 语义约定 + Langfuse 属性常量
span_record_processor.py / span_context.py / instrumentation.py / callback_handler.py
redaction.py                     # 导出前脱敏
trajectory_events.py / content_addressing.py / tool_outcome.py
error_reporting.py / demand.py / setup.py / config.py / runtime.py
```

### 16.3 trace 结构与属性（可直接在 Langfuse 里按会话过滤）

```python
# ── span 命名（分层，便于按模式与会话聚合）
# agent_observability.py:927  agent.<mode>.<session_id>
# ── Langfuse 属性（常量来自 openjiuwen.extensions.observability.semconv）
span.set_attribute(LANGFUSE_OBSERVATION_TYPE, "agent")   # :363 顶层 observation 类型
span.set_attribute(LANGFUSE_SESSION_ID, session_id)      # :365 / :976 会话维度聚合键
span.set_attribute(LANGFUSE_OBSERVATION_OUTPUT, ...)     # :1014 / :1021
# :1001 注释说明：不填该属性会导致 Langfuse 顶层 output 为空
```

### 16.4 token 与耗时：实时事件 + 事后 trace 双通道

```python
# ── 实时（推给前端，无需外部系统即可看用量）
EventType.CHAT_USAGE_METADATA   # message.py:639 每次模型调用的 token 明细
EventType.CHAT_USAGE_SUMMARY    # message.py:640 一轮汇总
EventType.CONTEXT_USAGE         # message.py:651 上下文占用
# ── 事后（Langfuse / OTLP）
# 按 LANGFUSE_SESSION_ID 过滤会话；耗时来自 span 起止；工具结果与错误由 tool_outcome.py / error_reporting.py 标注
# ── 前端侧：Web 前端内置 OpenTelemetry projector 与轨迹浏览器
#   （channels/web/frontend 的 NOTICE.md 自述）
```

### 16.5 覆盖范围

| 覆盖对象 | 说明 |
|---|---|
| Agent 主链路（对话 / 团队 / 工作流） | `agent.<mode>.<session_id>` 分层 span |
| Team（多 Agent 协同） | `agents/swarm/team_manager.py:170` 注释明确「mirroring sync_agent_observability」 |
| 技能演化 / Symphony | `agent_observability.py:136` 说明：即使 `enabled=false`，这些内部路径仍可能拉起 OTel（用于内部指标） |
| 上下文压缩 | `context_compression_handler.py` |

### 16.6 注意事项
- **默认关闭**：`agent_observability.enabled` 默认不为 true，需要贵方显式打开；打开后热生效，无需重启。
- **不自带 collector 与存储**：需要外部 Langfuse 实例（自部署或 SaaS）或 OTLP endpoint。若贵方要求「开箱自带观测后端」，属部署侧选型，我们可提供 Langfuse 自部署方案。
- **单进程单 TracerProvider**：若贵方在同一进程内嵌入自己的 OTel 初始化，需要与我们共用 provider（`agent_observability.py:17,65` 已说明约束），否则会出现导出互相覆盖。
- **脱敏策略可配**：`redaction.py` 提供导出前脱敏；若贵方要求「模型输入输出不得出内网」，建议 `exporter: file` 落地后由贵方侧导入，或只导出到内网 Langfuse。

**源码位置**：`agent-core/openjiuwen/extensions/observability/`（文件清单见 16.2）；`jiuwenswarm/agents/harness/{agent_observability.py:17,65,124,128-149,316-317,363,365,927,976,1001,1014,1021, observability_runtime.py:39,52-53}`；`jiuwenswarm/agents/swarm/team_manager.py:170`；`agent-protocol/.../message.py:639,640,651`。

---

## 17. 项目管理与目录树：分支创建、切换、合并与删除接口

### 结论
**支持**（项目管理、目录树、分支创建与切换、提交与推送）。**分支合并与分支删除需扩展实现**：产品侧的 git 能力覆盖「建分支 / 切分支 / 提交 / 推送 / 看差异 / 撤销重做」，**没有** merge、delete_branch、分支列表与提交历史（log）四个接口。

### 17.1 项目管理 RPC（完整清单）

```python
# agent-protocol/.../message.py:196-215
ReqMethod.PROJECT_INFO / PROJECT_PINNED_SESSIONS / PROJECT_GET_SESSIONS / PROJECT_GET_CRON_SESSIONS
ReqMethod.PROJECT_CRON_RESOLVE_BINDING
ReqMethod.PROJECT_LIST / PROJECT_CREATE / PROJECT_RENAME / PROJECT_PIN
ReqMethod.PROJECT_REMOVE          # 软删（进回收态）
ReqMethod.PROJECT_RESTORE         # 从回收态恢复
ReqMethod.PROJECT_LIFECYCLE       # 生命周期事件（配合 project.lifecycle.updated）
ReqMethod.PROJECT_SESSIONS_ARCHIVE / PROJECT_SESSIONS_DELETE_ARCHIVED
```

### 17.2 git 能力 RPC（12 个，实测清单）

```python
# agent-protocol/.../message.py:216-227
ReqMethod.PROJECT_GIT_STATUS            # 当前分支 / 脏状态
ReqMethod.PROJECT_GIT_PROBE             # 是否 git 仓库、仓库形态探测
ReqMethod.PROJECT_GIT_INIT              # 初始化仓库，可指定 initial_branch
ReqMethod.PROJECT_GIT_SWITCH_BRANCH     # ✅ 切换分支
ReqMethod.PROJECT_GIT_CREATE_BRANCH     # ✅ 创建分支
ReqMethod.PROJECT_GIT_COMMIT / PROJECT_GIT_PUSH
ReqMethod.PROJECT_GIT_DIFF_STATUS / PROJECT_GIT_TURN_DIFF_LIST / PROJECT_GIT_TURN_DIFF
ReqMethod.PROJECT_GIT_DISCARD_TURN_CHANGES / PROJECT_GIT_REDO_TURN_CHANGES
# ❌ 无 PROJECT_GIT_MERGE / PROJECT_GIT_DELETE_BRANCH / PROJECT_GIT_LIST_BRANCHES / PROJECT_GIT_LOG
```

```python
# ── 实现落点（jiuwenswarm/server/runtime/gateway_adapter/project_adapter.py）
# 方法声明：:1279-1282（ReqMethod.PROJECT_GIT_STATUS / PROBE / INIT / SWITCH_BRANCH …）
get_project_git_service().probe(project)                        # :498
get_project_git_service().status(project)                       # :500
get_project_git_service().init(project, initial_branch=…)       # :525
get_project_git_service().create_branch(…)                      # :556
get_project_git_service().switch_branch(…)                      # :573
get_project_git_service().commit(…)                             # :661
get_project_git_service().push(…)                               # :718
def _git_status_payload(project, repo_status) -> dict           # :436 统一响应结构
def _git_error_payload(error) -> dict                           # :468 统一错误结构
# 底层服务：jiuwenswarm/server/runtime/session/project_git.py（含项目创建时自动建仓 :139 ensure_on_project_create）
```

### 17.3 关于「合并」：产品能检测，不能执行

```python
# jiuwenswarm/server/runtime/session/project_git.py:264
# 返回 (is_transient, kind)：kind 为 "merge" / "rebase" / "cherry-pick" 等
# :284 判定表片段：("merge", "MERGE_HEAD")
# 语义：当仓库处于 merge / rebase / cherry-pick 进行中状态时，产品能识别并告知前端「有未完成的合并」，
#       但产品不提供发起 merge / 解决冲突 / 完成合并的接口。
```

### 17.4 目录树与文件枚举

```python
# ── 工作空间目录树（协议成员由 §9.2 的兼容扩展补入）
ReqMethod.WORKSPACE_TREE      = "workspace.tree"       # REST GET /workspace/tree
ReqMethod.WORKSPACE_PREVIEW   = "workspace.preview"    # REST GET /workspace/preview
ReqMethod.WORKSPACE_USAGE     = "workspace.usage"      # REST GET /workspace/usage
# 另有 REST：GET /workspace/download（pod_rest_map.py:285）
# 实现：jiuwenswarm/server/runtime/gateway_adapter/workspace_file_adapter.py:200-231
#   :209 WorkspaceService(get_multi_tenant_user_workspace_dir())
#   :212 tree / :214 preview / :217 delete
#   :221-226 extract_triple + resolve_effective_quota（按用户配额）
#   :229 asyncio.to_thread（不阻塞事件循环）/ :231 WorkspaceError → build_error_response

# ── 其他枚举入口
ReqMethod.FILES_LIST / FILES_GET                       # message.py:252-253
ReqMethod.PATH_SELECT_DIRECTORY / PATH_SELECT_FILES    # 目录内枚举（注入式选择）
ReqMethod.SKILLS_FILES_LIST / SKILLS_FILES_GET
```

### 17.5 注意事项
- **分支合并与删除需扩展实现**：建议路径两条——① 由集成方在项目目录内用外部 git 工具链完成合并/删除，产品侧只负责状态与差异展示（`project.git.status` 会如实反映结果）；② 若贵方要求产品提供，我们可按 `_ROUTE_ROWS` + `project_git.py` 的现有形态补 `merge` / `delete_branch` 两个方法（改动集中在 `project_git.py` 与 `project_adapter.py`）。
- **没有分支列表与提交历史**：前端展示分支列表目前靠 `project.git.status` 返回的当前分支 + 前端本地记录；若贵方需要完整分支列表与 log，属需扩展实现。
- **git 操作限定在项目目录内**：不做跨节点 git 协调；多节点共享同一项目时请按八项答复 §5.1 的 NFS 方案挂载。
- **目录树越界由服务端拦**：`WorkspaceService.resolve()` 会把路径限制在该租户工作空间内，越界返回结构化错误码（见 §15.4）。

**源码位置**：`agent-protocol/.../message.py:196-227`；`jiuwenswarm/server/runtime/gateway_adapter/{project_adapter.py:136,139,436,468,486-500,515-539,556,573,609,661,687,718,1279-1282, workspace_file_adapter.py:200-231}`；`jiuwenswarm/server/runtime/session/project_git.py:264,284`；`jiuwenswarm/extensions/runtime_management_extension/pod_rest_map.py:281-285`；`agent_gateway/channel_manager/web/http_routes/mapped.py:819,824,833`。

---

## 18. 会话分支：分支数据模型；分支创建、切换、合并与删除接口

### 结论
**支持。** 有明确的分支数据模型：`session.fork` 从既有会话的**指定截断点**分出一个新会话，血缘记在返回的 `source_session_id`；切换与回退接口完整。**分支合并需扩展实现**（无会话合并接口）。

### 18.1 分支 RPC 清单

```python
# agent-protocol/.../message.py
ReqMethod.SESSION_SWITCH            # :152  切换当前会话
ReqMethod.SESSION_FORK              # :156  分叉（创建分支）
ReqMethod.SESSION_REWIND            # :158  回退到指定点
ReqMethod.SESSION_REWIND_AND_RESTORE# :159  回退并恢复文件
ReqMethod.SESSION_REWIND_COMPACT    # :160  回退并压缩上下文
ReqMethod.SESSION_REWIND_CONTEXT    # :161  只回退上下文
ReqMethod.SESSION_RESTORE_FILES     # :162  恢复文件
# 相关：session.rebind_project（换绑项目）、session.pin、session.preview、session.archived.list
```

### 18.2 分支数据模型（fork 的入参与返回）

```python
# ── 分叉（jiuwenswarm/server/handlers/bootstrap.py:280 handle_session_fork）
# 注册：server/dispatch.py:212  ReqMethod.SESSION_FORK: handle_session_fork
# REST：POST /sessions/{session_id}/actions/fork（pod_rest_map.py）
class SessionForkInput:                    # :322 组装
    channel_id: str
    source_session_id: str                 # 必填（:313 为空 → BAD_REQUEST）
    target_session_id: str | None          # 可指定新会话编号，不指定则服务端生成
    title: str
    cutoff_message_id: str                 # ← 截断点：fork_point.message_id
    cutoff_role: str                       #           fork_point.role
    cutoff_content: str                    #           fork_point.content
    cutoff_timestamp: Any                  #           fork_point.timestamp
    session_equipment_override: dict | None  # 新分支可换装备（模型 / 工具集）
# 旧参数已废弃：传 side_conversation → BAD_REQUEST "side_conversation is no longer supported"（:293-296）

# ── 两段式提交（保证失败不留半个分支）
prepared = await runtime.prepare_session_fork(SessionForkInput(...))     # :318
result   = await runtime.commit_session_provision(
    prepared, timing=SessionProvisionCommitTiming.BEFORE_RESULT_DELIVERY # :332
)

# ── 返回（分支血缘在这里）
payload = {
    "session_id": result.session_id,           # 新分支会话
    "source_session_id": result.source_session_id,   # ← 血缘：从哪个会话分出
    "title": result.title,
    "moa_enabled": result.moa_enabled,
}                                                # :343-348
```

### 18.3 切换与回退

```python
# ── 切换
# server/dispatch.py:224  ReqMethod.SESSION_SWITCH: handle_session_switch → server/handlers/session.py:732
# REST：POST /sessions/{session_id}/actions/switch

# ── 回退家族（同一个 handler 按 req_method 分流）
# server/dispatch.py:228-230  SESSION_REWIND / REWIND_AND_RESTORE / REWIND_COMPACT → handle_session_rewind
#                             :231  SESSION_REWIND_CONTEXT → handle_session_rewind_context
# 实现：server/handlers/session.py:892（rewind）/ :1091（rewind_context）
# REST：POST /sessions/{id}/actions/{rewind, rewind-restore, rewind-compact, rewind-context, restore-files}
```

### 18.4 注意事项
- **没有「合并两个会话分支」的接口**：fork 是单向分叉，两侧各自演进后不汇合。若贵方需要「把分支 B 的对话成果并入 A」，属**需扩展实现**；可行的过渡做法是在 A 里用 `history.get` 取 B 的消息再作为上下文注入（我们可协助设计）。
- **没有「分支列表」接口**：分支关系靠 `source_session_id` 字段自行还原（`session.list` 返回的会话元数据里带该字段）。若贵方需要「按源会话列出所有分支」，属需扩展实现（改动在会话元数据查询侧）。
- **删除分支 = 删除会话**：没有独立的「删分支」语义，走 `session.delete` / `session.archive`。
- **`fork_point` 的四个字段是组合定位**：按 `message_id` 优先，其余字段用于同一消息多条候选时的辅助匹配；建议贵方只传 `message_id`。

**源码位置**：`agent-protocol/.../message.py:152,156,158-162`；`jiuwenswarm/server/dispatch.py:28,107-109,212,224,228-231`；`jiuwenswarm/server/handlers/bootstrap.py:280-350`（含 `:293-296,313,318,322,332,343-348`）；`jiuwenswarm/server/handlers/session.py:732,892,1091`；`jiuwenswarm/extensions/runtime_management_extension/pod_rest_map.py:56-62`。

---

## 19. 文件变更通知：主动通知能力；文件变更订阅、事件序号与补偿机制

### 结论
**支持（需配置，限 git 项目）。** 产品提供的是**面向项目 diff 的三档订阅 + 四个变更事件**，由 Gateway 侧轮询指纹后主动推。**通用文件变更订阅（任意目录 / 任意文件）、事件序号与断线补偿需扩展实现。**

### 19.1 订阅接口（Gateway 独立 WebSocket 路由 `/ws/git`）

```python
# ── agent_gateway/channel_manager/web/git_ws_handler.py
# 订阅（首次响应是快照，走 channel.send_response）
{ "action": "diff_watch",        "project_id": ..., "source": "current" | "last_turn" }
{ "action": "diff_files_watch",  ... }     # 文件列表层
{ "action": "diff_detail_watch", ... }     # 已订阅文件的 hunk 内容层
# 退订
{ "action": "diff_unwatch", "scope": "all" | "files" | "detail" }
_VALID_SOURCES      = {"current", "last_turn"}          # :31-36
_VALID_UNWATCH_SCOPES = {"all", "files", "detail"}
# 断连清理：unregister_ws + cleanup_ws（:53）
```

### 19.2 四个变更事件

```python
# 事件名（EventType 之外的通道事件，经 WebChannel.send_event 推）
"project.git.diff_changed"          # summary 指纹变化（设计文档 §3.6）
"project.git.diff_files_changed"    # 文件列表指纹变化
"project.git.diff_detail_changed"   # 已订阅文件的 hunk 变化
"project.git.error"                 # git 命令失败（含结构性错误码）
```

### 19.3 变化检测与推送策略（可调参数）

```python
# ── agent_gateway/channel_manager/web/git_diff_watcher.py  class GitDiffWatcherRegistry
POLL_INTERVAL_SEC   = 2.0     # :24 轮询间隔（秒）
DEBOUNCE_SEC        = 0.3     # :26 合并连续 mark_dirty，只重算一次
ERROR_BACKOFF_SEC   = 5.0     # :28 出错后退避
MAX_PUSH_FAILURES   = 3       # :38 推送连续失败 3 次 → 自动回收孤儿 watcher
_STRUCTURAL_ERROR_CODES = {"NOT_GIT_REPOSITORY", "PROJECT_DIR_MISSING"}   # :30 命中则暂停该 watcher
# 聚合与去重：按 project_id 聚合共享结果；summary / files / detail 三层 fingerprint（hashlib）
# :633 首轮先 seed 指纹，避免刚订阅就推一次冗余 diff_changed
# :589 去重说明：summary 未变时不重复推 diff_changed
# :1088-1130 _push_diff_changed(ws, "project.git.diff_changed", payload)
def mark_dirty(project_id) -> None      # 业务改完文件立刻唤醒，不必等 2 秒
def cleanup_ws(ws) / cleanup_project(project_id) -> None
```

### 19.4 关于「事件序号与补偿」

```python
# 这些事件调用 send_event 时 seq / stream_id 明确传 None
#   git_ws_handler.py:13 注释；git_diff_watcher.py:194
# 全仓检索无 last_seq / from_seq / resume_from / catch_up 之类入口 → 无断线补偿
# 订阅语义是「快照 + 增量」：重连后重新 diff_watch 会先拿一次快照，因此客户端不需要补历史事件，
# 但也无法判断「断线期间漏了几次变更」——只能以最新快照为准。
```

### 19.5 产品内部另有文件监听，但不面向客户端

```python
# jiuwenswarm/agents/harness/common/memory/manager.py
from watchdog.observers import Observer          # :351-352
self._file_observer = Observer()                 # :395  监听记忆目录，刷新本地索引
# :416 未安装 watchdog 时降级：日志 "watchdog not installed, file watching disabled"
# 用途：服务端自身记忆索引一致性，不向客户端推事件
```

> 易混淆点：`issue.watch_once`（`message.py:619-622`）是**定时任务**的一次触发动作（`server/handlers/schedule.py:32,135`；`server/agent_ws_server.py:3791 _handle_schedule_request(..., "issue_watch_once")`；`agents/harness/common/auto_harness/service.py:679`），与文件变更监听无关。

### 19.6 注意事项
- **通用文件变更订阅需扩展实现**：目前只有 git 项目的 diff 变更会推；非 git 项目目录、工作空间任意文件、租户目录都没有变更事件（`workspace.*` 的写操作也不发事件）。若贵方需要「订阅某目录/文件变更」，建议路径：在 Gateway 侧按 §19.3 的 Registry 形态加一层 watchdog 订阅（产品内部已有 watchdog 依赖，见 19.5），我们可评估补齐。
- **事件序号与补偿需扩展实现**：现有订阅是「快照 + 增量」语义，重连后重新订阅即得最新状态，功能上可用，但**无法检测漏事件次数**。若贵方验收要求「事件可编号、可补发」，需要为 `project.git.*` 事件填 `seq` 并加缓冲（改动集中在 `git_diff_watcher.py` 与 `WebChannel.send_event` 调用点）。
- **检测靠轮询不是内核通知**：默认 2 秒轮询 + 0.3 秒 debounce，因此「改完立刻收到」在亚秒级到 2 秒之间；对时延敏感的场景可让业务侧调 `mark_dirty` 立即唤醒。
- **推送失败会自动回收订阅**：连续 3 次推送失败（客户端僵死）后 watcher 被回收，客户端需重新订阅——这是防止服务端资源泄漏的有意设计。

**源码位置**：`agent_gateway/channel_manager/web/{git_ws_handler.py:5-16,31-36,53, git_diff_watcher.py:24,26,28,30,38,109,194,589,633,803,1051,1088-1130}`；`agent_gateway/channel_manager/web/{web_connect.py:580-592, outbound.py:176-181}`；`jiuwenswarm/agents/harness/common/memory/manager.py:351-352,395,416`；`jiuwenswarm/server/handlers/schedule.py:32,62,135`；`jiuwenswarm/server/agent_ws_server.py:3791,15311`；`jiuwenswarm/agents/harness/common/auto_harness/service.py:679`；`agent-protocol/.../message.py:619-622`。

---

## 附录 E：公共主干（十一项共用的一条调用链）

贵方若要把本文的接口串成端到端用例，下面这条主干是十一项全部经过的路径：

```
浏览器 / IM / TUI / E2A 客户端
  │  私有 WS 帧 {type:"req", method, params, request_id}          ← §9
  ├─ Gateway：web_connect.py:1742 dispatch_client_request()
  │            lifecycle_handlers.py:23  req_method = ReqMethod(method)
  │            invoke.py:114 dispatch_web_request()
  │            （REST 等价入口：http_routes/mapped.py）
  │  E2A 信封（common/e2a/models.py）
  └─ AgentServer：agent_ws_server.py:3065 → :3287 dispatch_parsed_request()
                  front/router.py:123 MethodRouter.dispatch()
                    ├─ :106 is_control_method → :173 _dispatch_control → :178 bind_control_tenant
                    └─ 执行类 → front/admission.py:74 ExecutionAdmission.dispatch()
                                 :75 CHAT_CANCEL 短路；:93 RequestRegistry.register → :96 DUPLICATE_REQUEST   ← §10
                                 队列 32 / 120 s；DRAINING/FAILED 拒绝                                        ← §12
                              → worker/dispatcher.py:450 → server/dispatch.py:126,128 HANDLERS 总表
                              → runtime/agent_adapter/interface.py（会话与轮次语义）                            ← §10,§18
                              → 事件回推：web_connect.py:580 send_event / :1093 send_session_event             ← §13
      旁路：/file-api/*（container_file_http.py:620）                                                            ← §11,§14,§15
            /ws/git（git_ws_handler.py）                                                                        ← §17,§19
```

## 附录 F：取证索引（按组件汇总，便于逐条复核）

| 组件 | 本文涉及的关键文件 |
|---|---|
| 协议真源 | `agent-protocol/openjiuwen-protocol/openjiuwen_protocol/message.py`（`ReqMethod:83-624`、`EventType:625-688`、`Mode:689`、`Message:759`） |
| 协议兼容扩展 | `jiuwenswarm/common/schema/message.py:151-153`；`agent_gateway/common/protocol_compat.py:82` |
| Gateway 接入 | `channel_manager/web/{web_connect.py, invoke.py, lifecycle_handlers.py, outbound.py, http_routes/mapped.py, container_file_http.py, git_ws_handler.py, git_diff_watcher.py}`；`routing/session_sharing.py`；`common/security/{ws_origin.py, link_mtls.py}`；`common/attachment_safety.py` |
| AgentServer 接入与准入 | `server/agent_ws_server.py`；`server/front/{connection.py, admission.py, router.py, protocol.py, event_forwarder.py}`；`server/supervisor/runtime_supervisor.py`；`server/worker/dispatcher.py`；`server/dispatch.py` |
| 运行时与会话 | `server/runtime/agent_adapter/{interface.py, interface_deep.py}`；`server/runtime/session/{session_metadata.py, session_message_store.py, session_message_service.py, model.py, project_git.py}`；`server/handlers/{chat.py, bootstrap.py, session.py, schedule.py, team.py, commands.py}` |
| 工具与观测 | `agents/harness/common/rails/{stream_event_rail.py, permissions/_auto_permission/artifact_authorization.py}`；`agents/harness/common/tools/cron/cron_runtime.py`；`agents/harness/{agent_observability.py, observability_runtime.py}`；`agent-core/openjiuwen/extensions/observability/` |
| 文件与附件 | `common/{file_download_token.py, verified_download_assets.py, e2a/*}`；`server/runtime/attachments/*`；`server/runtime/gateway_adapter/{workspace_file_adapter.py, project_adapter.py, memory_adapter.py, config_adapter.py}` |
| 跨节点与 REST | `extensions/runtime_management_extension/{pod_rest_map.py, runtime_routed_client.py}` |

## 附录 G：核对方式与验收建议

```powershell
# 方法名与事件名（协议真源，一处可查全部）
# 打开 agent-protocol/openjiuwen-protocol/openjiuwen_protocol/message.py，按 §9.1 的行号区间检索

# 某方法从入口到 handler 的完整链路（以 session.fork 为例）
#   message.py:156 → dispatch.py:212 → handlers/bootstrap.py:280 → prepare_session_fork / commit_session_provision

# 限制值（§14.4 / §15.3 表格里的常量）
#   在对应文件按常量名检索即可，例如 _TOOL_RESULT_TEXT_LIMIT、_MAX_DOCUMENT_BYTES、POLL_INTERVAL_SEC
```

**建议的验收断言（与对照表每行末尾的【断言点】一致）**
1. 取消用 `chat.interrupt`、回答用 `chat.user_answer`，用旧名应被拒或走兼容路径。
2. 同一 `request_id` 重发只执行一次（第二次返回 `DUPLICATE_REQUEST`）。
3. 断连后 `session.get_metadata` 可见 `client_disconnected`，`chat.resume` 能续跑；主动终止后不再续跑。
4. 工具结果超过 60000 字符时 `raw_output_truncated=true`，且 `session.assets.list` 能列到 `send_file_to_user` 的产物。
5. 文档上传超 100 MiB 或命中黑名单扩展名被拒，响应含 `forbidden_formats()` 清单。
6. 开 `agent_observability.enabled` 后 Langfuse 内可按 `LANGFUSE_SESSION_ID` 找到会话 trace，顶层 output 非空。
7. `project.git.merge` / `delete_branch` 不存在（应返回方法不支持）；`session.fork` 返回的 `source_session_id` 指回源会话。
8. 改一个 git 跟踪的文件，2 秒内（或 `mark_dirty` 后立刻）收到 `diff_changed` + `diff_files_changed`；断线重连后必须重新订阅；非 git 目录不产生任何事件。
