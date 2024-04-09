---
title: 'v1/chat/completions/'
weight: 1000
---

{{< hint info >}}
**توجه**  
در نظر داشته باشید که Gilas APIs از لحاظ فنی و نحوه کارکرد و قابلیت‌ها کاملا شبیه OpenAI APIs هستند. به همین منظور پیشنهاد میکنیم که برای آگاهی از نحوه‌ی کارکرد API ها به مستندات [OpenAI API Reference](https://platform.openai.com/docs/api-reference/chat) و [OpenAI Documentation](https://platform.openai.com/docs/guides/text-generation) ارجاع کنید.
{{< /hint >}}

مدل‌های تولید متن قابل دسترس از طریق Gilas API توانایی درک زبان طبیعی و کدنویسی را دارند. این مدل‌ها به ورودی‌های خود با خروجی‌های متنی که بسیار شبیه به جواب‌های انسانی هستند پاسخ می‌دهند. ورودی‌های این مدل‌ها همچنین به عنوان "prompt" شناخته می‌شوند. طراحی یک پرامپت اساساً چگونگی "برنامه‌ریزی" یک مدل زبانی بزرگ است.

شما می‌توانید با استفاده از این مدل‌ها برنامه‌هایی بسازید که توانایی انجام کارهای زیر را دارند:
-	پیش‌نویس اسناد
-	نوشتن کد کامپیوتری
-	پاسخ به سوالات درباره پایگاه داده‌های متنی
-	تحلیل متون
-	ارائه رابط زبان طبیعی 
-	تعلیم علوم مختلف به زبان انسانی
-	ترجمه زبان‌های مختلف
-	شبیه‌سازی شخصیت‌ها برای بازی‌های کامپیوتری

برای استفاده از این مدل‌ها کافی است که یک درخواست شامل ورودی‌ها و کلید API خود را به اندپوینت مناسب ارسال کنید.

## Chat Completions API

مدل‌های تولید متن لیستی از پیام‌ها را به عنوان ورودی دریافت کرده و یک پیام تولید شده توسط مدل را به عنوان خروجی ارائه می‌دهند. اگرچه اندپویت چت برای ایجاد گفتگوهای چند مرحله‌ای طراحی شده است، اما از این اندپوینت  برای انجام وظایف  تک‌ مرحله‌ای بدون هیچ گفتگویی نیز می‌توان استفاده کرد.

نمونه‌ای از فواخوانی  APIی چت را در زیر مشاهده کنید:


{{< tabs "quick start - code" >}}
{{< tab "curl" >}}

```shell
curl https://api.gilas.io/v1/chat/completions  \
-H "Content-Type: application/json" \
-H "Authorization: Bearer $GILAS_API_KEY" \
-d '{
    "model": "gpt-3.5-turbo",
    "messages": [
      {
        "role": "system",
        "content": "You are a helpful assistant."
      },
      {
        "role": "user",
        "content": "Who won the world series in 2020?"
      },
      {
        "role": "assistant",
        "content": "The Los Angeles Dodgers won the World Series in 2020."
      },
      {
        "role": "user",
        "content": "Where was it played?"
      }
    ]
  }'
```

{{< /tab >}}
{{< tab "Go" >}}


```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io/ioutil"
	"net/http"
	"os"
)

func main() {
	url := "https://api.gilas.io/v1/chat/completions"
	payload := map[string]interface{}{
		"model": "gpt-3.5-turbo",
		"messages": []map[string]string{
			{
				"role":    "system",
				"content": "You are a helpful assistant.",
			},
			{
				"role":    "user",
				"content": "Who won the world series in 2020?",
			},
			{
				"role":    "assistant",
				"content": "The Los Angeles Dodgers won the World Series in 2020.",
			},
			{
				"role":    "user",
				"content": "Where was it played?",
			},
		},
	}

	jsonPayload, _ := json.Marshal(payload)
	req, _ := http.NewRequest("POST", url, bytes.NewBuffer(jsonPayload))
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("Authorization", "Bearer "+os.Getenv("GILAS_API_KEY"))

	client := &http.Client{}
	resp, _ := client.Do(req)
	defer resp.Body.Close()

	body, _ := ioutil.ReadAll(resp.Body)
	fmt.Println(string(body))
}


```

{{< /tab >}}

{{< tab "Node.js" >}}

```js
const axios = require('axios');

async function makeRequest() {
    const url = 'https://api.gilas.io/v1/chat/completions';
    const payload = {
        model: 'gpt-3.5-turbo',
        "messages": [
            {
                "role": "system",
                "content": "You are a helpful assistant."
            },
            {
                "role": "user",
                "content": "Who won the world series in 2020?"
            },
            {
                "role": "assistant",
                "content": "The Los Angeles Dodgers won the World Series in 2020."
            },
            {
                "role": "user",
                "content": "Where was it played?"
            }
        ]
    };

    try {
        const response = await axios.post(url, payload, {
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${process.env.GILAS_API_KEY}`
            }
        });
        console.log(response.data);
    } catch (error) {
        console.error('Error making the request:', error);
    }
}

