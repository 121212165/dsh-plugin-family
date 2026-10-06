# dsh 插件开发完全教程

> 面向：想在 DeepSeek Harness (dsh) 上写插件的人，包括非职业开发者。
> 本文 = 官方技能书（cordis-plugin-development）+ 官方文档站（develop/reference）+ 社区手册（dsh-handbook 829★ 第 4 章）+ **20 个真实上架插件的全部踩坑** 的融合。
> 权威判定顺序：装机内的 cordis_inspect_query > 已安装包的 lib/types 声明 > 官方文档 > 本文。

## 0. 三十秒决策树：你要做哪种插件

| 你想要 | 做什么 | 参考章节 |
|---|---|---|
| 模型能调用一个能力（查数据/写文件/调 API） | Host 插件 + `ctx.tools.register` | §3.2、§4 |
| 用户打 `/命令` 触发一个动作 | Host 插件 + `ctx.commands.register` | §3.1 |
| 每回合给模型注入一段上下文（状态/提醒/规则） | Host 插件 + `ctx.systemPrompt.section` | §3.3 |
| 监听会话事件做统计/告警/归档 | Host 插件 + `ctx.on('session/event')` 或投影 | §3.4、§5 |
| 在网页界面里加面板/卡片/装饰 | Client 插件（client.js + slots） | §6 |
| 给 agent 接一个外部 MCP 服务 | 配置型 MCP bundle | 官方 references/mcp-bundle.md |
| 自动调节模型行为（降档/改写请求） | waterfall 监听（agent/request 等） | 社区手册第 4 章 |

**最小骨架永远是同一套**：一个目录 + `package.json`（声明 `dsh.bundle.patch`）+ `cordis.patch.yml`（把插件插进 profile）+ `index.js/ts`（导出 `apply(ctx, config)`）。

## 1. 十分钟跑通第一个插件

### 1.1 四个文件

`my-plugin/package.json`（Host-only 插件**零依赖零构建**）：

```json
{
  "name": "@local/my-plugin",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "main": "./index.js",
  "exports": { ".": "./index.js", "./package.json": "./package.json" },
  "dsh": { "bundle": { "patch": "./cordis.patch.yml" } }
}
```

`my-plugin/cordis.patch.yml`：

```yaml
- insert:
    - id: my-plugin
      name: '@local/my-plugin'
      config: {}
```

`my-plugin/index.js`：

```js
export const name = 'my-plugin';

export function apply(ctx, config) {
  console.log('[my-plugin] loaded');
}
```

外加一个 `.gitignore`（`node_modules/`、`lib/`、`package-lock.json`——Windows Git Bash 的 `ln -s` 是深拷贝，node_modules 千万别进 git）。

### 1.2 安装与验证（官方流程）

```bash
dsh plugin --profile web add "你的插件目录绝对路径"
dsh --profile web --dump-config | grep my-plugin   # 看到 "- id: my-plugin" = 挂上了
```

在 dsh web 会话里，官方的 `plugin_manager` 工具（`action: install_bundle`）会自动完成 pnpm 安装 + bundle 选择，**不要**手写 profile 的 package.json 或手动跑 pnpm。

**验证纪律**（官方 verification.md）：安装结果的 `application` 字段才是生效凭证（applied / restart-required / overridden / failed 四态，`overridden` = 有更高优先级层赢了你）；重启 web 后开新会话，让模型调一次你的能力确认真实可见——"装上"不等于"能看到"。

### 1.3 插件的三种导出形态（不要混用）

```js
export function apply(ctx, config) {}          // 函数形态（最常用）
export default class MyService extends Service { static inject = ['tools'] }  // 服务形态
```

资源注册一律放在 `apply` 内，配 `ctx.effect(() => cleanup)` 或 `ctx.on(...)`（框架自动清理）。

## 2. 命名与 manifest 红线

