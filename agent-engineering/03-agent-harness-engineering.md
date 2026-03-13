# Agent 迭代机制调研：Harness Engineering 全景

> **定位**：Agent 迭代开发的行业最佳实践调研，从配置化到自主进化的完整路径。
>
> **Author**: [Shucheng Yin](https://github.com/yinshucheng)（尹树成）
>
> **Date**: 2026-03

---

## 一、行业共识：Harness Engineering 是核心范式

2026 年初，**Harness Engineering** 已成为 Agent 迭代开发的核心范式。

> "每当 Agent 犯一个错，你工程化一个方案，让它永远不再犯这个错。" — Mitchell Hashimoto

**Agent = Model + Harness**：

| 层 | 类比 | 职责 |
|---|---|---|
| Agent | 应用 | 业务逻辑 |
| Harness | 操作系统 | 上下文工程、约束执行、工具管理、轨迹采集 |
| Context Window | RAM | 有限工作记忆，需主动管理 |
| Model | CPU | 可替换的推理能力 |

**三条开发者准则**（Phil Schmid）：
1. **Start Simple** — 提供原子工具，让模型做规划
2. **Build to Delete** — 模块化，新模型会淘汰当前逻辑
3. **Harness is the Dataset** — 竞争优势是运行轨迹，不是 prompt

**核心参考**：
- [OpenAI - Harness Engineering](https://openai.com/index/harness-engineering/)
- [Phil Schmid - Agent Harness in 2026](https://www.philschmid.de/agent-harness-2026)
- [Martin Fowler - Harness Engineering](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html)

### 1.2 辩论：Big Model vs Big Harness

来源：[Latent.Space — Is Harness Engineering real?](https://www.latent.space/p/ainews-is-harness-engineering-real)（2026-03-05）

**Big Model 派（Harness 价值有限）**：

| 论据 | 来源 |
|---|---|
| Claude Code 的 harness 极其简单（minimal），核心价值在模型本身 | Boris Cherny & Cat Wu（Anthropic, Claude Code 团队） |
| 推理模型出现之前，大量 harness 工程是在给非推理模型"凑出"推理能力。推理模型一出，那些 harness 全废了 | Noam Brown（OpenAI） |
| Scale AI SWE-Atlas：Opus 4.6 在 Claude Code 比通用 SWE-Agent 好 2.5 分，但 GPT 5.2 反过来——harness 差异在误差范围内 | Scale AI |
| **Bitter Lesson**：长期看，模型能力的提升会吞掉所有 harness 的临时价值 | 经典 ML 观点 |

**Big Harness 派（Harness 有真实价值）**：

| 论据 | 来源 |
|---|---|
| 所有生产级 Agent 收敛到同一核心循环：`while (tool calls): execute → capture → append → call again`。这个循环本身就是 harness | Latent.Space 观察 |
| "The Model Harness is Everything — 从 AI 获取价值的最大障碍是你自己的 context 和 workflow 工程能力" | Jerry Liu（LlamaIndex） |
| **一个下午优化 harness，15 个 LLM 全部提升**——只改 harness 不换模型 | Pi 实验 |
| Cursor 估值 $50B，卖的不是模型，卖的是 harness | 市场验证 |
| LangChain 只改 Harness 不换模型，从 Top 30 跳到 Top 5 | LangChain benchmark |

**我的立场**：

不是 Model vs Harness 的零和博弈。关键洞察：
1. **谁先把 harness 做好，谁就能在当前模型能力下多吃一层红利。** Bitter Lesson 可能 3-5 年后生效，这中间的窗口期就是 harness 的价值期。
2. **评测 + 配置管理是刚需，不受 Big Model 论影响。** 就算换最强模型，没有评测体系你也不知道效果好不好。
3. **Harness 的核心价值不只是让模型跑得好，更是让迭代可量化、可追溯、可自动化。** 这是工程管理价值，和模型能力无关。

### 1.3 控制论视角：Harness 是第三次反馈回路革命

| 时代 | 之前（人工操作） | 之后（设计反馈回路） |
|---|---|---|
| 1780s 蒸汽机 | 工人手动拧阀门调压力 | 瓦特离心调速器自动调节 |
| 2010s 服务器 | 运维手动重启/扩容 | Kubernetes 声明期望状态，控制器自动协调 |
| 2026 Agent | 工程师手动改 prompt/调参数 | 设计评估环境 + 反馈回路，Agent 自动迭代 |

**关键洞见**：
- 以前代码库只有低层反馈（编译器、测试、linter），高层架构判断只有人类能做。LLM 第一次把高层反馈回路闭合了。
- **最难的不是让 AI 做事，而是把你脑子里的"什么叫好"外化成机器可读。**
- **人类从"划桨"变成"掌舵"**：永远不会失业，但工作从拧阀门变成设计调速器。

---

## 二、迭代生命周期：从人工到自动化

### 2.1 完整迭代循环

```
假设 → 配置变更 → 评测 → 部署 → 监控 → 反馈 → 下一轮假设
```

**关键转变**：工程师的角色从"写代码"变成"设计环境"。

### 2.2 四步路径与行业对齐

| 路径 | 行业对应 | 代表实践 |
|---|---|---|
| 第一步：配置化 | Prompt/Config 版本化 | OpenClaw Markdown-as-Config、LaunchDarkly AI Configs |
| 第二步：评测化 | Eval-Driven Development | Anthropic 评测方法论、Promptfoo |
| 第三步：API 化 | CI/CD for AI | Braintrust GitHub Action、LangSmith Pipeline |
| 第四步：Meta-Agent | 自主实验循环 | autoresearch、Ralph Loop、DSPy、AI Scientist |

---

## 三、评测驱动开发（Eval-Driven Development）

### 3.1 Anthropic 的评测方法论（最权威参考）

来源：[Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)

**八步落地法**：

| 步骤 | 做什么 | 实际意义 |
|---|---|---|
| 0. 尽早开始 | 20-50 个简单任务即可起步 | 不需要等"完美"的评测集 |
| 1. 从手动测试转化 | 把现在手动跑的自动化 | 已有的评测脚本就是起点 |
| 2. 从真实失败中提取 | Bug tracker、用户投诉 → 测试用例 | 线上 badcase 是最好的 eval 来源 |
| 3. 定义无歧义的成功标准 | 两个专家打分应一致 | 工具调用准确率可量化 |
| 4. 设计 Reference Solution | 证明任务可解、grader 配置正确 | 先跑通"最优情况" |
| 5. 组合多种 Grader | 确定性检查 + LLM-as-Judge + 状态检查 | 不能只靠一种评分方式 |
| 6. 确保问题足够难 | 太简单无法区分好坏 | 评测集需要梯度 |
| 7. 迭代评测本身 | 评测也需要调试 | 评测系统是持续演进的 |
| 8. 读 Transcript | 看完整执行轨迹，不只看分数 | 轨迹分析比分数更有价值 |

### 3.2 Grader 组合策略

| Grader 类型 | 检查什么 | 工具 |
|---|---|---|
| **确定性检查** | 工具是否被正确调用、返回格式是否正确 | 代码断言 |
| **LLM-as-Judge** | 生成质量、用户意图满足度 | GPT-4 / Claude 做评分 |
| **状态检查** | 最终结果是否正确反映在环境中 | 端到端验证 |

### 3.3 LLM-as-Judge 注意事项

- 使用二元或低精度评分（Pass/Fail 或 1-3 分），不用 1-10
- 复杂标准拆分：一个 evaluator 评一个维度
- 要求 step-by-step reasoning 后再给分
- 用更强的模型做 Judge（Claude Opus 评 Sonnet 输出）
- **已知偏见**：位置偏见、长度偏见、讨好偏见
- **现实**：74% 的团队仍然结合人工评估

### 3.4 非确定性处理

Agent 行为天然非确定——同一输入跑 5 次可能 3 次成功。

- `pass@k`：k 次中至少成功一次（衡量能力上限）
- `pass^k`：k 次全部成功（衡量可靠性）

---

## 四、配置化最佳实践

### 4.1 行业共识：Prompt 是版本化的软件构件

| 实践 | 说明 |
|---|---|
| Prompt 与代码解耦 | 通过 identifier 引用，不直接内嵌 |
| 完整版本历史 | 修改时间、作者、变更原因 |
| 环境管理 | dev → staging → production 各自独立 |
| 运行时热更新 | 不需要重新部署即可更新 |
| 即时回滚 | 发现问题可立即回退 |

### 4.2 OpenClaw 的 Markdown-as-Config

OpenClaw 采用分层 Markdown 文件作为配置：

```
workspace/
├── SOUL.md        # Agent 人格/价值观/规则
├── IDENTITY.md    # 名称、角色、风格
├── AGENTS.md      # 行为指令
├── TOOLS.md       # 工具使用说明
├── MEMORY.md      # 长期记忆（Agent 可写）
└── skills/
    └── <skill-name>/
        └── SKILL.md   # Skill 定义
```

**可借鉴的设计**：
- **分层注入**：SOUL → IDENTITY → AGENTS → TOOLS → Skills → 对话历史
- **Skill 按需加载**：Skills 列表注入 Prompt，内容通过 read 工具按需读取（省 token）
- **模型三级覆盖**：全局默认 → Agent 默认 → Per-Agent 覆盖 + Fallback 链

### 4.3 配置 Schema 的关键要素

综合各框架，Agent 配置应覆盖：

```yaml
agent:
  name: "food-recommend-agent"
  version: "1.2.0"
  description: "美食推荐 Agent"

  model:
    primary: "deepseek-v3"
    fallbacks: ["gpt-4o"]
    params:
      temperature: 0.7
      max_tokens: 4096

  prompts:
    system: "file://prompts/system.md"
    few_shot: "file://prompts/examples.json"

  tools:
    enabled: ["poi-search", "menu-query", "review-summary"]
    disabled: ["payment"]
    mcp_servers: ["food-mcp", "location-mcp"]

  skills:
    - name: "nearby-search"
      trigger: "用户问附近餐厅"

  constraints:
    max_turns: 20
    timeout_ms: 30000
```

### 4.4 配置版本化的分层模型

| 层级 | 内容 | 变更频率 |
|---|---|---|
| Layer 1: Prompt Bundle | System prompt + 推理模式 | 高（业务方频繁调整） |
| Layer 2: Model & Params | 模型选择 + temperature/top-p | 中 |
| Layer 3: Tool Contract | 工具 schema + 权限 | 低 |
| Layer 4: Constraints | 执行规则、护栏 | 低 |

---

## 五、CI/CD 工具链：让评测自动化

### 5.1 推荐技术栈

```
评估框架:   Promptfoo（开源，零云依赖，原生支持 GitLab/GitHub CI）
可观测性:   Langfuse（MIT 协议，可自托管）或 Arize Phoenix
Agent评估:  DeepEval（Tool-calling + 推理步骤评估）
CI/CD:     GitLab CI / GitHub Actions
配置管理:   Git-based（CODEOWNERS + MR 审核）
安全:      Promptfoo Red Team（50+ 漏洞检测插件）
```

### 5.2 Promptfoo — 最适合团队场景的评测工具

**为什么选 Promptfoo**：
- 开源 MIT，本地运行，数据不离开基础设施
- 原生支持 GitLab/GitHub CI
- YAML 声明式配置，多模型对比
- 内置 Red Team 安全扫描

**配置示例**：

```yaml
# promptfooconfig.yaml
prompts:
  - file://prompts/system.md
  - file://prompts/system-v2.md

providers:
  - openai:gpt-4o
  - anthropic:claude-3-sonnet

tests:
  - vars:
      query: "附近最好的火锅店"
    assert:
      - type: icontains
        value: "poi-search"
      - type: llm-rubric
        value: "推荐结果相关且有用"
      - type: latency
        threshold: 5000
      - type: cost
        threshold: 0.05
```

### 5.3 CI Pipeline 设计

```yaml
# .gitlab-ci.yml / GitHub Actions
stages:
  - lint       # 配置格式检查
  - evaluate   # 评测
  - gate       # 质量门禁
  - deploy     # 部署配置

evaluate_agent:
  stage: evaluate
  script:
    - npm install -g promptfoo
    - promptfoo eval -c promptfooconfig.yaml -o results.json
  rules:
    - changes:
        - prompts/**/*
        - configs/agent*.yaml

quality_gate:
  stage: gate
  script:
    - |
      PASS_RATE=$(jq '.results.stats.successes / (.results.stats.successes + .results.stats.failures) * 100' results.json)
      if (( $(echo "$PASS_RATE < 95" | bc -l) )); then
        echo "FAILED: Pass rate ${PASS_RATE}% < 95%"
        exit 1
      fi
```

**核心模式**：Config 文件变更 → 自动触发评测 → MR 中显示结果 → 质量门禁 → 部署

### 5.4 评测框架对比

| 工具 | 最适合 | CI/CD | 开源 | 特色 |
|---|---|---|---|---|
| **Promptfoo** | 安全 + 开发迭代 | GitLab/GitHub 原生 | 是 (MIT) | Red Team，50+ provider |
| **Braintrust** | 完整 eval-to-prod 闭环 | GitHub Action | 否 | Notion/Stripe 在用 |
| **LangSmith** | LangChain 生态 | pytest/Vitest | 否 | Multi-Turn Eval |
| **Arize Phoenix** | 可观测性优先 | 需自定义 | 是 | OpenTelemetry 原生 |
| **Langfuse** | 自建工作流 | Webhook | 是 (MIT) | 灵活，强 prompt 管理 |
| **DeepEval** | Agent 行为评估 | Python | 是 | Tool-calling 评估 |

### 5.5 可观测性三个层次

| 层次 | 评估方式 | 知道什么 |
|---|---|---|
| **黑盒** | 只看输入和最终输出 | 结果对不对 |
| **轨迹（Trajectory）** | 评估完整工具调用序列 | 过程合不合理 |
| **单步（White-Box）** | 逐步评估每个工具调用 | 哪一步出了问题 |

---

## 六、自主实验循环：从 Meta-Agent 到 Self-Evolving Agent

> **核心趋势（2026 Q1）**：Agent 不只是被迭代，还能自我迭代。

### 6.1 思想脉络

```
手动调参 → DSPy 自动寻优 → Ralph Wiggum Loop → autoresearch → AI Scientist → DGM
   |            |                    |                |              |            |
  人工       框架自动化          循环+验证        自主研究     全流程论文     自我改写
```

| 阶段 | 代表 | 人类角色 | Agent 角色 |
|---|---|---|---|
| 手动迭代 | Prompt Engineering | 全程操作 | 执行指令 |
| 自动寻优 | DSPy Optimizer | 定义目标函数 | 搜索最优配置 |
| 循环验证 | Ralph Wiggum Loop | 写测试/验收标准 | 循环直到通过 |
| 自主研究 | autoresearch | 写 program.md | 自主实验、保留/丢弃 |
| 全流程科研 | Sakana AI Scientist | 选研究方向 | 从想法到论文 |
| 自我进化 | Darwin Gödel Machine | 观察 | 改自己的代码 |

### 6.2 Karpathy autoresearch — 自主研究的最小实现

来源：[github.com/karpathy/autoresearch](https://github.com/karpathy/autoresearch)（2026-03）

**三文件极简架构**：

| 文件 | 角色 | 谁编辑 |
|---|---|---|
| `prepare.py` | 固定常量：数据、评估、tokenizer | 不动 |
| `train.py` | 模型+优化器+训练循环 | **Agent** |
| `program.md` | Agent 的指令（如何实验） | **Human** |

**实验循环**：

```
LOOP FOREVER:
  1. 读当前 git 状态
  2. 修改 train.py（提出假设）
  3. git commit
  4. uv run train.py > run.log 2>&1    # 固定 5 分钟
  5. 提取 val_bpb（验证 bits per byte）
  6. if 改善 → keep（推进分支）
     if 未改善 → git reset（丢弃）
  7. 记录到 results.tsv
  8. 永不停止，直到人类中断
```

**关键设计决策**：

| 决策 | 理由 |
|---|---|
| 固定 5 分钟时间盒 | 不同架构/超参可公平比较 |
| 单一指标 val_bpb | 无歧义，vocab-size 无关 |
| 自动保留/回滚 | Agent 自主决定进化方向 |
| Git 分支隔离 | 每次实验可追溯、可回退 |
| NEVER STOP | 人可以睡觉，Agent 永远不停 |

**实际产出**：一夜 ~100 次实验（12 次/小时 × 8 小时）

**启发**：
- **Markdown-as-Program**：`program.md` 就是 Agent 的"灵魂"
- **固定时间盒 + 单一指标 + 自动保留/回滚** = 最小可行的自主进化环
- 不需要复杂框架，bash + git + 一个 AI Agent 就能实现自主研究

### 6.3 Ralph Wiggum Loop — 自主编码循环

来源：[github.com/snarktank/ralph](https://github.com/snarktank/ralph)

```bash
# 最简 Ralph Loop
while :; do cat PROMPT.md | claude-code ; done
```

**与 autoresearch 的异同**：

| 维度 | Ralph Wiggum | autoresearch |
|---|---|---|
| **目标** | 通过测试/类型检查 | 降低 val_bpb |
| **验证** | 外部确定性检查（linter/test） | 单一数值指标 |
| **回滚** | Agent 自行修复 | git reset |
| **适用** | 软件工程任务 | ML 研究实验 |
| **共性** | 循环 + 外部验证 + 自主迭代 |

**关键原则**：
- **必须有外部验证**：没有确定性测试的任务不适合 Ralph
- **Context 管理**：每次循环清空重来，避免上下文污染
- **适合明确定义的任务**：迁移、重构、测试覆盖扩展、批量代码变更

### 6.4 Gas Town — 多 Agent 并行工厂

来源：Steve Yegge, [github.com/steveyegge/gastown](https://github.com/steveyegge/gastown)（2026）

```
Mayor（编排者）→ 分解任务为 Beads → 分配给 Polecat（工人 Agent）→ 并行执行 → Refinery（合并）
```

**演进路径**：Ralph（单 Agent 循环）→ Gas Town（多 Agent 工厂）→ ?（自主进化工厂）

### 6.5 Sakana AI Scientist — 全流程科研自动化

来源：[sakana.ai/ai-scientist](https://sakana.ai/ai-scientist/)

| 阶段 | 做什么 |
|---|---|
| Ideation | 基于已有代码库自动生成研究想法 |
| Experiment | 自动写代码、跑实验、收集结果 |
| Writing | 生成完整论文（含图表、引用） |
| Peer Review | AI 自动做同行评审 |

**里程碑**：v2 论文通过了 ICLR 2025 Workshop 的同行评审（首个 AI 全自动生成并通过 peer review 的论文）。

### 6.6 Darwin Gödel Machine — 自我改写代码的 Agent

来源：Sakana AI, [OpenReview](https://openreview.net/forum?id=pUpzQZTvGY)（2025）

Agent 不只优化任务代码，还**改写自己的代码**（包括改写"如何改写自己"的逻辑）。

**实测效果**：SWE-bench：20.0% → 50.0%

**这是 autoresearch 的终极形态**：不只是 Agent 改 train.py，而是 Agent 改自己的 program.md。

### 6.7 DSPy — 最成熟的自动化 Prompt 优化框架

Stanford 出品，16 万月下载，GitHub 16K+ stars。

**实测效果**（Arize 实验）：

| 方法 | 准确率 |
|---|---|
| Base Prompt | 68% |
| Few-Shot Prompting | 74% |
| Meta Prompting | 84% |
| **DSPy Optimization** | **94%** |

### 6.8 统一模式提炼

所有这些实践共享一个核心模式：

```
┌─────────────────────────────────────────────┐
│           自主迭代的三要素                     │
│                                              │
│  1. 固定的评估环境（不可被 Agent 修改）         │
│     - autoresearch: prepare.py + val_bpb     │
│     - Ralph: test suite / linter             │
│     - AI Scientist: peer review rubric       │
│                                              │
│  2. 可修改的搜索空间（Agent 的自由度）          │
│     - autoresearch: train.py                 │
│     - Ralph: 整个代码库                       │
│     - DSPy: prompt + few-shot examples       │
│                                              │
│  3. 自动保留/回滚机制                          │
│     - autoresearch: git keep/reset           │
│     - Ralph: 循环直到通过                     │
│     - DSPy: 保留最优 prompt                   │
└─────────────────────────────────────────────┘
```

---

## 七、落地建议

### 7.1 配置化（第一步）

1. **定义 Agent Config Schema**（参考 4.3）
2. **参考 microsoft/apm 的包管理思路**：
   - [github.com/microsoft/apm](https://github.com/microsoft/apm) — Agent Package Manager
   - 核心理念：Agent 配置像 npm 包一样管理（安装、版本、依赖）
   - **可借鉴**：分层目录结构、包依赖管理、配置与框架解耦
3. 配置文件 Git 管理，MR 审核
4. 版本化分层（Prompt Bundle / Model / Tool / Constraints 各自独立版本）

### 7.2 评测化（第二步）

1. **Anthropic 八步法落地**：20-50 个 case 起步，从真实失败中提取
2. **Grader 三件套**：确定性检查 + LLM-as-Judge + 状态检查
3. **接入 Promptfoo**：YAML 声明式配置，开源零云依赖
4. **CI 集成**：config 变更 → 自动跑评测 → MR 贴结果 → 质量门禁

### 7.3 可观测性（与评测并行）

- 每次运行保存完整执行轨迹（工具调用序列、推理步骤、token 消耗）
- 推荐 Langfuse（MIT，可自托管）或 Arize Phoenix
- 即使不做微调，轨迹数据也是长期护城河
- **Phil Schmid: "Harness is the Dataset"**

### 7.4 自主循环（第四步）

- **统一模式**：固定评估环境 + 可修改搜索空间 + 自动保留/回滚
- 技术路径：DSPy Optimizer（系统级）+ TextGrad（精细级）
- 工程实现：autoresearch 模式（bash + git + Agent 就够了）

### 7.5 角色权限分离

| 角色 | 可编辑 | 不可访问 |
|---|---|---|
| 业务方（PM/运营） | Prompt、测试用例、评估标准 | 模型配置、基础设施 |
| AI 工程师 | Agent 配置、工具集成、模型选择 | 生产部署权限 |
| 平台工程师 | CI/CD Pipeline、部署配置 | Prompt 内容 |

---

## 八、关键参考文献

### 必读

| 文档 | 链接 |
|---|---|
| OpenAI "Harness Engineering" | https://openai.com/index/harness-engineering/ |
| Phil Schmid "Agent Harness in 2026" | https://www.philschmid.de/agent-harness-2026 |
| Anthropic "Demystifying evals for AI agents" | https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents |
| Anthropic "Writing tools for agents — with agents" | https://www.anthropic.com/engineering/writing-tools-for-agents |
| Latent.Space "Is Harness Engineering real?" | https://www.latent.space/p/ainews-is-harness-engineering-real |

### 工具文档

| 工具 | 链接 |
|---|---|
| Promptfoo | https://www.promptfoo.dev/docs/ |
| Langfuse | https://langfuse.com/self-hosting |
| DSPy | https://dspy.ai/ |
| Arize Phoenix | https://arize.com/docs/phoenix |
| OpenClaw | https://github.com/openclaw/openclaw |
| Microsoft APM | https://github.com/microsoft/apm |

### 自主实验循环

| 文档 | 链接 |
|---|---|
| Karpathy autoresearch | https://github.com/karpathy/autoresearch |
| Ralph Wiggum Loop | https://github.com/snarktank/ralph |
| Gas Town | https://github.com/steveyegge/gastown |
| Sakana AI Scientist v2 | https://arxiv.org/abs/2504.08066 |
| Darwin Gödel Machine | https://openreview.net/forum?id=pUpzQZTvGY |

### 扩展阅读

| 文档 | 链接 |
|---|---|
| Martin Fowler "Harness Engineering" | https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html |
| Evidently AI LLM-as-Judge 指南 | https://www.evidentlyai.com/llm-guide/llm-as-a-judge |
| Amazon Agent 评测实践 | https://aws.amazon.com/blogs/machine-learning/evaluating-ai-agents-real-world-lessons-from-building-agentic-systems-at-amazon/ |

---

*[Builder Notes](https://github.com/yinshucheng/builder-notes) by [Shucheng Yin](https://github.com/yinshucheng)*
