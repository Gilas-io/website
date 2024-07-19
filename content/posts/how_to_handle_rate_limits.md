---
title: "مدیریت محدودیت‌های Rate limit"
description: "راهنمایی برای جلوگیری و مدیریت خطاهای محدودیت Rate limit در استفاده از API"
tags:
- rate_limit
weight: 
og_image: "/posts/how_to_handle_rate_limits/banner.jpg"
---

{{< postcover src="/posts/how_to_handle_rate_limits/banner.jpg" >}}

# چگونه با محدودیت‌های Rate limit برخورد کنیم

هنگامی که به طور مکرر `Gilas API` را فراخوانی می‌کنید، ممکن است با پیام‌های خطایی مانند `429: 'Too Many Requests'` یا `RateLimitError` مواجه شوید. این پیام‌های خطا به دلیل تجاوز از محدودیت‌های Rate limit رخ می‌دهند.

محدودیت‌های Rate limit به منظور حفظ کیفیت خدمات ما برای همه کاربران است. Rate limit بر روی تعداد دفعاتی که یک کاربر می‌تواند در یک دوره زمانی مشخص به خدمات ما دسترسی پیدا کند، محدودیت اعمال می‌کند.

این راهنما نکاتی را برای جلوگیری و مدیریت خطاهای محدودیت Rate limit را ارایه می‌دهد.

## چرا محدودیت‌های Rate limit وجود دارند

محدودیت‌های Rate limit یک روش معمول برای `API`ها هستند و به دلایل مختلفی اعمال می‌شوند.

- اول، آنها به محافظت در برابر سوءاستفاده یا استفاده نادرست از `API` کمک می‌کنند. به عنوان مثال، یک عامل مخرب می‌تواند `API` را با درخواست‌های زیاد اشباع کند تا  باعث اختلال در خدمات شود. با تنظیم محدودیت‌های Rate limit، می‌توان از این نوع فعالیت‌ها جلوگیری کرد.
- دوم، محدودیت‌های Rate limit به اطمینان از دسترسی عادلانه همه به `API` کمک می‌کنند. اگر یک فرد یا سازمان تعداد زیادی درخواست ارسال کند، می‌تواند `API` را برای دیگران کند کند. با محدود کردن تعداد درخواست‌هایی که یک کاربر می‌تواند ارسال کند، `Gilas.io` اطمینان می‌دهد که همه فرصت استفاده از `API` را بدون تجربه کندی دارند.
- در نهایت، محدودیت‌های Rate limit می‌توانند به `Gilas.io` در مدیریت بار کلی زیرساخت‌هایش کمک کنند. اگر درخواست‌ها به `API` به طور چشمگیری افزایش یابد، می‌تواند سرورها را تحت فشار قرار دهد و باعث مشکلات عملکردی شود. با تنظیم محدودیت‌های Rate limit، `Gilas.io` می‌تواند تجربه‌ای روان و مداوم برای همه کاربران حفظ کند.

اگرچه برخورد به محدودیت‌های Rate limit می‌تواند اعصاب خردکن باشد، اما این محدودیت‌ها برای محافظت از عملکرد قابل اعتماد `API` برای کاربرانش وجود دارند.

## محدودیت‌های Rate limit پیش‌فرض

محدودیت Rate limit  شما به طور خودکار بر اساس چندین عامل تنظیم می‌شود. با افزایش استفاده شما از `Gilas API` و پرداخت موفقیت‌آمیز صورت‌حساب، ما به طور خودکار سطح استفاده شما را افزایش می‌دهیم.



اطلاعات بیشتر در مورد محدودیت‌های Rate limit را اینجا بخوانید: [محدودیت‌های Rate limit](/ratelimit)


## مثال خطای محدودیت Rate limit

یک خطای محدودیت Rate limit زمانی رخ می‌دهد که درخواست‌های `API` خیلی سریع ارسال شوند. اگر از کتابخانه پایتون `OpenAI` استفاده می‌کنید، آنها به شکل زیر خواهند بود:

```
RateLimitError: Rate limit reached for default-codex in organization org-{id} on requests per min. Limit: 20.000000 / min. Current: 24.000000 / min. Contact support@openai.com if you continue to have issues or if you’d like to request an increase.
```

در زیر کد نمونه‌ای برای ایجاد یک خطای محدودیت Rate limit آورده شده است.

{{< hint info >}}
**برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.**
{{< /hint >}} 

```python

from openai import OpenAI # for calling the OpenAI API
import os

client = OpenAI(
    api_key=os.environ.get(("GILAS_API_KEY", "<کلید API خود را اینجا بسازید https://dashboard.gilas.io/apiKey>")), 
    base_url="https://api.gilas.io/v1/" # Gilas APIs
)

for _ in range(1000):
    client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": "Hello"}],
        max_tokens=10,
    )
```

## چگونه از خطاهای محدودیت Rate limit جلوگیری کنیم

### تلاش مجدد

