go---
title: "ترکیب GPT-4o با RAG - ساخت اپلیکیشن تطبیق لباس"
description: "این پروژه قدرت مدل GPT-4o را در تحلیل تصاویر لباس‌ها و استخراج ویژگی‌های کلیدی مانند رنگ، سبک و نوع نشان می‌دهد."
tags:
- embeddings
- gpt-4o
weight: 2005
og_image: "/posts/how_to_combine_gpt4o_with_rag_outfit_assistant/banner.png"
---

{{< postcover src="/posts/how_to_combine_gpt4o_with_rag_outfit_assistant/banner.png" >}}

به اپلیکیشن تطبیق لباس خوش آمدید! این پروژه قدرت مدل `GPT-4o` را در تحلیل تصاویر لباس‌ها و استخراج ویژگی‌های کلیدی مانند رنگ، سبک و نوع نشان می‌دهد. هسته اصلی اپلیکیشن ما بر اساس این مدل پیشرفته تحلیل تصویر که توسط `OpenAI` توسعه یافته است، استوار است که به ما امکان می‌دهد ویژگی‌های لباس ورودی را به دقت شناسایی کنیم.

با استفاده از قابلیت‌های مدل `GPT-4o`، ما از یک الگوریتم تطبیق سفارشی و تکنیک `RAG` برای جستجو در پایگاه دانش خود برای آیتم‌هایی که با ویژگی‌های شناسایی شده مطابقت دارند، استفاده می‌کنیم. این الگوریتم عواملی مانند سازگاری رنگ و هماهنگی سبک را در نظر می‌گیرد تا به کاربران پیشنهادات مناسبی ارائه دهد. 

استفاده از ترکیب `GPT-4o` + `RAG` چندین مزیت دارد:

1. **درک متنی**: `GPT-4o` می‌تواند تصاویر ورودی را تحلیل کرده و اطلاعات مربوط به زمینه را ، مانند اشیاء، صحنه‌ها و فعالیت‌های نمایش داده شده نیز درک کند.
2. **پایگاه دانش غنی**: `RAG` قابلیت‌های تولیدی `GPT-4` را با یک جزء بازیابی که به یک مجموعه بزرگ از اطلاعات در زمینه‌های مختلف دسترسی دارد، ترکیب می‌کند. این بدان معناست که سیستم می‌تواند پیشنهاداتی بر اساس طیف گسترده‌ای از اطلاعات را ارائه دهد.
3. **سفارشی‌سازی**: این رویکرد امکان سفارشی‌سازی آسان برای پاسخگویی به نیازها یا ترجیحات خاص کاربران در برنامه‌های مختلف را فراهم می‌کند. چه در تطبیق پیشنهادات با سلیقه کاربر در هنر یا ارائه محتوای آموزشی بر اساس سطح یادگیری دانش‌آموز، سیستم می‌تواند برای ارائه تجربیات شخصی‌سازی شده تطبیق یابد.

به طور کلی، رویکرد `GPT-4o` + `RAG` یک راه‌حل قدرتمند و انعطاف‌پذیر برای برنامه‌های مختلف مرتبط با مد ارائه می‌دهد که از نقاط قوت هر دو تکنیک تولیدی و بازیابی مبتنی بر هوش مصنوعی بهره می‌برد.

### تنظیم محیط

ابتدا پکیج‌های لازم را نصب کرده و برخی توابع کمکی که بعداً استفاده خواهیم کرد را می‌نویسیم.

{{< hint info >}}
**برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.**
{{< /hint >}} 

```python
%pip install openai --quiet
%pip install tenacity --quiet
%pip install tqdm --quiet
%pip install numpy --quiet
%pip install typing --quiet
%pip install tiktoken --quiet
%pip install concurrent --quiet
```

```python
import pandas as pd
import numpy as np
import json
import ast
import tiktoken
import concurrent
from openai import OpenAI
from tqdm import tqdm
from tenacity import retry, wait_random_exponential, stop_after_attempt
from IPython.display import Image, display, HTML
from typing import List

client = OpenAI(
    api_key=os.environ.get(("GILAS_API_KEY", "<کلید API خود را اینجا بسازید https://dashboard.gilas.io/apiKey>")), 
    base_url="https://api.gilas.io/v1/" # Gilas APIs
)

GPT_MODEL = "gpt-4o"
EMBEDDING_MODEL = "text-embedding-3-large"
EMBEDDING_COST_PER_1K_TOKENS = 0.00013
```

