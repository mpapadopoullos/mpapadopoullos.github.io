+++
date = '2026-03-26T15:20:20+01:00'
title = 'TradingAgents Analysis'
[params]
    description = ''
+++

# TradingAgents

This post will analyze the TradingAgents research paper and its associated code.
TradingAgents leverages a multi-agent framework to simulate a professional trading firm with distinct roles.

The system is built using Python and the `LangGraph` library to orchestrate the interactions between agents in a structured workflow.

## LangChain and LangGraph

LangChain and LangGraph are Python frameworks for building applications powered by LLMs. LangGraph is an orchestration engine for complex, stateful agent workflows.

### Define a tool

```python
from langchain_core.tools import tool
from typing import Annotated

@tool
def get_weather(
    city: Annotated[str, "The city, e.g. San Francisco"]
) -> str:
    """Get weather for a given city."""
    # perform an API request to get the weather in <city>
    return f"It's always sunny in {city}!"
```

### Define an agent

```python
import os
from langchain.agents import create_agent

os.environ["GOOGLE_API_KEY"] = "[...]"

agent = create_agent(
    model="google_genai:gemini-2.5-flash-lite",
    tools=[get_weather],
    system_prompt="You are a helpful assistant",
)

# Run the agent
response = agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in Malaga?"}]}
)
```

# Code Analysis

## File: tradingagents/graph/setup.py

Handles the setup and configuration of the agent graph.

```python
class GraphSetup:
    [...]
    def setup_graph(self, [...]):
        [...]
        workflow = StateGraph(AgentState)
```

* https://reference.langchain.com/python/langgraph/graph/state/StateGraph

## File: tradingagents/graph/trading_graph.py

Main class that orchestrates the trading agents framework.


```python
class TradingAgentsGraph:
    def __init__(self, [...]):
        [...]
        self.graph_setup = GraphSetup(
            self.quick_thinking_llm,
            self.deep_thinking_llm,
            self.tool_nodes,
            self.bull_memory,
            self.bear_memory,
            self.trader_memory,
            self.invest_judge_memory,
            self.portfolio_manager_memory,
            self.conditional_logic,
        )
        [...]
```
[...]

## File: tradingagents/dataflows/interface.py

[...]

# References

* https://arxiv.org/pdf/2412.20138
* https://github.com/TauricResearch/TradingAgents
* https://www.langchain.com/
* https://www.alphavantage.co/
* https://github.com/ranaroussi/yfinance
* https://openrouter.ai/
* https://ollama.com/