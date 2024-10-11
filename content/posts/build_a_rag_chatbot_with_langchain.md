---
title: "ساخت چت‌ بات مبتنی بر RAG با استفاده از LangChain"
description: "این راهنما نحوه طراحی و پیاده‌سازی یک چت‌بات با استفاده از مدل‌های زبانی را توضیح می‌دهد و شامل مراحل نصب، مدیریت تاریخچه مکالمات و استفاده از `LangGraph` برای حفظ پیام‌ها است."
tags:
- chatbot
- rag
- langchain
weight: 2003
og_image: "/posts/build_a_rag_chatbot_with_langchain/banner.jpg"
---

{{< postcover src="/posts/build_a_rag_chatbot_with_langchain/banner.jpg" >}}

در این پست یک مثال از نحوه طراحی و پیاده‌سازی یک چت‌بات با استفاده از یک `RAG` را بررسی خواهیم کرد. این چت‌بات قادر به انجام مکالمه و به خاطر سپاری تعاملات قبلی است. در این مثال می‌خواهیم از قابلیت های عامل (`Agent`) و `Chains` که از طریق پکیج `LangChain` قابل دسترس هستند استفاده کنیم.

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

## زنجیره یا chain

ابندا به کد زیر نگاهی بیاندازید:


{{< hint info >}}
**برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.**
{{< /hint >}} 


```python
from langchain_openai import ChatOpenAI
import bs4
from langchain.chains import create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain_community.document_loaders import WebBaseLoader
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_openai import OpenAIEmbeddings
from langchain_text_splitters import RecursiveCharacterTextSplitter


llm = ChatOpenAI(
    api_key="GILAS_API_KEY",
    base_url="https://api.gilas.io/v1/",
    model="gpt-4o-mini")

# 1. Load, chunk and index the contents of the blog to create a retriever.
loader = WebBaseLoader(
    web_paths=("https://lilianweng.github.io/posts/2023-06-23-agent/",),
    bs_kwargs=dict(
        parse_only=bs4.SoupStrainer(
            class_=("post-content", "post-title", "post-header")
        )
    ),
)
docs = loader.load()

text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
splits = text_splitter.split_documents(docs)
vectorstore = InMemoryVectorStore.from_documents(
    documents=splits, embedding=OpenAIEmbeddings()
)
retriever = vectorstore.as_retriever()


# 2. Incorporate the retriever into a question-answering chain.
system_prompt = (
    "You are an assistant for question-answering tasks. "
    "Use the following pieces of retrieved context to answer "
    "the question. If you don't know the answer, say that you "
    "don't know. Use three sentences maximum and keep the "
    "answer concise."
    "\n\n"
    "{context}"
)

prompt = ChatPromptTemplate.from_messages(
    [
        ("system", system_prompt),
        ("human", "{input}"),
    ]
)

question_answer_chain = create_stuff_documents_chain(llm, prompt)
rag_chain = create_retrieval_chain(retriever, question_answer_chain)
```

```python
response = rag_chain.invoke({"input": "What is Task Decomposition?"})
response["answer"]
```

```shell
"Task decomposition is the process of breaking down a complicated task into smaller, more manageable steps. Techniques like Chain of Thought (CoT) and Tree of Thoughts enhance this process by guiding models to think step by step and explore multiple reasoning possibilities. This approach helps in simplifying complex tasks and provides insight into the model's reasoning."
```

توجه داشته باشید که ما از `create_stuff_documents_chain` و `create_retrieval_chain` استفاده کرده‌ایم، به‌طوری که اجزای اصلی راه‌حل ما عبارتند از:

- `retriever`
- `prompt`
- `LLM`

این کار فرایند افزودن تاریخچه‌ی مکالمه را ساده‌تر می‌کند.

### افزودن تاریخچه‌ی مکالمه

زنجیره‌ای که ساخته‌ایم از ورودی پرسش به‌صورت مستقیم برای بازیابی `context` مرتبط استفاده می‌کند. اما در یک محیط مکالمه‌ای، ممکن است برای درک پرسش کاربر به پیشینه‌ی مکالمه‌ هم نیاز باشد. به‌عنوان مثال، این مکالمه را در نظر بگیرید:


    انسان: «Task Decomposition چیست؟»

    هوش مصنوعی: «`Task Decomposition` به معنای تجزیه وظایف پیچیده به مراحل کوچک‌تر و ساده‌تر است تا برای یک عامل یا مدل قابل مدیریت‌تر شود.»

    انسان: «روش‌های رایج برای انجام آن چیست؟»