### ایجاد `Embeddings`
اکنون پایگاه دانش را با انتخاب یک پایگاه داده و ساخت `embeddings` برای آن تولید خواهیم کرد. من از فایل `sample_styles.csv` در پوشه داده‌ها برای این کار استفاده می‌کنم. این نمونه‌ای از یک مجموعه داده بزرگتر است که حاوی `~44K` آیتم است. این مرحله را می‌توان با استفاده از یک پایگاه داده برداری آماده جایگزین کرد.

```python
styles_filepath = "data/sample_clothes/sample_styles.csv"
styles_df = pd.read_csv(styles_filepath, on_bad_lines='skip')
print(styles_df.head())
print("Opened dataset successfully. Dataset has {} items of clothing.".format(len(styles_df)))
```

{{< ltr >}}
```
      id gender masterCategory subCategory articleType baseColour  season  \
0  27152    Men        Apparel     Topwear      Shirts       Blue  Summer   
1  10469    Men        Apparel     Topwear     Tshirts     Yellow    Fall   
2  17169    Men        Apparel     Topwear      Shirts     Maroon    Fall   
3  56702    Men        Apparel     Topwear      Kurtas       Blue  Summer   
4  47062  Women        Apparel  Bottomwear     Patiala      Multi    Fall   

     year   usage                       productDisplayName  
0  2012.0  Formal       Mark Taylor Men Striped Blue Shirt  
1  2011.0  Casual   Flying Machine Men Yellow Polo Tshirts  
2  2011.0  Casual  U.S. Polo Assn. Men Checks Maroon Shirt  
3  2012.0  Ethnic                  Fabindia Men Blue Kurta  
4  2012.0  Ethnic        Shree Women Multi Colored Patiala  
Opened dataset successfully. Dataset has 1000 items of clothing.
```
{{< /ltr >}}

اکنون `embeddings` را برای کل مجموعه داده تولید خواهیم کرد. می‌توانیم اجرای این `embeddings` را به صورت موازی انجام دهیم تا اطمینان حاصل کنیم که اسکریپت برای مجموعه داده‌های بزرگتر مقیاس‌پذیر است. در اینصورت زمان ایجاد `embeddings` برای مجموعه داده کامل `44K` از ~4 ساعت به ~2-3 دقیقه کاهش می‌یابد.
```python
## Batch Embedding Logic

# Simple function to take in a list of text objects and return them as a list of embeddings
@retry(wait=wait_random_exponential(min=1, max=40), stop=stop_after_attempt(10))
def get_embeddings(input: List):
    response = client.embeddings.create(
        input=input,
        model=EMBEDDING_MODEL
    ).data
    return [data.embedding for data in response]


# Splits an iterable into batches of size n.
def batchify(iterable, n=1):
    l = len(iterable)
    for ndx in range(0, l, n):
        yield iterable[ndx : min(ndx + n, l)]
     

# Function for batching and parallel processing the embeddings
def embed_corpus(
    corpus: List[str],
    batch_size=64,
    num_workers=8,
    max_context_len=8191,
):
    # Encode the corpus, truncating to max_context_len
    encoding = tiktoken.get_encoding("cl100k_base")
    encoded_corpus = [
        encoded_article[:max_context_len] for encoded_article in encoding.encode_batch(corpus)
    ]

    # Calculate corpus statistics: the number of inputs, the total number of tokens, and the estimated cost to embed
    num_tokens = sum(len(article) for article in encoded_corpus)
    cost_to_embed_tokens = num_tokens / 1000 * EMBEDDING_COST_PER_1K_TOKENS
    print(
        f"num_articles={len(encoded_corpus)}, num_tokens={num_tokens}, est_embedding_cost={cost_to_embed_tokens:.2f} USD"
    )

    # Embed the corpus
    with concurrent.futures.ThreadPoolExecutor(max_workers=num_workers) as executor:
        
        futures = [
            executor.submit(get_embeddings, text_batch)
            for text_batch in batchify(encoded_corpus, batch_size)
        ]

        with tqdm(total=len(encoded_corpus)) as pbar:
            for _ in concurrent.futures.as_completed(futures):
                pbar.update(batch_size)

        embeddings = []
        for future in futures:
            data = future.result()
            embeddings.extend(data)

        return embeddings
    

# Function to generate embeddings for a given column in a DataFrame
def generate_embeddings(df, column_name):
    # Initialize an empty list to store embeddings
    descriptions = df[column_name].astype(str).tolist()
    embeddings = embed_corpus(descriptions)

    # Add the embeddings as a new column to the DataFrame
    df['embeddings'] = embeddings
    print("Embeddings created successfully.")
```

