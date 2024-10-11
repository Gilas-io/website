---
title: "ساخت چت‌ بات با استفاده از LangChain"
description: "این راهنما نحوه طراحی و پیاده‌سازی یک چت‌بات با استفاده از مدل‌های زبانی را توضیح می‌دهد و شامل مراحل نصب، مدیریت تاریخچه مکالمات و استفاده از `LangGraph` برای حفظ پیام‌ها است."
tags:
- chatbot
- langchain
weight: 2001 
og_image: "/posts/build_a_chatbot_with_langchain/banner.jpg"
---

{{< postcover src="/posts/build_a_chatbot_with_langchain/banner.jpg" >}}

در این پست یک مثال از نحوه طراحی و پیاده‌سازی یک چت‌بات با استفاده از یک `LLM` را بررسی خواهیم کرد. این چت‌بات قادر به انجام مکالمه و به خاطر سپاری تعاملات قبلی است. در نظر داشته باشید که برای تولید مکالمات پیچیده تر میتوانید از قابلیت های عامل (`Agent`) و `RAG` که از طریق پکیج `LangChain` قابل دسترس هستند استفاده کنید.

## آماده سازی محیط

برای این آموزش به `langchain-core` و `langgraph` نیاز خواهیم داشت:

```bash
pip install langchain langchain-core langgraph>0.2.27 langchain_openai
```

یا با استفاده از `Conda`:

```bash
conda install langchain langchain-core langgraph>0.2.27 langchain_openai -c conda-forge
```