برای پاسخ دادن به سوال دوم، سیستم ما باید بفهمد که "آن" به `Task Decomposition` اشاره دارد.

برای این کار باید دو مورد را در کد موجود به‌روزرسانی کنیم:

1. **پرامپت**: پرامپت را به‌روزرسانی کنیم تا از تاریخچه‌ی پیام‌ها به‌عنوان ورودی پشتیبانی کند.
2. **سوالات افزودن زمینه به**: اضافه کردن یک زیرزنجیره `sub-chain` که آخرین سوال کاربر را گرفته و آن را در زمینه‌ی تاریخچه‌ی چت بازنویسی می‌کند. می‌توان این کار را به‌سادگی به‌عنوان ساختن یک `retriever` آگاه به تاریخچه در نظر گرفت. در حالی که قبلاً داشتیم:
   
`query` -> `retriever`

حالا به این شکل تبدیل می‌شود:

`(query, conversation history)` -> `LLM` -> `rephrased query` -> `retriever`

### مرتبط‌سازی سوال

ابتدا باید یک `sub-chain` تعریف کنیم که پیام‌های تاریخی و آخرین سوال کاربر را بگیرد و سوال را در صورتی که به اطلاعاتی در تاریخچه اشاره دارد، بازنویسی کند.

ما از پرامپتی استفاده می‌کنیم که شامل یک متغیر به نام `MessagesPlaceholder` تحت نام "chat_history" است. این کار به ما امکان می‌دهد که لیستی از پیام‌ها را با استفاده از کلید ورودی `chat_history` به پرامپت ارسال کنیم و این پیام‌ها بعد از پیام سیستم و قبل از پیام انسانی که حاوی آخرین سوال است، وارد می‌شوند.

توجه داشته باشید که ما از تابع کمکی `create_history_aware_retriever` برای این مرحله استفاده می‌کنیم، که در حالتی را که `chat_history` خالی باشد را مدیریت می‌کند، و در غیر این صورت، به‌طور متوالی `prompt | llm | StrOutputParser() | retriever` را اعمال می‌کند.

تابع `create_history_aware_retriever` زنجیره‌ای را می‌سازد که کلیدهای ورودی `input` و `chat_history` را به‌عنوان ورودی می‌پذیرد و همان ساختار خروجی را مانند `retriever` دارد.

```python
from langchain.chains import create_history_aware_retriever
from langchain_core.prompts import MessagesPlaceholder

contextualize_q_system_prompt = (
    "Given a chat history and the latest user question "
    "which might reference context in the chat history, "
    "formulate a standalone question which can be understood "
    "without the chat history. Do NOT answer the question, "
    "just reformulate it if needed and otherwise return it as is."
)

contextualize_q_prompt = ChatPromptTemplate.from_messages(
    [
        ("system", contextualize_q_system_prompt),
        MessagesPlaceholder("chat_history"),
        ("human", "{input}"),
    ]
)
history_aware_retriever = create_history_aware_retriever(
    llm, retriever, contextualize_q_prompt
)
```

این زنجیره، بازنویسی سوال ورودی را به `retriever` اضافه می‌کند تا فرآیند بازیابی اطلاعات شامل زمینه مکالمه شود.

حالا می‌توانیم زنجیره کامل پرسش و پاسخ (QA) خود را بسازیم. این کار به سادگی با به‌روزرسانی `retriever` به `history_aware_retriever` جدید انجام می‌شود.

دوباره از `create_stuff_documents_chain` برای تولید یک `question_answer_chain` استفاده خواهیم کرد. این تابع ورودی‌هایی با کلیدهای `context`، `chat_history`، و `input` را می‌پذیرد و زمینه بازیابی شده را همراه با تاریخچه مکالمه و سوال ورودی می‌گیرد تا یک پاسخ تولید کند. 

ما در نهایت زنجیره `rag_chain` نهایی خود را با استفاده از `create_retrieval_chain` می‌سازیم. این زنجیره، `history_aware_retriever` و `question_answer_chain` را به صورت متوالی به کار می‌گیرد و خروجی‌های میانی مانند `retrieved context` را برای راحتی نگه می‌دارد. این تابع دارای کلیدهای ورودی `input` و `chat_history` است و شامل `input`، `chat_history`، `context`، و `answer` در خروجی خود می‌باشد.

