---
title: "مدیریت متن‌های طولانی‌تر از طول کانتکست مدل"
description: "راهنمایی برای مدیریت متن‌های طولانی‌تر از طول کانتکست مدل‌های امبدینگ با استفاده از روش‌های برش و تکه‌بندی."
tags:
- embedding
weight: 
og_image: "/posts/embedding_long_inputs/banner.png"
---

{{< postcover src="/posts/embedding_long_inputs/banner.png" >}}

# مدیریت متن‌های طولانی‌تر از طول کانتکست مدل

مدل‌های `embedding` نمی‌توانند متنی که از طول کانتکست حداکثری مدل بیشتر باشد را پردازش کنند. طول حداکثری بسته به مدل متفاوت است و بر اساس _توکن‌ها_ اندازه‌گیری می‌شود، نه طول رشته. اگر با مفهموم توکن‌ آشنا نیستید، به پست [شمردن تعداد توکن‌ها با `tiktoken`](/how_to_count_tokens_with_tiktoken) مراجعه کنید.

این `notebook` نشان می‌دهد که چگونه متن‌هایی که طولانی‌تر از طول کانتکست حداکثری مدل هستند را مدیریت کنید. ما از مدل `text-embedding-3-small` استفاده خواهیم کرد، اما همین ایده‌ها را می‌توان به مدل‌ها و وظایف دیگر نیز اعمال کرد. برای اطلاعات بیشتر در مورد `embeddings`، به [راهنمای اندپوینت  `Embeddings`](/apis/embeddings) مراجعه کنید.

## 1. طول کانتکست مدل

ابتدا، مدل را انتخاب کرده و یک تابع برای دریافت `embeddings` از `API` تعریف می‌کنیم.

{{< hint info >}}
**برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.**
{{< /hint >}} 

```python
from openai import OpenAI
import os
import openai
from tenacity import retry, wait_random_exponential, stop_after_attempt, retry_if_not_exception_type

client = OpenAI(
    api_key=os.environ.get(("GILAS_API_KEY", "<کلید API خود را اینجا بسازید https://dashboard.gilas.io/apiKey>")), 
    base_url="https://api.gilas.io/v1/" # Gilas APIs
)

EMBEDDING_MODEL = 'text-embedding-3-small'
EMBEDDING_CTX_LENGTH = 8191
EMBEDDING_ENCODING = 'cl100k_base'

# let's make sure to not retry on an invalid request, because that is what we want to demonstrate
@retry(wait=wait_random_exponential(min=1, max=20), stop=stop_after_attempt(6), retry=retry_if_not_exception_type(openai.BadRequestError))
def get_embedding(text_or_tokens, model=EMBEDDING_MODEL):
    return client.embeddings.create(input=text_or_tokens, model=model).data[0].embedding
```

مدل `text-embedding-3-small` دارای طول کانتکست 8191 توکن با کدگذاری `cl100k_base` است و می‌توانیم ببینیم که عبور از این حد باعث خطا می‌شود.
```python
long_text = 'AGI ' * 5000
try:
    get_embedding(long_text)
except openai.BadRequestError as e:
    print(e)
```

{{< ltr >}}
```
Error code: 400 - {'error': {'message': "This model's maximum context length is 8192 tokens, however you requested 10001 tokens (10001 in your prompt; 0 for the completion). Please reduce your prompt; or completion length.", 'type': 'invalid_request_error', 'param': None, 'code': None}}
```
{{</ ltr >}}

واضح است که باید از  بروز اینگونه خطاها جلوگیری کنیم، به خصوص زمانی که  با تعداد زیادی `embedding` سر و کار داریم. با این حال، ممکن است با متن‌هایی مواجه شویم که طولانی‌تر از طول کانتکست حداکثری هستند. در زیر، روش‌های اصلی برای مدیریت این متن‌های طولانی‌تر را توضیح می‌دهیم و دستورالعمل‌هایی برای آنها ارائه می‌دهیم: (1) به سادگی برش دادن متن به طول حداکثری مجاز، و (2) تکه‌بندی متن و `embedding` هر تکه به صورت جداگانه.

## 1. برش دادن متن ورودی

