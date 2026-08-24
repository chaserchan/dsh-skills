# DSH 插件开发踩坑库（实战记录，接手勿重复）

> **维护（跨会话）**：任何会话在本机做 DSH 开发时，遇到新坑/新经验，按 `dsh-dev/SKILL.md` 的"经验沉淀契约"**追加到本文件**（已有条目勿删改，只追加），并同步到版本化源 `D:\job\developer\DSH\skills\dsh-dev\references\pitfalls.md` 后提交推送 dsh-skills 仓库（Gitee + GitHub）。动手前先通读本文件，已记录的坑不要重复试。

> 每条都是本机实测踩过的坑，含根因与解法。新坑请追加到这里并同步进 dsh-skills 仓库。

> ## 🔎 总则（陈公子硬性习惯 · 必须遵守）：遇问题先查记忆，全程沉淀
> 所有开发过程都要求沉淀，因此**遇到问题/新坑/需决策时，第一步是查已有记忆，不要从零重解**：
> 1. 本文件 `pitfalls.md`（跨会话共享踩坑库，已记录的坑不要重复踩）；
> 2. 工作区 `AGENTS.md` + 插件/项目 `DEVELOPING.md`（本地沉淀）；
> 3. **Obsidian vault**（`D:\job\obsidian\chaseman`，可检索）——搜相关笔记；
> 4. 查到的结论与现状不符 → 记"过期修正"（新事实覆盖旧记忆）。
> 查完记忆仍无解，才重新调研/实验。

## 1. 插件"莫名 404 / 被禁" → 先查 dshmarket 停用名单（真根因，2026-08-23 插桩证实）

**现象**：`dsh-media-capture` 的 client 端 `/plugins/<id>/client.js` 恒 404、`__DSH_BOOT__` 没有它；host 能被加载（`--dump-config` 有 entry）；浏览器无 icon、无 console 报错。

**真根因**：**dshmarket（第三方插件市场）的持久化停用名单**里有该插件。

- 文件：`$DSH_HOME\profiles\web\.dsh-market\state.json`，原值 `{"disabled":["dsh-media-capture"],...}`；
- 机制（`dshmarket/lib/routes.js`）：启动时重放 `disabled` 名单 → `setEntryDisabled(name,true)` → `Entry.update` 销毁 fiber → 出 graph → client 404；并挂 `internal/plugin` **自愈守卫**，任何复活尝试都会被再次禁用；
- 插桩铁证：`Entry.update ← dshmarket/lib/themes.js setEntryDisabled ← routes.js`（此前 9 种改法全无效的原因就是被它按回去）；
- **解法**：`state.json` 的 `disabled` 置空（`{"disabled":[],...}`），插件代码零改动，重启即恢复。

**次要诊断**（旧结论，保留备查）：cordis `internal/plugin` 的 `parentMatch=false` auto-disable 是**表象**（fiber 被 dshmarket 销毁时的附带事件），不是根因；`file:` 本地 + `dsh.client` 双面组合本身可正常工作（media-capture 现以 `link:` 形态稳定进 graph）。

**排查顺序**：DSH 插件 404/被禁 → ① 先查 `.dsh-market/state.json` 的 `disabled` 名单；② 再查 cordis loader 内部。**不要**先逐字段改插件。

## 2. `dsh plugin add <目录>`（link 安装）不自动装依赖

link 方式把包链接进 profile，但**不会**安装它的 `dependencies` → host `import` 第三方包时 MODULE_NOT_FOUND。**必须在插件目录内自行 `pnpm install`**。

## 3. link → registry 切换遇 `ERR_PNPM_EPERM`

pnpm 替换 junction 时 Windows 权限报错（`importPackage ... EPERM`）。解法：
```sh
pnpm remove <pkg>                              # 先移除依赖声明
Remove-Item -Recurse -Force node_modules\<pkg> # 手动删残留 junction（关键）
pnpm add <pkg>@<version> --registry=https://registry.npmjs.org
```

## 4. npmmirror 镜像同步延迟

发布新版本后 npmmirror 可能滞后（实例：0.1.2 发布后镜像仍显示最新 0.1.1，`ERR_PNPM_NO_MATCHING_VERSION`）。解法：`pnpm add <pkg>@<ver> --registry=https://registry.npmjs.org` 直连官方源。

## 5. npm 发布 2FA 策略（2026 新规）

classic automation token（`npm_` 前缀）**不再被允许绕过 2FA 直发包**（403："Two-factor authentication or granular access token with bypass 2fa enabled is required"；classic token 计划 2026-12 移除）。解法：**Granular Access Token（bypass 2FA）** 或 OTP。发布必须显式 `--registry=https://registry.npmjs.org/`（本机默认 npmmirror）。

## 6. 手写 client bundle 的格式红线

- 必须是 `window.__ModuleLoader__.load({id, factory})` 单文件（lazy-CJS）；
- `id` 用**裸包名**；`require` 只能引基线模块；
- **只导出 `apply` + `inject`**。失败案例：多导出 `exports.name`（`dsh-media-capture` 有、`global-prompt` 无）——虽未证实为根因，但整体对齐能减少变量。

## 7. GitHub 直连被墙、SSH 不可用