- **id 和包名必须全局唯一**；row id 与插件名重复 = 启动失败（`command X is already registered` / duplicate id）；
- 显示名与描述放 `locale/en.json` + `locale/zh.json`（字段 `{"meta": {"title", "description"}}`，文件名必须是 language id）；
- 图标：package.json 顶层 `"icon": "./icon.svg"`，相对路径，SVG/PNG/JPEG/WebP ≤256KiB，绝对路径/URL/越出目录的路径都会被拒；
- **exports 白名单**：`./locale/*.json`、`./icon.svg`、`./package.json` 必须放行，否则 locale 静默不生效（icon 不受此限）；
- 格式错误的 locale 产生诊断但保留有效文本——不会挂掉插件。

## 3. 五大扩展点（全部带真实代码）

### 3.1 命令（用户 / 触发）

```ts
export const inject = ['commands'];

ctx.commands.register({
  name: 'mycmd',
  description: '一句话说明，斜杠菜单会显示',
  input: { hint: '<参数提示>' },
  handler: ({ rawInput }) => {
    if (!rawInput?.trim()) return { kind: 'error', text: '用法：/mycmd <参数>' };
    return { kind: 'success', text: `完成：${rawInput}` };
  },
});
```

纪律：
- **handler 永远不许抛异常**——web UI 不显示 handler 异常，用户看到的就是"输入消失、无响应"（我们踩过的最大坑）。统一包装：

```ts
const guarded = (args) => {
  try { return handler(args); }
  catch (error) { return { kind: 'error', text: `命令 ${name} 内部出错：${String(error)}` }; }
};
```

- 参数解析**接受人话**：模糊匹配 id 前缀/标题子串（如 `/relay 图片取色`），别让用户背编号；
- 返回 `{ kind: 'success'|'error', text }`，异步 handler 返回 Promise 同样合法。

### 3.2 工具（模型调用）

```ts
export const inject = ['tools'];

ctx.tools.register(defineTool({
  name: 'my_lookup',
  description: '写给模型看：什么场景该调它、参数怎么给。这一行决定模型会不会用对。',
  parameters: {
    query: { type: 'string', required: true, description: '关键词' },
  },
  output: {
    schema: { type: 'string' },
    render: (_args, value) => [{ type: 'text', text: value }],
  },
  async execute(args, exec) {
    // args 已按 schema 自动校验；exec.signal 必须尊重（取消时停止工作）
    return doLookup(args.query, exec.signal);
  },
}));
```

进阶字段（官方 adding-a-tool cookbook）：

| 字段 | 用途 |
|---|---|
| `output.presentationMeta(args, value)` | 从规范值派生可回放 JSON，持久化到 tool/result（UI 卡片回放用） |
| `presentCall(args)` / `presentResult(args, result)` | UI 卡片：`{ card: 'generic', title, kind, rawInput }`，kind 枚举 read/edit/delete/move/search/execute/fetch/other |
| `producer` | 后台任务：`ctx.jobs.start({ kind, label, owner: exec.agent, run })`，返回 `{ kind: 'background', jobId }` |
| `timeoutMs` / `isConcurrencySafe` | 截止时间 / 并发安全标记 |

卡片硬规则：**纯函数**（不做 I/O、不读会话状态、不用时钟随机数）；格式错误软校验返回 undefined（回退 generic 卡），展示绝不能让回放崩溃；UI 格式（console 块/diff/相对路径）**不进模型可见内容**。

⚠ **关键真相（官方文档原话）**：*内置 Web Client 不消费 presentCall/presentResult*——Client 插件要在 `tool.call.toolview` slot 按工具名注册渲染器。所以：generic 卡是 Host 侧的正确契约（未来客户端渲染的基础），但**要在默认网页里看到漂亮卡片，还需要一个 Client 插件注册 `tool.call.toolview`**。这是家族踩过的认知坑。

