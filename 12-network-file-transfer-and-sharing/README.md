## اشتراک‌ گذاری فایل روی شبکه
### 🐧 فصل دوازدهم کتاب How Linux Works


در فصل‌های قبلی ، شبکه را از دید Linux بررسی کردیم. از interface و IP گرفته تا TCP ، UDP ، socket و سرویس‌ های شبکه . حالا یک سؤال کاملاً عملی مطرح می‌ شود **وقتی دو یا چند ماشین روی شبکه داریم ، فایل‌ ها را چطور بین آن‌ ها منتقل یا به‌ صورت دائمی share کنیم ؟**

برای این مسئله راه‌ حل واحدی وجود ندارد. گاهی فقط می‌ خواهیم یک فایل را خیلی سریع به ماشین دیگری بدهیم. گاهی لازم است حجم زیادی فایل را مرتباً بین چند سیستم synchronize کنیم. در بعضی محیط‌ ها Linux باید با Windows کار کند ، در بعضی محیط‌ ها لازم است یک filesystem از NAS روی Linux mount شود و در بعضی سناریوها هم storage ابری را می‌ خواهیم مثل یک filesystem محلی در اختیار برنامه‌ ها قرار دهیم.

فصل دوازدهم دقیقاً این تفاوت‌ ها را بررسی می‌ کند و برای سناریوهای مختلف سراغ روش‌ هایی مانند Quick Copy ، rsync ، Samba/CIFS ، SSHFS ، NFS و FUSE-based filesystems می‌ رود. کتاب همچنین عمداً وارد راه‌ حل‌ های بسیار بزرگ و پیچیده‌ ی file sharing در مقیاس سازمان‌ های عظیم نمی‌ شود ، چون چنین محیط‌ هایی به طراحی و زیرساخت بسیار بیشتری نیاز دارند.

---
📚 Table of Contents