```python
from langchain.chains import create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain

qa_prompt = ChatPromptTemplate.from_messages(
    [
        ("system", system_prompt),
        MessagesPlaceholder("chat_history"),
        ("human", "{input}"),
    ]
)


question_answer_chain = create_stuff_documents_chain(llm, qa_prompt)

rag_chain = create_retrieval_chain(history_aware_retriever, question_answer_chain)
```

در زیر یک سؤال و یک سؤال پیگیری مطرح می‌کنیم که نیاز به تولید زمینه دارد تا پاسخی منطقی بازگردانده شود. به دلیل اینکه زنجیره ما شامل یک ورودی `chat_history` است، کلاینت باید تاریخچه چت را مدیریت کند. ما می‌توانیم این کار را با افزودن پیام‌های ورودی و خروجی به یک لیست انجام دهیم:

```python
from langchain_core.messages import AIMessage, HumanMessage

chat_history = []

question = "What is Task Decomposition?"
ai_msg_1 = rag_chain.invoke({"input": question, "chat_history": chat_history})
chat_history.extend(
    [
        HumanMessage(content=question),
        AIMessage(content=ai_msg_1["answer"]),
    ]
)

second_question = "What are common ways of doing it?"
ai_msg_2 = rag_chain.invoke({"input": second_question, "chat_history": chat_history})

print(ai_msg_2["answer"])
```

```shell
Common ways of task decomposition include using simple prompting techniques, such as asking for "Steps for XYZ" or "What are the subgoals for achieving XYZ?" Additionally, task-specific instructions can be employed, like "Write a story outline" for writing tasks, or human inputs can guide the decomposition process.
```

ما منطق برنامه‌ای برای ترکیب تاریخچه چت اضافه کرده‌ایم، اما هنوز هم به‌صورت دستی آن را در برنامه خود می‌گنجانیم. در محیط تولید، برنامه سوال و جواب معمولاً تاریخچه چت را در یک پایگاه داده ذخیره کرده و آن را به درستی خوانده و به‌روزرسانی می‌کند.

`LangGraph` یک لایه ذخیره‌سازی داخلی پیاده‌سازی می‌کند که آن را برای برنامه‌های چت که از چندین نوبت مکالمه پشتیبانی می‌کنند، ایده‌آل می‌سازد.

پیچیدن مدل چت ما در یک برنامه ساده `LangGraph` این امکان را به ما می‌دهد که به‌طور خودکار تاریخچه پیام‌ها را ذخیره کنیم و توسعه برنامه‌های چندنوبتی را ساده‌تر کنیم.

`LangGraph` همراه با یک چک‌پوینتر ساده در حافظه ارائه می‌شود که در زیر از آن استفاده می‌کنیم. برای جزئیات بیشتر از جمله نحوه استفاده از بک‌اندهای مختلف ذخیره‌سازی (مثل `SQLite` یا `Postgres`)، به مستندات آن مراجعه کنید.

برای یک راهنمای کامل در مورد نحوه مدیریت تاریخچه پیام‌ها، به راهنمای «How to add message history (memory)» سر بزنید.

```python
from typing import Sequence

from langchain_core.messages import BaseMessage
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import START, StateGraph
from langgraph.graph.message import add_messages
from typing_extensions import Annotated, TypedDict


# We define a dict representing the state of the application.
# This state has the same input and output keys as `rag_chain`.
class State(TypedDict):
    input: str
    chat_history: Annotated[Sequence[BaseMessage], add_messages]
    context: str
    answer: str


# We then define a simple node that runs the `rag_chain`.
# The `return` values of the node update the graph state, so here we just
# update the chat history with the input message and response.
def call_model(state: State):
    response = rag_chain.invoke(state)
    return {
        "chat_history": [
            HumanMessage(state["input"]),
            AIMessage(response["answer"]),
        ],
        "context": response["context"],
        "answer": response["answer"],
    }


# Our graph consists only of one node:
workflow = StateGraph(state_schema=State)
workflow.add_edge(START, "model")
workflow.add_node("model", call_model)

# Finally, we compile the graph with a checkpointer object.
# This persists the state, in this case in memory.
memory = MemorySaver()
app = workflow.compile(checkpointer=memory)
```

این برنامه به‌صورت پیش‌فرض از چندین رشته مکالمه پشتیبانی می‌کند. ما یک دیکشنری پیکربندی ارائه می‌دهیم که شناسه‌ای منحصر به فرد برای یک رشته مشخص می‌کند تا کنترل کنیم کدام رشته اجرا شود. این قابلیت به برنامه اجازه می‌دهد تا از تعاملات با چندین کاربر پشتیبانی کند.

