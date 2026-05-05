# TradingAgents 学习文档整理 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 创建 4 份中文学习文档，覆盖快速上手、架构全景、Agent 深入理解、二次开发扩展。

**Architecture:** 纯文档任务，不涉及代码修改。4 个独立 markdown 文件，每个聚焦一个主题。现有 `docs/architecture.md` 英文版将被中文版覆盖。

**Tech Stack:** Markdown，内容来自源码阅读。

---

### Task 1: docs/getting-started.md — 快速上手

**Files:**
- Create: `docs/getting-started.md`

- [ ] **Step 1: 编写 getting-started.md**

内容包括：

```markdown
# 快速上手

## 环境准备

Python >=3.10，推荐使用虚拟环境：

```bash
conda create -n tradingagents python=3.13
conda activate tradingagents
```

## 安装

```bash
git clone https://github.com/TauricResearch/TradingAgents.git
cd TradingAgents
pip install .
```

或使用 Docker：

```bash
cp .env.example .env    # 填入你的 API Key
docker compose run --rm tradingagents
```

## 配置 API Key

TradingAgents 支持多个 LLM 提供商，按需配置对应的 API Key：

| 提供商 | 环境变量 | 模型示例 |
|--------|---------|---------|
| OpenAI | `OPENAI_API_KEY` | gpt-5.4, gpt-5.4-mini |
| Google | `GOOGLE_API_KEY` | gemini-3.1-pro |
| Anthropic | `ANTHROPIC_API_KEY` | claude-sonnet-4-6 |
| xAI | `XAI_API_KEY` | grok-4 |
| DeepSeek | `DEEPSEEK_API_KEY` | deepseek-v4 |
| Qwen | `DASHSCOPE_API_KEY` | qwen-max |
| GLM | `ZHIPU_API_KEY` | glm-4 |
| OpenRouter | `OPENROUTER_API_KEY` | 动态模型选择 |
| Ollama | 无需 Key（本地） | 本地模型 |

最简方式：复制 `.env.example` 为 `.env`，填入你拥有的 Key。

## 第一次运行

### CLI 方式

```bash
tradingagents
```

启动后会出现交互界面，选择股票代码、分析日期、LLM 提供商和研究深度。

### Python API 方式

参考项目根目录的 `main.py`：

```python
from tradingagents.graph.trading_graph import TradingAgentsGraph
from tradingagents.default_config import DEFAULT_CONFIG

config = DEFAULT_CONFIG.copy()
config["deep_think_llm"] = "gpt-5.4-mini"
config["quick_think_llm"] = "gpt-5.4-mini"

ta = TradingAgentsGraph(debug=True, config=config)
_, decision = ta.propagate("NVDA", "2024-05-10")
print(decision)
```

## 关键配置项

`tradingagents/default_config.py` 中的 `DEFAULT_CONFIG`：

| 配置项 | 默认值 | 说明 |
|--------|-------|------|
| `llm_provider` | `"openai"` | LLM 提供商 |
| `deep_think_llm` | `"gpt-5.4"` | 复杂推理用模型（Research Manager、Portfolio Manager） |
| `quick_think_llm` | `"gpt-5.4-mini"` | 快速推理用模型（其他所有 Agent） |
| `max_debate_rounds` | `1` | Bull/Bear 辩论轮数 |
| `max_risk_discuss_rounds` | `1` | 风控辩论轮数 |
| `output_language` | `"English"` | 输出语言（如 `"Chinese"`） |
| `checkpoint_enabled` | `False` | 是否启用断点续跑 |
| `data_vendors` | 全 `"yfinance"` | 数据源配置 |

可通过环境变量覆盖路径：

| 环境变量 | 说明 |
|---------|------|
| `TRADINGAGENTS_RESULTS_DIR` | 运行日志存放目录 |
| `TRADINGAGENTS_CACHE_DIR` | 缓存和检查点目录 |
| `TRADINGAGENTS_MEMORY_LOG_PATH` | 记忆日志文件路径 |

## 运行结果在哪看

所有输出存放在 `~/.tradingagents/` 目录下：

```
~/.tradingagents/
  logs/                          # 运行日志（按股票代码子目录）
    NVDA/
      TradingAgentsStrategy_logs/
        full_states_log_2024-05-10.json   # 完整状态快照
  cache/
    checkpoints/                 # 断点续跑数据（需启用）
  memory/
    trading_memory.md            # 决策记忆日志
