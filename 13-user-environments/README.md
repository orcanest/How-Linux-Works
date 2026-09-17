## محیط کاربر
### 🐧 فصل سیزدهم کتاب How Linux Works


فصل سیزدهم با موضوع User Environments به نقطه‌ ای می‌ رسد که بین زیرساخت Linux و محیطی که کاربر هر روز با آن کار می‌ کند ارتباط برقرار می‌ شود. تا اینجا بیشتر درباره‌ ی کرنل ، process ها ، filesystem ، شبکه ، service ها و shell صحبت شده بود اما در نهایت کاربر باید بتواند یک shell را باز کند و در یک محیط کاری مشخص شروع به کار کند.

این محیط از هیچ به وجود نمی‌ آید. وقتی shell اجرا می‌ شود ، تعدادی تنظیمات از فایل‌ های مختلف خوانده می‌ شوند و چیزهایی مثل PATH ، prompt ، alias ها ، editor ، pager ، umask و دیگر environment variable ها شکل می‌ گیرند.

بخش زیادی از این تنظیمات در فایل‌ های موجود در home directory کاربر قرار دارند. این فایل‌ ها معمولاً نامشان با '.' شروع می‌ شود به همین دلیل به آن‌ ها dot files گفته می‌ شود برای مثال :

- .bashrc
- .bash_profile
- .profile
- .cshrc

این فایل‌ ها هیچ ویژگی جادویی خاصی به خاطر نقطه‌ ی ابتدای نامشان ندارند فقط بسیاری از ابزارها و file manager ها آن‌ ها را به‌ صورت پیش‌ فرض نمایش نمی‌ دهند تا home directory بیش از حد شلوغ نشود.

نکته‌ ی مهم فصل این است که startup file ها اگر به‌ مرور زمان بی‌ دلیل بزرگ و پیچیده شوند ، می‌ توانند منبع مشکلات جدی شوند. یک environment ممکن است در یک shell درست کار کند ولی در login از طریق SSH ، در یک terminal emulator یا در یک shell غیر interactive  رفتار متفاوتی داشته باشد.

---
📚 Table of Contents

