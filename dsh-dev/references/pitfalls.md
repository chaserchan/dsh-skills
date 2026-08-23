# DSH 插件开发踩坑库（实战记录，接手勿重复）

> 每条都是本机实测踩过的坑，含根因与解法。新坑请追加到这里并同步进 dsh-skills 仓库。

## 1. `file:` 本地依赖 + `dsh.client` 双面组合 → cordis auto-disable（高危，典型案例）

**现象**：`dsh-media-capture` 的 client 端 `/plugins/<id>/client.js` 恒 404、`__DSH_BOOT__` 没有它；host 能被加载（`--dump-config` 有 entry）。

**诊断证据**（cordis 运行时注入）：
```
global-prompt:   fiber=true disabled=false（稳定 enabled，进 graph）✅
media-capture:   首次 fiber=true disabled=false（进 graph）→ 之后 disabled=true（被 auto-disable）❌
```
根因：cordis `internal/plugin` 机制——`fiber.parent.fiber?.entry !== fiber.entry`（`parentMatch=false`）时判定"脱管"，标记 `disabled` 并移出 graph。

**已试 9 项全部无效**（勿重复）：id≠name、host 加/去 inject、`ctx.inject`、去 `ctx.logger`、`immediately:true`、`file:` tarball、手动标准包、移出 bundles、patch id≠name。

**对策**：不要逐字段碰运气。**整体复制已知能进 graph 的模板**（`dsh-plugin-global-prompt` 工程形态）再替换功能；或走 registry 发布形态。

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
