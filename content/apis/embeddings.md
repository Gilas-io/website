---
title: 'v1/embeddings/'
weight: 1001
---

مدل‌های Embeddings توانایی اندازه‌گیری مرتبط بودن متون به یکدیگر را دارند. از Embeddings برای منظورهای مختلفی استفاده می‌شود, از جمله:

- جستجوی میان متون برای یافتن متن‌هایی که از جنبه‌های متفاوتی نزدیک به هم هستند.
- دسته بندی متون با توجه به نزدیکی محتوای آنها.
- توسعه سیستم‌های توصیه کننده
- تشخیص الگوهای متنی غیر متعارف

یک Embedding یک بردار از اعداد اعشاری است. فاصله بین دو متن بر اساس فاصله بین بردارهای آنها اندازه‌گیری می‌شود.

## Embeddings API

برای ساخت یک بردار Embeddings کافی است تا متن مورد نظر خود را همراه با مدلی که می‌خواهید برای ساخت Embeddings استفاده کنید به اندپوینت /v1/embeddings ارسال کنید.

نمونه‌ای از فواخوانی  APIی Embeddings را در زیر مشاهده کنید:



```shell
curl https://api.gilas.io/v1/embeddings \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $GILAS_API_KEY" \
  -d '{
    "input": "Your text string goes here",
    "model": "text-embedding-3-small"
  }'
```

پاسخ مدل به درخواست شما یک بردار از اعداد اعشاری همراه با تعدادی متادیتا می‌باشد.
```shell
{
  "object": "list",
  "data": [
    {
      "object": "embedding",
      "index": 0,
      "embedding": [
        -0.006929283495992422,
        -0.005336422007530928,
        ... (omitted for spacing)
        -4.547132266452536e-05,
        -0.024047505110502243
      ],
    }
  ],
  "model": "text-embedding-3-small",
  "usage": {
    "prompt_tokens": 5,
    "total_tokens": 5
  }
}
```

{{< hint info >}}
**توجه**  
در نظر داشته باشید که Gilas APIs از لحاظ فنی و نحوه کارکرد و قابلیت‌ها کاملا شبیه OpenAI APIs هستند. به همین منظور پیشنهاد میکنیم که برای آگاهی از نحوه‌ی کارکرد API ها به مستندات [OpenAI API Reference](https://platform.openai.com/docs/api-reference/embeddings) و [OpenAI Documentation](https://platform.openai.com/docs/guides/embeddings) ارجاع کنید.
{{< /hint >}}

