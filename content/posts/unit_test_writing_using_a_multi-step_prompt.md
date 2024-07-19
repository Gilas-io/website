---
title: "نوشتن یونیت تست با پرامپت چند مرحله‌ای"
description: "راهنمای نوشتن تست‌های واحد با استفاده از پرامپت‌های چند مرحله‌ای برای بهبود دقت و پوشش تست‌ها."
tags:
- prompt-engineering
weight: 
og_image: "/posts/unit_test_writing_using_a_multi-step_prompt/banner.png"
---

{{< postcover src="/posts/unit_test_writing_using_a_multi-step_prompt/banner.png" >}}

برای انجام وظایف پیچیده‌ای مثل نوشتن یونیت تست برای کد پایتون بهتر است از روش پراپمت چند مرحله ای یا chain of thoughts استفاده کنیم. برخلاف یک پرامپت تکی، یک پرامپت چند مرحله‌ای متن را از `GPT` تولید کرده و سپس آن متن را به پرامپت‌های بعدی می‌دهد. 

این روش می‌تواند در مواردی که می‌خواهید `GPT` قبل از پاسخ دادن به موضوع فکر کند یا قبل از انجام کاری ابتدا برای آن برنامه‌ریزی کند، مفید باشد.

در این `notebook`، از یک پرامپت ۳ مرحله‌ای برای نوشتن یونیت تست‌ در `Python` استفاده می‌کنیم که شامل مراحل زیر است:

1. **توضیح**: با دادن یک تابع `Python`، از `GPT` می‌خواهیم که توضیح دهد تابع چه کاری انجام می‌دهد و چرا.
2. **برنامه‌ریزی**: از `GPT` می‌خواهیم که مجموعه‌ای از یونیت تست‌ها برای تابع برنامه‌ریزی کند. در اینجا منظور ما از برنامه‌ریزی چیزی شبیه به در نظر گرفتن تست های مختلف برای پوشش دادن حالت‌های مختلف است.
3. **اجرا**: در نهایت، به `GPT` دستور می‌دهیم که یونیت تست‌هایی را بر اساس برنامه‌ریزی انجام شده بنویسد.

##  نوشتن توابع کمکی

{{< hint info >}}
**برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.**
{{< /hint >}} 


```python
# imports needed to run the code in this notebook
import ast  # used for detecting whether generated Python code is valid
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ.get(("GILAS_API_KEY", "<کلید API خود را اینجا بسازید https://dashboard.gilas.io/apiKey>")), 
    base_url="https://api.gilas.io/v1/" # Gilas APIs
)

color_prefix_by_role = {
    "system": "\033[0m",  # gray
    "user": "\033[0m",  # gray
    "assistant": "\033[92m",  # green
}


def print_messages(messages, color_prefix_by_role=color_prefix_by_role) -> None:
    """Prints messages sent to or from GPT."""
    for message in messages:
        role = message["role"]
        color_prefix = color_prefix_by_role[role]
        content = message["content"]
        print(f"{color_prefix}\n[{role}]\n{content}")


def print_message_delta(delta, color_prefix_by_role=color_prefix_by_role) -> None:
    """Prints a chunk of messages streamed back from GPT."""
    if "role" in delta:
        role = delta["role"]
        color_prefix = color_prefix_by_role[role]
        print(f"{color_prefix}\n[{role}]\n", end="")
    elif "content" in delta:
        content = delta["content"]
        print(content, end="")
    else:
        pass

```

## نوشتن تابعی برای تولید یونیت تست‌ها

در زیر تابعی را مشاهده می‌کنید که کار تولید یونیت تست ها را به عهده دارد.
توجه کنید که چطور این تابع خروجی‌های تولید شده توسط `GPT` به عنوان ورودی برای مرحله بعدی فراخوانی `GPT` استفاده می‌کند.

لطفا برای درک بهتر عملکرد کد به کامنت‌های فارسی داخل کد توجه کنید.


