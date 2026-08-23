<div align="center">

# dsh-skills

一套为 AI Agent 工作流打磨的自研技能集，聚焦**仓库与文档的"门面工程"**——建仓、许可证、README，一条龙。

[![GitHub](https://img.shields.io/badge/GitHub-chaserchan%2Fdsh--skills-181717?style=flat&logo=github&logoColor=ffffff&colorA=000000&colorB=000000)](https://github.com/chaserchan/dsh-skills)
[![Gitee](https://img.shields.io/badge/Gitee-chasechan%2Fdsh--skills-21b7a3?style=flat&colorA=000000&colorB=000000)](https://gitee.com/chasechan/dsh-skills)

</div>

三个技能相互配合，构成**建仓三件套**：先问权限（`repo-create`）→ 补许可证（`repo-license`）→ 写 README（`readme-author`）。全部经过真实项目验证（dsh-plugin-global-prompt 插件从建仓到上架的完整流程）。

## 技能清单

| 技能 | 功能 | 何时触发 |
|---|---|---|
| [`readme-author`](readme-author/SKILL.md) | 编写/重写高质量 README（门面即漏斗：首屏法则、四步法、删段测试、QA 清单） | 写 README、"README 不高級/帮我改改/参考大项目怎么写" |
| [`repo-license`](repo-license/SKILL.md) | 为仓库选对开源许可证并生成正确 LICENSE（决策树 + 官方全文 + 版权人规范） | 建仓缺 LICENSE、"该用什么协议"、"MIT 还是 GPL" |
| [`repo-create`](repo-create/SKILL.md) | 建仓前按内容评估可见性并友好询问，创建后复核权限 + 落盘三件套自查 | 创建仓库/建仓/"公开还是私有"/"推到远端" |

## 安装

把需要的技能目录复制到技能目录（如 `~/.agents/skills/` 或 DSH 技能目录）：

```sh
cp -r readme-author repo-license repo-create ~/.agents/skills/
```

技能即装即用，重启会话后由技能目录自动发现。

## 目录结构

```
dsh-skills/
├── readme-author/     # README 门面工程（SKILL.md + references + evals）
├── repo-license/      # 开源许可证落地（SKILL.md + references）
├── repo-create/       # 建仓规范：权限先问（SKILL.md + references + evals）
├── LICENSE            # MIT
└── README.md
```

## 维护

- 本机激活副本在 `~/.agents/skills/`；本仓库是**版本化源**，改技能请先改这里再同步到激活目录（反之亦然，保持两边一致）；
- 新增技能：在仓库根新建 `<skill-name>/SKILL.md`（+ references/evals），README 清单同步加一行。

## License

[MIT](LICENSE) © 2026 逐鹿科技
