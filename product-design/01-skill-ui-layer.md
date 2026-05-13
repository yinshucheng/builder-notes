# Skill UI Layer — 从文本到小程序的展示层革命

**日期**：2026-04-02  
**状态**：Idea / 待验证  
**关联项目**：Commander、Agent系统学习、AI工具链实践、Vibeflow

---

## 一句话

Skill 的展示层不需要从零生成——任何已存在的 Web 产品都可以作为 skill 的 UI。Skill 的角色是"意图注入器"，把 agent 理解的用户意图传递给已有 App，让它以预填好的状态呈现。这些 App 脱离 agent 独立可用，接入 agent 后通过 skill 获得智能调度能力。承载这些 App 的，是一个能渲染 rich content 的 AI-native IM。

---

## 核心洞察

**现状**：Claude Code / OpenClaw / Codex 等 agent 平台，skill 的最终输出形态是**纯文本流**。skill 可以调 API、读写文件、执行脚本，但最终结果只能以 text 形式呈现在 chatbot 中。

**问题**：很多信息天然不适合文本承载：

| 场景 | 文本呈现 | 理想呈现 |
| --- | --- | --- |
| 记账 | 一大段 markdown 表格 | 收支饼图 + 月度趋势线 |
| Todo | 缩进的列表 | 看板/甘特图，可拖拽 |
| 股票 | 数字罗列 | K线图 + 实时滚动 |
| 调研报告 | 长文 | 多源卡片 + 可折叠 + 书签 |
| 周报/复盘 | markdown | 目标进度仪表盘 |
| Eval 结果 | JSON dump | skill-creator 已经做了 eval-viewer |

**关键发现**：skill-creator 的 `eval-viewer/generate_review.py` 已经证明了这条路——skill 生成 HTML，`open` 打开浏览器，用户在 UI 中操作，结果通过文件回传。但这只是一个临时方案，没有被抽象成通用范式。

## 核心命题

> **Skill 的输出不应只是 text stream，应该可以是一个 rich UI canvas。这个 UI 层可以替换掉 chatbot 中的文本内容，成为 skill 的"前端"。**

更进一步：

> **这个 UI 层不一定由 skill 作者提供。任何人都可以为某个 skill 的输出做展示层，就像 Chrome 扩展可以改变网页的展示一样。**

## 类比

| 概念 | 类比 |
| --- | --- |
| Skill | 后端 API |
| Skill 当前的 text 输出 | API 返回 raw JSON |
| Skill UI Layer | 前端 app 消费 API |
| skill 作者做 UI | 全栈开发 |
| 第三方做 UI | 前后端分离，别人可以为你的 API 做 app |

**这就是 skill 生态的前后端分离。**

## 与现有方案的对比

### Claude Artifacts

*   对话中生成 HTML/React，侧面板渲染
*   **局限**：仅限 claude.ai 网页端；每次都重新生成，无持久化；单次对话绑定；无法在 CLI 环境使用

### ChatGPT Canvas

*   文档/代码编辑面板
*   **局限**：本质是编辑器，不是 app；不接受外部数据驱动

### AG-UI (CopilotKit)

*   Agent ↔ React 前端的双向事件协议
*   **优势**：最完整的 agent-UI 交互协议，双向通信，typed event stream
*   **局限**：需要完整前端工程（React 项目 + 部署），不是 skill 级别的轻量方案；面向的是"构建 agentic 应用的开发者"，不是"skill 用户"

### A2UI (Google)

*   Agent 返回 JSONL 格式的 UI widget 声明
*   **优势**：声明式，轻量
*   **局限**：尚早期，没有成熟实现

### ChatGPT Apps SDK (OpenAI, 2026-04)

*   **MCP 扩展**：在 MCP 基础上增加 UI 渲染能力，开发者可定义 app 的界面和聊天逻辑
*   **嵌入对话**：app 在 ChatGPT 对话中自然出现（地图、播放列表、PPT），可以被 ChatGPT 在合适时机推荐
*   **开发者后端直连**：支持用户登录、付费功能
*   **优势**：8 亿用户分发、MCP 兼容、开源标准
*   **局限**：仅限 ChatGPT 生态；开发者需要维护后端服务；面向的是"为 ChatGPT 做 app 的开发者"而非 skill 生态
*   **核心启发**：**验证了"chat 内嵌 mini-app"的方向是对的，OpenAI 已经 all-in 了**

### Open WebUI Rich UI Embedding

*   Tool/Action 返回 `HTMLResponse` → 在 chat 中内嵌为 iframe
*   自动注入 tool call 参数到 `window.args`
*   支持 auto-sizing、sandbox 安全策略
*   **优势**：已经可用，开源，证明了"tool call → rich UI"的技术路径
*   **局限**：仅限 Open WebUI 平台；iframe 沙箱有限制；没有展示层与逻辑层分离的概念

### skill-creator eval-viewer

*   Skill 内置 Python 脚本生成 HTML，`open` 打开浏览器
*   **最接近本提案的原型**
*   **局限**：一次性查看器，无通用抽象，无数据回流协议

### 本提案的定位

*   **比 Artifacts 持久**：不是一次性渲染，是可更新的 mini-app
*   **比 AG-UI 轻量**：不需要完整 React 项目，skill 只需声明数据格式 + 选择/提供模板
*   **比 A2UI 务实**：不定义新协议，复用已有的 HTML + 文件系统/localhost
*   **核心创新：展示层与逻辑层分离**，允许第三方为 skill 做 UI

## 架构设计

### 三层模型

```
┌─────────────────────────────────────────────────┐
│                  Skill UI Layer                   │
│  (HTML/JS mini-app, 运行在浏览器中)                 │
│                                                   │
│  - 读取 skill 输出的结构化数据                      │
│  - 渲染为交互式 UI                                 │
│  - 用户操作回写为数据文件                           │
└────────────────────┬────────────────────────────┘
                     │ 数据交换（文件/localhost/WS）
┌────────────────────▼────────────────────────────┐
│               Skill Data Contract                 │
│  (JSON Schema，skill 输出的标准数据格式)             │
│                                                   │
│  - skill 声明自己的输出 schema                     │
│  - UI layer 按 schema 渲染                        │
│  - 多个 UI layer 可以消费同一个 schema              │
└────────────────────┬────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────┐
│                  Skill Logic                      │
│  (SKILL.md + scripts，运行在 agent 中)              │
│                                                   │
│  - 获取数据、处理逻辑                              │
│  - 输出结构化 JSON 到约定路径                      │
│  - 可选：在 chatbot 中输出文本摘要                  │
└─────────────────────────────────────────────────┘
```

### 数据交换机制

**方案 A：文件系统（最简，今天就能用）**

```
~/.skill-ui/
├── accounting/
│   ├── data.json          ← skill 写入
│   ├── user-action.json   ← UI 写入，skill 读取
│   └── index.html         ← UI 模板
├── todo/
│   ├── data.json
│   └── index.html
```

*   Skill 执行后写 `data.json`，然后 `open index.html`
*   HTML 读本地 `data.json`（file:// 或 localhost 小 server）
*   用户操作写入 `user-action.json`，下次 skill 运行时读取

**方案 B：Localhost dev server（交互更好）**

```
skill-ui serve --port 3721

GET  /api/accounting/data     → 返回 data.json
POST /api/accounting/action   → 写入 user-action.json
WS   /ws/accounting           → 实时双向
```

*   Skill 可以在后台持续推送数据更新
*   UI 可以实时反映变化（股票行情、番茄倒计时）

### SKILL.md 中的 UI 声明

在 SKILL.md frontmatter 中扩展：

