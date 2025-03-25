---
title: "آموزش پیکربندی Continue.dev"
description: "راهنمای جامع تنظیم افزونه Continue.dev برای تکمیل خودکار کد با استفاده از مدل Codestral و از طریق Gilas API، به همراه تنظیمات پیشرفته و پیکربندی config.json."
tags:
  - code auto-completion
  - tab completion
  - AI-integration
  - code-editor
  - continue.dev
  - gilas api
  - codestral
  - تکمیل کد
  - برنامه نویسی
  - هوش مصنوعی
  - افزونه
  - ویرایشگر کد
  - تکمیل خودکار کد
  - کدنویسی با هوش مصنوعی
  - پیکربندی
weight: 2000
og_image: "/posts/continue_dev_configuration/banner.png"
---

{{< postcover src="/posts/continue_dev_configuration/banner.png" >}}

پیکربندی افزونه continue.dev با استفاده از فایل config.json فرآیندی ساده است که در زیر آن را شرح داده ایم.

برای آشنایی بیشتر با اکستنشن یا افزونه‌ی Continue.dev پیشنهاد می‌دهیم مقاله‌ی [راهنمای جامع Continue.dev برای کدنویسی با هوش مصنوعی](/posts/continue_dev_extension_explained) را مطالعه کنید.

## پیکربندی افزونه‌ی Continue.dev 

توجه داشته باشید که هیچ نیازی به لاگین کردن برای تنظیم افزونه نیست. 

1. **ایجاد حساب کاربری در گیلاس:**

   - ابتدا یک [حساب کاربری جدید](https://dashboard.gilas.io) در گیلاس بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید.
   - سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey) بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.

2. **بار کردن فایل `config.json`**

ابتدا بر روی آیکون Continue در نوار کناری در IDE خود (VS Code یا JetBrains) کلیک کرده و سپس با کلیک بر روی نماد "چرخ دنده" در هدر صفحه تنظیمات باز میشود. سپس، روی دکمه‌ای با عنوان "Open Config File" کلیک کنید.

3. **افزودن مدل‌ به افزونه**

در فایل config.json  میتوانید افزونه را به مدل‌های متنوعی که پلتفرم هوش مصنوعی گیلاس  برای تکمیل خودکار کد یا چت و نوشتن کد ارایه می‌دهد متصل کنید. فقط کافی است هر مدل را با تنظیم صحیح آن به لیست `models` برای چت و تولید کد و یا به `tabAutocompleteModel` برای تکمیل خودکار کد اضافه کنید.

فایل زیر نمونه‌ی پیکربندی کامل اکستنشن Continue.dev با استفاده از APIهای گیلاس برای چت و تولید کد و همچنین تکمیل خودکار کد است.
برای استفاده از آن کافی است که آن را در فایل config.json خود کپی کرده و مقدار `Your-Gilas-Api-Key` را با کلید API ساخته شده در مرحله‌ی اول جایگزین کنید.

```json
{
  "models": [
    {
      "title": "Claude 3.7 Sonnet",
      "provider": "anthropic",
      "model": "claude-3-7-sonnet-latest",
      "apiBase": "https://api.gilas.io/v1",
      "apiKey": "Your-Gilas-Api-Key"
    },
    {
      "title": "Claude 3.5 Haiku",
      "provider": "anthropic",
      "model": "claude-3-5-haiku-latest",
      "apiBase": "https://api.gilas.io/v1",
      "apiKey": "Your-Gilas-Api-Key"
    },
    {
      "title": "GPT-4o",
      "provider": "openai",
      "model": "gpt-4o",
      "apiBase": "https://api.gilas.io/v1",
      "apiKey": "Your-Gilas-Api-Key"
    },
    {
      "title": "o3-mini",
      "provider": "openai",
      "model": "o3-mini",
      "apiBase": "https://api.gilas.io/v1",
      "apiKey": "Your-Gilas-Api-Key"
    },
    {
      "title": "GPT-4o-mini",
      "provider": "openai",
      "model": "gpt-4o-mini",
      "apiBase": "https://api.gilas.io/v1",
      "apiKey": "Your-Gilas-Api-Key"
    },
    {
      "title": "Deepseek Chat",
      "provider": "openai",
      "model": "deepseek-chat",
      "apiBase": "https://api.gilas.io/v1",
      "apiKey": "Your-Gilas-Api-Key"
    },
    {
      "title": "Deepseek Reasoner",
      "provider": "openai",
      "model": "deepseek-reasoner	",
      "apiBase": "https://api.gilas.io/v1",
      "apiKey": "Your-Gilas-Api-Key"
    }
  ],
  "tabAutocompleteModel": {
    "title": "Codestral",
    "provider": "mistral",
    "model": "codestral-latest",
    "apiBase": "https://api.gilas.io/v1",
    "apiKey": "Your-Gilas-Api-Key"
  },
  "slashCommands": [
    {
      "name": "edit",
      "description": "Edit selected code"
    },
    {
      "name": "comment",
      "description": "Write comments for the selected code"
    },
    {
      "name": "share",
      "description": "Export the current chat session to markdown"
    },
    {
      "name": "cmd",
      "description": "Generate a shell command"
    },
    {
      "name": "commit",
      "description": "Generate a git commit message"
    }
  ],
  "docs": [],
  "contextProviders": [
    {
      "name": "url"
    }
  ]
}
```
لطفا مطمین شوید که پس از ذخیره‌ی فایل خطایی در خصوص خطا وجود در فایل دریافت نکنید.

