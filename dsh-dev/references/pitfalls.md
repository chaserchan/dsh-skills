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


## 45. 往 UTF-8 文件追加内容：别用 PowerShell Set-Content 重写（会整份乱码）（2026-08-24 实战）

**现象**：用 Set-Content 重写一个本来就好的 UTF-8 文本文件，结果整份变成乱码（GBK 噪声 + U+FFFD）。

**根因**：Windows PowerShell 5.1 的 Set-Content 用 .NET 源编码重建文件，遇文件里已有 em-dash/中文间隔号/中文标点等特殊字节时，与 -Encoding UTF8 不一致 → 重写破坏既有字节，中文全乱码。

**正确做法（按优先级）**：
1. Node 追加（最稳，纯 UTF-8，无 shell 转义坑）：require('fs').appendFileSync(path, txt, 'utf8')
2. PowerShell 用 Add-Content -Encoding UTF8（只 append，不重写已有正文）。
3. 绝不用 Set-Content 重写一个含中文/特殊字符的 UTF-8 文件。

**修复路径（一旦写坏）**：文件在 git 仓库里则 git checkout -- <file> 恢复 HEAD 即干净；之后用 Node 追加，不要再用 PS 重写。

**已试无效**：用 PowerShell -replace 正则去修反引号/反斜杠（PS here-string + 正则里互相吞，越修越乱）；根治是换 Node 或 Add-Content，不碰 Set-Content。

## 46. DSH 槽位四种 kind：chain 槽注册必须带 select（选举制），不能带 id/order（2026-08-25 实战）
**现象**：client 插件注册 `conversation.chat.turnTail` 时抛 `Uncaught Error: chain slot conversation.chat.turnTail requires options.select`，且**整个 apply 回滚**——主题已 layered 也会被连带 dispose，化身/卡片全部消失（连坐效应，一个槽注册错全插件挂）。
**根因**：DSH slots 系统有四种 kind：single（单占）/ keyed（按 key 分格）/ list（按 id 多行）/ **chain（选举制）**。chain 槽的契约（ui-renderer lib/client.js `spec.kind === chain` 分支）：每个 entry 注册时必须提供 `select(ownerProps)` 纯函数——渲染时依次询问，返回非 null 的第一个 entry 接管渲染（返回值作为 `matched` prop 传给组件）；返回 null 即让位。`id`/`order` 是 list 槽的选项，chain 校验只要求 select。
**官方姿势**（dsh-client-ui-deliverables 注册 turnTail 的写法）：
```js
ctx.slots.inject(conversation.chat.turnTail, () => ctx.slots.register({
  name: conversation.chat.turnTail,
  select: (owner) => { /* 纯读，无副作用；返回 {imgs} 或 null */ },
  locale: NS,          // 可选
  inject: () => ({})   // 可选：per-entry 服务注入
}, MyComponent));       // 组件收到 { ...ownerProps, matched }
```
**证据**：web-frontend dist `casechain:if(n.select===void 0)throw new Error(...)`；ui-renderer `matched = entry.select(ownerProps); if (matched !== null) { elected = guarded(entry, ..., {...ownerProps, matched}); break; }`。
**选举顺序注意**：官方 bundle 先加载先注册，故 deliverables 产物行优先；自己的 select 返回 null 时让位（无图/无产物互不干扰，有冲突时产物行赢——合理）。select 在渲染期调用，必须是纯函数（禁 setState/副作用），异常会被 catch 当 declined。
**chain 槽已知成员**：`conversation.chat.turnTail`、`conversation.composer`。写插件前先看 `dsh-client-ui-conversation/lib/types/client/contract/slots.d.ts` 确认目标槽的 kind。
**已试无效**：带 `id`+`order` 注册 chain 槽（直接抛错，apply 回滚全插件连坐）。

## 47. 给登录/注入 UI 做"可配置外观 + 主题跟随"（2026-08 dsh-user-system branding 实测）
**需求**：登录页标题/副标题/底部/logo/背景/主题色可配置，且**自动跟随 DSH 主题插件**；用户管理面板也要跟随主题。
**关键模式**：
- **对外观类配置用"公开 config 接口"**：登录页未登录也要渲染 → 加 `GET /api/config`（公开，无需认证）返回外观对象；admin 用 `PUT /api/config`（admin 认证）改。存 store 并落盘 users.json，字段白名单化 + 全部带默认值（留空=主题默认）。
- **主题跟随用 `--dsw-alias-*` 变量**：注入 UI 本就用这些变量 → `auto`（默认）时**不覆盖**，天然跟随 DSH 主题插件；`light`/`dark` 时在目标元素 `.style.setProperty('--dsw-alias-*', 值)` 覆盖即可；`primaryColor`/`cardColor` 再覆盖 `--dsw-alias-brand-primary`/`--dsw-alias-bg-layer-3`。
- **登录专属 vs 全局**：`theme`/`primaryColor`/`cardColor` 同时作用登录遮罩+管理面板（需求"面板也跟随主题"）；`title`/`subtitle`/`footer`/`logo`/`bgImage`/`bgColor` 只作用登录页。
- **用共享函数避免重复**：`applyTheme(el, branding)`（设置/清除主题变量）在两个根元素复用；`applyLogin(ov, branding)`（文本/logo/背景）仅登录遮罩。
- 颜色输入用 **text**（留空=主题默认）而非 `type=color`（后者无法表达空值）。
**验证**：Playwright mock `/api/config` → 断言登录遮罩 `title/subtitle/footer` 文本 + `style` 里的 `--dsw-alias-bg-layer-3`/`--dsw-alias-brand-primary`；管理面板 `.dsh-uds-modal` 的 theme vars；「登录设置」tab 表单预填值。

## 48. DSH 前端是 CSS-modules 哈希类名：用"后缀匹配"稳定定位，不要写死哈希前缀（2026 实战）
**现象**：给 DSH 工作台做定制皮肤时，类名是 `uV2eYG_root`/`qDHVXG_searchInput`/`pI_x6G_sidebarCol` 这类 `前缀哈希_语义后缀`。直接写 `.uV2eYG_root` 会在官方更新后失效。
**根因**：DSH 前端用 CSS modules，类名 = 随机哈希前缀 + 稳定语义后缀。哈希前缀每次构建/版本会变，后缀是稳定的（root/searchInput/sidebarCol/sessionRow/projectRow/turnStatus/turnStatusClock/bubble/action/toolRow）。
**正确姿势**：用**后缀匹配**定位——`[class^="_root"]`、`[class$="_searchInput"]`、`[class*="_sessionRow"]`。真正稳定、跨版本可用。要精确匹配某类用 `[class*="_xxx"].uV2eYG_xxx` 组合（针对已知后代精确）。
**佐证**：e2e 抓取 `document.querySelectorAll('[class*="_sessionRow"]')` 稳定命中；`[class*="_action"]::before` 能量点误伤图标按钮（见 49）。
**已试无效**：写死 `.uV2eYG_card` 这类哈希前缀类名做皮肤（官方一更新就断）。

## 49. 选择器过宽会误伤：`[class*="_action"]` 命中大量图标按钮，能量点全乱（2026 实战）
**现象**：给"工具行动行"加琥珀色状态点 `[class*="_action"]::before`，结果消息底部一排**图标按钮**（复制/编辑/重试等）全都冒出黄点，挤成一排很丑。
**根因**：`[class*="_action"]` 子串匹配会命中**所有含 _action 的元素**，包括纯图标按钮（p-xYUq_action/_8_XoUG_action 是操作图标行，不是工具调用行）。CSS-modules 后缀过于通用。
**正确姿势**：收窄选择器到真正有文本语义的类——用 `[class*="_toolRow"]`/`[class*="_toolCall"]`（工具调用行），而不是 `[class*="_action"]`（图标按钮行）。能量点只加在工具行。
**佐证**：e2e 验证 `getComputedStyle(el,'::before').backgroundColor` 发现 p-xYUq_action/_8_XoUG_action 都被置成 rgb(255,200,87)，去掉 _action 后恢复 none。
**教训**：给"动作/行动行"加装饰要先分清"工具调用文字行"vs"图标操作按钮"，后缀 _action 太泛。