```python
# example of a function that uses a multi-step prompt to write unit tests

def unit_tests_from_function(
    function_to_test: str,  # Python function to test, as a string
    unit_test_package: str = "pytest",  # unit testing package; use the name as it appears in the import statement
    approx_min_cases_to_cover: int = 7,  # minimum number of test case categories to cover (approximate)
    print_text: bool = False,  # optionally prints text; helpful for understanding the function & debugging
    explain_model: str = "gpt-4o-mini",  # model used to generate text plans in step 1
    plan_model: str = "gpt-4o-mini",  # model used to generate text plans in steps 2 and 2b
    execute_model: str = "gpt-4o-mini",  # model used to generate code in step 3
    temperature: float = 0.4,  # temperature = 0 can sometimes get stuck in repetitive loops, so we use 0.4
    reruns_if_fail: int = 1,  # if the output code cannot be parsed, this will re-run the function up to N times
) -> str:
    """Returns a unit test for a given Python function, using a 3-step GPT prompt."""

    # مرحله اول: ارزیابی تابع ورودی و توضیح نحوه عملکرد آن

    explain_system_message = {
        "role": "system",
        "content": "You are a world-class Python developer with an eagle eye for unintended bugs and edge cases. You carefully explain code with great detail and accuracy. You organize your explanations in markdown-formatted, bulleted lists.",
    }
    explain_user_message = {
        "role": "user",
        "content": f"""Please explain the following Python function. Review what each element of the function is doing precisely and what the author's intentions may have been. Organize your explanation as a markdown-formatted, bulleted list.
        ```python
        {function_to_test}
        ```""",
    }

    explain_messages = [explain_system_message, explain_user_message]
    if print_text:
        print_messages(explain_messages)

    explanation_response = client.chat.completions.create(model=explain_model,
    messages=explain_messages,
    temperature=temperature,
    stream=True)

    explanation = ""
    for chunk in explanation_response:
        delta = chunk.choices[0].delta
        if print_text:
            print_message_delta(delta)
        if "content" in delta:
            explanation += delta.content
    explain_assistant_message = {"role": "assistant", "content": explanation}

    # مرحله دوم: برنامه ریزی برای تولید چندین یونیت تست بر اساس کد تابع و توضیحات تولید شده در مرحله قبل

    # Asks GPT to plan out cases the units tests should cover, formatted as a bullet list
    plan_user_message = {
        "role": "user",
        "content": f"""A good unit test suite should aim to:
        - Test the function's behavior for a wide range of possible inputs
        - Test edge cases that the author may not have foreseen
        - Take advantage of the features of `{unit_test_package}` to make the tests easy to write and maintain
        - Be easy to read and understand, with clean code and descriptive names
        - Be deterministic, so that the tests always pass or fail in the same way

        To help unit test the function above, list diverse scenarios that the function should be able to handle (and under each scenario, include a few examples as sub-bullets).""",
    }

    plan_messages = [
        explain_system_message,
        explain_user_message,
        explain_assistant_message,
        plan_user_message,
    ]
    
    if print_text:
        print_messages([plan_user_message])
    plan_response = client.chat.completions.create(model=plan_model,
    messages=plan_messages,
    temperature=temperature,
    stream=True)

    plan = ""
    for chunk in plan_response:
        delta = chunk.choices[0].delta
        if print_text:
            print_message_delta(delta)
        if "content" in delta:
            explanation += delta.content
    plan_assistant_message = {"role": "assistant", "content": plan}

    # مرحله ۲-۲: اگر توضیحات تولید شده خیلی کوتاه است از مدل می‌خواهیم که کار خود را مجددا انجام دهد. برای بررسی میزان توضیحات تعداد بولت پوینت های تولید شده را می‌شماریم.

    num_bullets = max(plan.count("\n-"), plan.count("\n*"))
    elaboration_needed = num_bullets < approx_min_cases_to_cover
    if elaboration_needed:
        elaboration_user_message = {
            "role": "user",
            "content": f"""In addition to those scenarios above, list a few rare or unexpected edge cases (and as before, under each edge case, include a few examples as sub-bullets).""",
        }
        elaboration_messages = [
            explain_system_message,
            explain_user_message,
            explain_assistant_message,
            plan_user_message,
            plan_assistant_message,
            elaboration_user_message,
        ]
        if print_text:
            print_messages([elaboration_user_message])

        elaboration_response = client.chat.completions.create(model=plan_model,
        messages=elaboration_messages,
        temperature=temperature,
        stream=True)

        elaboration = ""
        for chunk in elaboration_response:
            delta = chunk.choices[0].delta
        if print_text:
            print_message_delta(delta)
        if "content" in delta:
            explanation += delta.content
        elaboration_assistant_message = {"role": "assistant", "content": elaboration}

    # مرحله سوم: تولید یونیت تست ها بر اساس خروجی مرحله قبل

    # create a markdown-formatted prompt that asks GPT to complete a unit test
    package_comment = ""
    if unit_test_package == "pytest":
        package_comment = "# below, each test case is represented by a tuple passed to the @pytest.mark.parametrize decorator"
    execute_system_message = {
        "role": "system",
        "content": "You are a world-class Python developer with an eagle eye for unintended bugs and edge cases. You write careful, accurate unit tests. When asked to reply only with code, you write all of your code in a single block.",
    }
    execute_user_message = {
        "role": "user",
        "content": f"""Using Python and the `{unit_test_package}` package, write a suite of unit tests for the function, following the cases above. Include helpful comments to explain each line. Reply only with code, formatted as follows:

        ```python
        # imports
        import {unit_test_package}  # used for our unit tests
        {{insert other imports as needed}}

        # function to test
        {function_to_test}

        # unit tests
        {package_comment}
        {{insert unit test code here}}
        ```""",
    }

    execute_messages = [
        execute_system_message,
        explain_user_message,
        explain_assistant_message,
        plan_user_message,
        plan_assistant_message,
    ]

    if elaboration_needed:
        execute_messages += [elaboration_user_message, elaboration_assistant_message]

    execute_messages += [execute_user_message]
    if print_text:
        print_messages([execute_system_message, execute_user_message])

    execute_response = client.chat.completions.create(model=execute_model,
        messages=execute_messages,
        temperature=temperature,
        stream=True)

    execution = ""
    for chunk in execute_response:
        delta = chunk.choices[0].delta
        if print_text:
            print_message_delta(delta)
        if delta.content:
            execution += delta.content

    # check the output for errors
    code = execution.split("```python")[1].split("```")[0].strip()
    try:
        # پارس کردن کد تولید شده برای اینکه از صحت سینتکس آن مطمپن شویم

        ast.parse(code)
    except SyntaxError as e:
        print(f"Syntax error in generated code: {e}")
        if reruns_if_fail > 0:
            print("Rerunning...")
            return unit_tests_from_function(
                function_to_test=function_to_test,
                unit_test_package=unit_test_package,
                approx_min_cases_to_cover=approx_min_cases_to_cover,
                print_text=print_text,
                explain_model=explain_model,
                plan_model=plan_model,
                execute_model=execute_model,
                temperature=temperature,
                reruns_if_fail=reruns_if_fail
                - 1,  # decrement rerun counter when calling again
            )

    # return the unit test as a string
    return code

```

