---
title: "تولید خروجی‌های یکسان با پارامتر seed"
description: "توسعه‌دهندگان اکنون می‌توانند پارامتر seed را در درخواست‌های Chat Completion استفاده کنند تا خروجی‌های نسبتاً ثابتی دریافت کنند. لطفاً توجه داشته باشید که این ویژگی در مرحله بتا است و در حال حاضر فقط برای مدلهای gpt-4-1106-preview و gpt-3.5-turbo-1106 و مدل‌های بعد از آنها پشتیبانی می‌شود.
"
weight: 2009
---

توسعه‌دهندگان اکنون می‌توانند پارامتر `seed` را در درخواست‌های `Chat Completion` استفاده کنند تا خروجی‌های نسبتاً ثابتی دریافت کنند. لطفاً توجه داشته باشید که این ویژگی در مرحله بتا است و در حال حاضر فقط برای مدلهای `gpt-4-1106-preview` و `gpt-3.5-turbo-1106` و مدل‌های بعد از آنها پشتیبانی می‌شود.


خروجی API‌های `Chat Completions` و `Completions` به طور پیش‌فرض غیر قطعی هستند (که به این معناست که خروجی‌های مدل ممکن است از درخواست به درخواست متفاوت باشند)، اما اکنون با استفاده از چند پارامتر در سطح مدل، می‌توانید خروجی‌ها را تا حدی کنترل کنید. این امکان برای بازتولید نتایج و تست‌ها بسیار مفید است.

### پیاده‌سازی خروجی‌های ثابت

برای دریافت خروجی‌های نسبتاً قطعی در درخواست‌های API:

1. پارامتر `seed` را به هر عدد صحیح دلخواه خود تنظیم کنید، اما از همان مقدار در تمام درخواست‌ها استفاده کنید. به عنوان مثال، 12345.
2. سایر پارامترها (مثل `prompt`، `temperature`، `top_p`، و غیره) را در تمام درخواست‌ها به مقادیر ثابتی تنظیم کنید.
3. در پاسخ، فیلد `system_fingerprint` را بررسی کنید. اثر انگشت سیستم یک شناسه برای ترکیب فعلی وزن‌های مدل، زیرساخت و سایر گزینه‌های پیکربندی مورد استفاده سرورهای OpenAI برای تولید پاسخ است. این مقدار هر زمان که پارامترهای درخواست تغییر کند یا OpenAI پیکربندی زیرساخت‌هایش را به‌روزرسانی کند (که ممکن است چند بار در سال اتفاق بیفتد) تغییر می‌کند.
4. اگر `seed`، پارامترهای دیگر درخواست API، و `system_fingerprint` در تمام درخواست‌ها مطابقت داشته باشند، خروجی‌های مدل عمدتاً یکسان خواهند بود. با این حال، هنوز احتمال کمی وجود دارد که پاسخ‌ها حتی وقتی پارامترهای درخواست و `system_fingerprint` مطابقت دارند، متفاوت باشند که این به دلیل عدم قطعیت ذاتی مدل‌هاست.


### مثال: تولید خروجی با یک `seed` ثابت

در این مثال، ما نحوه تولید یک قطعه کوتاه با استفاده از یک `seed` ثابت را نشان می‌دهیم. این می‌تواند به ویژه در سناریوهایی که نیاز به تولید نتایج ثابت برای تست، اشکال‌یابی یا برای برنامه‌هایی که نیاز به خروجی‌های ثابت دارند، مفید باشد.

#### Python SDK

{{< hint info >}}
**برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.**
{{< /hint >}} 


