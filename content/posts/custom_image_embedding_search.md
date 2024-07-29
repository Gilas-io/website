---
title: "RAG چندوجهی با CLIP و GPT-4 Vision"
description: "ادغام چندوجهی RAG با استفاده از CLIP و GPT-4 Vision برای بهبود جستجو و پاسخ‌دهی به سوالات."
tags:
- vision
- embedding
- RAG
weight: 
og_image: "/posts/custom_image_embedding_search/banner.png"
---

{{< postcover src="/posts/custom_image_embedding_search/banner.png" >}}

# RAG چندوجهی با CLIP Embeddings و GPT-4 Vision

استفاده از سیستم‌های RAG چندوجهی با افزودن حالت‌های اضافی به RAG های ساده‌ی مبتنی بر متن٬ قابلیت‌ `LLM`ها در پاسخ‌دهی به سوالات را با ارائه زمینه اضافی و پایه‌گذاری داده‌های متنی برای درک بهتر، بهبود می‌بخشد.

با اتخاذ رویکرد ارایه شده در پست  [ساخت اپلیکیشن تطبیق لباس](/how_to_combine_gpt4o_with_rag_outfit_assistant)، ما تصاویر را برای جستجوی شباهت میان آنها امبدینگ می‌کنیم و از فرآیند از دست دادن اطلاعات در کپشن‌نویسی متنی جلوگیری می‌کنیم تا دقت بازیابی را افزایش دهیم.

استفاده از `CLIP-based embeddings` همچنین امکان `fine-tune` با داده‌های خاص یا به‌روزرسانی با تصاویر دیده‌نشده را نیز فراهم می‌کند.

این تکنیک از طریق جستجو در یک پایگاه دانش سازمانی با تصاویر فنی ارائه‌شده توسط کاربر برای ارائه اطلاعات مرتبط نشان داده می‌شود.


ابتدا پکیج های مربوطه را نصب کنید.

```python
#installations
%pip install clip
%pip install torch
%pip install pillow
%pip install faiss-cpu
%pip install numpy
%pip install git+https://github.com/openai/CLIP.git
%pip install openai
```

سپس تمام بسته‌های مورد نیاز را وارد کنید.

{{< hint info >}}
**برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.**
{{< /hint >}} 

```python
# model imports
import faiss
import json
import torch
from openai import OpenAI
import torch.nn as nn
from torch.utils.data import DataLoader
import clip

# helper imports
from tqdm import tqdm
import json
import os
import numpy as np
import pickle
from typing import List, Union, Tuple

# visualisation imports
from PIL import Image
import matplotlib.pyplot as plt
import base64

client = OpenAI(
    api_key=os.environ.get(("GILAS_API_KEY", "<کلید API خود را اینجا بسازید https://dashboard.gilas.io/apiKey>")), 
    base_url="https://api.gilas.io/v1/" # Gilas APIs
)
```

حالا بیایید مدل `CLIP` را بارگذاری کنیم.

```python
#load model on device. The device you are running inference/training on is either a CPU or GPU if you have.
device = "cpu"
model, preprocess = clip.load("ViT-B/32",device=device)
```

حال آماده هستیم تا:
1. پایگاه داده‌ی امبدینگ تصاویر را ایجاد می‌کنیم
2. یک پرس و جو به مدل vision را آماده می‌کنیم
3. جستجوی معنایی را انجام می‌دهیم
4. پرس و جو کاربر را به تصویر ارسال می‌کنیم

# ایجاد پایگاه داده‌ی امبدینگ تصویر

در این مرحله پایگاه دانش امبدینگ تصویر خود را از یک دایرکتوری از تصاویر ایجاد خواهیم کرد. ما ازین پایگاه دانش برای ارایه اطلاعات در مورد تصاویری که کاربر آپلود می‌کند استفاده می‌کنیم.

ما همچنین یک فایل `description.json` داریم که برای هر تصویر در پایگاه دانش ما یک ورودی دارد. این فایل دارای دو کلید است: 'image_path' و 'description' که هر تصویر را به یک توضیح مفید از آن تصویر برای کمک به پاسخ‌دهی به سوال کاربر نگاشت می‌کند.

تابع زیر برای دریافت تمام تصاویر موجود در یک دایرکتوری خاص است.