حال با باز کردن محیط Continue باید لیست مدل‌های ثبت شده برای چت را نیز مشاهده کنید.

{{< img image="/posts/codestral_auto_completions_setup/result.png" alt="تکمیل خودکار کد" width="750px">}}

همچنین تکمیل شدن خودکار کد نیز باید در حین تایپ کردن به صورت اتوماتیک اتفاق بیوفتد.

## سوالات متداول در مورد اکستنشن Continue.dev


{{< collapsible header="آیا برای استفاده از اکستنشن Continue.dev باید اول لاگین کنم؟" headerStyle="font-weight:bold; margin-top:10px" arrowText="نمایش جواب">}}
 خیر. برای استفاده از اکستنشن نیازی به لاگین کردن به اکستنشن نیست. فقط کافیه از طریق پنل کاربری گیلاس کلید API خودتون رو بسازین و طبق راهنمای بالا اون رو در فایل `config.json` استفاده کنید.
{{< /collapsible >}}

{{< collapsible header="کدام مدل‌های تولید کد را می‌شود به اکستنشن Continue.dev وصل کرد؟ "  headerStyle="font-weight:bold; margin-top:10px" arrowText="نمایش جواب">}}
 تمام [مدل‌های تولید متن](https://gilas.io/models/#text-generation) که از طریق پلتفرم گیلاس در دسترس قرار دارند را می‌توان به اکستنشن اضافه کرد. برای این کار کافی هست که مدل‌های مورد نظر خود را به لیست `models` در فایل `config.json` اضافه کنید.
{{< /collapsible >}}


{{< collapsible header="چطور از اتصال صحیح اکستنشن به APIهای گیلاس مطمین شوم؟"  headerStyle="font-weight:bold; margin-top:10px" arrowText="نمایش جواب">}}
 برای این کار کافی‌ست که از طریق نوار ابزار کنار ادیتور اکستنشن را باز کنید و در محیط چت سوالی را بپرسید. در صورت اتصال صحیح اکستنشن به APIهای گیلاس، پاسخ سوالی که پرسیده‌اید را دریافت خواهید کرد. در غیر این صورت، پیغام خطایی دریافت خواهید کرد. در صورت دریافت پیام خطا فایل `config.json`را بررسی کنید و مطمین شوید که از الگوی ارایه شده در بالا پیروی می‌کند. همچنین مطمین شوید که کلید API خود را به درستی وارد کرده‌اید و همچنین مقدار `apiBase` را برای هر مدل معادل آدرس APIهای گیلاس قرار داده اید.
{{< /collapsible >}}


{{< collapsible header="چرا تکمیل خودکار کد پس از مدتی دیگر کار نمی‌کند؟ " headerStyle="font-weight:bold; margin-top:10px" arrowText="نمایش جواب">}}
در صورتی که تکمیل خودکار کد قبلا کار میکرده، احتمالا اعتبار کیف پول شما در پلتفرم هوش مصنوعی گیلاس تمام شده است. برای شارژ‌ مجدد کیف پول خود کافی‌ست به [پنل کاربری گیلاس](https://dashboard.gilas.io) مراجعه کنید و کیف پول خود را شارژ کنید.
دلیل دیگر می‌تواند  تغییرات در فایل `config.json` باشد. برای رفع این مشکل، فایل `config.json` را بررسی کنید و مطمئن شوید که همه تنظیمات به درستی انجام شده‌اند.
{{< /collapsible >}}


{{< collapsible header="آیا امکان استفاده از مدل‌های دیگر مثل OpenAI یا Anthtopic برای تکمیل خودکار کد وجود دارد؟ " headerStyle="font-weight: bold; margin-top:10px" arrowText="نمایش جواب">}}
خیر. هیچکدام از مدل‌های شرکت‌های OpenAI یا Anthropic برای تکمیل خودکار کد آموزش ندیده‌اند و تنها مدل `codestral` از شرکت Mistral قابلیت تکمیل خودکار کد با کیفیت بالا را دارد.
{{< /collapsible >}}

