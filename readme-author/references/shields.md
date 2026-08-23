# 徽章（shields.io）用法

## 基本规则

- **只放有信息量的徽章**：版本、CI、覆盖率、下载量、License、平台兼容。没有数据源就没有徽章。
- **统一样式**：同一行徽章用一致的 `style=flat` 与 colorA/colorB，视觉才"贵"。
- 一行放 4–7 个封顶；超过就是徽章墙（anti-pattern）。

## 常用模板

```markdown
<!-- npm 版本（包名需 URL 编码） -->
[![npm](https://img.shields.io/npm/v/<pkg>?style=flat&colorA=000000&colorB=000000)](https://www.npmjs.com/package/<pkg>)

<!-- npm 下载量 -->
[![Downloads](https://img.shields.io/npm/dt/<pkg>?style=flat&colorA=000000&colorB=000000)](https://www.npmjs.com/package/<pkg>)

<!-- GitHub Actions CI（owner/repo） -->
[![CI](https://img.shields.io/github/actions/workflow/status/<owner>/<repo>/<workflow.yml>?branch=main&style=flat&colorA=000000&colorB=000000)](https://github.com/<owner>/<repo>/actions)

<!-- 覆盖率（以 codecov 为例） -->
[![Coverage](https://img.shields.io/codecov/c/github/<owner>/<repo>?style=flat&colorA=000000&colorB=000000)](https://codecov.io/gh/<owner>/<repo>)

<!-- License -->
[![License](https://img.shields.io/github/license/<owner>/<repo>?style=flat&colorA=000000&colorB=000000)](LICENSE)

<!-- 平台/静态 label -->
[![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20macOS%20%7C%20Linux-000000?style=flat&colorA=000000&colorB=000000)](https://github.com/<owner>/<repo>)

<!-- Gitee 链接徽章（无官方 data 源时用静态 label） -->
[![Gitee](https://img.shields.io/badge/Gitee-<owner>%2F<repo>-21b7a3?style=flat&colorA=000000&colorB=000000)](https://gitee.com/<owner>/<repo>)

<!-- GitHub 链接徽章 -->
[![GitHub](https://img.shields.io/badge/GitHub-<owner>%2F<repo>-181717?style=flat&logo=github&colorA=000000&colorB=000000)](https://github.com/<owner>/<repo>)
```

## 注意

- 徽章 URL 中的空格要编码为 `%20`（或 `_`），`/` 编码为 `%2F`。
- 徽章必须可点击，指向真实页面；别放裸图（无链接的徽章是装饰垃圾）。
- 中文 label 可用，但静态 label 需 URL 编码；保持与 README 语言一致。
- license 徽章指向仓库内 `LICENSE` 文件（确保文件真的存在——配合 repo-license 技能）。
