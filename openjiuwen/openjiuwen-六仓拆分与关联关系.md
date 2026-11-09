# openJiuwen 六仓拆分与关联关系

> 取证时间：2026-11-09（Asia/Shanghai）
> 取证方式：按 `jiuwenswarm/pyproject.toml` 钉的 rev 逐仓克隆，再用 `Select-String` 全仓扫 import / 依赖声明 / 配置引用；所有结论都带**文件:行号**，可逐条复核。
> 结论适用范围：下表六个 HEAD 快照。升级版本后需重新核对（拆分状态在变）。

## 0. 一句话结论

**jiuwenswarm 是母仓，能力确实在往外拆，但拆到一半。** 拆分状态分四档：

| 档 | 状态 | 具体 |
|---|---|---|
| **A 已拆出、被正式依赖** | 独立仓 + 母仓按 rev 钉 + 有 import | `agent-core`、`agent-gateway`、`agent-protocol`、`agent-runtime/foundation` |
| **B 已拆出、母仓仍留副本或文档** | 真身在别的仓，母仓目录里还有旧内容且**被 git 跟踪** | `jiuwenbox`（母仓留 108 文件源码副本）、`AgentSSAS`（母仓只剩设计文档） |
| **C 独立仓、母仓目录里躺着但不跟踪** | 母仓靠 **PyPI 包**依赖它，目录只是本地镜像 | `agent-memory`（`JiuwenMemory>=0.2.0`） |
| **D 完全没拆** | 仍在母仓内 | Web 前端与全部通道、RPC handler、产品级 harness、企业多租户、runtime 编排 |

---

## 1. 六仓清单

| 目录 | remote | 分支 / HEAD | 日期 | 磁盘文件 | 定位（README 自述） |
|---|---|---|---|---|---|
| `jiuwenswarm` | `github.com/openJiuwen-ai/jiuwenswarm.git` | `develop` @ `c1b9700` | 2026-11-06 | 7944（git 跟踪 6910） | AgentServer 产品层（母仓） |
| `agent-core` | `gitcode.com/openJiuwen/agent-core.git` | detached @ `f3f8f57` | 2026-11-05 | 7041 | `openjiuwen` 内核框架（agent 编排 / 工具 / 记忆 / 沙箱 provider） |
| `agent-gateway` | `gitcode.com/openJiuwen/agent-gateway.git` | detached @ `cff3b1b` | 2026-11-05 | 794 | Gateway 传输层（WS / HTTP / 通道管理 / 租户 key） |
| `agent-protocol` | `gitcode.com/openJiuwen/agent-protocol.git` | detached @ `cbbadde` | 2026-11-03 | 708 | 协议与名字：`A4P`、`openjiuwen-protocol`、`AgentRegistry` |
| `agent-runtime` | `github.com/openJiuwen-ai/agent-runtime.git` | `main` @ `34ed6e8` | — | 335 | 「runtime and deployment system for AI agents」开发→生产 |
| `agent-security` | `github.com/openJiuwen-ai/agent-security.git` | `develop` @ `1e0b3b9` | — | 620 | 「openJiuwen's collection of security capabilities」行为检测 / 隔离执行 / 审计 |
| `jiuwenswarm/agent-memory` | `github.com/openJiuwen-ai/agent-memory.git` | `d106b0d`（`[mirror]` 前缀） | 2026-11-09 | 1061 | 「Framework-Agnostic · Layered Memory」长期记忆基础设施 |

⚠️ **两个必须知道的坑**

1. **同一项目在 gitcode 与 github 两处镜像**：`pyproject.toml` 钉的是 `gitcode.com/openJiuwen/*`，而 `agent-runtime` / `agent-security` / `agent-memory` 的本地克隆 remote 是 `github.com/openJiuwen-ai/*`。核对版本时**以 pyproject 钉的 gitcode rev 为准**。
2. **本地 `agent-runtime` 在 `main`，但母仓钉的是 `develop`**（`pyproject.toml:89`）→ 用本地这份查细节会偏差，需要 `git fetch origin develop` 再看。

---

## 2. 依赖方向

