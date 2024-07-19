---
title: "فراخوانی توابع توسط مدل"
description: این notebook نحوه استفاده از API Chat Completions را در ترکیب با توابع خارجی برای گسترش قابلیت های مدل های GPT را نشان می دهد."
tags:
- function-call
weight: 2003
og_image: "/posts/how_to_call_functions_with_chat_models/banner.png" 
---

{{< postcover src="/posts/how_to_call_functions_with_chat_models/banner.png" >}}


این notebook نحوه استفاده از API Chat Completions را در ترکیب با توابع خارجی برای گسترش قابلیت های مدل های GPT را نشان می دهد. 

پارامتر `tools` یک پارامتر اختیاری در API Chat Completion است که می تواند برای ارائه مشخصات تابع استفاده شود. هدف از این امر فراهم کردن امکان تولید آرگومان های تابعی است که با مشخصات ارائه شده مطابقت دارند. 

توجه داشته باشید که API هیچ تابعی را اجرا نمی کند٬ بلکه مشخصات تابعی که متناسب با متن ورودی است را تعیین می‌کند و این بر عهده توسعه دهندگان است که با استفاده از خروجی های مدل توابع را اجرا کنند. 

در پارامتر `tools` اگر پارامتر `functions` ارائه شود، به طور پیش فرض مدل تصمیم می گیرد که کی مناسب است که یکی از توابع را استفاده کند.  همچنین میشود مدل را به استفاده از یک تابع خاص مجبور کرد . اینکار را با تنظیم پارامتر `tool_choice` به `{"type": "function", "function": {"name": "my_function"}` انجام می‌دهیم.

همچنین API می تواند مجبور شود که از هیچ تابعی استفاده نکند با تنظیم پارامتر `tool_choice` به مقدار`"none"`.

 اگر تابعی استفاده شود، خروجی حاوی `"finish_reason": "tool_calls"`  به علاوه یک شیء `tool_calls` که نام تابع و آرگومان های تابع تولید شده را دارد خواهد بود.

{{< hint warning >}} 
توجه: ورودی‌های داده شده و خروجی‌های تولید شده توسط مدل در این مثال به زبان انگلیسی هستند. برای تولید خروجی به زبان فارسی٬ کافی‌ست از مدل بخواهید که خروجی را به زبان فارسی تولید کند.
{{< /hint >}} 


 این notebooks شامل 2 بخش زیر است:
 
 - چگونه آرگومان های تابع را تولید کنیم: مجموعه ای از توابع را مشخص کنید و از API برای تولید آرگومان های تابع استفاده کنید. 
 
 - چگومه توابع را با آرگومان های تولید شده توسط مدل فراخوانی کنیم: اجرای واقعی توابع با آرگومان های تولید شده توسط مدل.

 ## تولید آرگومان های تابع توسط مدل

 {{< hint info >}}
**برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.**
{{< /hint >}} 

```python
!pip install scipy --quiet
!pip install tenacity --quiet
!pip install tiktoken --quiet
!pip install termcolor --quiet
!pip install openai --quiet
```

```python
import json
from openai import OpenAI
from tenacity import retry, wait_random_exponential, stop_after_attempt
from termcolor import colored 

client = OpenAI(
    api_key=os.environ.get(("GILAS_API_KEY", "<کلید API خود را اینجا بسازید https://dashboard.gilas.io/apiKey>")), 
    base_url="https://api.gilas.io/v1/" # Gilas APIs
)
```

**Utilities**

اول چند ابزار (utility function) را برای ارتباط با API Chat Completions و حفظ و ردیابی وضعیت مکالمه تعریف کنیم.

تابع زیر وظیفه ارسال درخواست API به مدل را دارد.

```python
@retry(wait=wait_random_exponential(multiplier=1, max=40), stop=stop_after_attempt(3))
def chat_completion_request(messages, tools=None, tool_choice=None, model=GPT_MODEL):
    try:
        response = client.chat.completions.create(
            model=model,
            messages=messages,
            tools=tools,
            tool_choice=tool_choice,
        )
        return response
    except Exception as e:
        print("Unable to generate ChatCompletion response")
        print(f"Exception: {e}")
        return e

```

و این تابع مسیول چاپ کردن پیام‌ها است.