SSH 22/443 均 `Connection closed`（网络阻断）。解法：**HTTPS + 凭据持久化**（`~/.git-credentials` + `git config --global credential.https://github.com.helper store`）；建仓用 GitHub API（`Authorization: token` + `User-Agent` 头必填）。

## 8. Gitee 建仓默认私有，且 API 无法转公开

Gitee API 创建仓库时 `private` 参数（表单/JSON 各种形态）**全部无效**，建出来恒为私有（账号默认可见性=私有）；PATCH 转公开 406/400/422 全被拒（PATCH 需 `name` 字段但仍有 422）。解法：建仓后**用户在网页改**（管理 → 基本设置 → 仓库类型 → 公开）；代码推送不受影响。

## 9. settings namespace 正则

`/^[a-z][a-z0-9-]*$/`（小写 kebab），违反复合即装载抛错。host/client 两侧必须同名（命名空间是配对键）。

## 10. profile 操作与运行中 web 的锁冲突

web 运行期间改 profile（pnpm install 等）可能遇锁文件 EPERM。先停 web（或直接替换 node_modules）再操作；改完让守护进程/用户重启。

## 11. dshmarket 卡片描述机制（不是 bug，是设计）

市场卡片描述**不读 package.json**，只读云端目录 `awesome-dsh-plugin.com/plugins.json`（`{zh,en}` 对象；来自 awesome-dsh-plugin 精选列表的 `data/plugins/*.yml`）。字符串 description 在卡片上显示为空。上架门槛：`dsh.bundle`（只有 dsh.client 会被拒）、仓库满 1 天、≥10 commits、`dsh-plugin` topic、描述属实。详见 dshmarket插件上架指南（Obsidian DSH 目录）。

## 12. 系统提示词注入即生效

`systemPrompt.section` 的 `text` 每次模型步进重新求值——设置保存后**下一条回复即生效**（含已有会话），无需重启；空文本段落被渲染器过滤（留空=停用，天然开关）。

## 13. media-capture 完整排障链：装载 → 激活 → 可见（2026-08-23 实战，三段独立问题）

DSH 插件"界面不出现"要分三段排查，每段现象与根因不同：

1. **装载**（client bundle 404、`__DSH_BOOT__` 无）→ 根因是 dshmarket `.dsh-market/state.json` 停用名单（见 §1），**插件代码零改动**；
2. **激活**（bundle 200、无 console、apply 未跑）→ client bundle **只导出 `apply`+`inject`**（移除 `exports.name`），`inject` 对齐基线模板（`["slots",...]`）；用 console 诊断日志（apply 入口）确认；
3. **可见**（apply 已跑、无 icon）→ **先确认需求是"新增可见元素"还是"劫持既有元素"**。composer 工具行加按钮要用**官方槽位 `conversation.input.left`**（list/session，渲染在「＋」旁）：
   ```js
   ctx.slots.inject("conversation.input.left", () => ctx.slots.register({
     name: "conversation.input.left", id: "dmc-camera", order: 10,
   }, () => react.createElement(CameraButton, { setOpen })));
   ```
   （dsh-client-ui-plan 注册 `conversation.input.plan` 同款；ui-conversation 在 L7256 `leftItems = renderSlot("conversation.input.left", zone)` 渲染）。
   **不要用 DOM 劫持「＋」按钮**——它会顶掉斜杠命令菜单，且不新增可见元素（本案例初次"无 icon"的根因）。

## 14. 客户端 UI 排障法（无头浏览器不可用时）

无 gstack browse/无浏览器工具时，用**可观测修复**：在 client 的 apply 入口、槽位挂载点、事件处理加 `console.log("[tag] ...")`，让用户硬刷新后回传 Console。三步定位：`apply running`（激活？）→ `xxx mounted`（挂载？）→ 交互日志（逻辑？）。诊断日志在定位后保留（`[dmc]` 前缀）或删除均可。

## 15. DSH 会话 token 只存内存 → 重启后全员被迫重登（真根因，2026-08 dsh-user-system 实测）

**现象**：用户登录后，每次重新打开页面（尤其移动端）都要重新登录；"cookie 明明没到期"。

**真根因**：`store.js` 的 `sessions = new Map()` 是**进程态**——DSH 被 watchdog 重启/崩溃轮或任何进程重启，**全部会话清空**（cookie 在浏览器里 8h 未失效，但服务端已查无此 session → `/session` 返回未认证 → 强制登录）。

**解法**：会话落盘。token 以 `sha256(token)` 作 Map 键（不存明文），`createSession`/`revokeToken` 立即 `persist()`，`load()` 用 `restoreSessions()` 恢复并过滤过期/已删除/禁用用户。验证：reload 后 `validateToken` 仍有效。cookie `SameSite=Strict→Lax`（移动端/反代更稳）。

## 16. 移动端窄表格 + `.cards` 基础规则写在 `@media` 之后 → 卡片被盖掉（实测踩两次）

**现象**：移动端（375px）管理面板"用户与权限"表格 520px 超出视口，靠容器内 `overflow-x:auto` 滚动；移动端滚动条常隐藏，用户觉得"宽度有问题/坏了"。改成卡片堆叠后，卡片仍空白。

