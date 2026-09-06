## دیسک‌ ها و فایل‌ سیستم‌ ها
### 🐧 فصل چهارم کتاب How Linux Works

فصل سوم دیدیم که kernel چطور device هایی مثل disk رو معرفی و مدیریت می‌کنه. فصل چهارم یک قدم جلوتر میره. چطور واقعاً با این disk ها کار کنیم ؟ چطور پارتیشن‌ بندیشون کنیم ، فایل‌ سیستم روشون بسازیم و این لایه‌ ها چطور روی هم قرار میگیرن ؟

در این فصل دیگه فقط با این سؤال رو به‌ رو نیستیم که «dick چیه؟» ، بلکه می‌خوایم بفهمیم وقتی یک فایل معمولی مثل ```home/user/file.txt/``` رو روی یک سیستم لینوکسی باز می‌کنیم ، پشت صحنه چه اتفاقی میفته و داده‌ ی اون فایل چطور از لایه‌ های مختلف storage عبور می‌ کنه.

کتاب فصل رو از partition ها شروع می‌کنه ، بعد وارد filesystem ها میشه ، mount و ```etc/fstab/``` رو بررسی می‌کنه ، درباره‌ ی filesystem checking و swap صحبت می‌کنه و در نهایت وارد LVM میشه. در بخش پایانی هم از abstraction های سطح بالا پایین‌ تر میاد و ساختار داخلی  filesystem هایی مثل ext2/ext3/ext4 و مفهوم inode رو بررسی می‌کنه.

---
📚 Table of Contents

