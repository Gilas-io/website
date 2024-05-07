---
title: "فرمت‌ ورودی و خروجی داده ها"
description: "مدل‌های چت، یگ سری از پیام‌ها را به عنوان ورودی می‌پذیرند و یک پیام نوشته شده توسط AI را به عنوان خروجی برمی‌گردانند.
این راهنما با چند نمونه فراخوانی API فرمت چت را نشان می‌دهد."
tags:
- chat-completions
weight: 2007
og_image: "/posts/how_to_format_inputs_to_chatgpt_models/banner.jpg" 
---

{{< postcover src="/posts/how_to_format_inputs_to_chatgpt_models/banner.jpg" >}}


مدل‌های چت، یگ سری از پیام‌ها را به عنوان ورودی می‌پذیرند و یک پیام نوشته شده توسط AI را به عنوان خروجی برمی‌گردانند.
این راهنما با چند نمونه فراخوانی API فرمت چت را نشان می‌دهد.



{{< hint info >}}
**برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.**
{{< /hint >}} 

```python
# if needed, install and/or upgrade to the latest version of the OpenAI Python library
%pip install --upgrade openai
```

```python
# imports
from openai import OpenAI # for calling the OpenAI API
import os # for getting API token from env variable OPENAI_API_KEY

client = OpenAI(
    api_key=os.environ.get(("GILAS_API_KEY", "<کلید API خود را اینجا بسازید https://dashboard.gilas.io/apiKey>")), 
    base_url="https://api.gilas.io/v1/" # Gilas APIs
)
```


## مثالی از فراخوانی Chat Completions API

برای فراخوانی Chat Completions API پارامترهای زیر باید به مدل ارسال شوند:

- `model`: نام مدلی که می‌خواهید استفاده کنید (مانند `gpt-3.5-turbo`, `gpt-4-turbo`)
- `messages`: لیستی از شیء‌های پیام، جایی که هر شیء دو فیلد ضروری دارد:
    - `role`: نقش فرستنده پیام (`system`، `user`، `assistant` یا `tool`)
    - `content`: محتوای پیام (مثلاً نوشتن یک شعر زیبا)

پیام‌ها همچنین می‌توانند یک فیلد اختیاری `name` را شامل شوند، که به فرستنده نام می‌دهد. مثلاً `example-user`، `Hichkas`. نام‌ها نباید شامل فاصله باشند.

پارامترهای اختیاری:

- `frequency_penalty`: توکن‌ها را بر اساس تعداد تکرارشان جریمه می‌کند، در نتیجه تکرار توکن‌ها را کم می‌کند.
- `logit_bias`: احتمال توکن‌های مشخصی را با مقادیر bias تغییر می‌دهد.
- `logprobs`: اگر `true` باشد، log احتمالات توکن‌های خروجی را برمی‌گرداند.
- `top_logprobs`: تعداد توکن‌های بیشتر محتمل را برای بازگرداندن در هر موقعیت مشخص می‌کند.
- `max_tokens`: حداکثر تعداد توکن‌های تولید شده در پاسخ را تنظیم می‌کند.
- `n`: تعداد مشخصی از پاسخ را برای هر ورودی تولید می‌کند.
- `presence_penalty`: توکن‌های جدید را بر اساس حضورشان در متن جریمه می‌کند.
- `response_format`: فرمت خروجی را مشخص می‌کند، مثلاً حالت JSON.
- `seed`: با یک `seed` مشخص، نمونه‌گیری قطعی را تضمین می‌کند.
- `stop`: تا 4 دنباله را مشخص می‌کند که API باید در آن‌ها تولید توکن را متوقف کند.
- `stream`: دلتاهای پیام‌های جزئی را به محض تولید توکن جدید می‌فرستد.
- `temperature`: درجه نمونه‌برداری را بین 0 و 2 تنظیم می‌کند.
- `top_p`: از نمونه‌برداری هسته استفاده می‌کند؛ توکن‌های با top_p حجم احتمال را در نظر می‌گیرد.
- `tools`: فهرست توابعی که مدل ممکن است صدا بزند.
- `tool_choice`: تماس‌های تابع مدل را کنترل می‌کند (none/auto/function).
- `user`: شناسه یکتا برای نظارت بر کاربر نهایی و تشخیص سوء استفاده.

از ژانویه 2024، می‌توانید یک لیست از توابع را ارسال کنید که به GPT بگوید آیا می‌تواند یک JSON را برای ورودی به یک تابع تولید کند. [این پست](/how_to_call_functions_with_chat_models) مثال‌های جالبی را در مورد نحوه فراخوانی توابع با استفاده از LLM ارایه می‌دهد.


معمولاً، یک گفتگو با پیام `system` آغاز می‌شود که به `assistant` می‌گوید چگونه رفتار کند، و پس از آن پیام‌های `user` و `assistant` به نوبت دنبال می‌شوند، هر چند مجبور به دنبال کردن این فرمت نیستید. ما معمولا با استفاده از پیام `system` مدل را برنامه ریزی یا به عبارتی مهندسی پرامپت می‌کنیم.


بیایید نگاهی به یک نمونه فراخوانی API چت بیندازیم تا ببینیم فرمت چت در عمل چگونه کار می‌کند.

```python
# Example OpenAI Python library request
MODEL = "gpt-3.5-turbo"
response = client.chat.completions.create(
    model=MODEL,
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "نوروز چه روزی است؟"},
    ],
    temperature=0,
)

print(json.dumps(json.loads(response.model_dump_json()), indent=4))
```

<div style="direction: ltr" >

