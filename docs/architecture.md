# TradingAgents Architecture

## Agent Pipeline (execution order)

1. **Analyst Team** (parallel data gathering) — market, social, news, fundamentals analysts each call tool nodes (yfinance/Alpha Vantage)
2. **Research Team** (structured debate) — Bull and Bear researchers alternate for `max_debate_rounds`, then Research Manager synthesizes into a `ResearchPlan`
3. **Trader** — converts ResearchPlan into a `TraderProposal` with entry/stop-loss/sizing
4. **Risk Management Team** (circular debate) — Aggressive → Conservative → Neutral rotate for `max_risk_discuss_rounds`
5. **Portfolio Manager** — produces final `PortfolioDecision` with rating (Buy/Overweight/Hold/Underweight/Sell)

## Key Source Layout

- `tradingagents/graph/` — LangGraph orchestration: `trading_graph.py` (main `TradingAgentsGraph` class), `setup.py` (graph construction), `propagation.py` (state init), `conditional_logic.py` (routing), `signal_processing.py` (rating extraction)
- `tradingagents/agents/` — agent implementations in `analysts/`, `researchers/`, `managers/`, `trader/`, `risk_mgmt/`; `schemas.py` has Pydantic structured-output models; `utils/` has state types and tool wrappers
- `tradingagents/llm_clients/` — provider factory (`factory.py`), per-provider clients (OpenAI-compatible, Anthropic, Google, Azure). OpenAI-compatible covers: xAI, DeepSeek, Qwen, GLM, Ollama, OpenRouter
- `tradingagents/dataflows/` — market data abstraction over yfinance and Alpha Vantage
- `cli/` — Typer-based CLI with Rich display

## Dual-LLM Pattern

Every run creates two LLM instances:
- **deep_thinking_llm** — used by Research Manager and Portfolio Manager for complex reasoning
- **quick_thinking_llm** — used by all other agents

Both are created via `create_llm_client()` in `llm_clients/factory.py`. Provider-specific kwargs (Google thinking_level, OpenAI reasoning_effort, Anthropic effort) are injected based on config.

## State Management

Three TypedDict state objects flow through the graph:
- `AgentState` (inherits `MessagesState`) — carries all reports, debate states, and the final decision
- `InvestDebateState` — tracks bull/bear debate history and judge decision
- `RiskDebateState` — tracks aggressive/conservative/neutral debate history

All defined in `agents/utils/agent_states.py`.

## Structured Output

Three agents use Pydantic schemas for structured output (defined in `agents/schemas.py`):
- `ResearchPlan` — Research Manager's output
- `TraderProposal` — Trader's output
- `PortfolioDecision` — Portfolio Manager's output

Each schema has a `render_*()` function that converts the Pydantic model back to markdown for backward compatibility with the rest of the pipeline. Schema field descriptions serve as the model's output instructions.

## Memory System

Append-only markdown log at `~/.tradingagents/memory/trading_memory.md`:
- Decisions stored with deferred reflection (outcomes unknown at decision time)
- On next same-ticker run: fetches returns, generates compact reflections
- Past context = 5 same-ticker decisions + 3 cross-ticker lessons, injected into Portfolio Manager prompt

## Checkpoint/Resume

Opt-in via `config["checkpoint_enabled"] = True`. Per-ticker SQLite databases at `~/.tradingagents/cache/checkpoints/`. Auto-cleared on successful completion.

## Configuration

`tradingagents/default_config.py` defines `DEFAULT_CONFIG`. Key settings: `llm_provider`, `deep_think_llm`, `quick_think_llm`, `max_debate_rounds`, `max_risk_discuss_rounds`, `data_vendors`, `output_language`. Override via env vars: `TRADINGAGENTS_RESULTS_DIR`, `TRADINGAGENTS_CACHE_DIR`, `TRADINGAGENTS_MEMORY_LOG_PATH`.