```

JSON 日志中包含每个阶段的完整内容：分析师报告、辩论记录、交易提案、最终决策。
```

- [ ] **Step 2: 检查文件内容**

确认所有文件路径引用正确（`main.py`、`default_config.py` 存在），配置项名称与源码一致。

- [ ] **Step 3: Commit**

```bash
git add docs/getting-started.md
git commit -m "docs: add Chinese getting-started guide"
```

---

### Task 2: docs/architecture.md — 架构全景（中文版）

**Files:**
- Modify: `docs/architecture.md`（覆盖现有英文版）

- [ ] **Step 1: 重写 architecture.md 为中文**

完整内容：

```markdown
# 架构全景

## 执行流水线

TradingAgents 的核心是一个五阶段 Agent 执行流水线，使用 LangGraph 编排：

```
分析师团队（并行数据采集）
    ↓
研究团队（Bull/Bear 辩论 → Research Manager 综合）
    ↓
交易员（研究计划 → 交易提案）
    ↓
风控团队（三方辩论：激进/保守/中立）
    ↓
投资组合经理（最终决策）
```

每个阶段处理不同类型的信息，下游 Agent 接收上游所有输出作为上下文。

## 核心源码目录

| 目录 | 职责 |
|------|------|
| `tradingagents/graph/` | LangGraph 编排：图构建、状态流转、条件路由、信号处理 |
| `tradingagents/agents/` | Agent 实现：analysts/、researchers/、managers/、trader/、risk_mgmt/ |
| `tradingagents/llm_clients/` | LLM 提供商集成：工厂模式，惰性导入 |
| `tradingagents/dataflows/` | 市场数据抽象：yfinance、Alpha Vantage，按类别路由 |
| `cli/` | Typer CLI + Rich 实时展示 |

关键文件：

- `graph/trading_graph.py` — 主入口 `TradingAgentsGraph` 类，协调所有组件
- `graph/setup.py` — `GraphSetup` 类，构建 LangGraph 节点和边
- `graph/propagation.py` — `Propagator` 类，创建初始状态
- `graph/conditional_logic.py` — `ConditionalLogic` 类，控制辩论轮转路由
- `graph/signal_processing.py` — `SignalProcessor` 类，从决策中提取评级
- `agents/schemas.py` — Pydantic 结构化输出模型
- `agents/utils/agent_states.py` — 状态类型定义
- `llm_clients/factory.py` — `create_llm_client()` 提供商工厂
- `default_config.py` — 全部配置项及默认值

## 双 LLM 模式

每次运行创建两个 LLM 实例：

- **deep_thinking_llm** — 给 Research Manager 和 Portfolio Manager 使用，处理复杂推理
- **quick_thinking_llm** — 给其他所有 Agent 使用，快速响应

两者都通过 `create_llm_client()` 创建，支持提供商特定参数：
- Google: `thinking_level`
- OpenAI: `reasoning_effort`
- Anthropic: `effort`

## 状态管理

三层 TypedDict 状态对象在图中流转：

**AgentState**（继承 MessagesState）— 主状态容器：
- 输入：`company_of_interest`、`trade_date`
- 分析师报告：`market_report`、`sentiment_report`、`news_report`、`fundamentals_report`
- 辩论状态：`investment_debate_state`、`risk_debate_state`
- 决策：`investment_plan`、`trader_investment_plan`、`final_trade_decision`
- 记忆：`past_context`

**InvestDebateState** — 投资辩论追踪：
- `bull_history` / `bear_history` — 各方论点历史
- `history` — 合并的辩论记录
- `judge_decision` — Research Manager 的综合结论
- `count` — 当前辩论轮数

**RiskDebateState** — 风控辩论追踪：
- `aggressive_history` / `conservative_history` / `neutral_history` — 各方论点历史
- `current_*_response` — 各方最新回应
- `latest_speaker` — 上一个发言者（用于轮转）
- `judge_decision` — Portfolio Manager 的最终决策
- `count` — 当前辩论轮数

## 结构化输出

三个决策 Agent 使用 Pydantic schema 做结构化输出（定义在 `agents/schemas.py`）：

| Agent | Schema | 输出 |
|-------|--------|------|
| Research Manager | `ResearchPlan` | recommendation + rationale + strategic_actions |
| Trader | `TraderProposal` | action (Buy/Hold/Sell) + reasoning + 可选 entry_price/stop_loss/position_sizing |
| Portfolio Manager | `PortfolioDecision` | rating (5级) + executive_summary + investment_thesis + 可选 price_target/time_horizon |

每个 schema 有对应的 `render_*()` 函数将 Pydantic 对象转回 markdown，保持与系统其他部分（记忆日志、CLI 展示、报告文件）的兼容。

Schema 字段的 `description` 直接作为模型的输出指令，prompt 正文只需提供上下文和评级指导。

评级体系：
- Portfolio Rating（5 级）：Buy / Overweight / Hold / Underweight / Sell
- Trader Action（3 级）：Buy / Hold / Sell

## 记忆系统

基于追加式 markdown 日志（`~/.tradingagents/memory/trading_memory.md`）：

1. **存储决策** — 每次运行结束，`propagate()` 将决策存为 "pending" 状态
2. **延迟反思** — 下次对同一股票运行时，获取实际收益（原始收益和相对 SPY 的 alpha），生成 2-4 句反思
3. **注入上下文** — 向 Portfolio Manager 注入过去经验：5 条同股票决策 + 3 条跨股票教训

可通过 `TRADINGAGENTS_MEMORY_LOG_PATH` 覆盖日志路径，通过 `memory_log_max_entries` 限制已解决条目数量。

## Checkpoint/Resume

通过 `config["checkpoint_enabled"] = True` 启用。使用 LangGraph 的 SqliteSaver 在每个节点后保存状态。

- 检查点存储：`~/.tradingagents/cache/checkpoints/<TICKER>.db`
- 每个 ticker 独立 SQLite 数据库，支持并发
- 成功完成后自动清除
- 中断后重新运行相同 ticker+日期会从上次成功的节点恢复

CLI 参数：`--checkpoint`（启用）、`--clear-checkpoints`（重置）。
```