- [From empty disk to data file](#from-empty-disk-to-data-file)
- [Disk partitioning](#disk-partitioning)
- [Types of Partition Tables](#types-of-partition-tables)
- [Partitioning tools](#partitioning-tools)
- [Device Mapper and LVM](#device-mapper-and-lvm)
- [Partition Table in Kernel](#partition-table-in-kernel)
- [Changing Partition Tables](#changing-partition-tables)
- [fdisk vs parted](#fdisk-vs-parted)
- [Creating a Partition Table](#creating-a-partition-table)
- [Disk and Cylinder Geometry](#disk-and-cylinder-geometry)
- [Reading from SSD disks](#reading-from-ssd-disks)
- [File systems](#file-systems)
- [VFS](#vfs)
- [Types of filesystems](#types-of-filesystems)
- [Creating a file system](#creating-a-filesystem)
- [Mounting filesystem](#Mounting-filesystem)
- [UUID](#uuid)
- [Disk buffering and caching](#disk-buffering-and-caching)
- [Mount options](#mount-options)
- [Remounting a file system](#remounting-a-file-system)
- [Filesystem table](#filesystem-table)
- [fstab alternatives](#fstab-alternatives)
- [Filesystem capacity](#Filesystem-capacity)
- [Check and repair the file system](#check-and-repair-the-file-system)
- [Special purpose filesystems](#special-purpose-filesystems)
- [swap space](#swap-space)
- [Using a Partition as Swap](#using-a-Partition-as-Swap)
- [Using a file as Swap](#using-a-file-as-swap)
- [How much swap do we need?](#how-much-swap-do-we-need)
- [Introduction to Logical Volume Manager](#introduction-to-logical-volume-manager)
- [Working with LVM](#working-with-lvm)
- [Creating Physical Volume and Volume Group](#creating-physical-volume-and-volume-group)
- [Creating Logical Volumes](#creating-logical-volumes)
- [Working with Logical Volumes](#working-with-logical-volumes)
- [Delete Logical Volume](#delete-logical-volume)
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

### From empty disk to data file

یک دیسک لینوکسی چند تا لایه داره:

اول **partition table** که مشخص می‌ کنه دیسک به چه بخش‌ هایی تقسیم شده ، بعد خود **partition ها** ، بعدش **filesystem** که ساختار فایل‌ ها و دایرکتوری‌ ها رو مدیریت می‌کنه و در نهایت داده‌ ی خود فایل‌ ها. برای دسترسی به داده‌ ی یک فایل ، kernel باید از لایه‌ های پایین‌ تر عبور کنه تا به filesystem مناسب برسه و filesystem هم از روی ساختار خودش مشخص کنه فایل مورد نظر کجاست. در ساده‌ ترین حالت میشه این ساختار رو اینطور دید:

```
Disk 
├── Partition Table 
├── Partition
│ └── Filesystem
│     ├── Filesystem Data Structures
│     └── File Data
└── Partition
```


<img width="100%" height="465" alt="image" src="https://github.com/user-attachments/assets/cbd44881-5977-453b-98bf-f724f492c65a" />


کتاب تأکید می‌ کنه که این تصویر یک schematic ساده‌ ست و قرار نیست اندازه‌ ی واقعی بخش‌ های مختلف دیسک رو نمایش بده. برای دسترسی به این داده‌ ها ، کرنل از یک سیستم لایه‌ بندی‌ شده استفاده می‌کنه. از پایین ، سخت‌ افزار storage قرار داره ، بعد زیرسیستم SCSI و driver های دیگه که در فصل قبل دیدیم، بعد رابط block device و لایه‌ی partition ها و از اون‌ جا به بعد مسیر می‌تونه از filesystem عبور کنه یا در بعضی موارد مستقیماً با خود block device کار بشه. یعنی از دید ساده می‌ تونیم دو مسیر اصلی داشته باشیم :

```
Hardware
↓
SCSI / Other Device Subsystems
↓
Block Device Interface
↓
Partition
↓
Filesystem
↓
File Data
```

یا در صورت دسترسی مستقیم :
```
Hardware
↓
SCSI / Other Device Subsystems
↓
Block Device Interface
↓
Direct Device Access
```

<img width="100%" height="615" alt="image" src="https://github.com/user-attachments/assets/3356dbf6-2478-45fd-97e0-64954539d2d8" />


در این تصویر ساده‌ شده LVM هم نمایش داده نشده ، اما در معماری واقعی می‌ تونه بخشی از همین لایه‌ ی block device رو تشکیل بده و یک لایه‌ ی abstraction بین block device های فیزیکی و filesystem ایجاد کنه.

---

### Disk partitioning

پارتیشن‌ ها زیر مجموعه‌ های یک دیسک کامل هستن. توی لینوکس با یک عدد بعد از اسم device کامل نام‌ گذاری میشن، مثل:

- /dev/sda1
- /dev/sda2
- /dev/sdb3


<img width="100%" height="151" alt="image" src="https://github.com/user-attachments/assets/0aa33c2c-06cb-45d1-a68f-e0a8d124febe" />


کرنل هر partition رو هم به‌ صورت یک ```block device``` جدا در اختیار سیستم قرار میده. یعنی ```dev/sda/``` نماینده‌ ی کل دیسکه و ```dev/sda1/``` و ```dev/sda2/``` نماینده‌ ی partition های مختلف همون دیسک هستن. Partition ها در بخشی از دیسک به اسم **partition table** تعریف میشن که بهش **disk label** هم گفته میشه.

در گذشته داشتن چند partition روی یک دیسک کاربرد های بیشتری داشت. یکی از دلایلش محدودیت‌ های سیستم‌ های قدیمی برای boot بود و دلیل دیگه این بود که administrator می‌ خواست بخشی از فضا رو برای سیستم‌ عامل و سرویس‌ های مهم رزرو کنه تا کاربرها نتونن تمام فضای سیستم رو پر کنن.

امروزه هم partition های جداگانه هنوز در بسیاری از سیستم‌ها استفاده میشن. برای مثال ممکنه boot/ و root و EFI System Partition و home و filesystem یا swap به شکل جداگانه وجود داشته باشن. kernel می‌تونه هم‌ زمان هم کل دیسک و هم partition های داخل اون رو به‌ عنوان block device در اختیار قرار بده ، ولی معمولاً نباید بدون دلیل روی کل دیسک و partition های اون به‌ طور هم‌ زمان عملیات filesystem انجام بدیم. یکی از مواردی که دسترسی مستقیم به کل دیسک منطقیه ، کپی کردن یا image گرفتن از کل دیسکه.

---

### Types of Partition Tables

دو نوع اصلی partition table که باید بشناسیم :

#### 🔹 MBR Partition

پارتیشن **MBR** یا **Master Boot Record** روش قدیمی‌تر partitioning محسوب میشه. در مدل MBR محدودیت‌ هایی وجود داره و یکی از معروف‌ ترین اون‌ ها محدودیت تعداد partition های primary و محدودیت‌ های مربوط به اندازه‌ی دیسکه. در MBR سه نوع partition داریم :

- پارتیشن Primary : پارتیشن معمولی که مستقیماً در partition table اصلی MBR تعریف میشه. در مدل MBR حداکثر چهار primary partition می‌تونیم داشته باشیم.
- پارتیشن Extended : اگر بیشتر از چهار partition لازم داشته باشیم ، می‌تونیم یکی از entry های primary رو به extended partition اختصاص بدیم. Extended partition خودش محلی برای نگهداری partition های Logical فراهم می‌کنه.
- پارتیشن Logical : پارتیشن هایی هستن که داخل extended partition قرار می‌گیرن. 

به این ترتیب ساختار می‌ تونه چیزی شبیه این باشه :
```
MBR
├── Primary
├── Primary
├── Extended
│ ├── Logical
│ ├── Logical
│ └── Logical
└── Primary
```

#### 🔹 GPT Partition

پارتیشن **GPT** یا **GUID Partition Table** استاندارد جدیدتر و انعطاف‌ پذیرتریه. GPT محدودیت‌ های ساختار MBR رو تا حد زیادی برطرف کرده و برای سیستم‌ های مدرن ، مخصوصاً سیستم‌ هایی که از UEFI استفاده می‌کنن ، انتخاب رایج‌ تریه.

---

### Partitioning tools

برای کار با partition table چند ابزار معروف وجود داره :

- ابزار parted : ابزار خط فرمانی برای مدیریت partition هاست و از MBR و GPT پشتیبانی می‌کنه. برای مشاهده‌ ی partition table ها می‌تونید از ```parted -l``` استفاده کنید.
- ابزار gparted : نسخه‌ ی گرافیکی مبتنی بر parted هست و برای کاربرهایی که محیط GUI رو ترجیح میدن کاربردیه.
- ابزار fdisk : یکی از ابزارهای  معروف Linux برای partitioning هستش. نسخه‌ های مدرن fdisk از partition table های مختلف از جمله MBR و GPT پشتیبانی می‌کنن.


---

### Device Mapper and LVM

یک نکته‌ ی مهم در سیستم‌ هایی که LVM دارن اینه که ممکنه در partition table با نوعی partition type مربوط به Linux LVM مواجه بشید.در MBR ، مقدار تاریخی 8e برای Linux LVM استفاده می‌شد. در سیستم‌های GPT ، نوع partition با GUID مشخص میشه و نباید انتظار داشته باشید همان مقدار 8e را ببینید. همچنین device هایی مثل ```*-dev/dm/``` و ```*dev/mapper/``` به **Device Mapper** مربوط هستن. اما Device Mapper فقط مخصوص LVM نیست ، فناوری‌ هایی مثل LVM و device های رمزنگاری‌ شده ، بعضی پیاده‌ سازی‌ های RAID و multipath که می‌ تونن از Device Mapper استفاده کنن. پس دیدن ```dev/dm-0/``` به‌ تنهایی به این معنی نیست که حتماً LVM روی سیستم وجود دارد.

---

### Partition Table in Kernel

وقتی کرنل partition table یک block device رو می‌ خونه ، اطلاعات partition های پیدا شده در kernel log قابل مشاهده‌ ست. مثلاً می‌تونید از ```journalctl -k``` برای دیدن پیام‌ های kernel استفاده کنید. این اطلاعات برای troubleshooting زمانی مفیده که یک partition ایجاد شده ولی سیستم هنوز اون رو به شکلی که انتظار دارید مشاهده نمی‌کنه.

---

### Changing Partition Tables

دیدن partition table کاملاً بی‌خطره ، ولی تغییر دادن اون می‌ تونه بسیار خطرناک باشه. اگر partition رو حذف یا بازتعریف کنید ، ممکنه metadata لازم برای پیدا کردن داده‌ های اون partition از بین بره. برای همین قبل از هر تغییر جدی روی partition table:

- از داده‌ های مهم backup بگیرید.
- مطمئن بشید device درست رو انتخاب کردید.
- بررسی کنید partition موردنظر در حال استفاده نباشه.
- اگر filesystem روی partition mount شده، اون رو unmount کنید.
- قبل از اجرای دستورهای destructive چند بار device name رو بررسی کنید.

یک اشتباه ساده بین ```dev/sda/``` و ```dev/sdb/``` می‌ تونه نتیجه‌ ی بسیار جدی داشته باشه.

---

### fdisk vs parted

یک تفاوت مهم بین fdisk و parted در نحوه‌ی اعمال تغییرات هستش ، در fdisk شما وارد یک محیط تعاملی می‌ شید. می‌ تونید تغییرات مختلف رو طراحی کنید ، partition ها رو حذف یا ایجاد کنید و قبل از نوشتن نهایی ، partition table رو بررسی کنید.

- برای نمایش partition table داخل fdisk معمولاً از ```p``` استفاده می‌کنید.
- برای حذف ```d``` استفاده می‌کنید.
- برای ساخت partition جدید ```n``` استفاده می‌کنید.
- برای نوشتن تغییرات روی دیسک ```w``` استفاده میشه.
- اگر متوجه شدید اشتباهی انجام دادید و هنوز تغییرات رو روی دیسک ننوشته‌اید ، می‌تونید با ```q``` بدون ذخیره خارج بشید. این یکی از مزیت‌های مهم محیط تعاملی fdisk برای کارهای حساسه.

در مقابل، parted بیشتر به شکل command-oriented کار می‌کنه و تغییرات معمولاً در همان زمان اجرای command اعمال میشن. بنابراین باید هنگام اجرای هر دستور دقت بیشتری داشته باشید.

هر دو ابزار در **user space** اجرا میشن و برای دسترسی به block device از interface های kernel استفاده می‌ کنن. بعد از تغییر partition table کرنل باید از تغییرات مطلع بشه تا partition های جدید یا تغییر کرده رو دوباره در block-device layer در نظر بگیره.

---

### Creating a Partition Table

کتاب یک مثال عملی با fdisk ارائه میده که در اون روی یک فلش حدوداً 4 گیگابایتی یک partition table جدید ساخته میشه و دو partition ایجاد میشه:

- یک partition حدود ۲۰۰MB
- یک partition که بیشتر فضای باقی‌مانده رو استفاده می‌ کنه.

هدف اصلی این مثال، خود اندازه‌ها نیست بلکه یاد گرفتن workflow کار با fdisk هستش. فرآیند کلی :

```
View partition table
↓
Delete previous partitions if needed
↓
Create new partition
↓
Check partition table
↓
Write changes
```

- دستور ```d``` برای حذف partition استفاده میشه.
- دستور ```n``` برای ساخت partition جدید.
- دستور ```p``` برای مشاهده‌ ی partition table داخل fdisk.
- دستور ```w``` برای نوشتن تغییرات روی دیسک.
- اگر متوجه شدید اشتباهی انجام داده‌اید و هنوز چیزی نوشته نشده دستور ```q``` برای خروج بدون ذخیره کاربرد دارد.

نکته‌ ی مهم اینه که در مثال‌ های آموزشی partitioning نباید device مورد استفاده در کتاب رو کورکورانه روی سیستم خودتون اجرا کنید. ممکنه ```dev/sdb/``` در سیستم کتاب یک فلش USB باشه، ولی در سیستم شما ```dev/sdb/``` یک دیسک اصلی و حاوی اطلاعات مهم باشه.

به همین دلیل در virtualbox یک partition جدید اضافه می کنیم و آن را پارتیشن بندی می کنیم :
```
Partitions :
1 primary =>  partition number: 2 /dev/sdb2 Size:1G
1 extended => partition number: 3 /dev/sdb3 Size:8G
  ├── Logical => partition number: 1 /dev/sdb5 Size:2G
  └── Logical => partition number: 2 /dev/sdb6 Size:3G
> 3G Free for Future
```
<img width="100%" height="512" alt="image" src="https://github.com/user-attachments/assets/56a242ea-1341-492f-8be1-3ff46cf77e46" />

<img width="100%" height="566" alt="image" src="https://github.com/user-attachments/assets/29080c4b-9b95-4312-9263-6412ac12258f" />

---

### Disk and Cylinder Geometry

دیسک‌ های قدیمی با قطعات متحرک ، ساختار مکانیکی مشخصی دارند. یک hard disk قدیمی شامل platter هایی هست که روی یک spindle می‌چرخن. یک head روی یک arm قرار داره و arm می‌تونه head رو روی سطح platter جا به‌ جا کنه. وقتی arm در یک موقعیت مشخص قرار گرفته ، head می‌ تونه روی مسیر دایره‌ ای مشخصی از سطح دیسک قرار بگیره ، به چنین مجموعه‌ ای در مدل قدیمی آدرس‌دهی ، **cylinder** گفته میشد. مدل قدیمی آدرس‌ دهی دیسک با سه مفهوم شناخته میشد ```Cylinder``` و ```Head``` و ```Sector``` که بهش **CHS** گفته میشه.

<img width="100%" height="335" alt="image" src="https://github.com/user-attachments/assets/ceed3b19-eeec-4db8-80b5-768478bcbd7e" />

در این مدل، cylinder مشخص می‌ کنه head در چه موقعیت شعاعی قرار داره ، head مشخص می‌ کنه کدام سطح مورد نظر قرار گرفته و sector هم بخشی از مسیر چرخشی دیسکه.اما این مدل برای storage های مدرن دیگه تصویر واقعی خوبی از نحوه‌ ی دسترسی به داده ارائه نمی‌دهد.

در دیسک‌ های امروزی ، سیستم‌ عامل عمدتاً با **LBA — Logical Block Addressing** کار می‌ کنه. در LBA به‌ جای اینکه سیستم‌ عامل مجبور باشه درباره‌ی cylinder و head و sector فیزیکی تصمیم بگیره ، بلوک‌ ها با شماره‌ های منطقی آدرس‌ دهی میشن در نتیجه : ```Logical Block 0``` و ```Logical Block 1``` و ```Logical Block 2``` و ... به‌ عنوان واحدهای قابل دسترسی در اختیار سیستم قرار می‌گیرن. با این حال ، ردپای CHS هنوز در بعضی ساختارهای قدیمی مثل MBR دیده میشه و ممکنه ابزارها مقادیری برای CHS نمایش بدن که الزاماً بیانگر geometry واقعی سخت‌افزار مدرن نیستن.

---

### Reading from SSD disks

دیسک‌ های بدون قطعه‌ ی متحرک مثل SSD از نظر فیزیکی کاملاً با hard disk های قدیمی متفاوت هستن. در SSD دیگه platter ، spindle و head متحرک نداریم اما این به معنی بی‌ اهمیت شدن تمام مفاهیم مربوط به layout نیست. SSD داده رو در ساختارهای داخلی خودش ، از جمله **page ها** و **block ها** مدیریت می‌ کنه. کنترلر SSD بین logical block هایی که سیستم‌ عامل می‌ بینه و ساختار فیزیکی NAND flash قرار می‌گیره بنابراین سیستم‌ عامل مستقیماً با page های فیزیکی NAND کار نمی‌ کنه. یکی از موضوعات مهم در اینجا **partition alignment** هستش. اگر شروع partition با مرزهای مناسب storage هم‌ تراز نباشه ممکنه بعضی عملیات‌ های read و مخصوصاً write باعث کار اضافه در لایه‌ های پایین‌ تر storage بشن.

این موضوع مخصوصاً در نسل‌ های قدیمی‌ تر SSD و storage های مختلف اهمیت بیشتری داشت. ابزارهای مدرن partitioning معمولاً partition ها رو با alignment مناسب ایجاد می‌کنن و alignment یک مگابایت یک convention رایج برای شروع partition های مدرن محسوب میشه. بنابراین معمولاً نیازی نیست کاربر به‌صورت دستی این محاسبات رو انجام بده ، ولی در سیستم‌ های خاص یا storage های خاص، alignment همچنان موضوع مهمی برای بررسیه.

<img width="100%" height="500" alt="Difference-Between-HSS-and-SSD" src="https://github.com/user-attachments/assets/70c20d29-4b8e-40b9-abcb-9e4aa3d475fe" />

---

### File systems

آخرین حلقه‌ ی اتصال بین kernel و user space برای کار معمول با storage و **filesystem** هستش. وقتی دستور هایی مثل ```ls``` و ```cd``` و ```cp``` و ```mv``` و ```rm``` اجرا می‌ کنید ، معمولاً با یک filesystem سروکار دارید ، نه مستقیماً با block های خام دیسک. Filesystem ساختاریه که روی یک block device یا منبع ذخیره‌ سازی قرار می‌ گیره و داده‌ ها و metadata مربوط به فایل‌ ها و دایرکتوری‌ ها رو مدیریت می‌ کنه. از دید مفهومی می‌تونیم filesystem رو شبیه یک database در نظر بگیریم که می‌دونه:

- چه فایل‌هایی وجود دارند.
- چه directory هایی وجود دارند.
- سطح دسترسی permission هر object چیه.
- هر فایل چه metadata ای دارد.
- داده‌ ی فایل کجا قرار گرفته.
- کدام block ها آزاد هستن.
- کدام block ها استفاده شدن.

ساختار درختی filesystem اونقدر انعطاف‌ پذیره که الزاماً مجبور نیست به storage فیزیکی متصل باشه. برای مثال ```proc/``` و ```sys/``` از filesystem های ویژه‌ ای استفاده می‌ کنن که داده‌ های kernel و system information رو به شکل فایل و directory در اختیار user space قرار میدن. Filesystem ها معمولاً در kernel پیاده‌ سازی میشن ، اما راه‌ هایی هم برای پیاده‌سازی filesystem در user space وجود دارد. یکی از معروف‌ ترین این روش‌ها **(FUSE (Filesystem in Userspace** است.

---

### VFS

یک لایه‌ ی بسیار مهم در kernel لینوکس **(VFS (Virtual File System** هستش. VFS نقش یک abstraction layer رو ایفا می‌ کنه. یعنی برنامه‌ ی user space نباید برای هر filesystem مختلف یک API جداگانه یاد بگیره. مثلاً برنامه می‌تونه از system call هایی استفاده می کنه مثل :

- open()
- read()
- write()
- close()
- stat()

لایه‌ ی VFS درخواست رو دریافت می‌کنه و اون رو به implementation مناسب filesystem منتقل می‌کنه. به همین دلیل یک برنامه می‌ تونه با filesystem های بسیار متفاوت کار کنه بدون اینکه لازم باشه منطق مخصوص هر filesystem رو داخل خودش پیاده‌ سازی کنه. این موضوع شبیه نقشی هست که SCSI subsystem برای  device های مختلف ایفا می‌کنه :

```
User Space
↓ 
VFS 
↓ 
Filesystem Implementation 
↓ 
Block Layer 
↓ 
Device
```
در نتیجه VFS یکی از مهم‌ترین abstraction های linux هستش.

---

### Types of filesystems

چند filesystem مهم که در لینوکس با اون‌ ها رو به‌ رو می‌ شید:

**فایل سیستم ext4 :** یکی از filesystem های بسیار رایج لینوکس هستش قبل از اون هم فایل سیستم های ```ext2``` و ```ext3``` وجود داشتن و ext3 به‌ عنوان توسعه‌ ای از ext2 با اضافه شدن journal شناخته می‌شد. ext4 قابلیت‌ ها و محدودیت‌ های متفاوتی نسبت به نسل‌های قبلی دارد و در بسیاری از توزیع‌ های لینوکس filesystem شناخته‌ شده‌ای محسوب میشه.

**فایل سیستم btrfs :** یک filesystem مدرن‌ تر لینوکس هستش که با هدف‌ هایی مثل مقیاس‌پذیری ، مدیریت پیشرفته‌ تر storage و قابلیت‌ های filesystem مدرن طراحی شده.

**فایل سیستم FAT Family :** خانواده‌ی FAT شامل filesystem هایی مثل ```msdos``` و ```vfat``` و ```exfat``` هستند. این filesystem ها ریشه در اکوسیستم DOS/Windows دارن و به دلیل compatibility بالا روی media های قابل‌ جا به‌ جایی زیاد دیده میشن مثل: 

- USB flash drive
- memory card
- external storage

**فایل سیستم XFS :** یک filesystem قدرتمند و پرکاربرد در محیط‌ های لینوکسی هستش و مخصوصاً در بعضی توزیع‌ های enterprise مثل RHEL کاربرد زیادی داشته.

**فایل سیستم HFS+ :** مربوط به نسل‌ های قدیمی‌ تر سیستم‌ عامل‌ های Apple محسوب میشه.

**فایل سیستم ISO 9660 :** استاندارد filesystem مربوط به CD-ROM هاست. این filesystem با هدف سازگاری بین سیستم‌ های مختلف طراحی شده و محدودیت‌ها و ویژگی‌ های خاص media های نوری رو دارد.

---

### Creating a filesystem

بعد از partitioning ، مرحله‌ ی بعدی ساخت filesystem هستش ، این کار در user space انجام میشه. یکی از دستورهای رایج : 

```mkfs -t filesystem device```

برای مثال:

```mkfs -t ext4 /dev/sda1```

در سیستم‌ های مدرن معمولاً می‌تونید مستقیماً از ابزار اختصاصی filesystem هم استفاده کنید :

```mkfs.ext4 /dev/sda1```

دستور mkfs در واقع یک frontend برای ابزارهای مختلف filesystem محسوب میشه مثلاً ```mkfs.ext4``` به ابزارهای خانواده‌ ی e2fsprogs مربوط میشه و در سیستم‌های رایج ، mkfs.ext4 به mke2fs متصل یا ارجاع داده میشه.

<img width="100%" height="393" alt="image" src="https://github.com/user-attachments/assets/0bdb5c95-4e9a-4937-bc06-a9b1876793bb" />

#### 🔹 Superblock

در filesystem هایی مثل ext4 یک ساختار بسیار مهم به اسم superblock وجود دارد. Superblock اطلاعات سطح بالایی درباره‌ ی filesystem رو نگه می‌دارد برای مثال اطلاعاتی درباره‌ ی :

- اندازه‌ ی filesystem
- اندازه‌ ی block
- وضعیت filesystem
- تعداد block ها
- تعداد inode ها
- و metadata های مهم دیگر

در این ساختار یا ساختارهای مرتبط ذخیره میشن. به دلیل اهمیت  superblock، در filesystem هایی مثل ext2/ext3/ext4 برای replica های backup از superblock هم وجود دارند.

#### 🔹 Important mkfs warning ⚠️

اگر روی یک partition که قبلاً داده داشته ```mkfs.ext4 /dev/sda1``` اجرا کنید ، دارید filesystem جدیدی روی اون device ایجاد می‌کنید. این کار metadata قبلی filesystem رو از بین میبره و می‌ تونه دسترسی عادی به داده‌ های قبلی رو نابود کنه. بنابراین قبل از mkfs همیشه باید مطمئن بشید device موردنظر واقعاً همون device هست که می‌خواید format کنید.

---

### Mounting filesystem

فرآیند وصل کردن یک filesystem به namespace فایل سیستم در حال اجرای لینوکس رو **mounting** گفته میشه برای mount کردن معمولاً به دو چیز نیاز دارید ، filesystem یا device منبع و mount point مثلاً:

```mkdir /mnt/data```
```mount /dev/sdb1 /mnt/data```

در اینجا ```dev/sdb1/``` منبع filesystem هستش و ```mnt/data/``` هم mount point محسوب میشه. بعد از mount باید ```mnt/data/``` به ریشه‌ ی filesystem جدید تبدیل میشه و فایل‌ ها و  directory های داخل اون از این مسیر قابل مشاهده هستن. اگر لازم باشه می‌تونید نوع filesystem رو هم مشخص کنید:

```mount -t ext4 /dev/sdb1 /mnt/data```

اما در بسیاری از موارد mount می‌تونه نوع filesystem رو از روی metadata تشخیص بده و نیازی به t- نیست. برای دیدن filesystem های mount شده هم دستور ```mount``` رو بدون آرگومان اجرا کنید.
برای جدا کردن filesystem :
```umount /mnt/data```
یا در بعضی موارد:
```umount /dev/sdb1```

---

### UUID

برای mount کردن بر اساس اسم device یک مشکل مهم دارد. نام‌ هایی مثل ```dev/sda/``` و ```dev/sdb/``` لزوما شناسه‌ ی دائمی یک سخت‌افزار مشخص نیستن. ترتیب شناسایی device ها ممکنه تغییر کنه و در نتیجه device که امروز ```dev/sdb1/``` هست، ممکنه بعد از تغییر configuration یا ترتیب discovery نام دیگری داشته باشه. برای همین filesystem ها معمولاً یک **UUID** دارند. UUID یک شناسه‌ی نسبتاً پایدار برای filesystem هستش که هنگام ساخت filesystem تولید میشه برای مشاهده‌ ی UUID ها می‌تونید از ```blkid``` استفاده کنید.

<img width="100%" height="90" alt="image" src="https://github.com/user-attachments/assets/70c89f5d-11b5-4019-bd3f-fa00ef8b69ef" />

می‌تونید به‌ جای وابستگی به نام device از UUID استفاده کنید.

```mount UUID=... /mnt```

به همین دلیل در تنظیمات دائمی mount، مخصوصاً ```etc/fstab/``` استفاده از UUID بسیار رایج هستش.

---

### Disk buffering and caching

لینوکس مثل سیستم‌ های یونیکسی ، برای بهبود performance عملیات I/O از buffering و caching استفاده می‌ کنه. وقتی یک process تغییری در فایل ایجاد می‌ کنه ، این به این معنی نیست که در همان لحظه تمام داده‌ ی مربوط به تغییر حتماً روی storage فیزیکی نوشته شده. Kernel می‌تونه داده‌ ها و metadata های تغییر کرده رو برای مدتی در RAM نگه داره و بعداً اون‌ ها رو به storage منتقل کنه. 

این کار باعث میشه تعداد و نحوه‌ ی عملیات I/O بهتر مدیریت بشه و performance افزایش پیدا کنه. از طرف دیگه Kernel از RAM برای cache کردن داده‌های خوانده‌ شده هم استفاده می‌کنه. اگر یک داده قبلاً در page cache موجود باشه ، ممکنه درخواست بعدی بدون مراجعه‌ی مستقیم به storage پاسخ داده بشه.

#### 🔹 sync

برای درخواست flush کردن داده‌ های buffered به storage می‌تونید از دستور ```sync``` استفاده کنید. همچنین وقتی یک filesystem به شکل صحیح unmount میشه ، kernel قبل از جدا کردن filesystem باید عملیات لازم برای نوشتن داده‌های pending رو انجام بده. به همین دلیل خاموش کردن ناگهانی سیستم می‌ تونه خطرناک باشه چون ممکنه بخشی از metadata یا داده هنوز در RAM بوده باشه و روی storage نوشته نشده باشه. البته sync به‌ تنهایی جایگزین shutdown صحیح سیستم نیست.

---

### Mount options

دستور mount هم option های بسیار زیادی دارد. این option ها رو میشه به دو گروه کلی تقسیم کرد:

-  گروه اول General Options  : به‌ صورت public در mount کردن کاربرد دارن.
- گروه دوم Filesystem-Specific Options : مربوط به یک filesystem خاص هستن.

برای مشخص کردن mount option ها معمولاً از ```o-``` استفاده کنید که filesystem رو به‌ صورت read-only mount می‌کنه برای مثال :

```mount -o ro /dev/sdb1 /mnt```

#### 🔹 ro and rw

- ro → read-only
- rw → read-write

#### 🔹 exec and noexec

با ```exec``` اجازه ی اجرای برنامه‌ ها از filesystem رو میدهد در مقابل ```noexec``` اجرای مستقیم  executable ها از اون mount point رو محدود می‌ کند.

#### 🔹 suid and nosuid

 با ```suid``` اجازه‌ ی اعمال semantics مربوط به set-user-ID و set-group-ID روی executable ها رو حفظ می‌کنه. در مقابل ```nosuid``` اجرای setuid/setgid از اون filesystem رو محدود می‌ کند.

#### 🔹 -r

برای mount کردن به شکل read-only استفاده میشه معادل مفهومی اون ```ro``` هست.

#### 🔹 -n

از update کردن ```etc/mtab/``` جلوگیری می‌ کنه. این option در شرایط خاصی مثل مراحل اولیه‌ ی boot یا محیط‌ های recovery می‌تونه کاربرد داشته باشه. در سیستم‌ های مدرن ```etc/mtab/``` ممکنه symbolic link به فایل یا interface دیگری مثل ```proc/mounts/``` باشه ، بنابراین رفتار دقیق mtab در همه‌ ی سیستم‌ ها یکسان نیست.

---

### Remounting a file system

گاهی filesystem از قبل mount شده ، ولی می‌خواید mount option های اون رو تغییر بدید. در این حالت می‌تونید از ```remount``` استفاده کنید. یکی از کاربرد های مهمش زمانی هست که filesystem root در محیط recovery به شکل read-only در دسترسه و می‌خواید اون رو read-write کنید. مثلاً:

```mount -o remount,rw /```

در این حالت filesystem جدیدی mount نمی‌کنید بلکه option های mount موجود رو تغییر می‌دید.

---

### Filesystem table

برای اینکه filesystem ها موقع boot به‌صورت خودکار mount بشن، لینوکس معمولاً از فایل ```etc/fstab/``` استفاده می‌ کنه. هر entry در این فایل معمولاً شش فیلد داره :

```device   mountpoint   filesystem   options   dump   fsck-order```

برای مثال:
```UUID=...   /   ext4   defaults   0   1```

<img width="100%" height="214" alt="image" src="https://github.com/user-attachments/assets/2c3871c1-d1bc-4245-9d92-8a0a71a217fd" />

- فیلد اول Device : می‌ تونه اسم device به صورت dev/sda1/ یا UUID یا روش‌ های دیگر شناسایی مثل LABEL باشه در سیستم‌ های مدرن استفاده از UUID یا identifier های پایدار معمولاً ترجیح داده میشه.
- فیلد دوم Mount Point : مشخص می‌ کنه filesystem کجا mount بشه. مثلاً ```/``` یا ```home/``` یا ```mnt/data/``` .
- فیلد سوم Filesystem Type : نوع filesystem رو مشخص می‌ کنه مثلاً : ext4 یا xfs یا vfat یا btrfs یا اگر entry مربوط به swap باشه.
- فیلد چهارم Options : این فیلد mount option ها رو مشخص می‌ کنه. مثلاً  defaults یا ترکیبی مثل defaults,noatime .
- فیلد پنجم dump : این فیلد برای ابزار قدیمی dump استفاده می‌ شده ، در بسیاری از سیستم‌ های امروزی مقدار اون 0 قرار داده میشه.
- فیلد ششم fsck order : این فیلد مشخص می‌کنه filesystem ها در چه ترتیبی توسط fsck بررسی بشن. به‌ صورت سنتی 1 برای root filesystem استفاده میشه . filesystem های دیگر معمولاً 2 می‌ گیرن و filesystem هایی که نباید توسط fsck بررسی بشن 0 می‌ گیرن . برای swap و filesystem های pseudo مثل proc/ معمولاً مقدار 0 استفاده میشه.

#### 🔹 Mount with fstab

اگر یک filesystem در etc/fstab/ تعریف شده باشه ، می‌ تونید mount point رو به‌ صورت ساده به mount بدید. مثلاً اگر entry مربوط به cdrom/ در fstab وجود داشته باشه ```mount /cdrom``` کافیه. برای mount کردن همه‌ی entry های مناسب در fstab می‌تونید از ```mount -a``` استفاده کنید. البته entry هایی که گزینه‌ ی ```noauto``` دارن توسط mount -a mount نمیشن.

#### 🔹 fstab Options

- اول defaults : مجموعه‌ ای از mount option های پیش‌ فرض رو فعال می‌ کنه.
- دوم noauto : باعث میشه entry هنگام ```mount -a``` به‌ صورت خودکار mount نشه. برای media های قابل‌ جا به‌ جایی می‌ تونه مفید باشه. 
- سوم user : اجازه میده کاربر معمولی mount مربوط به اون entry رو انجام بده، البته با محدودیت‌ های خودش. 
-چهارم errors : برای filesystem هایی مثل ext2/ext3/ext4 قابل استفاده‌ ست و رفتار filesystem در شرایط خطا رو مشخص می‌کنه. برای مثال ```errors=remount-ro``` یعنی در شرایط خاص خطای  filesystem ، سیستم تلاش کنه filesystem رو read-only کنه.

---

### fstab alternatives

تنها راه مدیریت mount ها  /etc/fstab نیست. در بعضی سیستم‌ ها ممکنه تنظیمات مرتبط در ```/etc/fstab.d/``` وجود داشته باشه یا سیستم از روش‌های دیگری برای مدیریت filesystem ها استفاده کنه. در سیستم‌ هایی که systemd دارن ، mount ها می‌ تونن به شکل systemd mount units هم مدیریت بشن. systemd در بسیاری از سیستم‌ ها می‌ تونه entry های /etc/fstab رو به unit های مورد نیاز تبدیل و مدیریت کنه. بنابراین در یک سیستم modern ممکنه چیزی که در fstab نوشته شده ، در نهایت به بخشی از dependency graph مربوط به systemd تبدیل بشه.

---

### Filesystem capacity

#### 🔹 df

برای دیدن ظرفیت و میزان استفاده‌ ی filesystem های mount شده از ابزار ```df``` استفاده میشه. مثلاً ```df -h``` خواندن خروجی رو با واحدهای قابل‌ فهم‌ تر مثل GB و MB راحت‌ تر می‌ کنه. در filesystem هایی مثل ext4 ممکنه بخشی از فضای filesystem به‌عنوان **reserved blocks** کنار گذاشته بشه. این فضای رزرو شده کمک می‌کنه وقتی filesystem تقریباً کاملاً پر شده، system service ها و administrator همچنان مقداری فضا در اختیار داشته باشن. بنابراین ممکنه ```Used + Available``` دقیقاً برابر با کل فضای نمایش‌ داده‌ شده نباشه. این موضوع مخصوصاً برای  filesystem های سیستمی اهمیت داره ، چون پر شدن کامل یک filesystem می‌تونه باعث مشکلات جدی برای سرویس‌ ها بشه.


<img width="100%" height="133" alt="image" src="https://github.com/user-attachments/assets/edaec75b-de8c-4ffc-bea1-72860e21ee17" />


#### 🔹 du

برای اینکه بفهمیم کدام فایل‌ها و directory ها بیشترین فضا رو مصرف می‌کنن ، از ```du``` استفاده می‌ کنیم. مثلاً ```*/du -s /var``` . گزینه‌ ی ```s-``` برای نمایش summary استفاده میشه. در نتیجه به‌ جای نمایش جزئیات تمام subdirectory ها ، جمع فضای مصرفی مسیر مورد نظر رو می‌ بینید.


<img width="100%" height="245" alt="Screenshot from 2026-09-06 12-00-24" src="https://github.com/user-attachments/assets/5b681e34-21b2-4b80-9767-4e43cd2b3b42" />

---

### Check and repair the file system

فایل سیستم های یونیکسی فقط مجموعه‌ای از block های ساده نیستن بلکه اونها metadata و ساختارهای پیچیده‌ ای دارن که باید با هم سازگار باشن. برای مثال filesystem باید بدونه : 

- کدام inode ها استفاده شدن.
- کدام block ها آزاد هستن.
- کدام directory entry به کدام inode اشاره می‌ کنه.
- در link count ها چه مقداری دارن.
- آیا metadata های filesystem با هم سازگار هستن. 

اگر سیستم ناگهانی خاموش بشه، ممکنه بخشی از تغییرات در RAM باقی مونده باشه و تمام تغییرات مورد نیاز روی storage ثبت نشده باشه. برای بررسی و در صورت امکان تعمیر filesystem از ```fsck``` استفاده میشه.  fsck در واقع یک frontend برای filesystem-specific checker هاست. مثلاً برای filesystem های خانواده‌ی ext از ```e2fsck``` ابزار اصلی بررسیه استفاده میشه.

**⚠️ هرگز fsck را روی filesystem در حال استفاده اجرا نکنید!** : این یکی از مهم‌ ترین هشدارهای این بخشه. اگر filesystem در حال mount و استفاده باشه و هم‌ زمان fsck ساختارهای اون رو تغییر بده ، ممکنه kernel و fsck هر دو در حال تغییر یا مشاهده‌ ی یک metadata مشترک باشن. این وضعیت می‌تونه باعث corruption بیشتر بشه. بنابراین filesystem معمولاً باید unmount شده باشه. برای root filesystem ، شرایط پیچیده‌ تره چون سیستم در حال اجرا خودش از / استفاده می‌کنه. در محیط‌ های recovery یا single-user میشه root filesystem رو در شرایط مناسب به شکل read-only در اختیار داشت و سپس عملیات بررسی انجام داد.

**مراحل بررسی و fsck** : در حالت تعاملی، filesystem checker می‌ تونه مرحله‌ به‌ مرحله ساختارهای مختلف filesystem رو بررسی کنه. اگر inconsistency پیدا بشه ، ممکنه از شما بپرسه که آیا می‌خواید مشکل اصلاح بشه یا نه. یکی از حالت‌ های معروف زمانی اتفاق میفته که filesystem یک inode پیدا می‌ کنه که در هیچ directory entry قابل دسترسی نیست. به چنین چیزی می‌تونیم **orphaned inode** یا inode بی‌ نام بگیم. در filesystem های ext ، در صورت امکان recovery فایل می‌ تونه در ``` lost+found``` قرار بگیره و اسم فایل معمولاً بر اساس inode number ساخته میشه.

**فایل سیستم های Journaled** : فایل سیستم هایی مثل ext3 و  ext4 از journaling استفاده می‌ کنن. Journal کمک می‌کنه filesystem بعد از crash یا shutdown ناگهانی راحت‌ تر به یک وضعیت سازگار برگرده. اما یک نکته‌ ی مهم : **journal به معنی تضمین مطلق سلامت همه‌ی داده‌های application نیست.** Journal در درجه‌ی اول برای حفظ consistency ساختارهای filesystem طراحی شده و رفتار دقیق مربوط به data و metadata به filesystem و mount mode وابسته‌ ست. در نتیجه نباید این جمله رو بگیم که «چون ext4 journal داره، پس هیچ‌ وقت fsck لازم نیست».

**دستور e2fsck -fy** : در مثال‌های مربوط به بررسی ext filesystem ممکنه دستور ```e2fsck -fy``` رو ببینید. گزینه‌ی ```f-``` برای force کردن check استفاده میشه و ```y-``` باعث میشه پاسخ yes به پرسش‌های repair به‌ صورت خودکار داده بشه. بنابراین این دستور بسیار قدرتمنده و باید با احتیاط استفاده بشه. این دستور صرفاً به معنی «flush کردن journal» نیست بلکه e2fsck یک filesystem checker/repair tool محسوب میشه.

**روش‌های نجات data** : در بدترین شرایط ، قبل از اینکه روی filesystem خراب عملیات repair انجام بدید ، ممکنه بخواید یک image کامل از device بگیرید. یکی از ابزارهایی که برای این کار استفاده میشه ```dd``` است. برای مثال:

```dd if=/dev/sdb of=disk.img```

البته در عمل برای storage های خراب ، dd ساده همیشه بهترین ابزار recovery نیست و ابزارهایی مثل ddrescue ممکنه برای media های دارای read error مناسب‌ تر باشن. همچنین ممکنه filesystem رو به‌ صورت read-only mount کنید تا داده‌ها رو تا حد ممکن بدون تغییر نجات بدید. برای بررسی سطح پایین filesystem های ext هم ابزار ```debugfs``` وجود داره. این ابزار می‌ تونه برای inspection و در شرایط خاص برای استخراج اطلاعات filesystem استفاده بشه.

---

### Special purpose filesystems

**فایل سیستم proc :** مربوط به ```proc/``` که اطلاعات process ها و بسیاری از اطلاعات kernel رو در اختیار user space قرار میده. برای مثال:

```cat /proc/cpuinfo```

اطلاعات CPU رو نمایش میده. همچنین directory هایی مثل ```<proc/<pid/``` اطلاعات مربوط به process های مختلف رو در اختیار میذارن. با این حال، طراحی kernel به‌ مرور به این سمت رفته که اطلاعاتی که ذاتاً درباره‌ی process نیستن، در موارد مناسب از طریق sysfs در ```sys/``` ارائه بشن.

**فایل سیستم sysfs :** مسیر اصلی اون در ```sys/``` هست و اطلاعات ساختار یافته‌ ای درباره‌ ی :

- devices
- drivers
- bus
- kernel objects
- Software topology 

در اختیار user space قرار میده.

**فایل سیستم tmpfs :** یک filesystem موقته که فضای اون عمدتاً از memory و در صورت نیاز از swap پشتیبانی میشه. یکی از محل‌ های رایج استفاده از آن ```run/``` هست. به همین دلیل داده‌ های داخل tmpfs معمولاً با reboot باقی نمی‌ مونن.

**فایل سیستم squashfs :** یک filesystem فشرده و معمولاً read-only هست. برای package یک filesystem کامل در قالبی فشرده کاربرد داره. یکی از نمونه‌ های شناخته‌ شده‌ ی استفاده از squashfs ، بعضی package ها و image های نرم‌ افزاری مثل snap هستن.

**فایل سیستم overlay :** فایل سیستم یا filesystem layer هایی مثل OverlayFS امکان ترکیب چند directory tree رو فراهم می‌کنن. در محیط‌های container ، این قابلیت بسیار مهمه چون می‌تونه یک لایه‌ ی read-only image رو با یک writable layer ترکیب کنه.

---

### swap space

همه‌ ی partition ها لزوماً filesystem ندارن. یک partition می‌تونه به عنوان **swap space** استفاده بشه. Swap بخشی از storage هستش که سیستم مدیریت حافظه‌ ی مجازی لینوکس می‌ تونه برای جا به‌ جایی بعضی page های حافظه از RAM به storage از اون استفاده کنه. به این عملیات **swapping** گفته میشه. وقتی یک page از memory به swap منتقل میشه ، فضای RAM آزاد میشه تا برای کاربرد های دیگر استفاده بشه. برای مشاهده‌ ی وضعیت حافظه و swap می‌تونید از ```free``` استفاده کنید. مثلاً ```free -h``` اطلاعات رو با واحد های قابل‌ فهم‌ تر نمایش میده.

<img width="100%" height="69" alt="image" src="https://github.com/user-attachments/assets/42618581-8713-4f62-9ca4-6f44aefff53c" />

---

### Using a Partition as Swap

برای استفاده از یک partition به‌عنوان swap ، چند مرحله داریم. اول باید مطمئن بشید partition واقعاً خالیه و داده‌ ی مهمی روی اون نیست بعد ```mkswap /dev/sdb1``` یک swap signature روی device ایجاد می‌ کنه بعد ```swapon /dev/sdb1``` اون swap area رو به pool فعال swap اضافه می‌ کنه. برای بررسی ```swapon --show``` یا ```free -h``` مفیده.
برای فعال شدن خودکار swap هنگام boot ، می‌تونید یک entry در ```etc/fstab/``` قرار بدید. مثلاً :

```/dev/sda5 none swap sw 0 0```

البته امروزه میشه از UUID مربوط به swap هم استفاده کرد. Swap signature خودش UUID داره و استفاده از identifier پایدار می‌تونه از وابستگی به device name جلوگیری کنه :

```UUID=... none swap sw 0 0```

---

### Using a file as Swap

اگر repartition کردن دیسک عملی نباشه ، می‌تونید به‌ جای partition یک **regular file** به‌ عنوان swap استفاده کنید. ایده اینه :

```
Regular File
     ↓
mkswap
     ↓
swapon
     ↓
Swap Space
```
برای ایجاد فایل میشه از dd استفاده کرد:

```dd if=/dev/zero of=swap_file bs=1024k count=num_mb```

بعد ```mkswap swap_file``` و ```swapon swap_file``` . فعال کردن swap file در سیستم‌ های مدرن نیازمند رعایت بعضی محدودیت‌ های filesystem و permission هم هست، بنابراین بهتره روش توصیه‌ شده‌ ی همان توزیع رو هم در نظر بگیرید. برای خارج کردن یک swap از pool فعال ```swapoff swap_file ``` یا برای partition از ```swapoff /dev/sdb1``` استفاده میشه. هنگام swapoff سیستم باید بتواند page های فعال موجود در آن swap area را در RAM یا swap های دیگر جا بدهد.

---

### How much swap do we need

یک قانون قدیمی در دنیای Unix می‌ گفت **swap باید حداقل دو برابر RAM باشه**. اما این قانون مربوط به دوره‌ ایه که مقدار RAM سیستم‌ ها بسیار کمتر از امروز بود. امروز نمیشه یک عدد ثابت برای تمام سیستم‌ها تعیین کرد. مثلاً سیستمی با  4GB RAM و سیستمی با 128GB RAM نیازهای یکسانی ندارن. حتی کاربرد سیستم هم مهمه ، روی یک سیستم desktop ممکنه swap برای مدیریت بهتر memory pressure و بعضی workload ها مفید باشه. روی یک سرور high-performance ممکنه administrator بخواد سیستم تا حد ممکن از disk I/O ناشی از swapping دور بمونه. 

یک نکته‌ ی مهم اینه که **استفاده‌ی زیاد و مداوم از swap معمولاً نشونه‌ی کمبود RAM یا memory pressure بالاست** و می‌تونه performance سیستم رو شدیداً کاهش بده. Swap جایگزین واقعی RAM نیست و اگر سیستم مرتباً مجبور باشه page های فعال رو بین RAM و storage جابه‌جا کنه ، latency به‌ شدت افزایش پیدا می‌کنه. کتاب هم توضیح میده که قانون «دو برابر RAM» مربوط به دوره‌ ای بوده که چندین کاربر روی یک ماشین کار می‌کردن و سیستم می‌ تونست memory مربوط به user های inactive رو به swap منتقل کنه.

**⚠️ سیستم بدون Swap** : بعضی administrator ها عمداً روی بعضی سیستم‌ ها swap رو فعال نمی‌کنن. برای مثال ممکنه روی یک server خاص که latency و performance اهمیت بسیار زیادی داره administrator بخواد از swap جلوگیری کنه. اما برای یک سیستم عمومی ، حذف کامل swap می‌تونه خطرناک باشه. اگر RAM و swap هر دو تمام بشن ، kernel ممکنه **OOM killer** رو فعال کنه تا با terminate کردن یک یا چند process مقداری memory آزاد کنه. این رفتار می‌ تونه باعث بسته شدن application ها یا service های مهم بشه. پس اینکه «سیستم swap نداره» لزوماً نشونه‌ ی performance بهتر نیست بلکه این یک تصمیم معماریه که باید با workload سیستم هماهنگ باشه.

---

### Introduction to Logical Volume Manager

تا اینجا با مدیریت مستقیم دیسک از طریق partition آشنا شدیم. در مدل های قدیمی مشکل اینجاست که partition ها layout نسبتاً ثابتی دارن  :
```
Disk
 ↓
Partition
 ↓
File
```

فرض کنید بعد از نصب سیستم متوجه بشید home/ فضای کافی نداره. اگر filesystem مستقیماً روی partition باشه ، تغییر layout ممکنه نیازمند عملیات پیچیده‌ای باشه مثل :

- backup
- Change partition
- Change filesystem
- Data migration
- reboot
- reinstall

همین موضوع وقتی بخواید یک disk جدید اضافه کنید هم خودش رو نشون میده. مثلاً اگر یک disk جدید به سیستم اضافه کنید ، بدون abstraction مناسب باید filesystem جدیدی روی اون بسازید و بعد یک mount point جدید برای اون انتخاب کنید. **LVM** برای حل بخش بزرگی از همین مشکلات طراحی شده. LVM یک لایه‌ ی اضافه بین physical block device ها و filesystem قرار میده. ساختار کلی :

```
Physical Volume
        ↓
Volume Group
        ↓
Logical Volume
        ↓
Filesystem
```

چند **(Physical Volume (PV** می‌ تونن در یک **(Volume Group (VG** قرار بگیرن. Volume Group مثل یک pool بزرگ از فضای storage عمل می‌ کنه. بعد از داخل اون pool می‌تونیم **LogicalVolume (LV)** ایجاد کنیم. 

<img width="100%" height="269" alt="image" src="https://github.com/user-attachments/assets/3872802f-4210-4fbb-a31e-70638d9d4033" />

کتاب هم دقیقاً LVM رو به‌ عنوان لایه‌ ای بین physical block device ها و filesystem معرفی می‌ کنه و توضیح میده که PV ها معمولاً block device هایی مثل partition هستن و VG یک data pool عمومی ایجاد می‌ کنه.

**رابطه‌ ی PV ، VG و LV** مثلاً:
```
/dev/sdb1 ─┐
           ├── Volume Group
/dev/sdc1 ─┘
                 │
          ┌──────┴──────┐
          ↓             ↓
        LV1           LV2
          ↓             ↓
       ext4           swap
```

خود Logical Volume یک block device محسوب میشه. یعنی می‌ تونید روی اون filesystem بسازید:

```mkfs.ext4 /dev/myvg/mylv```

یا اون رو به‌عنوان swap استفاده کنید. تفاوت مهم با partition اینه که شما معمولاً مجبور نیستید خودتون layout دقیق LV ها روی PV ها رو تعیین کنید. LVM این mapping رو مدیریت می‌ کنه.

**مزیت‌ های LVM** :

- اجازه میدهد PV جدید به VG اضافه کنید.
- ظرفیت VG رو افزایش بدید.
- اجازه میدهد PV رو از VG خارج کنید، اگر فضای کافی برای جابه‌جایی داده‌ها وجود داشته باشه.
- اجازه میدهد LV ها رو resize کنید.
- در بسیاری از موارد این تغییرات رو بدون reboot انجام بدید.
- در بسیاری از filesystem ها، عملیات resize رو بدون unmount انجام بدید.

کتاب هم روی همین flexibility تأکید می‌کنه و اضافه می‌کنه که در محیط‌های cloud حتی ممکنه اضافه کردن block storage جدید بدون خاموش کردن ماشین انجام بشه.

---

### Working with LVM

این LVM مجموعه‌ ای از ابزارهای user-space دارد و بسیاری از command های آشنای LVM در واقع frontend هایی برای مجموعه‌ ی ابزار LVM2 هستن. برای دیدن Volume Group ها از دستور ```vgs``` یا اطلاعات کامل‌ تر از دستور ```vgdisplay``` استفاده میشود.

**اطلاعات Volume Group** : اطلاعات مهم VG شامل مواردی میشه مثل :

- تعداد PV ها
- تعداد LV ها
- اندازه‌ ی کل
- فضای آزاد

یک مفهوم مهم در LVM هم **(Physical Extent (PE** هستش. Physical Extent یک واحد تخصیص در PV محسوب میشه. به‌جای اینکه LVM برای هر byte به‌ صورت جداگانه metadata داشته باشه ، فضای PV رو به extent های بزرگ‌ تری تقسیم می‌ کنه. مثلاً اندازه‌ ی رایج یک extent می‌ تونه حدود 4MiB باشه.

#### 🔹 Logical Volume (LV)

برای دیدن **Logical Volume** یا LV ها از ```lvs``` یا اطلاعات کامل‌ تر از ```lvdisplay``` استفاده میشه. Logical Volume ها در نهایت به device های Device Mapper متصل میشن. ممکنه device واقعی مربوط به یک LV چیزی مثل ```dev/dm-0/``` باشه. اما برای استفاده‌ ی راحت‌ تر، symbolic link هایی در مسیرهایی مثل ```dev/mapper/``` ایجاد میشن مثلاً :

```/dev/mapper/myvg-mylv```

ممکنه به device mapper مربوطه اشاره کنه. همچنین معمولاً مسیرهایی مثل ```dev/myvg/mylv/``` برای دسترسی به LV وجود دارن. این نام‌ ها نسبت به ```dev/dm-0/``` خوانا تر و برای استفاده‌ ی administrator مناسب‌ ترن.

#### 🔹 Physical Volume (PV)

برای دیدن **Physical Volume** یا PV ها از ```pvs``` و ```pvdisplay``` استفاده میشه. خود PV معمولاً بر اساس block device ای که روش قرار گرفته شناخته میشه، مثلاً ```dev/sdb1/``` اما PV یک UUID هم داره. LVM از metadata ذخیره‌ شده روی PV برای شناختن ساختار volume group استفاده می‌کنه.

---

### Creating Physical Volume and Volume Group

لازم نیست حتماً یک disk کامل رو partition کنید تا بتونید از اون به‌ عنوان PV استفاده کنید. در بعضی setup ها یک block device کامل مثل ```dev/sdb/``` هم می‌ تونه مستقیماً به‌ عنوان PV استفاده بشه. اما استفاده از partition می‌تونه مزایایی داشته باشه، مخصوصاً اگر بخواید partition table و boot layout مشخصی داشته باشید.

#### 🔹 Creating VG

فرض کنید ```dev/sdb1/``` قرار است عضو VG جدیدی بشه. می‌تونید از:

```vgcreate myvg /dev/sdb1```

استفاده کنید. در بعضی شرایط لازم نیست قبل از vgcreate حتماً pvcreate رو جداگانه اجرا کنید. اگر device شرایط مناسب داشته باشه ، LVM می‌ تونه PV metadata مورد نیاز رو در جریان ایجاد VG آماده کنه. اما در workflow های مدیریتی ممکنه اول صریحاً از:

```pvcreate /dev/sdb1```

استفاده کنید و بعد : 

```vgcreate myvg /dev/sdb1```

انجام بدید. برای اضافه کردن PV جدید به VG از دستور زیر استفاده میشه :

```vgextend myvg /dev/sdc1```

در نتیجه VG حالا ظرفیت بیشتری داره :

```
/dev/sdb1
      ↓
    PV
      ↓
   myvg
      ↑
    PV
      ↑
/dev/sdc1
```

---

### Creating Logical Volumes

مرحله‌ ی بعد ساخت LV هستش با ```lvcreate``` می‌تونید LV جدید ایجاد کنید مثلاً:

```lvcreate --size 10G --name mylv myvg```

یا می‌تونید اندازه رو بر اساس تعداد extent ها مشخص کنید برای مثال:

```lvcreate --extents 100 --name mylv myvg```

پیش‌ فرض mapping در مثال‌ های ساده معمولاً **linear** هست. یعنی LV از مجموعه‌ ای از extent های VG ساخته میشه بدون اینکه ویژگی‌ هایی مثل mirroring یا RAID به‌ صورت خودکار ایجاد شده باشه.بعد از ساخت LV می‌ تونید وضعیت VG رو بررسی کنید ```vgdisplay``` اگر تمام extent های VG رو به LV ها اختصاص نداده باشید مقداری ```Free PE``` باقی میمونه. این فضای آزاد بعداً می‌ تونه برای رشد LV ها استفاده بشه.

---

### Working with Logical Volumes

حالا که LV آماده‌ ست از دید filesystem تقریباً مثل یک block device معمولی رفتار می‌ کنه. می‌ تونید روی اون filesystem بسازید :

```mkfs.ext4 /dev/myvg/mylv```

بعد mount کنید:

```mkdir /mnt/data```

```mount /dev/myvg/mylv /mnt/data```

در اینجا مسیر```dev/myvg/mylv/``` یک LV هستش اما filesystem بالای اون دیگه لازم نیست بدونه که زیر خودش چند PV و چند physical disk وجود دارد. از دید filesystem مسیر ```dev/myvg/mylv/``` یک block device هستش.

---

### Delete Logical Volume

برای حذف LV از ```lvremove``` استفاده میشه. مثلاً ```lvremove myvg/mylv2``` دقت کنید syntax اینجا مهمه ، ```VG/LV``` یعنی ```myvg/mylv2``` نه ```myvg mylv2``` اگر syntax رو اشتباه وارد کنید، ممکنه command-line parser برداشت متفاوتی از argument ها داشته باشه. از طرف دیگه lvremove یک عملیات destructive هستش بنابراین نباید صرفاً چون prompt پرسید : ```Do you really want to remove``` به‌ صورت کورکورانه y بزنید. قبل از حذف همیشه ```lvs``` و ```lsblk``` بزنید و در صورت نیاز ```mount``` رو بررسی کنید تا مطمئن بشید LV درست رو انتخاب کردید. اگر LV در حال استفاده باشه ، معمولاً باید ابتدا مصرف‌ کننده‌ ها و filesystem مربوطه رو مدیریت کنید.

---

### 
