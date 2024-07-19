---
title: "ساخت چت‌باتی برای تعامل با Amazon S3"
description: "این کد نحوه تعامل با توابع `ChatGPT` برای انجام کارهای مرتبط با `Amazon S3 buckets` را نشان می‌دهد. این notebook شامل عملکردهای کلیدی `S3 bucket` مانند اجرای دستورات ساده برای لیست کردن٬ جستجوی یک فایل خاص در تمامی `buckets`، آپلود یک فایل به یک `bucket`، و دانلود یک فایل از یک `bucket` است. `Chat API` این قابلیت را دارد که دستورات کاربر را درک کند، پاسخ‌های زبان طبیعی تولید کند و  توابع مناسب را بر اساس ورودی کاربر انتخاب ‌کند.
"
tags:
- function-call
- chatbot
weight: 2005
og_image: "/posts/how_to_automate_s3_storage_with_functions/banner.jpg" 
---

{{< postcover src="/posts/how_to_automate_s3_storage_with_functions/banner.jpg" >}}

این کد نحوه تعامل با توابع `ChatGPT` برای انجام کارهای مرتبط با `Amazon S3 buckets` را نشان می‌دهد. این notebook شامل عملکردهای کلیدی `S3 bucket` مانند اجرای دستورات ساده برای لیست کردن٬ جستجوی یک فایل خاص در تمامی `buckets`، آپلود یک فایل به یک `bucket`، و دانلود یک فایل از یک `bucket` است. `Chat API` این قابلیت را دارد که دستورات کاربر را درک کند، پاسخ‌های زبان طبیعی تولید کند و  توابع مناسب را بر اساس ورودی کاربر انتخاب ‌کند.


{{< hint info >}}
**برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.**
{{< /hint >}} 


### پیش نیازها:

برای اجرای این notebook، یک کلید دسترسی `AWS` با دسترسی نوشتن در `S3 bucket` ایجاد کنید و آن‌ها را در یک env فایل همراه با کلید `Gilas API` ذخیره کنید. قالب فایل `.env`:

```bash
AWS_ACCESS_KEY_ID=<your-key>
AWS_SECRET_ACCESS_KEY=<your-key>
GILAS_API_KEY=<your-key>
```

پکیج های لازم را نصب کنید.

```bash
! pip install openai
! pip install boto3
! pip install tenacity
! pip install python-dotenv
```

```python
from openai import OpenAI
import json
import boto3
import os
import datetime
from urllib.request import urlretrieve

# load environment variables
from dotenv import load_dotenv
load_dotenv() 

# Create openai client
client = OpenAI(
   api_key=os.environ.get(("GILAS_API_KEY", "<کلید API خود را اینجا بسازید https://dashboard.gilas.io/apiKey>")), 
   base_url="https://api.gilas.io/v1/" # Gilas APIs
)

GPT_MODEL = "gpt-4o-mini"

# Optional - if you had issues loading the environment file, you can set the AWS values using the below code
# os.environ['AWS_ACCESS_KEY_ID'] = ''
# os.environ['AWS_SECRET_ACCESS_KEY'] = ''

# Create S3 client
s3_client = boto3.client('s3')
```

### توابع:

برای اتصال سوالات یا دستورات کاربر به تابع مناسب، ما باید جزئیات تابع مورد نیاز و پارامترهای مورد انتظار را به `ChatGPT` ارائه دهیم.

```python
# Functions dict to pass S3 operations details for the GPT model
functions = [
    {   
        "type": "function",
        "function":{
            "name": "list_buckets",
            "description": "List all available S3 buckets",
            "parameters": {
                "type": "object",
                "properties": {}
            }
        }
    },
    {
        "type": "function",
        "function":{
            "name": "list_objects",
            "description": "List the objects or files inside a given S3 bucket",
            "parameters": {
                "type": "object",
                "properties": {
                    "bucket": {"type": "string", "description": "The name of the S3 bucket"},
                    "prefix": {"type": "string", "description": "The folder path in the S3 bucket"},
                },
                "required": ["bucket"],
            },
        }
    },
    {   
        "type": "function",
        "function":{
            "name": "download_file",
            "description": "Download a specific file from an S3 bucket to a local distribution folder.",
            "parameters": {
                "type": "object",
                "properties": {
                    "bucket": {"type": "string", "description": "The name of the S3 bucket"},
                    "key": {"type": "string", "description": "The path to the file inside the bucket"},
                    "directory": {"type": "string", "description": "The local destination directory to download the file, should be specificed by the user."},
                },
                "required": ["bucket", "key", "directory"],
            }
        }
    },
    {
        "type": "function",
        "function":{
            "name": "upload_file",
            "description": "Upload a file to an S3 bucket",
            "parameters": {
                "type": "object",
                "properties": {
                    "source": {"type": "string", "description": "The local source path or remote URL"},
                    "bucket": {"type": "string", "description": "The name of the S3 bucket"},
                    "key": {"type": "string", "description": "The path to the file inside the bucket"},
                    "is_remote_url": {"type": "boolean", "description": "Is the provided source a URL (True) or local path (False)"},
                },
                "required": ["source", "bucket", "key", "is_remote_url"],
            }
        }
    },
    {
        "type": "function",
        "function":{
            "name": "search_s3_objects",
            "description": "Search for a specific file name inside an S3 bucket",
            "parameters": {
                "type": "object",
                "properties": {
                    "search_name": {"type": "string", "description": "The name of the file you want to search for"},
                    "bucket": {"type": "string", "description": "The name of the S3 bucket"},
                    "prefix": {"type": "string", "description": "The folder path in the S3 bucket"},
                    "exact_match": {"type": "boolean", "description": "Set exact_match to True if the search should match the exact file name. Set exact_match to False to compare part of the file name string (the file contains)"}
                },
                "required": ["search_name"],
            },
        }
    }
]
```

### توابع کمکی:

حال توابع کمکی لازم برای تعامل با سرویس `S3` مانند لیست کردن `buckets`،  فهرست‌بندی اشیاء، دانلود و آپلود فایل‌ها، و جستجوی فایل‌های خاص را می‌نویسیم.

```python
def datetime_converter(obj):
    if isinstance(obj, datetime.datetime):
        return obj.isoformat()
    raise TypeError(f"Object of type {obj.__class__.__name__} is not JSON serializable")

def list_buckets():
    response = s3_client.list_buckets()
    return json.dumps(response['Buckets'], default=datetime_converter)

def list_objects(bucket, prefix=''):
    response = s3_client.list_objects_v2(Bucket=bucket, Prefix=prefix)
    return json.dumps(response.get('Contents', []), default=datetime_converter)

def download_file(bucket, key, directory):
    
    filename = os.path.basename(key)
    
    # Resolve destination to the correct file path
    destination = os.path.join(directory, filename)
    
    s3_client.download_file(bucket, key, destination)
    return json.dumps({"status": "success", "bucket": bucket, "key": key, "destination": destination})

def upload_file(source, bucket, key, is_remote_url=False):
    if is_remote_url:
        file_name = os.path.basename(source)
        urlretrieve(source, file_name)
        source = file_name
       
    s3_client.upload_file(source, bucket, key)
    return json.dumps({"status": "success", "source": source, "bucket": bucket, "key": key})

def search_s3_objects(search_name, bucket=None, prefix='', exact_match=True):
    search_name = search_name.lower()
    
    if bucket is None:
        buckets_response = json.loads(list_buckets())
        buckets = [bucket_info["Name"] for bucket_info in buckets_response]
    else:
        buckets = [bucket]

    results = []

    for bucket_name in buckets:
        objects_response = json.loads(list_objects(bucket_name, prefix))
        if exact_match:
            bucket_results = [obj for obj in objects_response if search_name == obj['Key'].lower()]
        else:
            bucket_results = [obj for obj in objects_response if search_name in obj['Key'].lower()]

        if bucket_results:
            results.extend([{"Bucket": bucket_name, "Object": obj} for obj in bucket_results])

    return json.dumps(results)
```

ما نام توابع را به همراه تابع مربوطه برای اجرا بر اساس پاسخ‌های `ChatGPT` در یک `dict` نگه‌داری می‌کنیم.

```python
available_functions = {
    "list_buckets": list_buckets,
    "list_objects": list_objects,
    "download_file": download_file,
    "upload_file": upload_file,
    "search_s3_objects": search_s3_objects
}
```

### توابع ChatGPT:

در زیر یک تابع ساده برای ارتباط با `ChatGPT` را مشاهده می‌کنید.

```python
def chat_completion_request(messages, functions=None, function_call='auto', 
                            model_name=GPT_MODEL):
    
    if functions is not None:
        return client.chat.completions.create(
            model=model_name,
            messages=messages,
            tools=functions,
            tool_choice=function_call)
    else:
        return client.chat.completions.create(
            model=model_name,
            messages=messages)
```

### مدیریت مکالمه با چت‌بات:

در اینجا یک تابع اصلی برای چت‌بات ایجاد می‌کنیم که ورودی کاربر را دریافت می‌کند، آن را به `Chat API` ارسال کرده و پاسخ مدل را دریافت می‌کند، تابعی را که مدل انتخاب کرده است فراخوانی کرده و پاسخ نهایی را به کاربر بازمی‌گرداند.

```python
def run_conversation(user_input, topic="S3 bucket functions.", is_log=False):

    system_message=f"Don't make assumptions about what values to plug into functions. Ask for clarification if a user request is ambiguous. If the user ask question not related to {topic} response your scope is {topic} only."
    
    messages = [{"role": "system", "content": system_message},
                {"role": "user", "content": user_input}]
    
    # Call the model to get a response
    response = chat_completion_request(messages

, functions=functions)
    response_message = response.choices[0].message
    
    if is_log:
        print(response.choices)
    
    # check if GPT wanted to call a function
    if response_message.tool_calls:
        function_name = response_message.tool_calls[0].function.name
        function_args = json.loads(response_message.tool_calls[0].function.arguments)
        
        # Call the function
        function_response = available_functions[function_name](**function_args)
        
        # Add the response to the conversation
        messages.append(response_message)
        messages.append({
            "role": "tool",
            "content": function_response,
            "tool_call_id": response_message.tool_calls[0].id,
        })
        
        # Call the model again to summarize the results
        second_response = chat_completion_request(messages)
        final_message = second_response.choices[0].message.content
    else:
        final_message = response_message.content

    return final_message
```

### تست:

قبل از تست چت‌بات ابتدا مطمئن شوید که مقادیر  `<file_name>`، `<bucket_name>` و `<directory_path>` را مقادیر درست جایگزین کنید.

#### لیست و جستجو:

با لیست کردن تمام `buckets` موجود شروع کنیم.

```python
print(run_conversation('لطفا همه S3 Bucket های را لیست کن'))
```

می‌توانید از بات بخواهید تا یک نام فایل خاص را در همه `buckets` یا یک `bucket` خاص جستجو کند.

```python
search_file = '<file_name>'
print(run_conversation(f'دنبال فایلی با نام {search_file} در تمام سطل ها بگرد'))

search_word = '<file_name_part>'
bucket_name = '<bucket_name>'
print(run_conversation(f'دنبال فایلی که اسمش شامل {search_word} در باکت {bucket_name} بگرد'))
```

مدل باید از کاربر در صورت وجود ابهام در مقادیر پارامترها توضیح بخواهد.

```python
print(run_conversation('یک فایل را جستجو کن'))

# Output:
# مطمئناً، برای اینکه بتوانم به شما کمک کنم آنچه را که می‌خواهید پیدا کنید، لطفاً نام فایل و نام سطل S3 را ارائه دهید. همچنین، آیا جستجو باید دقیقاً با نام فایل مطابقت داشته باشد یا باید به مطابقت‌های جزئی نیز توجه کند؟
```

#### اعتبارسنجی:

ما مدل را طوری آموزش داده‌ایم که درخواست‌های نامربوط را رد کند. حال می خواهیم آن را آزمایش کنیم و ببینیم چگونه در عمل کار می‌کند.

```python
# the model should not answer details not related to the scope
print(run_conversation('هوا امروز چطوره؟'))

# Output:
# پوزش می‌خواهم برای سوءتفاهم، اما من فقط می‌توانم در زمینه عملکردهای سطل S3 کمک کنم. لطفاً یک سوال مرتبط با عملکردهای سطل S3 بپرسید؟
```

توابع ارائه شده محدود به فقط بازیابی اطلاعات نیستند. آن‌ها همچنین می‌توانند به کاربر در آپلود یا دانلود فایل‌ها نیز کمک کنند.

#### دانلود یک فایل:

```python
search_file = '<file_name>'
bucket_name = '<bucket_name>'
local_directory = '<directory_path>'
print(run_conversation(f'فایل {search_file} را از {bucket_name} به یک {local_directory} دیکشنری دانلود کن'))
```

#### آپلود یک فایل:

```python
local_file = '<file_name>'
bucket_name = '<bucket_name>'
print(run_conversation(f'فایل {local_file} را به سبد {bucket_name} آپلود کن'))
```

چت‌بات ها در آینده‌ی بسیار نزدیک تعامل ما با دنیای اطراف را به کلی تغییر می‌دهند. امیدواریم که این آموزش‌ها بتواند به شما برای پرورش ایده‌های نو و پیاده‌سازی آنها کمک کند.
