---
title: "ساخت یک Chatbot Agent با Node.js"
description: "مدل‌های چت، یگ سری از پیام‌ها را به عنوان ورودی می‌پذیرند و یک پیام نوشته شده توسط AI را به عنوان خروجی برمی‌گردانند.
این راهنما با چند نمونه فراخوانی API فرمت چت را نشان می‌دهد."
tags:
- openai
- chatbot
- function-call
- node.js
- gilas.io
- blog
weight: 2004
og_image: "/posts/how_to_build_an_agent_with_the_node_sdk/banner.jpg" 
---

{{< postcover src="/posts/how_to_build_an_agent_with_the_node_sdk/banner.png" >}}


 قابلیت فراخوانی تابع در مدل‌های GPT به برنامه شما اجازه می دهد توابع داخلی برنامه را بر اساس ورودی های کاربر فراخوانی کند. این به این معنی است که برنامه می تواند عملیات مختلفی از جمله، جستجو در وب، ارسال ایمیل، یا رزرو بلیط از طرف کاربران را انجام دهد، که این امر برنامه شما را قدرتمندتر از یک چت بات معمولی می کند. 
 
 در این پست، شما برنامه‌ای می سازید که از آخرین نسخه از OpenAI SDK Node.js استفاده می کند. این برنامه در مرورگر اجرا می شود،

 اگر Node.js بر روی کامپیوتر شما نصب نیست٬ می‌توانید از طریق [Scrimba](https://scrimba.com/scrim/c6r3LkU9) کد برنامه را نوشته و اجرا کنید.


 ## نحوه عملکرد برنامه
  
این برنامه یک عامل یا Agent ساده است که به شما کمک می کند تا فعالیت های مورد علاقه‌تان را در اطراف خود پیدا کنید.
   
این برنامه دسترسی به دو تابع، `getLocation()` و `getCurrentWeather()` دارد، که به این معنی است که می تواند متوجه شود که شما کجا قرار دارید و هوا در حال حاضر چگونه است.
    
لازم به ذکر است که GPT هیچ کدی را برای شما اجرا نمی کند. فقط به برنامه شما می گوید که کدام توابع را باید در یک سناریو مشخص استفاده کند، و فراخوانی آنها را به برنامه واگذار می کند.
     
هنگامی که Agent مکان شما و هوا را می داند، از دانش داخلی GPT برای پیشنهاد فعالیت های مناسب محلی برای شما استفاده می کند.

 
 {{< hint info >}}
**برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.**
{{< /hint >}} 

## نوشتن کد برنامه

ابتدا یک کلاینت از OpenAI بسازید.

```js
import OpenAI from "openai";
 
const openai = new OpenAI({
  apiKey: process.env.GILAS_API_KEY, // <کلید API خود را اینجا بسازید https://dashboard.gilas.io/apiKey>
  baseUrl: "https://api.gilas.io/v1/",
  dangerouslyAllowBrowser: true,
});
```
اگر کد خود را در محیط مرورگر در Scrimba اجرا می کنید٬ مقدار متغیر `dangerouslyAllowBrowser: true`    را تنظیم کنید.

### ساخت دو تابع برای برنامه

ابتدا یک تابع به نام `getLocation` می‌سازیم که یا استفاده از سرویس [IP API](https://ipapi.co/) محل یوزر را تشخیص می‌دهد.

```js
async function getLocation() {
  const response = await fetch("https://ipapi.co/json/");
  const locationData = await response.json();
  return locationData;
}
```

سرویس IP API مجموعه ای از داده ها در مورد مکان شما را برمی گرداند، از جمله عرض و طول جغرافیایی که ما آنها را به عنوان آرگومان ها در تابع دوم `getCurrentWeather` استفاده می کنیم. ما از [Open Meteo API](https://open-meteo.com/) برای دریافت داده های هوای فعلی استفاده می کند.

```js
async function getCurrentWeather(latitude, longitude) {
  const url = `https://api.open-meteo.com/v1/forecast?latitude=${latitude}&longitude=${longitude}&hourly=apparent_temperature`;
  const response = await fetch(url);
  const weatherData = await response.json();
  return weatherData;
}
```

### معرفی توابع به مدل GPT

برای اینکه مدل GPT هدف این توابع را بفهمد، ما باید آنها را برای مدل توصیف کنیم. برای این کار٬ یک آرایه به نام `tools` ایجاد می کنیم که شامل یک شیء برای هر تابع است. هر شیء دو کلید خواهد داشت: `type`, `function` و کلید `function` سه زیرکلید دارد: `name`, `description` و `parameters`.

در زیر نحوه توصیف هر یک از توابع که شامل اسم تابع٬ آرگومان‌های ورودی و توضیحی در مورد عملکرد تابع است را مشاهده می‌کنید.

```js
const tools = [
  {
    type: "function",
    function: {
      name: "getCurrentWeather",
      description: "Get the current weather in a given location",
      parameters: {
        type: "object",
        properties: {
          latitude: {
            type: "string",
          },
          longitude: {
            type: "string",
          },
        },
        required: ["longitude", "latitude"],
      },
    }
  },
  {
    type: "function",
    function: {
      name: "getLocation",
      description: "Get the user's location based on their IP address",
      parameters: {
        type: "object",
        properties: {},
      },
    }
  },
];
```

### ساخت آرایه‌ای از پیام‌ها

ما همچنین باید یک آرایه `messages` تعریف کنیم. این آرایه همه پیام های رفت و برگشت بین برنامه ما و مدل را پیگیری می کند. اولین شیء در آرایه باید همیشه یک پیغام سیستمی باشد تا  به مدل بگوید چگونه رفتار کند.

```js
const messages = [
  {
    role: "system",
    content:
      "You are a helpful assistant. Only use the functions you have been provided with.",
  },
];
```

### ساخت تابع agent

اکنون آماده ایم تا منطق برنامه خود را که در تابع `agent` قرار دارد ، بسازیم. این تابع یک آرگومان به نام `userInput` را می گیرد.

کار را با افزودن `userInput` به آرایه پیام ها شروع می کنیم. این بار، `role` این پیام را برابر با `user` قرار می‌دهیم تا مدل بداند که این ورودی از کاربر است.
 
```js
async function agent(userInput) {
  messages.push({
    role: "user",
    content: userInput,
  });
  const response = await openai.chat.completions.create({
    model: "gpt-4-turbo",
    messages: messages,
    tools: tools,
  });
  console.log(response);
}
```

سپس، یک درخواست به اندپوینت Chat completions از طریق متود `chat.completions.create()` در SDK Node ارسال می کنیم.

 این متود یک شیء با مقادیر زیر را به عنوان آرگومان می گیرد.
 
 - `model`: تصمیم می گیرد که از کدام مدل GPT استفاده کند (در مورد اینجا `gpt-4-turbo`).
 - `messages`: تاریخچه کامل پیام ها بین کاربر و مدل تا این لحظه. 
 - `tools`: لیستی از ابزارهایی که مدل ممکن است فراخوانی کند. در حال حاضر، فقط توابع به عنوان یک ابزار پشتیبانی می شوند.
 

## اجرای برنامه

حال می‌خواهیم سعی کنیم تا agent را با یک ورودی که نیاز به یک فراخوانی تابع برای دادن پاسخ مناسب به سوال ما را دارد، اجرا کنیم.

```js
agent("محل جغرافیایی من کجاست؟");
```

با اجرای کد بالا پاسخ زیر در کنسول چاپ می‌شود.

<div style="direction: ltr" >

```js
{
    id: "chatcmpl-84ojoEJtyGnR6jRHK2Dl4zTtwsa7O",
    object: "chat.completion",
    created: 1696159040,
    model: "gpt-4-turbo",
    choices: [{
        index: 0,
        message: {
            role: "assistant",
            content: null,
            tool_calls: [
              id: "call_CBwbo9qoXUn1kTR5pPuv6vR1",
              type: "function",
              function: {
                name: "getLocation",
                arguments: "{}"
              }
            ]
        },
        logprobs: null,
        finish_reason: "tool_calls" // Model wants us to call a function
    }],
    usage: {
        prompt_tokens: 134,
        completion_tokens: 6,
        total_tokens: 140
    }
     system_fingerprint: null
}
```

</div>

این پاسخ به ما می گوید که برنامه باید یکی از توابع خود را فراخوانی کند، زیرا مقدار: `finish_reason` برابر با `tool_calls` است. نام تابع  در کلید `response.choices[0].message.tool_calls[0].function.name` قرار دارد که برابر با `getLocation` ٬تابعی از برنامه که ما به مدل معرفی کرده‌ایم٬ است.

### فراخوانی تابع در داخل برنامه

حال که مدل تابع متناسب با ورودی کاربر که باید فراخوانی شود را به برگرداند٬ باید آن را در داخل برنامه فراخوانی کنیم.

برای این کار نام توابعی که به مدل معرفی کرده ایم را در یک ثابت به نام `availableTools` ذخیره می‌کنیم.

```js
const availableTools = {
  getCurrentWeather,
  getLocation,
};
```

و حال با استفاده از نام تابع برگردانده شده از سوی مدل می‌توانیم تابع اصلی را پیدا کرده و آن را با پارامترهای مورد نیاز که آنها هم توسط مدل تولید شده اند فراخوانی کنیم.

البته در این جا هنوز نیازی به فراخوانی تابع با مقادیر آرگومان‌ها نیست.

```js
const { finish_reason, message } = response.choices[0];
 
if (finish_reason === "tool_calls" && message.tool_calls) {
  const functionName = message.tool_calls[0].function.name;
  const functionToCall = availableTools[functionName];
  const functionArgs = JSON.parse(message.tool_calls[0].function.arguments);
  const functionArgsArr = Object.values(functionArgs);
  const functionResponse = await functionToCall.apply(null, functionArgsArr);
  console.log(functionResponse);
}
```

خروجی برای یوزر ما:

```
{ip: "193.212.60.170", network: "193.212.60.0/23", version: "IPv4", city: "Oslo", region: "Oslo County", region_code: "03", country: "NO", country_name: "Norway", country_code: "NO", country_code_iso3: "NOR", country_capital: "Oslo", country_tld: ".no", continent_code: "EU", in_eu: false, postal: "0026", latitude: 59.955, longitude: 10.859, timezone: "Europe/Oslo", utc_offset: "+0200", country_calling_code: "+47", currency: "NOK", currency_name: "Krone", languages: "no,nb,nn,se,fi", country_area: 324220, country_population: 5314336, asn: "AS2119", org: "Telenor Norge AS"}
```

حال این داده ها را به یک پیغام جدید در آرایه `messages` اضافه می کنیم، جایی که نام تابعی را که فراخوانی کردیم را نیز مشخص می کنیم


```js
messages.push({
  role: "function",
  name: functionName,
  content: `The result of the last function was this: ${JSON.stringify(
    functionResponse
  )}
  `,
});
```

توجه داشته باشید که `role` به `function` تغییر کرده است. این به مدل می گوید که پارامتر `content` نتیجه فراخوانی تابع را دارد و نه ورودی کاربر را.

 حال، ما باید یک درخواست جدید به مدل با این آرایه `messages` به روز شده ارسال کنیم تا مدل با استفاده داده های جدید مجددا سعی کند که به سوال کاربر پاسخ دهد. برای این کار بهتر است یک حلقه ایجاد کنیم تا این رفت و برگشت بین agent و مدل را برای چند دفعه انجام دهد. انتظار ما این است که مدل نهایتا بتواند سوال کاربر را به درستی جواب دهد.


گد پایین شامل یک حلقه است که به برنامه اجازه می‌ده تا کل روند را تا پنج بار اجرا کند. اگر ما `finish_reason: "tool_calls"` را از مدل دریافت کنیم، فقط نتیجه فراخوانی تابع را به آرایه `messages` اضافه می کنیم و به تکرار بعدی حلقه می رویم. در صورتی که `finish_reason: "stop"` را دریافت کنیم، در این صورت معنی اش این است که مدل پاسخ مناسب را پیدا کرده است، بنابراین می توانیماز حلقه خارج شویم
.
```js
for (let i = 0; i < 5; i++) {
  const response = await openai.chat.completions.create({
    model: "gpt-4",
    messages: messages,
    tools: tools,
  });
  const { finish_reason, message } = response.choices[0];
 
  if (finish_reason === "tool_calls" && message.tool_calls) {
    const functionName = message.tool_calls[0].function.name;
    const functionToCall = availableTools[functionName];
    const functionArgs = JSON.parse(message.tool_calls[0].function.arguments);
    const functionArgsArr = Object.values(functionArgs);
    const functionResponse = await functionToCall.apply(null, functionArgsArr);
 
    messages.push({
      role: "function",
      name: functionName,
      content: `
          The result of the last function was this: ${JSON.stringify(
            functionResponse
          )}
          `,
    });
  } else if (finish_reason === "stop") {
    messages.push(message);
    return message.content;
  }
}
return "The maximum number of iterations has been met without a suitable answer. Please try again with a more specific input.";
```

### اجرای برنامه‌ی نهایی

حالا آماده امتحان کردن برنامه‌ی نهایی هستیم. من از agent خواهم خواست که بر اساس مکان من و آب و هوای فعلی، چند فعالیت مناسب را پیشنهاد کند.

```js
const response = await agent(
  "لطفا بر اساس موقعیت جغرافیایی من و آب و هوای فعلی جند فعالیت مناسب را پیشنهاد بده."
);
console.log(response);
```

و این جوابی هست که از برنامه می‌گیرم.

<div style="direction: ltr">

```
Based on your current location in Oslo, Norway and the weather (15°C and snowy),
here are some activity suggestions:
 
1. A visit to the Oslo Winter Park for skiing or snowboarding.
2. Enjoy a cosy day at a local café or restaurant.
3. Visit one of Oslo's many museums. The Fram Museum or Viking Ship Museum offer interesting insights into Norway’s seafaring history.
4. Take a stroll in the snowy streets and enjoy the beautiful winter landscape.
5. Enjoy a nice book by the fireplace in a local library.
6. Take a fjord sightseeing cruise to enjoy the snowy landscapes.
 
Always remember to bundle up and stay warm. Enjoy your day!
```

</div>

اگر نگاهی به لیست پیام‌های رد و بدل شده بین agent  و مدل بیاندازیم -  `response.choices[0].message` - می بینیم که مدل به برنامه دستور اجرای هر دو تابع را داده است. ابتدا، دستور اجرای تابع `getLocation` و سپس   تابع `getCurrentWeather` را با "longitude": "10.859", "latitude": "59.955" به عنوان آرگومان ها.

این داده ای است که از اولین فراخوانی تابعی که انجام دادیم، برگردانده شده است.

<div style="direction: ltr">

```
{"role":"assistant","content":null,"tool_calls":[{"id":"call_Cn1KH8mtHQ2AMbyNwNJTweEP","type":"function","function":{"name":"getLocation","arguments":"{}"}}]}
{"role":"assistant","content":null,"tool_calls":[{"id":"call_uc1oozJfGTvYEfIzzcsfXfOl","type":"function","function":{"name":"getCurrentWeather","arguments":"{\n\"latitude\": \"10.859\",\n\"longitude\": \"59.955\"\n}"}}]}
```

</div>


تبریک! شما توانستید یک عامل AI را که قادر به انجام کارهایی که کاربر از و می‌خواهد٬ است را ساختید.

اگر به دنبال چالش بیشتری هستید، می‌توانید دامنه کارهایی را که agent شما از پس انجام آنها بر میاد را با تعریف توابع جدید بهبود دهید.
