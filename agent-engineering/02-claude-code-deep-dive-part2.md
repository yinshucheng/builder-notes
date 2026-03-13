# Claude Code 深度解析（下）：MCP、Skills 与上下文工程

> **定位**：面向工程师的深度技术分享
>
> **前置阅读**：[Claude Code 深度解析（上）](01-claude-code-deep-dive-part1.md)
>
> **Author**: [Shucheng Yin](https://github.com/yinshucheng)（尹树成）
>
> **Date**: 2026-01

---

## 1. MCP：AI 时代的 USB 接口

### 1.1 原理

**MCP (Model Context Protocol)** 让 Agent 能连接任意外部系统。

**核心概念：**
* **Tools**：Agent 可以调用的能力（如 `read_file`, `query_db`）
* **Resources**：Agent 可以读取的数据（如数据库 Schema、日志）

### MCP 与传统 API 的对比

| 维度 | 传统 API 调用 | MCP Tool 调用 |
|:--|:--|:--|
| **服务发现** | 服务注册到注册中心 | Tool 定义注入到 `tools` 字段 |
| **调用决策** | 业务代码的 if/else 规则 | LLM 根据语义理解自主决定 |
| **参数构造** | 代码硬编码或模板填充 | LLM 根据上下文自动构造 |
| **调用编排** | 开发者预设流程 | LLM 动态决定调用顺序和组合 |

MCP 的 Tool 注入相当于"服务发现"，但调用编排从"确定性代码"变成了"LLM 语义决策"。

* `description` 就是 Prompt，写给 LLM 看，越清楚调用越准
* `input_schema` 约束参数格式，LLM 会严格遵守
* Tool 结果作为 `tool_result` 回到对话，形成闭环

### 工具的统一抽象

从 Agent 视角，**所有工具都是平等的**：

```json
{
  "tools": [
    // 内置工具
    {"name": "bash", "description": "执行shell命令", "input_schema": {}},
    {"name": "read_file", "description": "读取文件", "input_schema": {}},
    // MCP 提供的工具
    {"name": "query_db", "description": "查询数据库", "input_schema": {}},
    {"name": "search_nearby_food", "description": "搜索附近美食", "input_schema": {}}
  ]
}
```

**Agent 不知道也不关心**：
* 工具是内置的还是 MCP 提供的
* 工具的实现语言（Python/Node/Rust）
* 工具运行在本地还是远程

**设计优势**：扩展能力与内置能力无差别，Agent 可以无缝使用任何工具。

### 1.2 MCP 实战案例

#### 案例 1：自然语言查附近美食

```python
# mcp_food.py
@server.tool("search_nearby_food")
def search_nearby_food(latitude: float, longitude: float, keyword: str = None) -> str:
    """
    搜索附近美食商家
    - latitude/longitude: 用户位置坐标
    - keyword: 搜索关键词（如"火锅"、"奶茶"）
    返回：商家列表，包含名称、评分、距离、人均价格
    """
    return food_api.search(latitude, longitude, keyword)
```

使用效果：

```
> 我在望京 SOHO，中午吃什么好？

Agent: [调用 search_nearby_food] latitude=39.99, longitude=116.48
Agent: 推荐以下午餐：
1. 🍜 西贝莜面村 - 评分 4.8 | 500m | ¥68
2. 🍱 大米先生 - 评分 4.6 | 300m | ¥28
3. 🥗 Wagas - 评分 4.7 | 800m | ¥55
```

#### 案例 2：任务管理 MCP

```python
# mcp_tasks.py
@server.tool("get_today_task")
def get_today_task() -> str:
    """获取今日任务"""
    return task_api.get_today_task()

@server.tool("start_pomodoro")
def start_pomodoro(task_id: str, duration: int = 25) -> str:
    """为指定任务启动番茄钟"""
    return task_api.start_pomodoro(task_id, duration)
```

用自然语言管理待办事项：

```
> 今天有什么任务？
Agent: [调用 get_today_task] ...

> 开始第一个任务的番茄钟
Agent: [调用 start_pomodoro] 25分钟番茄钟已启动 ✅
```

---

## 2. Skills：SOP 的代码化

### 2.1 原理

**Skills** 是封装好的 Prompt + Workflow，把复杂任务拆解为 Agent 能理解的步骤。

关键特性：
* **可组合性**：模块化设计，支持多技能叠加
* **可移植性**：多平台通用（Claude Apps, Claude Code, API）
* **渐进式加载**：Claude 最初只看到技能名称和描述，按需加载完整内容

### 2.2 渐进式披露（Progressive Disclosure）

Skills 不会一次性把所有指令塞给模型，而是分三层按需展开：

| 加载阶段 | 类比 | 加载内容 | 触发条件 |
|:--|:--|:--|:--|
| **第一层：元数据** | 图书馆目录 | SKILL.md 中的名称和描述 | 启动即加载 |
| **第二层：核心指令** | 翻开书本正文 | 具体规则、工作流逻辑 | 按需触发（调用 `use_skill`） |
| **第三层：外部资源** | 查看书后的附录 | 依赖的脚本、大型参考文档 | 执行中触发 |

**为什么这样设计？**

* **节省 Token**：不相关的 Skill 不占用 Context
* **动态组合**：Skill 可以调用其他 Skill，形成工作流
* **避免干扰**：模型只看到当前任务相关的指令

### 2.3 两种执行模式

**非 Fork 模式（默认）**：

```
┌─────────────────────────────────────────────────────────────┐
│                    主对话 API 请求                            │
├─────────────────────────────────────────────────────────────┤
│  messages: [                                                │
│    ...之前的对话,                                            │
│    [Assistant] tool_use: { name: "Skill", input: {...} }    │
│    [User] tool_result: { content: "完整 SKILL.md 内容" }     │ ← 注入！
│    [Assistant] 继续执行...                                   │
│    ...所有后续消息都在主对话中累积                              │
│  ]                                                          │
└─────────────────────────────────────────────────────────────┘
```

**Fork 模式（context: fork）**：

```
┌──────────────────────────────┐  ┌──────────────────────────────┐
│        主对话 API 请求         │  │    子代理 API 请求（隔离）     │
├──────────────────────────────┤  ├──────────────────────────────┤
│  messages: [                 │  │  messages: [                 │
│    ...之前的对话,             │  │    [User] SKILL.md + 任务指令 │
│    [Assistant] Task call     │  │    [Assistant] 执行步骤1...   │
│    [User] "子代理执行结果摘要"│  │    ...中间步骤不影响主对话     │
│  ]                           │  │  ]                           │
└──────────────────────────────┘  └──────────────────────────────┘
```

### 2.4 实战案例：Superpowers 开发工作流

[Superpowers](https://github.com/obra/superpowers) 是一个完整的软件开发工作流插件，展示了 Skills 的最佳实践：

| Skill | 用途 | 触发时机 |
|:--|:--|:--|
| `brainstorming` | 苏格拉底式设计讨论 | 开始写代码前 |
| `writing-plans` | 将设计拆解为小任务 | 设计确认后 |
| `test-driven-development` | 强制 RED-GREEN-REFACTOR | 实现阶段 |
| `systematic-debugging` | 4 阶段根因分析 | 遇到 Bug 时 |
| `verification-before-completion` | 确保真的修好了 | 声称完成前 |

```
User: "帮我实现一个用户积分系统"

Agent (with Superpowers):
1. [brainstorming] 不急着写代码，先问清楚需求
   - "积分怎么获取？消费还是签到？"
   - "积分能兑换什么？有过期吗？"

2. [writing-plans] 设计确认后，拆解任务
   - Task 1: 创建积分表结构 (2min)
   - Task 2: 实现积分增加接口 (3min)
   ...

3. [subagent-driven-development] 启动子 Agent 逐个执行

4. [verification-before-completion] 最后验证，跑测试
```

**价值**：把"先想清楚再动手"变成 Agent 的强制行为。

---

## 3. 上下文工程：算法视角

从 Context Construction 的角度重新审视所有组件。Agent 的本质就是**动态构建 Context** 的过程。

### 3.1 Context 组装视图

```
[System Message]  <-- 静态层 (Base Distribution)
├── Identity & Core Instructions
├── CLAUDE.md Content  [Hard-coded Injection]
├── Tool Definitions (MCP Tools Schema)  [Function Definitions]
└── Current Time/OS Info

[Conversation History] <-- 动态层 (Runtime State)
├── User: "帮我查一下数据库"
├── Assistant: ... {tool_call: query_db}
├── User (Tool Result): "Result: ..."  [Dynamic Injection]
│   ... (多轮对话) ...

[Current Turn] <-- 交互层 (Instruction)
├── User: "/review-pr" (Skill Trigger)
└── (Skill Prompt Expansion): "你现在是Code Reviewer..."  [ICL]
```

### 3.2 各组件的算法本质

| 组件 | 算法视角 | 作用 |
|:--|:--|:--|
| **CLAUDE.md** | Global Attention Mask / Soft Prompt | 设定模型的 Base Distribution |
| **MCP Tools** | Constrained Decoding | 重塑输出空间，遇到特定意图时输出 JSON |
| **MCP Resources** | Agent-Driven RAG | 按需检索，Context 爆炸的主要来源 |
| **Skills** | In-Context Learning / Instruction Tuning | 通过强先验收敛解空间 |
| **Slash Commands** | Meta-Operations | 操作 Context 本身（`/clear` = 重置，`/compact` = 压缩） |

### 3.3 一个 Task 的 Context 演变

假设任务："查一下数据库里的用户表结构"

| Step | Context 变化 | Token 消耗 |
|:--|:--|:--|
| 1. 初始状态 | `[System Prompt + Tools Def]` | ~1k |
| 2. 用户指令 | + `[User: "查用户表结构"]` | ~1.1k |
| 3. 工具调用 | + `[Call list_tables]` + `[Result: ['users', 'orders']]` | ~1.2k |
| 4. 进一步探索 | + `[Call get_schema("users")]` | ~1.3k |
| 5. 数据注入 | + `[Result: 500 行 SQL DDL]` | **~5k** |
| 6. 最终回答 | 基于 Context 中的 DDL 回答 | ~5.5k |

**关键时刻在 Step 5**：数据库的知识被物理地搬运到了 Context 中。模型现在"知道"了表结构。

### 3.4 核心心法

1. **Tools 是索引**：工具定义不占多少空间，但它给了模型访问无限世界的索引。
2. **Resources 是缓存**：只把当前任务急需的数据加载到 Context 中，用完即弃（通过 `/compact`）。
3. **Skills 是微调**：用 Prompt 替代微调，在 Runtime 动态加载特定领域的专家知识。

这就是为什么 Claude Code 比 Copilot 强的原因：Copilot 的 Context 是被动填充的（IDE 里的代码片段），而 Claude Code 的 Context 是 **Agent 主动探索和构建的**。

---

## 4. 落地思考：Internal-MCP

如果将 MCP 引入团队内部架构：

```
[ Engineer / PM ]
       │
       ▼
[ Internal Agent (Claude/DeepSeek) ]
       │
       ├───> [ MCP: DevOps ] ────> [ K8s / Jenkins ]
       │
       ├───> [ MCP: Data ] ──────> [ MySQL / Hive ]
       │
       └───> [ MCP: Monitor ] ───> [ Sentry / Prometheus ]
```

**杀手级场景**：

1. **Oncall Copilot**：
   "帮我查一下 `order-service` 昨晚 22:00 的报错 Top 3，并关联对应的 Git Commit。"
   Agent 自动聚合 Sentry + Gitlab 信息。

2. **Data Analyst Agent**：
   "统计上周 DAU 下跌的用户群体的特征。"
   Agent 自动写 SQL → 跑查询 → 出图表。

---

## 5. 总结

1. **拥抱 Agentic Workflow**：从"写代码"转向"设计任务"。
2. **理解 Context Engineering**：Agent 的核心能力不是生成文本，而是动态构建上下文。
3. **尝试 MCP**：不要只做 API 的消费者，开始做 MCP 的生产者。
4. **用 Skills 固化最佳实践**：把团队的 SOP 变成 Agent 的强制行为。

> *"The best way to predict the future is to invent it."*
> 让我们一起把这个未来带进团队。

---

*[Builder Notes](https://github.com/yinshucheng/builder-notes) by [Shucheng Yin](https://github.com/yinshucheng)*
