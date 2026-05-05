# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

TradingAgents is a multi-agent LLM financial trading framework built with LangGraph. Python >=3.10, setuptools build.

## Commands

```bash
pip install .                              # install
pytest                                     # all tests
pytest -m unit                             # unit tests only
pytest tests/test_signal_processing.py     # single test file
tradingagents                              # CLI (after install)
python -m cli.main                         # CLI from source
```

## Architecture (see docs/architecture.md for details)

- `tradingagents/graph/trading_graph.py` — main `TradingAgentsGraph` class, entry point
- `tradingagents/agents/` — agents: analysts, researchers, managers, trader, risk_mgmt
- `tradingagents/llm_clients/factory.py` — `create_llm_client()` provider factory
- `tradingagents/default_config.py` — `DEFAULT_CONFIG` with all settings