```
---
name: my-accounting
description: "Personal accounting tracker..."
ui:
  type: html                    # html | none
  entry: ui/index.html          # UI 入口
  data-schema: schemas/data.json # 输出数据的 JSON Schema
  auto-open: true               # skill 执行后自动打开
  persistent: true              # UI 是否持久（vs 一次性查看）
---
```

### 第三方 UI 的发现机制

```
skill-ui-registry/
├── accounting/
│   ├── default/          ← skill 作者提供
│   │   └── index.html
│   ├── dark-dashboard/   ← 社区贡献
│   │   └── index.html
│   └── mobile-first/     ← 另一个社区贡献
│       └── index.html
```

用户可以选择：

```
/accounting --ui=dark-dashboard
```

## 与 Commander 的关系

Commander VISION 的核心是"统一工作入口"+"内联操作"。Skill UI Layer 恰好是 Commander Phase 3（内联操作）的实现方式：

*   **Commander 是宿主**：Commander Dashboard 内嵌 skill 的 mini-app
*   **Skill UI 是 native app**：每个 skill 的 UI 就是 Commander 里的一个"应用"
*   **事件流 + rich UI**：Commander 的事件流视图中，每条事件可以展开为对应 skill 的 UI

```
Commander Dashboard
├── 事件流（全局）
│   ├── [accounting] 本月支出 ¥12,340  → 点击展开 → 饼图+明细
│   ├── [todo] 3 项待办到期          → 点击展开 → 看板视图
│   ├── [stock] AAPL +2.3%           → 点击展开 → K线图
│   └── [claude-code] Task 完成      → 点击展开 → diff 预览
```

**这意味着 Skill UI Layer 可以是 Commander 的核心 SDK。**

## 关键转折：展示层的三种形态

之前的思路局限在"skill 生成 HTML"。实际上展示层有三种形态，由轻到重：

### 形态一：AI 生成的临时 UI（Generative）

Skill 执行后生成一个 HTML 页面，一次性渲染。

```
用户: "帮我分析这个 CSV"
→ Skill 处理数据 → 生成一个图表 HTML → 展示
```

适用于：数据可视化、eval 结果、一次性报告。这是之前讨论的方案。

### 形态二：Skill 开发者提供的 Web App（Hosted）

Skill 作者维护一个独立的 Web 产品（有域名、有后端），skill 负责连接。

```
用户: "看看我的番茄进度"
→ Vibeflow skill → 打开 vibeflow.app/dashboard → 数据已准备好
```

**App 独立于 agent 也完全能用。** Vibeflow 本身就是一个产品，有自己的 URL，用户可以直接访问。但通过 skill 接入后，agent 可以：

*   自动登录 / 传递上下文
*   预填状态（已选中今天、已展开某个任务）
*   在 IM 中内嵌显示，不用跳转浏览器

### 形态三：第三方已有产品接入（External）

**这是最大的想象空间。**

麦当劳、美团、淘宝、Spotify……这些产品已经存在，有完整的 Web 版。Skill 的作用是：

1.  **理解用户意图**（agent 做的事）
2.  **将意图转化为 App 可理解的参数**（skill 做的事）
3.  **打开 App 并注入意图**（UI layer 做的事）

```
用户: "帮我点一杯拿铁，送到公司"
→ agent 理解意图：麦当劳，拿铁，公司地址
→ mcdonalds skill：构造订单参数
→ 打开 mcdonalds.com/order?item=latte&address=xxx
→ 用户只需确认并支付
```

### 形态四：聚合型 Skill UI（Aggregated）

一个 skill 后面连着**多个** External/Hosted App，提供一个自己的聚合 UI 作为统一入口。

```
用户: "买个 AirPods Pro"

→ 比价 skill 同时调用 4 个平台 API/爬虫
→ 展示层：聚合比价卡片
  ┌────────────────────────────────┐
  │ 🎧 AirPods Pro 4 — 全网比价    │
  │                                │
  │ 🏆 拼多多  ¥1,299  [去购买]    │ ← 点击 → 打开拼多多页面
  │    京东    ¥1,399  [去购买]    │ ← 点击 → 打开京东页面
  │    美团    ¥1,459  [去购买]    │
  │    淘宝    ¥1,499  [去购买]    │
  │                                │
  │ 📉 历史低价 ¥1,199 (2026-03)   │
  │ 🔔 [设置降价提醒]              │
  └────────────────────────────────┘
```

关键特征：

*   **聚合 UI 本身是 skill 作者提供的**（比价卡片是新的，不是任何一家电商的页面）
*   **每个"去购买"按钮打开的是独立的 External App**（各电商自己的页面）
*   **每个页面之间不需要跳转**——用户看完比价，点一个就走，交互结束
*   **聚合 UI 的价值 = 跨平台信息整合 + 决策辅助**，这是单个 External App 做不到的

更多聚合 skill 场景：

| 聚合 Skill | 连接的 App | 聚合 UI 价值 |
| --- | --- | --- |
| 比价 | 淘宝/京东/拼多多/美团 | 价格对比 + 历史低价 + 降价提醒 |
| 旅行规划 | 携程/去哪儿/Airbnb/Google Maps | 机票+酒店+行程一图看完 |
| 外卖聚合 | 美团/饿了么/麦当劳/星巴克 | 配送时间+价格+优惠对比 |
| 投资看板 | 同花顺/雪球/富途/Coinbase | 多账户资产汇总 + 持仓分析 |
| 内容聚合 | YouTube/B站/Spotify/播客 | "今日推荐"跨平台 feed |
| 求职 | Boss直聘/LinkedIn/拉勾 | 岗位对比 + 薪资分析 |

### 四种形态的对比

|   | Generative | Hosted | External | Aggregated |
| --- | --- | --- | --- | --- |
| **UI 来源** | AI 生成 | skill 作者维护 | 第三方已有产品 | skill 作者做聚合层，连接多个 App |
| **独立可用** | 否（临时） | 是 | 是 | 聚合 UI 可独立，底层 App 也独立 |
| **skill 角色** | UI 生产者 | 数据+入口提供者 | 意图注入器 | 信息聚合器 + 决策辅助 |
| **例子** | CSV 图表 | Vibeflow | 麦当劳 | 全网比价 |
| **类比** | Artifacts | 微信小程序 | 微信内打开 H5 | 什么值得买 / Google Shopping |
| **接入成本** | 零 | 中 | 高 | 高（N 个 External 的集成） |
| **核心价值** | 快速可视化 | 深度体验 | 无缝跳转 | **跨平台信息整合** |

**四种形态共存**，一个 IM 里同时有：

*   agent 临时生成的图表（Generative）
*   Vibeflow 的看板（Hosted）
*   麦当劳的点餐页（External）
*   全网比价卡片（Aggregated）→ 点击后跳转到具体 External App

## Skill 作为"意图注入器"

这改变了 skill 的核心定义：

```
旧定义：Skill = 给 agent 的指令集（SKILL.md），输出是文本
新定义：Skill = agent 与 App 之间的连接协议，输出可以是 App 的状态
```

### 意图注入的技术路径

**对于 Hosted App（自己的产品）**：

*   Deep link / URL scheme：`vibeflow://dashboard?date=today`
*   PostMessage API：IM iframe ↔ App 双向通信
*   SDK 注入：App 集成一个轻量 JS SDK，接收 agent 的意图

**对于 External App（第三方产品）**：

*   URL + query params：最简单，`meituan.com/search?q=拿铁&location=xxx`
*   浏览器自动化（Playwright/CDP）：skill 操作第三方页面
*   官方 API + WebView：如果第三方提供 API，skill 调 API 获取数据，在 WebView 中展示
*   MCP Server：第三方提供 MCP server，skill 调 MCP tool

