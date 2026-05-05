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
