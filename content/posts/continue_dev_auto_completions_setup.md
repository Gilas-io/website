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
weight: 2001
og_image: "/posts/codestral_auto_completions_setup/banner.jpg"
---

{{< postcover src="/posts/codestral_auto_completions_setup/banner.jpg" >}}

درین پست نحوه تنظیم ابزار محبوب [Continue.dev](https://www.continue.dev/) برای تکمیل خودکار کد code auto-completion که گاهی از آن به عنوان Tab completion هم یاد می‌شود را با استفاده از مدل Codestral از طریق Gilas API بررسی میکنیم.

ابزارهای تکمیل خودکار کد برای تکمیل خطوط کد باز یا ناقص طراحی شده است و به شما امکان می‌دهد به سرعت و با دقت بیشتری کد خود را تکمیل کنید. مدل Codestral که توسط Gilas API در دسترس است٬ یک مدل پیشرفته تولید کد است که به طور خاص برای وظایف تولید کد مانند تکمیل و تولید کد در بین خطوط بهینه‌سازی شده است.
همچنین این مدل قابلیت تکمیل متن به زبان انگلیسی٬ فارسی و غیره را نیز دارد که در نوشتن کامنت یا متون دیگر می‌تواند مورد استفاده قرار بگیرد.

{{< hint info >}}
برای آگاهی بیشتر در مورد قابلیت‌های مدل Codestral به پست مربوط به [معرفی مدل Codestral](/posts/introduction_to_codestral) مراجعه کنید.
{{< /hint >}} 


## تنظیم افزونه‌ی Continue.dev

افزونه‌ی [Continue.dev](https://www.continue.dev/) یک افزونه برای ویرایشگرهای VS Code, Cursor و JetBrains است که به شما امکان می‌دهد به سرعت و با دقت بیشتری کد خود را تکمیل کنید. این افزونه همچنین امکان چت و تولید کد را نیز دارد.

### مراحل فعال‌سازی

1. **ایجاد حساب کاربری در گیلاس:**
   - ابتدا یک [حساب کاربری جدید](https://dashboard.gilas.io) در گیلاس بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید.
   - سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey) بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.

2. **نصب افزونه‌ی Continue.dev:**
   - افزونه‌ی Continue.dev را در ویرایشگر مورد نظر خود نصب کنید. (راهنما: [نصب افزونه](https://docs.continue.dev/getting-started/install))

3. **تنظیمات افزونه:**
   - بر روی آیکون continue.dev در نوار کناری ویرایشگر کلیک کنید و به صفحه تنظیمات بروید.

{{< img image="/posts/codestral_auto_completions_setup/step1.png" alt="تنظیمات افزونه" width="750px">}}

4. **پیکربندی مدل Codestral:**
   - در فایل `config.json` باز شده، مدل Codestral را در لیست `TabAutoCompleteModel` همراه با آدرس Gilas API Url و کلید API ساخته شده در مرحله اول قرار دهید. همچنین می‌توانید از دیگر مدلهای گیلاس برای چت کردن (لیست `models`) نیز استفاده کنید.

```json
// config.json
{
  "models": [
    {
      "title": "O1 mini",
      "model": "o1-mini",
      "provider": "openai",
      "apiBase": "https://api.gilas.io/v1",
      "apiKey": "Your-Gilas-Api-Key"
    },
    {
      "title": "GPT-4o",
      "model": "gpt-4o",
      "provider": "openai",
      "apiBase": "https://api.gilas.io/v1",
      "apiKey": "Your-Gilas-Api-Key"
    },
    {
      "title": "GPT-4o Mini",
      "model": "gpt-4o-mini",
      "provider": "openai",
      "apiBase": "https://api.gilas.io/v1",
      "apiKey": "Your-Gilas-Api-Key"
    },
    {
      "title": "Mistral Large",
      "model": "mistral-large-latest",
      "provider": "mistral",
      "apiBase": "https://api.gilas.io/v1",
      "apiKey": "Your-Gilas-Api-Key"
    },
    {
      "title": "Mistral Small",
      "model": "mistral-small-latest",
      "provider": "mistral",
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
  
  // ادامه‌ی فایل
  // ...
}
```

پس از ذخیره فایل، باید تکمیل شدن خطوط برنامه به صورت خودکار اتفاق بیفتد.

همچنین با باز کردن محیط Continue باید لیست مدل‌های ثبت شده برای چت را نیز مشاهده کنید.

{{< img image="/posts/codestral_auto_completions_setup/result.png" alt="تکمیل خودکار کد" width="750px">}}

## نتیجه‌گیری

با دنبال کردن این مراحل، می‌توانید از قابلیت‌های پیشرفته مدل Codestral برای تکمیل خودکار کد در ویرایشگرهای مختلف بهره‌مند شوید. برای اطلاعات بیشتر و راهنمایی‌های بیشتر، به سایر مقالات ما مراجعه کنید.
