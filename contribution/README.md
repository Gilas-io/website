# راهنمای مشارکت در تولید پست‌های بلاگ

قبل شروع راهنمای مشارکت در بلاگ وب‌سایت گیلاس ❤️ قدردان و متشکر ❤️ هستیم که ما را در دموکراتیک کردن هوش مصنوعی در جامعه ایران یاری می‌کنید.

این وب‌سایت توسط فریم‌ورک (HUGO)[https://gohugo.io/] ساخته شده است. جهت ران کردن وب‌سایت در محیط لوکال خود می‌تونید با استفاده از (راهنمای نصب)[https://gohugo.io/installation/] ابتدا Hugo را روی کامپیوتر خود نصب کنید و بعد از آن این رپو را clone کرده و در مسیر root رپو دستور زیر را اجرا کنید:

```
hugo serve
```

**اجرای وب‌سایت به صورت لوکال اختیاری است و پیش‌نیازی برای مشارکت در تولید پست‌های بلاگ نیست.**



## پیش‌نیازها

 ابتدا یک Fork از این repo تهیه کنید. از این پس بعد از اعمال تغییرات روی رپوی فورک شده آنها را با باز کردن یک Pull Request بر روی رپوی اصلی٬ به برنچ master اضافه خواهید کرد.

## پیشنهاد پست برای ترجمه
 اگر قصد ترجمه پستی مرتبط با LLMها را دارید ابتدا آن را در صفحه‌ی  [پست‌های پیشنهادی](/blog-posts.md) ثبت کنید تا از ترجمه مجدد پست‌های تکراری جلوگیری شود.
 برای اینکار پست‌ مورد نظر خود را به فرمت ارایه شده در صفحه پست‌های پیشنهادی و با وضعیت "منتظر مترجم" از طریق باز کردن یک PR جدید پیشنهاد دهید. بعد از بررسی محتوای پست٬ درخواست شما تایید می‌شود و پست پیشنهادی به لیست اضافه می‌شود.


## مشارکت در ترجمه پست‌ها

برای مشارکت در ترجمه پست‌ها ابتدا صفحه [پست‌های پیشنهادی](/blog-posts.md) را بررسی کنید و یک پست را به دلخواه خود انتخاب کنید. 

در صورتی که وضعیت پست انتخابی "در حال ترجمه" باشد٬ می‌توانید در ترجمه آن پست با کامیت کردن روی PR باز شده برای آن پست مشارکت کنید. در این صورت یک پست ممکن است توسط بیش از یک نفر ترجمه شود که باعث تسریع روند ترجمه آن پست خواهد شد.

در صورتی که قصد ترجمه پستی با وضعیت "منتظر مترجم" را دارید:

-  ابتدا در رپوی Fork شده خود یک برنچ برای این پست درست کنید.
-  سپس در مسیر `content/posts/` یک فایل خالی با اسم متناسب با آن پست تولید کنید.
- حالا یک PR بر روی رپوی اصلی باز کرده و اسم آن را حتما با `[DRAFT]` شروع کنید. (نمونه)[https://github.com/Gilas-io/website/pull/2]

تا اینجای کار شما یک PR جدید برای مشارکت در ترجمه پست انتخابی خود باز کرده‌اید که برای دیگر کاربران هم قابل دیدن است.

حال باید فایل [پست‌های پیشنهادی](/blog-posts.md) بروزرسانی کنید تا به دیگران اطلاع دهید که شما در حال ترجمه این پست هستید. برای این کار:

-  ابتدا در رپوی Fork شده خود یک برنچ جدید درست کنید.
- بعد از آن به صفحه [پست‌های پیشنهادی](/blog-posts.md)  رفته و  وضعیت پست انتخابی خود را به "در حال ترجمه" تغییر دهید و شماره PR باز شده در مرحله قبل را برای آن ثبت کنید.
- اسم PR جدید را `Update blog-posts.md` بگذارید.

با مرج شدن این PR دیگران متوجه خواهند شد که شما در حال ترجمه کدام پست هستید و می‌توانند در این کار به شما کنند.


## راهنمای ترجمه پست‌ها

برای ترجمه پست‌ها نیازی نیست که همه کار را خودتان انجام دهید. شما می‌توانید با استفاده از یک LLM پیش‌نویس اول را به صورت اتوماتیک به فارسی ترجمه کنید. برای این کار می‌توانید از سرویس رایگان (Google Gemini)[https://gemini.google.com/app] استفاده کنید.

پیشنهاد ما این است که از پرامپت زیر برای ترجمه متن اصلی به فارسی استفاده کنید.

```
Read the following text, enclosed within <text></text> tags, thoroughly to understand its content. \n
Then, translate it into fluent and natural Farsi language, maintaining the original information flow and including all essential information. \n
Retain all technical terms in English, as the intended audience is software engineers.\n

<text>
متن انگلیسی را اینجا قرار دهید.
</text>

```

 از آنجا که ممکن است متن تولید شده کاملا روان و طبیعی به نظر نیاید٬ لازم است که آن را ویرایش کنید تا به فارسی روان تبدیل شود.

### ساخت بدنه‌ی فایل پست

هر پست در قالب یک فایل با فرمت `.md` ساخته می‌شود که هدر آن باید از فرمت زیر پیروی کند.

```
---
title: "عنوان پست"
description: "توضیحاتی در مورد محتوای پست که توسط موتورهای جستجو استفاده می‌شود"

# لیستی از تگ‌ها که مربوط به محتوای پست می‌شوند و برای دسته‌بندی پست‌ها استفاده می‌شوند. چند مثال از تگ‌ها در زیر آمده است:
tags:
- text-to-speech
- image-processing
- ...
# اگر تصویری مناسب با محتوای پست دارید٬ ابتدا در مسیر `static/posts` یک فولدر با نام اسم فایل پست بسازید و بعد آن تصویر را با نام `banner.*` در آنجا قرار دهید. بعد از آن مسیر عکس را بدون `static` در هدر زیر قرار دهید. مثال:
og_image: "/posts/<file-name>/banner.png" 
---

<<محتوای پست اینجا قرار می‌گیرید>>
```

برای مثال می‌توانید پست‌هایی را که قبلا ساخته شده‌اند بررسی کنید تا با ساختار یک فایل پست بیشتر آشنا شوید.


### ابزارهای فرمت‌دهی متن

برای فرمت‌دهی مناسب به محتوای پست می‌توانید از ابزارهای زیر استفاده کنید:

**فرمت‌دهی کد**:

برای نمایش کد در فرمت مناسب لطفا آن را داخل به شکل زیر بنویسید. مثال:

```
    ```python
    << code >>
    ```
```

**نمایش tooltip**:

با استفاده از این کد کوتاه می توانید یک {{< tooltip text="هاور متن" note="text hover" >}} برای کلمه مورد نظر خود ایجاد کنید.

```
{{< tooltip text="مدل زبانی بزرگ" note="Large Language Model" >}} 
```

**تولید tab**

{{< tabs "tabs" >}}

{{< tab "تب اول" >}}
برای نمایش محتوا به صورت تب کافی است که از فرمت زیر استفاده کنید:

```
{{< tabs "uniqueid" >}}

{{< tab "لینوکس" >}}
متن تست
{{< /tab >}}
{{< tab "ویندوز" >}}
متن تست ۲
{{< /tab >}}

{{< /tabs >}}
```
{{< /tab >}}
{{< tab "تب دوم" >}}
محتوا
{{< /tab >}}

{{< /tabs >}}

**راهنما**

{{< hint info >}}
برای نمایش متن به صورت راهنما می‌توانید از فرمت زیر استفاده کنید:
{{< /hint >}} 

```
{{< hint info | warning | danger >}}
با انتخاب هر یک از مقادیر info, warning و یا danger رنگ باکسی که متن در داخل آن قرار دارد تغییر می‌کند.
{{< /hint >}} 


```

**اسکرول باکس**

در صورتی که بخشی از محتوا خیلی طولانی است و خواندن آن برای یادگیری ضروری نیست٬ مثلا خروجی کد یا نمایش فایل csv٬ می‌توانید آن بخش از محتوا را داخل یک اسکرول باکس قرار دهید تا ارتفاع آن بیشتر از حد مجاز نشود.

```
{{< scrollbox >}}
محتوای خیلی طولانی
{{< /scrollbox >}}
```

مثال:

{{< scrollbox >}}

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.   

Duis autem vel eum iriure dolor in hendrerit in vulputate velit esse molestie consequat, vel illum dolore eu feugiat nulla facilisis at vero eros et accumsan et iusto odio dignissim qui blandit praesent luptatum zzril delenit augue duis dolore te feugait nulla facilisi. Lorem ipsum dolor sit amet, consectetuer adipiscing elit, sed diam nonummy nibh euismod tincidunt ut laoreet dolore magna aliquam erat volutpat.   

Ut wisi enim ad minim veniam, quis nostrud exerci tation ullamcorper suscipit lobortis nisl ut aliquip ex ea commodo consequat. Duis autem vel eum iriure dolor in hendrerit in vulputate velit esse molestie consequat, vel illum dolore eu feugiat nulla facilisis at vero eros et accumsan et iusto odio dignissim qui blandit praesent luptatum zzril delenit augue duis dolore te feugait nulla facilisi.   

{{< /scrollbox >}}

**فایل صوتی**

برای نمایش و پخش فایل صوتی ابتدا با استفاده از https://base64.guru/converter/encode/audio آن را به فرمت `base64`تبدیل کرده و سپس به شکل زیر در صفحه قرار دهید. هنگام تبدیل فایل از طریق ابزار ذکر شده مطمین شوید که نوع خروجی را برابر با `Data URI - data:content/type;base64` قرار دهید.

```
<audio controls="controls">
                    <source src="data:audio/wav;base64,.......
</audio>
```


**چند ستونی**

{{< columns >}}
برای محتوا در چند ستون کنار هم می‌توانید آن را به صورت فوق فرمت‌دهی کنید.

<--->

```
{{< columns >}}

ستون اول
<--->
ستون دوم
<--->
ستون سوم

{{< /columns >}}
```
{{< /columns >}}


## درخواست بررسی پست

زمانی که ترجمه و ویرایش یک پست به پایان رسید٬ کامیت‌های نهایی را به PR خود اضافه کنید و کلمه `[DRAFT]` در عنوان PR را به `[REVIEW]` تغییر دهید. در این صورت ما بازبینی نهایی متن پست را انجام می‌دهیم و آن را در برنچ master مرج می‌کنیم.



