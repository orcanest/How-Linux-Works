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

