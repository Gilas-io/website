---
title: "پردازش اسناد PDF برای برنامه‌های RAG"
description: "استفاده از `GPT-4V` برای تبدیل اسناد PDF به محتوای قابل استفاده در برنامه‌های RAG"
tags:
- RAG
weight: 
og_image: "/posts/parse_pdf_docs_for_rag/banner.png"
---

{{< postcover src="/posts/parse_pdf_docs_for_rag/banner.png" >}}

# پردازش اسناد PDF برای برنامه‌های RAG

این `Notebook` نشان می‌دهد چگونه می‌توان از `GPT-4V` برای تبدیل اسناد PDF مانند اسلایدها یا خروجی‌های صفحات وب به محتوای قابل استفاده برای برنامه‌های `RAG` استفاده کرد.

این تکنیک می‌تواند در صورتی که داده‌های غیرساختارمند زیادی دارید که حاوی اطلاعات ارزشمندی هستند و می‌خواهید به عنوان بخشی از پایپ‌لاین `RAG` خود آنها را بازیابی کنید، مورد استفاده قرار گیرد.

به عنوان مثال، می‌توانید یک `Knowledge Assistant` بسازید که بتواند به سوالات کاربران درباره شرکت یا محصول شما بر اساس اطلاعات موجود در اسناد PDF پاسخ دهد.

درین مثال ما از اسناد مربوط به `API`های `OpenAI` استفاده کرده‌ایم که شامل تکنیک‌های مختلفی هستند که می‌توانند به عنوان بخشی از پروژه‌های `LLM` استفاده شوند.

## آماده‌سازی داده‌ها

در این بخش، داده‌های ورودی خود را پردازش می‌کنیم تا برای بازیابی آماده شوند.

این کار را به دو روش انجام خواهیم داد:

1. استخراج متن با `pdfminer`
2. تبدیل صفحات PDF به تصاویر برای تحلیل آنها با `GPT-4V`

می‌توانید روش اول را نادیده بگیرید اگر می‌خواهید فقط از محتوای استنباط شده از تحلیل تصویر استفاده کنید.

### تنظیمات

باید چند کتابخانه نصب کنیم تا PDF را به تصاویر تبدیل کرده و متن را استخراج کنیم (اختیاری).