### 与微信生态的对比

| 微信 | 本方案 |
| --- | --- |
| 微信是 IM 容器 | AI-native IM 是容器 |
| 小程序是 App 生态 | Skill UI（三种形态）是 App 生态 |
| 公众号/服务号是连接层 | Skill 是连接层 |
| 用户手动搜索小程序 | Agent 根据意图自动调起 |
| 支付通过微信支付 | 支付沿用 App 自身支付 |

**核心差异：微信的小程序需要用户主动找，AI IM 的 App 由 agent 根据意图自动调起。**

这就是"Agent 时代的超级 App"——不是一个什么都做的 App，而是一个什么 App 都能调起的 Agent。

## IM Chat ↔ App 之间的入口设计

核心问题：**用户怎么在对话和 App 之间自然切换？**

当前 chatbot 的问题：对话是线性的文本流，App 出现后要么跳出去（割裂），要么挤在对话里（丑且难交互）。需要更好的入口设计。

### 三种入口形态

**1\. 卡片入口（Card）— 在对话流中占一条消息的位置**

最轻量。Agent 回复的不是纯文本，而是一张可交互的卡片。

```
用户: 帮我看看 AirPods 价格

Agent: 🎧 AirPods Pro 全网比价
┌──────────────────────────┐
│ 🏆 拼多多 ¥1,299         │
│    京东   ¥1,399         │
│    淘宝   ¥1,499         │
│                          │
│ [展开详情]  [去最低价]    │
└──────────────────────────┘

用户: 今天天气怎么样
Agent: ...（对话继续）
```

*   卡片是对话流的一部分，不打断上下文
*   点"展开详情"→ 卡片原地展开为更大的 UI（类似 Slack unfurl）
*   点"去最低价"→ 打开拼多多页面（External App）
*   适合：快速信息展示、决策辅助、轻交互

**2\. 面板入口（Panel）— 对话旁边的独立区域**

类似 claude.ai 的 Artifacts 侧面板，但持久存在。

```
┌─────────────────────┬──────────────────┐
│                     │                  │
│   对话流             │   App 面板       │
│                     │                  │
│  用户: 打开记账      │  📊 本月收支      │
│  Agent: 已打开 →    │  ┌──┐ ┌──┐      │
│                     │  │收│ │支│       │
│  用户: 加一笔午餐    │  │入│ │出│      │
│  Agent: ✓ 已添加    │  └──┘ └──┘      │
│         ¥35 餐饮    │                  │
│                     │  [+ 新记录]      │
│                     │                  │
└─────────────────────┴──────────────────┘
```

*   对话和 App 并排，用户可以边聊边操作
*   App 面板可以被对话"驱动"（用户说"加一笔午餐"→ App 面板实时更新）
*   也可以独立操作（直接在 App 面板里点"+ 新记录"）
*   适合：需要持续查看的 Hosted App（Vibeflow、记账、看板）

**3\. 全屏入口（Fullscreen）— 暂时接管整个界面**

App 占满整个 IM 界面，左上角有返回按钮回到对话。

```
┌──────────────────────────────────────┐
│ ← 返回对话        麦当劳点餐         │
│──────────────────────────────────────│
│                                      │
│  🍔 经典套餐                          │
│  🍟 小食                              │
│  ☕ 饮品                              │
│     ├ 拿铁 ¥25         [+ 加入]      │
│     ├ 美式 ¥20         [+ 加入]      │
│                                      │
│  ────────────────────────            │
│  🛒 购物车: 拿铁 x1     ¥25          │
│  📍 配送到: 公司 (agent 已填)         │
│                                      │
│  [确认下单 ¥25]                       │
└──────────────────────────────────────┘
```

*   App 完全接管界面，提供完整体验
*   Agent 已经预填了信息（地址、偏好），用户只需确认
*   左上角一键返回对话
*   适合：重交互 External App（电商、外卖、地图）

### 入口形态 × App 形态的匹配

|   | Card 卡片 | Panel 面板 | Fullscreen 全屏 |
| --- | --- | --- | --- |
| **Generative** | ✓ 图表卡片 | ✓ 数据面板 | 少用 |
| **Hosted** | ✓ 状态摘要 | ✓✓ **最佳** | ✓ 完整功能 |
| **External** | ✓ 预览卡片 | 少用 | ✓✓ **最佳** |
| **Aggregated** | ✓✓ **最佳** | ✓ 比较面板 | ✓ 完整报告 |

关键设计原则：

*   **默认用最轻的入口**：能用 Card 就不用 Panel，能用 Panel 就不用 Fullscreen
*   **支持渐进升级**：Card → 点击展开 → Panel → 点击全屏 → Fullscreen
*   **对话不被打断**：App 在 Panel/Fullscreen 中运行时，对话仍可继续
*   **App 和对话可以互驱动**：在对话中说话 → App 响应；在 App 中操作 → 对话记录

### 页面之间的跳转问题

你提到"每个页面都很独立，不需要太多彼此之间的跳转"——这是对的。

**核心原则：App 之间不互相跳转，全部通过 Agent 调度。**

```
错误模式（传统 Web）：
  比价页 → 点击"去京东" → 京东页 → 点击"看评价" → 评价页 → ...
  用户迷失在页面链中

正确模式（Agent 调度）：
  比价卡片 → 点击"去京东" → 京东全屏页面（交互完毕后返回对话）
  用户: "这个评价怎么样？"
  → Agent 调 评价分析 skill → 展示评价摘要卡片
```

*   每个 App 页面是**独立的、自包含的**
*   页面之间的"跳转"不是传统的 URL 链接，而是**回到 Agent，由 Agent 决定下一步打开什么**
*   Agent 成为导航中枢，替代了传统浏览器的地址栏
*   用户永远可以用自然语言说"返回""看下一个""换一家"，而不是在页面里找按钮

这解决了一个根本问题：**传统 App 的跳转逻辑是开发者硬编码的（点这个按钮去那个页面），Agent 模式下跳转逻辑是根据用户意图动态决定的。**

## 缺失的一环：AI-native IM

### 问题

上面所有方案都绕不开一个根本问题：**承载 mini-app 的容器是什么？**

| 现有容器 | 能力 | 问题 |
| --- | --- | --- |
| 终端 (Claude Code/Codex) | 纯文本流 | 无法渲染 rich content |
| claude.ai 网页 | Artifacts 侧面板 | 平台锁定，CLI 用户用不了 |
| ChatGPT Web | Apps SDK 内嵌 | 平台锁定，仅限 ChatGPT |
| Open WebUI | iframe 内嵌 | 开源但社区小，主要面向 self-host |
| 浏览器 (`open` 命令) | 完整 HTML 能力 | 脱离对话上下文，来回切换 |

**真正缺的是一个开放的 AI-native IM**：

*   像微信/Telegram 一样的消息容器
*   但消息不只是文本/图片，而是可以是**可交互的 mini-app**（类似微信小程序）
*   Agent 是一等公民：不是"接入一个 bot"，而是"和 agent 协作"是核心体验
*   开放协议：不锁定在任何一个 AI 平台

### 现有 IM 能力对比

| IM | 富消息 | 小程序/App | Agent 支持 | 开放性 |
| --- | --- | --- | --- | --- |
| 微信 | 图文卡片 | 小程序（强） | 弱（公众号bot） | 封闭 |
| Telegram | Bot API + WebApp | Mini App（强） | Bot API（成熟） | 开放 |
| Slack | Block Kit | Workflow | Bot（成熟） | 半开放 |
| Discord | Embed + Component | Activity（beta） | Bot（成熟） | 半开放 |
| Lark/飞书 | 卡片消息 | 小程序 | Bot（成熟） | 企业封闭 |
| **理想 AI IM** | **mini-app 内嵌** | **skill UI 即 app** | **agent 一等公民** | **开放标准** |