- [12.1 Quick Copy](#121-quick-copy)
- [12.2 rsync](#122-rsync)
  - [12.2.1 Getting Started with rsync](#1221-getting-started-with-rsync)
  - [12.2.2 Making Exact Copies of a Directory Structure](#1222-making-exact-copies-of-a-directory-structure)
  - [12.2.4 Excluding Files and Directories](#1224-excluding-files-and-directories)
  - [12.2.5 Checking Transfers, Adding Safeguards, and Using Verbose Mode](#1225-checking-transfers-adding-safeguards-and-using-verbose-mode)
  - [12.2.6 Compressing Data](#1226-compressing-data)
  - [12.2.7 Limiting Bandwidth](#1227-limiting-bandwidth)
  - [12.2.8 Transferring Files to Your Computer](#1228-transferring-files-to-your-computer)
  - [12.2.9 Further rsync Topics](#1229-further-rsync-topics)
- [12.3 Introduction to File Sharing](#123-introduction-to-file-sharing)
  - [12.3.1 File Sharing Usage and Performance](#1231-file-sharing-usage-and-performance)
  - [12.3.2 File Sharing Security](#1232-file-sharing-security)
- [12.4 Sharing Files with Samba](#124-sharing-files-with-samba)
  - [12.4.1 Server Configuration](#1241-server-configuration)
  - [12.4.2 Server Access Control](#1242-server-access-control)
  - [12.4.3 Passwords](#1243-passwords)
  - [12.4.4 Manual Server Startup](#1244-manual-server-startup)
  - [12.4.5 Diagnostics and Logfiles](#1245-diagnostics-and-logfiles)
  - [12.4.6 File Share Configuration](#1246-file-share-configuration)
  - [12.4.7 Home Directories](#1247-home-directories)
  - [12.4.8 Printer Sharing](#1248-printer-sharing)
  - [12.4.9 The Samba Client](#1249-the-samba-client)
- [12.5 SSHFS](#125-sshfs)
- [12.6 NFS](#126-nfs)
  - [NFS vs SSHFS](#nfs-vs-sshfs)
  - [Mounting NFS](#mounting-nfs)
  - [NFS and Transport](#nfs-and-transport)
  - [NFS Security](#nfs-security)
  - [Automounter](#automounter)
  - [NFS Servers](#nfs-servers)
- [12.7 Cloud Storage](#127-cloud-storage)
  - [Cloud Storage and Filesystem](#cloud-storage-and-filesystem)
  - [FUSE for Cloud](#fuse-for-cloud)
- [12.8 The State of Network File Sharing](#128-the-state-of-network-file-sharing)
  - [NFS Security Limitations](#nfs-security-limitations)
  - [Andrew File System — AFS](#andrew-file-system--afs)
  - [Kernel-based Network Filesystems](#kernel-based-network-filesystems)
  - [Moving to User Space](#moving-to-user-space)
- [Choosing the Right Tool for File Transfer and File Sharing](#choosing-the-right-tool-for-file-transfer-and-file-sharing)
- [Practical Tips](#practical-tips)
- [Summary](#summary)

---

### 12.1 Quick Copy

گاهی اصلاً به یک file-sharing system کامل نیاز نداریم. فرض کنید روی یک Linux machine چند فایل دارید و می‌ خواهید فقط برای چند دقیقه یا چند ساعت آن‌ ها را در اختیار یک ماشین دیگر روی شبکه‌ ی محلی قرار دهید. در چنین شرایطی می‌ توانید از یک web server بسیار ساده استفاده کنید. کتاب برای Python قدیمی نمونه‌ ای با SimpleHTTPServer ارائه می‌کند ، اما در Python 3 نام این module به http.server تغییر کرده است در Python 3 :

```
python3 -m http.server
```

این دستور directory فعلی را به‌ صورت موقت از طریق HTTP ارائه می‌ کند و به‌ طور پیش‌ فرض روی port 8000 هم listen می‌ دهد. مثلاً اگر IP ماشین شما 10.1.2.4 باشد ، ماشین دیگر می‌ تواند به http://10.1.2.4:8000 متصل شود. اگر directory شامل فایل‌ هایی مثل file1.txt و file2.zip و image.iso باشد ، مرورگر می‌ تواند آن‌ ها را نمایش دهد و در صورت مناسب بودن محتوا امکان دریافت شان وجود دارد. این روش بسیار ساده است و تقریباً هیچ configuration خاصی لازم ندارد اما باید محدودیت امنیتی آن را کاملاً جدی گرفت. این یک file-sharing service امن و کامل نیست اگر روی network نا مطمئن اجرا شود ، هر کسی که به این service دسترسی داشته باشد ممکن است فایل‌ های آن directory را ببیند. بنابراین چنین روشی برای انتقال موقت و شبکه‌ی شخصی و آزمایش و محیط lab و کپی سریع چند فایل مناسب است و برای اشتراک‌ گذاری دائمی و شبکه‌ ی عمومی و داده‌ های حساس و محیطی که کاربران ناشناس به آن دسترسی دارند مناسب نیست. همچنین باید دقت کرد که server از همان directory که command را در آن اجرا کرده‌ اید فایل ارائه می‌ کند. بنابراین اجرای آن در directory اشتباه می‌ تواند ناخواسته فایل‌ های بیشتری را در معرض دید قرار دهد. خود کتاب نیز هشدار می‌ دهد که این روش را فقط در شبکه‌ ای که به آن اعتماد دارید استفاده کنید.

---

### 12.2 rsync

وقتی فقط یک یا دو فایل ندارید و تعداد فایل‌ ها زیاد می‌ شود ، روش‌ هایی مثل scp -r هنوز کار می‌کنند ، اما برای synchronization مداوم چندان مناسب نیستند. فرض کنید یک directory در دو ماشین دارید :

```
Machine A
└── project/
    ├── a
    ├── b
    └── c

Machine B
└── project/
    ├── a
    ├── b
    └── c
```

بعد فقط فایل b تغییر می‌ کند اگر هر بار کل directory را دوباره copy کنید ، مقدار زیادی داده‌ ی غیرضروری جا به‌ جا می‌ شود. اینجاست که rsync اهمیت پیدا می‌ کند. rsync یک file synchronization utility است که می‌ تواند وضعیت source و destination را با هم مقایسه کند و فقط بخش‌ هایی را که لازم است منتقل کند. کتاب آن را ابزار استانداردی برای synchronization روی Linux معرفی می‌ کند و توضیح می‌ دهد که برای transfer های مکرر و قابل‌ اعتماد بسیار مناسب است.

#### 12.2.1 Getting Started with rsync

برای استفاده از rsync میان دو host ، برنامه‌ ی rsync باید در دو طرف در دسترس باشد و باید روشی برای دسترسی از یک ماشین به دیگری وجود داشته باشد. رایج‌ ترین روش ، استفاده از SSH است از نظر ظاهری command آن شبیه scp است :

```
rsync file1 file2 user@host:
```

اگر username local و remote متفاوت باشد :

```
rsync file1 file2 user@host:
```

در سیستم‌ های امروزی ، وقتی rsync با remote shell استفاده می‌ شود ، SSH معمولاً transport مورد استفاده است. نکته‌ ی مهم این است که rsync فقط برای شبکه نیست. می‌ توان از آن برای copy بین دو location روی همان machine نیز استفاده کرد :

```
filesystem A
    ↓
rsync
    ↓
filesystem B
```

مثلاً برای synchronization بین دو filesystem مختلف. اگر rsync روی remote نصب نباشد ممکن است چیزی شبیه این دریافت کنید :

```
rsync: connection unexpectedly closed
rsync error: error in rsync protocol data stream (code 12)
```

این وضعیت می‌ تواند به این معنی باشد که در سمت remote برنامه‌ی rsync پیدا نشده است. حتی اگر package نصب باشد ، ممکن است executable آن در PATH کاربری که SSH connection با آن برقرار شده قرار نداشته باشد. در چنین شرایطی می‌توان مسیر برنامه‌ ی remote را با ```--rsync-path``` مشخص کرد البته مسیر واقعی بسته به سیستم متفاوت است مثلاً :

```
rsync --rsync-path=/usr/bin/rsync ...
```

البته مسیر واقعی بسته به سیستم متفاوت است.

**انتقال فایل‌ های معمولی**

بدون option خاص ، rsync فایل‌ ها را منتقل می‌ کند مثلاً ```rsync file1 host:``` فایل را به home directory کاربر remote می‌ فرستد. برای destination مشخص از ```rsync file1 host:/tmp``` استفاده می‌ کنیم.

**انتقال directory**

اگر بدون گزینه‌ ی recursive یک directory بدهید ، rsync آن را به همان شکل مورد انتظار یک directory tree منتقل نمی‌کند و ممکن است با پیامی مانند ```skipping directory dir``` روبرو شوید. برای انتقال یک directory کامل معمولاً از ```rsync -a dir host:dest_dir``` استفاده می‌ شود. گزینه‌ ی a- یا archive mode مجموعه‌ ای از رفتارهای مفید را فعال می‌ کند از جمله recursion و نگهداری بسیاری از metadata های فایل که در این حالت rsync می‌تواند مواردی را حفظ کند مانند :

- subdirectories
- symbolic links
- permissions
- file modes
- timestamps
- device information

البته حفظ owner و group و برخی metadata ها به این نیز وابسته است که user اجراکننده چه مجوزهایی داشته باشد.

**Dry Run**

وقتی مطمئن نیستید نتیجه‌ ی یک transfer دقیقاً چه خواهد بود ، بهتر است ابتدا از ```rsync -n``` استفاده کنید. n- به معنی dry run است. یعنی rsync بررسی می‌ کند چه تغییراتی باید انجام شود ، اما واقعاً فایل‌ ها را منتقل نمی‌ کند. برای اینکه output هم نمایش داده شود ```rsync -nva dir host:dest_dir``` اینجا :

- -n → dry run
- -v → verbose
- -a → archive mode

است این ترکیب قبل از عملیات حساس ، مخصوصاً همراه delete-- ، بسیار مهم است.

#### 12.2.2 Making Exact Copies of a Directory Structure

به‌ صورت پیش‌ فرض ، rsync destination را به یک mirror کامل از source تبدیل نمی‌ کند فرض کنید source :

```
dir/
├── a
└── b
```

باشد و destination علاوه بر a و b یک فایل قدیمی هم داشته باشد :

```
dir/
├── a
├── b
└── c
```

اگر فقط rsync معمولی اجرا شود ، فایل اضافی c حذف نمی‌ شود بعد از synchronization ممکن است داشته باشید a و b و c اگر هدف شما یک mirror دقیق باشد ، این رفتار کافی نیست در این حالت از ```rsync -a --delete dir host:dest_dir``` استفاده می‌ شود. --delete فایل‌ هایی را که در source وجود ندارند ، از destination حذف می‌ کند. این گزینه بسیار قدرتمند و در عین حال خطرناک است. مثلاً اگر source را اشتباه مشخص کنید ، ممکن است فایل‌ هایی که واقعاً نمی‌ خواستید حذف شوند ، در destination پاک شوند. بنابراین workflow مناسب این است ```rsync -an --delete dir host:dest_dir``` ابتدا بررسی شود و فقط پس از تأیید نتیجه ```rsync -a --delete dir host:dest_dir``` واقعاً اجرا شود. -n در اینجا نقش یک لایه‌ ی محافظ بسیار مهم دارد. کتاب نیز صراحتاً توصیه می‌ کند پیش از استفاده از --delete ، مقصد را بررسی کنید و در صورت تردید از dry run استفاده کنید.

این تفاوت زمانی که --delete نیز استفاده می‌ شود بسیار مهم‌ تر می‌ شود مثلاً اشتباه کردن بین ```rsync -a --delete dir host:dest_dir``` و ```rsync -a --delete dir/ host:dest_dir``` می‌ تواند ساختار مقصد را به‌ صورت غیرمنتظره تغییر دهد و حتی باعث حذف فایل‌ های نا مرتبط شود.

**یک نکته‌ ی مهم درباره‌ ی Tab Completion**

برخی shell ها وقتی directory را با Tab کامل می‌ کنند ، slash انتهایی را نیز اضافه می‌ کنند یعنی ممکن است چیزی که ابتدا dir بوده ، بعد از completion تبدیل شود به dir/ ، برای command های عادی ممکن است این تفاوت چندان مهم نباشد اما در rsync و به‌خصوص --delete می‌ تواند مهم و حتی خطرناک باشد.

- Figure 12-1
- Figure 12-2

تفاوت رفتار dir و dir/ و خطر ترکیب این موضوع با --delete در متن کتاب به‌ صورت مشخص مطرح شده است.

#### 12.2.4 Excluding Files and Directories

یکی از قابلیت‌های مهم rsync این است که می‌توان بخشی از source را از transfer کنار گذاشت.
مثلاً فرض کنید:

```
src/
├── app/
├── data/
└── .git/
```

داریم و نمی‌خواهیم .git منتقل شود.
می‌توان نوشت:

```
rsync -a --exclude=.git src host:
```

نکته‌ی مهم این است که --exclude با pattern کار می‌کند، نه صرفاً یک path مطلق.
بنابراین:

```
--exclude=.git
```

می‌تواند مواردی با این نام را در مسیر transfer حذف کند.
اگر بخواهید یک مورد مشخص در ساختار transfer را هدف بگیرید، می‌توان pattern دقیق‌تری نوشت:

```
--exclude=/src/.git
```

در اینجا / اول به root filesystem اشاره نمی‌کند؛ بلکه نسبت به base directory مربوط به transfer تفسیر می‌شود.

**چند exclude**

لازم نیست فقط یک exclude داشته باشید.
می‌توانید چند مورد را پشت سر هم قرار دهید:

```
rsync -a \
    --exclude=.git \
    --exclude=*.tmp \
    --exclude=cache/ \
    src/ host:dest/
```

**exclude-from**

اگر الگوهای زیادی دارید، بهتر است آن‌ها را در یک فایل قرار دهید:

```
.git
*.tmp
cache/
node_modules/
```

و سپس:

```
rsync -a --exclude-from=exclude.txt src/ host:dest/
```

این روش برای backupها و synchronizationهای دائمی بسیار مناسب‌تر است.

**include**

اگر exclude pattern بیش از حد گسترده باشد، می‌توان در موارد مناسب از --include برای وارد کردن موارد خاص استفاده کرد.
درک ترتیب و pattern matching include/exclude در rsync برای ساخت ruleهای دقیق اهمیت زیادی دارد.

#### 12.2.5 Checking Transfers, Adding Safeguards, and Using Verbose Mode

یکی از دلایل سرعت rsync این است که قرار نیست همیشه محتوای تمام فایل‌ها را byte-by-byte مقایسه کند.
در حالت معمول، rsync می‌تواند از ترکیبی از:

- file size
- last modification time

برای تشخیص اینکه فایل ظاهراً تغییر کرده یا نه استفاده کند.
اگر source و destination اندازه و timestamp سازگار داشته باشند، فایل ممکن است نیازی به انتقال مجدد نداشته باشد.
این روش بسیار سریع است، اما گاهی verification بیشتری لازم است.

**--checksum**

با:

```
rsync -c
```

یا:

```
rsync --checksum
```

محتوای فایل با checksum مقایسه می‌شود.
این کار نیازمند خواندن داده و مصرف CPU است، بنابراین نسبت به بررسی ساده‌ی metadata پرهزینه‌تر است.
اما در مواردی که:

- محتوای فایل مهم است
- اندازه‌ی فایل‌ها اغلب برابر است
- اطمینان بیشتر از metadata لازم است

می‌تواند ارزش داشته باشد.

**--ignore-existing**

با:

```
rsync --ignore-existing
```

فایل‌هایی که از قبل در destination وجود دارند overwrite نمی‌شوند.
این رفتار برای بعضی سناریوهای import یا archive مفید است.

**--backup**

با:

```
rsync -b
```

یا:

```
rsync --backup
```

وقتی قرار است یک فایل موجود overwrite شود، rsync قبل از آن نسخه‌ی قبلی را با suffix مناسب نگه می‌دارد.
Suffix پیش‌فرض معمولاً:

```
~
```

است.
می‌توان آن را تغییر داد:

```
rsync --backup --suffix=.old ...
```

**--update**

با:

```
rsync -u
```

اگر نسخه‌ی destination جدیدتر از source باشد، rsync آن فایل را overwrite نمی‌کند.
این گزینه در سناریوهایی که timestamp مقصد باید محافظت شود، کاربرد دارد.

**Verbose**

به‌صورت پیش‌فرض rsync نسبتاً ساکت است و بیشتر زمانی output می‌دهد که مشکلی وجود داشته باشد.
برای مشاهده‌ی فایل‌ها:

```
rsync -v ...
```

و برای جزئیات بیشتر:

```
rsync -vv ...
```

استفاده می‌شود.
برای دریافت summary نیز:

```
rsync --stats ...
```

مفید است.
Workflow خوبی برای یک انتقال حساس می‌تواند این باشد:

```
rsync -anv source/ host:destination/
```

ابتدا dry run.
بعد:

```
rsync -av source/ host:destination/
```

اجرای واقعی.
و اگر mirror دقیق می‌خواهید:

```
rsync -anv --delete source/ host:destination/
```

قبل از:

```
rsync -av --delete source/ host:destination/
```

#### 12.2.6 Compressing Data

اگر اتصال بین دو host کند باشد، انتقال داده‌ی خام ممکن است زمان زیادی ببرد.
rsync گزینه‌ی:

```
-z
```

را برای compression در حین انتقال ارائه می‌دهد.
مثلاً:

```
rsync -az dir host:dest_dir
```

در این حالت داده‌ی انتقالی فشرده می‌شود و در سمت دیگر دوباره unpack/decompress می‌شود.
این قابلیت مخصوصاً روی:

- slow links
- internet connections
- high-latency links

می‌تواند مفید باشد.
اما compression همیشه سرعت را بیشتر نمی‌کند.
روی یک LAN بسیار سریع، ممکن است CPUهای دو طرف بیشتر از چیزی که شبکه لازم دارد درگیر compression و decompression شوند.
در چنین حالتی:

```
fast network
+
slow compression
=
possibly slower transfer
```

بنابراین -z باید بر اساس workload و سرعت link انتخاب شود، نه اینکه همیشه روشن باشد.

#### 12.2.7 Limiting Bandwidth

گاهی مشکل انتقال فایل این نیست که خیلی کند است؛ مشکل این است که بیش از حد سریع است.
مثلاً اگر از یک اینترنت با uplink محدود استفاده کنید و یک synchronization بزرگ شروع کنید، rsync می‌تواند بخش زیادی از ظرفیت uplink را مصرف کند.
در نتیجه traffic دیگری مثل:

- HTTP requests
- SSH sessions
- DNS traffic
- other uploads

نیز تحت تأثیر قرار می‌گیرد.
برای این وضعیت:

```
--bwlimit
```

وجود دارد.
مثلاً:

```
rsync --bwlimit=100000 -a dir host:dest_dir
```

با این کار می‌توانید سرعت انتقال را محدود کنید تا شبکه برای کارهای دیگر نیز ظرفیت داشته باشد.
این قابلیت در backupهای دوره‌ای و synchronizationهای بزرگ بسیار مفید است.

#### 12.2.8 Transferring Files to Your Computer

rsync فقط برای:

```
local → remote
```

نیست.
می‌توانید مسیر را برعکس کنید:

```
remote → local
```

مثلاً:

```
rsync -a host:src_dir dest_dir
```

در اینجا source روی host remote قرار دارد و destination روی machine فعلی است.
اگر username متفاوت باشد:

```
rsync -a user@host:src_dir dest_dir
```

این ویژگی باعث می‌شود rsync هم برای ارسال و هم برای دریافت مناسب باشد.

#### 12.2.9 Further rsync Topics

وقتی تعداد فایل‌ها زیاد می‌شود، rsync یکی از اولین ابزارهایی است که باید به ذهن برسد.
یکی از کاربردهای مهم آن، انتقال batchای مجموعه‌ی بزرگی از فایل‌ها به چند host مختلف است.
از آنجا که rsync synchronization انجام می‌دهد، در transferهای تکراری معمولاً لازم نیست تمام داده از ابتدا منتقل شود.
این موضوع برای:

- backup
- mirror
- distribution
- deployment
- archive synchronization

بسیار ارزشمند است.

**Backup با rsync**

می‌توان یک filesystem را با یک storage دیگر همگام کرد.
مثلاً به‌صورت مفهومی:

```
Local Filesystem
       ↓
     rsync
       ↓
Remote Storage
```

و در synchronizationهای برنامه‌ریزی‌شده، --delete می‌تواند destination را به mirror دقیق تبدیل کند.
اما یک هشدار مهم:

> Mirror با Backup کامل یکسان نیست.

اگر فایلی به‌صورت اشتباه حذف شود و synchronization با --delete انجام شود، حذف می‌تواند به backup mirror نیز منتقل شود.
به همین دلیل برای backupهای واقعی معمولاً باید retention، versioning یا snapshot نیز در طراحی در نظر گرفته شود.
کتاب همچنین اشاره می‌کند که storage ابری مانند S3 می‌تواند در کنار ابزارهای دیگر و rsync برای ساختن یک backup system استفاده شود.

---

### 12.3 Introduction to File Sharing

تا اینجا بیشتر درباره‌ی copy و synchronization صحبت کردیم.
حالا مسئله عوض می‌شود.
فرض کنید نمی‌خواهیم فایل را هر بار کپی کنیم.
می‌خواهیم یک directory روی ماشین A داشته باشیم و ماشین B بتواند آن را طوری استفاده کند که انگار بخشی از filesystem خودش است.
این همان network file sharing است.
در این حالت:

```
Machine A
└── /data

Machine B
└── /mnt/data
```

و /mnt/data روی B نماینده‌ی یک directory remote روی A می‌شود.
این مدل با copy فرق دارد.
در copy:

```
file
   ↓
duplicate
```

در file sharing:

```
remote storage
   ↓
mounted/accessed remotely
```

است.
کتاب در اینجا تأکید می‌کند که قبل از انتخاب روش file sharing باید اول مشخص کنید چرا اصلاً می‌خواهید file sharing داشته باشید. همین تصمیم روی performance، architecture و حتی امنیت اثر می‌گذارد.

#### 12.3.1 File Sharing Usage and Performance

در شبکه‌های Unix سنتی، دو دلیل مهم برای sharing وجود داشت:

- Convenience
- Lack of local storage

مثلاً چند workstation می‌توانستند به storage متمرکز یک server دسترسی داشته باشند.
به جای اینکه برای هر machine مقدار زیادی storage محلی تهیه کنیم:

```
Workstation A ─┐
Workstation B ─┼── Central Storage
Workstation C ─┘
```

همه از storage متمرکز استفاده می‌کردند.
این مدل از نظر مدیریت storage مزایایی دارد، اما یک هزینه‌ی مهم دارد:
storage شبکه معمولاً latency بیشتری نسبت به storage محلی دارد.

چرا latency مهم است؟
فرض کنید برنامه بخواهد یک فایل بزرگ را از ابتدا تا انتها بخواند.
این نوع دسترسی:

```
sequential access
```

برای network storage نسبتاً مناسب است.
چون server می‌تواند داده را با ترتیب قابل‌پیش‌بینی بخواند و buffer کند.
مثلاً streaming ویدئو یا audio معمولاً الگوی دسترسی نسبتاً قابل‌پیش‌بینی دارد.
اما حالتی مثل:

- باز کردن تعداد زیادی فایل کوچک
- جستجوی مکرر در directory
- ویرایش پروژه‌ی بزرگ
- خواندن و نوشتن تصادفی

مشکل‌سازتر است.
در این حالت هر access ممکن است شامل مراحل متعددی باشد:

```
Client request
   ↓
Network
   ↓
Server receives request
   ↓
Server processes request
   ↓
Filesystem lookup
   ↓
Storage access
   ↓
Response
   ↓
Network
   ↓
Client
```

حتی اگر حجم داده کم باشد، latency می‌تواند زیاد شود.
به همین دلیل ممکن است CPU سمت client زمان زیادی را منتظر network I/O بماند.
کتاب به‌خصوص هشدار می‌دهد که برای workloadهایی مثل توسعه‌ی نرم‌افزار یا ویرایش فایل‌های بزرگ، storage محلی معمولاً انتخاب بسیار بهتری است. در مقابل، workloadهایی با داده‌ی حجیم و دسترسی ترتیبی ممکن است با network storage عملکرد قابل‌قبولی داشته باشند.

پس قبل از اینکه بگویید:

> «می‌خواهم home directory را روی شبکه قرار بدهم.»

بهتر است بپرسید:

- چه نوع access patternی دارم؟
- چند فایل؟
- چقدر random access؟
- چقدر latency اهمیت دارد؟
- اگر network قطع شود چه اتفاقی می‌افتد؟

#### 12.3.2 File Sharing Security

امنیت همیشه بخشی از طراحی اولیه‌ی network file sharing در سیستم‌های قدیمی نبوده است.
به همین دلیل باید سه موضوع را جداگانه بررسی کنیم:

- Authentication
- Authorization
- Encryption

**Authentication**
چه کسی هستی؟

**Authorization**
اجازه داری چه کاری انجام بدهی؟

**Encryption**
آیا داده در مسیر بین client و server قابل مشاهده است یا خیر؟

این سه مفهوم یکی نیستند.
ممکن است server کاربر را به‌درستی authenticate کند، ولی داده در مسیر plaintext منتقل شود.
یا ممکن است ارتباط encrypted باشد ولی user بیش از چیزی که باید دسترسی داشته باشد.

**اگر network قابل اعتماد نیست**

اگر مطمئن نیستید network بین دو machine امن است، باید لایه‌ی امنیتی مناسب اضافه کنید.
کتاب به ابزارهایی مانند:

- stunnel
- IPSec
- VPN

اشاره می‌کند که می‌توانند در لایه‌های پایین‌تر به محافظت از traffic کمک کنند.
این نکته مخصوصاً درباره‌ی NFS و بعضی configurationهای سنتی مهم است.

---

### 12.4 Sharing Files with Samba

اگر در شبکه Windows machine دارید، یکی از مهم‌ترین روش‌های file sharing، SMB است.
SMB مخفف:

```
Server Message Block
```

است.
Samba مجموعه‌ی نرم‌افزارهای Unix/Linux برای پیاده‌سازی SMB و تعامل با شبکه‌های Windows است.
Samba فقط برای این نیست که Windows به Linux دسترسی داشته باشد؛ Linux می‌تواند با Samba به Windows server نیز متصل شود.
به‌صورت مفهومی:

```
Windows
    ↕
   SMB
    ↕
Samba
    ↕
Linux
```

macOS نیز از SMB پشتیبانی می‌کند.
کتاب هدف خود را در این بخش محدود می‌کند و وارد تمام پیچیدگی‌های یک Samba deployment بزرگ نمی‌شود؛ تمرکز بیشتر روی share کردن فایل و printer در محیط‌های نسبتاً ساده است.
برای راه‌اندازی کلی Samba، چهار کار اصلی مطرح می‌شود:

1. ایجاد/configure کردن smb.conf
2. تعریف file shareها
3. در صورت نیاز تعریف printer shareها
4. اجرای daemonهای Samba

در packageهای توزیع معمولاً بخش زیادی از setup پایه توسط package manager و service manager انجام می‌شود.

#### 12.4.1 Server Configuration

فایل اصلی configuration Samba:

```
smb.conf
```

است.
در بسیاری از distributionها این فایل در:

```
/etc/samba/smb.conf
```

قرار دارد.
در installationهای custom ممکن است location متفاوت باشد.
ساختار smb.conf بر اساس sectionهاست.
مثلاً:

```
[global]
    ...
```

برای تنظیمات عمومی server استفاده می‌شود.
سپس shareها نیز section جداگانه دارند.

**[global]**

بخش [global] روی رفتار کلی Samba اثر می‌گذارد.
مثلاً می‌توان مشخص کرد:

```
[global]
    netbios name = LINUXSERVER
    server string = My Linux Samba Server
    workgroup = MYNETWORK
```

**netbios name**
نامی که Samba برای server استفاده می‌کند.
اگر تنظیم نشود، Samba معمولاً از hostname سیستم استفاده می‌کند.

**server string**
یک description کوتاه برای server است.

**workgroup**
نام workgroup یا در بعضی سناریوها domain مربوط به محیط Windows را مشخص می‌کند.
NetBIOS یک API و naming mechanism قدیمی مرتبط با شبکه‌های Windows است و در بسیاری از configurationهای SMB سنتی دیده می‌شود.

**تست کردن configuration Samba**

قبل از اینکه server را در اختیار clientها قرار دهید، یکی از ابزارهای بسیار مهم:

```
testparm
```

است.
این command configuration را parse و بررسی می‌کند و اگر syntax یا configuration مشکل‌داری وجود داشته باشد، به شما هشدار می‌دهد.
مثلاً:

```
testparm
```

می‌تواند sectionهای موجود را نشان دهد و مشخص کند configuration قابل‌خواندن است.
در troubleshooting Samba بهتر است بعد از تغییر smb.conf ابتدا:

```
testparm
```

را اجرا کنید و سپس سراغ service بروید.

#### 12.4.2 Server Access Control

Samba گزینه‌های مختلفی برای کنترل access در اختیار دارد.
یکی از این گزینه‌ها:

```
interfaces =
```

است.
می‌توانید مشخص کنید Samba روی کدام network interface یا networkها در دسترس باشد.
مثلاً:

```
interfaces = enp0s31f6
```

یا با network range مناسب.
برای اینکه فقط interfaceهای مشخص‌شده واقعاً مورد استفاده قرار گیرند، می‌توان از:

```
bind interfaces only = yes
```

استفاده کرد.

**valid users**
با:

```
valid users = user1, user2
```

مشخص می‌کنید چه userهایی اجازه‌ی دسترسی داشته باشند.

**guest ok**
با:

```
guest ok = yes
```

می‌توان guest access ایجاد کرد.
اما این کار باید با احتیاط انجام شود، مخصوصاً اگر share قابل نوشتن باشد.
در network خصوصی و کنترل‌شده ممکن است کاربرد داشته باشد، ولی روی شبکه‌ی نامطمئن می‌تواند access غیرضروری ایجاد کند.

**browseable**
این گزینه تعیین می‌کند share در browserهای شبکه قابل مشاهده باشد یا نه.
مثلاً:

```
browseable = no
```

یعنی share لزوماً در فهرست browse شده دیده نمی‌شود، اما اگر نام دقیق آن را بدانید ممکن است هنوز بتوانید به آن متصل شوید.
این گزینه access control واقعی نیست؛ بیشتر روی discovery و visibility اثر دارد.

#### 12.4.3 Passwords

Samba یک موضوع مهم دارد:
سیستم password محلی Unix با مدل authentication شبکه‌ی Windows/SMB دقیقاً یکی نیست.
بنابراین در configurationهای سنتی، Samba می‌تواند password database مخصوص خودش داشته باشد.
یکی از backendهای کلاسیک:

```
tdbsam
```

است.
در configuration سنتی می‌توان چیزی شبیه این داشت:

```
[global]
    security = user
    passdb backend = tdbsam
    obey pam restrictions = yes
```

محل database نیز می‌تواند در configuration مشخص شود.
مثلاً:

```
passdb backend = tdbsam:/path/to/database
```

این بخش برای محیط‌های کوچک مناسب است، در حالی که integration با domain infrastructure می‌تواند طراحی متفاوتی داشته باشد.

**smbpasswd**

برای مدیریت userهای Samba از:

```
smbpasswd
```

استفاده می‌شود.
اما یک نکته‌ی مهم وجود دارد:
user مورد نظر باید در سیستم Linux نیز وجود داشته باشد.
مثلاً:

```
smbpasswd -a username
```

یک user موجود Linux را به Samba password database اضافه می‌کند.
برای حذف:

```
smbpasswd -x username
```

برای disable موقت:

```
smbpasswd -d username
```

و برای enable دوباره:

```
smbpasswd -e username
```

برای تغییر password:

```
smbpasswd username
```

کاربر نیز بسته به configuration می‌تواند password خودش را تغییر دهد.

**unix password sync**

یک گزینه‌ی مهم:

```
unix password sync = yes
```

است.
اگر فعال باشد، تغییر password Samba می‌تواند به password عادی Unix نیز گره بخورد.
این رفتار ممکن است برای یک environment مناسب باشد، ولی می‌تواند باعث سردرگمی شود اگر کاربر انتظار نداشته باشد password Samba و password Linux به هم مرتبط شوند.
obey pam restrictions نیز می‌تواند باعث شود تغییر password Samba تابع policyهایی شود که PAM برای تغییر passwordهای سیستم اعمال می‌کند.

#### 12.4.4 Manual Server Startup

اگر Samba را از package distribution نصب کرده باشید، معمولاً لازم نیست daemonها را به‌صورت دستی اجرا کنید.
سیستم service manager باید آن‌ها را مدیریت کند.
برای بررسی می‌توان از:

```
systemctl --type=service
```

یا وضعیت serviceهای مرتبط استفاده کرد.
اما در installationهایی که از source انجام شده‌اند، ممکن است نیاز باشد daemonها را مستقیماً اجرا کنید.
دو daemon سنتی مهم عبارت‌اند از:

- nmbd
- smbd

**nmbd**
nmbd با سرویس‌های NetBIOS name-related کار می‌کند.

**smbd**
نام درست:

```
smbd
```

است و کار اصلی handling درخواست‌های SMB را انجام می‌دهد.
کتاب برای installationهای دستی نمونه‌هایی مثل اجرای daemon در daemon mode با -D ارائه می‌کند.
در سیستم‌های مدرن، بهتر است system manager مانند systemd این processها را supervise کند.
اگر smb.conf تغییر کند، daemon باید configuration جدید را دریافت کند؛ در محیط‌های modern معمولاً استفاده از service manager برای restart/reload روش بهتری است.

#### 12.4.5 Diagnostics and Logfiles

وقتی Samba مشکل startup داشته باشد، ممکن است خطا مستقیماً روی terminal دیده شود.
اما برای مشکلات runtime باید سراغ logها رفت.
در configurationهای سنتی، فایل‌هایی مانند:

- log.smbd
- log.nmbd

وجود دارند.
معمولاً در مسیرهایی مانند:

```
/var/log/samba/
```

قرار می‌گیرند.
همچنین ممکن است logهای اختصاصی برای clientهای مختلف وجود داشته باشند.
بنابراین برای troubleshooting Samba:

```
testparm
+
systemctl status
+
journalctl
+
/var/log/samba/
```

ترکیب بسیار مفیدی است.

#### 12.4.6 File Share Configuration

برای export کردن یک directory به SMB client، در smb.conf یک section برای share تعریف می‌کنیم.
مثلاً به‌صورت مفهومی:

```
[documents]
    path = /srv/documents
    comment = Shared Documents
    guest ok = no
    writable = yes
    printable = no
```

**path**
مشخص می‌کند share به کدام directory در Linux اشاره می‌کند.

**comment**
توضیحی برای share است.

**guest ok**
مشخص می‌کند guest access مجاز باشد یا خیر.

**writable**
اگر فعال باشد، share قابل نوشتن است.
یک نکته‌ی امنیتی مهم:

> guest + writable ترکیب خطرناکی است و نباید بدون دلیل استفاده شود.

**printable**
برای directory share باید false باشد.

**veto files**
می‌توان بعضی filenameها را از طریق share پنهان یا منع کرد.
مثلاً:

```
veto files = /*.o/bin/
```

یعنی patternهای مشخصی از دسترسی از طریق share کنار گذاشته شوند.
این mechanism می‌تواند برای جلوگیری از export شدن بعضی فایل‌ها مفید باشد.

#### 12.4.7 Home Directories

Samba قابلیت ویژه‌ای به نام:

```
[homes]
```

دارد.
با استفاده از آن می‌توان home directory کاربران را به‌صورت dynamic share کرد.
مثلاً:

```
[homes]
    comment = Home Directories
    browseable = no
    writable = yes
```

در این مدل Samba می‌تواند بر اساس userای که login کرده است، home directory مربوط به او را پیدا کند.
در حالت معمول این اطلاعات از account سیستم و home directory تعریف‌شده برای user گرفته می‌شود.
مثلاً اگر user در سیستم home داشته باشد:

```
/home/alice
```

Samba می‌تواند share مربوط به home او را به‌شکل مناسب ارائه کند.
اگر بخواهید home directoryهای Windows با محل homeهای Linux یکسان نباشند، می‌توان از substitutionهایی مانند:

```
%S
```

در path استفاده کرد.
این امکان زمانی مفید است که مثلاً homeهای Samba در:

```
/u/<username>
```

قرار داشته باشند.

#### 12.4.8 Printer Sharing

Samba فقط برای file sharing نیست.
می‌توان printerهای Linux را نیز برای Windows clientها قابل دسترسی کرد.
در محیطی که CUPS استفاده می‌شود، بخش ویژه‌ای مانند:

```
[printers]
    comment = Printers
    browseable = yes
    printing = CUPS
    printable = yes
    writable = no
```

می‌تواند به configuration اضافه شود.
در اینجا:

```
printing = CUPS
```

به Samba می‌گوید از CUPS به‌عنوان سیستم چاپ Unix استفاده کند.
برای استفاده‌ی درست از این integration، Samba باید با قابلیت‌های لازم CUPS ساخته/configure شده باشد.
بسته به topology و policy، می‌توان access printerها را محدود کرد و حتی در برخی محیط‌ها guest access را به‌صورت کنترل‌شده فعال کرد.

#### 12.4.9 The Samba Client

Samba فقط server نیست.
در Linux می‌توان با:

```
smbclient
```

به یک remote SMB server متصل شد.
برای مشاهده‌ی shareهای یک server:

```
smbclient -L -U username SERVER
```

پس از اجرا، ممکن است password درخواست شود.
در فهرست shareها مواردی مانند:

- Disk
- Printer
- IPC

وجود دارند.
برای file shareهای معمول بیشتر به shareهای Disk توجه می‌کنیم.

**ورود به یک share**

مثلاً:

```
smbclient -U username '\\SERVER\sharename'
```

بعد از ورود، prompt شبیه:

```
smb: \>
```

ظاهر می‌شود.
در این محیط commandهایی مانند:

- get
- put
- cd
- lcd
- pwd
- help

قابل استفاده‌اند.

**get**
فایل را از remote به local می‌آورد:

```
get file
```

**put**
فایل local را به remote می‌فرستد:

```
put file
```

**cd**
در directory remote حرکت می‌کند.

**lcd**
directory local را تغییر می‌دهد.

**pwd**
current directory سمت remote را نشان می‌دهد.

**دستورات local**

با:

```
!command
```

می‌توان command را روی machine محلی اجرا کرد.
مثلاً:

```
!pwd
!ls
```

این مدل باعث می‌شود smbclient از نظر تجربه‌ی کار شبیه یک FTP client قدیمی باشد، با این تفاوت که protocol زیرین SMB است.

**استفاده از CIFS به‌عنوان Filesystem**

اگر بخواهید فقط گهگاه فایل‌ها را بگیرید، smbclient کافی است.
اما اگر بخواهید share را طوری در اختیار سیستم قرار دهید که شبیه یک filesystem محلی باشد، می‌توان از CIFS استفاده کرد.
مثلاً به‌صورت مفهومی:

```
mount -t cifs SERVER:sharename /mnt/share ...
```

در این حالت share remote به یک mount point محلی متصل می‌شود.
برای استفاده از آن معمولاً باید ابزارهای CIFS مربوط به distribution نصب باشند.
نکته‌ی امنیتی مهم این است که password را مستقیماً در command line قرار دادن می‌تواند باعث exposure آن در history یا process information شود.
بنابراین در سیستم‌های واقعی بهتر است credentials با روش مناسب و امن‌تری مدیریت شوند.
کتاب syntax سنتی mount را نشان می‌دهد، اما در deploymentهای modern می‌توان از credential file و optionهای مناسب استفاده کرد.

---

### 12.5 SSHFS

برای اشتراک‌گذاری ساده بین Linux machineها، SSHFS یکی از گزینه‌های بسیار راحت است.
ایده‌ی SSHFS این است:
یک filesystem در user space اجرا می‌شود، یک SSH connection ایجاد می‌کند و فایل‌های سمت remote را در یک mount point محلی ارائه می‌دهد.
بنابراین:

```
Application
   ↓
Local mount point
   ↓
SSHFS
   ↓
SSH / SFTP
   ↓
Remote filesystem
```

یکی از جذاب‌ترین ویژگی‌های SSHFS این است که اگر SSH از قبل روی server کار می‌کند، معمولاً infrastructure جدید و پیچیده‌ای لازم ندارید.

**نصب**

SSHFS در بسیاری از distributionها package جداگانه دارد و ممکن است به‌صورت پیش‌فرض نصب نشده باشد.

**Mount**

ساختار اصلی:

```
sshfs username@host:dir mountpoint
```

مثلاً:

```
sshfs user@server:/home/user/data ~/remote-data
```

بعد:

```
ls ~/remote-data
```

فایل‌های remote را می‌بینید.
اگر username همان username فعلی باشد، می‌توان آن را حذف کرد:

```
sshfs server:/path ~/remote-data
```

همچنین اگر فقط home directory remote را بخواهید، می‌توان remote path را نیز ساده‌تر کرد.

**Unmount**

چون SSHFS یک filesystem در user space است، unmount آن برای user عادی با ابزارهای FUSE انجام می‌شود.
در سیستم‌هایی که fusermount در دسترس است:

```
fusermount -u ~/remote-data
```

و superuser می‌تواند از:

```
umount ~/remote-data
```

استفاده کند.
در بعضی سیستم‌های جدید ممکن است ابزار FUSE متفاوت باشد، بنابراین باید documentation مربوط به سیستم بررسی شود.

**چرا بهتر است به‌صورت regular user mount شود؟**

برای حفظ ownership، identity و security model طبیعی کاربر، معمولاً بهتر است SSHFS توسط همان userای که قرار است فایل‌ها را استفاده کند mount شود، نه اینکه بی‌دلیل root آن را mount کند.

**مزایای SSHFS**

مهم‌ترین مزیت:

```
minimal setup
```

است.
در remote host معمولاً کافی است SFTP در SSH فعال باشد.
مزیت دوم این است که به configuration خاصی در سطح network file sharing نیاز ندارد.
اگر SSH connection برقرار است، SSHFS نیز می‌تواند از همان مسیر استفاده کند.
یعنی تفاوتی نمی‌کند remote host روی:

- trusted LAN
- VPN
- remote network
- Internet

قرار داشته باشد؛ البته quality و security واقعی connection همچنان اهمیت دارد.

**معایب SSHFS**

مهم‌ترین عیب:

```
performance overhead
```

است.
داده باید از لایه‌های مختلف SSH، encryption، transport و FUSE عبور کند.
بنابراین SSHFS همیشه سریع‌ترین روش ممکن برای network filesystem نیست.
همچنین multiuser scenarioها محدودتر و پیچیده‌تر از سیستم‌هایی مانند NFS هستند.
در نتیجه SSHFS برای:

- دسترسی شخصی
- کارهای توسعه‌ای سبک
- انتقال یا مشاهده‌ی فایل
- lab environment

بسیار مناسب است، ولی برای یک infrastructure بزرگ چندکاربره باید گزینه‌های دیگر را نیز بررسی کرد.

---

### 12.6 NFS

یکی از سیستم‌های سنتی و بسیار مهم برای file sharing بین Unix/Linux:

```
NFS
```

یا:

```
Network File System
```

است.
NFS در محیط‌هایی که چند Linux/Unix machine باید به storage مشترک دسترسی داشته باشند، سابقه‌ی طولانی دارد.
ایده‌ی اصلی:

```
NFS Server
    │
    │ exported filesystem
    ▼
NFS Client
    │
    ▼
mount point
```

در client، directory remote می‌تواند به یک mount point local متصل شود و برنامه‌ها معمولاً همان filesystem interface معمول Linux را می‌بینند.

#### NFS vs SSHFS

NFS و SSHFS هر دو می‌توانند یک directory remote را روی client قابل‌استفاده کنند، ولی فلسفه‌ی آن‌ها متفاوت است.

```
SSHFS:
سادگی
+
SSH
+
user-space
را هدف می‌گیرد.
```

```
NFS:
network filesystem
+
filesystem semantics
+
integration عمیق‌تر
+
مناسب‌تر برای shared infrastructure
است.
```

به همین دلیل NFS در networkهای داخلی و storageهای NAS بسیار رایج است.

#### Mounting NFS

ساختار کلی:

```
mount -t nfs server:directory mountpoint
```

مثلاً:

```
mount -t nfs server:/data /mnt/data
```

در بسیاری از موارد mount می‌تواند filesystem type را خودش تشخیص دهد، بنابراین -t nfs همیشه ضروری نیست.
اما دانستن آن برای فهم command و کنترل بهتر مفید است.
برای جزئیات optionها می‌توان از:

```
man 5 nfs
```

کمک گرفت.

#### NFS and Transport

NFS می‌تواند در سناریوهای مختلف از TCP یا UDP استفاده کند و versionها و configurationهای متعددی دارد.
بنابراین NFS یک protocol بسیار ساده و تک‌حالتی نیست.
در محیط‌های مختلف ممکن است مواردی مانند:

- NFS version
- transport
- authentication
- encryption
- mount options

تغییر کنند.
به همین دلیل کتاب فقط بخش‌های پایه‌ی client-side را توضیح می‌دهد و وارد همه‌ی configurationهای NFS نمی‌شود.

#### NFS Security

یکی از optionهای مهم:

```
sec=
```

است.
این option روش security/authentication مورد استفاده را تعیین می‌کند.
در بعضی networkهای کوچک و کاملاً بسته ممکن است host-based restrictions استفاده شوند.
اما در محیط‌های حساس‌تر می‌توان از روش‌هایی مبتنی بر:

```
Kerberos
```

استفاده کرد.
این روش‌ها configuration بیشتری لازم دارند.

> نکته‌ی مهم:
> NFS را نباید صرفاً به دلیل اینکه روی LAN اجرا می‌شود، خودکار امن فرض کرد.

اگر network قابل اعتماد نیست، authentication و encryption باید به‌صورت جدی طراحی شوند. کتاب نیز اشاره می‌کند که بسیاری از قابلیت‌های security در NFS به‌صورت پیش‌فرض فعال نیستند و امنیت قوی‌تر نیازمند configuration بیشتری است.

#### Automounter

یک مشکل مهم در network filesystem این است که filesystem remote ممکن است همیشه در دسترس نباشد.
فرض کنید /etc/fstab طوری تنظیم شده که:

NFS server

در boot mount شود.
اگر server down باشد یا network هنوز آماده نباشد، boot ممکن است با delay یا failure همراه شود.
برای همین automounter اهمیت پیدا می‌کند.
به جای اینکه filesystem را از همان boot همیشه mount کنیم:

```
boot
 ↓
mount NFS immediately
```

می‌توان کاری کرد که:

```
boot
 ↓
no immediate mount

user accesses path
 ↓
automount
 ↓
NFS connection
```

برقرار شود.
کتاب به ابزارهای سنتی automounting و سپس systemd mount/automount units اشاره می‌کند و تأکید دارد که automount می‌تواند وابستگی مستقیم boot به network storage را کاهش دهد.

#### NFS Servers

راه‌اندازی NFS server از client پیچیده‌تر است.
در سمت server باید componentهای مربوط به NFS را اجرا کرد و مشخص کرد چه directoryهایی export شوند.
یکی از configurationهای مهم:

```
/etc/exports
```

است.
همچنین سرویس‌های مربوط به:

- nfsd
- mountd

نقش مهمی دارند.
اما کتاب عمداً وارد آموزش کامل ساخت NFS server نمی‌شود؛ یکی از دلایل اصلی این است که در بسیاری از محیط‌ها تهیه‌ی یک NAS که خودش NFS server را مدیریت کند، از ساختن و نگهداری infrastructure کامل NFS ساده‌تر است.
NASهای زیادی نیز در داخل خود از Linux استفاده می‌کنند و قابلیت‌هایی مانند:

- RAID
- NFS
- SMB
- Cloud backup
- Management UI

را آماده ارائه می‌کنند.

---

### 12.7 Cloud Storage

یک نوع دیگر storage شبکه‌ای، cloud storage است.
نمونه‌هایی که کتاب مطرح می‌کند:

- Amazon S3
- Google Cloud Storage

هستند.
این سیستم‌ها از نظر performance معمولاً با local network storage رقابت نمی‌کنند.
اما دو مزیت بزرگ دارند:

```
Maintenance کم
+
زیرساخت storage و backup مدیریت‌شده
```

یعنی شما لازم نیست خودتان تمام سخت‌افزار storage، replication و بخشی از عملیات نگهداری را مدیریت کنید.

#### Cloud Storage and Filesystem

Cloud storage معمولاً API یا HTTP interface مخصوص خودش را دارد.
اما گاهی برنامه‌ای روی Linux انتظار دارد با یک directory و filesystem معمولی کار کند.
اینجاست که یک لایه‌ی مهم وارد می‌شود:

```
FUSE
```

یا:

```
Filesystem in Userspace
```

FUSE اجازه می‌دهد بخش قابل‌توجهی از منطق filesystem در user space اجرا شود، در حالی که kernel یک interface filesystem در اختیار آن برنامه قرار می‌دهد.
به‌صورت مفهومی:

```
Application
   ↓
Filesystem API
   ↓
Kernel / FUSE
   ↓
User-space filesystem process
   ↓
Cloud API
   ↓
Cloud Storage
```

این معماری امکان می‌دهد یک resource که ذاتاً filesystem سنتی نیست، از دید application شبیه filesystem به نظر برسد.

#### FUSE for Cloud

برای cloud storageهای مختلف implementationهای مختلف FUSE وجود دارند.
مثلاً برای storageهایی مثل S3 ممکن است چند implementation متفاوت وجود داشته باشد.
دلیل منطقی این کار این است که FUSE handler در user space قرار دارد و می‌تواند کارهایی مثل:

- Authentication
- Protocol translation
- Encryption
- Caching
- Filename mapping

را بدون تغییر جدی در kernel انجام دهد.
این انعطاف‌پذیری یکی از موضوعات مهم فصل است.
کتاب وارد installation دقیق clientهای cloud storage نمی‌شود، چون implementation هر provider متفاوت است.

---

### 12.8 The State of Network File Sharing

این بخش شاید از نظر فنی ساده‌تر از بعضی قسمت‌های دیگر به نظر برسد، اما از مهم‌ترین بخش‌های مفهومی فصل است.
تا اینجا چندین روش مختلف دیدیم:

- Quick Copy
- rsync
- Samba/CIFS
- SSHFS
- NFS
- FUSE-based cloud filesystems

سؤال طبیعی این است:
چرا هنوز یک روش واحد و کامل برای network file sharing نداریم؟
جواب ساده‌ای وجود ندارد.
چون مسئله‌ی network file sharing هم‌زمان باید چند چیز دشوار را حل کند:

- Performance
- Security
- Authentication
- Authorization
- Filesystem semantics
- Network failures
- Multiuser support
- Compatibility
- Scalability

هر راهکار بخشی از این مسئله را بهتر حل می‌کند.

#### NFS Security Limitations

NFS یک technology قدیمی و قدرتمند است، اما security آن به configuration وابستگی زیادی دارد.
در حالت پایه، ممکن است security خیلی قوی نباشد و برای رسیدن به سطح مناسب لازم باشد infrastructure بیشتری به آن اضافه کنیم.
در مقابل، بعضی نسخه‌ها و implementationهای مدرن CIFS/SMB قابلیت‌های امنیتی قوی‌تری در خود protocol دارند.
اما security تنها مشکل نیست.
حتی اگر authentication و encryption درست باشند، network latency همچنان باقی می‌ماند.
و اگر remote filesystem موقتاً unreachable شود، برنامه‌ای که شدیداً به آن وابسته است ممکن است بسیار کند شود یا حتی در operationهای فایل گیر کند.

#### Andrew File System — AFS

یکی از تلاش‌های مهم برای حل مشکلات network file sharing:

```
AFS
```

یا:

```
Andrew File System
```

بود.
AFS از دهه‌ی 1980 طراحی شد و هدفش ارائه‌ی یک راه‌حل جامع‌تر برای distributed file access بود.
اما چرا AFS همه‌جا استفاده نمی‌شود؟
یکی از دلایل مهم، پیچیدگی dependencyهای آن است.
برای مثال، security mechanism آن به:

```
Kerberos
```

وابسته است.
Kerberos قدرتمند است، ولی راه‌اندازی و نگهداری یک Kerberos infrastructure ساده نیست.
یعنی برای استفاده از یک file-sharing technology باید ابتدا بخش‌های دیگری از identity و authentication infrastructure را نیز آماده کنید.
برای سازمان‌هایی مانند:

- Universities
- Large institutions
- Financial organizations

این هزینه ممکن است قابل‌قبول باشد.
اما برای یک کاربر یا network کوچک:

- NFS
- CIFS
- SSHFS

ممکن است انتخاب ساده‌تری باشند.
این مسئله نمونه‌ی خوبی از trade-off بین:

```
Power
+
Security
+
Complexity
```

است.

#### Kernel-based Network Filesystems

یکی دیگر از موضوعات مهم این بخش این است که بسیاری از network filesystem clientهای سنتی، مخصوصاً NFS، به‌شدت با kernel درگیر بوده‌اند.
در نگاه اول این موضوع منطقی به نظر می‌رسد:

```
Filesystem
    ↓
Kernel
```

اما network filesystemها بسیار پیچیده‌تر از یک local filesystem هستند.
Kernel باید با موضوعاتی مانند:

- network failures
- authentication
- encryption
- remote naming
- server state
- protocol differences

درگیر شود.
authentication به‌خصوص چیزی نیست که به‌سادگی بتوان همه‌ی منطق آن را در kernel نگه داشت.
علاوه بر آن، هرچه client پیچیده‌تر باشد، توسعه‌دهندگان کمتری می‌توانند روی آن کار کنند، چون برای توسعه به دانش kernel programming و زیرساخت مرتبط نیاز دارند.
این موضوع دامنه‌ی توسعه و نگهداری سیستم را محدود می‌کند.

#### Moving to User Space

اینجاست که FUSE و cloud storage اهمیت پیدا می‌کنند.
در معماری‌های جدیدتر می‌توان بخش زیادی از client را در user space نگه داشت.
در نتیجه:

- Authentication
- Encryption
- Protocol handling
- Cloud API
- Filename handling

می‌توانند بیرون از kernel انجام شوند.
FUSE به kernel یک interface مشخص می‌دهد، اما منطق واقعی filesystem می‌تواند در یک process user-space قرار داشته باشد.
به‌صورت مفهومی:

```
Application
   ↓
Filesystem API
   ↓
Kernel / FUSE
   ↓
User-space implementation
   ↓
Remote service
```

این طراحی انعطاف بیشتری ایجاد می‌کند.
اگر یک provider cloud protocol جدیدی داشته باشد، لازم نیست برای هر تغییر کوچک kernel را تغییر دهید.
همچنین developerهای بیشتری می‌توانند در user space implementation بنویسند.
کتاب همین موضوع را یکی از مسیرهای مهم آینده‌ی network file sharing می‌داند: انتقال بیشتر functionality به user space و استفاده از security mechanismهایی مانند TLS به جای وابستگی شدید به infrastructure پیچیده‌ی kernel یا Kerberos.

---

### Choosing the Right Tool for File Transfer and File Sharing

بعد از خواندن این فصل، بهتر است این ابزارها را با هم قاطی نکنیم.

#### 🔹 Quick Copy

وقتی فقط چند فایل را برای مدت کوتاه می‌خواهید منتقل کنید:

```
Python HTTP server
```

مناسب است.
ویژگی اصلی:

- سریع
- ساده
- موقت

اما security آن محدود است.

#### 🔹 rsync

وقتی هدف:

- Synchronization
- Backup
- Mirror
- Repeated transfer

باشد، معمولاً انتخاب مناسب‌تری است.
نکته‌های مهم:

- rsync -a
- rsync -n
- rsync -v
- rsync --delete
- rsync --exclude
- rsync --checksum
- rsync --backup
- rsync --update
- rsync -z
- rsync --bwlimit

#### 🔹 Samba / CIFS

وقتی Linux و Windows باید با هم file sharing داشته باشند:

```
SMB
+
Samba
```

انتخاب اصلی است.
سمت Linux می‌توان با:

```
smbclient
```

کار کرد یا share را با CIFS به‌صورت filesystem mount کرد.

#### 🔹 SSHFS

وقتی SSH از قبل روی server در دسترس است و یک روش ساده برای دسترسی به فایل‌های remote می‌خواهید:

```
SSHFS
```

گزینه‌ی بسیار خوبی است.
مخصوصاً برای:

- یک کاربر
- lab
- development
- دسترسی remote ساده

#### 🔹 NFS

برای shared filesystem بین Unix/Linux و storageهایی مثل NAS:

```
NFS
```

یکی از راه‌حل‌های سنتی و مهم است.
اما security، network availability و automount را باید جدی گرفت.

#### 🔹 FUSE / Cloud Filesystems

وقتی resource اصلی:

- Object storage
- Cloud storage
- Remote API

باشد و بخواهیم آن را مثل filesystem ببینیم، FUSE می‌تواند bridge مناسبی ایجاد کند.

---

### Practical Tips

#### 🔹 rsync --delete را بدون dry run اجرا نکنید

ترتیب مناسب:

```
rsync -an --delete source/ host:destination/
```

بررسی:
چه چیزی قرار است حذف شود؟
و بعد:

```
rsync -a --delete source/ host:destination/
```

اجرای واقعی.

#### 🔹 در rsync به dir و dir/ دقت کنید

```
dir
→ directory itself

dir/
→ contents of directory
```

این یکی از ساده‌ترین جاهایی است که یک اشتباه کوچک می‌تواند نتیجه‌ی بزرگی ایجاد کند.

#### 🔹 Network storage را با local storage یکی ندانید

اگر workload شما:

- random access
- many small files
- frequent metadata operations

دارد، latency شبکه می‌تواند به‌شدت اثرگذار باشد.
برای:

- large sequential reads
- streaming
- centralized storage

network storage می‌تواند مناسب‌تر باشد.

#### 🔹 Authentication با Encryption فرق دارد

اینکه user login می‌کند به این معنی نیست که traffic رمزنگاری شده است.
همیشه جداگانه بپرسید:

- Who are you?
- What can you access?
- Can someone read the traffic?

#### 🔹 Mount شدن network filesystem به boot می‌تواند خطرناک باشد

اگر remote server در دسترس نباشد، dependency بین boot و network storage می‌تواند مشکل‌ساز شود.
برای filesystemهایی که همیشه لازم نیستند، automount می‌تواند معماری مناسب‌تری باشد.

---

### Summary

فصل دوازدهم نشان می‌دهد برای انتقال یا اشتراک‌گذاری فایل روی شبکه یک راه‌حل واحد وجود ندارد.
rsync برای synchronization و backup بسیار مناسب است، Samba/CIFS برای ارتباط Linux و Windows کاربرد دارد، SSHFS راهی ساده برای دسترسی به فایل‌های remote از طریق SSH است و NFS برای shared filesystemهای Unix/Linux و NASها اهمیت دارد. در طرف دیگر، FUSE امکان می‌دهد storageهایی مانند cloud object storage از دید برنامه‌ها شبیه filesystem به نظر برسند.
مفهوم اصلی فصل این است که هنگام انتخاب راهکار باید نوع workload، performance، امنیت، سادگی و رفتار سیستم هنگام قطع شبکه را هم‌زمان در نظر گرفت.

- Figure 12-1
- Figure 12-2









