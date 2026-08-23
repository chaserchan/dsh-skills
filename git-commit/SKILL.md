---
name: git-commit
description: >-
  对 git 仓库执行规范的提交/推送/发版（提交、commit、push、推送、发版、同步远端、把改动提交上去、发布新版本）。
  适用于任何对已有 git 仓库做提交/推送的操作：提交前自动做"建仓三件套"自检（README 是否达标、LICENSE 是否齐全、
  是否有敏感文件/密钥混入暂存区），按 Conventional Commits 写提交信息，推送到全部已配置远端（如 Gitee + GitHub）
  并回执确认。也适用于"发版"（版本号 + npm 发布 + 双仓库同步）。凡是用户说"提交 git / commit / push / 推到远端 /
  发个版本"，都应使用本技能——即使仓库很小或改动很小，提交前的自检与规范信息也不可省。与 repo-create（建仓）、
  repo-license（许可证）、readme-author（README）配合：自检发现问题时调用对应技能补齐后再提交。
---

# Git Commit（提交规范）

## 身份与使命

把"提交 git"从随手操作变成**有纪律的交付**：提交前自检、提交信息有语义、推送全远端、发版有回执。你的工作不是机械执行 `git commit -m "update"`，而是每次提交都像"对外交付"一样经得起看。

## 工作流

### 第 1 步：提交前自检（先于一切 git 操作）

1. **敏感文件拦截**：暂存区/工作区若有 `.env`、`*.pem`、`id_rsa`、密钥格式串（`gho_`/`npm_`/`sk-`）、内网地址、客户数据 → **停止提交**，提示处理（加入 `.gitignore` 或移除），绝不带着敏感内容提交；
2. **README 达标**（readme-author 标准）：改动涉及对外可见变更（新功能/改名/发版）时，README 缺失或不达标 → 用 readme-author 补齐后再提交；
3. **LICENSE 齐全**（repo-license 标准）：开源仓库缺 LICENSE → 用 repo-license 补齐（默认按 package.json 的 `license` 字段）；
4. **暂存范围明确**：只 add 本次改动相关文件，不无脑 `git add -A` 卷入无关文件/构建产物/node_modules；
5. **改动核对**：`git status` / `git diff --stat` 与用户意图一致，无遗漏无多余。

### 第 2 步：写规范提交信息（Conventional Commits）

`<type>(<scope>): <摘要>`，摘要用祈使句、说明"做了什么"，不带空洞词。

- `feat` 新功能 · `fix` 修复 · `docs` 文档 · `chore` 杂务 · `refactor` 重构 · `test` 测试 · `perf` 性能
- 示例：`feat(composer): add camera capture button next to the plus action`

### 第 3 步：推送 + 回执

- 推送到**全部已配置远端**（本机惯例：`origin`=Gitee、`github`=GitHub），逐条确认 exit 0；
- 回执确认：`git log --oneline -1` + 双远端 `ls-remote` HEAD 一致；
- 涉及**发版**：bump 版本号 → `npm publish --registry=https://registry.npmjs.org/`（显式官方源；2FA 用 bypass 令牌）→ 双仓库 push → `npm view` 确认。

## 决策规则

- **有敏感文件必停**：任何密钥/内网/客户数据，宁可停下来问，不可带着提交（这是唯一"无条件拦截"项）。
- 提交信息空洞（"update"/"fix bug"/"修改"）→ 重写，直到能让人 30 秒看懂这次改动。
- 多远端：本机惯例是 Gitee + GitHub 双同步，漏推一端等于没交付，必须逐端回执。
- `--force` / 改写历史：**必须用户明确同意**才做，并说明风险。
- 提交前自检发现问题：先调用 repo-license / readme-author 补齐，再继续提交（补丁并入同一提交或单独提交均可，说明清楚即可）。

## 边界

- 不替用户决定"要不要提交哪些文件"——暂存范围以用户意图为准，拿不准先列出来问；
- 不擅自 `force push`、不清理他人分支、不改写共享历史；
- 不发版除非用户明确要求（发版涉及版本号与 npm，属于对外动作）；
- 技能负责提交/推送/发版的**纪律与自检**，不是 git 的说明书（常规 git 操作直接执行即可，不用解释每条命令）。

## 反面清单（Anti-patterns）

- 带着 `.env`/密钥/内网地址提交（最高频事故）。
- 无脑 `git add -A` 把 node_modules/构建产物/无关文件一起提交。
- 空洞提交信息：`update`、`fix`、`修改`、`提交一下`。
- 只推一个远端（比如只推 Gitee 忘 GitHub），不确认另一端。
- push 失败不追查（网络/凭据问题丢给用户）。
- 发版不 bump 版本号就 publish（npm 会拒绝重复版本，或覆盖 tag）。

## 复查循环（提交完成后）

1. `git status` 干净？暂存范围对？
2. 提交信息是 Conventional 格式、能看懂？
3. 所有远端都推了、HEAD 一致（ls-remote）？
4. 涉及发版：npm 版本确认、双仓库同步确认？
5. 敏感文件确认没进历史（万一进了，提示用户处理并给出清理建议）？

## 配合技能

- 自检发现缺 LICENSE → `repo-license`；README 不达标 → `readme-author`；要建新仓库 → `repo-create`（本技能只管已有仓库的提交/推送/发版）。
