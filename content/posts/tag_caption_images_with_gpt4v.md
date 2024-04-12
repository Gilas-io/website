---
title: "تگ زدن تصاویر و تولید کپشن برای آنها"
description: "این Notebook توضیح می‌دهد که چگونه می‌توان از GPT-4-Vision برای برچسب زدن و توضیح تصاویر بهره برد. ما می‌توانیم از توانایی‌های GPT-4V استفاده کنیم تا تصاویر ورودی را همراه با اطلاعات تکمیلی در مورد آنها پردازش کند و برچسب‌ها یا توضیحات مربوط به را خروجی دهد. سپس می‌توان توضیحات تصویر را با استفاده از یک مدل زبانی (در این نوت‌بوک، ما از GPT-4-turbo استفاده خواهیم کرد) برای تولید توضیحات بیشتر اصلاح کرد."
tags:
- openai
- gpt-4-turbo
- image-processing
- gilas.io
- blog
weight: 2000
og_image: "/posts/tag_caption_images_with_gpt4v/banner.png" 
---

{{< postcover src="/posts/tag_caption_images_with_gpt4v/banner.png" >}}


## تولید خودکار برچسب  برای تصاویر و توضیح محتوای آنها با استفاده از GPT-4-Vision 


این Notebook توضیح می‌دهد که چگونه می‌توان از GPT-4-Vision برای برچسب زدن و توضیح تصاویر بهره برد. ما می‌توانیم از توانایی‌های GPT-4V استفاده کنیم تا تصاویر ورودی را همراه با اطلاعات تکمیلی در مورد آنها پردازش کند و برچسب‌ها یا توضیحات مربوط به را خروجی دهد. سپس می‌توان توضیحات تصویر را با استفاده از یک مدل زبانی (در این نوت‌بوک، ما از GPT-4-turbo استفاده خواهیم کرد) برای تولید توضیحات بیشتر اصلاح کرد.

 تولید محتوای متنی از تصاویر می‌تواند در موارد متنوعی استفاده شود، به خصوص برای جستجوی میان تصاویر. در این  ‌Notebook جستجوی میان تصاویر را با استفاده از کلمات کلیدی تولید شده و توضیحات محصول - هم از ورودی متن و هم از ورودی تصویر - نشان خواهیم داد. به عنوان مثال، ما از مجموعه‌ای از تصاویر مبلمان استفاده خواهیم کرد، آنها را با کلمات کلیدی مربوطه برچسب می‌زنیم و توضیحات کوتاه و توصیفی تولید می‌کنیم. 
  

{{< hint info >}}
**برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.**
{{< /hint >}} 

 Setup

 ```python
 # Install dependencies if needed
%pip install openai
%pip install scikit-learn
 ```

 ```python
from IPython.display import Image, display
import pandas as pd
from sklearn.metrics.pairwise import cosine_similarity
import numpy as np
from openai import OpenAI

# Initializing OpenAI client
client = OpenAI(
    api_key=os.environ.get(("GILAS_API_KEY", "<کلید API خود را اینجا بسازید https://dashboard.gilas.io/apiKey>")), 
    base_url="https://api.gilas.io/v1/" # Gilas APIs
)

# Loading dataset
dataset_path =  "data/amazon_furniture_dataset.csv"
df = pd.read_csv(dataset_path)
df.head()
 ```