```
agent-core        ← 底座：pyproject 里【零】git+ 钉法，谁都能用它
    ↑
agent-protocol    ← 名字层（ReqMethod / A4P / E2A 信封 / AgentRegistry）
    ↑
agent-gateway ────→ 钉 core + protocol + runtime-foundation
    ↑
jiuwenswarm ──────→ 钉上面全部 + agent-runtime/foundation + JiuwenMemory(PyPI)
    ‖
agent-security    ← 不 import，HTTP 服务（jiuwenbox）；客户端在 agent-core
agent-memory      ← 独立仓，母仓按 PyPI 包依赖（目录只是本地镜像）
```

### 2.1 `jiuwenswarm/pyproject.toml` 钉法逐条

| 行 | 内容 |
|---|---|
| `:17` | `a4p @ git+…/agent-protocol.git@agent-protocol-V0.2.0#subdirectory=A4P` |
| `:22` | `openjiuwen @ git+…/agent-core.git@f3f8f576…` |
| `:27` | `openjiuwen-protocol @ git+…/agent-protocol.git@cbbadde…#subdirectory=openjiuwen-protocol` |
| `:28-29` | 注释「Gateway 传输层已拆分为独立仓 agent-gateway」+ `agent-gateway @ …@cff3b1b…` |
| **`:23`** | **`JiuwenMemory>=0.2.0`**（PyPI，不是 git 钉） |
| `:88-89` | 注释「Enterprise / link-mtls / manager-config… Dockerfile.claw strips this line」+ `openjiuwen-runtime-foundation @ …/agent-runtime.git@develop#subdirectory=foundation` |
| `:140-143` | extras：`distribute → openjiuwen[postgres,zmq]`、`claude`、`codex`、`cua` |
| `:229-242` | 注释说明「agent-gateway（独立仓）与母仓各自钉 agent-core / 协议的 git rev，rev 由母仓升级节奏主导，agent-gateway 跟随同步」 |
| `:247-250` | `[tool.uv.sources]` 同一套映射 |

### 2.2 其他仓的钉法

- `agent-gateway/pyproject.toml:16,17` 钉 `openjiuwen` + `openjiuwen-protocol`（与母仓同 rev）；`:89` 在 `[tool.uv.sources]` 里钉 `openjiuwen-runtime-foundation`；`:85-86` 两个入口 `agent-gateway`、`agent-gateway-acp`。
- `agent-core/pyproject.toml`：**没有任何 git+ 钉法** → 它是依赖链最底层。
- `agent-runtime/foundation/pyproject.toml`：包名 `openjiuwen-runtime-foundation` 0.1.0，依赖 sqlalchemy / pymysql / cryptography / pydantic（**不依赖 agent-core**）。
- `agent-memory/pyproject.toml:75-80`：依赖 `openjiuwen`，注释写明「openjiuwen 的 MemoryProvider ABC，供 ExternalMemoryRail / JiuwenMemoryRail 驱动」。

---

## 3. import 计数（jiuwenswarm 生产代码，排除 tests）

| 外部包 | import 行数 |
|---|---|
| `openjiuwen`（agent-core） | **1398** |
| `agent_gateway` | **239** |
| `openjiuwen_protocol` | 44 |
| `a4p` | 7 |
| `openjiuwen_runtime`（agent-runtime/foundation） | 4 |
| `jiuwenbox` | 1（可选 import，失败即回落） |
| 合计 | 1689 |

复现命令（PowerShell，中文路径必须 `-LiteralPath`）：

```powershell
Set-Location -LiteralPath 'D:\开源项目源码\jiuwenswarm'
$files = Get-ChildItem -Recurse -Include *.py -Path 'jiuwenswarm' |
  Where-Object { $_.FullName -notmatch '\\tests\\' }
$files | Select-String -Pattern '^\s*(?:from|import)\s+(openjiuwen_runtime_foundation|openjiuwen_protocol|agent_gateway|openjiuwen_runtime|jiuwenbox|a4p|openjiuwen)(?:[.\s]|$)' |
  ForEach-Object { $_.Matches[0].Groups[1].Value } | Group-Object | Sort-Object Count -Descending
```

---

## 4. 模块级关联表（拆出去的仓 ↔ 母仓哪些模块在用）