```python
!pip install --upgrade openai 

import openai
import asyncio
from IPython.display import display, HTML

from utils.embeddings_utils import (
    get_embedding,
    distances_from_embeddings
)

GPT_MODEL = "gpt-3.5-turbo-1106"

async def get_chat_response(
    system_message: str, user_request: str, seed: int = None, temperature: float = 0.7
):
    try:
        messages = [
            {"role": "system", "content": system_message},
            {"role": "user", "content": user_request},
        ]

        response = openai.chat.completions.create(
            model=GPT_MODEL,
            messages=messages,
            seed=seed,
            max_tokens=200,
            temperature=temperature,
        )

        response_content = response.choices[0].message.content
        system_fingerprint = response.system_fingerprint
        prompt_tokens = response.usage.prompt_tokens
        completion_tokens = response.usage.total_tokens - response.usage.prompt_tokens

        table = f"""
        <table>
        <tr><th>Response</th><td>{response_content}</td></tr>
        <tr><th>System Fingerprint</th><td>{system_fingerprint}</td></tr>
        <tr><th>Number of prompt tokens</th><td>{prompt_tokens}</td></tr>
        <tr><th>Number of completion tokens</th><td>{completion_tokens}</td></tr>
        </table>
        """
        display(HTML(table))

        return response_content
    except Exception as e:
        print(f"An error occurred: {e}")
        return None

def calculate_average_distance(responses):
    """
    این تابع میانگین فاصله بین جاسازی‌های پاسخ‌ها را محاسبه می‌کند.
    فاصله بین جاسازی‌ها معیاری برای شباهت پاسخ‌ها است.
    """
    # محاسبه جاسازی‌ها برای هر پاسخ
    response_embeddings = [get_embedding(response) for response in responses]

    # محاسبه فاصله‌ها بین اولین پاسخ و بقیه
    distances = distances_from_embeddings(response_embeddings[0], response_embeddings[1:])

    # محاسبه میانگین فاصله
    average_distance = sum(distances) / len(distances)

    # بازگشت میانگین فاصله
    return average_distance
```

ابتدا، بیایید چند نسخه مختلف از یک قطعه کوتاه در مورد "یک سفر به مریخ" بدون پارامتر `seed` تولید کنیم. این رفتار پیش‌فرض است:

```python
topic = "a journey to Mars"
system_message = "You are a helpful assistant."
user_request = f"Generate a short excerpt of news about {topic}."

responses = []

async def get_response(i):
    print(f'Output {i + 1}\n{"-" * 10}')
    response = await get_chat_response(
        system_message=system_message, user_request=user_request
    )
    return response

responses = await asyncio.gather(*[get_response(i) for i in range(5)])
average_distance = calculate_average_distance(responses)
print(f"The average similarity between responses is: {average_distance}")
```

خروجی:

{{< scrollbox >}}

```bash
Output 1
----------
Response	"NASA's Mars mission reaches critical stage as spacecraft successfully enters orbit around the red planet. The historic journey, which began over a year ago, has captured the world's attention as scientists and astronauts prepare to land on Mars for the first time. The mission is expected to provide valuable insights into the planet's geology, atmosphere, and potential for sustaining human life in the future."
System Fingerprint	fp_772e8125bb
Number of prompt tokens	29
Number of completion tokens	76
Output 2
----------
Response	"NASA's Perseverance rover successfully landed on Mars, marking a major milestone in the mission to explore the red planet. The rover is equipped with advanced scientific instruments to search for signs of ancient microbial life and collect samples of rock and soil for future return to Earth. This historic achievement paves the way for further exploration and potential human missions to Mars in the near future."
System Fingerprint	fp_772e8125bb
Number of prompt tokens	29
Number of completion tokens	76
Output 3
----------
Response	"SpaceX successfully launched the first manned mission to Mars yesterday, marking a historic milestone in space exploration. The crew of four astronauts will spend the next six months traveling to the red planet, where they will conduct groundbreaking research and experiments. This mission represents a significant step towards establishing a human presence on Mars and paves the way for future interplanetary travel."
System Fingerprint	fp_772e8125bb
Number of prompt tokens	29
Number of completion tokens	72
Output 4
----------
Response	"NASA's latest Mars mission exceeds expectations as the Perseverance rover uncovers tantalizing clues about the Red Planet's past. Scientists are thrilled by the discovery of ancient riverbeds and sedimentary rocks, raising hopes of finding signs of past life on Mars. With this exciting progress, the dream of sending humans to Mars feels closer than ever before."
System Finger

print	fp_772e8125bb
Number of prompt tokens	29
Number of completion tokens	72
Output 5
----------
Response	"NASA's Perseverance Rover Successfully Lands on Mars, Begins Exploration Mission In a historic moment for space exploration, NASA's Perseverance rover has successfully landed on the surface of Mars. After a seven-month journey, the rover touched down in the Jezero Crater, a location scientists believe may have once held a lake and could potentially contain signs of ancient microbial life. The rover's primary mission is to search for evidence of past life on Mars and collect rock and soil samples for future return to Earth. Equipped with advanced scientific instruments, including cameras, spectrometers, and a drill, Perseverance will begin its exploration of the Martian surface, providing valuable data and insights into the planet's geology and potential habitability. This successful landing marks a significant milestone in humanity's quest to understand the red planet and paves the way for future manned missions to Mars. NASA's Perseverance rover is poised to unravel the mysteries of Mars and unlock new possibilities
System Fingerprint	fp_772e8125bb
Number of prompt tokens	29
Number of completion tokens	200
The average similarity between responses is: 0.1136714512418833
```

