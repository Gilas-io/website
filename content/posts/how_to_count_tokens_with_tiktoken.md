---
title: "شمردن تعداد توکن‌ها با tiktoken"
description: "پکیج tiktoken یک توکن ساز سریع و open source است که توسط OpenAI توسعه پیدا کرده است. با دادن یک رشته متن (مثلاً، 'tiktoken is great!') و یک encoding (مثلاً، 'cl100k_base')، یک توکن ساز می تواند رشته متن را به یک لیست از توکن ها تقسیم کند."
tags:
- openai
- tiktoken
- gilas.io
- blog
weight: 2005
og_image: "/posts/how_to_count_tokens_with_tiktoken/banner.png" 
---

{{< postcover src="/posts/how_to_count_tokens_with_tiktoken/banner.png" >}}

پکیج tiktoken یک توکن ساز سریع و open source است که توسط OpenAI توسعه پیدا کرده است. با دادن یک رشته متن (مثلاً، "tiktoken is great!") و یک encoding (مثلاً، "cl100k_base")، یک توکن ساز می تواند رشته متن را به یک لیست از توکن ها تقسیم کند (مثلاً، ["t", "ik", "token", " is", " great", "!"]). 

تقسیم کردن رشته های متن به توکن ها مفید است زیرا مدل های GPT متن را به صورت توکن می بینند. دانستن تعداد توکن های موجود در یک رشته متن می تواند به شما بگوید (الف) آیا رشته برای پردازش توسط مدل بیش از حد مجاز طولانی است و (ب) فراخوانی API مدل چقدر هزینه دارد (زیرا استفاده از آن بر اساس توکن قیمت گذاری می شود).


{{< hint warning >}} 
توجه: ورودی‌های داده شده و خروجی‌های تولید شده توسط مدل در این مثال به زبان انگلیسی هستند. برای تولید خروجی به زبان فارسی٬ کافی‌ست از مدل بخواهید که خروجی را به زبان فارسی تولید کند.
{{< /hint >}} 


 ## Encodings
 
  یک encoding مشخص می کند که چگونه متن به توکن تبدیل می شود. مدل های مختلف از encoding های مختلفی استفاده می کنند. tiktoken سه encoding را که توسط مدل های OpenAI استفاده می شوند پشتیبانی می کند:

| Encoding name | OpenAI models |
|---|---|
| `cl100k_base` | `gpt-4-turbo`, `gpt-3.5-turbo`, `text-embedding-ada-002`, `text-embedding-3-small`, `text-embedding-3-large` |
| `p50k_base` |  Codex models, `text-davinci-002`, `text-davinci-003` |
| `r50k_base` | GPT-3 models like `davinci` |


شما می توانید encoding را برای یک مدل با استفاده از `tiktoken.encoding_for_model()` به شکل زیر بازیابی کنید:

```python
encoding = tiktoken.encoding_for_model('gpt-3.5-turbo')
```

توجه داشته باشید که p50k_base به طور قابل توجهی با r50k_base همپوشانی دارد و برای برنامه های غیر کد، معمولاً همان توکن ها را ارائه می دهند.

## کتابخانه‌های tiktoken برای زبان‌های برنامه‌نویسی مختلف
 
  برای encoding های cl100k_base و p50k_base:

- Python: [tiktoken](https://github.com/openai/tiktoken/blob/main/README.md)
- NET / C#: [SharpToken](https://github.com/dmitry-brazhenko/SharpToken), [TiktokenSharp](https://github.com/aiqinxuancai/TiktokenSharp)
- Java: [jtokkit](https://github.com/knuddelsgmbh/jtokkit)
- Go: [tiktoken-go](https://github.com/pkoukk/tiktoken-go)
- Rust: [tiktoken-rs](https://github.com/zurawiki/tiktoken-rs)


## متون چطوری به توکن‌ها تبدیل می‌شوند؟

در انگلیسی، طول توکن ها معمولاً از یک کاراکتر تا یک کلمه متغیرند (مثلاً، "t" یا " great")، اگرچه در برخی زبان ها توکن ها می توانند کوتاه تر از یک کاراکتر یا بلندتر از یک کلمه باشند.

 فضاهای خالی معمولاً با شروع کلمات دسته بندی می شوند (مثلاً، " is" به جای "is " یا " "+"is"). 
 
برای مشاهده ساخت رشته ای از توکن‌ها از روی یک متن می‌توانید از [Tiktokenizer](https://tiktokenizer.vercel.app/) استفاده کنید.

## استفاده از پکیج tiktoken در پایتون

 در صورت نیاز، tiktoken را با pip نصب کنید

```python
%pip install --upgrade tiktoken
%pip install --upgrade openai
```

 برای بارگذاری یک encoding بر اساس نام، از `tiktoken.get_encoding()`  استفاده کنید. اولین باری که این کد را اجرا می کنید، نیاز به اتصال به اینترنت برای دانلود آن دارید. اجراهای بعدی نیازی به اتصال به اینترنت ندارند.

 ```python
import tiktoken

encoding = tiktoken.get_encoding("cl100k_base")
```

برای بارگذاری خودکار encoding صحیح
از `tiktoken.encoding_for_model()` استفاده کنید.

```python
encoding = tiktoken.encoding_for_model("gpt-3.5-turbo")
```

حالا می‌توانید یک متن را به توکن تبدیل کنید.

```python
encoding.encode("tiktoken is great!")

-> [83, 1609, 5963, 374, 2294, 0]
```

تعداد توکن ها را با شمارش طول لیست بازگردانده شده توسط `.encode()` بشمارید.

```python
def num_tokens_from_string(string: str, encoding_name: str) -> int:
    """Returns the number of tokens in a text string."""
    encoding = tiktoken.get_encoding(encoding_name)
    num_tokens = len(encoding.encode(string))
    return num_tokens


num_tokens_from_string("tiktoken is great!", "cl100k_base")

-> 6
```


برای تبدیل توکن ها به متن از `encoding.decode()` استفاده کنید.

```python
encoding.decode([83, 1609, 5963, 374, 2294, 0])

-> 'tiktoken is great!'
```

تابع `.decode_single_token_bytes()` به طور ایمن یک توکن عدد صحیح تکی را به بایت هایی که نماینده آن است تبدیل می کند.

```python
[encoding.decode_single_token_bytes(token) for token in [83, 1609, 5963, 374, 2294, 0]]

-> [b't', b'ik', b'token', b' is', b' great', b'!']
```

## شمارش توکن‌ها برای فراخوانی Chat Completions API

 در زیر یک تابع نمونه برای شمارش توکن‌ها برای پیام‌های ارسالی به gpt-3.5-turbo یا gpt-4-turbo وجود دارد. توجه داشته باشید که روش دقیق شمارش توکن‌ها از پیام‌ها ممکن است از مدل به مدل تغییر کند. شمارش‌های حاصل از تابع زیر را یک تخمین در نظر بگیرید، نه یک ضمانت. به خصوص، درخواست‌هایی که از Function call استفاده می‌کنند، توکن‌های اضافی را بر روی تخمین‌های محاسبه شده در زیر مصرف خواهند کرد.

 ```python
def num_tokens_from_messages(messages, model="gpt-3.5-turbo"):
    """Return the number of tokens used by a list of messages."""
    try:
        encoding = tiktoken.encoding_for_model(model)
    except KeyError:
        print("Warning: model not found. Using cl100k_base encoding.")
        encoding = tiktoken.get_encoding("cl100k_base")
    if model in {
        "gpt-3.5-turbo-0613",
        "gpt-3.5-turbo-16k-0613",
        "gpt-4-0314",
        "gpt-4-32k-0314",
        "gpt-4-0613",
        "gpt-4-32k-0613",
        }:
        tokens_per_message = 3
        tokens_per_name = 1
    elif model == "gpt-3.5-turbo-0301":
        tokens_per_message = 4  # every message follows <|start|>{role/name}\n{content}<|end|>\n
        tokens_per_name = -1  # if there's a name, the role is omitted
    elif "gpt-3.5-turbo" in model:
        print("Warning: gpt-3.5-turbo may update over time. Returning num tokens assuming gpt-3.5-turbo-0613.")
        return num_tokens_from_messages(messages, model="gpt-3.5-turbo-0613")
    elif "gpt-4" in model:
        print("Warning: gpt-4 may update over time. Returning num tokens assuming gpt-4-0613.")
        return num_tokens_from_messages(messages, model="gpt-4-0613")
    else:
        raise NotImplementedError(
            f"""num_tokens_from_messages() is not implemented for model {model}. See https://github.com/openai/openai-python/blob/main/chatml.md for information on how messages are converted to tokens."""
        )
    num_tokens = 0
    for message in messages:
        num_tokens += tokens_per_message
        for key, value in message.items():
            num_tokens += len(encoding.encode(value))
            if key == "name":
                num_tokens += tokens_per_name
    num_tokens += 3  # every reply is primed with <|start|>assistant<|message|>
    return num_tokens

 ```

 {{< hint info >}}
**برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.**
{{< /hint >}} 

```python
# let's verify the function above matches the Gilas API response

from openai import OpenAI
import os

client = OpenAI(
    api_key=os.environ.get(("GILAS_API_KEY", "<کلید API خود را اینجا بسازید https://dashboard.gilas.io/apiKey>")), 
    base_url="https://api.gilas.io/v1/" # Gilas APIs
)

example_messages = [
    {
        "role": "system",
        "content": "You are a helpful, pattern-following assistant that translates corporate jargon into plain English.",
    },
    {
        "role": "system",
        "name": "example_user",
        "content": "New synergies will help drive top-line growth.",
    },
    {
        "role": "system",
        "name": "example_assistant",
        "content": "Things working well together will increase revenue.",
    },
    {
        "role": "system",
        "name": "example_user",
        "content": "Let's circle back when we have more bandwidth to touch base on opportunities for increased leverage.",
    },
    {
        "role": "system",
        "name": "example_assistant",
        "content": "Let's talk later when we're less busy about how to do better.",
    },
    {
        "role": "user",
        "content": "This late pivot means we don't have time to boil the ocean for the client deliverable.",
    },
]

for model in [
    "gpt-3.5-turbo-0301",
    "gpt-3.5-turbo-0613",
    "gpt-3.5-turbo",
    "gpt-4-0314",
    "gpt-4-0613",
    "gpt-4",
    ]:
    print(model)
    # example token count from the function defined above
    print(f"{num_tokens_from_messages(example_messages, model)} prompt tokens counted by num_tokens_from_messages().")
    # example token count from the Gilas API
    response = client.chat.completions.create(model=model,
    messages=example_messages,
    temperature=0,
    max_tokens=1)
    print(f'{response.usage.prompt_tokens} prompt tokens counted by the Gilas API.')
    print()
```

خروجی:

{{< scrollbox >}}
```
gpt-3.5-turbo-0301
127 prompt tokens counted by num_tokens_from_messages().
127 prompt tokens counted by the Gilas API.

gpt-3.5-turbo-0613
129 prompt tokens counted by num_tokens_from_messages().
129 prompt tokens counted by the Gilas API.

gpt-3.5-turbo
Warning: gpt-3.5-turbo may update over time. Returning num tokens assuming gpt-3.5-turbo-0613.
129 prompt tokens counted by num_tokens_from_messages().
129 prompt tokens counted by the Gilas API.

gpt-4-0314
129 prompt tokens counted by num_tokens_from_messages().
129 prompt tokens counted by the Gilas API.

gpt-4-0613
129 prompt tokens counted by num_tokens_from_messages().
129 prompt tokens counted by the Gilas API.

gpt-4
Warning: gpt-4 may update over time. Returning num tokens assuming gpt-4-0613.
129 prompt tokens counted by num_tokens_from_messages().
129 prompt tokens counted by the Gilas API.
```
{{< /scrollbox >}}