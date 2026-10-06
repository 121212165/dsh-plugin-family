# dsh-plugin-family

DeepSeek Harness（dsh）插件家族总目录 —— **22 个插件，按工作流组织，全部零配置原则：选上就能用，不填表单、不学格式**。

一条命令安装任意插件：

```bash
dsh plugin --profile web add github:121212165/<仓库名>
```

## ⭐ 明星插件（先看这三个）

| 插件 | 一句话 | 入口 |
|---|---|---|
| [dsh-plugin-ide-hub](https://github.com/121212165/dsh-plugin-ide-hub) | 一个入口管住所有编码 IDE：8 个 IDE 零 API 盘点（含 Trae / Qoder / CatPaw 国产适配）、用量统计、会话一键恢复、规则一份本体多 IDE 下发 | `/ide-hub` `/hub-usage` `/hub-sessions` `/hub-init` `/today` |
| [dsh-plugin-task-forge](https://github.com/121212165/dsh-plugin-task-forge) | 把大白话编译成任务书，跨窗口 AI 无损交接：回读握手逼出理解损耗，缺口闭环、版本号跨窗口唯一权威 | `/forge` `/relay` `/ack` `/answer` `/forge-list` |
| [dsh-plugin-quota](https://github.com/121212165/dsh-plugin-quota) | 实时用量仪表 + 下一步预估 + 预算刹车：每回合给模型一行实时 token/花费仪表，模型自己看得见、自己刹 | `/qm` `/qm-top` `quota_meter` 工具 |

## 全家族清单

### 会话与知识库

| 插件 | 一句话 | 入口 |
|---|---|---|
| [transcript](https://github.com/121212165/dsh-plugin-transcript) | 会话转录归档：user/assistant/tool 全量落 JSONL → Markdown，接 Obsidian 工作流 | 自动归档 |
| [transcript-search](https://github.com/121212165/dsh-plugin-transcript-search) | 跨会话全文搜索，与 transcript 的 JSONL 契约互通 | `/find` `session_search` |
| [obsidian-push](https://github.com/121212165/dsh-plugin-obsidian-push) | 转录推送 Obsidian 库：frontmatter + 内容哈希幂等去重，vault 自动发现；`/archive` 一键归档链 | `/obsidian-push` `/archive` |
| [html-report](https://github.com/121212165/dsh-plugin-html-report) | 会话渲染成自包含 HTML 报告（深色模式、无外链），分享/归档用 | `/report` |
| [session-insights](https://github.com/121212165/dsh-plugin-session-insights) | 跨会话统计总览：会话数、token、趋势 | `/insights` |
| [pinboard](https://github.com/121212165/dsh-plugin-pinboard) | 置顶便签注入系统提示：中途加便签下一回合生效，不用重启 | `/pin` `/pins` `pin_add` |
| [fact-vault](https://github.com/121212165/dsh-plugin-fact-vault) | 事实便签库：登记可复用的项目事实，tag 加权检索 | `/fact` `fact_find` |
| [prompt-vault](https://github.com/121212165/dsh-plugin-prompt-vault) | 提示词弹药库：markdown 导入、标签、使用计数，自带 8 条中文种子提示词 | `/pv` |

### 钱与用量

| 插件 | 一句话 | 入口 |
|---|---|---|
| [quota](https://github.com/121212165/dsh-plugin-quota) ⭐ | 实时用量仪表 + 下一步预估 + 撑满/预算双预测 | `/qm` |
| [cost-ledger](https://github.com/121212165/dsh-plugin-cost-ledger) | 花费持久台账：按月 JSONL、月报、CSV 导出、环比结论 | `/ledger` |
| [relay-quota](https://github.com/121212165/dsh-plugin-relay-quota) | 查任意 OpenAI 兼容中转的余额/用量 | `/quota` `quota_check` |
| [spend-forecast](https://github.com/121212165/dsh-plugin-spend-forecast) | 按日均消耗预测余量撑几天、几号烧穿 | `/forecast` |
| [price-aware](https://github.com/121212165/dsh-plugin-price-aware) | 价格感知注入（宿主侧参考实现风格） | 自动注入 |
| [token-telemetry](https://github.com/121212165/dsh-plugin-token-telemetry) | token 遥测 web 面板（官方 useProjection 模式示范） | web 面板 |

### 工具与质量

| 插件 | 一句话 | 入口 |
|---|---|---|
| [tool-trace](https://github.com/121212165/dsh-plugin-tool-trace) | 工具调用追踪：做了什么、慢在哪，只记大小不记内容 | `/tools-stats` |
| [error-radar](https://github.com/121212165/dsh-plugin-error-radar) | 错误率/连败/p95 雷达，区分"系统性故障"和"抖动" | `/radar` `/health` |
| [cache-guard](https://github.com/121212165/dsh-plugin-cache-guard) | 前缀缓存健康守卫：检测前缀被改写导致的缓存失效多花钱，提醒模型住手 | `/health` `cache_status` |

### 扫描与研究

| 插件 | 一句话 | 入口 |
|---|---|---|
| [deep-scan](https://github.com/121212165/dsh-plugin-deep-scan) | GitHub 赛道深度扫描器：多词四通道搜索 + 自动挖 topic 迭代，止损规则内置 | `/deep-scan` `deep_scan` |
| [eco-scan](https://github.com/121212165/dsh-plugin-eco-scan) | dsh 插件生态市场扫描器：topic 全仓 + npm 周下载差值 + 增长评分 | `/eco-scan` `eco_scan` |

### 编排与界面

| 插件 | 一句话 | 入口 |
|---|---|---|
| [task-forge](https://github.com/121212165/dsh-plugin-task-forge) ⭐ | 大白话 → 任务书 → 跨窗口无损交接（回读握手） | `/forge` |
| [ide-hub](https://github.com/121212165/dsh-plugin-ide-hub) ⭐ | 跨 IDE 统一管理层（8 适配器含国产） | `/ide-hub` |
| [family-cards](https://github.com/121212165/dsh-plugin-family-cards) | 家族 11 个工具的 web UI 卡片渲染器（补官方 Client 不消费 presentCall 的缺口） | 自动渲染 |

## 推荐工作流组合

- **先装这三个**：`quota`（看见钱）→ `ide-hub`（看见所有 IDE）→ `task-forge`（跨窗口派活）；
- **归档链**：`transcript` → `obsidian-push` → `transcript-search`（/archive 一键跑全链）；
- **省钱组合**：`cache-guard`（缓存失效告警）+ `spend-forecast`（烧穿预测）+ `cost-ledger`（月度对账）；
- **插件之间会互相握手**：task-forge 读 quota 的 `summary.json` 预算契约；ide-hub 的 `/today` 聚合 quota / cost-ledger / task-forge / tool-trace 四家发布在磁盘上的事实。

## 安装通用说明

```bash
# ① 装进 profile（git 包自动跑 prepare 构建）
dsh plugin --profile web add github:121212165/<仓库名>
```

② 需要自定义配置时，把对应仓库根目录 `cordis.patch.yml` 的条目**并进** `$DSH_HOME/profiles/<profile>/cordis.patch.yml` 的同一个 YAML 数组（不要另起文档追加）；不并也能用——家族默认配置即可工作。

③ 重启 dsh 生效。自检：`dsh --profile web --dump-config | grep <插件名>`。

安装排坑（死代理 / pnpm 拦构建脚本 / peer 警告）的完整清单见 [ide-hub README](https://github.com/121212165/dsh-plugin-ide-hub#装不上时先查这三样2026-10-03-干净房实测踩点)。

## 质量与原则

- **零配置**：每个插件选上即用，默认值开箱可用；必须填的配置不存在。
- **测试**：家族合计 380+ 个 `node --test`（纯函数层 + 装配层），task-forge 的 mock-ctx harness 是全家装配层测试模板。
- **坏配置启动点名**：配置填错启动即失败并点名插件，绝不带着不可用配置静默运行。
- **借鉴来源制度化**：每个借鉴型插件的 README 都有"借鉴来源与差异"表，写明借鉴了什么、改了什么、哪些原创。

## 想自己写一个插件？

看 [docs/DSH-PLUGIN-DEV-TUTORIAL.md](docs/DSH-PLUGIN-DEV-TUTORIAL.md) —— dsh 插件开发完全教程：官方技能书 + 官方文档站 + 社区手册（dsh-handbook）+ 20 个真实上架插件的全部踩坑的融合，30 秒决策树开篇，十分钟跑通第一个插件。

## License

全部 MIT。
