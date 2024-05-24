---
title: "خوشه‌بندی K-means در پایتون با استفاده از Gilas API"
description: "استفاده از الگوریتم K-means برای خوشه‌بندی داده‌ها و نام‌گذاری خوشه‌ها با استفاده از GPT-4"
tags:
- embeddings
- clustering
weight: 
og_image: "/posts/clustering/banner.png"
---

{{< postcover src="/posts/clustering/banner.png" >}}


## خوشه‌بندی K-means در پایتون با استفاده از Gilas API

ما از یک الگوریتم ساده `k-means` برای نشان دادن چگونگی انجام خوشه‌بندی استفاده می‌کنیم. خوشه‌بندی می‌تواند به کشف گروه‌های ارزشمند و پنهان در داده‌ها کمک کند.


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
{{< ltr >}}
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
{{< /ltr >}}


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

داده های نهایی به شکل زیر می‌باشند:

```python
# imports
import numpy as np
import pandas as pd
from ast import literal_eval

# load data
datafile_path = "./data/fine_food_reviews_with_embeddings_1k.csv"

df = pd.read_csv(datafile_path)
df["embedding"] = df.embedding.apply(literal_eval).apply(np.array)  # convert string to numpy array
matrix = np.vstack(df.embedding.values)
matrix.shape

```
خروجی

```
(1000, 1536)
```

### 1. یافتن خوشه‌ها با استفاده از K-means

ما ساده‌ترین استفاده از `K-means` را نشان می‌دهیم. شما می‌توانید تعداد خوشه‌هایی که بهترین تناسب را با یوزکیس شما دارند انتخاب کنید.
```python
from sklearn.cluster import KMeans

n_clusters = 4

kmeans = KMeans(n_clusters=n_clusters, init="k-means++", random_state=42)
kmeans.fit(matrix)
labels = kmeans.labels_
df["Cluster"] = labels

df.groupby("Cluster").Score.mean().sort_values()

```

{{< ltr >}}
```
Cluster
0    4.105691
1    4.191176
2    4.215613
3    4.306590
Name: Score, dtype: float64
```
{{< /ltr >}}

```python
from sklearn.manifold import TSNE
import matplotlib
import matplotlib.pyplot as plt

tsne = TSNE(n_components=2, perplexity=15, random_state=42, init="random", learning_rate=200)
vis_dims2 = tsne.fit_transform(matrix)

x = [x for x, y in vis_dims2]
y = [y for x, y in vis_dims2]

for category, color in enumerate(["purple", "green", "red", "blue"]):
    xs = np.array(x)[df.Cluster == category]
    ys = np.array(y)[df.Cluster == category]
    plt.scatter(xs, ys, color=color, alpha=0.3)

    avg_x = xs.mean()
    avg_y = ys.mean()

    plt.scatter(avg_x, avg_y, marker="x", color=color, s=100)
plt.title("Clusters identified visualized in language 2d using t-SNE")

```

{{< ltr >}}
```
Text(0.5, 1.0, 'Clusters identified visualized in language 2d using t-SNE')
```
{{< /ltr >}}

{{< img image="/posts/clustering/cluster.png" alt="cluster" >}}