- [Guidelines for Creating Startup Files](#guidelines-for-creating-startup-files)
- [When to Alter Startup Files](#when-to-alter-startup-files)
- [Shell Startup File Elements](#shell-startup-file-elements)
- [Startup File Order and Examples](#startup-file-order-and-examples)
- [Default User Settings](#default-user-settings)
- [Startup File Pitfalls](#startup-file-pitfalls)
- [Further Startup Topics](#further-startup-topics)
- [How the Pieces of the Startup Environment Fit Together](#how-the-pieces-of-the-startup-environment-fit-together)
- [A Simple, Clean Bash Environment Example](#a-simple-clean-bash-environment-example)
- [Practical Tips from the Chapter](#practical-tips-from-the-chapter)
- [Summary](#summary)

---

### Guidelines for Creating Startup Files

اولین نکته‌ ای که هنگام ساخت startup file باید در نظر بگیریم این است که این فایل برای چه کسی نوشته می‌ شود. اگر startup file فقط برای خودمان باشد ، اشتباه موجود در آن مستقیماً فقط روی خودمان اثر می‌ گذارد. اما اگر این فایل قرار است default یک سیستم باشد یا برای چندین user استفاده شود ، شرایط کاملاً فرق می‌ کند. مثلاً تصور کنید یک startup file را برای 10 تا user نصب کرده‌ اید و بعد متوجه می‌ شوید یک خط آن اشتباه بوده است. حالا همان خطا باید در 10 تا environment مختلف اصلاح شود بنابراین کتاب روی دو اصل اساسی تأکید می‌ کند : 

#### 🔹 Simplicity

باید startup file را تا حد امکان کوتاه ، ساده ، قابل فهم و قابل تغییر نگه دارید. هر command ، variable ، شرط یا dependency که به آن اضافه می‌ کنید ، یک نقطه‌ ی بالقوه برای failure است. در عمل ، startup file جای مناسبی برای قرار دادن هر configuration ممکن نیست. مثلاً اگر فقط برای اینکه یک ابزار خاص بهتر کار کند ، چندین conditional و command پیچیده وارد .bashrc کنید ،  ممکن است بعد ها همان configuration با shell دیگری ، SSH ، script یا environment متفاوت تداخل ایجاد کند.

#### 🔹 Readability

باید startup file برای شخص دیگری نیز قابل فهم باشد بنابراین comment ها اهمیت زیادی دارند مثلاً بهتر است این :

```
PATH="$HOME/bin:$PATH"
```

با توضیح مناسب همراه شود :

```
# Put user-installed programs ahead of system commands.
PATH="$HOME/bin:$PATH"
```

مخصوصاً در فایل‌ هایی که قرار است default چندین user باشند ، comment مناسب بخش مهمی از نگهداری فایل محسوب می‌شود . پیام این بخش ساده است :

> باید startup file آن‌ قدر ساده باشد که خراب کردنش سخت و اصلاح کردنش آسان باشد.

کتاب این دو اصل ، یعنی Simplicity و Readability را مهم‌ ترین راهنمای طراحی startup file برای کاربران دیگر می‌ داند.

---

### When to Alter Startup Files

قبل از اینکه یک startup file را تغییر دهیم ، باید از خودمان بپرسیم آیا واقعاً لازم است اینجا را تغییر بدهم ؟ بعضی تغییرات کاملاً منطقی هستند ، مثل تغییر prompt ، افزودن یک software محلی ضروری و رفع یک startup file خراب . اما نباید صرفاً برای راحتی شخصی ، configuration پیش‌ فرض سیستم را بدون دلیل دستکاری کرد. کتاب در مورد نرم‌ افزارهای local یک توصیه‌ ی مهم دارد اینکه قبل از اینکه configuration عمومی startup را تغییر دهید ، wrapper script را بررسی کنید. فرض کنید یک برنامه‌ ی خاص به argument یا environment ویژه‌ ای نیاز دارد به جای اینکه bashrc. را پر از شرط‌ های مربوط به آن برنامه کنید ، می‌ توانید یک wrapper بسازید که همان برنامه را با تنظیمات لازم اجرا کند. این روش باعث می‌ شود تغییر مورد نظر در همان جایی باقی بماند که واقعاً به آن نیاز دارد. همچنین startup file های توزیع Linux ممکن است با فایل‌ های دیگری در```etc/``` ، تعامل داشته باشند بنابراین تغییر یک فایل به‌ ظاهر ساده ممکن است روی رفتارهای دیگری نیز اثر بگذارد در نتیجه ، وقتی default های distribution درست کار می‌ کنند ، تغییر آن‌ ها باید با دقت انجام شود.

---

### Shell Startup File Elements

حالا کتاب وارد اجزای اصلی یک shell startup file می‌ شود. مهم‌ ترین مواردی که بررسی می‌ شوند هرکدام از این موارد بخشی از environment کاری shell را می‌ سازند عبارت‌اند از:

- Command Path
- Manual Page Path
- Prompt
- Aliases
- Permissions Mask

#### 🔹 The Command Path

یکی از مهم‌ ترین environment variable های shell هم ```PATH``` است. PATH فهرستی از directory هاست که shell هنگام اجرای یک command در آن‌ ها به دنبال executable می‌ گردد. مثلاً وقتی می‌ نویسیم ```ls``` باید shell بفهمد فایل اجرایی ls کجاست. یکی از directory های موجود در PATH پیدا می‌ شود و executable مناسب اجرا می‌ شود. یک PATH معمولی در سیستم Linux می‌ تواند شامل این مسیرها باشد و کتاب این ترتیب را عمداً مهم می‌ داند :

```
/usr/local/bin
/usr/bin
/bin
```

چرا ؟ چون /usr/local/bin معمولاً محلی برای برنامه‌ های نصب‌ شده یا customized شده توسط administrator است. بنابراین اگر نسخه‌ای از یک برنامه در ```/usr/local/bin``` قرار داشته باشد و نسخه‌ ی دیگری در ```/usr/bin``` وجود داشته باشد ، نسخه‌ ی local می‌ تواند اول پیدا شود. این یک روش ساده برای override کردن بعضی برنامه‌ های عمومی بدون تغییر فایل‌ های packaged سیستم است.

به‌ صورت مفهومی ```/usr/bin``` بیشتر محل executable های نصب‌ شده توسط سیستم package management است. در مقابل ```/usr/local/bin``` محل رایجی برای software های local و administrator-managed است به همین دلیل ترتیب آن‌ ها در PATH مهم است.


##### 🔹 Programs Located in Special Paths

همه‌ ی executable ها لزوماً در همین سه directory قرار ندارند مثلاً در بعضی سیستم‌ ها ```/usr/games``` برای command های مربوط به بازی وجود دارد. برنامه‌ های عمومی system utility نیز ممکن است در مسیرهای sbin قرار داشته باشند :

```
/usr/local/sbin
/usr/sbin
/sbin
```

اما کتاب توصیه می‌ کند PATH را برای هر نصب نرم‌ افزاری بی‌ دلیل بزرگ نکنید. اگر یک software محل نصب مخصوص خودش را دارد ، در بسیاری از موارد می‌ توان به‌ جای افزودن دائمی آن directory به PATH ، یک symbolic link در```/usr/local/bin``` ایجاد کرد. این کار باعث می‌ شود PATH به یک فهرست بسیار طولانی از محل‌ های مختلف تبدیل نشود.

##### 🔹 $HOME/bin

بعضی کاربران directory شخصی خودشان را برای script ها و برنامه‌ های local ایجاد می‌ کنند مثلاً ```HOME/bin$```  و آن را در ابتدای PATH قرار می‌ دهند ```PATH="$HOME/bin:$PATH"``` . در این حالت اگر فایلی با نام یک command سیستم در HOME/bin$ داشته باشید ، نسخه‌ ی موجود در home ابتدا پیدا می‌ شود. کتاب همچنین به convention جدید تری اشاره می‌ کند ، ```HOME/.local/bin$```. بنابراین می‌ توانید برنامه‌ های شخصی را در ```local/bin/~.``` قرار دهید.

**یک مثال مناسب** : 

```
PATH="$HOME/.local/bin:$HOME/bin:/usr/local/bin:/usr/bin:/bin"
export PATH
```

این فقط یک نمونه است و PATH واقعی هر سیستم ممکن است به configuration distribution ، package manager و نیاز کاربر وابسته باشد.

##### 🔹 Adding a Dot to the PATH

یکی از مواردی که باید در PATH با آن بسیار محتاط بود ```.``` است. قرار دادن ```.``` در PATH یعنی shell می‌ تواند executable های directory فعلی را بدون نوشتن /. پیدا کند. مثلاً اگر فایل اجرایی ```ls``` در directory فعلی باشد ، shell ممکن است آن را پیدا کند. در ظاهر این ویژگی راحت است ، اما مشکل امنیتی جدی ایجاد می‌ کند. فرض کنید یک archive از اینترنت دریافت کرده‌ اید و داخل آن فایلی با نام ```ls``` وجود دارد. اگر ```.``` در PATH باشد و shell در آن directory باشد ، ممکن است به‌ جای bin/ls/ فایل موجود در directory فعلی اجرا شود. حتی قرار دادن ```.``` در انتهای PATH نیز کاملاً بی‌خطر نیست. چرا؟ چون کاربر ممکن است command را اشتباه تایپ کند مثلاً به‌ جای ```ls ``` بنویسد ```sl``` ، اگر فایل malicious با همین نام در directory فعلی وجود داشته باشد ، shell ممکن است آن را پیدا کند.

بنابراین قاعده‌ ی عملی این است که ```.```  را در PATH قرار ندهید و برای اجرای برنامه‌ ای ازcurrent directory ،  صراحتاً بنویسید ```program/.``` . این کار باعث می‌ شود تصمیم اجرای executable از current directory کاملاً آشکار باشد.

کتاب دو مشکل را برای ```.``` مطرح می‌ کند ، Security و Inconsistency . چون وجود ```.``` باعث می‌ شود معنی یک command وابسته به directory فعلی تغییر کند.

#### 🔹 The Manual Page Path

دومین path مهم در محیط shell مربوط به manual page هاست ، ```MANPATH``` . در سیستم‌ های Unix قدیمی ، MANPATH برای مشخص کردن محل manual page ها استفاده می‌ شده است. اما در Linux مدرن معمولاً نباید بی‌ دلیل آن را به‌ صورت دستی تنظیم کنید. دلیل این است که سیستم می‌ تواند default های لازم را خودش از configuration هایی مانند ```etc/manpath.config/``` به دست آورد. اگر MANPATH را به‌ صورت دستی تنظیم کنید ، ممکن است منطق default سیستم را override کنید. در نتیجه اگر هدف شما فقط این است که manual page های معمول سیستم درست پیدا شوند ، بهتر است به مکانیزم پیش‌ فرض اعتماد کنید. اگر به directory های اضافی برای man page ها نیاز دارید ، باید دقیقاً بدانید که چگونه آن‌ ها را به configuration سیستم اضافه می‌ کنید و چرا.

#### 🔹 The Prompt

همچنین prompt چیزی است که قبل از هر command در shell می‌ بینیم. در Bash این prompt معمولاً از ```PS1``` می‌ آید. یک prompt خوب الزاماً prompt بزرگی نیست. کتاب پیشنهاد می‌ کند prompt تا جای ممکن ساده باشد و قرار دادن اطلاعاتی مانند username و hostname و current directory و history number گاهی مفید است ، اما اضافه کردن همه‌چیز صرفاً برای اینکه prompt پیشرفته‌ تر به نظر برسد ، لزوماً کار مفیدی نیست. مثلاً prompt بسیار شلوغ می‌ تواند :

```
user@hostname:~/very/long/path/project [git:main] [python:3.12] [venv] $
```

باشد ، اما این اطلاعات اگر دائماً لازم نباشند فقط  فضای terminal را مصرف می‌ کنند.

##### 🔹 Special Characters

هنگام طراحی prompt باید مراقب character هایی باشید که shell معنا ی خاصی برایشان دارد. کتاب به‌ خصوص درباره‌ ی این character ها هشدار می‌ دهد : ```{ } = & < >```. یکی از مهم‌ ترین آن‌ ها ```>``` است چون > در shell برای redirection استفاده می‌ شود. اگر کاربری بخشی از terminal را اشتباه copy/paste کند و این character به‌ شکل command وارد shell شود ، ممکن است فایل نا خواسته ایجاد یا truncate شود.

##### 🔹 Some Important Escapes in Bash

در Shell ، متغیر PS1 مشخص می‌ کند Prompt چگونه نمایش داده شود. می‌ توان با استفاده از escape sequence های مختلف ، اطلاعاتی مانند نام کاربر، hostname و مسیر فعلی را در Prompt نمایش داد :

```
- \u username
- \h short hostname
- \! history number
- \w current working directory
- \W last path component
- \$ $ for user and # for root
```

برای مثال :

```PS1='\u@\h:\W\$ '```

ممکن است Prompt به شکل زیر نمایش داده شود :

```Alex@server:project$```

در این مثال نام کاربر، hostname کوتاه ، آخرین بخش مسیر فعلی و نوع دسترسی کاربر در Prompt نمایش داده می‌ شود. این نوع Prompt اطلاعات مفیدی در اختیار کاربر قرار می‌ دهد ، اما همچنان نسبتاً ساده و خوانا باقی می‌ ماند.

#### 🔹 Aliases

یک قابلیت shell است که اجازه می‌ دهد shell قبل از اجرای command یک string را با string دیگری جایگزین کند مثلاً : 

```alias ll='ls -l'```

حالا ```ll``` به‌ صورت تقریبی به ```ls -l``` تبدیل می‌ شود. alias برای shortcut های ساده مفید است اما مشکلاتی نیز دارد.

##### 🔹 Problem One: Arguments

همچنین alias برای مدیریت پیچیده‌ی argument ها مناسب نیست. مثلاً اگر alias بخواهد رفتار command را بر اساس ورودی تغییر دهد ، خیلی زود از حالت ساده خارج می‌ شود در چنین شرایطی یک shell function گزینه‌ ی مناسب‌ تری است.

##### 🔹 Problem Two: Finding Where It's Defined

ممکن است بدانید ```which ll``` چه چیزی است ، اما ابزارهایی مثل which همیشه به شما نمی‌ گویند alias دقیقاً در کدام startup file تعریف شده است. برای دیدن خود alias در Bash بهتر است از ```type ll``` یا ```alias ll``` استفاده کنید.

##### 🔹 Problem Three: Subshells and Noninteractive Shells

معمولا alias ها مثل environment variable ها به‌ صورت عمومی به child process ها export نمی‌ شوند. به همین دلیل نباید روی این فرض حساب کنید که alias های interactive شما در shell های دیگر یا shell script ها نیز وجود خواهند داشت مثلاً :

```alias rm='rm -i'```

ممکن است در terminal شما کار کند ، اما نباید انتظار داشته باشید یک script مستقل نیز همین alias را ببیند.

##### 🔹 Aliasing Standard Commands

یکی از خطاهای کلاسیک این است که : 

```alias ls='ls -F'```

این تغییر شاید کوچک به نظر برسد ، اما حالا کاربری که نمی‌ داند alias وجود دارد ، تصور می‌ کند دارد ls معمولی را اجرا می‌ کند. این مسئله زمانی بدتر می‌ شود که alias argument هایی را اضافه یا حذف کند و رفتار command را پنهان کند. به همین دلیل کتاب توصیه می‌ کند تا حد امکان برای رفتارهای پیچیده‌ تر از shell function یا script استفاده شود.

##### 🔹 When Does an Alias Make Sense ?

همچنین alias هنوز برای shortcut های خیلی ساده خوب است مثلاً : 

```
alias ll='ls -l'
alias la='ls -la'
```

از طرف دیگر، اگر هدف تغییر environment variable باشد ، shell script معمولی گزینه‌ ی مناسبی نیست چون script در shell جداگانه اجرا می‌ شود و تغییرات environment آن به shell والد برنمی‌ گردد در چنین مواردی shell function بهتر است مثلاً :

```
set_project_env() {
    export PROJECT_ENV=development
}
```

#### 🔹 The Permissions Mask

یکی دیگر از اجزای مهم startup environment هم ```umask``` است. umask مشخص می‌ کند permission های پیش‌ فرضی که برنامه‌ ها هنگام ایجاد file و directory به کار می‌ برند چگونه محدود شوند. به زبان ساده ، umask به سیستم می‌ گوید ، چه permission هایی نباید به‌ صورت پیش‌ فرض داده شوند ؟ . دو مقدار مهمی که کتاب بررسی می‌ کند ، 077 و 022 هستند.


##### 🔹 umask 077

مقدار ```umask 077``` یک انتخاب بسیار restrictive است در نتیجه فایل‌ ها و directory های جدید به‌ طور پیش‌ فرض برای user های دیگر قابل دسترسی نخواهند بود. این برای سیستم چند کاربره‌ای که privacy فایل‌ های کاربران اهمیت دارد می‌ تواند انتخاب مناسبی باشد. مشکل اینجاست که بعضی user ها ممکن است بعداً بخواهند فایل‌ ها را با user دیگری share کنند و permission را درست تنظیم نکنند در نتیجه ممکن است به تغییرهای دستی نامناسب و حتی استفاده از permission های خطرناک مثل world-writable برسند.

##### 🔹 umask 022

مقدار ```umask 022``` کمتر restrictive است در نتیجه فایل‌ های جدید معمولاً طوری ایجاد می‌ شوند که دیگر کاربران بتوانند آن‌ ها را بخوانند ، البته permission نهایی به default mode خود برنامه نیز وابسته است. این حالت در بعضی محیط‌ ها مفید است. مثلاً ممکن است daemon هایی که با pseudo-user های مختلف اجرا می‌ شوند بتوانند فایل‌ های عمومی‌ تر را ببینند. کتاب در اینجا یک نکته‌ ی جالب هم می‌ گوید ، بعضی application ها خودشان umask را override می‌ کنند. مثلاً برنامه‌ های mail ممکن است برای جلوگیری از افشای داده‌ های خصوصی ، خودشان به 077 تغییر کنند بنابراین umask یک default است ، نه تضمینی مطلق که هیچ برنامه‌ ای آن را تغییر نمی‌ دهد.

---

### Startup File Order and Examples

تا اینجا فهمیدیم داخل startup file چه چیزهایی قرار می‌ گیرند اما سؤال مهم بعدی این است که کدام startup file را باید تغییر بدهیم ؟ ، این قسمت از سخت‌ ترین بخش‌ های موضوع است ، چون shell ها startup file های مختلفی دارند و نوع shell نیز مهم است. دو shell اصلی بررسی‌ شده در فصل bash و tcsh هستند.

#### 🔹 The bash Shell

با Bash می‌ توان با فایل‌ هایی کار کرد مانند :

- .bash_profile
- .profile
- .bash_login
- .bashrc


اما قبل از اینکه ترتیبشان را بررسی کنیم باید دو distinction مهم را بشناسیم :

- interactive / noninteractive
- login / non-login

این دو مفهوم با هم یکی نیستند.

##### 🔹 Interactive Shell

یک shell که برای تعامل مستقیم با کاربر استفاده می‌ شود interactive است ، در این shell معمولاً کاربر command تایپ می‌ کند مثلاً :

- terminal
- SSH session
- console shell

##### 🔹 Noninteractive Shell

معمولا shell هایی که برای اجرای script ها یا کارهای غیر تعاملی اجرا می‌ شوند noninteractive هستند مثلاً ```bash script.sh``` معمولاً به startup file های interactive وابسته نیست. این distinction مهم است ، چون اگر در bashrc. یک command خاص قرار دهید ، نباید فرض کنید آن command هنگام اجرای هر shell script نیز اجرا می‌ شود. کتاب نیز صراحتاً می‌گوید noninteractive shell ها معمولاً startup file های مورد بحث این فصل را نمی‌ خوانند.

##### 🔹 Login Shells

معمولا login shell در اصل shell اولیه‌ ای است که هنگام login کاربر ایجاد می‌ شود مثلاً login from console و  SSH login می‌ تواند به login shell منتهی شود. برای تشخیص login shell در Bash می‌ توان ```echo "$0"``` را اجرا کرد. اگر اولین character خروجی - باشد ، Bash آن shell را login shell در نظر می‌ گیرد مثلاً ```bash-``` نشان‌ دهنده‌ ی login shell است.

##### 🔹 Order of Reading the Files

وقتی Bash به‌ صورت login shell اجرا شود ، ابتدا ```etc/profile/``` را می‌ خواند سپس در home directory کاربر به‌ ترتیب دنبال :

```
.bash_profile
.bash_login
.profile
```

می‌ گردد. نکته‌ ی مهم این است که Bash اولین فایلی را که وجود داشته باشد اجرا می‌ کند و به سراغ بقیه نمی‌ رود یعنی اگر ```bash_profile.```  وجود داشته باشد ، وجود هم‌ زمان ```bash_login.``` و ```profile.``` به این معنی نیست که هر سه اجرا می‌ شوند.

##### 🔹 Non-Login Shells

یک non-login shell ، interactive shell است که login shell نیست ترمینال‌ هایی مانند xterm و GNOME Terminal ، معمولاً چنین shell هایی را ایجاد می‌ کنند ، مگر اینکه به‌ صورت login shell پیکربندی شده باشند. در Bash ،  non-login interactive shell معمولاً ابتدا تنظیمات global مربوط به Bash و سپس فایل bashrc/.~ را می‌ خواند.

در سیستم‌ هایی که فایل global زیر وجود دارد ```etc/bash.bashrc/``` ممکن است این فایل نیز در startup یک non-login Bash خوانده شود اما باید توجه داشت که نام و وجود این فایل global می‌ تواند به distribution وابسته باشد.

##### 🔹 Why Are Login and Non-Login Separate?

این تفاوت ریشه‌ ی تاریخی دارد در Unix قدیمی ، کاربر معمولاً ابتدا روی یک terminal واقعی login می‌ کرد و سپس در همان session shell های دیگری ایجاد می‌ کرد. منطقی نبود هر بار که یک subshell باز می‌ شود ، تمام تنظیمات محیط دوباره محاسبه و command های سنگین اجرا شوند. به همین دلیل به‌ تدریج یک الگوی رایج شکل گرفت :

```
login shell
→ environment initialization

non-login shell
→ interactive settings
```

برای همین بعضی کاربران :

```
.bash_profile
```

را محل setup اولیه و :

```
.bashrc
```

را محل alias و تنظیمات interactive قرار می‌ دادند اما desktop های مدرن همیشه دقیقاً با مدل قدیمی login console کار نمی‌ کنند. display manager ممکن است login گرافیکی را انجام دهد و سپس session و terminal ها را به‌ شکل متفاوتی آغاز کند. به همین دلیل یک environment ممکن است در SSH وجود داشته باشد ولی داخل terminal desktop وجود نداشته باشد ، یا برعکس ، اگر startup file ها درست به هم متصل نشده باشند. کتاب دقیقاً به همین مسئله اشاره می‌ کند و می‌ گوید در بعضی desktop environment ها ممکن است لازم باشد setup های اصلی environment در bashrc. نیز در دسترس باشند ، در حالی که برای console یا remote login وجود یک bash_profile. همچنان مهم است.

##### 🔹 Building a Shared .bashrc

یکی از روش‌ های تمیز این است که configuration اصلی را در ```bashrc.``` قرار دهید مثلاً : 

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

در این مدل ، bash_profile. می‌ تواند فقط bashrc. را source کند :

```
. "$HOME/.bashrc"
```

یا:

```
source "$HOME/.bashrc"
```

این کار باعث می‌ شود configuration مشترک فقط در یک محل نگهداری شود.

##### 🔹 Why Does Sourcing Matter?

وقتی می‌ نویسیم :

``` . "$HOME/.bashrc" ```

محتوای bashrc. در همان shell اجرا می‌ شود. این با اجرای :

``` "$HOME/.bashrc" ```

به‌ عنوان یک executable مستقل یکسان نیست در حالت source ، تغییر environment به shell فعلی برمی‌ گردد.

##### 🔹 Checking Whether a Shell Is Interactive

گاهی می‌ خواهید بخشی از startup فقط در interactive shell اجرا شود و Bash variable ویژه‌ ای دارد ```-$``` ، اگر کاراکتر ```i``` در آن وجود داشته باشد، shell interactive است ، یک الگوی کلاسیک :

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
این روش به شما اجازه می‌ دهد command هایی که مخصوص terminal هستند ، در shell script های غیر interactive  اجرا نشوند.

##### 🔹 A Practical Pattern for .bash_profile

می‌ توان bash_profile. را بسیار کوچک نگه داشت :

```
if [ -f "$HOME/.bashrc" ]; then
    . "$HOME/.bashrc"
fi
```

هدف این است که :

```
login shell
   ↓
.bash_profile
   ↓
.bashrc
   ↓
shared interactive environment
```
ایجاد شود. این روش از duplicate کردن PATH ، EDITOR ، PAGER ، umask و prompt در چند فایل جلوگیری می‌ کند.

#### 🔹 The tcsh Shell

دومین shell بررسی‌ شده در کتاب  ```tcsh``` است. tcsh یک نسخه‌ ی توسعه‌ یافته از csh است و قابلیت‌ هایی مانند command-line editing و filename completion و command completion را ارائه می‌ کند. ساختار startup آن با Bash متفاوت است. در tcsh ، فایل اصلی ```tcshrc.``` است. اگر این فایل وجود نداشته باشد ، shell می‌ تواند سراغ ```cshrc.``` برود.
ترتیب این fallback به این دلیل است که tcshrc. می‌ تواند قابلیت‌های خاص tcsh را در خود داشته باشد که ممکن است در csh وجود نداشته باشند. با این حال ، کتاب توصیه می‌ کند برای configuration عمومی ترجیحاً cshrc. حفظ شود ، چون احتمال اینکه فایل در یک سیستم دیگر با csh نیز استفاده شود وجود دارد. نکته‌ ی دیگر این است که distinction پیچیده‌ی login و non-login در tcsh به همان شکل Bash وجود ندارد و startup اصلی از tcshrc. یا cshrc. انجام می‌ شود.

##### 🔹 An Example .cshrc

ساختار یک configuration ساده می‌ تواند چیزی شبیه این باشد :

```
setenv PATH $HOME/bin:/usr/local/bin:/usr/bin:/bin

setenv EDITOR vi
setenv VISUAL vi
setenv PAGER less
setenv LESS meiX

umask 022

set prompt="%m%% "
```

در tcsh/csh syntax با Bash فرق می‌ کند مثلاً به‌ جای :

```
export EDITOR=vi
```

از:

```
setenv EDITOR vi
```

استفاده می‌ شود ، همچنین prompt escape ها نیز متفاوت‌اند از جمله :

```
%n    username
%m    hostname
%/    current directory
%h    history number
%l    current terminal
%%    %
```

مثلاً :

```
set prompt="%m%% "
```

می‌ تواند hostname را همراه % نمایش دهد.

---

### Default User Settings

وقتی administrator می‌ خواهد startup file های default را برای user های جدید طراحی کند ، تست کردن مستقیم روی account واقعی خودش روش خوبی نیست. دلیل واضح است چون home directory واقعی معمولاً پر از configuration های قبلی است برای همین کتاب یک روش بسیار بهتر پیشنهاد می‌ کند :

``` ایجاد یک test user + خالی home directory + ساخت startup file از صفر ```

#### 🔹 The Right Way to Test

ابتدا یک user آزمایشی با home خالی ایجاد کنید سپس startup file های شخصی خودتان را به او copy نکنید به‌ جای آن ، configuration را از ابتدا بسازید بعد تست کنید :

- console login
- SSH login
- interactive shell
- non-login shell
- terminal window
- manual pages
- graphical environment

وقتی configuration در همه‌ ی این حالت‌ ها درست کار کرد ، می‌ توان یک test user دوم ساخت و startup file های test user اول را روی آن آزمایش کرد. این مرحله کمک می‌کند مطمئن شوید configuration فقط به environment شخصی user اول وابسته نیست. کتاب دقیقاً پیشنهاد می‌ کند setup را روی یک user تازه آزمایش کنید و سپس با user دوم نیز copy و test کنید تا configuration برای توزیع به کاربران دیگر آماده‌ تر شود.

#### 🔹 Shell Defaults

کتاب Bash را به‌ عنوان default shell پیشنهادی برای user های جدید در Linux مطرح می‌ کند ، معمولاً Bash روی بیشتر Linux distribution ها default است و Bash همان shell است که بسیاری از user ها برای script ها نیز استفاده می‌ کنند و از GNU readline استفاده می‌ کند همچنین امکانات مناسب و قابل‌ فهمی برای redirection و file descriptors دارد.

استفاده از یک shell یکسان برای interactive work و shell scripting یک مزیت آموزشی و عملی محسوب می‌شود. البته  user های باتجربه ممکن است از csh یا tcsh یا ksh یا zsh یا fish یا shell های دیگر استفاده کنند. اما اگر user preference خاصی ندارد ، کتاب پیشنهاد می‌ کند Bash default باشد. این به معنای ممنوع بودن shell های دیگر نیست بلکه user می‌ تواند shell مورد علاقه‌ ی خود را انتخاب کند. برای تغییر shell login نیز می‌ توان از ```chsh``` استفاده کرد ، البته بسته به policy سیستم. کتاب همچنین اشاره می‌ کند که shell هایی مانند zsh و fish در میان بعضی کاربران محبوب‌ اند و انتخاب shell ultimately به نیاز و ترجیح کاربر بستگی دارد.

#### 🔹 Editor

یکی دیگر از اجزای user environment هم default editor است. در سیستم‌ های Unix قدیمی ، دو نام بسیار شناخته‌ شده vi و emacs بودند. دلیل اهمیت آن‌ ها این است که احتمال وجود داشتنشان در سیستم‌ های Unix بسیار زیاد بوده است. از طرف دیگر، در بسیاری از Linux distribution ها ```nano``` ، برای user های تازه‌ کار انتخاب ساده‌ تری است. در اینجا هم همان اصل قبلی برقرار است پس default configuration را بیش از حد شخصی و پیچیده نکنید. اگر قرار است startup یا editor configuration را برای user های زیادی تهیه کنید ، نباید editor را با تنظیمات سنگین و عجیب تغییر دهید. مثلاً تغییرهای کوچک قابل‌ قبول‌ اند ، اما تغییر قابل‌ توجه رفتار editor ممکن است user را غافلگیر کند. پس بهتر است ترکیبی از default editor و Minimal configuration داشته باشیم. کتاب حتی درباره‌ ی تنظیمات editor نیز تأکید می‌ کند که default configuration باید تا حد امکان سبک باشد و تغییر هایی که رفتار یا ظاهر editor را به‌ شدت عوض می‌ کنند برای default user environment مناسب نیستند.

#### 🔹 Pager

بعضی command ها output طولانی تولید می‌ کنند مثلاً ```man bash``` یا command هایی که log یا متن طولانی نمایش می‌ دهند برای این کار از pager استفاده می‌ شود ، یکی از معروف‌ ترین pager ها ```less``` است. می‌ توان environment variable مربوط به pager را چنین تنظیم کرد :

```
PAGER=less
export PAGER
```

در این حالت برنامه‌ هایی که از PAGER استفاده می‌ کنند ، می‌ توانند خروجی طولانی خود را به less بسپارند. این یکی از  default های ساده و منطقی برای user environment است.

---

### Startup File Pitfalls

این بخش یکی از مهم‌ ترین قسمت‌ های فصل است. startup file در محیط‌ های مختلف اجرا می‌ شود و بنابراین نباید فرض کنیم همیشه شرایط یکسانی وجود دارد کتاب چند اشتباه مهم را مشخص می‌ کند.

#### 🔹 Running Graphical Commands in Startup

نباید command های graphical را به‌ صورت عمومی در startup file یک shell قرار دهید. ممکن است shell در محیطی اجرا شود که اصلاً graphical session ندارد مثلاً :

- SSH session
- console
- cron
- script
- container

در چنین محیطی اجرای یک graphical command ممکن است fail شود یا startup را مختل کند بنابراین startup عمومی shell باید تا حد امکان مستقل از GUI باشد.

#### 🔹 Manually Setting DISPLAY

قرار دادن ```DISPLAY``` در startup file ایده‌ ی خوبی نیست. مقدار DISPLAY به graphical session وابسته است و display manager یا محیط گرافیکی ممکن است مقدار مناسب را خودش تعیین کند. اگر مقدار را به‌ صورت hard-code در startup file قرار دهید ، ممکن است session های گرافیکی مختلف با آن تداخل پیدا کنند. کتاب نیز صراحتاً می‌ گوید DISPLAY را در shell startup file تنظیم نکنید ، زیرا می‌ تواند graphical session را به‌ هم بزند.

#### 🔹 Manually Setting Terminal Type

نباید فرض کنید ```terminal type``` همیشه یک مقدار ثابت دارد. تنظیمات مربوط به terminal باید با محیط واقعی session هماهنگ باشند. قرار دادن مقدار ثابت در startup file می‌ تواند باعث شود shell یک terminal را اشتباه تشخیص دهد این مسئله مخصوصاً در محیط‌ هایی مثل SSH یا console یا terminal emulator که ممکن است شرایط متفاوتی داشته باشند اهمیت دارد.

#### 🔹 Insufficient Comments

یک startup file بدون comment مناسب بعد ها بسیار سخت‌ تر قابل نگهداری است. به‌ خصوص اگر user جدید یا administrator دیگری بخواهد آن را اصلاح کند مثلاً :

```
PATH="$HOME/bin:$PATH"
```
به‌ تنهایی کاملاً واضح نیست که چرا HOME/bin$ در ابتدای PATH قرار گرفته است اما :

```
# Prefer programs and scripts installed in the user's local bin directory.
PATH="$HOME/bin:$PATH"
```

هدف را مشخص می‌ کند کتاب روی comment های توصیفی در default startup file ها تأکید ویژه دارد.

#### 🔹 Producing Output in Startup Files

معمولاً startup file نباید command هایی اجرا کند که به‌ صورت نا خواسته روی ```stdout``` خروجی تولید می‌ کنند مثلاً اگر در bashrc. چیزی مانند :

``` echo "Welcome!" ```

قرار دهید ، ممکن است هنگام SSH هم script یا ابزارهایی که انتظار output تمیز دارند ، رفتار نامطلوب ایجاد کند. در یک interactive terminal شاید این message بد به نظر نرسد ، اما startup file ممکن است در context های دیگری نیز source شود بنابراین بهتر است command های startup فقط configuration مورد نیاز را انجام دهند و از output غیر ضروری خود داری شود.

#### 🔹 LD_LIBRARY_PATH

کتاب هشدار بسیار مهمی درباره‌ ی ```LD_LIBRARY_PATH``` می‌ دهد اینکه **این variable را به‌ صورت عمومی در shell startup file قرار ندهید**. LD_LIBRARY_PATH روی نحوه‌ ی پیدا کردن shared library ها اثر می‌گذارد در نتیجه اگر آن را به‌ صورت global در environment کاربر قرار دهید ، ممکن است تعداد زیادی از برنامه‌ ها به‌ جای library های سیستم ، library های دیگری را بارگذاری کنند این موضوع می‌ تواند باعث :

- incompatibility
- unexpected behavior
- application failure
- security problems

شود ، بنابراین اگر یک برنامه به library خاصی نیاز دارد بهتر است configuration آن برنامه را در همان محدود ه‌ی مورد نیازش نگه دارید ، نه اینکه environment تمام shell های کاربر را تغییر دهید. کتاب این مورد را در کنار اجرای graphical command ، تنظیم DISPLAY ، terminal type ، نبودن comment و command های دارای stdout output به‌ عنوان یکی از pitfalls اصلی startup file مطرح می‌ کند.

---

### Further Startup Topics

فصل سیزدهم عمدتاً روی shell startup تمرکز دارد و وارد جزئیات کامل startup محیط‌ های graphical نمی‌ شود. در Linux مدرن ، login گرافیکی نیز خودش ممکن است مجموعه‌ ای از startup mechanism ها را داشته باشد. نمونه‌ هایی که کتاب ذکر می‌ کند :

- .xsession
- .xinitrc

و همچنین مجموعه‌ ای از configuration های مربوط به محیط‌ هایی مانند GNOME و KDE این بخش بسیار بزرگ و وابسته به نوع desktop environment است. برخلاف shell startup ، یک روش واحد که در تمام Linux desktop ها برای graphical session startup استفاده شود وجود ندارد بنابراین هدف این فصل بررسی کامل GUI startup نیست. اما اصل قبلی اینجا هم درست است شامل ساده ، واضح و قابل پیش‌ بینی بودن بهتر از این است که startup environment را با ده‌ ها script و شرط مختلف پیچیده کنیم. به‌خصوص در مورد default user ها ، بهتر است تا جای ممکن configuration گرافیکی را نیز دستکاری نکنیم ، مگر اینکه واقعاً ضرورتی وجود داشته باشد. کتاب در پایان فصل تأکید می‌ کند که startup environment گرافیکی حوزه‌ ی بزرگی است و روش راه‌ اندازی آن در Linux یکدست نیست بنابراین همان اصل سادگی که در shell startup اهمیت دارد ، در GUI startup نیز مفید است.

---

### How the Pieces of the Startup Environment Fit Together

اگر بخواهیم تمام فصل را کنار هم قرار دهیم ، هنگام شروع یک Bash تعاملی ، چیزی شبیه این اتفاق می‌ افتد :

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

اما بسته به نوع shell و نوع login ، startup file متفاوت است ، برای Bash :

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

یک configuration مینیمال می‌ تواند به این صورت باشد :

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

و bash_profile. می‌ تواند فقط مسئول load کردن bashrc. باشد :

```
if [ -f "$HOME/.bashrc" ]; then
    . "$HOME/.bashrc"
fi
```

این مثال عمداً ساده است قرار نیست همه‌ ی امکانات Bash را وارد startup کنیم. هدف این است که environment مشخص باشد و بدانیم هر خط چرا وجود دارد.

---

### Practical Tips from the Chapter

#### 🔹 Don't Make PATH Unnecessarily Long

به‌ جای اینکه برای هر software یک directory جدید اضافه کنید ، از ساختارهایی مثل :

```
$HOME/.local/bin
$HOME/bin
/usr/local/bin
```

به‌ صورت منطقی استفاده کنید.

#### 🔹 Don't Put . in PATH

برای اجرای executable از directory فعلی صریحاً بنویسید ```program/.``` نه ```program``` ، با فرض اینکه '.' در PATH قرار دارد.

#### 🔹 Don't Mix Up .bashrc and .bash_profile

همچنین login و non-login shell رفتار یکسانی ندارند. اگر configuration مشترکی دارید ، یک محل مشخص برای آن تعیین کنید و فایل دیگر را به آن source کنید.

#### 🔹 Keep Startup Files Scoped to the Real Environment

ممکن است SSH ، terminal ، console و GUI شرایط متفاوتی داشته باشند. به همین دلیل مواردی مثل DISPLAY و terminal type و graphical commands ، نباید بی‌ دلیل در startup عمومی shell قرار بگیرند.

#### 🔹 Keep Aliases for Simple Tasks

برای logic پیچیده معمولاً function و script و wrapper انتخاب بهتری است.

#### 🔹 Set umask According to System Needs

077 و 022 هرکدام کاربرد متفاوتی دارند و انتخاب آن‌ ها باید بر اساس نیاز اشتراک فایل و حریم خصوصی سیستم انجام شود.

#### 🔹 Startup Files Are Not the Place for System-Wide Configuration

هر خطی که در startup قرار می‌ دهید روی process های بیشتری اثر می‌ گذارد. بنابراین بهتر است configuration برنامه را در خود برنامه ، wrapper یا مکان مناسب دیگری قرار دهید ، نه اینکه همه‌چیز را وارد bashrc. کنید.
---

### Summary

فصل سیزدهم توضیح می‌ دهد که startup file ها محیط کاری shell را می‌ سازند. PATH ، prompt ، alias ها ، umask و variable هایی مثل EDITOR و PAGER از مهم‌ ترین اجزای این محیط هستند. در Bash باید تفاوت login ، non-login ، interactive و noninteractive را بشناسیم و startup file مناسب هر کدام را درست مدیریت کنیم. اصل کلیدی فصل این است که **startup environment را ساده ، خوانا و قابل پیش‌ بینی نگه دارید و فقط  تنظیماتی را وارد آن کنید که واقعاً لازم‌ اند**.
