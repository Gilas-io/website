---
title: 'v1/files'
weight: 1005
---



اندپوینت فایل امکان بارگذاری و مدیریت فایل‌ها را به منظور `fine-tuning` کردن یک مدل به شما می‌دهد. اندازه هر فایل می‌تواند تا 25 مگابایت باشد و هر کاربر می‌تواند حداکثر ۱۰ فایل را بر روی سرور بارگذاری کند. در صورت نیاز به بارگذاری فایل‌های بیشتر می‌توانید فایل‌های قدیمی‌تر خود را حذف کنید.

## آپلود فایل  

```shell
POST https://api.gilas.io/v1/files
```
بارگذاری فایل با هدف استفاده برای `fine-tuning` کردن یک مدل.

اندپوینت `Fine-tuning API` تنها از فایل‌های `.jsonl` پشتیبانی می‌کند. ورودی این فایل‌ها باید فرمت‌های مشخصی برای آموزش مدل‌های گفتگو یا تکمیل‌ها داشته باشد. برای اطلاع بیشتر در مورد فرمت داده‌های آموزشی برای تنظیم دقیق یا fine-tuning کردن مدلها به [/v1/fine_tuning/jobs](/apis/fine-tuning) مراجعه کنید.


{{< tabs "create - code" >}}
{{< tab "curl" >}}
```shell
curl https://api.gilas.io/v1/files \
  -H "Authorization: Bearer $GILAS_API_KEY" \
  -F purpose="fine-tune" \
  -F file="@mydata.jsonl"

```
{{< /tab >}}
{{< tab "python" >}}
```python
from openai import OpenAI
client = OpenAI(
    # This is the default and can be omitted
    api_key=os.environ.get("GILAS_API_KEY"),
    base_url="https://api.gilas.io/v1/"
)

client.files.create(
  file=open("mydata.jsonl", "rb"),
  purpose="fine-tune"
)
```
{{< /tab >}}
{{< /tabs >}}

خروجی:

```json
{
  "id": "file-abc123",
  "object": "file",
  "bytes": 120000,
  "created_at": 1677610602,
  "filename": "mydata.jsonl",
  "purpose": "fine-tune",
}
```

### بدنه درخواست (Request body)

`Required` `file` **`file`**  
فایل مورد نظر برای آپلود (نه نام فایل).
  
`Required` `string` **`purpose`**  
رشته‌ای که هدف فایل آپلود شده را مشخص می‌کند. برای `Fine-tuning` از مقدار "fine-tune" استفاده کنید.

## لیست فایل‌ها  

```shell
GET https://api.gilas.io/v1/files
```

لیستی از فایل‌هایی که به کاربر تعلق دارند را برمی‌گرداند.

{{< tabs "get - code" >}}
{{< tab "curl" >}}
```shell
curl https://api.gilas.io/v1/files \
  -H "Authorization: Bearer $GILAS_API_KEY"
```
{{< /tab >}}
{{< tab "python" >}}
```python
from openai import OpenAI
client = OpenAI(
    # This is the default and can be omitted
    api_key=os.environ.get("GILAS_API_KEY"),
    base_url="https://api.gilas.io/v1/"
)

client.files.list()
```
{{< /tab >}}
{{< /tabs >}}

خروجی

```json
{
  "data": [
    {
      "id": "file-abc123",
      "object": "file",
      "bytes": 175,
      "created_at": 1613677385,
      "filename": "my_dataset.jsonl",
      "purpose": "fine-tuning",
    },
    { ... }
  ],
  "object": "list"
}
```

## دریافت اطلاعات فایل  

```shell
GET https://api.gilas.io/v1/files/{file_id}
```

اطلاعاتی در مورد فایل مشخص شده را برمی‌گرداند.


{{< tabs "get one - code" >}}
{{< tab "curl" >}}
```shell
curl https://api.gilas.io/v1/files/file-abc123 \
  -H "Authorization: Bearer $GILAS_API_KEY"
```
{{< /tab >}}
{{< tab "python" >}}
```python
from openai import OpenAI
client = OpenAI(
    # This is the default and can be omitted
    api_key=os.environ.get("GILAS_API_KEY"),
    base_url="https://api.gilas.io/v1/"
)

client.files.retrieve("file-abc123")
```
{{< /tab >}}
{{< /tabs >}}

خروجی

```json
{
  "id": "file-abc123",
  "object": "file",
  "bytes": 120000,
  "created_at": 1677610602,
  "filename": "mydata.jsonl",
  "purpose": "fine-tune",
}
```

### پارامترهای مسیر (Path parameters)


`Required` `string` **`file_id`**    
شناسه فایل که برای این درخواست مورد استفاده قرار می‌گیرد (الزامی).
  

## حذف فایل  

```shell
DELETE https://api.gilas.io/v1/files/{file_id}
```

فایلی را حذف کنید.


{{< tabs "delete - code" >}}
{{< tab "curl" >}}
```shell
curl https://api.gilas.io/v1/files/file-abc123 \
  -X DELETE \
  -H "Authorization: Bearer $GILAS_API_KEY"
```
{{< /tab >}}
{{< tab "python" >}}
```python
from openai import OpenAI
client = OpenAI(
    # This is the default and can be omitted
    api_key=os.environ.get("GILAS_API_KEY"),
    base_url="https://api.gilas.io/v1/"
)

client.files.delete("file-abc123")
```
{{< /tab >}}
{{< /tabs >}}

خروجی

```json
{
  "id": "file-abc123",
  "object": "file",
  "deleted": true
}
```

### پارامترهای مسیر (Path parameters)

`Required` `string` **`file_id`**    
شناسه فایل که برای این درخواست مورد استفاده قرار می‌گیرد (الزامی).

## دریافت محتوای فایل  

```shell
GET https://api.gilas.io/v1/files/{file_id}/content
```

محتوای فایل مشخص شده را برمی‌گرداند.


{{< tabs "retrieve - code" >}}
{{< tab "curl" >}}
```shell
curl https://api.gilas.io/v1/files/file-abc123/content \
  -H "Authorization: Bearer $GILAS_API_KEY" > file.jsonl
```
{{< /tab >}}
{{< tab "python" >}}
```python
from openai import OpenAI
client = OpenAI(
    # This is the default and can be omitted
    api_key=os.environ.get("GILAS_API_KEY"),
    base_url="https://api.gilas.io/v1/"
)

content = client.files.content("file-abc123")
```
{{< /tab >}}
{{< /tabs >}}

### پارامترهای مسیر (Path parameters)

`Required` `string` **`file_id`**    
شناسه فایل که برای این درخواست مورد استفاده قرار می‌گیرد (الزامی).


{{< hint info >}}
**توجه**  
در نظر داشته باشید که Gilas APIs از لحاظ فنی و نحوه کارکرد و قابلیت‌ها کاملا شبیه OpenAI APIs هستند. به همین منظور پیشنهاد میکنیم که برای آگاهی از نحوه‌ی کارکرد API ها به مستندات [OpenAI API Reference](https://platform.openai.com/docs/api-reference/files/) ارجاع کنید.
{{< /hint >}}