## 50. 官方硬编码灰色不在 token 体系：uV2eYG_card / gdEzaW_bubble 都是 rgb(44,44,46)（2026 实战）
**现象**：深空驾驶舱主题下，输入框外层卡片、排队队列用户消息气泡仍是**浅灰色块**，跟深空底格格不入。
**根因**：DSH 某些组件用**硬编码颜色**而非 --dsw-alias-* 变量（如 `.uV2eYG_card` 背景 `rgb(44,44,46)`、`gdEzaW_bubble` 也是 `rgb(44,44,46)`），这些不在 overrideTokens 覆盖范围内，token 层够不着。
**正确姿势**：用 CSS 后缀匹配对具体组件类**强制覆盖**——`.uV2eYG_card`/`.uV2eYG_root`/`.uV2eYG_scroll`/`.uV2eYG_grow` 全部 `background:color-mix(in srgb,var(--dsw-alias-bg-layer-*),transparent)` + `!important`；`[class*="_bubble"]` 同理。交给玻璃拟态统一。
**佐证**：e2e 抓到 `uV2eYG_card bg rgb(44,44,46)`、`gdEzaW_bubble bg rgb(44,44,46)`，覆盖后变深空 `color(srgb 0.086/0.118/0.2 @0.88)`。
**注意**：这类硬编码灰官方可能更新，覆盖时保留后缀类名（uV2eYG_* / gdEzaW_*），别只写 hash 前缀。

## 51. 气泡定位：position:fixed 不随拖动，改 absolute 挂在 host 子元素（2026 实战）
**现象**：右下角 AI core 悬浮球加了互动气泡，球被拖走后气泡还留在原位（错位）；且 hover 才显示不满"常显"。
**根因**：① 气泡 `position:fixed` 相对视口定位，球（`position:fixed`）虽能拖但气泡没跟随；`host.offsetLeft` 对 fixed 元素取到 0，计算错位。② hover 触发，非 hover 不显示。
**正确姿势**：气泡作为 **host 子元素 + `position:absolute`** 相对 host 定位（`bottom:calc(100%+12px)` 球上方），天然随球移动；`overflow:visible`（host 是圆形 overflow:hidden 会裁掉气泡）；默认 `showBubble(true)` 常显 + 半透明低干扰。
**佐证**：e2e `bubbleIsChild:true`(球子元素)、`bubblePos:absolute`；fixed + offsetLeft 方案导致气泡错位到左上角。
**教训**：跟随可拖动元素的浮动层，优先做子元素 absolute，别用 fixed+JS 算坐标。

## 52. 官方类 border 被"更具体选择器"压制：`[class*="_turnStatus"]` 去不掉边框，要补官方类名（2026 实战）
**现象**：给 `[class*="_turnStatus"]` 设 `border:none !important`，Deep diving 状态条仍有 `1px solid rgb(42,61,99)` 边框。
**根因**：官方可能用更具体的 `.Md3f7G_turnStatus`（CSS-modules 具体类）设定 border，其优先级/特殊性压过 `[class*="_turnStatus"]`（属性选择器特殊性较低），`!important` 在同级 !important 下仍由更具体者胜出。
**正确姿势**：同时写官方类名 `[class*="_turnStatus"], .Md3f7G_turnStatus{ border:0 none !important; }`，用组合选择器提高特殊性；必要加 `border-style:none`/`outline:none`。
**佐证**：e2e `directBorder: 1px solid rgb(42,61,99)`，补官方类名后 `border:0px none`。
**教训**：`!important` + 属性选择器仍可能输给官方具体 CSS-modules 类，需补官方类名或提高特殊性。


## 53. DSH 前端加 Three.js 3D：运行时动态加载 ESM，不能 import 本地 node_modules（2026 实战）
**现象**：想在 DSH 插件客户端（lazy-CJS bundle）加 Three.js 3D 背景/化身。
**根因**：DSH client bundle 是 lazy-CJS（`__ModuleLoader__.load`），不是 ES module，无法 `import three`；three@0.185+ 只发 ESM（无 three.min.js UMD），本地 od node_modules 的 three 浏览器无法直接用（Node 打包路径）。
**正确姿势**：运行时动态注入 `<script type="module">`，里面 `const THREE = await import("https://cdn.jsdelivr.net/npm/three@0.185.1/build/three.module.js")`，挂 `window.__THREE` 再 dispatch 事件。**用事件 + 轮询双兜底**（module import 时序竞态：监听器可能拿到 undefined）。bundle 不膨胀，CDN 才加载。
**佐证**：e2e `window.__THREE` true；手动 dispatch vs 轮询兜底（轮询稳）。
**注意**：three 0.185 无 UMD（unpkg three.min.js 404），用 jsdelivr `build/three.module.js`（650KB）。

## 54. headless Edge 截不到 WebGL canvas：WebGL 层不参与普通 captureScreenshot（2026 实战）
**现象**：Three.js 背景/粒子 canvas 在 headless 截图里永远看不到，但 `canvas.toDataURL()` 有内容（WebGL 确实渲染了）。
**根因**：headless+SwiftShader 下 WebGL canvas 不走普通 canvas 合成，`Page.captureScreenshot` 抓不到 WebGL 光栅化结果（只抓到 DOM/2D canvas）。这是 headless 局限，**不代表真实浏览器不可见**。
**正确姿势**：验证 WebGL 视觉效果**必须用真实（非 headless）浏览器肉眼确认**，headless 截图只能验证"canvas 存在 + toDataURL 非空 + 无 console NaN 报错"。别用 headless 截图判 WebGL 粒子"不可见"。
**佐证**：toDataURL=30634（有内容）但截图纯深空；z-index 拉满仍看不见 → 实为 headless 局限。

## 55. DSH 插件 reload 会多次 apply：挂载元素必须防重复（2026 实战）
**现象**：加了右下角化身/背景，DSH 插件 HMR reload 后界面叠出多个相同的球/元素。
**根因**：DSH 插件热重载会多次调用 `apply`，`mountAvatar`/`mountXxx` 若无"先移除旧元素"保护，每次 apply 都新建一个 → 叠出多个。
**正确姿势**：凡用固定 id 挂载的宿主（`#dcp-avatar`/`#dcp-aurora`），创建前先 `document.getElementById(id)?.remove()`，保证唯一。或用 `ctx.effect` 清理。
**佐证**：console 多次 `[dcp] cockpit client apply`，右下角两个球 → 加 remove 后唯一。

## 56. originkit 组件库：MCP 取源码（需 key）+ catalog 免 key 本地打包（2026 实战）
**现象**：想用 originkit 的动画组件（snowfall/particlesphere/starburst）。
**根因**：originkit 是 MCP 组件库（`mcp.originkit.dev/vellumai`），`list_components` 免 key（catalog 本地打包 `src/component-index.json`），`get_component` 需 key（每天 10 次）。JSON-RPC 2.0 + `Authorization: Bearer <key>`；响应可能是 JSON 或 SSE（需解析 data: 行）。
**正确姿势**：`POST https://mcp.originkit.dev/vellumai`，body `{jsonrpc:"2.0",id:1,method:"tools/call",params:{name,arguments}}`。组件是 React/TSX，DSH 纯前端需**提取其核心算法**（three.js Points/InstancedMesh）用原生 JS 重写，不能直接用 React 组件。
**佐证**：fetch particlesphere 返回 200，tsx 含 three.js 核心（InstancedMesh+AdditiveBlending）。
**安全**：key 存 `~/.dsh/dsh-cockpit/originkit-key.json`（本地持久非 git 仓库），**绝不写进会被 git push 的 reminders/skills**。


