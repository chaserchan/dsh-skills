---
name: repo-license
description: >-
  为新建/已有 git 仓库选择并落地合适的开源许可证（LICENSE、许可证、开源协议、license、MIT、Apache、
  GPL、BSD、版权声明）。适用于：用户创建/初始化 git 仓库（git init、建 GitHub/Gitee 仓库）、说"仓库没有
  LICENSE"、"帮我加个开源许可证"、"这个项目该用什么协议"、"MIT 还是 GPL"、发布开源包/插件前补许可证。
  也适用于 README 里要写 License 章节、package.json 的 license 字段、版权人(©)声明。凡是涉及"仓库该用哪个
  开源协议、LICENSE 文件怎么写、版权归属怎么写"，都应使用本技能——即使仓库已经建好但缺 LICENSE，也要主动
  补上（缺 LICENSE 默认"保留所有权利"，是仓库创建的高频缺陷）。
---

# Repo License（开源许可证落地）

## 身份与使命

每次创建或发布 git 仓库时，确保有一个**合适且正确落地**的开源许可证。缺 LICENSE 的仓库在法律上默认"保留所有权利"——别人不能合法使用你的代码；选错协议或写错版权人，比没有更麻烦。你的工作：选型 → 生成 → 集成 → 自检。

本技能是流程 + 判断型技能，不是法律意见：拿不准时（公司 IP、多贡献者、商用产品）明确建议咨询专业人士或 [choosealicense.com](https://choosealicense.com)。

## 工作流

### 第 1 步：选型（先问清三个问题）

动笔前必须明确：

1. **项目性质**：个人作品 / 公司项目 / 教学示例 / 库 vs 应用？
2. **版权人**：个人（姓名/昵称）还是公司（法人名）？—— LICENSE 第一行 `Copyright (c) <年> <版权人>` 必须是**真实法律主体**，不允许留占位。
3. **协议倾向**：宽松（permissive）还是传染（copyleft）？是否有专利顾虑？

用 [references/choose.md](references/choose.md) 的决策树选型，并在回复里**一句话说明为什么选它**（不要静默决定）。

### 第 2 步：生成 LICENSE 文件

- 从**官方全文**复制（opensource.org / choosealicense.com / SPDX），不允许简写、节选、自己改写。
- 替换版权行：`Copyright (c) <当前年份> <版权人>`；多版权人用多行。
- 文件名：`LICENSE`（或 `LICENSE.md`；与 README 链接一致）。
- 常用协议全文模板见 [references/choose.md](references/choose.md)（MIT/Apache-2.0/BSD-3/GPL-3.0 等）。

### 第 3 步：集成（License 不是孤立文件）

- **README**：加 `## License` 节，`[MIT](LICENSE)` 链接（配合 readme-author 技能）。
- **package.json**：`"license": "MIT"`（SPDX 标识符）；Python 用 `license` 字段、Rust 用 Cargo.toml `license`。
- 平台设置（如可行）：GitHub/Gitee 仓库 About 页 License 选择器。
- 提交并推送（如果仓库已存在远程）。

### 第 4 步：自检

- [ ] LICENSE 文件存在且为官方全文
- [ ] 版权行：年份正确、版权人是真实主体、无占位符
- [ ] README License 节 + 链接
- [ ] package.json / 清单文件 license 字段一致（SPDX）
- [ ] 选型理由能一句话说清

## 决策规则（选型速查）

| 场景 | 推荐 | 理由 |
|---|---|---|
| 个人库/插件/工具（默认） | **MIT** | 最宽松、生态最常见、一行版权声明即可 |
| 大厂偏好/需要明确专利授权 | **Apache-2.0** | 显式专利授权条款，公司法律部通常认可 |
| 要求衍生作品保持开源 | **GPL-3.0** | 强传染；库用 GPL 会劝退商业用户 |
| 网络服务想约束服务端 | **AGPL-3.0** | 云端部署也受 copyleft 约束 |
| 学术/公司偏好 BSD 措辞 | **BSD-3-Clause** | 与 MIT 等价宽松，措辞传统 |
| 文件级传染的折中 | **MPL-2.0** | 修改文件才传染，适合组件库 |
| 放弃权利/纯内容 | **Unlicense / CC0** | 完全放弃版权；慎用（含商标/署名顾虑） |

不确定时的默认动作：**MIT**（个人项目），并说明可随时换。

## 边界

- 不提供法律意见；公司 IP、多人贡献、商用/专利敏感 → 建议咨询法务。
- 不静默决定协议：至少一句话向用户说明选型理由；版权人拿不准就问。
- 不改写官方文本；不创建"自创协议"。
- 不替用户决定"要不要开源"——内部仓库可以不放开源 LICENSE（但应明确告知其含义）。
- 若仓库有第三方代码/依赖，提醒核对是否与所选协议兼容。

## 反面清单（Anti-patterns）

- **无 LICENSE**：默认保留所有权利——创建仓库必须补。
- **占位版权行**：`Copyright (c) [year] [name]` 原样提交。
- **节选/改写协议文本**：License 必须全文，一个字不改。
- **选型无理由**：随手贴个 GPL 或 Apache，不问项目性质。
- **LICENSE 与 README/package.json 不一致**：三处必须同协议。
- **版权人写错**：把 GitHub 用户名当公司主体、把 AI 当版权人。
- **年份固化**：新仓库用创建年份；大版本重构可更新（MIT 惯例无需逐年改）。

## 复查循环

1. 选型理由说得清吗？（一句话）
2. LICENSE 是官方全文 + 正确版权行吗？
3. README / package.json / 平台三处一致吗？
4. 有第三方代码的协议兼容性提醒过吗？

## 参考资料

- [references/choose.md](references/choose.md) —— 决策树、各协议一句话说明、常用协议官方全文模板
- 权威来源：https://choosealicense.com · https://opensource.org/licenses · https://spdx.org/licenses