بصری‌سازی خوشه‌ها در یک پروجکشن دو بعدی. در این اجرا، خوشه سبز (#1) به نظر می‌رسد که تفاوت زیادی با دیگران دارد. بیایید چند نمونه از هر خوشه را ببینیم.

### 2. نمونه‌های متنی در خوشه‌ها و نام‌گذاری خوشه‌ها

بیایید نمونه‌های تصادفی از هر خوشه را نشان دهیم. ما از `gpt-4` برای نام‌گذاری خوشه‌ها بر اساس نمونه تصادفی ۵ بررسی از آن خوشه استفاده خواهیم کرد.

```python

# Reading a review which belong to each group.
rev_per_cluster = 5

for i in range(n_clusters):
    print(f"Cluster {i} Theme:", end=" ")

    reviews = "\n".join(
        df[df.Cluster == i]
        .combined.str.replace("Title: ", "")
        .str.replace("\n\nContent: ", ":  ")
        .sample(rev_per_cluster, random_state=42)
        .values
    )

    messages = [
        {"role": "user", "content": f'What do the following customer reviews have in common?\n\nCustomer reviews:\n"""\n{reviews}\n"""\n\nTheme:'}
    ]

    response = client.chat.completions.create(
        model="gpt-4",
        messages=messages,
        temperature=0,
        max_tokens=64,
        top_p=1,
        frequency_penalty=0,
        presence_penalty=0)
    print(response.choices[0].message.content.replace("\n", ""))

    sample_cluster_rows = df[df.Cluster == i].sample(rev_per_cluster, random_state=42)
    for j in range(rev_per_cluster):
        print(sample_cluster_rows.Score.values[j], end=", ")
        print(sample_cluster_rows.Summary.values[j], end=":   ")
        print(sample_cluster_rows.Text.str[:70].values[j])

    print("-" * 100)

```

{{< ltr >}}
```
Cluster 0 Theme: The theme of these customer reviews is food products purchased on Amazon.
5, Loved these gluten free healthy bars, saved $$ ordering on Amazon:   These Kind Bars are so good and healthy & gluten free.  My daughter ca
1, Should advertise coconut as an ingredient more prominently:   First, these should be called Mac - Coconut bars, as Coconut is the #2
5, very good!!:   just like the runts<br />great flavor, def worth getting<br />I even o
5, Excellent product:   After scouring every store in town for orange peels and not finding an
5, delicious:   Gummi Frogs have been my favourite candy that I have ever tried. of co
----------------------------------------------------------------------------------------------------
Cluster 1 Theme: Pet food reviews
2, Messy and apparently undelicious:   My cat is not a huge fan. Sure, she'll lap up the gravy, but leaves th
4, The cats like it:   My 7 cats like this food but it is a little yucky for the human. Piece
5, cant get enough of it!!!:   Our lil shih tzu puppy cannot get enough of it. Everytime she sees the
1, Food Caused Illness:   I switched my cats over from the Blue Buffalo Wildnerness Food to this
5, My furbabies LOVE these!:   Shake the container and they come running. Even my boy cat, who isn't 
----------------------------------------------------------------------------------------------------
Cluster 2 Theme: All the reviews are about different types of coffee.
5, Fog Chaser Coffee:   This coffee has a full body and a rich taste. The price is far below t
5, Excellent taste:   This is to me a great coffee, once you try it you will enjoy it, this 
4, Good, but not Wolfgang Puck good:   Honestly, I have to admit that I expected a little better. That's not 
5, Just My Kind of Coffee:   Coffee Masters Hazelnut coffee used to be carried in a local coffee/pa
5, Rodeo Drive is Crazy Good Coffee!:   Rodeo Drive is my absolute favorite and I'm ready to order more!  That
----------------------------------------------------------------------------------------------------
Cluster 3 Theme: The theme of these customer reviews is food and drink products.
5, Wonderful alternative to soda pop:   This is a wonderful alternative to soda pop.  It's carbonated for thos
5, So convenient, for so little!:   I needed two vanilla beans for the Love Goddess cake that my husbands 
2, bot very cheesy:   Got this about a month ago.first of all it smells horrible...it tastes
5, Delicious!:   I am not a huge beer lover.  I do enjoy an occasional Blue Moon (all o
3, Just ok:   I bought this brand because it was all they had at Ranch 99 near us. I
----------------------------------------------------------------------------------------------------
```
{{< /ltr >}}

مهم است که توجه داشته باشید که خوشه‌ها لزوماً با آنچه شما قصد استفاده از آنها را دارید مطابقت نخواهند داشت. تعداد بیشتری از خوشه‌ها بر الگوهای خاص‌تر تمرکز خواهند کرد، در حالی که تعداد کمتری از خوشه‌ها معمولاً بر بزرگترین تفاوت‌ها در داده‌ها تمرکز خواهند کرد.