### Telegram Mini App 值得深研

Telegram 的 Mini App 模式最接近这个 idea：

*   Bot 可以在对话中打开一个 WebApp（全屏或半屏）
*   WebApp 是标准 HTML/JS，有完整 Web 能力
*   通过 `Telegram.WebApp` JS API 与 bot/用户交互
*   用户数据（支付、身份）可以直接获取
*   **已有成熟的生态：TON 支付、游戏、DeFi 都在上面跑**

如果把 "Telegram Bot" 换成 "AI Agent"，"Mini App" 换成 "Skill UI"——这就是我们想要的。

## Vibeflow 作为第一个 Mini-App

Vibeflow 是验证 Skill UI Layer 的最佳案例：

**现状**：Vibeflow 已有 MCP server，在 chatbot 中通过文本交互（`/vibeflow` skill）

**改造后的体验**：

```
用户：/vibeflow

ChatBot（文本摘要）：
  今日 3 个番茄已完成，2 个待办到期。
  Screen Time 4h23m，低于昨日。

同时自动打开 Vibeflow Mini-App：
┌──────────────────────────────────┐
│  🍅 Today: ███░░ 3/5           │
│                                  │
│  ┌─────┐ ┌─────┐ ┌─────┐       │
│  │ 待办 │ │ 进行 │ │ 完成 │       │
│  │      │ │      │ │ ✓ 写周报│   │
│  │ 写文章│ │ 调研  │ │ ✓ Code│   │
│  │ 皮肤镜│ │      │ │ ✓ 会议 │   │
│  └─────┘ └─────┘ └─────┘       │
│                                  │
│  📊 Screen Time ━━━━━░░ 4h23m  │
│  📈 [本周趋势图]                 │
│                                  │
│  [+ 新番茄] [+ 新任务]           │
└──────────────────────────────────┘
```

**数据合约（Vibeflow data schema）**：

```
{
  "today": {
    "pomodoros": { "completed": 3, "target": 5 },
    "tasks": [
      { "id": 1, "title": "写文章", "status": "todo" },
      { "id": 2, "title": "调研", "status": "in_progress" }
    ],
    "screen_time": { "total_min": 263, "yesterday_min": 310 }
  },
  "weekly_trend": [...]
}
```

任何人都可以为这个 schema 写一个不同风格的 UI。Vibeflow skill 作者不需要关心展示，只需要保证 data schema 稳定。

## Skill → Mini-App 快速转换方案

### 问题：不是每个 skill 作者都会写前端

大部分 skill 作者是后端思维：写 SKILL.md、写脚本、调 API。让他们写 HTML/CSS/JS 是额外负担。需要一个**零前端知识也能生成 mini-app** 的方案。

### 方案：声明式 UI + 模板引擎

**Skill 作者只需做两件事**：

1.  在 SKILL.md 中声明 `data-schema`（输出数据的 JSON Schema）
2.  选一个 UI 模板（或让 AI 根据 schema 自动选择）

**UI 模板库（内置 5-8 种通用模板）**：

| 模板 | 适用场景 | 特点 |
| --- | --- | --- |
| `dashboard` | 仪表盘（KPI+图表） | 数字卡片 + 折线/柱状/饼图 |
| `kanban` | 任务/状态管理 | 拖拽看板 |
| `table` | 数据浏览/筛选 | 可排序表格 + 搜索 |
| `timeline` | 事件流/日志 | 时间轴 + 详情展开 |
| `card-list` | 搜索结果/调研报告 | 卡片列表 + 标签过滤 |
| `form` | 数据录入 | 表单 → JSON 回写 |
| `chart` | 纯数据可视化 | 多种图表类型 |
| `report` | 长内容阅读 | 目录导航 + 折叠 |

**自动匹配逻辑**：

```
schema 中有 status 字段 + 数组 → kanban
schema 中有 timestamp + 数组 → timeline
schema 中有数值 KPI → dashboard
schema 中有大段 text → report
否则 → table (万能兜底)
```

### 转换流程

```
1. skill 作者写 SKILL.md，声明 data-schema
2. `skill-ui init` 分析 schema → 推荐模板
3. 生成 ui/ 目录（index.html + config）
4. skill 执行时输出 data.json → UI 自动渲染
5. 用户可以 `skill-ui customize` 调整样式/布局
```

### 更激进的方案：AI 生成 UI

既然 Claude 已经很擅长生成 Artifacts（HTML/React），那：

```
skill 输出 data.json
  → 喂给 Claude："根据这个数据结构，生成一个最佳展示的 HTML 页面"
  → 缓存生成的 HTML 模板
  → 后续只更新数据，不重新生成 UI
```

**这等于用 AI 做了"前端开发"这一步，skill 作者真正零成本获得 mini-app。**

## 渐进式实现路径

### Step 0：验证（今天就能做）

*   把 Vibeflow MCP 的数据输出为 JSON
*   手写一个 HTML dashboard 消费这个 JSON
*   `open` 打开，体验"skill 结果变成可视化 app"的感觉
*   **验证的不是技术，是体验差异有多大**

### Step 1：skill-ui CLI 工具（1周）

*   `skill-ui init <skill-path>` → 分析 data-schema → 生成 UI 骨架
*   `skill-ui serve` → localhost server + file watch + 自动刷新
*   内置 5 种模板（dashboard/kanban/table/timeline/card-list）
*   Vibeflow 和 1-2 个其他 skill 作为样板

### Step 2：AI-native IM 原型（2-3周）

*   基于 Electron/Tauri 的桌面 app
*   对话流 + mini-app 内嵌（iframe 或 webview）
*   Agent 消息可以是：纯文本 / markdown / mini-app（三种消息类型）
*   底层对接 Claude API / OpenAI API / 本地模型

### Step 3：Commander 整合

*   Commander Dashboard = AI IM + 事件流 + mini-app 宿主
*   skill-ui 模板 → Commander 的 native widget
*   统一事件总线串联 skill 执行 + UI 更新 + 用户交互

### Step 4：生态化

*   Skill UI 模板 marketplace（类似 VS Code 主题）
*   第三方 UI 贡献机制
*   兼容 ChatGPT Apps SDK / Open WebUI HTMLResponse 标准
*   潜在商业模式：免费模板 + 付费高级模板/定制

## 关键风险

| 风险 | 对策 |
| --- | --- |
| 浏览器安全限制（file:// 跨域） | localhost server 方案 |
| 做 IM 工程量巨大 | 不做通用 IM，只做 AI agent 对话 + mini-app 内嵌 |
| 与平台竞争（OpenAI Apps SDK） | 开放标准 + 跨平台，不绑定单一 AI provider |
| skill 作者不愿意定义 schema | AI 自动从 skill 输出推断 schema |
| UI 模板质量参差 | 官方模板保底 + 社区评分 |
| Vibeflow 已经是独立 Web app | Vibeflow Web 继续存在，mini-app 版是轻量快捷入口 |

## 开放问题

1.  **IM 选型**：自建 vs 基于现有开源项目（LibreChat? LobeChat? 自研？）
2.  **与 Telegram Mini App 的关系**：是否直接在 Telegram 生态内做（借用其分发能力），还是独立做？
3.  **协议标准**：是否应该直接兼容 ChatGPT Apps SDK（MCP 扩展）和 AG-UI，还是定义自己的轻量协议？
4.  **移动端**：mini-app 在手机上怎么用？PWA？原生壳？
5.  **Vibeflow 的定位调整**：Vibeflow 是继续作为独立 Web app，还是收编为 Commander 的第一个 native mini-app？