```python
config = {"configurable": {"thread_id": "abc123"}}

result = app.invoke(
    {"input": "What is Task Decomposition?"},
    config=config,
)
print(result["answer"])
```

```shell
Task decomposition is the process of breaking down a complicated task into smaller, more manageable steps. Techniques like Chain of Thought (CoT) and Tree of Thoughts enhance this process by guiding models to think step by step and explore multiple reasoning possibilities. This approach helps in simplifying complex tasks and provides insight into the model's reasoning.
```

نحوه‌ی کارکرد برنامه به صورت گرافیکی در زیر نمایش داده شده است:

{{< img image="/posts/build_a_rag_chatbot_with_langchain/diagram.png" alt="cluster" width="750px">}}

## عامل یا `Agent`

عامل ها (`Agents`) از قابلیت‌های استدلالی مدل‌های زبانی بزرگ (`LLMs`) برای تصمیم‌گیری در زمان اجرا استفاده می‌کنند. استفاده از عوامل به شما اجازه می‌دهد تا بخشی از تصمیم‌گیری در فرآیند بازیابی اطلاعات را به آن‌ها واگذار کنید. اگرچه رفتار آن‌ها کمتر از زنجیره‌ها (`chains`) قابل پیش‌بینی است، اما استفاده از آنها در موارد زیر مزایایی دارد:

- عوامل به صورت مستقیم ورودی مورد نیاز برای `retriever` را تولید می‌کنند، بدون این‌که لزوماً نیاز باشد ما صراحتاً در تولید زمینه (contextualization) مداخله کنیم، همان‌طور که در مثال‌های قبل انجام دادیم.
- عامل ها می‌توانند چندین مرحله بازیابی اطلاعات را برای پاسخ به یک درخواست اجرا کنند، یا حتی از اجرای مرحله بازیابی صرف‌نظر کنند (برای مثال، در پاسخ به یک سلام ساده از کاربر).

### ابزار بازیابی (`Retrieval tool`)
عامل ها می‌توانند به "ابزارها" (tools) دسترسی داشته باشند و اجرای آن‌ها را مدیریت کنند. در این مثال، ما `retriever` خود را به یک ابزار `LangChain` تبدیل می‌کنیم تا توسط عامل استفاده شود:

```python
from langchain.tools.retriever import create_retriever_tool

tool = create_retriever_tool(
    retriever,
    "blog_post_retriever",
    "Searches and returns excerpts from the Autonomous Agents blog post.",
)
tools = [tool]
tool.invoke("task decomposition")
```
### سازنده عامل `Agent constructor`

حالا که ابزارها و مدل زبانی بزرگ (`LLM`) را تعریف کرده‌ایم، می‌توانیم عامل را ایجاد کنیم. ما از `LangGraph` برای ساخت عامل استفاده خواهیم کرد. در حال حاضر از یک رابط کاربری سطح بالا برای ساخت عامل استفاده می‌کنیم، اما ویژگی جالب `LangGraph` این است که این رابط کاربری سطح بالا بر پایه یک API سطح پایین و قابل کنترل است، در صورتی که بخواهید منطق عامل را تغییر دهید.

```python
from langgraph.prebuilt import create_react_agent

agent_executor = create_react_agent(llm, tools)
```

حالا می‌توانیم آن را امتحان کنیم. توجه داشته باشید که تاکنون عامل  `stateless` است و هنوز باید حافظه را اضافه کنیم.

```python
query = "What is Task Decomposition?"

for event in agent_executor.stream(
    {"messages": [HumanMessage(content=query)]},
    stream_mode="values",
):
    event["messages"][-1].pretty_print()
```

{{< collapsible header="نمایش  خروجی" >}}