## 57. DSH 背景层会被内容区玻璃块遮挡/干扰，且 "全屏覆盖" vs "不干扰文字" 难两全（2026 实战）
**现象**：给深空驾驶舱加全屏背景动画（雪花/Tornado/Morphing Rings），要么太小看不见，要么太大盖住界面/压文字，反复调不达意。
**根因**：DSH 内容区（sidebar/messages/composer）是**半透明玻璃**，背景 canvas 在其下；想让背景"全屏铺满"就会透过玻璃压到文字，想"不干扰文字"就得极淡 → 变成"蚊子大/看不见"。这是结构性矛盾，不是参数能完美调和。
**正确姿势**：① 涉及 DSH 工作台的背景动画要克制——**极淡 + 只在空白区域**（如 aurora 极光这类一直低调的）；② 真实全屏背景动画（Tornado/粒子）更适合**独立 demo 页**（originkit 官网那种），不适合塞进 DSH 工作台；③ 用户说"去背景"时立即停用 initThreeScene，别反复调。
**佐证**：canvas 全屏动画 avg=4~17，但截图里要么太亮盖屏、要么被玻璃块遮住显得淡；用户反复"不够满/太亮/太丑/去掉"。
**教训**：DSH 工作台的性质决定背景必须极谦逊，跟"产品演示页"的全屏动画需求冲突。定程度时先问用户"背景要极简底纹还是全屏主角"。

## 58. has 字符确认 canvas 有内容 ≠ 肉眼可见：litPixels 采样差异大（2026 实战）
**现象**：写 e2e 验证 canvas 背景"有内容"（getImageData 采到 lit>0），但截图/肉眼几乎看不到。
**根因**：`getImageData` 采样是**井字形跳采**（如 i+=4*399），对稀疏粒子时采样点容易全落在空白处；且加色混合（lighter）的微小粒子 alpha 低，阈值(>12)下被判暗。lit>0 只证明"有像素"，不证明"肉眼可感知"。
**正确姿势**：验证粒子背景"可见性"要**靠浏览器截图 + 肉眼**，不能只看 litPixel 统计。lit 数值只能当"有内容"的粗判，不能当"够亮够满"的验收标准。
**佐证**：同代码不同版本 lit 从 71(满)→16→4→2(淡)，但每版截图肉眼感受和 lit 不完全对应。
**教训**：视觉验收必须看截图/真机，canvas 像素统计只作辅助。

## 59. textarea 蓝边框去掉：border/shadow 用高特异性 + 官方类名压，outline 是默认 focus ring（2026 实战）
**现象**：`<textarea class="uV2eYG_input">` 有蓝色边框，`[class*="_input"]{border:none !important}` 压不住。
**根因**：官方用**具体类**（`.uV2eYG_input`）+ 可能内联样式 / `:focus-visible`，属性选择器 `[class*="_input"]` 特异性不够（同坑 #52）。
**正确姿势**：用**官方类名 + 高特异性**——`textarea.uV2eYG_input, .uV2eYG_input[data-phase]` 设 `border:0 !important / box-shadow:none !important / outline:0 !important`。这样 border/shadow 能清掉（e2e 实测 `border:0px none / shadow:none`）。但 `outline` 是浏览器默认 focus ring，`outline:none` 也压不住时属 Chrome 强制，需 `:focus-visible` 或 JS 内联。
**佐证**：e2e border=0px none + shadow=none，outline 仍报 rgb white 3px（浏览器默认 focus ring，非蓝）。

## 60. DSH 插件 settings.plugin.item 卡片不显示：host 未 serve namespace 或槽位/时机不对（2026-08-26 实战）
**现象**：按官方 adding-a-settings-card 路径（host installSettingsSection + client settings.plugin.item）注册工作台设置，设置页始终看不到卡片。
**根因链**（多因子，逐层排查后才能定位）：
① host 若 import `@deepseek-ai/dsh-settings` 且插件是 **link: 安装**（Node 从插件真实路径找依赖，够不到 profile node_modules）→ ESM 解析失败 → **host 崩、整个 DSH profile 起不来**（必须 junction/或放 profile 的 node_modules）。
② `settings.plugin.item` 卡片显示 = **Host 服务的 namespace（settingsScope.describe().view.namespaces）∩ client 注册的卡片 key**——官方 ui-settings-plugins 的 ConfigurablePluginsTabController.publish() 只显示交集；host 没 serve → 永不显示。
③ client 注册若用 `ctx.slots` 直接注册（非 `settingsScope` 作用域的 scoped.slots）或时序早于消费端订阅 → 卡片不在 ledger。
**正确姿势**：
- host：`installSettingsSection(ctx, settingsNamespace("你的-名"), schema, entry, {setSource, onChange})`，且 host 包必须在 profile node_modules 里可解析（link: 需 junction）；
- client：`ctx.slots.inject("settings.plugin.item", () => ctx.slots.register({name, id, key: 同名namespace, locale, inject}, Card))`——key 必须与 host namespace 一致；
- 验证：client 里 `ctx.settingsScope.describe().getSnapshot().view.namespaces` 必须含你的 namespace（这是"消费端会不会分发"的唯一裁决）。
**佐证**：describe().view.namespaces 含 dsh-cockpit 后仍不显示 → 才怀疑消费端/时机；我最后用 settings.section（见 61）直接进侧边栏菜单，绕开 plugin.item 的切片复杂度。

## 61. 设置侧边栏菜单项：注册 settings.section 槽，不是 settings.plugin.item/plugin.tab（2026-08-26 实战）
**现象**：要把"工作台设置"做成设置侧边栏的直接菜单项（跟通用设置/模型/插件并列），用 settings.plugin.item 或 settings.plugins.tab 都不对（那是"插件区"里的内容）。
**根因**：设置侧边栏菜单项（通用设置/模型/Agent 预设/插件）来自官方 **`settings.section` 槽**（dsh-cordis-client-runner 的 settings sections store 消费；client-ui-settings-models 注册 id=models/label=nav 即侧边栏项）。settings.plugin.item = 插件配置页里的卡片；settings.plugins.tab = 插件页内的 tab。
**正确姿势**：`ctx.slots.inject("settings.section", () => ctx.slots.register({name:"settings.section", id:"你的-id", order:15, label:()=>"菜单标题", inject:()=>({t:k=>k})}, SectionComponent))`——id 成侧边栏 nav 身份、label 成菜单标题、component 是点击后渲染的设置页。e2e 抓 nav 里含"工作台设置"。
**佐证**：设置侧边栏 items = [通用设置,手机访问,模型,插件,工作台设置,Agent 预设,插件市场]（含工作台设置）。

## 62. DSH 消息全文不渲染的隐藏杀手：extractImages 里 for...of 全局正则（LOCAL_RE is not iterable）打断消息组装链（2026-08-26 实战）
**现象**：整个会话的消息文字全部不显示（不只图片），但 console 无"明显"相关报错、或只有一个 Uncaught TypeError in extractImages。
**根因**：`const LOCAL_RE = /.../gi` 是**全局正则对象**，直接 `for (const m of LOCAL_RE)` 会抛 `TypeError: LOCAL_RE is not iterable`（全局正则本身不可迭代，必须 `text.matchAll(LOCAL_RE)`）。此错误发生在 conversationEvents.update（图片累积）→ ConversationNodeAssembler.acceptMatch 链上 → **消息节点组装中断 → 全文不渲染**。
**正确姿势**：遍历全局正则一律用 `text.matchAll(RE)`（或 .exec 循环）；`for...of 正则对象` 必抛 not iterable。同函数内 MD_IMG_RE/COCKPIT_RE 用了 matchAll 而 LOCAL_RE 忘记 → 一字之差毁全链。
**佐证**：console `Uncaught TypeError: LOCAL_RE is not iterable at extractImages ... at ConversationNodeAssembler.acceptMatch`；改 matchAll 后消息恢复。

