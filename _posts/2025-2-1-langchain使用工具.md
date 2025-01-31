---
title:  "langchain: 使用工具"
date:   2025-2-1 04:53:00 +0800
tags:
  - python
  - langchain
  - AI
---

让模型使用工具  
  
ReAct：Reason Action  

步骤一 推理->行动->观察  
步骤二 推理->行动->观察  
......                       
步骤N  推理->结果  
<br/>
[这里有很多提示词模版](https://smith.langchain.com/hub)  


```python
from langchain_openai import ChatOpenAI
from langchain.tools import BaseTool
from langchain import hub
from langchain.agents import create_structured_chat_agent
from langchain.agents import AgentExecutor
from langchain.memory import ConversationBufferMemory
import os

model = ChatOpenAI(model="moonshot-v1-8k", base_url="https://api.moonshot.cn/v1", temperature=0, api_key=os.getenv("OPENAI_API_KEY"))


class TextLengthTool(BaseTool):
    """
    定义工具类，继承BaseTool。name和description是必须的，ai会读取他们。
    """
    name = "文本字数计算工具"
    description = "当你被要求计算文本字数时，使用此工具。"
    
    def _run(self, text:str) -> str:
        return len(text)

"""把工具放进列表"""
tools = [TextLengthTool()]


"""
拉取 https://smith.langchain.com/hub 的提示模版"hwchase17/structured-chat-agent"
这个提示词模版让模型遵循ReAct，并把工具介绍作为变量(name和description)。
"""
prompt = hub.pull("hwchase17/structured-chat-agent")


"""初始化agent"""
agent = create_structured_chat_agent(
    llm=model, 
    tools=tools, 
    prompt=prompt
)

memory = ConversationBufferMemory(
    memory_key="chat_history", # 前面的提示词模版里记忆的变量名是chat_history
    return_messages=True
)

"""agent执行器"""
agent_executor = AgentExecutor.from_agent_and_tools(
    agent=agent, 
    tools=tools,
    memory=memory,
    handle_parsing_errors=True, # 模型没有按照ReAct框架输出，会出现解析错误，这个可以把错误发送回模型，让模型自行推理错误。
    verbose=True # 执行中行动过程日志会被打印出来
)

response = agent_executor.invoke({"input": "'今天吃了什么'，这句话的字数是多少？"})
print(response)
"""
> Entering new AgentExecutor chain...
{
  "action": "文本字数计算工具",
  "action_input": {"text": "今天吃了什么"}
}6```
{
  "action": "Final Answer",
  "action_input": "这句话的字数是6。"
}
```

> Finished chain.
{'input': "'今天吃了什么'，这句话的字数是多少？", 'chat_history': [HumanMessage(content="'今天吃了什么'，这句话的字数是多少？"), AIMessage(content='这句话的字数是6。')], 'output': '这句话的字数是6。'}
"""
```