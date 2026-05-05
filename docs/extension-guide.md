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

写完文件后，不需要 commit。