## 63. 回车发送/输入框行为：别拦截 DSH 官方 InputBar 的 Enter，它原生就是提交（2026-08-26 实战）
**现象**：自加 keydown 拦截（preventDefault + 点发送钮）实现"回车发送"，结果：慢、双路径、运行中回车会打断 agent。
**根因**：DSH 官方 InputBar（dsh-client-ui-conversation L3779-3791）**原生处理 Enter**：`if (e.key==="Enter" && !e.shiftKey) { ... keyboard.arbitrate("enter", composing); e.preventDefault(); keyboard.submit(resolveSubmitMode(...)) }`。官方本来就是 Enter=提交、Shift+Enter=换行、运行中有 busy 判断。我再拦截 = 与官方双路径冲突（都 preventDefault/都 submit）→ 慢/误杀。
**正确姿势**：**不要拦截**——官方 InputBar 已支持 Enter 提交/Shift+Enter 换行；如需确认行为改 placeholder 即可（增强时只动 placeholder，不碰 keydown）。官方提交机注释："Enter during the Host round-trip is a no-op"（提交中 Enter 自动无操作，不会误杀）。
**佐证**：删掉自加 keydown 后回车发送恢复正常（官方原生）。

## 64. 页面卡死/无响应的排查方向：先查 console 异常链，再看"改 style 触发 MutationObserver 自触发循环"（2026-08-26 实战）
**现象**：DSH 页面突然无响应/卡死（可看 console 里的异常或干脆白屏）。
**根因**（两个该会话踩过的高危点）：
① `MutationObserver.observe(..., {attributes:true, attributeFilter:["style"]})` + 回调里**又改元素 style** → 改 style 触发 observer → 再改 → **无限循环** → 卡死。修复：只监听 childList / 或在回调里加 cancelled 标志防自触发。
② 全屏 canvas 动画（three.js/粒子）若逐帧渲染海量对象（未 clamp 的 segTotal/弧半径）→ 单帧几十万线段 → 卡死。修复：几何段数上限 + reduced-motion 降级。
**正确姿势**：MutationObserver 改样式时用 `mo.observe(el,{childList:true,subtree:true})`（不监听 attributes），或回调内 `if(!cancelled) requestAnimationFrame(apply)` 防自触发；canvas 动画必须 clamp 每帧运算量。
**佐证**：console `Cannot access 'applyBackgroundToDom' before initialization` 是 TDZ（const 声明在调用后）同样会卡设置注册；attributes 监听自触发循环是"改 style 无限循环"。

## 65. DSH e2e 工作区选择页守卫：headless 新 profile 登录后停在"选择一个工作区"（textarea readOnly），必须先点工作区才能测 composer（2026-08-26 实战）
**现象**：headless Edge 用**全新 user-data-dir** 登录 DSH 后，`textarea.uV2eYG_input` 存在但 `readOnly:true`、placeholder="选择一个工作区开始"，任何输入（ta.value 赋值 / CDP 真实键盘）都无效，斜杠菜单/回车发送全测不了。
**根因**：DSH 页面默认路由停在**工作区选择页**（body 文本：新会话/工作区/niucloud-master/plugin 等），composer 是只读占位；只有点击某个工作区卡片（如 `plugin`）后才渲染可交互会话视图。
**正确姿势**：e2e 脚本里登录后先 `[...document.querySelectorAll('button,[role="button"],li,div')].find(e=>(e.textContent||'').trim()==='plugin').click()` 进入工作区（等 3-4s），再操作 composer；或复用**已选过工作区的持久 profile**（如 dcp-e2e-profile）跳过选择页。headless 里无 `_sessionRow`/composer 不可交互也是此页特征（与 isMainUi 守卫同理）。
**佐证**：e2e-diag 抓到 ta readOnly:true + placeholder 选择工作区；点 plugin 后 readOnly:false + taPh 变为"给Chaeman发消息"，输入/斜杠菜单/回车全部恢复可测。

## 66. React 受控 textarea 模拟输入：直接 ta.value=+input 事件可能被重置，用原生属性 setter 或 CDP 真实键盘（2026-08-26 实战）
**现象**：e2e 里 `ta.value='/'` + `dispatchEvent(new Event('input',{bubbles:true}))` 后读回来 ta.value 是空（React 受控组件重置）；同一写法在另一次测超长文本又成功（sh=964），行为不稳定。
**根因**：React 受控组件以内部 state 为准，绕过原生 setter 的直接赋值 + input 事件在部分状态（readOnly/未进入会话页/组件刚渲染）下不被 React 采纳，渲染时被重置回空。
**正确姿势**（二选一，均已实测）：
- 原生 setter（React 一定会感知）：`const setter=Object.getOwnPropertyDescriptor(HTMLTextAreaElement.prototype,'value').set; setter.call(ta,'文本'); ta.dispatchEvent(new Event('input',{bubbles:true}))`；
- CDP 真实键盘输入（最保真，模拟手打）：`Input.dispatchKeyEvent {type:'keyDown', key:'/', text:'/', windowsVirtualKeyCode:191}` + keyUp，逐字符。
**佐证**：e2e-final2 用原生 setter 设 40 行 → scrollHeight 964 生效（限高/滚动可测）；e2e-slashtab 用 CDP 键盘输入 '/' → 斜杠菜单正常弹出。

## 67. 斜杠命令菜单 Tab 选中：拦截 Tab 后向 textarea 派发合成 Enter keydown（复用官方选中路径）；click 选项按钮无效（2026-08-26 实战）
**现象**：DSH 斜杠菜单（`[role='listbox'][aria-label='触发候选建议']`）打开后按 Tab 焦点跳到 `+` 号按钮（默认行为）；想"Tab 直接选中"时直接 `option.click()` 选了但 textarea 值不变、菜单不关。
**根因**：官方菜单选项 button 的 React onClick 未挂简单 click 路径（或挂在 pointer 事件上），`el.click()` 走不通；而官方**已原生处理 Enter**（选中高亮项 + 插入 `/cmd ` 带空格 + 关菜单），未处理 Tab。
**正确姿势**：textarea 上绑定 keydown：`if(e.key==='Tab'){ const menu=document.querySelector("[role='listbox'][aria-label='触发候选建议']"); if(menu && (menu.getBoundingClientRect().height>1 || menu.offsetParent!==null)){ e.preventDefault(); e.stopPropagation(); const ev=new KeyboardEvent('keydown',{key:'Enter',code:'Enter',bubbles:true,cancelable:true}); Object.defineProperty(ev,'keyCode',{value:13}); Object.defineProperty(ev,'which',{value:13}); ta.dispatchEvent(ev); } }`——与官方真实 Enter 完全一致（焦点也留在 textarea）；菜单关闭后 Tab 恢复默认（跳 + 号）。
**佐证**：e2e-simtest：dispatch-Enter → `v:"/agent-teams "` 菜单关；click-option → `v:"/"` 菜单仍开。e2e-slashtab 全项 PASS：menuOpened/inserted/menuClosed/focusStayed 全 true。
**补充**：composer 限高现状（官方已改）：官方 `.uV2eYG_scroll{max-height:var(--dsh-composer-text-max-height);overflow-y:auto}`、`.uV2eYG_input{height:100%}`（不再是固定 52px）+ 注入 `textarea.uV2eYG_input,.uV2eYG_input[data-phase="plain"],textarea[data-phase="plain"]{max-height:180px !important;overflow:hidden auto !important;min-height:56px !important}` 与 `.uV2eYG_grow,.uV2eYG_scroll{overflow:hidden !important}` 已生效：超长文本 180px 封顶 + 内部滚动（sh 964 / scrollTop 可设），双滚动条已消除。