```python
def get_image_paths(directory: str, number: int = None) -> List[str]:
    image_paths = []
    count = 0
    for filename in os.listdir(directory):
        if filename.endswith('.jpeg'):
            image_paths.append(os.path.join(directory, filename))
            if number is not None and count == number:
                return [image_paths[-1]]
            count += 1
    return image_paths
direc = 'image_database/'
image_paths = get_image_paths(direc)
```

در مرحله بعد، یک تابع برای دریافت امبدینگ‌های تصویر از مدل `CLIP` می‌نویسیم. ابتدا تصویر را با استفاده از تابع `preprocess` که قبلاً دریافت کردیم، پیش‌پردازش می‌کنیم. این تابع چندین کار از جمله تغییر اندازه، نرمال‌سازی، تنظیم کانال رنگ و غیره انجام می‌دهد تا اطمینان حاصل کند که ورودی به مدل `CLIP` از فرمت و ابعاد صحیح برخوردار است، 

سپس این تصاویر پیش‌پردازش شده را با هم ترکیب می‌کنیم تا بتوانیم آنها را به مدل به صورت یکجا و نه در یک حلقه ارسال کنیم. و در نهایت خروجی مدل که یک آرایه از امبدینگ‌ها است را برمی‌گردانیم.

```python
def get_features_from_image_path(image_paths):
  images = [preprocess(Image.open(image_path).convert("RGB")) for image_path in image_paths]
  image_input = torch.tensor(np.stack(images))
  with torch.no_grad():
    image_features = model.encode_image(image_input).float()
  return image_features
image_features = get_features_from_image_path(image_paths)
```

اکنون می‌توانیم پایگاه داده امبدینگ خود را ایجاد کنیم.

```python
index = faiss.IndexFlatIP(image_features.shape[1])
index.add(image_features)
```

و همچنین `json` خود را برای نگاشت تصویر-توضیح ایمپورت کرده و یک لیست از `json`ها ایجاد می‌کنیم. همچنین یک تابع کمکی برای جستجو در این لیست برای یک تصویر خاص ایجاد می‌کنیم تا بتوانیم توضیح آن تصویر را بدست آوریم.

```python
data = []
image_path = 'train1.jpeg'
with open('description.json', 'r') as file:
    for line in file:
        data.append(json.loads(line))
def find_entry(data, key, value):
    for entry in data:
        if entry.get(key) == value:
            return entry
    return None
```

حال بیایید یک تصویر نمونه را نمایش دهیم که توسط کاربر آپلود شده است. این یک قطعه فناوری است که در `CES 2024` رونمایی شد. این دستگاه `DELTA Pro Ultra Whole House Battery Generator` نام دارد.

```python
im = Image.open(image_path)
plt.imshow(im)
plt.show()
```

{{< img image="/posts/custom_image_embedding_search/train1.jpeg" alt="training image" >}}

# پرس و جوی مدل

حالا بیایید ببینیم `GPT-4 Vision` (که قبلاً این فناوری را ندیده است) آن را چگونه برچسب‌گذاری می‌کند.

ابتدا باید یک تابع برای کدگذاری تصویرمان به صورت `base64` بنویسیم زیرا این فرمتی است که مدل انتظار دریافت آن را دارد. سپس یک تابع `image_query`  ایجاد می‌کنیم تا به
ما اجازه دهد با ورودی تصویر به `LLM` پرس و جو کنیم.

```python
def encode_image(image_path):
    with open(image_path, 'rb') as image_file:
        encoded_image = base64.b64encode(image_file.read())
        return encoded_image.decode('utf-8')

def image_query(query, image_path):
    response = client.chat.completions.create(
        model='gpt-4-vision-preview',
        messages=[
            {
            "role": "user",
            "content": [
                {
                "type": "text",
                "text": query,
                },
                {
                "type": "image_url",
                "image_url": {
                    "url": f"data:image/jpeg;base64,{encode_image(image_path)}",
                },
                }
            ],
            }
        ],
        max_tokens=300,
    )
    # Extract relevant features from the response
    return response.choices[0].message.content
image_query('Write a short label of what is show in this image?', image_path)
```

{{< ltr >}}
```
'Autonomous Delivery Robot'
```
{{</ ltr >}}

