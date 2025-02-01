---
title:  "langchain:让模型自己编写工具使用"
date:   2025-2-2 06:53:00 +0800
tags:
  - python
  - langchain
  - AI
---

利用PythonAstREPLTool可以让模型自己编写工具使用

```terminal
pip install langchain_experimental
```

```python
from langchain_experimental.tools import PythonAstREPLTool
from langchain_experimental.agents.agent_toolkits import create_python_agent 
```

```python
from langchain_openai import ChatOpenAI
from langchain_experimental.tools import PythonAstREPLTool
from langchain_experimental.agents.agent_toolkits import create_python_agent # 创建能执行python代码的agent执行器


model = ChatOpenAI(model="moonshot-v1-8k", base_url="https://api.moonshot.cn/v1", temperature=0)

agent_executor = create_python_agent(
    llm=model,
    tool=PythonAstREPLTool(),
    verbose=True,
    agent_executor_kwargs={"handle_parsing_errors": True}
)

agent_executor.invoke({"input": "斐波那契数列的第40个数字是多少"})
```

```terminal
> Entering new AgentExecutor chain...
To find the 40th Fibonacci number, I can write a Python function that calculates Fibonacci numbers iteratively.
Action: python_repl_ast
Action Input: ```python
def fibonacci(n):
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return a

fibonacci(40)
```
Observation: 102334155
Thought:I now know the final answer
Final Answer: 102334155

> Finished chain.
```
