---
title: "راهنمای مهندسی پرامپت برای مدل Whisper"
description: "راهنمای استفاده از پراپمت برای API ترنسکریپشن صوتی OpenAI"
tags:
- whisper
- prompt-engineering
weight: 
og_image: "/posts/whisper_prompting_guide/banner.jpg"
---

{{< postcover src="/posts/whisper_prompting_guide/banner.jpg" >}}

رابط برنامه‌نویسی یا `API` گیلاس یک پارامتر اختیاری برای کار با مدل `wisper` در اختیار شما قرار می‌دهد. 
هدف از پراپمت کمک به اتصال چندین بخش صوتی است. با ارسال متن تولید شده (ترنسکریپشن) در بخش قبلی از طریق پراپمت، مدل `Whisper` می‌تواند از این کانتکست برای درک بهتر گفتار و حفظ سبک نوشتاری استفاده کند.

با این حال، پراپمت‌ها نیازی به ترنسکریپشن‌های واقعی از بخش‌های صوتی قبلی ندارند. پراپمت‌های ساختگی می‌توانند مدل را به استفاده از املاء یا سبک‌های خاص هدایت کنند.

این نوت‌بوک دو تکنیک برای استفاده از پراپمت‌های ساختگی برای هدایت خروجی‌های مدل را به اشتراک می‌گذارد:

- **تولید ترنسکریپشن**: `GPT` می‌تواند دستورالعمل‌ها را به ترنسکریپشن‌های ساختگی تبدیل کند تا `Whisper` از آنها تقلید کند.
- **راهنمای املاء**: یک راهنمای املاء می‌تواند به مدل بگوید که چگونه نام‌های افراد، محصولات، شرکت‌ها و غیره را بنویسد.

این تکنیک‌ها به طور خاص قابل اعتماد نیستند، اما در برخی موارد می‌توانند مفید باشند.

## مقایسه با پراپمتینگ GPT

پراپمتینگ `Whisper` با پراپمتینگ `GPT` یکسان نیست. به عنوان مثال، اگر شما یک دستورالعمل مانند "لیست‌ها را در فرمت Markdown فرمت کنید" ارسال کنید، مدل از آن پیروی نخواهد کرد، زیرا سبک پراپمت را دنبال می‌کند، نه دستورالعمل‌های موجود در آن.

