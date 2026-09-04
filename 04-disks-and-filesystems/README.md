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





