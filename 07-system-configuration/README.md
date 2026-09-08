## تنظیمات سیستم 
### 🐧 فصل دوم کتاب How Linux Works

فصل‌ های قبل بیشتر روی زیرساخت داخلی Linux تمرکز داشتند. از Kernel و device ها و filesystem ها گرفته تا boot و شروع user space. در این فصل وارد بخش دیگری از سیستم می‌ شویم قسمت‌ هایی که باعث می‌ شوند برنامه‌ های user space بتوانند با configuration های سیستم ، کاربران ، زمان سیستم ، logging و کارهای زمان‌ بندی‌ شده تعامل داشته باشند.

---
📚 Table of Contents

- [system logging](#system-logging)
- [Checking Your Log Setup](#checking-your-log-setup)
- [Searching and Monitoring Logs](#searching-and-monitoring-logs)
- [Logfile Rotation](#logfile-rotation)
- [Journal Maintenance](#journal-maintenance)
- [A Closer Look at System Logging](#a-closer-look-at-system-logging)
- [User Management Files](#user-management-files)
- [The etc passwd File](#the-etc-passwd-file)
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

### The Structure of /etc

دایرکتوری ```etc/``` یکی از مهم‌ ترین دایرکتوری های Linux است. بخش بزرگی از configuration مربوط به یک سیستم در این دایرکتوری قرار دارد. در سیستم‌ های قدیمی‌ تر، تعداد زیادی برنامه configuration خود را مستقیماً به‌ صورت فایل‌ های جداگانه در ```etc/``` قرار می‌ دادند. با افزایش تعداد package ها ، این روش باعث می‌ شد ```etc/```  شلوغ شود. مشکل دیگری نیز وجود داشت ، فرض کنید package یک فایل configuration مانند ```etc/example.conf/``` را نصب کند. administrator آن را برای نیازهای سیستم تغییر دهد بعد package update شود. اگر package فایل را دوباره جایگزین کند ، ممکن است customization administrator از بین برود. به همین دلیل در سیستم‌ های جدید تر استفاده از subdirectory ها و configuration های جداگانه رایج‌ تر شده است مثلاً ```/etc/systemd/``` یا ```/etc/rsyslog.d/``` یا ```/etc/pam.d/``` در این مدل، configuration های local می‌ توانند در محل مناسب خود قرار بگیرند و فایل‌ های پیش‌فرض package نیز کمتر در معرض تغییر مستقیم باشند.

#### 🔹 What should be in /etc ? 

یک guideline خوب این است که configuration های قابل شخصی‌ سازی مربوط به یک ماشین در ```etc/```   قرار بگیرند مثلاً ```etc/passwd/``` اطلاعات کاربران local را نگه می‌ دارد یا ```/etc/network/``` در سیستم‌ هایی که از آن استفاده می‌ کنند ، configuration شبکه را در خود دارد. در مقابل ، configuration های default که قرار نیست administrator مستقیماً تغییر دهد ، ممکن است در محل‌ هایی مانند ```/usr/lib/``` قرار بگیرند. مثلاً unit های systemd ارائه‌ شده توسط package ها معمولاً در مسیرهایی مانند ```/usr/lib/systemd/system/``` قرار دارند ، در حالی که override ها و unit های local در ```/etc/systemd/system/``` قرار می‌ گیرند.

---

### User Management Files

در Linux و Unix از چندین کاربر مستقل پشتیبانی می‌ کنند. اما Kernel مفهوم ```username``` را نمی‌ شناسد. Kernel بیشتر با ```UID``` کار می‌ کند مثلاً ```UID 0``` یا ```UID 1000``` یا ```UID 1001``` برای Kernel شناسه‌ های عددی هستند. نام‌ هایی مانند ```root``` و ```alice``` و ```bob``` در user space معنا پیدا می‌ کنند.

بنابراین وقتی یک برنامه بخواهد بداند UID 1000 متعلق به چه کاربری است ، باید یک mapping از UID به username پیدا کند. به‌ طور سنتی این اطلاعات در```/etc/passwd``` قرار دارند. اما در سیستم‌ های مدرن الزاماً همه کاربران از ```/etc/passwd```  نمی‌آیند. می‌ توان از سرویس‌ های شبکه‌ ای مانند LDAP یا سیستم‌ های دیگر نیز برای user database استفاده کرد.

---

### The etc passwd File














