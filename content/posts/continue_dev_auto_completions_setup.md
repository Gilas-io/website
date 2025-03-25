---
title: " اتصال افزونه‌ی Continue.dev به Gilas API برای تکمیل خودکار کد"
description: "آموزش تنظیم افزونه‌ی محبوب Continue.dev برای تکمیل خودکار کد در با استفاده از مدل Codestral و از طریق Gilas API."
tags:
- code auto-completion
- tab completion
- AI-integration
- code-editor
- continue.dev
- تکمیل کد
- برنامه نویسی
- هوش مصنوعی
- افزونه
- ویرایشگر کد
- تکمیل خودکار کد
- کدنویسی با هوش مصنوعی
weight: 2000
og_image: "/posts/codestral_auto_completions_setup/banner.jpg"
---

{{< postcover src="/posts/codestral_auto_completions_setup/banner.jpg" >}}

درین پست نحوه تنظیم ابزار محبوب [Continue.dev](https://www.continue.dev/) برای تکمیل خودکار کد code auto-completion که گاهی از آن به عنوان Tab completion هم یاد می‌شود را با استفاده از مدل Codestral از طریق Gilas API بررسی میکنیم.

ابزارهای تکمیل خودکار کد برای تکمیل خطوط کد باز یا ناقص طراحی شده است و به شما امکان می‌دهد به سرعت و با دقت بیشتری کد خود را تکمیل کنید. مدل Codestral که توسط Gilas API در دسترس است٬ یک مدل پیشرفته تولید کد است که به طور خاص برای وظایف تولید کد مانند تکمیل و تولید کد در بین خطوط بهینه‌سازی شده است.
همچنین این مدل قابلیت تکمیل متن به زبان انگلیسی٬ فارسی و غیره را نیز دارد که در نوشتن کامنت یا متون دیگر می‌تواند مورد استفاده قرار بگیرد.

 برای آگاهی از تمام امکانات این اکستنشن و نحوه‌ی استفاده بهینه از آن مقاله‌ی [راهنمای جامع Continue.dev برای کدنویسی با هوش مصنوعی](/posts/continue_dev_extension_explained) را مطالعه کنید 

## تنظیم اکستنشن Continue.dev

**توجه:** برای پیکربندی اکستنشن‌ نیازی به لاگین کردن به Continue.dev ندارید.

افزونه‌ی [Continue.dev](https://www.continue.dev/) یک افزونه برای ویرایشگرهای VS Code, Cursor و JetBrains است که به شما امکان می‌دهد به سرعت و با دقت بیشتری کد خود را تکمیل کنید. این افزونه همچنین امکان چت و تولید کد را نیز دارد.

### مراحل فعال‌سازی

1. **ایجاد حساب کاربری در گیلاس:**
   - ابتدا یک [حساب کاربری جدید](https://dashboard.gilas.io) در گیلاس بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید.
   - سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey) بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.

2. **نصب افزونه‌ی Continue.dev:**
   - افزونه‌ی Continue.dev را در ویرایشگر مورد نظر خود نصب کنید. (راهنما: [نصب افزونه](https://docs.continue.dev/getting-started/install))

3. **تنظیمات افزونه:**
   - بر روی آیکون continue.dev در نوار کناری ویرایشگر کلیک کنید٬ سپس بر روی آیکون تنظیمات در بالا سمت راست یا چپ کلیک کرده و در نمای جدید بر روی دکمه‌ی `Open Config File` کلیک کنید.

{{< img image="/posts/codestral_auto_completions_setup/step1.png" alt="تنظیمات افزونه" width="750px">}}

4. **پیکربندی مدل Codestral:**
   - در فایل `config.json` باز شده، مدل Codestral را در لیست `TabAutoCompleteModel` همراه با آدرس Gilas API Url و کلید API ساخته شده در مرحله اول قرار دهید. 

```json
  {
   "tabAutocompleteModel": {
    "title": "Codestral",
    "provider": "mistral",
    "model": "codestral-latest",
    "apiBase": "https://api.gilas.io/v1",
    "apiKey": "Your-Gilas-Api-Key"
  },
}
```

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

{{< hint info >}}
برای پیکربندی کامل افزونه جهت تکمیل خودکار کد و چت با مدل‌های تولید کد لطفا مقاله‌ی [پیکربندی افزونه‌ی Continue.dev](/posts/continue_dev_configuration) را مطالعه کنید.
{{< /hint >}}

پس از ذخیره فایل، باید تکمیل شدن خطوط برنامه به صورت خودکار اتفاق بیفتد. لطفا مطمین شوید که پس از ذخیره‌ی فایل خطایی در خصوص خطا وجود در فایل دریافت نکنید.

همچنین با باز کردن محیط Continue باید لیست مدل‌های ثبت شده برای چت را نیز مشاهده کنید.

{{< img image="/posts/codestral_auto_completions_setup/result.png" alt="تکمیل خودکار کد" width="750px">}}


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