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
