## دستورات پایه‌ ی لینوکس چطور واقعاً کار می‌کنند؟ 
### فصل دوم کتاب How Linux Works

فصل اول کتاب معماری کلی یک سیستم لینوکس را معرفی کرد:  ``` Hardware → Kernel → User Space ```

در فصل دوم یک قدم جلوتر می‌رویم و سراغ ابزارهایی میرویم که تقریباً هر روز هنگام کار با لینوکس از آن‌ها استفاده می‌کنیم. اما هدف این فصل فقط یاد گرفتن چند دستور نیست. هدف این است که بفهمیم **این دستورات در پشت صحنه چگونه با Shell، Processها، Kernel و Filesystem ارتباط برقرار می‌کنند**. وقتی این ارتباط را درک کنیم، بسیاری از رفتارهایی که در ابتدا عجیب به نظر می‌رسند، کاملاً منطقی می‌شوند.

---
📚 Table of Contents
- [What is Shell and how does it execute commands?](#️-What-is-Shell-and-how-does-it-execute-commands?)
- [stderr, stdin & stdout]( #️-stdin-and-stdout-and-stderr)
- [pipe]( #️-pipe) 
-
-
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

### 🔹 stdin

ورودی stdin یا Standard Input ، ورودی استاندارد Process است. در یک Shell معمولی، این ورودی معمولاً به Terminal متصل است. برای مثال اگر بنویسیم : ```cat``` و هیچ فایلی به آن ندهیم، cat منتظر دریافت ورودی می‌ماند. هر چیزی که تایپ کنیم، cat همان را روی خروجی چاپ می‌کند. 

<img width="100%" height="130" alt="image" src="https://github.com/user-attachments/assets/c9d69edd-b0cb-4b9e-bdac-4ee25e8fbea3" />

هر چیزی که تایپ کنیم ، cat همان را روی خروجی چاپ می‌کند. برای پایان دادن به ورودی می‌توانیم ```Ctrl+D``` را بزنیم. ```Ctrl+D``` در اینجا در واقع یک کاراکتر معمولی نیست. در Terminal باعث می‌شود وضعیت EOF به Process منتقل شود، یعنی داده‌ ی بیشتری برای خواندن وجود ندارد.

### 🔹 stdout

خروجی stdout یا Standard Output ، خروجی معمولی Process است. مثلاً ```echo hello``` خروجی hello را روی stdout می‌نویسد.

<img width="100%" height="60" alt="image" src="https://github.com/user-attachments/assets/5c986079-4a8b-4d60-9477-1103a40f45b5" />


### 🔹 stderr

جریان stderr یا Standard Error d یک جریان جداگانه‌ ای برای پیام‌ های خطا است. برای مثال ```ls /does-not-exist``` پیام خطا از stderr ارسال می‌شود. این جداسازی بسیار مهم است، زیرا می‌توانیم خروجی عادی و خطاها را به مقصدهای متفاوتی بفرستیم.


<img width="100%" height="60" alt="image" src="https://github.com/user-attachments/assets/90b81fd9-2470-4574-8d0d-59fc78ad6e1d" />

---

### ⚙️ Pipe