错误处理模式：注册表会把 execute 异常收敛成 isError 结果；领域状态（如"进程非零退出"）应写进规范值让渲染器解释，而不是抛异常。

### 3.3 系统提示注入（每回合给模型看的东西）

```ts
export const inject = ['systemPrompt'];

ctx.systemPrompt.section({
  name: 'my-status',           // 全局唯一，重名抛错
  order: 700,                  // 升序拼接；工具区从 1000 起
  text: (assembleContext) => {
    // 每次组装时调用；assembleContext.scope 可区分会话/agent（官方 AssembleContext）
    return computeStatusText(); // 空字符串 = 本回合不注入
  },
});
```

- 组装时机：**每次模型步骤前**，每次都求值 text 函数——读文件/查状态都来得及，但保持轻；
- `systemPrompt.context({...})` 用于需要持久化为 user-role 快照的动态上下文（只在变化或压缩时写入历史）；
- `ctx.systemPrompt.variable(name, provider)` 供 `{{变量}}` 插值；
- 需要唤醒 agent 时用 `agent.followup()`——`agent.inject()` 只投递不唤醒（空闲 agent 会一直闲着）。

### 3.4 会话事件（监听发生的事）

```ts
ctx.on('assistant/message', (session, event) => {
  const usage = event.data?.usage;
  // 记账/统计/告警
});
```

可用事件：`assistant/message`、`tool/call`、`tool/result`、`session/*`、`agent/created`、`turn/end` 等。规则：
- **只做轻工作**：监听在事件流上，重活放缓存/投影；
- **不要往 session append 新 type 的事件**——没有 `ignorable: true` 信封的话，session 会拒绝 reopen。派生状态走投影或自有存储。

### 3.5 会话投影（官方推荐的状态模型）

每会话的派生状态（计数/列表/汇总）官方模型是 `ctx.sessionProjections`：

```ts
import { z } from 'zod';

declare module '@deepseek-ai/dsh-session-projection' {
  interface SessionProjectionStateMap { 'my-state': MyState }
}
import type {} from '@deepseek-ai/dsh-session-projection/types';

ctx.sessionProjections.register({
  key: 'my-state',
  stateSchema: z.object({ count: z.number().int().nonnegative() }),
  stateVersion: 1,                       // 字段/语义变化时 +1
  init: () => ({ count: 0 }),
  apply: (state, event) =>
    event.type === 'assistant/message' ? { count: state.count + 1 } : state,  // 无关事件必须返回同一引用
});
```

读取：`ctx.sessionProjections.stateOf(session, 'my-state')`。
收益：框架增量驱动（无关事件零开销）、检查点持久化、**fork/resume 后从日志重放恢复**——自有内存 map 做不到最后这条。注册用 `ctx.inject(['sessionProjections'], ...)` 条件通道，seam 缺失时插件其余功能照常活。

### 3.6 waterfall（改写模型行为，高级）

`agent/request`、`tools/pre-execute`、`tools/post-execute` 等 waterfall 可以改写决策。铁律：
- 不拥有决策就 `return next()`；
- **`next()` 是 Promise 必须 await**（不 await 直接 spread 会拿到空对象——社区手册第 4 章的头条坑）；
- 改写决策要 spread：`{ ...decision, messages }`，保住 `startsRequestSeries` 等字段；
- 单调拒绝用 `ctx.tools.guard()`；隐藏工具用 `ctx.tools.restrict()`；观测最终结果用 `tools/result` 而不是 `tools/post-execute`。

## 4. 状态与存储

| 需求 | 方案 |
|---|---|
| 每会话派生状态（客户端要读 / 要 fork-resume 安全） | sessionProjections 单元（§3.5） |
| 跨会话聚合（今日合计等） | 自有 JSONL sidecar（`~/.dsh/<plugin>/...`）——官方允许的"插件自有派生缓存" |
| 配置 | `export const Config = Schema.object({...})`，用户在 patch 层改，升级存活 |

