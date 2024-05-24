---
title: "دسته‌بندی با استفاده از embeddings"
description: "این نوت‌بوک مثالی از دسته‌بندی متن با استفاده از `embeddings` را به نمایش می‌دهد. در این مثال، امتیاز یک بررسی غذایی (از ۱ تا ۵) بر اساس `embedding` متن بررسی و پیش‌بینی می‌شود."
tags:
- embeddings
- classification
weight: 
og_image: "/posts/classification_using_embeddings/banner.png"
---

{{< postcover src="/posts/classification_using_embeddings/banner.png" >}}

## دسته‌بندی با استفاده از embeddings

راه‌های زیادی برای دسته‌بندی متن وجود دارد. این نوت‌بوک مثالی از دسته‌بندی متن با استفاده از `embeddings` را نمایش می‌دهد.

در این نوت‌بوک امتیاز بررسی غذایی (از ۱ تا ۵) بر اساس `embedding` متن بررسی و پیش‌بینی می‌شود. ما دیتاست را به مجموعه‌های آموزشی و آزمایشی تقسیم می‌کنیم تا بتوانیم عملکرد مدل را بر روی داده‌های دیده نشده به طور واقعی ارزیابی کنیم.

## جمع آوری داده ها

مجموعه داده‌ای که در این مثال استفاده شده است، نظرات کاربران در مورد غذاهای مختلف در آمازون می‌باشد. این مجموعه داده شامل 568,454 نظر در مورد غذاهای مختلف است که تا اکتبر 2012 توسط کاربران آمازون ثبت شده‌اند. ما از یک زیرمجموعه از این داده‌ها که شامل 1,000 نظر جدیدتر می‌باشد برای مقاصد آموزشی استفاده خواهیم کرد. این نظرات به زبان انگلیسی نوشته شده‌اند و به طور کلی یا مثبت هستند یا منفی. هر نظر شامل `ProductId`، `UserId`، امتیاز (`Score`)، عنوان نظر (`Summary`) و متن نظر (`Text`) می‌باشد.

ما عنوان نظر و متن نظر را به یک متن ترکیبی واحد تبدیل خواهیم کرد. مدل این متن ترکیبی را `encode` کرده و یک وکتور تکی تولید خواهد کرد.

برای اجرای این نوت‌بوک، نیاز به نصب پکیج‌های زیر دارید: `pandas`، `openai`، `transformers`، `plotly`، `matplotlib`، `scikit-learn`، `torch` (وابسته به `transformers`)، `torchvision`، و `scipy`.

```python
import pandas as pd
import tiktoken

# load & inspect dataset
input_datapath = "data/fine_food_reviews_1k.csv"  # to save space, we provide a pre-filtered dataset
df = pd.read_csv(input_datapath, index_col=0)
df = df[["Time", "ProductId", "UserId", "Score", "Summary", "Text"]]
df = df.dropna()
df["combined"] = (
    "Title: " + df.Summary.str.strip() + "; Content: " + df.Text.str.strip()
)
df.head(2)
```

نمایش:

{{< scrollbox >}}

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Time</th>
      <th>ProductId</th>
      <th>UserId</th>
      <th>Score</th>
      <th>Summary</th>
      <th>Text</th>
      <th>combined</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1351123200</td>
      <td>B003XPF9BO</td>
      <td>A3R7JR3FMEBXQB</td>
      <td>5</td>
      <td>where does one  start...and stop... with a tre...</td>
      <td>Wanted to save some to bring to my Chicago fam...</td>
      <td>Title: where does one  start...and stop... wit...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1351123200</td>
      <td>B003JK537S</td>
      <td>A3JBPC3WFUT5ZP</td>
      <td>1</td>
      <td>Arrived in pieces</td>
      <td>Not pleased at all. When I opened the box, mos...</td>
      <td>Title: Arrived in pieces; Content: Not pleased...</td>
    </tr>
  </tbody>
