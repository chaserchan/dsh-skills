<div align="center">

# dsh-skills

一套为 AI Agent 工作流打磨的自研技能集，聚焦**仓库与文档的"门面工程"**——建仓、许可证、README，一条龙。

[![GitHub](https://img.shields.io/badge/GitHub-chaserchan%2Fdsh--skills-181717?style=flat&logo=github&logoColor=ffffff&colorA=000000&colorB=000000)](https://github.com/chaserchan/dsh-skills)
[![Gitee](https://img.shields.io/badge/Gitee-chasechan%2Fdsh--skills-21b7a3?style=flat&colorA=000000&colorB=000000)](https://gitee.com/chasechan/dsh-skills)

</div>

五个技能覆盖**仓库生命周期与 DSH 开发**：建仓（`repo-create`）→ 补许可证（`repo-license`）→ 写 README（`readme-author`），日常**提交/推送/发版**由 `git-commit` 把关，**DSH 插件开发**由 `dsh-dev` 全流程护航。全部经过真实项目验证（dsh-plugin-global-prompt 插件从建仓到上架的完整流程）。

## 技能清单

| 技能 | 功能 | 何时触发 |
|---|---|---|
| [`dsh-dev`](dsh-dev/SKILL.md) | DSH 开发与插件开发：双半区工程形态、装载/验证、发布、踩坑规避（官方架构 + 实战经验） | "做个 DSH 插件 / 插件不显示 / /plugins 404 / 加设置项 / 注入系统提示词" |
| [`git-commit`](git-commit/SKILL.md) | 提交/推送/发版规范：提交前三件套自检（README/License/敏感文件）+ Conventional Commits + 双远端回执 | "提交 git / commit / push / 推到远端 / 发个版本" |
| [`readme-author`](readme-author/SKILL.md) | 编写/重写高质量 README（门面即漏斗：首屏法则、四步法、删段测试、QA 清单） | 写 README、"README 不高級/帮我改改/参考大项目怎么写" |
| [`repo-license`](repo-license/SKILL.md) | 为仓库选对开源许可证并生成正确 LICENSE（决策树 + 官方全文 + 版权人规范） | 建仓缺 LICENSE、"该用什么协议"、"MIT 还是 GPL" |
| [`repo-create`](repo-create/SKILL.md) | 建仓前按内容评估可见性并友好询问，创建后复核权限 + 落盘三件套自查 | 创建仓库/建仓/"公开还是私有"/"推到远端" |
| [`obsidian-bases`](obsidian-bases/SKILL.md) | 创建/编辑 Obsidian Bases（.base 文件：视图、过滤器、公式、汇总） | "Obsidian 的 .base 文件 / 数据库视图 / 表格视图 / filters / formulas" |
| [`obsidian-cli`](obsidian-cli/SKILL.md) | 通过 Obsidian CLI 操作 vault（读写笔记、搜索、任务、属性），支持插件/主题开发调试 | "操作 Obsidian vault / 命令行管理笔记 / 开发调试 Obsidian 插件" |
| [`obsidian-markdown`](obsidian-markdown/SKILL.md) | 创建/编辑 Obsidian Flavored Markdown（wikilink、embed、callout、properties） | "写 Obsidian 笔记 / wikilinks / callouts / frontmatter / tags" |

## 安装

把需要的技能目录复制到技能目录（如 `~/.agents/skills/` 或 DSH 技能目录）：

```sh
cp -r dsh-dev git-commit readme-author repo-license repo-create obsidian-bases obsidian-cli obsidian-markdown ~/.agents/skills/
```

技能即装即用，重启会话后由技能目录自动发现。

## 目录结构

```
dsh-skills/
├── dsh-dev/           # DSH 开发与插件开发（SKILL.md + references + evals）
├── git-commit/        # 提交/推送/发版规范（SKILL.md）
├── readme-author/     # README 门面工程（SKILL.md + references + evals）
├── repo-license/      # 开源许可证落地（SKILL.md + references）
├── repo-create/       # 建仓规范：权限先问（SKILL.md + references + evals）
├── obsidian-bases/    # Obsidian Bases（SKILL.md + references）
├── obsidian-cli/      # Obsidian CLI（SKILL.md）
├── obsidian-markdown/ # Obsidian Flavored Markdown（SKILL.md + references）
├── LICENSE            # MIT
└── README.md
```

## 维护

- 本机激活副本在 `~/.agents/skills/`；本仓库是**版本化源**，改技能请先改这里再同步到激活目录（反之亦然，保持两边一致）；
- 新增技能：在仓库根新建 `<skill-name>/SKILL.md`（+ references/evals），README 清单同步加一行。

## License

[MIT](LICENSE) © 2026 逐鹿科技
