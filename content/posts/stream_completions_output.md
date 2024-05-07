---
title: "استریم کردن خروجی تولید متن"
description: "به طور پیش فرض، هنگامی که شما از Gilas API درخواست تولید متن می کنید، کل متن تولید شده در یک رسپانس به کلاینت برگردانده می شود. اگر قصد تولید خروجی‌های طولانی را دارید٬ تولید متن می تواند چندین ثانیه طول بکشد. برای دریافت سریعتر پاسخ ها٬ می توانید متن را در حالی که تولید می شود 'stream' کنید. در این صورت پاسخ های ناتمام را به صورت استریم از سمت سرور دریافت خواهید کرد."
tags:
- openai
- streaming
- gilas.io
- blog
weight: 2006
og_image: "/posts/stream_completions_output/banner.png" 
---

{{< postcover src="/posts/stream_completions_output/banner.png" >}}


به طور پیش فرض، هنگامی که شما از Gilas API درخواست تولید متن می کنید، کل متن تولید شده در یک رسپانس به کلاینت برگردانده می شود. اگر قصد تولید خروجی‌های طولانی را دارید٬ تولید متن می تواند چندین ثانیه طول بکشد.
برای دریافت سریعتر پاسخ ها٬ می توانید متن را در حالی که تولید می شود `stream` کنید. در این صورت پاسخ های
ناتمام را به صورت استریم از سمت سرور دریافت خواهید کرد.

{{< hint warning >}} 
توجه: ورودی‌های داده شده و خروجی‌های تولید شده توسط مدل در این مثال به زبان انگلیسی هستند. برای تولید خروجی به زبان فارسی٬ کافی‌ست از مدل بخواهید که خروجی را به زبان فارسی تولید کند.
{{< /hint >}} 

