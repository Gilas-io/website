---
title: "ترکیب قابلیت vision با فراخوانی توابع"
description: "این Notebook نشان می‌دهد چگونه می‌توان از توانایی‌های بصری GPT-4 برای درک محتوای یک ویدیو و تولید متن متناسب با آن و نهایتا تبدیل متن تولید شده به صدا استفاده کرد. GPT-4 به طور مستقیم ویدیوها را به عنوان ورودی قبول نمی‌کند، اما می‌توانیم از قابلیت vision و طول کانتکست 128K برای توصیف فریم‌های این راهنما شامل دو مرحله است:"
tags:
- function-call
- vision
weight: 2001
og_image: "/posts/gpt_with_vision_for_video_understanding/banner.png" 
---

{{< postcover src="/posts/gpt_with_vision_for_video_understanding/banner.png" >}}




مدل جدید `GPT-4 Turbo`،  اکنون امکان فراخوانی توابع با قابلیت‌های دیداری (vision)و استدلال بهتر را فراهم می‌کند. استفاده از تصاویر با فراخوانی توابع، موارد کاربرد جدید را امکان‌پذیر می‌کند و به شما اجازه می‌دهد فراتر از OCR و توضیحات تصاویر بروید.

ما دو مثال را برای نشان دادن استفاده از فراخوانی توابع با `GPT-4 Turbo` با قابلیت دیداری بررسی خواهیم کرد:

1. شبیه‌سازی یک دستیار خدمات مشتری 
2. تحلیل یک نمودار سازمانی برای استخراج اطلاعات کارکنان

{{< hint info >}}
**برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.**
{{< /hint >}} 


```python
!pip install pymupdf --quiet
!pip install openai --quiet
!pip install matplotlib --quiet
!pip install instructor --quiet

import base64
import os
from enum import Enum
from io import BytesIO
from typing import Iterable, List, Literal, Optional

import fitz
import instructor
import matplotlib.pyplot as plt
import pandas as pd
from IPython.display import display
from PIL import Image
from openai import OpenAI
from pydantic import BaseModel, Field
```

## 1. شبیه‌سازی یک دستیار خدمات مشتری

ما یک دستیار خدمات مشتری را برای یک سرویس تحویل شبیه‌سازی خواهیم کرد که توانایی تحلیل تصاویر بسته‌ها را دارد. این دستیار اقدامات زیر را بر اساس تحلیل تصویر انجام خواهد داد:

- اگر بسته در تصویر آسیب دیده باشد، به‌طور خودکار بازپرداخت را انجام می‌دهد.
- اگر بسته خیس به نظر برسد، محصول جایگزینی را ارسال می‌کند.
- اگر بسته به نظر عادی و بدون آسیب باشد، آن را به یک نماینده ارجاع می‌دهد.

بیایید نمونه‌هایی از تصاویر بسته‌ها را بررسی کنیم که دستیار خدمات مشتری آن‌ها را تحلیل خواهد کرد تا اقدام مناسب را تعیین کند. ما تصاویر را به عنوان رشته‌های base64 برای پردازش توسط مدل کدگذاری خواهیم کرد.

```python
# تابع برای کدگذاری تصویر به صورت base64
def encode_image(image_path: str):
    if not os.path.exists(image_path):
        raise FileNotFoundError(f"Image file not found: {image_path}")
    with open(image_path, "rb") as image_file:
        return base64.b64encode(image_file.read()).decode('utf-8')

# نمونه تصاویر برای تست
image_dir = "images"
image_files = os.listdir(image_dir)
image_data = {}
for image_file in image_files:
    image_path = os.path.join(image_dir, image_file)
    image_data[image_file.split('.')[0]] = encode_image(image_path)
    print(f"Encoded image: {image_file}")

def display_images(image_data: dict):
    fig, axs = plt.subplots(1, 3, figsize=(18, 6))
    for i, (key, value) in enumerate(image_data.items()):
        img = Image.open(BytesIO(base64.b64decode(value)))
        ax = axs[i]
        ax.imshow(img)
        ax.axis("off")
        ax.set_title(key)
    plt.tight_layout()
    plt.show()

display_images(image_data)
```

{{< postcover src="/posts/using_gpt4_vision_with_function_calling/packages.png" >}}


ما موفق به کدگذاری نمونه تصاویر به صورت رشته‌های base64 و نمایش آن‌ها شدیم. دستیار خدمات مشتری این تصاویر را تحلیل خواهد کرد تا اقدام مناسب را بر اساس وضعیت بسته تعیین کند.

حالا بیایید توابع/ابزارهای لازم برای پردازش سفارش، مانند ارجاع سفارش به نماینده، بازپرداخت سفارش و جایگزینی سفارش را تعریف کنیم. ما از مدل‌های `Pydantic` برای تعریف ساختار داده‌ها برای اقدامات سفارش استفاده خواهیم کرد.