توجه: باید `poppler` را روی سیستم خود نصب کنید تا کتابخانه `pdf2image` کار کند. می‌توانید دستورالعمل‌های نصب آن را [اینجا](https://pypi.org/project/pdf2image/) دنبال کنید.

```python
%pip install pdf2image
%pip install pdfminer
%pip install openai
%pip install scikit-learn
%pip install rich
%pip install tqdm
%pip install concurrent
```
```python
# Imports
from pdf2image import convert_from_path
from pdf2image.exceptions import (
    PDFInfoNotInstalledError,
    PDFPageCountError,
    PDFSyntaxError
)
from pdfminer.high_level import extract_text
import base64
from io import BytesIO
import os
import concurrent
from tqdm import tqdm
from openai import OpenAI
import re
import pandas as pd 
from sklearn.metrics.pairwise import cosine_similarity
import json
import numpy as np
from rich import print
from ast import literal_eval
```

### پردازش فایل
```python
def convert_doc_to_images(path):
    images = convert_from_path(path)
    return images

def extract_text_from_doc(path):
    text = extract_text(path)
    page_text = []
    return text
```

#### تست با یک مثال

می‌توانید مسیر فایل زیر را به یک فایل `PDF` که بر روی کامپیوترتان قرار دارد تغییر دهید.

```python
file_path = "data/example_pdfs/fine-tuning-deck.pdf"

images = convert_doc_to_images(file_path)
```
```python
text = extract_text_from_doc(file_path)
```
```python
for img in images:
    display(img)
```

{{< scrollbox >}}

{{< img image="/posts/parse_pdf_docs_for_rag/p1.png" >}}

{{< img image="/posts/parse_pdf_docs_for_rag/p2.png" >}}

{{< img image="/posts/parse_pdf_docs_for_rag/p3.png" >}}

{{< img image="/posts/parse_pdf_docs_for_rag/p4.png" >}}

{{< img image="/posts/parse_pdf_docs_for_rag/p5.png" >}}

{{< img image="/posts/parse_pdf_docs_for_rag/p6.png" >}}

{{< img image="/posts/parse_pdf_docs_for_rag/p7.png" >}}

{{< /scrollbox >}}

### تحلیل تصویر با `GPT-4V`

پس از تبدیل یک فایل PDF به چندین تصویر، از `GPT-4V` برای تحلیل محتوا بر اساس تصاویر استفاده خواهیم کرد.

{{< hint info >}}
**برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.**
{{< /hint >}} 

```python
# imports
from openai import OpenAI # for calling the OpenAI API
import os

client = OpenAI(
    api_key=os.environ.get(("GILAS_API_KEY", "<کلید API خود را اینجا بسازید https://dashboard.gilas.io/apiKey>")), 
    base_url="https://api.gilas.io/v1/" # Gilas APIs
)


# Converting images to base64 encoded images in a data URI format to use with the ChatCompletions API
def get_img_uri(img):
    buffer = BytesIO()
    img.save(buffer, format="jpeg")
    base64_image = base64.b64encode(buffer.getvalue()).decode("utf-8")
    data_uri = f"data:image/jpeg;base64,{base64_image}"
    return data_uri
```
```python
system_prompt = '''
You will be provided with an image of a pdf page or a slide. Your goal is to talk about the content that you see, in technical terms, as if you were delivering a presentation.

If there are diagrams, describe the diagrams and explain their meaning.
For example: if there is a diagram describing a process flow, say something like "the process flow starts with X then we have Y and Z..."

If there are tables, describe logically the content in the tables
For example: if there is a table listing items and prices, say something like "the prices are the following: A for X, B for Y..."

DO NOT include terms referring to the content format
DO NOT mention the content type - DO focus on the content itself
For example: if there is a diagram/chart and text on the image, talk about both without mentioning that one is a chart and the other is text.
Simply describe what you see in the diagram and what you understand from the text.

You should keep it concise, but keep in mind your audience cannot see the image so be exhaustive in describing the content.

Exclude elements that are not relevant to the content:
DO NOT mention page numbers or the position of the elements on the image.

------

If there is an identifiable title, identify the title to give the output in the following format:

{TITLE}

{Content description}

If there is no clear title, simply return the content description.

'''

def analyze_image(img_url):
    response = client.chat.completions.create(
    model="gpt-4-vision-preview",
    temperature=0,
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
        }
    ],
        max_tokens=300,
        top_p=0.1
    )

    return response.choices[0].message.content
```

#### تست با یک مثال

حال می‌خواهیم محتوای متنی یکی از صفحات را نشان دهیم.

```python
img = images[2]
data_uri = get_img_uri(img)


res = analyze_image(data_uri)
print(res)
```
```
What is Fine-tuning

Fine-tuning a model consists of training the model to follow a set of given input/output examples. This will teach the model to behave in a certain way when confronted with a similar input in the future.

We recommend using 50-100 examples even if the minimum is 10.

The process involves starting with a public model, using training data to train the model, and resulting in a fine-tuned model.

```

#### پردازش همه اسناد

حال که از درست بودن کد خود مطمین شدیم تمام فایل های موجود در پوشه را برای پردازش شدن استفاده می‌کنیم.

```python
files_path = "data/example_pdfs"

all_items = os.listdir(files_path)
files = [item for item in all_items if os.path.isfile(os.path.join(files_path, item))]

def analyze_doc_image(img):
    img_uri = get_img_uri(img)
    data = analyze_image(img_uri)
    return data
```

ما همه فایل‌ها را در پوشه نمونه فهرست کرده و آنها را پردازش خواهیم کرد:
1. استخراج متن
2. تبدیل اسناد به تصاویر
3. تحلیل صفحات با `GPT-4V`

توجه: این کار حدود ~2 دقیقه طول می‌کشد. می‌توانید این مرحله را نادیده بگیرید و مستقیماً فایل نتیجه را بارگذاری کنید (به زیر مراجعه کنید).
```python
docs = []

for f in files:
    
    path = f"{files_path}/{f}"
    doc = {
        "filename": f
    }
    text = extract_text_from_doc(path)
    doc['text'] = text
    imgs = convert_doc_to_images(path)
    pages_description = []
    
    print(f"Analyzing pages for doc {f}")
    
    # Concurrent execution
    with concurrent.futures.ThreadPoolExecutor(max_workers=8) as executor:
        
        # Removing 1st slide as it's usually just an intro
        futures = [
            executor.submit(analyze_doc_image, img)
            for img in imgs[1:]
        ]
        
        with tqdm(total=len(imgs)-1) as pbar:
            for _ in concurrent.futures.as_completed(futures):
                pbar.update(1)
        
        for f in futures:
            res = f.result()
            pages_description.append(res)
        
    doc['pages_description'] = pages_description
    docs.append(doc)
```
```
Analyzing pages for doc rag-deck.pdf
100%|██████████████████████████████████████████████████████████████████| 19/19 [00:32<00:00,  1.72s/it]
Analyzing pages for doc models-page.pdf
100%|████████████████████████████████████████████████████████████████████| 9/9 [00:25<00:00,  2.80s/it]
Analyzing pages for doc evals-decks.pdf
100%|██████████████████████████████████████████████████████████████████| 12/12 [00:29<00:00,  2.44s/it]
Analyzing pages for doc fine-tuning-deck.pdf
100%|████████████████████████████████████████████████████████████████████| 6/6 [00:19<00:00,  3.32s/it]
```

```python
# Saving result to file for later
json_path = "data/parsed_pdf_docs.json"

with open(json_path, 'w') as f:
    json.dump(docs, f)
```
```python
# Optional: load content from the saved file
with open(json_path, 'r') as f:
    docs = json.load(f)
```

### امبدینگ محتوا
قبل از  `embedding` محتوا، محتوای صفحات را از هم جدا می‌کنیم و برای هر صفحه یک بردار امبدینگ جداگانه می‌سازیم.
برای سناریوهای واقعی، می‌توانید روش‌های پیشرفته‌تری برای تقسیم محتوا بررسی کنید.

```python
# Chunking content by page and merging together slides text & description if applicable
content = []
for doc in docs:
    # Removing first slide as well
    text = doc['text'].split('\f')[1:]
    description = doc['pages_description']
    description_indexes = []
    for i in range(len(text)):
        slide_content = text[i] + '\n'
        # Trying to find matching slide description
        slide_title = text[i].split('\n')[0]
        for j in range(len(description)):
            description_title = description[j].split('\n')[0]
            if slide_title.lower() == description_title.lower():
                slide_content += description[j].replace(description_title, '')
                # Keeping track of the descriptions added
                description_indexes.append(j)
        # Adding the slide content + matching slide description to the content pieces
        content.append(slide_content) 
    # Adding the slides descriptions that weren't used
    for j in range(len(description)):
        if j not in description_indexes:
            content.append(description[j])
```
```python
for c in content:
    print(c)
    print("\n\n-------------------------------\n\n")
```
```python
# Cleaning up content
# Removing trailing spaces, additional line breaks, page numbers and references to the content being a slide
clean_content = []
for c in content:
    text = c.replace(' \n', '').replace('\n\n', '\n').replace('\n\n\n', '\n').strip()
    text = re.sub(r"(?<=\n)\d{1,2}", "", text)
    text = re.sub(r"\b(?:the|this)\s*slide\s*\w+\b", "", text, flags=re.IGNORECASE)
    clean_content.append(text)
```
```python
for c in clean_content:
    print(c)
    print("\n\n-------------------------------\n\n")
```
```python
# Creating the embeddings
# We'll save to a csv file here for testing purposes but this is where you should load content in your vectorDB.
df = pd.DataFrame(clean_content, columns=['content'])
print(df.shape)
df.head()
```
```
(64, 1)
```

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>content</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Overview\nRetrieval-Augmented Generationenhanc...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>What is RAG\nRetrieve information to Augment t...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>When to use RAG\nGood for  ✅\nNot good for  ❌\...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Technical patterns\nData preparation\nInput pr...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Technical patterns\nData preparation\nchunk do...</td>
    </tr>
  </tbody>
</table>

شروع ساخت بردار امبدینگ:


```python
embeddings_model = "text-embedding-3-large"

def get_embeddings(text):
    embeddings = client.embeddings.create(
      model="text-embedding-3-small",
      input=text,
      encoding_format="float"
    )
    return embeddings.data[0].embedding
```
```python
df['embeddings'] = df['content'].apply(lambda x: get_embeddings(x))
df.head()
```

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>content</th>
      <th>embeddings</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Overview\nRetrieval-Augmented Generationenhanc...</td>
      <td>[-0.014744381, 0.03017278, 0.06353764, 0.02110...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>What is RAG\nRetrieve information to Augment t...</td>
      <td>[-0.024337867, 0.022921458, -0.00971687, 0.010...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>When to use RAG\nGood for  ✅\nNot good for  ❌\...</td>
      <td>[-0.011084231, 0.021158217, -0.00430421, 0.017...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Technical patterns\nData preparation\nInput pr...</td>
      <td>[-0.0058343858, 0.0408407, 0.054318383, 0.0190...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Technical patterns\nData preparation\nchunk do...</td>
      <td>[-0.010359385, 0.03736894, 0.052995477, 0.0180...</td>
    </tr>
  </tbody>
</table>

```python
# Saving locally for later
data_path = "data/parsed_pdf_docs_with_embeddings.csv"
df.to_csv(data_path, index=False)
```
```python
# Optional: load data from saved file
df = pd.read_csv(data_path)
df["embeddings"] = df.embeddings.apply(literal_eval).apply(np.array)
```

## تولید با استفاده از بازیابی

آخرین مرحله از فرآیند، تولید خروجی‌ها در پاسخ به پرسش‌های ورودی است، پس از بازیابی محتوا به عنوان زمینه برای پاسخ.
```python
system_prompt = '''
    You will be provided with an input prompt and content as context that can be used to reply to the prompt.
    
    You will do 2 things:
    
    1. First, you will internally assess whether the content provided is relevant to reply to the input prompt. 
    
    2a. If that is the case, answer directly using this content. If the content is relevant, use elements found in the content to craft a reply to the input prompt.

    2b. If the content is not relevant, use your own knowledge to reply or say that you don't know how to respond if your knowledge is not sufficient to answer.
    
    Stay concise with your answer, replying specifically to the input prompt without mentioning additional information provided in the context content.
'''

model="gpt-4-turbo-preview"

def search_content(df, input_text, top_k):
    embedded_value = get_embeddings(input_text)
    df["similarity"] = df.embeddings.apply(lambda x: cosine_similarity(np.array(x).reshape(1,-1), np.array(embedded_value).reshape(1, -1)))
    res = df.sort_values('similarity', ascending=False).head(top_k)
    return res

def get_similarity(row):
    similarity_score = row['similarity']
    if isinstance(similarity_score, np.ndarray):
        similarity_score = similarity_score[0][0]
    return similarity_score

def generate_output(input_prompt, similar_content, threshold = 0.5):
    
    content = similar_content.iloc[0]['content']
    
    # Adding more matching content if the similarity is above threshold
    if len(similar_content) > 1:
        for i, row in similar_content.iterrows():
            similarity_score = get_similarity(row)
            if similarity_score > threshold:
                content += f"\n\n{row['content']}"
            
    prompt = f"INPUT PROMPT:\n{input_prompt}\n-------\nCONTENT:\n{content}"
    
    completion = client.chat.completions.create(
        model=model,
        temperature=0.5,
        messages=[
            {
                "role": "system",
                "content": system_prompt
            },
            {
                "role": "user",
                "content": prompt
            }
        ]
    )

    return completion.choices[0].message.content
```
```python
# Example user queries related to the content
example_inputs = [
    'What are the main models you offer?',
    'Do you have a speech recognition model?',
    'Which embedding model should I use for non-English use cases?',
    'Can I introduce new knowledge in my LLM app using RAG?',
    'How many examples do I need to fine-tune a model?',
    'Which metric can I use to evaluate a summarization task?',
    'Give me a detailed example for an evaluation process where we are looking for a clear answer to compare to a ground truth.',
]
```
```python
# Running the RAG pipeline on each example
for ex in example_inputs:
    print(f"[deep_pink4][bold]QUERY:[/bold] {ex}[/deep_pink4]\n\n")
    matching_content = search_content(df, ex, 3)
    print(f"[grey37][b]Matching content:[/b][/grey37]\n")
    for i, match in matching_content.iterrows():
        print(f"[grey37][i]Similarity: {get_similarity(match):.2f}[/i][/grey37]")
        print(f"[grey37]{match['content'][:100]}{'...' if len(match['content']) > 100 else ''}[/[grey37]]\n\n")
    reply = generate_output(ex, matching_content)
    print(f"[turquoise4][b]REPLY:[/b][/turquoise4]\n\n[spring_green4]{reply}[/spring_green4]\n\n--------------\n\n")
```

در زیر لیست کامل و طولانی سوال و جواب ها را مشاهده می‌کنید. هر سوال با استفاده از اطلاعات بازیابی شده از فایل های `PDF` توسط مدل پاسخ داده شده اند.

```
QUERY: What are the main models you offer?

Matching content:
Similarity:  0.43
Models - OpenAI API
The content lists various API endpoints and their corresponding latest models:

Similarity:  0.39

26/02/2024, 17:58
Models - OpenAI API
The Moderation models are designed to check whether content co...

Similarity:  0.39
The content describes various models provided by OpenAI, focusing on moderation models and GPT base ...

REPLY:
The main models we offer include:
- For completions: gpt-3.5-turbo-instruct, babbage-002, and davinci-002.
- For embeddings: text-embedding-3-small, text-embedding-3-large, and text-embedding-ada-002.
- For fine-tuning jobs: gpt-3.5-turbo, babbage-002, and davinci-002.
- For moderations: text-moderation-stable and text-moderation.
Additionally, we have the latest models like gpt-3.5-turbo-16k and fine-tuned versions of gpt-3.5-turbo.

-----------------------------------------------------------------------------

QUERY: Do you have a speech recognition model?

Matching content:
Similarity:  0.53
The content describes various models related to text-to-speech, speech recognition, embeddings, and ...

Similarity:  0.50

26/02/2024, 17:58
Models - OpenAI API
MODEL
DE S CRIPTION
tts-1
New  Text-to-speech 1
The latest tex...

Similarity:  0.44
Technical patterns
Data preparation: augmenting content
What does “Augmenting content” mean?
Augmenti...

REPLY:
Yes, the Whisper model is a general-purpose speech recognition model mentioned in the content, capable of multilingual speech recognition, speech translation, and language identification. The v2-large model, referred to as "whisper-1", is available through an API and is optimized for faster performance.

-----------------------------------------------------------------------------

QUERY: Which embedding model should I use for non-English use cases?

Matching content:
Similarity:  0.57
The content describes various models related to text-to-speech, speech recognition, embeddings, and ...

Similarity:  0.46

26/02/2024, 17:58
Models - OpenAI API
Multilingual capabilities
GPT-4 outperforms both previous larg...

REPLY:
For non-English use cases, you should use the "V3 large" embedding model, as it is described as the most capable for both English and non-English tasks, with an output dimension of 3,072.

-----------------------------------------------------------------------------

QUERY: Can I introduce new knowledge in my LLM app using RAG?

Matching content:
Similarity:  0.50
What is RAG
Retrieve information to Augment the model’s knowledge and Generate the output
“What is y...

Similarity:  0.49
When to use RAG
Good for  ✅
Not good for  ❌
●
●
Introducing new information to the model
●
Teaching ...

REPLY:
Yes, you can introduce new knowledge in your LLM app using RAG by retrieving information from a knowledge base or external sources to augment the model's knowledge and generate outputs relevant to the queries posed.

```

## جمع‌بندی

در این `Notebook`، یاد گرفتیم چگونه یک پایپ‌لاین `RAG` ساده بر اساس اسناد PDF توسعه دهیم. این شامل موارد زیر است:

- چگونه اسناد PDF را پردازش کنیم، با استفاده از اسلایدها و خروجی از یک صفحه HTML به عنوان مثال، با استفاده از یک کتابخانه پایتون و همچنین `GPT-4V` برای تفسیر تصاویر
- چگونه محتوای استخراج شده را پردازش کنیم، تمیز کنیم و به چندین قطعه تقسیم کنیم
- چگونه محتوای پردازش شده را با استفاده از `Gilas API` امبد کنیم
- چگونه محتوای مرتبط با یک پرسش ورودی را بازیابی کنیم
- چگونه با استفاده از `GPT-4-turbo` پاسخی با استفاده از محتوای بازیابی شده به عنوان زمینه تولید کنیم

می‌توانید تکنیک‌های پوشش داده شده در این `Notebook` را به موارد استفاده مختلف اعمال کنید، مانند دستیارانی که می‌توانند به داده‌های اختصاصی شما دسترسی داشتهباشند، ربات‌های خدمات مشتری یا FAQ که می‌توانند از سیاست‌های داخلی شما بخوانند، یا هر چیزی که نیاز به استفاده از اسناد غنی دارد که به عنوان تصاویر بهتر درک می‌شوند.