makeRequest();
```
{{< /tab >}}

{{< tab "Python" >}}


```python
import requests
import os

def make_request():
    url = 'https://api.gilas.io/v1/chat/completions'
    headers = {
        'Content-Type': 'application/json',
        'Authorization': f'Bearer {os.getenv("GILAS_API_KEY")}'
    }
    payload = {
        'model': 'gpt-3.5-turbo',
        "messages": [
            {
                "role": "system",
                "content": "You are a helpful assistant."
            },
            {
                "role": "user",
                "content": "Who won the world series in 2020?"
            },
            {
                "role": "assistant",
                "content": "The Los Angeles Dodgers won the World Series in 2020."
            },
            {
                "role": "user",
                "content": "Where was it played?"
            }
        ]
    }

    response = requests.post(url, json=payload, headers=headers)
    return response.json()

if __name__ == "__main__":
    result = make_request()
    print(result)

```
{{< /tab >}}
{{< /tabs >}}

ورودی اصلی در این فراخوان API پارامتر Messages است. Messages باید یک آرایه از شیء پیام باشند، جایی که هر شیء شامل role (با مقادیر "system"، "user" یا "assistant") و content است. چت ها می‌توانند به عنوان یک پیام کوتاه یا چندین نوبت پشت سر هم قرار بگیرند.

به طور معمول، یک گفتگو با یک پیام سیستمی شروع می‌شود، سپس پیام‌های کاربر و دستیار به تناوب دنبال می‌شوند.
پیام سیستمی به تنظیم رفتار assistant کمک می‌کند. به عنوان مثال، می‌توانید شخصیت assistant را تغییر دهید یا دستورالعمل‌های خاصی را در مورد چگونگی رفتار آن در طول گفتگو ارائه دهید. با این حال توجه داشته باشید که پیام سیستمی اختیاری است.

پیام‌های کاربر درخواست‌ها یا نظراتی را به assistant برای دریافت پاسخ فراهم می‌کنند. پیام‌های assistant پاسخ‌های قبلی assistant را ذخیره می‌کنند، اما همچنین می‌توانند توسط شما نوشته شوند تا نمونه‌های رفتار مورد نیاز را شبیه‌سازی کنند.

زمانی که دستورالعمل‌های کاربر به پیام‌های قبلی ارجاع می‌دهند قرار دادن تاریخچه مکالمه در لیست Messages مهم است. در مثال بالا، سوال نهایی کاربر در مورد “?Where was is played“ تنها در کانتکست پیام‌های قبلی در مورد سری جهانی 2020 معنی دارد. از آنجا که مدل‌ها stateless هستند و درخواست‌های قبلی را ذخیره نمی‌کنند، تمام اطلاعات مربوطه باید به عنوان بخشی از تاریخچه گفتگو در هر درخواست فراهم شود. اگر طول Messages نتواند در محدودیت توکن مدل جا بگیرد، باید به شکلی کوتاه شود.

## Chat Completions response format

 نمونه از فرمت پاسخ Chat Completions API در زیر آمده است:

```shell
{
  "choices": [
    {
      "finish_reason": "stop",
      "index": 0,
      "message": {
        "content": "The 2020 World Series was played in Texas at Globe Life Field in Arlington.",
        "role": "assistant"
      },
      "logprobs": null
    }
  ],
  "created": 1677664795,
  "id": "chatcmpl-7QyqpwdfhqwajicIEznoc6Q47XAyW",
  "model": "gpt-3.5-turbo-0613",
  "object": "chat.completion",
  "usage": {
    "completion_tokens": 17,
    "prompt_tokens": 57,
    "total_tokens": 74
  }
}
```

 پاسخ assistant در مسیر choices[0].message.content قرار دارد.


برای آگاهی کامل از قابلیت‌های Chat Completions API لطفا  [مستندات وب‌سایت OpenAI](https://platform.openai.com/docs/guides/text-generation) را مطالعه کنید.


همچنین  [OpenAI API Reference](https://platform.openai.com/docs/api-reference/chat) شامل مستندات مربوط به Chat Completions API می‌باشد که مطالعه آنها برای استفاده از این اندپوینت بسیار اهمیت دارد.