همانطور که می‌بینیم، مدل تلاش می‌کند از اطلاعاتی که آموزش دیده است استفاده کند اما به دلیل ندیدن چیزی مشابه به تصویر بالا در داده‌های آموزشی خود شی داخل تصویر را اشتباه تشخصی می‌دهد. این به این دلیل است که تصویر مبهم است و استنتاج آن دشوار است.

# جستجوی معنایی

حالا بیایید جستجوی شباهت را برای یافتن دو تصویر مشابه در پایگاه دانش خود انجام دهیم. این کار را با دریافت امبدینگ‌های مسیر تصویر ورودی کاربر، بازیابی ایندکس‌ها و فاصله‌های تصاویر مشابه در پایگاه داده خود انجام می‌دهیم. فاصله میان امبدینگ‌ها شاخص ما برای شباهت خواهد بود و فاصله کمتر به معنای شباهت بیشتر است. سپس نتایج جستجو را بر اساس فاصله به ترتیب نزولی مرتب می‌کنیم.

```python
image_search_embedding = get_features_from_image_path([image_path])
distances, indices = index.search(image_search_embedding.reshape(1, -1), 2) #2 signifies the number of topmost similar images to bring back
distances = distances[0]
indices = indices[0]
indices_distances = list(zip(indices, distances))
indices_distances.sort(key=lambda x: x[1], reverse=True)
```


بیایید ببینیم چه چیزی برگردانده است (این تصاویر را به ترتیب شباهت نمایش می‌دهیم):

```python
#display similar images
for idx, distance in indices_distances:
    print(idx)
    path = get_image_paths(direc, idx)[0]
    im = Image.open(path)
    plt.imshow(im)
    plt.show()
```

{{< img image="/posts/custom_image_embedding_search/train2.jpeg" alt="training image" >}}

{{< img image="/posts/custom_image_embedding_search/train17.jpeg" alt="training image" >}}

می‌بینیم که دو تصویر را برگردانده است که شامل `DELTA Pro Ultra Whole House Battery Generator` هستند. در یکی از تصاویر نیز پس‌زمینه‌ای وجود دارد که ممکن است مدل را منحرف کند اما مدل موفق به یافتن تصویر صحیح شده است.

# پرس و جو کاربر

حالا با انتخاب شبیه‌ترین تصویر می‌خواهیم آنرا و همراه توضیحاتی که از آن تصویر در پایگاه داده خود داریم در کنار  پرس و جو کاربر از آن تصویر به مدل ارسال کنیم تا کاربر بتواند در مورد آن تصویر سوالاتی را از مدل بپرسد. اینجا جایی است که توانایی vision مدل حایز اهمیت می‌شود، جایی که می‌توانید سوالات عمومی که مدل به طور خاص برای آنها آموزش ندیده است را از مدل بپرسید تا جواب‌هایی با دقت بالا دریافت کنید.

در مثال زیر، ما در مورد ظرفیت آیتم مورد نظر سوال خواهیم کرد.

```python
similar_path = get_image_paths(direc, indices_distances[0][0])[0]
element = find_entry(data, 'image_path', similar_path)

user_query = 'What is the capacity of this item?'
prompt = f"""
Below is a user query, I want you to answer the query using the description and image provided.

user query:
{user_query}

description:
{element['description']}
"""
image_query(prompt, similar_path)
```

{{< ltr >}}
```
'The portable home battery DELTA Pro has a base capacity of 3.6kWh. This capacity can be expanded up to 25kWh with additional batteries. The image showcases the DELTA Pro, which has an impressive 3600W power capacity for AC output as well.'
```
{{</ ltr >}}

و می‌بینیم که مدل قادر به پاسخ دادن به سوال است هرچند که هیچ اطلاعاتی در مورد این محصول خاص نداشته و تنها با اتکا به اطلاعات بازیابی شده از پایگاه داده امبدینگ و توضیحات مرتبط با تصایر قادر به پاسخگویی به سوالات است. این تنها با تطبیق مستقیم تصاویر و از آنجا جمع‌آوری توضیحات مرتبط به عنوان زمینه ممکن بود.

# نتیجه‌گیری

در این پست، ما نحوه استفاده از مدل `CLIP`، یک مثال از ایجاد پایگاه داده امبدینگ تصویر با استفاده از مدل `CLIP`، انجام جستجوی معنایی و در نهایت ارائه یک پرس و جو کاربر برای پاسخ به سوال را ارایه دادیم.