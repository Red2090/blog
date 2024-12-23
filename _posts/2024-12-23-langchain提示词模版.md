---
layout: post
title:  "langchain提示词"
date:   2024-12-23 14:20:00 +0800
categories: jekyll update
---

```python
from langchain_openai import ChatOpenAI
from langchain.prompts import SystemMessagePromptTemplate, HumanMessagePromptTemplate, AIMessagePromptTemplate
from langchain.prompts import FewShotChatMessagePromptTemplate

"""定义系统prompt模板"""
system_template_text = """
请你充当一家外贸公司的翻译，你可以把文本从{input_language}翻译成{output_language}
翻译时请尽量保留客户原本的语气。输出内容不要有任何额外的解释或说明。
"""

"""定义用户prompt模板:"""
human_template_text = "文本：{text}\n语言风格：{style}"

"""创建模版实例"""
system_prompt_template = SystemMessagePromptTemplate.from_template(system_template_text)
human_prompt_template = HumanMessagePromptTemplate.from_template(human_template_text)

"""在实例加入变量"""
system_prompt = system_prompt_template.format(input_language="英语", output_language="中文")
human_prompt = human_prompt_template.format(text="I'm so hungry could eat a horse", style="文言文")


model = ChatOpenAI(model="moonshot-v1-8k", base_url="https://api.moonshot.cn/v1")

response = model.invoke([
    system_prompt,
    human_prompt
])
print(response.content)
```

<pre style="font-family: 'Courier New', monospace; font-size: 14px;">
<font face="Courier New">
def hello_world():
    print("Hello, world!！！！！")
</font>
</pre>