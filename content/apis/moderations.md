---
title: 'v1/moderations/'
weight: 1004
next: /posts/p1
---



مدل‌های Moderation وظیفه بررسی متن ورودی و مشخص کردن اینکه آیا متن دارای محتوای نامناسب است را دارند. اگر متن ورودی نامناسب تضخیص داده شود, فراخوانی‌های بعدی به مدل‌های دیگر برای این متن رد خواهند شد.


این مدل‌ها محتوای متن ورودی را از جنبه های زیر مورد بررسی قرار می‌دهند:
- نفرت‌پراکنی
- آزار و اذیت
- خودآزاری
- خشونت
- سکس

## Moderations API

استفاده از اندپوینت Moderations رایگان است. توصیه می‌شود برای افزایش دقت مدل متن ورودی را به chunkهایی به طول حداکثر ۲۰۰۰ کاراکتر تبدیل کنید. 

نمونه‌ای از فواخوانی  APIی  moderations را در زیر مشاهده کنید:


```shell
curl https://api.gilas.io/v1/moderations \
  -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $GILAS_API_KEY" \
  -d '{"input": "Sample text goes here"}'
```

در زیر نمونه‌ای از خروجی تولید شده توسط این اندپوینت را مشاهده می‌کنید.

- در صورتی که کلید `flagged` برابر با مقدار `true` باشد, مدل تشخیص داده است که محتوای متن ورودی مغایر با قوانین OpenAI است.
- `categories` شامل دسته‌بندی های مختلفی که محتوای متن را بررسی کرده‌اند می‌شود. بسته به محتوای متن ورودی ممکن است تعدادی از این دسته‌بندی ها مقدار `true` داشته باشند که معنی آن این است که متن ورودی دارای محتوایی از این دسته است.
- `category_scores` نمایش عددی میزان اعتماد مدل در طبقه‌بندی متن بر اساس هر دسته می‌باشد.

```shell
{
  "id": "modr-XXXXX",
  "model": "omni-moderation-latest",
  "results": [
    {
      "flagged": true,
      "categories": {
        "sexual": false,
        "hate": false,
        "harassment": false,
        "self-harm": false,
        "sexual/minors": false,
        "hate/threatening": false,
        "violence/graphic": false,
        "self-harm/intent": false,
        "self-harm/instructions": false,
        "harassment/threatening": true,
        "violence": true,
      },
      "category_scores": {
        "sexual": 1.2282071e-06,
        "hate": 0.010696256,
        "harassment": 0.29842457,
        "self-harm": 1.5236925e-08,
        "sexual/minors": 5.7246268e-08,
        "hate/threatening": 0.0060676364,
        "violence/graphic": 4.435014e-06,
        "self-harm/intent": 8.098441e-10,
        "self-harm/instructions": 2.8498655e-11,
        "harassment/threatening": 0.63055265,
        "violence": 0.99011886,
      }
    }
  ]
}
```

{{< hint info >}}
**توجه**  
در نظر داشته باشید که Gilas APIs از لحاظ فنی و نحوه کارکرد و قابلیت‌ها کاملا شبیه OpenAI APIs هستند. به همین منظور پیشنهاد میکنیم که برای آگاهی از نحوه‌ی کارکرد API ها به مستندات [OpenAI API Reference](https://platform.openai.com/docs/api-reference/moderations) و [OpenAI Documentation](https://platform.openai.com/docs/guides/moderation) ارجاع کنید.
{{< /hint >}}