#### دو گزینه برای ایجاد `embeddings`:
خط بعدی `embeddings` را برای مجموعه داده لباس نمونه ایجاد می‌کند. این کار حدود 0.02 ثانیه برای پردازش و حدود ~30 ثانیه برای نوشتن نتایج به یک فایل لوکال .csv زمان می‌برد. این فرآیند از مدل `text_embedding_3_large` استفاده می‌کند که با قیمت `$0.0004/1K` توکن قیمت‌گذاری شده است (برای اطلاع از قیمت‌ استفاده از مدل‌های قابل دسترس از طریق پلتفرم گیلاس لطفا [صفحه هزینه‌ها](/pricing) را مطالعه کنید). با توجه به اینکه مجموعه داده حدود `1K` ورودی دارد، این عملیات تقریباً `$0.004` هزینه خواهد داشت. اگر تصمیم به کار با کل مجموعه داده `44K` ورودی بگیرید، این عملیات 2-3 دقیقه زمان می‌برد و تقریباً `$0.15` هزینه خواهد داشت.


```python
generate_embeddings(styles_df, 'productDisplayName')
print("Writing embeddings to file ...")
styles_df.to_csv('data/sample_clothes/sample_styles_with_embeddings.csv', index=False)
print("Embeddings successfully stored in sample_styles_with_embeddings.csv")
```

{{< ltr >}}
```
num_articles=1000, num_tokens=8280, est_embedding_cost=0.00 USD

1024it [00:01, 724.12it/s]                         


Embeddings created successfully.
Writing embeddings to file ...
Embeddings successfully stored in sample_styles_with_embeddings.csv
```

{{< /ltr >}}

```python
print(styles_df.head())
print("Opened dataset successfully. Dataset has {} items of clothing along with their embeddings.".format(len(styles_df)))
```

{{< ltr >}}
```
      id gender masterCategory subCategory articleType baseColour  season  \
0  27152    Men        Apparel     Topwear      Shirts       Blue  Summer   
1  10469    Men        Apparel     Topwear     Tshirts     Yellow    Fall   
2  17169    Men        Apparel     Topwear      Shirts     Maroon    Fall   
3  56702    Men        Apparel     Topwear      Kurtas       Blue  Summer   
4  47062  Women        Apparel  Bottomwear     Patiala      Multi    Fall   

     year   usage                       productDisplayName  \
0  2012.0  Formal       Mark Taylor Men Striped Blue Shirt   
1  2011.0  Casual   Flying Machine Men Yellow Polo Tshirts   
2  2011.0  Casual  U.S. Polo Assn. Men Checks Maroon Shirt   
3  2012.0  Ethnic                  Fabindia Men Blue Kurta   
4  2012.0  Ethnic        Shree Women Multi Colored Patiala   

                                          embeddings  
0  [0.006903026718646288, 0.0004031236458104104, ...  
1  [-0.04371623694896698, -0.008869604207575321, ...  
2  [-0.027989011257886887, 0.05884069576859474, -...  
3  [-0.004197604488581419, 0.029409468173980713, ...  
4  [-0.05241541564464569, 0.015912825241684914, -...  
Opened dataset successfully. Dataset has 1000 items of clothing along with their embeddings.
```
{{< /ltr >}}

### ساخت الگوریتم تطبیق

در این بخش، ما یک الگوریتم بازیابی شباهت کسینوسی برای یافتن آیتم‌های مشابه در دیتافریم خود توسعه خواهیم داد. در حالی که کتابخانه `sklearn` یک تابع شباهت کسینوسی داخلی ارائه می‌دهد، به‌روزرسانی‌های اخیر در `SDK` آن منجر به مشکلات سازگاری شده است، بنابراین ما محاسبه شباهت کسینوسی استاندارد خود را پیاده‌سازی می‌کنیم.

