## ابزار های توسعه در Linux
### 🐧 فصل پانزدهم کتاب How Linux Works

فصل پانزدهم کتاب How Linux Works وارد دنیای ابزارهایی می‌ شود که پشت صحنه‌ ی برنامه‌ های موجود روی Linux را می‌ سازند. در فصل‌ های قبلی بیشتر با خود سیستم ، kernel و process ها ، storage ، networking و desktop آشنا شدیم. اینجا یک قدم به عقب برمی‌ گردیم و می‌ بینیم برنامه‌ هایی که روی این سیستم اجرا می‌ شوند اصلاً چگونه ساخته شده‌ اند. موضوع اصلی فصل فقط «کامپایل کردن یک فایل C» نیست. هدف این است که مسیر یک برنامه را از source code تا executable دنبال کنیم و بفهمیم در هر مرحله چه اتفاقی می‌ افتد :

```
Source Code
    |
    v
Preprocessor
    |
    v
Compiler
    |
    v
Object Files
    |
    v
Linker
    |
    +------------------+
    |                  |
    v                  v
Static Libraries    Shared Libraries
    |                  |
    v                  v
Executable        Runtime Loader
                       |
                       v
                   Process
```

در این مسیر مفاهیمی مانند compiler ، linker ، object file ، static library ، shared library ، dynamic linker/loader ، header file ، preprocessor و make اهمیت پیدا می‌ کنند. فصل همچنین نگاهی به Lex و Yacc، زبان‌ های scripting ، Python ، Perl ، زبان‌ های دیگری مانند Ruby و Tcl ، و در نهایت Java دارد. خود کتاب تأکید می‌ کند که شناخت ابزارهای development فقط برای programmer ها مهم نیست بلکه ابزارهای توسعه نقش قابل‌ توجهی در مدیریت و شناخت سیستم‌های Unix/Linux دارند. از میان موضوعات این فصل ، بحث shared library ها اهمیت ویژه‌ ای دارد ، چون برای فهمیدن اجرای بسیاری از برنامه‌ های Linux باید بدانیم dependency های آن‌ ها در زمان اجرا چگونه پیدا و بارگذاری می‌ شوند.


---
📚 Table of Contents