存储纪律：
- 写文件一律 **temp+rename 原子写**；
- JSONL 追加而非全量重写（并发安全）；
- 自有数据带 `v: 1` 信封字段（别人/旧版会读你写的）；
- 路径 `expandHome` + `mkdirSync(dirname, { recursive: true })`；
- **重活放进 sessionProjections，监听里只做轻活**。

## 5. 测试三层（缺一层=质量 illusion）

1. **纯函数单测**（node --test）：决策逻辑零依赖，毫秒级覆盖全分支——把逻辑从 apply 里抽出来；
2. **装配层 harness 测试**：mock ctx（commands/tools/systemPrompt 捕获 + 假会话录 followup）挂**真实 apply()** + **真实临时目录**，驱动完整用户流程。这是"测试全绿但用户不能用"的解药——纯函数测试验证代码对，harness 验证装配对；
3. **实机探针**：`dsh plugin add` 后 `--dump-config` 确认挂载；**启动探针**——故意给非法 config，启动必须失败并点名你的插件（静默降级 = 埋雷）；headless 真模型冒烟一次完整用户流程。

验证纪律（官方）：用已连接的页面做视觉验证；安装结果的 `application` 字段决定是否生效；不做 mock 截图器、不模拟 DOM。

## 6. Client 插件（网页界面）

网页里加面板/装饰 = 第二个模块（client.js）：

```js
window.__ModuleLoader__.load({
  id: '@local/my-client',
  factory(require) {
    const React = require('react');
    const h = React.createElement;
    function Panel() { return h('div', { style: { padding: 8 } }, 'hello'); }
    return {
      inject: ['slots'],
      apply(ctx) {
        ctx.slots.inject('conversation.composer.dock', () => ctx.slots.register({
          name: 'conversation.composer.dock', id: 'my-panel', order: 5,
        }, Panel));
      },
    };
  },
});
```

铁律：样式**只用 `--dsw-alias-*` 主题 token**（字面色仅限艺术品）；**禁止 require 任何 @deepseek-ai/dsh-client-* 包**（要控件就抄宿主实现改类名前缀）；不写 DOM 到组件外；Conversation 行走 `ctx.uiConversation.events.register()` + `conversation.chat.node` slot；工具卡片渲染注册在 `tool.call.toolview` slot。

## 7. 发布

1. GitHub 公开仓（README 带借鉴来源与差异表——社区文化）；
2. 打 topic：`dsh-plugin`、`deepseek-harness`、`dsh`（三大目录站自动发现）；
3. 投 awesome 收录（awesome-dsh-plugin 仓库 PR，data/plugins/ 一插件一文件）；
4. 可选：发 npm（`files` 白名单只带 lib/src/locale/icon/cordis.patch.yml）；
5. 挂 profile：`dsh plugin --profile <p> add <路径>`。

## 8. 踩坑大全（20 个真实插件攒的，按出现频率排序）

