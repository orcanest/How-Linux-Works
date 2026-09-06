## لینوکس چطور boot میشه ؟
### 🐧 فصل پنجم کتاب How Linux Works

در این فصل می‌ بینیم وقتی کامپیوتر را روشن می‌کنیم، قبل از اینکه محیط کاربری، سرویس‌ها، shell یا حتی process های معمول لینوکس در دسترس باشند ، چه اتفاقی می‌ افتد و چگونه کنترل سیستم از firmware به boot loader ، سپس به Linux kernel و در نهایت به user space منتقل می‌ شود.

تا اینجا با ساختار سیستم لینوکس، دستورات، device ها، دیسک‌ها و فایل‌ سیستم‌ ها آشنا شدیم. اما یک سؤال اساسی هنوز باقی مانده است: **قبل از اینکه این اجزا در دسترس باشند، خود Linux kernel چطور از روی دیسک وارد حافظه می‌شود و شروع به اجرا می‌کند؟**

---
📚 Table of Contents

- [Introduction](#introduction)
- [Startup Messages](#startup-messages)
- [Kernel Initialization and Boot Options](#kernel-initialization-and-boot-options)
- [Kernel Parameters](#kernel-parameters)
- [Boot Loaders](#boot-loaders)
- [Boot Loader Tasks](#boot-loader-tasks)
- [Boot Loader Overview](#boot-loader-overview)
- [GRUB Introduction](#grub-introduction) 
- [Exploring devices and partitions with the GRUB Command Line](#exploring-devices-and-partitions-with-the-grub-command-line)
- [GRUB Configuration](#grub-configuration)
- [GRUB Installation](#grub-installation)
- [UEFI Secure Boot Problems](#uefi-secure-boot-problems)
- [Chainloading Other Operating Systems](#chainloading-other-operating-systems)
- [Boot Loader Details](#boot-loader-details)
- [MBR Boot](#mbr-boot)
- [UEFI Boot](#ufei-boot)
- [How GRUB Works](#how-grub-works)
- [Tips](#tips)

---

### Introduction

فرآیند boot از جایی شروع می‌ شود که هنوز Linux kernel اجرا نشده است. بنابراین در اولین مراحل ، خبری از driver های معمول لینوکس ، filesystem driver ها و process ها یا حتی ```/``` واقعی سیستم نیست. در یک نمای کلی ، روند به این شکل است :

```
Firmware
    │
    ├── BIOS
    │ or
    ├──  UEFI
    │
    ▼
Boot Loader
    │
    │ kernel image
    │ kernel parameters
    │ initrd/initramfs
    ▼
Linux Kernel
    │
    ├── CPU initialization
    ├── Memory initialization
    ├── Bus discovery
    ├── Device discovery
    ├── Kernel subsystems
    ├── Root filesystem
    │
    ▼
PID 1 / init
    │
    ▼
  User Space
    │
    ├── Services
    ├── Networking
    ├── Login
    └── Applications
```

در ساده‌ ترین بیان ، ابتدا BIOS یا UEFI سخت‌ افزار را برای شروع boot آماده می‌ کند و boot code را پیدا و اجرا می‌ کند. این boot code معمولاً به یک boot loader مانند GRUB می‌ رسد. Boot loader باید Linux kernel را پیدا کند ، آن را در حافظه قرار دهد و پارامترهای مورد نیاز را به آن بدهد. سپس کنترل را به kernel منتقل می‌کند. Kernel در ادامه CPU و memory را مقدار دهی اولیه می‌ کند ، bus ها و device ها را شناسایی می‌کند ، زیرسیستم‌ های مورد نیاز را راه‌ اندازی می‌کند ، root filesystem را آماده می‌ کند و در نهایت اولین process مربوط به user space را اجرا می‌کند. این process همان PID 1 است که معمولاً init نامیده می‌شود. در سیستم‌ های امروزی، رایج‌ ترین پیاده‌ سازی init یعنی **systemd** این نقش را بر عهده دارد. 

از این نقطه به بعد وارد user space می‌شویم. فصل پنجم عمدتاً روی **boot loader** و **kernel** تمرکز دارد. فصل ششم دقیقاً از جایی ادامه می‌دهد که kernel اولین user-space process را اجرا می‌کند و سپس به init ، systemd ، initramfs و فرآیند کامل راه‌اندازی user space می‌پردازد.

نکته‌ ی مهم این است که مراحل اولیه‌ ی boot در بسیاری از توزیع‌ های امروزی به دلیل splash screen و سرعت زیاد سخت‌ افزار به‌ سادگی دیده نمی‌ شوند. بنابراین برای فهم Linux boot ، باید این مراحل را جداگانه بررسی کنیم.

---

### Startup Messages

سیستم‌ های Unix و Linux هنگام boot پیام‌ های تشخیصی زیادی تولید می‌کنند. این پیام‌ ها اطلاعاتی درباره‌ ی مراحل مختلف شروع سیستم ارائه می‌ دهند. در ابتدای boot ، بخش مهمی از این پیام‌ ها از kernel می‌آیند. بعد از اینکه kernel وارد مرحله‌ ی user space شد ، process ها و برنامه‌ های initialization نیز پیام‌ های مربوط به خودشان را تولید می‌کنند. این پیام‌ ها ممکن است شامل موارد ی مانند موارد زیر باشند:

- Kernel version
- Supported CPUs
- Memory information
- Detected devices
- Disks
- Partitions
- Drivers
- Filesystems
- Mounting root filesystem
- Starting init
- Starting systemd

مشکل اینجاست که پیام‌ های boot معمولاً ظاهر بسیار مرتب و یکسانی ندارند و بعضی از آن‌ها نیز برای کاربر عادی چندان واضح نیستند. از طرف دیگر، سخت‌افزارهای امروزی بسیار سریع‌ تر از گذشته boot می‌شوند. در نتیجه پیام‌ های kernel ممکن است آنقدر سریع روی صفحه ظاهر شوند که عملاً فرصت خواندن آن‌ها را نداشته باشید. به همین دلیل بیشتر توزیع‌های Linux از **splash screen** و روش‌ های مشابه استفاده می‌کنند تا جزئیات boot را از دید کاربر پنهان کنند.

#### 🔹 View kernel messages with journalctl

اگر سیستم از systemd استفاده کند ، یکی از بهترین راه‌ ها برای مشاهده‌ ی پیام‌های kernel استفاده از ```journalctl -k``` است. گزینه‌ی k- باعث می‌ شود پیام‌ های مربوط به kernel نمایش داده شوند. برای مشاهده‌ ی boot های قبلی نیز می‌توان از گزینه‌ی b- استفاده کرد برای مثال:

```journalctl -k -b```

یا می‌ توان یک boot مشخص را بررسی کرد :

```journalctl -k -b -1```

که معمولاً برای مشاهده‌ ی boot قبلی استفاده می‌ شود. جزئیات مربوط به journal و journalctl در فصل ۷ کتاب بررسی می‌ شود.

#### 🔹 System without systemd

اگر سیستم از systemd استفاده نکند ، روش‌ های دیگری برای مشاهده‌ ی پیام‌های kernel وجود دارد.یکی از مکان‌ های رایج ```var/log/kern.log/``` است. البته وجود و نحوه‌ ی استفاده از این فایل به سیستم logging توزیع بستگی دارد. روش دیگر استفاده از ```dmesg``` است. dmesg پیام‌ های موجود در kernel ring buffer را نمایش می‌دهد.

#### 🔹 User space messages

بعد از اینکه kernel شروع شد ، user space نیز پیام‌ های مربوط به initialization خود را تولید می‌کند. در سیستم‌های قدیمی‌ تر ، startup script ها ممکن بود پیام‌ ها را مستقیماً به console بفرستند و بعد از پایان boot دیگر به‌ سادگی قابل مشاهده نباشند. در سیستم‌ هایی که systemd و journald استفاده می‌ کنند ، بخش زیادی از پیام‌ های diagnostic مربوط به startup و runtime در journal ثبت می‌ شود و بنابراین بررسی آن‌ ها بعداً نیز امکان‌ پذیر است. این تفاوت مهم است :

```
Kernel
   │
   └── Kernel messages

User Space
   │
   └── Service / initialization messages
```

بنابراین وقتی یک سیستم boot نمی‌ شود، باید بدانیم دقیقاً در کدام مرحله متوقف شده است. پیام‌ های boot یکی از مهم‌ ترین منابع برای پیدا کردن این نقطه هستند. 

---

### Kernel Initialization and Boot Options

وقتی Linux kernel شروع به اجرا می‌کند، initialization را در یک ترتیب کلی انجام می‌ دهد. ترتیب دقیق داخلی kernel بسیار پیچیده‌ تر از این فهرست ساده است، اما برای درک فرآیند boot می‌توان آن را به شکل زیر در نظر گرفت :

1. بررسی و شناسایی CPU
2. بررسی و مقداردهی اولیه‌ ی memory
3. شناسایی device bus ها
4. شناسایی device ها
5. راه‌ اندازی زیر سیستم‌ های کمکی kernel
6. آماده‌ سازی و mount کردن root filesystem
7. شروع user space

این مراحل کاملاً مستقل از هم نیستند و بین آن‌ ها dependency وجود دارد. برای مثال ، kernel برای دسترسی به یک disk ممکن است به چند لایه‌ ی مختلف نیاز داشته باشد:

```
Storage Device
      │
      ▼
Bus Support
      │
      ▼
SCSI / Storage Subsystem
      │
      ▼
Device Driver
      │
      ▼
Block Device
      │
      ▼
Filesystem
```

این همان موضوعی است که در فصل سوم هنگام بررسی device ها و SCSI با آن آشنا شدیم.

⚠️ **مشکل مهم: driver قبل از root filesystem** : اینجا یک مشکل جالب به وجود می‌آید. فرض کنید root filesystem روی دیسکی قرار دارد که برای دسترسی به آن به یک driver خاص نیاز است. اما kernel هنوز root filesystem را mount نکرده است و ممکن است driver مورد نیاز هم به صورت loadable kernel module باشد و هنوز در kernel اصلی قرار نگرفته باشد ، در این حالت :

```
Kernel
│
├── Driver required
│
├── Driver not loaded yet
│
└── Driver is on real filesystem
```

یعنی kernel برای mount کردن root filesystem به driver نیاز دارد، اما driver را باید از چیزی بخواند که خودش هنوز نتوانسته mount کند. این همان مشکل معروف boot است که با initial RAM filesystem یا initrd/initramfs حل می‌شود. جزئیات initramfs در فصل ششم بررسی خواهد شد.

#### 🔹 Kernel approaching user space

در انتهای initialization، kernel بخشی از memory را که دیگر لازم ندارد آزاد می‌ کند و ساختارهای داخلی خود را برای ادامه‌ ی کار آماده می‌کند. پیام‌ هایی شبیه این ممکن است در log دیده شوند :

```Freeing unused kernel memory: ... ```

```Write protecting the kernel read-only data: ...```

این پیام‌ ها نشان می‌ دهند kernel در حال انجام آخرین کارهای داخلی قبل از ورود جدی به user space است. در kernel های جدید ممکن است پیامی شبیه این نیز دیده شود:

```Run /init as init process```

این نقطه اهمیت زیادی دارد ، چون از اینجا به بعد اولین user-space process وارد بازی می‌شود. اگر سیستم از initramfs استفاده کند، ممکن است init/ در محیط initramfs ابتدا اجرا شود و بعد از آماده شدن root واقعی ، فرآیند boot به سیستم اصلی منتقل شود. این موضوع در فصل ششم با جزئیات بیشتری بررسی می‌شود.

در یک سیستم معمولی ، بعد از آماده شدن root filesystem می‌توان پیام‌ هایی درباره‌ ی filesystem و سپس PID 1 مشاهده کرد. برای نمونه : 

```
EXT4-fs (...): mounted filesystem ...
systemd[1]: ...
systemd[1]: Detected architecture ...
```
وقتی پیام‌ هایی مانند systemd[1] را می‌ بینیم ، به‌ وضوح وارد user space شده‌ایم. عدد 1 همان PID مربوط به اولین process سیستم است.

---

### Kernel Parameters

کرنل لینوکس هنگام شروع فقط یک image خام نیست که بدون هیچ اطلاعاتی اجرا شود. Boot loader مجموعه‌ ای از **kernel parameters** را نیز به kernel تحویل می‌ دهد. این پارامترها متن‌ هایی هستند که رفتار kernel یا اجزای مختلف boot را مشخص می‌کنند. آن‌ ها می‌ توانند برای موارد مختلفی استفاده شوند ، از جمله :

- مشخص کردن root filesystem
- کنترل مقدار پیام‌ های diagnostic
- تنظیم رفتار برخی driver ها
- تنظیم گزینه‌ های مربوط به سخت‌ افزار
- تغییر نحوه‌ ی boot
- انتقال بعضی گزینه‌ ها به init

#### 🔹 View kernel parameters 

پارامترهایی که به kernel در حال اجرای فعلی داده شده‌ اند را می‌ توان از این فایل ```cat /proc/cmdline``` مشاهده کرد. نمونه‌ ای از خروجی ممکن است شبیه این باشد :

<img width="100%" height="40" alt="image" src="https://github.com/user-attachments/assets/fe426d96-b340-4621-b954-8234b2a649d6" />

فایل ```/proc/cmdline``` نشان می‌ دهد kernel فعلی با چه command line ای اجرا شده است.

#### 🔹 Different forms of parameter

پارامترهای Kernel معمولاً به شکل های مختلف دیده می‌ شوند : 

به صورت Single-word flags ، مثلاً ```ro``` و ```quiet``` و ```splash```. پارامترهای key=value مثلاً ```...=root=UUID``` و ```vt.handoff=1```. هر پارامتر معنای خاص خودش را دارد و بعضی از آن‌ها به یک kernel subsystem یا driver مشخص مربوط می‌شوند.

#### 🔹root parameter

یکی از مهم‌ترین پارامترهای boot ، پارامتر ```root``` است. این پارامتر مشخص می‌ کند root filesystem کجاست. بدون اینکه kernel بتواند root filesystem را پیدا و آماده کند ، نمی‌ تواند به شکل عادی وارد user-space startup شود. root می‌تواند با یک device path مشخص شود ```root=/dev/sda1``` اما این روش به نام‌گذاری device ها وابسته است.

روش دیگر استفاده از logical volume است : ```root=/dev/mapper/my-system-root``` و روش رایج دیگر استفاده از UUID است : ```root=UUID=17f12d53-c3d7-4ab3-943e-a0a72366c9fa```. استفاده از UUID یا شناسه‌ های پایدار مشابه مزیت مهمی دارد ، چون به ترتیب نام‌ گذاری device ها وابسته نیست.

برای مثال ممکن است یک دیسک در یک boot به شکل ```dev/sda/``` و در شرایط دیگری به دلیل ترتیب متفاوت discovery به شکل دیگری شناخته شود. بنابراین ```dev/sda1/``` یک نام وابسته به device mapping است، در حالی که ```UUID``` شناسه‌ ای مربوط به خود filesystem است.

#### 🔹ro parameter

پارامتر```ro``` یعنی root filesystem در ابتدای boot به صورت read-only آماده شود. این رفتار برای مراحل اولیه‌ی boot مفید است، چون filesystem می‌تواند قبل از شروع عملیات عادی با ابزارهایی مانند fsck بررسی شود. بعد از پایان بررسی و آماده شدن سیستم ، root filesystem معمولاً به حالت read-write remount می‌ شود ، به صورت مفهومی :

```
Root filesystem
       │
       ▼
Read-only
       │
       ▼
Filesystem check / initialization
       │
       ▼
Read-write
```
#### 🔹Unknown parameters

اگر kernel با پارامتری مواجه شود که خودش آن را نمی‌ شناسد ، بعضی از این پارامترها را برای مرحله‌ ی بعد نگه می‌ دارد و در هنگام شروع user space به init منتقل می‌ کند. این موضوع امکان استفاده از بعضی پارامترها برای خود init را فراهم می‌کند. برای مثال در یونیکس option مانند ```s-``` می‌ تواند برای درخواست single-user mode به init منتقل شود. بنابراین همه‌ی چیزهایی که در kernel command line می‌بینیم الزاماً توسط خود kernel مصرف نمی‌شوند. بخشی از command line ممکن است برای برنامه‌ ی بعدی در زنجیره‌ ی boot باشد.

#### 🔹Kernel parameters documentation

برای پارامترهای عمومی می‌ توان از```bootparam(7)``` استفاده کرد. برای جزئیات دقیق‌ تر نیز kernel documentation شامل اطلاعات مربوط به kernel parameters است.

---

### Boot Loaders

قبل از اینکه Linux kernel و init شروع شوند، یک برنامه باید kernel را پیدا و اجرا کند. این برنامه **boot loader** است. در نگاه اول وظیفه‌ ی boot loader ساده به نظر میرسد اما مسئله بسیار پیچیده‌ تر است :

```
Find kernel
     │
     ▼
Load kernel into memory
     │
     ▼
Pass kernel parameters
     │
     ▼
Start kernel
```
#### 🔹 big problem

برای اینکه kernel بتواند فایل kernel را از filesystem بخواند ، به driver و filesystem support نیاز دارد. اما خود kernel هنوز اجرا نشده است. پس boot loader باید بدون استفاده از Linux kernel بتواند به storage دسترسی پیدا کند ، به بیان دیگر :

```
Kernel not yet running
│
├── So Linux drivers are not available
│
└── The boot loader must access the disk itself
```

این یکی از تفاوت‌های مهم boot loader با برنامه‌های عادی Linux است.

#### 🔹 BIOS and UEFI

روی PC ها ، boot loader در مراحل ابتدایی با کمک firmware سیستم به storage دسترسی پیدا می‌کند. دو فناوری اصلی BIOS و UEFI  که در این فصل بررسی می‌شوند. در روش‌های قدیمی BIOS و firmware روش‌ های سطح پایین‌ تری برای دسترسی به storage فراهم می‌کنند. یکی از مفاهیم مهم در اینجا **Logical Block Addressing یا LBA** است. LBA به جای اینکه boot code مجبور باشد جزئیات فیزیکی قدیمی دیسک را مدیریت کند ، بلوک‌ های منطقی دیسک را با شماره‌ ی آن‌ ها مورد دسترسی قرار می‌ دهد. این روش ساده است اما محدودیت‌ های خودش را دارد. برای boot مشکلی نیست ، چون boot loader فقط در مدت کوتاهی به این روش نیاز دارد. بعد از اینکه kernel اجرا شد ، kernel می‌تواند driver های کامل و سریع خودش را برای storage به کار بگیرد.

#### 🔹 BIOS or UEFI detection

یکی از روش‌ های بررسی اینکه سیستم با UEFI boot شده یا BIOS ، استفاده از ```efibootmgr```است. اگر اطلاعات مربوط به boot target های UEFI در دسترس باشد ، سیستم در محیط UEFI اجرا شده است. راه دیگر بررسی این مسیر```sys/firmware/efi/``` است. وجود این مسیر معمولاً نشان می‌ دهد kernel در محیط UEFI boot شده است. نکته‌ ی مهم این است که وجود یا نبود این مسیر را باید به عنوان نشانه‌ ی **محیط boot فعلی** در نظر گرفت ، نه صرفاً اینکه firmware سخت‌ افزار از قابلیت UEFI برخوردار است یا نه.

#### 🔹 Boot loader access to filesystem

فقط Boot loader های مدرن نمی‌ دانند disk کجاست. بسیاری از آن‌ ها می‌توانند partition table را بخوانند و filesystem های مختلف را نیز تا حدی parse کنند. این موضوع بسیار مهم است، چون boot loader می‌تواند مستقیماً فایل‌هایی مانند ```boot/vmlinuz/``` و ```boot/initrd.img/``` را پیدا کند. در نتیجه boot loader می‌تواند نسبت به گذشته انعطاف‌ پذیرتر باشد. به‌ طور کلی، یک الگوی تاریخی وجود دارد :

```
Linux kernel
│
└── Support for new storage technology
│
▼
Boot loader
│
└── Simpler implementation of the same feature
```

یعنی boot loader مجبور است بخشی از قابلیت‌ های لازم برای دسترسی به storage را مستقل از kernel فراهم کند.

---

### Boot Loader Tasks

یک boot loader لینوکسی معمولاً باید چند قابلیت اصلی داشته باشد : 

#### 🔹 Choosing between multiple kernels

ممکن است چند kernel روی یک سیستم نصب شده باشد و Boot loader باید بتواند یکی از آن‌ها را انتخاب کند. برای مثال :

- Linux 6.x
- Linux 6.x - previous
- Linux recovery

این قابلیت هنگام نصب kernel جدید اهمیت زیادی دارد، چون اگر kernel جدید مشکل داشته باشد، امکان انتخاب kernel قبلی وجود دارد.

#### 🔹 Choosing a different set of kernel parameters

یک kernel می‌ تواند با command line های متفاوت boot شود و هرکدام ممکن است پارامترهای متفاوتی داشته باشند ، مثلاً :

- Normal boot
- Recovery boot
- Debug boot
- Single-user boot

#### 🔹 Manual editing

یک Boot loader باید بتواند اجازه دهد کاربر در صورت نیاز configuration مربوط به boot را موقتاً تغییر دهد. این قابلیت برای troubleshooting بسیار مهم است ، مثلاً ممکن است یک parameter اشتباه باعث شود root filesystem پیدا نشود. کاربر می‌تواند در GRUB وارد configuration editor شود، پارامتر را موقتاً تغییر دهد و boot را امتحان کند. این تغییر معمولاً موقتی است و لزوماً configuration دائمی سیستم را تغییر نمی‌ دهد.

#### 🔹 Booting other operating systems

یک Boot loader می‌تواند علاوه بر Linux ، سیستم‌ عامل دیگری را نیز boot کند. برای مثال Linux و  Windows. همچنین GRUB می‌تواند در بعضی سناریوها به جای اجرای مستقیم kernel لینوکس ، boot loader سیستم‌ عامل دیگر را اجرا کند ، این فرآیند را **chainloading** می‌ نامیم. boot loader ها در طول زمان امکانات بیشتری پیدا کرده‌اند. از menu و command-line گرفته تا history و امکانات مختلف برای انتخاب kernel و parameter. این قابلیت‌ ها برای کسانی که kernel سفارشی می‌سازند یا در حال debug کردن boot هستند بسیار مفیدند.

---

### Boot Loader Overview

انواع Boot loader های مختلفی در دنیای Linux وجود داشته‌ اند یا هنوز در بعضی سیستم‌ها دیده می‌شوند.

#### 🔹 GRUB — Grand Unified Boot Loader

یکی از رایج‌ ترین boot loader های Linux است و نسخه‌ های مربوط به BIOS/MBR و UEFI دارد. تمرکز اصلی این فصل روی GRUB است.

#### 🔹 LILO

یکی از boot loader های قدیمی Linux بود و از نخستین boot loader های مهم این سیستم‌ عامل محسوب می‌ شد. ELILO نیز نسخه‌ ای برای EFI بود.

#### 🔹 SYSLINUX

می‌تواند از filesystem های مختلف استفاده کند و خانواده‌ ای از boot loader های سبک محسوب می‌ شود.
#### 🔹 LOADLIN

برای boot کردن kernel Linux از محیط MS-DOS طراحی شده بود.
#### 🔹 coreboot

قبلاً با نام LinuxBIOS نیز شناخته می‌ شد ، بیشتر یک firmware replacement است تا صرفاً یک boot loader معمولی و می‌ تواند در بعضی پیکربندی‌ ها kernel را نیز در فرآیند boot وارد کند.

#### 🔹 Linux Kernel EFISTUB

کرنل لینوکس می‌تواند قابلیت EFISTUB داشته باشد تا firmware UEFI بتواند kernel را مستقیماً از EFI System Partition اجرا کند.
#### 🔹 efilinux

یک UEFI boot loader است که بیشتر به عنوان نمونه و مرجع برای boot loader های UEFI مطرح شده است.

کتاب تقریباً تمام تمرکز خود را روی **GRUB** قرار می‌دهد ، چون GRUB هم امکانات زیادی دارد و هم نمونه‌ ی مناسبی برای فهم ساختار boot loader هاست. در عمل ممکن است یک توزیع Linux رفتار و ظاهر boot loader را به شدت تغییر داده باشد ، بنابراین همیشه از ظاهر صفحه‌ ی boot نمی‌توان با اطمینان فهمید دقیقاً چه boot loader استفاده می‌شود. یکی از بهترین روش‌های یادگیری boot loader این است که بتوانیم به boot prompt یا menu آن برسیم و kernel name و kernel parameters را مشاهده و تغییر دهیم.

---

### GRUB Introduction

کلمه GRUB مخفف ```Grand Unified Boot Loader``` است. در این فصل تمرکز روی **GRUB 2** است. نسخه‌ ی قدیمی‌ تر آن با نام GRUB Legacy شناخته میشد و دیگر نسخه‌ ی فعال اصلی محسوب نمی‌شود. یکی از مهم‌ترین قابلیت‌ های GRUB، توانایی پیمایش filesystem است. این قابلیت باعث می‌شود GRUB بتواند فایل‌ های kernel و configuration را پیدا کند.

#### 🔹 Entering the GRUB menu

در سیستم‌ های BIOS معمولاً می‌ توان هنگام startup کلید SHIFT را نگه داشت. در سیستم‌های UEFI معمولاً ESC برای نمایش menu استفاده می‌شود. البته رفتار دقیق این کلید ها می‌تواند به توزیع و تنظیمات firmware بستگی داشته باشد. در بسیاری از سیستم‌ ها boot loader آنقدر سریع عمل می‌ کند که menu را اصلاً نمی‌بینید. در menu GRUB می‌ توان با ```e configuration``` مربوط به entry انتخاب‌ شده را ویرایش کرد.

<img width="100%" height="374" alt="image" src="https://github.com/user-attachments/assets/5fdad68e-70f9-45c8-9780-17f53f725a2a" />

<img width="100%" height="374" alt="image" src="https://github.com/user-attachments/assets/0c72bb4d-942e-420e-9bb8-aaab98c1ebf0" />

#### 🔹 A GRUB tip

یکی از گیج‌ کننده‌ ترین موضوعات برای افراد تازه‌ کار این است که GRUB خودش از Linux kernel استفاده نمی‌کند. این کاملاً منطقی است :

```
GRUB
│
│ Must run the Linux kernel
│
▼
Linux kernel
```
پس GRUB نمی‌ تواند برای انجام وظیفه‌ ی خودش به Linux kernel وابسته باشد. GRUB محیط و command های خودش را دارد ، مثلاً ```ls``` و ```insmod``` و ```set``` و ```echo``` در GRUB وجود دارند.  insmod در GRUB نیز command مربوط به خودش است و برای load کردن **GRUB modules** استفاده می‌شود. نباید آن را با insmod یا مفاهیم module های Linux kernel یکی دانست. GRUB حتی مفهوم خودش از kernel و root را دارد که ممکن است با معنای آن‌ ها در Linux متفاوت باشد.

#### 🔹 Two different meanings of root

این مهم‌ ترین نکت ه‌ی مفهومی این بخش است ، وقتی در configuration GRUB چیزی مانند ```root=UUID``` می‌بینید ، این root یک **kernel parameter** است و به root filesystem لینوکس مربوط می‌ شود. اما وقتی در GRUB با چیزی مانند ```set root=(hd0,msdos1)``` مواجه می‌ شوید ، این root مربوط به خود **GRUB** است. GRUB root یعنی filesystem ای که GRUB در آن دنبال فایل‌ هایی مثل kernel و initrd می‌ گردد ، بنابراین: ```GRUB root ≠ Linux root filesystem```. ممکن است این دو در یک partition قرار داشته باشند، اما از نظر مفهومی دو چیز متفاوت‌ اند.

در configuration نمونه‌ ی کتاب، GRUB ابتدا root خودش را مشخص می‌ کند ، سپس با search می‌ تواند filesystem دارای UUID مشخص را پیدا کند ، بعد kernel را از آن filesystem می‌خواند و در نهایت initrd را نیز مشخص می‌ کند.

---

### Exploring devices and partitions with the GRUB Command Line

درGRUB یک command-line interface دارد که می‌تواند برای بررسی device ها، partition ها و   filesystem ها استفاده شود. برای ورود به آن از داخل GRUB menu می‌توان c را فشار داد. در این حالت prompt زیر را می‌بینید :

```grub>```

#### 🔹 ls command in GRUB

در GRUB ، ls می‌تواند دو کار متفاوت انجام دهد. اگر بدون argument اجرا شود ```grub> ls``` میتواند GRUB device هایی را که می‌شناسد نمایش می‌دهد مثلاً ```(hd0) (hd0,msdos1)```. در این مثال ```hd0``` اولین disk است که GRUB آن را می‌شناسد و ```hd0,msdos1``` یک partition روی آن disk است.

#### 🔹 msdos and gpt

پیشوند ```msdos``` نشان‌ دهنده‌ی partition table از نوع MBR است. در مقابل، چیزی شبیه ```(hd0,gpt1)``` به GPT اشاره می‌ کند. پس naming scheme خود GRUB نیز اطلاعاتی درباره‌ ی نوع partition table به ما می‌ دهد.

#### 🔹 Device names are not stable

نام GRUB device ها را بر اساس ترتیب discovery می‌ بیند ، ```hd0``` و ```hd1``` و ```hd2```. بنابراین شماره‌ ی یک disk الزاماً یک شناسه‌ ی دائمی برای خود disk نیست. همانند چیزی که در فصل‌ های قبلی درباره‌ ی dev/sda دیدیم ، ترتیب discovery می‌تواند تغییر کند. به همین دلیل GRUB نیز قابلیت جست و جوی filesystem بر اساس UUID را دارد.

#### 🔹 ls -l in GRUB

برای دریافت اطلاعات بیشتر می‌ توان از ```grub> ls -l``` استفاده کرد. این command می‌ تواند اطلاعاتی را نشان دهد مانند:

- Filesystem type
- UUID
- Size
- Sector size
- Partition information

در نمونه‌ ی کتاب ، GRUB یک partition با filesystem خانواده‌ ی ext را نشان می‌ دهد و UUID آن را نیز نمایش می‌ دهد.

#### 🔹 File Navigation

 اگر ls روی یک filesystem مشخص اجرا شود ، دیگر به جای فهرست device ها ، محتوای filesystem را نمایش می‌ دهد مثلاً ```grub> ls (hd0,msdos1)/``` یا اگر متغیر $root تنظیم شده باشد ```grub> ls ($root)/```. ممکن است خروجی شامل مواردی مانند :

- /etc
- /bin
- /dev
- /boot

در این حالت دیگر ls درباره‌ ی device ها صحبت نمی‌کند ، بلکه واقعاً در حال پیمایش filesystem است. برای مشاهده‌ی /boot هم ```grub> ls ($root)/boot``` . این قابلیت یکی از ویژگی‌ های بسیار مهم GRUB است، چون نشان می‌ دهد GRUB بدون Linux kernel می‌ تواند filesystem را بخواند.

#### 🔹 GRUB variables

برای مشاهده‌ ی متغیرهای فعلی GRUB باید ```grub> set``` را اجرا کنید. ممکن است چیزهایی شبیه این ببینید:

```prefix=(hd0,msdos1)/boot/grub```

```root=hd0,msdos1```

یکی از مهم‌ ترین متغیرها `	``prefix$``` است. prefix مشخص می‌ کند GRUB configuration و فایل‌ های کمکی خودش را کجا انتظار دارد مثلاً: 

```prefix=(hd0,msdos1)/boot/grub```

یعنی GRUB انتظار دارد فایل‌ های مربوط به خودش را در مسیر boot/grub/ روی آن filesystem پیدا کند.

#### 🔹 Exit from the command line

می‌توان با ESC به menu برگشت. اگر configuration لازم برای boot آماده شده باشد، می‌توان دستور ```boot``` را اجرا کرد تا GRUB فرآیند boot را ادامه دهد. این نکته مهم است که تغییراتی که در GRUB command line یا editor انجام می‌ دهید معمولاً برای همان boot هستند و برای اصلاح دائمی باید configuration اصلی سیستم را تغییر دهید.

---

### GRUB Configuration

دایرکتوری GRUB بسته به توزیع و layout سیستم ممکن است معمولاً یکی از این مسیرها باشد ، ```boot/grub/``` یا ```boot/grub2/```. در این directory معمولاً فایل‌ ها و module های مورد نیاز GRUB قرار دارند. مهم‌ ترین فایل configuration هم ```grub.cfg``` است همچنین ممکن است directory هایی مانند ```i386-pc``` و فایل‌ های module با پسوند ```mod.``` وجود داشته باشند.

> 💡 مستقیماً grub.cfg را ویرایش نکنید! ، نکته‌ ی بسیار مهم ```grub.cfg``` معمولاً به صورت خودکار تولید می‌شود. بنابراین نباید configuration دائمی را با ویرایش مستقیم آن مدیریت کرد. در عوض از ابزارهایی مانند ```grub-mkconfig``` استفاده می‌ شود. در بعضی توزیع‌ ها نام ابزار می‌ تواند ```grub2-mkconfig``` باشد.

#### 🔹 Structure of grub.cfg

فایل grub.cfg مجموعه‌ ای از command های مخصوص GRUB است. در ابتدای آن معمولاً initialization ،  تعریف variable ها و تنظیمات و موارد مشابه دیده می‌شود. مثلاً ممکن است command هایی وجود داشته باشند مانند : 

- loadfont
- load_video
- insmod

بعدتر entry های مختلف boot قرار می‌گیرند. هر entry معمولاً با ```menuentry``` شروع می‌ شود. نمونه‌ ی ساده‌ شده :

```
menuentry 'Ubuntu' {
    insmod ...
    set root=...
    search ...
    linux /boot/vmlinuz-... root=UUID=... ro quiet splash
    initrd /boot/initrd.img-...
}
```
در این configuration چند مفهوم مهم را می‌ توان دید : 

- اول **set root** : این root مربوط به GRUB است. یعنی GRUB باید filesystem مورد نظر را برای پیدا کردن فایل‌ ها انتخاب کند.
- دوم **search** : در GRUB می‌تواند با استفاده از UUID filesystem موردنظر را پیدا کند. این کار باعث می‌ شود configuration کمتر به ترتیب نام‌ گذاری device ها وابسته باشد.
- سوم **linux** : در GRUB command قسمت linux ، مسیر image مربوط به Linux kernel مشخص می‌ شود. مثلاً ```linux /boot/vmlinuz/``` که GRUB این فایل را از filesystem که به عنوان GRUB root پیدا کرده است می‌ خواند. بعد از نام kernel، kernel parameters قرار می‌ گیرند ```root=UUID``` و ```ro``` و ```quiet``` و ```splash```.
- چهارم **initrd** : دستور ```initrd``` مسیر image مربوط به initial RAM filesystem را مشخص می‌ کند. مثلاً ```initrd /boot/initrd.img/``` این image در مرحله‌ ی boot قبل یا همزمان با kernel در حافظه قرار می‌ گیرد و برای آماده‌ سازی محیط اولیه‌ ی boot استفاده می‌ شود. جزئیات initramfs در فصل ششم بررسی می‌شود.

#### 🔹 Multiple kernels in GRUB

بسیاری از توزیع‌ ها چند kernel را در configuration نگه می‌ دارند. ممکن است entry های قدیمی‌ تر در یک ```submenu``` قرار بگیرند تا menu اصلی بیش از حد شلوغ نشود. این همان چیزی است که هنگام انتخاب kernel های قدیمی‌ تر در بعضی سیستم‌ ها مشاهده می‌ کنید.

#### 🔹 Generate new configuration

برای مشاهده‌ ی configuration تولید شده با ```grub-mkconfig``` می‌ تواند اجرا شود. این command به صورت معمول configuration تولید شده را روی standard output قرار می‌ دهد. برای نوشتن آن در فایل ```grub-mkconfig -o /boot/grub/grub.cfg``` استفاده می‌ شود. در Fedora و بعضی سیستم‌ ها ممکن است مسیر و command با grub2 متفاوت باشد.

#### 🔹 /etc/grub.d

بخش مهم دیگر configuration، دایرکتوری ```/etc/grub.d/``` است. تقریباً هر فایل اصلی موجود در این directory یک shell script است که بخشی از grub.cfg را تولید می‌کند. نکته‌ی بسیار مهم اینکه **خود GRUB این shell scriptها را هنگام boot اجرا نمی‌کند**. روند درست این است :

```
/etc/grub.d/*
    │
    ▼
grub-mkconfig
    │
    ▼
grub.cfg
    │
    ▼
GRUB at boot
    │
    ▼
Run grub.cfg
```

یعنی script ها در user space اجرا می‌ شوند تا configuration نهایی تولید شود. GRUB هنگام boot فقط configuration تولید شده را اجرا می‌ کند. در فایل grub.cfg می‌توان comment هایی مانند``` ### BEGIN /etc/grub.d/00_header ### ``` این دید که نشان می‌ دهند هر بخش configuration از کجا تولید شده است.

#### 🔹 File arrangement

نام فایل‌ های etc/grub.d معمولاً با عدد شروع می‌شود. مثلاً ```00_header``` یا ```10_linux``` یا ```30_os-prober``` یا ```40_custom```. این شماره‌ ها روی ترتیب پردازش script ها تأثیر دارند. عدد پایین‌ تر معمولاً زود تر پردازش می‌ شود.

#### 🔹 Customizing GRUB

برای custom configuration می‌توان از ```custom.cfg``` استفاده کرد. مسیر رایج در ```/boot/grub/custom.cfg``` است. دو روش مرتبط با /etc/grub.d نیز وجود دارد : ```40_custom``` و ```41_custom```.

ویرایش مستقیم 40_custom ممکن است با update های package تحت تأثیر قرار گیرد ، بنابراین استفاده از custom.cfg روش تمیز تری برای بسیاری از custom entry هاست. 41_custom معمولاً configuration مربوط به custom.cfg را هنگام boot وارد می‌ کند.

یک نکته‌ ی مهم این است که تغییرات مربوط به 41_custom و custom.cfg الزاماً در خروجی جدید grub-mkconfig به صورت entry های تولید شده ظاهر نمی‌ شوند ، چون GRUB می‌تواند custom.cfg را در زمان boot بخواند. توزیع‌ ها همچنین ممکن است script های اضافی خود شان را داشته باشند. برای مثال ممکن است گزینه‌ هایی مانند memtest86+ از طریق script های مخصوص توزیع وارد GRUB شوند. 

> 💡 قبل از تغییر configuration مهم است از فایل فعلی backup داشته باشید و مطمئن شوید مسیر درست GRUB را تغییر می‌ دهید.

---

### GRUB Installation

نصب GRUB با configuration آن متفاوت است. خوشبختانه در نصب معمول یک توزیع Linux، installer خود توزیع این کار را انجام می‌ دهد. اما در شرایطی ممکن است لازم باشد GRUB را خودتان نصب کنید مانند :

- Restore a bootable disk
- Create a custom boot sequence
- Transfer the system
- Repair the boot loader
- Prepare the disk for another system

قبل از نصب باید بدانید سیستم با چه روش boot می‌شود مثلاً ```BIOS/MBR``` یا ```UEFI```. چون target مربوط به GRUB در این دو حالت متفاوت است. همچنین باید محل GRUB directory را مشخص کنید. مسیر معمول در ```boot/grub/``` است. در سیستم‌ های UEFI نیز باید mount point مربوط به EFI System Partition را بدانید که معمولاً در ```boot/efi/``` است.

#### 🔹 GRUB is modular

یک GRUB در یک سیستم modular است. برای اینکه بتواند module های بعدی را load کند، ابتدا باید بتواند filesystem ای را بخواند که GRUB directory در آن قرار دارد. بنابراین بخشی از GRUB باید از همان ابتدا توانایی دسترسی به filesystem مورد نظر را داشته باشد. برای مثال در بعضی سیستم‌ های Linux ممکن است module مربوط به ext filesystem و در صورت نیاز module مربوط به LVM لازم باشد.

#### 🔹 grub-install

ابزار اصلی نصب GRUB هم ```grub-install```  است. این command را نباید با ابزارهای قدیمی یا نام‌ های مشابه اشتباه گرفت. در یک نمونه‌ ی BIOS/MBR ، اگر disk مورد نظر در ```dev/sda/``` باشد، ممکن است command به شکل ```grub-install /dev/sda``` باشه . این مثال به معنای نصب GRUB روی target مربوط به آن disk در حالت BIOS/MBR هستش اما در سیستم UEFI روش نصب متفاوت است.

#### 🔹 Installing GRUB with UEFI

در UEFI معمولاً boot loader به عنوان فایل efi. روی EFI System Partition قرار می‌ گیرد. grub-install می‌تواند این فرآیند را مدیریت کند و نمونه‌ ی کلی :

```grub-install --efi-directory=efi_dir --bootloader-id=name```

که در آن efi_dir محل مناسب ESP در سیستم فعلی است. در سیستم‌ های رایج ، ESP ممکن است در ```boot/efi/``` هم mount شده باشد. علاوه بر قرار دادن فایل‌ها ، firmware باید بداند boot loader جدید وجود دارد. این اطلاعات معمولاً در **NVRAM** مربوط به UEFI ذخیره می‌ شود و ابزار ```efibootmgr``` برای مدیریت boot entry ها استفاده می‌ شود. در بسیاری از نصب‌ ها ```grub-install``` اگر شرایط لازم فراهم باشد ، این مرحله را نیز انجام می‌ دهد.

#### 🔹 GRUB installation warning

نصب اشتباه GRUB می‌تواند boot سیستم را خراب کند. بنابراین قبل از اجرای grub-install باید دقیقاً بدانید:

- که disk مورد نظر کدام است ؟
- سیستم BIOS است یا UEFI ؟
- محل GRUB directory کجاست ؟
- محل ESP کجاست ؟
- آیا disk دیگری نیز به سیستم متصل است ؟
- آیا سیستم dual-boot است ؟ 

اشتباه در target می‌تواند باعث شود سیستم دیگر boot نشود.

---

### UEFI Secure Boot Problems

یکی از تفاوت‌ های مهم سیستم‌ های مدرن  UEFI وجود **Secure Boot** است. Secure Boot یک مکانیزم امنیتی در firmware UEFI است که می‌ تواند اجرای boot loader را به software هایی محدود کند که با یک کلید مورد اعتماد sign شده‌ اند. به صورت مفهومی :

```
UEFI
│
├── Is the boot loader valid and signed?
│
├── Yes → Execute
│
└── No → reject
```

این قابلیت با هدف جلوگیری از اجرای boot software غیرمجاز یا دست‌ کاری‌ شده طراحی شده است. Microsoft نیز در سیستم‌ های دارای Windows 8 و نسخه‌ های بعد Secure Boot را به عنوان بخشی از الزامات مربوط به سخت‌ افزارهای Windows مطرح کرد. در نتیجه اگر روی چنین سیستمی یک boot loader unsigned قرار دهید ، firmware ممکن است آن را اجرا نکند.

#### 🔹 Linux and Secure Boot

توزیع‌ های بزرگ Linux معمولاً برای این مشکل راه‌ حل آماده دارند. آن‌ها boot loader های signed ارائه می‌ کنند. یکی از معماری‌ های رایج این است :

```
UEFI
  │
  ▼
Signed shim
  │
  ▼
GRUB
  │
  ▼
Linux kernel
```

برنامه‌ shim یک برنامه‌ ی کوچک signed است که بین UEFI و GRUB قرار می‌ گیرد. UEFI ابتدا shim را که مورد اعتماد است اجرا می‌ کند و shim سپس GRUB را راه‌ اندازی می‌ کند. در بعضی پیکربندی‌ های امنیتی ، زنجیره‌ ی اعتماد می‌ تواند حتی فراتر برود و kernel نیز باید دارای امضای معتبر باشد.

#### 🔹 Why is Secure Boot important for developers ?

اگر در حال آزمایش boot loader شخصی ، kernel سفارشی یا boot sequence سفارشی باشید ، Secure Boot می‌ تواند مانع اجرای code شما شود. یکی از راه‌ ها برای آزمایش ، غیرفعال کردن Secure Boot در UEFI settings است. اما باید توجه داشت که این کار می‌تواند برای dual-boot ها مناسب نباشد. همچنین بهتر است Secure Boot را صرفاً به عنوان یک مزاحمت در نظر نگیریم. در محیطی که امنیت boot اهمیت دارد ، هدف آن جلوگیری از اجرای boot software غیرمجاز است.

---

### Chainloading Other Operating Systems

کاربرد GRUB فقط برای boot کردن Linux نیست. اگر سیستم چند operating system داشته باشد، GRUB می‌تواند به جای اجرای مستقیم Linux kernel، boot loader سیستم‌عامل دیگری را اجرا کند. به این روش **Chainloading** گفته می‌ شود. ایده‌ ی chainloading این است :

```
GRUB
│
└── Run the boot loader of another operating system
│
▼
Operating System
```

#### 🔹 Chainloading in UEFI

قابلیت UEFI برای این کار انعطاف‌ پذیری زیادی دارد، چون چند boot loader می‌ توانند روی EFI System Partition قرار بگیرند. برای مثال می‌توان directory هایی مانند این داشت :

```
EFI/
├── Microsoft/
├── ubuntu/
├── grub/
└── ...
```
هر boot loader فایل‌ های efi. خودش را دارد.

#### 🔹 Chainloading in MBR

در سیستم‌ های قدیمی MBR، قابلیت GRUB می‌ تواند boot sector یا boot loader مربوط به partition دیگری را اجرا کند. یک نمونه‌ ی ساده برای Windows ممکن است مفهومی شبیه این داشته باشد :

```
menuentry "Windows" {
    insmod chain
    insmod ntfs
    set root=(hd0,3)
    chainloader +1
}
```

در اینجا ```insmod chain``` ماژول مربوط به chainloading را در GRUB load می‌کند و ```insmod ntfs``` پشتیبانی لازم برای NTFS را وارد می‌کند همچنین ```set root=(hd0,3)``` فایل سیستم یا پارتیشن مورد نظر را مشخص می‌کند و ```chainloader +1``` به GRUB می‌گوید boot code موجود در ابتدای آن partition را برای اجرای boot loader بعدی استفاده کند. 

در حالت دیگری ، chainloader می‌تواند یک فایل مشخص را مستقیماً اجرا کند. برای مثال یک boot loader قدیمی DOS می‌تواند به شکل زیر مشخص شود :

```
menuentry "DOS" {
    insmod chain
    insmod fat
    set root=(hd0,3)
    chainloader /io.sys
}
```

این مثال نشان می‌ دهد chainloading الزاماً به یک partition boot sector محدود نیست و می‌ تواند در بعضی سناریوها فایل boot loader مشخصی را نیز هدف قرار دهد.

---

### Boot Loader Details

تا اینجا رفتار boot loader را از دید عملی بررسی کردیم. حالا می‌توانیم کمی عمیق‌ تر شویم و ببینیم وقتی کامپیوتر روشن می‌شود، boot loader دقیقاً چگونه وارد صحنه می‌شود. برای PC ها دو مدل اصلی مورد بحث قرار می‌ گیرند ```MBR``` و ```UEFI``` این دو روش تفاوت اساسی در نحوه‌ ی قرار گرفتن boot code و اجرای آن دارند.

### MBR Boot

در روش قدیمی BIOS/MBR، disk دارای MBR است. MBR علاوه بر اطلاعات partition، فضای بسیار کوچکی برای boot code دارد. BIOS بعد از انجام **POST** این بخش ابتدایی را از disk می‌ خواند و اجرا می‌ کند.

#### 🔹 MBR size problem

فضای boot code در MBR بسیار کوچک است. در توضیح کتاب، حدود 441 بایت ، برای بخش code در نظر گرفته می‌شود در حالی که کل sector سنتی MBR شامل موارد دیگری مانند partition table و signature نیز هست. بنابراین یک boot loader کامل نمی‌ تواند در این فضای کوچک قرار بگیرد. در نتیجه از **multistage boot loader** استفاده می‌ شود.

```
MBR
 │
 │ boot.img / first-stage code
 │
 ▼
Second stage / GRUB core
 │
 ▼
GRUB modules
 │
 ▼
grub.cfg
 │
 ▼
Linux kernel
```

کد اولیه فقط آنقدر بزرگ است که بتواند بخش بعدی boot loader را پیدا و load کند.

#### 🔹 The space between the MBR and the first partition

در بسیاری از نصب‌ های قدیمی ، بخش بیشتری از GRUB در فضای بین MBR و اولین partition قرار می‌گرفت. این روش یک ضعف دارد اینکه فضای disk یک filesystem عادی نیست و ممکن است برنامه‌ های دیگر آن را overwrite کنند. به همین دلیل این مدل boot از نظر طراحی چندان تمیز یا ایمن نیست ، اما در بسیاری از نصب‌ های قدیمی GRUB استفاده شده است.

#### 🔹 GPT problem

معمولاً GPT اطلاعات خودش را در بخش‌ هایی از disk قرار می‌ دهد که روش قدیمی استفاده از فضای بعد از MBR را محدود می‌ کند. در BIOS boot از یک disk با GPT ، برای GRUB می‌ توان از یک partition کوچک اختصاصی به نام ```BIOS Boot Partition``` استفاده کرد. این partition برای نگهداری بخش لازم از boot loader استفاده می‌ شود. در پیکربندی‌ های GPT/BIOS ، این partition یک نوع خاص دارد. UUID/type identifier مرتبطی که در متن کتاب برای آن آمده ```21686148-6449-6E6F-744E-656564454649``` است. این سناریو امروزه نسبت به UEFI/GPT کمتر رایج است ، چون معمولاً GPT با UEFI همراه است.

---

### UEFI Boot 

محدودیت‌ های BIOS زیاد بود و به همین دلیل EFI به عنوان جایگزین آن توسعه پیدا کرد. نسخه‌ی مدرن این استاندارد  UEFI هستش که مخفف **Unified Extensible Firmware Interface** است. UEFI امکانات بسیار بیشتری نسبت به BIOS دارد ، از جمله :

- Read partition table
- Access filesystem
- Execute .efi files
- Boot manager
- NVRAM boot entries
- In some firmwares, internal shell

و GPT نیز در معماری‌ های مدرن UEFI بسیار رایج است.
I System Partition

یکی از مهم‌ ترین تفاوت‌ های UEFI با MBR این است که boot code دیگر الزاماً خارج از filesystem قرار ندارد. در عوض یک partition ویژه وجود دارد که به آن ```EFI System Partition``` یا ```ESP``` گفته می‌ شود. ESP معمولاً با filesystem نوع ```VFAT / FAT``` ساخته می‌ شود. در Linux معمولاً در ```boot/efi/``` ممکن است mount شود. در داخل آن directory ای به نام ```EFI``` وجود دارد. ساختارش می‌ تواند چیزی شبیه این باشد : 

```
/boot/efi/
└── EFI/
    ├── Microsoft/
    ├── ubuntu/
    ├── grub/
    └── ...
```

هر boot loader معمولاً directory مربوط به خودش را دارد. فایل‌ های boot loader دارای پسوند efi. هستند. برای مثال ممکن است فایل‌ هایی مانند ```grubx64.efi``` و ```shimx64.efi``` وجود داشته باشند.

#### 🔹 ESP vs BIOS Boot Partition

این دو را نباید یکی دانست !  ```BIOS Boot Partition``` و ```EFI System Partition``` هدف و ساختار متفاوتی دارند. ESP یک filesystem واقعی برای فایل‌ های EFI است ، در حالی که BIOS Boot Partition برای نگهداری بخش‌ هایی از boot loader در سناریو های BIOS/GPT استفاده می‌ شود.

#### 🔹 UEFI and boot loader

نکته‌ ی مهم اینکه نمی‌ توان boot loader قدیمی BIOS را صرفاً داخل ESP کپی کرد و انتظار داشت UEFI آن را اجرا کند. Boot loader باید برای UEFI ساخته شده باشد ، مثلاً برای GRUB باید نسخه‌ ی UEFI آن نصب شود. در UEFI هم firmware می‌تواند filesystem مربوط به ESP را بخواند و فایل efi. را مستقیماً اجرا کند. بنابراین فرآیند بسیار ساده‌ تر از مدل چند مرحله‌ای MBR است.

---

### How GRUB Works

در پایان فصل می‌ توان کل فرآیند GRUB را به چند مرحله‌ ی اصلی تقسیم کرد :

- **مرحله 1 ← Firmware سخت‌افزار را آماده می‌ کند** : BIOS یا UEFI ابتدا سخت‌ افزار را مقداردهی اولیه می‌ کند. سپس بر اساس boot order، storage device های مناسب را بررسی می‌ کند تا boot code یا boot entry موردنظر را پیدا کند.

- **مرحله 2 ← اجرای boot code** : وقتی Firmware boot code را پیدا می‌ کند و آن را اجرا می‌کند از اینجا GRUB وارد صحنه می‌شود. در BIOS، این می‌تواند از MBR شروع شود. در UEFI، firmware می‌تواند فایل EFI مربوط به boot loader را مستقیماً اجرا کند.

- **مرحله 3 ← بارگذاری GRUB core** : بخش اولیه‌ی GRUB اجرا می‌ شود و باید GRUB core را پیدا و load کند. محل این core به نوع boot بستگی دارد.

- **مرحله 4 ← initialization خود GRUB** : وقتی که GRUB core initialization را انجام می‌ دهد. از این نقطه GRUB می‌تواند disk و filesystem هایی را که module های لازم برایشان وجود دارد ، بهتر مدیریت کند.

- **مرحله 5 ← پیدا کردن boot partition و configuration** : باید GRUB فایل سیستم مربوط به خودش را پیدا کند. ممکن است ابتدا از device name استفاده کند ```(hd0,msdos1)``` و سپس با UUID search کند تا partition درست را پیدا کند. سپس configuration مربوط به GRUB را load می‌ کند. معمولاً این configuration همان ```grub.cfg``` است.

- **مرحله 6 ← فرصت تغییر configuration** : در این مرحله GRUB به کاربر اجازه می‌ دهد configuration entry را تغییر دهد. این قابلیت برای recovery و troubleshooting بسیار مهم است. مثلاً می‌توان ```root``` را تغییر داد یا یک kernel parameter را اضافه یا حذف کرد.

- **مرحله 7 ← اجرای configuration** : بعد از timeout یا انتخاب کاربر، GRUB command های موجود در configuration را اجرا می‌ کند. این command ها شامل مواردی هستند مانند :

- set
- search
- insmod
- linux
- initrd

- **مرحله 8 ← load کردن module های مورد نیاز** : در صورت نیاز GRUB module های بیشتری را load می‌ کند. بعضی module ها ممکن است از قبل در بخش ابتدایی boot loader در دسترس باشند. بعضی دیگر هنگام اجرای configuration از filesystem مربوط به GRUB خوانده می‌ شوند.

- **مرحله ۹ ← اجرای kernel** : در نهایت GRUB به مرحله‌ ای میرسد که kernel را load و اجرا می‌کند. در configuration دستور ```linux``` میتواند kernel image را مشخص کرده کند و دستور ```initrd``` در صورت نیاز initial RAM filesystem را مشخص می‌ کند سپس GRUB با دستور ``` boot``` فرآیند اجرای configuration را به نقطه‌ ی انتقال کنترل به Linux kernel می‌ رساند. از اینجا به بعد دیگر مسئولیت اصلی با Linux kernel است.

#### 🔹 Where exactly is the GRUB core ?

یکی از سؤال‌ های مهم در boot این است که GRUB core کجا قرار دارد؟ بسته به معماری boot، چند حالت وجود دارد : 

- **حالت اول ← فضای بین MBR و اولین partition** : در بعضی نصب‌ های BIOS/MBR بخشی از GRUB در فضای بعد از MBR و قبل از اولین partition قرار می‌ گیرد.
- **حالت دوم ← یک partition معمولی** : بعضی اجزای GRUB ممکن است در filesystem یک partition قرار گرفته باشند.
- **حالت سوم ← یک boot partition ویژه** : در سیستم‌ های مدرن یا بعضی پیکربندی‌ های خاص boot code می‌ تواند در partition اختصاصی مانند ```ESP``` یا partition های مخصوص دیگر قرار بگیرد.

#### 🔹 BIOS and boot.img

در مدل BIOS باید firmware ابتدا sector مربوط به MBR را بخواند. این بخش کوچک ، که از فایل‌ هایی مانند boot.img در ساختار GRUB می‌آید ، هنوز GRUB core کامل نیست. وظیفه‌ ی اصلی آن این است که محل بخش بعدی GRUB را پیدا کند و آن را load کند.

```
BIOS
 │
 ▼
MBR / boot.img
 │
 ▼
GRUB core
 │
 ▼
GRUB modules
 │
 ▼
grub.cfg
```

cI and ESP

در UEFI هم firmware میتواند ESP  را بخواند. بنابراین GRUB می‌ تواند به شکل یک فایل .efi روی ESP قرار داشته باشد. Firmware همان فایل را اجرا می‌ کند. اگر Secure Boot فعال باشد ، ممکن است قبل از GRUB یک shim signed قرار بگیرد :

```
UEFI
 │
 ▼
shim
 │
 ▼
GRUB
 │
 ▼
Linux kernel
```

#### 🔹 initrd and the next step

در بسیاری از سیستم‌ های Linux ، داستان با اجرای مستقیم kernel تمام نمی‌شود. Boot loader ممکن است علاوه بر kernel یک image مربوط به initial RAM filesystem را نیز در memory قرار دهد. در GRUB این کار با دستور ```initrd``` مشخص می‌ شود. این image در مرحله‌ ی اولیه‌ ی boot برای فراهم کردن محیط و driver های لازم جهت رسیدن به root filesystem واقعی استفاده می‌ شود. جزئیات دقیق initial RAM filesystem و اینکه چگونه kernel از آن به root filesystem واقعی منتقل می‌ شود در فصل ششم بررسی خواهد شد.

---

### Tips

فصل پنجم در اصل یک زنجیره‌ ی انتقال کنترل را توضیح می‌ دهد :

```
Hardware
    │
    ▼
BIOS / UEFI
    │
    ▼
Boot Loader
    │
    ▼
  GRUB
    │
    ├── Find filesystem
    ├── Find kernel
    ├── Find initrd
    ├── Set kernel parameters
    │    
    ▼
Linux Kernel
    │
    ├── CPU
    ├── Memory
    ├── Bus
    ├── Devices
    ├── Drivers
    ├── Kernel subsystems
    ├── Root filesystem
    │
    ▼
PID 1 / init
    │
    ▼
User Space
```

چند نکته ی اصلی که باید از این فصل در ذهن بماند :

1. **اولین مرحله‌ی boot مربوط به Firmware است** : روی PC های مدرن معمولاً با UEFI سروکار داریم ، ولی BIOS/MBR هنوز برای فهم boot سنتی اهمیت دارد.
2. **همیشه Boot loader قبل از Linux kernel اجرا می‌ شود**: بنابراین نمی‌ تواند به driver های معمول Linux وابسته باشد.
3. **باید Boot loader بتواند storage و filesystem را تا حد لازم خودش مدیریت کند**. 
4. **محیط GRUB ، یک محیط مستقل از Linux kernel است** : command هایی مانند ls ، set و insmod در این محیط متعلق به خود GRUB هستند.
5. **الزاماً root در GRUB و root در Linux یک معنی ندارند** : set root معمولاً مربوط به filesystem مورد استفاده‌ ی خود GRUB است ، در حالی که root بعد از command linux یک kernel parameter برای تعیین root filesystem لینوکس است.
6. **جست و جو UUID برای پیدا کردن filesystem بسیار مهم است** : هم kernel و هم GRUB می‌توانند از شناسه‌ های پایدار استفاده کنند تا به ترتیب نام‌ گذاری device ها وابسته نباشند.
7. **پارامتر های kernel رفتار kernel را مشخص می‌کنند** : proc/cmdline راه ساده‌ ای برای دیدن command line مربوط به kernel در حال اجراست.
8. **معنای ro یعنی read-only بودن اولیه‌ی root filesystem است** : این حالت برای مراحل اولیه‌ ی بررسی و آماده‌ سازی filesystem مناسب است و سیستم بعداً می‌ تواند آن را read-write کند.
9. **قابلیت initrd/initramfs مشکل dependency مربوط به driver های لازم برای رسیدن به root filesystem را حل می‌کند** ، جزئیات آن در فصل ششم خواهد آمد.
10. **مدل های MBR و UEFI دو مدل اصلی boot در PC هستند** : MBR به فضای بسیار محدود boot code متکی است و معمولاً به multistage boot نیاز دارد.
11. **همیشه UEFI از EFI System Partition استفاده می‌کند** : در ESP فایل‌ های efi. مربوط به boot loader ها قرار می‌ گیرند.
12. **قابلیت Secure Boot اجرای boot software را به code های مورد اعتماد محدود می‌کند** : توزیع‌ های بزرگ Linux معمولاً با استفاده از signed boot loader ها و shim با این سازوکار کار می‌ کنند.
13. **قابلیت Chainloading امکان اجرای boot loader یک سیستم‌عامل دیگر از طریق GRUB را فراهم می‌کند**.
14. **در نهایت GRUB کنترل را به Linux kernel منتقل می‌ کند**.

و مهم‌ تر از همه ، نقطه‌ ای که باید در پایان این فصل کاملاً روشن باشد این است :

```
Firmware
   ↓
Boot Loader
   ↓
Linux Kernel
   ↓
Root Filesystem
   ↓
PID 1 / init
   ↓
User Space
```

فصل پنجم تقریباً تا مرز ورود به user space پیش میرود. در پایان این فصل kernel کنترل را در دست دارد و آماده است اولین user-space process را اجرا کند.