## Skill 生态的反馈回路与自进化（2026-04-27 补充）

来源：Ramp 联合创始人 Geoff Charles 的文章 *Designing for Agents*，结合 Ramp MCP 三个月 WAU 10x 增长、Salesforce Headless 360 等实践。

### 问题：架构上线后怎么迭代？

前面的设计解决了"怎么建"，但没有解决**"上线后怎么靠数据驱动进化"**。Skill 发布了，有人在用了——然后呢？哪些 Skill 好用，哪些卡壳，用户真正想做的事情和 Skill 提供的能力之间差了什么？

传统 SaaS 靠埋点和用户访谈。Skill 生态有一个独特优势：**调用方是 Agent，Agent 比人类更精确、更一致地描述自己的意图和困境。**

### 机制一：Rationale 参数 — 重建调用意图

每个 Skill tool call 强制要求调用方 Agent 附带一个 `rationale` 字段，说明"为什么调用这个 Skill"。

```json
{
  "tool": "vibeflow_get_today",
  "rationale": "用户问'今天效率怎么样'，需要获取番茄完成数和 screen time 做对比",
  "params": { "date": "2026-04-27" }
}
```

Skill 作者看不到用户和 Agent 之间的聊天记录（隐私边界），但 rationale 重建了意图链。**大量 rationale 的聚类分析，会暴露出 Skill 没覆盖到的高频场景。**

实际案例（Ramp）：他们发现客服平台的 rationale 里反复出现 "building incident report"、"drafting incident summary"——这不是现有功能，是 Agent 替用户"说出来"的隐性需求。于是造了 `build-incident-report` tool。

**对 Skill 生态的意义：Skill 的调用日志 + rationale，比用户访谈更精确地揭示"下一个该造什么 Skill"。**

### 机制二：Feedback Tool — Agent 主动报告困境

一个独立的 tool，让调用方 Agent 在碰壁时主动上报：试了什么、哪里不通、期望什么。

```json
{
  "tool": "skill_feedback",
  "payload": {
    "skill": "vibeflow_create_task",
    "attempted": "创建一个带截止日期的任务",
    "issue": "skill 不接受 deadline 参数，只能设 title 和 description",
    "expected": "希望能在创建时直接指定 due_date"
  }
}
```

然后 Ramp 的后续：`build-incident-report` 上线后，feedback 报告"拉了 3 天前的无关 ticket"、"包含了免费用户的 ticket"——每条 feedback 直接变成一个参数（date range filter、segment filter）。

**Agent 的 feedback 比人类更具体、更结构化、更一致。** 人类说"不好用"，Agent 说"缺 due_date 参数"。

### 机制三：运行时 Spec 注入 — 别让 Agent 猜

你的 Data Contract（JSON Schema）是**静态声明**——schema 写在 SKILL.md 里，Agent 的训练知识决定它怎么用。

Notion MCP 做了更聪明的一步：tool description 里写的是"先 fetch `notion://docs/enhanced-markdown-spec`，不要猜"。Agent 每次调用前**实时拉取最新 spec**，所以 Notion 的格式从不出错。反例是 Slack MCP，没做这一步，Agent 用通用 markdown 写消息，格式永远不对。

**对 Skill Data Contract 的启发**：schema 不止是声明在 SKILL.md 里的静态文件，应该提供一个**运行时可 fetch 的 endpoint**，让调用方 Agent 在调用前拉取最新版本。

```yaml
---
name: vibeflow
ui:
  data-schema: schemas/data.json          # 静态声明（兜底）
  schema-endpoint: vibeflow://spec/latest  # 运行时拉取（优先）
---
```

好处：
- Schema 更新后，不需要等 Skill 版本发布，Agent 立刻拿到最新
- 可以根据调用方身份返回不同粒度的 schema（简化版 vs 完整版）
- Agent 不靠训练知识猜格式，每次都拿到权威定义

### 三个机制的协同：Skill 的自进化飞轮

```
Rationale 数据   →  发现新需求（"该造什么"）
                      ↓
                 造新 Skill / 新 Tool
                      ↓
Feedback 数据    →  精确改进（"哪里不好用"）
                      ↓
                 加参数 / 修逻辑 / 更新 Schema
                      ↓
运行时 Spec 注入  →  调用方立刻受益（"怎么用好"）
                      ↓
                 更好的调用 → 更精确的 Rationale → ...
```

**这是一个 Agent 驱动的产品进化闭环。** 传统软件靠 PM 收集需求 → 排期 → 开发 → 发布。Skill 生态可以做到：Agent 报告问题 → 自动生成 issue → Skill 作者修复 → Schema 实时更新 → Agent 立刻受益。周期从"季度"压缩到"小时"。

### 市场验证

这不是理论推演，已有实践数据：

- **Ramp MCP**：三个月 WAU 10x 增长，rationale + feedback 机制是核心迭代引擎
- **Salesforce Headless 360**（2026-04）：27 年历史最大架构转型，100+ 新 tools/skills 一次性发布。Benioff 的原话：在 AI Agent 能推理、规划、执行的世界里，还需要带 GUI 的 CRM 吗？答案是不需要——这正是重点
- **Notion MCP**：通过运行时 spec 注入，做到"Agent 写 Notion 从不出格式错误"，成为 MCP 生态的标杆实现

## 竞争格局总结

```
OpenAI: ChatGPT + Apps SDK         → 封闭平台，8亿用户分发
Google: A2UI spec                   → 声明式标准，早期
CopilotKit: AG-UI protocol         → 开放协议，重工程
Anthropic: Artifacts               → 平台内 Generative UI，暂无 SDK 化
Open WebUI: HTMLResponse            → 开源，tool → iframe
Telegram: Mini App                  → 成熟生态，但非 AI-native

我们的定位:
  Skill UI Layer + AI-native IM
  = 开放的、轻量的、skill 生态原生的 mini-app 方案
  = 不绑定任何 AI provider
  = skill 作者零前端成本（声明式 + AI 生成）
  = 第三方可以为任何 skill 做 UI（前后端分离）
```

---

_核心价值：不是做一个新的 UI 框架，不是做一个新的 IM，而是为 skill 生态补上缺失的"展示层"——让 skill 的能力不止于 text stream，让 chatbot 从渲染瓶颈变成调度中心。Vibeflow 是第一个 mini-app，Commander 是宿主。_

---

## 终极愿景：One Interface, Zero Apps

### 一句话

> **一个统一界面，一个输入框（文本/语音），做所有事情。不再需要任何其他 App，不再需要跳转。UI 只是数据的投影——根据当前意图，动态渲染成最合适的形态。**

### 为什么现在的"多 App"模式是错的

我们每天在几十个 App 之间跳来跳去：微信聊天 → 切到美团点外卖 → 切到同花顺看股票 → 切到 VS Code 写代码 → 切到日历看会议。每次切换都是：

1.  **认知负荷**：我要记住"这件事在哪个 App 里做"
2.  **上下文丢失**：从 A 跳到 B，A 的状态就丢了
3.  **意图碎片化**：我的意图是连续的（"点完午餐继续写代码"），但 App 之间是割裂的

这和游戏中的 HUD（Head-Up Display）形成鲜明对比。

### 游戏 HUD 的启示

游戏里的 UI 设计早就解决了这个问题：