| 外部仓 | 母仓使用方（import 计数） | 具体关联点（文件:行号） |
|---|---|---|
| **agent-core**（`openjiuwen`） | `agents/harness` 649、`server/runtime` 327、`agents/swarm` 121、`server/jiuwenmate` 79、`resources/agent` 49、`common` 38、`runtime` 18、`server/handlers` 15、`runtime/symphony` 15 | rails / 工具 / MCP 客户端 / 上下文压缩 / 沙箱 provider / 外接记忆 provider：`openjiuwen/core/memory/external/{provider.py(MemoryProvider ABC),mem0_provider,openviking_memory_provider,lakebase_memory_provider,openjiuwen_memory_provider,agentarts_memory_provider}.py` |
| **agent-gateway** | `server/runtime` 61、`runtime` 23、`common/auth` 16、`extensions/runtime_management_extension` 14、`agents/harness` 13、`extensions/video_duplex` 9、`extensions/agentos` 9、`server/jiuwenmate` 7 | WS 传输与帧上限（`agent_gateway/common/ws_limits.py`）、租户 key（`agent_gateway/common/tenant_key.py`）、断点恢复（`common/interrupt_resume.py`）、文件 HTTP（`channel_manager/web/container_file_http.py`）、下载 token（`common/file_download_token.py`） |
| **agent-protocol** | `common/e2a` 21、`server/runtime` 10、`common/schema` 4、`enterprise/capabilities` 3 | 方法名定义 `openjiuwen-protocol/openjiuwen_protocol/message.py:83`（`class ReqMethod`）、`:188-192`（`memory.*` 五个）；A4P 在 `agents/harness` 7 处 |
| **agent-runtime**（只用 `foundation/`） | `common/security` 2、`runtime` 2 | `jiuwenswarm/common/security/link_mtls.py:20,25` ← `openjiuwen_runtime.foundation.security.{link_mtls_config,link_profile}`；`jiuwenswarm/runtime/checkpointer.py:130,188` ← `foundation.db.{postgresql_handler,utils}`。**agent-gateway 侧用得更多**：`agent_gateway/config/audit/access.py:16` ← `foundation.audit`；`agent_gateway/config/enterprise/tables/{a2a_models,application_config_models,cron_job_models}.py` ← `foundation.db.table_def` |
| **agent-security** | **零 import**（只有字符串与配置注释） | 见 §6 |
| **agent-memory**（`JiuwenMemory`） | `agents/harness/common/memory/jiuwen_rail_builder.py:181-183,218` | 见 §5 |

---

## 5. `agent-memory`：独立仓，但母仓靠 PyPI 包依赖（档 C）

**目录形态**：`jiuwenswarm/agent-memory/` 里有自己的 `.git`（remote `github.com/openJiuwen-ai/agent-memory.git`，HEAD `d106b0d`，提交信息前缀 `[mirror]`），但母仓 git **完全不跟踪它**：

```
git ls-files agent-memory      → 0 个文件
git status --porcelain         → ?? agent-memory/
无 .gitmodules（整个母仓没有子模块）
```

→ 它是**开发者本地把独立仓克隆进母仓目录的镜像**，方便同仓改代码；母仓真正依赖的是 **PyPI 包 `JiuwenMemory>=0.2.0`**（`pyproject.toml:23`）。

**母仓实际用到了它的什么**（`jiuwenswarm/agents/harness/common/memory/jiuwen_rail_builder.py:181-195`）：

```python
from jiuwen_memory.api import Scope, Surface
from jiuwen_memory.common.security import new_request_context
from jiuwen_memory.common.security.types import AuthContext, Role

context = new_request_context(
    AuthContext(
        actor=Scope(org=org, user=user),   # ← org + user 两轴
        role=Role.USER,                    # ← 带角色的访问控制
        credential_type="internal",
        auth_method="internal",
        authenticated_at=datetime.now(UTC),
    ),
    surface=Surface.INTERNAL,
)
```

并且 `build_jiuwen_memory_rail(...)` 的 docstring 明确：**`agent_name` 优先作为 scope 的 agent 轴**，使记忆按 agent 隔离，缺省才回落 config 的 `agent_id`，再缺省 `"jiuwenswarm"`。

**配置面**（`jiuwenswarm/resources/config.yaml`）：

| 行 | 键 | 含义 |
|---|---|---|
| `:165-167` | 注释 | 用 `JiuwenMemoryRail` + `AgentMemoryMemoryProvider` 替换原生 `MemoryRail`；jiuwen 变体依赖 site-packages 的 `jiwen_memory_adapter` 包，**按需显式开启** |
| `:177-179` | `mode: ${JIUWEN_MEMORY_MODE:-sdk}`、`base_url: ${JIUWEN_MEMORY_BASE_URL:-http://127.0.0.1:8137}`、`api_key` | `http` = 远程 AgentMemory HTTP 服务（带 Bearer）；`sdk` = 进程内内核 |
| `:260` | `provider: ${MEMORY_EXTERNAL_PROVIDER:-}` | 可选 `openjiuwen / mem0 / openviking / lakebase / jiuwenmemory` |
| `:303-304` | `mode: ${JIUWEN_MEMORY_MODE:-server}`、`tenant_id: ${JIUWEN_MEMORY_TENANT_ID:-default}` | **`tenant_id` 注释为「org 轴」** |

