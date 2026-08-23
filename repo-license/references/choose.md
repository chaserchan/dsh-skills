# 许可证选型决策与模板

## 决策树（先答三问：项目性质 / 版权人 / 协议倾向）

```
项目是公司 IP 或商用、专利敏感？
├─ 是 → 咨询法务；倾向 Apache-2.0（有专利授权条款）
└─ 否（个人/团队开源）→ 下一个问题

你希望衍生作品必须保持开源吗（copyleft）？
├─ 是，且是桌面/本地软件 → GPL-3.0
├─ 是，且是网络服务（云端也要传染）→ AGPL-3.0
├─ 只约束修改过的文件（组件库折中）→ MPL-2.0
└─ 无所谓 / 希望最大采用 → 下一个问题

项目类型与生态惯例？
├─ npm/前端库、插件、工具（默认）→ MIT（生态最常见，一行版权声明）
├─ 需要显式专利授权 / 大厂规范 → Apache-2.0
├─ 学术机构或公司偏好 BSD 措辞 → BSD-3-Clause
├─ 想完全放弃权利（纯内容/示例）→ Unlicense 或 CC0（慎用）
└─ 拿不准 → MIT，并说明可随时更换
```

## 各协议一句话说明（用于向用户解释选型理由）

| 协议 | 一句话 | 典型场景 |
|---|---|---|
| **MIT** | 最宽松：任何人可用/改/商用，只需保留版权声明 | 库、插件、工具（默认） |
| **Apache-2.0** | 宽松 + 显式专利授权条款，法律上更严谨 | 大厂项目、有专利顾虑 |
| **BSD-3-Clause** | 与 MIT 等价宽松，措辞传统（禁止用作者名义背书） | 学术、公司 |
| **GPL-3.0** | 强传染：衍生作品必须开源同协议 | 要求生态保持开源 |
| **AGPL-3.0** | GPL + 网络服务也算"分发"，云端也传染 | SaaS、服务器软件 |
| **MPL-2.0** | 文件级传染：改过的文件才开源，其余可闭源 | 组件库折中 |
| **Unlicense/CC0** | 放弃版权，公共领域 | 纯内容、示例、个人极简 |

## LICENSE 文件写法要点

- 文件名：`LICENSE`（GitHub/Gitee 自动识别）；也可 `LICENSE.md`，但 README 链接要对齐。
- 版权行格式：`Copyright (c) <年份> <版权人>`。版权人 = 真实法律主体（个人真实姓名或公司法人名；用 git config 的 user.name 时先确认它是个人名而非昵称）。
- 多版权人：`Copyright (c) 2026 A\nCopyright (c) 2026-2027 B`（续行）。
- 年份：新仓库用创建年份；MIT 惯例无需逐年更新，重大重构可更新范围。
- 全文必须来自官方（opensource.org / choosealicense.com / SPDX），一字不改，只替换版权行。

## 官方全文来源

- MIT：https://opensource.org/license/mit · https://choosealicense.com/licenses/mit/
- Apache-2.0：https://www.apache.org/licenses/LICENSE-2.0.txt
- GPL-3.0：https://www.gnu.org/licenses/gpl-3.0.txt
- AGPL-3.0：https://www.gnu.org/licenses/agpl-3.0.txt
- BSD-3-Clause：https://opensource.org/license/bsd-3-clause
- MPL-2.0：https://www.mozilla.org/en-US/MPL/2.0/
- Unlicense：https://unlicense.org/
- SPDX 标识符总表：https://spdx.org/licenses

## MIT 全文模板（可直接替换版权行使用）

```text
MIT License

Copyright (c) <年份> <版权人>

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## 集成清单

- [ ] LICENSE 文件（官方全文 + 正确版权行）
- [ ] README `## License` 节 + 链接
- [ ] package.json `"license": "SPDX标识"`（或各语言清单字段）
- [ ] GitHub/Gitee 仓库 About → License 选择器（可选）
- [ ] 第三方依赖协议兼容性提醒（copyleft 依赖会传染你的分发）
