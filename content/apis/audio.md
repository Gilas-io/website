---
title: 'v1/audio/'
weight: 1002
---



مدل‌های صوتی برای تبدیل صوت به متن و همچنین متن به صوت قابل استفاده هستند. این مدل‌های از دقت بسیار بالاتری نسبت به سرویس‌های مشابه برخوردار بوده و همچنین صوت تولید شده شباهت زیادی به صدای انسان دارد.
مدل Whisper قادر به تبدیل صوت به متن است و می‌توانند زبان‌های مختلف را به یکدیگر ترجمه کند.
مدل TTS (Text-to-Speech) هم قادر به تبدیل متن به صوت به زبان‌های مختلف است.

## تبدیل متن به صوت

 برای تبدیل متن به صوت کافی‌ است درخواستی مشابه مثال زیر را به اندپوینت `v1/audio/speech` ارسال کنید. 

برای آگاهای از پارامترهای قابل استفاده در هنگام ارسال درخواست به این اندپوینت می‌توانید [مستندات OpenAI](https://platform.openai.com/docs/api-reference/audio/createSpeech) را مطالعه کنید.


```shell
curl https://api.gilas.io/v1/audio/speech \
  -H "Authorization: Bearer $GILAS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "tts-1",
    "input": "درختان برای رشد به خاک, هوا و نور خورشید نیاز دارند.",
    "voice": "alloy"
  }' \
  --output speech.mp3
```

خروجی این درخواست یک فایل صوتی با فرمت .mp3 خواهد بود.

## تبدیل صوت به متن

 برای تبدیل صوت به متن کافی‌ است درخواستی مشابه مثال زیر را به اندپوینت `v1/audio/transcriptions` ارسال کنید.

برای آگاهای از پارامترهای قابل استفاده در هنگام ارسال درخواست به این اندپوینت می‌توانید [مستندات OpenAI](https://platform.openai.com/docs/api-reference/audio/createTranscription) را مطالعه کنید.

{{< tabs "transcriptions" >}}
{{< tab "Default" >}}

Request:
```shell
curl https://api.gilas.io/v1/audio/transcriptions \
  -H "Authorization: Bearer $GILAS_API_KEY" \
  -H "Content-Type: multipart/form-data" \
  -F file="@/path/to/file/audio.mp3" \
  -F model="whisper-1"
```

Response:
```shell
{
  "text": "Imagine the wildest idea that you've ever had, and you're curious about how it might scale to something that's a 100, a 1,000 times bigger. This is a place where you can get to do that."
}
```

{{< /tab >}}
{{< tab "Word Timestamps" >}}

Request:
```shell
curl https://api.gilas.io/v1/audio/transcriptions \
  -H "Authorization: Bearer $GILAS_API_KEY" \
  -H "Content-Type: multipart/form-data" \
  -F file="@/path/to/file/audio.mp3" \
  -F "timestamp_granularities[]=word" \
  -F model="whisper-1" \
  -F response_format="verbose_json"
```

Response:
```shell
{
  "task": "transcribe",
  "language": "english",
  "duration": 8.470000267028809,
  "text": "The beach was a popular spot on a hot summer day. People were swimming in the ocean, building sandcastles, and playing beach volleyball.",
  "words": [
    {
      "word": "The",
      "start": 0.0,
      "end": 0.23999999463558197
    },
    ...
    {
      "word": "volleyball",
      "start": 7.400000095367432,
      "end": 7.900000095367432
    }
  ]
}
```
{{< /tab >}}

{{< tab "Segment timestamps" >}}

Request:
```shell
curl https://api.gilas.io/v1/audio/transcriptions \
  -H "Authorization: Bearer $GILAS_API_KEY" \
  -H "Content-Type: multipart/form-data" \
  -F file="@/path/to/file/audio.mp3" \
  -F "timestamp_granularities[]=segment" \
  -F model="whisper-1" \
  -F response_format="verbose_json"
```

Response:
```shell
{
  "task": "transcribe",
  "language": "english",
  "duration": 8.470000267028809,
  "text": "The beach was a popular spot on a hot summer day. People were swimming in the ocean, building sandcastles, and playing beach volleyball.",
  "segments": [
    {
      "id": 0,
      "seek": 0,
      "start": 0.0,
      "end": 3.319999933242798,
      "text": " The beach was a popular spot on a hot summer day.",
      "tokens": [
        50364, 440, 7534, 390, 257, 3743, 4008, 322, 257, 2368, 4266, 786, 13, 50530
      ],
      "temperature": 0.0,
      "avg_logprob": -0.2860786020755768,
      "compression_ratio": 1.2363636493682861,
      "no_speech_prob": 0.00985979475080967
    },
    ...
  ]
}

```
{{< /tab >}}
{{< /tabs >}}

## ترجمه صوتی

 برای ترجمه صوت به زبان مورد نظر خود کافی‌ است درخواستی مشابه مثال زیر را به اندپوینت `v1/audio/translations` ارسال کنید. 

برای آگاهای از پارامترهای قابل استفاده در هنگام ارسال درخواست به این اندپوینت می‌توانید [مستندات OpenAI](https://platform.openai.com/docs/api-reference/audio/createTranslation) را مطالعه کنید.


Requst:
```shell
curl https://api.gilas.io/v1/audio/translations \
  -H "Authorization: Bearer $GILAS_API_KEY" \
  -H "Content-Type: multipart/form-data" \
  -F file="@/path/to/file/german.m4a" \
  -F model="whisper-1"
```

Response:
```shell
{
  "text": "سلام. اسم من مارتین هست و اهل آلمان هستم. برنامه‌ی شما امروز چی هست؟"
}
```

{{< hint info >}}
**توجه**  
در نظر داشته باشید که Gilas APIs از لحاظ فنی و نحوه کارکرد و قابلیت‌ها کاملا شبیه OpenAI APIs هستند. به همین منظور پیشنهاد میکنیم که برای آگاهی از نحوه‌ی کارکرد API ها به مستندات [OpenAI API Reference](https://platform.openai.com/docs/api-reference/audio) و [OpenAI Documentation](https://platform.openai.com/docs/guides/text-to-speech) ارجاع کنید.
{{< /hint >}}