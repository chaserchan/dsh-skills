# DSH 架构速查（官方 + 本机实测）

> 来源：官方仓库 deepseek-ai/deepseek-harness（docs/architecture、docs/cookbook、AGENTS.md）+ 本机安装结构实测。

## 一、核心理念（官方）

- **一切皆插件（everything is a plugin）**：harness 构建在 vendored Cordis 上；新行为一律走**文档化扩展点**，禁止修改 agent-loop。
- **注册都是 effect**：每个贡献都经 `ctx.effect()` / `ctx.on()`；registry 的 `register()` 返回 disposer。
- **Capability seam 三角色**：Service Definition / Service Provider / Consumer，完整能力是三件套。
- **Waterfall 监听必须 `next()`**：不调用即短路链条（如 `tools/pre-execute`）。
- **Model-visible ⟺ logged**：任何进模型请求的内容必须可从会话日志重建。
- 包约定：`@deepseek-ai/dsh-<name>`、ESM（`"type": "module"`）、`@deepseek-ai/cordis` 为 peerDependency、`@deepseek-ai/schemastery` 在 dependencies。

## 二、功能 → 机制映射（官方 cookbook 摘录，选型表）

| 产品功能 | 插件机制 |
|---|---|
| 工具 | `ctx.tools.register()`（defineTool 或原始 JSON Schema）；schema 自动流入装配 |
| 钩子/权限门禁 | `tools/pre-execute`（返回 `{kind:'deny', reason}` 或 `next()`）、`tools/execute`、`tools/post-execute`、`tools/result`；`ctx.tools.guard()` 单调拒绝；`ctx.tools.restrict()` 工具过滤 |
| 系统提示词 | `ctx.systemPrompt.section({name, order, text})`，支持排序与作用域局部覆盖 |
| UI 插件 | 监听 `session/event`（`assistant/chunk` 等）；输入驱动回 `agent.followup()` / `agent.steer()`；业务行注册 `ConversationNodeDefinition` + keyed renderer |
| 设置 | settings 命名空间（host 注册 + 浏览器半侧同 key 配对）；`$DSH_HOME/settings.yaml` 热加载 |
| 计划/目标 | `ctx.goals`、`ctx.planMode` 等能力缝 |
| 子代理 | `ctx.subagents` provider 注册表 + `dsh-tool-subagent` |
| MCP | 每服务器一个插件：发现工具 → `ctx.tools.register()` |
| skill | section + 工具注册；调用时 `inject()` 注入内容 |
| 记忆 | section provider + 工具 |
| 定时任务 | 插件注册调度工具；触发 → 空闲 `followup(..., {source:{kind:'cron'}})` / 忙碌 `inject()` |
| 热重载 | 每个注册都是 effect → HMR 直接生效 |

## 三、双半区插件（本机实战验证的核心形态）

一个包住两个半侧：

```
pkg/
├── package.json      # exports["."] host；exports["./client"] 浏览器 bundle；dsh.client；dsh.bundle
├── cordis.patch.yml  # dsh.bundle 指向的 patch（insert loader entry）
├── lib/index.js      # host：cordis 插件 apply(ctx)
└── lib/client.js     # 浏览器：window.__ModuleLoader__.load({id, factory})
```

浏览器半侧如何被服务（本机源码实测 `dsh-client-modules`）：
1. 扫描**已启用的 Loader entry**（`entry.fiber` 存活且非 disabled）；
2. 解析包 `exports["./client"]`，哈希后写入 `window.__DSH_BOOT__` 启动图；
3. 经 `/plugins/<id>/client.js` 提供（no-cache，rev 作缓存失效）；
4. **host 半侧 fiber 必须稳定 enabled**，否则条目被移出 graph（client 404）。

## 四、本机安装结构速查

| 项 | 路径 |
|---|---|
| DSH Home | `~/.dsh`（`settings.yaml` 用户设置，热加载） |
| web profile | `~/.dsh/profiles/web`（`package.json` deps+bundles、`cordis.patch.yml`、`node_modules`） |
| DSH 安装目录 | npm 全局 `@deepseek-ai/dsh`（bin.js、config/agent-presets） |
| 官方包源码 | `@deepseek-ai/dsh/node_modules/@deepseek-ai/dsh-*`（含 cordis.patch.yml 可当 patch 语法参考） |
| 关键服务 | `dsh-settings-file`（settings.yaml 热加载）、`dsh-system-prompt`（systemPrompt 服务）、`dsh-client-modules`（client graph）、`dsh-host-plugin-inventory`（插件清单 RPC） |
| 命令 | `dsh web` / `dsh --profile web --dump-config` / `dsh plugin --profile web add <pkg>` |

## 五、常用验证点（装载是否成功的可观测信号）

- `--dump-config` 组合树里有该 entry；
- `/plugins/<id>/client.js` 返回 200（浏览器侧已进 graph）；
- 主页 HTML 的 `window.__DSH_BOOT__` entries 含该 id（带 inject 顺序 + `immediately`）；
- 设置页/对应槽位出现 UI（GUI 刷新后）。