**根因**：
- 桌面表格在窄屏用 `overflow-x:auto` 内滚，属"能滚但看不出"，体验差；
- 加移动端卡片后，`.${uid}cards{display:none}`（桌面隐藏）如果写在 `@media` **之后**，与 `@media` 里的 `display:block` **同级优先级、后来居上** → 移动端也被 `display:none` 盖掉 → 表格+卡片双双隐藏 → 整块空白。**残留两个 `.cards{display:none}` 更是必踩。**

**解法**：桌面表格 + 移动端卡片**双视图**（`renderUsers` 同时输出 `.desk`(表格) 和 `.cards`(卡片容器)）；`@media(max-width:720px)` 里 `.desk{display:none}` + `.cards{display:block}`；**基础的 `.cards{display:none}` 必须写在 `@media` 之前**。改完用 Playwright 在 375px 实测 `overflowX===0` + DOM `display` 断言。

## 17. 登录/品牌 icon 用用户提供的 SVG + `flex:none`

给 logo 行（flex 容器）内联 SVG 必须带 `style="flex:none"`（否则在 flex 里被拉伸/变形）；SVG 用 `fill="currentColor"` 才能跟随 DSH 主题变量。用户给出成品 SVG 时**原样替换**，不要重画。

## 18. `file:` 依赖不复制 `test/`，且 smoke.mjs mock 缺 `inject`

pnpm `file:` 安装只按 `package.json` 的 `files` 字段拷贝——`test/` 常不在 node_modules 副本里，`test/smoke.mjs` 在 node_modules 内跑会 `MODULE_NOT_FOUND`。且该 smoke mock `ctx` 无 `inject(["webServer"])` 方法，`enabled:true` 分支抛 `ctx.inject is not a function`（**既有问题，非新代码引入**）。判定后端改动是否破坏导入：跑 `apply(enabled:false)`（打印 OK 即 import/config 无误），逻辑层用 `node` 直接单测 store.js。

## 19. 改包名/加 bundle 必须重装 → `cannot resolve profile bundle`；本地插件用 `link:`（2026-08 dsh-user-system 改名实测）

**现象**：改了插件 `package.json` 的 `name`（`dsh-user-system` → `dsh-plugin-user-system`）+ profile 依赖 key + bundle 名后，`dsh web` 报 `cannot resolve profile bundle "dsh-plugin-user-system"`。

**根因**：**改包名 ≠ 重装**。`node_modules/` 里还是旧目录名 `dsh-user-system`，pnpm `dsh.profile.bundles` 解析新包名找不到目录 → 报错。改名后目录名/内容都要重新物化（+1 -3 重建）。

**两类成因**（`cannot resolve profile bundle "X"`）：
1. `bundles` 数组加了 X，但 `dependencies` 没 X 条目 / 源码目录没建 → 移除幽灵 bundle 或补齐依赖；
2. 改了 `package.json` `name`，但没重装 → `pnpm install` 重装即可（会重建 `node_modules/<新名>`）。

**关键：本地插件一律用 `link:`，不用 `file:`**。
- `profile/pnpm-workspace.yaml` 里 `nodeLinker: hoisted` 时，`file:` 本地插件是**复制进 node_modules（非软链/junction）**，且实测 `pnpm install --force` **不会**重新拷贝文件内容（显示 `Packages: -2` 之类却不变）；改源码必须手动 `cp -r` 或重装。
- 改用 `"<pkg>": "link:D:/job/.../<dir>"`，pnpm 建成**软链**，源码改动实时生效，无需重装/手动cp。用户已把 media-capture & user-system 切 `link:` 验证通过。
- 例外：`link:` **不自动装依赖**（见 §2）。若 `link` 包有第三方依赖，需确保其依赖能通过 profile 的 hoisted node_modules 解析，或在该包目录内自行安装。

## 20. 往 DSH 宿主 UI（如左侧栏）注入自定义元素 → 必须镜像宿主项计算样式（2026-08 dsh-user-system 实测）

**现象**：往 DSH 左侧栏注入的自定义菜单项（用户管理 / 退出账号 等）走浏览器默认 `<button>` 样式，与"设置"等宿主项**高度/宽度/内边距/对齐不一致**，偏左、对不齐；只对某一项复制盒模型、别的没有 = **必然错位**；宿主折叠(窄屏 rail)或主题切换后再次错位。

**根因**：DSH 侧边栏是 React 组件，宿主项样式是运行时计算出来的。注入的原生元素若假设默认样式、或硬编码固定值，永远对不齐；且宿主折叠态(rail)、主题切换会动态改宿主盒模型。

**解法**：用一个**统一构建函数** `makeSideItem(id, icon, label, onClick)`：
1. 找到宿主锚点项（如文本"设置"的可见按钮）；
2. `getComputedStyle(host)` 读 height/width/padding/box-sizing/display/alignItems/gap/color 等，**克隆到注入项**；
3. 用 `setInterval`/`ResizeObserver` **动态跟随宿主盒模型**——含窄屏 rail：`justifyContent:center` + `padding:0` + 隐藏文字(span display:none) + `width/height` 实时同步；
4. 所有注入项走同一函数（不要每项各写一份样式）。

**通用教训**：**往任何第三方宿主 UI 注入元素，不要假设默认样式，要镜像宿主项的计算样式 + 动态适配**——这是"宿主注入"类需求的通用规律。