</table>

{{< /scrollbox >}}


```python

embedding_model = "text-embedding-3-small"
max_tokens = 8000  # the maximum for text-embedding-3-small is 8191
embedding_encoding = "cl100k_base"

# subsample to 1k most recent reviews and remove samples that are too long
top_n = 1000
df = df.sort_values("Time").tail(top_n * 2)  # first cut to first 2k entries, assuming less than half will be filtered out
df.drop("Time", axis=1, inplace=True)

encoding = tiktoken.get_encoding(embedding_encoding)

# omit reviews that are too long to embed
df["n_tokens"] = df.combined.apply(lambda x: len(encoding.encode(x)))
df = df[df.n_tokens <= max_tokens].tail(top_n)
```

حال از Gilas API برای تولید امبدینگ ها استفاده می‌کنیم.


{{< hint info >}}
**برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.**
{{< /hint >}} 

```python

from openai import OpenAI # for calling the OpenAI API
import os

client = OpenAI(
    api_key=os.environ.get(("GILAS_API_KEY", "<کلید API خود را اینجا بسازید https://dashboard.gilas.io/apiKey>")), 
    base_url="https://api.gilas.io/v1/" # Gilas APIs
)

def get_embedding(query)
    query_embedding_response = client.embeddings.create(
        model=embedding_model,
        input=query,
    )
    return query_embedding_response.data[0].embedding


# This may take a few minutes
df["embedding"] = df.combined.apply(lambda x: get_embedding(x))
df.to_csv("data/fine_food_reviews_with_embeddings_1k.csv")
```

حال که بردار امبدینگ‌ها را برای تمام نظرات تولید کردیم نگاهی به نحوه استفاده از آنها می‌کنیم.

```python
import pandas as pd
import numpy as np
from ast import literal_eval

from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import classification_report, accuracy_score

datafile_path = "data/fine_food_reviews_with_embeddings_1k.csv"

df = pd.read_csv(datafile_path)
df["embedding"] = df.embedding.apply(literal_eval).apply(np.array)  # convert string to array

# split data into train and test
X_train, X_test, y_train, y_test = train_test_split(
    list(df.embedding.values), df.Score, test_size=0.2, random_state=42
)

# train random forest classifier
clf = RandomForestClassifier(n_estimators=100)
clf.fit(X_train, y_train)
preds = clf.predict(X_test)
probas = clf.predict_proba(X_test)

report = classification_report(y_test, preds)
print(report)

```

{{< ltr >}}
```
              precision    recall  f1-score   support

           1       0.90      0.45      0.60        20
           2       1.00      0.38      0.55         8
           3       1.00      0.18      0.31        11
           4       0.88      0.26      0.40        27
           5       0.76      1.00      0.86       134

    accuracy                           0.78       200
   macro avg       0.91      0.45      0.54       200
weighted avg       0.81      0.78      0.73       200
```
{{< /ltr >}}

می‌توانیم ببینیم که مدل به خوبی توانسته است بین دسته‌ها تمایز قائل شود. نظرات ۵ ستاره بهترین عملکرد را نشان می‌دهند و این امر تعجب‌آور نیست، زیرا این نظرات در دیتاست بیشترین تعداد را دارند.
```python
from utils.embeddings_utils import plot_multiclass_precision_recall

plot_multiclass_precision_recall(probas, y_test, [1, 2, 3, 4, 5], clf)
```

{{< ltr >}}
```
RandomForestClassifier() - Average precision score over all classes: 0.90
```
{{< /ltr >}}

{{< img image="/posts/classification_using_embeddings/random_forest-classifier.png" alt="random forest classifier" >}}

تعجب‌آور نیست که پیش‌بینی نظرات ۵ ستاره و ۱ ستاره آسان‌تر است. شاید با داده‌های بیشتر، تفاوت‌های بین ۲ تا ۴ ستاره بهتر پیش‌بینی شود.