```python
MODEL = "gpt-4-turbo"

class Order(BaseModel):
    """نمایانگر سفارشی با جزئیات مانند شناسه سفارش، نام مشتری، نام محصول، قیمت، وضعیت و تاریخ تحویل است."""
    order_id: str = Field(..., description="شناسه منحصر به فرد سفارش")
    product_name: str = Field(..., description="نام محصول")
    price: float = Field(..., description="قیمت محصول")
    status: str = Field(..., description="وضعیت سفارش")
    delivery_date: str = Field(..., description="تاریخ تحویل سفارش")

# توابع جایگزین برای پردازش سفارش
def get_order_details(order_id):
    return Order(
        order_id=order_id,
        product_name="Product X",
        price=100.0,
        status="Delivered",
        delivery_date="2024-04-10",
    )

def escalate_to_agent(order: Order, message: str):
    return f"Order {order.order_id} has been escalated to an agent with message: `{message}`"

def refund_order(order: Order):
    return f"Order {order.order_id} has been refunded successfully."

def replace_order(order: Order):
    return f"Order {order.order_id} has been replaced with a new order."

class FunctionCallBase(BaseModel):
    rationale: Optional[str] = Field(..., description="دلیل برای اقدام.")
    image_description: Optional[str] = Field(
        ..., description="توضیحات دقیق تصویر بسته."
    )
    action: Literal["escalate_to_agent", "replace_order", "refund_order"]
    message: Optional[str] = Field(
        ...,
        description="پیام به نماینده اگر اقدام به ارجاع به نماینده باشد",
    )

    def __call__(self, order_id):
        order: Order = get_order_details(order_id=order_id)
        if self.action == "escalate_to_agent":
            return escalate_to_agent(order, self.message)
        if self.action == "replace_order":
            return replace_order(order)
        if self.action == "refund_order":
            return refund_order(order)

class EscalateToAgent(FunctionCallBase):
    """ارجاع به نماینده برای کمک بیشتر."""
    pass

class OrderActionBase(FunctionCallBase):
    pass

class ReplaceOrder(OrderActionBase):
    """ابزار برای جایگزینی سفارش."""
    pass

class RefundOrder(OrderActionBase):
    """ابزار برای بازپرداخت سفارش."""
    pass
```

### شبیه‌سازی پیام‌های کاربر و پردازش تصاویر بسته‌ها
ما پیام‌های کاربر را که حاوی تصاویر بسته‌ها هستند شبیه‌سازی خواهیم کرد و تصاویر را با استفاده از مدل `GPT-4 Turbo`  پردازش خواهیم کرد. مدل ابزار مناسب را بر اساس تحلیل تصویر و اقدامات پیش‌فرض برای بسته‌های آسیب دیده، خیس یا عادی شناسایی خواهد کرد. سپس اقدام شناسایی شده را بر اساس شناسه سفارش پردازش خواهیم کرد و نتایج را نمایش خواهیم داد.

```python
ORDER_ID = "12345" 
INSTRUCTION_PROMPT = "You are a customer service assistant for a delivery service, equipped to analyze images of packages. If a package appears damaged in the image, automatically process a refund according to policy. If the package looks wet, initiate a replacement. If the package appears normal and not damaged, escalate to agent. For any other issues or unclear images, escalate to agent. You must always use tools!"

def delivery_exception_support_handler(test_image: str):
    payload = {
        "model": MODEL,
        "response_model": Iterable[RefundOrder | ReplaceOrder | EscalateToAgent],
        "tool_choice": "auto",  
        "temperature": 0.0,  
        "seed": 123, 
    }
    payload["messages"] = [
        {
            "role": "user",
            "content": INSTRUCTION_PROMPT,
        },
        {
            "role": "user",
            "content": [
                {
                    "type": "image_url",
                    "image_url": {
                        "url": f"data:image/jpeg;base64,{image_data[test_image]}"
                    }
                },
            ],
        }
    ]
    function_calls = instructor.from_openai(
        OpenAI(
            api_key=os.environ.get(("GILAS_API_KEY", "<کلید API خود را اینجا بسازید https://dashboard.gilas.io/apiKey>")), 
            base_url="https://api.gilas.io/v1/" # Gilas APIs
        ), mode=instructor.Mode.PARALLEL_TOOLS
    ).chat.completions.create(**payload)
    for tool in function_calls:
        print(f"- Tool call: {tool.action} for provided img: {test_image}")
        print(f"- Parameters: {tool}")
        print(f">> Action result: {tool(ORDER_ID)}")
        return tool

print("Processing delivery exception support for different package images...")

print("\n===================== Simulating user message 1 =====================")
assert delivery_exception_support_handler("damaged_package").action == "refund_order"

print("\n===================== Simulating user message 2 =====================")
assert delivery_exception_support_handler("normal_package").action == "escalate_to_agent"

print("\n===================== Simulating user message 3 =====================")
assert delivery_exception_support_handler("wet_package").action == "replace_order"
```


