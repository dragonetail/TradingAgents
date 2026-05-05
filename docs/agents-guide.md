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