## تنظیمات سیستم 
### 🐧 فصل دوم کتاب How Linux Works

فصل‌ های قبل بیشتر روی زیرساخت داخلی Linux تمرکز داشتند. از Kernel و device ها و filesystem ها گرفته تا boot و شروع user space. در این فصل وارد بخش دیگری از سیستم می‌ شویم قسمت‌ هایی که باعث می‌ شوند برنامه‌ های user space بتوانند با configuration های سیستم ، کاربران ، زمان سیستم ، logging و کارهای زمان‌ بندی‌ شده تعامل داشته باشند.

---
📚 Table of Contents

- [system logging](#system-logging)
- [Checking Your Log Setup](#checking-your-log-setup)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)

--- 

### system logging

یکی از مهم‌ ترین بخش‌های یک سیستم لینوکس Logging است. تقریباً هر سیستم پیچیده‌ ای زمانی به مشکل می‌ خورد  ممکن است یک سرویس اجرا نشود،  یک device درست شناسایی نشود ، filesystem مشکل پیدا کند ، یک daemon crash کند یا configuration اشتباهی باعث رفتار غیرمنتظره شود. در چنین شرایطی ، یکی از اولین جاهایی که باید بررسی شود، log سیستم است. بسیاری از برنامه‌ های سیستمی پیام‌ های تشخیصی خود را به یک سیستم logging مانند syslog ارسال می‌ کنند. در مدل قدیمی daemon به نام ```syslogd``` منتظر پیام‌های log می‌ ماند و بر اساس نوع پیام تصمیم می‌ گیرد آن را کجا بفرستد. مقصد می‌ تواند شامل فایل ، terminal یا console ، کاربر مشخص ، یک سیستم دیگر ، database یا حتی هیچ مقصدی ، در صورتی که پیام طبق configuration نادیده گرفته شود. در سیستم‌ های امروزی که systemd استفاده می‌ کنند، بخش مهمی از این functionality توسط ```systemd-journald``` ارائه می‌ شود.

این به معنی آن نیست که rsyslog یا syslog-ng دیگر وجود ندارند. در بسیاری از سیستم‌ها journald در کنار یک logger قدیمی اجرا می‌ شود و می‌ تواند پیام‌ ها را به آن logger منتقل کند. یکی از تفاوت‌ های مهم journald با logfile های سنتی این است که journal فقط مجموعه‌ ای از خطوط متن ساده نیست بلکه اطلاعات ساختاری بیشتری را همراه پیام‌ ها نگه می‌ دارد. یک پیام log می‌ تواند اطلاعاتی داشته باشد مانند :

```
timestamp
process name
PID
UID
systemd unit
hostname
message
```

در سیستم syslog قدیمی دو مفهوم مهم دیگر نیز وجود دارند ```facility``` و ```priority / severity```. facility مشخص می‌ کند پیام به کدام دسته از سرویس‌ ها یا subsystem ها مربوط است و severity یا priority اهمیت پیام را مشخص می‌ کند. در syslog معمولاً priority از 0 تا  7 تعریف می‌ شود :

```
0 = emerg
1 = alert
2 = crit
3 = err
4 = warning
5 = notice
6 = info
7 = debug
```

بنابراین یک logger می‌ تواند مثلاً بگوید پیام‌ های مربوط به authentication را در یک فایل جدا ذخیره کن یا پیام‌ های severity بالا را به console بفرست. این separation یکی از دلایل انعطاف‌ پذیری سیستم logging قدیمی است.

---

### Checking Your Log Setup

فایل ```etc/passwd/``` یک فایل متنی است که اطلاعات اصلی حساب‌ های کاربری را نگه می‌ دارد. هر خط با : به هفت field تقسیم می‌ شود :

```username:password:UID:GID:GECOS:home:shell```

مثال: ```juser:x:3119:1000:J. Random User:/home/juser:/bin/bash``` هفت field عبارت‌ اند از :

- Login name
- Password field
- User ID
- Group ID
- Real name / GECOS
- Home directory
- Login shell

Login name

نامی که کاربر هنگام login استفاده می‌کند.

مثلاً:

juser
Password field

در سیستم‌های shadow password، معمولاً این field مقدار:

x

دارد.

این x به معنی password واقعی نیست؛ یعنی اطلاعات password در:

/etc/shadow

نگهداری می‌شود.

نکته مهم دیگر:

Password در Linux معمولاً به‌صورت plaintext ذخیره نمی‌شود و آنچه در shadow وجود دارد نیز خود password نیست، بلکه یک مشتق رمزنگاری‌شده از آن است.

UID

UID شناسه عددی user برای user space و Kernel است.

مثلاً:

1000

ممکن است UID یک کاربر عادی باشد.

UID 0 برای superuser است.

GID

GID گروه اصلی کاربر است.

این عدد باید با گروهی در database گروه‌ها مرتبط باشد.

GECOS

این field معمولاً اطلاعاتی مانند نام واقعی کاربر را نگه می‌دارد.

ممکن است شامل اطلاعات اضافی مانند:

نام
شماره اتاق
تلفن

نیز باشد.

Home directory

مثلاً:

/home/juser

directory اصلی کاربر است.

Shell

مثلاً:

/bin/bash

برنامه‌ای است که هنگام شروع session متنی کاربر اجرا می‌شود.

Shell باید در سیستم قابل اجرا باشد؛ برای بعضی عملیات مانند chsh نیز باید shell موردنظر در:

/etc/shells

قرار داشته باشد.

ساختار فایل passwd

فایل /etc/passwd syntax نسبتاً سخت‌گیرانه‌ای دارد.

نباید comment یا blank line را مانند یک configuration file معمولی به آن اضافه کرد.

یک entry می‌تواند وجود داشته باشد حتی اگر home directory مربوط به آن هنوز ساخته نشده باشد؛ مفهوم account در user space الزاماً به معنی وجود فیزیکی home directory نیست.

همچنین /etc/passwd تنها روش ممکن برای user database نیست؛ سرویس‌های شبکه‌ای مانند LDAP نیز می‌توانند user information را در اختیار سیستم قرار دهند.

Figure 7-1

