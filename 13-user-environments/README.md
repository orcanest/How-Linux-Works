## محیط کاربر
### 🐧 فصل سیزدهم کتاب How Linux Works


فصل سیزدهم با موضوع User Environments به نقطه‌ای می‌رسد که بین زیرساخت Linux و محیطی که کاربر هر روز با آن کار می‌کند ارتباط برقرار می‌شود. تا اینجا بیشتر درباره‌ی kernel، processها، filesystem، شبکه، serviceها و shell صحبت شده بود؛ اما در نهایت کاربر باید بتواند یک shell را باز کند و در یک محیط کاری مشخص شروع به کار کند.

این محیط از هیچ به وجود نمی‌آید. وقتی shell اجرا می‌شود، تعدادی تنظیمات از فایل‌های مختلف خوانده می‌شوند و چیزهایی مثل PATH، prompt، aliasها، editor، pager، umask و دیگر environment variableها شکل می‌گیرند.

بخش زیادی از این تنظیمات در فایل‌های موجود در home directory کاربر قرار دارند. این فایل‌ها معمولاً نامشان با . شروع می‌شود؛ به همین دلیل به آن‌ها dot files گفته می‌شود. برای مثال:

- .bashrc
- .bash_profile
- .profile
- .cshrc

این فایل‌ها هیچ ویژگی جادویی خاصی به خاطر نقطه‌ی ابتدای نامشان ندارند؛ فقط بسیاری از ابزارها و file managerها آن‌ها را به‌صورت پیش‌فرض نمایش نمی‌دهند تا home directory بیش از حد شلوغ نشود.

نکته‌ی مهم فصل این است که startup fileها اگر به‌مرور زمان بی‌دلیل بزرگ و پیچیده شوند، می‌توانند منبع مشکلات جدی شوند. یک environment ممکن است در یک shell درست کار کند ولی در login از طریق SSH، در یک terminal emulator یا در یک shell غیرinteractive رفتار متفاوتی داشته باشد.

ساختار فصل سیزدهم در ویرایش سوم به این صورت است: 13.1 تا 13.7 و زیر‌بخش‌های مربوط به command path، manual page path، prompt، aliasها، umask، startup fileهای Bash و tcsh و defaultهای shell، editor و pager.

---
📚 Table of Contents