اگر قبلاً یک پایگاه داده برداری تنظیم کرده‌اید، می‌توانید این مرحله را رد کنید. اکثر پایگاه‌های داده استاندارد دارای توابع جستجوی خود هستند که مراحل بعدی این راهنما را ساده می‌کنند. با این حال، ما قصد داریم نشان دهیم که الگوریتم تطبیق می‌تواند برای برآورده کردن نیازهای خاص، مانند یک تعیین کردن یک حد آستانه‌ی خاص یا تعداد مشخصی از تطابق‌های بازگشتی، سفارشی شود.

تابع `find_similar_items` چهار پارامتر را می‌پذیرد:
- `embedding`: `embedding` که می‌خواهیم برای آن تطابق پیدا کنیم.
- `embeddings`: لیستی از `embeddings` برای جستجو در میان بهترین تطابق‌ها.
- `threshold` (اختیاری): این پارامتر حداقل امتیاز شباهت برای در نظر گرفتن یک تطابق را مشخص می‌کند. یک آستانه بالاتر منجر به تطابق‌های نزدیک‌تر (بهتر) می‌شود، در حالی که یک آستانه پایین‌تر اجازه می‌دهد تا آیتم‌های بیشتری بازگردانده شوند، اگرچه ممکن است به اندازه تطابق اولیه نزدیک نباشند.
- `top_k` (اختیاری): این پارامتر تعداد آیتم‌هایی را که از آستانه داده شده فراتر می‌روند و بالاترین امتیاز تطابق را دارند، تعیین می‌کند.
```python
def cosine_similarity_manual(vec1, vec2):
    """Calculate the cosine similarity between two vectors."""
    vec1 = np.array(vec1, dtype=float)
    vec2 = np.array(vec2, dtype=float)


    dot_product = np.dot(vec1, vec2)
    norm_vec1 = np.linalg.norm(vec1)
    norm_vec2 = np.linalg.norm(vec2)
    return dot_product / (norm_vec1 * norm_vec2)


def find_similar_items(input_embedding, embeddings, threshold=0.5, top_k=2):
    """Find the most similar items based on cosine similarity."""
    
    # Calculate cosine similarity between the input embedding and all other embeddings
    similarities = [(index, cosine_similarity_manual(input_embedding, vec)) for index, vec in enumerate(embeddings)]
    
    # Filter out any similarities below the threshold
    filtered_similarities = [(index, sim) for index, sim in similarities if sim >= threshold]
    
    # Sort the filtered similarities by similarity score
    sorted_indices = sorted(filtered_similarities, key=lambda x: x[1], reverse=True)[:top_k]

    # Return the top-k most similar items
    return sorted_indices
```

```python
def find_matching_items_with_rag(df_items, item_descs):
   """Take the input item descriptions and find the most similar items based on cosine similarity for each description."""
   
   # Select the embeddings from the DataFrame.
   embeddings = df_items['embeddings'].tolist()

   
   similar_items = []
   for desc in item_descs:
      
      # Generate the embedding for the input item
      input_embedding = get_embeddings([desc])
    
      # Find the most similar items based on cosine similarity
      similar_indices = find_similar_items(input_embedding, embeddings, threshold=0.6)
      similar_items += [df_items.iloc[i] for i in similar_indices]
    
   return similar_items
```

### ماژول تحلیل

در این ماژول، ما از `gpt-4o` برای تحلیل تصاویر ورودی و استخراج ویژگی‌های مهم مانند توضیحات دقیق، سبک‌ها و غیره استفاده می‌کنیم. تحلیل از طریق یک فراخوانی ساده `API` انجام می‌شود، جایی که ما URL تصویر را برای تحلیل ارائه می‌دهیم و از مدل می‌خواهیم ویژگی‌های مرتبط را شناسایی کند.

برای اطمینان از اینکه مدل نتایج دقیقی بازمی‌گرداند، از تکنیک‌های خاصی در پراپمت خود استفاده می‌کنیم:

1. **مشخص کردن فرمت خروجی**: ما به مدل دستور می‌دهیم که یک `JSON` با ساختار از پیش تعریف شده بازگرداند که شامل:
   - `items` (str[]): لیستی از رشته‌ها، هر کدام نمایانگر عنوان مختصری برای یک آیتم لباس، شامل سبک، رنگ و جنسیت. این عناوین به `productDisplayName` در پایگاه داده اصلی ما شباهت دارند.
   - `category` (str): دسته‌ای که بهترین نمایانگر آیتم داده شده است. مدل از لیست تمام `articleTypes` منحصر به فرد موجود در دیتافریم اصلی سبک‌ها انتخاب می‌کند.
   - `gender` (str): برچسبی که نشان می‌دهد آیتم برای کدام جنسیت است. مدل از گزینه‌های `[Men, Women, Boys, Girls, Unisex]` انتخاب می‌کند.

2. **دستورالعمل‌های واضح و مختصر**:
   - ما دستورالعمل‌های (prompt) واضحی در مورد اینکه عناوین آیتم‌ها باید شامل چه چیزی باشند و فرمت خروجی چگونه باید باشد، ارائه می‌دهیم. خروجی باید در فرمت `JSON`باشد، اما بدون تگ `json` که معمولاً در پاسخ مدل وجود دارد.

3. **مثال single-shot**:
   - برای روشن‌تر کردن خروجی مورد انتظار، یک مثال از ورودی و یک خروجی مطلوب را به مدل ارائه می‌دهیم. اگرچه این کار ممکن است تعداد توکن‌های استفاده شده (و در نتیجه هزینه فراخوانی) را افزایش دهد، اما به هدایت مدل کمک می‌کند و منجر به عملکرد کلی بهتر می‌شود.

با پیروی از این رویکرد ساختاریافته، هدف ما این است که اطلاعات دقیق و مفیدی از مدل `gpt-4o` برای تحلیل بیشتر و ادغام در پایگاه داده خود به دست آوریم.
```python
def analyze_image(image_base64, subcategories):
    response = client.chat.completions.create(
        model=GPT_MODEL,
        messages=[
            {
            "role": "user",
            "content": [
                {
                "type": "text",
                "text": """Given an image of an item of clothing, analyze the item and generate a JSON output with the following fields: "items", "category", and "gender". 
                           Use your understanding of fashion trends, styles, and gender preferences to provide accurate and relevant suggestions for how to complete the outfit.
                           The items field should be a list of items that would go well with the item in the picture. Each item should represent a title of an item of clothing that contains the style, color, and gender of the item.
                           The category needs to be chosen between the types in this list: {subcategories}.
                           You have to choose between the genders in this list: [Men, Women, Boys, Girls, Unisex]
                           Do not include the description of the item in the picture. Do not include the ```json ``` tag in the output.
                           
                           Example Input: An image representing a black leather jacket.

                           Example Output: {"items": ["Fitted White Women's T-shirt", "White Canvas Sneakers", "Women's Black Skinny Jeans"], "category": "Jackets", "gender": "Women"}
                           """,
                },
                {
                "type": "image_url",
                "image_url": {
                    "url": f"data:image/jpeg;base64,{image_base64}",
                },
                }
            ],
            }
        ],
        max_tokens=300,
    )
    # Extract relevant features from the response
    features = response.choices[0].message.content
    return features
```

### تست پراپمت با تصاویر نمونه

برای ارزیابی اثربخشی پراپمت خود، بیایید آن را با تصویری از مجموعه داده خود آزمایش کنیم. ما از تصاویر موجود در پوشه `"data/sample_clothes/sample_images"` استفاده خواهیم کرد و اطمینان حاصل می‌کنیم که تنوعی از سبک‌ها، جنسیت‌ها و انواع را شامل شود. نمونه‌های انتخاب شده عبارتند از:

2133.jpg
{{< img image="/posts/how_to_combine_gpt4o_with_rag_outfit_assistant/2133.jpg" alt="cluster" >}}

4226.jpg
{{< img image="/posts/how_to_combine_gpt4o_with_rag_outfit_assistant/4226.jpg" alt="cluster" >}}

7143.jpg
{{< img image="/posts/how_to_combine_gpt4o_with_rag_outfit_assistant/7143.jpg" alt="cluster" >}}