> **对「用户画像 / 记忆隔离」答复的影响（重要）**：记忆引擎层**有** `Scope(org, user)` + `agent` 轴 + `Role` + `Surface` 的访问控制模型，还支持 http 模式带 Bearer 的远程记忆服务。之前「只有自由文本 Markdown、没有隔离机制」的说法要修正为：**引擎有 org/user/agent 三轴与角色模型，缺的是把这些暴露成对外管理接口，以及默认值（`__default__` / `tenant_id=default`）需要集成层按租户填**。

---

## 6. `agent-security`：安全能力集，母仓零 import（走 HTTP）

**包清单**（各自独立 pyproject）：

| 目录 | 包名 | 内部结构 |
|---|---|---|
| `jiuwenbox/` | `jiuwenbox` 0.1.0（FastAPI + mcp） | `src` / `docker` / `scripts` / `tests` |
| `agent_ssas/` | `agent-ssas` 0.1.0 | `src` / `tests` / `design` / `examples` / `patches`（181 文件 / 120 py） |
| `agent_shield/` | `agent-shield` 0.1.0 | `detection` / `intercepts` / `policy_engine` / `tool_permission` / `schema` / `native` |
| `security_gateway/` | `security-gateway` 0.1.0 | `src` / `tests` |
| `foundation/` | `security-mask` 0.1.0 | `security_mask` |

**jiuwenbox 是什么**（`agent-security/jiuwenbox/README.md`）：

> `jiuwenbox` is a lightweight Linux sandbox service for running agent tools and code snippets with layered isolation. It exposes a FastAPI server for sandbox lifecycle management, file transfer, …

**关联方式 = HTTP，不是 import**：

| 侧 | 位置 | 内容 |
|---|---|---|
| 母仓配置 | `jiuwenswarm/common/config.py:3759-3769` | `idle_check_interval`（默认 None = 让 jiuwenbox 用自身默认）、`fallback_on_failure`、`token`（jiuwenswarm↔jiuwenbox Bearer）；超时通过 **`PUT /api/v1/timeout`** 写到 jiuwenbox server 根 policy |
| 母仓常量 | `jiuwenswarm/common/config.py:3787-3788` | `JIUWENBOX_API_TOKEN_ENV = "JIUWENBOX_API_TOKEN"`（与 jiuwenbox server / CLI / provider HTTP client 共享） |
| **客户端在 agent-core** | `openjiuwen/extensions/sys_operation/sandbox/providers/jiuwenbox.py`；`openjiuwen/core/sys_operation/sandbox/launchers/pre_deployment_launcher.py:35-45` | `sandbox_type in ("jiuwenbox","jiuwenbox-conch")` → `build_jiuwenbox_shared_scope_key` / `delete_jiuwenbox_sandbox` |
| Gateway 侧 | `agent_gateway/common/config.py:2238-2453` | 同一套 sandbox 配置；`:2453` 注释「agent-server boot 时判断要不要自动拉起 jiuwenbox 子进程」；`:2340` `<=0` 归一化为 None（禁用淘汰）与 jiuwenbox server 端一致 |
| 母仓权限侧 | `jiuwenswarm/agents/harness/common/rails/permissions/execution_provider_contract.py:7,19-36` | `JIUWENBOX_SANDBOX_EXECUTION_TOOLS = {"bash","mcp_exec_command","code"}`，决定自动执行是否必须留在 JiuwenBox 内 |
| 母仓路径保护 | `…/rails/permissions/protected_paths.py:8` | `"jiuwenbox"` 在受保护路径名单里 |

**档 B 证据（拆到一半）**：母仓里还有一份 **被 git 跟踪** 的 jiuwenbox 源码副本：

```
git ls-files jiuwenbox  → 108 个文件（真身 agent-security/jiuwenbox = 121 文件）
```