- [13.1 Guidelines for Creating Startup Files](#131-guidelines-for-creating-startup-files)
  - [Simplicity](#simplicity)
  - [Readability](#readability)
- [13.2 When to Alter Startup Files](#132-when-to-alter-startup-files)
- [13.3 Shell Startup File Elements](#133-shell-startup-file-elements)
  - [13.3.1 The Command Path](#1331-the-command-path)
    - [The Difference Between /usr/local/bin and /usr/bin](#the-difference-between-usrlocalbin-and-usrbin)
    - [Programs Located in Special Paths](#programs-located-in-special-paths)
    - [$HOME/bin](#homebin)
    - [Adding a Dot to the PATH](#adding-a-dot-to-the-path)
  - [13.3.2 The Manual Page Path](#1332-the-manual-page-path)
  - [13.3.3 The Prompt](#1333-the-prompt)
    - [Special Characters](#special-characters)
    - [A Simple Prompt in Bash](#a-simple-prompt-in-bash)
    - [Some Important Escapes in Bash](#some-important-escapes-in-bash)
  - [13.3.4 Aliases](#1334-aliases)
    - [Problem One: Arguments](#problem-one-arguments)
    - [Problem Two: Finding Where It's Defined](#problem-two-finding-where-its-defined)
    - [Problem Three: Subshells and Noninteractive Shells](#problem-three-subshells-and-noninteractive-shells)
    - [Aliasing Standard Commands](#aliasing-standard-commands)
    - [When Does an Alias Make Sense?](#when-does-an-alias-make-sense)
  - [13.3.5 The Permissions Mask](#1335-the-permissions-mask)
    - [umask 077](#umask-077)
    - [umask 022](#umask-022)
- [13.4 Startup File Order and Examples](#134-startup-file-order-and-examples)
  - [13.4.1 The bash Shell](#1341-the-bash-shell)
    - [Interactive Shell](#interactive-shell)
    - [Noninteractive Shell](#noninteractive-shell)
    - [Login Shells](#login-shells)
    - [Order of Reading the Files](#order-of-reading-the-files)
    - [Non-Login Shells](#non-login-shells)
    - [Why Are Login and Non-Login Separate?](#why-are-login-and-non-login-separate)
    - [Building a Shared .bashrc](#building-a-shared-bashrc)
    - [Why Does Sourcing Matter?](#why-does-sourcing-matter)
    - [Checking Whether a Shell Is Interactive](#checking-whether-a-shell-is-interactive)
    - [A Practical Pattern for .bash_profile](#a-practical-pattern-for-bash_profile)
  - [13.4.2 The tcsh Shell](#1342-the-tcsh-shell)
    - [An Example .cshrc](#an-example-cshrc)
- [13.5 Default User Settings](#135-default-user-settings)
  - [The Right Way to Test](#the-right-way-to-test)
  - [13.5.1 Shell Defaults](#1351-shell-defaults)
  - [13.5.2 Editor](#1352-editor)
  - [13.5.3 Pager](#1353-pager)
- [13.6 Startup File Pitfalls](#136-startup-file-pitfalls)
  - [Running Graphical Commands in Startup](#running-graphical-commands-in-startup)
  - [Manually Setting DISPLAY](#manually-setting-display)
  - [Manually Setting Terminal Type](#manually-setting-terminal-type)
  - [Insufficient Comments](#insufficient-comments)
  - [Producing Output in Startup Files](#producing-output-in-startup-files)
  - [LD_LIBRARY_PATH](#ld_library_path)
- [13.7 Further Startup Topics](#137-further-startup-topics)
- [How the Pieces of the Startup Environment Fit Together](#how-the-pieces-of-the-startup-environment-fit-together)
- [A Simple, Clean Bash Environment Example](#a-simple-clean-bash-environment-example)
- [Practical Tips from the Chapter](#practical-tips-from-the-chapter)
- [Summary](#summary)

---

### 13.1 Guidelines for Creating Startup Files

اولین نکته‌ای که هنگام ساخت startup file باید در نظر بگیریم این است که این فایل برای چه کسی نوشته می‌شود.

اگر startup file فقط برای خودمان باشد، اشتباه موجود در آن مستقیماً فقط روی خودمان اثر می‌گذارد. اما اگر این فایل قرار است default یک سیستم باشد یا برای چندین user استفاده شود، شرایط کاملاً فرق می‌کند.

مثلاً تصور کنید یک startup file را برای ده user نصب کرده‌اید و بعد متوجه می‌شوید یک خط آن اشتباه بوده است. حالا همان خطا باید در ده environment مختلف اصلاح شود.

بنابراین کتاب روی دو اصل اساسی تأکید می‌کند:

#### Simplicity

startup file را تا حد امکان:

- کوتاه
- ساده
- قابل فهم
- قابل تغییر

نگه دارید.

هر command، variable، شرط یا dependency که به آن اضافه می‌کنید، یک نقطه‌ی بالقوه برای failure است.

در عمل، startup file جای مناسبی برای قرار دادن هر configuration ممکن نیست.

مثلاً اگر فقط برای اینکه یک ابزار خاص بهتر کار کند، چندین conditional و command پیچیده وارد .bashrc کنید، ممکن است بعدها همان configuration با shell دیگری، SSH، script یا environment متفاوت تداخل ایجاد کند.

#### Readability

startup file باید برای شخص دیگری نیز قابل فهم باشد.

بنابراین commentها اهمیت زیادی دارند.

مثلاً بهتر است این:

```
PATH="$HOME/bin:$PATH"
```

با توضیح مناسب همراه شود:

```
# Put user-installed programs ahead of system commands.
PATH="$HOME/bin:$PATH"
```

مخصوصاً در فایل‌هایی که قرار است default چندین user باشند، comment مناسب بخش مهمی از نگهداری فایل محسوب می‌شود.

پیام این بخش ساده است:

> startup file باید آن‌قدر ساده باشد که خراب کردنش سخت و اصلاح کردنش آسان باشد.

کتاب این دو اصل، یعنی Simplicity و Readability را مهم‌ترین راهنمای طراحی startup file برای کاربران دیگر می‌داند.

---

### 13.2 When to Alter Startup Files

قبل از اینکه یک startup file را تغییر دهیم، باید از خودمان بپرسیم:

> آیا واقعاً لازم است اینجا را تغییر بدهم؟

بعضی تغییرات کاملاً منطقی هستند، مثل:

- تغییر prompt
- افزودن یک software محلی ضروری
- رفع یک startup file خراب

اما نباید صرفاً برای راحتی شخصی، configuration پیش‌فرض سیستم را بدون دلیل دستکاری کرد.

کتاب در مورد نرم‌افزارهای محلی یک توصیه‌ی مهم دارد:

> قبل از اینکه configuration عمومی startup را تغییر دهید، wrapper script را بررسی کنید.

فرض کنید یک برنامه‌ی خاص به argument یا environment ویژه‌ای نیاز دارد. به جای اینکه .bashrc را پر از شرط‌های مربوط به آن برنامه کنید، می‌توانید یک wrapper بسازید که همان برنامه را با تنظیمات لازم اجرا کند.

این روش باعث می‌شود تغییر مورد نظر در همان جایی باقی بماند که واقعاً به آن نیاز دارد.

همچنین startup fileهای توزیع Linux ممکن است با فایل‌های دیگری در:

```
/etc
```

تعامل داشته باشند. بنابراین تغییر یک فایل به‌ظاهر ساده ممکن است روی رفتارهای دیگری نیز اثر بگذارد.

در نتیجه، وقتی defaultهای distribution درست کار می‌کنند، تغییر آن‌ها باید با دقت انجام شود.

---

### 13.3 Shell Startup File Elements

حالا کتاب وارد اجزای اصلی یک shell startup file می‌شود.

مهم‌ترین مواردی که بررسی می‌شوند عبارت‌اند از:

- Command Path
- Manual Page Path
- Prompt
- Aliases
- Permissions Mask

هرکدام از این موارد بخشی از environment کاری shell را می‌سازند.

#### 13.3.1 The Command Path

یکی از مهم‌ترین environment variableهای shell:

```
PATH
```

است.

PATH فهرستی از directoryهاست که shell هنگام اجرای یک command در آن‌ها به دنبال executable می‌گردد.

مثلاً وقتی می‌نویسیم:

```
ls
```

shell باید بفهمد فایل اجرایی ls کجاست.

یکی از directoryهای موجود در PATH پیدا می‌شود و executable مناسب اجرا می‌شود.

یک PATH معمولی در سیستم Linux می‌تواند شامل این مسیرها باشد:

```
/usr/local/bin
/usr/bin
/bin
```

کتاب این ترتیب را عمداً مهم می‌داند:

```
/usr/local/bin
/usr/bin
/bin
```

چرا؟

چون /usr/local/bin معمولاً محلی برای برنامه‌های نصب‌شده یا customized شده توسط administrator است.

بنابراین اگر نسخه‌ای از یک برنامه در:

```
/usr/local/bin
```

قرار داشته باشد و نسخه‌ی دیگری در:

```
/usr/bin
```

وجود داشته باشد، نسخه‌ی local می‌تواند اول پیدا شود.

این یک روش ساده برای override کردن بعضی برنامه‌های عمومی بدون تغییر فایل‌های packaged سیستم است.

##### The Difference Between /usr/local/bin and /usr/bin

به‌صورت مفهومی:

```
/usr/bin
```

بیشتر محل executableهای نصب‌شده توسط سیستم package management است.

در مقابل:

```
/usr/local/bin
```

محل رایجی برای softwareهای local و administrator-managed است.

به همین دلیل ترتیب آن‌ها در PATH مهم است.

##### Programs Located in Special Paths

همه‌ی executableها لزوماً در همین سه directory قرار ندارند.

مثلاً در بعضی سیستم‌ها:

```
/usr/games
```

برای commandهای مربوط به بازی وجود دارد.

برنامه‌های عمومی system utility نیز ممکن است در مسیرهای sbin قرار داشته باشند:

```
/usr/local/sbin
/usr/sbin
/sbin
```

اما کتاب توصیه می‌کند PATH را برای هر نصب نرم‌افزاری بی‌دلیل بزرگ نکنید.

اگر یک software محل نصب مخصوص خودش را دارد، در بسیاری از موارد می‌توان به‌جای افزودن دائمی آن directory به PATH، یک symbolic link در:

```
/usr/local/bin
```

ایجاد کرد.

این کار باعث می‌شود PATH به یک فهرست بسیار طولانی از محل‌های مختلف تبدیل نشود.

##### $HOME/bin

بعضی کاربران directory شخصی خودشان را برای scriptها و برنامه‌های محلی ایجاد می‌کنند:

```
$HOME/bin
```

و آن را در ابتدای PATH قرار می‌دهند:

```
PATH="$HOME/bin:$PATH"
```

در این حالت اگر فایلی با نام یک command سیستم در $HOME/bin داشته باشید، نسخه‌ی موجود در home ابتدا پیدا می‌شود.

کتاب همچنین به convention جدیدتری اشاره می‌کند:

```
$HOME/.local/bin
```

بنابراین می‌توانید برنامه‌های شخصی را در:

```
~/.local/bin
```

قرار دهید.

**یک مثال مناسب**

```
PATH="$HOME/.local/bin:$HOME/bin:/usr/local/bin:/usr/bin:/bin"
export PATH
```

این فقط یک نمونه است؛ PATH واقعی هر سیستم ممکن است به configuration distribution، package manager و نیاز کاربر وابسته باشد.

##### Adding a Dot to the PATH

یکی از مواردی که باید در PATH با آن بسیار محتاط بود:

```
.
```

است.

قرار دادن:

```
.
```

در PATH یعنی shell می‌تواند executableهای directory فعلی را بدون نوشتن ./ پیدا کند.

مثلاً اگر فایل اجرایی:

```
ls
```

در directory فعلی باشد، shell ممکن است آن را پیدا کند.

در ظاهر این ویژگی راحت است، اما مشکل امنیتی جدی ایجاد می‌کند.

فرض کنید یک archive از اینترنت دریافت کرده‌اید و داخل آن فایلی با نام:

```
ls
```

وجود دارد.

اگر . در PATH باشد و shell در آن directory باشد، ممکن است به‌جای /bin/ls فایل موجود در directory فعلی اجرا شود.

حتی قرار دادن . در انتهای PATH نیز کاملاً بی‌خطر نیست.

چرا؟

چون کاربر ممکن است command را اشتباه تایپ کند.

مثلاً به‌جای:

```
ls
```

بنویسد:

```
sl
```

اگر فایل maliciousای با همین نام در directory فعلی وجود داشته باشد، shell ممکن است آن را پیدا کند.

بنابراین قاعده‌ی عملی این است:

> . را در PATH قرار ندهید.

برای اجرای برنامه‌ای از directory فعلی، صراحتاً بنویسید:

```
./program
```

این کار باعث می‌شود تصمیم اجرای executable از directory فعلی کاملاً آشکار باشد.

کتاب دو مشکل را برای . مطرح می‌کند:

- Security
- Inconsistency

چون وجود . باعث می‌شود معنی یک command وابسته به directory فعلی تغییر کند.

#### 13.3.2 The Manual Page Path

دومین path مهم در محیط shell مربوط به manual pageهاست:

```
MANPATH
```

در سیستم‌های Unix سنتی، MANPATH برای مشخص کردن محل manual pageها استفاده می‌شده است.

اما در Linux مدرن معمولاً نباید بی‌دلیل آن را به‌صورت دستی تنظیم کنید.

دلیل این است که سیستم می‌تواند defaultهای لازم را خودش از configurationهایی مانند:

```
/etc/manpath.config
```

به دست آورد.

اگر MANPATH را به‌صورت دستی تنظیم کنید، ممکن است منطق default سیستم را override کنید.

در نتیجه اگر هدف شما فقط این است که manual pageهای معمول سیستم درست پیدا شوند، بهتر است به مکانیزم پیش‌فرض اعتماد کنید.

اگر به directoryهای اضافی برای man pageها نیاز دارید، باید دقیقاً بدانید که چگونه آن‌ها را به configuration سیستم اضافه می‌کنید و چرا.

#### 13.3.3 The Prompt

prompt چیزی است که قبل از هر command در shell می‌بینیم.

در Bash این prompt معمولاً از:

```
PS1
```

می‌آید.

یک prompt خوب الزاماً prompt بزرگی نیست.

کتاب پیشنهاد می‌کند prompt تا جای ممکن ساده باشد.

قرار دادن اطلاعاتی مانند:

- username
- hostname
- current directory
- history number

گاهی مفید است، اما اضافه کردن همه‌چیز صرفاً برای اینکه prompt «پیشرفته‌تر» به نظر برسد، لزوماً کار مفیدی نیست.

مثلاً prompt بسیار شلوغ می‌تواند:

```
user@hostname:~/very/long/path/project [git:main] [python:3.12] [venv] $
```

باشد، اما این اطلاعات اگر دائماً لازم نباشند فقط فضای terminal را مصرف می‌کنند.

##### Special Characters

هنگام طراحی prompt باید مراقب characterهایی باشید که shell معنای خاصی برایشان دارد.

کتاب به‌خصوص درباره‌ی این characterها هشدار می‌دهد:

```
{ } = & < >
```

یکی از مهم‌ترین آن‌ها:

```
>
```

است.

چون > در shell برای redirection استفاده می‌شود.

اگر کاربری بخشی از terminal را اشتباه copy/paste کند و این character به‌شکل command وارد shell شود، ممکن است فایل ناخواسته ایجاد یا truncate شود.

##### A Simple Prompt in Bash

نمونه‌ای که کتاب پیشنهاد می‌کند:

```
PS1='\u\$ '
```

در این expression:

```
\u
```

نام کاربر فعلی را قرار می‌دهد.

و:

```
\$
```

برای user معمولی:

```
$
```

و برای root:

```
#
```

نمایش می‌دهد.

##### Some Important Escapes in Bash

```
\u    username
\h    hostname کوتاه
\!    history number
\w    current working directory
\W    آخرین component مسیر
\$    $ برای user و # برای root
```

مثلاً:

```
PS1='\u@\h:\W\$ '
```

ممکن است چیزی شبیه این نشان دهد:

```
saeid@server:project$
```

این نوع prompt معمولاً اطلاعات مفیدی دارد، اما هنوز نسبتاً ساده باقی می‌ماند.

#### 13.3.4 Aliases

alias یک قابلیت shell است که اجازه می‌دهد shell قبل از اجرای command یک string را با string دیگری جایگزین کند.

مثلاً:

```
alias ll='ls -l'
```

حالا:

```
ll
```

به‌صورت تقریبی به:

```
ls -l
```

تبدیل می‌شود.

alias برای shortcutهای ساده مفید است.

اما مشکلاتی نیز دارد.

##### Problem One: Arguments

alias برای مدیریت پیچیده‌ی argumentها مناسب نیست.

مثلاً اگر alias بخواهد رفتار command را بر اساس ورودی تغییر دهد، خیلی زود از حالت ساده خارج می‌شود.

در چنین شرایطی یک shell function گزینه‌ی مناسب‌تری است.

##### Problem Two: Finding Where It's Defined

ممکن است بدانید:

```
which ll
```

اما ابزارهایی مثل which همیشه به شما نمی‌گویند alias دقیقاً در کدام startup file تعریف شده است.

برای دیدن خود alias در Bash بهتر است از:

```
type ll
```

یا:

```
alias ll
```

استفاده کنید.

##### Problem Three: Subshells and Noninteractive Shells

aliasها مثل environment variableها به‌صورت عمومی به child processها export نمی‌شوند.

به همین دلیل نباید روی این فرض حساب کنید که aliasهای interactive شما در shellهای دیگر یا shell scriptها نیز وجود خواهند داشت.

مثلاً:

```
alias rm='rm -i'
```

ممکن است در terminal شما کار کند، اما نباید انتظار داشته باشید یک script مستقل نیز همین alias را ببیند.

##### Aliasing Standard Commands

یکی از خطاهای کلاسیک این است:

```
alias ls='ls -F'
```

این تغییر شاید کوچک به نظر برسد، اما حالا کاربری که نمی‌داند alias وجود دارد، تصور می‌کند دارد ls معمولی را اجرا می‌کند.

این مسئله زمانی بدتر می‌شود که alias argumentهایی را اضافه یا حذف کند و رفتار command را پنهان کند.

به همین دلیل کتاب توصیه می‌کند تا حد امکان برای رفتارهای پیچیده‌تر از:

```
shell function
```

یا:

```
script
```

استفاده شود.

##### When Does an Alias Make Sense?

alias هنوز برای shortcutهای خیلی ساده خوب است.

مثلاً:

```
alias ll='ls -l'
alias la='ls -la'
```

چنین aliasهایی رفتار پیچیده‌ای ندارند.

از طرف دیگر، اگر هدف تغییر environment variable باشد، shell script معمولی گزینه‌ی مناسبی نیست؛ چون script در shell جداگانه اجرا می‌شود و تغییرات environment آن به shell والد برنمی‌گردد.

در چنین مواردی shell function بهتر است.

مثلاً:

```
set_project_env() {
    export PROJECT_ENV=development
}
```

#### 13.3.5 The Permissions Mask

یکی دیگر از اجزای مهم startup environment:

```
umask
```

است.

umask مشخص می‌کند permissionهای پیش‌فرضی که برنامه‌ها هنگام ایجاد file و directory به کار می‌برند چگونه محدود شوند.

به زبان ساده، umask به سیستم می‌گوید:

> چه permissionهایی نباید به‌صورت پیش‌فرض داده شوند؟

دو مقدار مهمی که کتاب بررسی می‌کند:

- 077
- 022

هستند.

##### umask 077

```
umask 077
```

یک انتخاب بسیار restrictive است.

در نتیجه فایل‌ها و directoryهای جدید به‌طور پیش‌فرض برای userهای دیگر قابل دسترسی نخواهند بود.

این برای سیستم چندکاربره‌ای که privacy فایل‌های کاربران اهمیت دارد می‌تواند انتخاب مناسبی باشد.

مشکل اینجاست که بعضی userها ممکن است بعداً بخواهند فایل‌ها را با user دیگری share کنند و permission را درست تنظیم نکنند.

در نتیجه ممکن است به تغییرهای دستی نامناسب و حتی استفاده از permissionهای خطرناک مثل world-writable برسند.

##### umask 022

مقدار:

```
umask 022
```

کمتر restrictive است.

در نتیجه فایل‌های جدید معمولاً طوری ایجاد می‌شوند که دیگر کاربران بتوانند آن‌ها را بخوانند، البته permission نهایی به default mode خود برنامه نیز وابسته است.

این حالت در بعضی محیط‌ها مفید است؛ مثلاً ممکن است daemonهایی که با pseudo-userهای مختلف اجرا می‌شوند بتوانند فایل‌های عمومی‌تر را ببینند.

کتاب در اینجا یک نکته‌ی جالب هم می‌گوید:

بعضی applicationها خودشان umask را override می‌کنند.

مثلاً برنامه‌های mail ممکن است برای جلوگیری از افشای داده‌های خصوصی، خودشان به 077 تغییر کنند.

بنابراین umask یک default است، نه تضمینی مطلق که هیچ برنامه‌ای آن را تغییر نمی‌دهد.

---

### 13.4 Startup File Order and Examples

تا اینجا فهمیدیم داخل startup file چه چیزهایی قرار می‌گیرند.

اما سؤال مهم بعدی این است:

> کدام startup file را باید تغییر بدهیم؟

این قسمت از سخت‌ترین بخش‌های موضوع است، چون shellها startup fileهای مختلفی دارند و نوع shell نیز مهم است.

دو shell اصلی بررسی‌شده در فصل:

- bash
- tcsh

هستند.

#### 13.4.1 The bash Shell

Bash می‌تواند با فایل‌هایی مانند:

- .bash_profile
- .profile
- .bash_login
- .bashrc

کار کند.

اما قبل از اینکه ترتیبشان را بررسی کنیم باید دو distinction مهم را بشناسیم:

- interactive / noninteractive
- login / non-login

این دو مفهوم با هم یکی نیستند.

##### Interactive Shell

shellی که برای تعامل مستقیم با کاربر استفاده می‌شود interactive است.

مثلاً:

- terminal
- SSH session
- console shell

در این shell معمولاً کاربر command تایپ می‌کند.

##### Noninteractive Shell

shellهایی که برای اجرای scriptها یا کارهای غیرتعاملی اجرا می‌شوند، معمولاً noninteractive هستند.

برای مثال:

```
bash script.sh
```

معمولاً به startup fileهای interactive وابسته نیست.

این distinction مهم است، چون اگر در .bashrc یک command خاص قرار دهید، نباید فرض کنید آن command هنگام اجرای هر shell script نیز اجرا می‌شود.

کتاب نیز صراحتاً می‌گوید noninteractive shellها معمولاً startup fileهای مورد بحث این فصل را نمی‌خوانند.

##### Login Shells

login shell در اصل shell اولیه‌ای است که هنگام login کاربر ایجاد می‌شود.

مثلاً:

- login from console
- SSH login

می‌تواند به login shell منتهی شود.

برای تشخیص login shell در Bash می‌توان:

```
echo "$0"
```

را اجرا کرد.

اگر اولین character خروجی - باشد، Bash آن shell را login shell در نظر می‌گیرد.

مثلاً:

```
-bash
```

نشان‌دهنده‌ی login shell است.

##### Order of Reading the Files

وقتی Bash به‌صورت login shell اجرا شود، ابتدا:

```
/etc/profile
```

را می‌خواند.

سپس در home directory کاربر به‌ترتیب دنبال:

```
.bash_profile
.bash_login
.profile
```

می‌گردد.

نکته‌ی مهم این است که Bash اولین فایلی را که وجود داشته باشد اجرا می‌کند و به سراغ بقیه نمی‌رود.

یعنی اگر:

```
.bash_profile
```

وجود داشته باشد، وجود هم‌زمان:

```
.bash_login
.profile
```

به این معنی نیست که هر سه اجرا می‌شوند.

##### Non-Login Shells

یک non-login shell، interactive shellای است که login shell نیست.

ترمینال‌هایی مانند:

- xterm
- GNOME Terminal

معمولاً چنین shellهایی را ایجاد می‌کنند، مگر اینکه به‌صورت login shell پیکربندی شده باشند.

در Bash، non-login interactive shell معمولاً ابتدا تنظیمات global مربوط به Bash و سپس فایل:

```
~/.bashrc
```

را می‌خواند.

در سیستم‌هایی که فایل global زیر وجود دارد:

```
/etc/bash.bashrc
```

ممکن است این فایل نیز در startup یک non-login Bash خوانده شود.

اما باید توجه داشت که نام و وجود این فایل global می‌تواند به distribution وابسته باشد.

##### Why Are Login and Non-Login Separate?

این تفاوت ریشه‌ی تاریخی دارد.

در Unix قدیمی، کاربر معمولاً ابتدا روی یک terminal واقعی login می‌کرد و سپس در همان session shellهای دیگری ایجاد می‌کرد.

منطقی نبود هر بار که یک subshell باز می‌شود، تمام تنظیمات محیط دوباره محاسبه و commandهای سنگین اجرا شوند.

به همین دلیل به‌تدریج یک الگوی رایج شکل گرفت:

```
login shell
→ environment initialization

non-login shell
→ interactive settings
```

برای همین بعضی کاربران:

```
.bash_profile
```

را محل setup اولیه و:

```
.bashrc
```

را محل alias و تنظیمات interactive قرار می‌دادند.

اما desktopهای مدرن همیشه دقیقاً با مدل قدیمی login console کار نمی‌کنند.

Display manager ممکن است login گرافیکی را انجام دهد و سپس session و terminalها را به‌شکل متفاوتی آغاز کند.

به همین دلیل یک environment ممکن است در SSH وجود داشته باشد ولی داخل terminal desktop وجود نداشته باشد، یا برعکس، اگر startup fileها درست به هم متصل نشده باشند.

کتاب دقیقاً به همین مسئله اشاره می‌کند و می‌گوید در بعضی desktop environmentها ممکن است لازم باشد setupهای اصلی environment در .bashrc نیز در دسترس باشند، در حالی که برای console یا remote login وجود یک .bash_profile همچنان مهم است.

##### Building a Shared .bashrc

یکی از روش‌های تمیز این است که configuration اصلی را در:

```
.bashrc
```

قرار دهید.

مثلاً:

```
# Command path
PATH="$HOME/bin:/usr/local/bin:/usr/bin:/bin"

# Prompt
PS1='\u\$ '

# Preferred editor
EDITOR=vi
VISUAL=vi

# Pager
PAGER=less

# Less options
LESS=meiX

# Environment variables
export PATH EDITOR VISUAL PAGER LESS

# Default permissions
umask 022
```

در این مدل، .bash_profile می‌تواند فقط .bashrc را source کند:

```
. "$HOME/.bashrc"
```

یا:

```
source "$HOME/.bashrc"
```

این کار باعث می‌شود configuration مشترک فقط در یک محل نگهداری شود.

##### Why Does Sourcing Matter?

وقتی می‌نویسیم:

```
. "$HOME/.bashrc"
```

محتوای .bashrc در همان shell اجرا می‌شود.

این با اجرای:

```
"$HOME/.bashrc"
```

به‌عنوان یک executable مستقل یکسان نیست.

در حالت source، تغییر environment به shell فعلی برمی‌گردد.

##### Checking Whether a Shell Is Interactive

گاهی می‌خواهید بخشی از startup فقط در interactive shell اجرا شود.

Bash variable ویژه‌ای دارد:

```
$-
```

اگر character:

```
i
```

در آن وجود داشته باشد، shell interactive است.

یک الگوی کلاسیک:

```
case $- in
    *i*)
        # interactive shell settings
        ;;
    *)
        # non-interactive shell
        ;;
esac
```

این روش به شما اجازه می‌دهد commandهایی که مخصوص terminal هستند، در shell scriptهای غیرinteractive اجرا نشوند.

##### A Practical Pattern for .bash_profile

می‌توان .bash_profile را بسیار کوچک نگه داشت:

```
if [ -f "$HOME/.bashrc" ]; then
    . "$HOME/.bashrc"
fi
```

هدف این است که:

```
login shell
   ↓
.bash_profile
   ↓
.bashrc
   ↓
shared interactive environment
```

ایجاد شود.

این روش از duplicate کردن PATH، EDITOR، PAGER، umask و prompt در چند فایل جلوگیری می‌کند.

#### 13.4.2 The tcsh Shell

دومین shell بررسی‌شده در کتاب:

```
tcsh
```

است.

tcsh یک نسخه‌ی توسعه‌یافته از csh است و قابلیت‌هایی مانند:

- command-line editing
- filename completion
- command completion

را ارائه می‌کند.

ساختار startup آن با Bash متفاوت است.

در tcsh، فایل اصلی:

```
.tcshrc
```

است.

اگر این فایل وجود نداشته باشد، shell می‌تواند سراغ:

```
.cshrc
```

برود.

ترتیب این fallback به این دلیل است که .tcshrc می‌تواند قابلیت‌های خاص tcsh را در خود داشته باشد که ممکن است در csh وجود نداشته باشند.

با این حال، کتاب توصیه می‌کند برای configuration عمومی ترجیحاً .cshrc سنتی حفظ شود، چون احتمال اینکه فایل در یک سیستم دیگر با csh نیز استفاده شود وجود دارد.

نکته‌ی دیگر این است که distinction پیچیده‌ی login و non-login در tcsh به همان شکل Bash وجود ندارد و startup اصلی از .tcshrc یا .cshrc انجام می‌شود.

##### An Example .cshrc

ساختار یک configuration ساده می‌تواند چیزی شبیه این باشد:

```
setenv PATH $HOME/bin:/usr/local/bin:/usr/bin:/bin

setenv EDITOR vi
setenv VISUAL vi
setenv PAGER less
setenv LESS meiX

umask 022

set prompt="%m%% "
```

در tcsh/csh syntax با Bash فرق می‌کند.

مثلاً به‌جای:

```
export EDITOR=vi
```

از:

```
setenv EDITOR vi
```

استفاده می‌شود.

همچنین prompt escapeها نیز متفاوت‌اند؛ از جمله:

```
%n    username
%m    hostname
%/    current directory
%h    history number
%l    current terminal
%%    %
```

مثلاً:

```
set prompt="%m%% "
```

می‌تواند hostname را همراه % نمایش دهد.

---

### 13.5 Default User Settings

وقتی administrator می‌خواهد startup fileهای default را برای userهای جدید طراحی کند، تست کردن مستقیم روی account واقعی خودش روش خوبی نیست.

دلیل واضح است:

home directory واقعی معمولاً پر از configurationهای قبلی است.

برای همین کتاب یک روش بسیار بهتر پیشنهاد می‌کند:

```
ایجاد یک test user
+
home directory خالی
+
ساخت startup file از صفر
```

#### The Right Way to Test

ابتدا یک user آزمایشی با home خالی ایجاد کنید.

سپس startup fileهای شخصی خودتان را به او copy نکنید.

به‌جای آن، configuration را از ابتدا بسازید.

بعد تست کنید:

- console login
- SSH login
- interactive shell
- non-login shell
- terminal window
- manual pages
- graphical environment

وقتی configuration در همه‌ی این حالت‌ها درست کار کرد، می‌توان یک test user دوم ساخت و startup fileهای test user اول را روی آن آزمایش کرد.

این مرحله کمک می‌کند مطمئن شوید configuration فقط به environment شخصی user اول وابسته نیست.

کتاب دقیقاً پیشنهاد می‌کند setup را روی یک user تازه آزمایش کنید و سپس با user دوم نیز copy و test کنید تا configuration برای توزیع به کاربران دیگر آماده‌تر شود.

#### 13.5.1 Shell Defaults

کتاب Bash را به‌عنوان default shell پیشنهادی برای userهای جدید در Linux مطرح می‌کند.

دلایلی که ارائه می‌شود شامل این موارد است:

- Bash روی بیشتر Linux distributionها default است
- Bash همان shellای است که بسیاری از userها برای scriptها نیز استفاده می‌کنند
- Bash از GNU readline استفاده می‌کند
- Bash امکانات مناسب و قابل‌فهمی برای redirection و file descriptors دارد

استفاده از یک shell یکسان برای interactive work و shell scripting یک مزیت آموزشی و عملی محسوب می‌شود.

البته userهای باتجربه ممکن است از:

- csh
- tcsh
- ksh
- zsh
- fish

یا shellهای دیگر استفاده کنند.

اما اگر user preference خاصی ندارد، کتاب پیشنهاد می‌کند Bash default باشد.

این به معنای ممنوع بودن shellهای دیگر نیست؛ user می‌تواند shell مورد علاقه‌ی خود را انتخاب کند.

برای تغییر shell login نیز می‌توان از:

```
chsh
```

استفاده کرد، البته بسته به policy سیستم.

کتاب در نسخه‌ی سوم همچنین اشاره می‌کند که shellهایی مانند zsh و fish در میان بعضی کاربران محبوب‌اند و انتخاب shell ultimately به نیاز و ترجیح کاربر بستگی دارد.

#### 13.5.2 Editor

editor پیش‌فرض یکی دیگر از اجزای user environment است.

در سیستم‌های Unix قدیمی، دو نام بسیار شناخته‌شده:

- vi
- emacs

بودند.

دلیل اهمیت آن‌ها این است که احتمال وجود داشتنشان در سیستم‌های Unix بسیار زیاد بوده است.

از طرف دیگر، در بسیاری از Linux distributionها:

```
nano
```

برای userهای تازه‌کار انتخاب ساده‌تری است.

در اینجا هم همان اصل قبلی برقرار است:

> default configuration را بیش از حد شخصی و پیچیده نکنید.

اگر قرار است startup یا editor configuration را برای userهای زیادی تهیه کنید، نباید editor را با تنظیمات سنگین و عجیب تغییر دهید.

مثلاً تغییرهای کوچک قابل‌قبول‌اند، اما تغییر قابل‌توجه رفتار editor ممکن است user را غافلگیر کند.

پس بهتر است:

```
Default editor
+
Minimal configuration
```

داشته باشیم.

کتاب حتی درباره‌ی تنظیمات editor نیز تأکید می‌کند که configuration پیش‌فرض باید تا حد امکان سبک باشد و تغییرهایی که رفتار یا ظاهر editor را به‌شدت عوض می‌کنند برای default user environment مناسب نیستند.

#### 13.5.3 Pager

بعضی commandها output طولانی تولید می‌کنند.

مثلاً:

```
man bash
```

یا commandهایی که log یا متن طولانی نمایش می‌دهند.

برای این کار از pager استفاده می‌شود.

یکی از معروف‌ترین pagerها:

```
less
```

است.

می‌توان environment variable مربوط به pager را چنین تنظیم کرد:

```
PAGER=less
export PAGER
```

در این حالت برنامه‌هایی که از PAGER استفاده می‌کنند، می‌توانند خروجی طولانی خود را به less بسپارند.

این یکی از defaultهای ساده و منطقی برای user environment است.

---

### 13.6 Startup File Pitfalls

این بخش یکی از مهم‌ترین قسمت‌های فصل است.

startup file در محیط‌های مختلف اجرا می‌شود و بنابراین نباید فرض کنیم همیشه شرایط یکسانی وجود دارد.

کتاب چند اشتباه مهم را مشخص می‌کند.

#### Running Graphical Commands in Startup

نباید commandهای graphical را به‌صورت عمومی در startup file یک shell قرار دهید.

ممکن است shell در محیطی اجرا شود که اصلاً graphical session ندارد.

مثلاً:

- SSH session
- console
- cron
- script
- container

در چنین محیطی اجرای یک graphical command ممکن است fail شود یا startup را مختل کند.

بنابراین startup عمومی shell باید تا حد امکان مستقل از GUI باشد.

#### Manually Setting DISPLAY

قرار دادن:

```
DISPLAY=...
```

در startup file ایده‌ی خوبی نیست.

مقدار DISPLAY به graphical session وابسته است و display manager یا محیط گرافیکی ممکن است مقدار مناسب را خودش تعیین کند.

اگر مقدار را به‌صورت hard-code در startup file قرار دهید، ممکن است sessionهای گرافیکی مختلف با آن تداخل پیدا کنند.

کتاب نیز صراحتاً می‌گوید DISPLAY را در shell startup file تنظیم نکنید، زیرا می‌تواند graphical session را به‌هم بزند.

#### Manually Setting Terminal Type

نباید به‌صورت عمومی فرض کنید:

```
terminal type
```

همیشه یک مقدار ثابت دارد.

تنظیمات مربوط به terminal باید با محیط واقعی session هماهنگ باشند.

قرار دادن مقدار ثابت در startup file می‌تواند باعث شود shell یک terminal را اشتباه تشخیص دهد.

این مسئله مخصوصاً در محیط‌هایی مثل:

- SSH
- console
- terminal emulator

که ممکن است شرایط متفاوتی داشته باشند اهمیت دارد.

#### Insufficient Comments

یک startup file بدون comment مناسب بعدها بسیار سخت‌تر قابل نگهداری است.

به‌خصوص اگر user جدید یا administrator دیگری بخواهد آن را اصلاح کند.

مثلاً:

```
PATH="$HOME/bin:$PATH"
```

به‌تنهایی کاملاً واضح نیست که چرا $HOME/bin در ابتدای PATH قرار گرفته است.

اما:

```
# Prefer programs and scripts installed in the user's local bin directory.
PATH="$HOME/bin:$PATH"
```

هدف را مشخص می‌کند.

کتاب روی commentهای توصیفی در default startup fileها تأکید ویژه دارد.

#### Producing Output in Startup Files

startup file معمولاً نباید commandهایی اجرا کند که به‌صورت ناخواسته روی:

```
stdout
```

خروجی تولید می‌کنند.

مثلاً اگر در .bashrc چیزی مانند:

```
echo "Welcome!"
```

قرار دهید، ممکن است هنگام SSH، script یا ابزارهایی که انتظار output تمیز دارند، رفتار نامطلوب ایجاد کند.

در یک interactive terminal شاید این message بد به نظر نرسد، اما startup file ممکن است در contextهای دیگری نیز source شود.

بنابراین بهتر است commandهای startup فقط configuration مورد نیاز را انجام دهند و از output غیرضروری خودداری شود.

#### LD_LIBRARY_PATH

کتاب هشدار بسیار مهمی درباره‌ی:

```
LD_LIBRARY_PATH
```

می‌دهد:

> این variable را به‌صورت عمومی در shell startup file قرار ندهید.

LD_LIBRARY_PATH روی نحوه‌ی پیدا کردن shared libraryها اثر می‌گذارد.

در نتیجه اگر آن را به‌صورت global در environment کاربر قرار دهید، ممکن است تعداد زیادی از برنامه‌ها به‌جای libraryهای سیستم، libraryهای دیگری را بارگذاری کنند.

این موضوع می‌تواند باعث:

- incompatibility
- unexpected behavior
- application failure
- security problems

شود.

بنابراین اگر یک برنامه به library خاصی نیاز دارد، بهتر است configuration آن برنامه را در همان محدوده‌ی مورد نیازش نگه دارید، نه اینکه environment تمام shellهای کاربر را تغییر دهید.

کتاب این مورد را در کنار اجرای graphical command، تنظیم DISPLAY، terminal type، نبودن comment و commandهای دارای stdout output به‌عنوان یکی از pitfalls اصلی startup file مطرح می‌کند.

---

### 13.7 Further Startup Topics

فصل سیزدهم عمدتاً روی shell startup تمرکز دارد و وارد جزئیات کامل startup محیط‌های graphical نمی‌شود.

در Linux مدرن، login گرافیکی نیز خودش ممکن است مجموعه‌ای از startup mechanismها را داشته باشد.

نمونه‌هایی که کتاب ذکر می‌کند:

- .xsession
- .xinitrc

و همچنین مجموعه‌ای از configurationهای مربوط به محیط‌هایی مانند:

- GNOME
- KDE

این بخش بسیار بزرگ و وابسته به نوع desktop environment است.

برخلاف shell startup، یک روش واحد که در تمام Linux desktopها برای graphical session startup استفاده شود وجود ندارد.

بنابراین هدف این فصل بررسی کامل GUI startup نیست.

اما اصل قبلی اینجا هم صادق است:

- ساده
- واضح
- قابل پیش‌بینی

بودن بهتر از این است که startup environment را با ده‌ها script و شرط مختلف پیچیده کنیم.

به‌خصوص در مورد default userها، بهتر است تا جای ممکن configuration گرافیکی را نیز دستکاری نکنیم، مگر اینکه واقعاً ضرورتی وجود داشته باشد.

کتاب در پایان فصل تأکید می‌کند که startup environment گرافیکی حوزه‌ی بزرگی است و روش راه‌اندازی آن در Linux یکدست نیست؛ بنابراین همان اصل سادگی که در shell startup اهمیت دارد، در GUI startup نیز مفید است.

---

### How the Pieces of the Startup Environment Fit Together

اگر بخواهیم تمام فصل را کنار هم قرار دهیم، هنگام شروع یک Bash تعاملی، چیزی شبیه این اتفاق می‌افتد:

```
Login / Terminal
      ↓
Shell starts
      ↓
Startup files
      ↓
Environment
      ├── PATH
      ├── EDITOR
      ├── VISUAL
      ├── PAGER
      ├── LESS
      ├── PS1
      └── umask
      ↓
Interactive shell
      ↓
User commands
```

اما بسته به نوع shell و نوع login، startup file متفاوت است.

برای Bash:

```
                 Bash
                  │
        ┌─────────┴─────────┐
        │                   │
     Login              Non-login
        │                   │
 /etc/profile         global bashrc
        │                   │
.bash_profile         ~/.bashrc
.bash_login
.profile
```

این مدل دقیقاً به همین دلیل اهمیت دارد که یک configuration ممکن است در یک نوع shell اعمال شود ولی در نوع دیگر نه.

---

### A Simple, Clean Bash Environment Example

یک configuration مینیمال می‌تواند به این صورت باشد:

```
# User-local programs first.
PATH="$HOME/.local/bin:$HOME/bin:/usr/local/bin:/usr/bin:/bin"

# Simple prompt.
PS1='\u@\h:\W\$ '

# Default editor.
EDITOR=vi
VISUAL=vi

# Default pager.
PAGER=less

# Less behavior.
LESS=meiX

# Export environment variables.
export PATH EDITOR VISUAL PAGER LESS

# Default file permissions.
umask 022
```

و .bash_profile می‌تواند فقط مسئول load کردن .bashrc باشد:

```
if [ -f "$HOME/.bashrc" ]; then
    . "$HOME/.bashrc"
fi
```

این مثال عمداً ساده است.

قرار نیست همه‌ی امکانات Bash را وارد startup کنیم.

هدف این است که environment مشخص باشد و بدانیم هر خط چرا وجود دارد.

---

### Practical Tips from the Chapter

#### 🔹 Don't Make PATH Unnecessarily Long

به‌جای اینکه برای هر software یک directory جدید اضافه کنید، از ساختارهایی مثل:

```
$HOME/.local/bin
$HOME/bin
/usr/local/bin
```

به‌صورت منطقی استفاده کنید.

#### 🔹 Don't Put . in PATH

برای اجرای executable از directory فعلی صریحاً بنویسید:

```
./program
```

نه:

```
program
```

با فرض اینکه . در PATH قرار دارد.

#### 🔹 Don't Mix Up .bashrc and .bash_profile

login و non-login shell رفتار یکسانی ندارند.

اگر configuration مشترکی دارید، یک محل مشخص برای آن تعیین کنید و فایل دیگر را به آن source کنید.

#### 🔹 Keep Startup Files Scoped to the Real Environment

SSH، terminal، console و GUI ممکن است شرایط متفاوتی داشته باشند.

به همین دلیل مواردی مثل:

- DISPLAY
- terminal type
- graphical commands

نباید بی‌دلیل در startup عمومی shell قرار بگیرند.

#### 🔹 Keep Aliases for Simple Tasks

برای logic پیچیده:

- function
- script
- wrapper

معمولاً انتخاب بهتری است.

#### 🔹 Set umask According to System Needs

077 و 022 هرکدام کاربرد متفاوتی دارند و انتخاب آن‌ها باید بر اساس نیاز اشتراک فایل و حریم خصوصی سیستم انجام شود.

#### 🔹 Startup Files Are Not the Place for System-Wide Configuration

هر خطی که در startup قرار می‌دهید روی processهای بیشتری اثر می‌گذارد.

بنابراین بهتر است configuration برنامه را در خود برنامه، wrapper یا مکان مناسب دیگری قرار دهید، نه اینکه همه‌چیز را وارد .bashrc کنید.

---

### Summary

فصل سیزدهم توضیح می‌دهد که startup fileها محیط کاری shell را می‌سازند. PATH، prompt، aliasها، umask و variableهایی مثل EDITOR و PAGER از مهم‌ترین اجزای این محیط هستند.

در Bash باید تفاوت login، non-login، interactive و noninteractive را بشناسیم و startup file مناسب هرکدام را درست مدیریت کنیم.

اصل کلیدی فصل این است:

> startup environment را ساده، خوانا و قابل پیش‌بینی نگه دارید و فقط تنظیماتی را وارد آن کنید که واقعاً لازم‌اند.