با آزمایش پراپمت با این تصاویر متنوع، می‌توانیم توانایی آن را در تحلیل دقیق و استخراج ویژگی‌های مرتبط از انواع مختلف آیتم‌های لباس و لوازم جانبی ارزیابی کنیم.

ما به یک تابع کمکی برای کدگذاری تصاویر .jpg به base64 نیاز داریم.
```python
import base64

def encode_image_to_base64(image_path):
    with open(image_path, 'rb') as image_file:
        encoded_image = base64.b64encode(image_file.read())
        return encoded_image.decode('utf-8')
```

```python
# Set the path to the images and select a test image
image_path = "data/sample_clothes/sample_images/"
test_images = ["2133.jpg", "7143.jpg", "4226.jpg"]

# Encode the test image to base64
reference_image = image_path + test_images[0]
encoded_image = encode_image_to_base64(reference_image)
```

```python
# Select the unique subcategories from the DataFrame
unique_subcategories = styles_df['articleType'].unique()

# Analyze the image and return the results
analysis = analyze_image(encoded_image, unique_subcategories)
image_analysis = json.loads(analysis)

# Display the image and the analysis results
display(Image(filename=reference_image))
print(image_analysis)
```

{{< ltr >}}
```
{'items': ["Slim Fit Blue Men's Jeans", "White Men's Sneakers", "Men's Silver Watch"], 'category': 'Shirts', 'gender': 'Men'}
```
{{< /ltr >}}

سپس، خروجی تحلیل تصویر را پردازش کرده و از آن برای فیلتر کردن و نمایش آیتم‌های مطابق از مجموعه داده خود استفاده می‌کنیم. در اینجا یک توضیح از کد ارائه شده است:

1. **استخراج نتایج تحلیل تصویر**: ما توضیحات آیتم، دسته‌بندی و جنسیت را از دیکشنری `image_analysis` استخراج می‌کنیم.

2. **فیلتر کردن مجموعه داده**: ما دیتافریم `styles_df` را فیلتر می‌کنیم تا فقط آیتم‌هایی که با جنسیت تحلیل تصویر مطابقت دارند (یا یونی‌سکس هستند) و آیتم‌هایی که از همان دسته‌بندی تصویر تحلیل شده نیستند، شامل شود.

3. **یافتن آیتم‌های مطابق**: ما از تابع `find_matching_items_with_rag` برای یافتن آیتم‌هایی در مجموعه داده فیلتر شده که با توضیحات استخراج شده از تصویر تحلیل شده مطابقت دارند، استفاده می‌کنیم.

4. **نمایش آیتم‌های مطابق**: ما از یک HTML برای نمایش تصاویر آیتم‌های مطابق ایجاد می‌کنیم. مسیرهای تصویر را با استفاده از شناسه آیتم‌ها ساخته و هر تصویر را به رشته HTML اضافه می‌کنیم. در نهایت، از `display(HTML(html))` برای نمایش تصاویر در نوت‌بوک استفاده می‌کنیم.

کد زیر نشان می‌دهد که چگونه می‌توان از نتایج تحلیل تصویر برای فیلتر کردن یک مجموعه داده و نمایش بصری آیتم‌هایی که با ویژگی‌های تصویر تحلیل شده مطابقت دارند، استفاده کرد.
```python
# Extract the relevant features from the analysis
item_descs = image_analysis['items']
item_category = image_analysis['category']
item_gender = image_analysis['gender']

# Filter data such that we only look through the items of the same gender (or unisex) and different category
filtered_items = styles_df.loc[styles_df['gender'].isin([item_gender, 'Unisex'])]
filtered_items = filtered_items[filtered_items['articleType'] != item_category]
print(str(len(filtered_items)) + " Remaining Items")

# Find the most similar items based on the input item descriptions
matching_items = find_matching_items_with_rag(filtered_items, item_descs)

# Display the matching items (this will display 2 items for each description in the image analysis)
html = ""
paths = []
for i, item in enumerate(matching_items):
    item_id = item['id']
        
    # Path to the image file
    image_path = f'data/sample_clothes/sample_images/{item_id}.jpg'
    paths.append(image_path)
    html += f'<img src="{image_path}" style="display:inline;margin:1px"/>'

# Print the matching item description as a reminder of what we are looking for
print(item_descs)
# Display the image
display(HTML(html))
```