```python
example_function = """def pig_latin(text):
    def translate(word):
        vowels = 'aeiou'
        if word[0] in vowels:
            return word + 'way'
        else:
            consonants = ''
            for letter in word:
                if letter not in vowels:
                    consonants += letter
                else:
                    break
            return word[len(consonants):] + consonants + 'ay'

    words = text.lower().split()
    translated_words = [translate(word) for word in words]
    return ' '.join(translated_words)
"""

unit_tests = unit_tests_from_function(
    example_function,
    approx_min_cases_to_cover=10,
    print_text=True
)

```

{{< ltr >}}
```
[system]
You are a world-class Python developer with an eagle eye for unintended bugs and edge cases. You carefully explain code with great detail and accuracy. You organize your explanations in markdown-formatted, bulleted lists.

[user]
Please explain the following Python function. Review what each element of the function is doing precisely and what the author's intentions may have been. Organize your explanation as a markdown-formatted, bulleted list.

def pig_latin(text):
    def translate(word):
        vowels = 'aeiou'
        if word[0] in vowels:
            return word + 'way'
        else:
            consonants = ''
            for letter in word:
                if letter not in vowels:
                    consonants += letter
                else:
                    break
            return word[len(consonants):] + consonants + 'ay'

    words = text.lower().split()
    translated_words = [translate(word) for word in words]
    return ' '.join(translated_words)


[user]
A good unit test suite should aim to:
- Test the function's behavior for a wide range of possible inputs
- Test edge cases that the author may not have foreseen
- Take advantage of the features of `pytest` to make the tests easy to write and maintain
- Be easy to read and understand, with clean code and descriptive names
- Be deterministic, so that the tests always pass or fail in the same way

To help unit test the function above, list diverse scenarios that the function should be able to handle (and under each scenario, include a few examples as sub-bullets).

[user]
In addition to those scenarios above, list a few rare or unexpected edge cases (and as before, under each edge case, include a few examples as sub-bullets).

[system]
You are a world-class Python developer with an eagle eye for unintended bugs and edge cases. You write careful, accurate unit tests. When asked to reply only with code, you write all of your code in a single block.

[user]
Using Python and the `pytest` package, write a suite of unit tests for the function, following the cases above. Include helpful comments to explain each line. Reply only with code, formatted as follows:

# imports
import pytest  # used for our unit tests
{insert other imports as needed}

# function to test
def pig_latin(text):
    def translate(word):
        vowels = 'aeiou'
        if word[0] in vowels:
            return word + 'way'
        else:
            consonants = ''
            for letter in word:
                if letter not in vowels:
                    consonants += letter
                else:
                    break
            return word[len(consonants):] + consonants + 'ay'

    words = text.lower().split()
    translated_words = [translate(word) for word in words]
    return ' '.join(translated_words)


# unit tests
# below, each test case is represented by a tuple passed to the @pytest.mark.parametrize decorator
{insert unit test code here}

execute messages: [{'role': 'system', 'content': 'You are a world-class Python developer with an eagle eye for unintended bugs and edge cases. You write careful, accurate unit tests. When asked to reply only with code, you write all of your code in a single block.'}, {'role': 'user', 'content': "Please explain the following Python function. Review what each element of the function is doing precisely and what the author's intentions may have been. Organize your explanation as a markdown-formatted, bulleted list.\n\n```python\ndef pig_latin(text):\n    def translate(word):\n        vowels = 'aeiou'\n        if word[0] in vowels:\n            return word + 'way'\n        else:\n            consonants = ''\n            for letter in word:\n                if letter not in vowels:\n                    consonants += letter\n                else:\n                    break\n            return word[len(consonants):] + consonants + 'ay'\n\n    words = text.lower().split()\n    translated_words = [translate(word) for word in words]\n    return ' '.join(translated_words)\n\n```"}, {'role': 'assistant', 'content': ''}, {'role': 'user', 'content': "A good unit test suite should aim to:\n- Test the function's behavior for a wide range of possible inputs\n- Test edge cases that the author may not have foreseen\n- Take advantage of the features of `pytest` to make the tests easy to write and maintain\n- Be easy to read and understand, with clean code and descriptive names\n- Be deterministic, so that the tests always pass or fail in the same way\n\nTo help unit test the function above, list diverse scenarios that the function should be able to handle (and under each scenario, include a few examples as sub-bullets)."}, {'role': 'assistant', 'content': ''}, {'role': 'user', 'content': 'In addition to those scenarios above, list a few rare or unexpected edge cases (and as before, under each edge case, include a few examples as sub-bullets).'}, {'role': 'assistant', 'content': ''}, {'role': 'user', 'content': "Using Python and the `pytest` package, write a suite of unit tests for the function, following the cases above. Include helpful comments to explain each line. Reply only with code, formatted as follows:\n\n```python\n# imports\nimport pytest  # used for our unit tests\n{insert other imports as needed}\n\n# function to test\ndef pig_latin(text):\n    def translate(word):\n        vowels = 'aeiou'\n        if word[0] in vowels:\n            return word + 'way'\n        else:\n            consonants = ''\n            for letter in word:\n                if letter not in vowels:\n                    consonants += letter\n                else\n                    break\n            return word[len(consonants):] + consonants + 'ay'\n\n    words = text.lower().split()\n    translated_words = [translate(word) for word in words]\n    return ' '.join(translated_words)\n\n\n# unit tests\n# below, eachtest case is represented by a tuple passed to the @pytest.mark.parametrize decorator\n{insert unit test code here}\n```"}]
```

{{< /ltr >}}

```python
print(unit_tests)
```

{{< ltr >}}

```
# imports
import pytest

