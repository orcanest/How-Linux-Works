## مقدمه‌ ای بر کامپایل نرم‌افزار با زبان C
### 🐧 فصل شانزدهم کتاب How Linux Works

این فصل وارد مرحله‌ ای می‌ شود که بعد از نوشتن source code اهمیت پیدا می‌کند : **creation و configuration و installation و maintenance برای software از روی source code**. در Linux بسیاری از نرم‌افزارهای غیرتجاری به‌ صورت source distribution منتشر می‌ شوند و کاربر می‌ تواند آن‌ ها را روی سیستم خودش build و نصب کند. یکی از دلایل این موضوع تنوع زیاد Linux ، معماری‌ های سخت‌ افزاری و تفاوت میان سیستم‌ هاست. توزیع یک binary یکسان برای همه‌ ی ترکیب‌ های ممکن سیستم‌ عامل و architecture دشوار است. علاوه بر این ، انتشار source code باعث می‌ شود کاربران بتوانند bug ها را برطرف کنند ، feature های جدید اضافه کنند و software را متناسب با محیط خودشان بسازند. البته اینکه بتوانیم یک software را از source code نصب کنیم به این معنی نیست که همیشه باید این کار را انجام دهیم. Linux distributions معمولاً روش ساده‌ تر و قابل‌ مدیریت‌ تری برای نصب و به‌ روزرسانی software دارند و برای بخش‌ های مهم سیستم نیز معمولاً security update ها را سریع‌ تر ارائه می‌ کنند. با این حال ، نصب مستقیم از source در بعضی شرایط مزایایی دارد :

- کنترل بیشتر روی configuration و build options
- امکان نصب software در مسیر دلخواه
- امکان داشتن چند version از یک package
- کنترل version دقیق software
- درک بهتر نحوه‌ ی کار package

به همین دلیل این فصل به‌ جای بررسی تمام build system های Linux ، روی یک روش رایج برای **C software** یعنی **GNU Autoconf و GNU Autotools** تمرکز می‌ کند. این روش روی ابزارهایی مانند `make` بنا شده و آشنایی با آن به درک سایر build system ها نیز کمک می‌ کند.

---
📚 Table of Contents

