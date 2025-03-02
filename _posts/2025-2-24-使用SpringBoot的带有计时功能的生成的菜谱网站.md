---
title:  "使用SpringBoot的带有计时功能的生成菜谱的网站"
date:   2025-2-24 00:09:00 +0800
tags:
  - Java
  - SpringBoot
---

这是一个可以生成菜谱，并在烹饪步骤中帮你计时的网站。

我花费了一个周末来编写网站代码，并在周日下午把它部署到服务器。把spring boot部署到服务器异常轻松（也是按着AI的步骤来的），还不需要nginx，所以.NET那个网站还没部署，这个先部署了。

AI帮了我大部分的忙，它带我熟悉了SpringBoot的项目结构。没有AI————准确的来说，是没有deepseek r1这个等级的AI，我什么也做不了。

项目用的是SpringBoot MVC的结构，AI的部分使用的是langchain4j，langchain的Java版本。代码和prompt会放在后面记录一下。

网站没有域名，输入公网ip访问 221.12.72.203:8080



项目结构
```markdown
   src/main/java
   ├── com.cookingiseasy.cookingiseasy
   │   ├── controller          # 控制器层
   │   │   ├── ApiController    # REST API接口
   │   │   └── PageController   # 页面路由控制
   │   ├── model               # 数据模型
   │   │   ├── Recipe          # 菜谱实体
   │   │   └── CookingStep     # 烹饪步骤实体
   │   │
   │   ├── repository          # 数据访问层
   │   │   └── RecipeRepository  # JPA仓库接口
   │   │
   │   ├── service             # 服务层
   │   │   ├── IAIService        # AI服务接口
   │   │   ├── AIServiceBase    # LangChain基类
   │   │   └── MoonShotAIService # AI具体实现
   │   │
   │   └── dto                 # 数据传输对象
   │       ├── RecipeRequest   # 请求结构体
   │       └── RecipeResponse  # 响应结构体
   │
   resources
   ├── static                 # 静态资源
   │   ├── images             # 图片资源
   │   └── ring.mp3          # 提示音效
   │
   └── templates              # 模板文件
       ├── generate.html      # 生成页面
       └── index.html         # 首页
```

API
   菜谱生成接口
   端点：POST /api/generate-recipe

请求示例：
```json
{
  "dishDescription": "法式红酒炖牛肉，要求慢火烹饪"
}
```

响应格式：
```json
{
"steps": [
    ["preparation", "牛腩500g切3cm方块...", 0],
    ["timer", "红酒腌制30分钟...", 1800],
    ["action", "大火快速翻煎表面...", 0]
  ]
}
```




这是AIServiceBaseClassUsingLangChain类，他的子类可以根据不同的配置重写类中的getApiUrl()getApiKey()getModelName()getTemperature()和getSystemPrompt()方法。

```java
package com.cookingiseasy.cookingiseasy.service;

import com.cookingiseasy.cookingiseasy.dto.RecipeResponse;
import com.fasterxml.jackson.databind.ObjectMapper;
import dev.langchain4j.data.message.ChatMessage;
import dev.langchain4j.data.message.SystemMessage;
import dev.langchain4j.data.message.UserMessage;
import dev.langchain4j.model.openai.OpenAiChatModel;

import java.util.List;

public abstract class AIServiceBaseClassUsingLangChain implements IAIService{
    protected abstract String getApiUrl();
    protected abstract String getApiKey();
    protected abstract String getModelName();

    protected double getTemperature() {
        return 0.3;
    }

    @Override
    public RecipeResponse generateRecipe(String dishDescription) {
        OpenAiChatModel chatModel = buildChatModel();
        List<ChatMessage> messages = buildMessages(dishDescription);
        String answer = getAiResponse(chatModel, messages);
        return parseAnswerToResponse(answer);
    }

    protected String getSystemPrompt() {
        return """
            请根据给定的菜品名称和描述，生成制作这样的菜的步骤、用量和等待时间。要求格式如下：

                  [
                    ["preparation", "400g切厚片的雪花牛小排、30ml苹果木烟熏威士忌、20g现磨黑松露碎、预热至220℃的火山石烤盘", 0],
                    ["action", "用玫瑰盐晶体在黑金色岩石板上研磨盐粒", 0],
                    ["timer", "将牛排静置室温回温15分钟。让肉纤维自然舒展，肌红蛋白均匀分布如同大理石纹路逐渐晕染",900],
                    ["action", "将火山石烤盘烧至表面泛白热", 0],
                    ["timer", "高温炙烤牛排单面90秒。瞬间锁住粉红肉汁，焦糖化的黄金脆壳发出咔嚓轻响，油脂滴落激起云雾般的烟熏香", 90],
                    ["action", "淋入威士忌点燃蓝色火焰", 0],
                    ["timer", "火焰熄灭后静置焐热3分钟。酒精燃烧带出苹果木甜香，余温让中心呈现半透明宝石红溏心，切面如落日熔金般流淌蜜色肉汁", 180],
                    ["action", "撒入黑松露碎与金箔粉", 0],
                    ["timer", "用喷枪快速灼烧表层3秒。高温激活松露的麝香气息，金箔在热浪中闪烁，形成视觉与味觉的双重盛宴", 3]
                  ]
            
            preparation中，要求整个回复只能有一个preparation，preparation中的每一个食材都应该为(多少量的，不能用适量这样模糊不清的词语)(怎么样的，被怎么处理的)(什么东西)结构来描述，并且包括食材，佐料，和特殊的器具。
            
            timer中，要求只有涉及到煮，烧，炖，烤等需要把控时间的操作时才可以使用timer，如果这个步骤时间低于5秒，就必须把他归为action，不能用timer表示，计时的时间必须是一个明确的数字且不能是0，并且必须与timer列表最后一位相符，timer列表最后一位的单位是秒。在timer列表的第二项描述内容中，你的生成结构应该为(什么动作如关上锅盖焖煮)(时间如20分钟)，(这么做的原因和效果!!!要丰富!!!)。
            
            action中，要求一个action只能拥有一个步骤，对于多个步骤，你需要多个action列表。
            
            你只需要返回这个符合格式的列表，不需要返回其他内容。""";
    }

    protected OpenAiChatModel buildChatModel() {
        return OpenAiChatModel.builder()
                .baseUrl(getApiUrl())
                .apiKey(getApiKey())
                .modelName(getModelName())
                .temperature(getTemperature())
                .build();
    }

    protected List<ChatMessage> buildMessages(String userInput) {
        return List.of(
                SystemMessage.from(getSystemPrompt()),
                UserMessage.from(userInput)
        );
    }

    protected String getAiResponse(OpenAiChatModel chatModel, List<ChatMessage> messages) {
        return chatModel.generate(messages).content().text();
    }

    private RecipeResponse parseAnswerToResponse(String answer) {
        try {
            ObjectMapper mapper = new ObjectMapper();
            List<List<Object>> steps = mapper.readValue(answer, List.class);
            return new RecipeResponse(steps);
        } catch (Exception e) {
            throw new RuntimeException(
                    "AI响应解析失败: " + e.getMessage() + "\n原始内容:\n" + answer, e
            );
        }
    }
}

```



![图片]({{site.baseurl}}/assets/images/2025022401.png)