## 68. DSH 插件"主界面守卫"三大坑：登录文案判据误杀 / 15s 窗口不够 / 轮询漏累加 — 整插件不挂载（2026-08-27 验证 70186 实战）
**现象**：dsh-cockpit 在**已登录主界面**上整个不挂载：avatar 无、背景无、composer placeholder 不改（仍是官方"描述你想要构建的内容"）、回车发送/斜杠菜单增强全失效，console 仅一条 `[dcp] cockpit client apply` 无后续。设置页/`--dump-config`/`/plugins/<id>/client.js` 全正常（200）——**装载链路 OK 但 UI 不生效**。
**根因**（3 层，需逐层验证）：
① `isMainUi()` 用**登录文案否定判据**（`/登录后进入|选择工作区|sign in|请登录/` 匹配 `bodyInnerText.slice(0,400)`）：dsh-user-system 插件会在**已登录主界面**渲染"登录后进入 DeepSeek Harness | 账号 | 密码 | 验证码 | 登录"登录表单（位于 body DOM 前 400 字符内）→ 误判为登录页 → 整个 cockpit 跳过挂载。任何其他插件往页面注入含这些词的可视文本都会误杀。
② 等待窗口 15s 不足：DSH 冷启动（尤其带用户体系等插件 + headless 新 profile）页面先停"工作区选择页"（placeholder"选择一个工作区开始"）再渲染主界面，`_sessionRow` 出现常 >15s → 插件提前放弃（打印"非主界面，跳过"再不复起）。**插件 apply 时机早于 React 主界面渲染，窗口必须覆盖慢加载**。
③ `waitMain` 只在首次调用累加 `waited`，**轮询 setInterval 回调里漏掉 `waited += 500`** → `waited>=15000` 永不成立 → "非主界面"兜底日志永不打印、poll 永不结束（轻微资源泄漏 + 行为不确定）。
**正确姿势**：主界面判据只用**结构判据** `!!document.querySelector("[class*='_sessionRow']")`（登录页/工作区选择页不渲染会话行，结构自足；**绝不用文本判据**——其他插件注入的自由文本会误伤）；等待窗口 15s→60s；轮询回调内 `waited += 500` 保证超时兜底真实生效。
**佐证**：probe4 同页 `hasLoginText:true`（正则命中）+ `hasSessionRow:true`（确实是主界面）→ isMainUi() 却 false；修复后同页 MOUNT `placeholder:"给Chaeman发消息", hasAvatar:true, hasAurora:true, dcpEls:21`，全部注册日志（theme tokens layered/images definition/工作台设置 section）出现。
**教训**：DSH "主界面/登录页"判定永远选**结构**（会话行/sidebar 容器），不选**语义文本**（文案会被其他插件污染）；对慢加载要"长窗口轮询"而非"短窗口放弃"。


## 69. DSH e2e 三个新坑：持久 profile 落在"单会话专注视图"判据失效 / SVG className 是对象 / agent 忙时普通 Enter 进排队不发送（2026-08-27 验证 70186 二次实战）
**现象**：
① 用**同一持久 user-data-dir（.tmp-dcp-cdp）重启** headless Edge 打开 3080，页面落到**单会话专注视图**（body 以"文件浏览/导出会话日志/复制ID/Session log/轨迹"开头，**无侧边栏会话列表**），`[class*='_sessionRow']` 不存在 → cockpit 结构判据 isMainUi() 60s 超时 → "非主界面，跳过挂载"，placeholder 仍是官方。而第一个 Edge 实例（主界面视图：新会话/工作区/会话列表）挂载成功（placeholder 给Chaeman发消息、dcpEls>0、theme tokens layered）。
② CDP Runtime.evaluate 里 `el.className.split()` 抛 Uncaught——SVG 图标元素的 `className` 是 **SVGAnimatedString 对象**，不是字符串。
③ **agent 处理中** InputBar placeholder 变 `Cmd/Ctrl+Enter 插话发送全部排队消息`（官方插话模式）：此时普通 Enter **不发送**——composer 清空（afterVal=""）但消息不出现（sentOnce=0），进入"排队消息"计数（出现"N 条排队消息"按钮）。
**根因**：① DSH web 前端记住了上次会话视图（单会话专注模式无 sidebar）；结构判据 `_sessionRow` 只对"主界面视图"成立——**判据没错，是验证环境视图不对**（与 #68 判据选型互补；该视图属"无侧边栏独立 UI"，cockpit 有意跳过）。② Web API 特性（SVG 的 className 是动画对象）。③ 官方 busy 判断（提交中 Enter no-op/排队）——与 #63 同样结论：官方原生行为，别当 bug 修。
**正确姿势**：
- cockpit 挂载/发送验证前先确认页面是**主界面视图**（有 `_sessionRow` + 侧边栏）；落在单会话专注视图就 Page.reload 或点"新会话"回主界面；最稳：重启浏览器后 reload 一次再判。
- DOM 取 class 一律 `el.getAttribute('class')`（HTML/SVG 通吃），不要 `el.className`。
- Enter 发送断言前提：agent 空闲 + 无排队消息（检查 placeholder 含"插话"即跳过或先清排队）。
**佐证**：旧实例同页 `sessionCls:["hHd-Xa_newSession","YDXeBa_sessionRow",...]` + sessionRowAny:true → cockpit 全量挂载；新实例同页 `sessionCls:["hHd-Xa_newSession","nL4_yW_sessionLogButton"]` + sessionRowAny:false → 60s 超时跳过，bodyHead 开头"文件浏览/导出会话日志"。SVG className 坑：v1 脚本 `__err:"Uncaught"`，v2 改 getAttribute 后通过。插话坑：send 模式 `filled:"DCPSEND70186", afterVal:"", sentOnce:0` + "2 条排队消息"按钮。

**已试无效**：点「打开侧边栏」按钮（此时 `_sessionRow` 立即出现）后 Page.reload → 页面仍恢复为侧边栏收起（第二次 apply 判非主界面跳过，但首次 apply 已成功挂载一次并打印全部注册日志）——展/收状态未随点击持久化（或持久化键不在 view|route|session|focus|mode 字面键中），reload 读回旧收起状态。恢复干净验证环境的最稳做法：同一浏览器会话内点开侧边栏后**立即**验证（不要再 reload），或直接清 localStorage 相关键。

## 68. 官方 SnapshotSelectorHook（useSessions 等）是 React hooks：只能在组件渲染期调用，setInterval/回调里调用必挂（2026-08-27 战情室实战）
**现象**：在 shell.overlay 组件里用 `useEffect(() => { setInterval(() => { const s = useSessions(...); }, 1000); })` 拿当前会话 id → 界面组件渲染了但状态永远是初始值/null，"⤢ 分屏"点了没反应（dispatch 的 id=null）。
**根因**：`useSessions` / `useWorkspaces` 等 framework **SnapshotSelectorHook** 本质是 React hooks（内部依赖渲染调度/订阅器），**只允许在组件渲染期调用**（官方 AgentPresetLabel 就是渲染期 `useSessions((state) => state.byId[sessionId]?.agentPreset)`）。放进 setInterval/事件回调 = 违反 hooks 规则 → 静默失败/返回旧值。
**正确姿势**：组件渲染期直接调用 `const cur = props.useSessions((st) => st && st.current || null)`（带 selector 取小字段）；store 变化时 framework 自动触发重渲染取新值，无需自建轮询。调用放 try/catch（拿不到返回 null）。
**佐证**：e2e 改渲染期调用后：点击"⤢ 分屏"→ 当前会话自动加入分屏板（open:true / cards:1），useSessions.current 精确取到活跃会话 id。