而且母仓 `jiuwenswarm/common/config.py:4120-4145` 找 policy 模板时**先找本地源码目录** `ancestor/jiuwenbox/src/jiuwenbox/configs`，找不到才 `import jiuwenbox` 走已装 wheel（`:4131`，带 `# type: ignore[import-not-found]`）→ **母仓自带副本优先于已拆出的包**。

**`AgentSSAS` 同理但更彻底**：母仓 `AgentSSAS/` 48 文件里**只有 3 个 py**，只剩 `design/ docs/ examples/ patches/`（且被 git 跟踪，如 `AgentSSAS/design/AgentSSAS_01_…md`）；真身 `agent-security/agent_ssas/` 有 `src` + `tests`（181 文件 / 120 py）→ **代码已搬走，设计文档留在母仓**。

> **对「运行时资源回收」答复的影响**：沙箱侧的空闲 TTL（`idle_ttl_seconds`，jiuwenbox 默认 600s）属于**独立安全仓 agent-security/jiuwenbox** 的能力，可通过 `PUT /api/v1/timeout` 调整、`<=0` 表示禁用淘汰。答复口径可以说：**沙箱隔离与空闲回收由可独立部署/升级的安全组件 jiuwenbox 提供，属可插拔能力**。

---

## 7. `agent-runtime`：母仓只用它的 `foundation/`

**包清单**：`foundation`(`openjiuwen-runtime-foundation` 0.1.0)、`server`(`agent-runtime-server` 1.0.0)、`service`(`openjiuwen-runtime-service`)、`management`(`openjiuwen-runtime-management`)、`cli`(`agent-runtime-cli`)、`applications/{a2a-service, ir-execution-service, openjiuwen-runtime-applications-llm-agent, lowcode-agent-runner, versatile-adapter, openjiuwen-runtime-examples-workflow-agent}`，另有 `agent-studio/`、`docker/`。

**母仓 / Gateway 只用到 `foundation` 下的这些模块**：

| 模块 | 使用方 |
|---|---|
| `foundation.security.link_mtls_config` / `link_profile` | `jiuwenswarm/common/security/link_mtls.py:20,25`、`agent_gateway/common/security/link_mtls.py:20,25`（两侧同构，企业链路 mTLS） |
| `foundation.db.postgresql_handler` / `db.utils(is_mysql,is_postgresql)` | `jiuwenswarm/runtime/checkpointer.py:130,188`（断点/检查点持久化） |
| `foundation.audit` | `agent_gateway/config/audit/access.py:16`（访问审计） |
| `foundation.db.table_def` | `agent_gateway/config/enterprise/tables/{a2a_models,application_config_models,cron_job_models}.py`（**企业版数据表定义**） |

> **对「多用户并发 / 多租户」答复的影响**：企业版的持久化表结构与审计、mTLS 落在 **runtime-foundation 层**（被 Gateway 使用），不在 jiuwenswarm 里。租户身份与池仍在 jiuwenswarm（`enterprise/tenant/`），持久化 schema 在 runtime-foundation。

**不需要单独细看**：`applications/`（低代码 / IR 执行 / a2a / workflow 示例）、`cli/`、`server/`、`service/`、`management/`、`agent-studio/` —— 那是独立产品面，jiuwenswarm 不 import。

---

## 8. 还没拆出来的（全在母仓）

### 8.1 `jiuwenswarm/jiuwenswarm/` 一级模块（文件数）

| 模块 | 文件数 | 说明 |
|---|---|---|
| `channels/` | **2151** | **通道层只拆了一半**（见 8.2） |
| `server/` | 556 | RPC dispatch + 各 adapter：传输拆走了，**handler 没拆** |
| `resources/` | 487 | `config.yaml`（约 1900 行）、agent 模板、skills |
| `agents/` | 463 | 产品级 harness（rails / 工具 / 记忆 / 权限） |
| `extensions/` | 143 | agentos / video_duplex / redis / runtime_management |
| `common/` | 135 | 配置读取、e2a、auth、security |
| `runtime/` | 56 | **jiuwenswarm 自己的 runtime 编排**（与 `agent-runtime` 仓不是一回事） |
| `enterprise/` | 45 | **多租户全在这里**：`tenant/{tenant_identity,tenant_agent_pool,async_lru_cache,tenant_paths}`、`team/quota`、`capabilities/` |
| `office` 17 / `clouddoc` 12 / `observability` 11 / `diagnosis` 11 / `infrastructure` 10 / `instance_manager` 6 / `acp` 4 / `cli` 7 / `scripts` 3 / `tests` 10 | | |

