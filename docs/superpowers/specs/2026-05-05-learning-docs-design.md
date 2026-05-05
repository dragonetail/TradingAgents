# 学习文档整理方案

## 背景

面向个人开发者学习和使用 TradingAgents 项目。文档全中文，覆盖四个目标：理解架构设计、快速上手使用、深入理解各 Agent、二次开发/扩展。

## 文档结构

共 4 个文件，分主题独立，可按需阅读：

```
docs/
  getting-started.md      # 快速上手
  architecture.md         # 架构全景（改写现有英文版为中文）
  agents-guide.md         # Agent 深入理解
  extension-guide.md      # 二次开发/扩展
```

### 文档 1: `docs/getting-started.md` — 快速上手

内容：
- 环境准备（Python >=3.10、虚拟环境）
- 安装方式（pip install .、Docker）
- API Key 配置（provider → key 映射、.env 文件）
- 第一次运行（CLI + Python API，以 main.py 为示例）
- 配置项速查（DEFAULT_CONFIG 关键字段及含义）
- 运行结果位置（~/.tradingagents/ 目录结构、日志文件）

### 文档 2: `docs/architecture.md` — 架构全景

内容（将现有英文版改写为中文）：
- 五阶段 Agent 执行流水线及流程描述
- 核心源码目录及职责
- 双 LLM 模式（deep_thinking vs quick_thinking）
- 三层状态对象（AgentState / InvestDebateState / RiskDebateState）
- 结构化输出机制（Pydantic schemas + render 函数）
- 记忆系统（决策日志、延迟反思、跨股票学习）
- Checkpoint/Resume 机制

### 文档 3: `docs/agents-guide.md` — Agent 深入理解

内容（按执行顺序排列）：
- 4 个分析师（market/social/news/fundamentals）各自的工具和数据源
- Bull/Bear 研究员的辩论机制
- Research Manager 综合辩论 → ResearchPlan
- Trader 转化为 TraderProposal
- 3 个风控辩论者（aggressive/conservative/neutral）轮转机制
- Portfolio Manager 最终决策 → PortfolioDecision
- Agent 间数据流向（哪些字段传递给下游）
- 可选分析师配置（selected_analysts 参数）

### 文档 4: `docs/extension-guide.md` — 二次开发/扩展

内容：
- 添加新 LLM provider：在 llm_clients/ 下新建 client 类，注册到 factory.py
- 添加新数据源：在 dataflows/ 下实现接口，配置 data_vendors
- 添加新 Agent：实现 agent 函数，在 graph/setup.py 中加入节点
- 自定义 Prompt：各 agent prompt 的位置和修改方式
- 测试：现有测试结构、marker 说明、如何为新功能写测试

## 原则

- 全中文，专业术语保留英文原文（如 Agent、LLM、Checkpoint）
- 每个文档聚焦一个主题，独立可读
- 代码示例直接引用项目中的实际文件和路径
- 不重复 CLAUDE.md 中已有的信息（命令列表）