{{< ltr >}}
```
513 Remaining Items
["Slim Fit Blue Men's Jeans", "White Men's Sneakers", "Men's Silver Watch"]
```
{{< /ltr >}}

### Guardrail

در زمینه استفاده از مدل‌های زبان بزرگ (LLMs) مانند `GPT-4o`، "ریل‌های محافظ" یا Guardrail به مکانیزم‌ها یا چک‌هایی اشاره دارد که برای اطمینان از اینکه خروجی مدل در محدوده پارامترها یا مرزهای مطلوب باقی می‌ماند، قرار داده می‌شوند. این ریل‌های محافظ برای حفظ کیفیت و مرتبط بودن پاسخ‌های مدل بسیار مهم هستند، به ویژه هنگامی که با وظایف پیچیده یا ظریف سروکار داریم.

ریل‌های محافظ به دلایل مختلف مفید هستند:

1. **دقت**: آنها به اطمینان از اینکه خروجی مدل دقیق و مرتبط با ورودی ارائه شده است، کمک می‌کنند.
2. **ثبات**: آنها ثبات در پاسخ‌های مدل را حفظ می‌کنند، به ویژه هنگامی که با ورودی‌های مشابه یا مرتبط سروکار داریم.
3. **ایمنی**: آنها از تولید محتوای مضر، توهین‌آمیز یا نامناسب توسط مدل جلوگیری می‌کنند.
4. **ارتباط زمینه‌ای**: آنها اطمینان می‌دهند که خروجی مدل به طور زمینه‌ای مرتبط با وظیفه یا حوزه خاصی است که برای آن استفاده می‌شود.

در اینجا ما از `GPT-4o` برای تحلیل تصاویر مد و پیشنهاد آیتم‌هایی که با لباس اصلی مطابقت دارند، استفاده می‌کنیم. برای پیاده‌سازی ریل‌های محافظ، می‌توانیم **نتایج را اصلاح کنیم**: پس از دریافت پیشنهادات اولیه از `GPT-4o`، می‌توانیم تصویر اصلی و آیتم‌های پیشنهادی را به مدل بازگردانیم. سپس می‌توانیم از `GPT-4o` بخواهیم ارزیابی کند که آیا هر آیتم پیشنهادی واقعاً با لباس اصلی مطابقت دارد یا خیر.

این به مدل امکان می‌دهد تا خود را تصحیح کند و خروجی خود را بر اساس بازخورد یا اطلاعات اضافی تنظیم کند. با پیاده‌سازی این ریل‌های محافظ و امکان تصحیح خود، می‌توانیم قابلیت اطمینان و مفید بودن خروجی مدل را در زمینه تحلیل و توصیه مد افزایش دهیم.

برای تسهیل این کار، یک پراپمت می‌نویسیم که از مدل برای پاسخ ساده "بله" یا "خیر" به این سوال که آیا آیتم‌های پیشنهادی با لباس اصلی مطابقت دارند یا خیر، درخواست می‌کند. این پاسخ دودویی به ساده‌سازی فرآیند اصلاح کمک می‌کند و اطمینان می‌دهد که بازخورد واضح و قابل اجرایی از مدل دریافت می‌شود.

```python
def check_match(reference_image_base64, suggested_image_base64):
    response = client.chat.completions.create(
        model=GPT_MODEL,
        messages=[
            {
            "role": "user",
            "content": [
                {
                "type": "text",
                "text": """ You will be given two images of two different items of clothing.
                            Your goal is to decide if the items in the images would work in an outfit together.
                            The first image is the reference item (the item that the user is trying to match with another item).
                            You need to decide if the second item would work well with the reference item.
                            Your response must be a JSON output with the following fields: "answer", "reason".
                            The "answer" field must be either "yes" or "no", depending on whether you think the items would work well together.
                            The "reason" field must be a short explanation of your reasoning for your decision. Do not include the descriptions of the 2 images.
                            Do not include the ```json ``` tag in the output.
                           """,
                },
                {
                "type": "image_url",
                "image_url": {
                    "url": f"data:image/jpeg;base64,{reference_image_base64}",
                },
                },
                {
                "type": "image_url",
                "image_url": {
                    "url": f"data:image/jpeg;base64,{suggested_image_base64}",
                },
                }
            ],
            }
        ],
        max_tokens=300,
    )
    # Extract relevant features from the response
    features = response.choices[0].message.content
    return features
```

