# DSH 插件工程模板（已验证形态）

> 本机唯一"本地源码 + 稳定进 graph"的 client 插件是 `dsh-plugin-global-prompt`（v0.1.3，npm/Gitee/GitHub 已发布）。
> 新插件开发：**整体复制它的工程形态再替换功能**，不要逐字段发明。

## 一、package.json（完整形态）

```json
{
  "name": "dsh-plugin-xxx",
  "version": "0.1.0",
  "type": "module",
  "main": "lib/index.js",
  "exports": {
    ".": "./lib/index.js",
    "./client": "./lib/client.js",
    "./package.json": "./package.json"
  },
  "files": ["lib", "cordis.patch.yml", "README.md"],
  "license": "MIT",
  "dsh": {
    "bundle": { "patch": "./cordis.patch.yml" },
    "client": {
      "inject": [
        "@deepseek-ai/dsh-client-connection",
        "@deepseek-ai/dsh-client-runtime",
        "@deepseek-ai/dsh-client-ui-settings",
        "@deepseek-ai/dsh-api-remotes"
      ],
      "platform": "web",
      "immediately": true
    }
  },
  "dependencies": { "@deepseek-ai/schemastery": "^3.18.1" }
}
```

要点：`exports["./client"]` 必填；`dsh.client.inject` 用官方包列表（保证浏览器基线依赖就绪）；**`dsh.bundle` 必填**（可安装/可上架）；`files` 白名单含 client 与 patch。

## 二、host 半侧（lib/index.js）

```js
import z from "@deepseek-ai/schemastery";

// settings namespace：小写 kebab（^[a-z][a-z0-9-]*$），host/client 两侧同名
const NS = "my-plugin";

export function apply(ctx) {
  ctx.inject(["settings", "systemPrompt"], (sctx) => {
    const scope = sctx.settings.register(NS, z.object({ text: z.string().default("") }));
    // 动态注入系统提示词（text 可函数，每次模型步进重求值）
    sctx.systemPrompt.section({
      name: "user:my-section",
      order: 5,
      text: () => scope.get()?.text ?? "",
    });
  });
}
```

铁律：host `apply` 用 `ctx.inject([...services], cb)` 等服务就绪；**不要** `export const name/inject`（与失败案例对比）。

## 三、浏览器半侧（lib/client.js）—— 必须的 bundle 格式

```js
window.__ModuleLoader__.load({
  id: "dsh-plugin-xxx",           // 裸包名
  factory: (require) => {
    var module = { exports: {} };
    var exports = module.exports;
    let react = require("react");
    let runtime = require("@deepseek-ai/dsh-client-runtime/client"); // 基线模块
    // ... 业务代码 ...
    exports.apply = apply;
    exports.inject = ["slots", "locale", "settingsScope"]; // 具名服务注入
    return module.exports;
  },
});
```

- `require` 只能引**基线模块**（react、react/jsx-runtime、`@deepseek-ai/dsh-client-runtime/client`、`@deepseek-ai/dsh-client-ui-slots` 等已内置者）；
- **只导出 `apply` + `inject`**，不要额外导出 `name`（对照失败案例）；
- CSS 可注入 `<style data-plugin-css>`（dataset.plugin = 包名）。

## 四、通用设置行（settings.general.item 槽）

```js
function apply(ctx) {
  const scope = ctx.settingsScope.bind({ namespace: "my-plugin" });
  const store = runtime.defineStore({
    init: () => ({ value: "" }),
    actions: { sync: (d, v) => { d.value = v; } },
  });
  ctx.slots.inject("settings.general.item", () => ctx.slots.register({
    name: "settings.general.item",
    id: "my-plugin",
    order: 10,
    store,
    locale: "settings.myPlugin",
    inject: (actions) => ({ save: (text) => scope.set("text", text) }),
  }, MyRowComponent));
}
```

组件 props：`{ t, useStore, save }`（t=locale，useStore=store，save=inject 面）。

## 五、设置卡片（settings.plugin.item，官方 adding-a-settings-card 路径）

- Host：`installSettingsSection(ctx, NS, Config, config, { validate, setSource, onChange })`（`@deepseek-ai/dsh-settings`）；
- 浏览器：`ctx.slots.register({name:'settings.plugin.item', key: NS, ...}, Card)`，经 `ctx.settingsScope.bind({namespace: NS})` 读写（`scope.set/unset`，快照含 `value/base/user`，覆盖判定看 `user` 层**是否出现**）；
- 字段 `role('secret')` 值永不出现在响应；`applies:'restart'` 告诉表层重启才生效。

## 六、cordis.patch.yml（bundle patch）

```yaml
# 与 dsh.bundle.patch 配套；装进 profile 即自动启用
- insert:
    - id: my-plugin
      name: 'dsh-plugin-xxx'
```

## 七、装载与验证

```sh
# 语法与冒烟
node --check lib/client.js && node --check lib/index.js
# 组合树
dsh --profile web --dump-config | grep <name>
# 独立 probe（不打扰 3080）
dsh --profile web --port 3110 --no-open
# 浏览器 bundle
Invoke-WebRequest http://127.0.0.1:3110/plugins/<id>/client.js   # 期望 200
```

> 本地开发用 link 安装时，**插件目录内必须自行 `pnpm install`**（link 不会自动装依赖）；生产/发布用 registry 版本号安装最稳。
