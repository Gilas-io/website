---
title: "استفاده از Redis به عنوان پایگاه داده وکتور"
description: "راهنمای استفاده از Redis به عنوان پایگاه داده وکتور با استفاده از ماژول RediSearch."
tags:
- redis
- vector-database
- embeddings
weight: 
og_image: "/posts/getting-started-with-redis-search/banner.png"
---

{{< postcover src="/posts/getting-started-with-redis-search/banner.png" >}}


# استفاده از Redis به عنوان پایگاه داده وکتور

این پست مقدمه‌ای بر استفاده از Redis به عنوان پایگاه داده وکتور است. Redis یک پایگاه داده مقیاس‌پذیر است که می‌تواند با استفاده از [ماژول RediSearch](https://oss.redislabs.com/redisearch/) به عنوان پایگاه داده وکتور استفاده شود. ماژول RediSearch به شما امکان می‌دهد وکتورها را در Redis ایندکس و جستجو کنید. این نوت‌بوک به شما نشان می‌دهد که چگونه از ماژول RediSearch برای ایندکس و جستجوی وکتورهایی که با استفاده از `Gilas API` ایجاد و در Redis ذخیره شده‌اند، استفاده کنید.


بیشتر برنامه‌نویسان با پیش‌زمینه وب احتمالاً با Redis آشنا هستند. در هسته خود، Redis یک key-value store متن‌باز است که می‌تواند به عنوان کش، پیام‌رسان و پایگاه داده استفاده شود. توسعه‌دهندگان Redis را به دلیل سرعت بالا، اکوسیستم بزرگ کتابخانه‌های کلاینت و استفاده توسط شرکت‌های بزرگ انتخاب می‌کنند.

علاوه بر استفاده‌های سنتی از Redis، Redis همچنین [ماژول‌های Redis](https://redis.io/modules) را ارائه می‌دهد که راهی برای گسترش Redis با انواع داده‌ها و دستورات جدید است. مثال‌هایی از ماژول‌ها شامل [RedisJSON](https://redis.io/docs/stack/json/)، [RedisTimeSeries](https://redis.io/docs/stack/timeseries/)، [RedisBloom](https://redis.io/docs/stack/bloom/) و [RediSearch](https://redis.io/docs/stack/search/) هستند.

## RediSearch چیست؟

[ماژول RediSearch](https://redis.io/modules) امکاناتی از قبیل جستجو، ایندکس‌گذاری ثانویه، جستجوی متن کامل و جستجوی وکتور را برای Redis فراهم می‌کند. برای استفاده از RediSearch، ابتدا باید ایندکس‌هایی بر روی داده‌های Redis خود ثبت کنید. سپس می‌توانید از کلاینت‌های RediSearch برای جستجوی آن داده‌ها استفاده کنید. برای اطلاعات بیشتر در مورد مجموعه ویژگی‌های RediSearch، به [README](./README.md) یا [مستندات RediSearch](https://redis.io/docs/stack/search/) مراجعه کنید.

## روش‌های Deploy کردن Redis

روش‌های مختلفی برای استقرار یا Deploy کردن Redis وجود دارد. برای توسعه لوکال سریع‌ترین روش استفاده از [کانتینر داکر Redis Stack](https://hub.docker.com/r/redis/redis-stack) است که در اینجا از آن استفاده خواهیم کرد. Redis Stack شامل تعدادی از ماژول‌های Redis است که می‌توانند با هم استفاده شوند تا یک پایگاه داده چندمدلی سریع و موتور جستجو ایجاد کنند.

برای یوزکیس‌های پروداکشن، ساده‌ترین راه برای شروع استفاده از سرویس [Redis Cloud](https://redislabs.com/redis-enterprise-cloud/overview/) است.  همچنین می‌توانید Redis را بر روی زیرساخت خود با استفاده از [Redis Enterprise](https://redislabs.com/redis-enterprise/overview/) نصب و راه‌اندازی کنید. Redis Enterprise یک سرویس Redis کاملاً مدیریت شده است که می‌تواند در Kubernetes، بصورت on-prem محل یا cloud مستقر شود.

علاوه بر این، هر ارائه‌دهنده بزرگ ابری ([AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-e6y7ork67pjwg?sr=0-2&ref_=beagle&applicationId=AWSMPContessa)، [Google Marketplace](https://console.cloud.google.com/marketplace/details/redislabs-public/redis-enterprise?pli=1)، یا [Azure Marketplace](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/garantiadata.redis_enterprise_1sp_public_preview?tab=Overview)) Redis Enterprise را  نیز به عنوان یک سرویس مدیریت شده ارائه می‌دهند.


## ران کردن Redis

برای ساده نگه داشتن این مثال، از کانتینر داکر Redis Stack استفاده خواهیم کرد که می‌توانیم به صورت زیر آن را ران کنیم:

```bash
$ docker-compose up -d
```

این همچنین شامل رابط کاربری [RedisInsight](https://redis.com/redis-enterprise/redis-insight/) برای مدیریت پایگاه داده Redis شما است که پس از شروع کانتینر داکر از طریق آدرس [http://localhost:8001](http://localhost:8001) قابل دسترس است.

 در مرحله بعد، کلاینت خود را برای ارتباط با پایگاه داده Redis که تازه ایجاد کرده‌ایم، وارد و ایجاد می‌کنیم.

## نصب پکیج‌ها

پکیج `Redis-Py`یک کلاینت پایتون برای ارتباط با Redis است. ما از این پکیج برای ارتباط با پایگاه داده `Redis-stack` خود استفاده خواهیم کرد.
```python
! pip install redis wget pandas openai
```

## آماده‌سازی کلید Gilas API

برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید. سپس از آن کلید برای ساخت کلاینت `OpenAI` استفاده کنید.

```python
from openai import OpenAI
import os

client = OpenAI(
    api_key=os.environ.get(("GILAS_API_KEY", "<کلید API خود را اینجا بسازید https://dashboard.gilas.io/apiKey>")), 
    base_url="https://api.gilas.io/v1/" # Gilas APIs
)
```

## بارگذاری داده‌ها

در این بخش، داده‌های جاسازی شده‌ای که قبلاً به وکتور تبدیل شده‌اند را بارگذاری خواهیم کرد. از این داده‌ها برای ایجاد یک ایندکس در Redis و سپس جستجوی وکتورهای مشابه استفاده خواهیم کرد.

برای ساخت وکتور از روی داده های متنی پیشنهاد می‌کنیم [پست‌های مربوط به آموزش تولید بردار embeddings](https://gilas.io/tags/embeddings) را مطالعه کنید.


```python
import sys
import numpy as np
import pandas as pd
from typing import List

# استفاده از تابع کمکی در nbutils.py برای دانلود و خواندن داده‌ها
# این باید بین 5-10 دقیقه طول بکشد
if os.getcwd() not in sys.path:
    sys.path.append(os.getcwd())
import nbutils

nbutils.download_wikipedia_data()
data = nbutils.read_wikipedia_data()

data.head()
```

## اتصال به Redis

حالا که پایگاه داده Redis خود را اجرا کرده‌ایم، می‌توانیم با استفاده از کلاینت Redis-py به آن متصل شویم. از آدرس پیش‌فرض برای پایگاه داده Redis که `localhost:6379` است، استفاده خواهیم کرد.

```python
import redis
from redis.commands.search.indexDefinition import (
    IndexDefinition,
    IndexType
)
from redis.commands.search.query import Query
from redis.commands.search.field import (
    TextField,
    VectorField
)

REDIS_HOST =  "localhost"
REDIS_PORT = 6379
REDIS_PASSWORD = ""

# اتصال به Redis
redis_client = redis.Redis(
    host=REDIS_HOST,
    port=REDIS_PORT,
    password=REDIS_PASSWORD
)
redis_client.ping()
```

## ایجاد یک ایندکس جستجو در Redis

نمونه کدهای زیر نشان می‌دهند که چگونه یک ایندکس جستجو در Redis تعریف و ایجاد کنیم. ما:

1. برخی از ثابت‌ها را برای تعریف ایندکس خود مانند متریک فاصله و نام ایندکس تنظیم می‌کنیم.
2. طرح ایندکس را با فیلدهای RediSearch تعریف می‌کنیم.
3. ایندکس را ایجاد می‌کنیم.

```python
VECTOR_DIM = len(data['title_vector'][0]) # طول وکتورها
VECTOR_NUMBER = len(data)                 # تعداد اولیه وکتورها
INDEX_NAME = "embeddings-index"           # نام ایندکس جستجو
PREFIX = "doc"                            # پیشوند برای کلیدها
DISTANCE_METRIC = "COSINE"                # متریک فاصله برای وکتورها (مثلاً COSINE، IP، L2)

# تعریف فیلدهای RediSearch برای هر یک از ستون‌های مجموعه داده
title = TextField(name="title")
url = TextField(name="url")
text = TextField(name="text")
title_embedding = VectorField("title_vector",
    "FLAT", {
        "TYPE": "FLOAT32",
        "DIM": VECTOR_DIM,
        "DISTANCE_METRIC": DISTANCE_METRIC,
        "INITIAL_CAP": VECTOR_NUMBER,
    }
)
text_embedding = VectorField("content_vector",
    "FLAT", {
        "TYPE": "FLOAT32",
        "DIM": VECTOR_DIM,
        "DISTANCE_METRIC": DISTANCE_METRIC,
        "INITIAL_CAP": VECTOR_NUMBER,
    }
)
fields = [title, url, text, title_embedding, text_embedding]

# بررسی وجود ایندکس
try:
    redis_client.ft(INDEX_NAME).info()
    print("ایندکس از قبل وجود دارد")
except:
    # ایجاد ایندکس RediSearch
    redis_client.ft(INDEX_NAME).create_index(
        fields = fields,
        definition = IndexDefinition(prefix=[PREFIX], index_type=IndexType.HASH)
)
```

## بارگذاری اسناد در ایندکس

حالا که یک موتور جستجوی ایندکس داریم، می‌توانیم اسناد را در آن بارگذاری کنیم. از همان اسنادی که در مثال‌های قبلی استفاده کردیم، استفاده خواهیم کرد. در Redis، می‌توان از انواع داده‌های `HASH` یا `JSON` (اگر از RedisJSON علاوه بر RediSearch استفاده می‌کنید) برای ذخیره اسناد استفاده کرد. در این مثال از نوع داده `HASH` استفاده خواهیم کرد. سلول‌های زیر نشان می‌دهند که چگونه اسناد را در ایندکس بارگذاری کنیم.

```python
def index_documents(client: redis.Redis, prefix: str, documents: pd.DataFrame):
    records = documents.to_dict("records")
    for doc in records:
        key = f"{prefix}:{str(doc['id'])}"

        # ایجاد وکتورهای بایتی برای عنوان و محتوا
        title_embedding = np.array(doc["title_vector"], dtype=np.float32).tobytes()
        content_embedding = np.array(doc["content_vector"], dtype=np.float32).tobytes()

        # جایگزینی لیست اعداد اعشاری با وکتورهای بایتی
        doc["title_vector"] = title_embedding
        doc["content_vector"] = content_embedding

        client.hset(key, mapping = doc)
```

```python
index_documents(redis_client, PREFIX, data)
print(f"بارگذاری {redis_client.info()['db0']['keys']} سند در ایندکس جستجوی Redis با نام: {INDEX_NAME}")
```

## جستجوی ساده وکتور با استفاده از Query Embeddings

حالا که یک موتور جستجوی ایندکس و اسناد بارگذاری شده داریم، می‌توانیم جستجوهای خود را اجرا کنیم. در زیر یک تابع ارائه می‌دهیم که یک جستجو را اجرا کرده و نتایج را برمی‌گرداند. با استفاده از این تابع، چند جستجو اجرا می‌کنیم که نشان می‌دهد چگونه می‌توانید از Redis به عنوان پایگاه داده وکتور استفاده کنید.
```python
def search_redis(
    redis_client: redis.Redis,
    user_query: str,
    index_name: str = "embeddings-index",
    vector_field: str = "title_vector",
    return_fields: list = ["title", "url", "text", "vector_score"],
    hybrid_fields = "*",
    k: int = 20,
    print_results: bool = True,
) -> List[dict]:

    # ایجاد وکتور جاسازی شده از پرس و جوی کاربر
    embedded_query = client.embeddings.create(input=user_query,
                                            model="text-embedding-3-small",
                                            )["data"][0]['embedding']

    # آماده‌سازی پرس و جو
    base_query = f'{hybrid_fields}=>[KNN {k} @{vector_field} $vector AS vector_score]'
    query = (
        Query(base_query)
         .return_fields(*return_fields)
         .sort_by("vector_score")
         .paging(0, k)
         .dialect(2)
    )
    params_dict = {"vector": np.array(embedded_query).astype(dtype=np.float32).tobytes()}

    # اجرای جستجوی وکتور
    results = redis_client.ft(index_name).search(query, params_dict)
    if print_results:
        for i, article in enumerate(results.docs):
            score = 1 - float(article.vector_score)
            print(f"{i}. {article.title} (امتیاز: {round(score ,3) })")
    return results.docs
```

```python
results = search_redis(redis_client, 'modern art in Europe', k=10)
```

```
0. Museum of Modern Art (امتیاز: 0.875)
1. Western Europe (امتیاز: 0.868)
2. Renaissance art (امتیاز: 0.864)
3. Pop art (امتیاز: 0.86)
4. Northern Europe (امتیاز: 0.855)
5. Hellenistic art (امتیاز: 0.853)
6. Modernist literature (امتیاز: 0.847)
7. Art film (امتیاز: 0.843)
8. Central Europe (امتیاز: 0.843)
9. European (امتیاز: 0.841)

```

```python
results = search_redis(redis_client, 'Famous battles in Scottish history', vector_field='content_vector', k=10)
```

```
0. Battle of Bannockburn (امتیاز: 0.869)
1. Wars of Scottish Independence (امتیاز: 0.861)
2. 1651 (امتیاز: 0.853)
3. First War of Scottish Independence (امتیاز: 0.85)
4. Robert I of Scotland (امتیاز: 0.846)
5. 841 (امتیاز: 0.844)
6. 1716 (امتیاز: 0.844)
7. 1314 (امتیاز: 0.837)
8. 1263 (امتیاز: 0.836)
9. William Wallace (امتیاز: 0.835)

```

## جستجوهای هیبریدی با Redis

مثال‌های قبلی نشان دادند که چگونه می‌توان جستجوهای وکتور را با RediSearch اجرا کرد. در این بخش، نشان خواهیم داد که چگونه می‌توان جستجوی وکتور را با سایر فیلدهای RediSearch برای جستجوی هیبریدی ترکیب کرد. در مثال زیر، جستجوی وکتور را با جستجوی متن کامل ترکیب خواهیم کرد.

```python
def create_hybrid_field(field_name: str, value: str) -> str:
    return f'@{field_name}:"{value}"'

# search the content vector for articles about famous battles in Scottish history and only include results with Scottish in the title
results = search_redis(redis_client,
                       "Famous battles in Scottish history",
                       vector_field="title_vector",
                       k=5,
                       hybrid_fields=create_hybrid_field("title", "Scottish")
                       )
```

```
0. First War of Scottish Independence (امتیاز: 0.892)
1. Wars of Scottish Independence (امتیاز: 0.889)
2. Second War of Scottish Independence (امتیاز: 0.879)
3. List of Scottish monarchs (امتیاز: 0.873)
4. Scottish Borders (امتیاز: 0.863)

```

```python
# run a hybrid query for articles about Art in the title vector and only include results with the phrase "Leonardo da Vinci" in the text
results = search_redis(redis_client,
                       "Art",
                       vector_field="title_vector",
                       k=5,
                       hybrid_fields=create_hybrid_field("text", "Leonardo da Vinci")
                       )

# find specific mention of Leonardo da Vinci in the text that our full-text-search query returned
mention = [sentence for sentence in results[0].text.split("\n") if "Leonardo da Vinci" in sentence][0]
mention
```

```
0. Art (امتیاز: 1.0)
1. Paint (امتیاز: 0.896)
2. Renaissance art (امتیاز: 0.88)
3. Painting (امتیاز: 0.874)
4. Renaissance (امتیاز: 0.846)

```

## ایندکس HNSW

تا کنون، ما از ایندکس ``FLAT`` یا "brute-force" برای اجرای جستجوهای خود استفاده کرده‌ایم. Redis همچنین از ایندکس ``HNSW`` که یک ایندکس تقریبی سریع است، پشتیبانی می‌کند.ایندکس ``HNSW`` یک ایندکس مبتنی بر گراف است که از یک گراف کوچک قابل پیمایش سلسله‌مراتبی برای ذخیره وکتورها استفاده می‌کند. ایندکس ``HNSW`` یک انتخاب خوب برای مجموعه داده‌های بزرگ است که در آن می‌خواهید جستجوهای تقریبی را اجرا کنید.

ایندکس `HNSW` در بیشتر موارد زمان بیشتری برای ساخت و مصرف حافظه بیشتر نسبت به ``FLAT`` خواهد داشت، اما برای اجرای جستجوها، به ویژه برای مجموعه داده‌های بزرگ، سریع‌تر خواهد بود.

سلول‌های زیر نشان می‌دهند که چگونه یک ایندکس ``HNSW`` ایجاد کرده و با استفاده از همان داده‌های قبلی، جستجوهایی را با آن اجرا کنیم.

```python
# re-define RediSearch vector fields to use HNSW index
title_embedding = VectorField("title_vector",
    "HNSW", {
        "TYPE": "FLOAT32",
        "DIM": VECTOR_DIM,
        "DISTANCE_METRIC": DISTANCE_METRIC,
        "INITIAL_CAP": VECTOR_NUMBER
    }
)
text_embedding = VectorField("content_vector",
    "HNSW", {
        "TYPE": "FLOAT32",
        "DIM": VECTOR_DIM,
        "DISTANCE_METRIC": DISTANCE_METRIC,
        "INITIAL_CAP": VECTOR_NUMBER
    }
)
fields = [title, url, text, title_embedding, text_embedding]
```

```python
import time
# بررسی وجود ایندکس
HNSW_INDEX_NAME = INDEX_NAME + "_HNSW"

try:
    redis_client.ft(HNSW_INDEX_NAME).info()
    print("ایندکس از قبل وجود دارد")
except:
    # ایجاد ایندکس RediSearch
    redis_client.ft(HNSW_INDEX_NAME).create_index(
        fields = fields,
        definition = IndexDefinition(prefix=[PREFIX], index_type=IndexType.HASH)
    )

# since RediSearch creates the index in the background for existing documents, we will wait until
# indexing is complete before running our queries. Although this is not necessary for the first query,
# some queries may take longer to run if the index is not fully built. In general, Redis will perform
# best when adding new documents to existing indices rather than new indices on existing documents.
while redis_client.ft(HNSW_INDEX_NAME).info()["indexing"] == "1":
    time.sleep(5)
```

```python
results = search_redis(redis_client, 'modern art in Europe', index_name=HNSW_INDEX_NAME, k=10)
```

```
0. Western Europe (امتیاز: 0.868)
1. Northern Europe (امتیاز: 0.855)
2. Central Europe (امتیاز: 0.843)
3. European (امتیاز: 0.841)
4. Eastern Europe (امتیاز: 0.839)
5. Europe (امتیاز: 0.839)
6. Western European Union (امتیاز: 0.837)
7. Southern Europe (امتیاز: 0.831)
8. Western civilization (امتیاز: 0.83)
9. Council of Europe (امتیاز: 0.827)

```

```python
# compare the results of the HNSW index to the FLAT index and time both queries
def time_queries(iterations: int = 10):
    print(" ----- ایندکس FLAT ----- ")
    t0 = time.time()
    for i in range(iterations):
        results_flat = search_redis(redis_client, 'modern art in Europe', k=10, print_results=False)
    t0 = (time.time() - t0) / iterations
    results_flat = search_redis(redis_client, 'modern art in Europe', k=10, print_results=True)
    print(f"زمان جستجوی ایندکس FLAT: {round(t0, 3)} ثانیه\n")
    time.sleep(1)
    print(" ----- ایندکس HNSW ------ ")
    t1 = time.time()
    for i in range(iterations):
        results_hnsw = search_redis(redis_client, 'modern art in Europe', index_name=HNSW_INDEX_NAME, k=10, print_results=False)
    t1 = (time.time() - t1) / iterations
    results_hnsw = search_redis(redis_client, 'modern art in Europe', index_name=HNSW_INDEX_NAME, k=10, print_results=True)
    print(f"زمان جستجوی ایندکس HNSW: {round(t1, 3)} ثانیه")
    print(" ------------------------ ")
time_queries()
```

```
----- FLAT Index ----- 
0. Museum of Modern Art (امتیاز: 0.875)
1. Western Europe (امتیاز: 0.867)
2. Renaissance art (امتیاز: 0.864)
3. Pop art (امتیاز: 0.861)
4. Northern Europe (امتیاز: 0.855)
5. Hellenistic art (امتیاز: 0.853)
6. Modernist literature (امتیاز: 0.847)
7. Art film (امتیاز: 0.843)
8. Central Europe (امتیاز: 0.843)
9. Art (امتیاز: 0.842)
زمان جستجوی ایندکس FLAT: 0.263 ثانیه

 ----- HNSW Index ------ 
0. Western Europe (امتیاز: 0.867)
1. Northern Europe (امتیاز: 0.855)
2. Central Europe (امتیاز: 0.843)
3. European (امتیاز: 0.841)
4. Eastern Europe (امتیاز: 0.839)
5. Europe (امتیاز: 0.839)
6. Western European Union (امتیاز: 0.837)
7. Southern Europe (امتیاز: 0.831)
8. Western civilization (امتیاز: 0.83)
9. Council of Europe (امتیاز: 0.827)
زمان جستجوی ایندکس HNSW: 0.129 ثانیه
 ------------------------ 
```

