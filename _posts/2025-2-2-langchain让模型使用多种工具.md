---
title:  "langchain:让模型使用多种工具"
date:   2025-2-2 09:45:00 +0800
tags:
  - python
  - langchain
  - AI
---

将工具（继承BaseTool的类，或执行器）放进一个列表中，并在agent和agent执行器中传入

```python
from langchain_openai import ChatOpenAI
from langchain_experimental.tools import PythonAstREPLTool
from langchain_experimental.agents.agent_toolkits import create_csv_agent, create_python_agent
from langchain.agents import create_structured_chat_agent, AgentExecutor
from langchain.tools import BaseTool, Tool
from langchain import hub
from langchain.memory import ConversationBufferMemory
import os

"""工具1"""
class TextLengthTool(BaseTool):
    name: str = "文本字数计算工具"
    description: str = "当你需要计算文本包含的字数时，使用此工具"

    def _run(self, text:str) -> int:
        return len(text)

"""工具2"""    
csv_agent_executor = create_csv_agent(
    llm=ChatOpenAI(model="moonshot-v1-8k", base_url="https://api.moonshot.cn/v1", api_key=os.getenv("OPENAI_API_KEY"), temperature=0),
    path="house_price.csv",
    verbose=True,
    agent_executor_kwargs={"handle_parsing_errors": True},
    allow_dangerous_code=True
)    

"""工具3"""
python_agent_executor = create_python_agent(
    llm=ChatOpenAI(model="moonshot-v1-8k", base_url="https://api.moonshot.cn/v1", temperature=0),
    tool=PythonAstREPLTool(),
    verbose=True,
    agent_executor_kwargs={"handle_parsing_errors": True}
)

"""把三个工具放进工具列表"""
tools = [
    Tool(
        name="python代码工具",
        description="""当你需要借助python解释器时，使用这个工具。
        用自然语言把要求给这个工具，他会生成python代码并返回代码执行的结果。""",
        func=python_agent_executor.invoke
    ),
    Tool(
        name="csv工具",
        description="""当你需要回答有关house_price.csv文件的问题时，使用这个工具。
        它接受完整的问题作为输入，在使用Pandas库计算后，返回答案。""",
        func=csv_agent_executor.invoke
    ),
    TextLengthTool()
]

prompt = hub.pull("hwchase17/structured-chat-agent")

memory = ConversationBufferMemory(
    memory_key='chat_history',
    return_messages=True
)

"""agent"""
agent = create_structured_chat_agent(
    llm=ChatOpenAI(model="moonshot-v1-8k", base_url="https://api.moonshot.cn/v1", temperature=0),
    tools=tools, # 包含三个工具的工具的列表
    prompt=prompt
)

"""agent执行器"""
agent_executor = AgentExecutor.from_agent_and_tools(
    agent=agent,
    tools=tools,
    verbose=True,
    memory=memory,
    handle_parsing_errors=True
)

agent_executor.invoke({"input": "请计算house_price.csv文件中的price列的平均值。"})

```

输出：

````terminal
> Entering new AgentExecutor chain...
```json
{
  "action": "csv工具",
  "action_input": "计算house_price.csv文件中的price列的平均值"
}
```

> Entering new AgentExecutor chain...
Thought: To calculate the average value of the 'price' column in the dataframe `df`, I need to use the `mean()` function provided by pandas.

Action: python_repl_ast
Action Input: df['price'].mean()4766729.247706422
````

