## دستورات پایه‌ ی لینوکس چطور واقعاً کار می‌کنند؟ 
### 🐧 فصل دوم کتاب How Linux Works

فصل اول کتاب معماری کلی یک سیستم لینوکس را معرفی کرد:  ``` Hardware → Kernel → User Space ```

در فصل دوم یک قدم جلوتر می‌ رویم و سراغ ابزارهایی میرویم که تقریباً هر روز هنگام کار با لینوکس از آن‌ها استفاده می‌کنیم. اما هدف این فصل فقط یاد گرفتن چند دستور نیست. هدف این است که بفهمیم **این دستورات در پشت صحنه چگونه با Shell و Process ها و Kernel و Filesystem ارتباط برقرار می‌کنند**. وقتی این ارتباط را درک کنیم ، بسیاری از رفتارهایی که در ابتدا عجیب به نظر می‌رسند ، کاملاً منطقی می‌ شوند.

---
📚 Table of Contents

- [Shell](#shell)
- [Stdin and stdout and stderr](#stdin-and-stdout-and-stderr)
- [File commands](#file-commands)
- [Navigating Directories](#navigating-directories)
- [Why should cd be built-in ?](#why-should-cd-be-built-in)
- [mkdir and rmdir](#mkdir-and-rmdir)
- [Globbing or Wildcards](#globbing-or-wildcards)
- [grep and Regular Expression](#grep-and-regular-expression)
- [less](#less)
- [diff](#diff)
- [file](#file)
- [find](#file)
- [locate](#locate)
- [head and tail and sort](#head-and-tail-and-sort)
- [passwd](#passwd)
- [chsh](#chsh)
- [dot files](#dot-files)
- [Shell Variables and Environment Variables](#shell-variables-and-environment-variables)
- [PATH Variable](#path-variable)
- [Special characters and command line shortcuts](#special-characters-and-command-line-shortcuts)
- [Text Editors](#text-editors)
- [man](#man)
- [Redirection](#redirection)
- [pipe](#pipe)
- [stderr and Redirection](#stderr-and-redirection)
- [Error Message](#error-message)
- [List and manage processes](#list-and-manage-processes)
- [Signals](#signals)
- [Job Control in Shell](#job-control-in-shell)
- [introduction to permissions](#introduction-to-permissions)
- [Symbolic Link](#symbolic-link)
- [Compressing and archiving with gzip and tar](#Compressing-and-archiving-with-gzip-and-tar)
- [Linux Directory Overview](#linux-directory-overview)
- [kernel and boot](#kernel-and-boot)
- [Running commands with Superuser access](running-commands-with-superuser-access)
- [Big picture](#big-picture)
- [From Commands to Understanding Linux](#from-commands-to-understanding-linux)
- [Key points of the second season](#key-points-of-the-second-season)
- [Tips](#tips)
  
---

### Shell

شل (Shell) مثل bash ، فقط یک برنامه برای دریافت دستورات نیست. Shell در واقع یک محیط تعاملی و در عین حال یک محیط برنامه‌نویسی کوچک است که به ما اجازه می‌ دهد برنامه‌ های مختلف را با یکدیگر ترکیب کنیم. این موضوع یکی از ایده‌ های اصلی Unix است:

> 💡 هر ابزار یک کار مشخص را به‌خوبی انجام دهد و بتوان ابزارها را در کنار یکدیگر قرار داد.

وقتی دستوری را در Shell وارد می‌کنیم، اتفاقات مختلفی در پشت صحنه رخ می‌ دهد. Shell ابتدا خط فرمان را parse می‌ کند و مواردی زیر را بررسی می‌کند سپس اگر لازم باشد Process جدیدی ایجاد می‌کند :

- Command name
- Arguments
- Quoting
- Globbing
- Redirection
- Pipe
- Variable Expansion


یکی از مفاهیم مهم اینجا fork() است. با ()fork یک Process جدید ایجاد می‌شود که در ابتدا یک کپی از Parent Process است. بعد Child Process معمولاً از یکی ازsystem call های ()exec استفاده می‌ کند تا برنامه‌ ی جدید را اجرا کند. در این حالت، Child Process برنامه‌ ی قبلی خودش را با برنامه‌ ی جدید جایگزین می‌کند. برای مثال وقتی می‌ نویسیم ```ls``` باید Shell برنامه‌ی ls را پیدا کند و آن را اجرا کند. به‌صورت ساده می‌توان این فرایند را چنین تصور کرد:
```
Shell
  │
  ├── fork()
  │
  └── Child Process
          │
          └── exec() → ls
```
به همین دلیل ()fork و()exec از مفاهیم بسیار مهم در درک نحوه‌ ی اجرای برنامه‌ ها در Unix و Linux هستند.

---

### stdin and stdout and stderr 

یکی از مهم‌ ترین ایده‌ هایی که در این فصل با آن آشنا می‌شویم Standard I/O است. هر Process به‌ طور معمول سه File Descriptor استاندارد دارد: 
- 0 → stdin
- 1 → stdout
- 2 → stderr

#### 🔹 stdin

ورودی stdin یا Standard Input ، ورودی استاندارد Process است. در یک Shell معمولی، این ورودی معمولاً به Terminal متصل است. برای مثال اگر بنویسیم : ```cat``` و هیچ فایلی به آن ندهیم، cat منتظر دریافت ورودی می‌ماند. هر چیزی که تایپ کنیم، cat همان را روی خروجی چاپ می‌کند. 

<img width="100%" height="130" alt="image" src="https://github.com/user-attachments/assets/c9d69edd-b0cb-4b9e-bdac-4ee25e8fbea3" />

هر چیزی که تایپ کنیم ، cat همان را روی خروجی چاپ می‌کند. برای پایان دادن به ورودی می‌توانیم ```Ctrl+D``` را بزنیم. ```Ctrl+D``` در اینجا در واقع یک کاراکتر معمولی نیست. در Terminal باعث می‌شود وضعیت EOF به Process منتقل شود، یعنی داده‌ ی بیشتری برای خواندن وجود ندارد.

#### 🔹 stdout

خروجی stdout یا Standard Output ، خروجی معمولی Process است. مثلاً ```echo hello``` خروجی hello را روی stdout می‌نویسد.

<img width="100%" height="60" alt="image" src="https://github.com/user-attachments/assets/5c986079-4a8b-4d60-9477-1103a40f45b5" />


#### 🔹 stderr

جریان stderr یا Standard Error d یک جریان جداگانه‌ ای برای پیام‌ های خطا است. برای مثال ```ls /does-not-exist``` پیام خطا از stderr ارسال می‌شود. این جداسازی بسیار مهم است، زیرا می‌توانیم خروجی عادی و خطاها را به مقصدهای متفاوتی بفرستیم.


<img width="100%" height="60" alt="image" src="https://github.com/user-attachments/assets/90b81fd9-2470-4574-8d0d-59fc78ad6e1d" />

---

### File commands

دستورهای ساده‌ ای مثل ```ls, cp, mv, touch, rm``` شاید در نگاه اول فقط ابزارهای روزمره باشند، اما هرکدام نمونه‌ ای از نحوه‌ ی تعامل User Space با Kernel و Filesystem هستند.
#### 🔹 ls

دستور ```ls``` اطلاعات مربوط به یک Directory را نمایش می‌دهد. مثلاً ```ls``` یا ```ls -l``` ، در حالت l- اطلاعات بیشتری مانند : 

- File type
- Permissions
- Owner
- Group
- Size
- Last modified time
- File name

<img width="100%" height="71" alt="image" src="https://github.com/user-attachments/assets/c1082e53-8ca3-4c0c-82f1-46d390a6c74a" />


دستور ```ls```  برای دریافت این اطلاعات از امکانات سیستم فایل و system call های مرتبط استفاده می‌ کند. 

#### 🔹 cp

دستور cp برای کپی کردن فایل‌ ها استفاده می‌شود. در سطح مفهومی، کپی کردن یک فایل یعنی : 
```
Read source
     ↓
Write destination
```

یعنی داده از فایل مبدأ خوانده شده و در فایل مقصد نوشته می‌شود. البته پیاده‌سازی واقعی cp می‌تواند بسته به نوع فایل،   Filesystem ، بهینه‌ سازی‌ ها و گزینه‌ های استفاده‌ شده پیچیده‌ تر باشد. 

``` $ cp file1 file2 ```
``` $ cp file dir ```

#### 🔹 mv

دستور mv برای move یا rename  نام فایل‌ ها استفاده می‌ شود. 

> 💡 یک نکته‌ی مهم : اگر مبدأ و مقصد روی یک Filesystem باشند، mv معمولاً نیازی ندارد کل داده‌ ی فایل را دوباره کپی کند. در چنین حالتی عملیات می‌تواند عمدتاً با تغییر Entry های directory انجام شود. به همین دلیل جا به‌ جا کردن یک فایل بزرگ در یک Filesystem معمولاً بسیار سریع است. اما اگر مبدأ و مقصد روی Filesystem های متفاوت باشند، دیگر چنین جا به‌ جایی ساده‌ای ممکن نیست و mv باید عملاً داده را به مقصد منتقل کند.

``` $ mv file1 file2 ```

#### 🔹 touch

دستور:

``` $ touch file.txt ```

اگر فایل وجود نداشته باشد، معمولاً آن را ایجاد می‌کند. اگر فایل از قبل وجود داشته باشد، محتوای آن را تغییر نمی‌ دهد و زمان‌های مربوط به فایل را به‌ روز رسانی می‌کند. این ویژگی باعث شده touch در Shell Script ها و Automation نیز کاربرد داشته باشد.

#### 🔹 rm

دستور:

``` $ rm file.txt ```

برای حذف یک فایل استفاده می‌شود. در Unix، حذف فایل معمولاً به معنای حذف نام یا Directory Entry مربوط به آن فایل است به همین دلیل مفهوم unlink اهمیت دارد. اگر دیگر هیچ Hard Link  به data وجود نداشته باشد و Process دیگری نیز فایل را باز نگه نداشته باشد ، فضای مربوط به فایل در نهایت قابل استفاده‌ ی مجدد خواهد بود. بنابراین نباید تصور کنیم که rm یعنی **فوراً bit های data از روی دیسک پاک می شوند** ، اما از دید کاربر، فایل دیگر از مسیر معمول قابل دسترسی نیست و مهم‌تر از همه در Shell معمولی، rm سطل زباله ندارد بنابراین : 

``` $ rm -r directory ```

را باید با دقت بسیار زیادی استفاده کرد. 

#### 🔹 echo

دستور:

``` $ echo hello ```

آرگومان‌های خود را روی stdout چاپ می‌کند. echo به‌خصوص برای مشاهده‌ ی مقدار متغیرهای Shell و آزمایش رفتار Shell بسیار کاربردی است :

``` echo $PATH ```

<img width="100%" height="40" alt="image" src="https://github.com/user-attachments/assets/3ee39428-cc80-4a75-bbf4-63838f446f86" />

---

### Navigating directories

تمام ساختار File system لینوکس از یک نقطه شروع می‌شود ```/``` ،  این همان Root Directory است. نباید آن را با کاربر root اشتباه گرفت.

#### 🔹 Absolute Path

مسیری که از ```/``` شروع شود، یک Absolute Path است:

```
/etc/passwd
/var/log/syslog
/home/user
```

#### 🔹 Relative Path
مسیری که از ```/``` شروع نشود، معمولاً یک Relative Path است:
```
notes.txt
./notes.txt
../notes.txt
```
این مسیر نسبت به Directory فعلی تفسیر می‌شود. 

#### 🔹 '.' & '..'

دو نماد بسیار مهم:
.   → Current Directory
..  → Patent Directory
مثلاً ``` .. cd``` یک سطح به Directory والد برمی‌گردد.


#### 🔹 pwd
دستور```pwd``` ، مسیر دایرکتوری کاری فعلی Shell را نمایش می‌دهد. مثلاً:
```
$ pwd
/home/user
```
اگرچه ممکن است مسیر را در Prompt نیز ببینیم، اما این موضوع به تنظیمات Shell بستگی دارد. همچنین Symbolic Link ها می‌ توانند باعث شوند مسیری که می‌بینیم با مسیر physical  متفاوت باشد. برای درخواست نمایش مسیر physical می‌ توان از```pwd -P``` استفاده کرد.


<img width="100%" height="40" alt="image" src="https://github.com/user-attachments/assets/87fce485-57db-40ef-bb63-91ced0acc072" />

---

### Why should cd be built-in

یکی از نکات جالب این فصل، دلیل Built-in بودن ```cd``` است. در ظاهر ممکن است فکر کنیم ```cd``` هم مثل ```ls``` یک برنامه‌ی معمولی است. اما اگر ```cd``` یک Process مستقل بود، تغییر Directory فقط در همان Process اتفاق می‌افتاد. برای مثال :
```
Shell
│
├── fork()
│
└── cd process
  │
  └── change working directory
```

وقتی Child Process تمام می‌شد، Shell اصلی همچنان در Directory قبلی باقی می‌ماند. اما هدف ما این است که **خود Shell  Directory** کاری‌ اش را تغییر دهد. به همین دلیل ```cd``` باید بتواند وضعیت داخلی Shell را تغییر دهد و در Shell هایی مثل Bash به‌ صورت Built-in پیاده‌ سازی شده است.
---

### mkdir and rmdir

برای ایجاد Directory از دستور ```mkdir mydir``` و برای حذف یک Directory خالی از دستور ```rmdir mydir``` استفاده می‌کنیم. 

<img width="100%" height="110" alt="image" src="https://github.com/user-attachments/assets/7e2f7b62-ede6-4601-ba7a-8be4bdbc0e60" />


دستور **rmdir** فقط Directory خالی را حذف می‌کند. برای حذف یک Directory همراه با محتویات آن از دستور ```rm -r mydir``` استفاده می‌شود.

<img width="100%" height="141" alt="image" src="https://github.com/user-attachments/assets/477305b3-2a63-494a-b101-9ed8c0944c8f" />

> ⚠️ هنگام کار با root باید با این دستور بسیار محتاط بود.
---

### globbing or wildcards

شل (Shell) می‌تواند الگوهایی را روی نام فایل‌ها اعمال کند ، دو مورد مهم :

- *  → هر تعداد کاراکتر
- ?  → دقیقاً یک کاراکتر

مثلاً می‌تواند فایل‌هایی را پیدا کند :
<img width="100%" height="40" alt="image" src="https://github.com/user-attachments/assets/ff15014f-c1eb-4b87-a68e-26b9c38df5a3" />

نکته‌ی بسیار مهم این است که معمولاً خود ```ls``` **عملیات Globbing را انجام نمی‌دهد**. این Shell است که قبل از اجرای برنامه، الگو را Expand می‌کند. مثلاً:
``` ls *.log ```
ممکن است قبل از اجرای ```ls``` به چیزی شبیه این تبدیل شود:
``` ls error.log system.log access.log```
بنابراین ```ls``` دیگر * را نمی‌بیند. این طراحی باعث می‌شود برنامه‌ های Unix ساده‌ تر باشند، چون لازم نیست هر برنامه منطق Wildcard را خودش پیاده‌ سازی کند.

---

### grep and regular expression

#### 🔹 grep

دستور ```grep``` برای پیدا کردن خطوطی استفاده می‌ شود که با یک Pattern مطابقت دارند. 
مثلاً:
``` grep "error" logfile.txt ```
تمام خطوطی را که شامل error هستند نمایش می‌دهد. وقتی grep روی چند فایل اجرا شود، معمولاً نام فایل نیز در خروجی نمایش داده می‌شود.
مثلاً:
``` grep "error" *.log```
اینجا ابتدا Shell اول ```log.*``` را Expand می‌کند و سپس ```grep``` فایل‌های واقعی را دریافت می‌کند.

#### 🔹 Regular Expression (RegEx)

دستور```grep``` همچنین می‌تواند از Regular Expression برای جستجوی الگوهای پیچیده‌ تر استفاده کند. Regular Expression موضوع گسترده‌ ای است و در بسیاری از ابزارهای Unix و زبان‌های برنامه‌ نویسی کاربرد دارد. بنابراین ```grep``` فقط یک ابزار ساده برای جستجوی متن نیست بلکه می‌تواند به یکی از ابزارهای اصلی پردازش متن در Linux تبدیل شود.

---

### less

اگر فایل بزرگی داشته باشیم، اجرای ```cat large.log``` ممکن است حجم زیادی از اطلاعات را به سرعت روی Terminal چاپ کند. برای مرور فایل‌ های بزرگ بهتر است از دستور ```less``` استفاده کنیم : 

``` less large.log ```

در less:
- Space → Next page
- b → Previous page
- q → Exit
- / → Search forward
- ? → Search backward
- n → Next result

یکی از کاربردهای بسیار مهم:
``` grep "error" logfile.txt | less ```
است. در اینجا خروجی ```grep``` از طریق Pipe مستقیماً وارد ```less``` می‌شود.

---

### diff

دستور```diff``` برای مقایسه‌ی دو فایل استفاده می‌شود.
``` diff file1 file2 ```

گزینه‌ ی:
``` diff -u file1 file2 ```
خروجی را در قالب Unified Diff نمایش می‌دهد؛ فرمتی که برای ابزارهای مختلف و بررسی تغییرات بسیار کاربردی است.

---

### file

دستور ```file``` نوع فایل را بر اساس اطلاعات موجود در خود فایل تشخیص می‌دهد . این نکته مهم است، زیرا Linux الزاماً نوع فایل را از پسوند آن تعیین نمی‌کند. بنابراین حتی اگر نام یک فایل گمراه‌ کننده باشد، ```file``` می‌تواند با بررسی محتوای آن اطلاعات مفیدی ارائه کند.

<img width="100%" height="70" alt="image" src="https://github.com/user-attachments/assets/42347dce-759f-44ca-810d-48cc5d42816d" />

---

### find

برای جستجوی بازگشتی در Directory ها از find استفاده می‌کنیم. مثلاً:
``` find . -name '*.log' ```

<img width="100%" height="70" alt="image" src="https://github.com/user-attachments/assets/e7d75caa-f538-47c0-865a-ca4ccc307324" />

برای جستجوی بازگشتی در Directory ها از find استفاده می‌کنیم. مثلاً:
``` find . -name '*.log' ```

این دستور در Directory فعلی و زیرشاخه‌ های آن به دنبال فایل‌ هایی با الگوی ```log.``` می‌گردد.نکته‌ی بسیار مهم این است که الگوی *.log را داخل کوتیشن قرار داده‌ایم:
``` '*.log' ```
چرا؟ چون نمی‌خواهیم Shell قبل از اجرای ```find``` این Pattern را Expand کند. ما می‌خواهیم خود ```find``` این Pattern را دریافت و تفسیر کند.

---

### locate

دستور ```locate``` نیز برای پیدا کردن فایل‌ها استفاده می‌شود، اما روش کار آن با ```find``` متفاوت است. ```find``` معمولاً Filesystem را هنگام اجرای دستور جستجو می‌کند. اما ```locate``` معمولاً یک Database یا Index از نام فایل‌ها را جستجو می‌کند. به همین دلیل ```locate``` معمولاً سریع‌ تر و ```find``` جستجوی مستقیم‌ تر و انعطاف‌ پذیرتراست. اما Index مربوط به ```locate``` ممکن است به‌ روز نباشد بنابراین ممکن است فایلی که اخیراً ایجاد شده هنوز در نتایج ```locate``` وجود نداشته باشد.

---

### head and tail and sort

#### 🔹head & tail
برای مشاهده‌ ی ابتدای یک فایل از دستور : ``` head file.txt ``` استفاده می کنیم و برای مشاهده‌ ی انتهای آن از دستور ``` tail file.txt ``` استفاده می‌کنیم. به‌صورت معمول هرکدام ۱۰ خط را نمایش می‌دهند. برای مشخص کردن تعداد خطوط:
```
head -n 20 file.txt 
tail -n 20 file.txt
```
#### 🔹 Sort

این دستور```sort file.txt``` برای مرتب کردن خطوط استفاده می شود.

<img width="100%" height="130" alt="image" src="https://github.com/user-attachments/assets/f45c1519-7294-4486-9a44-8e5052d6386d" />

برای مرتب‌ سازی عددی از دستور ```sort -n file.txt``` استفاده می‌شود.
<img width="100%" height="130" alt="image" src="https://github.com/user-attachments/assets/cf8af6b7-b25b-477f-9233-fbc6bee9856a" />

برای مرتب‌ سازی معکوس از دستور ```sort -r file.txt``` می‌توان استفاده کرد.
<img width="100%" height="130" alt="image" src="https://github.com/user-attachments/assets/52536730-dd1f-47e0-9f8e-42abc00589bf" />

این ابزارها وقتی با Pipe ترکیب شوند بسیار قدرتمندتر می‌ شوند: 
<img width="100%" height="151" alt="image" src="https://github.com/user-attachments/assets/7a731c8b-ab09-4445-b3ab-8b1c5a08c2eb" />

---

### passwd

دستور```passwd``` برای تغییر Password کاربر استفاده می‌شود. کتاب همچنین درباره‌ ی انتخاب Password مناسب صحبت می‌کند و بر اهمیت استفاده از رمزهای عبور مناسب و قابل‌ حفظ تأکید دارد.

---

### chsh

برای تغییر Shell پیش‌ فرض کاربر می‌ توان از```chsh``` استفاده کرد. برای مثال Shell هایی مانند:
- bash
- zsh
- ksh

می‌ توانند به‌عنوان Shell پیش‌ فرض استفاده شوند. مثال‌های کتاب عمدتاً بر پایه‌ی bash هستند.

---

### dot files

فایل‌ها و Directory هایی که نامشان با ```.``` شروع می‌شود، معمولاً Dot Files نامیده می‌شوند.
مثلاً:
- .bashrc
- .profile
- .ssh

این فایل‌ ها ویژگی جادویی یا نوع متفاوتی از Filesystem ندارند در واقع فقط نامشان با ```.``` شروع می‌شود. ابزارهایی مانند ls به‌ صورت پیش‌ فرض آن‌ها را نمایش نمی‌ دهند و برای مشاهده‌ ی آن‌ ها از دستور```ls -a``` استفاده می‌کنیم.

<img width="100%" height="40" alt="image" src="https://github.com/user-attachments/assets/1ce48590-ce7e-4cd5-8dc2-881b05910fec" />

> 💡 در رابطه با Dot Files  و Globbing یک نکته‌ ی مهم این است که Pattern معمولی ```*``` به‌ صورت پیش‌ فرض با نام‌ هایی که با ```.``` شروع می‌شوند Match نمی‌شود.بنابراین ```* ls``` لزوماً تمام فایل‌های Directory را نشان نمی‌دهد ، حتی Pattern ```*.```نیز می‌تواند با ```. , ..``` مطابقت داشته باشد. به همین دلیل هنگام کار با Dot Files باید Pattern ها را با دقت انتخاب کرد.

---

### Shell Variables and Environment Variables

شل (Shell) می‌تواند متغیرهایی برای نگهداری اطلاعات داشته باشد مثلاً:```STUFF=value``` که برای برای دریافت مقدار از دستور ```echo $STUFF``` استفاده می‌کنیم. اما بین Shell Variable و Environment Variable تفاوت وجود دارد.

#### 🔹 Shell Variable

یک Shell Variable در محیط همان Shell وجود دارد مثلاً ```STUFF=value```.

#### 🔹 Environment Variable

برای اینکه متغیر به Process هایی که Shell اجرا می‌کند منتقل شود، باید آن را Export کنیم : ```export STUFF ```. بعد Child Process ها می‌توانند این مقدار را در Environment خود دریافت کنند. این موضوع یکی از روش‌ های مهم انتقال Configuration به برنامه‌ هاست.

---

### PATH Variable

یکی از مهم‌ ترین Environment Variable ها ، PATH هستش و برای مشاهده‌ی آن از دستور ```echo $PATH``` استفاده میکنیم. ممکن است چیزی شبیه این ببینیم:
```/usr/local/bin:/usr/bin:/bin```

Directoryها با ```:``` از یکدیگر جدا می‌شوند و وقتی می‌نویسیم ```ls``` باید Shell بفهمد فایل اجرایی ls کجاست. برای این کار Directory های موجود در PATH را به ترتیب بررسی می‌کند. ترتیب اهمیت دارد ، اگر چند فایل اجرایی با نام یکسان وجود داشته باشند، موردی که زودتر در PATH پیدا شود، معمولاً انتخاب می‌شود. 
می‌توان Directory جدیدی را به ابتدای PATH اضافه کرد:
``` PATH=dir:$PATH ```

یا به انتهای آن:
``` PATH=$PATH:dir ```

این موضوع درک نحوه‌ ی اجرای برنامه‌ها در Shell را بسیار مهم می‌کند.

---

### Special characters and command line shortcuts

در کار با Shell با Character های خاص زیادی مواجه می‌شویم.
- *    → Star / Asterisk
- |    → Pipe
- ~    → Tilde
- `    → Backtick
- #    → Hash
شناختن نام این Character ها هنگام مطالعه‌ی Documentation و صحبت با سایر برنامه‌ نویس‌ها و System Administrator ها مفید است.

شل (Shell) همچنین shortcuts هایی برای ویرایش سریع Command Line دارد.
- Ctrl+A → ابتدای خط
- Ctrl+E → انتهای خط
- Ctrl+W → حذف کلمه‌ی قبلی
- Ctrl+U → حذف از مکان‌نما تا ابتدای خط
- Ctrl+K → حذف از مکان‌نما تا انتهای خط
یاد گرفتن این shortcuts ها شاید در ابتدا جزئی به نظر برسد، اما هنگام کار طولانی با Terminal سرعت کار را به شکل محسوسی افزایش می‌ دهد.
---

###  Text Editors

برای کار جدی با Unix/Linux باید بتوانیم فایل‌ های متنی را ویرایش کنیم. بخش بزرگی از Configuration سیستم، به‌خصوص در etc/ به شکل Text File ذخیره می‌شود. کتاب در این بخش به دو ویرایشگر کلاسیک می‌پردازد:
- vi
- Emacs

#### 🔹 Emacs

ویرایشگر Emacs یک محیط بسیار قدرتمند و قابل توسعه است و امکانات گسترده‌ ای برای ویرایش و کار با متن ارائه می‌دهد. راهنمای داخلی آن نیز یکی از ویژگی‌ های مهم آن است.

#### 🔹 vi

ویرایشگر vi نیز اهمیت بسیار زیادی در دنیای Unix دارد.یکی از دلایل اهمیت آن این است که در بسیاری از محیط‌ های Unix و Linux در دسترس است و برای کار از طریق Terminal طراحی شده است. برای System Administrator ها دانستن حداقل اصول کار با یک ویرایشگر مانند vi بسیار ارزشمند است.

---

###  man

هیچ System Administrator ای تمام دستورات Linux را حفظ نیست. به همین دلیل یکی از مهارت‌ های مهم این است که بدانیم چطور اطلاعات مورد نیازمان را پیدا کنیم.

#### 🔹 man

اولین ابزار مهم ```man``` است. مثلاً ```man ls``` که صفحه‌ی Manual مربوط به ```ls``` را نمایش می‌دهد. صفحات man بیشتر برای **Reference** طراحی شده‌ اند، نه آموزش قدم‌ به‌ قدم.بنابراین ممکن است در ابتدا کمی خشک و فنی به نظر برسند.

#### 🔹 man -k

اگر نام دقیق دستور را نمی‌دانیم ، می‌توانیم Keyword جستجو کنیم مثلاً ```man -k sort``` این دستور صفحات Manual مرتبط با عبارت مورد نظر را پیدا می‌کند.
بخش‌های مختلف Manual
صفحات man به Sections مختلف تقسیم شده‌اند و برخی از مهم‌ ترین آن‌ ها :

- 1 → User commands
- 2 → System calls
- 3 → Library functions
- 4 → Device files / special files
- 5 → File formats / configuration files
- 8 → System administration commands


بنابراین ممکن است یک نام، چند صفحه‌ی Manual داشته باشد مثلاً ```man passwd``` معمولاً به Manual مربوط به Command passwd می‌رود. اما ```man 5 passwd``` به Section مربوط به File Format می‌ رود و ساختار /etc/passwd را توضیح می‌دهد. این موضوع اهمیت Section های man را نشان می‌دهد.

#### 🔹 info

در کنار man ابزار دیگری به نام ```info```وجود دارد. ```info``` توسط پروژه‌ی GNU برای Documentation گسترده‌ تر طراحی شده و بعضی پروژه‌های GNU اطلاعات بیشتری را در قالب Info ارائه می‌کنند.
#### 🔹 usr/share/doc/
بعضی Package ها نیز Documentation خود را در مسیرهایی مانند:
``` /usr/share/doc ```
قرار می‌دهند.بنابراین وقتی دنبال اطلاعات یک Package هستیم، فقط به man محدود نیستیم.

---

###  Redirection

حالا که مفهوم stdin، stdout و stderr را می‌دانیم، می‌توانیم جریان‌ های ورودی و خروجی را کنترل کنیم.
#### 🔹 > 
برای فرستادن stdout به یک فایل از دستور : 
```command > output.txt```

استفاده می‌کنیم. اگر فایل وجود نداشته باشد، ایجاد می‌شود و اگر وجود داشته باشد ، به‌ صورت معمول محتوای قبلی آن جایگزین می‌شود. به این رفتار **Clobbering** گفته می‌شود.

#### 🔹 >>
اگر بخواهیم خروجی را به انتهای فایل اضافه کنیم از دستور : 
```command >> output.txt```
استفاده می‌کنیم. در این حالت محتوای قبلی حفظ می‌شود.

---

###  pipe


یکی از قدرتمندترین ویژگی‌های Unix این است که stdin و stdout فقط به Keyboard و Terminal محدود نیستند. می‌توانیم خروجی یک Process را مستقیماً به ورودی Process دیگری وصل کنیم.این کار با Pipe انجام می‌شود:

```grep "error" logfile.txt | less```

در اینجا:

```
Logfile.txt
│ 
▼ 
Grep
│
Stdout
│
▼
Pipe
│
Stdin
│ 
▼ 
less
```

لازم نیست grep بداند less وجود دارد. less هم لازم نیست بداند خروجی از grep آمده است. Shell این اتصال را برای ما ایجاد می‌کند. این همان فلسفه‌ی معروف Unix است: **ابزارهای کوچک و مستقل را می‌توان برای انجام کارهای پیچیده‌ تر با یکدیگر ترکیب کرد.**

برای اتصال stdout یک برنامه به stdin برنامه‌ی دیگر از pipe استفاده می‌کنیم : 

```command1 | command2```


می‌توانیم چندین Command را نیز به هم متصل کنیم : 

```command1 | command2 | command3 | command4```

این یکی از پایه‌های اصلی فلسفه‌ی Unix است.

---

###  stderr and Redirection

جریان های stdout و stderr دو جریان متفاوت هستند مثلاً:

```ls existing-file > output.txt```

خروجی عادی به فایل می‌ رود ، اما اگر خطایی رخ دهد ، ممکن است پیام خطا همچنان روی Terminal نمایش داده شود. دلیل آن این است که پیام خطا از stderr آمده است.
برای Redirect کردن stderr:

```command 2> errors.txt```

استفاده می‌کنیم. عدد 2 همان File Descriptor مربوط به stderr است. به‌ صورت مشابه:
- 0 → stdin
- 1 → stdout
- 2 → stderr

#### 🔹 Redirecting stdout and stderr together

برای فرستادن stderr به همان مقصد stdout می‌توان از ```2>&1``` استفاده کرد مثلاً:
```command > output.txt 2>&1```
در این حالت هر دو جریان به output.txt فرستاده می‌شوند.


#### 🔹 stdin with <

می‌توان ورودی استاندارد را نیز از یک فایل گرفت:
```command < input.txt```
اما بسیاری از ابزارهای Unix مستقیماً نام فایل را به‌عنوان Argument قبول می‌کنند، بنابراین استفاده از < در همه‌ ی موارد ضروری نیست.

---

###  Error Message

یکی از مهارت‌های مهم System Administration ، خواندن **Error Message** است. پیام‌ های خطای Unix/Linux معمولاً اطلاعات مفیدی ارائه می‌کنند. برای مثال ```No such file or directory``` ، در سطح System Call معمولاً با این Error Code مرتبط است ```ENOENT```.
#### 🔹 Common errors

- خطای No such file or directory  : یعنی فایل یا مسیر مورد نظر پیدا نشده است.
- خطای File exists : یعنی چیزی که می‌خواهید ایجاد کنید، از قبل وجود دارد.
- خطای Permission denied : یعنی Process دسترسی لازم برای انجام عملیات را ندارد.
- خطای  Segmentation fault : این خطا معمولاً زمانی رخ می‌دهد که برنامه به شکلی نامعتبر به حافظه دسترسی پیدا کند و Kernel Process را متوقف کند.

#### 🔹 First Error

اگر یک برنامه چندین Error تولید می‌کند، همیشه لازم نیست همه‌ ی آن‌ها را هم‌ زمان بررسی کنیم. گاهی اولین Error باعث ایجاد تمام Error های بعدی شده است. بنابراین یک روش عملی مناسب این است : 
First Error → Find Root Cause → Fix it → Check remaining errors

همچنین باید تفاوت Warning و Error را بدانیم. **Warning** لزوماً به معنای توقف برنامه نیست! ممکن است برنامه با وجود Warning به اجرای خود ادامه دهد.

---

###  List and manage processes

همان‌ طور که در فصل اول دیدیم ، Process یعنی یک برنامه‌ ی در حال اجرا و هر Process یک شناسه‌ ی عددی دارد که به آن PID یا **Process ID** گفته می‌شود.

#### 🔹 ps

برای مشاهده‌ی Process ها می‌توان از دستور ```ps``` استفاده کرد. بسته به گزینه‌ هایی که استفاده می‌کنیم، اطلاعات مختلفی نمایش داده می‌ شود. یکی از ستون‌های مهم PID است.

<img width="100%" height="70" alt="image" src="https://github.com/user-attachments/assets/af83433c-463d-42e8-a734-9dd437ba8263" />

#### 🔹 STAT

در خروجی ps ممکن است ستونی به نام STAT ببینیم این ستون وضعیت Process را نشان می‌ دهد برای مثال:
- R → Running / Runnable
- S → Sleeping

<img width="100%" height="100" alt="image" src="https://github.com/user-attachments/assets/0f3cdae6-da5d-4fde-b546-1e03ed279cf1" />

#### 🔹 TIME

ستون TIME زمان CPU مصرف‌ شده توسط Process را نشان می‌دهد. این مقدار با زمانی که از شروع Process گذشته یکی نیست. ممکن است Process چند دقیقه قبل شروع شده باشد، اما فقط چند ثانیه واقعاً CPU دریافت کرده باشد. 

---

###  Signals

برای کنترل Process ها از Signal استفاده می‌ شود.Signal را می‌ توان به‌ صورت یک پیام یا Notification از Kernel به Process تصور کرد. برای ارسال Signal از```kill``` استفاده می‌کنیم. مثلاً: 
```kill PID```
به‌صورت پیش‌ فرض Signal مربوط به Termination را ارسال می‌ کند.

#### 🔹 SIGTERM
سیگنال ```SIGTERM``` به Process می‌گوید که خاتمه پیدا کند.Process فرصت دارد Signal را دریافت کند و در صورت امکان:
- منابع خود را آزاد کند.
- فایل‌ ها را ببندد.
- وضعیت خود را ذخیره کند.
- به شکل کنترل‌ شده خارج شود.

#### 🔹 SIGSTOP

سیگنال ```SIGSTOP```  کار Process را متوقف می‌ کند. Process از بین نمیرود فقط اجرای آن متوقف می‌ شود.

#### 🔹 SIGCONT

سیگنال ```SIGCONT``` برای ادامه دادن Process متوقف‌ شده استفاده می‌شود.

#### 🔹 SIGINT

وقتی در Terminal میزنیم Ctrl+C معمولاً سیگنال ```SIGINT``` ارسال می‌شود. این Signal به Process می‌گوید که کار فعلی را متوقف کند.

#### 🔹 SIGKILL ☠️

سیگنال ```SIGKILL``` متفاوت است.Process نمی‌تواند آن را Catch یا Ignore کند و Kernel مستقیماً Process را متوقف می‌کند. به همین دلیل:
```kill -9 PID```

باید معمولاً آخرین راه‌ حل باشد. قبل از آن بهتر است ابتدا از Signal هایی مانند SIGTERM استفاده کنیم تا Process فرصت Shutdown صحیح داشته باشد.  برای دیدن لیست سیگنال ها از دستور ```kill -l``` استفاده می کنیم .

<img width="100%" height="227" alt="image" src="https://github.com/user-attachments/assets/2a39cd39-a333-4acb-ac1f-bd0124dc4d77" />

---

###  Job Control in Shell

شل (Shell) قابلیت دیگری به نام **Job Control** دارد. با  Ctrl+Z می‌توان یک Job را موقتاً متوقف کرد. سپس ```fg``` آن را به Foreground برمی‌گرداند و ```bg``` باعث می‌شود Job در Background ادامه پیدا کند. همچنین می‌ توان یک Command را از ابتدا در Background اجرا کرد ``` & command```. این قابلیت برای کارهایی که زمان زیادی طول می‌کشند بسیار کاربردی است.

---

###  introduction to permissions

بخش Permission یکی از مهم‌ ترین بخش‌های امنیت Linux است. با ```ls -l``` ممکن است چیزی شبیه این ببینیم ```-rw-r--r–``` این رشته اطلاعات مربوط به Type و Permission فایل را نشان می‌ دهد.
<img width="100%" height="52" alt="image" src="https://github.com/user-attachments/assets/347c8e94-a146-4e3c-ba04-e707096ccccb" />

#### 🔹 Permissions structure

اولین Character نوع فایل را مشخص می‌ کند :  
- ' - ' → Regular file
- d → Directory
- l → Symbolic link

 بعد سه گروه Permission داریم:
- Owner
- Group Owner
- Other

هر گروه نیز سه Permission دارد:
- r → read
- w → write
- x → execute
بنابراین -rw-r--r-- را می‌توان چنین تقسیم کرد:
```
- rw- r-- r--
│ │   │   │
│ │   │   └── Other
│ │   └────── Group
│ └────────── Owner
└──────────── File type
```

#### 🔹 Setuid  

گاهی در Permission های فایل اجرایی به‌ جای x حرف s دیده می‌شود. این می‌ تواند نشان‌ دهنده‌ی setuid باشد. در این حالت برنامه هنگام اجرا می‌ تواند با Effective User ID مربوط به مالک فایل اجرا شود. یکی از مثال‌ های شناخته‌ شده ، passwd است که در سیستم‌ های سنتی برای تغییر اطلاعات مربوط به Password به دسترسی‌ های خاص نیاز دارد. البته جزئیات دقیق Permission و نحوه‌ ی پیاده‌ سازی این ابزارها به سیستم و Distribution بستگی دارد.

#### 🔹 Changing Permissions with chmod

برای تغییر Permission از ```chmod``` استفاده می‌شود. مثلاً:
```chmod g+r file```
یعنی Permission خواندن را برای Group اضافه کن یا:
```chmod go+r file```
یعنی Permission خواندن را برای Group و Other اضافه کن.

#### 🔹 Absolute / Octal Permissions

روش دیگری برای تنظیم Permission استفاده از اعداد Octal است.مثلاً:
```chmod 644 file```
برخی حالت‌ های بسیار رایج : 

- 644 فایل معمولی قابل‌خواندن برای دیگران
- 600 فایل خصوصی
- 755 معمولاً برای Executable ها و Directory های public
- 700 دایرکتوری یا فایل خصوصی برای owner

مفهوم عدد ها از ترکیب Permission ها می‌آید:
- r = 4
- w = 2
- x = 1
بنابراین:
- 7 = 4 + 2 + 1 = rwx
- 6 = 4 + 2     = rw-
- 5 = 4+ 1      = r-x
- 4 = 4         = r--
مثلاً ```644``` : 

- 6 → rw-
- 4 → r--
- 4 → r--

#### 🔹 Directory Permissions

دسترسی Permission روی Directory کمی متفاوت از فایل است. به‌ صورت ساده:

- مجوز r امکان مشاهده‌ ی Entry های Directory
- مجوز w امکان ایجاد/حذف/تغییر Entry ها، مشروط به سایر محدودیت‌ها
- مجوز x امکان Traverse کردن Directory

نکته‌ ی مهم این است که برای دسترسی به یک فایل داخل Directory معمولاً داشتن x روی Directory ضروری است. بنابراین ممکن است کاربری Permission خواندن محتویات یک Directory را نداشته باشد، اما بسته به Permission های دیگر بتواند به یک فایل مشخص در آن مسیر دسترسی پیدا کند.

#### 🔹 umask

برای تعیین Permission های پیش‌ فرض فایل‌ ها و Directory های جدید از ```umask``` استفاده می‌شود. مثلاً: ```umask 022``` یا ```umask 077``` . در واقع umask مشخص می‌کند چه Permission هایی نباید به‌ صورت پیش‌ فرض در زمان ایجاد فایل یا Directory اختصاص داده شوند.

---

###  Symbolic Link

یک Symbolic Link فایل کوچکی است که به مسیر فایل یا Directory دیگری اشاره می‌کند. مثلاً:

```ln -s target linkname```

در خروجی ```ls -l``` ممکن است چیزی شبیه این ببینیم:

```linkname -> target```

و Type آن با ```l``` نمایش داده می‌شود. Symbolic Link شبیه Shortcut در Windows است، اما مدل پیاده‌سازی آن دقیقاً یکسان نیست.





<img width="100%" height="181" alt="image" src="https://github.com/user-attachments/assets/e67d179f-40c0-4fba-bc7e-bae9cae9fbaa" />

#### 🔹 Broken Symbolic Link

اگر مقصد Symbolic Link وجود نداشته باشد، Link همچنان ممکن است وجود داشته باشد. اما برنامه‌ ای که بخواهد از طریق آن به مقصد دسترسی پیدا کند، معمولاً با خطا مواجه می‌شود. این وضعیت را می‌ توان Broken Symlink نامید.

#### 🔹 Hard Link

اگر هنگام اجرای ```ln``` گزینه‌ ی ```s-``` را استفاده نکنیم ```ln target linkname``` یک **Hard Link** ایجاد می‌ شود. Hard Link صرفاً یک اشاره‌ گر به نام دیگر نیست بلکه یک Directory Entry دیگر برای همان فایل و همان inode است. به همین دلیل مفهوم Hard Link با Symbolic Link کاملاً متفاوت است. به‌ صورت ساده: 

``` Symbolic Link : link → path → target ```

```
Hard Link :
name1 ─┐
       ├── same inode / same file data
name2 ─┘
```
این تفاوت یکی از مفاهیم مهم Filesystem در Unix/Linux است. 

---

###  Compressing and archiving with gzip and tar

یکی از تفاوت‌ های مهمی که باید بدانیم این است که Compression و Archiving یک چیز نیستند.
#### 🔹 gzip

ابزار gzip یک Compression است.مثلاً:

``` gzip file.txt```
فایل را به شکل فشرده تبدیل می‌کند. برای باز کردن آن : 

```gunzip file.txt.gz```

#### 🔹 tar

 ابزار tar برای ایجاد Archive استفاده می‌شود. مثلاً:

```tar cvf archive.tar file1 file2```

options :

- c → create
- v → verbose
- f → archive file
در این حالت چند فایل در یک Archive قرار می‌گیرند، اما Archive به‌ تنهایی الزاماً فشرده نیست.

#### 🔹 Extract Archive

برای extract یک فایل archive :

```tar xvf archive.tar```

- Option : x → extract

قبل از extract Archive بهتر است محتویات آن را بررسی کنیم:

```tar tvf archive.tar```

این کار می‌ تواند از ایجاد فایل‌ ها در مسیر نا مناسب جلو گیری کند. 

<img width="100%" height="131" alt="image" src="https://github.com/user-attachments/assets/e636d7a6-2be5-4240-8994-4d32fc9db738" />

#### 🔹 tar.gz

یک فایل ```archive.tar.gz``` در واقع دو مرحله دارد:
```
tar archive
     ↓
gzip compression
```
در گذشته می‌ توانستیم ابتدا gzip را باز کنیم و سپس Archive را استخراج کنیم. اما tar گزینه‌ی z را دارد:

```tar xzvf archive.tar.gz```

که باعث می‌شود gzip نیز در فرایند کار لحاظ شود.

#### 🔹 other tools

دو ابزار رایج دیگر```bzip2``` و ```xz``` هستند. هر کدام ویژگی‌ های متفاوتی دارند :

- سرعت
- میزان Compression
- مصرف منابع

---

###  Linux Directory Overview

فایل سیستم لینوکس از ```/``` شروع می‌شود. برخی از مهم‌ ترین Directory ها :

<img width="100%" height="54" alt="image" src="https://github.com/user-attachments/assets/fe597ea1-af1b-46fe-b79a-55cd19003b70" />


#### 🔹 /bin

محل قرارگیری برنامه‌ های اجرایی ضروری سیستم که در Distribution های جدید ممکن است bin/ به usr/bin/ متصل شده باشد.

#### 🔹 /dev

 محل قرارگیری File Interface برای Device ها مثلاً : 
- /dev/sda 
- /dev/null
- /dev/tty

#### 🔹 /etc

محل قرارگیری فایل‌ های Configuration سیستم هستش مثلاً:
- /etc/passwd
- /etc/fstab
- /etc/hosts

#### 🔹 /home

محل قرارگیری Home Directory کاربران معمولی مثلاً:
- /home/user

#### 🔹 /lib

محل قرارگیری Library های مورد نیاز برنامه‌ ها و بخش‌ های مختلف سیستم. در سیستم‌ های جدید ممکن است این مسیر نیز با ساختار usr/lib/ یکپارچه شده باشد.

#### 🔹 /proc

یک Filesystem مجازی برای ارائه‌ ی اطلاعات مربوط به Process ها و Kernel و وضعیت سیستم است. مثلاً:
- /proc/1
- /proc/cpuinfo
- /proc/meminfo

#### 🔹 /run

محل قرارگیری اطلاعات Runtime سیستم است برای مثال اطلاعاتی که فقط در زمان اجرای سیستم مورد نیاز هستند.

#### 🔹 /sys

یک Interface فایل‌ محور برای اطلاعات مربوط به Kernel و Device ها و Hardware. مثلاً:
- /sys/block

#### 🔹 /sbin

محل قرارگیری برنامه‌ های مرتبط با System Administration است و در بسیاری از Distribution های جدید، sbin/ نیز ممکن است با ساختار usr/ یکپارچه شده باشد.

#### 🔹 /tmp

محل قرارگیری برای فایل‌های موقت است و نباید فرض کنیم هر چیزی که داخل tmp/ قرار می‌دهیم برای همیشه باقی می‌ماند.

#### 🔹 /usr

یکی از بزرگ‌ ترین بخش‌ های User Space که شامل بخش زیادی از مانند : برنامه‌ ها و Library ها و  Documentation و داده‌ های مربوط به نرم‌افزارها است.

#### 🔹 /var

محل قرارگیری داده‌ هایی که در طول اجرای سیستم تغییر می‌کنند مثلاً:
- /var/log
- /var/cache

#### 🔹 /boot

محل قرارگیری فایل‌ های مرتبط با Boot، از جمله Kernel و فایل‌ های مورد نیاز Bootloader.


#### 🔹 /media

در بسیاری از سیستم‌ها برای Mount کردن Removable Media مانند USB استفاده می‌شود.

#### 🔹 /opt

محلی برای برخی نرم‌ افزارهای اضافی یا Third-party است. استفاده از این Directory به Distribution و روش نصب نرم‌افزار بستگی دارد.

---

###  kernel and boot

معمولاً Kernel لینوکس به شکل یک فایل قابل‌ بارگذاری روی سیستم وجود دارد. در بسیاری از سیستم‌ ها فایل‌ هایی با نام‌ هایی مانند ```vmlinuz/``` یا ```boot/vmlinuz/``` دیده می‌شوند. Bootloader در فرایند Boot Kernel را در حافظه قرار می‌ دهد و اجرای سیستم را به آن منتقل می‌کند.

#### 🔹 Kernel Modules

 لینوکس از Loadable Kernel Modules نیز پشتیبانی می‌کند.این Module ها می‌توانند در صورت نیاز Load شوند و قابلیت‌هایی مانند Driver ها را در اختیار Kernel قرار دهند. در بسیاری از سیستم‌ ها Module های Kernel در مسیری مانند ```lib/modules/``` قرار دارند.

---

###  Running commands with Superuser access

در Linux لازم نیست برای هر کار مدیریتی یک Shell کامل با دسترسی root باز کنیم. استفاده از یک Root Shell می‌تواند ریسک بیشتری داشته باشد، زیرا ممکن است یک Command اشتباه مستقیماً با بالاترین سطح دسترسی اجرا شود. به همین دلیل در بسیاری از سیستم‌ها از```sudo``` استفاده می‌شود مثلاً:

```sudo cat /etc/shadow```
در این حالت کاربر می‌تواند یک Command مشخص را با Permission های مورد نیاز اجرا کند.

#### 🔹 /etc/sudoers

تنظیمات مربوط به دسترسی sudo در فایل‌هایی مانند ```etc/sudoers/``` و فایل‌های Include شده از آن قرار می‌گیرند. می‌توان مشخص کرد: چه کاربری ، چه گروهی ، روی چه سیستم‌ هایی و چه Command هایی با چه شرایطی بتوانند با ```sudo``` اجرا شوند.

#### 🔹 Why visudo ?

برای ویرایش sudoers بهتر است از ```visudo``` استفاده شود. دلیل آن این است که visudo Syntax فایل را بررسی می‌کند. یک اشتباه کوچک در Configuration مربوط به sudo می‌تواند دسترسی مدیریتی را مختل کند. بنابراین روش امن‌تری برای تغییر Configuration است : 

``` Sudoers → visudo → syntax check → save ```

#### 🔹 Logging in to use sudo

استفاده از sudo معمولاً در سیستم Log می‌شود. محل دقیق Log به Distribution و Configuration سیستم بستگی دارد. در برخی سیستم‌ ها می‌توان اطلاعات مربوط به sudo را در Journal مشاهده کرد مثلاً:

``` journalctl SYSLOG_IDENTIFIER=sudo ```

در برخی سیستم‌ های دیگر ممکن است اطلاعات Authentication در فایل‌ هایی مانند ```/var/log/auth.log ``` ثبت شود.

---

### 🧠 Big picture

قدرت خط فرمان فقط به تعداد Command هایی که بلدیم محدود نمی‌شود. قدرت واقعی زمانی ظاهر می‌شود که بفهمیم این ابزارهای کوچک چگونه با یکدیگر ارتباط برقرار می‌کنند. برای مثال:

```grep "error" /var/log/*.log | sort | less```

در ظاهر فقط چند Command ساده است اما پشت آن مفاهیم زیادی وجود دارد:

```
Globbing → File arguments → grep → stdout → Pipe → sort → stdout → Pipe → less
```

همین ترکیب ابزارهای کوچک است که Shell را به یکی از قدرتمندترین محیط‌های کاری Unix/Linux تبدیل می‌کند.

---

### 🚀 From Commands to Understanding Linux

بعد از این فصل، دیگر فقط چند Command جدید یاد نگرفته‌ایم. با مفاهیمی آشنا شده‌ ایم که در فصل‌ های بعدی بارها به آن‌ ها برمی‌گردیم :

- Shell
- Process
- PID
- Signal
- File Descriptor
- stdin
- stdout
- stderr
- Pipe
- Redirection
- Globbing
- Environment Variable
- PATH
- Filesystem
- Permission
- Symbolic Link
- Hard Link
- Kernel Interface
- sudo
- /proc
- /sys

این مفاهیم پایه‌ ای هستند که بعد ها هنگام بررسی Process ها، Network، Storage، Boot، Shell Programming و System Administration دوباره با آن‌ها رو به‌ رو خواهیم شد.
در واقع ، بسیاری از چیزهایی که در Linux «فقط کار می‌کنند»، نتیجه‌ی مستقیم تصمیماتی هستند که Kernel و Shell در پشت صحنه می‌گیرند و این دقیقاً همان چیزی است که How Linux Works تلاش می‌کند آموزش دهد : **به‌ جای اینکه فقط بدانیم «چه دستوری بزنیم» ، بفهمیم «چرا این دستور این‌ طور کار می‌کند»**

---

### 💡 Tips

نکته‌ی اصلی فصل دوم این است که Command های پایه‌ ی Unix/Linux را نباید به‌ عنوان مجموعه‌ ای از دستورات جداگانه و برای حفظ کردن ببینیم.پشت این دستورات، مجموعه‌ ای از مفاهیم به یکدیگر متصل هستند:

```
Shell
│
├── Process
│
├── System Calls
│
├── File Descriptors
│ ├── stdin
│ ├── stdout
│ └── stderr
│
├── Pipes
│
├── Redirection
│
├── Globbing
│
├── Environment Variables
  └── PATH
       │
       ▼
   Programs
       │
       ▼
    Kernel
       │
       ▼
Filesystem / Hardware

```

وقتی این ارتباط‌ ها را بفهمیم، رفتار بسیاری از Command ها دیگر عجیب نیست : 

- چرا cat بدون Argument منتظر می‌ماند؟ چون stdin دارد.
- چرا خروجی grep را می‌ توان به less داد؟ چون stdout یک جریان است و Shell می‌تواند آن را به stdin Process دیگری متصل کند.
- چرا cd باید Built-in باشد؟ چون باید Working Directory خود Shell را تغییر دهد.
- چرا mv در یک Filesystem می‌تواند تقریباً آنی باشد؟ چون در بسیاری از موارد لازم نیست داده‌ی فایل دوباره کپی شود.
- چرا log.* را grep یا ls مستقیماً تفسیر نمی‌کنند؟ چون Shell قبل از اجرای برنامه Globbing را انجام می‌دهد.
- چرا PATH اهمیت دارد؟ چون Shell از آن برای پیدا کردن Executable ها استفاده می‌کند.
- چرا stderr با stdout فرق دارد؟ چون Process ها چند File Descriptor استاندارد جداگانه دارند و می‌توان هرکدام را مستقل Redirect کرد.
- چرا rm مثل Delete در یک محیط گرافیکی نیست؟ چون در Unix/Linux مفهوم حذف فایل با حذف Directory Entry و مفهوم unlink ارتباط دارد.

فصل دوم یک پل مهم بین مفاهیم معماری فصل اول و کار واقعی با Linux است. در فصل اول یاد گرفتیم :
``` 
Hardware
   ↓
Kernel
   ↓
User Space
```
در فصل دوم دیدیم که Command هایی که هر روز در User Space اجرا می‌کنیم ، در نهایت با همین معماری ارتباط دارند. از ls و cat گرفته تا grep، ps، chmod و sudo، همه بخشی از یک سیستم بزرگ‌ تر هستند. هر چه این ارتباط‌ ها را بهتر بفهمیم، Linux برایمان کمتر شبیه مجموعه‌ ای از Command های حفظی و بیشتر شبیه یک سیستم منطقی و قابل پیش‌بینی خواهد شد و این پایه‌ ای است که فصل‌ های بعدی کتاب روی آن ساخته می‌شوند.
