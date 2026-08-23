# DSH 插件开发踩坑库（实战记录，接手勿重复）

> **维护（跨会话）**：任何会话在本机做 DSH 开发时，遇到新坑/新经验，按 `dsh-dev/SKILL.md` 的"经验沉淀契约"**追加到本文件**（已有条目勿删改，只追加），并同步到版本化源 `D:\job\developer\DSH\skills\dsh-dev\references\pitfalls.md` 后提交推送 dsh-skills 仓库（Gitee + GitHub）。动手前先通读本文件，已记录的坑不要重复试。

> 每条都是本机实测踩过的坑，含根因与解法。新坑请追加到这里并同步进 dsh-skills 仓库。

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
