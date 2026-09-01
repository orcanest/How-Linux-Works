## دستورات پایه‌ ی لینوکس چطور واقعاً کار می‌کنند؟ 
### 🐧 فصل دوم کتاب How Linux Works

فصل اول کتاب معماری کلی یک سیستم لینوکس را معرفی کرد:  ``` Hardware → Kernel → User Space ```

در فصل دوم یک قدم جلوتر می‌رویم و سراغ ابزارهایی میرویم که تقریباً هر روز هنگام کار با لینوکس از آن‌ها استفاده می‌کنیم. اما هدف این فصل فقط یاد گرفتن چند دستور نیست. هدف این است که بفهمیم **این دستورات در پشت صحنه چگونه با Shell، Processها، Kernel و Filesystem ارتباط برقرار می‌کنند**. وقتی این ارتباط را درک کنیم، بسیاری از رفتارهایی که در ابتدا عجیب به نظر می‌رسند، کاملاً منطقی می‌شوند.

---
📚 Table of Contents
- [What is Shell and how does it execute commands?](#️-What-is-Shell-and-how-does-it-execute-commands?)
- [Stderr, stdin & stdout]( #️-stdin-and-stdout-and-stderr)
- [Pipe]( #️-pipe) 
- [File Commands]()
- [Navigating Directories]()
- [Why should cd be built-in?]()
- [mkdir and rmdir]()


---

### ⚙️ **What is Shell and how does it execute commands?**
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

### ⚙️ stdin and stdout and stderr 

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

### ⚙️ Pipe

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

---

### ⚙️ File Commands

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

### ⚙️ Navigating Directories

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

### ⚙️ Why should cd be built-in?

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

### ⚙️ mkdir and rmdir

برای ایجاد Directory از دستور ```mkdir mydir``` و برای حذف یک Directory خالی از دستور ```rmdir mydir``` استفاده می‌کنیم. 