- [ ] **Step 2: 验证内容准确性**

确认所有类名、函数名、字段名与源码一致。

- [ ] **Step 3: Commit**

```bash
git add docs/architecture.md
git commit -m "docs: rewrite architecture guide in Chinese"
```

---

### Task 3: docs/agents-guide.md — Agent 深入理解

**Files:**
- Create: `docs/agents-guide.md`

- [ ] **Step 1: 编写 agents-guide.md**

完整内容：

```markdown
# Agent 深入理解

本文按执行顺序详细介绍每个 Agent 的职责、输入/输出和工具。

## 阶段一：分析师团队

四个分析师并行运行，各自采集不同维度的数据并生成报告。可通过 `selected_analysts` 参数选择启用哪些分析师（默认全部启用）。

所有分析师共享相同的工作模式：
1. 接收 `company_of_interest` 和 `trade_date`
2. 调用绑定的工具获取数据
3. 生成分析报告写入对应状态字段

### Market Analyst（技术分析师）

- **源码**: `tradingagents/agents/analysts/market_analyst.py`
- **工具**: `get_stock_data`（股价数据）、`get_indicators`（技术指标，最多选 8 个，包括 SMA、EMA、MACD、RSI、布林带等）
- **输入**: `company_of_interest`、`trade_date`
- **输出**: `market_report` — 技术分析报告，包含趋势分析和 Markdown 表格
- **数据源**: `core_stock_apis` + `technical_indicators` 类别（默认 yfinance）

### Social Media Analyst（社交媒体分析师）

- **源码**: `tradingagents/agents/analysts/social_media_analyst.py`
- **工具**: `get_news`（搜索公司相关新闻和社交媒体讨论）
- **输入**: `company_of_interest`、`trade_date`
- **输出**: `sentiment_report` — 社交媒体情绪分析报告
- **数据源**: `news_data` 类别（默认 yfinance）

### News Analyst（新闻分析师）

- **源码**: `tradingagents/agents/analysts/news_analyst.py`
- **工具**: `get_news`（公司新闻）、`get_global_news`（宏观经济新闻）
- **输入**: `company_of_interest`、`trade_date`
- **输出**: `news_report` — 全球新闻和宏观经济分析报告
- **数据源**: `news_data` 类别（默认 yfinance）

### Fundamentals Analyst（基本面分析师）

- **源码**: `tradingagents/agents/analysts/fundamentals_analyst.py`
- **工具**: `get_fundamentals`（公司概览）、`get_balance_sheet`（资产负债表）、`get_cashflow`（现金流量表）、`get_income_statement`（利润表）
- **输入**: `company_of_interest`、`trade_date`
- **输出**: `fundamentals_report` — 基本面分析报告
- **数据源**: `fundamental_data` 类别（默认 yfinance）

## 阶段二：研究团队

### Bull Researcher（看多研究员）

- **源码**: `tradingagents/agents/researchers/bull_researcher.py`
- **使用的 LLM**: `quick_thinking_llm`
- **输入**: 所有分析师报告 + `investment_debate_state`（辩论历史）
- **输出**: 更新 `investment_debate_state` — 在 `bull_history` 中追加看多论点

工作方式：阅读所有分析师报告和之前的辩论历史，构建看多论据，直接反驳 Bear 的观点。

### Bear Researcher（看空研究员）

- **源码**: `tradingagents/agents/researchers/bear_researcher.py`
- **使用的 LLM**: `quick_thinking_llm`
- **输入**: 所有分析师报告 + `investment_debate_state`（辩论历史）
- **输出**: 更新 `investment_debate_state` — 在 `bear_history` 中追加看空论点

工作方式：与 Bull 对称，构建看空论据，反驳 Bull 的观点。

### 辩论机制

Bull 和 Bear 交替发言，由 `conditional_logic.py` 控制：
- 每轮 Bull → Bear 各发言一次
- 轮数由 `max_debate_rounds` 控制（默认 1）
- 辩论结束后交给 Research Manager

### Research Manager（研究经理）

- **源码**: `tradingagents/agents/managers/research_manager.py`
- **使用的 LLM**: `deep_thinking_llm`
- **输入**: `investment_debate_state`（完整辩论历史）
- **输出**: `investment_plan`（结构化 `ResearchPlan`）、更新 `investment_debate_state.judge_decision`

工作方式：评估整个辩论，使用结构化输出 `ResearchPlan`，包含：
- `recommendation` — 5 级评级（Buy/Overweight/Hold/Underweight/Sell）
- `rationale` — 辩论关键论据总结
- `strategic_actions` — 给 Trader 的具体执行建议

## 阶段三：交易员

### Trader

- **源码**: `tradingagents/agents/trader/trader.py`
- **使用的 LLM**: `quick_thinking_llm`
- **输入**: `company_of_interest`、`investment_plan`、所有分析师报告
- **输出**: `trader_investment_plan`（结构化 `TraderProposal`）

工作方式：将 Research Manager 的投资计划转化为具体交易提案，使用结构化输出 `TraderProposal`，包含：
- `action` — 3 级方向（Buy/Hold/Sell）
- `reasoning` — 2-4 句决策理由
- 可选：`entry_price`、`stop_loss`、`position_sizing`

## 阶段四：风控团队

### 三方辩论者

三个辩论者使用相同的状态结构和轮转机制，各自持有不同立场：

| Agent | 源码 | 立场 |
|-------|------|------|
| Aggressive Debator | `risk_mgmt/aggressive_debator.py` | 追求高回报，承担高风险 |
| Conservative Debator | `risk_mgmt/conservative_debator.py` | 保护资产，控制波动 |
| Neutral Debator | `risk_mgmt/neutral_debator.py` | 平衡视角，稳健策略 |

- **使用的 LLM**: `quick_thinking_llm`
- **输入**: `risk_debate_state`（辩论历史）、所有分析师报告、`trader_investment_plan`
- **输出**: 更新 `risk_debate_state` — 在各自 `*_history` 中追加论点

### 辩论轮转机制

由 `conditional_logic.py` 控制，循环发言：
- 轮转顺序：Aggressive → Conservative → Neutral → Aggressive → ...
- 轮数由 `max_risk_discuss_rounds` 控制（默认 1）
- 每个发言者能看到其他两人的最新回应
- `latest_speaker` 字段追踪上一个发言者，决定下一个发言者
- 辩论结束后交给 Portfolio Manager

## 阶段五：投资组合经理

### Portfolio Manager

- **源码**: `tradingagents/agents/managers/portfolio_manager.py`
- **使用的 LLM**: `deep_thinking_llm`
- **输入**: `risk_debate_state`（完整风控辩论）、`investment_plan`、`trader_investment_plan`、`past_context`（历史经验）
- **输出**: `final_trade_decision`（结构化 `PortfolioDecision`）、更新 `risk_debate_state.judge_decision`

工作方式：综合风控辩论和所有前置信息，使用结构化输出 `PortfolioDecision`，包含：
- `rating` — 5 级评级（Buy/Overweight/Hold/Underweight/Sell）
- `executive_summary` — 2-4 句行动计划
- `investment_thesis` — 基于辩论证据的详细分析
- 可选：`price_target`、`time_horizon`

如果记忆日志中有历史经验，会作为 `past_context` 注入到 prompt 中。

## Agent 间数据流向

```
Market Analyst ──→ market_report ──────────────────────────┐
Social Analyst ──→ sentiment_report ────────────────────────┤
News Analyst ────→ news_report ─────────────────────────────┤
Fundamentals ───→ fundamentals_report ──────────────────────┤
                                                            ↓