| # | 坑 | 症状 | 解法 |
|---|---|---|---|
| 1 | handler 抛异常被 web 静默吞掉 | 用户：输入消失、无响应 | 所有 handler try/catch 包装，异常变可见错误文本 |
| 2 | locale/icon 不生效 | 插件管理器显示包名+英文 | exports 白名单放行 `./locale/*`、`./icon.svg` |
| 3 | 改了代码没生效 | 行为还是旧的 | **必须 `npm run build`**——profile 加载的是 lib/；typecheck 不产出 |
| 4 | 命令/工具重名 | startup failed: already registered | 命名前查 dump-config；家族命令带独特前缀 |
| 5 | 非法 config 静默降级 | 功能坏了没人知道 | apply() 开头显式校验并 throw（启动探针文化） |
| 6 | source kind 用老写法 `'plugin'` | 持久化报 requires a producer-owned source kind | v4 用 `kind: 'plugin:<name>'` 或自有 kind |
| 7 | followup() 静默无效/抛错 | 命令成功但模型无反应 | try/catch + 系统提示 section 双通道兜底 |
| 8 | node_modules 进 git | 仓库几百 MB | .gitignore 三件套；**别用 ln -s**（Windows Git Bash 是深拷贝） |
| 9 | 依赖版本线 | 依赖链断裂 | 官方包版本用 `>=0.1.2-rc.1 <0.2.0` |
| 10 | TS 模块增强 TS2664 | Invalid module name in augmentation | augmentation 前先 `import type {} from '...'` |
| 11 | 装 @deepseek-ai 包打乱 hoisted 树 | Cannot find package '@deepseek-ai/dsh-scope' | 显式补装缺失的 peers（同版本线） |
| 12 | Schema 默认值不存在 | 直接调 apply 缺字段 | mock/测试里手动补全 config 默认值 |
| 13 | 纯函数返回值没存回 | 状态永远不变 | fold 纯函数的结果要 Map.set 存回 |
| 14 | Windows 路径 | mkdir 崩溃/首次 import 即崩 | dirname + recursive + expandHome |
| 15 | 写脚本批量改代码 | `\n` 变真换行、文件写错目录 | 用 Edit 工具；每次操作前 pwd 确认 |
| 16 | 水印依赖版本过老 | 各 rc 线不兼容 | peerDependencies 用范围锁 |
| 17 | 测试全绿但用户不能用 | 差距巨大 | 缺装配层测试——harness 挂真实 apply() |
| 18 | 长会话前缀暴涨 | 每轮成本线性涨 | 用 quota 类插件监控 + 主动开新会话 |
| 19 | 中转"密钥无效" | 上游拒绝 | 先 curl 中转看真错误：额度耗尽 ≠ key 无效 |
| 20 | sidecar 并发写丢数据 | 行丢失 | append-only 或 temp+rename，别全量覆写 |

## 9. 资源地图（按权威度）

**官方（随产品分发，最权威）**：
- 技能书：`%APPDATA%\npm\node_modules\@deepseek-ai\dsh\node_modules\@deepseek-ai\dsh-agent-preset\skills\` 下三本：`cordis-plugin-development`（主流程+references+模板）、`cordis-composition-reference`（patch 方言+全部可挂载包清单）、`editing-cordis-compositions`（agent preset 编辑）；
- 文档站：deepseek-harness.github.io/deepseek-harness——`develop/basic|framework|practice|cordis-tutorial(01-07)` + `reference/`（架构、tool-execution-pipeline、全部子系统、cookbook 五篇：adding-a-tool 必读）；
- `cordis_inspect_query`：web 会话里直接查 Service 方法/事件名/Config schema/Client Slots/Theme tokens（Creator 模式开箱即用）；
- 官方仓库：deepseek-ai/deepseek-harness（packages/README.md 是包分组地图）。

**社区（高质量）**：
- dsh-handbook（829★）——第 4 章插件实战（waterfall + await next 坑 + 三层验证）；
- dsh-plugin-template（110★）——带 CI/eslint/client.tsx 的全功能模板；
- build-deepseek-harness-plugin（44★）/ deepseek-harness-plugin-creator（40★）——创建与校验工具；
- dsh-context（1841★）、dsh-plugin-radar（1466★）——UI 插件与生态雷达的标杆实现；
- awesome-dsh-plugin（17770★）/ dsh-plugin-shop（1002★）——目录与市场（发布后去收录）。

**本家（ Workspace）**：
- `dsh-plugin-quota` —— 投影模型 + 卡片 + 零配置的参考实现；
- `dsh-plugin-task-forge` —— 命令组 + 模糊匹配 + 防弹 handler + harness 测试的参考实现；
- `FAMILY-DEEPENING-PLAN.md` / `DSH-PLUGIN-OFFICIAL-PRACTICES.md` / `dsh-audit-report.md` —— 审计与深化记录。