ساده‌ترین راه حل، برش دادن متن ورودی به طول حداکثری مجاز است. از آنجا که طول کانتکست بر اساس توکن‌ها اندازه‌گیری می‌شود، باید قبل از اینکه متن را برش دهیم آن را توکن‌سازی کنیم. `API` ورودی‌ها را هم به صورت متن و هم به صورت توکن قبول می‌کند، بنابراین تا زمانی که مطمئن باشید که از کدگذاری مناسب استفاده می‌کنید، نیازی به تبدیل توکن‌ها به فرم رشته‌ای نیست. در زیر یک مثال از چنین تابع برش‌دهی آورده شده است.
```python
import tiktoken

def truncate_text_tokens(text, encoding_name=EMBEDDING_ENCODING, max_tokens=EMBEDDING_CTX_LENGTH):
    """Truncate a string to have `max_tokens` according to the given encoding."""
    encoding = tiktoken.get_encoding(encoding_name)
    return encoding.encode(text)[:max_tokens]
```

مثال قبلی ما اکنون بدون خطا کار می‌کند.
```python
truncated = truncate_text_tokens(long_text)
len(get_embedding(truncated))
```

## 2. تکه‌بندی متن ورودی

اگرچه برش دادن کار می‌کند، اما حذف متن‌های بالقوه مرتبط یک مشکل واضح است. روش دیگر این است که متن ورودی را به تکه‌ها تقسیم کرده و سپس هر تکه را به صورت جداگانه `embedding` کنیم. سپس می‌توانیم `embedding` تکه‌ها را به صورت جداگانه استفاده کنیم یا آنها را به نوعی ترکیب کنیم، مانند میانگین‌گیری (وزن‌دهی شده بر اساس اندازه هر تکه).

ما از تابع زیر که یک دنباله را به تکه‌ها تقسیم می‌کند، استفاده خواهیم کرد.

```python
from itertools import islice

def batched(iterable, n):
    """Batch data into tuples of length n. The last batch may be shorter."""
    # batched('ABCDEFG', 3) --> ABC DEF G
    if n < 1:
        raise ValueError('n must be at least one')
    it = iter(iterable)
    while (batch := tuple(islice(it, n))):
        yield batch
```

اکنون یک تابع تعریف می‌کنیم که یک رشته را به توکن‌ها کدگذاری کرده و سپس آن را به تکه‌ها تقسیم می‌کند.

```python
def chunked_tokens(text, encoding_name, chunk_length):
    encoding = tiktoken.get_encoding(encoding_name)
    tokens = encoding.encode(text)
    chunks_iterator = batched(tokens, chunk_length)
    yield from chunks_iterator
```

در نهایت، می‌توانیم یک تابع بنویسیم که درخواست‌های `embedding` را به صورت ایمن مدیریت کند، حتی زمانی که متن ورودی طولانی‌تر از طول کانتکست حداکثری باشد. فلگ `average` می‌تواند به `True` تنظیم شود تا میانگین وزنی `embedding` تکه‌ها را برگرداند، یا به `False` تنظیم شود تا فقط لیست `embedding` تکه‌ها را بدون تغییر برگرداند.

```python
import numpy as np

def len_safe_get_embedding(text, model=EMBEDDING_MODEL, max_tokens=EMBEDDING_CTX_LENGTH, encoding_name=EMBEDDING_ENCODING, average=True):
    chunk_embeddings = []
    chunk_lens = []
    for chunk in chunked_tokens(text, encoding_name=encoding_name, chunk_length=max_tokens):
        chunk_embeddings.append(get_embedding(chunk, model=model))
        chunk_lens.append(len(chunk))

    if average:
        chunk_embeddings = np.average(chunk_embeddings, axis=0, weights=chunk_lens)
        chunk_embeddings = chunk_embeddings / np.linalg.norm(chunk_embeddings)  # normalizes length to 1
        chunk_embeddings = chunk_embeddings.tolist()
    return chunk_embeddings
```

بار دیگر، می‌توانیم متن‌های طولانی را مدیریت کنیم.

```python
average_embedding_vector = len_safe_get_embedding(long_text, average=True)
chunks_embedding_vectors = len_safe_get_embedding(long_text, average=False)

print(f"Setting average=True gives us a single {len(average_embedding_vector)}-dimensional embedding vector for our long text.")
print(f"Setting average=False gives us {len(chunks_embedding_vectors)} embedding vectors, one for each of the chunks.")
```

{{< ltr >}}

```
Setting average=True gives us a single 1536-dimensional embedding vector for our long text.
Setting average=False gives us 2 embedding vectors, one for each of the chunks.
```

{{</ ltr >}}

در برخی موارد، ممکن است منطقی باشد که تکه‌ها را بر اساس مرزهای پاراگراف یا جمله تقسیم کنید تا به حفظ معنای متن کمک کنید.