```python
def pretty_print_conversation(messages):
    role_to_color = {
        "system": "red",
        "user": "green",
        "assistant": "blue",
        "function": "magenta",
    }
    
    for message in messages:
        if message["role"] == "system":
            print(colored(f"system: {message['content']}\n", role_to_color[message["role"]]))
        elif message["role"] == "user":
            print(colored(f"user: {message['content']}\n", role_to_color[message["role"]]))
        elif message["role"] == "assistant" and message.get("function_call"):
            print(colored(f"assistant: {message['function_call']}\n", role_to_color[message["role"]]))
        elif message["role"] == "assistant" and not message.get("function_call"):
            print(colored(f"assistant: {message['content']}\n", role_to_color[message["role"]]))
        elif message["role"] == "function":
            print(colored(f"function ({message['name']}): {message['content']}\n", role_to_color[message["role"]]))
```

### **مفاهیم اولیه**

می‌خواهیم برخی از مشخصات تابع را برای ارتباط با یک API آب و هوای فرضی ایجاد کنیم. ما این مشخصات تابع را به API Chat Completions می دهیم تا آرگومان های تابعی را تولید کنیم که با مشخصات مطابقت دارند.

پیشنهاد می‌کنیم مشخصات توابع توصیف شده در زیر را بررسی کنید تا متوجه شوید هر تابع چه آرگومان‌های ورودی رو دریافت می‌کند و چه وظیفه‌ای را بر عهده دارد.

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_current_weather",
            "description": "Get the current weather",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "The city and state, e.g. San Francisco, CA",
                    },
                    "format": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "The temperature unit to use. Infer this from the users location.",
                    },
                },
                "required": ["location", "format"],
            },
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_n_day_weather_forecast",
            "description": "Get an N-day weather forecast",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "The city and state, e.g. San Francisco, CA",
                    },
                    "format": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "The temperature unit to use. Infer this from the users location.",
                    },
                    "num_days": {
                        "type": "integer",
                        "description": "The number of days to forecast",
                    }
                },
                "required": ["location", "format", "num_days"]
            },
        }
    },
]
```

با استفاده از پیغام `system` زیر از مدل می‌خواهیم که در صورتی که کاربر در مورد آب و هوای فعلی سوال کرد، چند سوال تکمیلی برای تکمیل اطلاعات خود از کاربر بپرسد.

```python
messages = []
messages.append({"role": "system", "content": "Don't make assumptions about what values to plug into functions. Ask for clarification if a user request is ambiguous."})
messages.append({"role": "user", "content": "What's the weather like today"})
chat_response = chat_completion_request(
    messages, tools=tools
)
assistant_message = chat_response.choices[0].message
messages.append(assistant_message)
assistant_message
```

خروجی:
{{< ltr >}}
```
ChatCompletionMessage(content='Sure, could you please tell me the location for which you would like to know the weather?', role='assistant', function_call=None, tool_calls=None)
```
{{< /ltr >}}


هنگامی که کاربر اطلاعات تکمیلی را ارائه دهد، مدل آرگومان های مناسب تابع را برای ما تولید می کند.

```python
messages.append({"role": "user", "content": "I'm in Glasgow, Scotland."})
chat_response = chat_completion_request(
    messages, tools=tools
)
assistant_message = chat_response.choices[0].message
messages.append(assistant_message)
assistant_message
```

خروجی مدل همراه با اسم تابعی که باید فراخوانی شود و آرگومان‌های ورودی آن:

{{< ltr >}}
```
ChatCompletionMessage(content=None, role='assistant', function_call=None, tool_calls=[ChatCompletionMessageToolCall(id='call_2PArU89L2uf4uIzRqnph4SrN', function=Function(arguments='{\n  "location": "Glasgow, Scotland",\n  "format": "celsius"\n}', name='get_current_weather'), type='function')])
```
{{< /ltr >}}

با تغییر متنی که به مدل می دهیم، می توانیم آن را به تابع دیگری که برای آن تعریف کرده‌ایم، متمایل کنیم.

```python
messages = []
messages.append({"role": "system", "content": "Don't make assumptions about what values to plug into functions. Ask for clarification if a user request is ambiguous."})
messages.append({"role": "user", "content": "what is the weather going to be like in Glasgow, Scotland over the next x days"})
chat_response = chat_completion_request(
    messages, tools=tools
)
assistant_message = chat_response.choices[0].message
messages.append(assistant_message)
assistant_message
```

خروجی:
{{< ltr >}}
```
ChatCompletionMessage(content='Sure, I can help you with that. How many days would you like to get the weather forecast for?', role='assistant', function_call=None, tool_calls=None)
```
{{< /ltr >}}

دوباره، مدل از ما برای توضیحات بیشتر سوال می کند زیرا هنوز اطلاعات کافی را برای تولید آرکومان‌های لازم برای تابع ندارد. در این مورد مدل از مکانی که کاربر بیان کرده مطلع است، اما نمی‌داند پیش‌بینی آب و هوا را برای چند روز باید انجام دهد.

```python
messages.append({"role": "user", "content": "5 days"})
chat_response = chat_completion_request(
    messages, tools=tools
)
chat_response.choices[0]
```

{{< ltr >}}
```
Choice(finish_reason='tool_calls', index=0, logprobs=None, message=ChatCompletionMessage(content=None, role='assistant', function_call=None, tool_calls=[ChatCompletionMessageToolCall(id='call_ujD1NwPxzeOSCbgw2NOabOin', function=Function(arguments='{\n  "location": "Glasgow, Scotland",\n  "format": "celsius",\n  "num_days": 5\n}', name='get_n_day_weather_forecast'), type='function')]), internal_metrics=[{'cached_prompt_tokens': 128, 'total_accepted_tokens': 0, 'total_batched_tokens': 273, 'total_predicted_tokens': 0, 'total_rejected_tokens': 0, 'total_tokens_in_completion': 274, 'cached_embeddings_bytes': 0, 'cached_embeddings_n': 0, 'uncached_embeddings_bytes': 0, 'uncached_embeddings_n': 0, 'fetched_embeddings_bytes': 0, 'fetched_embeddings_n': 0, 'n_evictions': 0, 'sampling_steps': 40, 'sampling_steps_with_predictions': 0, 'batcher_ttft': 0.035738229751586914, 'batcher_initial_queue_time': 0.0007979869842529297}])
```
{{< /ltr >}}

### **اجبار مدل به استفاده از توابعی خاص یا هیچ تابعی**

 ما می توانیم مدل را مجبور کنیم که از یک تابع خاص استفاده کند، به عنوان مثال `get_n_day_weather_forecast` با استفاده از آرگومان `function_call`. با این کار، ما مدل را مجبور می کنیم که خودش فرضیاتی را در مورد نحوه استفاده از آن بکند.

 ```python
 # in this cell we force the model to use get_n_day_weather_forecast
