# DSH 插件发布流程（npm / Gitee / GitHub / dshmarket）

## 一、发布前检查（git-commit 三件套自检）

- [ ] README 达标（readme-author 标准：tagline/安装/快速开始/License 节）
- [ ] LICENSE 齐全且与 package.json `license` 字段一致（repo-license）
- [ ] 无敏感文件入暂存（.env/密钥/内网地址）
- [ ] 版本号已 bump（npm 同版本号不能重复 publish）

## 二、npm 发布

```sh
# 本机默认 registry 是 npmmirror，发布必须显式指定官方源
npm publish --registry=https://registry.npmjs.org/
# 验证
npm view <pkg> version --registry=https://registry.npmjs.org
```

- 2FA：需 **bypass-2FA 的 Granular Access Token**（classic automation token 已被 2026 策略拦截，见 pitfalls #5）；token 存 `~/.npmrc` 与 Obsidian 账号笔记。
- npm 自动打包 `LICENSE`/`README`/`package.json`；`files` 白名单控制其余。

## 三、Gitee + GitHub 双仓库

- **建仓**（repo-create 技能）：先评估内容 → 友好询问可见性 → API 创建（GitHub `private` 字段有效；**Gitee 建出来恒私有**，需网页改公开，见 pitfalls #8）。
- **凭据**：
  - Gitee：Windows 凭据管理器存储（`git credential fill` 可取的 access token）；SSH（git@gitee.com，本机 ed25519 免密）。
  - GitHub：SSH 被墙 → HTTPS + `~/.git-credentials` store helper（pitfalls #7）。
- **同步**：`git push origin main`（Gitee）+ `git push github main`（GitHub），双端 HEAD 回执一致。

## 四、dshmarket 上架（可选，让市场卡片显示中文描述）

市场描述来自 [awesome-dsh-plugin/awesome-dsh-plugin](https://github.com/awesome-dsh-plugin/awesome-dsh-plugin) 的 `data/plugins/<owner>__<repo>.yml`（不是 package.json）：

```yaml
url: https://github.com/<owner>/<repo>
name: <owner>/<repo>
category: ui            # ui/usage/tools/session/... 见官方列表
description:
  en: 'One-line English description ending with a period.'
  zh: '一句话中文描述，以句号结尾。'
```

**硬性要求**（CI 检查）：① `dsh.bundle` 声明（只有 dsh.client 会被拒）；② 仓库满 1 天；③ ≥10 commits；④ `dsh-plugin` GitHub topic；⑤ npm repository 指回同一仓库；⑥ 描述属实无营销词。

流程：fork → 加 YAML → `npm ci && node scripts/generate-readme.mjs` 重生成双 README → PR（通常一天内生效）。

## 五、发版清单（git-commit 技能）

1. bump 版本号（package.json）；
2. README/CHANGELOG 同步（readme-author / document-release）；
3. `npm publish --registry=npmjs` → `npm view` 确认；
4. 双仓库 push → `git ls-remote` 双端 HEAD 一致；
5. （可选）dshmarket 上架 PR。