خروجی


{{< ltr >}}

Processing delivery exception support for different package images...
{{< /ltr >}}

{{< ltr >}}

===================== Simulating user message 1 =====================
- Tool call: refund_order for provided img: damaged_package
- Parameters: rationale='The package is visibly damaged with significant tears and crushing, indicating potential harm to the contents.' image_description='The package in the image shows extensive damage, including deep creases and tears in the cardboard. The package is also wrapped with extra tape, suggesting prior attempts to secure it after damage.' action='refund_order' message=None
>> Action result: Order 12345 has been refunded successfully.


{{< /ltr >}}

{{< ltr >}}

===================== Simulating user message 2 =====================
- Tool call: escalate_to_agent for provided img: normal_package
- Parameters: rationale='The package appears normal and not damaged, requiring further assistance for any potential issues not visible in the image.' image_description='A cardboard box on a wooden floor, appearing intact and undamaged, with no visible signs of wear, tear, or wetness.' action='escalate_to_agent' message='Please review this package for any issues not visible in the image. The package appears normal and undamaged.'
>> Action result: Order 12345 has been escalated to an agent with message: `Please review this package for any issues not visible in the image. The package appears normal and undamaged.`


{{< /ltr >}}

{{< ltr >}}

===================== Simulating user message 3 =====================
- Tool call: replace_order for provided img: wet_package
- Parameters: rationale='The package appears wet, which may have compromised the contents, especially since it is labeled as fragile.' image_description="The package in the image shows significant wetness on the top surface, indicating potential water damage. The box is labeled 'FRAGILE', which suggests that the contents are delicate and may be more susceptible to damage from moisture." action='replace_order' message=None
>> Action result: Order 12345 has been replaced with a new order.

{{< /ltr >}}

## 2. تحلیل نمودار سازمانی برای استخراج اطلاعات کارکنان

برای مثال دوم، ما نمودار سازمانی را تحلیل خواهیم کرد تا اطلاعات کارکنان، مانند نام‌ها، نقش‌ها، مدیران و نقش‌های مدیران را استخراج کنیم.

 ما از `GPT-4 Turbo` با قابلیت دیداری برای پردازش تصویر نمودار سازمانی و استخراج داده‌های ساختاریافته درباره کارکنان استفاده خواهیم کرد. در واقع، فراخوانی توابع به ما اجازه می‌دهد فراتر از OCR برویم و روابط سلسله مراتبی را درون نمودار ترجمه کنیم.

ما با یک نمونه نمودار سازمانی در فرمت PDF که می‌خواهیم تحلیل کنیم شروع خواهیم کرد و صفحه اول PDF را به یک تصویر JPEG برای تحلیل تبدیل خواهیم کرد.

```python
# تابع برای تبدیل صفحه PDF به تصویر JPEG
def convert_pdf_page_to_jpg(pdf_path: str, output_path: str, page_number=0):
    if not os.path.exists(pdf_path):
        raise FileNotFoundError(f"PDF file not found: {pdf_path}")
    doc = fitz.open(pdf_path)
    page = doc.load_page(page_number) 
    pix = page.get_pixmap()
    pix.save(output_path)

def display_img_local(image_path: str):
    img = Image.open(image_path)
    display(img)

pdf_path = 'data/org-chart-sample.pdf'
output_path = 'org-chart-sample.jpg'

convert_pdf_page_to_jpg(pdf_path, output_path)
display_img_local(output_path)
```


{{< postcover src="/posts/using_gpt4_vision_with_function_calling/chart.png" >}}


تصویر نمودار سازمانی با موفقیت از فایل PDF استخراج و نمایش داده شد. حالا بیایید تابعی برای تحلیل تصویر نمودار سازمانی با استفاده از `GPT-4 Turbo` تعریف کنیم. این تابع اطلاعاتی درباره کارکنان، نقش‌های آن‌ها و مدیران آن‌ها از تصویر استخراج خواهد کرد. ما از فراخوانی توابع/ابزارها برای مشخص کردن پارامترهای ورودی ساختار سازمانی، مانند نام کارمند، نقش و نام و نقش مدیر استفاده خواهیم کرد. ما از مدل‌های `Pydantic` برای تعریف ساختار داده‌ها استفاده خواهیم کرد.