Bull Researcher ←── 所有报告 + debate_state ──→ debate_state
Bear Researcher ←── 所有报告 + debate_state ──→ debate_state
                              ↓
Research Manager ←── debate_state ──→ investment_plan
                              ↓
Trader ←── investment_plan + 所有报告 ──→ trader_investment_plan
                              ↓
Aggressive ←── 所有报告 + trader_plan + risk_state → risk_state
Conservative ←── 所有报告 + trader_plan + risk_state → risk_state
Neutral ←── 所有报告 + trader_plan + risk_state → risk_state
                              ↓
Portfolio Manager ←── risk_state + investment_plan + past_context
                              ↓
                    final_trade_decision
```

## 可选分析师

`TradingAgentsGraph` 构造函数的 `selected_analysts` 参数控制启用哪些分析师：

```python
# 只启用市场分析和基本面分析
ta = TradingAgentsGraph(
    selected_analysts=["market", "fundamentals"],
    config=config,
)
```

未启用的分析师对应的状态字段（如 `sentiment_report`）保持为空字符串，不影响下游流程。
```

- [ ] **Step 2: 验证源码引用**

确认所有文件路径、函数签名、状态字段名与源码一致。

- [ ] **Step 3: Commit**

```bash
git add docs/agents-guide.md
git commit -m "docs: add Chinese agents guide"
```