برای استریم کردن متن ها، هنگام فراخوانی `v1/chat/completions` مقدار پارامتر `stream=true` را تنظیم کنید.
این درخواست یک شیء برمی گرداند که پاسخ را به عنوان [data-only server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events#event_stream_format) برگشت می دهد. 

توجه داشته باشید که استفاده از `stream=true` باعث می شود که مدیریت محتوای متن دشوارتر شود،
زیرا کار ارزیابی تکه های جزئی متن ممکن است دشوارتر باشند. مخصوصا زمانی که قصد تولید خروجی `JSON` را دارید توصیه می‌شود از حالت استریم استفاده نکنید.

یکی دیگر از معایب پاسخ های استریم این است که پاسخ دیگر شامل فیلد `usage` نیست که به شما بگوید چند توکن مصرف شده است. 
برای حل این مشکل٬ پس از دریافت و ترکیب همه پاسخ ها، می توانید تعداد توکن‌های مصرف شده را با استفاده از [tiktoken](/posts/how_to_count_tokens_with_tiktoken) محاسبه کنید.

 
  آنچه که در این نوت‌بوک یاد خواهید گرفت:
  
   1. خروجی معمولی اندپوینت تولید متن چگونه است 
   
   2. خروجی استریم اندپوینت تولید متن چگونه است
   
   3. چقدر زمان با streaming تولید متن صرفه جویی می شود

{{< hint info >}}
**برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.**
{{< /hint >}} 

 ```python
import time
from openai import OpenAI
import os

client = OpenAI(
    api_key=os.environ.get(("GILAS_API_KEY", "<کلید API خود را اینجا بسازید https://dashboard.gilas.io/apiKey>")), 
    base_url="https://api.gilas.io/v1/" # Gilas APIs
)
 ```

## خروجی معمولی اندپوینت تولید متن چگونه است 

 با فراخوانی API ChatCompletions ابتدا تمامی متن تولید می شود و سپس همه آن یکباره برگشت داده می شود.

پاسخ می تواند از مسیر `response.choices[0].message` استخراج شود. 

و محتوای پاسخ می تواند از مسیر `response.choices[0].message.content` استخراج شود.

```python
# Example of an OpenAI ChatCompletion request
# https://platform.openai.com/docs/guides/text-generation/chat-completions-api

# record the time before the request is sent
start_time = time.time()

# send a ChatCompletion request to count to 100
response = client.chat.completions.create(
    model='gpt-3.5-turbo',
    messages=[
        {'role': 'user', 'content': 'Count to 100, with a comma between each number and no newlines. E.g., 1, 2, 3, ...'}
    ],
    temperature=0,
)
# calculate the time it took to receive the response
response_time = time.time() - start_time

# print the time delay and text received
print(f"Full response received {response_time:.2f} seconds after request")
print(f"Full response received:\n{response}")
```

خروجی:

```
Full response received 5.27 seconds after request
Full response received:
ChatCompletion(id='chatcmpl-8ZB8ywkV5DuuJO7xktqUcNYfG8j6I', choices=[Choice(finish_reason='stop', index=0, logprobs=None, message=ChatCompletionMessage(content='1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99, 100.', role='assistant', function_call=None, tool_calls=None))], created=1703395008, model='gpt-3.5-turbo', object='chat.completion', system_fingerprint=None, usage=CompletionUsage(completion_tokens=299, prompt_tokens=36, total_tokens=335))
```

همچنین اطلاعات مربوط به تعداد تون مصرف شده برای تولید این متن از مسیر `response.choices[0].message.content` قابل دسترس است.

## خروجی استریم اندپوینت تولید متن چگونه است

 با فراخوانی  API ChatCompletions به صورت استریم، پاسخ به طور تدریجی در تکه هایی به نام chunk از طریق یک [event stream](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events#event_stream_format) برگشت داده می شود. در پایتون، شما می توانید با استفاده از یک حلقه for روی این تکه ها ایتریت کنید تا در نهایت بتوانید متن کامل رو بسازید.

 ```python
# For more info https://platform.openai.com/docs/api-reference/streaming#chat/create-stream

# a ChatCompletion request
response = client.chat.completions.create(
    model='gpt-3.5-turbo',
    messages=[
        {'role': 'user', 'content': "What's 1+1? Answer in one word."}
    ],
    temperature=0,
    stream=True  # this time, we set stream=True
)

for chunk in response:
    print(chunk)
    print(chunk.choices[0].delta.content)
    print("****************")
 ```

خروجی:

{{< scrollbox >}}

```
ChatCompletionChunk(id='chatcmpl-8ZB9m2Ubv8FJs3CIb84WvYwqZCHST', choices=[Choice(delta=ChoiceDelta(content='', function_call=None, role='assistant', tool_calls=None), finish_reason=None, index=0, logprobs=None)], created=1703395058, model='gpt-3.5-turbo-0613', object='chat.completion.chunk', system_fingerprint=None)

****************
ChatCompletionChunk(id='chatcmpl-8ZB9m2Ubv8FJs3CIb84WvYwqZCHST', choices=[Choice(delta=ChoiceDelta(content='2', function_call=None, role=None, tool_calls=None), finish_reason=None, index=0, logprobs=None)], created=1703395058, model='gpt-3.5-turbo-0613', object='chat.completion.chunk', system_fingerprint=None)
2
****************
ChatCompletionChunk(id='chatcmpl-8ZB9m2Ubv8FJs3CIb84WvYwqZCHST', choices=[Choice(delta=ChoiceDelta(content=None, function_call=None, role=None, tool_calls=None), finish_reason='stop', index=0, logprobs=None)], created=1703395058, model='gpt-3.5-turbo-0613', object='chat.completion.chunk', system_fingerprint=None)
None
****************
```

{{< /scrollbox >}}


همانطور که می بینید، پاسخ های streaming یک فیلد delta به جای فیلد message دارند. delta می تواند چیزهایی مانند:

- یک توکن نقش (مثلا، {"role": "assistant"}) 
- یک توکن محتوا (مثلا، {"content": "\n\n"}) 
- هیچ چیز (مثلا، {}) وقتی که استریم تمام شده است

را شامل باشد.

## چقدر زمان با streaming تولید متن صرفه جویی می شود

حالا بیایید از gpt-3.5-turbo بخواهیم دوباره تا 100 بشمارد و جواب ها رو به صورت استریم ارسال کند تا ببینیم این کار چقدر طول می کشد.

```python
# record the time before the request is sent
start_time = time.time()

# send a ChatCompletion request to count to 100
response = client.chat.completions.create(
    model='gpt-3.5-turbo',
    messages=[
        {'role': 'user', 'content': 'Count to 100, with a comma between each number and no newlines. E.g., 1, 2, 3, ...'}
    ],
    temperature=0,
    stream=True  # again, we set stream=True
)
# create variables to collect the stream of chunks
collected_chunks = []
collected_messages = []
# iterate through the stream of events
for chunk in response:
    chunk_time = time.time() - start_time  # calculate the time delay of the chunk
    collected_chunks.append(chunk)  # save the event response
    chunk_message = chunk.choices[0].delta.content  # extract the message
    collected_messages.append(chunk_message)  # save the message
    print(f"Message received {chunk_time:.2f} seconds after request: {chunk_message}")  # print the delay and text

# print the time delay and text received
print(f"Full response received {chunk_time:.2f} seconds after request")
# clean None in collected_messages
collected_messages = [m for m in collected_messages if m is not None]
full_reply_content = ''.join([m for m in collected_messages])
print(f"Full conversation received: {full_reply_content}")

```

{{< scrollbox >}}

```
Message received 0.31 seconds after request: 
Message received 0.31 seconds after request: 1
Message received 0.34 seconds after request: ,
Message received 0.34 seconds after request:  
Message received 0.34 seconds after request: 2
Message received 0.39 seconds after request: ,
Message received 0.39 seconds after request:  
Message received 0.39 seconds after request: 3
Message received 0.42 seconds after request: ,
Message received 0.42 seconds after request:  
Message received 0.42 seconds after request: 4
Message received 0.47 seconds after request: ,
Message received 0.47 seconds after request:  
Message received 0.47 seconds after request: 5
Message received 0.51 seconds after request: ,
Message received 0.51 seconds after request:  
Message received 0.51 seconds after request: 6
Message received 0.55 seconds after request: ,
Message received 0.55 seconds after request:  
Message received 0.55 seconds after request: 7
Message received 0.59 seconds after request: ,
Message received 0.59 seconds after request:  
Message received 0.59 seconds after request: 8
Message received 0.63 seconds after request: ,
Message received 0.63 seconds after request:  
Message received 0.63 seconds after request: 9
Message received 0.67 seconds after request: ,
Message received 0.67 seconds after request:  
Message received 0.67 seconds after request: 10
Message received 0.71 seconds after request: ,
Message received 0.71 seconds after request:  
Message received 0.71 seconds after request: 11
Message received 0.75 seconds after request: ,
Message received 0.75 seconds after request:  
Message received 0.75 seconds after request: 12
Message received 0.98 seconds after request: ,
Message received 0.98 seconds after request:  
Message received 0.98 seconds after request: 13
Message received 1.02 seconds after request: ,
Message received 1.02 seconds after request:  
Message received 1.02 seconds after request: 14
Message received 1.04 seconds after request: ,
Message received 1.04 seconds after request:  
Message received 1.04 seconds after request: 15
Message received 1.08 seconds after request: ,
Message received 1.08 seconds after request:  
Message received 1.08 seconds after request: 16
Message received 1.12 seconds after request: ,
Message received 1.12 seconds after request:  
Message received 1.12 seconds after request: 17
Message received 1.16 seconds after request: ,
Message received 1.16 seconds after request:  
Message received 1.16 seconds after request: 18
Message received 1.19 seconds after request: ,
Message received 1.19 seconds after request:  
Message received 1.19 seconds after request: 19
Message received 1.23 seconds after request: ,
Message received 1.23 seconds after request:  
Message received 1.23 seconds after request: 20
Message received 1.27 seconds after request: ,
Message received 1.27 seconds after request:  
Message received 1.27 seconds after request: 21
Message received 1.31 seconds after request: ,
Message received 1.31 seconds after request:  
Message received 1.31 seconds after request: 22
Message received 1.35 seconds after request: ,
Message received 1.35 seconds after request:  
Message received 1.35 seconds after request: 23
Message received 1.39 seconds after request: ,
Message received 1.39 seconds after request:  
Message received 1.39 seconds after request: 24
Message received 1.43 seconds after request: ,
Message received 1.43 seconds after request:  
Message received 1.43 seconds after request: 25
Message received 1.47 seconds after request: ,
Message received 1.47 seconds after request:  
Message received 1.47 seconds after request: 26
Message received 1.51 seconds after request: ,
Message received 1.51 seconds after request:  
Message received 1.51 seconds after request: 27
Message received 1.55 seconds after request: ,
Message received 1.55 seconds after request:  
Message received 1.55 seconds after request: 28
Message received 1.59 seconds after request: ,
Message received 1.59 seconds after request:  
Message received 1.59 seconds after request: 29
Message received 1.59 seconds after request: ,
Message received 1.59 seconds after request:  
Message received 1.59 seconds after request: 30
Message received 1.59 seconds after request: ,
Message received 1.59 seconds after request:  
Message received 1.59 seconds after request: 31
Message received 1.59 seconds after request: ,
Message received 1.59 seconds after request:  
Message received 1.60 seconds after request: 32
Message received 1.60 seconds after request: ,
Message received 1.60 seconds after request:  
Message received 1.60 seconds after request: 33
Message received 1.60 seconds after request: ,
Message received 1.60 seconds after request:  
Message received 1.67 seconds after request: 34
Message received 1.67 seconds after request: ,
Message received 1.67 seconds after request:  
Message received 1.68 seconds after request: 35
Message received 1.68 seconds after request: ,
Message received 1.68 seconds after request:  
Message received 1.86 seconds after request: 36
Message received 1.86 seconds after request: ,
Message received 1.86 seconds after request:  
Message received 1.90 seconds after request: 37
Message received 1.90 seconds after request: ,
Message received 1.90 seconds after request:  
Message received 1.94 seconds after request: 38
Message received 1.94 seconds after request: ,
Message received 1.94 seconds after request:  
Message received 1.98 seconds after request: 39
Message received 1.98 seconds after request: ,
Message received 1.98 seconds after request:  
Message received 2.05 seconds after request: 40
Message received 2.05 seconds after request: ,
Message received 2.05 seconds after request:  
Message received 2.09 seconds after request: 41
Message received 2.09 seconds after request: ,
Message received 2.09 seconds after request:  
Message received 2.14 seconds after request: 42
Message received 2.14 seconds after request: ,
Message received 2.14 seconds after request:  
Message received 2.14 seconds after request: 43
Message received 2.14 seconds after request: ,
Message received 2.14 seconds after request:  
Message received 2.14 seconds after request: 44
Message received 2.14 seconds after request: ,
Message received 2.14 seconds after request:  
Message received 2.14 seconds after request: 45
Message received 2.14 seconds after request: ,
Message received 2.14 seconds after request:  
Message received 2.15 seconds after request: 46
Message received 2.15 seconds after request: ,
Message received 2.15 seconds after request:  
Message received 2.30 seconds after request: 47
Message received 2.30 seconds after request: ,
Message received 2.30 seconds after request:  
Message received 2.30 seconds after request: 48
Message received 2.30 seconds after request: ,
Message received 2.30 seconds after request:  
Message received 2.30 seconds after request: 49
Message received 2.30 seconds after request: ,
Message received 2.30 seconds after request:  
Message received 2.31 seconds after request: 50
Message received 2.31 seconds after request: ,
Message received 2.31 seconds after request:  
Message received 2.39 seconds after request: 51
Message received 2.39 seconds after request: ,
Message received 2.39 seconds after request:  
Message received 2.40 seconds after request: 52
Message received 2.40 seconds after request: ,
Message received 2.40 seconds after request:  
Message received 2.48 seconds after request: 53
Message received 2.48 seconds after request: ,
Message received 2.48 seconds after request:  
Message received 2.49 seconds after request: 54
Message received 2.49 seconds after request: ,
Message received 2.49 seconds after request:  
Message received 2.68 seconds after request: 55
Message received 2.68 seconds after request: ,
Message received 2.68 seconds after request:  
Message received 2.72 seconds after request: 56
Message received 2.72 seconds after request: ,
Message received 2.72 seconds after request:  
Message received 2.77 seconds after request: 57
Message received 2.77 seconds after request: ,
Message received 2.77 seconds after request:  
Message received 2.80 seconds after request: 58
Message received 2.80 seconds after request: ,
Message received 2.80 seconds after request:  
Message received 2.85 seconds after request: 59
Message received 2.85 seconds after request: ,
Message received 2.85 seconds after request:  
Message received 2.88 seconds after request: 60
Message received 2.88 seconds after request: ,
Message received 2.88 seconds after request:  
Message received 2.88 seconds after request: 61
Message received 2.88 seconds after request: ,
Message received 2.88 seconds after request:  
Message received 2.89 seconds after request: 62
Message received 2.89 seconds after request: ,
Message received 2.89 seconds after request:  
Message received 2.89 seconds after request: 63
Message received 2.89 seconds after request: ,
Message received 2.89 seconds after request:  
Message received 2.92 seconds after request: 64
Message received 2.92 seconds after request: ,
Message received 2.92 seconds after request:  
Message received 3.37 seconds after request: 65
Message received 3.37 seconds after request: ,
Message received 3.37 seconds after request:  
Message received 3.38 seconds after request: 66
Message received 3.38 seconds after request: ,
Message received 3.38 seconds after request:  
Message received 3.38 seconds after request: 67
Message received 3.38 seconds after request: ,
Message received 3.38 seconds after request:  
Message received 3.38 seconds after request: 68
Message received 3.38 seconds after request: ,
Message received 3.38 seconds after request:  
Message received 3.42 seconds after request: 69
Message received 3.42 seconds after request: ,
Message received 3.42 seconds after request:  
Message received 3.43 seconds after request: 70
Message received 3.43 seconds after request: ,
Message received 3.43 seconds after request:  
Message received 3.46 seconds after request: 71
Message received 3.46 seconds after request: ,
Message received 3.46 seconds after request:  
Message received 3.47 seconds after request: 72
Message received 3.47 seconds after request: ,
Message received 3.47 seconds after request:  
Message received 3.50 seconds after request: 73
Message received 3.50 seconds after request: ,
Message received 3.50 seconds after request:  
Message received 3.51 seconds after request: 74
Message received 3.51 seconds after request: ,
Message received 3.51 seconds after request:  
Message received 3.52 seconds after request: 75
Message received 3.52 seconds after request: ,
Message received 3.52 seconds after request:  
Message received 3.54 seconds after request: 76
Message received 3.54 seconds after request: ,
Message received 3.54 seconds after request:  
Message received 3.56 seconds after request: 77
Message received 3.56 seconds after request: ,
Message received 3.56 seconds after request:  
Message received 3.59 seconds after request: 78
Message received 3.59 seconds after request: ,
Message received 3.59 seconds after request:  
Message received 3.59 seconds after request: 79
Message received 3.59 seconds after request: ,
Message received 3.59 seconds after request:  
Message received 3.59 seconds after request: 80
Message received 3.59 seconds after request: ,
Message received 3.59 seconds after request:  
Message received 3.61 seconds after request: 81
Message received 3.61 seconds after request: ,
Message received 3.61 seconds after request:  
Message received 3.65 seconds after request: 82
Message received 3.65 seconds after request: ,
Message received 3.65 seconds after request:  
Message received 3.85 seconds after request: 83
Message received 3.85 seconds after request: ,
Message received 3.85 seconds after request:  
Message received 3.90 seconds after request: 84
Message received 3.90 seconds after request: ,
Message received 3.90 seconds after request:  
Message received 3.95 seconds after request: 85
Message received 3.95 seconds after request: ,
Message received 3.95 seconds after request:  
Message received 4.00 seconds after request: 86
Message received 4.00 seconds after request: ,
Message received 4.00 seconds after request:  
Message received 4.04 seconds after request: 87
Message received 4.04 seconds after request: ,
Message received 4.04 seconds after request:  
Message received 4.08 seconds after request: 88
Message received 4.08 seconds after request: ,
Message received 4.08 seconds after request:  
Message received 4.12 seconds after request: 89
Message received 4.12 seconds after request: ,
Message received 4.12 seconds after request:  
Message received 4.18 seconds after request: 90
Message received 4.18 seconds after request: ,
Message received 4.18 seconds after request:  
Message received 4.18 seconds after request: 91
Message received 4.18 seconds after request: ,
Message received 4.18 seconds after request:  
Message received 4.18 seconds after request: 92
Message received 4.18 seconds after request: ,
Message received 4.18 seconds after request:  
Message received 4.19 seconds after request: 93
Message received 4.19 seconds after request: ,
Message received 4.19 seconds after request:  
Message received 4.20 seconds after request: 94
Message received 4.20 seconds after request: ,
Message received 4.20 seconds after request:  
Message received 4.23 seconds after request: 95
Message received 4.23 seconds after request: ,
Message received 4.23 seconds after request:  
Message received 4.27 seconds after request: 96
Message received 4.27 seconds after request: ,
Message received 4.27 seconds after request:  
Message received 4.39 seconds after request: 97
Message received 4.39 seconds after request: ,
Message received 4.39 seconds after request:  
Message received 4.39 seconds after request: 98
Message received 4.39 seconds after request: ,
Message received 4.39 seconds after request:  
Message received 4.41 seconds after request: 99
Message received 4.41 seconds after request: ,
Message received 4.41 seconds after request:  
Message received 4.41 seconds after request: 100
Message received 4.41 seconds after request: .
Message received 4.41 seconds after request: None
Full response received 4.41 seconds after request
Full conversation received: 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99, 100.
```
{{< /scrollbox >}}


مقایسه زمان در مثال‌های بالا نشان می‌دهند که هر دو درخواست حدود 4 تا 5 ثانیه طول کشید تا کاملا تکمیل شوند. زمان درخواست ها بسته به بار و سایر عوامل تصادفی متفاوت خواهد بود. با این حال، با درخواست streaming، ما اولین توکن را پس از 0.1 ثانیه دریافت کردیم.