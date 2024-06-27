---
title: "بهبود ترنسکریپشن‌های Whisper: تکنیک‌های پیش و پس پردازش"
description: "این نوت‌بوک راهنمایی برای بهبود ترنسکریپشن‌های Whisper ارائه می‌دهد. با استفاده از تکنیک‌های برش و تقسیم‌بندی، کیفیت ترنسکریپشن‌ها را افزایش دهید."
tags:
- whisper
weight: 
og_image: "/posts/whisper_processing_guide/banner.png"
---

{{< postcover src="/posts/whisper_processing_guide/banner.png" >}}

این پست راهنمایی برای بهبود ترنسکریپشن‌های `Whisper` ارائه می‌دهد. ما داده‌های صوتی را از طریق برش و تقسیم‌بندی بهینه‌سازی می‌کنیم تا کیفیت ترنسکریپشن‌های `Whisper` را افزایش دهیم. پس از ترنسکریپشن، خروجی را با افزودن علائم نگارشی، تنظیم اصطلاحات محصول (مثلاً 'five two nine' به '529') و کاهش مشکلات `Unicode` بهبود می‌بخشیم. این استراتژی‌ها به بهبود وضوح ترنسکریپشن‌ها کمک می‌کنند، اما به یاد داشته باشید که سفارشی‌سازی بر اساس یوزکیس خاص شما ممکن است مفید باشد.

## راه‌اندازی

برای شروع، بیایید چند کتابخانه مختلف را وارد کنیم:

- کتابخانه‌ی [PyDub](http://pydub.com/) یک کتابخانه ساده و آسان برای استفاده در پایتون برای وظایف پردازش صوتی مانند برش، ترکیب و صادرات فایل‌های صوتی است.

- کلاس `Audio` از ماژول `IPython.display` به شما امکان می‌دهد یک کنترل صوتی ایجاد کنید که می‌تواند صدا را در نوت‌بوک‌های `Jupyter` پخش کند و راهی ساده برای پخش داده‌های صوتی مستقیماً در نوت‌بوک شما فراهم می‌کند.

- برای فایل صوتی ما، از یک فایل صوتی که متنش توسط `ChatGPT` نوشته شده و توسط نویسنده خوانده شده است، استفاده خواهیم کرد. این فایل صوتی نسبتاً کوتاه است، اما امیدواریم ایده‌ای تصویری از چگونگی اعمال این مراحل پیش و پس پردازش به هر فایل صوتی به شما بدهد.

{{< hint info >}}
**برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.**
{{< /hint >}} 

```python
from openai import OpenAI
import os
import urllib
from IPython.display import Audio
from pathlib import Path
from pydub import AudioSegment
import ssl

client = OpenAI(
    api_key=os.environ.get(("GILAS_API_KEY", "<کلید API خود را اینجا بسازید https://dashboard.gilas.io/apiKey>")), 
    base_url="https://api.gilas.io/v1/" # Gilas APIs
)
```

```python
# set download paths
earnings_call_remote_filepath = "https://cdn.openai.com/API/examples/data/EarningsCall.wav"

# set local save locations
earnings_call_filepath = "data/EarningsCall.wav"

# download example audio files and save locally
ssl._create_default_https_context = ssl._create_unverified_context
urllib.request.urlretrieve(earnings_call_remote_filepath, earnings_call_filepath)
```

گاهی اوقات، فایل‌هایی با سکوت طولانی در ابتدا می‌توانند باعث شوند `Whisper` صدا را به اشتباه ترنسکرایب کند. ما از `Pydub` برای تشخیص و برش سکوت استفاده خواهیم کرد.

در اینجا، آستانه دسیبل را روی 20 تنظیم کرده‌ایم. می‌توانید این مقدار را بر اساس نیاز خود تغییر دهید.

```python
# Function to detect leading silence
# Returns the number of milliseconds until the first sound (chunk averaging more than X decibels)
def milliseconds_until_sound(sound, silence_threshold_in_decibels=-20.0, chunk_size=10):
    trim_ms = 0  # ms

    assert chunk_size > 0  # to avoid infinite loop
    while sound[trim_ms:trim_ms+chunk_size].dBFS < silence_threshold_in_decibels and trim_ms < len(sound):
        trim_ms += chunk_size

    return trim_ms
```

```python
def trim_start(filepath):
    path = Path(filepath)
    directory = path.parent
    filename = path.name
    audio = AudioSegment.from_file(filepath, format="wav")
    start_trim = milliseconds_until_sound(audio)
    trimmed = audio[start_trim:]
    new_filename = directory / f"trimmed_{filename}"
    trimmed.export(new_filename, format="wav")
    return trimmed, new_filename
```

```python
def transcribe_audio(file,output_dir):
    audio_path = os.path.join(output_dir, file)
    with open(audio_path, 'rb') as audio_data:
        transcription = client.audio.transcriptions.create(
            model="whisper-1", file=audio_data)
        return transcription.text
```

گاهی اوقات، در ترنسکریپشن‌ها تزریق کاراکترهای `Unicode` مشاهده می‌شود، حذف هر کاراکتر غیر `ASCII` باید به کاهش این مشکل کمک کند.

به خاطر داشته باشید که اگر در حال ترنسکرایب به زبان‌های یونانی، سیریلیک، عربی، چینی و غیره هستید٬ نباید از این تابع استفاده کنید.

```python
# Define function to remove non-ascii characters
def remove_non_ascii(text):
    return ''.join(i for i in text if ord(i)<128)
```

این تابع فرمت‌بندی و علائم نگارشی را به ترنسکریپشن ما اضافه می‌کند. `Whisper` یک ترنسکریپشن با علائم نگارشی و بدون فرمت‌بندی تولید می‌کند.

برای آگاهی دقیق‌تر از نحوه پرامپت کردن مدل `whisper` پیشنهاد می‌دهیم
 [این پست](/posts/whisper_prompting_guide)
  را مطالعه کنید.


```python
# Define function to add punctuation
def punctuation_assistant(ascii_transcript):

    system_prompt = """You are a helpful assistant that adds punctuation to text.
      Preserve the original words and only insert necessary punctuation such as periods,
     commas, capialization, symbols like dollar sings or percentage signs, and formatting.
     Use only the context provided. If there is no context provided say, 'No context provided'\n"""
    response = client.chat.completions.create(
        model="gpt-3.5-turbo",
        temperature=0,
        messages=[
            {
                "role": "system",
                "content": system_prompt
            },
            {
                "role": "user",
                "content": ascii_transcript
            }
        ]
    )
    return response
```

فایل صوتی ما در مورد داده‌های مالی است که شامل بسیاری از محصولات مالی است. این تابع می‌تواند کمک کند تا اگر `Whisper` این نام‌های محصولات مالی را به اشتباه ترنسکرایب کرد، آنها تصحیح شوند.

```python
# Define function to fix product mispellings
def product_assistant(ascii_transcript):
    system_prompt = """You are an intelligent assistant specializing in financial products;
    your task is to process transcripts of earnings calls, ensuring that all references to
     financial products and common financial terms are in the correct format. For each
     financial product or common term that is typically abbreviated as an acronym, the full term 
    should be spelled out followed by the acronym in parentheses. For example, '401k' should be
     transformed to '401(k) retirement savings plan', 'HSA' should be transformed to 'Health Savings Account (HSA)'
    , 'ROA' should be transformed to 'Return on Assets (ROA)', 'VaR' should be transformed to 'Value at Risk (VaR)'
, and 'PB' should be transformed to 'Price to Book (PB) ratio'. Similarly, transform spoken numbers representing 
financial products into their numeric representations, followed by the full name of the product in parentheses. 
For instance, 'five two nine' to '529 (Education Savings Plan)' and 'four zero one k' to '401(k) (Retirement Savings Plan)'.
 However, be aware that some acronyms can have different meanings based on the context (e.g., 'LTV' can stand for 
'Loan to Value' or 'Lifetime Value'). You will need to discern from the context which term is being referred to 
and apply the appropriate transformation. In cases where numerical figures or metrics are spelled out but do not 
represent specific financial products (like 'twenty three percent'), these should be left as is. Your role is to
 analyze and adjust financial product terminology in the text. Once you've done that, produce the adjusted 
 transcript and a list of the words you've changed"""
    response = client.chat.completions.create(
        model="gpt-4",
        temperature=0,
        messages=[
            {
                "role": "system",
                "content": system_prompt
            },
            {
                "role": "user",
                "content": ascii_transcript
            }
        ]
    )
    return response
```

این تابع یک فایل جدید با نام 'trimmed' به نام فایل اصلی اضافه می‌کند.

```python
# Trim the start of the original audio file
trimmed_audio = trim_start(earnings_call_filepath)
```

```python
trimmed_audio, trimmed_filename = trim_start(earnings_call_filepath)
```

فایل صوتی گزارش درآمدی  ما نسبتاً کوتاه است، بنابراین بخش‌ها را به طور مناسب تنظیم خواهیم کرد. به خاطر داشته باشید که می‌توانید طول بخش‌ها را به دلخواه تنظیم کنید.

```python
# Segment audio
trimmed_audio = AudioSegment.from_wav(trimmed_filename)  # Load the trimmed audio file

one_minute = 1 * 60 * 1000  # Duration for each segment (in milliseconds)

start_time = 0  # Start time for the first segment

i = 0  # Index for naming the segmented files

output_dir_trimmed = "trimmed_earnings_directory"  # Output directory for the segmented files

if not os.path.isdir(output_dir_trimmed):  # Create the output directory if it does not exist
    os.makedirs(output_dir_trimmed)

while start_time < len(trimmed_audio):  # Loop over the trimmed audio file
    segment = trimmed_audio[start_time:start_time + one_minute]  # Extract a segment
    segment.export(os.path.join(output_dir_trimmed, f"trimmed_{i:02d}.wav"), format="wav")  # Save the segment
    start_time += one_minute  # Update the start time for the next segment
    i += 1  # Increment the index for naming the next file
```

```python
# Get list of trimmed and segmented audio files and sort them numerically
audio_files = sorted(
    (f for f in os.listdir(output_dir_trimmed) if f.endswith(".wav")),
    key=lambda f: int(''.join(filter(str.isdigit, f)))
)
```

```python
# Use a loop to apply the transcribe function to all audio files
transcriptions = [transcribe_audio(file, output_dir_trimmed) for file in audio_files]
```

```python
# Concatenate the transcriptions
full_transcript = ' '.join(transcriptions)
```

```python
print(full_transcript)
```

اولین نسخه از ترنسکریپت تولید شده:

{{< ltr >}}
```
Good afternoon, everyone. And welcome to FinTech Plus Sync's second quarter 2023 earnings call. I'm John Doe, CEO of FinTech Plus. We've had a stellar Q2 with a revenue of 125 million, a 25% increase year over year. Our gross profit margin stands at a solid 58%, due in part to cost efficiencies gained from our scalable business model. Our EBITDA has surged to 37.5 million, translating to a remarkable 30% EBITDA margin. Our net income for the quarter rose to 16 million, which is a noteworthy increase from 10 million in Q2 2022. Our total addressable market has grown substantially thanks to the expansion of our high yield savings product line and the new RoboAdvisor platform. We've been diversifying our asset-backed securities portfolio, investing heavily in collateralized. debt obligations, and residential mortgage-backed securities. We've also invested $25 million in AAA rated corporate bonds, enhancing our risk adjusted returns. As for our balance sheet, total assets reached $1.5 billion with total liabilities at $900 million, leaving us with a solid equity base of $600 million. Our debt-to-equity ratio stands at 1.5, a healthy figure considering our expansionary phase. We continue to see substantial organic user growth, with customer acquisition cost dropping by 15% and lifetime value growing by 25%. Our LTVCAC ratio is at an impressive 3.5%. In terms of risk management, we have a value-at-risk model in place with a 99%... confidence level indicating that our maximum loss will not exceed 5 million in the next trading day. We've adopted a conservative approach to managing our leverage and have a healthy tier one capital ratio of 12.5%. Our forecast for the coming quarter is positive. We expect revenue to be around 135 million and 8% quarter over quarter growth driven primarily by our cutting edge blockchain solutions and AI driven predictive analytics. We're also excited about the upcoming IPO of our FinTech subsidiary Pay Plus, which we expect to raise 200 million, significantly bolstering our liquidity and paving the way for aggressive growth strategies. We thank our shareholders for their continued faith in us and we look forward to an even more successful Q3. Thank you so much.
```
{{< /ltr >}}

حال می‌خواهیم کارکترهای اسکی را از متن تولید شده حذف کنیم:

```python
# Remove non-ascii characters from the transcript
ascii_transcript = remove_non_ascii(full_transcript)
```

```python
print(ascii_transcript)
```

{{< ltr >}}
```
Good afternoon, everyone. And welcome to FinTech Plus Sync's second quarter 2023 earnings call. I'm John Doe, CEO of FinTech Plus. We've had a stellar Q2 with a revenue of 125 million, a 25% increase year over year. Our gross profit margin stands at a solid 58%, due in part to cost efficiencies gained from our scalable business model. Our EBITDA has surged to 37.5 million, translating to a remarkable 30% EBITDA margin. Our net income for the quarter rose to 16 million, which is a noteworthy increase from 10 million in Q2 2022. Our total addressable market has grown substantially thanks to the expansion of our high yield savings product line and the new RoboAdvisor platform. We've been diversifying our asset-backed securities portfolio, investing heavily in collateralized. debt obligations, and residential mortgage-backed securities. We've also invested $25 million in AAA rated corporate bonds, enhancing our risk adjusted returns. As for our balance sheet, total assets reached $1.5 billion with total liabilities at $900 million, leaving us with a solid equity base of $600 million. Our debt-to-equity ratio stands at 1.5, a healthy figure considering our expansionary phase. We continue to see substantial organic user growth, with customer acquisition cost dropping by 15% and lifetime value growing by 25%. Our LTVCAC ratio is at an impressive 3.5%. In terms of risk management, we have a value-at-risk model in place with a 99%... confidence level indicating that our maximum loss will not exceed 5 million in the next trading day. We've adopted a conservative approach to managing our leverage and have a healthy tier one capital ratio of 12.5%. Our forecast for the coming quarter is positive. We expect revenue to be around 135 million and 8% quarter over quarter growth driven primarily by our cutting edge blockchain solutions and AI driven predictive analytics. We're also excited about the upcoming IPO of our FinTech subsidiary Pay Plus, which we expect to raise 200 million, significantly bolstering our liquidity and paving the way for aggressive growth strategies. We thank our shareholders for their continued faith in us and we look forward to an even more successful Q3. Thank you so much.
```
{{< /ltr >}}

حال می‌خواهیم علامت‌های نگارشی صحیح را به متن اضافه کنیم:

```python
# Use punctuation assistant function
response = punctuation_assistant(ascii_transcript)
```

```python
# Extract the punctuated transcript from the model's response
punctuated_transcript = response.choices[0].message.content
```

```python
print(punctuated_transcript)
```

{{< ltr >}}
```
Good afternoon, everyone. And welcome to FinTech Plus Sync's second quarter 2023 earnings call. I'm John Doe, CEO of FinTech Plus. We've had a stellar Q2 with a revenue of $125 million, a 25% increase year over year. Our gross profit margin stands at a solid 58%, due in part to cost efficiencies gained from our scalable business model. Our EBITDA has surged to $37.5 million, translating to a remarkable 30% EBITDA margin. Our net income for the quarter rose to $16 million, which is a noteworthy increase from $10 million in Q2 2022. Our total addressable market has grown substantially thanks to the expansion of our high yield savings product line and the new RoboAdvisor platform. We've been diversifying our asset-backed securities portfolio, investing heavily in collateralized debt obligations, and residential mortgage-backed securities. We've also invested $25 million in AAA rated corporate bonds, enhancing our risk-adjusted returns. As for our balance sheet, total assets reached $1.5 billion with total liabilities at $900 million, leaving us with a solid equity base of $600 million. Our debt-to-equity ratio stands at 1.5, a healthy figure considering our expansionary phase. We continue to see substantial organic user growth, with customer acquisition cost dropping by 15% and lifetime value growing by 25%. Our LTVCAC ratio is at an impressive 3.5%. In terms of risk management, we have a value-at-risk model in place with a 99% confidence level indicating that our maximum loss will not exceed $5 million in the next trading day. We've adopted a conservative approach to managing our leverage and have a healthy tier one capital ratio of 12.5%. Our forecast for the coming quarter is positive. We expect revenue to be around $135 million and 8% quarter over quarter growth driven primarily by our cutting-edge blockchain solutions and AI-driven predictive analytics. We're also excited about the upcoming IPO of our FinTech subsidiary Pay Plus, which we expect to raise $200 million, significantly bolstering our liquidity and paving the way for aggressive growth strategies. We thank our shareholders for their continued faith in us and we look forward to an even more successful Q3. Thank you so much.
```
{{< /ltr >}}

و در نهایت می‌خواهیم به مدل کمک کنیم تا از اسم صحیح محصولات مالی در هنگام تولید ترنسکریپت استفاده کند.

```python
# Use product assistant function
response = product_assistant(punctuated_transcript)
```

```python
# Extract the final transcript from the model's response
final_transcript = response.choices[0].message.content
```

```python
print(final_transcript)
```

{{< ltr >}}
```
Good afternoon, everyone. And welcome to FinTech Plus Sync's second quarter 2023 earnings call. I'm John Doe, CEO of FinTech Plus. We've had a stellar second quarter (Q2) with a revenue of $125 million, a 25% increase year over year. Our gross profit margin stands at a solid 58%, due in part to cost efficiencies gained from our scalable business model. Our Earnings Before Interest, Taxes, Depreciation, and Amortization (EBITDA) has surged to $37.5 million, translating to a remarkable 30% EBITDA margin. Our net income for the quarter rose to $16 million, which is a noteworthy increase from $10 million in second quarter (Q2) 2022. Our total addressable market has grown substantially thanks to the expansion of our high yield savings product line and the new RoboAdvisor platform. We've been diversifying our asset-backed securities portfolio, investing heavily in Collateralized Debt Obligations (CDOs), and Residential Mortgage-Backed Securities (RMBS). We've also invested $25 million in AAA rated corporate bonds, enhancingour risk-adjusted returns. As for our balance sheet, total assets reached $1.5 billion with total liabilities at $900 million, leaving us with a solid equity base of $600 million. Our Debt-to-Equity (D/E) ratio stands at 1.5, a healthy figure considering our expansionary phase. We continue to see substantial organic user growth, with Customer Acquisition Cost (CAC) dropping by 15% and Lifetime Value (LTV) growing by 25%. Our LTV to CAC (LTVCAC) ratio is at an impressive 3.5%. In terms of risk management, we have a Value at Risk (VaR) model in place with a 99% confidence level indicating that our maximum loss will not exceed $5 million in the next trading day. We've adopted a conservative approach to managing our leverage and have a healthy Tier 1 Capital ratio of 12.5%. Our forecast for the coming quarter is positive. We expect revenue to be around $135 million and 8% quarter over quarter growth driven primarily by our cutting-edge blockchain solutions and AI-driven predictive analytics. We're also excited about the upcoming Initial Public Offering (IPO) of our FinTech subsidiary Pay Plus, which we expect to raise $200 million, significantly bolstering our liquidity and paving the way for aggressive growth strategies. We thank our shareholders for their continued faith in us and we look forward to an even more successful third quarter (Q3). Thank you so much.

Words Changed:
1. Q2 -> second quarter (Q2)
2. EBITDA -> Earnings Before Interest, Taxes, Depreciation, and Amortization (EBITDA)
3. Q2 2022 -> second quarter (Q2) 2022
4. CDOs -> Collateralized Debt Obligations (CDOs)
5. RMBS -> Residential Mortgage-Backed Securities (RMBS)
6. D/E -> Debt-to-Equity (D/E)
7. CAC -> Customer Acquisition Cost (CAC)
8. LTV -> Lifetime Value (LTV)
9. LTVCAC -> LTV to CAC (LTVCAC)
10. VaR -> Value at Risk (VaR)
11. IPO -> Initial Public Offering (IPO)
12. Q3 -> third quarter (Q3)
```
{{< /ltr >}}