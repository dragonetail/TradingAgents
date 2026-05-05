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