# function to test
def pig_latin(text):
    def translate(word):
        vowels = 'aeiou'
        if word[0] in vowels:
            return word + 'way'
        else:
            consonants = ''
            for letter in word:
                if letter not in vowels:
                    consonants += letter
                else:
                    break
            return word[len(consonants):] + consonants + 'ay'

    words = text.lower().split()
    translated_words = [translate(word) for word in words]
    return ' '.join(translated_words)


# unit tests
@pytest.mark.parametrize('text, expected', [
    ('hello world', 'ellohay orldway'),  # basic test case
    ('Python is awesome', 'ythonPay isway awesomeway'),  # test case with multiple words
    ('apple', 'appleway'),  # test case with a word starting with a vowel
    ('', ''),  # test case with an empty string
    ('123', '123'),  # test case with non-alphabetic characters
    ('Hello World!', 'elloHay orldWay!'),  # test case with punctuation
    ('The quick brown fox', 'ethay ickquay ownbray oxfay'),  # test case with mixed case words
    ('a e i o u', 'away eway iway oway uway'),  # test case with all vowels
    ('bcd fgh jkl mnp', 'bcday fghay jklway mnpay'),  # test case with all consonants
])
def test_pig_latin(text, expected):
    assert pig_latin(text) == expected

```
{{< /ltr >}}

مطمئن شوید که هر کدی را قبل از استفاده بررسی کنید، زیرا `GPT` ممکن است اشتباهات زیادی مرتکب شود (به خصوص در وظایف مبتنی بر کاراکتر مانند این). برای دریافت بهترین نتایج پیشنهاد می‌دهیم که از قوی‌ترین مدل `GPT` (در این تاریخ `gpt-4o`) استفاده کنید.