messages = []
messages.append({"role": "system", "content": "Don't make assumptions about what values to plug into functions. Ask for clarification if a user request is ambiguous."})
messages.append({"role": "user", "content": "Give me a weather report for Toronto, Canada."})
chat_response = chat_completion_request(
    messages, tools=tools, tool_choice={"type": "function", "function": {"name": "get_n_day_weather_forecast"}}
)
chat_response.choices[0].message
```

```
ChatCompletionMessage(content=None, role='assistant', function_call=None, tool_calls=[ChatCompletionMessageToolCall(id='call_MapM0kaNZBR046H4tAB2UGVu', function=Function(arguments='{\n  "location": "Toronto, Canada",\n  "format": "celsius",\n  "num_days": 1\n}', name='get_n_day_weather_forecast'), type='function')])
```

```python
# if we don't force the model to use get_n_day_weather_forecast it may not
messages = []
messages.append({"role": "system", "content": "Don't make assumptions about what values to plug into functions. Ask for clarification if a user request is ambiguous."})
messages.append({"role": "user", "content": "Give me a weather report for Toronto, Canada."})
chat_response = chat_completion_request(
    messages, tools=tools
)
chat_response.choices[0].message
```

{{< ltr >}}
```
ChatCompletionMessage(content=None, role='assistant', function_call=None, tool_calls=[ChatCompletionMessageToolCall(id='call_z8ijGSoMLS7xcaU7MjLmpRL8', function=Function(arguments='{\n  "location": "Toronto, Canada",\n  "format": "celsius"\n}', name='get_current_weather'), type='function')])
```
{{< /ltr >}}

ما همچنین می توانیم مدل را مجبور کنیم که از هیچ تابعی استفاده نکند. 

```python
messages = []
messages.append({"role": "system", "content": "Don't make assumptions about what values to plug into functions. Ask for clarification if a user request is ambiguous."})
messages.append({"role": "user", "content": "Give me the current weather (use Celcius) for Toronto, Canada."})
chat_response = chat_completion_request(
    messages, tools=tools, tool_choice="none"
)
chat_response.choices[0].message
```

{{< ltr >}}
```
ChatCompletionMessage(content='{\n  "location": "Toronto, Canada",\n  "format": "celsius"\n}', role='assistant', function_call=None, tool_calls=None)
```
{{< /ltr >}}

### **فراخوانی موازی توابع**

مدل های جدیدتر مانند gpt-4-turbo یا gpt-4o-mini می توانند در یک نوبت چندین تابع را فراخوانی کنند.

```python
messages = []
messages.append({"role": "system", "content": "Don't make assumptions about what values to plug into functions. Ask for clarification if a user request is ambiguous."})
messages.append({"role": "user", "content": "what is the weather going to be like in San Francisco and Glasgow over the next 4 days"})
chat_response = chat_completion_request(
    messages, tools=tools, model='gpt-4o-mini'
)