در نهایت، باید تعیین کنیم که کدام یک از آیتم‌های شناسایی شده واقعاً با لباس مطابقت دارند.

```python
# Select the unique paths for the generated images
paths = list(set(paths))

for path in paths:
    # Encode the test image to base64
    suggested_image = encode_image_to_base64(path)
    
    # Check if the items match
    match = json.loads(check_match(encoded_image, suggested_image))
    
    # Display the image and the analysis results
    if match["answer"] == 'yes':
        display(Image(filename=path))
        print("The items match!")
        print(match["reason"])
```

{{< ltr >}}
```
The items match!
The black shirt and blue jeans create a classic and casual outfit that works well together.
```

```
The items match!
The black shirt and the white sneakers with red and beige accents can work well together. Black is a versatile color that pairs well with many shoe options, and the white sneakers can add a sporty and casual touch to the outfit.
```

```
The items match!
The black button-up shirt is casual and versatile, making it compatible with the white and red athletic shoes for a relaxed and sporty look.
```

```
The items match!
The black shirt pairs well with the light blue jeans, creating a classic and balanced color combination that is casual and stylish.
```

```
The items match!
Both the black shirt and the black watch have a sleek and coordinated look, making them suitable to be worn together as part of an outfit.
```
{{< /ltr >}}

همانطور که مشاهده می‌کنید لیست اولیه آیتم‌ها فیلتر شده و منجر به انتخابی دقیق‌تر شده که با لباس مطابقت دارد. علاوه بر این، مدل توضیحاتی در مورد اینکه چرا هر آیتم به عنوان یک تطابق خوب در نظر گرفته شده است، ارائه می‌دهد که بینش‌های ارزشمندی در فرآیند تصمیم‌گیری ارائه می‌دهد.

### نتیجه‌گیری

در این بلاگ ما کاربرد `GPT-4o` و تکنیک‌های دیگر یادگیری ماشین را در حوزه مد بررسی کردیم. ما نشان دادیم که چگونه می‌توان تصاویر لباس‌ها را تحلیل کرد، ویژگی‌های مرتبط را استخراج کرد و از این اطلاعات برای یافتن آیتم‌های مطابق که با لباس اصلی مطابقت دارند، استفاده کرد. از طریق پیاده‌سازی ریل‌های محافظ و مکانیزم‌های تصحیح خود، پیشنهادات مدل را فیلتر کردیم تا اطمینان حاصل کنیم که دقیق و مرتبط با زمینه هستند.

این رویکرد چندین کاربرد عملی در دنیای واقعی دارد، از جمله:

1. **دستیارهای خرید شخصی‌سازی شده**: خرده‌فروشان می‌توانند از این تکنولوژی برای ارائه پیشنهادات لباس شخصی‌سازی شده به مشتریان استفاده کنند، تجربه خرید را بهبود بخشیده و رضایت مشتری را افزایش دهند.
2. **اپلیکیشن‌های کمد مجازی**: کاربران می‌توانند تصاویر لباس‌های خود را آپلود کرده و یک کمد مجازی ایجاد کنند و پیشنهاداتی برای آیتم‌های جدید که با لباس‌های موجود آنها مطابقت دارند، دریافت کنند.
3. **طراحی و استایل مد**: طراحان و استایلیست‌های مد می‌توانند از این ابزار برای آزمایش ترکیب‌ها و سبک‌های مختلف استفاده کنند و فرآیند خلاقانه را ساده‌تر کنند.

با این حال، یکی از ملاحظات مهم **هزینه** است. استفاده از مدل‌های زبان بزرگ و مدل‌های تحلیل تصویر می‌تواند هزینه‌هایی را به همراه داشته باشد، به ویژه اگر به طور گسترده استفاده شوند. مهم است که به صرفه‌جویی در هزینه‌های پیاده‌سازی این تکنولوژی‌ها توجه شود.

امیدواریم که این پست توانسته باشد جنبه‌های جدیدی از کاربردهای هوش مصنوعی را به شما معرفی کند.