## 69. 官方 conversation.session.header.actions 对第三方晚注册条目不渲染（occupant 固定）；拿到活跃会话/全量快照的正路是 shell.overlay 的 useSessions（2026-08-27 战情室实战）
**现象**：按官方 slot 文档注册 `ctx.slots.inject("conversation.session.header.actions", ...)` 打 log 成功，但动作区永远只有官方 2 个条目（agent-preset/job-list，`[class*="headerActions"]` kids:2），自己注册的按钮不出现。
**根因**：header.actions 是 **session scope** list 槽，第三方 apply 时注入的 entry 不在已挂载会话的 ledger 里（时序/作用域），且官方渲染 `renderSlot("conversation.session.header.actions", {})` 只投影已声明的 occupants；官方 agent-preset 自己在会话初始化 effect 里注册（order:-10）才生效。
**正确姿势**：要"当前会话 + 全量会话列表"快照 → 用 **shell.overlay**（scope root，standardProps 带 `useSessions`，已证实第三方可渲染，avatar 就挂这）：`useSessions` 返回 `{ids, byId, current(活跃会话!), phase, ...}`（SessionListState，字段在 dsh-client-runtime/lib/types/client/sessions/service.d.ts）。需要"每会话一个按钮/列表"的通用入口优先 overlay，别往 header/hero 的作用域槽里塞（第三方晚注册不保证渲染）。
**佐证**：header.actions 注入 console 成功但 splitBtn false；改 overlay seat（id dcp-split-seat）后按钮渲染 + current 取值正确。

## 70. DSH 会话日志 = 多帧 zstd 追加写：node:zlib 的 zstdDecompressSync 只解首帧，必须 scanZstdFrames 切帧逐帧解（2026-08-27 分屏板数据源实战）
**现象**：`fs.readFileSync(session.jsonl.zstd)` 后 `zstdDecompressSync(buf)` 只解出 1 行（session header）；18MB 文件疑"只有 header"。`GET /api/session.export?sessionId=` 返回的是 **application/zip**（不是明文 jsonl，含 session.jsonl + media/ 目录）。~/.dsh/sessions 下 281 个会话全解出 1 行 → 误判"消息不落盘"。
**根因**：DSH 的 JSONL 后端是**每批独立 zstd frame 追加写**（官方 dsh-session-persistence-jsonl/zstd.js，ZSTD_MAGIC=0x28B52FFD LE，带 checksum 帧）；Node 内置 zstdDecompressSync 只处理单个完整帧 → 多帧输入只输出第一帧。
**正确姿势**：
- 自己实现 `scanZstdFrames(buffer)`（官方同款：magic+frame descriptor 解析 frame header/block/checksum 边界，逻辑在 dsh-session-persistence-jsonl/lib/index.js L503-566，约 60 行可抄）→ 得到帧 range 数组 → 每帧 `zstdDecompressSync(subarray)` 拼接；
- **性能**：尾部流式读取只解**最后 K 帧**（每帧一个追加批次，几 KB 级）+ 头部 8 帧取 session/title；别整文件解（18MB zstd → 数百 MB 明文）；
- 消息事件类型（渲染按此取字段）：`user/message`(data.content[].text)、`assistant/message`(data.message.content[].text+tool-use)、`tool/call`(data.name)、`step/end`、`turn/start`、`turn/end`(data.reason.kind)、`session/title`(data.title)；compaction/summary、*-chunks 流式块跳过；
- 会话文件定位：`~/.dsh/sessions/<ws-slug>/<sessionId>/session.jsonl.zstd`，**id 在目录名**（文件恒叫 session.jsonl.zstd）；空/导入会话可能只有 header 一行（messages=0 正常）。
**佐证**：scanZstdFrames+尾帧解码 → 52698 帧文件最后 2 帧解出 8 行（assistant/message/turn/end 等）；头 8 帧取到 title（"你把我规划下…"）；host `/cockpit/session-tail` 端点按此实现，单测通过。

## 71. build-client.mjs 只拼接不做语法检查：JavaScript 语法错误会"成功构建"→ 页面 Failed to load plugins（2026-08-27 工作台设置样式实战）
**现象**：改完 client.src.js 后 `node build-client.mjs` 输出 built 成功（无报错），但页面白屏显示 `Failed to load plugins ... bundle /plugins/dsh-cockpit/client.js loaded without registering "dsh-cockpit" via __ModuleLoader__.load`。
**根因**：build-client.mjs 只是按标记拼接内联源码字符串（__VOICE_ENGINE__ 等），**不经过任何语法分析**；语法错误（如对象字面量键 `avatar.enabled: true` 没带引号 → Unexpected token '.'）被原样带进产出 bundle，加载阶段直接抛错 → 插件注册失败。页面"Failed to load plugins"是一个 **cafa8df4 加载器桩**（不是插件本身报错）。
**正确姿势**：
- 每次 build 后（或 build 前）先 `node --check client/src 文件` 验证语法（ESM 文件 check 兼容）；报错行号直接指向源码；
- **对象字面量里带 `.` 的键必须引号**（`"avatar.enabled": true`），这是本次实测踩到的具体错误；
- 若页面出现 "loaded without registering ... via __ModuleLoader__.load" → 第一时间就是 bundle 语法/运行时错误（先 node --check，再看 console）。
**佐证**：node --check 报 `client.src.js:1606 SyntaxError: Unexpected token '.'`（`{ avatar.enabled: ... }[k]` 无引号键）；改为传入 DEFAULTS[k] 后语法 OK、重建成功、页面恢复。

## 72. 官方设置页样式对齐：Setting-Cell 行结构（figma 501:30011）抄 LanguageRow.module.css 即可，别自绘 inline style 方程组（2026-08-27 实战）
**现象**：自绘工作台设置页（inline style：fontSize 13、bg-layer-3、圆角 6、18px checkbox）与官方设置页（通用设置/模型）视觉割裂，用户报告"样式没有统一"。
**根因**：官方设置行是统一 **Setting-Cell**（figma 501:30011，典型实现 `@deepseek-ai/dsh-client-locale` 的 LanguageRow.module.css）。
**正确姿势**（实测对齐值）：
- 结构：`.row{border-bottom:1px solid var(--dsw-alias-border-l2);align-items:center;gap:8px;padding:16px 0;display:flex}` + `.rowText{flex-direction:column;flex:1;gap:4px;min-width:0;padding-right:48px;display:flex}`（左列 title+desc，右控件）；
- 控件：`.selector{background:var(--dsw-alias-bg-module-platform);height:36px;border-radius:18px;padding:0 14px;font-size:14px;color:var(--dsw-alias-label-primary);display:inline-flex;border:none}`（**胶囊 select/按钮**，hover 用 `--dsw-alias-interactive-bg-hover`）；开关为 28px 圆形（官方 iconButton toggle）；
- 值直接抄官方 CSS 变量（bg-module-platform 实测 rgb(16,22,38)），**不要**用 bg-layer-3/圆角 6 的自编参数；
- 页面标题 14-15px/500-600 primary + 副标题 tertiary 13px。
**佐证**：e2e 抓官方 LanguageRow computed style（selector 36px/18px/rgb(16,22,38)）→ 重写后 .dcp-ws-row 实测 rowPadY "14px 0px"、selector 36px/18px/rgb(16,22,38)、switch 28x28，截图观感与官方一致。