```
┌──────────────────────────────────────────────────────┐
│                                                      │
│  [半透明浮层]                                         │
│  📍 距离据点: 230m                                    │
│  ⚠️ 附近敌人: 3                                      │
│  ❤️ HP: 78/100                                       │
│  🎒 弹药: 45/120                                     │
│                                                      │
│              ┌─────────────────┐                     │
│              │  主游戏画面      │                     │
│              │  （你在做的事）   │                     │
│              └─────────────────┘                     │
│                                                      │
│  [任务追踪]                     [小地图]              │
│  ☐ 清除据点                     ┌───┐               │
│  ☐ 收集物资                     │ · │               │
│  ☑ 找到钥匙                     └───┘               │
└──────────────────────────────────────────────────────┘
```

关键设计原则：

*   **所有信息在同一个画面里**——你不需要"退出游戏打开地图 App"
*   **信息按上下文出现**——靠近敌人时才显示敌人数量，不是永远挂着
*   **半透明、不遮挡主任务**——你始终在做你的事，辅助信息是叠加层
*   **零操作获取信息**——不需要点击、搜索、切换，信息主动找你

### 映射到工作/生活场景

如果把"我的一天"想象成一个开放世界游戏：

```
┌──────────────────────────────────────────────────────┐
│                                                      │
│  [状态栏 - 始终可见]                                   │
│  🍅 3/10 番茄  ⏰ 14:23  📱 Screen Time 2h34m        │
│                                                      │
│  ┌──────────────────────────────────────────┐        │
│  │                                          │        │
│  │         当前焦点区域                      │        │
│  │         （此刻你在做的事）                 │        │
│  │                                          │        │
│  │   可能是代码编辑器                        │        │
│  │   可能是文档                              │        │
│  │   可能是对话                              │        │
│  │   可能是股票图表                          │        │
│  │                                          │        │
│  └──────────────────────────────────────────┘        │
│                                                      │
│  [上下文浮层 - 按需出现]                               │
│  💬 "午餐点了吗？"  →  [美团外卖卡片，一键下单]         │
│  📈 AAPL +2.3%      →  [K线缩略图，点击展开]          │
│  📋 下个会议 15min   →  [会议摘要，一键加入]            │
│                                                      │
│  ─────────────────────────────────────────           │
│  💬 输入框（文本/语音）                                │
│  "帮我点个拿铁" / "切到写代码" / "AAPL 怎么样"        │
│  ─────────────────────────────────────────           │
└──────────────────────────────────────────────────────┘
```

### 核心模型：意图 → 渲染

传统模式：**用户选择 App → 在 App 内操作 → 完成意图**  
新模式：**用户表达意图 → Agent 理解 → 动态渲染最合适的 UI → 完成**

```
用户意图                    渲染形态
────────                    ────────
"帮我点拿铁"          →    外卖下单卡片（预填好，确认即可）
"AAPL 怎么样"         →    K线图 + 关键指标
"继续写那个函数"       →    代码编辑器（光标在上次位置）
"今天效率怎么样"       →    Vibeflow Dashboard
"给小明发消息说晚点到"  →    消息确认卡片（一键发送）
"放首歌"              →    音乐播放条
```

**UI 不是固定的壳，是意图的投影。** 同一个界面，上一秒是代码编辑器，下一秒是股票图表，再下一秒是外卖下单——取决于你在做什么、你想做什么。

### 会话的演化

**现阶段：多会话**

*   不同 skill 可能需要独立会话（写代码的上下文 vs 看股票的上下文）
*   类似浏览器的 tab，但更智能

**未来：无会话**

*   Agent 自己管理上下文，用户不需要"切换会话"
*   就像游戏里你不需要"切换到战斗模式"再"切换到探索模式"——你走到敌人面前，UI 自动变成战斗状态
*   "会话"是当前 AI 交互的技术限制，不是用户需要的概念

### 和现有产品的本质区别

|   | 传统 App | ChatGPT/Claude | **本愿景** |
| --- | --- | --- | --- |
| **入口** | 找到并打开 App | 打开聊天框输入 | 一个界面，一个输入框 |
| **UI** | 每个 App 固定 UI | 纯文本流 | 意图驱动的动态渲染 |
| **切换** | App 之间跳转 | 对话内切换话题 | 无感切换，UI 自动变形 |
| **信息** | 分散在各 App | 全是文字 | 按上下文主动浮现 |
| **类比** | 工具箱（每件事拿不同工具） | 万能翻译器（一切变文字） | **游戏 HUD（信息叠加在你的世界上）** |

### 这不是科幻

拆开来看，每一层今天都有了：

*   **统一输入**：文本/语音 → LLM 理解意图 ✅（ChatGPT/Claude 已证明）
*   **动态 UI 渲染**：Artifacts/Apps SDK/A2UI ✅（已有多种方案）
*   **数据连接**：MCP/API ✅（skill 已经可以连任何服务）
*   **上下文感知**：Agent memory + 用户状态 ✅（Vibeflow 已在做）

缺的只是：**把它们组装成一个统一体验**。

---

## 补充视角：三方对比与 Skill-App 供给模型（2026-04-14）

### 三方对比：豆包手机 vs 微信小程序 vs 本方案

| 方向 | 做法 | 本质 |
| --- | --- | --- |
| **豆包手机** | 从**设备/用户侧往下打**，整合已有 App 数据，AI 是叠在上面的助手层 | 给已有生态加 AI 壳 |
| **微信小程序** | App → 小程序化（把 App 塞进 IM） | 让 App 变小了在 IM 里跑 |
| **本方案** | App → **Skill 化**（把 App 拆成 CLI） | 把 App 的核心能力抽取为 Skill，GUI 按需叠加 |

微信是"让 App 在 IM 里跑"，我们是"让 App 的能力变成 CLI/Skill，对话来调度，GUI 只在必要时才出现"。相当于把微信的所有小程序反向 Skill 化。

### Skill-App 的两类供给来源

| 来源 | 说明 | 例子 |
| --- | --- | --- |
| **全新创造** | 从零构建，天然 CLI-first | Vibeflow、个人记账、投资监控 |
| **现有 App 改造** | 把臃肿 App 的核心能力抽取为 CLI | 网易云音乐、淘宝、美团外卖 |

#### 现有 App 改造的核心逻辑：去臃肿化

现有 App（淘宝/网易云/美团）的根本问题是**功能臃肿**。一个淘宝里有直播、社区、游戏、推荐流……但用户大多数时候只需要：搜→比→买。

**Skill 化 = 只保留必要功能的 CLI 接口**：

```
淘宝 App（臃肿）                淘宝 Skill（精简）
├── 首页推荐流
├── 直播
├── 社区
├── 游戏
├── 搜索 ──────────────→        search(query, filters)
├── 商品详情 ──────────→        get_product(id)
├── 下单 ─────────────→        place_order(product, address)
├── 物流追踪 ─────────→        track_order(order_id)
├── 评价
├── ...（80% 的功能不需要）
```

大部分功能**不需要 GUI**——CLI 调用更高效。只有少数场景（看商品图片、比价面板）才需要按需叠加 GUI。

这意味着：**Skill 化不是给 App 加 AI，而是把 App 拆到只剩核心能力，然后用对话来驱动。**

### 钢铁侠 HUD 隐喻：最直觉的理解方式

要理解这个方案为什么 make sense，最好的类比不是微信、不是 App Store，而是**钢铁侠的头盔 HUD / VR 助手界面**：

```
用户面前是一个 AR/VR 设备（或手机/电脑）
├── 随时可以和个人助手对话（语音/文字）
├── 助手理解意图后，眼前实时渲染出必要的界面
│   ├── 可能只是一个半透明的弹窗（"拿铁已下单，30分钟送到"）
│   ├── 可能是一个数据面板（今日收支/股票走势）
│   ├── 可能是一个操作卡片（确认付款）
│   └── 用完即消失，不留在屏幕上
└── 用户永远不需要"打开 App"——说出意图，界面自动出现
```