- [The C Compiler](#the-c-compiler)
- [make](#make)
- [Lex and Yacc](#lex-and-yacc)
- [Scripting Languages](#scripting-languages)
- [Java](#java)
- [Looking Forward: Compiling Packages](#looking-forward-compiling-packages)
- [Summary](#summary)

---

### The C Compiler

زبان C یکی از مهم‌ ترین زبان‌ های موجود در دنیای Unix و Linux است و تعداد زیادی از utility های سیستم و بسیاری از  application ها با C یا C++ نوشته شده‌ اند. برای اجرای یک برنامه‌ ی C هم source code باید به شکلی تبدیل شود که processor بتواند آن را اجرا کند. source code همان چیزی است که programmer می‌ نویسد و می‌ تواند از یک یا چند فایل تشکیل شده باشد. در مقابل ، زبان‌ های scripting که در بخش‌ های بعدی فصل بررسی می‌ شوند ، معمولاً نیاز ندارند که source code آن‌ ها به یک executable native تبدیل شود و معمولاً توسط interpreter اجرا می‌ شوند. روی بیشتر Unix system ها compiler معمول C ، کامپایلر GNU یعنی gcc است که اغلب با نام سنتی cc نیز دیده می‌ شود. clang از پروژه‌ ی LLVM نیز compiler مهم دیگری است. فایل source مربوط به C معمولاً پسوند c. دارد. یک برنامه‌ ی ساده می‌ تواند در فایلی مانند hello.c قرار داشته باشد :

```c
#include <stdio.h>

int main() {
    printf("Hello, World.\n");
}
```

برای compile کردن آن کافی است ```cc hello.c``` را اجرا کنید. در این حالت compiler به‌ صورت پیش‌ فرض executable با نام ```a.out```می‌ سازد. معمولاً بهتر است نام executable را خودمان مشخص کنیم مثلا ```cc -o hello hello.c``` . حالا executable ایجاد شده hello نام دارد و مانند دیگر executable های Linux قابل اجراست به این صورت ```hello/.``` برای برنامه‌ های کوچک ، همین command ممکن است تقریباً همه‌ ی چیزی باشد که لازم دارید اما وقتی تعداد source file ها یا dependency ها زیاد شود ، روند ساخت برنامه پیچیده‌ تر می‌ شود. کتاب همچنین اشاره می‌ کند که ابزارهای لازم برای compile کردن C لزوماً روی همه‌ ی distribution ها به‌ صورت پیش‌ فرض نصب نیستند. در Debian/Ubuntu می‌ توان از package هایی مانند build-essential و در Fedora/CentOS از مجموعه‌ ی Development Tools استفاده کرد.

#### 🔹 Compiling Multiple Source Files

قرار دادن کل یک برنامه‌ ی واقعی در یک فایل C معمولاً مناسب نیست. هرچه برنامه بزرگ‌ تر شود ، مدیریت یک فایل عظیم دشوارتر می‌ شود. به همین دلیل programmer ها source code را به چند بخش تقسیم می‌ کنند و هر بخش را در فایل جداگانه قرار می‌ دهند مثلاً تصور کنید دو فایل داریم main.c و aux.c که main.c نقطه‌ ی شروع برنامه را دارد : 

```c
void hello_call();

int main() {
    hello_call();
}
```

و aux.c پیاده‌ سازی تابع را نگه می‌ دارد :

```c
#include <stdio.h>

void hello_call() {
    printf("Hello, World.\n");
}
```

وقتی source file های جداگانه داریم ، معمولاً هرکدام را ابتدا به یک object file تبدیل می‌ کنیم. برای این کار از option -c استفاده می‌ شود :

```
cc -c main.c
cc -c aux.c
```

خروجی :

```
main.o
aux.o
```

خواهد بود. Object file حاوی binary object code است ، اما هنوز executable کامل نیست. این object file تقریباً به فرم قابل‌ استفاده برای processor نزدیک شده ، ولی هنوز چند مسئله باقی مانده است :

- سیستم‌ عامل هنوز نمی‌ داند چگونه آن object file را مستقیماً به‌ عنوان یک process اجرا کند.
- ممکن است object file به symbol ها یا function های موجود در object file های دیگر وابسته باشد.
- ممکن است به library های سیستم نیاز داشته باشد.
- اطلاعات لازم برای ساخت executable نهایی هنوز باید resolve شود.

بنابراین مرحله‌ ی بعدی linking است. در Unix ابزار اصلی linker یعنی ```ld``` است اما programmer ها معمولاً ld را مستقیماً اجرا نمی‌کنند و compiler این کار را برای آن‌ها انجام می‌ دهد. برای ساخت executable از دو object file بالا :

```cc -o myprog main.o aux.o```

در اینجا compiler در مرحله‌ ی نهایی linker را فراخوانی می‌ کند و main.o و aux.o را به executable نهایی myprog تبدیل می‌ کند. هرچه تعداد source file ها بیشتر شود ، مدیریت دستی command های compile و link دشوارتر می‌ شود. همین مسئله دلیل اصلی اهمیت ابزارهایی مانند make است که در بخش بعدی بررسی می‌ شود.

#### 🔹 Linking with Libraries

یک executable مفید معمولاً فقط از source code خود برنامه تشکیل نشده است بسیاری از function هایی که برنامه به آن‌ ها نیاز دارد ، از قبل در library ها پیاده‌ سازی شده‌ اند. یک C library را می‌ توان مجموعه‌ ای از component های از قبل کامپایل‌ شده در نظر گرفت. این component ها معمولاً شامل object code و header file هایی هستند که interface آن‌ ها را برای compiler توصیف می‌ کنند. برای مثال ، library های استاندارد یا سیستم می‌ توانند function های مختلفی برای  کار با فایل و input/output و ریاضیات و terminal و networking و پردازش متن و و بسیاری عملیات دیگر فراهم کنند. Library ها عمدتاً هنگام link time وارد ماجرا می‌ شوند و به این فرآیند **linking against a library** می‌گوییم. یکی از error های رایج linker ، خطای ```undefined reference``` است. مثلاً اگر برنامه‌ ای از تابعی مانند()initscr استفاده کند ولی library مناسب هنگام linking در اختیار linker قرار نگرفته باشد ، ممکن است چیزی شبیه این ببینید :

```undefined reference to 'initscr'```

معنای این پیام این است که linker در object file ها symbol مورد نیاز را پیدا نکرده و در نتیجه نمی‌ تواند executable نهایی را کامل کند البته undefined reference همیشه به معنی missing library نیست ممکن است object file دیگری که function را پیاده‌ سازی می‌کند در command وجود نداشته باشد یا ترتیب یا dependency های link درست نباشد یا symbol مورد نظر در library اشتباه جست‌ وجو شده باشد اما اگر function متعلق به یک library باشد ، باید library مربوطه را پیدا کنیم.

#### 🔹 The -l Option

برای مشخص کردن library از```l-``` استفاده می‌ شود مثلاً اگر library موردنظر ```libcurses.a``` باشد ، نامی که به l- می‌ دهیم ```curses``` خواهد بود به این صورت ```cc -o badobject badobject.o -lcurses``` و قاعده‌ ی مهم این است که برای یک library مانند ```libsomething.a``` معمولاً بخش lib ابتدایی و suffix فایل را در l- نمی‌ نویسیم :

```
libsomething.a
     |
     +----> -lsomething
```

#### 🔹 The -L Option

همیشه Library ها در directory های استاندارد نیستند. در لینوکس library ها معمولاً در مسیرهایی مانند lib/ و usr/lib/ و زیرشاخه‌ های architecture-specific آن‌ ها دیده می‌ شوند. اما یک برنامه ممکن است به library در مسیری غیر معمول نیاز داشته باشد. برای معرفی directory اضافی به linker از ```L- ``` استفاده می‌ شود مثلاً :

```
cc -o badobject badobject.o \
    -lcurses \
    -L/usr/junk/lib \
    -lcrud
```

در این command هم ```L/usr/junk/lib-``` به linker می‌ گوید هنگام جست‌ وجوی library ها directory مشخص‌ شده را نیز بررسی کند.

#### 🔹 Finding Functions Inside Libraries

اگر بخواهید بفهمید یک library شامل یک function خاص هست یا نه ، یکی از ابزارهای مفید nm است مثلاً ```nm --defined-only libcurses.a ``` ، این command symbol های تعریف‌ شده در library را نمایش می‌ دهد. اگر output زیاد باشد می‌ توانید آن را به less بدهید مثلا ```nm --defined-only libcurses.a | less```. برای پیدا کردن خود library نیز ممکن است لازم باشد از ابزارهایی مثل locate استفاده کنید. در بعضی distribution ها library ها در directory های architecture-specific مانند ```/usr/lib/x86_64-linux-gnu/``` قرار گرفته‌ اند ، بنابراین نباید فرض کنید همه‌ ی library ها دقیقاً مستقیماً در usr/lib/ قرار دارند.

#### 🔹 The C Standard Library

یکی از مهم‌ ترین library های موجود در سیستم ، C standard library است. نام پایه‌ ی نسخه‌ ی static آن معمولاً ```libc.a``` است. این library شامل component های اساسی مورد استفاده در برنامه‌ های C است. در حالت معمول compiler آن را به برنامه اضافه می‌ کند ، مگر اینکه صراحتاً خلاف آن درخواست شده باشد. در عمل بسیاری از برنامه‌ های Linux از نسخه‌ ی shared این library استفاده می‌ کنند و همین موضوع ما را به بحث shared library ها می‌ رساند.

#### 🔹 Working with Shared Libraries

دو نوع اصلی library که باید از هم تشخیص دهید static library و shared library هستند. Static library معمولاً پسوند ```a.``` دارد و Shared library معمولاً با ```so.``` شناخته می‌ شود.

#### 🔹 Static Libraries

فرض کنید library مانند ```libcurses.a``` داریم. وقتی یک executable را با static library link می‌ کنیم ، linker code موردنیاز آن library را در executable نهایی قرار می‌ دهد. یعنی به‌ صورت مفهومی :

```
libcurses.a
     |
     | copy required code
     v
+--------------------+
|      myprog        |
|                    |
|  program code      |
|  library code      |
+--------------------+
```

در نتیجه executable برای اجرای آن بخش از code دیگر به فایل اصلی static library وابسته نیست. این ویژگی چند مزیت دارد. مثلاً اگر فایل a. بعداً تغییر کند ، executable قبلی به‌ صورت خودکار تحت تأثیر نسخه‌ی جدید قرار نمی‌ گیرد اما static linking هزینه‌هایی هم دارد. اگر تعداد زیادی executable از یک library استفاده کنند ، هر executable copy خودش از code library را در اختیار خواهد داشت بنابراین اندازه‌ ی executable ها بزرگ‌ تر و مصرف فضای disk افزایش می‌ یابد همچنین اصلاح یک library قدیمی نیازمند rebuild کردن executable های وابسته است ، اگر library دچار مشکل امنیتی یا عملکردی شود ، برنامه‌ های قبلی خودکار از نسخه‌ ی اصلاح‌ شده استفاده نمی‌ کنند. اگر library تغییر کند ، executable قبلی همچنان code را دارد که در زمان link داخل خودش قرار گرفته است. برای استفاده از code جدید معمولاً باید دوباره link یا rebuild شود.

#### 🔹 Shared Libraries

برای حل بخشی از این مشکلات Shared library طراحی شده است. هنگام linking یک shared library ، linker معمولاً code کامل آن library را داخل executable قرار نمی‌ دهد. در عوض executable اطلاعاتی درباره‌ ی dependency خود روی library نگه می‌ دارد. مثلاً به‌ جای اینکه کل code یک library داخل executable قرار گیرد ، executable ممکن است بداند که به library با نام مشخص نیاز دارد. بعد هنگام اجرای برنامه ، dynamic linker/loader library مورد نیاز را پیدا می‌ کند و آن را در address space process بارگذاری می‌ کند. مدل مفهومی :

```
            +----------------+
             |   Executable   |
             +----------------+
                     |
                     | needs
                     v
             +----------------+
             | Shared Library |
             +----------------+
                     |
                     v
              Runtime Loader
```

یکی از مزیت‌ های مهم این مدل این است که چند process می‌ توانند code یک shared library را در memory به‌ شکل مشترک استفاده کنند. مثلاً اگر تعداد زیادی process از یک library مشترک استفاده کنند ، لازم نیست برای هر process یک copy مستقل از code library در memory نگه داشته شود. مزیت دیگر این است که می‌ توان library را update کرد و اگر compatibility حفظ شود ، برنامه‌ های موجود می‌ توانند نسخه‌ ی جدید را استفاده کنند. این ویژگی در update های معمول Linux بسیار مهم است. package manager ممکن است shared library ها را update کند و پس از reboot ، process های جدید از نسخه‌ ی جدید library استفاده کنند. البته shared library ها کاملاً بدون مشکل نیستند. مدیریت dependency ها پیچیده‌ تر می‌ شود و ناسازگاری نسخه‌ ها می‌ تواند باعث شکست برنامه شود. برای استفاده‌ ی درست از shared library ها باید چهار موضوع را بلد باشید :


1. چطور dependency های shared library یک executable را ببینیم.
2. چگونه executable یک shared library را پیدا می‌ کند.
3. چگونه یک برنامه را به shared library خاصی link کنیم.
4. چگونه از مشکلات رایج shared library ها جلوگیری کنیم.

#### 🔹 How to List Shared Library Dependencies

معمولاً Shared library ها در مسیرهایی مانند lib/ و usr/lib/ پیدا می‌ شوند البته ممکن است directory های اضافی دیگری نیز وجود داشته باشند. یک shared library معمولاً نامی شامل so. دارد مثلاً libc.so.6 یا libc-2.15.so . برای دیدن dependency های shared یک executable می‌ توان از```ldd /bin/bash``` استفاده کرد و خروجی ممکن است چیزی شبیه این باشد :

```
linux-vdso.so.1 (...)
libc.so.6 => /lib/.../libc.so.6 (...)
libpthread.so.0 => /lib/.../libpthread.so.0 (...)
/lib64/ld-linux-x86-64.so.2 (...)
```

در خروجی ldd معمولاً نام library به مسیر library را مشاهده می‌ کنید. سمت چپ dependency مورد نیاز executable است و سمت راست در صورت resolve شدن محلی است که loader library را پیدا کرده است. همچنین خط مربوط به ```ld-linux``` یا loader مشابه آن نشان می‌ دهد dynamic linker/loader واقعی کجاست.

#### 🔹 The Dynamic Linker/Loader

برنامه‌ ای مانند ld.so مسئول پیدا کردن و بارگذاری shared library ها در زمان اجرای program است. Executable معمولاً همه‌ ی path های واقعی library ها را به‌ صورت hardcoded در خود نگه نمی‌ دارد. آنچه بیشتر می‌ داند ، نام library یا اطلاعات runtime search path است بعد dynamic linker این اطلاعات را با وضعیت سیستم ترکیب می‌ کند و library واقعی را پیدا می‌ کند به همین دلیل این دو موضوع را باید از هم جدا کنید این تفاوت یکی از مهم‌ ترین مفاهیم فصل است :

```
Link Time
    |
    +--> executable records library dependency

Run Time
    |
    +--> dynamic linker finds actual library
```


#### 🔹 How ld.so Finds Shared Libraries

یکی از مشکلات رایج shared library ها این است که executable در زمان اجرا نمی‌ تواند library مورد نیازش را پیدا کند. Dynamic linker برای این کار چند منبع را بررسی می‌ کند.

#### 🔹 Runtime Search Path

اگر executable یک runtime library search path از پیش تعریف‌ شده داشته باشد ، loader می‌ تواند ابتدا از آن استفاده کند در اصطلاح تاریخی کتاب به این مورد rpath گفته می‌ شود.

#### 🔹 ld.so.cache

یکی دیگر از منابع ، cache مربوط به dynamic linker است در ```etc/ld.so.cache/``` این cache بر اساس directory هایی ساخته می‌شود که در ```etc/ld.so.conf/``` و فایل‌ های include شده توسط آن قرار دارند. ld.so.conf ممکن است فایل‌ های دیگری را هم از directory هایی مانند ```/etc/ld.so.conf.d/``` هم include کند. هر خط معمولاً یک directory را مشخص می‌ کند که باید برای پیدا کردن shared library ها در cache لحاظ شود. مثلاً ممکن است چیزی شبیه این وجود داشته باشد:

```
/lib/i686-linux-gnu
/usr/lib/i686-linux-gnu
```

مسیرهای استاندارد lib/ و usr/lib/ در مدل کتاب implicit هستند و لازم نیست الزاماً در ld.so.conf تکرار شوند. اگر configuration مربوط به ld.so.conf یا shared library directory ها تغییر کرد ، cache نیز باید دوباره ساخته شود. برای این کار ```ldconfig -v``` استفاده می‌شود. option -v اطلاعات بیشتری درباره‌ ی library هایی که ldconfig به cache اضافه یا تغییر داده نمایش می‌ دهد.

##### 🔹 Avoid Overusing /etc/ld.so.conf

کتاب هشدار می‌ دهد که نباید عادت کنید هر directory عجیب و غیرمعمولی را به /etc/ld.so.conf اضافه کنید. اگر هر library با یک directory اختصاصی داشته باشیم و همه‌ ی آن‌ ها را وارد cache کنیم ، ممکن است سیستم به‌ مرور پیچیده شود و organization ضعیفی پیدا کند با library های همنام دچار conflict شود. برای software که نیاز به یک library غیرمعمول دارد ، بهتر است dependency را در سطح همان executable مدیریت کنیم تا اینکه search path کل سیستم را تغییر دهیم.

#### 🔹 How to Link Programs Against Shared Libraries

فرض کنید یک shared library به نام ```libweird.so.1``` در مسیر ```/opt/obscure/lib``` قرار دارد و myprog به آن نیاز دارد. به‌ جای وارد کردن این directory در ld.so.conf ، می‌ توان هنگام build کردن executable یک runtime path برای آن تعریف کرد مثلاً :

```
cc -o myprog myprog.o \
    -Wl,-rpath=/opt/obscure/lib \
    -L/opt/obscure/lib \
    -lweird
```

در این command دو option متفاوت داریم ```L/opt/obscure/lib-``` که برای مرحله‌ ی link time است یعنی linker باید هنگام ساخت executable بتواند library را پیدا کند. در مقابل ```Wl,-rpath=/opt/obscure/lib-``` ، مسیر runtime را داخل executable قرار می‌ دهد تا dynamic linker هنگام اجرای program بداند باید این directory را هم برای shared library بررسی کند و این دو کار یکی نیستند.

```
-L
 |
 +--> Where to find the library while linking

-rpath
 |
 +--> Where to look for the library while running
```

کتاب تأکید می‌ کند که حتی اگر از Wl,-rpath- استفاده کنید همچنان ممکن است L- لازم باشد ، چون linker باید خود library را در زمان build پیدا کند. اگر بخواهید runtime library search path یک binary موجود را تغییر دهید ، ابزاری مانند ```patchelf``` وجود دارد ولی کتاب ترجیح می‌ دهد این کار تا حد امکان در زمان compile/link انجام شود. در این بخش کتاب همچنین به ELF (Executable and Linkable Format) اشاره می‌ کند که format استاندارد executable ها و library های Linux است.

#### 🔹 How to Avoid Problems with Shared Libraries

معمولاً Shared library ها بسیار قدرتمند و انعطاف‌ پذیرند ، اما استفاده‌ ی نادرست از آن‌ ها می‌ تواند سیستم را بسیار پیچیده کند. کتاب سه دسته‌ ی مهم از مشکل را مطرح می‌ کند مانند Missing Libraries و Terrible Performance و Mismatched Libraries و در میان این مشکلات ، به‌ شکل ویژه روی ```LD_LIBRARY_PATH``` تمرکز می‌ کند.

#### 🔹 LD_LIBRARY_PATH

می دانیم که LD_LIBRARY_PATH یک environment variable است که به dynamic linker می‌گوید directory های مشخصی را هنگام جست‌ وجوی shared library ها بررسی کند مثلاً برای ```export LD_LIBRARY_PATH=/opt/myapp/lib``` این روش می‌ تواند برای اجرای یک برنامه‌ ی خاص که library هایش در location غیرمعمول قرار دارند مفید باشد. اما مشکل اینجاست که LD_LIBRARY_PATH فقط روی یک برنامه‌ ی خاص تأثیر نمی‌گذارد بلکه اگر آن را در محیط shell خود set کنید ، بسیاری از برنامه‌ هایی که از همان shell اجرا می‌ کنید آن را به ارث خواهند برد. این مسئله می‌ تواند باعث شود یک program به library اشتباه متصل شود. مثلاً ممکن است یک library با نام یکسان در```usr/lib/``` و ```opt/something/lib/``` وجود داشته باشد. با تنظیم global LD_LIBRARY_PATH ممکن است program که انتظار دارید از library سیستم استفاده کند ، ناگهان نسخه‌ ی دیگری را دریافت کند. این موضوع می‌ تواند باعث شود dependency ها را خراب و compatibility را از بین ببرد و performance را کاهش دهد و باعث رفتارهای عجیب و دشوار برای debugging شود به همین دلیل کتاب صراحتاً توصیه می‌ کند LD_LIBRARY_PATH را در shell startup file ها قرار ندهید همچنین هنگام compile کردن software نباید بی‌ دلیل از آن استفاده کنید.

#### 🔹 Using LD_LIBRARY_PATH Safely

اگر واقعاً مجبور هستید یک برنامه‌ ی قدیمی یا خاص را با library های غیرمعمول اجرا کنید و امکان rebuild یا patch کردن binary را ندارید ، بهتر است این environment variable فقط برای همان program تنظیم شود. روش پیشنهادی کتاب استفاده از wrapper script است. مثلاً اگر executable اصلی ```opt/crummy/bin/crummy.bin/``` باشد و library های مورد نیاز آن در```opt/crummy/lib/``` قرار داشته باشند ، می‌ توان wrapper مانند این ایجاد کرد :

```sh
#!/bin/sh

LD_LIBRARY_PATH=/opt/crummy/lib
export LD_LIBRARY_PATH

exec /opt/crummy/bin/crummy.bin "$@"
```

به این ترتیب LD_LIBRARY_PATH فقط در environment همان program قرار می‌ گیرد و کل shell session را آلوده نمی‌ کند. این روش نسبت به قرار دادن یک path غیرمعمول در startup file بسیار کنترل‌شده‌ تر است.

#### 🔹 Library Version Mismatches

یک مشکل دیگر زمانی اتفاق می‌ افتد که API یا ABI یک library بین version های مختلف تغییر کند و program نصب‌ شده با نسخه‌ ی جدید سازگار نباشد برای جلوگیری از این وضعیت ، کتاب دو رویکرد اصلی را مطرح می‌کند :

- استفاده از یک روش مشخص و قابل‌ کنترل برای نصب shared library ها ، از جمله runtime search path مناسب.
- استفاده از نسخه‌ ی static برای library های obscure ، در شرایطی که dependency را نمی‌ توان به‌ شکل مطمئن مدیریت کرد.

در عمل باید دقت کرد که «قابل‌ تعویض بودن» یک shared library به compatibility مناسب interface و ABI آن وابسته است صرفاً اینکه نام فایل library یکسان باشد تضمین نمی‌ کند executable با نسخه‌ ی جدید سالم کار کند.

#### 🔹 Working with Header (Include) Files and Directories

در source هم Header file ها فایل‌ های اضافی هستند که معمولاً declaration مربوط به type ها و function ها را در اختیار compiler قرار می‌ دهند یک نمونه‌ ی معروف ```stdio.h```است. وقتی برنامه‌ ای چیزی مانند : 

```c
#include <stdio.h>
```

می‌نویسد ، اطلاعات لازم درباره‌ ی function هایی مانند()printf از طریق header در اختیار compiler قرار می‌ گیرد. Header file ها بخش مهمی از build process هستند و تعداد زیادی از compile error ها به این موضوع مربوط می‌ شوند که compiler نتوانسته header یا library مور دنیاز را پیدا کند. گاهی مشکل از خود source code است. مثلاً programmer فراموش کرده directive مناسب include# را اضافه کند. گاهی نیز header وجود دارد ولی compiler directory صحیح را برای پیدا کردن آن جست‌ وجو نمی‌ کند.

#### 🔹 Include File Problems

فرض کنید compiler با این خطا رو به‌ رو شود ```fatal error: notfound.h: No such file or directory``` و source code شامل : 

```c
#include <notfound.h>
```

باشد این پیام می‌گوید compiler نتوانسته header مورد نظر را در include path پیدا کند. در Unix ، یکی از include directory های استاندارد ```usr/include/``` است. اما پروژه‌ ها می‌ توانند header های خودشان را در directory های دیگری قرار دهند. مثلاً اگر header در این مسیر باشد ```usr/junk/include/``` می‌ توان directory را با ```I-``` به compiler معرفی کرد مثلا ```cc -c -I/usr/junk/include badinclude.c``` ، در اینجا I- به compiler می‌ گوید هنگام پردازش include ها directory مشخص‌ شده را نیز جست‌ وجو کند. نکته‌ ی مهم این است که l- مربوط به search path مربوط به header هاست و نباید آن را با L- که برای library های linker است اشتباه گرفت.

#### 🔹 Angle Brackets vs Double Quotes

دو شکل رایج #include عبارت‌ اند از :

```c
#include <myheader.h>
```

و:

```c
#include "myheader.h"
```

در حالت اول ، header به‌ عنوان یک header در مسیرهای include مورد جست‌ وجو قرار می‌ گیرد. در حالت دوم ، معمولاً منظور header است که بخشی از source code پروژه محسوب می‌ شود و اغلب در همان directory source یا یک مسیر نسبتاً نزدیک قرار دارد برای مثال :

```c
#include "myheader.h"
```

معمولاً نشان می‌ دهد که myheader.h بخشی از خود پروژه است ، نه یک header عمومی سیستم. اگر چنین include هم  resolve نشود ، یکی از احتمال‌ ها این است که source package که در اختیار دارید ناقص است و بخشی از source یا header های لازم در آن وجود ندارد.

#### 🔹 The C Preprocessor

یک نکته‌ ی بسیار مهم این است که compiler به‌ تنهایی مسئول انجام تمام کارهای مربوط به include# و macro ها نیست. قبل از اینکه source code اصلی توسط compiler پردازش شود ، C preprocessor آن را آماده می‌ کند در Unix این ابزار معمولاً ```cpp``` نام دارد. در GCC نیز می‌ توانید preprocessor را با ```gcc -E source.c``` اجرا کنید. Preprocessor source code را بازنویسی می‌ کند و بسیاری از directive هایی که با ```#``` شروع می‌ شوند را پردازش می‌ کند. سه گروه مهم directive هایی که کتاب معرفی می‌ کند عبارت‌ اند ازInclude files و Macro definitions و Conditionals .

#### 🔹 Include Files

برای include کردن یک فایل :

```c
#include <stdio.h>
```

یا:

```c
#include "myheader.h"
```

استفاده می‌ شود. در واقع l- که قبل‌ تر دیدیم نیز به preprocessor کمک می‌ کند directory های اضافی را برای پیدا کردن include file ها جست‌ وجو کند.


#### 🔹 Macro Definitions

با :

```c
#define BLAH something
```

می‌ توان macro تعریف کرد. Preprocessor بعد هنگام عبور از source code ، occurrence های BLAH را با مقدار تعریف‌ شده جایگزین می‌ کند. به‌ صورت قراردادی macro ها اغلب با حروف بزرگ نوشته می‌ شوند :

```c
#define BUFFER_SIZE 1024
```

اما preprocessor از نظر فنی الزام نمی‌ کند که نام macro حتماً uppercase باشد. یک macro حتی می‌ تواند ظاهری شبیه function یا variable داشته باشد و همین قابلیت گاهی باعث پیچیدگی و خطا های سخت برای debugging می‌ شود. همچنین لازم نیست macro حتماً در خود source code تعریف شود. می‌ توان آن را هنگام compile از command line تعریف کرد ```gcc -DBLAH=something source.c``` این تقریباً معادل تعریف macro در source code با define# است.

#### 🔹 Conditionals

امکان دارد Preprocessor بخش‌ هایی از source code را به‌ صورت شرطی وارد compilation کند. سه directive مهم ifdef# و if# و endif# هستند برای مثال :

```c
#ifdef DEBUG
fprintf(stderr, "This is a debugging message.\n");
#endif
```

اگر macro با نام DEBUG تعریف شده باشد ، خط ()fprintf به compiler داده می‌ شود اگر DEBUG تعریف نشده باشد ، preprocessor آن بخش را حذف می‌ کند. در مورد if # نیز expression بررسی می‌ شود و اگر مقدار آن صفر باشد ، کد داخل conditional به compiler منتقل نمی‌ شود. این قابلیت یکی از پایه‌ های مهم ساخت code های قابل‌ تنظیم برای environment ها و platform های مختلف است. نکته‌ ی بسیار مهم این است که preprocessor از C syntax آگاه نیست. یعنی preprocessor واقعاً نمی‌ فهمد که function و variable و type و statement چی هستند. آنچه می‌ فهمد directive ها و macro های خودش است و source code را بر اساس همان قواعد بازنویسی می‌ کند. می‌ توانید preprocessor را جداگانه اجرا کنید ، اما در حالت معمول compiler خودش آن را در build process اجرا می‌ کند و programmer نیازی به اجرای مستقیم cpp ندارد.

---

### make

وقتی یک برنامه از چند source file تشکیل شده یا compiler به option های خاصی نیاز دارد ، compile کردن دستی خیلی سریع پیچیده می‌ شود. مثلاً اگر پروژه‌ ای این فایل‌ ها را داشته باشد :

- main.c
- aux.c
- network.c
- parser.c
- config.c

ممکن است مجبور شوید چندین command مختلف را اجرا کنید و dependency ها را به خاطر بسپارید. حتی مهم‌ تر از آن ، اگر فقط یکی از source file ها تغییر کند ، معمولاً نیازی نیست کل برنامه از صفر rebuild شود. ابزار سنتی Unix برای مدیریت این مسئله ```make``` است.  اگر در پروژه فایلی مانند ```Makefile``` یا ```makefile``` دیدید ، احتمالاً build project با make مدیریت می‌ شود. کتاب همچنین اشاره می‌ کند که make خودش سیستم بسیار بزرگی است و آنچه در این فصل می‌ بینیم فقط بخش کوچکی از قابلیت‌ های آن است. بسیاری از package های Linux نیز یک لایه‌ ی دیگر روی make یا ابزارهای مشابه دارند مثلاً autotools که در فصل 16 بررسی خواهد شد.


#### 🔹 Core Idea of make

مهم‌ ترین مفهوم make هم target است. Target همان چیزی است که می‌ خواهیم بسازیم یا به آن برسیم ، Target می‌ تواند یک file واقعی باشد :

```
main.o
myprog
```

یا حتی یک label باشد مانند all یا clean یا install ، یک target ممکن است به target های دیگری وابسته باشد مثلاً :

```
myprog
   |
   +-- main.o
   |
   +-- aux.o
```

این dependency ها مشخص می‌ کنند برای ساختن myprog چه چیزهایی باید ابتدا وجود داشته باشند بعد برای ساختن هر target ، یک rule وجود دارد که command های لازم را مشخص می‌ کند به‌ صورت مفهومی :

```
Target
   |
   +--> Dependencies
   |
   +--> Rule
```

این مدل به make اجازه می‌ دهد build را به‌ صورت dependency graph دنبال کند و فقط چیزهایی را بسازد که لازم هستند.

#### 🔹 A Sample Makefile

بر اساس مثال‌ های قبلی ، می‌ توان یک Makefile ساده برای main.c و aux.c ساخت. یک نمونه :

```makefile
# object files
OBJS = aux.o main.o

all: myprog

myprog: $(OBJS)
	$(CC) -o myprog $(OBJS)
```

اولین خط ``` object files#``` یک comment است بعد ```OBJS = aux.o main.o``` یک macro/variable با نام OBJS تعریف می‌ کند وقتی بعداً بنویسیم ```(OBJS)$``` این عبارت به ```aux.o main.o``` گسترش پیدا می‌ کند.

#### 🔹 The all Target

در این مثال ```all: myprog``` اولین target است. make وقتی بدون argument اجرا شود ، به‌ طور معمول اولین target موجود در Makefile را به‌ عنوان target پیش‌ فرض در نظر می‌ گیرد بنابراین ```make``` عملاً به سمت all می‌ رود. all وابسته به ```myprog``` است سپس myprog به ```aux.o و main.o``` وابسته است. این dependency graph در نهایت چنین رابطه‌ ای دارد :

```
all
 |
 v
myprog
 |
 +---- main.o ---- main.c
 |
 +---- aux.o  ---- aux.c
```

برای myprog هم Makefile می‌ گوید ```myprog: $(OBJS)``` و چون OBJS به aux.o main.o گسترش پیدا می‌ کند ، make متوجه می‌ شود executable به این دو object file نیاز دارد.

#### 🔹 Tabs in Makefiles

یکی از جزئیات مهم و مشهور syntax مربوط به Makefile این است که command های داخل rule به‌ صورت سنتی باید با Tab شروع شوند مثلاً :

```makefile
myprog: $(OBJS)
	$(CC) -o myprog $(OBJS)
```

فضای قبل از (CC)$ در این مثال باید tab باشد. اگر به‌ جای tab از whitespace نا مناسب استفاده شود ، ممکن است با خطایی مانند ```missing separator``` مواجه شوید. این خطا به این معنی است که make نتوانسته separator موردانتظارش را پیدا کند.

#### 🔹 Running the Sample

اگر main.c و aux.c را در همان directory داشته باشیم اجرای ```make``` می‌ تواند چیزی شبیه این انجام دهد :

```
cc -c -o aux.o aux.c
cc -c -o main.o main.c
cc -o myprog aux.o main.o
```

ابتدا object file ها ساخته می‌ شوند و در مرحله‌ ی آخر executable نهایی link می‌ شود.

<img width="100%" height="491" alt="image" src="https://github.com/user-attachments/assets/156217eb-9069-4c5f-ab80-4f73a85aa469" />

#### 🔹 Built-in Rules

نکته‌ ی جالب این است که در Makefile مثال بالا اصلاً rule برای تبدیل ```aux.c``` به ```aux.o``` ننوشتیم. پس make از کجا فهمید باید چه کاری انجام دهد ؟ پاسخ این است که make مجموعه‌ ای از built-in rules دارد. برای example های معمول C هم ، make می‌ داند که اگر target مثل ```aux.o``` لازم باشد و source مثل ```aux.c``` وجود داشته باشد ، می‌ تواند از compiler استفاده کند ```cc -c aux.c``` بنابراین make بخشی از build process را خودش می‌ شناسد. این built-in rules قابل تغییر و تکمیل هستند و می‌ توانید rule های اختصاصی خودتان را نیز تعریف کنید. یکی از دلایل usefulness make همین است که برای build های ساده لازم نیست تمام command های جزئی را در Makefile بنویسید.

#### 🔹 Final Program Build

بعد از اینکه object file های لازم آماده شدند ، مرحله‌ ی نهایی ساخت executable انجام می‌ شود برای مثال ``` $(CC) -o myprog $(OBJS)``` و چون ```(OBJS)$``` به ```aux.o main.o``` گسترش پیدا می‌ کند ، command نهایی تقریباً چنین است ```cc -o myprog aux.o main.o``` ، دوباره باید تأکید کرد که ابتدای command در rule باید tab باشد. اگر Makefile چیزی شبیه این داشته باشد و indentation صحیح نباشد ```Makefile:7: *** missing separator. Stop.```
ممکن است رخ دهد. درک این جزئیات هنگام ویرایش Makefile های قدیمی اهمیت زیادی دارد.
#### 🔹 Dependency Updates

یکی از مهم‌ ترین قابلیت‌ های make این است که build را تا حد ممکن incremental انجام می‌ دهد. هدف make صرفاً اجرای تمام command ها نیست. هدف این است که target ها را با dependency هایشان up to date کند و حداقل مقدار لازم کار انجام شود مثلاً اگر یک بار ```make``` را اجرا کنید ، myprog ساخته می‌ شود اگر بلافاصله دوباره ```make``` را اجرا کنید، معمولاً : 

```make: Nothing to be done for 'all'.```

می‌بینید چرا؟ چون make متوجه می‌ شود myprog از dependency هایش جدید تر است و هیچ source یا object file وابسته‌ ای بعد از آن تغییر نکرده است حالا این کار را انجام دهید ```touch aux.c``` اکنون timestamp aux.c جدید تر شده است. وقتی دوباره ```make``` اجرا شود ، make متوجه می‌ شود ```aux.c``` از ```aux.o```جدیدتر است. بنابراین ابتدا ```aux.o``` را دوباره می‌ سازد سپس چون ```myprog``` به aux.o وابسته است و aux.o حالا جدید تر از myprog شده ، خود executable نیز دوباره link می‌ شود. این فرآیند به‌ صورت زنجیره‌ ای پیش می‌ رود و اگر source دیگری تغییر نکرده باشد، main.o دوباره ساخته نمی‌ شود. این رفتار یکی از دلایل اصلی سرعت و usefulness ابزار make در پروژه‌ های بزرگ است.

#### 🔹 Command-Line Arguments and Options

همچنین make را می‌ توان با argument های مختلف اجرا کرد یکی از کاربرد های مهم این است که target مشخصی را مستقیماً درخواست کنیم مثلاً ```make aux.o``` یعنی فقط target aux.o را در نظر بگیر. همچنین می‌ توانید macro های Makefile را از command line override کنید. مثلاً برای استفاده از clang باید ```make CC=clang``` در این حالت مقدار CC که روی command line تعیین شده است جایگزین مقدار پیش‌ فرض cc می‌ شود. این روش برای آزمایش گزینه‌ های compiler ، library ها و preprocessor definition ها نیز بسیار مفید است.

#### 🔹 Running make Without a Makefile

در بعضی build های بسیار ساده حتی ممکن است Makefile هم لازم نباشد اگر built-in rules بتوانند target مورد نظر را بسازند ، ممکن است فقط بنویسید ```make something``` و اگر source هم ```something.c``` موجود باشد ، make بتواند rule های داخلی را دنبال کند. مثلاً ممکن است در نهایت commandی شبیه این اجرا شود ```cc something.o -o something``` این روش برای برنامه‌ های بسیار ساده مناسب است. اگر برنامه به library خاص یا include directory  اضافی یا compiler flag خاص یا link option های ویژه نیاز داشته باشد ، بهتر است یک Makefile صریح داشته باشید. کتاب همچنین اشاره می‌ کند که اجرای make بدون Makefile گاهی برای ابزارهایی مانند Fortran و Lex  و Yacc مفید است ، چون می‌ توانید اجازه دهید built-in rule های make نحوه‌ ی اجرای ابزار را به شما نشان دهند.

#### 🔹 Useful Options

دو option مهم ```n-``` و ```f file-``` هستند. n کاربردش command هایی را که make قصد اجرای آن‌ها را دارد چاپ می‌ کند ، ولی آن‌ ها را اجرا نمی‌کند ```make -n``` این option مخصوصاً قبل از یک target خطرناک مانند install بسیار مفید است. f- به make می‌ گوید به‌ جای Makefile یا makefile از فایل دیگری به‌ عنوان Makefile استفاده کند مثلا :

 ```make -f MyBuildFile```

#### 🔹 Standard Macros and Variables

در make اصطلاحات macro و variable گاهی شبیه یکدیگر به نظر می‌ رسند. در توضیح کتاب ، macro معمولاً مقداری است که در طول build تغییر چندانی نمی‌ کند ، در حالی که variable می‌ تواند در هنگام اجرای rule ها مقدارهای مختلفی داشته باشد.

#### 🔹 Standard Macros

- استاندارد **CC** مشخص می‌ کند از چه C compiler استفاده شود مقدار پیش‌ فرض معمول ```cc```است مثلاً ```make CC=clang```.
- استاندارد **CFLAGS** گزینه‌ های compiler برای ساخت object code هستند مثلاً ```CFLAGS = -O2 -Wall```.
- استاندارد **CPPFLAGS** برای option های مربوط به C preprocessor هستند مثلاً ```CPPFLAGS = -DDEBUG```.
- استاندارد **LDFLAGS** گزینه‌ هایی هستند که در مرحله‌ی linking به linker مربوط می‌ شوند برای مثال ```LDFLAGS = -L/usr/local/lib ```.
- استاندارد **LDLIBS** برای library option هایی استفاده می‌ شود که می‌ خواهید جدا از search path مربوط به LDFLAGS نگهداری شوند مثلا ```LDLIBS = -lm -lpng```.
- استاندارد **CXXFLAGS** در GNU make برای compiler flag های مربوط به C++ استفاده می‌ شود.

#### 🔹 Automatic Variables

برخی make variable های خاص که داخل rule ها به‌ صورت خودکار مقدار می‌ گیرند مثلا **@$** target فعلی را نشان می‌ دهد اگر rule این باشد :

```makefile
myprog: $(OBJS)
	$(CC) -o $@ $(OBJS)
```

در زمان ساخت myprog از ```@$``` به ```myprog``` گسترش پیدا می‌ کند. **<$** اولین dependency مربوط به target فعلی است مثلاً اگر rule داشته باشیم ```target: source.c``` آن‌ وقت ```<$``` به ```source.c``` تبدیل می‌ شود. **$** هم basename یا stem مربوط به target را نشان می‌ دهد مثلاً اگر target آن something.o باشد ، مقدار ```*$``` برابر ```something``` است این variable ها در pattern ها و rule های عمومی بسیار کاربرد دارند.

#### 🔹 GNU make vs Other make Implementations

باید توجه کرد که همه‌ ی نسخه‌ های make دقیقاً قابلیت‌ های GNU make را ندارند. GNU make تعداد زیادی extension ، built-in rule و feature دارد. این مسئله وقتی روی Linux کار می‌ کنید معمولاً مشکلی ایجاد نمی‌ کند ، اما اگر همان Makefile را به سیستم‌ هایی مانند Solaris یا بعضی BSD ها ببرید ، ممکن است بعضی option ها یا قابلیت‌ ها در دسترس نباشند. کتاب اشاره می‌ کند که این یکی از مشکلاتی است که build system های چند سکویی مانند GNU autotools برای آن طراحی شده‌ اند.

#### 🔹 Conventional Targets

بسیاری از Makefile ها علاوه بر target های اصلی، target های قراردادی و متداولی دارند که عملیات جانبی build را انجام می‌ دهند. یکی از رایج‌ ترین target ها ```make clean``` است. معمولاً تمام object file ها و executable های تولید شده را حذف می‌ کند با این کار می‌توان build را از وضعیت تمیز شروع کرد مثلاً : 

```makefile
clean:
	rm -f $(OBJS) myprog
```

در Makefile هایی که توسط GNU autotools ایجاد شده‌اند ، distclean معمولاً چیزهایی را که بخشی از source distribution اولیه نبوده‌اند حذف می‌ کند. این target می‌ تواند حتی فایل‌ هایی مانند Makefile تولید شده را نیز حذف کند. جزئیات آن در Chapter 16 بیشتر بررسی خواهد شد.

معمولاً target برای نصب program و فایل‌ های مرتبط در directory هایی است که Makefile به‌عنوان محل نصب در نظر گرفته است مثلاً ```make install```. می‌ تواند فایل‌ ها را در مسیرهایی مانند /usr/local/bin یا مکان‌ های مشابه قرار دهد. این target را نباید کورکورانه اجرا کرد ، چون ممکن است عملیات filesystem زیادی انجام دهد قبل از اجرای واقعی بهتر است دستور ```make -n install``` را اجرا کنید تا command هایی که قرار است اجرا شوند را ببینید.

بعضی پروژه‌ ها این target ها را برای اجرای تست‌ های project در اختیار قرار می‌ دهند با ```make test``` یا ```make check``` انجام می شود.

این target در Makefile های قدیمی‌ تر ممکن است برای تولید dependency مربوط به include file ها استفاده شده باشد. کتاب می‌ گوید چنین target هایی ممکن است compiler را با option هایی مانند ```M-``` اجرا کنند و حتی Makefile را تغییر دهند. امروزه این روش نسبت به گذشته متداول نیست ، ولی اگر در دستورالعمل یک پروژه‌ ی قدیمی با آن مواجه شدید ، باید بدانید هدف آن ایجاد dependency information برای source code بوده است.

همان‌ طور که قبلاً دیدیم ، all معمولاً target پیش‌ فرض Makefile است و به executable یا مجموعه‌ ی اصلی build وابسته می‌ شود.

#### 🔹 Makefile Organization

در Makefile های مختلف ممکن است style های متفاوتی وجود داشته باشد ، اما در بسیاری از آن‌ ها organization مشخصی مشاهده می‌ شود. در قسمت ابتدایی Makefile معمولاً dependency های library و include directory ها به‌ صورت گروه‌ بندی‌شده تعریف می‌ شوند مثلاً :

```makefile
MYPACKAGE_INCLUDES = -I/usr/local/include/mypackage
MYPACKAGE_LIB = -L/usr/local/lib/mypackage -lmypackage

PNG_INCLUDES = -I/usr/local/include
PNG_LIB = -L/usr/local/lib -lpng
```

بعد این مقادیر در macro های اصلی compiler و linker استفاده می‌ شوند :

```makefile
CFLAGS = $(CFLAGS) $(MYPACKAGE_INCLUDES) $(PNG_INCLUDES)
LDFLAGS = $(LDFLAGS) $(MYPACKAGE_LIB) $(PNG_LIB)
```

از نظر مفهومی ، هدف این است که dependency ها در جای مشخصی تعریف شوند تا rule های اصلی شلوغ و تکراری نشوند.

#### 🔹 Grouping Object Files

معمولاً Object file ها بر اساس executable که به آن‌ ها نیاز دارد گروه‌ بندی می‌ شوند. فرض کنید دو executable داریم boring و trite که هر دو از util.o استفاده می‌ کنند می‌ توان چیزی شبیه این داشت : 

```makefile
UTIL_OBJS = util.o

BORING_OBJS = $(UTIL_OBJS) boring.o
TRITE_OBJS = $(UTIL_OBJS) trite.o

PROGS = boring trite
```

و rule های build :
```makefile
all: $(PROGS)

boring: $(BORING_OBJS)
	$(CC) -o $@ $(BORING_OBJS) $(LDFLAGS)

trite: $(TRITE_OBJS)
	$(CC) -o $@ $(TRITE_OBJS) $(LDFLAGS)
```

مزیت این organization این است که dependency های هر executable دقیق باقی می‌ مانند نباید بدون دلیل دو executable کاملاً متفاوت را در یک rule ادغام کنیم :

```makefile
boring trite: ...
```

چنین کاری می‌ تواند dependency graph نادرستی ایجاد کند مثلاً اگر boring فقط به boring.c و trite فقط به trite.c وابسته باشند ، ترکیب کردن آن‌ ها در یک rule ممکن است باعث شود make فکر کند هر executable به source file مربوط به دیگری هم وابستگی دارد. در نتیجه تغییر boring.c می‌ تواند باعث rebuild شدن trite نیز شود همچنین جدا نگه داشتن target ها باعث می‌ شود که جا به‌ جا کردن rule ها ساده‌ تر باشد و حذف یک executable مستقل‌ تر باشد و grouping برنامه‌ ها راحت‌ تر تغییر کند و dependency graph دقیق‌ تر باقی بماند. اگر یک object file به rule خاصی نیاز دارد ، کتاب توصیه می‌ کند rule اختصاصی آن را نزدیک rule executable مربوطه قرار دهید  و اگر چند executable از یک object file مشترک استفاده می‌ کنند ، rule مربوط به آن object بهتر است بالاتر از rule های executable ها قرار گیرد.

---

### Lex and Yacc

ممکن است در هنگام compile کردن بعضی برنامه‌ هایی که configuration file یا command language را پردازش می‌ کنند با دو نام قدیمی ولی مهم مواجه شوید Lex و Yacc این دو ابزار از building block های مهم در ساخت parser ها و language processor ها هستند.

#### 🔹 Lex

ابزار Lex یک tokenizer یا lexical analyzer است. کار آن این است که ورودی متنی را بررسی کند و بخش‌ های مختلف آن را به token هایی تبدیل کند که parser بتواند با آن‌ ها کار کند در دنیای GNU/Linux نسخه‌ ی معروف آن ```flex``` است. هنگام link کردن بعضی برنامه‌ های تولید شده توسط Lex ممکن است به library option هایی مانند ```ll-``` یا ```lfl-``` نیاز داشته باشید.

#### 🔹 Yacc

ابزار Yacc نقش parser را دارد. ورودی token ها را دریافت می‌ کند و آن‌ ها را مطابق یک grammar پردازش می‌ کند تا structure مورد نظر زبان را تشخیص دهد نسخه‌ ی GNU آن ```bison``` است. برای compatibility با Yacc می‌ توان ```bison -y``` را استفاده کرد. بعضی برنامه‌ های مبتنی بر Yacc ممکن است هنگام linking به ```ly-``` نیاز داشته باشند به‌ صورت ساده :

```
Text
  |
  v
Lex / flex
  |
  v
Tokens
  |
  v
Yacc / bison
  |
  v
Parsed Structure
```

بنابراین Lex و Yacc را می‌ توان به‌ ترتیب در نقش tokenizer و parser دید.
---

### Scripting Languages

در گذشته مدیران Unix بیشتر با shell و ابزارهایی مانند awk سروکار داشتند ، اما در طول زمان تعداد زیادی scripting language قدرتمند وارد ecosystem Unix/Linux شدند. بخشی از utility هایی که در گذشته با C پیاده‌ سازی می‌ شدند ، در برخی موارد با  scripting language ها نوشته شدند مخصوصاً جایی که کارهایی مانند text processing یا automation بخش اصلی برنامه بود. تفاوت مهم scripting language ها با C این است که معمولاً برای اجرای مستقیم source code به interpreter نیاز دارند در نتیجه به‌ جای این مدل : 

```
Source
   |
   v
Compiler
   |
   v
Executable
```

اغلب با این مدل رو به‌ رو هستیم :

```
Script
   |
   v
Interpreter
   |
   v
Execution
```

#### 🔹 Shebang

یکی از مهم‌ ترین نکات مربوط به script ها ، خط اول آن‌ هاست که با ```!#``` شروع می‌ شود مثلاً :

```python
#!/usr/bin/python

print("Hello")
```

یا:

```
#!/usr/bin/env python
```

در این حالت pathname بعد از !# به executable مربوط به interpreter اشاره می‌ کند. هنگامی که Unix یک فایل executable را می‌بیند که با !# شروع شده است ، program مشخص‌ شده در همان خط را اجرا می‌ کند و script را در اختیار آن قرار می‌ دهد. پس حتی concept script بودن یک فایل فقط به shell محدود نیست. مثلاً کتاب یک مثال جالب با tail ارائه می‌ کند :

```
#!/usr/bin/tail -2

This program won't print this line,
but it will print this line...

and this line, too.
```

در اینجا interpreter واقعی یک shell یا Python نیست بلکه tail است. این نشان می‌ دهد که mechanism مربوط به shebang در سطح سیستم‌ عامل قرار دارد و می‌ تواند executable های متفاوتی را برای پردازش فایل text-based فراخوانی کند.

#### 🔹 Problems with Interpreter Paths

یکی از رایج‌ ترین مشکلات script ها ، اشتباه بودن pathname مربوط به interpreter است مثلاً اگر script بگوید ```usr/bin/tail/!#``` ولی در سیستم شما tail در ```bin/tail/``` قرار داشته باشد ، اجرای script می‌ تواند با خطایی مانند ```bad interpreter: No such file or directory``` مواجه شود. در مورد !# همچنین نباید روی پشتیبانی یکسان همه‌ ی سیستم‌ ها از چند argument حساب کنید. خط shebang در برخی سیستم‌ ها ممکن است argument ها را به‌ شکلی متفاوت parse کند ، بنابراین استفاده‌ ی پیچیده از چند argument در shebang می‌ تواند portability را خراب کند برای همین اگر command line مربوط به interpreter پیچیده است ، wrapper script یا روش دیگری می‌ تواند مطمئن‌ تر باشد.

#### 🔹 Python


زبان Python یکی از scripting language های مهم و پرکاربرد است کتاب چند ویژگی مهم Python را مطرح می‌ کند ، از جمله:

- text processing
- database access
- networking
- multithreading
- interactive mode
- Well organized object model

در بسیاری از سیستم‌ های مطرح‌ شده در کتاب executable مربوط به ```python``` است و معمولاً در مسیری مانند ```usr/bin/ ``` قرار دارد. Python فقط برای script های کوچک shell-like استفاده نمی‌ شود و در حوزه‌ های دیگری مانند data analysis و web applications و automation و software development نیز کاربرد دارد. این انعطاف‌ پذیری یکی از دلایل اصلی تبدیل شدن Python به یکی از زبان‌ های مهم در Linux ecosystem است.

#### 🔹 Perl

زبان Perl یکی از زبان‌ های قدیمی مهم Unix است Perl در زمینه‌ هایی مثل text processing و text conversion و file manipulation قدرت زیادی دارد. اگرچه در گذر زمان بخشی از کاربرد های Perl به زبان‌ هایی مانند Python منتقل شده، هنوز ممکن است utility ها و script های قدیمی و حتی بعضی ابزارهای مهم سیستم را ببینید که با Perl نوشته شده‌ اند. به همین دلیل حتی اگر خودتان قصد نوشتن Perl نداشته باشید ، شناخت کلی آن برای مدیریت Linux system مفید است مخصوصاً وقتی با script های قدیمی سروکار دارید.

#### 🔹 Other Scripting Languages

کتاب به چندین زبان دیگر نیز اشاره می‌ کند که ممکن است در Linux system با آن‌ ها برخورد کنید :

زبان **PHP** : یک language برای پردازش hypertext است و historically در dynamic web applications بسیار رایج بوده است علاوه بر web ، بعضی افراد از PHP برای standalone script نیز استفاده می‌ کنند.

زبان **Ruby** : یک زبان object-oriented است که در میان بعضی web developer ها و programmer های علاقه‌ مند به object-oriented programming محبوب بوده است.

زبان **JavaScript** : ابتدا بیش از همه به‌ عنوان زبان داخل web browser شناخته می‌ شد و برای مدیریت dynamic content صفحات وب استفاده می‌ شد اما بعد ها با runtime هایی مانند ```Node.js``` استفاده از JavaScript در server-side programming و scripting نیز گسترده‌ تر شد. Executable مربوط به Node.js معمولاً ```node``` نام  دارد.

زبان **Emacs Lisp** : نوعی از Lisp است که در محیط Emacs استفاده می‌ شود. برای توسعه و customization خود Emacs اهمیت ویژه‌ ای دارد.

زبان **MATLAB and Octave** : یک محیط و زبان تجاری برای محاسبات ریاضی ، مخصوصاً matrix-oriented programming است. Octave یک پروژه‌ ی آزاد است که در بسیاری از زمینه‌ ها شباهت زیادی به MATLAB دارد.

زبان **R** : یک زبان آزاد و بسیار مهم برای statistical analysis و پردازش داده‌ های آماری است.

محیط **Mathematica** : یک محیط و language تجاری برای محاسبات ریاضی و scientific computing است.

زبان **m4** : یک macro-processing language است که در ecosystem مربوط به GNU autotools کاربرد دارد. این زبان را معمولاً به‌ عنوان زبان عمومی برای script های روزمره نمی‌ بینید ، بلکه بیشتر در ابزارهای build و generation مورد استفاده قرار می‌ گیرد.

زبان **Tcl** : مخفف Tool Command Language است. یک scripting language نسبتاً ساده است که historically با toolkit گرافیکی Tk و ابزار automation مانند Expect ارتباط داشته است. اگرچه کاربرد Tcl نسبت به گذشته کمتر شده ، اما هنوز ممکن است در بعضی software ها ، script های قدیمی و پروژه‌ های embedded با آن رو به‌ رو شوید.

---

### Java

زبان Java از نظر مدل کلی یک زبان compiled است ، اما شیوه‌ ی اجرای آن با C تفاوت مهمی دارد. در مدل رایج Java ، source code مستقیماً executable native نهایی نمی‌ شود ابتدا source code به bytecode تبدیل می‌ شود سپس bytecode در محیط اجرای Java توسط یک virtual machine اجرا می‌ شود مدل مفهومی :

```
Java Source
     |
     v
   javac
     |
     v
Java Bytecode
     |
     v
   JVM
     |
     v
Execution
```

این virtual machine را نباید با virtual machine مربوط به hypervisor ها اشتباه گرفت در Linux معمولاً bytecode با پسوند ```class.``` ذخیره می‌ شود. کتاب دو نوع کلی Java compiler را از هم متمایز می‌ کند :

- اول compiler هایی که native machine code تولید می‌ کنند.
- دوم compiler هایی که bytecode تولید می‌ کنند تا توسط virtual machine اجرا شود.

آن چیزی که تقریباً همیشه در Linux با آن رو به‌ رو می‌ شوید ، مدل bytecode است.


#### 🔹 Java Runtime Environment

برای اجرای Java bytecode به runtime مناسب Java نیاز دارید. کتاب از Java Runtime Environment (JRE) به‌ عنوان محیطی نام می‌ برد که برنامه‌ های لازم برای اجرای Java bytecode را فراهم می‌ کند. در مدل ارائه‌ شده در کتاب ، فایل bytecode با command مانند ```java file.class``` اجرا می‌ شود. از نظر syntax عملی Java ، معمولاً class را با نام آن اجرا می‌ کنیم ، یعنی مثلاً اگر فایل ```Hello.class``` داشته باشیم ، command معمول چنین است ```java Hello``` نه ```java Hello.class``` ، پس باید بین مفهوم فایل class. و نام class در command اجرا تفاوت قائل شویم.

#### 🔹 JAR Files

ممکن است برنامه‌ ی Java به‌ جای یک فایل class. منفرد ، مجموعه‌ ای از class ها و resource های مختلف داشته باشد. برای packaging این موارد از ```jar.``` استفاده می‌ شود. JAR را می‌ توان نوعی archive برای فایل‌ های Java در نظر گرفت. برای اجرای یک JAR executable از ```java -jar file.jar``` استفاده می‌ شود. از این نظر JAR تا حدی شبیه یک archive package است که فایل‌ های class. و resource های مرتبط را در خود نگه می‌ دارد.

#### 🔹 JAVA_HOME

برخی محیط‌ ها نیاز دارند که location مربوط به Java installation مشخص باشد برای این کار ممکن است environment variable زیر تنظیم شود ```JAVA_HOME``` این variable معمولاً به prefix مربوط به installation Java اشاره می‌ کند مثلاً ```export JAVA_HOME=/opt/java``` مقدار دقیق بسته به محل نصب Java متفاوت است.

#### 🔹 CLASSPATH

یکی دیگر از environment variable هایی که ممکن است در سیستم‌ های Java ببینید ```CLASSPATH``` است. این variable مجموعه‌ ای از directory هایی را مشخص می‌ کند که Java باید برای class های مورد نیاز program بررسی کند مانند PATH که separator معمول directory ها در آن ```:``` است مثلاً ```export CLASSPATH=/opt/myapp/classes:/opt/lib/classes``` اما استفاده‌ی بی‌ دلیل از CLASSPATH می‌ تواند configuration را پیچیده کند ، چون محیط global روی محل جست‌ وجوی class ها تأثیر می‌ گذارد.

#### 🔹 The Java Development Kit

برای compile کردن source code جاوا به ```JDK``` یعنی Java Development Kit نیاز دارید. compiler اصلی آن ``` javac``` است مثلاً ```javac Hello.java``` که source code را به فایل‌ های class. تبدیل می‌ کند. JDK همچنین ابزار ```jar``` را در اختیار شما قرار می‌ دهد که برای ساختن و unpack کردن JAR file ها استفاده می‌ شود از نظر مفهومی jar تا حدی مشابه نقش tar برای archive های Unix است.

---

### Looking Forward: Compiling Packages

دنیای compiler ها و programming language ها بسیار گسترده‌ تر از چیزهایی است که در این فصل بررسی شد. زبان‌ های compiled جدیدتری به‌ طور مداوم توسعه پیدا می‌ کنند و کتاب در زمان نگارش خود به زبان‌ هایی مانند Go و Rust اشاره می‌ کند همچنین infrastructure هایی مانند ```LLVM``` فرآیند توسعه‌ ی compiler ها را ساده‌ تر و قدرتمند تر کرده‌ اند. LLVM فقط یک compiler برای یک language خاص نیست بلکه مجموعه‌ ای از infrastructure ها و component هایی است که توسعه‌ دهندگان compiler و language implementation می‌ توانند روی آن‌ ها build کنند. اگر بخواهید خود compiler ها ، language processing و طراحی language را عمیق‌ تر مطالعه کنید ، این موضوع به حوزه‌ ای بسیار بزرگ‌ تر تبدیل می‌ شود. اما برای یک Linux administrator مسئله‌ ی عملی مهم‌ تر این است که وقتی source code و ابزارهای build را در اختیار داریم ، چطور یک software package را واقعاً از source بسازیم و نصب کنیم ؟ تا اینجا اجزا ی پایه را می‌ شناسیم :

```
C Source
   |
   v
Preprocessor
   |
   v
Compiler
   |
   v
Object Files
   |
   v
Linker
   |
   +------> Static Libraries
   |
   +------> Shared Libraries
                   |
                   v
          Dynamic Linker/Loader
```

و برای پروژه‌ های چند فایلی :

```
Source Files
     |
     v
    make
     |
     v
Compiler / Linker
     |
     v
Executable
```

#### 🔹 Source-to-Execution Flow

در پایان این فصل ، مسیر یک برنامه‌ ی C را می‌ توان با جزئیات بیشتری این‌ طور دید :

```
                    C Source Code
                           |
                           v
                    C Preprocessor
                           |
             +-------------+-------------+
             |                           |
             v                           v
        Header Files                Macro Expansion
             |                           |
             +-------------+-------------+
                           |
                           v
                       Compiler
                           |
                           v
                     Object File
                          (.o)
                           |
                           v
                        Linker
                          (ld)
                           |
              +------------+------------+
              |                         |
              v                         v
      Static Libraries           Shared Libraries
          (.a)                       (.so)
              |                         |
              v                         v
      Code copied into          Dependency recorded
      executable                in executable
                                        |
                                        v
                                Runtime Loader
                                  (ld.so)
                                        |
                                        v
                                Shared Library
                                        |
                                        v
                                  Running Process
```

تفاوت static و shared library در این مسیر بسیار مهم است در static linking کد موردنیاز library هنگام link وارد executable می‌ شود. 

```
Library Code
     |
     v
Executable
```

در shared linking :

```
Executable
     |
     v
Needs libX.so
     |
     v
Dynamic Loader
     |
     v
Find libX.so
     |
     v
Load into process
```

به همین دلیل اگر یک executable در زمان اجرا با خطایی شبیه ```error while loading shared libraries``` روبرو شود ، مسئله دیگر مربوط به compiler نیست باید سراغ dependency های runtime ، loader ، search path ، cache ، rpath و در موارد خاص environment variable هایی مانند LD_LIBRARY_PATH برویم. این تفکیک build-time dependency و runtime dependency یکی از اصلی‌ ترین مفاهیم فصل پانزدهم است.

#### 🔹 Build Management with make

در پروژه‌ های کوچک ```cc -o hello hello.c``` کافی است اما با افزایش تعداد source file ها ، dependency ها نیز افزایش پیدا می‌ کنند :

```
main.c
  |
  v
main.o --------+
               |
aux.c          |
  |            |
  v            v
aux.o -------> myprog
```

در make این dependency graph را به‌ شکل explicit یا با استفاده از built-in rules می‌ شناسد و تلاش می‌ کند فقط بخش‌ های outdated را rebuild کند مثلاً اگر فقط ```aux.c``` تغییر کند دوباره ساخته می‌ شود ، ولی main.o نیازی به rebuild ندارد به همین دلیل make بیشتر از اینکه یک compiler wrapper ساده باشد ،  یک dependency-driven build system است :
```
aux.c
  |
  v
aux.o
  |
  v
myprog
```

#### 🔹 Tools to Remember

ابزار های اصلی فصل را می‌ توان این‌ گونه دسته‌ بندی کرد :

```
Compiler / Build
    cc
    gcc
    clang
    make

Linking
    ld
    nm
    ldconfig
    ldd
    patchelf

C Preprocessing
    cpp
    gcc -E

Library Handling
    ld.so
    ldconfig
    /etc/ld.so.conf
    /etc/ld.so.cache

Lex / Yacc
    flex
    bison
    xkbcomp

Scripting
    python
    perl
    node

Java
    javac
    java
    jar
```

این command ها همگی یک نقش مشترک ندارند بعضی برای build ، بعضی برای linking ، بعضی برای runtime ، بعضی برای language processing و بعضی برای اجرای program ها هستند.

---

### Summary

فصل پانزدهم با معرفی Development Tools ، ابزارها و مراحل اصلی ساخت و اجرای program در Linux را بررسی می‌ کند. در بخش C هم source code ابتدا توسط preprocessor پردازش می‌ شود و سپس compiler آن را به object file تبدیل می‌ کند. در پروژه‌ های چند فایلی ، object file ها با linker و در صورت نیاز با library ها ترکیب می‌ شوند تا executable ساخته شود.

در ادامه تفاوت static library و shared library بررسی می‌ شود. در shared linking ، executable به library وابسته می‌ ماند و dynamic linker/loader در زمان اجرا library مناسب را پیدا و load می‌ کند. مفاهیمی مانند ldd ، rpath ، /etc/ld.so.conf ، /etc/ld.so.cache ، ldconfig و LD_LIBRARY_PATH نیز برای درک runtime library dependencies معرفی می‌ شوند همچنین درباره‌ ی استفاده‌ ی محتاطانه از LD_LIBRARY_PATH و کاربرد wrapper script صحبت می‌ شود. 

بخش بعدی به header files و C preprocessor اختصاص دارد و نقش directive هایی مانند #include،   #define و conditional های ifdef# و if# را توضیح می‌ دهد. سپس make به‌ عنوان ابزار مدیریت build معرفی می‌ شود که با استفاده از target ، dependency و rule و با بررسی timestamp ها ، فقط بخش‌ های مورد نیاز را دوباره build می‌ کند. در این بخش variable هایی مانند CC ، CFLAGS ، CPPFLAGS ، LDFLAGS ، LDLIBS و CXXFLAGS و همچنین automatic variable ها معرفی می‌ شوند. 

در ادامه ، ابزارهای Lex/Yacc و مفهوم shebang برای اجرای script ها بررسی می‌ شوند. سپس زبان‌ هایی مانند Python و Perl و چند scripting language دیگر معرفی شده و در پایان مدل اجرای Java ، bytecode ، JVM ، JAR ، JAVA_HOME و CLASSPATH بررسی می‌ شود. 

در بخش پایانی نیز به ابزارها و فناوری‌ های جدید تری مانند Go ، Rust و LLVM اشاره می‌ شود و زمینه برای بحث درباره‌ ی ساخت و مدیریت package ها در فصل بعد فراهم می‌ شود.
در مجموع ، این فصل دیدی عملی از ارتباط source code ، preprocessor ، compiler ، object file ، linker ، library ، build system و runtime loader ارائه می‌ دهد و نشان می‌ دهد هر ابزار در کدام مرحله از فرایند توسعه و اجرای software در Linux نقش دارد.