---

### Task 4: docs/extension-guide.md — 二次开发/扩展

**Files:**
- Create: `docs/extension-guide.md`

- [ ] **Step 1: 编写 extension-guide.md**

完整内容：

```markdown
# 二次开发/扩展指南

## 添加新 LLM 提供商

### 1. 实现 Client 类

在 `tradingagents/llm_clients/` 下创建新文件，继承 `BaseLLMClient`（`base_client.py`）：

```python
# tradingagents/llm_clients/my_provider_client.py
from typing import Any, Optional
from .base_client import BaseLLMClient

class MyProviderClient(BaseLLMClient):
    def __init__(self, model: str, base_url: Optional[str] = None, **kwargs):
        super().__init__(model, base_url, **kwargs)

    def get_llm(self) -> Any:
        from langchain_community.chat_models import ChatMyProvider  # 惰性导入
        return ChatMyProvider(
            model=self.model,
            base_url=self.base_url,
            **self.kwargs,
        )

    def validate_model(self) -> bool:
        known = ["model-a", "model-b"]
        return self.model in known
```

必须实现：
- `get_llm()` — 返回 LangChain ChatModel 实例
- `validate_model()` — 返回模型是否在已知列表中

### 2. 注册到工厂

编辑 `tradingagents/llm_clients/factory.py`：

```python
# 在 create_llm_client() 中添加分支
if provider_lower == "my_provider":
    from .my_provider_client import MyProviderClient
    return MyProviderClient(model, base_url, **kwargs)