- [Software Build Systems](#software-build-systems)
- [Unpacking C Source Packages](#unpacking-c-source-packages)
- [GNU Autoconf](#gnu-autoconf)
- [Installation Practice](#installation-practice)
- [Applying a Patch](#applying-a-patch)
- [Troubleshooting Compiles and Installations](#troubleshooting-compiles-and-installations)
- [Looking Forward](#looking-forward)
- [Summary](#summary)

---

### Software Build Systems

در Linux فقط C وجود ندارد. از زبان‌ هایی مانند C و C++ گرفته تا scripting language هایی مثل Python ، هر محیط برنامه‌ نویسی ممکن است روش متفاوتی برای build و installation داشته باشد. توزیع‌ های Linux نیز خودشان مجموعه‌ ای از ابزارهای package management در اختیار دارند ، اما در این فصل تمرکز روی فرآیند build یک **C source package** است. فرآیند معمول نصب software از source code را می‌ توان به چهار مرحله‌ ی اصلی تقسیم کرد :

1. باز کردن source archive
2. انجام configure کردن package
3. برنامه‌ ها را build کردن با `make` یا build command مناسب
4. نصب package با `make install` یا ابزار نصب مربوط به همان build system

این فصل بیشتر روی فرآیند مبتنی بر **Autoconf** تمرکز دارد یعنی حالتی که یک package دارای script مانند `configure` است و این script محیط سیستم را بررسی کرده و فایل‌ های build مناسب را تولید می‌ کند. نکته‌ ی مهم این است که قبل از ورود به این فصل ، باید با مفاهیم پایه‌ ی compiler و linker و library و `make` که در فصل ۱۵ مطرح شدند آشنا باشید.

---

### Unpacking C Source Packages

معمولاً source distribution ها به‌ صورت archive هایی با پسوند ```tar.gz. و tar.bz2. و tar.xz.``` هایی زیر ارائه می‌ شوند ، قبل از extract کردن archive بهتر است ابتدا محتویات آن را بررسی کنید. برای این کار می‌ توان از دستور هایی مانند ```tar tvf package.tar``` یا برای archive هایی که با gzip فشرده شده‌ اند با ```tar ztvf package.tar.gz```  استفاده کرد. دلیل این بررسی فقط کنجکاوی نیست. مهم است که بدانید archive بعد از extract شدن چه ساختاری دارد. یک package مناسب معمولاً یک directory مخصوص خودش دارد :

```text
package-1.23/
    Makefile.in
    README
    main.c
    bar.c
    ...
```

در این حالت extract کردن package نسبتاً امن و مرتب است ، چون فایل‌ های package داخل یک directory جدا قرار می‌ گیرند اما ممکن است archive ساختار مناسبی نداشته باشد و فایل‌ هایی مانند این‌ ها را مستقیماً در directory فعلی قرار دهد :


```text
Makefile
README
main.c
...
```

اگر چنین archive را در یک directory شلوغ extract کنید ، فایل‌ های package با فایل‌ های دیگر مخلوط می‌ شوند و پیدا کردن یا پاک کردن آن‌ ها دشوار خواهد شد. بنابراین در چنین شرایطی بهتر است ابتدا یک directory جدید بسازید ، وارد آن شوید و سپس archive را extract کنید. موضوع مهم دیگر **امنیت archive** است. اگر archive شامل pathname های absolute مانند موارد زیر باشد :

```text
/etc/passwd
/etc/inetd.conf
```

باید به آن شدیداً مشکوک باشید. چنین archive نباید به‌ صورت عادی extract شود ، زیرا ممکن است قصد overwrite کردن فایل‌ های سیستم را داشته باشد و احتمال وجود malicious code یا Trojan در آن مطرح است. بعد از extract شدن source ، اولین فایل‌ هایی که باید بررسی شوند `README` و `INSTALL` هستند. فایل `README` معمولاً توضیحی درباره‌ ی package ، نحوه‌ ی استفاده ، نکات installation و اطلاعات عمومی پروژه دارد. فایل `INSTALL` نیز در بسیاری از پروژه‌ ها دستورالعمل دقیق‌ تری برای configure و compile و install ارائه می‌ دهد. هنگام خواندن این فایل‌ ها باید به **special compiler options و definitions و dependency ها و پیش‌نیازهای build** توجه ویژه‌ ای داشته باشید. فایل‌ های داخل source package را می‌ توان به‌ صورت کلی در چند گروه دید. گروه اول فایل‌ های مربوط به build system هستند ، مانند:

```text
Makefile
Makefile.in
configure
CMakeLists.txt
```

در package های قدیمی ممکن است `Makefile` مستقیماً برای ویرایش دستی در اختیار کاربر قرار گرفته باشد. اما در بسیاری از package های جدیدتر، configuration utility مانند **GNU Autoconf** یا **CMake** وجود دارد که براساس تنظیمات سیستم ، فایل build نهایی را تولید می‌ کند ، گروه دوم source file ها هستند :

```text
.c
.h
.cc
.C
.cxx
```

فایل‌ های C ممکن است در هر نقطه‌ای از source tree قرار داشته باشند. در پروژه‌ های ++C نیز suffix های مختلفی برای source file ها دیده می‌ شود. گروه سوم object file ها یا binary های از قبل ساخته‌ شده هستند مانند ```o.``` . در یک source distribution استاندارد معمولاً انتظار نداریم object file از قبل وجود داشته باشد. اگر object file یا executable آماده داخل source package دیده شود ، ممکن است package به‌ درستی آماده نشده باشد. البته در موارد خاص ممکن است maintainer نتواند بخشی از source code را منتشر کند و مجبور شده باشد object file را در package قرار دهد. در حالت معمول ، اگر object file یا binary از قبل وجود دارد بهتر است با ```make clean``` محیط build را پاک کنید تا build از source تازه شروع شود و نتیجه به فایل‌ های قدیمی وابسته نباشد.

---

### GNU Autoconf

در نگاه اول ممکن است به نظر برسد که C آن‌ قدر portable است که یک `Makefile` واحد بتواند روی هر سیستم اجرا شود ، اما در عمل تفاوت‌ های میان platform ها ، library ها ، compiler ها و امکانات موجود باعث می‌ شود چنین کاری دشوار باشد. راه‌حل‌ های قدیمی‌ تر شامل تهیه‌ ی `Makefile` جدا برای هر operating system یا ساختن `Makefile` بود که کاربر مجبور می‌ شد خودش آن را ویرایش کند. Autoconf ایده‌ ی متفاوتی دارد به‌ جای اینکه developer برای هر سیستم یک `Makefile` مستقل بنویسد ، اطلاعات سیستم در زمان configure بررسی می‌ شود و build files بر اساس همان محیط تولید می‌شوند. در package های مبتنی بر GNU Autoconf معمولاً فایل‌ هایی مانند این‌ ها را می‌بینیم :

```text
configure
Makefile.in
config.h.in
```
فایل‌ هایی که با `in.` تمام می‌ شوند template هستند. یعنی هنوز configuration نهایی داخل آن‌ ها قرار نگرفته است. `configure` سیستم را بررسی می‌ کند ، ویژگی‌ ها و prerequisite های لازم را تشخیص می‌ دهد و براساس آن template ها را به فایل‌ های مورد نیاز build تبدیل می‌ کند بنابراین اجرای ```configure/.``` ، فقط یک command ساده برای ساخت `Makefile` نیست بلکه این script مجموعه‌ ای از تست‌ ها را اجرا می‌ کند تا بفهمد سیستم فعلی چه امکاناتی دارد در صورت موفقیت ، معمولاً مواردی مانند این‌ ها تولید می‌ شوند :


```text
Makefile
config.h
config.cache
```

همچنین `config.cache` به configure اجازه می‌ دهد برخی نتیجه‌ ی تست‌ ها را ذخیره کند تا هنگام اجرای بعدی لازم نباشد همه‌ ی تست‌ ها را از ابتدا تکرار کند. نکته‌ ی مهم این است که **موفق بودن `configure` تضمین نمی‌ کند `make` نیز حتماً موفق شود**. configure فقط می‌ تواند بگوید محیط برای build تا حد مورد نیاز آماده به نظر می‌رسد. خطا های واقعی compilation یا linking ممکن است بعداً ظاهر شوند. برای build کردن package نیز باید ابزارهای توسعه روی سیستم وجود داشته باشند. در Debian و Ubuntu ، کتاب به `build-essential` اشاره می‌ کند و در سیستم‌ های Fedora-like استفاده از گروه **Development Tools** را مطرح می‌ کند.

#### 🔹 An Autoconf Example

برای دیدن فرآیند واقعی ، کتاب از **GNU coreutils** به‌ عنوان نمونه استفاده می‌ کند و package را به‌ جای نصب در بخش‌ های سیستمی ، داخل home directory کاربر قرار می‌ دهد. پس از دریافت و extract کردن source ، می‌توان package را با تعیین یک installation prefix اختصاصی configure کرد :

```bash
./configure --prefix=$HOME/mycoreutils
```

در این مرحله `configure` خروجی diagnostic زیادی تولید می‌ کند و بررسی می‌ کند که محیط build چه وضعیتی دارد. بعد از configure ، مرحله‌ ی build انجام می‌ شود با ```make```. اگر build موفق باشد ، می‌ توانید یکی از executable های ساخته‌ شده را مستقیماً از داخل source tree اجرا کنید برای مثال نسخه‌ ی تازه ساخته‌ شده‌ ی `ls` همچنین در این مرحله می‌ توان تست‌ های package را با ```make check``` اجرا کرد. برای بعضی package ها این مرحله مجموعه‌ ای از test ها را اجرا می‌ کند و ممکن است زمان قابل‌ توجهی طول بکشد و پس از آن نوبت installation است. قبل از نصب واقعی ، بهتر است اول یک **dry run** انجام شود با دستور ```make -n install``` که گزینه‌ ی `n-` باعث می‌ شود command هایی که قرار است اجرا شوند نمایش داده شوند اما واقعاً اجرا نشوند. در این مرحله باید خروجی را بررسی کنید و مطمئن شوید فایل‌ ها قرار نیست در مسیر نادرست نصب شوند بعد از بررسی می‌ توان installation واقعی را با ```make install``` انجام داد سپس با توجه به `--prefix=$HOME/mycoreutils`، فایل‌ ها در directory اختصاصی کاربر قرار می‌ گیرند و زیرشاخه‌ هایی مانند این‌ ها ایجاد می‌ شوند :

```text
mycoreutils/
├── bin/
├── share/
└── ...
```

این روش مزیت مهمی دارد اینکه package از بقیه‌ ی سیستم جدا باقی می‌ ماند و در صورت نیاز می‌ توان کل directory را حذف کرد بدون اینکه فایل‌ های اصلی سیستم تحت تأثیر قرار بگیرند.

#### 🔹 Installation Using a Packaging Tool

با دستور `make install` فایل‌ ها را مستقیماً در filesystem قرار می‌ دهد ، اما در بسیاری از Linux distributions روش بهتر این است که نرم‌افزار را به یک **package قابل مدیریت** تبدیل کنیم. در این حالت بعداً می‌ توان package را با package management system توزیع نصب ، حذف یا مدیریت کرد. در سیستم‌ های Debian-based ، کتاب به ابزار `checkinstall` اشاره می‌ کند. به‌ جای اجرای مستقیم ```make install``` می‌توان از ```checkinstall make install``` استفاده کرد. `checkinstall` فایل‌ هایی را که قرار است نصب شوند دنبال می‌ کند و آن‌ ها را داخل یک `deb.` قرار می‌ دهد. نتیجه را می‌ توان بعداً با `dpkg` نصب یا حذف کرد.

 در محیط‌ های RPM-based ، فرآیند کمی پیچیده‌ تر است. ابتدا می‌ توان با ```rpmdev-setuptree``` ، ساختار directory لازم برای package را ایجاد کرد و سپس از ```rpmbuild``` برای ادامه‌ ی فرآیند ساخت RPM استفاده کرد. ساخت RPM معمولاً نیازمند آشنایی بیشتر با package specification و ساختار RPM است ، بنابراین برای جزئیات این workflow باید به documentation و tutorial های مخصوص RPM مراجعه کرد.
 
#### 🔹 configure Script Options

مهم‌ ترین گزینه‌ ی `configure` در بسیاری از packageها `prefix--` است. به‌ صورت پیش‌ فرض ، installation های ساخته‌ شده با Autoconf معمولاً prefix را در مسیر ```usr/local/``` ، در نظر می‌ گیرند بنابراین فایل‌ ها معمولاً به مسیرهایی مانند ```usr/local/bin/ و usr/local/lib/```  می‌ روند.

برای تغییر این مکان می‌ توان از```configure --prefix=/some/path/.``` استفاده کرد. - یکی از مزیت‌ های مهم `configure` این است که معمولاً گزینه‌ ی `help--` را نیز ارائه می‌ دهد مانند ```configure --help/.``` این خروجی می‌ تواند بسیار طولانی باشد ، اما چند option برای شناختن اهمیت بیشتری دارند مثلا : 

- با ```bindir=directory--``` تعیین می‌ کند executable های معمولی در کجا نصب شوند.
- با ```sbindir=directory--``` محل نصب system executable ها را مشخص می‌ کند.
- با ```libdir=directory--``` مسیر نصب library ها را تعیین می‌ کند.
- با ```disable-shared-- ```  باعث می‌ شود package وshared library نسازد. بسته به نوع library ، این گزینه می‌ تواند بعضی مشکلات runtime مربوط به shared library ها را از بین ببرد.
- با ```with-package=directory--``` برای معرفی محل یک package یا dependency که در directory استاندارد قرار ندارد استفاده می‌ شود.

البته همه‌ ی script ها الزاماً این option را با همین نام یا syntax قبول نمی‌ کنند. به همین دلیل باید option های واقعی همان package را از `configure --help/.` یا documentation آن بررسی کرد.

#### 🔹 Separate Build Directories

گاهی لازم است یک package را با configuration های مختلف build کنید یا چند محیط متفاوت را با یک source tree آزمایش کنید. در این شرایط می‌ توان build directory جداگانه ایجاد کرد و `configure` را از آن directory روی source tree اصلی اجرا کرد. مزیت این روش این است که source tree اصلی کمتر دست‌ کاری می‌ شود. همچنین اگر بخواهید یک package را با چند set از configuration options یا حتی برای چند platform مختلف build کنید ، این ساختار بسیار مفید است. در این مدل، build directory می‌ تواند شامل یک مجموعه‌ ی symbolic link باشد که به فایل‌ های source tree اصلی اشاره می‌ کنند. این روش برای آزمایش configuration های مختلف بدون تغییر دادن مستقیم source tree اصلی بسیار مناسب است.

#### 🔹 Environment Variables

می‌ توان `configure` را با **environment variable** ها تحت تأثیر قرار داد. بسیاری از این variable ها در نهایت به `make` منتقل می‌ شوند و روی compilation و linking تأثیر می‌ گذارند. سه variable مهم که در این فصل مورد توجه قرار می‌ گیرند عبارت‌اند از:

```text
CPPFLAGS
CFLAGS
LDFLAGS
```

در اینجا یک تفاوت مهم وجود دارد برای معرفی directory های مربوط به header file ها معمولاً بهتر است از `CPPFLAGS` استفاده شود ، نه `CFLAGS`. دلیل این انتخاب این است که `configure` ممکن است preprocessor را جدا از compiler اجرا کند و در نتیجه flag هایی که باید روی preprocessing اعمال شوند بهتر است در `CPPFLAGS` قرار بگیرند. برای مثال ، اگر بخواهید یک preprocessor macro به نام `DEBUG` تعریف کنید :

```bash
CPPFLAGS=-DDEBUG ./configure
```

یا می‌توانید variable را به‌ عنوان argument به `configure` بدهید :

```bash
./configure CPPFLAGS=-DDEBUG
```

اگر header file ها در directory غیر استاندارد قرار داشته باشند ، می‌ توان مسیر آن‌ ها را این‌ گونه اضافه کرد :

```bash
CPPFLAGS=-Iinclude_dir ./configure
```

و اگر library ها در directory خاصی قرار داشته باشند ، می‌ توان مسیر آن‌ ها را برای linker معرفی کرد :

```bash
LDFLAGS=-Llib_dir ./configure
```

اما در مورد shared library ها مسئله فقط پیدا شدن library در زمان linking نیست. اگر library در مسیر غیر استاندارد باشد ، runtime dynamic linker نیز باید بتواند آن را بعداً پیدا کند. در این حالت می‌ توان علاوه بر `L-` از `rpath-` نیز استفاده کرد :

```bash
LDFLAGS="-Llib_dir -Wl,-rpath=lib_dir" ./configure
```

بنابراین باید بین دو مسئله تفاوت بگذاریم ```L-``` برای کمک به linker در زمان build و ```rpath-``` برای ثبت runtime library search path در executable طبق روش مورد بحث در کتاب ، همچنین باید هنگام نوشتن variable ها بسیار دقت کرد. یک اشتباه کوچک در flag ها می‌تواند باعث شود `configure` نتواند حتی یک برنامه‌ ی آزمایشی ساده را compile کند. برای نمونه ، اگر به‌اشتباه `l-` را به شکل `I` بنویسید :

```bash
CPPFLAGS=Iinclude_dir ./configure
```

ممکن است `configure` در نهایت خطایی مانند ```C compiler cannot create executables``` ایجاد کند در چنین شرایطی خود این پیام علت اصلی مشکل نیست بلکه باید به `config.log` مراجعه کرد تا command واقعی شکست‌ خورده و خطای compiler دیده شود.

#### 🔹 Autoconf Targets

با `Makefile` تولید شده توسط Autoconf فقط target های ساده‌ ای مانند `all` و `install` ندارد. معمولاً target های دیگری هم وجود دارند که برای مدیریت build مفید هستند.

#### 🔹 `make clean`

این target برای پاک کردن خروجی‌ های build استفاده می‌ شود و معمولاً object file ها ، executable ها و library های ساخته‌ شده را حذف می‌ کند.

#### 🔹 `make distclean`

از `clean` گسترده‌ تر است و علاوه بر build artifacts ، فایل‌ هایی را که `configure` تولید کرده است نیز حذف می‌ کند برای مثال :

```text
Makefile
config.h
config.log
...
```

هدف این است که source tree تقریباً به وضعیت یک source distribution تازه extract‌ شده برگردد.

#### 🔹 `make check`

این target test های package را اجرا می‌ کند. همه‌ ی package ها test suite ندارند ، اما در package هایی که چنین تست‌ هایی وجود دارد ، این target برای بررسی صحت build بسیار مفید است.

#### 🔹 `make install-strip`

این target مانند `make install` است ، با این تفاوت که executable ها و library های نصب‌ شده را strip می‌ کند و symbol table و اطلاعات debugging غیر ضروری را حذف می‌ کند در نتیجه binary ها فضای کمتری مصرف می‌ کنند ، هرچند اطلاعاتی که برای debugging مفید هستند نیز حذف می‌ شوند.

#### 🔹 Autoconf Logfiles

وقتی `configure` شکست می‌ خورد ، پیام terminal همیشه علت اصلی را به‌ وضوح نشان نمی‌ دهد. مهم‌ ترین فایل برای investigation در این مرحله ```config.log``` است. مشکل اینجاست که `config.log` ممکن است بسیار بزرگ باشد و پیدا کردن محل دقیق خطا در آن دشوار شود. یک روش این است که ابتدا به انتهای فایل بروید برای مثال در `less` با ```G``` انجام می شود. اما صرفاً رفتن به انتهای فایل همیشه بهترین روش نیست، زیرا `configure` در انتهای فایل اطلاعات بسیار زیادی درباره‌ ی environment ، output variable ها، cache variable ها و سایر definition ها ذخیره می‌ کند. روش بهتر این است که به انتهای فایل بروید و به‌ صورت معکوس دنبال عبارت مرتبط با خطا بگردید. در `less` می‌ توان از reverse search استفاده کرد با ```?``` و سپس عبارت مورد نظر را جست‌ وجو کرد. هدف این است که به بخش کوچکی برسید که در آن failure واقعی رخ داده و معمولاً چند خط قبل از آن command دیده می‌ شود که دلیل شکست را مشخص می‌ کند. در نتیجه ، هنگام برخورد با یک خطای کلی مانند ```C compiler cannot create executables``` نباید همان پیام را به‌ عنوان علت نهایی در نظر گرفت. باید `config.log` را باز کرد و command آزمایشی واقعی و error حاصل از آن را پیدا کرد.

#### 🔹 pkg-config

وقتی یک package به library های third-party وابسته باشد ، نگهداری همه‌ ی library ها در یک directory مشترک همیشه کار ساده‌ ای نیست. از طرف دیگر، اگر هر library با prefix متفاوتی نصب شود، پیدا کردن header ها و library های آن‌ ها برای package های دیگر دشوار می‌ شود. برای مثال  فرض کنید می‌ خواهید OpenSSH را compile کنید و این package به OpenSSL نیاز دارد که Configuration process باید بداند header file های OpenSSL کجا هستند و library های آن کجا هستند و چه flag هایی برای compilation لازم است و چه library هایی باید هنگام linking استفاده شوند. ابزار **pkg-config** برای همین مشکل طراحی شده است و Syntax کلی آن به این شکل است :

```bash
pkg-config options package1 package2 ...
```

برای مثال برای دیدن library های مورد نیاز یک package مانند zlib می‌ توان نوشت :

```bash
pkg-config --libs zlib
```

و خروجی ممکن است چیزی شبیه ```lz-``` باشد همچنین برای دیدن package هایی که `pkg-config` می‌ شناسد می‌ توان از ```pkg-config --list-all``` استفاده کرد.

#### 🔹 How pkg-config Works

در `pkg-config` اطلاعات package ها را از configuration file هایی با پسوند ```pc.``` می‌ خواند برای مثال ، یک فایل مانند ```openssl.pc``` ممکن است اطلاعاتی شبیه این داشته باشد :

```text
prefix=/usr
exec_prefix=${prefix}
libdir=${exec_prefix}/lib/x86_64-linux-gnu
includedir=${prefix}/include

Name: OpenSSL
Description: Secure Sockets Layer and cryptography libraries and tools
Version: ...
Requires:
Libs: -L${libdir} -lssl -lcrypto
Libs.private: -ldl -lz
Cflags: -I${includedir}
```

در این ساختار ```Libs``` اطلاعات لازم برای linker را مشخص می‌کند و ```Cflags``` اطلاعات لازم برای compile و include path ها را در اختیار build system قرار می‌ دهد. همچنین فایل `pc.` می‌ تواند به library path هایی اشاره کند که برای runtime اهمیت دارند. کتاب حتی به این نکته اشاره می‌ کند که می‌ توان flag هایی مانند `=Wl,-rpath-` را در library flags قرار داد تا runtime search path مشخص شود. اما سؤال مهم این است که خود `pkg-config` این `.` file ها را از کجا پیدا می‌ کند ؟ به‌ طور پیش‌فرض ، `pkg-config` به directory مربوط به `pkgconfig` در installation prefix خودش نگاه می‌ کند. برای مثال اگر `pkg-config` با prefix زیر نصب شده باشد :

```text
/usr/local
```

مسیر مورد انتظار می‌تواند ```usr/local/lib/pkgconfig``` باشد. یک نکته‌ی مهم دیگر این است که `pc .file ` مربوط به بسیاری از packageها فقط با نصب **development package** در دسترس قرار می‌ گیرد. برای مثال در Ubuntu ممکن است برای دسترسی به `openssl.pc` نیاز باشد package توسعه‌ ای مانند ```libssl-dev``` نصب شده باشد.

#### 🔹 Installing pkg-config Files in Nonstandard Locations

اگر `pc` .file در مسیر غیر استانداردی مانند ```opt/openssl/lib/pkgconfig/openssl.pc/``` قرار داشته باشد ، یک installation معمولی از `pkg-config` ممکن است به‌ صورت خودکار آن را پیدا نکند و کتاب دو راه اصلی را مطرح می‌ کند : اول می‌توان symbolic link یا copy از `pc` .file را در directory مرکزی `pkgconfig` قرار داد. دوم می‌ توان environment variable زیر را تنظیم کرد :

```text
PKG_CONFIG_PATH
```

و directory های اضافه‌ ای را که باید برای `pc` .file ها جست‌ وجو شوند به آن معرفی کرد. البته کتاب هشدار می‌ دهد که استفاده از این روش به‌ صورت system-wide همیشه راه‌ حل مناسبی نیست و باید با دقت انجام شود.

---

### Installation Practice

دانستن اینکه چگونه software را build کنیم کافی نیست بلکه باید بدانیم **چه زمانی** نصب از source ارزش دارد و **کجا** بهتر است آن را نصب کنیم. Linux distributions معمولاً تعداد زیادی software را از قبل package می‌ کنند و بنابراین قبل از شروع build دستی ، باید بررسی کرد که آیا package مناسب در repository های distribution وجود دارد یا نه. نصب از source مزایایی دارد می‌توانید default های package را customize کنید و هنگام installation شناخت بهتری از نحوه‌ ی کار package به‌ دست آورید و version مورد نظر خودتان را انتخاب کنید و backup کردن installation سفارشی را آسان‌تر کنید و همان software را در یک محیط دیگر، در صورت یکسان بودن architecture و نسبتاً جدا بودن محل installation راحت‌ تر منتقل کنید.

اما این روش معایبی نیز دارد ، اگر package مورد نظر از قبل توسط distribution نصب شده باشد ، installation مستقیم ممکن است فایل‌ های مهم را overwrite کند و مشکلاتی ایجاد شود بنابراین باید قبل از نصب بررسی کنید که آیا package مشابه از قبل در سیستم وجود دارد و آیا distribution برای آن package ارائه‌ ای دارد یا خیر. همچنین نصب از source زمان می‌ برد ، به‌ صورت خودکار upgrade نمی‌ شود و در package هایی که با network سروکار دارند ، این مسئله اهمیت بیشتری پیدا می‌ کند زیرا distribution ممکن است security update ها را بدون دخالت زیاد کاربر دریافت کند ، در حالی که یک package سفارشی چنین چیزی را تضمین نمی‌ کند.

از طرف دیگر، نصب دستی می‌ تواند با سوء استفاده از configuration های نادرست یا انتخاب اشتباه path ها باعث misconfiguration شود. بنابراین برای utility های پایه‌ ای که distribution از قبل به‌ خوبی مدیریت می‌ کند ، معمولاً نصب دستی سود چندانی ندارد. اما برای software هایی مثل network server ها که ممکن است کنترل کامل روی version و configuration آن‌ ها ضروری باشد ، build و installation دستی می‌ تواند ارزش بیشتری داشته باشد.

#### 🔹 Where to Install

در prefix برای software که به‌ صورت local و خارج از package management سیستم نصب شده ، این directory در مسیر ```usr/local/``` است. این انتخاب یک مزیت مهم دارد اینکه system upgrade ها معمولاً محتوای `/usr/local` را مانند فایل‌ های تحت مدیریت خودشان تغییر نمی‌ دهند. بنابراین برای local installation های کوچک ، `/usr/local` محل مناسبی است اما `/usr/local` یک محدودیت مهم نیز دارد اینکه اگر تعداد package های سفارشی زیاد شود ، directory به‌ مرور به مجموعه‌ ای از هزاران فایل تبدیل می‌ شود و ممکن است دیگر مشخص نباشد هر فایل متعلق به کدام package بوده است. در چنین شرایطی ، مشکل فقط شلوغی directory نیست بلکه مدیریت upgrade ، removal و tracking فایل‌ ها نیز دشوار می‌ شود. اگر installation های سفارشی به اندازه‌ ای زیاد شدند که `/usr/local` به محیطی نامنظم تبدیل شد ، راه‌ حل مناسب‌ تر این است که package واقعی برای software بسازید تا package management سیستم بتواند installation را دنبال و مدیریت کند.

---

### Applying a Patch

امروزه بسیاری از تغییرات software در Git repository یا branch های development در دسترس هستند ، اما هنوز هم با **patch**ها سروکار خواهید داشت. Patch مجموعه‌ ای از تغییرات است که می‌ تواند برای رفع bug یا اضافه کردن feature روی source tree اعمال شود. گاهی واژه‌ ی **diff** نیز برای patch استفاده می‌ شود ، زیرا برنامه‌ ی `diff` می‌ تواند فایل حاوی تغییرات را تولید کند. ابتدای patch معمولاً شامل اطلاعاتی درباره‌ ی فایل‌ های قدیمی و جدید است ، مثلاً : 

```text
--- src/file.c.orig
+++ src/file.c
```

یک نکته‌ ی مهم این است که patch ممکن است تغییرات چندین فایل را در خود داشته باشد. برای فهمیدن اینکه patch از کجا باید اعمال شود ، می‌ توان در ابتدای patch دنبال خطوطی مانند ```---``` گشت و path فایل‌ هایی که تغییر کرده‌ اند را بررسی کرد. فرض کنید patch به فایل ```src/file.c``` اشاره می‌ کند در این حالت باید در directory قرار داشته باشید که `src` در آن وجود دارد ، نه اینکه خودتان وارد `src` شده باشید سپس می‌ توان patch را با `patch` اعمال کرد :

```bash
patch -p0 < patch_file
```

اگر همه‌ چیز درست باشد ، `patch` تغییرات را اعمال می‌ کند و مجموعه‌ ی فایل‌ های source به‌ روزرسانی می‌ شوند. اما اگر پیام ```File to patch``` ظاهر شود معمولاً یکی از دو حالت مطرح است اینکه در directory اشتباه قرار دارید یا source code با version که patch برای آن ساخته شده مطابقت ندارد که حالت دوم مهم‌ تر است چون اگر patch مربوط به یک version متفاوت باشد ، ممکن است بتوانید بعضی فایل‌ ها را patch کنید اما مجموعه‌ ی source tree در نهایت ناسازگار شود و build دیگر موفق نباشد. گاهی path داخل patch شامل یک component اضافه است ، برای مثال ```package-3.42/src/file.c``` اگر خودتان در directory مربوط به package قرار دارید ، می‌ توانید یک component ابتدایی path را حذف کنید مثلا ```patch -p1 < patch_file```. به این ترتیب `package-3.42/` از ابتدای pathname کنار گذاشته می‌ شود و `patch` می‌ تواند فایل را در مسیر درست پیدا کند.


---

### Troubleshooting Compiles and Installations

اگر تفاوت میان **compiler error ، compiler warning ، linker error و shared library problem** را که در فصل ۱۵ مطرح شدند بشناسید ، troubleshooting بسیاری از build ها ساده‌ تر می‌ شود. اما هنگام خواندن خروجی `make` یک نکته‌ ی مهم وجود دارد اینکه همه‌ ی پیام‌ هایی که با `Error` ظاهر می‌ شوند الزاماً به یک شکل معنا ندارند. یک خطای واقعی می‌ تواند به این شکل باشد :

```text
make: *** [target] Error 1
```

اما ممکن است یک Makefile از قبل انتظار یک failure خاص را داشته باشد و آن را بی‌ اهمیت بداند. در این حالت ممکن است پیام شبیه این دیده شود :

```text
make: *** [target] Error 1 (ignored)
```

وجود `(ignored)` نشان می‌ دهد که آن خطا طبق منطق Makefile قرار نیست باعث شکست نهایی build شود. در package های بزرگ نیز GNU `make` ممکن است بارها خودش را فراخوانی کند. در این حالت شماره‌ هایی مانند :

```text
make[1]
make[2]
make[3]
```

نشان‌دهنده‌ ی nesting مختلف `make` هستند. برای پیدا کردن علت اصلی ، نباید از آخرین `Error` شروع کنید. معمولاً خطای واقعی چند خط یا چند مرحله قبل ، در اولین compiler error دیده می‌ شود. برای مثال ممکن است خروجی نهایی چیزی شبیه این باشد :

```text
compiler error involving file.c
make[3]: *** [file.o] Error 1
make[3]: Leaving directory ...
make[2]: *** [all] Error 2
make[1]: *** [all-recursive] Error 1
make: *** [all] Error 2
```

در چنین حالتی تقریباً تمام `make` error های پایین‌ دست نتیجه‌ ی همان compiler error اولیه هستند. بنابراین روش درست این است که ابتدا اولین خطای واقعی compiler یا preprocessor را پیدا کنید و همان را بررسی کنید ، نه اینکه صرفاً آخرین `Error 2` را دنبال کنید.

#### 🔹 Conflicting Types

ممکن است compiler خطایی درباره‌ ی `conflicting types` برای یک symbol گزارش کند و هم‌ زمان به declaration قبلی آن در header اشاره کند. برای مثال اگر program یک identifier را با type متفاوت دوباره declare کرده باشد، compiler متوجه ناسازگاری می‌ شود. در این وضعیت باید declaration های مربوط به symbol را بررسی کنید و مشخص کنید کدام declaration نادرست یا اضافی است. راه‌ حل ممکن است حذف declaration اضافی یا استفاده از conditional compilation مانند `ifdef#` باشد بسته به ساختار source code.

#### 🔹 Undeclared `time_t`

یکی از نمونه‌ های مهم خطا زمانی است که compiler نوعی مانند ```time_t ``` را نمی‌ شناسد. اغلب چنین خطایی به این معناست که header موردنیاز include نشده است. مثلاً ممکن است در source code متغیری مانند این داشته باشید :

```c
time_t v1;
```
و بعد آن را با تابع `()time` استفاده کنید. برای پیدا کردن header صحیح ، یکی از بهترین روش‌ ها استفاده از manual page هاست. برای مثال می‌ توان بررسی کرد :

```bash
man 2 time
```

یا:

```bash
man 3 time
```

در بخش `SYNOPSIS` معمولاً header مورد نیاز مشخص می‌ شود. در مثال کتاب ، استفاده از `()time` به :

```c
#include <time.h>
```
نیاز دارد بنابراین باید این header را در ابتدای source file قرار دهید و دوباره build را امتحان کنید.  وقتی compiler یک function یا type ناشناخته را گزارش می‌ کند ، به manual page مربوط به همان API مراجعه کنید و header مورد نیاز را از `SYNOPSIS` پیدا کنید.

#### 🔹 Missing Header File

خطای دیگری که ممکن است با آن رو به‌ رو شوید چیزی شبیه این است :

```text
src.c:4: pkg.h: No such file or directory
```

در این حالت preprocessor نتوانسته header موردنظر را پیدا کند و چند علت محتمل وجود دارد اینکه ممکن است source code به یک library وابسته باشد که هنوز نصب نشده است یا ممکن است خود library نصب باشد ولی development package آن وجود نداشته باشد یا ممکن است header در مسیر nonstandard قرار داشته باشد و compiler از آن directory خبر نداشته باشد. در حالت سوم معمولاً باید include path را با `I-` به `CPPFLAGS` اضافه کنید :

```bash
CPPFLAGS="-I/path/to/include" ./configure
```

اما ممکن است علاوه بر header ها ، library های همان dependency نیز در مسیر غیر استاندارد باشند و در آن صورت linker نیز به یک `L-` نیاز داشته باشد بنابراین missing header گاهی فقط یک include problem نیست و می‌ تواند نشانه‌ ی dependency ناقص باشد. اگر مطمئن نیستید کدام package آن header را فراهم می‌ کند ، می‌ توان از ابزارهای جست‌ وجوی package در distribution استفاده کرد. در Debian-based systems کتاب به `apt-file` اشاره می‌ کند :

```bash
apt-file search pkg.h
```

و در سیستم‌ هایی که از yum استفاده می‌ کنند :

```bash
yum provides '*/pkg.h'
```

این ابزارها می‌ توانند مشخص کنند کدام development package فایل header مورد نظر را ارائه می‌ کند. اگر نه library missing باشد و نه include path مشکل داشته باشد ، احتمال دیگری وجود دارد: شاید source code اساساً برای operating system یا platform فعلی طراحی نشده باشد. در چنین شرایطی باید `Makefile`، `README` و documentation package را برای platform های پشتیبانی‌ شده بررسی کرد.

#### 🔹 Command Not Found

یک خطا ی دیگر هنگام build ممکن است از طرف `make` دیده شود ```make: prog: Command not found``` ، این پیام یعنی build process می‌ خواهد برنامه‌ ای با نام `prog` را اجرا کند ولی آن را پیدا نمی‌کند اگر `prog` چیزی مانند cc یا gcc یا ld باشد احتمال دارد ابزارهای development روی سیستم نصب نشده باشند. در این شرایط ابتدا باید مطمئن شوید compiler و سایر ابزارهای build نصب هستند اما ممکن است program مورد نظر روی سیستم نصب باشد و مشکل فقط در `PATH` یا نحوه‌ ی reference کردن آن باشد. در چنین حالتی می‌ توان در Makefile مسیر کامل executable را مشخص کرد. یک حالت نسبتاً نادر دیگر این است که source package برنامه‌ ای را در همان directory build می‌ کند و بلافاصله بعد از ساخت ، آن را اجرا می‌ کند درحالی‌که Makefile فرض کرده current directory یعنی `.` داخل `PATH` قرار دارد. اگر `.` در `PATH` نباشد ، executable پیدا نمی‌ شود. در این حالت می‌ توان Makefile را طوری تغییر داد که به‌ جای ```prog``` از ```prog/.``` استفاده کند. راه دیگر این است که directory جاری را به‌ صورت موقت به `PATH` اضافه کنید هرچند این روش باید با دقت انجام شود.

---

### Looking Forward

این فصل فقط مقدمه‌ ای بر build و installation از source code بود و بعد از آشنایی با این فرآیند ، چند مسیر طبیعی برای ادامه‌ ی یادگیری وجود دارد. یکی از این مسیرها یادگیری build system های دیگری مانند CMake و SCons است. دانستن Autoconf فقط یک نقطه‌ ی شروع است و در پروژه‌ های مختلف ممکن است با build system های متفاوتی رو به‌ رو شوید. 

موضوع مهم دیگری که کتاب پیشنهاد می‌ کند ، **build کردن Linux kernel** است. build system kernel با build system معمول software های user space تفاوت زیادی دارد و configuration system مخصوص خودش را دارد که برای انتخاب feature ها و module ها طراحی شده است.

فرآیند build kernel در اصل قابل انجام است ، اما هنگام نصب kernel جدید باید احتیاط کرد و همیشه یک kernel قدیمی و سالم را در سیستم نگه داشت تا اگر kernel جدید boot نشد ، امکان بازگشت وجود داشته باشد.

در نهایت ، باید با **distribution-specific source packages** نیز آشنا شد. Linux distributions معمولاً source package های مخصوص خودشان را نگهداری می‌ کنند و ممکن است patch های مفیدی برای رفع bug یا افزودن functionality در آن‌ها وجود داشته باشد.

در این زمینه ابزارهایی برای build خودکار package ها نیز وجود دارند برای مثال در اکوسیستم Debian ابزارهایی مانند debuild و در محیط‌ های RPM-based ابزارهایی مانند mock به کار می‌ روند.

شناخت این سیستم‌ ها کمک می‌ کند از مرحله‌ ی **ساخت software از source** به مرحله‌ ی بالاتر یعنی **ساخت software به‌ شکل package قابل مدیریت در distribution** حرکت کنیم.

---

### Summary

فصل شانزدهم فرایند واقعی build و installation یک C software را دنبال می‌ کند ، از زمانی که source archive را دریافت می‌ کنیم تا زمانی که پکیج build و test و نصب می‌ شود.

 ابتدا باید archive را با دقت بررسی و در یک directory مناسب extract کنیم. سپس `README` و `INSTALL` را بخوانیم و مطمئن شویم build environment و dependency های مورد نیاز آماده هستند.

در package های مبتنی بر **GNU Autoconf** هم `configure` ویژگی‌ های سیستم را بررسی می‌ کند و بر اساس آن فایل‌ های build مانند `Makefile` را تولید می‌ کند. بعد `make` software را build می‌ کند و `make install` فایل‌ های ساخته‌ شده را در prefix تعیین‌ شده نصب می‌ کند.

گزینه‌ هایی مانند `prefix--`و `bindir--` و `libdir--` محل نصب را کنترل می‌ کنند، در حالی که `CPPFLAGS`و `CFLAGS` و `LDFLAGS` نحوه‌ی پیدا کردن header ها ، تنظیمات compiler و مسیر library ها را تحت تأثیر قرار می‌دهند. برای dependency های پیچیده نیز `pkg-config` اطلاعات مورد نیاز برای compile و link کردن را از `.pc` file ها فراهم می‌ کند.

در installation ، تفاوت میان نصب مستقیم از source و ساخت package اهمیت زیادی دارد. `/usr/local` برای local installation های محدود مناسب است ، اما با زیاد شدن software های سفارشی ، ساخت package واقعی مدیریت سیستم را بسیار ساده‌ تر می‌ کند.

در نتیجه، build کردن software روی Linux یک زنجیره‌ی مشخص دارد: **source archive، configuration، compilation، testing، installation و در صورت نیاز packaging**؛ و شناخت هر مرحله باعث می‌شود خطاها و dependencyهای آن مرحله را دقیق‌تر پیدا و برطرف کنیم.

در troubleshooting نیز مهم‌ ترین اصل این است که از میان خروجی طولانی `make` به آخرین خطا خیره نشویم. باید **اولین خطای واقعی compiler یا preprocessor یا linker** را پیدا کنیم ، چون بسیاری از error های بعدی فقط پیامد همان خطای اولیه هستند.
