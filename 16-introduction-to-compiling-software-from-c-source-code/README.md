## مقدمه‌ ای بر کامپایل نرم‌افزار با زبان C
### 🐧 فصل شانزدهم کتاب How Linux Works

این فصل وارد مرحله‌ای می‌شود که بعد از نوشتن source code اهمیت پیدا می‌کند: **ساخت، پیکربندی، نصب و نگهداری software از روی source code**. در Linux بسیاری از نرم‌افزارهای غیرتجاری به‌صورت source distribution منتشر می‌شوند و کاربر می‌تواند آن‌ها را روی سیستم خودش build و نصب کند.

یکی از دلایل این موضوع تنوع زیاد Linux، معماری‌های سخت‌افزاری و تفاوت میان سیستم‌هاست؛ توزیع یک binary یکسان برای همه‌ی ترکیب‌های ممکن سیستم‌عامل و architecture دشوار است. علاوه بر این، انتشار source code باعث می‌شود کاربران بتوانند bugها را برطرف کنند، featureهای جدید اضافه کنند و software را متناسب با محیط خودشان بسازند.

البته اینکه بتوانیم یک software را از source code نصب کنیم به این معنی نیست که همیشه باید این کار را انجام دهیم. Linux distributions معمولاً روش ساده‌تر و قابل‌مدیریت‌تری برای نصب و به‌روزرسانی software دارند و برای بخش‌های مهم سیستم نیز معمولاً security updateها را سریع‌تر ارائه می‌کنند. با این حال، نصب مستقیم از source در بعضی شرایط مزایایی دارد:

* کنترل بیشتر روی configuration و build options
* امکان نصب software در مسیر دلخواه
* امکان داشتن چند version از یک package
* کنترل version دقیق software
* درک بهتر نحوه‌ی کار package

به همین دلیل این فصل به‌جای بررسی تمام build systemهای Linux، روی یک روش رایج برای **C software** یعنی **GNU Autoconf و GNU Autotools** تمرکز می‌کند. این روش روی ابزارهایی مانند `make` بنا شده و آشنایی با آن به درک سایر build systemها نیز کمک می‌کند.

---
📚 Table of Contents

