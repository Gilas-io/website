---
title: "نمایش دو بعدی embeddings"
description: "استفاده از t-SNE برای کاهش ابعاد `embeddings` و نمایش آن‌ها در نمودار پراکندگی ۲ بعدی."
tags:
- visualization
- embeddings
weight: 
og_image: "/posts/visualizing_embeddings_in_2d/banner.png"
---

{{< postcover src="/posts/visualizing_embeddings_in_2d/banner.png" >}}


## نمایش دو بعدی `embeddings`

ما از `t-SNE` برای کاهش ابعاد `embeddings` از ۱۵۳۶ به ۲ استفاده خواهیم کرد. پس از کاهش ابعاد به دو بعد، می‌توانیم آن‌ها را در یک نمودار پراکندگی ۲ بعدی نمایش دهیم. 


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

### کاهش ابعاد

ما ابعاد را با استفاده از `t-SNE` به ۲ بعد کاهش می‌دهیم.
```python
import pandas as pd
from sklearn.manifold import TSNE
import numpy as np
from ast import literal_eval

# Load the embeddings
datafile_path = "data/fine_food_reviews_with_embeddings_1k.csv"
df = pd.read_csv(datafile_path)

# Convert to a list of lists of floats
matrix = np.array(df.embedding.apply(literal_eval).to_list())

# Create a t-SNE model and transform the data
tsne = TSNE(n_components=2, perplexity=15, random_state=42, init='random', learning_rate=200)
vis_dims = tsne.fit_transform(matrix)
vis_dims.shape
```

### رسم `embeddings`

ما هر نظر را با رنگ ستاره آن، از قرمز تا سبز، رنگ‌آمیزی می‌کنیم.

می‌توانیم مشاهده کنیم که حتی در ابعاد کاهش یافته ۲ بعدی، جداسازی داده‌ها به خوبی انجام شده است.
```python
import matplotlib.pyplot as plt
import matplotlib
import numpy as np

colors = ["red", "darkorange", "gold", "turquoise", "darkgreen"]
x = [x for x,y in vis_dims]
y = [y for x,y in vis_dims]
color_indices = df.Score.values - 1

colormap = matplotlib.colors.ListedColormap(colors)
plt.scatter(x, y, c=color_indices, cmap=colormap, alpha=0.3)
for score in [0,1,2,3,4]:
    avg_x = np.array(x)[df.Score-1==score].mean()
    avg_y = np.array(y)[df.Score-1==score].mean()
    color = colors[score]
    plt.scatter(avg_x, avg_y, marker='x', color=color, s=100)

plt.title("Amazon ratings visualized in language using t-SNE")
```

{{<img image="/posts/visualizing_embeddings_in_2d/banner.png" >}}