## 21. DSH 后端状态在进程内：改落盘数据文件不经重启不生效
**现象**：改了 `users.json` / store 数据文件（如重置密码），但登录/行为不变。
**根因**：store 状态在 DSH 进程内存，`persist()` 只写盘不 reload；进程内 store 不会自动读文件。
**解法**：改后端落盘数据 → **必须重启 DSH**。区分：后端(`app/store/permissions`)改动 → 重启；`serveFile` 提供的前端静态文件(`gate.js`/`login.html`)改动 → **强刷(Ctrl+Shift+R)即可**（serveFile 实时读盘）。

## 22. schemastery 无 `.optional()`；bundle 自动 `insert` 勿重复（duplicate loader entry id）
- `@deepseek-ai/schemastery` **没有 `.optional()`**——字段默认即可选，可选字符串写 `z.string()`；写 `.optional()` 报 `z.string(...).optional is not a function`。
- profile 的 `dsh.profile.bundles` 列出的包若声明 `dsh.bundle.patch`，则是 **bundle**，会自动 `insert` 其 patch 的 entry（如 `id: user-system`）。**profile 自己的 `cordis.patch.yml` 想改配置时，必须按 id 覆盖（顶层 `- id: xxx` + config），不能再 `insert` 同 id** → 否则 `duplicate loader entry id: xxx`。非 bundle（只有 `dsh.client.inject`、无 `dsh.bundle.patch`）才需手动 `insert`。

## 23. 注入 DSH Web UI：主题用 `--dsw-alias-*` 勿硬编码 hex；`box-sizing:border-box` 必加；原生 select 美化
- **主题跟随**：注入的登录/面板/侧边栏用 DSH 主题变量 `--dsw-alias-label-primary/tertiary`、`--dsw-alias-border-l2`、`--dsw-alias-bg-layer-3`、`--dsw-alias-brand-primary` 等，**不要硬编码 `#hex`**（否则 DSH 切白/深主题后 UI 不跟随）。
- 任何 `width:100%` + `padding` 的元素（输入框/按钮）**必须 `box-sizing:border-box`**，否则内容盒撑破容器溢出。
- 原生 `<select>` 默认丑：`appearance:none` + 内联 SVG 箭头(background-image) + 主题色边框美化。

## 24. DSH webserver 无后端认证 = 安全边界：前端门禁 ≠ 拦 API
DSH webserver 明确 `No TLS, auth`。`gate.js` 之类只做**前端门禁**（挡界面）；未登录者写脚本直接调 DSH `/api/*` 仍可读数据。真正的认证必须在 DSH 之前（前置网关/反代），**不要指望前端遮罩能拦 API**。

## 25. Windows/PowerShell 两个常见坑：中文注释乱码、curl.exe 传 JSON 引号失灵
- UTF-8 无 BOM 的 `.ps1` 被 PowerShell 按 GBK 读 → 中文乱码 → 字符串终结符错乱。**脚本里只用 ASCII**。
- `curl -d '{"a":1}'` 在 PS 引号失灵；用 `Invoke-WebRequest -Body (@{...}|ConvertTo-Json)` 或 `Invoke-RestMethod`。

## 26. 快捷诊断：Playwright 驱动系统 Edge；Node 单测 store
- 无 gstack browse 时：`p.chromium.launch(channel="msedge")` 驱动系统 Edge（免下载 chromium），做移动端/登录/面板布局实测 + 截图。
- Node 侧 store 逻辑：`node --input-type=module -e` 快速单测 `store.js`（`persistMode:"memory"`）。

## 27. 移动端 input 聚焦自动缩放：根因是字号 <16px，`viewport` meta 拦不住（2026-08 dsh-user-system 实测）
**现象**：手机上点击输入框，页面被放大（iOS/Chrome 的 input-focus 自动缩放）。用户第一反应是"viewport 没生效"。
**真根因**：`<meta name="viewport" content="width=device-width, initial-scale=1">` **拦不住**这个自动缩放。真正触发条件是**聚焦的 `<input>` 计算字号 < 16px**（浏览器为让输入框可读而自动放大）。只有 `input { font-size:16px }` 或 viewport `user-scalable=no` 能拦。
**解法**（保留无障碍优先）：
- **主解**：给注入的 `<input>`/`<select>` 显式 `font-size:16px`（覆盖继承的 14px）——可靠、live、不牺牲 pinch-zoom 无障碍。DSH 注入 UI 常用 `font:inherit`→14px，必踩。
- **补强**：服务端 `tapIndex` 把 viewport 强化为 `width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no`（需重启）；代价是**禁用整页 pinch-zoom**，如不需要可只留 `maximum-scale=1`。
- 验证：Playwright `getComputedStyle(input).fontSize` === "16px"（375px 移动端视口）。

