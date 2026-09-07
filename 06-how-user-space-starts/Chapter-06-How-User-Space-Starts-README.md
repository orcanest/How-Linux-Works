# وقتی Kernel کارش تموم میشه، User Space شروع میشه

> خلاصه و یادداشت‌های تکنیکی از فصل ششم کتاب *How Linux Works, 3rd Edition* — Brian Ward

## مقدمه

در فصل پنجم دیدیم که Kernel چطور از Bootloader کنترل سیستم را می‌گیرد، سخت‌افزار و Driverها را آماده می‌کند و در نهایت اولین Process در User Space را اجرا می‌کند.

این نقطه بسیار مهم است، چون با اجرای اولین Process در User Space، وارد بخش قابل‌مشاهده‌تر و قابل‌تغییرتر سیستم می‌شویم. Kernel مسیر نسبتاً کنترل‌شده‌ای برای راه‌اندازی دارد، اما User Space بسیار Modularتر و قابل‌سفارشی‌سازی‌تر است. برای فهمیدن Startup در User Space هم لازم نیست وارد برنامه‌نویسی سطح پایین Kernel شویم؛ با بررسی init، سرویس‌ها، Unitها و Dependencyها می‌توانیم ببینیم سیستم بعد از Kernel چطور خودش را کامل می‌کند.

به‌صورت کلی، User Space تقریباً در این مسیر شکل می‌گیرد:

1. `init`
2. سرویس‌های ضروری و سطح پایین، مثل `udevd` و سرویس‌های logging
3. تنظیمات شبکه
4. سرویس‌های سطح میانی و بالاتر، مثل `cron` و سرویس‌های چاپ
5. Login Prompt، محیط گرافیکی و Applicationهای سطح بالا، مثل Web Serverها

این فصل بیشتر روی `init`، `systemd`، روش سنتی `System V init`، Shutdown، `initramfs` و روش‌های Emergency Boot تمرکز دارد.

---

## 📚 Table of Contents