## 73. 主题插件三坑：① 插件别强制 setTheme ② overrideTokens 同 source 可运行时重注册（light/dark 双分支）③ alias 变量挂在 body（html 上 var() 解析不到会走 fallback）（2026-08-27 主题修复实战）
**现象①**：设置-外观 切"浅色/跟随系统"页面主区不变浅；刷新后永远回到深色。**根因**：插件 apply 里 `ctx.theme.setTheme("dark")` 每次都把官方持久化偏好（$DSH_HOME/settings.yaml 的 ui-theme.preference，官方 loopback 写盘）覆盖回深色 → 持久化"不生效"；且插件 overrideTokens 的 60+ 个 alias 全部 light/dark 都写深空暗值 → 浅色主题下 alias 仍被锁暗。
**现象②**（防环）：MutationObserver 监听 body[data-ds-dark-theme]+回调里无条件 overrideTokens(publish) → 官方 presenter 应用快照写 body attr（同值也写）→ 再触发 observer → publish → 循环/滞后。
**正确姿势**：
- **绝不 setTheme**（用户主题偏好是官方的，插件只管皮肤层）；overrideTokens 契约 = {light, dark} 双分支（requiresLightAndDark：BUILTIN_INSPECT_TOKENS 每项都要两值），light 分支引用官方 static 变量同源（`var(--dsw-static-neutral-bluish-00/#fff)`、label 浅色 `bluish-1000/700/600`、border `#0000001a`、tooltip `bluish-850`），dark 保留深空值；
- `ThemeRuntime.overrideTokens(source, tokens)` 是 **overrides Map set 语义**：同 source 重复调用=替换层并 publish → **theme 切换时按解析主题重注册即可**（订阅 ctx.theme.subscribe + **MutationObserver attributes 兜底**：回调只读 body attr，**只在 dark 值变化时 publish**，同值只更新 aurora 显隐 → 无循环）；
- alias 内联变量在 **body 元素**（官方 presenter 写 body）——`body{background:var(--dsw-alias-bg-base)}` OK，但 **html 上 var() 解析不到 → 走 fallback 深空**（html,body 双选择器写会锁死 body 背景）→ 只写 body；
- aurora 深空背景层显示条件收敛为"深色主题 && 用户开启背景层"（浅色强制 none）。
**佐证**：e2e：浅色 darkAttr:false/rootBg rgb(255,255,255)/aurora none；深色翻转；跟随系统模拟 dark/light 即时翻转；切浅色后 settings.yaml `ui-theme.preference: light`，**刷新后仍浅色**（修复前刷新必被拉回 dark）。

## 74. 官方 UI 选项/菜单项验证：必须真实鼠标事件（Input.dispatchMouseEvent 坐标），el.click() 不触发官方选择链→不写盘（2026-08-27 主题/外观实战）
**现象**：e2e 里 `document.querySelector(…).click()` 点"浅色"：DOM 立即生效（body attr 翻转）但 **settings.yaml 的 ui-theme.preference 永远是 dark** → 刷新后回到 dark，误判"官方持久化失效"。
**根因**：官方 AppearanceRow/菜单项的持久化写盘挂在**真实 pointer/mouse 事件链**（onClick 之外的选择/落点处理），合成 `.click()`（untrusted）只触发 React onClick 视觉，不触发写盘链。
**正确姿势**：e2e 验证官方 UI 交互一律 `getBoundingClientRect()` 取中心坐标 + `Input.dispatchMouseEvent(pressed/released)`；持久化验证读磁盘事实 `~/.dsh/settings.yaml` 的 ui-theme.preference（而非只看 DOM）；重载后读 body 属性 + 截图。真实点击后：settings.yaml=light + 刷新仍浅色（官方持久化本无 bug）。
**佐证**：同脚本 el.click()→yaml dark；改真实鼠标点击→yaml light + reload 后 darkAttr:false。

## 75. 浅色模式残留四件套（icon 白/handle 黑/字体白/黑框）的排查法与官方 handle 白化（2026-08-27 主题终审实战）
**现象**：浅色切换后用户逐细节验收发现：侧栏 icon 白色不可见、左侧栏 handle（`pI_x6G_handle` / `[data-side="sidebar"]`）中间黑色、正文白色、部分组件深色矩形残留——同一主题反复打回。
**排查法（一次到位，勿局部 patch）**：
1. **CSS 审计**：grep CSS 段裸硬编码色（`(?<![\w-])color:#e8f1ff`、`background:#0a0e1a` 之类）——正常态已全 var 化后逐元素 computed 审计才是真相：脚本在浅色下提取 icon svg color/handle bg+border/正文 color/弹层+根容器 bg/关键 alias var 实值（--dsw-alias-label-primary 应解析为 #0f1115 近黑、bg-base #fff）——**值对了就不是残留**；
2. **handle 白化**：官方 sidebar handle 浅色语义是**近黑实线**（border 用 label 系色），用户要求浅色=浅色线 → 加门控 `body:not([data-ds-dark-theme]) [class*="handle"]{background:var(--dsw-alias-border-l2)!important;border-color:…;box-shadow:none}` + `[class*="handle"] *{background:transparent;border-color:transparent}`（覆其内部描边元素）；
3. **变量链核对**：THEME_TOKENS 每键确认 LIGHT_TOKENS 有对应（漏一个就暗残留）；light 值一律 var(--dsw-static-*) 同源引用（官方 bluish-1000 #0f1115 等）；
4. **用户可能测旧 bundle**：/plugins/<id>/client.js 响应 `cache-control: no-cache` + src `?rev=`（rev 非文件 hash，会随构建变化）——**硬刷新即新版**；验收前先让用户 Ctrl+Shift+R 并核对 console bundle 版本。
**佐证**：审计 computed：浅色 icon rgb(97,102,107)/rgb(15,17,21)、handle 白化后 rgba(0,0,0,.1)、文本 #0f1115、dlg rgba(255,255,255,.86)、rootBg #fff——4 项全过；修复前 handle border 为 rgb(15,17,21)（近黑）。

## 76. 官方"繁忙时 Enter 键行为"= ui-conversation.busyEnter（queue/steer）——会话级覆盖的正确姿势（2026-08-27 实战）
**官方机制**：枚举 `BUSY_ENTER_BEHAVIORS=["queue","steer"]`（queue=排队发送 默认 / steer=插话发送），存 namespace `ui-conversation` 字段 busyEnter（持久于 ~/.dsh/settings.yaml）；仲裁=ComposerSubmissionPolicy.resolve(running, gesture, steeringAvailable)：**busy && steeringAvailable 时** plain Enter→preferred、chord(ctrl/cmd+Enter)→preferred 的反面；**steer 是 best-effort**（AgentLoop 会把窗口外提交转成下一次唤醒的 queue 项）；"Enter during the Host round-trip is a no-op"（#63）。
**会话级覆盖实现（插件侧，零侵入）**：
- UI：composer 权限按钮后 DOM 注入小胶囊 select（选项=跟随全局（默认）/排队发送/插话发送），样式复用 dcp-ws-selector 同款（alias 变量，双态自适应）；当前会话 id 经 shell.overlay 静默组件（useSessions.current → window.__dcpCurrentSession）共享给 DOM 侧；
- 持久化：per-session 用 localStorage（键 `dcp-busyEnter-v1` = {sid: mode}），**不新增 host schema**（避免改 host 必重启）；
- 行为：拦截 plain Enter（textarea keydown）当 busy && 会话级≠全局：**steer 覆盖=preventDefault+派发官方 chord(ctrl+Enter)**（官方原链 steer）；**queue 覆盖=临时 scope.set("busyEnter","queue")→派发 plain Enter→700ms 后写回原值**（净零，官方原链 queue）；
- 全局值**现读**（scope.getSnapshot() 每次拦截时取，subscribe 不可靠）；busy 信号复用 turnStatus 300ms tick 曝光 window.__dcpBusy。
**注意**：官方 busy Enter 的最终提交/排队现象受 steeringAvailable 语义窗口限制（很多 busy 时刻官方本身 no-op/转 queue）——**覆盖保证"用哪个模式"，最终行为=官方原链**；e2e 断言聚焦 intercept 探针（window.__dcpEnterOverride = {sid,mode,global,at}）而非"文本必清空"。
**佐证**：下拉 prevText="完全权限"（位置✓）；选择写 localStorage + reload 保持；busy+覆盖 → __dcpEnterOverride {mode:"queue",global:"steer"} 触发（steer 覆盖同理）；跟随全局（无会话级）→ 不拦截。

