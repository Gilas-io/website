---
title: 'راهنمای سریع'
weight: 2
description: ''
tags:
- openai
- llm
- large language model
- هوش مصنوعی
- گیلاس
---

این راهنمای سریع به شما کمک می‌کند تا محیط توسعه لوکال خود را راه‌اندازی کنید و اولین درخواست API خود را ارسال کنید. اگر توسعه‌دهنده با تجربه‌ای هستید یا می‌خواهید فوراً از Gilas API استفاده کنید، راهنمای APIs را برای شروع مطالعه کنید. این راهنمای سریع به شما کمک می‌کند تا بتوانید:

- محیط توسعه خود را راه‌اندازی کنید
- برخی از مفاهیم اساسی Gilas API را یاد بگیرید
- اولین درخواست API خود را ارسال کنید

اگر با چالشی روبرو شدید یا سوالی در مورد نحوه کار با Gilas API دارید، می‌توانید با [پشتیبان تلگرامی](https://t.me/gilas_dev) ما تماس بگیرید.

## ساخت حساب کاربری

ابتدا، یک  [حساب کاربری جدید](https://dashboard.gilas.io) ایجاد کنید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید. مطمئن شوید که این کلید را در جای امنی ذخیره کرده و آن را با کسی به اشتراک نگذارید.

## ارسال درخواست

ابتدا کلید API ساخته شده را به محیط ترمینال یا محیط اجرای کد اضافه کنید.

```shell
# MacOS | Linux
export GILAS_API_KEY='your-api-key-here'
# Windows
setx GILAS_API_KEY "your-api-key-here"
```
حالا با استفاده از نمونه کدهای زیر اولین درخواست Chat Completions  را ارسال کنید. چون کلید API در مرحله قبل تنظیم شده است، باید به صورت خودکار از طریق $GILAS_API_KEY به برنامه ارجاع داده شود. در غیر اینصورت می‌توانید به صورت دستی $GILAS_API_KEY را با کلید API خود جایگزین کنید.


{{< tabs "quick start - code" >}}
{{< tab "curl" >}}

```shell
curl https://api.gilas.io/v1/chat/completions  \
-H "Content-Type: application/json" \
-H "Authorization: Bearer $GILAS_API_KEY" \
-d '{
    "model": "gpt-4o-mini",
    "messages": [
      {
        "role": "system",
        "content": "You are an intelligent software developer. Please answer my questions."
      },
      {
        "role": "user",
        "content": "How to write a simple HTTP call via curl on the command line?"
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
		"model": "gpt-4o-mini",
		"messages": []map[string]string{
			{"role": "system", "content": "You are an intelligent software developer. Please answer my questions."},
			{"role": "user", "content": "How to write a simple HTTP call via curl on the command line?"},
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
        model: 'gpt-4o-mini',
        messages: [
            {
                role: 'system',
                content: 'You are an intelligent software developer. Please answer my questions.'
            },
            {
                role: 'user',
                content: 'How to write a simple HTTP call via curl on the command line?'
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
        'model': 'gpt-4o-mini',
        'messages': [
            {'role': 'system', 'content': 'You are an intelligent software developer. Please answer my questions.'},
            {'role': 'user', 'content': 'How to write a simple HTTP call via curl on the command line?'}
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

## استفاده از کتابخانه‌های OpenAI و mistralai

در صورتی که با کتابخانه‌های OpenAI در زبان‌های Python و Node.js  آشنایی دارید می‌توانید با استفاده از این کتابخانه‌ها هم درخواست خود را به ‌GILAS API ارسال کنید.

ابتدا کلید API ساخته شده را به محیط ترمینال یا محیط اجرای کد اضافه کنید.

```shell
# MacOS | Linux
export GILAS_API_KEY='your-api-key-here'
# Windows
setx GILAS_API_KEY "your-api-key-here"
```

{{< tabs "quick start - package" >}}
{{< tab "Node.js library" >}}

کتابخانه OpenAI را نصب کنید.

```shell
npm install --save openai mistralai
# or
yarn add openai

```

حالا با استفاده از کتابخانه نصب شده درخواست Chat Completions خود را به Gilas ارسال کنید.

```js
import OpenAI from 'openai';

const openai = new OpenAI({
  apiKey: process.env['GILAS_API_KEY'],
  baseURL: 'https://api.gilas.io/v1/'
});

async function main() {
  const chatCompletion = await openai.chat.completions.create({
    messages: [{ role: 'user', content: 'Say this is a test' }],
    model: 'gpt-4o-mini',
  });

  console.log(chatCompletion.choices[0]);
}

main();
```
{{< /tab >}}

{{< tab "Python library" >}}

کتابخانه OpenAI را نصب کنید.

```shell
pip install --upgrade openai mistralai

```

حالا با استفاده از کتابخانه نصب شده درخواست Chat Completions خود را به Gilas ارسال کنید.

```python
import os
from openai import OpenAI

client = OpenAI(
    # This is the default and can be omitted
    api_key=os.environ.get("GILAS_API_KEY"),
    base_url="https://api.gilas.io/v1/"
)

chat_completion = client.chat.completions.create(
    messages=[
        {
            "role": "user",
            "content": "Say this is a test",
        }
    ],
    model="gpt-4o-mini",
)

print(chat_completion.choices[0].message)
```
{{< /tab >}}
{{< /tabs >}}

## قدم بعدی

حال که به طور خلاصه با نحوه کار با مدلهای زبانی بزرگ آشنا شدید٬ پیشنهاد می‌کنیم که راهنمای ‌API مربوط به اندپوینت های مختلف را مطالعه کنید:

- راهنمای اندپوینت [`v1/chat/completions`](/apis/chat-completions/)  برای تولید متن
- راهنمای اندپوینت [`v1/embeddings`](/apis/embeddings/) برای تولید embeddings برای متون
- راهنمای اندپوینت [`v1/audio`](/apis/audio/) برای تبدیل متن به صوت و برعکس
- راهنمای اندپوینت [`v1/moderations`](/apis/moderations/) برای مدیریت متون ورودی و تولید شده

همچنین می‌توانید [بلاگ](/posts/) ما را برای مطالعه‌ی مثال های واقعی استفاده از هوش مصنوعی یا مدل‌های زبانی بزرگ از طریق Gilas API برای انجام کارهای بسیار متنوع و جذاب بررسی کنید.