## برچسب گذاری تصاویر


 در این بخش، ما از GPT-4V استفاده خواهیم کرد تا برچسب های مرتبط برای محصولات ایجاد کنیم. برای این کار از روش  zero-shot برای استخراج کلیدواژه ها استفاده می‌کنیم، و با استفاده از [embeddings](/models/#openai-embeddings) این کلیدواژه ها را deduplicate خواهیم کرد تا از داشتن چندین کلیدواژه که خیلی شبیه یکدیگر هستند جلوگیری کنیم. برای جلوگیری از تولید برچسب برای دیگر آیتم‌های موجود در عکس مثل لوازم دکوری داخل تصویر٬ از ترکیب تصاویر و عنوان محصولات استفاده خواهیم کرد تا از استخراج کلیدواژه‌های غیر مرتبط با موضوع اصلی تصویر جلوگیری کنیم.

 {{< hint warning >}} 
توجه: ورودی‌های داده شده و خروجی‌های تولید شده توسط مدل در این مثال به زبان انگلیسی هستند. برای تولید خروجی به زبان فارسی٬ کافی‌ست از مدل بخواهید که خروجی را به زبان فارسی تولید کند.
{{< /hint >}}

```python
system_prompt = '''
    You are an agent specialized in tagging images of furniture items, decorative items, or furnishings with relevant keywords that could be used to search for these items on a marketplace.
    
    You will be provided with an image and the title of the item that is depicted in the image, and your goal is to extract keywords for only the item specified. 
    
    Keywords should be concise and in lower case. 
    
    Keywords can describe things like:
    - Item type e.g. 'sofa bed', 'chair', 'desk', 'plant'
    - Item material e.g. 'wood', 'metal', 'fabric'
    - Item style e.g. 'scandinavian', 'vintage', 'industrial'
    - Item color e.g. 'red', 'blue', 'white'
    
    Only deduce material, style or color keywords when it is obvious that they make the item depicted in the image stand out.

    Return keywords in the format of an array of strings, like this:
    ['desk', 'industrial', 'metal']
    
'''

def analyze_image(img_url, title):
    response = client.chat.completions.create(
    model="gpt-4-turbo",
    messages=[
        {
            "role": "system",
            "content": system_prompt
        },
        {
            "role": "user",
            "content": [
                {
                    "type": "image_url",
                    "image_url": img_url,
                },
            ],
        },
        {
            "role": "user",
            "content": title
        }
    ],
        max_tokens=300,
        top_p=0.1
    )

    return response.choices[0].message.content
```

تست کردن نتیجه با چند تصویر مختلف

```python
examples = df.iloc[:5]
```

```python
for index, ex in examples.iterrows():
    url = ex['primary_image']
    img = Image(url=url)
    display(img)
    result = analyze_image(url, ex['title'])
    print(result)
    print("\n\n")
```

نتیجه:


<img src="/posts/tag_caption_images_with_gpt4v/416WaLx10jL._SS522_.jpg" alt="screenshot" style="width: 434px;"/>

```
['shoe rack', 'free standing', 'multi-layer', 'metal', 'white']
```

<img src="/posts/tag_caption_images_with_gpt4v/31SejUEWY7L._SS522_.jpg" alt="screenshot" style="width: 434px;"/>

```
['dining chairs', 'set of 2', 'leather', 'black']
```

<img src="/posts/tag_caption_images_with_gpt4v/41RgefVq70L._SS522_.jpg" alt="screenshot" style="width: 434px;"/>

```
['plant repotting mat', 'waterproof', 'portable', 'foldable', 'easy to clean', 'green']
```

### پیدا کردن برچسب‌های مشابه

برای پیدا و حذف کردن برچسب‌های تکراری از [embeddings](/models/#openai-embeddings) استفاده می‌کنیم. embeddings روشی برای نمایش برداری متون است که معنا و مفهوم متن را شامل می‌شود.

```python
# Feel free to change the embedding model here
def get_embedding(value, model="text-embedding-3-large"): 
    embeddings = client.embeddings.create(
      model=model,
      input=value,
      encoding_format="float"
    )
    return embeddings.data[0].embedding
```

```python
# Existing keywords
keywords_list = ['industrial', 'metal', 'wood', 'vintage', 'bed']
```

```python
df_keywords = pd.DataFrame(keywords_list, columns=['keyword'])
df_keywords['embedding'] = df_keywords['keyword'].apply(lambda x: get_embedding(x))
df_keywords
```

نمونه ای از نمایش برداری یک کلمه:

```
industrial	[-0.026137426, 0.021297162, -0.007273361, -0.0...]
```

```python
def compare_keyword(keyword):
    embedded_value = get_embedding(keyword)
    df_keywords['similarity'] = df_keywords['embedding'].apply(lambda x: cosine_similarity(np.array(x).reshape(1,-1), np.array(embedded_value).reshape(1, -1)))
    most_similar = df_keywords.sort_values('similarity', ascending=False).iloc[0]
    return most_similar

def replace_keyword(keyword, threshold = 0.6):
    most_similar = compare_keyword(keyword)
    if most_similar['similarity'] > threshold:
        print(f"Replacing '{keyword}' with existing keyword: '{most_similar['keyword']}'")
        return most_similar['keyword']
    return keyword
```

```python
# Example keywords to compare to our list of existing keywords
example_keywords = ['bed frame', 'wooden', 'vintage', 'old school', 'desk', 'table', 'old', 'metal', 'metallic', 'woody']
final_keywords = []

for k in example_keywords:
    final_keywords.append(replace_keyword(k))
    
final_keywords = set(final_keywords)
print(f"Final keywords: {final_keywords}")
```

همانطور که در پایین مشاهده می‌کنید با استفاده از [embeddings](/models/#openai-embeddings) توانستیم برچسب‌های مشابه را شناسایی کنیم:

```
Replacing 'bed frame' with existing keyword: 'bed'
Replacing 'wooden' with existing keyword: 'wood'
Replacing 'vintage' with existing keyword: 'vintage'
Replacing 'metal' with existing keyword: 'metal'
Replacing 'metallic' with existing keyword: 'metal'
Replacing 'woody' with existing keyword: 'wood'
Final keywords: {'table', 'desk', 'bed', 'old', 'vintage', 'metal', 'wood', 'old school'}
```

## توضیح محتوای داخل یک عکس

در این بخش، از GPT-4V برای تولید توضیحات تصویر استفاده خواهیم کرد و سپس از روش few-shot examples با GPT-4-turbo برای تولید عنوان برای هر تصویر استفاده خواهیم کرد.
اگر روش زیر برای دیتابیس تصاویر شما به درستی عمل نمی‌کند می‌توانید مدل را برای تصاویر خود fine-tune کنید.

```python
# Cleaning up dataset columns
selected_columns = ['title', 'primary_image', 'style', 'material', 'color', 'url']
df = df[selected_columns].copy()
```

حال از مدل GPT-4V برای توضیح محتوای تصاویر استفاده می‌کنیم:

```python
describe_system_prompt = '''
    You are a system generating descriptions for furniture items, decorative items, or furnishings on an e-commerce website.
    Provided with an image and a title, you will describe the main item that you see in the image, giving details but staying concise.
    You can describe unambiguously what the item is and its material, color, and style if clearly identifiable.
    If there are multiple items depicted, refer to the title to understand which item you should describe.
    '''

def describe_image(img_url, title):
    response = client.chat.completions.create(
    temperature=0.2,
    messages=[
        {
            "role": "system",
            "content": describe_system_prompt
        },
        {
            "role": "user",
            "content": [
                {
                    "type": "image_url",
                    "image_url": img_url,
                },
            ],
        },
        {
            "role": "user",
            "content": title
        }
    ],
    max_tokens=300,
    )

    return response.choices[0].message.content
```

نمونه‌هایی از خروجی مدل روی تصاویر مختلف:

```python
for index, row in examples.iterrows():
    print(f"{row['title'][:50]}{'...' if len(row['title']) > 50 else ''} - {row['url']} :\n")
    img_description = describe_image(row['primary_image'], row['title'])
    print(f"{img_description}\n--------------------------\n")
```

```
GOYMFK 1pc Free Standing Shoe Rack, Multi-layer Me... - https://www.amazon.com/dp/B0CJHKVG6P :

This is a free-standing shoe rack featuring a multi-layer design, constructed from metal for durability. The rack is finished in a clean white color, which gives it a modern and versatile look, suitable for various home decor styles. It includes several horizontal shelves dedicated to organizing shoes, providing ample space for multiple pairs.
--------------------------

subrtex Leather ding Room, Dining Chairs Set of 2,... - https://www.amazon.com/dp/B0B66QHB23 :

This image showcases a set of two dining chairs. The chairs are upholstered in black leather, featuring a sleek and modern design. They have a high back with subtle stitching details that create vertical lines, adding an element of elegance to the overall appearance. The legs of the chairs are also black, maintaining a consistent color scheme and enhancing the sophisticated look. These chairs would make a stylish addition to any contemporary dining room setting.
```

حال می‌توانیم از توضیحات تولید شده برای تولید caption برای هر تصویر استفاده کنیم:

```python
caption_system_prompt = '''
Your goal is to generate short, descriptive captions for images of furniture items, decorative items, or furnishings based on an image description.
You will be provided with a description of an item image and you will output a caption that captures the most important information about the item.
Your generated caption should be short (1 sentence), and include the most relevant information about the item.
The most important information could be: the type of the item, the style (if mentioned), the material if especially relevant and any distinctive features.
'''

few_shot_examples = [
    {
        "description": "This is a multi-layer metal shoe rack featuring a free-standing design. It has a clean, white finish that gives it a modern and versatile look, suitable for various home decors. The rack includes several horizontal shelves dedicated to organizing shoes, providing ample space for multiple pairs. Above the shoe storage area, there are 8 double hooks arranged in two rows, offering additional functionality for hanging items such as hats, scarves, or bags. The overall structure is sleek and space-saving, making it an ideal choice for placement in living rooms, bathrooms, hallways, or entryways where efficient use of space is essential.",
        "caption": "White metal free-standing shoe rack"
    },
    {
        "description": "The image shows a set of two dining chairs in black. These chairs are upholstered in a leather-like material, giving them a sleek and sophisticated appearance. The design features straight lines with a slight curve at the top of the high backrest, which adds a touch of elegance. The chairs have a simple, vertical stitching detail on the backrest, providing a subtle decorative element. The legs are also black, creating a uniform look that would complement a contemporary dining room setting. The chairs appear to be designed for comfort and style, suitable for both casual and formal dining environments.",
        "caption": "Set of 2 modern black leather dining chairs"
    },
    {
        "description": "This is a square plant repotting mat designed for indoor gardening tasks such as transplanting and changing soil for plants. It measures 26.8 inches by 26.8 inches and is made from a waterproof material, which appears to be a durable, easy-to-clean fabric in a vibrant green color. The edges of the mat are raised with integrated corner loops, likely to keep soil and water contained during gardening activities. The mat is foldable, enhancing its portability, and can be used as a protective surface for various gardening projects, including working with succulents. It's a practical accessory for garden enthusiasts and makes for a thoughtful gift for those who enjoy indoor plant care.",
        "caption": "Waterproof square plant repotting mat"
    }
]

formatted_examples = [[{
    "role": "user",
    "content": ex['description']
},
{
    "role": "assistant", 
    "content": ex['caption']
}]
    for ex in few_shot_examples
]

formatted_examples = [i for ex in formatted_examples for i in ex]
```

```python
def caption_image(description, model="gpt-4-turbo"):
    messages = formatted_examples
    messages.insert(0, 
        {
            "role": "system",
            "content": caption_system_prompt
        })
    messages.append(
        {
            "role": "user",
            "content": description
        })
    response = client.chat.completions.create(
    model=model,
    temperature=0.2,
    messages=messages
    )

    return response.choices[0].message.content
```

نمونه‌هایی از خروجی مدل روی تصاویر مختلف:

```python
examples = df.iloc[5:8]
for index, row in examples.iterrows():
    print(f"{row['title'][:50]}{'...' if len(row['title']) > 50 else ''} - {row['url']} :\n")
    img_description = describe_image(row['primary_image'], row['title'])
    print(f"{img_description}\n--------------------------\n")
    img_caption = caption_image(img_description)
    print(f"{img_caption}\n--------------------------\n")
```

```
LOVMOR 30'' Bathroom Vanity Sink Base Cabine, Stor... - https://www.amazon.com/dp/B0C9WYYFLB:

Image description:
This is a LOVMOR 30'' Bathroom Vanity Sink Base Cabinet featuring a classic design with a rich brown finish. The cabinet is designed to provide ample storage with three drawers on the left side, offering organized space for bathroom essentials...
--------------------------
Short caption:
LOVMOR 30'' classic brown bathroom vanity base cabinet with three drawers and additional storage space.
--------------------------

Folews Bathroom Organizer Over The Toilet Storage,... - https://www.amazon.com/dp/B09NZY3R1T :

Image description:
This is a 4-tier bathroom organizer designed to fit over a standard toilet, providing a space-saving storage solution. The unit is constructed with a sturdy metal frame in a black finish, which offers both durability and a sleek, modern look....
--------------------------
Short caption:
Modern 4-tier black metal bathroom organizer with adjustable shelves and baskets, designed to fit over a standard toilet for space-saving storage.
--------------------------
```

## جستجو میان تصاویر

در این مرحله از کلمات کلیدی و عناوین تولید شده برای جستجوی میان تصاویر بر اساس متن یا تصویر استفاده خواهیم کرد.

ما از مدل [embeddings](/models/#openai-embeddings) خود به منظور تولید بردارها برای کلمات کلیدی و عناوین استفاده خواهیم کرد و آن‌ها را با متن ورودی یا عنوان تولید شده از یک تصویر ورودی مقایسه خواهیم کرد. توجه داشته باشید که اگر ورودی یک تصویر باشد ابتدا با استفاده از روش‌های بالا آن تصویر را به صورت متنی توصیف می‌کنیم و سپس بر اساس آن توصیف دیتابیس را برای پیدا کردن تصاویر مشابه جستجو می‌کنیم.

```python
# Df we'll use to compare keywords
df_keywords = pd.DataFrame(columns=['keyword', 'embedding'])
df['keywords'] = ''
df['img_description'] = ''
df['caption'] = ''
```

```python
# Function to replace a keyword with an existing keyword if it's too similar
def get_keyword(keyword, df_keywords, threshold = 0.6):
    embedded_value = get_embedding(keyword)
    df_keywords['similarity'] = df_keywords['embedding'].apply(lambda x: cosine_similarity(np.array(x).reshape(1,-1), np.array(embedded_value).reshape(1, -1)))
    sorted_keywords = df_keywords.copy().sort_values('similarity', ascending=False)
    if len(sorted_keywords) > 0 :
        most_similar = sorted_keywords.iloc[0]
        if most_similar['similarity'] > threshold:
            print(f"Replacing '{keyword}' with existing keyword: '{most_similar['keyword']}'")
            return most_similar['keyword']
    new_keyword = {
        'keyword': keyword,
        'embedding': embedded_value
    }
    df_keywords = pd.concat([df_keywords, pd.DataFrame([new_keyword])], ignore_index=True)
    return keyword
```

## آماده کردن دیتاست

```python
import ast

def tag_and_caption(row):
    keywords = analyze_image(row['primary_image'], row['title'])
    try:
        keywords = ast.literal_eval(keywords)
        mapped_keywords = [get_keyword(k, df_keywords) for k in keywords]
    except Exception as e:
        print(f"Error parsing keywords: {keywords}")
        mapped_keywords = []
    img_description = describe_image(row['primary_image'], row['title'])
    caption = caption_image(img_description)
    return {
        'keywords': mapped_keywords,
        'img_description': img_description,
        'caption': caption
    }
```

```python
df.shape

(312, 9)
```

پردازش همهٔ ۳۱۲ خط مجموعه داده مدتی زمان می‌برد. برای آزمایش این ایده، فقط آن را بر روی اولین ۵۰ خط اجرا خواهیم کرد که تقریباً ۲۰ دقیقه زمان می‌برد. اگر تمایل دارید این مرحله را انجام ندهید و مجموعه داده از پیش پردازش شده را بارگذاری کنید جدول پایین را مشاهده کنید.

{{< scrollbox >}}
    <table style="width: 100%; border-collapse: collapse;" >
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>title</th>
      <th>primary_image</th>
      <th>style</th>
      <th>material</th>
      <th>color</th>
      <th>url</th>
      <th>keywords</th>
      <th>img_description</th>
      <th>caption</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>GOYMFK 1pc Free Standing Shoe Rack, Multi-laye...</td>
      <td>https://m.media-amazon.com/images/I/416WaLx10j...</td>
      <td>Modern</td>
      <td>Metal</td>
      <td>White</td>
      <td>https://www.amazon.com/dp/B0CJHKVG6P</td>
      <td>[shoe rack, free standing, multi-layer, metal,...</td>
      <td>This is a free-standing shoe rack featuring a ...</td>
      <td>White metal free-standing shoe rack with multi...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>subrtex Leather ding Room, Dining Chairs Set o...</td>
      <td>https://m.media-amazon.com/images/I/31SejUEWY7...</td>
      <td>Black Rubber Wood</td>
      <td>Sponge</td>
      <td>Black</td>
      <td>https://www.amazon.com/dp/B0B66QHB23</td>
      <td>[dining chairs, set of 2, leather, black]</td>
      <td>This image features a set of two black dining ...</td>
      <td>Set of 2 sleek black faux leather dining chair...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Plant Repotting Mat MUYETOL Waterproof Transpl...</td>
      <td>https://m.media-amazon.com/images/I/41RgefVq70...</td>
      <td>Modern</td>
      <td>Polyethylene</td>
      <td>Green</td>
      <td>https://www.amazon.com/dp/B0BXRTWLYK</td>
      <td>[plant repotting mat, waterproof, portable, fo...</td>
      <td>This is a square plant repotting mat designed ...</td>
      <td>Waterproof green square plant repotting mat</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Pickleball Doormat, Welcome Doormat Absorbent ...</td>
      <td>https://m.media-amazon.com/images/I/61vz1Igler...</td>
      <td>Modern</td>
      <td>Rubber</td>
      <td>A5589</td>
      <td>https://www.amazon.com/dp/B0C1MRB2M8</td>
      <td>[doormat, absorbent, non-slip, brown]</td>
      <td>This is a rectangular doormat featuring a play...</td>
      <td>Pickleball-themed coir doormat with playful de...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>JOIN IRON Foldable TV Trays for Eating Set of ...</td>
      <td>https://m.media-amazon.com/images/I/41p4d4VJnN...</td>
      <td>X Classic Style</td>
      <td>Iron</td>
      <td>Grey Set of 4</td>
      <td>https://www.amazon.com/dp/B0CG1N9QRC</td>
      <td>[tv tray table set, foldable, iron, grey]</td>
      <td>This image showcases a set of two foldable TV ...</td>
      <td>Set of two foldable TV trays with grey wood gr...</td>
    </tr>
  </tbody>
    </table>
{{< /scrollbox >}}

```python
# Saving locally for later
data_path = "data/items_tagged_and_captioned.csv"
df.to_csv(data_path, index=False)

# Optional: load data from saved file
df = pd.read_csv(data_path)
```

### تولید embeddings برای برچسب‌ها و کپشن تصاویر

اکنون می‌توانیم از عناوین و کلمات کلیدی تولید شده برای همسان‌سازی محتوای مرتبط برای جستجوی متن ورودی یا عنوان استفاده کنیم. برای انجام این کار، یک ترکیب از کلمات کلیدی + عناوین را تهیه خواهیم کرد. 

 ایجاد embeddings حدود ۳ دقیقه زمان می‌برد. اگر تمایل دارید، 
می‌توانید مجموعه داده از پیش پردازش شده را بارگذاری کنید.

```python
df_search = df.copy()

def embed_tags_caption(x):
    if x['caption'] != '':
        keywords_string = ",".join(k for k in x['keywords']) + '\n'
        content = keywords_string + x['caption']
        embedding = get_embedding(content)
        return embedding

df_search['embedding'] = df_search.apply(lambda x: embed_tags_caption(x), axis=1)

df_search.head()
```

{{< scrollbox >}}
    <table style="width: 100%; border-collapse: collapse;" >
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>title</th>
      <th>primary_image</th>
      <th>style</th>
      <th>material</th>
      <th>color</th>
      <th>url</th>
      <th>keywords</th>
      <th>img_description</th>
      <th>caption</th>
      <th>embedding</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>GOYMFK 1pc Free Standing Shoe Rack, Multi-laye...</td>
      <td>https://m.media-amazon.com/images/I/416WaLx10j...</td>
      <td>Modern</td>
      <td>Metal</td>
      <td>White</td>
      <td>https://www.amazon.com/dp/B0CJHKVG6P</td>
      <td>['shoe rack', 'free standing', 'multi-layer', ...</td>
      <td>This is a free-standing shoe rack featuring a ...</td>
      <td>White metal free-standing shoe rack with multi...</td>
      <td>[-0.06596625, -0.026769113, -0.013789515, -0.0...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>subrtex Leather ding Room, Dining Chairs Set o...</td>
      <td>https://m.media-amazon.com/images/I/31SejUEWY7...</td>
      <td>Black Rubber Wood</td>
      <td>Sponge</td>
      <td>Black</td>
      <td>https://www.amazon.com/dp/B0B66QHB23</td>
      <td>['dining chairs', 'set of 2', 'leather', 'black']</td>
      <td>This image features a set of two black dining ...</td>
      <td>Set of 2 sleek black faux leather dining chair...</td>
      <td>[-0.0077859573, -0.010376813, -0.01928079, -0....</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Plant Repotting Mat MUYETOL Waterproof Transpl...</td>
      <td>https://m.media-amazon.com/images/I/41RgefVq70...</td>
      <td>Modern</td>
      <td>Polyethylene</td>
      <td>Green</td>
      <td>https://www.amazon.com/dp/B0BXRTWLYK</td>
      <td>['plant repotting mat', 'waterproof', 'portabl...</td>
      <td>This is a square plant repotting mat designed ...</td>
      <td>Waterproof green square plant repotting mat</td>
      <td>[-0.023248248, 0.005370147, -0.0048999498, -0....</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Pickleball Doormat, Welcome Doormat Absorbent ...</td>
      <td>https://m.media-amazon.com/images/I/61vz1Igler...</td>
      <td>Modern</td>
      <td>Rubber</td>
      <td>A5589</td>
      <td>https://www.amazon.com/dp/B0C1MRB2M8</td>
      <td>['doormat', 'absorbent', 'non-slip', 'brown']</td>
      <td>This is a rectangular doormat featuring a play...</td>
      <td>Pickleball-themed coir doormat with playful de...</td>
      <td>[-0.028953036, -0.026369056, -0.011363288, 0.0...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>JOIN IRON Foldable TV Trays for Eating Set of ...</td>
      <td>https://m.media-amazon.com/images/I/41p4d4VJnN...</td>
      <td>X Classic Style</td>
      <td>Iron</td>
      <td>Grey Set of 4</td>
      <td>https://www.amazon.com/dp/B0CG1N9QRC</td>
      <td>['tv tray table set', 'foldable', 'iron', 'grey']</td>
      <td>This image showcases a set of two foldable TV ...</td>
      <td>Set of two foldable TV trays with grey wood gr...</td>
      <td>[-0.030723095, -0.0051356032, -0.027088132, 0....</td>
    </tr>
  </tbody>
    </table>
{{< /scrollbox >}}


```python
# Keep only the lines where we have embeddings
df_search = df_search.dropna(subset=['embedding'])
print(df_search.shape)

(49, 10)
```

```python
# Saving locally for later
data_embeddings_path = "data/items_tagged_and_captioned_embeddings.csv"
df_search.to_csv(data_embeddings_path, index=False)

# Optional: load data from saved file
from ast import literal_eval
df_search = pd.read_csv(data_embeddings_path)
df_search["embedding"] = df_search.embedding.apply(literal_eval).apply(np.array)
```

### جستجوی بر اساس متن ورودی

برای جستجوی میان تصاویر بر اساس متن ورودی می‌توانیم [embeddings](/models/#openai-embeddings) منن ورودی را تولید کرده و آن را با embeddings تولید شده برای تصاویر که در مراحل قبل تولید کرده‌ایم مقایسه کنیم.

```python
# Searching for N most similar results
def search_from_input_text(query, n = 2):
    embedded_value = get_embedding(query)
    df_search['similarity'] = df_search['embedding'].apply(lambda x: cosine_similarity(np.array(x).reshape(1,-1), np.array(embedded_value).reshape(1, -1)))
    most_similar = df_search.sort_values('similarity', ascending=False).iloc[:n]
    return most_similar

user_inputs = ['shoe storage', 'black metal side table', 'doormat', 'step bookshelf', 'ottoman']

for i in user_inputs:
    print(f"Input: {i}\n")
    res = search_from_input_text(i)
    for index, row in res.iterrows():
        similarity_score = row['similarity']
        if isinstance(similarity_score, np.ndarray):
            similarity_score = similarity_score[0][0]
        print(f"{row['title'][:50]}{'...' if len(row['title']) > 50 else ''} ({row['url']}) - Similarity: {similarity_score:.2f}")
        img = Image(url=row['primary_image'])
        display(img)
        print("\n\n")
```

```
#متن ورودی برای جستجو 
Input: shoe storage

# تصویر متناسب با ورودی
GOYMFK 1pc Free Standing Shoe Rack, Multi-layer Me... (https://www.amazon.com/dp/B0CJHKVG6P) - Similarity: 0.62
```

<img src="/posts/tag_caption_images_with_gpt4v/416WaLx10jL._SS522_.jpg" alt="screenshot" style="width: 434px;"/>


### جستجوی بر اساس تصویر ورودی

اگر ورودی یک تصویر باشد، می‌توانیم برای یافتن تصاویر مشابه ابتدا  برای تصویر ورودی عنوان و برچسب‌ تولید می‌کنیم٬ سپس برای آنها [embeddings](/models/#openai-embeddings) می‌سازیم و نهایتا بر اساس embeddings های تولید شده در میان دیتابیس جستجو می‌کنیم.

```python
# We'll take a mix of images: some we haven't seen and some that are already in the dataset
example_images = df.iloc[306:]['primary_image'].to_list() + df.iloc[5:10]['primary_image'].to_list()

for i in example_images:
    img_description = describe_image(i, '')
    caption = caption_image(img_description)
    img = Image(url=i)
    print('Input: \n')
    display(img)
    res = search_from_input_text(caption, 1).iloc[0]
    similarity_score = res['similarity']
    if isinstance(similarity_score, np.ndarray):
        similarity_score = similarity_score[0][0]
    print(f"{res['title'][:50]}{'...' if len(res['title']) > 50 else ''} ({res['url']}) - Similarity: {similarity_score:.2f}")
    img_res = Image(url=res['primary_image'])
    display(img_res)
    print("\n\n")
    
```

ورودی:

<img src="/posts/tag_caption_images_with_gpt4v/31dCSKQ14YL._SS522_.jpg" alt="screenshot" style="width: 434px;"/>

خروجی:

<img src="/posts/tag_caption_images_with_gpt4v/414jZL4tnaL._SS522_.jpg" alt="screenshot" style="width: 434px;"/>


## جمع بندی

در این نوت‌بوک، ما بررسی کردیم که چگونه می‌توانیم از قابلیت‌های  GPT-4V برای برچسب زدن و توضیح دادن تصاویر استفاده کنیم. با ارائه تصاویر همراه با اطلاعات مرتبط به مدل، توانستیم تگ‌ها و توضیحاتی تولید کنیم که می‌توانند با استفاده از یک مدل زبانی مانند GPT-4-turbo بیشتر پالایش شده و برای ایجاد توضیحات استفاده شوند. این فرآیند کاربردهای عملی مختلفی دارد، به ویژه در بهبود عملکرد جستجو.