## 28. 第三方插件生态组合：正交分层 + GPL/MIT 许可证红线（2026-08 dsh-pocket 案例）
**场景**：想在一个 DSH 插件里"整合或记录"另一个第三方插件（如把 `dsh-pocket` 手机远程接入编进 `dsh-plugin-user-system`）。
**判断**：
- **职责正交的就"叠加/记录"，不要"整合"**。`dsh-pocket`（手机远程接入/反向代理/cloudflared 隧道）+ `dsh-user-system`（登录门禁/RBAC/共享策略）正交 → **分层叠加成双认证**（先 dsh-pocket 8 位访问密码 → DSH → user-system 账号登录），两者独立、各自升级。
- **License 红线（铁门槛）**：`dsh-pocket` 是 **GPL-2.0**；`dsh-plugin-user-system` 是 **MIT 且已发 npm**。把 GPL 代码整合进 MIT 包 → 整个包变 GPL 衍生、必须整体 GPL 开源，已发版本违规。**因许可证不同，不设依赖、不整合代码，只做可选文档记录**（README「推荐搭配插件」纯文字告知装配方式）。
- **记录进 README 而非功能**：第三方生态关系写进 README 的「推荐搭配插件（可选·非依赖）」小节，明确"本品可单独完整使用 + 该插件为可选加分项 + 叠加关系 + 许可证限制"。不要用 admin 面板显示插件清单（scope creep，违背单职责）。
**教训**：组合插件时**先看许可证**（GPL/AGPL 传染性强，勿与 MIT/BSD 包整合）；正交的插件用"文档记录 + 叠加"，不要硬揉。

## 29. DSH composer 图片硬限制：宽高必须 ≤2000px（消息发送门禁，2026-08 media-capture 实测）

**现象**：上传图片后 DSH 提示"图片宽高不能超过2000px"，**整条消息发不出**。

**根因**：DSH composer/模型图片输入限制**宽、高均 ≤2000px**；任一超限即拒绝整个提交（不只那张图）。之前只按宽缩放，竖屏长图高度超限仍被拒。

**解法**：提交前**必须自动压缩**：
```js
const scale = 2000 / Math.max(w, h);   // 按更长边缩放，宽/高都收敛到 2000px 内
// 若缩放后文件仍超字节上限（如 4MB），自动降质量重压（0.85→0.4）直到达标
```
已合规（宽高≤2000px 且字节不超）的图**原样提交**，不重编码。

**通用教训**：图片类插件必须"**先合规再提交**"，否则超限导致整条消息被拒；且要**按更长边缩放**（不是只压宽）。

## 30. client 插件访问 ctx 服务必须 inject（否则 "without inject" 报错）

**现象**：`cannot get property "sessions" without inject`（跨端运行时错误）。

**解法**：`exports.inject` 声明用到的**服务名**：
```js
exports.inject = ["slots", "locale", "settingsScope", "sessions", "conversation"];
```
否则访问 `ctx.sessions` / `ctx.conversation` 抛错。另外 `sessions.list` 是 `createSnapshotStore`，读当前会话 id 用 **`sessions.list.getSnapshot().current`**（**不是 `.get()`**）。

## 31. composer 草稿仅支持图片（视频无法直接进附件）

`conversation.createDraftImages` 只收 png/jpeg/webp/gif（`imageMediaType` 校验），**视频会抛错**；`browserDraftAttachment` 的 `kind` 恒为 "image"。所以**原始视频文件没有 host 附件通道**。

**解法**：视频要进 composer 只能**压缩成单张封面图**上传（media-capture 早期做法：`videoToCompressedImage` 取首帧→按更长边缩到 ≤2000px→JPEG）；如需原始视频，需 host 视频附件能力（当前 DSH 未提供），只能到接口层扩展。

**方案修订（过期修正，2026-08）**：media-capture 最初把视频压成封面图，但用户真实预期是"视频原样发出去"，而 DSH 无视频通道、做不到 → **最终决定：移除视频，只保留图片**（上传面板 `accept="image/*"`，无 `video/*`；删掉 `videoToCompressedImage`/`isVideoType`/`VIDEO_TYPES`）。教训：**"视频怎么处理"是用户决策，不是技术默认**——平台无通道时先问用户"要封面图/要多帧/还是移除视频"，别自作主张压封面。这也说明 DSH 生态：**附件 = 图片专用**，任何视频都进不了 composer（与上一节 §29 的门禁同源）。

## 32. 契约测试断言方向 & 文档脱节（2026-08 media-capture 收尾核对实测）

背景：把 media-capture 从"拍照/拍视频/上传"收敛到"只保留图片"后，用户要求核对。**功能代码本身没问题，但核对揪出文档/测试与实现严重脱节**，其中一处是**测试断言方向整个反了**。

### 32a. client 插件契约测试断言方向：要断言"不导出 name"，不是"有 name"

**现象**：`node test/smoke.mjs` 报 `❌ client.name 等于 dsh-media-capture` 和 `❌ host name 等于 dsh-media-capture`（2 项失败）。

**根因**：smoke.mjs 按"直觉"断言 `client.name === "dsh-media-capture"`，但 **DSH client/host 插件契约恰恰是不导出 `name`**（导出 name 会被 cordis `internal/plugin` 判 `disabled`，进不了浏览器 graph）。这个测试写反了契约方向。

**解法**：冒烟测试断言**反面**——`check(!client.name, ...)`、`check(!dshMod.name, ...)`，并补 `client.inject 不含 apply/name`。修正后 ALL PASS。