{{< /scrollbox >}}

اکنون، بیایید همان کد را با یک `seed` ثابت 123 و `temperature` برابر با 0 اجرا کنیم و پاسخ‌ها و `system_fingerprint` را مقایسه کنیم.

```python
SEED = 123
responses = []

async def get_response(i):
    print(f'Output {i + 1}\n{"-" * 10}')
    response = await get_chat_response(
        system_message=system_message,
        seed=SEED,
        temperature=0,
        user_request=user_request,
    )
    return response

responses = await asyncio.gather(*[get_response(i) for i in range(5)])

average_distance = calculate_average_distance(responses)
print(f"The average distance between responses is: {average_distance}")
```

:خروجی

{{< scrollbox >}}

```bash
Output 1
----------
Response	"NASA's Perseverance Rover Successfully Lands on Mars In a historic achievement, NASA's Perseverance rover has successfully landed on the surface of Mars, marking a major milestone in the exploration of the red planet. The rover, which traveled over 293 million miles from Earth, is equipped with state-of-the-art instruments designed to search for signs of ancient microbial life and collect rock and soil samples for future return to Earth. This mission represents a significant step forward in our understanding of Mars and the potential for human exploration of the planet in the future."
System Fingerprint	fp_772e8125bb
Number of prompt tokens	29
Number of completion tokens	113
Output 2
----------
Response	"NASA's Perseverance rover successfully lands on Mars, marking a historic milestone in space exploration. The rover is equipped with advanced scientific instruments to search for signs of ancient microbial life and collect samples for future return to Earth. This mission paves the way for future human exploration of the red planet, as scientists and engineers continue to push the boundaries of space travel and expand our understanding of the universe."
System Fingerprint	fp_772e8125bb
Number of prompt tokens	29
Number of completion tokens	81
Output 3
----------
Response	"NASA's Perseverance rover successfully lands on Mars, marking a historic milestone in space exploration. The rover is equipped with advanced scientific instruments to search for signs of ancient microbial life and collect samples for future return to Earth. This mission paves the way for future human exploration of the red planet, as NASA continues to push the boundaries of space exploration."
System Fingerprint	fp_772e8125bb
Number of prompt tokens	29
Number of completion tokens	72
Output 4
----------
Response	"NASA's Perseverance rover successfully lands on Mars, marking a historic milestone in space exploration. The rover is equipped with advanced scientific instruments to search for signs of ancient microbial life and collect samples for future return to Earth. This mission paves the way for future human exploration of the red planet, as scientists and engineers continue to push the boundaries of space travel and expand our understanding of the universe."
System Fingerprint	fp_772e8125bb
Number of prompt tokens	29
Number of completion tokens	81
Output 5
----------
Response	"NASA's Perseverance rover successfully lands on Mars, marking a historic milestone in space exploration. The rover is equipped with advanced scientific instruments to search for signs of ancient microbial life and collect samples for future return to Earth. This mission paves the way for future human exploration of the red planet, as scientists and engineers continue to push the boundaries of space travel."
System Fingerprint	fp_772e8125bb
Number of prompt tokens	29
Number of completion tokens	74
The average distance between responses is: 0.0449054397632461
```

{{< /scrollbox >}}

همانطور که مشاهده می‌کنیم، پارامتر `seed` به ما امکان می‌دهد تا نتایج بسیار ثابتی تولید کنیم.

### نتیجه‌گیری

ما نشان دادیم چگونه می‌توان با استفاده از مقدار `seed` ثابت، خروجی‌های ثابتی از مدل تولید کرد. این ویژگی به ویژه در سناریوهایی که بازتولید مهم است، مفید است. با این حال، توجه داشته باشید که اگرچه `seed` ثبات را تضمین می‌کند، کیفیت خروجی را تضمین نمی‌کند. توجه داشته باشید که وقتی می‌خواهید خروجی‌های قابل بازتولید استفاده کنید، باید `seed` را به همان عدد صحیح در درخواست‌های `Chat Completions` تنظیم کنید. همچنین باید هر پارامتر دیگری مانند `temperature`، `max_tokens` و غیره را تطبیق دهید.