### 8.2 通道层拆分边界

| 侧 | 内容 | 文件数 |
|---|---|---|
| `agent-gateway/agent_gateway/channel_manager/` | IM 通道管理（含 web 文件 HTTP、feishu 等） | 114 |
| `jiuwenswarm/jiuwenswarm/channels/web/` | **Web 前端 + web 通道**：`frontend/` 1787 文件（969 个 ts/tsx/js/vue + 资源），py 只有 6 个（`app_web.py`、`custom_agent_upload_http.py`、`designer_upload_http.py`、`hub_oauth.py`、`share_image_export.py`） | 1787 |
| `jiuwenswarm/jiuwenswarm/channels/` 其余 | `tui` 172、`browser` 65、`ide` 51、`process_cli` 40、`desktop` 19、`cli` 7、`acp` 2 | 364 |

→ **前端与全部终端类通道（TUI / IDE / 桌面 / 浏览器 / CLI）都在母仓**，Gateway 只拆走了服务端通道管理与传输。

### 8.3 对照：agent-core 与 agent-gateway 的内部结构

- `agent-core/openjiuwen/`：`core` 873、`agent_teams` 583、`harness` 596、`agent_evolving` 340、`rsi` 363、`symphony` 207、`dev_tools` 100、`extensions` 135、`harness_providers` 43、`harness_protocol` 16、`auto_harness` 1
- `agent-gateway/agent_gateway/`：`channel_manager` 114、`common` 98、`enterprise` 94、`extensions` 66、`config` 39、`routing` 17、`cron` 17、`runtime` 16、`storage` 24、`auth` 11、`message_handler` 9、其余（`config_poll` / `health_check` / `heartbeat` / `hooks` / `im_pipeline` / `storage_assembly`）

---

## 9. 两个容易混的同名概念（写答复时别踩）

1. **两层 harness**：`agent-core/openjiuwen/harness`（596 文件，内核 rails）与 `jiuwenswarm/jiuwenswarm/agents/harness`（463 文件，产品级 rails / 工具 / 权限）。同名不同物，引用时要写清仓名。
2. **两个 runtime**：`jiuwenswarm/jiuwenswarm/runtime/`（56 文件，AgentServer 自己的会话/检查点/交互编排）与 `agent-runtime` 仓（低代码 / IR 部署运行时产品）。二者无关。

---

## 10. 复现这六个仓

```powershell
# 母仓（github）
git clone --depth 1 -b develop https://github.com/openJiuwen-ai/jiuwenswarm.git

# 按 pyproject 钉的 rev 精确复现外部仓（gitcode 允许按 SHA 直接 fetch，无需鉴权）
git init <dst>; git remote add origin <url>
git fetch --depth 1 origin <rev>
git -c lfs.fetchexclude='*' checkout -f FETCH_HEAD
```

母仓克隆时 LFS smudge 会 404，用：

```powershell
$env:GIT_LFS_SKIP_SMUDGE='1'
git -c lfs.fetchexclude='*' checkout -f HEAD
```

`docs/assets/videos/*` 的 LFS 对象不在本地，**不要去读**。

---

## 11. 本次核对暴露的三处答复口径修正

| 客户问题 | 旧口径 | 修正后口径 |
|---|---|---|
| 4 用户画像 / 记忆 | 只有自由文本 Markdown，无隔离机制 | 记忆引擎（独立仓 `agent-memory` / PyPI `JiuwenMemory>=0.2.0`）**有** `Scope(org,user)` + agent 轴 + `Role` + `Surface`，并支持 http 模式带 Bearer 的远程记忆服务；**缺的是对外管理接口与默认值填充**（`memory.builtin.jiuwen.user_id` 默认 `__default__`、`tenant_id` 默认 `default`） |
| 2 运行时资源回收 | 空闲回收散在各处 | 沙箱侧空闲 TTL 属**可独立部署的安全组件 jiuwenbox**（`PUT /api/v1/timeout` 可调，`<=0` 禁用淘汰）→ 可插拔能力，措辞可更积极 |
| 1 多用户并发 | 多租户全在 jiuwenswarm | 租户身份与池在 jiuwenswarm（`enterprise/tenant/`），但**企业版表结构 / 审计 / mTLS 在 runtime-foundation 层**（Gateway 使用 `foundation.db.table_def`、`foundation.audit`、`foundation.security.*`） |