**通用教训**：DSH 插件契约测试**不要凭直觉写"应该有 name"**，要按实际契约（`exports` 只有 `apply` + `inject`；host 只有 `apply`）写**否定断言**。这一条被 media-capture 踩坑结论（id≠name、不导出 name）反向验证了两次。

### 32b. 收敛功能后，文档/元数据/注释必须同步

**现象**：功能已"只保留图片"，但排查时 `grep video|拍视频|上传视频|抽帧|detectDevice` 仍命中大量残留：
- `README.md` 整篇还描述**已删除**的拍照/拍视频/抽帧策略/`detectDevice` 三端/`劫持＋按钮`/`file:` 安装；
- `package.json` `description` 残留"拍照 / 拍视频 / 上传视频…视频抽帧成图"、`keywords` 有 `"video"`；
- `cordis.patch.yml` 注释还写 `hijacks the composer's "＋" button`（实际早改用官方 `conversation.input.left` 槽位）。

**根因**：迭代收敛需求（删功能）时只改了功能代码，**没同步 README / package.json description / 代码注释**，导致对外门面与实现不一致（发布描述会误导、README 让使用者产生错误预期）。

**解法**：改功能后**用一把 grep 确认无残留**（`grep -i "拍视频|上传视频|抽帧|video/|detectDevice"`），并同步更新：package.json `description`+`keywords`、README 整篇、`cordis.patch.yml` 顶部注释。README 是门面，应重写为准确版本（能力/压缩策略/实现方式/安装/文件结构/踩坑）。

**通用教训**：**删功能 ≠ 只删功能代码**。收敛/移除功能时，文档（README）、元数据（description/keywords）、架构注释（cordis.patch 顶部）必须一起改，否则"核对"时门面全露馅。对照清单：description / keywords / README 能力表 / 功能注释 / 安装说明 / 测试断言。

## 33. 智谱 GLM Coding Plan 接入 DSH：glm-5.3 是纯文本模型，三连坑（2026-08-23 实战）

**场景**：把智谱 Coding Plan key 配成 DSH LLM provider（`llm-pi-ai.providers.zhipu`，baseURL `https://open.bigmodel.cn/api/coding/paas/v4`，模型 glm-5.3）。

### 33a. 贴图报 400 `code:1210 messages.content.type 参数非法，取值范围 ['text']`

**根因**：**glm-5.3 在智谱端点就是纯文本模型**（端点硬限制，与 DSH 配置无关）。给模型声明 `input:[text,image]` 只会让图片真发出去然后被端点拒——DSH 文档明说 `input` 是"声明不是检查"。

### 33b. 撤回 input 声明后报 `model-unavailable: session already contains images`

**根因**：**会话日志保留真实图片块**（曾在 input 声明生效期间贴过图），纯文本模型回放历史即整会话锁死，每条消息都报错。

**解法**：① 会话内救活：切到 `(modlens vision)` 孪生条目（请求时把历史图片转证据文本）；② 最干净：新开会话先选好模型再贴图。

**正确姿势**：glm-5.3 保持无 input 声明（默认 text-only）；看图一律走 modlens 桥（孪生条目或贴图自动落路径由 `modlens_read_image` 读）。

**key 存放规范**：secret 写 `~/.dsh/.credentials.yaml` 的 `refs`（如 `ZHIPU_API_KEY`），`settings.yaml` 用 `apiKeyEnv` 引用——DSH 官方推荐 secret 不进配置文件。

### 33c. modlens 自动给新 zhipu 路由挂 `(modlens vision)` 孪生条目

**现象**：模型选择器出现"智谱 GLM (Coding Plan) (modlens vision)"，用户疑惑。

**机制**（modlens `dsh/index.js`）：自动扫描所有 provider 路由，`families=['deepseek','glm']` 且模型未声明 image、id 不匹配视觉正则（`glm-4.5v` 这类带 v 的才是原生视觉）→ 注册包装 twin。**这是设计行为不是配置错误**；声明 image 后孪生条目自动消失。

## 34. modlens 视觉桥引擎：provider 配好 ≠ 被选用；glm-4v-flash 输出上限 1024 装不下证据（2026-08-23 实战）

**现象**：`~/.modlens/config.json` 里 openai 引擎已配好（智谱 key），贴图仍失败或截断。

**根因1**：**引擎选择由 config.json 顶层 `provider` 键决定**（缺省 `antigravity-cli`），providers.openai 配置存在≠会被用。doctor 的 "Selected provider" 段才是真相。
**根因2**：**glm-4v-flash 输出上限 1024 tokens**（端点硬限制 `[1,1024]`），modlens 结构化证据（summary+全文OCR+布局）装不下 → `finish_reason=length` 截断；设 4096 直接 400。

**解法**（实测通过）：
```sh
modlens config set provider openai
modlens config set openai.model glm-4.1v-thinking-flash   # 免费视觉推理模型，输出 16K
modlens config set openai.extraBody '{"max_tokens":8192}'
```
`glm-4.1v-thinking-flash` 免费、16K 输出、GUI/图表理解强项，正适合读界面截图。

**机制**：dsh 插件**每次贴图临时 spawn CLI 子进程**（无 `-p` 参数），冷读 `~/.modlens/config.json` → **改配置即时生效，无需重启 dsh**。另可用 `MODLENS_DSH_CLI` 环境变量覆盖 CLI 路径。