- [16.1 Software Build Systems](#161-software-build-systems)
- [16.2 Unpacking C Source Packages](#162-unpacking-c-source-packages)
- [16.3 GNU Autoconf](#163-gnu-autoconf)
  - [16.3.1 An Autoconf Example](#1631-an-autoconf-example)
  - [16.3.2 Installation Using a Packaging Tool](#1632-installation-using-a-packaging-tool)
  - [16.3.3 configure Script Options](#1633-configure-script-options)
    - [Separate Build Directories](#separate-build-directories)
  - [16.3.4 Environment Variables](#1634-environment-variables)
  - [16.3.5 Autoconf Targets](#1635-autoconf-targets)
    - [`make clean`](#make-clean)
    - [`make distclean`](#make-distclean)
    - [`make check`](#make-check)
    - [`make install-strip`](#make-install-strip)
  - [16.3.6 Autoconf Logfiles](#1636-autoconf-logfiles)
  - [16.3.7 pkg-config](#1637-pkg-config)
    - [How pkg-config Works](#how-pkg-config-works)
    - [Installing pkg-config Files in Nonstandard Locations](#installing-pkg-config-files-in-nonstandard-locations)
- [16.4 Installation Practice](#164-installation-practice)
  - [16.4.1 Where to Install](#1641-where-to-install)
- [16.5 Applying a Patch](#165-applying-a-patch)
- [16.6 Troubleshooting Compiles and Installations](#166-troubleshooting-compiles-and-installations)
  - [16.6.1 Specific Errors](#1661-specific-errors)
    - [Conflicting Types](#conflicting-types)
    - [Undeclared `time_t`](#undeclared-time_t)
    - [Missing Header File](#missing-header-file)
    - [Command Not Found](#command-not-found)
- [16.7 Looking Forward](#167-looking-forward)
- [Summary](#summary)

---

### 16.1 Software Build Systems

در Linux فقط C وجود ندارد. از زبان‌هایی مانند C و C++ گرفته تا scripting languageهایی مثل Python، هر محیط برنامه‌نویسی ممکن است روش متفاوتی برای build و installation داشته باشد.

توزیع‌های Linux نیز خودشان مجموعه‌ای از ابزارهای package management در اختیار دارند، اما در این فصل تمرکز روی فرآیند build یک **C source package** است.

فرآیند معمول نصب software از source code را می‌توان به چهار مرحله‌ی اصلی تقسیم کرد:

1. باز کردن source archive
2. configure کردن package
3. build کردن برنامه‌ها با `make` یا build command مناسب
4. نصب package با `make install` یا ابزار نصب مربوط به همان build system

این فصل بیشتر روی فرآیند مبتنی بر **Autoconf** تمرکز دارد؛ یعنی حالتی که یک package دارای scriptای مانند `configure` است و این script محیط سیستم را بررسی کرده و فایل‌های build مناسب را تولید می‌کند.

نکته‌ی مهم این است که قبل از ورود به این فصل، باید با مفاهیم پایه‌ی compiler، linker، library و `make` که در فصل ۱۵ مطرح شدند آشنا باشید.

---

### 16.2 Unpacking C Source Packages

source distributionها معمولاً به‌صورت archiveهایی با پسوندهایی مانند زیر ارائه می‌شوند:

```text
.tar.gz
.tar.bz2
.tar.xz
```

قبل از extract کردن archive بهتر است ابتدا محتویات آن را بررسی کنید. برای این کار می‌توان از دستورهایی مانند موارد زیر استفاده کرد:

```bash
tar tvf package.tar
```

یا برای archiveهایی که با gzip فشرده شده‌اند:

```bash
tar ztvf package.tar.gz
```

دلیل این بررسی فقط کنجکاوی نیست. مهم است که بدانید archive بعد از extract شدن چه ساختاری دارد.

یک package مناسب معمولاً یک directory مخصوص خودش دارد:

```text
package-1.23/
    Makefile.in
    README
    main.c
    bar.c
    ...
```

در این حالت extract کردن package نسبتاً امن و مرتب است، چون فایل‌های package داخل یک directory جدا قرار می‌گیرند.

اما ممکن است archive ساختار مناسبی نداشته باشد و فایل‌هایی مانند این‌ها را مستقیماً در directory فعلی قرار دهد:

```text
Makefile
README
main.c
...
```

اگر چنین archiveای را در یک directory شلوغ extract کنید، فایل‌های package با فایل‌های دیگر مخلوط می‌شوند و پیدا کردن یا پاک کردن آن‌ها دشوار خواهد شد. بنابراین در چنین شرایطی بهتر است ابتدا یک directory جدید بسازید، وارد آن شوید و سپس archive را extract کنید.

موضوع مهم دیگر **امنیت archive** است. اگر archive شامل pathnameهای absolute مانند موارد زیر باشد:

```text
/etc/passwd
/etc/inetd.conf
```

باید به آن شدیداً مشکوک باشید. چنین archiveای نباید به‌صورت عادی extract شود، زیرا ممکن است قصد overwrite کردن فایل‌های سیستم را داشته باشد و احتمال وجود malicious code یا Trojan در آن مطرح است.

بعد از extract شدن source، اولین فایل‌هایی که باید بررسی شوند `README` و `INSTALL` هستند. فایل `README` معمولاً توضیحی درباره‌ی package، نحوه‌ی استفاده، نکات installation و اطلاعات عمومی پروژه دارد. فایل `INSTALL` نیز در بسیاری از پروژه‌ها دستورالعمل دقیق‌تری برای configure، compile و install ارائه می‌دهد.

هنگام خواندن این فایل‌ها باید به **special compiler options، definitions، dependencyها و پیش‌نیازهای build** توجه ویژه‌ای داشته باشید.

فایل‌های داخل source package را می‌توان به‌صورت کلی در چند گروه دید.

گروه اول فایل‌های مربوط به build system هستند، مانند:

```text
Makefile
Makefile.in
configure
CMakeLists.txt
```

در packageهای قدیمی ممکن است `Makefile` مستقیماً برای ویرایش دستی در اختیار کاربر قرار گرفته باشد. اما در بسیاری از packageهای جدیدتر، configuration utility مانند **GNU Autoconf** یا **CMake** وجود دارد که براساس تنظیمات سیستم، فایل build نهایی را تولید می‌کند.

گروه دوم source fileها هستند:

```text
.c
.h
.cc
.C
.cxx
```

فایل‌های C ممکن است در هر نقطه‌ای از source tree قرار داشته باشند. در پروژه‌های C++ نیز suffixهای مختلفی برای source fileها دیده می‌شود.

گروه سوم object fileها یا binaryهای از قبل ساخته‌شده هستند:

```text
.o
```

در یک source distribution استاندارد معمولاً انتظار نداریم object file از قبل وجود داشته باشد. اگر object file یا executable آماده داخل source package دیده شود، ممکن است package به‌درستی آماده نشده باشد. البته در موارد خاص ممکن است maintainer نتواند بخشی از source code را منتشر کند و مجبور شده باشد object file را در package قرار دهد.

در حالت معمول، اگر object file یا binary از قبل وجود دارد، بهتر است با:

```bash
make clean
```

محیط build را پاک کنید تا build از source تازه شروع شود و نتیجه به فایل‌های قدیمی وابسته نباشد.

---

### 16.3 GNU Autoconf

در نگاه اول ممکن است به نظر برسد که C آن‌قدر portable است که یک `Makefile` واحد بتواند روی هر سیستم اجرا شود، اما در عمل تفاوت‌های میان platformها، libraryها، compilerها و امکانات موجود باعث می‌شود چنین کاری دشوار باشد.

راه‌حل‌های قدیمی‌تر شامل تهیه‌ی `Makefile` جدا برای هر operating system یا ساختن `Makefile`ای بود که کاربر مجبور می‌شد خودش آن را ویرایش کند.

Autoconf ایده‌ی متفاوتی دارد: به‌جای اینکه developer برای هر سیستم یک `Makefile` مستقل بنویسد، اطلاعات سیستم در زمان configure بررسی می‌شود و build files بر اساس همان محیط تولید می‌شوند.

در packageهای مبتنی بر GNU Autoconf معمولاً فایل‌هایی مانند این‌ها را می‌بینیم:

```text
configure
Makefile.in
config.h.in
```

فایل‌هایی که با `.in` تمام می‌شوند template هستند. یعنی هنوز configuration نهایی داخل آن‌ها قرار نگرفته است.

`configure` سیستم را بررسی می‌کند، ویژگی‌ها و prerequisiteهای لازم را تشخیص می‌دهد و براساس آن templateها را به فایل‌های موردنیاز build تبدیل می‌کند.

بنابراین اجرای:

```bash
./configure
```

فقط یک command ساده برای ساخت `Makefile` نیست؛ این script مجموعه‌ای از تست‌ها را اجرا می‌کند تا بفهمد سیستم فعلی چه امکاناتی دارد.

در صورت موفقیت، معمولاً مواردی مانند این‌ها تولید می‌شوند:

```text
Makefile
config.h
config.cache
```

`config.cache` به configure اجازه می‌دهد برخی نتیجه‌ی تست‌ها را ذخیره کند تا هنگام اجرای بعدی لازم نباشد همه‌ی تست‌ها را از ابتدا تکرار کند.

نکته‌ی مهم این است که **موفق بودن `configure` تضمین نمی‌کند `make` نیز حتماً موفق شود**. configure فقط می‌تواند بگوید محیط برای build تا حد موردنیاز آماده به نظر می‌رسد؛ خطاهای واقعی compilation یا linking ممکن است بعداً ظاهر شوند.

برای build کردن package نیز باید ابزارهای توسعه روی سیستم وجود داشته باشند. در Debian و Ubuntu، کتاب به `build-essential` اشاره می‌کند و در سیستم‌های Fedora-like استفاده از گروه **Development Tools** را مطرح می‌کند.

#### 16.3.1 An Autoconf Example

برای دیدن فرآیند واقعی، کتاب از **GNU coreutils** به‌عنوان نمونه استفاده می‌کند و package را به‌جای نصب در بخش‌های سیستمی، داخل home directory کاربر قرار می‌دهد.

پس از دریافت و extract کردن source، می‌توان package را با تعیین یک installation prefix اختصاصی configure کرد:

```bash
./configure --prefix=$HOME/mycoreutils
```

در این مرحله `configure` خروجی diagnostic زیادی تولید می‌کند و بررسی می‌کند که محیط build چه وضعیتی دارد.

بعد از configure، مرحله‌ی build انجام می‌شود:

```bash
make
```

اگر build موفق باشد، می‌توانید یکی از executableهای ساخته‌شده را مستقیماً از داخل source tree اجرا کنید؛ برای مثال نسخه‌ی تازه ساخته‌شده‌ی `ls`.

همچنین در این مرحله می‌توان تست‌های package را اجرا کرد:

```bash
make check
```

برای بعضی packageها این مرحله مجموعه‌ای از testها را اجرا می‌کند و ممکن است زمان قابل‌توجهی طول بکشد.

پس از آن نوبت installation است. قبل از نصب واقعی، بهتر است اول یک **dry run** انجام شود:

```bash
make -n install
```

گزینه‌ی `-n` باعث می‌شود commandهایی که قرار است اجرا شوند نمایش داده شوند اما واقعاً اجرا نشوند.

در این مرحله باید خروجی را بررسی کنید و مطمئن شوید فایل‌ها قرار نیست در مسیر نادرست نصب شوند.

بعد از بررسی می‌توان installation واقعی را انجام داد:

```bash
make install
```

با توجه به `--prefix=$HOME/mycoreutils`، فایل‌ها در directory اختصاصی کاربر قرار می‌گیرند و زیرشاخه‌هایی مانند این‌ها ایجاد می‌شوند:

```text
mycoreutils/
├── bin/
├── share/
└── ...
```

این روش مزیت مهمی دارد: package از بقیه‌ی سیستم جدا باقی می‌ماند و در صورت نیاز می‌توان کل directory را حذف کرد بدون اینکه فایل‌های اصلی سیستم تحت تأثیر قرار بگیرند.

#### 16.3.2 Installation Using a Packaging Tool

`make install` فایل‌ها را مستقیماً در filesystem قرار می‌دهد، اما در بسیاری از Linux distributions روش بهتر این است که نرم‌افزار را به یک **package قابل مدیریت** تبدیل کنیم.

در این حالت بعداً می‌توان package را با package management system توزیع نصب، حذف یا مدیریت کرد.

در سیستم‌های Debian-based، کتاب به ابزار `checkinstall` اشاره می‌کند. به‌جای اجرای مستقیم:

```bash
make install
```

می‌توان از:

```bash
checkinstall make install
```

استفاده کرد.

`checkinstall` فایل‌هایی را که قرار است نصب شوند دنبال می‌کند و آن‌ها را داخل یک `.deb` قرار می‌دهد. نتیجه را می‌توان بعداً با `dpkg` نصب یا حذف کرد.

در محیط‌های RPM-based، فرآیند کمی پیچیده‌تر است. ابتدا می‌توان با:

```bash
rpmdev-setuptree
```

ساختار directory لازم برای package را ایجاد کرد و سپس از:

```bash
rpmbuild
```

برای ادامه‌ی فرآیند ساخت RPM استفاده کرد.

ساخت RPM معمولاً نیازمند آشنایی بیشتر با package specification و ساختار RPM است، بنابراین برای جزئیات این workflow باید به documentation و tutorialهای مخصوص RPM مراجعه کرد.

#### 16.3.3 configure Script Options

مهم‌ترین گزینه‌ی `configure` در بسیاری از packageها `--prefix` است.

به‌صورت پیش‌فرض، installationهای ساخته‌شده با Autoconf معمولاً prefix زیر را در نظر می‌گیرند:

```text
/usr/local
```

بنابراین فایل‌ها معمولاً به مسیرهایی مانند این‌ها می‌روند:

```text
/usr/local/bin
/usr/local/lib
...
```

برای تغییر این مکان می‌توان از:

```bash
./configure --prefix=/some/path
```

استفاده کرد.

یکی از مزیت‌های مهم `configure` این است که معمولاً گزینه‌ی `--help` را نیز ارائه می‌دهد:

```bash
./configure --help
```

این خروجی می‌تواند بسیار طولانی باشد، اما چند option برای شناختن اهمیت بیشتری دارند:

```text
--bindir=directory
```

تعیین می‌کند executableهای معمولی در کجا نصب شوند.

```text
--sbindir=directory
```

محل نصب system executableها را مشخص می‌کند.

```text
--libdir=directory
```

مسیر نصب libraryها را تعیین می‌کند.

```text
--disable-shared
```

باعث می‌شود package، shared library نسازد. بسته به نوع library، این گزینه می‌تواند بعضی مشکلات runtime مربوط به shared libraryها را از بین ببرد.

```text
--with-package=directory
```

برای معرفی محل یک package یا dependency که در directory استاندارد قرار ندارد استفاده می‌شود.

البته همه‌ی `configure` scriptها الزاماً این option را با همین نام یا syntax قبول نمی‌کنند. به همین دلیل باید optionهای واقعی همان package را از `./configure --help` یا documentation آن بررسی کرد.

##### Separate Build Directories

گاهی لازم است یک package را با configurationهای مختلف build کنید یا چند محیط متفاوت را با یک source tree آزمایش کنید.

در این شرایط می‌توان build directory جداگانه ایجاد کرد و `configure` را از آن directory روی source tree اصلی اجرا کرد.

مزیت این روش این است که source tree اصلی کمتر دست‌کاری می‌شود. همچنین اگر بخواهید یک package را با چند set از configuration options یا حتی برای چند platform مختلف build کنید، این ساختار بسیار مفید است.

در این مدل، build directory می‌تواند شامل یک مجموعه‌ی symbolic link باشد که به فایل‌های source tree اصلی اشاره می‌کنند.

این روش برای آزمایش configurationهای مختلف بدون تغییر دادن مستقیم source tree اصلی بسیار مناسب است.

#### 16.3.4 Environment Variables

`configure` را می‌توان با **environment variable**ها تحت تأثیر قرار داد. بسیاری از این variableها در نهایت به `make` منتقل می‌شوند و روی compilation و linking تأثیر می‌گذارند.

سه variable مهم که در این فصل مورد توجه قرار می‌گیرند عبارت‌اند از:

```text
CPPFLAGS
CFLAGS
LDFLAGS
```

در اینجا یک تفاوت مهم وجود دارد: برای معرفی directoryهای مربوط به header fileها معمولاً بهتر است از `CPPFLAGS` استفاده شود، نه `CFLAGS`.

دلیل این انتخاب این است که `configure` ممکن است preprocessor را جدا از compiler اجرا کند و در نتیجه flagهایی که باید روی preprocessing اعمال شوند بهتر است در `CPPFLAGS` قرار بگیرند.

برای مثال، اگر بخواهید یک preprocessor macro به نام `DEBUG` تعریف کنید:

```bash
CPPFLAGS=-DDEBUG ./configure
```

یا می‌توانید variable را به‌عنوان argument به `configure` بدهید:

```bash
./configure CPPFLAGS=-DDEBUG
```

اگر header fileها در directory غیر استاندارد قرار داشته باشند، می‌توان مسیر آن‌ها را این‌گونه اضافه کرد:

```bash
CPPFLAGS=-Iinclude_dir ./configure
```

و اگر libraryها در directory خاصی قرار داشته باشند، می‌توان مسیر آن‌ها را برای linker معرفی کرد:

```bash
LDFLAGS=-Llib_dir ./configure
```

اما در مورد shared libraryها مسئله فقط پیدا شدن library در زمان linking نیست. اگر library در مسیر غیر استاندارد باشد، runtime dynamic linker نیز باید بتواند آن را بعداً پیدا کند.

در این حالت می‌توان علاوه بر `-L` از `-rpath` نیز استفاده کرد:

```bash
LDFLAGS="-Llib_dir -Wl,-rpath=lib_dir" ./configure
```

بنابراین باید بین دو مسئله تفاوت بگذاریم:

```text
-L
```

برای کمک به linker در زمان build.

و:

```text
-rpath
```

برای ثبت runtime library search path در executable طبق روش مورد بحث در کتاب.

همچنین باید هنگام نوشتن variableها بسیار دقت کرد. یک اشتباه کوچک در flagها می‌تواند باعث شود `configure` نتواند حتی یک برنامه‌ی آزمایشی ساده را compile کند.

برای نمونه، اگر به‌اشتباه `-I` را به شکل `I` بنویسید:

```bash
CPPFLAGS=Iinclude_dir ./configure
```

ممکن است `configure` در نهایت خطایی مانند این ایجاد کند:

```text
C compiler cannot create executables
```

در چنین شرایطی خود این پیام علت اصلی مشکل نیست؛ باید به `config.log` مراجعه کرد تا command واقعی شکست‌خورده و خطای compiler دیده شود.

#### 16.3.5 Autoconf Targets

`Makefile` تولیدشده توسط Autoconf فقط targetهای ساده‌ای مانند `all` و `install` ندارد. معمولاً targetهای دیگری هم وجود دارند که برای مدیریت build مفید هستند.

##### `make clean`

این target برای پاک کردن خروجی‌های build استفاده می‌شود و معمولاً object fileها، executableها و libraryهای ساخته‌شده را حذف می‌کند.

##### `make distclean`

از `clean` گسترده‌تر است و علاوه بر build artifacts، فایل‌هایی را که `configure` تولید کرده است نیز حذف می‌کند؛ برای مثال:

```text
Makefile
config.h
config.log
...
```

هدف این است که source tree تقریباً به وضعیت یک source distribution تازه extract‌شده برگردد.

##### `make check`

این target testهای package را اجرا می‌کند. همه‌ی packageها test suite ندارند، اما در packageهایی که چنین تست‌هایی وجود دارد، این target برای بررسی صحت build بسیار مفید است.

##### `make install-strip`

این target مانند `make install` است، با این تفاوت که executableها و libraryهای نصب‌شده را strip می‌کند و symbol table و اطلاعات debugging غیرضروری را حذف می‌کند.

در نتیجه binaryها فضای کمتری مصرف می‌کنند، هرچند اطلاعاتی که برای debugging مفید هستند نیز حذف می‌شوند.

#### 16.3.6 Autoconf Logfiles

وقتی `configure` شکست می‌خورد، پیام terminal همیشه علت اصلی را به‌وضوح نشان نمی‌دهد.

مهم‌ترین فایل برای investigation در این مرحله:

```text
config.log
```

است.

مشکل اینجاست که `config.log` ممکن است بسیار بزرگ باشد و پیدا کردن محل دقیق خطا در آن دشوار شود.

یک روش این است که ابتدا به انتهای فایل بروید؛ برای مثال در `less`:

```text
G
```

اما صرفاً رفتن به انتهای فایل همیشه بهترین روش نیست، زیرا `configure` در انتهای فایل اطلاعات بسیار زیادی درباره‌ی environment، output variableها، cache variableها و سایر definitionها ذخیره می‌کند.

روش بهتر این است که به انتهای فایل بروید و به‌صورت معکوس دنبال عبارت مرتبط با خطا بگردید. در `less` می‌توان از reverse search استفاده کرد:

```text
?
```

و سپس عبارت موردنظر را جست‌وجو کرد.

هدف این است که به بخش کوچکی برسید که در آن failure واقعی رخ داده و معمولاً چند خط قبل از آن commandی دیده می‌شود که دلیل شکست را مشخص می‌کند.

در نتیجه، هنگام برخورد با یک خطای کلی مانند:

```text
C compiler cannot create executables
```

نباید همان پیام را به‌عنوان علت نهایی در نظر گرفت. باید `config.log` را باز کرد و command آزمایشی واقعی و error حاصل از آن را پیدا کرد.

#### 16.3.7 pkg-config

وقتی یک package به libraryهای third-party وابسته باشد، نگهداری همه‌ی libraryها در یک directory مشترک همیشه کار ساده‌ای نیست.

از طرف دیگر، اگر هر library با prefix متفاوتی نصب شود، پیدا کردن headerها و libraryهای آن‌ها برای packageهای دیگر دشوار می‌شود.

برای مثال، فرض کنید می‌خواهید OpenSSH را compile کنید و این package به OpenSSL نیاز دارد. Configuration process باید بداند:

* header fileهای OpenSSL کجا هستند
* libraryهای آن کجا هستند
* چه flagهایی برای compilation لازم است
* چه libraryهایی باید هنگام linking استفاده شوند

ابزار **pkg-config** برای همین مشکل طراحی شده است.

Syntax کلی آن به این شکل است:

```bash
pkg-config options package1 package2 ...
```

برای مثال برای دیدن libraryهای موردنیاز یک package مانند zlib می‌توان نوشت:

```bash
pkg-config --libs zlib
```

و خروجی ممکن است چیزی شبیه این باشد:

```text
-lz
```

همچنین برای دیدن packageهایی که `pkg-config` می‌شناسد می‌توان از:

```bash
pkg-config --list-all
```

استفاده کرد.

##### How pkg-config Works

`pkg-config` اطلاعات packageها را از configuration fileهایی با پسوند:

```text
.pc
```

می‌خواند.

برای مثال، یک فایل مانند:

```text
openssl.pc
```

ممکن است اطلاعاتی شبیه این داشته باشد:

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

در این ساختار:

```text
Libs
```

اطلاعات لازم برای linker را مشخص می‌کند و:

```text
Cflags
```

اطلاعات لازم برای compile و include pathها را در اختیار build system قرار می‌دهد.

همچنین فایل `.pc` می‌تواند به library pathهایی اشاره کند که برای runtime اهمیت دارند. کتاب حتی به این نکته اشاره می‌کند که می‌توان flagهایی مانند `-Wl,-rpath=...` را در library flags قرار داد تا runtime search path مشخص شود.

اما سؤال مهم این است که خود `pkg-config` این `.pc` fileها را از کجا پیدا می‌کند؟

به‌طور پیش‌فرض، `pkg-config` به directory مربوط به `pkgconfig` در installation prefix خودش نگاه می‌کند. برای مثال اگر `pkg-config` با prefix زیر نصب شده باشد:

```text
/usr/local
```

مسیر مورد انتظار می‌تواند این باشد:

```text
/usr/local/lib/pkgconfig
```

یک نکته‌ی مهم دیگر این است که `.pc` file مربوط به بسیاری از packageها فقط با نصب **development package** در دسترس قرار می‌گیرد. برای مثال در Ubuntu ممکن است برای دسترسی به `openssl.pc` نیاز باشد package توسعه‌ای مانند:

```text
libssl-dev
```

نصب شده باشد.

##### Installing pkg-config Files in Nonstandard Locations

اگر `.pc` file در مسیر غیر استانداردی مانند:

```text
/opt/openssl/lib/pkgconfig/openssl.pc
```

قرار داشته باشد، یک installation معمولی از `pkg-config` ممکن است به‌صورت خودکار آن را پیدا نکند.

کتاب دو راه اصلی را مطرح می‌کند:

اول، می‌توان symbolic link یا copyای از `.pc` file را در directory مرکزی `pkgconfig` قرار داد.

دوم، می‌توان environment variable زیر را تنظیم کرد:

```text
PKG_CONFIG_PATH
```

و directoryهای اضافه‌ای را که باید برای `.pc` fileها جست‌وجو شوند به آن معرفی کرد.

البته کتاب هشدار می‌دهد که استفاده از این روش به‌صورت system-wide همیشه راه‌حل مناسبی نیست و باید با دقت انجام شود.

---

### 16.4 Installation Practice

دانستن اینکه چگونه software را build کنیم کافی نیست؛ باید بدانیم **چه زمانی** نصب از source ارزش دارد و **کجا** بهتر است آن را نصب کنیم.

Linux distributions معمولاً تعداد زیادی software را از قبل package می‌کنند و بنابراین قبل از شروع build دستی، باید بررسی کرد که آیا package مناسب در repositoryهای distribution وجود دارد یا نه.

نصب از source مزایایی دارد.

می‌توانید:

* defaultهای package را customize کنید
* هنگام installation شناخت بهتری از نحوه‌ی کار package به‌دست آورید
* version موردنظر خودتان را انتخاب کنید
* backup کردن installation سفارشی را آسان‌تر کنید
* همان software را در یک محیط دیگر، در صورت یکسان بودن architecture و نسبتاً جدا بودن محل installation، راحت‌تر منتقل کنید

اما این روش معایبی نیز دارد.

اگر package موردنظر از قبل توسط distribution نصب شده باشد، installation مستقیم ممکن است فایل‌های مهم را overwrite کند و مشکلاتی ایجاد شود. بنابراین باید قبل از نصب بررسی کنید که آیا package مشابه از قبل در سیستم وجود دارد و آیا distribution برای آن package ارائه‌ای دارد یا خیر.

همچنین نصب از source زمان می‌برد، به‌صورت خودکار upgrade نمی‌شود و در packageهایی که با network سروکار دارند، این مسئله اهمیت بیشتری پیدا می‌کند؛ زیرا distribution ممکن است security updateها را بدون دخالت زیاد کاربر دریافت کند، در حالی که یک package سفارشی چنین چیزی را تضمین نمی‌کند.

از طرف دیگر، نصب دستی می‌تواند با سوءاستفاده از configurationهای نادرست یا انتخاب اشتباه pathها باعث misconfiguration شود.

بنابراین برای utilityهای پایه‌ای که distribution از قبل به‌خوبی مدیریت می‌کند، معمولاً نصب دستی سود چندانی ندارد. اما برای softwareهایی مثل network serverها که ممکن است کنترل کامل روی version و configuration آن‌ها ضروری باشد، build و installation دستی می‌تواند ارزش بیشتری داشته باشد.

#### 16.4.1 Where to Install

prefix سنتی برای softwareی که به‌صورت local و خارج از package management سیستم نصب شده، این directory است:

```text
/usr/local
```

این انتخاب یک مزیت مهم دارد: system upgradeها معمولاً محتوای `/usr/local` را مانند فایل‌های تحت مدیریت خودشان تغییر نمی‌دهند. بنابراین برای local installationهای کوچک، `/usr/local` محل مناسبی است.

اما `/usr/local` یک محدودیت مهم نیز دارد.

اگر تعداد packageهای سفارشی زیاد شود، directory به‌مرور به مجموعه‌ای از هزاران فایل تبدیل می‌شود و ممکن است دیگر مشخص نباشد هر فایل متعلق به کدام package بوده است.

در چنین شرایطی، مشکل فقط شلوغی directory نیست؛ مدیریت upgrade، removal و tracking فایل‌ها نیز دشوار می‌شود.

اگر installationهای سفارشی به اندازه‌ای زیاد شدند که `/usr/local` به محیطی نامنظم تبدیل شد، راه‌حل مناسب‌تر این است که package واقعی برای software بسازید تا package management سیستم بتواند installation را دنبال و مدیریت کند.

---

### 16.5 Applying a Patch

امروزه بسیاری از تغییرات software در Git repository یا branchهای development در دسترس هستند، اما هنوز هم با **patch**ها سروکار خواهید داشت.

Patch مجموعه‌ای از تغییرات است که می‌تواند برای رفع bug یا اضافه کردن feature روی source tree اعمال شود.

گاهی واژه‌ی **diff** نیز برای patch استفاده می‌شود، زیرا برنامه‌ی `diff` می‌تواند فایل حاوی تغییرات را تولید کند.

ابتدای patch معمولاً شامل اطلاعاتی درباره‌ی فایل‌های قدیمی و جدید است، مثلاً:

```text
--- src/file.c.orig
+++ src/file.c
```

یک نکته‌ی مهم این است که patch ممکن است تغییرات چندین فایل را در خود داشته باشد.

برای فهمیدن اینکه patch از کجا باید اعمال شود، می‌توان در ابتدای patch دنبال خطوطی مانند:

```text
---
```

گشت و path فایل‌هایی که تغییر کرده‌اند را بررسی کرد.

فرض کنید patch به این فایل اشاره می‌کند:

```text
src/file.c
```

در این حالت باید در directoryای قرار داشته باشید که `src` در آن وجود دارد، نه اینکه خودتان وارد `src` شده باشید.

سپس می‌توان patch را با `patch` اعمال کرد:

```bash
patch -p0 < patch_file
```

اگر همه‌چیز درست باشد، `patch` تغییرات را اعمال می‌کند و مجموعه‌ی فایل‌های source به‌روزرسانی می‌شوند.

اما اگر پیام زیر ظاهر شود:

```text
File to patch:
```

معمولاً یکی از دو حالت مطرح است:

1. در directory اشتباه قرار دارید.
2. source code با versionی که patch برای آن ساخته شده مطابقت ندارد.

حالت دوم مهم‌تر است. اگر patch مربوط به یک version متفاوت باشد، ممکن است بتوانید بعضی فایل‌ها را patch کنید اما مجموعه‌ی source tree در نهایت ناسازگار شود و build دیگر موفق نباشد.

گاهی path داخل patch شامل یک component اضافه است، برای مثال:

```text
package-3.42/src/file.c
```

اگر خودتان در directory مربوط به package قرار دارید، می‌توانید یک component ابتدایی path را حذف کنید:

```bash
patch -p1 < patch_file
```

به این ترتیب `package-3.42/` از ابتدای pathname کنار گذاشته می‌شود و `patch` می‌تواند فایل را در مسیر درست پیدا کند.

---

### 16.6 Troubleshooting Compiles and Installations

اگر تفاوت میان **compiler error، compiler warning، linker error و shared library problem** را که در فصل ۱۵ مطرح شدند بشناسید، troubleshooting بسیاری از buildها ساده‌تر می‌شود.

اما هنگام خواندن خروجی `make` یک نکته‌ی مهم وجود دارد: همه‌ی پیام‌هایی که با `Error` ظاهر می‌شوند الزاماً به یک شکل معنا ندارند.

یک خطای واقعی می‌تواند به این شکل باشد:

```text
make: *** [target] Error 1
```

اما ممکن است یک Makefile از قبل انتظار یک failure خاص را داشته باشد و آن را بی‌اهمیت بداند. در این حالت ممکن است پیام شبیه این دیده شود:

```text
make: *** [target] Error 1 (ignored)
```

وجود `(ignored)` نشان می‌دهد که آن خطا طبق منطق Makefile قرار نیست باعث شکست نهایی build شود.

در packageهای بزرگ نیز GNU `make` ممکن است بارها خودش را فراخوانی کند. در این حالت شماره‌هایی مانند:

```text
make[1]
make[2]
make[3]
```

نشان‌دهنده‌ی nesting مختلف `make` هستند.

برای پیدا کردن علت اصلی، نباید از آخرین `Error` شروع کنید. معمولاً خطای واقعی چند خط یا چند مرحله قبل، در اولین compiler error دیده می‌شود.

برای مثال ممکن است خروجی نهایی چیزی شبیه این باشد:

```text
compiler error involving file.c
make[3]: *** [file.o] Error 1
make[3]: Leaving directory ...
make[2]: *** [all] Error 2
make[1]: *** [all-recursive] Error 1
make: *** [all] Error 2
```

در چنین حالتی تقریباً تمام `make` errorهای پایین‌دست نتیجه‌ی همان compiler error اولیه هستند.

بنابراین روش درست این است که ابتدا اولین خطای واقعی compiler یا preprocessor را پیدا کنید و همان را بررسی کنید، نه اینکه صرفاً آخرین `Error 2` را دنبال کنید.

#### 16.6.1 Specific Errors

##### Conflicting Types

ممکن است compiler خطایی درباره‌ی `conflicting types` برای یک symbol گزارش کند و هم‌زمان به declaration قبلی آن در header اشاره کند.

برای مثال اگر program یک identifier را با type متفاوت دوباره declare کرده باشد، compiler متوجه ناسازگاری می‌شود.

در این وضعیت باید declarationهای مربوط به symbol را بررسی کنید و مشخص کنید کدام declaration نادرست یا اضافی است.

راه‌حل ممکن است حذف declaration اضافی یا استفاده از conditional compilation مانند `#ifdef` باشد؛ بسته به ساختار source code.

##### Undeclared `time_t`

یکی از نمونه‌های مهم خطا زمانی است که compiler نوعی مانند:

```text
time_t
```

را نمی‌شناسد.

اغلب چنین خطایی به این معناست که header موردنیاز include نشده است.

مثلاً ممکن است در source code متغیری مانند این داشته باشید:

```c
time_t v1;
```

و بعد آن را با تابع `time()` استفاده کنید.

برای پیدا کردن header صحیح، یکی از بهترین روش‌ها استفاده از manual pageهاست.

برای مثال می‌توان بررسی کرد:

```bash
man 2 time
```

یا:

```bash
man 3 time
```

در بخش `SYNOPSIS` معمولاً header موردنیاز مشخص می‌شود. در مثال کتاب، استفاده از `time()` به:

```c
#include <time.h>
```

نیاز دارد.

بنابراین باید این header را در ابتدای source file قرار دهید و دوباره build را امتحان کنید.

این روش عمومی‌تر از حدس زدن است: وقتی compiler یک function یا type ناشناخته را گزارش می‌کند، به manual page مربوط به همان API مراجعه کنید و header موردنیاز را از `SYNOPSIS` پیدا کنید.

##### Missing Header File

خطای دیگری که ممکن است با آن روبه‌رو شوید چیزی شبیه این است:

```text
src.c:4: pkg.h: No such file or directory
```

در این حالت preprocessor نتوانسته header موردنظر را پیدا کند.

چند علت محتمل وجود دارد.

ممکن است source code به یک library وابسته باشد که هنوز نصب نشده است. ممکن است خود library نصب باشد ولی development package آن وجود نداشته باشد. یا ممکن است header در مسیر nonstandard قرار داشته باشد و compiler از آن directory خبر نداشته باشد.

در حالت سوم معمولاً باید include path را با `-I` به `CPPFLAGS` اضافه کنید:

```bash
CPPFLAGS="-I/path/to/include" ./configure
```

اما ممکن است علاوه بر headerها، libraryهای همان dependency نیز در مسیر غیر استاندارد باشند و در آن صورت linker نیز به یک `-L` نیاز داشته باشد.

بنابراین missing header گاهی فقط یک include problem نیست و می‌تواند نشانه‌ی dependency ناقص باشد.

اگر مطمئن نیستید کدام package آن header را فراهم می‌کند، می‌توان از ابزارهای جست‌وجوی package در distribution استفاده کرد.

در Debian-based systems کتاب به `apt-file` اشاره می‌کند:

```bash
apt-file search pkg.h
```

و در سیستم‌هایی که از yum استفاده می‌کنند:

```bash
yum provides '*/pkg.h'
```

این ابزارها می‌توانند مشخص کنند کدام development package فایل header موردنظر را ارائه می‌کند.

اگر نه library missing باشد و نه include path مشکل داشته باشد، احتمال دیگری وجود دارد: شاید source code اساساً برای operating system یا platform فعلی طراحی نشده باشد. در چنین شرایطی باید `Makefile`، `README` و documentation package را برای platformهای پشتیبانی‌شده بررسی کرد.

##### Command Not Found

یک خطای دیگر هنگام build ممکن است از طرف `make` دیده شود:

```text
make: prog: Command not found
```

این پیام یعنی build process می‌خواهد برنامه‌ای با نام `prog` را اجرا کند ولی آن را پیدا نمی‌کند.

اگر `prog` چیزی مانند این‌ها باشد:

```text
cc
gcc
ld
```

احتمال دارد ابزارهای development روی سیستم نصب نشده باشند.

در این شرایط ابتدا باید مطمئن شوید compiler و سایر ابزارهای build نصب هستند.

اما ممکن است program موردنظر روی سیستم نصب باشد و مشکل فقط در `PATH` یا نحوه‌ی reference کردن آن باشد. در چنین حالتی می‌توان در Makefile مسیر کامل executable را مشخص کرد.

یک حالت نسبتاً نادر دیگر این است که source package برنامه‌ای را در همان directory build می‌کند و بلافاصله بعد از ساخت، آن را اجرا می‌کند؛ درحالی‌که Makefile فرض کرده directory جاری یعنی `.` داخل `PATH` قرار دارد.

اگر `.` در `PATH` نباشد، executable پیدا نمی‌شود.

در این حالت می‌توان Makefile را طوری تغییر داد که به‌جای:

```text
prog
```

از:

```text
./prog
```

استفاده کند.

راه دیگر این است که directory جاری را به‌صورت موقت به `PATH` اضافه کنید؛ هرچند این روش باید با دقت انجام شود.

---

### 16.7 Looking Forward

این فصل فقط مقدمه‌ای بر build و installation از source code بود و بعد از آشنایی با این فرآیند، چند مسیر طبیعی برای ادامه‌ی یادگیری وجود دارد.

یکی از این مسیرها یادگیری build systemهای دیگری مانند:

```text
CMake
SCons
```

است.

دانستن Autoconf فقط یک نقطه‌ی شروع است و در پروژه‌های مختلف ممکن است با build systemهای متفاوتی روبه‌رو شوید.

مسیر بعدی، استفاده از همین ابزارها برای softwareهای خودتان است. اگر software توسعه می‌دهید، انتخاب یک build system و یادگیری نحوه‌ی استفاده‌ی صحیح از آن بخشی از فرآیند development است.

موضوع مهم دیگری که کتاب پیشنهاد می‌کند، **build کردن Linux kernel** است. build system kernel با build system معمول softwareهای user space تفاوت زیادی دارد و configuration system مخصوص خودش را دارد که برای انتخاب featureها و moduleها طراحی شده است.

فرآیند build kernel در اصل قابل انجام است، اما هنگام نصب kernel جدید باید احتیاط کرد و همیشه یک kernel قدیمی و سالم را در سیستم نگه داشت تا اگر kernel جدید boot نشد، امکان بازگشت وجود داشته باشد.

در نهایت، باید با **distribution-specific source packages** نیز آشنا شد. Linux distributions معمولاً source packageهای مخصوص خودشان را نگهداری می‌کنند و ممکن است patchهای مفیدی برای رفع bug یا افزودن functionality در آن‌ها وجود داشته باشد.

در این زمینه ابزارهایی برای build خودکار packageها نیز وجود دارند؛ برای مثال در اکوسیستم Debian ابزارهایی مانند:

```text
debuild
```

و در محیط‌های RPM-based ابزارهایی مانند:

```text
mock
```

به کار می‌روند.

شناخت این سیستم‌ها کمک می‌کند از مرحله‌ی «ساخت software از source» به مرحله‌ی بالاتر یعنی «ساخت software به‌شکل package قابل مدیریت در distribution» حرکت کنیم.

---

### Summary

فصل شانزدهم فرایند واقعی build و installation یک C software را دنبال می‌کند؛ از زمانی که source archive را دریافت می‌کنیم تا زمانی که package build، test و نصب می‌شود.

ابتدا باید archive را با دقت بررسی و در یک directory مناسب extract کنیم. سپس `README` و `INSTALL` را بخوانیم و مطمئن شویم build environment و dependencyهای موردنیاز آماده هستند.

در packageهای مبتنی بر **GNU Autoconf**، `configure` ویژگی‌های سیستم را بررسی می‌کند و بر اساس آن فایل‌های build مانند `Makefile` را تولید می‌کند. بعد `make` software را build می‌کند و `make install` فایل‌های ساخته‌شده را در prefix تعیین‌شده نصب می‌کند.

گزینه‌هایی مانند `--prefix`، `--bindir` و `--libdir` محل نصب را کنترل می‌کنند، در حالی که `CPPFLAGS`، `CFLAGS` و `LDFLAGS` نحوه‌ی پیدا کردن headerها، تنظیمات compiler و مسیر libraryها را تحت تأثیر قرار می‌دهند. برای dependencyهای پیچیده نیز `pkg-config` اطلاعات موردنیاز برای compile و link کردن را از `.pc` fileها فراهم می‌کند.

در installation، تفاوت میان نصب مستقیم از source و ساخت package اهمیت زیادی دارد. `/usr/local` برای local installationهای محدود مناسب است، اما با زیاد شدن softwareهای سفارشی، ساخت package واقعی مدیریت سیستم را بسیار ساده‌تر می‌کند.

در troubleshooting نیز مهم‌ترین اصل این است که از میان خروجی طولانی `make` به آخرین خطا خیره نشویم. باید **اولین خطای واقعی compiler، preprocessor یا linker** را پیدا کنیم، چون بسیاری از errorهای بعدی فقط پیامد همان خطای اولیه هستند.

در نتیجه، build کردن software روی Linux یک زنجیره‌ی مشخص دارد: **source archive، configuration، compilation، testing، installation و در صورت نیاز packaging**؛ و شناخت هر مرحله باعث می‌شود خطاها و dependencyهای آن مرحله را دقیق‌تر پیدا و برطرف کنیم.
