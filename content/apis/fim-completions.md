---
title: 'v1/fim/completions/'
weight: 1001
---

اندپوینت تکمیل کد یا fill-in-the-middle برای تکمیل خطوط کد باز یا ناقص با استفاده از مدل Codestral طراحی شده است و به شما امکان می‌دهد به سرعت و با دقت بیشتری کد خود را تکمیل کنید. ازین مدل برای code autocorrection و code autocompletion توسط افزونه‌های IDE و ویرایشگرهای متن استفاده می‌شود. مدل Codestral یک مدل پیشرفته تولید کد است که به طور خاص برای وظایف تولید کد مانند تکمیل و تولید کد در بین خطوط بهینه‌سازی شده است.
همچنین این مدل قابلیت تکمیل متن به زبان انگلیسی٬ فارسی و غیره را نیز دارد که در نوشتن کامنت یا متون دیگر می‌تواند مورد استفاده قرار بگیرد.

## کاربردهای اندپوینت تکمیل کد

اگر کاربری هستید که قصد دارید Codestral را به عنوان بخشی از یک افزونه‌ی IDE مورد استفاده قرار دهید، توصیه می‌شود از اندپوینت `https://api.gilas.io/v1/fim/completions` استفاده کنید.
اگر در حال توسعه یک افزونه یا برنامه‌ای هستید که این اندپوینت‌ها را مستقیماً در اختیار کاربر قرار می‌دهد و انتظار دارید که کاربران کلیدهای `API` خود را وارد کنند، همچنان استفاده از این اندپوینت توصیه می‌شود. 

## ساخت حساب کاربری

ابتدا، یک  [حساب کاربری جدید](https://dashboard.gilas.io) ایجاد کنید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید. مطمئن شوید که این کلید را در جای امنی ذخیره کرده و آن را با کسی به اشتراک نگذارید.

## ارسال درخواست

هنگام استفاده از اندپوینت تکمیل کد٬ کاربران می‌توانند نقطه شروع کد را با استفاده از یک `prompt` و نقطه پایانی کد را با استفاده از یک `suffix` اختیاری و یک `stop` اختیاری تعریف کنند. سپس مدل `Codestral` کدی که باید بین این دو نقطه قرار گیرد را تولید می‌کند. این قابلیت مدل را برای وظایفی که به تولید یک بخش خاص از کد نیاز دارند ایده‌آل می‌کند. در ادامه، سه مثال آورده شده است:


### مثال ۱: تکمیل کد

{{< tabs "example 1 - code" >}}
{{< tab "curl" >}}

```shell
curl --location 'https://api.gilas.io/v1/fim/completions' \
--header 'Content-Type: application/json' \
--header 'Accept: application/json' \
--header "Authorization: Bearer $GILAS_API_KEY" \
--data '{
    "model": "codestral-latest",
    "prompt": "def f(",
    "suffix": "return a + b",
    "max_tokens": 64,
    "temperature": 0
}'
```

{{< /tab >}}

{{< tab "Python" >}}

```python
import os
from mistralai import Mistral

api_key = os.environ["GILAS_API_KEY"]
api_url = "https://api.gilas.io/v1/fim/completions"
client = Mistral(api_key=api_key, api_url=api_url)

model = "codestral-latest"
prompt = "def fibonacci(n: int):"
suffix = "n = int(input('Enter a number: '))\nprint(fibonacci(n))"

response = client.fim.complete(
    model=model,
    prompt=prompt,
    suffix=suffix,
    temperature=0,
    top_p=1,
)

print(
    f"""
{prompt}
{response.choices[0].message.content}
{suffix}
"""
)
```
{{< /tab >}}
{{< /tabs >}}


### مثال ۲: تولید کد
{{< tabs "example 2 - code" >}}
{{< tab "curl" >}}

```shell
curl --location 'https://api.gilas.io/v1/fim/completions' \
--header 'Content-Type: application/json' \
--header 'Accept: application/json' \
--header "Authorization: Bearer $GILAS_API_KEY" \
--data '{
    "model": "codestral-latest",
    "prompt": "def is_odd(n): \n return n % 2 == 1 \n def test_is_odd():", 
    "suffix": "",
    "max_tokens": 64,
    "temperature": 0
}'
```

{{< /tab >}}

{{< tab "Python" >}}

```python
import os
from mistralai import Mistral

api_key = os.environ["GILAS_API_KEY"]
api_url = "https://api.gilas.io/v1/fim/completions"
client = Mistral(api_key=api_key, api_url=api_url)

model = "codestral-latest"
prompt = "def is_odd(n): \n return n % 2 == 1 \ndef test_is_odd():"

response = client.fim.complete(model=model, prompt=prompt, temperature=0, top_p=1)

print(
    f"""
{prompt}
{response.choices[0].message.content}
"""
)
```
{{< /tab >}}
{{< /tabs >}}


### مثال ۳: استفاده از توکن پایانی (stop token)
{{< tabs "example 3 - code" >}}
{{< tab "curl" >}}

```shell
curl --location 'https://api.gilas.io/v1/fim/completions' \
--header 'Content-Type: application/json' \
--header 'Accept: application/json' \
--header "Authorization: Bearer $GILAS_API_KEY" \
--data '{
    "model": "codestral-latest",
    "prompt": "def is_odd(n): \n return n % 2 == 1 \n def test_is_odd():", 
    "suffix": "test_is_odd()",
    "stop": ["\n\n"],
    "max_tokens": 64,
    "temperature": 0
}'
```

{{< /tab >}}

{{< tab "Python" >}}

```python
import os
from mistralai import Mistral

api_key = os.environ["GILAS_API_KEY"]
api_url = "https://api.gilas.io/v1/fim/completions"
client = Mistral(api_key=api_key, api_url=api_url)

model = "codestral-latest"
prompt = "def is_odd(n): \n return n % 2 == 1 \ndef test_is_odd():"
suffix = "n = int(input('Enter a number: '))\nprint(fibonacci(n))"

response = client.fim.complete(
    model=model, prompt=prompt, suffix=suffix, temperature=0, top_p=1, stop=["\n\n"]
)

print(
    f"""
{prompt}
{response.choices[0].message.content}
"""
)
```
{{< /tab >}}
{{< /tabs >}}

برای اطلاع از تمام پارامترهای قابل استفاده هنگام فراخوانی اندپوینت تکمیل کد، می‌توانید به [مستندات Codestral API](https://docs.mistral.ai/api/#tag/fim) مراجعه کنید.

پست‌های زیر نحوه تنظیم برخی از افزونه‌های محبوب تکمیل کد (code autocompletion)  برای استفاده از مدل Codestral از طریق Gilas API را بررسی می‌کنند:
