---
title: "استفاده از MongoDB Atlas به عنوان پایگاه داده وکتور"
description: "در این پست نحوه‌ی استفاده از MongoDB Atla به عنوان پایگاه داده وکتور برای جستجوی معنایی در دیتابیسی از آرشیو فیلم‌ها را نشان می‌دهیم."
tags:
- mongodb
- embeddings
weight: 
og_image: "/posts/semantic_search_using_mongodb_atlas_vector_search/banner.jpg"
---


{{< postcover src="/posts/semantic_search_using_mongodb_atlas_vector_search/banner.jpg" >}}


این نوت‌بوک نحوه ساخت یک اپلیکیشن جستجوی معنایی در آرشیوی از فیلم‌ها را با استفاده از [جستجوی برداری `MongoDB Atlas`](https://www.mongodb.com/products/platform/atlas-vector-search) نشان می‌دهد.

## مرحله 1: تنظیم محیط

دو پیش‌نیاز برای ساخت این اپلیکیشن وجود دارد:

1. **کلاستر `MongoDB Atlas`**: برای ایجاد یک کلاستر رایگان `MongoDB Atlas`، ابتدا باید یک حساب کاربری `MongoDB Atlas` ایجاد کنید. برای این کار به [وب‌سایت `MongoDB Atlas`](https://www.mongodb.com/atlas/database) مراجعه کرده و روی "Register" کلیک کنید. به داشبورد [MongoDB Atlas](https://account.mongodb.com/account/login) بروید و کلاستر خود را تنظیم کنید. برای استفاده از اپراتور `$vectorSearch` باید `MongoDB Atlas` نسخه 6.0.11 یا بالاتر را اجرا کنید. لطفا پس از ساخت کلاستر خود اطلاعات ضروری مثل آدرس آی‌پی٬ نام کاربری و رمز عبور را در جایی ذخیره کنید. اگر به کمک بیشتری برای شروع نیاز دارید، به [آموزش `MongoDB Atlas`](https://www.mongodb.com/basics/mongodb-atlas-tutorial) مراجعه کنید.

2. **کلید `Gilas API`**:  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.


```python
import getpass

MONGODB_ATLAS_CLUSTER_URI = getpass.getpass("MongoDB Atlas Cluster URI:")
GILAS_API_KEY = getpass.getpass("Gilas API Key:")

```

{{< ltr >}}
```
MongoDB Atlas Cluster URI:··········
Gilas API Key:··········
```
{{< /ltr >}}

توجه: پس از اجرای مرحله بالا، از شما خواسته می‌شود که اطلاعات کاربری را وارد کنید.

برای این آموزش، از [مجموعه داده نمونه `MongoDB`](https://www.mongodb.com/docs/atlas/sample-data/) استفاده خواهیم کرد. ما از پایگاه داده "sample_mflix" استفاده خواهیم کرد که شامل مجموعه‌ای از فیلم‌هاست که هر سند آن شامل فیلدهایی مانند عنوان، خلاصه، ژانرها، بازیگران، کارگردانان و غیره است.

```python
import openai
import pymongo

client = pymongo.MongoClient(MONGODB_ATLAS_CLUSTER_URI)
db = client.sample_mflix
collection = db.movies

openai.api_key = OPENAI_API_KEY

ATLAS_VECTOR_SEARCH_INDEX_NAME = "default"
EMBEDDING_FIELD_NAME = "embedding_openai_nov19_23"
```

## مرحله 2: تنظیم تابع تولید `embeddings`

برای ساخت بردار امبدینگ از یک متن تابع زیر را می‌نویسیم.

```python
from openai import OpenAI
import os

client = OpenAI(
    api_key=os.environ.get(("GILAS_API_KEY", "<کلید API خود را اینجا بسازید https://dashboard.gilas.io/apiKey>")), 
    base_url="https://api.gilas.io/v1/" # Gilas APIs
)

model = "text-embedding-3-small"
def generate_embedding(text: str) -> list[float]:
    return client.embeddings.create(input=user_query,
                                            model="text-embedding-3-small",
                                            )["data"][0]['embedding']
```

## مرحله 3: ایجاد و ذخیره `embeddings`

هر سند در مجموعه داده نمونه `sample_mflix.movies` مربوط به یک فیلم است؛ ما عملیاتی را اجرا خواهیم کرد تا یک `vector embedding` برای داده‌های موجود در فیلد "plot" ایجاد کرده و آن را در پایگاه داده ذخیره کنیم.

```python
from pymongo import ReplaceOne

### به‌روزرسانی مجموعه با `embeddings`
requests = []

for doc in collection.find({'plot':{"$exists": True}}).limit(500):
  doc[EMBEDDING_FIELD_NAME] = generate_embedding(doc['plot'])
  requests.append(ReplaceOne({'_id': doc['_id']}, doc))

collection.bulk_write(requests)
```

پس از اجرای موارد بالا، اسناد در مجموعه "movies" شامل یک فیلد اضافی به نام "embedding" خواهند بود، همانطور که توسط متغیر `EMBEDDDING_FIELD_NAME` تعریف شده است، علاوه بر فیلدهای موجود مانند عنوان، خلاصه، ژانرها، بازیگران، کارگردانان و غیره.

توجه: ما این کار را به 500 سند محدود کرده‌ایم تا زمان صرفه‌جویی شود. اگر می‌خواهید این کار را بر روی کل مجموعه داده 23,000+ سند در پایگاه داده `sample_mflix` انجام دهید، کمی زمان خواهد برد. به طور جایگزین، می‌توانید از [مجموعه `sample_mflix.embedded_movies`](https://www.mongodb.com/docs/atlas/sample-data/sample-mflix/##sample_mflix.embedded_movies) استفاده کنید که شامل فیلد `plot_embedding` از پیش پر شده است که `embeddings` ایجاد شده با استفاده از مدل `text-embedding-3-small` `OpenAI` را شامل می‌شود و می‌توانید از آن با ویژگی جستجوی برداری `Atlas Search` استفاده کنید.

## مرحله 4: ایجاد یک ایندکس جستجوی برداری

ما یک ایندکس جستجوی برداری `Atlas` بر روی این مجموعه ایجاد خواهیم کرد که به ما امکان انجام جستجوی `Approximate KNN` را می‌دهد که جستجوی معنایی را قدرت می‌بخشد.
ما دو روش برای ایجاد این ایندکس را نشان خواهیم داد - رابط کاربری `Atlas` و استفاده از درایور پایتون `MongoDB`.

در صورت علاقه می‌توانید مستندات مربوط به  [ایجاد یک ایندکس جستجوی برداری](https://www.mongodb.com/docs/atlas/atlas-search/field-types/knn-vector/)
را مطالعه کنید.

اکنون به [رابط کاربری `Atlas`](cloud.mongodb.com) بروید و یک ایندکس جستجوی برداری `Atlas` با استفاده از مراحل توضیح داده شده [اینجا](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-tutorial/#create-the-atlas-vector-search-index) ایجاد کنید. فیلد 'dimensions' با مقدار 1536، مربوط به `text-embedding-ada002` `openAI` است.

از تعریف زیر در ویرایشگر `JSON` در رابط کاربری `Atlas` استفاده کنید.

{{< ltr >}}
```
{
  "mappings": {
    "dynamic": true,
    "fields": {
      "embedding": {
        "dimensions": 1536,
        "similarity": "dotProduct",
        "type": "knnVector"
      }
    }
  }
}
```
{{< /ltr >}}

(اختیاری) به طور جایگزین، می‌توانیم از [درایور `pymongo` برای ایجاد این ایندکس‌های جستجوی برداری به صورت برنامه‌ریزی شده](https://pymongo.readthedocs.io/en/stable/api/pymongo/collection.html#pymongo.collection.Collection.create_search_index) استفاده کنیم.
دستور پایتون داده شده در سلول زیر ایندکس را ایجاد خواهد کرد (این فقط برای نسخه‌های اخیر درایور پایتون `MongoDB` و کلاستر `Atlas` نسخه 7.0+ کار می‌کند).

```python
collection.create_search_index(
    {"definition":
        {"mappings": {"dynamic": True, "fields": {
            EMBEDDING_FIELD_NAME : {
                "dimensions": 1536,
                "similarity": "dotProduct",
                "type": "knnVector"
                }}}},
     "name": ATLAS_VECTOR_SEARCH_INDEX_NAME
    }
)
```

## مرحله 5: جستجوی داده‌های خود

نتایج جستجو در اینجا فیلم‌هایی را پیدا می‌کند که دارای خلاصه‌های معنایی مشابه با متن موجود در رشته جستجو هستند. این روش جستجو تفاوت های بسیار زیادی با جستجوی روش جستجوی کلمات کلیدی دارد و به کاربر این اجازه را می‌دهد که به استفاده از زبان طبیعی و تعریف آنجه در ذهن دارد در میان داده‌ها جستجو کند.

(اختیاری) [مستندات: اجرای جستجوی برداری](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/)

```python
def query_results(query, k):
  results = collection.aggregate([
    {
        '$vectorSearch': {
            "index": ATLAS_VECTOR_SEARCH_INDEX_NAME,
            "path": EMBEDDING_FIELD_NAME,
            "queryVector": generate_embedding(query),
            "numCandidates": 50,
            "limit": 5,
        }
    }
    ])
  return results
```

حال می‌توان با نوشتن توضیحاتی در مورد یک فیلم آن را در آرشیو فیلم‌ها جستجو کرد.

```python
query="imaginary characters from outerspace at war with earthlings"
movies = query_results(query, 5)

for movie in movies:
    print(f'Movie Name: {movie["title"]},\nMovie Plot: {movie["plot"]}\n')
```