```

如果新提供商使用 OpenAI 兼容 API，只需添加到 `_OPENAI_COMPATIBLE` 元组：

```python
_OPENAI_COMPATIBLE = (
    "openai", "xai", "deepseek", "qwen", "glm", "ollama", "openrouter",
    "my_provider",  # 新增
)
```

### 3. 更新模型目录

编辑 `tradingagents/llm_clients/model_catalog.py`，将新提供商的模型加入目录。

### 4. 配置使用

```python
config = DEFAULT_CONFIG.copy()
config["llm_provider"] = "my_provider"
config["deep_think_llm"] = "model-a"
config["quick_think_llm"] = "model-b"
```

## 添加新数据源

### 1. 实现数据函数

在 `tradingagents/dataflows/` 下创建新模块，实现与现有供应商相同签名的函数。参考 `y_finance.py` 中的函数签名：

```python
# tradingagents/dataflows/my_vendor.py

def get_stock_data_my_vendor(ticker, curr_date):
    """返回格式化 CSV 字符串的股价数据"""
    ...

def get_indicators_my_vendor(ticker, indicator, curr_date, lookback):
    """返回格式化字符串的技术指标数据"""
    ...
```

### 2. 注册到接口层

编辑 `tradingagents/dataflows/interface.py`：

1. 在顶部导入新供应商的函数
2. 在 `VENDOR_LIST` 中添加 `"my_vendor"`
3. 在 `VENDOR_METHODS` 中为每个工具方法添加新供应商的实现：

```python
VENDOR_METHODS = {
    "get_stock_data": {
        "alpha_vantage": get_alpha_vantage_stock,
        "yfinance": get_YFin_data_online,
        "my_vendor": get_stock_data_my_vendor,  # 新增
    },
    # ... 其他方法同理
}
```

### 3. 配置使用

按类别配置（所有工具使用同一数据源）：

```python
config["data_vendors"] = {
    "core_stock_apis": "my_vendor",
    "technical_indicators": "my_vendor",
}
```

或按单个工具覆盖：

```python
config["tool_vendors"] = {
    "get_stock_data": "my_vendor",  # 只覆盖这一个工具
}
```

系统支持逗号分隔的 fallback 链：

```python
config["data_vendors"] = {
    "core_stock_apis": "my_vendor, yfinance",  # 主用 my_vendor，失败回退 yfinance
}
```

## 添加新 Agent

### 1. 实现 Agent 函数

参考现有 Agent（如 `tradingagents/agents/analysts/market_analyst.py`）的模式：

```python
# tradingagents/agents/analysts/my_analyst.py
from langchain_core.prompts import ChatPromptTemplate

def create_my_analyst(llm, tools):
    prompt = ChatPromptTemplate.from_messages([
        ("system", "你是一个..."),
        ("human", "分析 {company_of_interest} 在 {trade_date} 的情况"),
    ])

    chain = prompt | llm.bind_tools(tools)

    def my_analyst(state):
        result = chain.invoke({
            "company_of_interest": state["company_of_interest"],
            "trade_date": state["trade_date"],
        })
        return {"my_report": result.content, "messages": [result]}

    return my_analyst
```

关键点：
- Agent 函数接收 `state` 字典（`AgentState` 类型）
- 返回一个字典，包含要更新的状态字段
- 需要工具的 Agent 通过 `bind_tools()` 绑定

### 2. 添加状态字段

在 `tradingagents/agents/utils/agent_states.py` 的 `AgentState` 中添加新字段：

```python
class AgentState(MessagesState):
    # ... 现有字段
    my_report: Annotated[str, "My custom analyst report"]
```

### 3. 注册到图

编辑 `tradingagents/graph/setup.py`：

1. 导入新 Agent 创建函数
2. 在 `setup_graph()` 中添加节点和边

```python
# 添加节点
graph.add_node("My_Analyst", my_analyst_node)