## 35. 本机 shell 网络出口分裂：curl.exe/Invoke-WebRequest 全挂，但 node fetch 正常（环境坑）

**现象**：pwsh 里 `Invoke-WebRequest`/`curl.exe` 访问任何 HTTPS（连 baidu）都失败：`HTTP:000` / `No credentials are available in the security package`；同机 **node 的 fetch 完全正常**（modlens CLI 直连智谱成功）。

**根因**：shell 层 TLS 栈/安全包问题，**不是网络不通**——别信 curl/IWR 的失败结论判定"机器出不了网"。

**解法**：需要联网验证时优先用 node：`node -e "fetch(...)"` 或直接跑对应工具的 node CLI。判定网络问题前先做个 node 对照实验。

## 36. OpenDesign daemon 端口：od 默认 7456，但 tools-dev 实际 7457（`OD_DAEMON_URL` 更正）

**现象**：`od lint <file>` / `od research` 等直连报 `failed to reach daemon at http://127.0.0.1:7456: ECONNREFUSED`，但 daemon 明明在跑（web 3000 可用）。

**根因**：`od` CLI（`apps/daemon/bin/od.mjs` → `dist/cli.js`）**默认连 `7456`**；而 `tools-dev run web --daemon-port 7457 --web-port 3000` 起的 daemon 在 **7457**。

**解法**：`$env:OD_DAEMON_URL='http://127.0.0.1:7457'`。

**附**：OpenDesign 数据根在 `%APPDATA%\Open Design\namespaces\default\data`（`app.sqlite` + `projects/<id>/` + `design-systems/<slug>/`），**不是** `D:\job\OpenDesign`。

## 37. OpenDesign design-system 的"发布"流程（body 才真实、目录是陈旧模板）

- run 生成的设计体系内容在**项目** `projects/<id>/DESIGN.md`（完整）；design-system 记录 `user:*` 的 `body` 字段从项目 workspace 的 `DESIGN.md` 读取。
- **发布** = `PATCH /api/design-systems/:id`，`body={"status":"published"}`；`status` 仅 `'draft'|'published'`（`DesignSystemStatus`）。
- **大坑**：`design-systems/<slug>/DESIGN.md` 可能是**陈旧占位模板**（只有 `# Title` + 各节 "List ..." 指令），而 API 返回的 `body` 才是真实内容。**判断内容看 `GET /api/design-systems/:id` 的 `body`，不要看目录文件**。

## 38. OpenDesign design-systems 目录按 workspace 隔离（"你的体系=0" 真根因）

**现象**：Web「你的体系」= 0，尽管 personal design-system 存在（`GET /api/design-systems` 无 header 能查到 153 个，含它）。

**根因**：`listAllDesignSystems`（`apps/daemon/src/design-systems/server-services.ts` L432-457）**按 workspace 隔离**：请求一旦带 `x-od-workspace-id` header，**personal（无 workspace 绑定）体系会被 `return false` 隐藏**。

**实证**：无 header → 153（含 personal）；带 workspace header → 152（personal 被过滤）。

**解法**（要让体系出现在某 workspace「你的体系」）：
1. 在该 workspace 下 `POST /api/design-systems`（带 `x-od-workspace-id` + `x-od-workspace-member-id` header，body 用 `title/category/surface/summary/body/status`）重建——用现有体系的 `body` 即可，不用重新生成；
2. 或用 `/design-systems/create`（setup 入口，新建项目工作区再生成）；
3. 切回个人（无 workspace）视图则 personal 体系可见。

**取 workspace/成员 id**：`app.sqlite` 的 `workspace_projects` 表（`workspace_id` + `created_by_workspace_member_id`，形如 `z25c001...`/`zgk...`）。

## 39. OpenDesign 新增 od 命令需 build 才进 dist

`od generate/rename/delete` 等加在 `apps/daemon/src/cli.ts`（源码）；`od.mjs` 实际加载 `../dist/cli.js`（**编译产物**）。**必须** `pnpm --filter @open-design/daemon build`（或 bootstrap）后 `dist/cli.js` 才含新命令，否则 `od --help` 看不到（`git` 只提交 `src` 改动，`dist` 是构建产物）。

## 40. OpenDesign 原型状态完备（用户硬性要求）+ `od lint` 校验

高保真原型**必须**含：懒加载 / 加载（骨架+spinner）/ 空 / 错误 / 过渡动画 + `prefers-reduced-motion`，且错误态别用 `--danger` 闪警（睡眠设备离线要用中性提示，符合 DESIGN §10.1）。

校验：`od lint <file> --fail-on none`（要先 `OD_DAEMON_URL` 连对 7457）→ `clean — 0 findings (P0/P1/P2)` 为通过。

## 41. Gitee push shallow 仓库被拒 + 建仓恒私有

- `! [remote rejected] main -> main (shallow update not allowed)`：本地 **shallow clone** 推 Gitee 被拒。解法 `git fetch --unshallow origin`（GitHub 慢可后台跑）补全历史再 push；超时被杀会残留 `.git/shallow.lock`，需删除。
- Gitee 建仓 `POST https://gitee.com/api/v5/user/repos` **恒私有**（`private` 参数无效），`{"name":...,"private":true}` 明确即可；chasechan token 在 Obsidian `个人/账号-代码托管.md`；SSH 推送用 `git@gitee.com:chasechan/<repo>.git`（本机 `id_ed25519` 已绑定 Gitee）。

