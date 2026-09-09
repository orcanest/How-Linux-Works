## تنظیمات سیستم 
### 🐧 فصل هفتم کتاب How Linux Works

فصل‌ های قبل بیشتر روی زیرساخت داخلی Linux تمرکز داشتند. از Kernel و device ها و filesystem ها گرفته تا boot و شروع user space. در این فصل وارد بخش دیگری از سیستم می‌ شویم قسمت‌ هایی که باعث می‌ شوند برنامه‌ های user space بتوانند با configuration های سیستم ، کاربران ، زمان سیستم ، logging و کارهای زمان‌ بندی‌ شده تعامل داشته باشند.

---
📚 Table of Contents

- [system logging](#system-logging)
- [Checking Your Log Setup](#checking-your-log-setup)
- [Searching and Monitoring Logs](#searching-and-monitoring-logs)
- [Logfile Rotation](#logfile-rotation)
- [Journal Maintenance](#journal-maintenance)
- [A Closer Look at System Logging](#a-closer-look-at-system-logging)
- [The Structure of etc](#the-structure-of-etc)
- [User Management Files](#user-management-files)
- [The etc passwd File](#the-etc-passwd-file)
- [Special Users](#special-users)
- [The /etc/shadow File](#the-etc-shadow-file)
- [Manipulating Users and Passwords](#manipulating-users-and-passwords)
- [Working with Groups](#working-with-groups)
- [getty and login](#getty-and-login)
- [Setting the Time](#setting-the-time)
- [Kernel Time Representation and Time Zones](#kernel-time-representation-and-time-zones)
- [Network Time](#network-time)
- [Scheduling Recurring Tasks with cron and Timer Units](#scheduling-recurring-tasks-with-cron-and-timer-units)
- [Installing Crontab Files](#installing-crontab-files)
- [System Crontab Files](#system-crontab-files)
- [Timer Units](#timer-units)
- [cron vs Timer Units](#cron-vs-timer-units)
- [Scheduling one time tasks with at](#scheduling-one-time-tasks-with-at)
- [Timer Unit Equivalents](#timer-unit-equivalents)
- [Timer Units Running as Regular Users](#timer-units-running-as-regular-users)
- [User Access Topics](#user-access-topics)
- [User IDs and User Switching](#user-ids-and-user-switching)
- [Process Ownership and Effective UID and Real UID and Saved UID](#process-ownership-and-effective-uid-and-real-uid-and-saved-uid)
- [User Identification and Authentication and Authorization](#user-identification-and-authentication-and-authorization)
- [Using Libraries for User Information](#using-libraries-for-user-information)
- [Pluggable Authentication Modules](#pluggable-authentication-modules)
- [PAM Configuration](#pam-configuration)
- [Tips on PAM Configuration Syntax](#tips-on-pam-configuration-syntax)
- [PAM and Passwords](#pam-and-passwords)
- [Tips](#tips)

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

اولین کار هنگام بررسی logging این است که ببینیم سیستم از چه logger هایی استفاده می‌ کند. در سیستم‌ های دارای    systemd ، ابزار اصلی ```journalctl```است. اجرای ```journalctl``` معمولاً پیام‌ های journal را نمایش می‌ دهد. برای بررسی خود daemon مربوط به journal می‌ توان وضعیت ```systemd-journald``` را بررسی کرد. در کنار journald ممکن است یکی از logger های قدیمی نیز نصب شده باشد. 

یکی از رایج‌ ترین آن‌ها ```rsyslogd```است. برای بررسی rsyslog معمولاً configuration اصلی را بررسی می‌ کنیم ```etc/rsyslog.conf``` و همچنین directory هایی مانند ```/etc/rsyslog.d/``` که ممکن است configuration های اضافی در آن‌ ها قرار داشته باشند. logger دیگری که ممکن است با آن مواجه شوید ```syslog-ng``` است که معمولاً configuration خود را در ```/etc/syslog-ng/``` نگه می‌ دارد.

#### 🔹 /var/log

بخش مهم دیگری از بررسی logging، directory در```/var/log``` است. بسیاری از logfile های سیستم در این directory قرار دارند. ممکن است فایل‌ هایی مانند ```/var/log/messages``` یا ```/var/log/syslog``` یا ```/var/log/auth.log``` یا ```/var/log/kern.log``` وجود داشته باشند. نام و وجود این فایل‌ ها به distribution و configuration سیستم بستگی دارد.

نکته مهم این است که **هر چیزی که داخل /var/log قرار دارد الزاماً توسط rsyslog نوشته نمی‌ شود**. ممکن است یک daemon مستقیماً logfile خودش را ایجاد کند. برای اینکه بفهمیم یک فایل مشخص توسط چه چیزی مدیریت می‌ شود ، باید configuration سرویس مربوطه را بررسی کنیم.

#### 🔹 journal storage location

اگر journald persistent storage داشته باشد ، معمولاً journal های دائمی را می‌توان در ```/var/log/journal/``` دید. اما اگر persistent journal فعال نباشد ، journal می‌ تواند فقط در فضای runtime مانند ```/run/log/journal/``` نگهداری شود. این تفاوت مهم است ، چون /run معمولاً روی tmpfs قرار دارد و محتوای آن بعد از reboot باقی نمی‌ ماند. بنابراین وجود /var/log/journal معمولاً نشانه مهمی برای persistent بودن journal است ، اما نبودن آن الزاماً به معنی نبودن journald نیست.

---

### Searching and Monitoring Logs

ابزار اصلی برای جست‌ وجو در journal هم ```journalctl``` است. اگر تعداد پیام‌ها زیاد باشد ، journalctl معمولاً از pager مانند less استفاده می‌ کند تا terminal با حجم زیادی از خروجی پر نشود. برای جست‌ وجوی مستقیم در field ها می‌ توان field را به command اضافه کرد مثلاً ```journalctl _PID=8792``` یعنی پیام‌ هایی را پیدا کن که PID آن‌ها 8792 است.

#### 🔹 Filtering by Time

یکی از مفید ترین روش‌ها ، محدود کردن log ها بر اساس زمان است. برای مشخص کردن زمان شروع ```journalctl -S``` یا ```journalctl --since ``` استفاده می‌ شود مثلاً ```journalctl -S -4h``` یعنی پیام‌ های چهار ساعت اخیر همچنین می‌ توان تاریخ و ساعت مشخص داد ```journalctl -S 06:00:00``` یا ```journalctl -S 2020-01-14``` . برای تعیین پایان محدوده ```journalctl -U ``` یا ```journalctl --until ``` استفاده می‌ شود.  می‌ توان هر دو را با هم استفاده کرد :

```journalctl -S '2026-09-07 08:00:00' -U '2026-09-07 09:00:00'```

#### 🔹 Filtering by Unit

اگر بخواهیم log های مربوط به یک systemd service را ببینیم ```journalctl -u ssh.service``` استفاده می‌ کنیم. معمولاً می‌ توان ```service.``` را هم حذف کرد ```journalctl -u ssh``` اگر نام unit را نمی‌ دانیم ، می‌ توان مقدارهای موجود برای field مربوط به systemd unit را دید ```journalctl -F _SYSTEMD_UNIT```  گزینه F- مقدارهای موجود برای یک field را نمایش می‌ دهد.

#### 🔹 Finding Fields

گاهی مسئله این نیست که بدانیم چه مقداری را جست‌ وجو کنیم بلکه مسئله این است که بدانیم چه field هایی اصلاً در journal وجود دارند. برای دیدن field های موجود ```journalctl -N``` استفاده می‌ شود. Field هایی که با underscore شروع می‌ شوند مانند  : 

```
_PID
_SYSTEMD_UNIT
```

معمولاً trusted fields هستند و client ارسال‌ کننده نمی‌ تواند مقدار آن‌ ها را به دلخواه تغییر دهد.

#### 🔹 Filtering by Text

می‌توان journal را بر اساس متن نیز جست‌ وجو کرد. برای جست‌ وجوی regex از ```journalctl -g 'kernel.*memory'``` استفاده می‌ شود. در این مثال ، پیام‌ هایی پیدا می‌ شوند که در آن‌ ها kernel آمده و در ادامه متن ، memory نیز دیده  می‌ شود. این روش بسیار شبیه استفاده از ```grep``` برای logfile های متنی است.

اما یک تفاوت مهم وجود دارد. اگر فقط پیام‌ های match شده را ببینیم ، ممکن است پیام‌ های مهمی که چند خط قبل یا بعد از آن اتفاق افتاده‌ اند را از دست بدهیم. بنابراین بعد از پیدا کردن یک timestamp مناسب ، معمولاً بهتر است یک بازه زمانی اطراف آن را نیز بررسی کنیم.

#### 🔹 Filtering by Boot

در journalctlm امکان جدا کردن log های مربوط به boot های مختلف وجود دارد. برای boot جاری از ```journalctl -b``` و برای boot قبلی ```journalctl -b -1``` و برای boot دو مرحله قبل ```journalctl -b -2``` استفاده می شود. این قابلیت برای بررسی crash یا boot failure بسیار مفید است. مثلاً اگر سیستم بعد از reboot دچار مشکل شده باشد ، می‌ توان log های boot قبلی را بررسی کرد.

#### 🔹 Filtering by Priority

می‌ توان پیام‌ ها را بر اساس priority محدود کرد مثلا ```journalctl -p 3``` یا محدوده‌ ای از priority ها را درخواست کرد.  priority های syslog از 0 تا 7  هستند. مثلاً priority های پایین‌ تر معمولاً پیام‌ های مهم‌ تری را نشان می‌ دهند.

#### 🔹 Monitoring Logs in Real Time

برای مشاهده پیا م‌های جدید به‌ صورت live از ```journalctl -f``` استفاده می‌ شود. این رفتار شبیه ```tail -f``` است ، با این تفاوت که journalctl مستقیماً با journal کار می‌ کند. این حالت برای troubleshooting سرویس‌ ها بسیار مفید است مثلاً ```journalctl -fu ssh.service``` می‌تواند log های سرویس SSH را به‌ صورت live نمایش دهد.

---

### Logfile Rotation

اگر logfile ها بدون محدودیت رشد کنند ، در نهایت می‌ توانند فضای disk را پر کنند. فرض کنید یک سرویس هر ثانیه چند پیام تولید کند. اگر logfile فقط append شود ، حجم آن می‌ تواند در مدت نسبتاً کوتاهی بسیار زیاد شود. برای حل این مسئله در سیستم‌ های دارای logfile ابزار معروفی به نام ```logrotate``` وجود دارد. ایده کلی log rotation این است که logfile فعلی قدیمی شود و logfile جدید جای آن قرار بگیرد مثلاً :

```
auth.log
auth.log.1
auth.log.2
auth.log.3
```

ممکن است نشان‌ دهنده نسل‌ های مختلف یک logfile باشند معمولاً ```auth.log``` فایل فعلی است. بعد از rotation هم ```auth.log``` قدیمی‌ تر می‌شود و ممکن است به ```auth.log.1```تغییر نام دهد. سپس نسل‌ های قدیمی‌ تر به ```auth.log.2``` و  ```auth.log.3``` و غیره تبدیل می‌ شوند. در نهایت قدیمی‌ ترین logfile ها حذف می‌ شوند. تعداد نسل‌ ها و زمان rotation به configuration بستگی دارد.

#### 🔹 Compressing old logs

قابلیت logrotate می‌تواند logfile های قدیمی را فشرده کند. در نتیجه ممکن است فایل‌ هایی مانند ```auth.log.2.gz``` را ببینید. این کار اجازه می‌ دهد اطلاعات تاریخی مدت بیشتری نگهداری شوند ، بدون اینکه به اندازه فایل‌ های متنی فضای disk مصرف کنند.

#### 🔹 open logfile

صرفاً Rotation برای rename کردن فایل نیست. ممکن است یک process هنوز file descriptor مربوط به logfile قدیمی را باز نگه داشته باشد. در این شرایط ، rename کردن فایل باعث نمی‌ شود process به‌صورت خودکار file descriptor خود را به فایل جدید منتقل کند. برای همین logrotate بسته به daemon ممکن است نیاز به HUP و reload و  restart یا روش مخصوص daemon داشته باشد تا process logfile جدید را باز کند. این یکی از دلایلی است که configuration مربوط به rotation را باید در کنار رفتار خود daemon بررسی کرد.

---

### Journal Maintenance

در journald برخلاف logfile های متنی ، journal را به‌ صورت database-like و ساختاریافته نگهداری می‌ کند. بنابراین روش نگهداری آن دقیقاً مانند ```auth.log``` و ```auth.log.1``` و ```auth.log.2``` نیست. journald می‌ تواند بر اساس حجم journal و فضای آزاد filesystem و زمان نگهداری و configuration مربوط به journal پیام‌ های قدیمی را حذف کند.

- برای مدیریت journal می‌توان از ```journalctl --disk-usage``` برای مشاهده میزان فضای مصرف‌ شده استفاده کرد.
- برای پاک‌ سازی بر اساس زمان نیز از ```journalctl --vacuum-time=30d``` می‌ تواند مورد استفاده قرار گیرد.
- همچنین می‌ توان بر اساس حجم از ```journalctl --vacuum-size=500M```  استفاده و محدود کرد.

در نتیجه journald یک سیستم مستقل برای مدیریت اندازه journal دارد و لزوماً نیازی نیست آن را با logrotate به همان شکلی که logfile های سنتی را مدیریت می‌ کنیم ، اداره کنیم.

---

### A Closer Look at System Logging

سیستم syslog سابقه طولانی در Unix دارد. ایده اصلی آن این بود که برنامه‌ ها لازم نباشد خودشان تصمیم بگیرند پیام‌ های diagnostic را دقیقاً در کجا ذخیره کنند. برنامه پیام را به logger می‌دهد و logger بر اساس configuration تصمیم می‌گیرد چه اتفاقی برای پیام بیفتد. این separation بسیار مفید است. مثلاً یک daemon می‌تواند فقط بگوید ```این یک پیام error است``` و logger تصمیم بگیرد ```این پیام باید در /var/log/example.log نوشته شود ``` یا ```این پیام باید به یک server مرکزی ارسال شود ``` یا ```این پیام باید هم در فایل و هم در remote server ذخیره شود```.

#### 🔹 Syslog and Network

یکی از قابلیت‌ های مهم سیستم logging این است که می‌تواند از network نیز استفاده کند.  در نتیجه یک ماشین می‌ تواند log های ماشین‌ های دیگر را دریافت کند. این موضوع امکان ساختن Central Logging Server را فراهم می‌کند مثلاً :

```
Server A ─┐
Server B ─┼──> Central Logger
Server C ─┘
```

در این مدل administrator می‌ تواند log های چندین ماشین را در یک مکان جمع‌ آوری کند. این روش برای  troubleshooting و auditing و monitoring و incident investigation بسیار مفید است.

#### 🔹 journald vs syslog

توانایی journald بیشتر روی جمع‌ آوری و سازمان‌ دهی log های همان سیستم تمرکز دارد.  یکی از مزایای آن این است که پیام فقط یک رشته متن نیست و metadata بیشتری همراه آن ذخیره می‌ شود. مثلاً می‌توان به‌ جای ```grep ssh /var/log/``` از field هایی مانند ```journalctl -u ssh.service``` استفاده کرد. journald همچنین می‌تواند log ها را به سیستم logging دیگری منتقل کند ، بنابراین در یک سیستم واقعی ممکن است این معماری را داشته باشیم :

```
Applications
     ↓
journald
     ↓
rsyslog / other logger
     ↓
Local files / Remote server
```

---

### The Structure of etc

دایرکتوری ```etc/``` یکی از مهم‌ ترین دایرکتوری های Linux است. بخش بزرگی از configuration مربوط به یک سیستم در این دایرکتوری قرار دارد. در سیستم‌ های قدیمی‌ تر، تعداد زیادی برنامه configuration خود را مستقیماً به‌ صورت فایل‌ های جداگانه در ```etc/``` قرار می‌ دادند. با افزایش تعداد package ها ، این روش باعث می‌ شد ```etc/```  شلوغ شود. مشکل دیگری نیز وجود داشت ، فرض کنید package یک فایل configuration مانند ```etc/example.conf/``` را نصب کند. administrator آن را برای نیازهای سیستم تغییر دهد بعد package update شود. اگر package فایل را دوباره جایگزین کند ، ممکن است customization administrator از بین برود. به همین دلیل در سیستم‌ های جدید تر استفاده از subdirectory ها و configuration های جداگانه رایج‌ تر شده است مثلاً ```/etc/systemd/``` یا ```/etc/rsyslog.d/``` یا ```/etc/pam.d/``` در این مدل، configuration های local می‌ توانند در محل مناسب خود قرار بگیرند و فایل‌ های پیش‌فرض package نیز کمتر در معرض تغییر مستقیم باشند.

#### 🔹 What should be in /etc ? 

یک guideline خوب این است که configuration های قابل شخصی‌ سازی مربوط به یک ماشین در ```etc/```   قرار بگیرند مثلاً ```etc/passwd/``` اطلاعات کاربران local را نگه می‌ دارد یا ```/etc/network/``` در سیستم‌ هایی که از آن استفاده می‌ کنند ، configuration شبکه را در خود دارد. در مقابل ، configuration های default که قرار نیست administrator مستقیماً تغییر دهد ، ممکن است در محل‌ هایی مانند ```/usr/lib/``` قرار بگیرند. مثلاً unit های systemd ارائه‌ شده توسط package ها معمولاً در مسیرهایی مانند ```/usr/lib/systemd/system/``` قرار دارند ، در حالی که override ها و unit های local در ```/etc/systemd/system/``` قرار می‌ گیرند.

---

### User Management Files

در Linux و Unix از چندین کاربر مستقل پشتیبانی می‌ کنند. اما Kernel مفهوم ```username``` را نمی‌ شناسد. Kernel بیشتر با ```UID``` کار می‌ کند مثلاً ```UID 0``` یا ```UID 1000``` یا ```UID 1001``` برای Kernel شناسه‌ های عددی هستند. نام‌ هایی مانند ```root``` و ```alice``` و ```bob``` در user space معنا پیدا می‌ کنند.

بنابراین وقتی یک برنامه بخواهد بداند UID 1000 متعلق به چه کاربری است ، باید یک mapping از UID به username پیدا کند. به‌ طور سنتی این اطلاعات در```/etc/passwd``` قرار دارند. اما در سیستم‌ های مدرن الزاماً همه کاربران از ```/etc/passwd```  نمی‌آیند. می‌ توان از سرویس‌ های شبکه‌ ای مانند LDAP یا سیستم‌ های دیگر نیز برای user database استفاده کرد.

---

### The etc passwd File

فایل ```etc/passwd/``` یک فایل متنی است که اطلاعات اصلی حساب‌ های کاربری را نگه می‌ دارد. هر خط با : به هفت field تقسیم می‌ شود : 

```username:password:UID:GID:GECOS:home:shell```

مثال ```juser:x:3119:1000:J. Random User:/home/juser:/bin/bash``` هفت field عبارت‌ اند از:

```
Login name
Password field
User ID
Group ID
Real name / GECOS
Home directory
Login shell
```

#### 🔹 Login name

نامی که user هنگام login استفاده می‌ کند مثلاً ```juser```.

#### 🔹 Password field

در سیستم‌ های shadow password ، معمولاً این field مقدار ```x``` دارد. این x به معنی password واقعی نیست بلکه اطلاعات password در ```etc/shadow/``` نگهداری می‌ شود. نکته مهم دیگر Password در Linux معمولاً به‌ صورت plaintext ذخیره نمی‌ شود و آنچه در shadow وجود دارد نیز خود password نیست، بلکه یک مشتق رمزنگاری‌ شده از آن است.

#### 🔹 UID

شناسه عددی user برای user space و Kernel است مثلاً ```1000``` ممکن است UID یک کاربر عادی  باشد. ```UID 0``` برای superuser است.

#### 🔹 GID

گروه اصلی کاربر است این عدد باید با گروهی در database گروه‌ها مرتبط باشد.

#### 🔹 GECOS

این field معمولاً اطلاعاتی مانند نام واقعی کاربر را نگه می‌ دارد. ممکن است شامل اطلاعات اضافی مانند نام ، شماره اتاق ، تلفن نیز باشد.

#### 🔹 Home directory

مثلاً ```/home/juser```  دایرکتوری اصلی user است.

#### 🔹 Shell

مثلاً ```/bin/bash``` برنامه‌ ای است که هنگام شروع session متنی کاربر اجرا می‌ شود. Shell باید در سیستم قابل اجرا باشد. برای بعضی عملیات مانند chsh نیز باید shell موردنظر در ```/etc/shells``` قرار داشته باشد.


#### 🔹 Structure of the passwd file

فایل ``` /etc/passwd syntax``` نسبتاً سخت‌ گیرانه‌ ای دارد. نباید comment یا blank line را مانند یک configuration file معمولی به آن اضافه کرد. یک entry می‌ تواند وجود داشته باشد حتی اگر home directory مربوط به آن هنوز ساخته نشده باشد. مفهوم account در user space الزاماً به معنی وجود فیزیکی home directory نیست. همچنین /etc/passwd تنها روش ممکن برای user database نیست. سرویس‌ های شبکه‌ ای مانند LDAP نیز می‌ توانند user information را در اختیار سیستم قرار دهند.

<img width="100%" height="182" alt="Screenshot from 2026-09-09 13-46-25" src="https://github.com/user-attachments/assets/fc1233de-143a-4c4a-a8a7-f800c02b5376" />

<img width="100%" height="226" alt="image" src="https://github.com/user-attachments/assets/633d29f3-5a01-46a6-b4a4-b8b30bac05d2" />


---

### Special Users

در /etc/passwd علاوه بر کاربران عادی ، user هایی وجود دارند که برای اهداف سیستمی ایجاد شده‌ اند. مهم‌ترین آن‌ها ```root``` است. root همیشه ```UID = 0``` دارد و primary GID آن ```GID = 0``` است.

#### 🔹 System Users

ممکن است user هایی مانند ```daemon``` یا ```nobody``` را ببینید. این user ها معمولاً برای اجرای سرویس‌ها یا محدود کردن privilege یک process استفاده می‌ شوند. مثلاً یک daemon به‌ جای اجرای مستقیم با root ، می‌ تواند با یک user کم‌ امتیاز اجرا شود. اگر daemon مورد حمله قرار بگیرد، attacker در حالت ایده‌ آل فقط privilege همان user را خواهد داشت. به چنین حساب‌ هایی در ادبیات Unix گاهی ```pseudo-user``` گفته می‌ شود. اما نکته بسیار مهم این است که این مفهوم برای Kernel یک user ویژه به حساب نمی‌آید. Kernel در سطح UID با آن‌ ها کار می‌ کند و UID 0 است که برای Kernel معنای ویژه دارد بنابراین nobody ذاتاً و به‌ صورت غیرقابل تغییر «کاربری که نمی‌تواند هیچ کاری انجام دهد» نیست و permission های آن به UID و group membership و capability ها و سایر مکانیزم‌ های امنیتی بستگی دارد.

---

### The etc shadow file

فایل ```etc/shadow/``` محل اصلی اطلاعات حساس مربوط به password در سیستم‌ های shadow password است. این اطلاعات می‌ تواند شامل password hash و زمان تغییر password و حداقل عمر password و حداکثر عمر password و زمان warning و زمان disable شدن account و اطلاعات مربوط به expiration باشد. در حالت عادی کاربران عادی نباید بتوانند این فایل را بخوانند.  دلیل اصلی این طراحی این است که اگر hash های password در اختیار تمام کاربران قرار بگیرد، attacker می‌ تواند به‌صورت offline روی آن‌ ها password guessing انجام دهد.

#### 🔹 Password Hash

اصطلاح دقیق‌ تر برای چیزی که در ```etc/shadow/``` ذخیره می‌ شود ```password hash / password verifier``` است، نه encrypted password. در حالت معمول password plaintext ذخیره نمی‌ شود. اگر field مربوط به password شامل ```*``` باشد ، معمولاً login با password برای آن account غیرفعال است. مقدار```!``` نیز در بسیاری از سیستم‌ ها برای lock کردن credential استفاده می‌ شود. رفتار دقیق بعضی special value ها به ابزارها و implementation سیستم بستگی دارد.

---

### Manipulating Users and Passwords

روش مناسب مدیریت user ها استفاده از command های مخصوص است ، نه ویرایش مستقیم /etc/passwd. برای تغییر password از ```passwd``` یا برای administrator از ```passwd username``` و برای تغییر نام یا اطلاعات GECOS از ```chfn username``` و برای تغییر shell از ```chsh username``` استفاده می‌ شود. shell انتخاب‌ شده باید در ```etc/shells/``` مجاز باشد. برای ساخت و حذف کاربران نیز بسته به distribution ابزارهایی مانند ```useradd``` یا ```userdel``` یا wrapper هایی مانند ```adduser``` وجود دارند.


#### 🔹 Why shouldn't you edit /etc/passwd directly ?

از نظر فنی /etc/passwd یک فایل متنی معمولی است و root می‌ تواند آن را با editor تغییر دهد. اما این کار خطرناک است. یک اشتباه کوچک در syntax می‌تواند باعث شود که user database خراب شود و login کار نکند و یک user اطلاعات نادرست داشته باشد و ابزارهای مدیریت user دچار مشکل شوند برای همین اگر واقعاً لازم باشد فایل را مستقیم ویرایش کنیم، ابزار ```vipw``` برای /etc/passwd طراحی شده است. vipw فایل را به‌ صورت کنترل‌ شده edit می‌ کند و locking و backup مناسب را در نظر می‌گیرد. برای /etc/shadow از دستور ```vipw -s``` استفاده می‌ شود.

--- 

### Working with Groups

گروه برای اشتراک‌ گذاری permission بین مجموعه‌ ای از user ها استفاده می‌ شوند. مثلاً می‌ توان گفت ```owner  = alice``` و ```group  = developers``` و سپس اعضای group بتوانند فایل را بخوانند یا تغییر دهند. اطلاعات group های local در ```etc/group/``` قرار دارد. هر خط چهار field دارد : ```group_name:password:GID:members``` مثلاً ```disk:*:6:juser,beazley```.

#### 🔹 Group name
نام گروه ```disk``` این نام هنگام نمایش permission ها نیز دیده می‌ شود.


#### 🔹 Group password

در سیستم‌ های مدرن تقریباً استفاده نمی‌ شود. معمولاً مقدارهایی مانند ```*``` یا ```x``` دیده می‌ شود. اگر x باشد ، ممکن است اطلاعات مربوط به group password در ```etc/gshadow/``` قرار داشته باشد.

#### 🔹 GID

شناسه عددی گروه است ```6```.

#### 🔹 Additional members

لیست username هایی است که به‌ عنوان member اضافی گروه معرفی شده‌ اند : ```juser,beazley``` . کاربر همچنین می‌ تواند از طریق primary GID خود عضو گروه باشد. بنابراین عضویت group فقط به field آخر /etc/group محدود نیست. برای مشاهده گروه‌ های کاربر از ```groups``` استفاده می‌ شود. بسیاری از distribution ها هنگام ایجاد یک user، group با همان نام user نیز ایجاد می‌ کنند.

<img width="100%" height="205" alt="image" src="https://github.com/user-attachments/assets/650bb2dd-dbef-400f-95f6-05b61799eb6b" />

<img width="1207" height="186" alt="image" src="https://github.com/user-attachments/assets/adb52325-7684-41da-baee-4f8e601dbec3" />

---

### getty and login

در یک terminal متنی ، برنامه‌ ای لازم است که terminal را برای login آماده کند. این وظیفه معمولاً با ```getty``` یا implementation هایی مانند ```agetty``` انجام می‌ شود. getty به terminal متصل می‌ شود و چیزی مانند ```login``` نمایش می‌ دهد. برای مثال ممکن است در process list چیزی شبیه این ببینیم ```ps ao args | grep getty``` و```/sbin/agetty ... tty1``` دیده شود.

#### 🔹  Login flow

```
getty 
     ↓
login prompt 
     ↓
username 
     ↓
login 
     ↓
authentication 
     ↓
user shell
```

معمولا getty مسئول login authentication کامل نیست. پس از وارد شدن username ، برنامه ```login``` می‌ تواند ادامه فرآیند را بر عهده بگیرد. login password را می‌ گیرد و authentication را انجام می‌ دهد. در سیستم‌ های امروزی PAM معمولاً در این بخش وارد می‌ شود. اگر authentication موفق باشد، login باید محیط user را آماده کند و shell مناسب را اجرا کند. در نهایت process مربوط به login معمولاً با shell کاربر جایگزین می‌ شود. از نظر process، این کار می‌ تواند با ```()exec``` انجام شود. بنابراین مسیر conceptually شبیه این است :

```
getty process
      ↓
login program
      ↓
user shell
```

امروزه در بسیاری از سیستم‌ ها login مستقیم روی virtual terminal تنها یکی از روش‌ های ورود است و روش‌ هایی مانند display manager و SSH و remote login نیز وجود دارند.

---

### Setting the Time

لینوکس باید یک مفهوم قابل اعتماد از زمان داشته باشد. دو ساعت مهم در یک PC معمولی وجود دارند ```System Clock``` و ```Real-Time Clock (RTC)``` که System Clock توسط Kernel مدیریت می‌ شود و RTC یک clock سخت‌ افزاری است که معمولاً با باتری تغذیه می‌ شود و می‌ تواند زمان را در هنگام خاموش بودن سیستم نیز حفظ کند. هنگام boot ، Kernel یا user space زمان RTC را می‌ خواند و بر اساس آن system clock را تنظیم می‌ کند.

#### 🔹 System Clock

همان زمانی است که بیشتر برنامه‌ های Linux هنگام درخواست زمان از سیستم دریافت می‌ کنند. می‌ توان آن را با ابزارهایی مانند ```date``` مشاهده کرد. برای تنظیم ساعت نیز ابزارهای مختلفی وجود دارند ، هرچند در سیستم‌ های مدرن بهتر است synchronization سرویس مدیریت زمان را به عهده بگیرد.

#### 🔹 Hardware Clock

برای مشاهده RTC از ```hwclock``` استفاده می‌ شود. برای همگام کردن RTC با system clock از ```hwclock --systohc --utc``` استفاده می‌ شود و گزینه utc-- مشخص می‌ کند که RTC به‌ عنوان UTC تفسیر شود.

#### 🔹 Why UTC ?

نگه داشتن RTC روی UTC باعث می‌ شود تغییرات timezone و daylight saving time کمتر باعث مشکل شوند. سیستم عامل هنگام نمایش زمان محلی ، UTC را با timezone مناسب تبدیل می‌ کند. البته در dual-boot با سیستم‌ هایی که RTC را به‌صورت local time نگه می‌ دارند ، باید configuration دو سیستم عامل با یکدیگر هماهنگ باشد.

---

### Kernel Time Representation and Time Zones

برای نمایش زمان در user space ، لینوکس باید یک representation استاندارد داشته باشد. یکی از مهم‌ ترین مفاهیم Unix epoch است ```1970-01-01 00:00:00 UTC```. زمان Unix به‌ طور مفهومی تعداد واحد های زمانی سپری‌ شده از این نقطه را نشان می‌ دهد. برای بسیاری از کاربردهای Linux ، این واحد seconds است. اما نباید این را با تمام representation های داخلی Kernel یکی دانست. Kernel subsystem های مختلفی برای clock های مختلف دارد  مانند   wall-clock time و monotonic time و boot time که کاربرد آن‌ ها متفاوت است.

#### 🔹 Time Zone

کرنل به‌ تنهایی مسئول نمایش زمان محلی مانند America/New_York یا  Asia/Tehran یا Europe/Berlin نیست. اطلاعات timezone در user space و timezone database استفاده می‌ شود. فایل ```/etc/localtime/ معمولاً فایل timezone مربوط به سیستم است و در بسیاری از distribution ها symbolic link  به یکی از فایل‌های ```/usr/share/zoneinfo/``` است بنابراین یک timestamp واحد می‌ تواند بر اساس timezone سیستم به شکل‌ های مختلف نمایش داده شود.

---

### Network Time

فقط وقتی RTC زمان را نگه می‌ دارد که آخرین بار تنظیم شده است. در طول زمان ممکن است clock سخت‌ افزاری یا system clock دچار drift شود. برای همین سیستم‌ ها معمولاً از یک time synchronization protocol استفاده می‌ کنند. پروتکل رایج ```NTP``` است. در سیستم‌ های دارای systemd ، یکی از گزینه‌ ها ```systemd-timesyncd``` است. این سرویس می‌ تواند system clock را با remote time server هماهنگ کند. configuration آن در ```timesyncd.conf``` قابل بررسی است. بسته به distribution ممکن است سرویس‌ های دیگری نیز استفاده شوند. یکی از مهم‌ ترین گزینه‌ ها ```chronyd``` است. chronyd برای محیط‌ هایی که اتصال شبکه دائمی ندارند نیز کاربرد خوبی دارد ، چون می‌ تواند زمان را در هنگام قطع اتصال مدیریت کند. پس نباید فرض کنیم هر Linux system الزاماً فقط از timesyncd استفاده می‌ کند.

---

### Scheduling Recurring Tasks with cron and Timer Units

بسیاری از کارهای administration باید به‌ صورت دوره‌ ای اجرا شوند مثلاً backup و cleanup و rotation و maintenance و report generation و synchronization . برای این کار دو مدل مهم وجود دارد ```cron``` و ```systemd timer units``` که کتاب هر دو را بررسی می‌ کند.

#### 🔹 cron

یک scheduler سنتی Unix/Linux است. کار زمان‌ بندی‌ شده‌ای که توسط cron اجرا می‌ شود cron job نام دارد. کاربر معمولاً schedule را در ```crontab``` قرار می‌ دهد. یک entry ساده : 

```15 09 * * * /home/juser/bin/spmake```

یعنی command مورد نظر هر روز ساعت 09:15 اجرا شود.

#### 🔹 Five cron fields

فرمت اصلی ```minute hour day-of-month month day-of-week command``` و محدوده‌ ها :

```
minute       0-59
hour         0-23
day of month 1-31
month        1-12
day of week  0-7
```

در بسیاری از cron ها  ```0 = Sunday``` و ```7 = Sunday``` است. علامت ```*``` یعنی تمام مقدارهای ممکن مثلاً ```* * * * *``` یعنی هر دقیقه و ```` * * * * 0``` یعنی در دقیقه صفر هر ساعت.

<img width="100%" height="255" alt="image" src="https://github.com/user-attachments/assets/8ed8d776-f979-459c-9727-f7f194185a89" />

#### 🔹 Combining values

- می‌ توان چند مقدار را با comma مشخص کرد ``` * * * * 0,30``` یعنی دقیقه 0 و 30 هر ساعت.
- می‌ توان range نیز داشت ```* * * 17-9 0``` یعنی در ساعت‌های 9 تا 17.
- همچنین syntax مربوط به step نیز در cron های رایج وجود دارد ```* * * */15*``` یعنی هر 15 دقیقه.

#### 🔹 A note about Day of Month and Day of Week

در cron های قدیمی ، اگر هر دو field مربوط به ```day of month``` و ```day of week``` محدود شده باشند ، رفتار آن‌ ها در بسیاری از implementation های رایج به‌ شکل OR است ، نه اینکه الزاماً هر دو شرط هم‌ زمان برقرار باشند. این یکی از بخش‌هایی است که می‌ تواند باعث اشتباه در طراحی crontab شود. بنابراین برای schedule های پیچیده باید manual مربوط به cron implementation همان سیستم را بررسی کرد.

--- 

### Installing Crontab Files

هر user می‌ تواند crontab خودش را داشته باشد : 

- برای مشاهده ```crontab -l```
- برای ویرایش ```crontab -e```
- برای حذف ```crontab -r```

استفاده می‌ شود. همچنین می‌ توان یک فایل آماده را نصب کرد ```crontab file``` یعنی محتویات file به‌عنوان crontab کاربر نصب شود. 

> 💡 دستور ```crontab -e``` مزیت مهمی دارد. ابزار معمولاً syntax را بررسی می‌ کند و در صورت وجود خطا اجازه می‌ دهد آن را اصلاح کنید.

#### 🔹 Environment in cron

یک cron job لزوماً همان environment را ندارد که وقتی در shell تعاملی command را اجرا می‌ کنید. این موضوع درباره موارد PATH و HOME و SHELL و environment variables و  working directory اهمیت دارد بنابراین بهتر است در script های cron مسیر executable ها مشخص باشد و environment مورد نیاز explicit باشد همچنین فرض نکنیم shell تعاملی دقیقاً همان environment را دارد.

---

### System Crontab Files

برای job های سراسری سیستم ، distribution ها معمولاً فایل ```etc/crontab/``` را در اختیار دارند. تفاوت مهم ***/etc/crontab*** با crontab معمولی این است که یک field اضافی برای username دارد. فرمت:

```
minute hour day-of-month month day-of-week user command
-> 42 6 * * * root /usr/local/bin/cleansystem
```

یعنی command در ساعت 06:42 و با user:root اجرا شود.

#### 🔹 /etc/cron.d

بسیاری از distribution ها directory زیر را نیز دارند ```/etc/cron.d/``` فایل‌ های داخل آن معمولاً syntax مشابه ```/etc/crontab``` دارند و می‌ توانند username را مشخص کنند مثلاً ```/etc/cron.d/backup``` ممکن است schedule مربوط به backup را نگه دارد.

#### 🔹 /etc/cron.daily

ممکن است directory هایی مانند /etc/cron.daily و /etc/cron.hourly و /etc/cron.weekly و /etc/cron.monthly نیز وجود داشته باشند. این directory ها معمولاً مستقیماً به معنی یک scheduler مستقل نیستند. اغلب یک cron job دیگر script های داخل آن‌ها را در زمان مناسب اجرا می‌کند. به همین دلیل برای فهمیدن زمان واقعی اجرای یک script باید configuration مربوط به cron را تا انتها دنبال کرد.

---

### Timer Units

در systemd یک روش جایگزین برای cron ارائه می‌ شود به نام ```timer unit``` . برای ساخت یک timer جدید معمولاً دو unit داریم ```example.timer``` و ```example.service``` که timer مسئول زمان‌ بندی است و service مسئول کاری است که باید اجرا شود این separation یک طراحی مهم است. Timer خودش نمی‌ گوید دقیقاً چه command باید انجام شود  بلکه مشخص می‌ کند چه زمانی unit دیگری را activate کند.

#### 🔹 Timer example

```
[Unit]
Description=Example timer unit

[Timer]
OnCalendar=*-*-* *:00,20,40
Unit=loggertest.service

[Install]
WantedBy=timers.target
```
این timer در  00 و 20 و  40 دقیقه هر ساعت فعال می‌ شود یعنی :

```
12:00
12:20
12:40
13:00
13:20
13:40
...
```
در OnCalendar از syntax تقویمی systemd استفاده می‌ کند که بسیار انعطاف‌ پذیرتر از schedule های ساده cron است.

#### 🔹 Timer related service

معمولاً timer با نام service متناظر را فعال می‌ کند. مثلاً ```loggertest.timer``` می‌تواند ```loggertest.service``` را فعال کند. 

```[Unit]
Description=Example service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/loggertest
```

در این مدل قرار دارد :
```
timer
   ↓
activation
   ↓
service
   ↓
ExecStart
```

#### 🔹 Activate Timer

پس از ایجاد unit ها ```systemctl daemon-reload``` سپس ```systemctl enable --now loggertest.timer``` می‌ تواند timer را enable و هم‌ زمان start کند. برای مشاهده timer ها از ```systemctl list-timers``` استفاده می‌ شود. این command اطلاعاتی مانند زمان activation بعدی و زمان activation قبلی و زمان باقی‌ مانده و unit مربوطه را نشان می‌ دهد.

#### 🔹 Monotonic Timers

در systemd timer ها فقط OnCalendar ندارند. گزینه‌ هایی نیز وجود دارند  مانند  :

```
OnBootSec=
OnStartupSec=
OnUnitActiveSec=
OnUnitInactiveSec=
```

این‌ ها برای schedule هایی مناسب‌ اند که به یک event مانند boot یا activation unit نسبت داده می‌ شوند. مثلاً ```OnBootSec=10min``` یعنی 10 دقیقه بعد از boot یا ```OnUnitActiveSec=1h``` یعنی یک ساعت بعد از آخرین activation موفق unit.

---

###  cron vs Timer Units

هر دو روش مفید هستند : 

- مزایای cron : ساده و قدیمی و بسیار شناخته‌ شده است در بسیاری از سیستم‌ها وجود دارد و ابزارها و script های زیادی برای آن نوشته شده‌اند . برای schedule های ساده بسیار سریع قابل فهم است. به همین دلیل cron با وجود systemd همچنان به‌ طور گسترده استفاده می‌ شود.

- مزایای systemd timer: در Timer unit ها با ecosystem خود systemd integration بیشتری دارند. مثلاً process ها زیر cgroup مربوط به service قرار می‌ گیرند و log ها در journal قابل مشاهده‌ اند. dependency های systemd در اختیار job قرار دارند و ordering قابل تعریف است همچنین failure status قابل مشاهده هستند. timer state با systemctl قابل بررسی و schedule های calendar پیچیده‌ تری وجود دارد. مثلاً :

```
systemctl status backup.service
or
journalctl -u backup.service
```

می‌ توانند اطلاعات اجرای job را نشان دهند در مقابل cron job ها معمولاً به شکل ساده‌ تر و مستقل‌ تری اجرا می‌ شوند. بنابراین جایگزینی systemd timer با cron همیشه ضروری نیست. برای یک task ساده ، cron ممکن است انتخاب بسیار خوبی باشد. برای کاری که tightly integrated با systemd service هاست ، timer unit می‌ تواند مناسب‌ تر باشد.

---

### Scheduling one time tasks with at

قابلیت cron برای task های تکرار شونده است اما اگر بخواهیم یک command را فقط یک‌ بار در آینده اجرا کنیم ، از ابزار ```at``` استفاده می کنیم. مثلاً conceptually از ```at 23:00``` سپس command های مورد نظر را وارد می‌ کنیم. بعد از پایان input ، job برای اجرای آینده ثبت می‌ شود.

#### 🔹 View Jobs at

- برای مشاهده job های منتظر از ```atq``` استفاده می‌ شود.
- برای حذف job از ```atrm jobid``` استفاده می‌ شود.

```
at
 ↓
schedule one job
 ↓
atq
 ↓
view pending jobs
 ↓
atrm
 ↓
remove job
```

---

### Timer Unit Equivalents

درsystemd می‌ تواند بسیاری از کاربرد های at را نیز پوشش دهد. یکی از روش‌ های راحت استفاده از ```systemd-run``` است. مثلاً با :

```systemd-run --on-calendar="..."```

می‌توان یک transient unit ساخت که در زمان مشخص اجرا شود. مزیت این روش این است که لازم نیست برای یک job ساده حتماً فایل timer. و service. دائمی بسازیم. systemd این unit را به‌ صورت موقت ایجاد می‌ کند و آن را در چارچوب مدیریت خودش اجرا می‌ کند. این روش برای task های یک‌ باره‌ای که می‌ خواهیم تحت کنترل systemd باشند ،  بسیار مناسب است.

---

### Timer Units Running as Regular Users

قابلیت systemd timer فقط برای root نیست ، کاربر عادی نیز می‌ تواند user unit داشته باشد. برای تعامل با systemd user manager از ```systemctl --user``` استفاده می‌ شود. مثلاً timer های user را نمایش می‌ دهد :

```systemctl --user list-timers```

#### 🔹 systemd-run --user

برای ایجاد transient user unit می‌ توان ```systemd-run --user``` استفاده کرد. در این حالت job با identity کاربر اجرا می‌ شود ، نه با root. این موضوع برای automation شخصی کاربر بسیار مفید است.

#### 🔹 Logout problem

یک نکته مهم وجود دارد. user manager معمولاً زمانی فعال است که user session وجود داشته باشد بنابراین اگر کاربر logout کند ، بسته به configuration سیستم ممکن است user manager نیز متوقف شود و timer های آن دیگر اجرا نشوند. برای اینکه user manager حتی بعد از logout نیز فعال بماند، می‌ توان از :

```loginctl enable-linger username```

استفاده کرد. Linger باعث می‌ شود user manager برای آن user مستقل از login session نیز بتواند باقی بماند. در این حالت user timer ها می‌ توانند حتی زمانی که user login نیست ، اجرا شوند.

---

### User Access Topics

تا اینجا درباره فایل‌ هایی صحبت کردیم که user information را ذخیره می‌ کنند. اکنون باید یک سؤال مهم‌ تر را بررسی کنیم : **Linux چگونه تعیین می‌ کند یک process با چه هویتی اجرا شود و چه دسترسی‌ هایی داشته باشد؟** ، در سطح Kernel مبنای اصلی  UID و GID و permission های مرتبط با آن‌ ها هستند. اما user authentication و authorization یک فرآیند چند مرحله‌ ای است.

---

### User IDs and User Switching

یک process می‌ تواند در شرایط خاص identity خود را تغییر دهد. یکی از مکانیزم‌ های مهم ```()setuid``` است. همچنین برنامه‌های setuid می‌توانند هنگام اجرا identity مؤثر متفاوتی داشته باشند. مثلاً یک executable با مالک root و  permission مثلاً ```rwsr-xr-x-``` دارای set-user-ID bit است. اگر یک کاربر عادی آن را اجرا کند، process می‌ تواند با ```Effective UID = 0``` اجرا شود. این دقیقاً یکی از مهم‌ ترین مکانیزم‌ های privilege escalation کنترل‌ شده در Unix است.

#### 🔹 Three important principles

به‌ صورت ساده ، سه رفتار مهم وجود دارد : 

1. Setuid executable

یک process عادی می‌ تواند برنامه‌ ای را که setuid شده اجرا کند و process حاصل effective identity متفاوتی داشته باشد.

2. Root

یک process دارای privilege مناسب root می‌تواند UID خود را تغییر دهد. در مدل سنتی این privilege با UID صفر مرتبط است. در Linux مدرن capability ها نیز در این تصمیم نقش دارند.

3. Non-root process

یک process عادی نمی‌تواند آزادانه خود را به هر UID دلخواهی تبدیل کند. Kernel محدودیت‌ هایی روی ()setuid و تغییر credential ها اعمال می‌ کند.

---

### Process Ownership and Effective UID and Real UID and Saved UID

یک process فقط یک UID ساده ندارد مهم‌ ترین UID ها عبارت‌ اند از Real UID و Effective UID و Saved UID.

#### 🔹 Real UID

نشان می‌ دهد process از طرف چه user ایجاد شده است مثلاً اگر ```alice``` یک برنامه را اجرا کند ```RUID = alice ``` باقی می‌ ماند.

#### 🔹 Effective UID

برای تصمیم‌ های permission بسیار مهم است. مثلاً Kernel هنگام بررسی دسترسی یک process به فایل ، در بسیاری از موارد از effective credentials آن استفاده می‌ کند. بنابراین ممکن است ```RUID = alice``` و ```EUID = root ``` باشد. این حالت می‌ تواند در setuid program ها رخ دهد.

#### 🔹 Saved UID

به process اجازه می‌ دهد بعضی credential های قبلی را برای بازگشت یا تغییر کنترل‌ شده نگه دارد. این موضوع در طراحی برنامه‌ هایی که privilege خود را موقتاً تغییر می‌ دهند اهمیت دارد.

> 💡 sudo tip
> 
> یک تصور اشتباه رایج این است که اگر یک برنامه با sudo اجرا شود ، تمام UID های آن به root تبدیل می‌ شوند. این الزاماً درست نیست در یک اجرای ```sudo command``` معمولاً program موردنظر effective privilege لازم را دریافت می‌کند ، اما real identity مربوط به user فرا خواننده همچنان می‌ تواند متفاوت باشد. بنابراین باید بین ```who started the process ``` و ```what identity the kernel uses for permission checks``` تفاوت قائل شد.

#### 🔹 Permission and Process Ownership

تصور ساده‌ ای مثل RUID مالک process است و حتماً می‌ تواند آن را kill کند بیش از حد ساده‌ سازی شده است. قواعد signal permission به credential های process ها و نوع signal و privilege ها وابسته‌ اند.  بنابراین برای فهم دقیق رفتار()kill باید permission های signal و UID ها و capability ها را با هم در نظر گرفت.

---

### User Identification and Authentication and Authorization

امنیت چند کاربره را می‌توان به سه مرحله مفهومی تقسیم کرد :  **Identification و Authentication و Authorization** .


#### 🔹 Identification

سؤال : شما چه کسی هستید؟ مثلاً ```username = alice``` و کاربر identity خود را اعلام می‌ کند.

#### 🔹 Authentication

سؤال : آیا واقعاً همان کسی هستید که ادعا می‌ کنید؟. روش‌ های authentication می‌توانند شامل ، password و SSH key و  smart card و token و certificate و biometric و multi-factor authentication باشند.

#### 🔹 Authorization

بعد از اینکه identity مشخص و authenticate شد ، سؤال این است : چه کاری اجازه دارید انجام دهید؟ مثلاً alice ممکن است اجازه خواندن یک فایل را داشته باشد اما اجازه تغییر آن را نداشته باشد.

#### 🔹 What role does the kernel play ?

کرنل عمد تاً با credential های عددی مانند UID و GID کار می‌ کند. Kernel معمولاً نمی‌ داند alice چه کسی است. همچنین password authentication در سطح user space انجام می‌ شود مثلاً :

```
login
   ↓
PAM
   ↓
authentication modules
   ↓
credentials
   ↓
process
   ↓
Kernel permission checks
```

بنابراین authentication و authorization را نباید یکی دانست.

---

### Using Libraries for User Information

فرض کنید یک برنامه UID خودش را دارد و می‌ خواهد username مربوط به آن را پیدا کند. یک روش ساده این است که UID را بگیرد و /etc/passwd را باز کند و خط‌ ها را یکی‌یکی بخواند و field ها را جدا کند و UID هر خط را بررسی و وقتی UID موردنظر پیدا شد ، username همان خط را برگرداند. مثلاً : 

```
UID = 1000

/etc/passwd
    ↓
line 1 → UID 0
line 2 → UID 1
line 3 → UID 2
...
line N → UID 1000
             ↓
          username
```

اما اگر هر برنامه این کار را خودش انجام دهد، سیستم بسیار پیچیده و تکراری می‌ شود. به همین دلیل library های استاندارد این کار را انجام می‌ دهند مثلاً ```()getpwuid``` می‌ تواند اطلاعات مربوط به UID را برگرداند.

#### 🔹 Library advantage

این abstraction فقط برای راحتی نیست. فرض کنید برنامه‌ ها مستقیماً /etc/passwd را می‌ خواندند اگر بعداً بخواهیم user database را به LDAP منتقل کنیم ، باید تمام برنامه‌ ها را تغییر دهیم. اما وقتی برنامه از standard library و name service mechanism استفاده می‌ کند ، configuration سیستم می‌ تواند تعیین کند user information از کجا بیاید. در نتیجه application نیازی ندارد بداند user data دقیقاً در /etc/passwd است یا از network service می‌ آید. این یکی از دلایل اهمیت abstraction در user management است.

#### 🔹 Password problem

انجام Mapping username به UID نسبتاً ساده است اما authentication با password پیچیده‌ تر است. مدل قدیمی این بود که password verifier داخل /etc/passwd قرار بگیرد. این طراحی چند مشکل داشت :

- همه باید به password field دسترسی داشته باشند یا دست‌ کم مکانیزم‌ های زیادی برای محدود کردن آن لازم بود.
- روش password verification انعطاف کمی داشت.
- سیستم به password به‌ عنوان روش اصلی authentication وابسته بود.
- روش‌ هایی مانند token ، smart card یا biometric نیازمند implementation های جداگانه بودند.

این محدودیت‌ ها باعث شکل‌ گیری shadow password و بعد PAM شدند.

--- 

### Pluggable Authentication Modules

برای flexible کردن authentication ، سیستم PAM یا Pluggable Authentication Modules ایجاد شد. PAM یک architecture برای استفاده از shared authentication modules است. ایده اصلی :

```
Application
     ↓
    PAM
     ↓
Authentication Modules
     ↓
Password / Account / Session / Other mechanisms
```

لازم نیست Application خودش تمام جزئیات authentication را پیاده‌ سازی کند. مثلاً برنامه‌ ای مانند ```login``` می‌ تواند authentication را به PAM بسپارد. PAM سپس بر اساس configuration تصمیم می‌ گیرد :

- آیا password چک شود؟
- آیا account فعال باشد؟
- آیا shell مجاز باشد؟
- آیا session ساخته شود؟
- آیا password تغییر کند؟
- آیا MFA اجرا شود؟

و غیره. PAM در سال 1995 به‌عنوان یک استاندارد توسط Sun Microsystems پیشنهاد شد و در Linux به یک بخش بسیار مهم از authentication architecture تبدیل شد.

---

### PAM Configuration

 محل قرارگیری configuration مربوط به PAM معمولاً در /etc/pam.d/ قرار دارد. در برخی سیستم‌ ها فایل /etc/pam.conf نیز ممکن است مورد استفاده باشد. در /etc/pam.d معمولاً برای هر application یک فایل وجود دارد. مثلاً :

```
/etc/pam.d/login
/etc/pam.d/sshd
/etc/pam.d/sudo
/etc/pam.d/chsh
```
هر خط configuration سه بخش اصلی دارد شامل function-type و  control-argument  و module . مثلاً :

```auth requisite pam_shells.so```

یعنی :

```
auth
    ↓
function

requisite
    ↓
control

pam_shells.so
    ↓
module
```

#### 🔹 PAM Function Types

چهار function type اصلی :
1. auth

احراز هویت کاربر مثلاً آیا password درست است؟

2. account

وضعیت account را بررسی می‌ کند. مثلاً آیا account منقضی شده ؟ یا آیا کاربر اجازه استفاده از این سرویس را دارد؟

3. session

کارهای مربوط به session را انجام می‌ دهد. مثلاً ایجاد environment و نمایش پیام و ثبت session  و mount کردن resource خاص. 

4. password

برای تغییر password یا credential استفاده می‌ شود. یک module می‌ تواند برای چند function استفاده شود مثلاً ```pam_unix.so``` می‌ تواند هنگام auth برای بررسی password و هنگام password برای تغییر password استفاده شود. پس باید همیشه این دو را با هم دید ```function + module```.

#### 🔹 Control Arguments and Stacked Rules

یکی از ویژگی‌ های مهم PAM این است که rule ها به‌ صورت stack اجرا می‌ شوند مثلاً :

```
Rule 1
  ↓
Rule 2
  ↓
Rule 3
  ↓
Rule 4
```

نتیجه یک rule می‌ تواند تعیین کند که ادامه دهد یا متوقف شود یا authentication را موفق اعلام کند یا authentication  را شکست‌ خورده اعلام کند. سه control argument مهم هم sufficient و requisite و required هستند.

#### 🔹 sufficient

اگر rule موفق شود ، برای موفقیت authentication کافی است و PAM می‌ تواند ادامه rule ها را اجرا نکند. اگر rule شکست بخورد ، PAM می‌ تواند به rule های بعدی ادامه دهد. 

#### 🔹 requisite

اگر موفق شود continue و اگر شکست بخورد fail immediately. یعنی failure این rule باعث توقف فوری stack می‌ شود.

#### 🔹 requisite

اگر موفق شود continue اگر شکست بخورد remember failure و continue یعنی PAM rule های بعدی را نیز اجرا می‌کند ، اما در پایان authentication را fail اعلام خواهد کرد این تفاوت بسیار مهم است بنابراین requisite با required یکسان نیست.

#### 🔹 PAM example for chsh

کتاب برای توضیح stack یک نمونه برای authentication مربوط به chsh ارائه می‌ کند :

```
auth    sufficient    pam_rootok.so
auth    requisite     pam_shells.so
auth    sufficient    pam_unix.so
auth    required      pam_deny.so
```

جریان منطقی آن چنین است :

- مرحله اول : ```pam_rootok.so``` بررسی می‌ کند آیا کاربر root است. اگر root باشد و rule هم sufficient باشد ، authentication موفق اعلام می‌شود و مراحل بعدی لازم نیستند. اگر root نباشد، ادامه می‌ دهیم.
- مرحله دوم : ```pam_shells.so``` بررسی می‌ کند shell کاربر در ```/etc/shells``` وجود داشته باشد. اگر shell مجاز نباشد requisite باعث می‌ شود authentication فوراً fail شود. اگر shell مجاز باشد ، ادامه می‌ دهیم.
- مرحله سوم : ```pam_unix.so``` کارش password کاربر را بررسی می‌ کند. اگر password درست باشد و control برابر sufficient باشد ، authentication موفق است. اگر password اشتباه باشد ، stack ادامه پیدا می‌ کند.
- مرحله چهارم : ```pam_deny.so``` همیشه failure ایجاد می‌ کند. چون required است، PAM در نهایت authentication را fail می‌ کند.

> 💡 نکته مهم این است که required باعث توقف فوری نمی‌شود بلکه اگر rule های دیگری بعد از آن باشند ، PAM می‌ تواند آن‌ ها را نیز پردازش کند ، اما failure ثبت‌ شده همچنان نتیجه نهایی را شکست‌ خورده می‌ کند.

<img width="100%" height="913" alt="image" src="https://github.com/user-attachments/assets/0dd45709-923b-4e17-86ef-0b3fa3a1685d" />

#### 🔹 Advanced Control Syntax

در PAM فقط sufficient و required و requisite وجود ندارد. یک syntax پیشرفته‌ تر نیز وجود دارد که داخل [ ... ] قرار می‌ گیرد. این syntax اجازه می‌ دهد رفتار PAM را بر اساس return value دقیق module کنترل کنیم. مثلاً می‌ توان برای یک نوع return value گفت ```success → continue``` و برای return value دیگری ```failure → ignore```. این قسمت بسیار قدرتمند است و configuration PAM را شبیه یک زبان کوچک برای کنترل جریان می‌ کند. برای جزئیات کامل باید ```pam.conf(5)``` و manual مربوط به module را بررسی کرد.

#### 🔹 Module Arguments

این module ها می‌ توانند argument داشته باشند مثلاً ```auth sufficient pam_unix.so nullok``` اینجا ```nullok``` یک argument مربوط به pam_unix.so است. این option می‌تواند اجازه دهد account هایی که password ندارند ، تحت شرایط مشخص authentication شوند. به همین دلیل option های PAM باید با دقت بسیار زیادی تنظیم شوند چون یک option کوچک می‌ تواند policy امنیتی سیستم را تغییر دهد.

--- 

### Tips on PAM Configuration Syntax

تنظیمات PAM configuration می‌ تواند بسیار پیچیده شود. برای پیدا کردن module های PAM می‌توان از ```man -k pam_``` استفاده کرد. این command manual page هایی را که نام یا keyword مرتبط با pam_ دارند پیدا می‌ کند. برای بررسی location یک module نیز می‌ توان از ابزارهایی مانند locate pam_unix.so استفاده کرد ، البته در سیستم‌ هایی که database مربوط به locate در دسترس و به‌ روز باشد. همچنین manual page مربوط به هر module باید بررسی شود ، چون argument ها و رفتار دقیق module ها متفاوت است.

#### 🔹 /etc/pam.d/other

یکی از فایل‌ های مهم PAM در /etc/pam.d/other است. این فایل می‌ تواند policy پیش‌ فرضی را برای application هایی که configuration اختصاصی ندارند فراهم کند. در بسیاری از سیستم‌ ها policy پیش‌ فرض به‌ صورت محافظه‌ کارانه authentication را رد می‌ کند. این رفتار امنیتی مهم است ، چون application ناشناخته نباید صرفاً به دلیل نداشتن configuration اختصاصی ، authentication را بدون policy مشخص قبول کند.

#### 🔹 PAM and the Order of Rules

ترتیب rule ها اهمیت زیادی دارد مثلاً این دو configuration الزاماً رفتار یکسانی ندارند :

```
auth sufficient pam_unix.so
auth requisite pam_shells.so

and

auth requisite pam_shells.so
auth sufficient pam_unix.so
```

در اولی ممکن است password صحیح باعث شود قبل از بررسی shell authentication تمام شود و در دومی shell ابتدا بررسی می‌ شود. بنابراین در PAM باید همزمان به سه چیز توجه کرد  Module و Control argument و Order و همچنین module argument ها.

---

### PAM and Passwords

سیستم‌ های قدیمی shadow password مجموعه‌ ای از ابزارها و configuration های مربوط به password داشتند. با شکل‌گیری PAM بخش زیادی از policy های authentication به PAM منتقل شد. فایل ```/etc/login.defs``` هنوز ممکن است در سیستم‌های Linux وجود داشته باشد و برای برخی policy ها و default های مربوط به account management استفاده شود. اما نباید تصور کرد که تمام password policy مدرن در login.defs قرار دارد. در سیستم‌ های PAM-based ،  بسیاری از تنظیمات password توسط PAM module ها و configuration آن‌ ها تعیین می‌ شوند.

#### 🔹 pam_unix.so

یکی از مهم‌ ترین module های استاندارد PAM هم **pam_unix.so** است. این module می‌ تواند در function های مختلف رفتارهای متفاوتی داشته باشد. مثلاً ```auth``` برای بررسی credential و```password``` برای تغییر password استفاده شود. روش ذخیره و hash کردن password نیز به configuration و implementation مربوط به module  و سیستم بستگی دارد. بنابراین نباید به‌ صورت مطلق گفت : **«Linux همیشه password ها را با SHA-512 ذخیره می‌ کند.»** نوع hash و format باید از configuration واقعی سیستم بررسی شود.

---

### Tips

در این فصل بخش‌ هایی از user space را دیدیم که مستقیماً با مدیریت روزمره Linux ارتباط دارند. تا اینجا دیدیم که چگونه به هم متصل می‌ شوند :

```
Applications
      ↓
System libraries
      ↓
Configuration
      ↓
Users / Services / Logging / Scheduling
```

مهم‌ ترین موضوعات فصل عبارت بودند از :

```
- System Logging → journald / syslog / rsyslog
- etc → System configuration
- etc/passwd, /etc/shadow, /etc/group → User and group information
- getty → login → PAM → shell
- System Clock → RTC → Network Time
- cron + systemd timer → Scheduled Tasks
- UID / GID  → Process Credentials → Permissions → Identification → Authentication → Authorization
```

لایه PAM نیز یک لایه بسیار مهم در این میان ایجاد می‌ کند تا application ها مجبور نباشند خودشان تمام جزئیات authentication را پیاده‌ سازی کنند.

در نتیجه معماری کلی که در این فصل دیده می‌ شود ، مجموعه‌ای از component های کوچک‌ تر و مستقل است که هرکدام یک وظیفه مشخص دارند اما از طریق configuration ، library ، process و service با یکدیگر ارتباط برقرار می‌ کنند.

از اینجا به بعد کتاب دوباره تمرکز خود را به Kernel و process ها برمی‌ گرداند. فصل بعد وارد جزئیات بیشتری درباره process ها ، resource utilization ، CPU ، memory و I/O می‌ شود.