```shell
===== Human Message =====

What is Task Decomposition?
======= Ai Message ======
Tool Calls:
  blog_post_retriever (call_WKHdiejvg4In982Hr3EympuI)
 Call ID: call_WKHdiejvg4In982Hr3EympuI
  Args:
    query: Task Decomposition
====== Tool Message =====
Name: blog_post_retriever

Fig. 1. Overview of a LLM-powered autonomous agent system.
Component One: Planning#
A complicated task usually involves many steps. An agent needs to know what they are and plan ahead.
Task Decomposition#
Chain of thought (CoT; Wei et al. 2022) has become a standard prompting technique for enhancing model performance on complex tasks. The model is instructed to “think step by step” to utilize more test-time computation to decompose hard tasks into smaller and simpler steps. CoT transforms big tasks into multiple manageable tasks and shed lights into an interpretation of the model’s thinking process.

Tree of Thoughts (Yao et al. 2023) extends CoT by exploring multiple reasoning possibilities at each step. It first decomposes the problem into multiple thought steps and generates multiple thoughts per step, creating a tree structure. The search process can be BFS (breadth-first search) or DFS (depth-first search) with each state evaluated by a classifier (via a prompt) or majority vote.
Task decomposition can be done (1) by LLM with simple prompting like "Steps for XYZ.\n1.", "What are the subgoals for achieving XYZ?", (2) by using task-specific instructions; e.g. "Write a story outline." for writing a novel, or (3) with human inputs.

(3) Task execution: Expert models execute on the specific tasks and log results.
Instruction:

With the input and the inference results, the AI assistant needs to describe the process and results. The previous stages can be formed as - User Input: {{ User Input }}, Task Planning: {{ Tasks }}, Model Selection: {{ Model Assignment }}, Task Execution: {{ Predictions }}. You must first answer the user's request in a straightforward manner. Then describe the task process and show your analysis and model inference results to the user in the first person. If inference results contain a file path, must tell the user the complete file path.

Fig. 11. Illustration of how HuggingGPT works. (Image source: Shen et al. 2023)
The system comprises of 4 stages:
(1) Task planning: LLM works as the brain and parses the user requests into multiple tasks. There are four attributes associated with each task: task type, ID, dependencies, and arguments. They use few-shot examples to guide LLM to do task parsing and planning.
Instruction:
======= Ai Message ======

Task Decomposition is a process used in complex problem-solving where a larger task is broken down into smaller, more manageable sub-tasks. This approach enhances the ability of models, particularly large language models (LLMs), to handle intricate tasks by allowing them to think step by step.

There are several methods for task decomposition:

1. **Chain of Thought (CoT)**: This technique encourages the model to articulate its reasoning process by thinking through the task in a sequential manner. It transforms a big task into smaller, manageable steps, which also provides insight into the model's thought process.

2. **Tree of Thoughts**: An extension of CoT, this method explores multiple reasoning possibilities at each step. It decomposes the problem into various thought steps and generates multiple thoughts for each step, creating a tree structure. The evaluation of each state can be done using breadth-first search (BFS) or depth-first search (DFS).

3. **Prompting Techniques**: Task decomposition can be achieved through simple prompts like "Steps for XYZ" or "What are the subgoals for achieving XYZ?" Additionally, task-specific instructions can guide the model, such as asking it to "Write a story outline" for creative tasks.

4. **Human Inputs**: In some cases, human guidance can be used to assist in breaking down tasks.

Overall, task decomposition is a crucial component in planning and executing complex tasks, allowing for better organization and clarity in the problem-solving process.
```

{{</ collapsible >}}

دوباره می‌توانیم از قابلیت حافظه داخلی `LangGraph` استفاده کنیم تا بروزرسانی‌های `stateful` را در حافظه ذخیره کنیم.

```python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()

agent_executor = create_react_agent(llm, tools, checkpointer=memory)
```

این تمام چیزی است که برای ساخت یک عامل `RAG` نیاز داریم.
توجه داشته باشید که اگر درخواستی وارد کنیم که نیاز به مرحله بازیابی اطلاعات ندارد، عامل مرحله‌ای برای بازیابی اجرا نمی‌کند:

```python
config = {"configurable": {"thread_id": "abc123"}}

for event in agent_executor.stream(
    {"messages": [HumanMessage(content="Hi! I'm bob")]},
    config=config,
    stream_mode="values",
):
    event["messages"][-1].pretty_print()
```

```shell
===== Human Message =====

Hi! I'm bob
===== Ai Message ======

Hello Bob! How can I assist you today?
```

علاوه بر این، اگر یک درخواست وارد کنیم که نیاز به مرحله بازیابی اطلاعات دارد، عامل ورودی مورد نیاز برای ابزار را تولید می‌کند:

```python
query = "What is Task Decomposition?"

for event in agent_executor.stream(
    {"messages": [HumanMessage(content=query)]},
    config=config,
    stream_mode="values",
):
    event["messages"][-1].pretty_print()
```

{{< collapsible header="نمایش  خروجی" >}}