```python
base64_img = encode_image(output_path)

class RoleEnum(str, Enum):
    """تعریف نقش‌های ممکن در یک سازمان."""
    CEO = "CEO"
    CTO = "CTO"
    CFO = "CFO"
    COO = "COO"
    EMPLOYEE = "Employee"
    MANAGER = "Manager"
    INTERN = "Intern"
    OTHER = "Other"

class Employee(BaseModel):
    employee_name: str = Field(..., description="نام کارمند")
    role: RoleEnum = Field(..., description="نقش کارمند")
    manager_name: Optional[str] = Field(None, description="نام مدیر، در صورت وجود")
    manager_role: Optional[RoleEnum] = Field(None, description="نقش مدیر، در صورت وجود")

class EmployeeList(BaseModel):
    """لیستی از کارکنان در ساختار سازمانی."""
    employees: List[Employee] = Field(..., description="لیستی از کارکنان")

def parse_orgchart(base64_img: str) -> EmployeeList:
    response = instructor.from_openai(OpenAI(
            api_key=os.environ.get(("GILAS_API_KEY", "<کلید API خود را اینجا بسازید https://dashboard.gilas.io/apiKey>")), 
            base_url="https://api.gilas.io/v1/" # Gilas APIs
        )).chat.completions.create(
        model='gpt-4-turbo',
        response_model=EmployeeList,
        messages=[
            {
                "role": "user",
                "content": 'Analyze the given organizational chart and very carefully extract the information.',
            },
            {
                "role": "user",
                "content": [
                    {
                        "type": "image_url",
                        "image_url": {
                            "url": f"data:image/jpeg;base64,{base64_img}"
                        }
                    },
                ],
            }
        ],
    )
    return response

# call the functions to analyze the organizational chart and parse the response
result = parse_orgchart(base64_img)

# tabulate the extracted data
df = pd.DataFrame([{
    'employee_name': employee.employee_name,
    'role': employee.role.value,
    'manager_name': employee.manager_name,
    'manager_role': employee.manager_role.value if employee.manager_role else None
} for employee in result.employees])

display(df)
```

{{< scrollbox >}}
{{< ltr >}}

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>employee_name</th>
      <th>role</th>
      <th>manager_name</th>
      <th>manager_role</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Juliana Silva</td>
      <td>CEO</td>
      <td>None</td>
      <td>None</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Kim Chun Hei</td>
      <td>CFO</td>
      <td>Juliana Silva</td>
      <td>CEO</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Chad Gibbons</td>
      <td>CTO</td>
      <td>Juliana Silva</td>
      <td>CEO</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Chiaki Sato</td>
      <td>COO</td>
      <td>Juliana Silva</td>
      <td>CEO</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Cahaya Dewi</td>
      <td>Manager</td>
      <td>Kim Chun Hei</td>
      <td>CFO</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Shawn Garcia</td>
      <td>Manager</td>
      <td>Chad Gibbons</td>
      <td>CTO</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Aaron Loeb</td>
      <td>Manager</td>
      <td>Chiaki Sato</td>
      <td>COO</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Drew Feig</td>
      <td>Employee</td>
      <td>Cahaya Dewi</td>
      <td>Manager</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Richard Sanchez</td>
      <td>Employee</td>
      <td>Cahaya Dewi</td>
      <td>Manager</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Sacha Dubois</td>
      <td>Intern</td>
      <td>Cahaya Dewi</td>
      <td>Manager</td>
    </tr>
    <tr>
      <th>10</th>
      <td>Olivia Wilson</td>
      <td>Employee</td>
      <td>Shawn Garcia</td>
      <td>Manager</td>
    </tr>
    <tr>
      <th>11</th>
      <td>Matt Zhang</td>
      <td>Intern</td>
      <td>Shawn Garcia</td>
      <td>Manager</td>
    </tr>
    <tr>
      <th>12</th>
      <td>Avery Davis</td>
      <td>Employee</td>
      <td>Aaron Loeb</td>
      <td>Manager</td>
    </tr>
    <tr>
      <th>13</th>
      <td>Harper Russo</td>
      <td>Employee</td>
      <td>Aaron Loeb</td>
      <td>Manager</td>
    </tr>
    <tr>
      <th>14</th>
      <td>Taylor Alonso</td>
      <td>Intern</td>
      <td>Aaron Loeb</td>
      <td>Manager</td>
    </tr>
  </tbody>
</table>

{{< /ltr >}}
{{< /scrollbox >}}

ما داده‌های استخراج شده از نمودار سازمانی را با موفقیت تحلیل کرده و در یک DataFrame نمایش دادیم. این روش به ما اجازه می‌دهد تا از قابلیت‌های `GPT-4 Turbo`  برای استخراج اطلاعات ساختاریافته از تصاویر، مانند نمودارهای سازمانی و دیاگرام‌ها، استفاده کنیم و داده‌ها را برای تحلیل بیشتر پردازش کنیم. با استفاده از فراخوانی توابع، ما می‌توانیم قابلیت مدل‌های چند‌حالته را برای انجام وظایف خاص یا فراخوانی توابع خارجی گسترش دهیم.