# 添加边（在合适位置插入）
graph.add_edge("Start", "My_Analyst")
graph.add_edge("My_Analyst", "Next_Node")
```

### 4. 添加工具节点

如果新 Agent 使用工具，在 `tradingagents/graph/trading_graph.py` 的 `_create_tool_nodes()` 中添加：

```python
self.tool_nodes = {
    # ... 现有节点
    "my_analyst": ToolNode([my_tool_1, my_tool_2]),
}
```

## 自定义 Prompt

每个 Agent 的 prompt 定义在其创建函数中，直接编辑对应文件：

| Agent | Prompt 位置 |
|-------|------------|
| Market Analyst | `agents/analysts/market_analyst.py` 中的 `ChatPromptTemplate` |
| Social Analyst | `agents/analysts/social_media_analyst.py` |
| News Analyst | `agents/analysts/news_analyst.py` |
| Fundamentals Analyst | `agents/analysts/fundamentals_analyst.py` |
| Bull/Bear Researcher | `agents/researchers/bull_researcher.py` / `bear_researcher.py` |
| Research Manager | `agents/managers/research_manager.py` |
| Trader | `agents/trader/trader.py` |
| Risk Debators | `agents/risk_mgmt/aggressive_debator.py` 等 |
| Portfolio Manager | `agents/managers/portfolio_manager.py` |

Prompt 使用 LangChain 的 `ChatPromptTemplate.from_messages()` 格式，包含 `system` 和 `human` 消息模板。

结构化输出 Agent（Research Manager、Trader、Portfolio Manager）的 schema 字段 `description` 也影响输出行为，修改时注意 `agents/schemas.py`。

## 测试

### 测试结构

```
tests/
  conftest.py                      # 公共 fixtures
  test_checkpoint_resume.py        # 断点续跑测试
  test_deepseek_reasoning.py       # DeepSeek 推理测试
  test_google_api_key.py           # Google API 测试
  test_memory_log.py               # 记忆日志测试
  test_model_validation.py         # 模型验证测试
  test_safe_ticker_component.py    # Ticker 安全处理测试
  test_signal_processing.py        # 信号处理测试
  test_structured_agents.py        # 结构化输出测试
  test_ticker_symbol_handling.py   # Ticker 符号测试
```

### 测试标记

```bash
pytest -m unit          # 快速隔离的单元测试
pytest -m integration   # 需要外部服务的集成测试
pytest -m smoke         # 快速冒烟测试
```

`conftest.py` 提供了惰性 LLM 客户端导入和占位 API Key，使测试套件无需真实凭证即可运行。

### 为新功能写测试

单元测试示例（不依赖外部服务）：

```python
import pytest

@pytest.mark.unit
def test_my_vendor_routing():
    """测试数据源路由逻辑"""
    from tradingagents.dataflows.interface import route_to_vendor
    # 配置和断言
```

集成测试示例（需要 API Key）：

```python
@pytest.mark.integration
def test_my_provider_llm():
    """测试 LLM 提供商实际调用"""
    from tradingagents.llm_clients import create_llm_client
    client = create_llm_client("my_provider", "model-a")
    llm = client.get_llm()
    result = llm.invoke("Hello")
    assert result.content
```

### 诊断脚本

`scripts/smoke_structured_output.py` 可以快速验证结构化输出是否正常工作：

```bash
python scripts/smoke_structured_output.py
```
```

- [ ] **Step 2: 验证内容**

确认所有文件路径、类名、函数名与源码一致。确认代码示例语法正确。

- [ ] **Step 3: Commit**

```bash
git add docs/extension-guide.md
git commit -m "docs: add Chinese extension guide"
```

---

### Task 5: 最终验证

- [ ] **Step 1: 检查所有文档链接和引用**

确认以下内容：
- 所有提到的源码文件路径确实存在
- 所有类名、函数名与源码一致
- `CLAUDE.md` 中引用了 `docs/architecture.md`（已存在）
- 配置项名称与 `default_config.py` 一致

- [ ] **Step 2: 检查文档间无冲突**

- `getting-started.md` 和 `architecture.md` 无重复内容（前者侧重操作，后者侧重设计）
- `agents-guide.md` 引用的状态字段名与 `architecture.md` 一致
- `extension-guide.md` 的代码示例引用的路径与实际结构一致

- [ ] **Step 3: Final commit**

```bash
git add -A docs/
git commit -m "docs: complete Chinese learning documentation set"
```
```