## 42. OpenDesign od generate 生成的中文在 HTML head/注释乱码（GBK）

**现象**：`od generate` 出的 HTML，`<title>`/head 注释里的 em-dash（—）/中文间隔号（·）显示为 `鈥?`/`鐫＄湢`（GBK 噪声），body 中文正常。

**根因**：od generate（deepseek runtime agent）把 head/注释里的 em-dash（—）/UTF-8 中文以非 UTF-8 编码写入（em-dash 触发编码错乱）。

**解法**：prompt/brief 明确「HTML head/注释避免 em-dash（—）/中文间隔号（·），非 ASCII 只放内容区，正文 UTF-8，注释用 ASCII」；已入 huashu-design `advanced-techniques.md` 交付自查「字符/编码」项。

## 43. 贴图"视觉引擎失败"：modlens 3.23.x 剥不出思考模型的 JSON（2026-08-24 实战）

**现象**：往 DSH 贴图，用户侧图片正常内嵌显示，但模型侧报 `A pasted image could not be read: the vision engine failed`。

**根因链（逐环实测）**：
1. DSH 贴图 → modlens 插件 spawn **包内引擎** `node_modules/@liustack/modlens/dist/main.js`（与全局 PATH 无关）；
2. glm-4.1v-thinking-flash 是思考型模型，回复把证据 JSON 裹进思考文本（"After careful analysis, here's the final JSON:"）；
3. **3.23.x/3.24.0 引擎的 JSON 提取器剥不出** → 判定失败；**3.24.1 已修**（同图同模型实测通过）。
4. 坑中坑：`dsh plugin update --latest` 走 npmmirror 拿到 **3.24.0（还是老解析）**，镜像没同步 3.24.1（§4 复现）→ 必须在 profile 目录显式 `pnpm add @liustack/modlens@3.24.1 --registry=https://registry.npmjs.org`。
5. `config set openai.structuredOutput true` 无效：智谱端点对思考模型不强制 response_format。

**贴图失败的急救通道（不重试、先恢复）**：
- 附件本体：`~/.dsh/attachments/v1/objects/<sha256前2位>/<sha256>`（会话日志里 attachmentId 即 sha256）；
- 日志定位：活跃日志 `~/.dsh/sessions/<workspace>--/session-<uuid>/session.jsonl.zstd`（**挑字节数大的**，几百字节的是存根）；用 profile node_modules 里的 `fzstd` 解压，找 `user/message` 事件的 `attachment` 块拿 sha256；
- 拷出来直接 `npx @liustack/modlens analyze -i <file>` 或跑插件内引擎读图。
- 升级引擎后**贴图即生效无需重启**（§34：每次贴图临时 spawn 磁盘上的引擎，冷读）。

**已试无效项**：structuredOutput true（网关不支持）、3.24.0（解析未修）、全局装 modlens CLI（无关——插件 spawn 的是包内引擎）。


## 44. OpenDesign（od）是设计引擎：通过 prompt 驱动，不要手动改 HTML（2026-08-24）

**原则**：陈公子明确"od 是主要 UI 路径"。让 od 生成/改设计时，agent 容易自己动手改 HTML/CSS（违背定位）。正确做法是把设计需求写成 prompt，交给 `od generate`，让 od 的 runtime agent 用 skill + design-system 自己产出，agent 只做验收。

**关键命令/参数（实测有效）**：
```
od generate --project <id> --design-system <id> --skill <id> --prompt-file <path> --json
```
- daemon URL 用 env：`set OD_DAEMON_URL=http://127.0.0.1:7457`（od.mjs 默认连 7456，而 `tools-dev run web` 起的 daemon 在 7457；不设 env 会 ECONNREFUSED）。
- cli 入口：`D:\job\developer\OpenDesign\open-design\apps\daemon\bin\od.mjs`（桌面版 `od` 不在 PATH）。
- 常用 skill：`redesign-existing-projects`（在现有页面上接图/改）、`ui-ux-pro-max`、`design-review`。
- runtime 默认 deepseek-harness（bin=dsh），见 `/api/agents`。

**坑**：
1. `od generate` 是**重新生成**而非打补丁，会覆盖同名文件（somnovita-landing/dashboard/mobile-app.html）。**先备份**项目目录再跑。
2. 生成是异步的：命令只返回 `runId/conversationId`（秒回），实际写盘在几分钟后。判断进度看项目目录文件的 `LastWriteTime`（`.od-skills` 写入=agent 被唤醒）。
3. 若 od 生成的 `<title>` 是中文，ui-ux-pro-max 纪律要求 head/title 用 ASCII（防 mojibake）→ prompt 里加硬约束。
4. 媒体（生图）写进 od 返回的项目 id，可能不是"当前设计项目"——用复制把文件拿进目标项目目录。
5. 产品图：`od media generate --surface image --model vela/gpt-image-2 --project <p> --prompt "..."; od media wait <task> --since <n>`（慢模型返回 `task <uuid> still running` 文本，poll 到 exit 0 拿 `{"file":...}`）。
