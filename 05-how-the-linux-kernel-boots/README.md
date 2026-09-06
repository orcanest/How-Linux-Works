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
- []()
- []()
- []()
- []()
- []()
- []()
- []()
- []()
- []()
- []()
- []()
- []()
- []()
- []()
- []()
- []()
- []()
- []()
- []()
- []()
- []()
- []()

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