برای جزئیات بیشتر، به [مستندات راهنمای نصب](https://python.langchain.com/docs/how_to/installation/) مراجعه کنید.

### LangSmith

بسیاری از برنامه‌هایی که با `LangChain` می‌سازید شامل مراحل متعددی با فراخوانی‌های متعدد `LLM` هستند. با پیچیده‌تر شدن این برنامه‌ها، توانایی بررسی دقیق آنچه در زنجیره یا عامل (`Agent`) شما اتفاق می‌افتد بسیار مهم می‌شود. بهترین راه برای انجام این کار استفاده از [LangSmith](https://smith.langchain.com) است.

پس از ثبت‌نام در لینک بالا، مطمئن شوید که متغیرهای محیطی خود را برای شروع `Tracing` تنظیم کرده‌اید:

```shell
export LANGCHAIN_TRACING_V2="true"
export LANGCHAIN_API_KEY="..."
```

یا، اگر در یک نوت‌بوک هستید، می‌توانید آنها را به این صورت تنظیم کنید:

```python
import os

os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "..."
```

## پیاده‌سازی

ابتدا می‌خواهیم یاد بگیریم که چگونه از یک مدل زبانی به تنهایی برای پیاده سازی چت‌بات استفاده کنیم. `LangChain` از مدل‌های زبانی مختلفی پشتیبانی می‌کند که می‌توانید به صورت متناوب از آنها استفاده کنید - مدل مورد نظر خود را در زیر انتخاب کنید!


{{< hint info >}}
**برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.**
{{< /hint >}} 

```python
from langchain_openai import ChatOpenAI

model = ChatOpenAI(
    api_key="GILAS_API_KEY",
    base_url="https://api.gilas.io/v1/",
    model="gpt-4o-mini")
```

بیایید ابتدا مدل را مستقیماً استفاده کنیم. `ChatModel`ها نمونه‌هایی از "Runnables" `LangChain` هستند، به این معنی که یک رابط استاندارد برای تعامل با آنها ارائه می‌دهند. برای فراخوانی ساده مدل، می‌توانیم لیستی از پیام‌ها را به متود `.invoke` ارسال کنیم.

```python
from langchain_core.messages import HumanMessage

model.invoke([HumanMessage(content="Hi! I'm Bob")])
```

```bash
AIMessage(content='Hi Bob! How can I assist you today?', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 10, 'prompt_tokens': 11, 'total_tokens': 21, 'completion_tokens_details': {'reasoning_tokens': 0}}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_1bb46167f9', 'finish_reason': 'stop', 'logprobs': None}, id='run-149994c0-d958-49bb-9a9d-df911baea29f-0', usage_metadata={'input_tokens': 11, 'output_tokens': 10, 'total_tokens': 21})
```

مدل به تنهایی هیچ درکی از `state` ندارد یا به عبارت دیگر هر درخواست به مدل به صورت `state-less` پردازش می‌شود. به عنوان مثال، اگر یک سوال پیگیری بپرسید:

```python
model.invoke([HumanMessage(content="What's my name?")])
```

```bash
AIMessage(content="I'm sorry, but I don't have access to personal information about individuals unless you've shared it with me in this conversation. How can I assist you today?", additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 30, 'prompt_tokens': 11, 'total_tokens': 41, 'completion_tokens_details': {'reasoning_tokens': 0}}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_1bb46167f9', 'finish_reason': 'stop', 'logprobs': None}, id='run-0ecab57c-728d-4fd1-845c-394a62df8e13-0', usage_metadata={'input_tokens': 11, 'output_tokens': 30, 'total_tokens': 41})

```
می‌توانیم ببینیم که چت بات پیغام‌های قبلی را در نظر نمی‌گیرد و نمی‌تواند به سوال پاسخ دهد.
برای حل این مشکل، باید کل تاریخچه مکالمه را به مدل منتقل کنیم. بیایید ببینیم وقتی این کار را انجام می‌دهیم چه اتفاقی می‌افتد:

```python
from langchain_core.messages import AIMessage

model.invoke(
    [
        HumanMessage(content="Hi! I'm Bob"),
        AIMessage(content="Hello Bob! How can I assist you today?"),
        HumanMessage(content="What's my name?"),
    ]
)
```

```bash
AIMessage(content='Your name is Bob! How can I help you today?', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 12, 'prompt_tokens': 33, 'total_tokens': 45, 'completion_tokens_details': {'reasoning_tokens': 0}}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_1bb46167f9', 'finish_reason': 'stop', 'logprobs': None}, id='run-c164c5a1-d85f-46ee-ba8a-bb511cfb0e51-0', usage_metadata={'input_tokens': 33, 'output_tokens': 12, 'total_tokens': 45})
```
و اکنون می‌توانیم ببینیم که پاسخ خوبی دریافت می‌کنیم! حال سوال این است که چگونه می‌توانیم انتقال مکالمات قبلی به چت بات را به بهترین شکل پیاده‌سازی کنیم؟

## نگهداری پیام

پکیج [LangGraph](https://langchain-ai.github.io/langgraph/) یک لایه حافظه داخلی پیاده‌سازی می‌کند که آن را برای برنامه‌های چت که باید مکالمات قبلی را به خاطر بسپارند، ایده‌آل می‌کند.
استفاده از `LangGraph` به ما امکان می‌دهد تا تاریخچه پیام‌ها را به طور خودکار حفظ کنیم که باعث تسهیل توسعه چت بات هامی‌شود.

پکیج `LangGraph` با یک چک‌پوینتر ساده در حافظه ارائه می‌شود که در زیر از آن استفاده می‌کنیم. برای جزئیات بیشتر، از جمله نحوه استفاده از بک‌اندهای مختلف (مانند `SQLite` یا `Postgres`) به [مستندات آن](https://langchain-ai.github.io/langgraph/concepts/persistence/) مراجعه کنید.

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import START, MessagesState, StateGraph

# Define a new graph
workflow = StateGraph(state_schema=MessagesState)


# Define the function that calls the model
def call_model(state: MessagesState):
    response = model.invoke(state["messages"])
    return {"messages": response}


# Define the (single) node in the graph
workflow.add_edge(START, "model")
workflow.add_node("model", call_model)

# Add memory
memory = MemorySaver()
app = workflow.compile(checkpointer=memory)
```

اکنون باید یک `config` ایجاد کنیم که هر بار به `runnable` منتقل شود. این پیکربندی شامل اطلاعاتی است که مستقیماً بخشی از ورودی نیست، اما همچنان مفید است. در این مورد، می‌خواهیم یک `thread_id` را تعریف کنیم که به صورت زیر است:

```python
config = {"configurable": {"thread_id": "abc123"}}
```

این کار به چت بات این امکان را می‌دهد تا چندین مکالمه‌ی همزمان را به طور جداگانه مدیریت کند.

سپس می‌توانیم برنامه را فراخوانی کنیم:

```python
query = "Hi! I'm Bob."

input_messages = [HumanMessage(query)]
output = app.invoke({"messages": input_messages}, config)
output["messages"][-1].pretty_print()  # output contains all messages in state
```

```shell
Ai Message:

Hi Bob! How can I assist you today?
```

```python
query = "What's my name?"

input_messages = [HumanMessage(query)]
output = app.invoke({"messages": input_messages}, config)
output["messages"][-1].pretty_print()
```

```shell
Ai Message:

Your name is Bob! How can I help you today?
```

عالی! چت‌بات ما اکنون چیزهایی درباره ما به خاطر می‌آورد. اگر پیکربندی را تغییر دهیم تا به یک `thread_id` متفاوت اشاره کند، می‌توانیم ببینیم که مکالمه را تازه شروع می‌کند.

```python
config = {"configurable": {"thread_id": "abc234"}}

input_messages = [HumanMessage(query)]
output = app.invoke({"messages": input_messages}, config)
output["messages"][-1].pretty_print()
```

```shell
Ai Message:

I'm sorry, but I don't have access to personal information about you unless you provide it. How can I assist you today?
```

با این حال، چون مکالمه‌ی قبلی را در حافظ نگه داشته ایم٬ همیشه می‌توانیم به آن برگردیم.

```python
config = {"configurable": {"thread_id": "abc123"}}

input_messages = [HumanMessage(query)]
output = app.invoke({"messages": input_messages}, config)
output["messages"][-1].pretty_print()
```

```shell
Ai Message:

Your name is Bob! If there's anything else you'd like to discuss or ask, feel free!
```

پس به این طریق می‌توانیم چت‌باتی داشته باشیم که با بسیاری از کاربران مکالمه دارد و تاریخچه‌ی هر مکالمه را به طور جداگانه به خاطر دارد!

برای پشتیبانی از `async`، تابع `call_model` را تبدیل به یک تابع `async` کنید و هنگام فراخوانی برنامه از `.ainvoke` استفاده کنید:

```python
# Async function for node:
async def call_model(state: MessagesState):
    response = await model.ainvoke(state["messages"])
    return {"messages": response}


# Define graph as before:
workflow = StateGraph(state_schema=MessagesState)
workflow.add_edge(START, "model")
workflow.add_node("model", call_model)
app = workflow.compile(checkpointer=MemorySaver())

# Async invocation:
output = await app.ainvoke({"messages": input_messages}, config)
output["messages"][-1].pretty_print()
```

در حال حاضر، تمام کاری که انجام داده‌ایم اضافه کردن یک لایه حافظه‌ی ساده به مدل است. حال می‌توانیم با اضافه کردن یک قالب پراپمت، مکالمات را پیچیده‌تر و شخصی‌تر کنیم.

## قالب‌های پراپمت

قالب‌های پراپمت به تبدیل اطلاعات خام کاربر به فرمتی که `LLM` می‌تواند با آن کار کند کمک می‌کنند. در حال حاضر ورودی خام کاربر فقط یک پیام است که ما به `LLM` ارسال می‌کنیم. بیایید اکنون آن را کمی پیچیده‌تر کنیم. ابتدا، می‌خواهیم یک پیام سیستمی با دستورالعمل‌های سفارشی را به مدل اضافه کنیم (اما همچنان پیام‌ها را به عنوان ورودی بپذیریم). سپس، ورودی‌های بیشتری را در کنار پیام‌ها به مدل خواهیم افزود.

برای اضافه کردن یک پیام سیستمی، یک `ChatPromptTemplate` ایجاد خواهیم کرد و از `MessagesPlaceholder` برای ارسال همه پیام‌ها استفاده می‌کنیم.

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "You talk like a pirate. Answer all questions to the best of your ability.",
        ),
        MessagesPlaceholder(variable_name="messages"),
    ]
)
```

اکنون می‌توانیم برنامه خود را به‌روزرسانی کنیم تا از این قالب استفاده کند:

```python
workflow = StateGraph(state_schema=MessagesState)


def call_model(state: MessagesState):
    # change-start
    chain = prompt | model
    response = chain.invoke(state)
    # change-end
    return {"messages": response}


workflow.add_edge(START, "model")
workflow.add_node("model", call_model)

memory = MemorySaver()
app = workflow.compile(checkpointer=memory)
```

حال برنامه را به همان روش قبلی فراخوانی می‌کنیم:

```python
config = {"configurable": {"thread_id": "abc345"}}
query = "Hi! I'm Jim."

input_messages = [HumanMessage(query)]
output = app.invoke({"messages": input_messages}, config)
output["messages"][-1].pretty_print()
```

```shell
Ai Message:

Ahoy there, Jim! What brings ye to these treacherous waters today? Be ye seekin’ treasure, tales, or perhaps a bit o’ knowledge? Speak up, matey!
```

```python
query = "What is my name?"

input_messages = [HumanMessage(query)]
output = app.invoke({"messages": input_messages}, config)
output["messages"][-1].pretty_print()
```

```
Ai Message:

Ye be callin' yerself Jim, if I be hearin' ye correctly! A fine name for a scallywag such as yerself! What else can I do fer ye, me hearty?
```

عالی! بیایید اکنون پراپمت خود را کمی پیچیده‌تر کنیم. فرض کنیم که قالب پراپمت اکنون به این شکل است:

```python
prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "You are a helpful assistant. Answer all questions to the best of your ability in {language}.",
        ),
        MessagesPlaceholder(variable_name="messages"),
    ]
)
```

توجه داشته باشید که یک ورودی جدید `language` به پراپمت اضافه کرده‌ایم. برنامه ما اکنون دو پارامتر دارد - ورودی `messages` و `language`. باید وضعیت برنامه خود را به‌روزرسانی کنیم تا این را منعکس کند:

```python
from typing import Sequence

from langchain_core.messages import BaseMessage
from langgraph.graph.message import add_messages
from typing_extensions import Annotated, TypedDict


class State(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    language: str


workflow = StateGraph(state_schema=State)


def call_model(state: State):
    chain = prompt | model
    response = chain.invoke(state)
    return {"messages": [response]}


workflow.add_edge(START, "model")
workflow.add_node("model", call_model)

memory = MemorySaver()
app = workflow.compile(checkpointer=memory)
```

```python
config = {"configurable": {"thread_id": "abc456"}}
query = "Hi! I'm Bob."
language = "Farsi"

input_messages = [HumanMessage(query)]
output = app.invoke(
    {"messages": input_messages, "language": language},
    config,
)
output["messages"][-1].pretty_print()
```

```shell
Ai Message:

سلام باب! چه کمکی میتونم بهت بکنم؟
```

برای کمک به درک آنچه در داخل اتفاق می‌افتد، به [این LangSmith trace](https://smith.langchain.com/public/15bd8589-005c-4812-b9b9-23e74ba4c3c6/r) نگاهی بیندازید.

## مدیریت تاریخچه مکالمه

یکی از مفاهیم مهم در ساخت چت‌بات‌ها، نحوه مدیریت تاریخچه مکالمه است. اگر تاریخچه‌ی مکالمات مدیریت نشود، لیست پیام‌ها به طور نامحدود رشد می‌کند و ممکن است پنجره زمینه (`context window`) `LLM` را پر کند. بنابراین، مهم است که مرحله‌ای اضافه کنید که اندازه پیام‌هایی که ارسال می‌کنید را محدود کند.

**مهم است که این کار را قبل از ساخت قالب پراپمت و بعد از بارگذاری پیام‌های قبلی از تاریخچه پیام انجام دهید.**

می‌توانیم این کار را با اضافه کردن یک مرحله ساده در جلوی پراپمت انجام دهیم که کلید `messages` را  تغییر دهد و سپس زنجیره جدید را در کلاس تاریخچه پیام `wrap` کنیم.

پکیج `LangChain` با چندین تابع کمکی داخلی برای [مدیریت لیست پیام‌ها](/docs/how_to/#messages) ارائه می‌شود. در این مورد، از تابع [trim_messages](/docs/how_to/trim_messages/) برای کاهش تعداد پیام‌هایی که به مدل ارسال می‌کنیم، استفاده خواهیم کرد. این ابزار به ما اجازه می‌دهد تا مشخص کنیم که چند توکن از حافظه را می‌خواهیم نگه داریم. همچنین پارامترهای دیگری مانند اینکه آیا می‌خواهیم پیام سیستمی را همیشه نگه داریم و آیا اجازه پیام‌های جزئی را بدهیم یا خیر نیز قابل تنظیم هستند.

```python
from langchain_core.messages import SystemMessage, trim_messages

trimmer = trim_messages(
    max_tokens=65,
    strategy="last",
    token_counter=model,
    include_system=True,
    allow_partial=False,
    start_on="human",
)

messages = [
    SystemMessage(content="you're a good assistant"),
    HumanMessage(content="hi! I'm bob"),
    AIMessage(content="hi!"),
    HumanMessage(content="I like vanilla ice cream"),
    AIMessage(content="nice"),
    HumanMessage(content="whats 2 + 2"),
    AIMessage(content="4"),
    HumanMessage(content="thanks"),
    AIMessage(content="no problem!"),
    HumanMessage(content="having fun?"),
    AIMessage(content="yes!"),
]

trimmer.invoke(messages)
```

برای استفاده از آن در زنجیره خود، فقط باید قبل از ارسال ورودی `messages` به پراپمت، از `trimmer` استفاده کنیم.

```python
workflow = StateGraph(state_schema=State)


def call_model(state: State):
    chain = prompt | model
    # change-start
    trimmed_messages = trimmer.invoke(state["messages"])
    response = chain.invoke(
        {"messages": trimmed_messages, "language": state["language"]}
    )
    # change-end
    return {"messages": [response]}


workflow.add_edge(START, "model")
workflow.add_node("model", call_model)

memory = MemorySaver()
app = workflow.compile(checkpointer=memory)
```

اکنون اگر از مدل نام خود را بپرسیم جواب را نمی‌داند زیرا آن بخش از تاریخچه چت را برش داده‌ایم:

```python
config = {"configurable": {"thread_id": "abc567"}}
query = "What is my name?"
language = "English"

input_messages = messages + [HumanMessage(query)]
output = app.invoke(
    {"messages": input_messages, "language": language},
    config,
)
output["messages"][-1].pretty_print()
```

```shell
Ai Message:

I don't know your name. If you'd like to share it, feel free!
```

اما اگر درباره اطلاعاتی که در چند پیام آخر بین ما و مدل رد و بدل شده است سوالی  بپرسیم، مدل آن را به خاطر می‌آورد:

```python
config = {"configurable": {"thread_id": "abc678"}}
query = "What math problem did I ask?"
language = "English"

input_messages = messages + [HumanMessage(query)]
output = app.invoke(
    {"messages": input_messages, "language": language},
    config,
)
output["messages"][-1].pretty_print()
```

```shell
Ai Message:

You asked what 2 + 2 equals.
```

اگر به [LangSmith trace](https://smith.langchain.com/public/04402eaa-29e6-4bb1-aa91-885b730b6c21/r) نگاهی بیندازید، می‌توانید دقیقاً ببینید که در پشت پرده چه اتفاقی می‌افتد.

## استریم

اکنون یک چت‌بات کاربردی داریم. با این حال، یکی از ملاحظات بسیار مهم UX برای برنامه‌های چت‌بات، استریم است.  گاهی اوقات ممکن است مدتی طول بکشد تا `LLM`ها پاسخ را تولید کنند، بنابراین برای بهبود تجربه کاربر، یکی از کارهایی که اکثر برنامه‌ها انجام می‌دهند این است که هر توکن را به محض تولید استریم کنند. این به کاربر اجازه می‌دهد تا تولید پاسخ را سریعتر ببیند.


به طور پیش‌فرض، `.stream` در برنامه `LangGraph` ما مراحل برنامه را استریم می‌کند. تنظیم `stream_mode="messages"` به ما اجازه می‌دهد تا توکن‌های خروجی را به جای آن استریم کنیم:

```python
config = {"configurable": {"thread_id": "abc789"}}
query = "Hi I'm Todd, please tell me a joke."
language = "English"

input_messages = [HumanMessage(query)]
for chunk, metadata in app.stream(
    {"messages": input_messages, "language": language},
    config,
    stream_mode="messages",
):
    if isinstance(chunk, AIMessage):  # Filter to just model responses
        print(chunk.content, end="|")
```

```shell
|Hi| Todd|!| Here|’s| a| joke| for| you|:

|Why| did| the| scare|crow| win| an| award|?

|Because| he| was| outstanding| in| his| field|!||
```