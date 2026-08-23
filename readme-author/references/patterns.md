# README 实证规律与范例拆解（taste library）

本文件是 `readme-author` 的"品味库"：从顶级开源项目 README 与 awesome-readme 收录的 100+ 优秀实例中提炼的规律。写 README 前先读这里建立直觉，而不是照抄模板。

## 一、实证对象

- **zustand**（pmndrs，17KB）：现代库 README 的标杆——短、有人格、示例驱动。
- **axios**（107KB）：大型库 README——TOC + Features + 浏览器兼容 + 分主题 API 参考。
- **sindresorhus/got**（24KB）：npm 包经典——Install 打头、Take a peek、Highlights。
- **awesome-readme**：收录 100+ 优秀实例并逐一点评"为什么好"。

## 二、高频出现的"制胜元素"（来自 100+ 实例的统计直觉）

按出现频率排序，越靠前越是基本盘：

1. **项目 logo / banner（hero）**——几乎必备；居中对齐，宽度可控。
2. **徽章（badges）**——版本、CI、下载量、覆盖率、License、平台；同一套样式。
3. **一句话清晰描述（tagline）**——"它是什么"，友好到新手也能懂。
4. **TOC（目录）**——中长篇必备，方便跳转；短篇不要。
5. **演示（GIF/截图）**——UI/CLI 项目必备；"一图胜千言"。
6. **安装指引**——step-by-step、可复制粘贴；多环境用可折叠块。
7. **使用示例（代码块）**——先于概念解释；展示真实输出。
8. **Features（利益导向要点）**——讲"能做什么/解决什么"，不是"技术栈清单"。
9. **外部链接**——website / docs / demo / 社区（Discord、讨论）。
10. **FAQ**——把高频疑问收在 README 里，而不是让人去翻 issue。
11. **"为什么做这个 / 为什么不用 X"（philosophy/motivation）**——有观点，是加分项。
12. **贡献指南 + License**——开源仓库的基本盘收尾。

## 三、范例拆解：每个标杆做对了什么

### zustand（小库的正确姿势）
- 首屏顺序：logo → 一排版式统一的徽章（flat，黑底黑标）→ 一句 tagline（"small, fast and scalable bearbones state-management…"，**类别+差异**一次说清）→ demo/docs 链接 → `npm install zustand`。
- **人格化文案**：bearbones、cute bear jokes——语气即品牌。
- **节标题 = 读者任务**："First create a store""Then bind your components, and that's it!"——不是"Introduction"。
- **Recipes 模式**：每个 Recipe 一个任务名标题 + 5–10 行可运行示例 + 一句解释；不堆砌。
- **警告用 callout**（`:warning:`），把易错点标在示例旁边。
- 结尾：TypeScript 用法、Best practices（外链 docs）、对比（Why zustand over redux/context）。
- 没有 TOC、没有 Features 列表——因为它够短，读者不需要。

### axios（大库的正确姿势）
- TOC 居首（长文档导航刚需）。
- **Features 是清单式**，一目了然。
- **Browser support 表格**——兼容性是它的卖点，值得一张表。
- Installing：包管理器 + CDN 双路。
- **Example 紧跟安装**——先跑起来，再进 API 参考。
- API 参考按主题分节（Request config / Interceptors / Errors / Cancellation…），每节配示例。
- 结论：大库 README 是"索引 + 高频用法"，低频细节也要在 README 内给足（因为它没有 docs 站）。

### got（npm 包经典套路）
- **Install 第一节**（sindresorhus 流派铁律：先让读者能用）。
- **Take a peek**：一个最小示例立刻展示"它长什么样"。
- **Highlights**：利益导向要点（原生 promise、超时、重试、流式…）。
- 文档/迁移指南外链；Comparison；Maintainers；"这些公司正在用"。
- 结论：npm 包的"快速上手优先"范式。

## 四、逐节决策（section-by-section）

| 元素 | 何时用 | 何时不用 |
|---|---|---|
| logo/banner | 几乎总是；没有 logo 就用风格化标题排版 | 别放低清/拉伸图 |
| 徽章 | 有真实数据就放（版本/CI/下载/license）；统一样式 | 无 CI 别放 CI 徽章；不堆 10 个 |
| tagline | 总是，一句，类别+差异 | 别写两句以上；别用营销腔 |
| 外部链接 | demo/docs/社区有意义时 | 别放一堆死链 |
| 安装 | 总是，可复制 | 别只写"见文档" |
| 最小示例 | 总是（除纯文档站） | 别放半成品代码 |
| TOC | >300 行或 8+ 节 | <50 行 |
| Features | 功能多/有对比价值时；benefit 导向 | 3 个以内功能别列清单 |
| 截图/GIF | UI/CLI/可视化 | 无真实截图就跳过，不放占位 |
| FAQ | 有真实高频问题 | 没有问题别硬编 |
| 哲学/动机 | 有观点时加分 | 没有就删 |
| 贡献指南 | 开源、期待贡献 | 内部/一次性工具 |
| License | 开源必备 | 内部工具可省 |

## 五、文案质感规则

- **一句 tagline 公式**：`[类别] + [唯一差异]`。反例："一个强大的工具"；正例（zustand 式）："small, fast and scalable … solution using simplified flux principles"。
- **动词开头**：节标题/要点用读者动作（Install、Create、Fetch），不用名词堆砌。
- **删空话**：强大的/无缝的/革命性的/赋能/一站式/极致 —— 全部删掉，用事实替代（"2KB、零依赖、类型安全"）。
- **给数字**：体积、性能、依赖数、支持版本——具体数字比形容词可信。
- **示例说话**：一个可运行示例 > 三句概念描述。
- **语气匹配读者**：库=精确+适度幽默；工具=利益+行动；内部=朴素。

## 六、i18n 惯例（中英双 README）

- 主 README 顶部放语言切换：`[English](README.md) | [中文](README.zh-CN.md)`。
- 结构同步；术语一致（术语表统一后再翻译）；链接指向各自语言版本。
- 翻译要"本地化"而非逐字直译；中文版可以用中文社区习惯的说法。