```
{
    "id": "chatcmpl-8dee9DuEFcg2QILtT2a6EBXZnpirM",
    "choices": [
        {
            "finish_reason": "stop",
            "index": 0,
            "logprobs": null,
            "message": {
                "content": "نوروز اولین روز فصل بهار است.",
                "role": "assistant",
                "function_call": null,
                "tool_calls": null
            }
        }
    ],
    "created": 1704461729,
    "model": "gpt-3.5-turbo-0613",
    "object": "chat.completion",
    "system_fingerprint": null,
    "usage": {
        "completion_tokens": 3,
        "prompt_tokens": 35,
        "total_tokens": 38
    }
}
```

</div>

همانطور که می‌بینید، شیء پاسخ دارای چند فیلد است:

- `id`: شناسه درخواست
- `choices`: لیستی از اشیاء تکمیل (فقط یکی، مگر اینکه شما n را بیشتر از 1 تنظیم کنید)
    - `finish_reason`: دلیلی که مدل تولید متن را متوقف کرده است (یا `stop`، یا `length` اگر حد `max_tokens` رسیده باشد)
    - `index`: ایندکس انتخاب در لیست.
    - `logprobs`: اطلاعات احتمال لاگ برای انتخاب.
    - `message`: شیء پیام تولید شده توسط مدل
        - `content`: محتوای پیام
        - `role`: نقش نویسنده این پیام.
        - `tool_calls`: فراخوانی‌های ابزار تولید شده توسط مدل، مانند فذاخوانی‌های تابع. اگر ابزار داده شده باشد
- `created`: زمان درخواست
- `model`: نام کامل مدل استفاده شده برای تولید پاسخ
- `object`: نوع شیء برگشتی (مثلاً chat.completion)
- `system_fingerprint`:  اثر انگشت، پیکربندی پشتیبانی که مدل با آن اجرا می‌شود را نشان می‌دهد.
- `usage`: تعداد توکن‌های استفاده شده برای تولید پاسخ‌ها، شامل پرامپت، تکمیل و کل


برای اینکه فقط پاسخ مدل را ببینید:

```python
response.choices[0].message.content

نوروز اولین روز فصل بهار است.
```

## Few-shot prompting

در برخی موارد، راحت‌تر است که آنچه را که می‌خواهید به مدل نشان دهید. 
یک راه برای انجام این کار، نشان دادن چند نمونه از روش حل یک مسیله به مدل است.

برای مثال:

```python
# An example of a faked few-shot conversation to prime the model into translating business jargon to simpler speech
response = client.chat.completions.create(
    model=MODEL,
    messages=[
        {"role": "system", "content": "You are a helpful, pattern-following assistant."},
        {"role": "user", "content": "Help me translate the following corporate jargon into plain English."},
        {"role": "assistant", "content": "Sure, I'd be happy to!"},
        {"role": "user", "content": "New synergies will help drive top-line growth."},
        {"role": "assistant", "content": "Things working well together will increase revenue."},
        {"role": "user", "content": "Let's circle back when we have more bandwidth to touch base on opportunities for increased leverage."},
        {"role": "assistant", "content": "Let's talk later when we're less busy about how to do better."},
        {"role": "user", "content": "This late pivot means we don't have time to boil the ocean for the client deliverable."},
    ],
    temperature=0,
)

print(response.choices[0].message.content)
```

```
Fractions represent parts of a whole. They have a numerator (top number) and a denominator (bottom number).
```

برای کمک به روشن کردن اینکه پیام‌های نمونه بخشی از یک گفتگوی واقعی نیستند و نباید توسط مدل به آنها ارجاع داده شود، می‌توانید سعی کنید فیلد `name` پیام‌های `system` را به `example_user` و `example_assistant` تنظیم کنید.

مثال:

```python
# The business jargon translation example, but with example names for the example messages
response = client.chat.completions.create(
    model=MODEL,
    messages=[
        {"role": "system", "content": "You are a helpful, pattern-following assistant that translates corporate jargon into plain English."},
        {"role": "system", "name":"example_user", "content": "New synergies will help drive top-line growth."},
        {"role": "system", "name": "example_assistant", "content": "Things working well together will increase revenue."},
        {"role": "system", "name":"example_user", "content": "Let's circle back when we have more bandwidth to touch base on opportunities for increased leverage."},
        {"role": "system", "name": "example_assistant", "content": "Let's talk later when we're less busy about how to do better."},
        {"role": "user", "content": "This late pivot means we don't have time to boil the ocean for the client deliverable."},
    ],
    temperature=0,
)

print(response.choices[0].message.content)
```

```
This sudden change in direction means we don't have enough time to complete the entire project for the client.
```

معمولا برای رسیدن به نتیجه مطلوب باید روش‌های مختلف را همراه با پیام‌های گوناگون امتحان کنید. در صورت علاقه می‌توانید این [دوره کامل مهندسی پرامپت](https://www.youtube.com/playlist?list=PLKI4_lXzsRRf_DNrqdzFBdV-VqknLGbZ7) را تماشا کنید.


## شمارش توکن‌ها

وقتی درخواست خود را ارسال می‌کنید، API پیام‌ها را به دنباله‌ای از توکن‌ها تبدیل می‌کند.

تعداد توکن‌های استفاده شده بر:
- هزینه درخواست
- زمان لازم برای تولید پاسخ
تاثیر می‌گذارد.

برای مطالعه بیشتر درباره شمارش توکن‌ها پست [شمردن تعداد توکن‌ها با tiktoken](/how_to_count_tokens_with_tiktoken) مطالعه کنید.