## 77. 注入 composer 工具栏的控件必须与官方 trigger 同构（透明无底框）；"用官方 alias 变量" ≠ 视觉统一（背景也是变量值）（2026-08-27 P0 实战）
**现象**：插入"完全权限"按钮旁的选择下拉用了 `background:var(--dsw-alias-bg-module-platform)` + 圆角 14 的胶囊——用户一眼指出"框架原生无底框，插件新增的非要有个底框"（明明并排）。
**根因**：官方权限按钮（Sh0Q9G_trigger 类）是**反直觉的透明**：`background:transparent; border:none; height:28px; font-size:13px; line-height:20px; padding:0 4px 0 8px; color:var(--dsw-alias-label-secondary); border-radius:24px; display:flex; gap:4px` ——**底色也是"透明度"而非颜色变量**；我用了"正式 selector 设计"（胶囊底）→ 与工具栏 flat 风格冲突。
**正确做法**：并排控件**先取对方 computed 逐值复刻**（写取证脚本：computed 对比 background/border/radius/height/font/padding/color/gap/display），同构后再提交；"样式用 alias 变量"只保证主题跟随，**不保证同构**（背景变量同样会画底）。
**佐证**：取证脚本输出 select vs perm：修复前 select bg=rgb(53,54,56)（有底）/perm transparent；修复后逐值一致（transparent/border none/24px/28px/13px/0 4px 0 8px/rgb(207,211,214)/gap 4/flex），双态截图（dcp-busyselect-dark/light.png）视觉并排无差异。

## 78. 脚本自动改码必须过语法验证闭环：插入脚本错找 `}` → 两函数嵌套错乱 → loader 53 entry 全挂（2026-08-29 dsh-cockpit 实战，此类事故历史上多次）
**现象**：用 insert-mountcontrol.mjs 脚本把 mountControlRoom（239 行）自动插入 client.src.js，build 后浏览器端**所有插件**报 "loaded without registering"（53 entry 全栈失败），控制室/avatar/mountPanels 全不挂载。
**排查弯路（勿重复）**：先疑 rev 缓存（实为内容 hash 一致，排除）→ 再疑环境问题（"53 entry 共性失败"实为**一个文件的语法错连累整个 loader**，不是多插件环境病）→ 又疑 V8 长源解析边界（V8 没有"文件长就解析失败"的病）。真凶定位靠 **acorn 二分定位 + `node --check`**：insert 脚本正则找 mountPanels 的闭合 `}` 时匹配到函数体内嵌套块的 `}`，mountControlRoom 整体插进了 mountPanels 体内 → 两函数嵌套错乱 → 源码本身语法非法（client.src.js 与产物同样报错，证明是源的错不是 build 的错）。
**正确姿势（四步验证链，强制）**：① 凡脚本/工具自动修改代码（插入/替换/生成/bundle），改完立即 `node --check` 或 acorn parse 验证语法，不过不许 build/部署；② 语法通过≠能用——影子启动验证（备用端口+独立 user-data-dir）确认 `__DSH_BOOT__` 含 entry、console 无 registering 报错，才许动生产；③ 生产重启前备份 patch+client（带时间戳）+ 写好 `disabled: true` 回退步骤；④ 排错方法论：报错位置不动=病灶在更前面（未闭合结构）；多 entry 同时失败=查公共 loader 入口而非逐个插件；node --check 比浏览器 console 精确；源文件与产物分开 check。
**佐证**：`node --check` 报 L2494 `Unexpected token 'function'`，改 const 后同位置报 `Unexpected token 'const'`（位置不动=病灶在前）；acorn 二分：注释 0..L1798→FAIL@2435、0..L1799→FAIL@2030（两函数体内都错位）。
**教训**：无验证的自动化比手工更危险——脚本改码必须语法验证闭环；"多插件共性失败"先怀疑最近改动的那一个文件，别急着归咎环境。

## 79. link 本地插件平台依赖未装 → boot 崩 ERR_MODULE_NOT_FOUND：插件目录必须自带精确宿主同版 node_modules（2026-09-05 dsh-wechat-devtools 实战）
**现象**：新装的 link 插件 dsh-wechat-devtools 让整个 `dsh --profile web` boot 崩：`failed to import loader entry wechat-devtools (dsh-wechat-devtools): Cannot find package '@deepseek-ai/dsh-tools' imported from D:\...\dsh-wechat-devtools\lib\index.js`（ERR_MODULE_NOT_FOUND）。
**根因**：`dsh plugin add <目录>`（link 方式）不装依赖（见反面清单）；插件 `lib/index.js` 里 `import { defineTool } from '@deepseek-ai/dsh-tools'`，而 ESM 解析从插件**真实路径**（`D:\job\developer\DSH\plugin\...`）向上找 node_modules，永远碰不到 profile（`C:\Users\...\.dsh\profiles\web`）与全局 dsh 的 node_modules——peer 语义"宿主提供"在 link 插件上不成立，插件目录必须自带实体。对照：cockpit/media-capture 等 link 插件能跑是因为 host 侧不 import 平台包（client 侧走 `__ModuleLoader__` 运行时注入）。
**解法**：插件目录 `pnpm add @deepseek-ai/dsh-tools@<宿主内置精确版本> --save-exact`。宿主版本查 `~/AppData/Roaming/npm/node_modules/@deepseek-ai/dsh/node_modules/@deepseek-ai/dsh-tools/package.json`（本次为 0.1.1-rc.1，npm 上有精确同版）。**不要**留 range 让 pnpm 自动解析——peer range `>=0.1.0-rc.6 <0.2.0` 会解析到 0.1.2-rc.1（比宿主新的平台 API，有漂移风险）。同版副本双实例无碍：registry 插件（agent-teams 等）本就是 pnpm auto-install-peers 的副本实例模式在跑。
**验证三连**：① 插件目录内 `node --input-type=module -e "import '@deepseek-ai/dsh-tools'"`（解析+导出可用）；② `node --check lib/index.js`（语法闸门，见 #78）；③ `dsh --profile web --dump-config` exit=0 且 entry 入树、`MODULE_NOT_FOUND|failed|duplicate|pending` 零命中。
**佐证**：修复前 boot 崩（ERR_MODULE_NOT_FOUND stack）；`pnpm add 0.1.1-rc.1` 后 resolved OK（defineTool: function）、syntax OK、dump-config exit=0 + `# == dsh-wechat-devtools / - id: wechat-devtools` 入树。
**教训**：link 插件首次装载就 boot 崩且报 `Cannot find package '@deepseek-ai/*'`，第一反应不是查 profile 注册，而是查插件目录有没有 node_modules；平台依赖的版本锚点是宿主内置版本，不是 npm latest。