assistant_message = chat_response.choices[0].message.tool_calls
assistant_message
```

{{< ltr >}}
```
[
    ChatCompletionMessageToolCall(id='call_8BlkS2yvbkkpL3V1Yxc6zR6u', function=Function(arguments='{"location": "San Francisco, CA", "format": "celsius", "num_days": 4}', name='get_n_day_weather_forecast'), type='function'),

    ChatCompletionMessageToolCall(id='call_vSZMy3f24wb3vtNXucpFfAbG', function=Function(arguments='{"location": "Glasgow", "format": "celsius", "num_days": 4}', name='get_n_day_weather_forecast'), type='function')
]
```
{{< /ltr >}}

## فراخوانی توابع با آرگومان های تولید شده توسط مدل

 حال نشان خواهیم داد چگونه توابعی را اجرا کنیم که ورودی های آنها توسط مدل تولید شده است، و از این برای پیاده‌سازی یک چت‌بات که می تواند سوالات ما را در مورد یک پایگاه داده پاسخ دهد.
 
  برای سادگی ما از پایگاه داده نمونه [Chinook](https://www.sqlitetutorial.net/sqlite-sample-database/) استفاده خواهیم کرد.
  
   توجه: تولید SQL در محیط production می‌تواند ریسک بالایی داشته باشد زیرا مدل ها در تولید SQL صحیح کاملاً قابل اعتماد نیستند.
   
###  **تعیین یک تابع برای اجرای پرس و جوهای SQL**
    
اول بیایید چند تابع کمکی مفید را برای استخراج داده ها از یک پایگاه داده SQLite تعریف کنیم.

```python
import sqlite3

conn = sqlite3.connect("data/Chinook.db")
print("Opened database successfully")

def get_table_names(conn):
    """Return a list of table names."""
    table_names = []
    tables = conn.execute("SELECT name FROM sqlite_master WHERE type='table';")
    for table in tables.fetchall():
        table_names.append(table[0])
    return table_names


def get_column_names(conn, table_name):
    """Return a list of column names."""
    column_names = []
    columns = conn.execute(f"PRAGMA table_info('{table_name}');").fetchall()
    for col in columns:
        column_names.append(col[1])
    return column_names


def get_database_info(conn):
    """Return a list of dicts containing the table name and columns for each table in the database."""
    table_dicts = []
    for table_name in get_table_names(conn):
        columns_names = get_column_names(conn, table_name)
        table_dicts.append({"table_name": table_name, "column_names": columns_names})
    return table_dicts

```

حالا می توانیم از این توابع کمکی برای استخراج نمایشی از ساختار پایگاه داده استفاده کنیم.#

```python
database_schema_dict = get_database_info(conn)
database_schema_string = "\n".join(
    [
        f"Table: {table['table_name']}\nColumns: {', '.join(table['column_names'])}"
        for table in database_schema_dict
    ]
)
```

مانند قبل، مشخصات تابع را برای تابعی که می خواهیم API برای آن آرگومان ایجاد کند، تعریف می کنیم. توجه کنید که ما ساختار پایگاه داده را در مشخصات تابع وارد می کنیم چون این برای اطلاعات برای تصمیم‌گیری مدل مهم خواهد بود.

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "ask_database",
            "description": "Use this function to answer user questions about music. Input should be a fully formed SQL query.",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": f"""
                                SQL query extracting info to answer the user's question.
                                SQL should be written using this database schema:
                                {database_schema_string}
                                The query should be returned in plain text, not in JSON.
                                """,
                    }
                },
                "required": ["query"],
            },
        }
    }
]
```
### **اجرای پرس و جوهای SQL**

