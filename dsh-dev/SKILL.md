---
name: dsh-dev
description: >-
  DSH（DeepSeek Harness）开发与 DSH 插件开发。适用于：开发/创建/调试/装载/发布 DSH 插件；把某个功能做成
  DSH 插件；写 package.json 的 dsh.client / dsh.bundle、cordis.patch.yml、client bundle（__ModuleLoader__）、
  settings 命名空间与设置卡片、slots 槽位、systemPrompt 系统提示词注入、工具/钩子插件；排查插件装不上、
  前端 /plugins/<id>/client.js 404、__DSH_BOOT__ 里没有它、插件不显示、设置不生效等问题；也适用于理解
  DSH 架构（cordis 分层、bundle/patch、client 模块图、settings/slots/capability seam）以及从官方仓库
  deepseek-ai/deepseek-harness 的文档/约定出发的 DSH 相关开发。凡用户说"做个 DSH 插件/DSH 开发/插件不显示/
  怎么给 DSH 加设置项/注入系统提示词"，都应使用本技能——即使没明说"DSH"（例如"给这个应用写个插件"）。
  本技能整合官方架构约定与本机实战踩坑经验，核心信条：一切皆插件；装载问题先对照"已知能进 graph 的模板"。
---

# DSH 开发与插件开发

## 身份与使命

DSH（DeepSeek Harness）是构建在 vendored Cordis 上的 **一切皆插件（everything is a plugin）** 的 agent harness。你的工作是：把功能做成**插件**，从开发、装载、验证到发布全流程一次到位，且不重复踩已知的坑。本技能 = 官方仓库（deepseek-ai/deepseek-harness）的架构/约定 + 本机实战经验（含踩坑库）。

## 成功定义（质量门槛）

1. **能装载**：`dsh --profile web --dump-config` 有该 entry；浏览器端 `GET /plugins/<id>/client.js` 返回 **200**，前端 `__DSH_BOOT__` 启动图含它；
2. **能看见**：功能在 GUI 中可见可用（设置页/槽位/按钮/面板）；
3. **能发布**：npm（含 `dsh.bundle`）+ Gitee/GitHub 双仓库同步；（可选）dshmarket 上架达标；
4. **不埋雷**：无敏感文件入库、不改官方包源码、不破坏既有插件。

## 工作流

### 第 1 步：定位环境（先于一切）

- `$DSH_HOME`（默认 `~/.dsh`）：`settings.yaml`（用户设置，热加载）、`profiles/web`（web profile）；
- web profile：`package.json`（dependencies + `dsh.profile.bundles`）、`cordis.patch.yml`（用户 patch 层）、`node_modules`（已装插件）；
- DSH 安装目录（npm 全局包 `@deepseek-ai/dsh` 及其 node_modules 下的 `@deepseek-ai/dsh-*` 源码）；
- 区分：**官方包源码**（只读参考，勿改）vs **你的插件**（可改）。

### 第 2 步：架构理解与扩展点选型

先读 [references/architecture.md](references/architecture.md)（官方功能→机制映射 + 本机结构速查），按功能选型：

| 想要 | 用 |
|---|---|
| 设置项/设置卡片 | settings 命名空间（双半侧同名）+ `settings.general.item` 行或 `settings.plugin.item` 卡片 |
| 系统提示词注入 | `ctx.systemPrompt.section({name, order, text})`（text 可函数，每步重求值） |
| 新增工具 | `ctx.tools.register()`（defineTool 或原始 JSON Schema） |
| 钩子/拦截 | `ctx.on('tools/pre-execute'|'agent/pre-step'|...)`，waterfall 必须 `next()` |
| UI 面板/弹层 | client 半侧 + slots（如 `shell.overlay`）或 ConversationNodeDefinition |
| 监听会话事件 | `ctx.on('session/event', ...)`，输入驱动回 `agent.followup()/steer()` |

### 第 3 步：搭建工程（复制已知可用模板，别从零发明）

DSH 插件 = **双半区同包**：
- host 半侧：`lib/index.js`（cordis 插件，`apply(ctx)`，用 `ctx.inject([...services], cb)`）；
- 浏览器半侧：`lib/client.js`（**必须**是 `window.__ModuleLoader__.load({id, factory})` 的 lazy-CJS bundle 格式，`exports.apply` + `exports.inject`）；
- `package.json`：`exports["./client"]`、`dsh.client{inject, platform:"web"}`、**`dsh.bundle{patch:"./cordis.patch.yml"}`**、`files` 白名单含 client 与 patch。

完整可复制的工程形态（含 global-prompt 这份已验证模板的逐字段清单）见 [references/templates.md](references/templates.md)。

### 第 4 步：装载启用

- 首选：`dsh plugin --profile web add <pkg>@<version>`（registry 版本号形态最稳；`link:` 本地形态也可但注意依赖问题）→ 若包声明了 `dsh.bundle` 会自动成为 profile layer；
- 或：profile `cordis.patch.yml` 追加 `- insert: [{id: <短id>, name: '<包名>'}]`（id 可与 name 不同）；
- 客户端自动被服务：`dsh-client-modules` 扫描**已启用的 loader entry** 中声明 `dsh.client` 的包 → 提供 `./client` 到 `/plugins/`。