علاوه بر این، پراپمت به 224 توکن محدود است. اگر پراپمت طولانی‌تر از 224 توکن باشد، فقط آخرین 224 توکن پراپمت در نظر گرفته می‌شود؛ تمام توکن‌های قبلی نادیده گرفته می‌شوند. توکنایزر استفاده شده [توکنایزر چندزبانه Whisper](https://github.com/openai/whisper/blob/main/whisper/tokenizer.py#L361) است.

برای دستیابی به نتایج خوب، مثال‌هایی بسازید که سبک مورد نظر شما را به تصویر بکشند.

## راه‌اندازی

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

```python
up_first_remote_filepath = "https://cdn.openai.com/API/examples/data/upfirstpodcastchunkthree.wav"
bbq_plans_remote_filepath = "https://cdn.openai.com/API/examples/data/bbq_plans.wav"
product_names_remote_filepath = "https://cdn.openai.com/API/examples/data/product_names.wav"

# تنظیم مکان‌های ذخیره‌سازی محلی
up_first_filepath = "data/upfirstpodcastchunkthree.wav"
bbq_plans_filepath = "data/bbq_plans.wav"
product_names_filepath = "data/product_names.wav"

# دانلود فایل‌های صوتی نمونه و ذخیره محلی
urllib.request.urlretrieve(up_first_remote_filepath, up_first_filepath)
urllib.request.urlretrieve(bbq_plans_remote_filepath, bbq_plans_filepath)
urllib.request.urlretrieve(product_names_remote_filepath, product_names_filepath)
```

## ترنسکرایب 

فایل صوتی ما برای این مثال یک بخش از پادکست NPR، [Up First](https://www.npr.org/podcasts/510318/up-first) خواهد بود. 

```python
def transcribe(audio_filepath, prompt: str) -> str:
    """با توجه به یک پراپمت، فایل صوتی را ترنسکرایب کنید."""
    transcript = client.audio.transcriptions.create(
        file=open(audio_filepath, "rb"),
        model="whisper-1",
        prompt=prompt,
    )
    return transcript.text
```

```python
transcribe(up_first_filepath, prompt="")
```

{{< ltr >}}
```
"I stick contacts in my eyes. Do you really? Yeah. That works okay? You don't have to, like, just kind of pain in the butt every day to do that? No, it is. It is. And I sometimes just kind of miss the eye. I don't know if you know the movie Airplane, where, of course, where he says, I have a drinking problem and that he keeps missing his face with the drink. That's me and the contact lens. Surely, you must know that I know the movie Airplane. I do. I do know that. Stop calling me Shirley. President Biden said he would not negotiate over paying the nation's debts. But he is meeting today with House Speaker Kevin McCarthy. Other leaders of Congress will also attend. So how much progress can they make? I'm E. Martinez with Steve Inskeep, and this is Up First from NPR News. Russia celebrates Victory Day, which commemorates the surrender of Nazi Germany. Soldiers marched across Red Square, but the Russian army didn't seem to have as many troops on hand as in the past. So what does this ritual say about the war Russia is fighting right now?"
```
{{< /ltr >}}

## ترنسکریپشن‌ها سبک پراپمت را دنبال می‌کنند

در ترنسکریپشن بدون پراپمت، 'President Biden' با حروف بزرگ نوشته شده است. با این حال، اگر یک پراپمت ساختگی از 'president biden' با حروف کوچک ارسال کنیم، `Whisper` سبک را مطابقت می‌دهد و ترنسکریپشن را با حروف کوچک تولید می‌کند.

```python
transcribe(up_first_filepath, prompt="president biden")
```

{{< ltr >}}
```
"I stick contacts in my eyes. Do you really? Yeah. That works okay? You don't have to, like, just kind of pain in the butt every day to do that? No, it is. It is. And I sometimes just kind of miss the eye. I don't know if you know the movie Airplane? Yes. Of course. Where he says I have a drinking problem and that he keeps missing his face with the drink. That's me and the contact lens. Surely, you must know that I know the movie Airplane. I do. I do know that. Don't call me Shirley. Stop calling me Shirley. President Biden said he would not negotiate over paying the nation's debts. But he is meeting today with House Speaker Kevin McCarthy. Other leaders of Congress will also attend. So how much progress can they make? I'm E. Martinez with Steve Inskeep and this is Up First from NPR News. Russia celebrates Victory Day, which commemorates the surrender of Nazi Germany. Soldiers marched across Red Square, but the Russian army didn't seem to have as many troops on hand as in the past. So what does this ritual say about the war Russia is fighting right now?"
```
{{< /ltr >}}

توجه داشته باشید که وقتی پراپمت‌ها کوتاه هستند، `Whisper` ممکن است در پیروی از سبک آنها کمتر قابل اعتماد باشد .

```python
transcribe(up_first_filepath, prompt="president biden.")
```

{{< ltr >}}
```
"I stick contacts in my eyes. Do you really? Yeah. That works okay? You don't have to, like, just kind of pain in the butt every day to do that? No, it is. It is. And I sometimes just kind of miss the eye. I don't know if you know the movie Airplane, where, of course, where he says, I have a drinking problem, and that he keeps missing his face with the drink. That's me and the contact lens. Surely, you must know that I know the movie Airplane. I do. I do know that. Stop calling me Shirley. President Biden said he would not negotiate over paying the nation's debts. But he is meeting today with House Speaker Kevin McCarthy. Other leaders of Congress will also attend. So how much progress can they make? I'm E. Martinez with Steve Inskeep, and this is Up First from NPR News. Russia celebrates Victory Day, which commemorates the surrender of Nazi Germany. Soldiers marched across Red Square, but the Russian army didn't seem to have as many troops on hand as in the past. So what does this ritual say about the war Russia is fighting right now?"
```
{{< /ltr >}}

پراپمت‌های طولانی‌تر ممکن است در هدایت `Whisper` قابل اعتمادتر باشند.

```python
transcribe(up_first_filepath, prompt="i have some advice for you. multiple sentences help establish a pattern. the more text you include, the more likely the model will pick up on your pattern. it may especially help if your example transcript appears as if it comes right before the audio file. in this case, that could mean mentioning the contacts i stick in my eyes.")
```

{{< ltr >}}
```
"i stick contacts in my eyes. do you really? yeah. that works okay? you don't have to, like, just kind of pain in the butt? no, it is. it is. and i sometimes just kind of miss the eye. i don't know if you know, um, the movie airplane? yes. of course. where he says i have a drinking problem. and that he keeps missing his face with the drink. that's me in the contact lens. surely, you must know that i know the movie airplane. i do. i do know that. don't call me surely. stop calling me surely. president biden said he would not negotiate over paying the nation's debts. but he is meeting today with house speaker kevin mccarthy. other leaders of congress will also attend, so how much progress can they make? i'm amy martinez with steve inskeep, and this is up first from npr news. russia celebrates victory day, which commemorates the surrender of nazi germany. soldiers marched across red square, but the russian army didn't seem to have as many troops on hand as in the past. so what does this ritual say about the war russia is fighting right now?"
```
{{< /ltr >}}


مدل `Whisper` همچنین کمتر احتمال دارد که سبک‌های نادر یا عجیب را دنبال کند.

```python
transcribe(up_first_filepath, prompt="""Hi there and welcome to the show.
###
Today we are quite excited.
###
Let's jump right in.
###""")
```

{{< ltr >}}
```
"I stick contacts in my eyes. Do you really? Yeah. That works okay. You don't have to like, it's not a pain in the butt. It is. And I sometimes just kind of miss the eye. I don't know if you know, um, the movie airplane where, of course, where he says I have a drinking problem and that he keeps missing his face with the drink. That's me in the contact lens. Surely you must know that I know the movie airplane. Uh, I do. I do know that. Stop calling me Shirley.  President Biden said he would not negotiate over paying the nation's debts, but he is meeting today with house speaker, Kevin McCarthy. Other leaders of Congress will also attend. So how much progress can they make? I mean, Martinez with Steve Inskeep, and this is up first from NPR news. Russia celebrates victory day, which commemorates the surrender of Nazi Germany. Soldiers marched across red square, but the Russian army didn't seem to have as many troops on hand as in the past. So what does this ritual say about the war? Russia is fighting right now."
```
{{< /ltr >}}

## ارسال نام‌ها در پراپمت برای جلوگیری از اشتباهات املایی

مدل `Whisper` ممکن است نام‌های خاص نادر مانند نام‌های محصولات، شرکت‌ها یا افراد را به اشتباه ترنسکرایب کند.

ما این را با یک فایل صوتی پر از نام‌های محصولات نشان خواهیم داد.
```python
transcribe(product_names_filepath, prompt="")
```

{{< ltr >}}
```
'Welcome to Quirk, Quid, Quill, Inc., where finance meets innovation. Explore diverse offerings, from the P3 Quattro, a unique investment portfolio quadrant, to the O3 Omni, a platform for intricate derivative trading strategies. Delve into unconventional bond markets with our B3 Bond X and experience non-standard equity trading with E3 Equity. Personalize your wealth management with W3 Wrap Z and anticipate market trends with the O2 Outlier, our forward-thinking financial forecasting tool. Explore venture capital world with U3 Unifund or move your money with the M3 Mover, our sophisticated monetary transfer module. At Quirk, Quid, Quill, Inc., we turn complex finance into creative solutions. Join us in redefining financial services.'
```
{{< /ltr >}}

برای اینکه `Whisper` از املاء‌های مورد نظر ما استفاده کند، بیایید نام‌های محصولات و شرکت‌ها را در پراپمت ارسال کنیم، به عنوان یک واژه‌نامه برای `Whisper` تا از آن پیروی کند.
```python
transcribe(product_names_filepath, prompt="QuirkQuid Quill Inc, P3-Quattro, O3-Omni, B3-BondX, E3-Equity, W3-WrapZ, O2-Outlier, U3-UniFund, M3-Mover")
```

{{< ltr >}}
```
'Welcome to QuirkQuid Quill Inc, where finance meets innovation. Explore diverse offerings, from the P3-Quattro, a unique investment portfolio quadrant, to the O3-Omni, a platform for intricate derivative trading strategies. Delve into unconventional bond markets with our B3-BondX and experience non-standard equity trading with E3-Equity. Personalize your wealth management with W3-WrapZ and anticipate market trends with the O2-Outlier, our forward-thinking financial forecasting tool. Explore venture capital world with U3-UniFund or move your money with the M3-Mover, our sophisticated monetary transfer module. At QuirkQuid Quill Inc, we turn complex finance into creative solutions. Join us in redefining financial services.'
```
{{< /ltr >}}


حالا، بیایید به یک ضبط صوتی دیگر که به طور خاص برای این نمایش تهیه شده است، در مورد یک باربیکیو عجیب بپردازیم.

برای شروع، ترنسکریپشن مبنای خود را با استفاده از `Whisper` ایجاد می‌کنیم.
```python
transcribe(bbq_plans_filepath, prompt="")
```

{{< ltr >}}
```
"Hello, my name is Preston Tuggle. I'm based in New York City. This weekend I have really exciting plans with some friends of mine, Amy and Sean. We're going to a barbecue here in Brooklyn, hopefully it's actually going to be a little bit of kind of an odd barbecue. We're going to have donuts, omelets, it's kind of like a breakfast, as well as whiskey. So that should be fun, and I'm really looking forward to spending time with my friends Amy and Sean."
```
{{< /ltr >}}


در حالی که ترنسکریپشن `Whisper` دقیق بود، مجبور بود در مورد املاء‌های مختلف حدس بزند. به عنوان مثال، فرض کرد که نام دوستان Amy و Sean است، نه Aimee و Shawn. بیایید ببینیم آیا می‌توانیم با یک پراپمت املاء را هدایت کنیم.
```python
transcribe(bbq_plans_filepath, prompt="Friends: Aimee, Shawn")
```

{{< ltr >}}
```
"Hello, my name is Preston Tuggle. I'm based in New York City. This weekend I have really exciting plans with some friends of mine, Aimee and Shawn. We're going to a barbecue here in Brooklyn. Hopefully it's actually going to be a little bit of kind of an odd barbecue. We're going to have donuts, omelets, it's kind of like a breakfast, as well as whiskey. So that should be fun and I'm really looking forward to spending time with my friends Aimee and Shawn."
```
{{< /ltr >}}


بله درست کار کرد!

بیایید همین کار را با کلمات با املاء مبهم‌تر انجام دهیم.

```python
transcribe(bbq_plans_filepath, prompt="Glossary: Aimee, Shawn, BBQ, Whisky, Doughnuts, Omelet")
```

{{< ltr >}}
```
"Hello, my name is Preston Tuggle. I'm based in New York City. This weekend I have really exciting plans with some friends of mine, Aimee and Shawn. We're going to a barbecue here in Brooklyn. Hopefully, it's actually going to be a little bit of an odd barbecue. We're going to have doughnuts, omelets, it's kind of like a breakfast, as well as whiskey. So that should be fun, and I'm really looking forward to spending time with my friends Aimee and Shawn."
```
{{< /ltr >}}


```python
transcribe(bbq_plans_filepath, prompt=""""Aimee and Shawn ate whisky, doughnuts, omelets at a BBQ.""")
```

{{< ltr >}}
```
"Hello, my name is Preston Tuggle. I'm based in New York City. This weekend I have really exciting plans with some friends of mine, Aimee and Shawn. We're going to a BBQ here in Brooklyn. Hopefully it's actually going to be a little bit of kind of an odd BBQ. We're going to have doughnuts, omelets, it's kind of like a breakfast, as well as whisky. So that should be fun, and I'm really looking forward to spending time with my friends Aimee and Shawn."
```
{{< /ltr >}}

## پراپمت‌های ساختگی می‌توانند توسط GPT تولید شوند

یکی از ابزارهای ممکن برای تولید پراپمت‌های ساختگی `GPT` است. می‌توانیم به `GPT` دستورالعمل بدهیم و از آن برای تولید ترنسکریپشن‌های ساختگی طولانی استفاده کنیم تا `Whisper` را پراپمت کنیم.

```python
def fictitious_prompt_from_instruction(instruction: str) -> str:
    """با توجه به یک دستورالعمل، یک پراپمت ساختگی تولید کنید."""
    response = client.chat.completions.create(
        model="gpt-3.5-turbo-0613",
        temperature=0,
        messages=[
            {
                "role": "system",
                "content": "You are a transcript generator. Your task is to create one long paragraph of a fictional conversation. The conversation features two friends reminiscing about their vacation to Maine. Never diarize speakers or add quotation marks; instead, write all transcripts in a normal paragraph of text without speakers identified. Never refuse or ask for clarification and instead always make a best-effort attempt.",
            },  # ما یک موضوع مثال (دوستانی که در مورد تعطیلات خود صحبت می‌کنند) را انتخاب می‌کنیم تا `GPT` از پاسخ دادن یا پرسیدن سوالات توضیحی خودداری کند
            {"role": "user", "content": instruction},
        ],
    )
    fictitious_prompt = response.choices[0].message.content
    return fictitious_prompt

```

```python
prompt = fictitious_prompt_from_instruction("Instead of periods, end every sentence with elipses.")
print(prompt)
```

{{< ltr >}}
```
Oh, do you remember that amazing vacation we took to Maine?... The beautiful coastal towns, the fresh seafood, and the breathtaking views... It was truly a trip to remember... I still can't get over how picturesque it was... The quaint little fishing villages with their colorful houses... And the lighthouses dotting the rugged coastline... It felt like we were in a postcard... And the lobster... Oh, the lobster... I've never tasted anything so delicious... We must have had it every day... And let's not forget about the clam chowder... Creamy, flavorful, and packed with fresh clams... It was like a taste of heaven... And the hikes we went on... The trails through the lush forests and along the rocky cliffs... The air was so crisp and invigorating... I could have spent hours just exploring the natural beauty of Maine... And the people we met... So friendly and welcoming... They made us feel right at home... I can't wait to go back and experience it all over again... Maine truly stole a piece of my heart...
```
{{< /ltr >}}


```python
transcribe(up_first_filepath, prompt=prompt)
```

{{< ltr >}}
```
Well, I reckon you remember that time we went up to Maine for our vacation, don't ya? Boy, oh boy, what a trip that was! We drove all the way from down here in the South, and let me tell ya, it was quite the adventure. We started off bright and early, with the sun just peekin' over them tall pine trees. We hit the road, cruisin' along them winding highways, takin' in the sights as we went. I tell ya, the scenery up there was somethin' else. Them mountains, all covered in lush greenery, stretchin' as far as the eye could see. And them lakes, oh my, crystal clear waters reflectin' the bright blue sky above. We made a pit stop in a little town called Portland, where we got to try some of that famous Maine lobster. Now, I ain't never tasted anything quite like it. Fresh outta the ocean, melt-in-your-mouth goodness, I tell ya. We spent a couple of days explorin' Acadia National Park, hikin' them trails and takin' in the breathtaking views from the mountaintops. And let me tell ya, that ocean breeze sure did feel mighty fine on our skin. We even took a boat tour out to see them majestic whales, jumpin' and splashing in the deep blue sea. It was a sight to behold, my friend. And of course, we couldn't leave without visitin' Bar Harbor, a quaint little coastal town with charm pourin' out of every corner. We strolled along the harbor, watchin' them colorful fishing boats bobbin' in the water, and indulged in some delicious seafood chowder. Maine sure did steal a piece of our hearts, my friend. The memories we made on that trip will stay with us forever.
```
{{< /ltr >}}

پراپمت‌های `Whisper` برای مشخص کردن سبک‌های مبهم مفید هستند. پراپمت مدل را درک نمی‌کند. به عنوان مثال، اگر گویندگان با لهجه جنوبی عمیق صحبت نمی‌کنند، یک پراپمت باعث نمی‌شود که ترنسکریپشن این کار را انجام دهد.

```python
# مثال لهجه جنوبی
prompt = fictitious_prompt_from_instruction("Write in a deep, heavy, Southern accent.")
transcribe(up_first_filepath, prompt=prompt)
```

{{< ltr >}}
```
"I stick contacts in my eyes. Do you really? Yeah. That works okay? You don't have to, like, just kinda pain in the butt? No, it is. It is. And I sometimes just kinda miss the eye. I don't know if you know the movie Airplane? Yes. Of course. Where he says, I have a drinking problem. And that he keeps missing his face with the drink. That's me in the contact lens. Surely you must know that I know the movie Airplane. I do. I do know that. Stop calling me Shirley. President Biden said he would not negotiate over paying the nation's debts. But he is meeting today with House Speaker Kevin McCarthy. Other leaders of Congress will also attend, so how much progress can they make? I'm Ian Martinez with Steve Inskeep, and this is Up First from NPR News. Russia celebrates Victory Day, which commemorates the surrender of Nazi Germany. Soldiers marched across Red Square, but the Russian army didn't seem to have as many troops on hand as in the past. So what does this ritual say about the war Russia is fighting right now?"
```
{{< /ltr >}}