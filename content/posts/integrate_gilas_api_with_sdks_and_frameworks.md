---
title: "اتصال فریم ورک ها و SDK ها به API گیلاس"
description: "در این پست کوتاه، نحوه پیکربندی فریم‌ورک‌ها و SDKهای مختلف برای دسترسی به API گیلاس را  بررسی می‌کنیم."
tags:
- sdk
- langchain
- llamaindex
- openai
weight: 2001
og_image: "/posts/integrate_gilas_api_with_sdks_and_frameworks/banner.png"
---

{{< postcover src="/posts/integrate_gilas_api_with_sdks_and_frameworks/banner.png" >}}


در این پست کوتاه، نحوه پیکربندی فریم‌ورک‌ها و SDKهای مختلف برای دسترسی به API گیلاس را  بررسی می‌کنیم.

در نظر داشته باشید که اگر در حال حاضر از سرویس‌دهنده‌های دیگری در کد خود استفاده می‌کنید، می‌توانید تنها با تغییر متغیرهای آدرس API و استفاده از کلید API گیلاس به راحتی به پلتفرم گیلاس منتقل شوید. هیچ کد دیگری در برنامه‌ی شما نیازی به تغییر نخواهد داشت.

برای این کار ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.


## فریم‌ورک LangChain
```python
import os
from langchain.chat_models import ChatOpenAI
from langchain.schema import (HumanMessage, SystemMessage)

chat = ChatOpenAI(
    api_key="YOUR_GILAS_API_KEY",
    base_url="https://api.gilas.io/v1",
    temperature=0)
human_message = "Translate from English to Farsi: I love playing Tennis"
chat([HumanMessage(content = human_message)])
```

به‌طور جایگزین، می‌توانید از متغیرهای محیطی استفاده کنید:
```python
import os

os.environ["OPENAI_API_KEY"] = "YOUR_GILAS_API_KEY"
os.environ["OPENAI_API_BASE"] = "https://api.gilas.io/v1"

chat = ChatOpenAI(temperature=0)
human_message = "Translate from English to Farsi: I love playing Tennis"
chat([HumanMessage(content = human_message)])
```

## فریم‌ورک Haystack
برای تنظیم API Key و base URL در Haystack، به شکل زیر عمل کنید:
```python
from haystack.components.generators import OpenAIGenerator

client = OpenAIGenerator(
    api_key="YOUR_GILAS_API_KEY",
    api_base_url="https://api.gilas.io/v1"
    model="gpt-4o")
response = client.run("What's Natural Language Processing? Be brief.")
print(response)
```

## فریم‌ورک LlamaIndex
برای LlamaIndex، می‌توانید تنظیمات را این‌گونه انجام دهید:
```python
from llama_index.llms.openai import OpenAI

llm = OpenAI(
    model="gpt-4o-mini",
    api_key="YOUR_GILAS_API_KEY",
    api_base="https://api.gilas.io/v1"
)

resp = llm.complete("Iran is ")
print(resp)
```

## فریم‌ورک Microsoft AutoGen

برای استفاده از مدل‌های ارايه شده از طریق گیلاس API در فریم‌ورک AutoGen، می‌توانید از کد زیر استفاده کنید:

```python
import os
import autogen

llm_config = {
    "config_list": [{"model": "gpt-4o", "api_key": "YOUR_GILAS_API_KEY", "base_url": "https://api.gilas.io/v1"}],
}

assistant = autogen.AssistantAgent(name="assistant", llm_config=llm_config)
```

## مجموعه SDKهای OpenAI

### برای Python
```python
from openai import OpenAI

client = OpenAI(
    api_key="YOUR_GILAS_API_KEY",
    api_base = "https://api.gilas.io/v1"
)

chat_completion = client.chat.completions.create(
    messages=[
        {
            "role": "user",
            "content": "Say this is a test",
        }
    ],
    model="gpt-4o",
)
```

### برای Node.js

```javascript
import OpenAI from 'openai';

const client = new OpenAI({
  apiKey: 'YOUR_GILAS_API_KEY',
  baseURL: 'https://api.gilas.io/v1'
});

async function main() {
  const chatCompletion = await client.chat.completions.create({
    messages: [{ role: 'user', content: 'Say this is a test' }],
    model: 'gpt-4o',
  });
}

main();
```

### برای Go
```go
package main

import (
	"context"

	"github.com/openai/openai-go"
	"github.com/openai/openai-go/option"
)

func main() {
	client := openai.NewClient(
		option.WithAPIKey("YOUR_GILAS_API_KEY"), 
        option.WithBaseURL("https://api.gilas.io/v1")
	)
	chatCompletion, err := client.Chat.Completions.New(context.TODO(), openai.ChatCompletionNewParams{
		Messages: openai.F([]openai.ChatCompletionMessageParamUnion{
			 openai.UserMessage("Say this is a test"),
		}),
		Model: openai.F(openai.ChatModelGPT4o),
	})
	if err != nil {
		panic(err.Error())
	}
	println(chatCompletion.Choices[0].Message.Content)
}
```

### برای Java
```java
import com.openai.client.OpenAIClient;
import com.openai.client.okhttp.OpenAIOkHttpClient;

OpenAIClient client = OpenAIOkHttpClient.builder()
    .apiKey("My API Key")
    .baseUrl("https://api.gilas.io/v1")
    .build();
```

##  برای Anthropic SDKs
برای پیکربندی Anthropic SDK، این مراحل را دنبال کنید:
```python
import anthropic

client = anthropic.Anthropic(
    api_key='YOUR_GILAS_API_KEY',
    base_url="https://api.gilas.io/v1"
)
message = client.messages.create(
    model="claude-3-5-sonnet-latest",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Hello, Claude"}
    ]
)
print(message.content)
```

با استفاده ازین نمونه‌ها باید بتوانید به سرعت به API GILAS متصل شوید و از آن برای پروژه‌های خود استفاده کنید.