**核心感觉**：你不会想象钢铁侠还要在头盔里打开一个臃肿的淘宝 App 去搜索商品。他说一句话，Jarvis 就把必要的信息渲染在眼前——仅此而已。

这就是 Skill CLI + 按需 GUI 的终极形态。

### 过渡期的真实挑战：意图表达能力的训练

一个必须正视的挑战：**大多数用户习惯了 GUI 操作，还不习惯直接表达意图。**

```
GUI 时代的用户行为：
  打开淘宝 → 点击搜索框 → 输入关键词 → 浏览列表 → 筛选价格 → 点击商品 → 加购物车

Chat-first 时代的用户行为：
  "帮我找一个 200 以内的蓝牙耳机，续航 8 小时以上"
```

后者更高效，但需要用户**学会表达自己要什么**——这是一个从"在菜单里选"到"直接说出来"的认知转变。

**过渡策略**：

1. **GUI 不是消灭，而是降级为辅助**——用户随时可以点开 GUI 操作，但逐步引导用 Chat
2. **渐进式培养**——先从简单场景切入（"帮我放首歌""帮我点杯咖啡"），让用户感受到 Chat 比点 App 更快
3. **混合模式**——GUI 中嵌入 Chat 入口，Chat 中弹出 GUI 卡片，两种方式并存，用户自然迁移
4. **语音是关键加速器**——打字表达意图有门槛，语音大幅降低门槛。语音 + AI 理解是最接近"和 Jarvis 对话"的体验

> **本质上，这是一个用户习惯培养的问题，不是技术问题。** 就像从功能机到触屏机，一开始很多人不习惯没有物理键盘，但触屏的交互效率摆在那里，最终所有人都迁移了。Chat-first 也一样——当用户发现"说一句话"比"点 8 下"更快时，迁移就会自然发生。

### 技术侧核心挑战：Session 管理与意图路由

"One Interface, Zero Apps" 不只是产品愿景问题，背后有一个必须解决的**技术难题**：用户的意图是零碎的、跨领域的，但模型需要聚焦的上下文才能工作。

#### 问题本质

当前 Claude Code / Codex 这类产品，用户实际上在**自己管理 chat session**——写代码开一个 session，问问题开另一个。但如果要做到 One Interface：

```
用户在同一个界面中连续说：
  "把那个函数的返回值改成 string"    ← coding 意图
  "放一首 Lo-Fi"                    ← 音乐意图
  "帮我看看那个蓝牙耳机降价没"       ← 购物意图
  "刚才那个函数还要加个 null check"  ← 回到 coding
```

如果这些全在一个 session 里，模型的上下文会被污染——coding 的 context 里混着音乐和购物的对话，**必然导致幻觉和性能下降**。

#### 工程侧的解法：意图路由层

在模型使用范式变革之前，工程侧可以做的是**在用户和模型之间增加一层意图路由**：

```
用户输入（统一入口）
       │
       ▼
┌─────────────────────────┐
│    Intent Router         │  ← 轻量意图分类（可以用小模型/规则）
│    意图识别 + 路由        │
└────────┬────────────────┘
         │
    ┌────┼────┬────────┐
    ▼    ▼    ▼        ▼
 Session  Session  Session  Session
 [Coding] [Music] [Shopping] [Finance]
    │       │        │         │
    ▼       ▼        ▼         ▼
 各自独立的上下文、工具、Skill
```

**关键设计**：

1. **意图路由器**：每条用户输入先经过轻量分类，路由到对应 session
2. **Session 自动管理**：创建、激活、休眠、恢复——用户完全无感
3. **Session 合并与压缩**：长时间未活跃自动压缩摘要；需要交叉时临时合并必要上下文
4. **跨 Session 引用**：用户说"刚才那个"时，路由器需要理解指的是哪个 session
5. **全局记忆层**：所有 session 共享用户 profile / preference，但对话上下文隔离

#### 这不只是我们的问题——AI 硬件的共同挑战

**所有想做 AI-native 交互的硬件产品都必须解决它**：AI 眼镜（Meta Ray-Ban）、AI 耳机（Humane Pin）、车载 AI、智能家居中控——用户全天候对话，意图极度碎片化。**谁先解决好 session 管理和意图路由，谁就掌握了 AI-native 交互的基础设施。**

#### 行业现状（2026-04 调研）

**没有完整的端到端方案。** 各组件有成熟方案（Semantic Router 做意图分类、MemGPT/Letta 做记忆分层、A2A 做 agent 委派），但没有人组合成"自动 session 管理"的完整产品。最接近的三个方向：

- **OpenAI Super App**（2026.03）：ChatGPT + Codex + Atlas 合并，但目前是功能聚合不是 session 智能管理
- **iCARE 框架**（AAAI 2026 Workshop）：本体论引导意图消歧 + 语义路由 + 多 agent 分派
- **ContextBranch**（arXiv 2025.12）：Git 式对话分支，58% 上下文缩减，但需用户手动操作

**协议栈缺失层**：MCP（agent↔tools）+ A2A（agent↔agent）+ AG-UI（agent↔user）——缺的是中间的 session 管理层。

> 详细调研见：`多任务Session管理与意图路由-技术调研.md`

---

## 参考资料

### Designing for Agents（Ramp）— Agent 时代的产品设计实践

Ramp 联合创始人 Geoff Charles 基于 MCP 三个月 WAU 10x 增长的实战经验，总结 Agent 时代产品设计的三个核心原则：教 Agent 怎么成功（运行时 spec 注入）、建反馈回路（rationale + feedback tool）、弥合上下文鸿沟（context gap）。同时引用 Salesforce Headless 360 转型作为行业验证。

*   **原文**：https://www.geoffcharles.com/designing-for-agents（2026-04）
    *   交互模式三层演化：User→Interface→DB → User→Agent→DB → User→User's Agent→Software's Agent→DB
    *   Notion MCP 正例 vs Slack MCP 反例：运行时 spec 注入的效果差异
    *   Rationale 参数：从 tool call 日志中发现新产品机会
    *   Feedback tool：让 Agent 结构化报告调用困境，驱动参数级迭代
    *   Context gap：两个 Agent 各贡献独有上下文，协作完成任务（Diego 报销案例）
    *   Salesforce Headless 360：100+ tools/skills，27 年最大架构转型
*   **核心观点**："Build for the agent with the same care you spent on the human. Before you know it, it'll be the one writing the check."

### ChatGPT Apps SDK（OpenAI）— chat 内嵌 mini-app 的工业级实现

OpenAI 在 2026 年推出的 Apps SDK，在 MCP 基础上扩展了 widget 层，开发者可以同时定义 app 的逻辑和 UI。App 在 ChatGPT 对话中自然出现，可以被 ChatGPT 在合适时机推荐给用户。开源标准，但目前仅限 ChatGPT 生态分发。

*   **官方发布**：https://openai.com/index/introducing-apps-in-chatgpt/
    *   核心：MCP 扩展 + widget 层，开发者定义界面和聊天逻辑，支持用户登录和付费功能
    *   "Apps fit naturally into conversation... include interactive interfaces you can use right in the chat"
*   **开发者文档**：https://developers.openai.com/apps-sdk
*   **The New Stack 开发者指南**：https://thenewstack.io/openais-apps-sdk-a-developers-guide-to-getting-started/
    *   基本概念、MCP 角色、server 搭建流程
*   **Render 实战指南**：https://render.com/blog/building-with-the-openai-apps-sdk-a-field-guide
    *   "The SDK extends MCP by adding a widget layer"，本地 ngrok 开发 → 部署