### 第 5 步：验证（先 probe，不打扰运行中的 3080）

```sh
dsh --profile web --dump-config | grep <name>      # entry 在不在
dsh --profile web --port 3110 --no-open            # 独立 probe
Invoke-WebRequest http://127.0.0.1:3110/plugins/<id>/client.js   # 期望 200
# 主页 HTML 的 __DSH_BOOT__ 应含该 id（带 inject 顺序与 immediately）
```

### 第 6 步：发布

npm（显式 `--registry=https://registry.npmjs.org/`，bypass-2FA GAT）+ Gitee/GitHub 双仓库 + 可选 dshmarket 上架——见 [references/release.md](references/release.md)。

## 决策规则（关键，官方 + 实战）

- **客户端 UI 必有 `exports["./client"]` + `dsh.client`**；host 侧 `apply` 用 `ctx.inject([...], cb)`，client 端只导出 `apply` + `inject`（**不要**额外导出 `name` 之类的字段——本机有失败案例）。
- **可安装/可上架必有 `dsh.bundle`**：只有 `dsh.client` 的包无法作为 profile layer 安装、上架被拒（"无法安装"是最常见打回原因）。
- **装载疑难杂症不要逐字段碰运气**：本机实测 `file:` 本地 + `dsh.client` 双面组合可能触发 cordis 的 `parentMatch=false` auto-disable（9 种字段改法全部无效）。遇到就**整体复制"已知能进 graph 的模板"**（global-prompt 形态）再替换功能，不要继续逆向。
- **settings 命名空间**：小写 kebab（`/^[a-z][a-z0-9-]*$/`），host 与 client 两侧同名；写 `$DSH_HOME/settings.yaml`，热加载即生效（系统提示词每步重求值）。
- **注册都是 effect**：`ctx.effect/ctx.on`，`register()` 返回 disposer；waterfall 监听必须调用 `next()` 否则短路链条。
- **验证先行**：任何装载改动先用独立 probe 验证，再让用户重启正式服务。

## 边界

- 不改官方包源码、不动 vendor；新行为走**文档化扩展点**，不修改 agent-loop（官方铁律）。
- 不破坏既有插件（如 `dsh-user-system`/`dsh-chaseman-link` 等本地插件）。
- 不提交敏感文件；不伪造功能/截图/性能数据。
- 改 profile 装载前先处理 pnpm 锁文件 EPERM（停 web 或直接替换 node_modules）。
- 摄像头/媒体类功能仅限 localhost 或 HTTPS（HTTP 非 localhost 浏览器会拒）。

## 反面清单（实战踩坑，接手勿重复）

- 只声明 `dsh.client` 没有 `dsh.bundle` → 无法正规安装、市场上架被拒。
- `file:` 本地依赖 + `dsh.client` 双面组合 → 可能被 cordis auto-disable（见 pitfalls 的 media-capture 案例）。
- `dsh plugin add <目录>`（link 方式）**不会**自动装依赖 → 插件目录内要自行 `pnpm install`。
- `link:` → registry 切换遇 `ERR_PNPM_EPERM` → 先 `pnpm remove` + 手动删残留 junction + 再 `pnpm add <pkg>@<ver> --registry=...`。
- npmmirror 镜像同步延迟 → 发布后拉不到新版本时用 `--registry=https://registry.npmjs.org` 直连。
- npm 发布 2FA：classic automation token 已被 2026 新策略拦截（403），必须用 **bypass-2FA 的 granular access token** 或 OTP。
- 手写 client bundle 必须完全遵循 `__ModuleLoader__.load` 格式；`require` 只能引基线模块（react、`@deepseek-ai/dsh-client-runtime/client` 等）。
- GitHub SSH 被墙（22/443 都连不上）→ 用 HTTPS + 凭据持久化（store helper）；Gitee 建仓 API 默认私有且无法 API 转公开 → 需网页改。
- settings namespace 违反正则 → 装载即抛错。
- 改 profile `package.json`/`node_modules` 前不确认 web 进程占用 → EPERM。

## 复查循环

1. `--dump-config` 有 entry？`/plugins/<id>/client.js` 200？`__DSH_BOOT__` 有？
2. GUI 刷新/重启后功能可见可用？
3. 发布四件套：npm 版本、Gitee、GitHub、LICENSE/README 齐全？
4. 敏感文件检查（.env/密钥/内网地址）？
5. 未改官方包、未破坏既有插件？

## 参考资料

- [references/architecture.md](references/architecture.md) —— 官方架构、功能→机制映射、本机安装结构速查
- [references/templates.md](references/templates.md) —— 插件工程模板（global-prompt 完整形态、client bundle 骨架、settings/slots/systemPrompt 代码骨架、patch）
- [references/pitfalls.md](references/pitfalls.md) —— 踩坑库（含 media-capture `parentMatch=false` 全案例）
- [references/release.md](references/release.md) —— 发布流程（npm / Gitee / GitHub / dshmarket 上架）