یکی از راه‌های آسان برای جلوگیری از خطاهای محدودیت Rate limit، تلاش مجدد خودکار درخواست‌ها با بازگشت نمایی تصادفی است. تلاش مجدد با بازگشت نمایی به معنای انجام یک وقفه کوتاه زمانی که یک خطای محدودیت Rate limit رخ می‌دهد، سپس تلاش مجدد برای درخواست ناموفق است. اگر درخواست همچنان ناموفق باشد، طول وقفه افزایش می‌یابد و فرآیند تکرار می‌شود. این کار تا زمانی که درخواست موفق شود یا حداکثر تعداد تلاش‌ها برسد ادامه می‌یابد.

این روش مزایای زیادی دارد:

- تلاش‌های مجدد خودکار به این معناست که می‌توانید از خطاهای محدودیت Rate limit بدون خرابی یا از دست دادن داده‌ها بازیابی کنید.
- بازگشت نمایی به این معناست که تلاش‌های مجدد اولیه شما می‌توانند سریع انجام شوند، در حالی که همچنان از تأخیرهای طولانی‌تر در صورت شکست تلاش‌های اولیه بهره‌مند می‌شوید.
- افزودن جیتتر تصادفی به تأخیر کمک می‌کند تا تلاش‌های مجدد همه در یک زمان انجام نشوند.

توجه داشته باشید که درخواست‌های ناموفق به محدودیت در دقیقه شما کمک می‌کنند، بنابراین ارسال مداوم یک درخواست کار نخواهد کرد.

در زیر چند راه‌حل نمونه آورده شده است.

#### مثال #1: استفاده از کتابخانه Tenacity

[Tenacity](https://tenacity.readthedocs.io/en/latest/) یک کتابخانه عمومی برای تلاش مجدد است که تحت مجوز Apache 2.0 نوشته شده و به زبان پایتون است تا وظیفه افزودن رفتار تلاش مجدد به تقریباً هر چیزی را ساده کند.

برای افزودن بازگشت نمایی به درخواست‌های خود، می‌توانید از `tenacity.retry` [دکوریتور](https://peps.python.org/pep-0318/) استفاده کنید. مثال زیر از تابع `tenacity.wait_random_exponential` برای افزودن بازگشت نمایی تصادفی به یک درخواست استفاده می‌کند.

توجه داشته باشید که کتابخانه Tenacity یک ابزار شخص ثالث است و ما هیچ تضمینی در مورد قابلیت اطمینان یا امنیت آن نمی‌دهد.
```python
from tenacity import (
    retry,
    stop_after_attempt,
    wait_random_exponential,
)  # برای بازگشت نمایی

@retry(wait=wait_random_exponential(min=1, max=60), stop=stop_after_attempt(6))
def completion_with_backoff(**kwargs):
    return client.chat.completions.create(**kwargs)


completion_with_backoff(model="gpt-4o-mini", messages=[{"role": "user", "content": "Once upon a time,"}])
```

#### مثال #2: استفاده از کتابخانه backoff

کتابخانه دیگری که دکوریتورهای تابع برای بازگشت و تلاش مجدد فراهم می‌کند، [backoff](https://pypi.org/project/backoff/) است.

مانند Tenacity، کتابخانه backoff یک ابزار ثالث است و ما هیچ تضمینی در مورد قابلیت اطمینان یا امنیت آن نمی‌دهیم.
```python
import backoff  # برای بازگشت نمایی

@backoff.on_exception(backoff.expo, openai.RateLimitError)
def completions_with_backoff(**kwargs):
    return client.chat.completions.create(**kwargs)


completions_with_backoff(model="gpt-4o-mini", messages=[{"role": "user", "content": "Once upon a time,"}])
```

#### مثال 3: پیاده‌سازی دستی بازگشت

اگر نمی‌خواهید از کتابخانه‌های ثالث استفاده کنید، می‌توانید منطق بازگشت خود را پیاده‌سازی کنید.

```python
import random
import time

# تعریف یک دکوریتور تلاش مجدد
def retry_with_exponential_backoff(
    func,
    initial_delay: float = 1,
    exponential_base: float = 2,
    jitter: bool = True,
    max_retries: int = 10,
    errors: tuple = (openai.RateLimitError,),
):
    """تلاش مجدد یک تابع با بازگشت نمایی."""

    def wrapper(*args, **kwargs):
        num_retries = 0
        delay = initial_delay

        while True:
            try:
                return func(*args, **kwargs)
            # تلاش مجدد در خطاهای مشخص شده
            except errors as e:
                # افزایش تعداد تلاش‌ها
                num_retries += 1
                # بررسی اینکه آیا حداکثر تلاش‌ها رسیده است
                if num_retries > max_retries:
                    raise Exception(
                        f"حداکثر تعداد تلاش‌ها ({max_retries}) تجاوز شده است."
                    )
                # افزایش تأخیر
                delay *= exponential_base * (1 + jitter * random.random())
                # ایجاد وقفه
                time.sleep(delay)
            except Exception as e:
                raise e
    return wrapper

@retry_with_exponential_backoff
def completions_with_backoff(**kwargs):
    return client.chat.completions.create(**kwargs)

completions_with_backoff(model="gpt-4o-mini", messages=[{"role": "user", "content": "Once upon a time,"}])
```