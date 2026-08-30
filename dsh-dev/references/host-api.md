# DSH 宿主 HTTP RPC API（编程式会话/工作区管理）

> 来源：2026-08-30 实战打通（源码 `dsh-host-apiproxy/lib/types/api/*.schema.js` + 实测），用于**编程式创建/管理会话与工作区归组**，不必依赖 GUI 点击。参数以 schema 文件为准（zod 定义，最权威）。

## 1. 载体与信封（typert RPC over HTTP）

- **HTTP 端点**：`POST http://127.0.0.1:3080/api/<namespace>.<method>`
- **请求信封**（`client-request`）：
  ```json
  {"type":"client-request","rpcId":"<随机uuid>","method":"session.create","payload":{...}}
  ```
- **响应信封**（`server-response`）：
  ```json
  {"type":"server-response","rpcId":"<同id>","result":{"ok":true,"value":{...}}}
  // 失败: {"ok":false,"error":{"code":"...","message":"...","details":{...}}}
  ```
- **信任校验**（`dsh-client-connection` 的 isTrustedApiRequest）：Host 必须 loopback 或 trustedHosts；`sec-fetch-site` 不得为 cross-site；带 `Origin` 时其 host 必须与请求 Host 一致。本机 node/curl **不带 Origin** 即可通过。
- **WebSocket 下行**（只读事件流，非 RPC 载体）：`/api/events.mux`（多路事件）、`/api/events.host`（宿主事件）。
- **单独 HTTP 端点**：`/api/respond`（审批回执，GUI 用）。

## 2. 已探明端点（按域）

### workspace.*（工作区实体注册表，dsh-workspace 包）
| 端点 | payload | 返回 |
|---|---|---|
| `workspace.list` | `{}` | `{items:[{workspaceId,path,title,sessionIds}]}` |

- 存储：`~/.dsh/storages/workspace.json`（`tables.workspaces` = workspaceId → {path,title,sessionIds,createdAt,updatedAt}；`global.workspaceIds`=顺序；`global.archivedSessionIds`=归档集合）。
- 服务方法（cordis 服务定义，`dsh-tool-cordis` 反射可见但 agent 工具未暴露）：`create(path,title?)`/`get(id)`/`list()`/`delete(id)`/`insertBefore`/`archiveSession`/`resolveByPath(path)`；Workspace 实体另有 `attachSession(id)`（对照 workspace.path 校验会话头 cwd，不匹配拒绝）、`detachSession`、`insertSessionBefore`、`sessionIds`、`status()`。
- **归组语义**：会话只在 `attachSession` 后进入该工作区分组；未 attach 的会话 = GUI "未分组工作区"（cwd 对但归组错也是未分组）。归档（archiveSession）只移出分组视图，日志/席位保留，可恢复。
- **删除 workspace 记录 ≠ 删会话/删目录**；重注册同路径会生成新 workspaceId，旧会话不会自动重新接纳。

### session.*（sessions.schema.js 为权威）
| 端点 | payload | 返回 |
|---|---|---|
| `session.create` | `{workspaceId?\|cwd?(互斥), sessionId?, agentPreset?, reuseWorkspaceBlank?}` | `{sessionId, agentPreset?}` |
| `session.selectModel` | `{sessionId, provider, model, reasoningEffort?}` | `{selected:{provider,model,reasoningEffort}}` |
| `session.rename` | `{sessionId, title}` | `{title, seq}` |
| `session.models` | `{sessionId}` | `{current:{provider,model,reasoningEffort?}, routable, groups, failures}` |
| `session.list` | `{cursor?}` | `{items:[{sessionId,updatedAt,running,blank,parentSessionId?,origin?,cwd?,agentPreset?,projections?}]}` |
| `session.fork` | `{sessionId, atSeq?}` | `{sessionId}` |
| `session.history` | `{sessionId, beforeSeq?, maxMessages?}` | `{events, hasMore, projections?}` |
| `session.prompt` | `{sessionId, mode:"queue"\|"steer", content:[{type:"text",text}], clientTimeZone?}` | `{accepted:true, command?}` |
| `session.cancel` | `{sessionId}` | `{accepted:true}` |
| `session.attachment` | `{sessionId, attachmentId}` | `{attachment, data}` |
| `session.updateQueue` | `{sessionId, itemId, action}` | `{accepted:true}` |

- **`session.create` 归组关键**：带 `workspaceId` 时，cwd 自动取 workspace.path，创建后自动 `workspace.attachSession(sessionId)`——**这是"新建会话选项目"的 GUI 等价操作**；不带则落未分组。
- **`session.fork` 继承归组**：`forkWorkspace(source)`——源会话已归组则 fork 子会话自动 attach 到同一工作区；普通 loose lineage 不继承。
- **权限预设**：create 无独立字段，随 `agentPreset`（system 预设：standard/code(PTC)/minimal/cordis(创造)，见 `agentPreset.list`）；模型/思考深度在 create 后用 `session.selectModel` 设。

### 其他域
- `agentPreset.list` → `{presets:[{id,trust,isDefault,name,description}], authorable, hasDocument}`
- `settings.*`（settings.schema.js，update/replace/mutate 带 expectedRevision）、`approvals.*`、`credentials.*`、`downloads.*`、`goals.*`、`host.*`、`jobs.*`、`llm.*`、`questions.*`、`session-search.*`、`skills.*`、`subagents.*`——参数都在 `dsh-host-apiproxy/lib/types/api/` 对应 schema 文件。

## 3. 架构要点（源码定位）

- `dsh-host-apiproxy`：宿主 API 处理器 + zod schema（`lib/types/api/*.schema.js` 是每个端点的权威参数/返回值定义；`lib/index.js` 的 sessions/workspace 域处理器）。
- `dsh-client-connection`：把 `/api` 前缀注册为 HTTP route（bridge 到 fetchHandler），并注册 ws downlink（/api/events.mux、/api/events.host）；含 isTrustedApiRequest 信任校验。
- `dsh-api-gateway`：typert 网关（`connection.rpc.intercept("/api", ...)`），endpoint=`<namespace>.<method>`，严格参数校验（invalid client-request message = 信封/参数不合 schema）。
- `dsh-workspace`：workspaceRegistry 实体注册表（见上）；启动时按会话头 cwd 引导分组历史会话。
- `dsh-tool-cordis`：把宿主服务反射为工具契约文档（workspaceRegistry 等方法可见），但**agent 工具侧并未暴露 session.create/attachSession**——编程式管理走 HTTP RPC，不走 agent 工具。

## 4. 实战结论（chaseman 场景）

1. **agent 侧工具创建会话固定落"未分组工作区"**（如 chaseman_create_role_session：cwd 对但未 attach → 未分组）；正确方式=HTTP RPC `session.create` 带 `workspaceId`。
2. **一键工具已沉淀**：`D:\job\framework\chaseman\tools\dsh-create-workspace-session.mjs <项目绝对路径> <会话标题> [模型] [模式]`——内部完成 workspace.list 查 id → create → selectModel(v4-flash/max) → rename → 归组核实。
3. 专项会话模型默认 `deepseek-v4-flash`（陈公子 2026-08-30 指定）；`session.rename` 自定义标题（不跟随工作区名）。