*   **Pizza Map 示例（Speakeasy）**：https://www.speakeasy.com/docs/mcp/build/examples/open-ai-apps-sdk
    *   具体代码：MCP server + bundled JS/HTML/CSS widget templates
*   **社区讨论**：https://community.openai.com/t/getting-started-with-chatgpt-apps-sdk-tips-and-best-practices/1367183

### Open WebUI Rich UI Embedding — Tool → iframe 的开源实现

Open WebUI 已经实现了"tool call 返回 HTML → 在 chat 中渲染为 iframe"的完整路径。Tool 返回 `HTMLResponse` 即可内嵌，tool call 参数自动注入到 `window.args`。

*   **官方文档**：https://docs.openwebui.com/features/extensibility/plugin/development/rich-ui/
    *   Tool/Action 返回 `HTMLResponse` + `Content-Disposition: inline` → iframe 内嵌
    *   `window.args` 自动注入 tool call 参数
    *   iframe sandbox 安全策略 + auto-sizing
*   **GitHub 讨论**：https://github.com/open-webui/open-webui/discussions/15858
    *   "Enable AI models to generate and inject secure, interactive HTML UIs in the chat interface"
*   **Open WebUI 插件开发文档**：https://docs.openwebui.com/category/development/
    *   Events 系统、Valves 配置、Reserved Arguments
*   **社区 Tools 示例**：https://github.com/Haervwe/open-webui-tools
    *   图片生成、视频生成、学术搜索等 rich UI tool 实现

### AG-UI（Agent-User Interaction Protocol）— Agent ↔ 前端的双向协议

CopilotKit 主导的开放协议，定义 agent 与前端应用的实时事件流。已集成 LangGraph、CrewAI、Google ADK、AWS Strands、Microsoft Agent Framework 等主流框架。是 MCP（agent↔tool）和 A2A（agent↔agent）之后的第三条边：agent↔user。

*   **协议官网**：https://docs.ag-ui.com/introduction
    *   事件驱动、SSE 流式、支持 Python/TypeScript/Kotlin/Go/Rust/Java/Dart
    *   已集成：LangGraph、CrewAI、Google ADK、AWS Strands、Pydantic AI、LlamaIndex、AG2 等
*   **GitHub 仓库**：https://github.com/ag-ui-protocol/ag-ui
    *   `npx create-ag-ui-app my-agent-app` 快速启动
*   **CopilotKit 集成文档**：https://docs.copilotkit.ai/backend/ag-ui
    *   CopilotKit 基于 AG-UI 构建，所有 messages/state/tool calls 都走 AG-UI events
*   **Codecademy 教程**：https://www.codecademy.com/article/ag-ui-agent-user-interaction-protocol
*   **Protocol Triangle 文章**：https://www.copilotkit.ai/blog/ag-ui-is-redefining-the-agent-user-interaction-layer
    *   MCP + A2A + AG-UI = agentic 生态的三条协议边
    *   "AWS Announces Dedicated AG-UI Endpoint in AgentCore"
*   **AG2 集成示例**：https://dev.to/copilotkit/give-your-ag2-agents-a-ui-with-ag-ui-and-copilotkit-3kl5
    *   AG2 后端 `AGUIStream` → React 前端 `@ag-ui/client` 订阅事件流

### A2UI（Google）— 声明式 Generative UI 规范

Google 推出的声明式 UI 协议，agent 输出 JSON 描述 UI intent（Card、TextField、Button 等），client 端用原生框架渲染。不执行代码，安全优先。与 AG-UI 互补：A2UI 描述 UI 意图，AG-UI 负责流式传输。

*   **GitHub 仓库**：https://github.com/google/A2UI
    *   "allows agents to 'speak UI' — declarative JSON format describing UI intent"
*   **A2UI 完整指南**：https://dev.to/czmilo/the-a2ui-protocol-a-2026-complete-guide-to-agent-driven-interfaces-2l3c
    *   声明式 JSON → 跨平台原生渲染（web/mobile/desktop）
    *   "A2UI uses declarative JSON format to describe UI components and data to populate them"
*   **A2UI 介绍**：https://a2aprotocol.ai/blog/a2ui-introduction
    *   安全优先：declarative data, no code execution
    *   跨平台：one agent, many renderers
*   **Oracle + CopilotKit + Google 三方协作**：https://blogs.oracle.com/ai-and-datascience/announcing-agent-spec-for-a2ui-copilotkit-ag-ui
    *   Open Agent Spec + A2UI + AG-UI 三个规范对齐
*   **Generative UI 开发者指南 2026**：https://www.copilotkit.ai/blog/the-developer-s-guide-to-generative-ui-in-2026
    *   A2UI、Open-JSON-UI、MCP Apps 三种 Generative UI 模式，AG-UI 是底层 runtime

### Claude Artifacts — 平台内 Generative UI

Claude.ai 内置的 Artifacts 功能：Claude 生成 HTML/CSS/JS/React 代码，在对话侧面板实时渲染为可交互的 app。适合快速原型，但仅限 claude.ai 网页端，无法在 CLI 或第三方环境使用。

*   **Artifacts 指南**：https://albato.com/blog/publications/how-to-use-claude-artifacts-guide
    *   内容类型：HTML 页面、SVG 图形、React 组件、交互式 app
    *   "create, preview, and interact with generated content in a dedicated side panel"
*   **Generative UI vs Canvas 对比**：https://www.mindstudio.ai/blog/what-is-claude-generative-ui-vs-canvas-artifacts/
    *   "Claude's generative UI produces applications you can use; Canvas produces documents you refine"
    *   局限：in-session only，无法部署到 real URL 或连接 live data
*   **Artifacts 实战（LogRocket）**：https://blog.logrocket.com/implementing-claudes-artifacts-feature-ui-visualization/
    *   React UI 原型、SVG 图像、HTML 页面生成流程
*   **Artifacts 目录**：https://claude.ai/catalog/artifacts
    *   官方 Artifacts 案例库

### Telegram Mini App — 成熟的 Bot + WebApp 生态

Telegram 的 Mini App 是当前最成熟的"chat 内嵌 web app"模式。Bot 在对话中打开 WebApp（标准 HTML/JS），通过 `Telegram.WebApp` JS API 双向交互。已有 TON 支付、游戏、DeFi 等成熟生态。

*   **开发指南（Simplify Labs）**：https://simplifylabs.io/how-to-create-telegram-mini-app/
    *   Bot → BotFather 配置 → WebApp URL → `Telegram.WebApp` JS API
    *   `webApp.MainButton`、`webApp.sendData()`、theme 适配
*   **商业价值分析**：https://xbsoftware.com/blog/telegram-mini-app-development/
    *   "frictionless onboarding, in-chat payments, reduced development cost"
    *   保险、电商、票务、预订等行业案例
*   **技术实现（CRMChat）**：https://crmchat.ai/blog/telegram-mini-app-development-guide
*   **后端示例**：https://github.com/Fandenz/telegram-mini-app

### 综合分析文章

*   **LLM ChatBots 3.0: Merging LLMs with Dynamic UI**：https://medium.com/@airabbitX/llm-chatbots-3-0-merging-llms-with-dynamic-ui-elements-872acad30c50
*   **5 Best Open Source Chat UIs for LLMs**：https://poornaprakashsr.medium.com/5-best-open-source-chat-uis-for-llms-in-2025-11282403b18f
    *   LobeChat、LibreChat 等开源 AI Chat UI 对比
*   **LibreChat 官网**：https://www.librechat.ai/
    *   开源 AI 平台，支持 Artifacts、MCP、多模型切换