حالا بیایید تابعی را پیاده سازی کنیم که واقعاً پرس و جوها را در برابر پایگاه داده اجرا می کند.

```python
def ask_database(conn, query):
    """Function to query SQLite database with a provided SQL query."""
    try:
        results = str(conn.execute(query).fetchall())
    except Exception as e:
        results = f"query failed with error: {e}"
    return results

def execute_function_call(message):
    if message.tool_calls[0].function.name == "ask_database":
        query = json.loads(message.tool_calls[0].function.arguments)["query"]
        results = ask_database(conn, query)
    else:
        results = f"Error: function {message.tool_calls[0].function.name} does not exist"
    return results
```

و حالا از مدل سوالی در مورد دیتا‌های داخل دیتابیس می‌پرسیم و انتظار داریم که مدل اسم تابع `ask_database` را به همراه آرگومان‌های لازم برای فراخوانی آن را تولید کند.

```python
messages = []
messages.append({"role": "system", "content": "Answer user questions by generating SQL queries against the Chinook Music Database."})
messages.append({"role": "user", "content": "Hi, who are the top 5 artists by number of tracks?"})
chat_response = chat_completion_request(messages, tools)
assistant_message = chat_response.choices[0].message
assistant_message.content = str(assistant_message.tool_calls[0].function)
messages.append({"role": assistant_message.role, "content": assistant_message.content})
if assistant_message.tool_calls:
    results = execute_function_call(assistant_message)
    messages.append({"role": "function", "tool_call_id": assistant_message.tool_calls[0].id, "name": assistant_message.tool_calls[0].function.name, "content": results})
pretty_print_conversation(messages)
```

{{< ltr >}}
```
system: Answer user questions by generating SQL queries against the Chinook Music Database.

user: Hi, who are the top 5 artists by number of tracks?

assistant: Function(arguments='{\n  "query": "SELECT Artist.Name, COUNT(Track.TrackId) AS TrackCount FROM Artist JOIN Album ON Artist.ArtistId = Album.ArtistId JOIN Track ON Album.AlbumId = Track.AlbumId GROUP BY Artist.ArtistId ORDER BY TrackCount DESC LIMIT 5;"\n}', name='ask_database')

function (ask_database): [('Iron Maiden', 213), ('U2', 135), ('Led Zeppelin', 114), ('Metallica', 112), ('Lost', 92)]
```
{{< /ltr >}}

مثالی دیگر:

```python
messages.append({"role": "user", "content": "What is the name of the album with the most tracks?"})
chat_response = chat_completion_request(messages, tools)
assistant_message = chat_response.choices[0].message
assistant_message.content = str(assistant_message.tool_calls[0].function)
messages.append({"role": assistant_message.role, "content": assistant_message.content})
if assistant_message.tool_calls:
    results = execute_function_call(assistant_message)
    messages.append({"role": "function", "tool_call_id": assistant_message.tool_calls[0].id, "name": assistant_message.tool_calls[0].function.name, "content": results})
pretty_print_conversation(messages)
```

{{< ltr >}}
```
system: Answer user questions by generating SQL queries against the Chinook Music Database.

user: Hi, who are the top 5 artists by number of tracks?

assistant: Function(arguments='{\n  "query": "SELECT Artist.Name, COUNT(Track.TrackId) AS TrackCount FROM Artist JOIN Album ON Artist.ArtistId = Album.ArtistId JOIN Track ON Album.AlbumId = Track.AlbumId GROUP BY Artist.ArtistId ORDER BY TrackCount DESC LIMIT 5;"\n}', name='ask_database')

function (ask_database): [('Iron Maiden', 213), ('U2', 135), ('Led Zeppelin', 114), ('Metallica', 112), ('Lost', 92)]

user: What is the name of the album with the most tracks?

assistant: Function(arguments='{\n  "query": "SELECT Album.Title, COUNT(Track.TrackId) AS TrackCount FROM Album JOIN Track ON Album.AlbumId = Track.AlbumId GROUP BY Album.AlbumId ORDER BY TrackCount DESC LIMIT 1;"\n}', name='ask_database')

function (ask_database): [('Greatest Hits', 57)]
```
{{< /ltr >}}