- [Introduction to init](#introduction-to-init)
- [Identifying Your init](#identifying-your-init)
- [systemd](#systemd)
  - [Units and Unit Types](#units-and-unit-types)
  - [Booting and Unit Dependency Graphs](#booting-and-unit-dependency-graphs)
  - [systemd Configuration](#systemd-configuration)
  - [systemd Operation](#systemd-operation)
  - [systemd Process Tracking and Synchronization](#systemd-process-tracking-and-synchronization)
  - [systemd Dependencies](#systemd-dependencies)
  - [systemd On-Demand and Resource-Parallelized Startup](#systemd-on-demand-and-resource-parallelized-startup)
  - [systemd Auxiliary Components](#systemd-auxiliary-components)
- [System V Runlevels](#system-v-runlevels)
- [System V init](#system-v-init)
  - [System V init: Startup Command Sequence](#system-v-init-startup-command-sequence)
  - [The System V init Link Farm](#the-system-v-init-link-farm)
  - [run-parts](#run-parts)
  - [System V init Control](#system-v-init-control)
  - [systemd System V Compatibility](#systemd-system-v-compatibility)
- [Shutting Down Your System](#shutting-down-your-system)
- [The Initial RAM Filesystem](#the-initial-ram-filesystem)
- [Emergency Booting and Single-User Mode](#emergency-booting-and-single-user-mode)
- [Looking Forward](#looking-forward)

---

## Introduction to init

`init` یک برنامه در User Space است؛ یعنی برخلاف Kernel، خودش بخشی از User Space محسوب می‌شود. این برنامه اولین Process مهمی است که Kernel پس از آماده‌سازی اولیه سیستم اجرا می‌کند و به‌طور سنتی مسئول شروع و توقف سرویس‌های ضروری سیستم است.

در بسیاری از سیستم‌های مدرن Linux، `init` توسط `systemd` پیاده‌سازی شده است. با این حال `systemd` تنها گزینه نیست و در محیط‌های مختلف ممکن است پیاده‌سازی‌های دیگری از init را ببینیم.

چند نمونه مهم:

- **systemd** → پیاده‌سازی رایج در بسیاری از Distributionهای مدرن
- **System V init** → روش سنتی و ترتیبی
- **Upstart** → پیاده‌سازی قدیمی‌تر که در نسخه‌های قدیمی Ubuntu استفاده می‌شد
- **runit** → در بعضی سیستم‌های سبک
- **initهای اختصاصی** → در برخی محیط‌های Embedded و سیستم‌هایی مثل Android

### مشکل اصلی System V init چه بود؟

در System V init، راه‌اندازی سیستم عمدتاً با مجموعه‌ای از Scriptها انجام می‌شد. این Scriptها معمولاً یکی پس از دیگری و به ترتیب مشخص اجرا می‌شدند و هرکدام یک Service را راه‌اندازی یا بخشی از سیستم را Configure می‌کردند.

این روش چند مزیت داشت:

- ساده و قابل فهم بود.
- Dependencyهای ساده را می‌شد نسبتاً راحت مدیریت کرد.
- برای Startupهای خاص می‌شد Scriptها را تغییر داد.
- ساختار کلی آن برای Administrator قابل مشاهده بود.

اما با بزرگ‌تر شدن سیستم‌ها، محدودیت‌های آن بیشتر مشخص شد.

### 1. مشکل Performance

وقتی Startup به‌شکل ترتیبی انجام شود، دو بخش مستقل از Boot معمولاً نمی‌توانند هم‌زمان اجرا شوند.

مثلاً اگر Service B هیچ وابستگی‌ای به Service A نداشته باشد، باز هم در یک Startup ترتیبی ممکن است مجبور شود منتظر تمام شدن A بماند.

نتیجه:

```text
Service A
   ↓
Service B
   ↓
Service C
```

در حالی که ممکن است واقعاً بتوان چنین ساختاری داشت:

```text
        ┌── Service B
Service A
        └── Service C
```

یعنی B و C بتوانند مستقل‌تر و هم‌زمان‌تر اجرا شوند.

### 2. مشکل مدیریت Processها

Scriptهای Startup معمولاً Daemonها را اجرا می‌کنند، اما بعد از اجرا همیشه مدیریت Process ساده نیست.

برای پیدا کردن PID یک Service ممکن است مجبور شویم:

- از `ps` استفاده کنیم.
- از ابزار مخصوص همان Service کمک بگیریم.
- به فایل‌هایی مانند PID file تکیه کنیم.

این موضوع وقتی Service چند Process داشته باشد یا Daemon خودش Processهای جدید ایجاد کند، پیچیده‌تر می‌شود.

### 3. Boilerplate زیاد در Scriptها

بسیاری از Startup Scriptها بخش‌هایی از کد مشابه دارند. این Boilerplate باعث می‌شود بعضی Scriptها طولانی‌تر و سخت‌تر برای خواندن و نگهداری شوند.

### 4. نبودن مفهوم قوی On-Demand Service

در مدل سنتی، بسیاری از Serviceها از همان ابتدای Boot اجرا می‌شدند، حتی اگر تا مدت زیادی کسی به آن‌ها نیاز نداشت.

ایده‌ی Serviceهای On-Demand می‌گوید:

> اگر یک Service فقط زمانی لازم است که یک Resource خاص مورد استفاده قرار بگیرد، لازم نیست حتماً از ابتدای Boot اجرا شود.

ابزارهایی مانند `inetd` در گذشته بخشی از این ایده را برای Network Serviceها پیاده می‌کردند.

پیاده‌سازی‌های جدیدتر init سعی کردند این مشکلات را با تغییر در نحوه‌ی Startup، Supervision و Dependency Management حل کنند.

---

## Identifying Your init

برای تشخیص اینکه سیستم از چه initای استفاده می‌کند، می‌توان ابتدا Manual مربوط به `init(1)` را بررسی کرد.

همچنین می‌توان ساختار سیستم را بررسی کرد.

اگر این مسیرها وجود داشته باشند:

```text
/usr/lib/systemd
/etc/systemd
```

احتمالاً سیستم از `systemd` استفاده می‌کند.

اگر مسیر:

```text
/etc/init
```

وجود داشته باشد و داخل آن چند فایل `.conf` دیده شود، احتمال استفاده از **Upstart** وجود دارد.

اگر موارد بالا وجود نداشته باشند ولی:

```text
/etc/inittab
```

وجود داشته باشد، احتمالاً سیستم از **System V init** استفاده می‌کند.

> این روش‌ها نشانه‌های عملی هستند؛ برای تشخیص دقیق، بهتر است خود Process شماره 1 و لینک `init` را نیز بررسی کنیم.

مثلاً:

```bash
ps -p 1 -o pid,comm,args
```

و:

```bash
readlink -f /sbin/init
```

---

# systemd

`systemd` یکی از مهم‌ترین نسل‌های جدید init در Linux است.

هدف آن فقط راه‌اندازی چند Service در هنگام Boot نیست. systemd تلاش می‌کند بخش‌های مختلف مدیریت User Space را با یک مدل یکپارچه‌تر مدیریت کند.

از جمله:

- Serviceها
- Filesystem Mountها
- Socketها
- Timerها
- Dependencyها
- Process Tracking
- بعضی سرویس‌های کمکی

systemd از نظر ایده، تا حدی از `launchd` در Apple الهام گرفته است.

یکی از تفاوت‌های مهم systemd با init سنتی این است که systemd می‌تواند Processهای مرتبط با یک Service را بهتر Track کند و آن‌ها را در یک ساختار مدیریتی مشترک قرار دهد.

همچنین systemd به‌جای اینکه یک Sequence ثابت از Scriptها داشته باشد، بیشتر بر اساس **Unitها و Dependencyها** کار می‌کند.

وقتی یک Unit فعال می‌شود، systemd Dependencyهای آن را نیز بررسی و در صورت نیاز فعال می‌کند.

در نتیجه Startup الزاماً به شکل:

```text
A → B → C → D
```

انجام نمی‌شود؛ Unitهای مستقل می‌توانند در صورت آماده بودن، هم‌زمان شروع شوند.

حتی بعد از Boot هم systemd می‌تواند در واکنش به Eventهای سیستم، Unitهای دیگری را فعال کند.

---

## Units and Unit Types

یکی از ایده‌های اصلی systemd مفهوم **Unit** است.

Unit را می‌توان یک واحد مدیریتی برای یک Task یا Resource در سیستم در نظر گرفت.

هر نوع Unit یک **Unit Type** دارد و هر Unit معمولاً Configuration مخصوص خودش را دارد.

مهم‌ترین Unit Typeهایی که در Startup زیاد دیده می‌شوند:

### Service Unit

برای مدیریت Daemonها و Serviceهای سیستم استفاده می‌شود.

مثلاً:

```text
sshd.service
```

### Target Unit

برای گروه‌بندی Unitهای دیگر و تعریف یک وضعیت یا هدف کلی استفاده می‌شود.

Target خودش لزوماً یک Process را اجرا نمی‌کند؛ بیشتر نقش یک نقطه‌ی گروه‌بندی و هماهنگ‌کننده را دارد.

### Socket Unit

یک Network Socket یا محل دریافت Connection را نمایندگی می‌کند.

یکی از کاربردهای مهم Socket Unit این است که systemd بتواند Service را فقط وقتی که واقعاً Connection دریافت شده است فعال کند.

### Mount Unit

اتصال یک Filesystem به یک Mount Point را نمایندگی می‌کند.

بنابراین systemd فقط Processها را مدیریت نمی‌کند؛ Resourceهای مختلف سیستم نیز می‌توانند در قالب Unit وارد مدل مدیریتی آن شوند.

لیست کامل Unit Typeها را می‌توان در Manual مربوط به `systemd(1)` مشاهده کرد.

---

## Booting and Unit Dependency Graphs

در هنگام Boot، معمولاً یک Unit سطح بالا فعال می‌شود که اغلب:

```text
default.target
```

نام دارد.

این Unit معمولاً به Target یا Unitهای دیگری وابسته است و از طریق آن‌ها بخش‌های مختلف سیستم فعال می‌شوند.

یک نکته مهم این است که Dependencyهای systemd را نباید یک Tree ساده تصور کنیم.

ممکن است:

```text
A → B
A → C
B → D
C → D
```

داشته باشیم.

در این حالت چند Branch دوباره به یک Unit مشترک می‌رسند.

بنابراین ساختار واقعی بیشتر شبیه یک **Graph** است.

برای مشاهده و تولید Dependency Graph می‌توان از:

```bash
systemd-analyze dot
```

استفاده کرد.

Graph کامل یک سیستم واقعی می‌تواند بسیار بزرگ باشد و برای همین معمولاً بهتر است آن را Filter کنیم یا فقط بخش کوچکی از Dependencyها را بررسی کنیم.

در یک نمونه ساده می‌توان ساختار را به‌شکل زیر تصور کرد:

```text
default.target
      │
      ▼
multi-user.target
      │
      ├── basic.target
      │      │
      │      └── sysinit.target
      │
      ├── cron.service
      │
      └── dbus.service
```

نکته مهم این است که این فقط یک مدل ساده‌شده است و سیستم واقعی Dependencyهای بسیار بیشتری دارد.

در بسیاری از سیستم‌ها `default.target` خودش به یک Target سطح بالاتر، مانند Target مربوط به محیط گرافیکی، Link شده است.

**Figure 6-1**

---

## systemd Configuration

فایل‌های Configuration مربوط به systemd در چندین Directory قرار دارند.

دو مسیر مهم عبارت‌اند از:

```text
/usr/lib/systemd/system
```

یا در بعضی سیستم‌ها:

```text
/lib/systemd/system
```

که محل Unitهای ارائه‌شده توسط Distribution و Packageهاست.

و:

```text
/etc/systemd/system
```

که محل Definitionها و Overrideهای محلی سیستم است.

### قانون مهم

برای تغییرات محلی، معمولاً باید `/etc/systemd/system` را ترجیح داد.

نباید Unitهای ارائه‌شده توسط Distribution را مستقیماً در مسیرهای Package-managed تغییر دهیم، چون Updateهای سیستم ممکن است تغییرات را جایگزین کنند.

برای دیدن Search Path واقعی systemd می‌توان از:

```bash
systemctl -p UnitPath show
```

استفاده کرد.

برای دیدن Directoryهای اصلی Unitها نیز می‌توان از:

```bash
pkg-config systemd --variable=systemdsystemunitdir
```

و:

```bash
pkg-config systemd --variable=systemdsystemconfdir
```

استفاده کرد.

### Unit File

ساختار Unit Fileها از مشخصات XDG Desktop Entry الهام گرفته شده و از Sectionهایی با نام داخل `[]` استفاده می‌کند.

یک Service Unit ممکن است چیزی شبیه این داشته باشد:

```ini
[Unit]
Description=Example Service
Requires=example.socket

[Service]
ExecStart=/usr/bin/example
```

Section مهم `Unit` معمولاً شامل:

- Description
- Dependencyها
- Relationshipهای Unit

است.

Section `Service` هم اطلاعات مربوط به اجرای Service را مشخص می‌کند، مثل:

- `ExecStart`
- `ExecStartPre`
- `ExecReload`
- و سایر گزینه‌های مرتبط با اجرای Process

Unit Fileها بسته به Distribution ممکن است کمی با هم تفاوت داشته باشند؛ مثلاً نام بعضی Serviceها در Fedora و Ubuntu متفاوت است.

### Variables در Unit File

در Unit Fileها ممکن است Variableهایی با `$` دیده شوند.

مثلاً:

```ini
ExecStart=/usr/sbin/sshd -D $OPTIONS $CRYPTO_POLICY
```

این Variableها ممکن است از `EnvironmentFile` بیایند.

مثلاً:

```ini
EnvironmentFile=/etc/sysconfig/sshd
```

یک Variable مهم دیگر:

```text
$MAINPID
```

است که به Process اصلی Track‌شده توسط systemd اشاره می‌کند.

مثلاً:

```ini
ExecReload=/bin/kill -HUP $MAINPID
```

این مدل برای Reload کردن Configuration یک Daemon بسیار رایج است.

### Specifierها

Specifierها با `%` شروع می‌شوند.

مثلاً:

```text
%n
```

می‌تواند نام Unit فعلی را مشخص کند.

و:

```text
%H
```

نام Host را نشان می‌دهد.

یکی از کاربردهای مهم Specifierها، ساخت Unitهای Instance-based است.

برای مثال:

```text
getty@.service
```

می‌تواند برای ساخت Instanceهای مختلف استفاده شود:

```text
getty@tty1
getty@tty2
```

بخشی که بعد از `@` می‌آید Instance نام دارد.

Specifierهایی مانند:

```text
%i
%I
```

برای استفاده از Instance در Unit File کاربرد دارند.

---

## systemd Operation

ابزار اصلی تعامل با systemd:

```bash
systemctl
```

است.

### مشاهده Unitهای فعال

```bash
systemctl list-units
```

این دستور Unitهای فعال را نمایش می‌دهد.

برای دیدن نام‌های کامل:

```bash
systemctl list-units --full
```

و برای دیدن Unitهای بیشتری که فقط به Active محدود نیستند:

```bash
systemctl list-units --all
```

### مشاهده وضعیت یک Unit

مثلاً:

```bash
systemctl status sshd.service
```

در خروجی می‌توان اطلاعاتی مانند این‌ها را دید:

- وضعیت Load
- وضعیت Active
- زمان Start
- Main PID
- تعداد Taskها
- CGroup
- Processهای مرتبط
- برخی Logهای اخیر

برای نمونه:

```text
Loaded: loaded (...; enabled)
Active: active (running)
Main PID: ...
Tasks: ...
CGroup: ...
```

### CGroup

systemd می‌تواند Processهای یک Service را در یک Control Group قرار دهد.

در نتیجه در Status ممکن است ساختاری مانند این ببینیم:

```text
/system.slice/sshd.service
```

و Processهای مرتبط با Service زیر آن نمایش داده شوند.

برای مشاهده‌ی CGroupهای systemd می‌توان از:

```bash
systemd-cgls
```

استفاده کرد.

### مشاهده Logهای یک Unit

برای دیدن پیام‌های مرتبط با یک Unit:

```bash
journalctl --unit=unit_name
```

استفاده می‌شود.

### Start / Stop / Restart

برای فعال کردن Unit:

```bash
systemctl start unit
```

برای متوقف کردن:

```bash
systemctl stop unit
```

برای Restart:

```bash
systemctl restart unit
```

### Reload

دو مفهوم مهم را باید از هم جدا کنیم.

اگر Service از Reload پشتیبانی کند:

```bash
systemctl reload unit
```

فقط Configuration همان Service را Reload می‌کند.

اما اگر خود Unit File را تغییر داده باشیم:

```bash
systemctl daemon-reload
```

باعث می‌شود systemd Configuration مربوط به Unitها را دوباره بخواند.

### Jobها

در systemd، درخواست‌هایی مانند Start، Stop و Restart به‌عنوان **Job** مدیریت می‌شوند.

برای دیدن Jobهای فعلی:

```bash
systemctl list-jobs
```

در زمان Boot ممکن است چند Job در حالت `waiting` باشند، چون منتظر یک Unit دیگر هستند.

نکته مهم:

> Job در systemd با Job Control مربوط به Shell یکی نیست.

### اضافه کردن Unit

Unitهای شخصی را معمولاً در:

```text
/etc/systemd/system
```

قرار می‌دهیم.

بعد می‌توانیم آن را فعال کنیم:

```bash
systemctl start unit
```

اگر Unit دارای `[Install]` باشد، برای فعال شدن خودکار در Boot معمولاً از:

```bash
systemctl enable unit
```

استفاده می‌کنیم.

این دو مفهوم را نباید یکی دانست:

```text
start   → همین الان فعال کن
enable  → برای Startupهای بعدی تنظیم کن
```

بنابراین:

```bash
systemctl start myservice
```

به‌تنهایی به معنی Start شدن Service در Boot بعدی نیست.

و:

```bash
systemctl enable myservice
```

هم به‌تنهایی الزاماً Service را همان لحظه Start نمی‌کند.

برای حذف یا غیرفعال کردن:

```bash
systemctl stop unit
systemctl disable unit
```

و در صورت نیاز بعد از آن می‌توان Unit File را حذف کرد.

---

## systemd Process Tracking and Synchronization

یکی از مشکلات قدیمی Service Management این است که Serviceها همیشه به یک شکل اجرا نمی‌شوند.

یک Service ممکن است:

- مستقیماً Process اصلی را نگه دارد.
- `fork()` کند.
- خودش Daemonize شود.
- Processهای فرزند ایجاد کند.
- Process اولیه را تمام کند و Process دیگری را ادامه دهد.

systemd برای Track کردن این ساختار از **cgroups** استفاده می‌کند.

به همین دلیل لازم نیست systemd فقط به PID اولیه‌ی یک Process تکیه کند.

### Type=simple

در این حالت Process اصلی Service همان Processی است که systemd انتظار دارد Service را اجرا کند و Process اصلی Fork نمی‌کند.

```ini
[Service]
Type=simple
```

مشکل این مدل این است که systemd لزوماً نمی‌داند Service چه زمانی واقعاً آماده‌ی ارائه‌ی سرویس است.

ممکن است Process شروع شده باشد، اما هنوز Initialization آن تمام نشده باشد.

### Type=forking

در این مدل Service Fork می‌کند و Process اولیه پایان پیدا می‌کند.

```ini
[Service]
Type=forking
```

systemd انتظار دارد Process اولیه بعد از Fork شدن پایان پیدا کند و این پایان را به‌عنوان نشانه‌ای از آماده شدن Service در نظر می‌گیرد.

### Type=notify

در این مدل خود Service می‌تواند وقتی آماده شد، به systemd اطلاع دهد.

```ini
[Service]
Type=notify
```

این روش برای Serviceهایی که می‌توانند دقیقاً لحظه‌ی آماده شدن خود را اعلام کنند، مناسب‌تر است.

### Type=dbus

در این مدل آماده شدن Service با ثبت شدن آن روی D-Bus مرتبط می‌شود.

```ini
[Service]
Type=dbus
```

### Type=oneshot

در `oneshot` هدف اجرای یک Process برای انجام یک کار مشخص است و Process بعد از انجام کار پایان پیدا می‌کند.

```ini
[Service]
Type=oneshot
```

در این حالت systemd Service را تا پایان Process، Started محسوب نمی‌کند.

برای `oneshot` معمولاً رفتار پیش‌فرض `RemainAfterExit=yes` هم مطرح است؛ بنابراین بعد از پایان Process، Unit می‌تواند همچنان Active در نظر گرفته شود.

### Type=idle

این حالت شبیه `simple` است، اما systemd اجرای Service را تا پایان Jobهای فعال به تعویق می‌اندازد.

ایده این است که Serviceهای دیگر فرصت کنند Startup خود را انجام دهند تا چند Service هم‌زمان روی خروجی یکدیگر مزاحمت ایجاد نکنند.

---

## systemd Dependencies

Dependencyها بخش بسیار مهمی از systemd هستند.

اگر Dependencyها بیش از حد سخت‌گیرانه باشند، یک Failure کوچک می‌تواند بخش بزرگی از سیستم را از کار بیندازد.

مثلاً فرض کنید Login Prompt را به Database Server وابسته کنیم و بگوییم Database حتماً باید موفق شود.

اگر Database خراب شود، Login Prompt هم ممکن است بالا نیاید و در نتیجه حتی نتوانیم وارد سیستم شویم تا مشکل Database را برطرف کنیم.

به همین دلیل systemd چند نوع Dependency دارد.

### Requires

Dependency سخت‌گیرانه است.

اگر Unit اصلی به Unit دیگری با `Requires` وابسته باشد، systemd تلاش می‌کند Dependency را فعال کند.

اگر Dependency Fail شود، Unit وابسته نیز ممکن است Deactivate شود.

```ini
[Unit]
Requires=example.service
```

### Wants

Dependency ضعیف‌تر است.

systemd تلاش می‌کند Unit موردنظر را فعال کند، اما Failure آن معمولاً باعث Failure Unit اصلی نمی‌شود.

```ini
[Unit]
Wants=example.service
```

این مدل برای ساختن سیستم‌های Fault-Tolerant مناسب‌تر است.

به‌طور کلی وقتی Dependency برای ادامه‌ی کار کاملاً حیاتی نیست، `Wants` معمولاً انتخاب انعطاف‌پذیرتری است.

### Requisite

در این حالت Dependency باید از قبل Active باشد.

اگر Dependency فعال نباشد، Activation Unit اصلی Fail می‌شود.

```ini
[Unit]
Requisite=example.service
```

### Conflicts

یک Dependency منفی است.

اگر Unit اصلی فعال شود، Unit متعارض در صورت Active بودن Deactivate می‌شود.

هم‌زمان فعال بودن Unitهای متعارض امکان‌پذیر نیست.

```ini
[Unit]
Conflicts=example.service
```

### مشاهده Dependencyها

می‌توان Dependencyهای مشخص یک Unit را با:

```bash
systemctl show -p Wants unit
```

یا:

```bash
systemctl show -p Requires unit
```

بررسی کرد.

---

### Ordering

Dependency و Ordering یکی نیستند.

مثلاً اگر بنویسیم:

```ini
Wants=foo.service
```

به این معنی نیست که همیشه باید اول `foo.service` کامل شود و بعد Unit فعلی اجرا شود.

Dependency فقط می‌گوید:

> این Unit را هم فعال کن.

اما برای تعیین ترتیب از:

```text
Before=
After=
```

استفاده می‌کنیم.

### Before

می‌گوید Unit فعلی باید قبل از Unit مشخص‌شده فعال شود.

```ini
Before=bar.target
```

### After

می‌گوید Unit فعلی باید بعد از Unit مشخص‌شده فعال شود.

```ini
After=bar.target
```

بنابراین می‌توانیم این دو مفهوم را جدا کنیم:

```text
Dependency
→ چه Unitهایی باید فعال شوند؟

Ordering
→ این Unitها با چه ترتیبی فعال شوند؟
```

این جداسازی یکی از دلایل مهم Parallel بودن Startup در systemd است.

---

### Default and Implicit Dependencies

همه‌ی Dependencyها الزاماً در Unit File نوشته نمی‌شوند.

systemd می‌تواند بعضی Dependencyها را به‌صورت خودکار اضافه کند.

این Dependencyهای داخلی برای جلوگیری از اشتباهات رایج و کوچک نگه داشتن Unit Fileها ایجاد می‌شوند.

مثلاً در بعضی شرایط، systemd در کنار یک `Wants`، Ordering مناسب مانند `After` را نیز در نظر می‌گیرد.

این Dependencyهای خودکار به Unit Type وابسته‌اند.

در صورت نیاز می‌توان Default Dependencyها را با:

```ini
DefaultDependencies=no
```

غیرفعال کرد.

---

### Conditional Dependencies

systemd فقط Dependencyهای Unit-based ندارد.

می‌توان فعال شدن یک Unit را به وضعیت Filesystem یا سیستم نیز وابسته کرد.

مثلاً:

```ini
ConditionPathExists=/some/path
```

فقط در صورتی True است که Path وجود داشته باشد.

یا:

```ini
ConditionPathIsDirectory=/some/path
```

که وجود Directory را بررسی می‌کند.

و:

```ini
ConditionFileNotEmpty=/some/file
```

که بررسی می‌کند File وجود داشته باشد و خالی نباشد.

اگر Condition مربوط به Unit هنگام Activation برقرار نباشد، آن Unit فعال نمی‌شود.

نکته مهم این است که False بودن یک Condition الزاماً Dependencyهای دیگر آن Unit را متوقف نمی‌کند؛ Condition مربوط به همان Unit بررسی می‌شود.

---

### `[Install]` و Enable کردن Unit

Dependency را می‌توان از سمت Dependency نیز تعریف کرد.

مثلاً:

```ini
[Install]
WantedBy=test2.target
```

در این مدل، وقتی Unit Enable می‌شود، systemd می‌تواند یک Symbolic Link در Directory مربوط به `.wants` ایجاد کند.

مثلاً:

```text
/etc/systemd/system/test2.target.wants/test1.target
```

به:

```text
/etc/systemd/system/test1.target
```

اشاره می‌کند.

فعال کردن:

```bash
systemctl enable test1.target
```

باعث ایجاد این Link می‌شود.

غیرفعال کردن:

```bash
systemctl disable test1.target
```

Link را حذف می‌کند.

نکته بسیار مهم:

```text
enable ≠ start
```

`enable` برای Bootهای آینده Configuration ایجاد می‌کند، اما به‌تنهایی Unit را همان لحظه Active نمی‌کند.

همچنین Unit Fileهایی که `[Install]` ندارند ممکن است به‌شکل Implicit فعال شوند و `disable` روی آن‌ها اثر مورد انتظار را نداشته باشد.

---

## systemd On-Demand and Resource-Parallelized Startup

یکی از قابلیت‌های مهم systemd این است که می‌تواند Startup یک Service را تا زمانی که واقعاً لازم نشده به تأخیر بیندازد.

ایده را می‌توان با دو Unit تصور کرد:

```text
Unit A → Service
Unit R → Resource
```

مثلاً Resource می‌تواند:

- Network Socket
- File
- Device

باشد.

### روند کلی

1. یک Unit برای Service ایجاد می‌کنیم.
2. Resource مورد استفاده‌ی Service را مشخص می‌کنیم.
3. برای Resource یک Unit ایجاد می‌کنیم.
4. Resource Unit را به Service Unit مرتبط می‌کنیم.

بعد:

1. systemd Resource Unit را فعال و Resource را Monitor می‌کند.
2. یک Client به Resource دسترسی پیدا می‌کند.
3. systemd Resource را در اختیار می‌گیرد و در صورت نیاز Input را Buffer می‌کند.
4. systemd Service Unit را فعال می‌کند.
5. Service آماده می‌شود.
6. Service کنترل Resource را تحویل می‌گیرد و از Input موجود استفاده می‌کند.

این مدل شباهت زیادی به ایده‌ی قدیمی `inetd` و `xinetd` دارد.

### نکته‌های مهم

Resource Unit باید تمام Resourceهایی را که Service ارائه می‌کند پوشش دهد.

همچنین Resource Unit باید به Service درست متصل باشد.

همه‌ی Serviceها نیز الزاماً نمی‌توانند با تمام روش‌های Resource Handoff کار کنند.

systemd علاوه بر Socket Unit، مفاهیمی مانند:

```text
socket unit
path unit
device unit
automount unit
```

را نیز دارد.

---

### یک مثال: Socket Unit و Echo Service

یک Echo Service ساده را تصور کنیم که روی TCP Port زیر Listen می‌کند:

```text
22222
```

Socket Unit:

```ini
[Unit]
Description=echo socket

[Socket]
ListenStream=22222
Accept=true
```

اینجا Socket مسئول Listen کردن روی Port است.

Service مربوطه می‌تواند:

```text
echo@.service
```

باشد:

```ini
[Unit]
Description=echo service

[Service]
ExecStart=/bin/cat
StandardInput=socket
```

ارتباط بین Socket و Service در این مثال از طریق Naming Convention انجام می‌شود.

چون هر دو Prefix مشابه دارند:

```text
echo.socket
echo@.service
```

systemd می‌داند که فعالیت روی Socket باید باعث Activation Service مربوطه شود.

برای شروع:

```bash
systemctl start echo.socket
```

بعد می‌توان با یک Client به Port متصل شد.

مثلاً:

```bash
telnet localhost 22222
```

اگر Client چیزی بفرستد، Service همان داده را برمی‌گرداند.

برای متوقف کردن:

```bash
systemctl stop echo.socket
```

`Accept=true` باعث می‌شود systemd Connectionهای ورودی را قبول کند و برای هر Connection یک Instance جداگانه از Service ایجاد کند.

به همین دلیل Unit Service به‌شکل:

```text
echo@.service
```

تعریف شده است.

اگر Service خودش مسئول Accept کردن Connection باشد، مدل متفاوتی استفاده می‌شود و لزوماً نباید از `@` و `Accept=true` استفاده کرد.

برای جزئیات دقیق‌تر Resource Handoff باید Manualهای مربوط به این Unitها مانند `systemd.socket(5)`، `systemd.path(5)` و `systemd.device(5)` را بررسی کرد.

---

## Boot Optimization with Auxiliary Units

همین ایده‌ی Resource Unit فقط برای On-Demand Serviceها نیست.

systemd می‌تواند از آن برای سریع‌تر کردن Boot نیز استفاده کند.

فرض کنید:

```text
Service E
```

یک Resource مهم:

```text
Resource R
```

را ارائه می‌دهد و Serviceهای:

```text
A
B
C
```

به آن Resource نیاز دارند.

در یک سیستم ترتیبی ممکن است:

```text
Start E
   ↓
E ready
   ↓
Start A
   ↓
A ready
   ↓
Start B
   ↓
B ready
   ↓
Start C
```

باشیم.

در نتیجه Startup طولانی می‌شود.

**Figure 6-2**

در systemd می‌توان Resource Unit را خیلی سریع‌تر در دسترس قرار داد و Serviceهای مستقل را هم‌زمان Start کرد:

```text
          ┌── Unit A
          │
Unit R ───┼── Unit B
          │
          ├── Unit C
          │
          └── Unit E
```

در این مدل Unitهای A، B، C و E می‌توانند هم‌زمان Startup خود را شروع کنند.

Unit R Resource را در اختیار قرار می‌دهد و وقتی Unit E آماده شد، E کنترل Resource را تحویل می‌گیرد.

نکته جالب این است که A، B یا C ممکن است اصلاً در طول Startup به Resource نیاز پیدا نکنند؛ اما این مدل اجازه می‌دهد در صورت نیاز، Resource از همان ابتدا در دسترس باشد.

**Figure 6-3**

این روش می‌تواند Boot را سریع‌تر کند، اما یک Trade-off هم دارد:

> اگر Unitهای زیادی هم‌زمان شروع شوند، ممکن است سیستم برای مدت کوتاهی به‌دلیل حجم زیاد فعالیت کندتر شود.

نمونه‌های واقعی این ایده را می‌توان در بعضی Unitهای مربوط به `systemd-journald` و D-Bus مشاهده کرد.

---

## systemd Auxiliary Components

systemd در طول زمان فقط به یک init system ساده محدود نمانده و اجزای دیگری نیز در کنار آن قرار گرفته‌اند.

چند نمونه مهم:

### systemd-udevd

`udevd` که در فصل Devices دیدیم، در سیستم‌های systemd به‌عنوان بخشی از مجموعه systemd ارائه می‌شود.

نام Process معمولاً:

```text
systemd-udevd
```

است.

### systemd-journald

سرویس Logging systemd است.

این سرویس پیام‌های مختلف سیستم را جمع‌آوری و مدیریت می‌کند و می‌تواند با مدل‌های Logging سنتی Unix نیز تعامل داشته باشد.

جزئیات بیشتر آن در فصل بعد بررسی می‌شود.

### systemd-resolved

یک Service مربوط به Name Resolution و DNS Cache است.

این سرویس مسئول بخشی از مدیریت و Cache کردن DNS در سیستم‌هایی است که از آن استفاده می‌کنند.

### systemd-fsck

برخی برنامه‌های داخل مجموعه systemd در واقع Wrapperهایی هستند که ابزارهای سنتی سیستم را اجرا می‌کنند و نتیجه را به systemd گزارش می‌دهند.

بنابراین اگر در Directoryهایی مانند:

```text
/lib/systemd
```

یا مسیرهای متناظر یک Distribution، برنامه‌ای را دیدیم که نمی‌شناسیم، Manual مربوط به آن می‌تواند نشان دهد چه Unit یا وظیفه‌ای را تکمیل می‌کند.

---

# System V Runlevels

در System V init، وضعیت کلی سیستم با مفهوم **Runlevel** مشخص می‌شد.

Runlevel عددی بین:

```text
0 ... 6
```

بود.

سیستم معمولاً بیشتر زمان خود را در یک Runlevel مشخص می‌گذراند و هنگام Shutdown یا تغییر وضعیت، به Runlevel دیگری منتقل می‌شد.

برای مشاهده Runlevel می‌توان از:

```bash
who -r
```

استفاده کرد.

نمونه:

```text
run-level 5 ...
```

Runlevelها برای حالت‌های مختلف سیستم استفاده می‌شدند؛ از جمله:

- Startup
- Shutdown
- Single-user mode
- Console mode
- Multi-user mode
- حالت گرافیکی

در بسیاری از سیستم‌های سنتی، Runlevelهای 2 تا 4 با حالت‌های متنی مرتبط بودند و Runlevel 5 برای شروع GUI Login استفاده می‌شد.

اما این شماره‌گذاری در سیستم‌های مدرن کمتر اهمیت دارد.

systemd همچنان برای سازگاری با System V از Runlevelها پشتیبانی می‌کند، ولی مدل اصلی خودش را بر اساس **Target Unit**ها طراحی کرده است.

---

# System V init

System V init یکی از قدیمی‌ترین مدل‌های init در Linux است.

امروزه در بیشتر Desktop و Serverهای مدرن کمتر دیده می‌شود، اما ممکن است در:

- سیستم‌های قدیمی
- Embedded Linux
- بعضی Routerها
- Packageهای قدیمی دارای Init Script
- محیط‌های سازگار با System V

با آن برخورد کنیم.

یک نصب معمولی System V init دو بخش اصلی دارد:

1. یک Configuration مرکزی
2. مجموعه‌ای از Boot Scriptها به‌همراه Symbolic Link Farm

فایل مرکزی معمولاً:

```text
/etc/inittab
```

است.

مثلاً:

```text
id:5:initdefault:
```

یعنی Runlevel پیش‌فرض 5 است.

### ساختار `inittab`

هر خط `inittab` معمولاً چهار Field دارد که با `:` جدا می‌شوند:

```text
id : runlevels : action : command
```

این چهار بخش عبارت‌اند از:

1. یک Identifier کوتاه
2. Runlevelهای مرتبط
3. Action
4. Command اختیاری

مثلاً:

```text
l5:5:wait:/etc/rc.d/rc 5
```

این خط هنگام ورود به Runlevel 5، برنامه‌ی:

```text
/etc/rc.d/rc 5
```

را اجرا می‌کند و با Action نوع `wait` منتظر می‌ماند تا آن Command تمام شود.

---

## System V init: Startup Command Sequence

دستور `rc` مخفف ایده‌ی **run commands** است.

وقتی:

```text
/etc/rc.d/rc 5
```

اجرا می‌شود، Scriptهای مربوط به Runlevel 5 از Directoryهایی مانند:

```text
/etc/rc5.d
```

یا:

```text
/etc/rc.d/rc5.d
```

اجرا می‌شوند.

ممکن است مواردی مانند این ببینیم:

```text
S10sysklogd
S12kerneld
S15netstd_init
S18netbase
S99httpd
S99sshd
```

حرف اول:

```text
S
```

یعنی Service باید در حالت Start اجرا شود.

عدد بعد از آن ترتیب اجرای Script را مشخص می‌کند:

```text
S10...
S12...
S15...
...
S99...
```

هرچه عدد کوچک‌تر باشد، Script زودتر اجرا می‌شود.

در نتیجه چیزی شبیه این اتفاق می‌افتد:

```text
S10sysklogd start
S12kerneld start
S15netstd_init start
...
S99sshd start
```

این فایل‌ها معمولاً Scriptهایی هستند که برنامه‌های واقعی موجود در مسیرهایی مانند `/sbin` یا `/usr/sbin` را Start می‌کنند.

برای فهمیدن اینکه یک Script دقیقاً چه کاری انجام می‌دهد، می‌توان آن را با ابزارهایی مانند `less` بررسی کرد.

### K در مقابل S

در بعضی Runlevelها فایل‌هایی با `K` نیز وجود دارند:

```text
K...
```

`K` از Kill می‌آید و به معنای Stop کردن Service است.

بنابراین:

```text
S → start
K → stop
```

است.

---

## The System V init Link Farm

فایل‌های داخل `rc*.d` معمولاً خود Script اصلی نیستند.

آن‌ها Symbolic Linkهایی به Scriptهای موجود در:

```text
init.d
```

هستند.

مثلاً:

```text
S10sysklogd -> ../init.d/sysklogd
S12kerneld  -> ../init.d/kerneld
S99httpd    -> ../init.d/httpd
```

تعداد زیادی Symbolic Link در چند Directory مختلف به این شکل یک **Link Farm** می‌سازند.

مزیت آن این است که یک Script می‌تواند برای چند Runlevel مورد استفاده قرار بگیرد.

این ساختار یک Convention است، نه الزام Kernel.

### Start و Stop کردن Service

معمولاً بهتر است Service را از Script داخل `init.d` کنترل کنیم، نه مستقیماً از Link داخل `rc*.d`.

مثلاً:

```bash
/etc/init.d/httpd start
```

و:

```bash
/etc/init.d/httpd stop
```

### تغییر Boot Sequence

برای جلوگیری از اجرای یک Service در یک Runlevel، یکی از روش‌های قدیمی تغییر Link Farm است.

مثلاً به‌جای حذف کامل:

```text
S99httpd
```

می‌توان نام آن را تغییر داد:

```text
_S99httpd
```

چون Scriptهایی که با `S` یا `K` شروع نمی‌شوند توسط مکانیزم اجرای Runlevel نادیده گرفته می‌شوند.

مزیت این روش این است که نام اصلی Service هنوز قابل تشخیص است و بعداً می‌توان Link را برگرداند.

برای اضافه کردن Service نیز می‌توان Script مناسب را در `init.d` ایجاد کرد و سپس Link مناسب را در Runlevel موردنظر قرار داد.

ترتیب اجرای Service مهم است؛ اگر Service خیلی زود اجرا شود ممکن است Dependencyهای لازم هنوز آماده نباشند.

---

## run-parts

`run-parts` یک ابزار ساده است که تعدادی برنامه‌ی Executable را از یک Directory به ترتیبی قابل‌پیش‌بینی اجرا می‌کند.

می‌توان آن را تقریباً این‌طور تصور کرد:

```text
Directory
   ↓
List executable files
   ↓
Run them in order
```

در بسیاری از سیستم‌ها از `run-parts` برای اجرای مجموعه‌ای از Scriptها استفاده می‌شود.

پیاده‌سازی آن بین Distributionها ممکن است متفاوت باشد.

برخی نسخه‌ها امکانات بیشتری مانند:

- Filter کردن فایل‌ها با Regular Expression
- ارسال Argument به برنامه‌ها

دارند.

در سیستم‌های Debian و Ubuntu ممکن است نسخه‌ی پیچیده‌تری از `run-parts` وجود داشته باشد.

نکته اصلی:

> `run-parts` خودش یک Init System نیست؛ فقط ابزاری برای اجرای مجموعه‌ای از برنامه‌ها در یک Directory است.

---

## System V init Control

برای کنترل System V init از:

```bash
telinit
```

استفاده می‌شود.

مثلاً برای تغییر Runlevel:

```bash
telinit 3
```

هنگام تغییر Runlevel، init تلاش می‌کند Processهایی را که برای Runlevel جدید لازم نیستند متوقف کند؛ بنابراین تغییر Runlevel باید با دقت انجام شود.

اگر فایل:

```text
/etc/inittab
```

را تغییر دهیم، init باید از تغییرات مطلع شود.

برای Reload کردن Configuration:

```bash
telinit q
```

و برای رفتن به Single-user mode:

```bash
telinit s
```

استفاده می‌شود.

---

## systemd System V Compatibility

یکی از ویژگی‌های مهم systemd این است که می‌تواند Init Scriptهای قدیمی System V را نیز مدیریت کند.

به‌صورت مفهومی روند کار چنین است:

1. systemd Target مربوط به Runlevel را فعال می‌کند.
2. Linkهای موجود در `rc<N>.d` را بررسی می‌کند.
3. Script متناظر را در `init.d` پیدا می‌کند.
4. برای Script یک Service Unit متناظر در نظر می‌گیرد.
5. Script را با Argument مناسب `start` یا `stop` اجرا می‌کند.
6. تلاش می‌کند Processهای ایجادشده توسط Script را به همان Service Unit نسبت دهد.

مثلاً:

```text
/etc/init.d/foo
```

می‌تواند به Serviceای مانند:

```text
foo.service
```

مرتبط شود.

در نتیجه می‌توان از:

```bash
systemctl status foo.service
```

یا:

```bash
systemctl restart foo.service
```

استفاده کرد.

اما نباید انتظار داشت Compatibility Mode تمام مزایای Native systemd را ایجاد کند.

برای مثال Init Scriptهای System V همچنان ماهیت ترتیبی خود را حفظ می‌کنند و نمی‌توان صرفاً با اجرای آن‌ها از تمام Parallelization سیستم systemd بهره برد.

---

# Shutting Down Your System

Shutdown نیز بخشی از وظایف init است.

روش استاندارد برای خاموش کردن Linux استفاده از:

```bash
shutdown
```

است.

### خاموش کردن فوری

```bash
shutdown -h now
```

`-h` به Halt اشاره دارد.

### Restart

```bash
shutdown -r now
```

و برای Restart با تأخیر:

```bash
shutdown -r +10
```

یعنی سیستم پس از 10 دقیقه Restart شود.

Argument مربوط به زمان برای `shutdown` مهم است و می‌تواند `now` یا شکل‌هایی مانند `+n` باشد.

در Shutdownهای زمان‌بندی‌شده، سیستم به Userهای Login شده اطلاع می‌دهد و ممکن است `/etc/nologin` ایجاد شود تا Loginهای معمولی محدود شوند.

وقتی زمان Shutdown برسد، `shutdown` از init می‌خواهد فرآیند Shutdown را شروع کند.

در systemd این فرآیند با Unitهای Shutdown انجام می‌شود.

در System V init معمولاً سیستم به Runlevelهای:

```text
0 → halt
6 → reboot
```

می‌رود.

### مراحل کلی Shutdown

به‌صورت مفهومی:

1. init از Processها می‌خواهد تمیز خارج شوند.
2. اگر Processی پاسخ ندهد، ابتدا `TERM` ارسال می‌شود.
3. اگر هنوز Process باقی مانده باشد، `KILL` استفاده می‌شود.
4. سیستم برای Shutdown آماده می‌شود و فایل‌ها و Stateهای لازم را تثبیت می‌کند.
5. Filesystemهای غیر از Root Unmount می‌شوند.
6. Root Filesystem به حالت Read-Only Remount می‌شود.
7. داده‌های Buffer شده روی Storage نوشته می‌شوند.
8. در نهایت Kernel با `reboot(2)` دستور Halt یا Reboot را اجرا می‌کند.

در این مسیر ابزارهایی مانند:

```text
reboot
halt
poweroff
```

نیز می‌توانند در نهایت همین فرآیند را تحریک کنند.

### Shutdown اجباری

برخی ابزارهای Shutdown گزینه‌ی:

```text
-f
```

دارند که Force را فعال می‌کند.

این گزینه می‌تواند باعث شود مراحل معمول Shutdown دور زده شوند و بنابراین باید با احتیاط بسیار زیاد استفاده شود؛ Shutdown غیرمنظم می‌تواند باعث از دست رفتن داده یا مشکلات Filesystem شود.

---

# The Initial RAM Filesystem

یکی از بخش‌های مهم Boot در Linux، **initramfs** است.

می‌توان initramfs را یک User Space کوچک و موقت در ابتدای Startup دانست که قبل از User Space اصلی وارد عمل می‌شود.

### چرا initramfs لازم است؟

فرض کنید Root Filesystem روی Storage خاصی قرار دارد.

Kernel برای Mount کردن Root Filesystem به Driver سخت‌افزار Storage نیاز دارد.

اما ممکن است Driver به‌صورت Loadable Kernel Module ارائه شده باشد.

Module خودش یک File است.

پس:

```text
برای Mount کردن Root
        ↓
Driver لازم است
        ↓
Driver به‌صورت Module است
        ↓
Module روی Filesystem قرار دارد
        ↓
Filesystem هنوز Mount نشده
```

این یک مشکل مرغ و تخم‌مرغ ایجاد می‌کند.

مثلاً اگر Root روی یک RAID Controller خاص باشد، Kernel ابتدا باید Driver مربوط به Controller را داشته باشد.

### راه‌حل

Bootloader یک Archive کوچک را قبل از اجرای Kernel در Memory قرار می‌دهد.

این Archive شامل مواردی مانند:

- Kernel Moduleها
- Driverهای لازم
- ابزارهای کمکی
- Scriptهای Startup موقت

است.

Kernel محتویات آن را به یک Filesystem موقت در RAM تبدیل می‌کند:

```text
initramfs
```

سپس آن را به‌عنوان Root موقت استفاده می‌کند و Init موجود در initramfs اجرا می‌شود.

روند کلی:

```text
Bootloader
    ↓
Load initramfs into RAM
    ↓
Kernel starts
    ↓
Temporary root = initramfs
    ↓
Load required drivers
    ↓
Find real root filesystem
    ↓
Mount real root
    ↓
Switch to real root
    ↓
Start real init
```

در این مرحله ابزارهای موجود در initramfs Driverهای لازم را Load می‌کنند و Root Filesystem واقعی را Mount می‌کنند.

بعد محیط موقت کنار گذاشته می‌شود و اجرای سیستم به Root واقعی منتقل می‌شود.

در بعضی سیستم‌ها این کار با مکانیزم‌هایی مانند:

```text
switch_root
```

انجام می‌شود.

### init در initramfs

Implementation بین Distributionها متفاوت است.

در بعضی سیستم‌ها Init موجود در initramfs یک Script نسبتاً ساده است که:

1. Driverها را آماده می‌کند.
2. `udevd` را اجرا می‌کند.
3. Root واقعی را پیدا و Mount می‌کند.
4. Init واقعی را اجرا می‌کند.

در سیستم‌هایی که systemd استفاده می‌کنند، ممکن است داخل initramfs بخش قابل‌توجهی از systemd نیز وجود داشته باشد، اما Configuration کامل User Space اصلی در آن وجود ندارد.

### آیا initramfs همیشه لازم است؟

خیر.

اگر Kernel تمام Driverهای لازم برای دسترسی به Root Filesystem را از قبل داشته باشد، می‌توان در بعضی سیستم‌ها بدون initramfs نیز Boot کرد.

این کار ممکن است کمی Startup را کوتاه‌تر کند.

اما روی سیستم‌های واقعی و پیچیده، initramfs معمولاً بسیار مهم است، مخصوصاً زمانی که مواردی مانند:

- LVM
- RAID
- Storage Controllerهای خاص
- Root رمزنگاری‌شده
- Mount بر اساس UUID
- سایر مراحل اولیه‌ی آماده‌سازی Storage

درگیر باشند.

برای آزمایش، می‌توان در GRUB به‌صورت موقت خط `initrd` را از Entry مربوط به Kernel حذف کرد، اما بهتر است Configuration دائمی GRUB را برای آزمایش اولیه تغییر ندهیم؛ چون یک اشتباه می‌تواند Boot سیستم را خراب کند.

### بررسی محتویات initramfs

برای بررسی محتویات Imageهای initramfs می‌توان از ابزارهایی مانند:

```bash
unmkinitramfs
```

استفاده کرد.

برای ساخت Image نیز ابزارهایی مانند:

```bash
mkinitramfs
```

و:

```bash
dracut
```

رایج هستند.

معمولاً ساخت initramfs به‌صورت دستی کار روزمره نیست و Distribution این کار را هنگام نصب یا Update Kernel انجام می‌دهد.

### initramfs در مقابل initrd

دو اصطلاح را باید از هم تفکیک کرد.

**initramfs** معمولاً به روش جدیدتری اشاره می‌کند که محتویات موقت با یک `cpio` Archive ارائه می‌شوند.

**initrd** روش قدیمی‌تری بود که بر اساس یک Disk Image برای Initial RAM Disk کار می‌کرد.

با این حال در Linux امروزی هنوز ممکن است اصطلاح `initrd` را ببینیم، حتی وقتی Image در واقع یک cpio-based initramfs است.

همچنین نام بعضی فایل‌ها و Configurationها هنوز شامل `initrd` باقی مانده است.

---

# Emergency Booting and Single-User Mode

وقتی سیستم به‌درستی Boot نمی‌شود، یکی از بهترین روش‌ها استفاده از یک **Live Image** یا **Rescue Image** است.

Live Image سیستمی است که می‌تواند بدون نصب شدن روی Disk اجرا شود.

بسیاری از Installation Imageهای Distributionها قابلیت Live یا Rescue نیز دارند.

یک Rescue Image اختصاصی هم می‌تواند روی Removable Media قرار بگیرد.

### چه مشکلاتی را می‌توان با Live/Rescue System حل کرد؟

کارهای رایج عبارت‌اند از:

- بررسی Filesystem بعد از Crash
- Reset کردن Password فراموش‌شده
- اصلاح فایل‌های حیاتی مانند `/etc/fstab`
- اصلاح Configurationهای مهم سیستم
- تعمیر مشکلات Boot
- Restore کردن سیستم از Backup

مزیت Live Environment این است که سیستم خراب‌شده لازم نیست تمام User Space خودش را اجرا کند.

در نتیجه می‌توان Storage را از بیرون بررسی و تعمیر کرد.

---

## Single-User Mode

راه دیگر، Boot شدن در **Single-user mode** است.

ایده این است که سیستم به‌جای اجرای تمام Serviceها، سریع‌تر وارد یک Root Shell شود.

در System V init معمولاً:

```text
runlevel 1
```

با Single-user mode مرتبط است.

در systemd، حالت مشابه با:

```text
rescue.target
```

نمایش داده می‌شود.

در بسیاری از سیستم‌ها می‌توان با اضافه کردن:

```text
-s
```

به Kernel/Boot Parameters درخواست ورود به حالت Single-user را مطرح کرد.

بسته به سیستم ممکن است برای ورود نیاز به Root Password باشد.

### محدودیت Single-user mode

Single-user mode امکانات بسیار محدودی دارد.

معمولاً:

- Network در دسترس نیست یا استفاده از آن دشوار است.
- GUI وجود ندارد.
- Terminal ممکن است محدود باشد.
- بسیاری از Serviceهای معمول سیستم اجرا نشده‌اند.

برای همین در بسیاری از سناریوهای جدی، **Live/Rescue Image** گزینه‌ی انعطاف‌پذیرتری است.

---

# Looking Forward

در این فصل مسیر ورود از Kernel به User Space را بررسی کردیم.

دیدیم که:

```text
Kernel
  ↓
init
  ↓
Services
  ↓
Network
  ↓
Mid/High-level Services
  ↓
Login / GUI / Applications
```

چطور شکل می‌گیرد.

همچنین دیدیم که:

- `init` نقطه شروع User Space است.
- System V init بر اساس Scriptهای ترتیبی و Runlevelها کار می‌کند.
- systemd مفهوم Unit و Dependency را وارد مرکز مدیریت Startup می‌کند.
- systemd می‌تواند Serviceها و Processهای آن‌ها را با کمک cgroups بهتر Track کند.
- Dependency و Ordering دو مفهوم متفاوت هستند.
- Unitها می‌توانند به‌صورت Parallel فعال شوند.
- Socket و Resource Unitها امکان On-Demand Activation را فراهم می‌کنند.
- systemd می‌تواند با Init Scriptهای قدیمی System V نیز Compatibility داشته باشد.
- Shutdown باید به‌صورت کنترل‌شده انجام شود.
- initramfs یک User Space موقت برای حل Dependency بین Kernel و Root Filesystem است.
- در شرایط خرابی، Single-user mode و Live/Rescue Image دو ابزار مهم برای Recovery هستند.

از اینجا به بعد، وارد بخش‌های عمیق‌تر User Space می‌شویم؛ جایی که فایل‌های Configuration سراسری، Logging، System Time، کاربران، سرویس‌های ضروری و اجزایی که systemd در طول اجرای سیستم مدیریت می‌کند، اهمیت بیشتری پیدا می‌کنند.