```shell
===== Human Message =====

What is Task Decomposition?
===== Ai Message ======
Tool Calls:
  blog_post_retriever (call_0rhrUJiHkoOQxwqCpKTkSkiu)
 Call ID: call_0rhrUJiHkoOQxwqCpKTkSkiu
  Args:
    query: Task Decomposition
===== Tool Message =====
Name: blog_post_retriever

Fig. 1. Overview of a LLM-powered autonomous agent system.
Component One: Planning#
A complicated task usually involves many steps. An agent needs to know what they are and plan ahead.
Task Decomposition#
Chain of thought (CoT; Wei et al. 2022) has become a standard prompting technique for enhancing model performance on complex tasks. The model is instructed to “think step by step” to utilize more test-time computation to decompose hard tasks into smaller and simpler steps. CoT transforms big tasks into multiple manageable tasks and shed lights into an interpretation of the model’s thinking process.

Tree of Thoughts (Yao et al. 2023) extends CoT by exploring multiple reasoning possibilities at each step. It first decomposes the problem into multiple thought steps and generates multiple thoughts per step, creating a tree structure. The search process can be BFS (breadth-first search) or DFS (depth-first search) with each state evaluated by a classifier (via a prompt) or majority vote.
Task decomposition can be done (1) by LLM with simple prompting like "Steps for XYZ.\n1.", "What are the subgoals for achieving XYZ?", (2) by using task-specific instructions; e.g. "Write a story outline." for writing a novel, or (3) with human inputs.

(3) Task execution: Expert models execute on the specific tasks and log results.
Instruction:

With the input and the inference results, the AI assistant needs to describe the process and results. The previous stages can be formed as - User Input: {{ User Input }}, Task Planning: {{ Tasks }}, Model Selection: {{ Model Assignment }}, Task Execution: {{ Predictions }}. You must first answer the user's request in a straightforward manner. Then describe the task process and show your analysis and model inference results to the user in the first person. If inference results contain a file path, must tell the user the complete file path.

Fig. 11. Illustration of how HuggingGPT works. (Image source: Shen et al. 2023)
The system comprises of 4 stages:
(1) Task planning: LLM works as the brain and parses the user requests into multiple tasks. There are four attributes associated with each task: task type, ID, dependencies, and arguments. They use few-shot examples to guide LLM to do task parsing and planning.
Instruction:
===== Ai Message ======

Task Decomposition is a technique used to break down complex tasks into smaller, more manageable steps. This approach is particularly useful in the context of autonomous agents and large language models (LLMs). Here are some key points about Task Decomposition:

1. **Chain of Thought (CoT)**: This is a prompting technique that encourages the model to "think step by step." By doing so, it can utilize more computational resources to decompose difficult tasks into simpler ones, making them easier to handle.

2. **Tree of Thoughts**: An extension of CoT, this method explores multiple reasoning possibilities at each step. It decomposes a problem into various thought steps and generates multiple thoughts for each step, creating a tree structure. This can be evaluated using search methods like breadth-first search (BFS) or depth-first search (DFS).

3. **Methods of Decomposition**: Task decomposition can be achieved through:
   - Simple prompting (e.g., asking for steps to achieve a goal).
   - Task-specific instructions (e.g., requesting a story outline for writing).
   - Human inputs to guide the decomposition process.

4. **Execution**: After decomposition, expert models execute the specific tasks and log the results, allowing for a structured approach to complex problem-solving.

Overall, Task Decomposition enhances the model's ability to tackle intricate tasks by breaking them down into simpler, actionable components.
```

{{</ collapsible >}}

در مثال بالا، به جای اینکه درخواست خود را به همان شکلی که هست به `tools` وارد کند، عامل کلماتی غیرضروری مثل "چیست" و "چه" را حذف کرد.

همین اصل به عامل اجازه می‌دهد که در صورت لزوم از زمینه مکالمه برای تصمیم‌گیری استفاده کند:

```python
query = "What according to the blog post are common ways of doing it? redo the search"

for event in agent_executor.stream(
    {"messages": [HumanMessage(content=query)]},
    config=config,
    stream_mode="values",
):
    event["messages"][-1].pretty_print()
```

توجه کنید که عامل توانست تشخیص دهد که "آن" در درخواست ما به "تقسیم وظایف" اشاره دارد و در نتیجه یک پرسش جستجوی منطقی٬ در این مثال "روش‌های رایج تقسیم وظایف" ایجاد کرد.



