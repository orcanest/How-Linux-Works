## اشتراک‌ گذاری فایل روی شبکه
### 🐧 فصل دوازدهم کتاب How Linux Works


در فصل‌های قبلی ، شبکه را از دید Linux بررسی کردیم. از interface و IP گرفته تا TCP ، UDP ، socket و سرویس‌ های شبکه . حالا یک سؤال کاملاً عملی مطرح می‌ شود **وقتی دو یا چند ماشین روی شبکه داریم ، فایل‌ ها را چطور بین آن‌ ها منتقل یا به‌ صورت دائمی share کنیم ؟**

برای این مسئله راه‌ حل واحدی وجود ندارد. گاهی فقط می‌ خواهیم یک فایل را خیلی سریع به ماشین دیگری بدهیم. گاهی لازم است حجم زیادی فایل را مرتباً بین چند سیستم synchronize کنیم. در بعضی محیط‌ ها Linux باید با Windows کار کند ، در بعضی محیط‌ ها لازم است یک filesystem از NAS روی Linux mount شود و در بعضی سناریوها هم storage ابری را می‌ خواهیم مثل یک filesystem محلی در اختیار برنامه‌ ها قرار دهیم.

فصل دوازدهم دقیقاً این تفاوت‌ ها را بررسی می‌ کند و برای سناریوهای مختلف سراغ روش‌ هایی مانند Quick Copy ، rsync ، Samba/CIFS ، SSHFS ، NFS و FUSE-based filesystems می‌ رود. کتاب همچنین عمداً وارد راه‌ حل‌ های بسیار بزرگ و پیچیده‌ ی file sharing در مقیاس سازمان‌ های عظیم نمی‌ شود ، چون چنین محیط‌ هایی به طراحی و زیرساخت بسیار بیشتری نیاز دارند.

---
📚 Table of Contents

- [Quick Copy](#quick-copy)
- [rsync](#rsync)
- [Introduction to File Sharing](#introduction-to-file-sharing)
- [Sharing Files with Samba](#sharing-files-with-samba)
- [SSHFS](#sshfs)
- [NFS](#nfs)
- [Cloud Storage](#cloud-storage)
- [The State of Network File Sharing](#the-state-of-network-file-sharing)
- [Choosing the Right Tool for File Transfer and File Sharing](#choosing-the-right-tool-for-file-transfer-and-file-sharing)
- [Practical Tips](#practical-tips)
- [Summery](#summery)

---

### Quick Copy

گاهی اصلاً به یک file-sharing system کامل نیاز نداریم. فرض کنید روی یک Linux machine چند فایل دارید و می‌ خواهید فقط برای چند دقیقه یا چند ساعت آن‌ ها را در اختیار یک ماشین دیگر روی شبکه‌ ی محلی قرار دهید. در چنین شرایطی می‌ توانید از یک web server بسیار ساده استفاده کنید. کتاب برای Python قدیمی نمونه‌ ای با SimpleHTTPServer ارائه می‌کند ، اما در Python 3 نام این module به http.server تغییر کرده است در Python 3 :

```
python3 -m http.server
```

این دستور directory فعلی را به‌ صورت موقت از طریق HTTP ارائه می‌ کند و به‌ طور پیش‌ فرض روی port 8000 هم listen می‌ دهد. مثلاً اگر IP ماشین شما 10.1.2.4 باشد ، ماشین دیگر می‌ تواند به http://10.1.2.4:8000 متصل شود. اگر directory شامل فایل‌ هایی مثل file1.txt و file2.zip و image.iso باشد ، مرورگر می‌ تواند آن‌ ها را نمایش دهد و در صورت مناسب بودن محتوا امکان دریافت شان وجود دارد. این روش بسیار ساده است و تقریباً هیچ configuration خاصی لازم ندارد اما باید محدودیت امنیتی آن را کاملاً جدی گرفت. این یک file-sharing service امن و کامل نیست اگر روی network نا مطمئن اجرا شود ، هر کسی که به این service دسترسی داشته باشد ممکن است فایل‌ های آن directory را ببیند. بنابراین چنین روشی برای انتقال موقت و شبکه‌ی شخصی و آزمایش و محیط lab و کپی سریع چند فایل مناسب است و برای اشتراک‌ گذاری دائمی و شبکه‌ ی عمومی و داده‌ های حساس و محیطی که کاربران ناشناس به آن دسترسی دارند مناسب نیست. همچنین باید دقت کرد که server از همان directory که command را در آن اجرا کرده‌ اید فایل ارائه می‌ کند. بنابراین اجرای آن در directory اشتباه می‌ تواند ناخواسته فایل‌ های بیشتری را در معرض دید قرار دهد. خود کتاب نیز هشدار می‌ دهد که این روش را فقط در شبکه‌ ای که به آن اعتماد دارید استفاده کنید.

---

### rsync

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

#### 🔹 Getting Started with rsync

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

#### 🔹 Transferring standard files

بدون option خاص ، rsync فایل‌ ها را منتقل می‌ کند مثلاً ```rsync file1 host:``` فایل را به home directory کاربر remote می‌ فرستد. برای destination مشخص از ```rsync file1 host:/tmp``` استفاده می‌ کنیم.

#### 🔹 Directory transfer

اگر بدون گزینه‌ ی recursive یک directory بدهید ، rsync آن را به همان شکل مورد انتظار یک directory tree منتقل نمی‌کند و ممکن است با پیامی مانند ```skipping directory dir``` روبرو شوید. برای انتقال یک directory کامل معمولاً از ```rsync -a dir host:dest_dir``` استفاده می‌ شود. گزینه‌ ی a- یا archive mode مجموعه‌ ای از رفتارهای مفید را فعال می‌ کند از جمله recursion و نگهداری بسیاری از metadata های فایل که در این حالت rsync می‌تواند مواردی را حفظ کند مانند :

- subdirectories
- symbolic links
- permissions
- file modes
- timestamps
- device information

البته حفظ owner و group و برخی metadata ها به این نیز وابسته است که user اجراکننده چه مجوزهایی داشته باشد.

#### 🔹 Dry Run

وقتی مطمئن نیستید نتیجه‌ ی یک transfer دقیقاً چه خواهد بود ، بهتر است ابتدا از ```rsync -n``` استفاده کنید. n- به معنی dry run است. یعنی rsync بررسی می‌ کند چه تغییراتی باید انجام شود ، اما واقعاً فایل‌ ها را منتقل نمی‌ کند. برای اینکه output هم نمایش داده شود ```rsync -nva dir host:dest_dir``` اینجا :

- -n → dry run
- -v → verbose
- -a → archive mode

است این ترکیب قبل از عملیات حساس ، مخصوصاً همراه delete-- ، بسیار مهم است.

#### 🔹 Making Exact Copies of a Directory Structure

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

#### 🔹 Tab Completion

برخی shell ها وقتی directory را با Tab کامل می‌ کنند ، slash انتهایی را نیز اضافه می‌ کنند یعنی ممکن است چیزی که ابتدا dir بوده ، بعد از completion تبدیل شود به dir/ ، برای command های عادی ممکن است این تفاوت چندان مهم نباشد اما در rsync و به‌خصوص --delete می‌ تواند مهم و حتی خطرناک باشد.

<img width="100%" height="427" alt="image" src="https://github.com/user-attachments/assets/e54b28aa-2d95-42e8-aff2-b6678fe0df50" />

<img width="100%" height="284" alt="image" src="https://github.com/user-attachments/assets/42708bf4-db6e-4de4-95ba-e0b4381aeb75" />

تفاوت رفتار dir و dir/ و خطر ترکیب این موضوع با --delete در متن کتاب به‌ صورت مشخص مطرح شده است.

#### 🔹 Excluding Files and Directories

یکی از قابلیت‌ های مهم rsync این است که می‌ توان بخشی از source را از transfer کنار گذاشت مثلاً فرض کنید :

```
src/
├── app/
├── data/
└── .git/
```

داریم و نمی‌خواهیم git. منتقل شود می‌ توان نوشت :

```
rsync -a --exclude=.git src host:
```
نکته ی مهم این است که exclude-- با pattern کار می‌ کند ، نه صرفاً یک path مطلق بنابراین :


```
--exclude=.git
```

می‌ تواند مواردی با این نام را در مسیر transfer حذف کند. اگر بخواهید یک مورد مشخص در ساختار transfer را هدف بگیرید ، می‌ توان pattern دقیق‌ تری نوشت :
```
--exclude=/src/.git
```

در اینجا / اول به root filesystem اشاره نمی‌ کند بلکه نسبت به base directory مربوط به transfer تفسیر می‌ شود.

#### 🔹 Multiple excludes
لازم نیست فقط یک exclude داشته باشید می‌ توانید چند مورد را پشت سر هم قرار دهید :

```
rsync -a \
    --exclude=.git \
    --exclude=*.tmp \
    --exclude=cache/ \
    src/ host:dest/
```

#### 🔹 exclude-from

اگر الگو های زیادی دارید ، بهتر است آن‌ ها را در یک فایل قرار دهید :

```
.git
*.tmp
cache/
node_modules/
```

و سپس :

```
rsync -a --exclude-from=exclude.txt src/ host:dest/
```

این روش برای backup ها و synchronization های دائمی بسیار مناسب‌ تر است.

#### 🔹 include

اگر exclude pattern بیش از حد گسترده باشد ، می‌ توان در موارد مناسب از include-- برای وارد کردن موارد خاص استفاده کرد. درک ترتیب و pattern matching include/exclude در rsync برای ساخت rule های دقیق اهمیت زیادی دارد.

#### 🔹 Checking Transfers, Adding Safeguards, and Using Verbose Mode

یکی از دلایل سرعت rsync این است که قرار نیست همیشه محتوای تمام فایل‌ ها را byte-by-byte مقایسه کند. در حالت معمول ، rsync می‌ تواند از ترکیبی از :

- file size
- last modification time

برای تشخیص اینکه فایل ظاهراً تغییر کرده یا نه استفاده کند. اگر source و destination اندازه و timestamp سازگار داشته باشند ، فایل ممکن است نیازی به انتقال مجدد نداشته باشد. این روش بسیار سریع است ، اما گاهی verification بیشتری لازم است.


#### 🔹 --checksum

با ```rsync -c``` یا ```rsync --checksum``` محتوا ی فایل با checksum مقایسه می‌ شود. این کار نیازمند خواندن داده و مصرف CPU است ، بنابراین نسبت به بررسی ساده‌ ی metadata پرهزینه‌ تر است. اما در مواردی که می‌ تواند ارزش داشته باشد : 

- محتوای فایل مهم است.
- اندازه‌ی فایل‌ها اغلب برابر است.
- اطمینان بیشتر از metadata لازم است.


#### 🔹 --ignore-existing

با```rsync --ignore-existing``` فایل‌هایی که از قبل در destination وجود دارند overwrite نمی‌ شوند. این رفتار برای بعضی سناریوهای import یا archive مفید است.

#### 🔹 --backup

با```rsync -b``` یا ```rsync --backup``` وقتی قرار است یک فایل موجود overwrite شود ، rsync قبل از آن نسخه‌ی قبلی را با suffix مناسب نگه می‌ دارد. Suffix پیش‌ فرض معمولاً ```~``` است. می‌توان آن را با ```rsync --backup --suffix=.old``` تغییر داد.

#### 🔹 --update

با```rsync -u``` اگر نسخه‌ی destination جدیدتر از source باشد ، rsync آن فایل را overwrite نمی‌ کند. این گزینه در سناریوهایی که timestamp مقصد باید محافظت شود ، کاربرد دارد.

#### 🔹 Verbose

به‌ صورت پیش‌ فرض rsync نسبتاً ساکت است و بیشتر زمانی output می‌ دهد که مشکلی وجود داشته باشد. برای مشاهده‌ ی فایل‌ ها با ```rsync -v``` و برای جزئیات بیشتر با ```rsync -vv``` استفاده می‌ شود. برای دریافت summary نیز```rsync --stats ``` مفید است. 

برای یک انتقال حساس Workflow می‌ تواند به این صورت ```rsync -anv source/ host:destination/```  باشد ابتدا dry run بعد اجرای واقعی با ```rsync -av source/ host:destination/``` و اگر mirror دقیق می‌ خواهید با ```rsync -anv --delete source/ host:destination/ ``` قبل از ```rsync -av --delete source/ host:destination/```

#### 🔹 Compressing Data

اگر اتصال بین دو host کند باشد ، انتقال داده‌ ی خام ممکن است زمان زیادی ببرد. rsync گزینه‌ ی z- را برای compression در حین انتقال ارائه می‌ دهد مثلاً ```rsync -az dir host:dest_dir``` در این حالت داده‌ ی انتقالی فشرده می‌ شود و در سمت دیگر دوباره unpack/decompress می‌ شود این قابلیت مخصوصاً روی :

- slow links
- internet connections
- high-latency links

می‌ تواند مفید باشد. اما compression همیشه سرعت را بیشتر نمی‌ کند. روی یک LAN بسیار سریع ، ممکن است CPU های دو طرف بیشتر از چیزی که شبکه لازم دارد درگیر compression و decompression شوند در چنین حالتی :

```
fast network
+
slow compression
=
possibly slower transfer
```

بنابراین z- باید بر اساس workload و سرعت link انتخاب شود ، نه اینکه همیشه روشن باشد.

#### 🔹 Limiting Bandwidth

گاهی مشکل انتقال فایل این نیست که خیلی کند است بلکه مشکل این است که بیش از حد سریع است. مثلاً اگر از یک اینترنت با uplink محدود استفاده کنید و یک synchronization بزرگ شروع کنید ، rsync می‌ تواند بخش زیادی از ظرفیت uplink را مصرف کند در نتیجه traffic دیگری نیز تحت تأثیر قرار می‌گیرد مثل :

- HTTP requests
- SSH sessions
- DNS traffic
- other uploads

برای این وضعیت ```bwlimit--``` وجود دارد مثلاً ```rsync --bwlimit=100000 -a dir host:dest_dir``` با این کار می‌توانید سرعت انتقال را محدود کنید تا شبکه برای کارهای دیگر نیز ظرفیت داشته باشد. این قابلیت در backup های دوره‌ ای و synchronization های بزرگ بسیار مفید است.

#### 🔹 Transferring Files to Your Computer

قابلیت rsync فقط برای local → remote نیست. می‌ توانید مسیر را برعکس کنید remote → local مثلاً ```rsync -a host:src_dir dest_dir``` در اینجا source روی host remote قرار دارد و destination روی machine فعلی است. اگر username متفاوت باشد ```rsync -a user@host:src_dir dest_dir``` این ویژگی باعث می‌ شود rsync هم برای ارسال و هم برای دریافت مناسب باشد.

#### 🔹 Further rsync Topics

وقتی تعداد فایل‌ ها زیاد می‌ شود ، rsync یکی از اولین ابزارهایی است که باید به ذهن برسد. یکی از کاربرد های مهم آن ، انتقال batch مجموعه‌ ی بزرگی از فایل‌ ها به چند host مختلف است. از آنجا که rsync synchronization انجام می‌ دهد، در transfer های تکراری معمولاً لازم نیست تمام داده از ابتدا منتقل شود این موضوع برای :

- backup
- mirror
- distribution
- deployment
- archive synchronization

بسیار ارزشمند است.

#### 🔹 Backup with rsync

می‌ توان یک filesystem را با یک storage دیگر همگام کرد مثلاً به‌ صورت مفهومی :

```
Local Filesystem
       ↓
     rsync
       ↓
Remote Storage
```

و در synchronization های برنامه‌ ریزی‌شده ، delete-- می‌تواند destination را به mirror دقیق تبدیل کند. اما یک هشدار مهم اینکه Mirror با Backup کامل یکسان نیست. اگر فایلی به‌ صورت اشتباه حذف شود و synchronization با delete-- انجام شود ، حذف می‌ تواند به backup mirror نیز منتقل شود. به همین دلیل برای backup های واقعی معمولاً باید retention ، versioning یا snapshot نیز در طراحی در نظر گرفته شود. کتاب همچنین اشاره می‌ کند که storage ابری مانند S3 می‌ تواند در کنار ابزارهای دیگر و rsync برای ساختن یک backup system استفاده شود.

---

### Introduction to File Sharing

تا اینجا بیشتر درباره‌ ی copy و synchronization صحبت کردیم حالا مسئله عوض می‌ شود. فرض کنید نمی‌ خواهیم فایل را هر بار کپی کنیم می‌خواهیم یک directory روی ماشین A داشته باشیم و ماشین B بتواند آن را طوری استفاده کند که انگار بخشی از filesystem خودش است این همان network file sharing است در این حالت :

```
Machine A
└── /data

Machine B
└── /mnt/data
```
و /mnt/data روی B نماینده‌ی یک directory remote روی A می‌ شود. این مدل با copy فرق دارد در copy :

```
file
   ↓
duplicate
```

در file sharing :

```
remote storage
   ↓
mounted/accessed remotely
```

است. کتاب در اینجا تأکید می‌ کند که قبل از انتخاب روش file sharing باید اول مشخص کنید چرا اصلاً می‌خواهید file sharing داشته باشید. همین تصمیم روی performance ، architecture و حتی امنیت اثر می‌ گذارد.

#### 🔹 File Sharing Usage and Performance

در شبکه‌ های Unix دو دلیل مهم برای sharing وجود داشت :

- Convenience
- Lack of local storage

مثلاً چند workstation می‌ توانستند به storage متمرکز یک server دسترسی داشته باشند. به جای اینکه برای هر machine مقدار زیادی local storage تهیه کنیم :

```
Workstation A ─┐
Workstation B ─┼── Central Storage
Workstation C ─┘
```

همه از storage متمرکز استفاده می‌ کردند. این مدل از نظر مدیریت storage مزایایی دارد ، اما یک هزینه‌ ی مهم دارد ، storage شبکه معمولاً latency بیشتری نسبت به local storage دارد.  چرا latency مهم است؟ فرض کنید برنامه بخواهد یک فایل بزرگ را از ابتدا تا انتها بخواند. این نوع دسترسی ```sequential access``` برای network storage نسبتاً مناسب است. چون server می‌ تواند داده را با ترتیب قابل‌ پیش‌ بینی بخواند و buffer کند. مثلاً streaming ویدئو یا audio معمولاً الگوی دسترسی نسبتاً قابل‌ پیش‌ بینی دارد اما حالتی مثل : 

- باز کردن تعداد زیادی فایل کوچک
- جستجوی مکرر در directory
- ویرایش پروژه‌ی بزرگ
- خواندن و نوشتن تصادفی

مشکل‌ ساز تر است. در این حالت هر access ممکن است شامل مراحل متعددی باشد :

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

حتی اگر حجم داده کم باشد ، latency می‌ تواند زیاد شود. به همین دلیل ممکن است CPU سمت client زمان زیادی را منتظر network I/O بماند. کتاب به‌ خصوص هشدار می‌ دهد که برای workload هایی مثل توسعه‌ ی نرم‌افزار یا ویرایش فایل‌ های بزرگ ، storage محلی معمولاً انتخاب بسیار بهتری است. در مقابل ، workload هایی با داده‌ ی حجیم و دسترسی ترتیبی ممکن است با network storage عملکرد قابل‌ قبولی داشته باشند. پس قبل از اینکه بگویید می‌خواهم home directory را روی شبکه قرار بدهم بهتر است بپرسید : 

- چه نوع access pattern دارم ؟
- چند فایل ؟
- چقدر random access ؟
- چقدر latency اهمیت دارد ؟
- اگر network قطع شود چه اتفاقی می‌ افتد ؟

#### 🔹 File Sharing Security

امنیت همیشه بخشی از طراحی اولیه‌ ی network file sharing در سیستم‌ های قدیمی نبوده است. به همین دلیل باید سه موضوع را جداگانه بررسی کنیم  با Authentication می توان هویت را تایید کرد ، با Authorization کاربر اجازه دارد با چه سطح دسترسی چه کاری انجام دهد و با Encryption آیا داده در مسیر بین client و server قابل مشاهده است یا خیر. این سه مفهوم یکی نیستند ممکن است server کاربر را به‌ درستی authenticate کند ، ولی داده در مسیر plaintext منتقل شود یا ممکن است ارتباط encrypted باشد ولی user بیش از چیزی که باید دسترسی داشته باشد.

اگر مطمئن نیستید network بین دو machine امن است ، باید لایه‌ ی امنیتی مناسب اضافه کنید. کتاب به ابزارهایی مانند stunnel و IPSec و VPN اشاره می‌ کند که می‌ توانند در لایه‌ های پایین‌ تر به محافظت از traffic کمک کنند. این نکته مخصوصاً درباره‌ ی NFS و بعضی configuration های سنتی مهم است.

---

### Sharing Files with Samba

اگر در شبکه Windows machine دارید، یکی از مهم‌ ترین روش‌ های file sharing، SMB است. SMB مخفف Server Message Block است. Samba مجموعه‌ ی نرم‌افزارهای Unix/Linux برای پیاده‌ سازی SMB و تعامل با شبکه‌ های Windows است. Samba فقط برای این نیست که Windows به Linux دسترسی داشته باشد بلکه Linux می‌ تواند با Samba به Windows server نیز متصل شود به‌ صورت مفهومی : 

```
Windows
    ↕
   SMB
    ↕
Samba
    ↕
Linux
```

همچنین macOS نیز از SMB پشتیبانی می‌ کند. کتاب هدف خود را در این بخش محدود می‌ کند و وارد تمام پیچیدگی‌ های یک Samba deployment بزرگ نمی‌ شود. تمرکز بیشتر روی share کردن فایل و printer در محیط‌ های نسبتاً ساده است. برای راه‌ اندازی کلی Samba ، چهار کار اصلی مطرح می‌ شود : 

1. ایجاد configure/ کردن smb.conf
2. تعریف file share ها
3. در صورت نیاز تعریف printer share ها
4. اجرای daemon های Samba

در package های توزیع معمولاً بخش زیادی از setup پایه توسط package manager و service manager انجام می‌ شود.

#### 🔹 Server Configuration


فایل اصلی configuration Samba نیز ```smb.conf``` است. در بسیاری از distribution ها این فایل در ```/etc/samba/smb.conf ``` قرار دارد. در installation های custom ممکن است location متفاوت باشد. ساختار smb.conf بر اساس section است. مثلاً ```[global]``` برای تنظیمات عمومی server استفاده می‌ شود سپس share ها نیز section جداگانه دارند.

بخش [global] روی رفتار کلی Samba اثر می‌ گذارد مثلاً می‌ توان مشخص کرد : 

```
[global]
    netbios name = LINUXSERVER
    server string = My Linux Samba Server
    workgroup = MYNETWORK
```

نامی که Samba برای server استفاده می‌کند **netbios name** است . اگر تنظیم نشود ، Samba معمولاً از hostname سیستم استفاده می‌ کند. **server string** یک description کوتاه برای server است. **workgroup** نام workgroup یا در بعضی سناریوها domain مربوط به محیط Windows را مشخص می‌ کند و NetBIOS یک API و naming mechanism قدیمی مرتبط با شبکه‌ های Windows است و در بسیاری از configuration های SMB دیده می‌شود.

**تست کردن configuration Samba** قبل از اینکه server را در اختیار client ها قرار دهید ، یکی از ابزارهای بسیار مهم ```testparm``` است. این command configuration را parse و بررسی می‌ کند و اگر syntax یا configuration مشکل‌ داری وجود داشته باشد ، به شما هشدار می‌ دهد مثلاً ```testparm``` می‌ تواند section های موجود را نشان دهد و مشخص کند configuration قابل‌ خواندن است. در troubleshooting Samba بهتر است بعد از تغییر smb.conf ابتدا ```testparm``` را اجرا کنید و سپس سراغ service بروید.

#### 🔹  Server Access Control

همچنین Samba هم option های مختلفی برای کنترل access در اختیار دارد. یکی از این option ها ```= interfaces``` است. می‌ توانید مشخص کنید Samba روی کدام network interface یا network ها در دسترس باشد مثلاً ```interfaces = enp0s31f6 ``` یا با network range مناسب. برای اینکه فقط interface های مشخص‌شده واقعاً مورد استفاده قرار گیرند ، می‌ توان از```bind interfaces only = yes``` استفاده کرد. 

برای **valid users** با ```valid users = user1, user2``` مشخص می‌ کنید چه user هایی اجازه‌ ی دسترسی داشته باشند و **guest ok** با ```guest ok = yes``` می‌توان guest access ایجاد کرد. اما این کار باید با احتیاط انجام شود، مخصوصاً اگر share قابل نوشتن باشد. در network خصوصی و کنترل‌ شده ممکن است کاربرد داشته باشد ، ولی روی شبکه‌ ی نامطمئن می‌ تواند access غیرضروری ایجاد کند.

این گزینه **browseable** تعیین می‌ کند share در browser های شبکه قابل مشاهده باشد یا نه مثلاً ```browseable = no``` یعنی share لزوماً در فهرست browse شده دیده نمی‌ شود ، اما اگر نام دقیق آن را بدانید ممکن است هنوز بتوانید به آن متصل شوید این گزینه access control واقعی نیست و بیشتر روی discovery و visibility اثر دارد.

#### 🔹 Passwords

یک موضوع مهم در Samba وجود دارد اینکه سیستم local password یونیکس با مدل authentication شبکه‌ ی Windows/SMB دقیقاً یکی نیست. بنابراین در configuration های سنتی ، Samba می‌ تواند password database مخصوص خودش داشته باشد. یکی از backend های کلاسیک ```tdbsam``` است. در configuration سنتی می‌ توان چیزی شبیه این داشت :

```
[global]
    security = user
    passdb backend = tdbsam
    obey pam restrictions = yes
```

محل database نیز می‌ تواند در configuration مشخص شود مثلاً ```passdb backend = tdbsam:/path/to/database``` این بخش برای محیط‌ های کوچک مناسب است ، در حالی که integration با domain infrastructure می‌ تواند طراحی متفاوتی داشته باشد.

برای مدیریت user های Samba از ```smbpasswd``` استفاده می‌ شود. اما یک نکته‌ ی مهم وجود دارد اینکه user مورد نظر باید در سیستم Linux نیز وجود داشته باشد مثلاً ```smbpasswd -a username``` یک user موجود Linux را به Samba password database اضافه می‌ کند.

برای حذف از ```smbpasswd -x username``` و برای disable موقت از ```smbpasswd -d username``` و برای enable دوباره از ```smbpasswd -e username``` و برای تغییر password از ```smbpasswd username``` استفاده می شود. کاربر نیز بسته به configuration می‌ تواند password خودش را تغییر دهد.

یک option مهم ```unix password sync = yes``` است. اگر فعال باشد ، تغییر password Samba می‌ تواند به password عادی Unix نیز گره بخورد. این رفتار ممکن است برای یک environment مناسب باشد ، ولی می‌ تواند باعث سردرگمی شود اگر کاربر انتظار نداشته باشد password Samba و password Linux به هم مرتبط شوند. obey pam restrictions نیز می‌ تواند باعث شود تغییر password Samba تابع policy هایی شود که PAM برای تغییر  password های سیستم اعمال می‌ کند.

#### 🔹 Manual Server Startup

اگر Samba را از package distribution نصب کرده باشید ، معمولاً لازم نیست daemon ها را به‌ صورت دستی اجرا کنید. سیستم service manager باید آن‌ ها را مدیریت کند برای بررسی می‌ توان از ```systemctl --type=service``` یا وضعیت service های مرتبط استفاده کرد. اما در installation هایی که از source انجام شده‌ اند ، ممکن است نیاز باشد daemon ها را مستقیماً اجرا کنید. دو daemon سنتی مهم عبارت‌ اند از nmbd و smbd .

قابلیت اینکه nmbd با سرویس‌ های NetBIOS name-related کار می‌ کند و smbd کار اصلی آن handling درخواست‌ های SMB را انجام می‌ دهد. کتاب برای installation های دستی نمونه‌ هایی مثل اجرای daemon در daemon mode با D- ارائه می‌ کند. در سیستم‌ های مدرن ، بهتر است system manager مانند systemd این process ها را supervise کند. اگر smb.conf تغییر کند ، daemon باید configuration جدید را دریافت کند. در محیط‌ های modern معمولاً استفاده از service manager برای restart/reload روش بهتری است.

#### 🔹 Diagnostics and Logfiles

وقتی Samba مشکل startup داشته باشد ، ممکن است خطا مستقیماً روی terminal دیده شود. اما برای مشکلات runtime باید سراغ log ها رفت. در configuration های سنتی ، فایل‌ هایی مانند log.smbd و log.nmbd وجود دارند. معمولاً در مسیر هایی مانند ```/var/log/samba/``` قرار می‌ گیرند. همچنین ممکن است log های اختصاصی برای client های مختلف وجود داشته باشند. بنابراین برای troubleshooting Samba :

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

#### 🔹 File Share Configuration

برای export کردن یک directory به SMB client ، در smb.conf یک section برای share تعریف می‌ کنیم مثلاً به‌ صورت مفهومی :

```
[documents]
    path = /srv/documents
    comment = Shared Documents
    guest ok = no
    writable = yes
    printable = no
```

توضیحات : *path** مشخص می‌کند share به کدام directory در Linux اشاره می‌کند و **comment** توضیحی برای share است و **guest ok** مشخص می‌ کند guest access مجاز باشد یا خیر و **writable** اگر فعال باشد ، share قابل نوشتن است. **printable** برای directory share باید false باشد و **veto files** می‌ توان بعضی filename ها را از طریق share پنهان یا منع کرد مثلاً ```veto files = /*.o/bin/ ``` یعنی pattern های مشخصی از دسترسی از طریق share کنار گذاشته شوند. این mechanism می‌ تواند برای جلوگیری از export شدن بعضی فایل‌ ها مفید باشد.

> 💡 یک نکته‌ ی امنیتی مهم ترکیب guest + writable خطرناک است و نباید بدون دلیل استفاده شود.

#### 🔹 Home Directories

در Samba قابلیت ویژه‌ ای به نام ```[homes]``` وجود دارد. با استفاده از آن می‌ توان home directory کاربران را به‌ صورت dynamic share کرد مثلاً : 

```
[homes]
    comment = Home Directories
    browseable = no
    writable = yes
```

در این مدل Samba می‌ تواند بر اساس user که login کرده است ، home directory مربوط به او را پیدا کند. در حالت معمول این اطلاعات از account سیستم و home directory تعریف‌ شده برای user گرفته می‌ شود. مثلاً اگر user در سیستم home داشته باشد ```home/alice/``` .  همچنین Samba می‌ تواند share مربوط به home او را به‌ شکل مناسب ارائه کند. اگر بخواهید home directory های Windows با محل home های Linux یکسان نباشند ، می‌ توان از substitution هایی مانند ```%S``` در path استفاده کرد. این امکان زمانی مفید است که مثلاً home های Samba در ```/u/<username>``` قرار داشته باشند.

#### 🔹 Printer Sharing

قابلیت Samba فقط برای file sharing نیست. می‌ توان printer های Linux را نیز برای Windows client ها قابل دسترسی کرد در محیطی که CUPS استفاده می‌ شود ، بخش ویژه‌ ای مانند :

```
[printers]
    comment = Printers
    browseable = yes
    printing = CUPS
    printable = yes
    writable = no
```

می‌ تواند به configuration اضافه شود. در اینجا ```printing = CUPS``` به Samba می‌ گوید از CUPS به‌ عنوان سیستم چاپ Unix استفاده کند. برای استفاده‌ ی درست از این integration، Samba باید با قابلیت‌ های لازم CUPS ساخته configure/ شده باشد. بسته به topology و policy ، می‌ توان access printerها را محدود کرد و حتی در برخی محیط‌ ها guest access را به‌ صورت کنترل‌ شده فعال کرد.

#### 🔹 The Samba Client

فقط Samba یک server نیست ، در Linux می‌ توان با ```smbclient``` به یک remote SMB server متصل شد. برای مشاهده‌ ی share های یک server با ```smbclient -L -U username SERVER```  پس از اجرا ، ممکن است password درخواست شود. در فهرست share ها مواردی مانند Disk و Printer و IPC وجود دارند. برای file share های معمول بیشتر به share های Disk توجه می‌ کنیم.

**ورود به یک share** مثلاً ```smbclient -U username '\\SERVER\sharename'``` بعد از ورود، prompt شبیه ```smb: \>``` ظاهر می‌ شود. در این محیط command هایی  قابل استفاده‌ اند مانند :

- با **get**  فایل را از remote به local می‌آورد : ```get file```.
- با **put** فایل local را به remote می‌فرستد : ```put file``` .
- با **cd** در directory remote حرکت می‌ کند.
- با **lcd** می تواند directory local را تغییر دهد.
- با **pwd** می تواند current directory سمت remote را نشان دهد.

**دستورات local** با ```command!``` می‌ توان command را روی machine محلی اجرا کرد. مثلاً ```pwd! ```. این مدل باعث می‌ شود smbclient از نظر تجربه‌ ی کار شبیه یک FTP client قدیمی باشد ، با این تفاوت که protocol زیرین SMB است. 

**استفاده از CIFS به‌عنوان Filesystem** ، اگر بخواهید فقط گاهی فایل‌ ها را بگیرید ، smbclient کافی است. اما اگر بخواهید share را طوری در اختیار سیستم قرار دهید که شبیه یک local filesystem  باشد ، می‌ توان از CIFS استفاده کرد. مثلاً ```mount -t cifs SERVER:sharename /mnt/share``` . در این حالت share remote به یک mount point local متصل می‌ شود. برای استفاده از آن معمولاً باید ابزارهای CIFS مربوط به distribution نصب باشند. 

> 💡 نکته‌ ی امنیتی مهم این است که password را مستقیماً در command line قرار دادن می‌ تواند باعث exposure آن در history یا process information شود. 

بنابراین در سیستم‌ های واقعی بهتر است credentials با روش مناسب و امن‌ تری مدیریت شوند. کتاب syntax سنتی mount را نشان می‌ دهد ، اما در deployment های modern می‌ توان از credential file و option های مناسب استفاده کرد.

---

### SSHFS

برای اشتراک‌ گذاری ساده بین Linux machine ها ، SSHFS یکی از گزینه‌ های بسیار راحت است. ایده‌ ی SSHFS این است که یک file system در user space اجرا می‌ شود ، یک SSH connection ایجاد می‌ کند و فایل‌ های سمت remote را در یک local mount point ارائه می‌ دهد بنابراین :

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

یکی از جذاب‌ ترین ویژگی‌ های SSHFS این است که اگر SSH از قبل روی server کار می‌ کند ، معمولاً infrastructure جدید و پیچیده‌ ای لازم ندارید. 

برای نصب SSHFS در بسیاری از distribution ها package جداگانه دارد و ممکن است به‌ صورت پیش‌فرض نصب نشده باشد. 

برای **Mount** کردن هم ساختار اصلی به صورت ```sshfs username@host:dir mountpoint``` می باشد مثلاً ```sshfs user@server:/home/user/data ~/remote-data``` سپس می توان با ```ls ~/remote-data``` فایل‌ های remote را می‌ بینید. اگر username همان username فعلی باشد ، می‌ توان آن را با ```sshfs server:/path ~/remote-data``` حذف کرد همچنین اگر فقط home directory remote را بخواهید، می‌ توان remote path را نیز ساده‌ تر کرد.

برای **Unmount** ، چون SSHFS یک filesystem در user space است unmount آن برای user عادی با ابزارهای FUSE انجام می‌ شود. در سیستم‌ هایی که fusermount در دسترس است ```fusermount -u ~/remote-data``` و superuser می‌تواند از ```umount ~/remote-data``` استفاده کند.

در بعضی سیستم‌ های جدید ممکن است ابزار FUSE متفاوت باشد ، بنابراین باید documentation مربوط به سیستم بررسی شود. **چرا بهتر است به‌صورت regular user mount شود؟** چون برای حفظ ownership ، identity و security model طبیعی کاربر، معمولاً بهتر است SSHFS توسط همان user که قرار است فایل‌ ها را استفاده کند mount شود ، نه اینکه بی‌ دلیل root آن را mount کند.

**مهم‌ ترین مزیت SSHFS** هم ```minimal setup``` است. در remote host معمولاً کافی است SFTP در SSH فعال باشد. مزیت دوم این است که به configuration خاصی در سطح network file sharing نیاز ندارد. اگر SSH connection برقرار است ، SSHFS نیز می‌ تواند از همان مسیر استفاده کند. یعنی تفاوتی نمی‌ کند remote host روی  trusted LAN یا VPN یا remote network یا Internet قرار داشته باشد البته quality و security واقعی connection همچنان اهمیت دارد.

**مهم‌ ترین عیب SSHFS** هم ```performance overhead```است. داده باید از لایه‌ های مختلف SSH ، encryption ، transport و FUSE عبور کند بنابراین SSHFS همیشه سریع‌ ترین روش ممکن برای network file system نیست. همچنین multiuser scenario ها محدود تر و پیچیده‌ تر از سیستم‌ هایی مانند NFS هستند. در نتیجه SSHFS برای دسترسی شخصی ، کارهای توسعه‌ ای سبک ، انتقال یا مشاهده‌ی فایل و lab environment بسیار مناسب است ولی برای یک infrastructure بزرگ چند کاربره باید گزینه‌ های دیگر را نیز بررسی کرد.

---

### NFS

یکی از سیستم‌ های سنتی و بسیار مهم برای file sharing بین Unix/Linux هم NFS یا Network File System است. NFS در محیط‌ هایی که چند Linux/Unix machine باید به storage مشترک دسترسی داشته باشند ، سابقه‌ ی طولانی دارد. ایده‌ ی اصلی :

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

در client ، directory remote می‌ تواند به یک mount point local متصل شود و برنامه‌ ها معمولاً همان filesystem interface معمول Linux را می‌ بینند.

#### 🔹 NFS vs SSHFS

هر دو یعنی NFS و SSHFS می‌ توانند یک directory remote را روی client قابل‌ استفاده کنند ، ولی فلسفه‌ ی آن‌ ها متفاوت است. هدف SSHFS سادگی و SSH و user-space است ولی NFS برای network file system و filesystem semantics و integration عمیق‌ تر و مناسب‌ تر برای shared infrastructure است به همین دلیل NFS در network های local و storage های NAS بسیار رایج است.

#### 🔹 Mounting NFS

ساختار کلی به صورت ```mount -t nfs server:directory mountpoint``` است مثلاً ```mount -t nfs server:/data /mnt/data``` در بسیاری از موارد mount می‌ تواند filesystem type را خودش تشخیص دهد ، بنابراین ``` t nfs``` همیشه ضروری نیست اما دانستن آن برای فهم command و کنترل بهتر مفید است. برای جزئیات option ها می‌ توان از ```man 5 nfs``` کمک گرفت.


#### 🔹 NFS and Transport

همچنین NFS می‌ تواند در سناریوهای مختلف از TCP یا UDP استفاده کند و version ها و configuration های متعددی دارد. بنابراین NFS یک protocol بسیار ساده و تک‌ حالتی نیست در محیط‌ های مختلف ممکن است مواردی مانند :

- NFS version
- transport
- authentication
- encryption
- mount options

تغییر کنند. به همین دلیل کتاب فقط بخش‌ های پایه‌ ی client-side را توضیح می‌ دهد و وارد همه‌ ی configuration های NFS نمی‌ شود.

#### 🔹 NFS Security

یکی از option های مهم ```=sec``` است. این option روش security/authentication مورد استفاده را تعیین می‌ کند. در بعضی network های کوچک و کاملاً بسته ممکن است host-based restrictions استفاده شوند. اما در محیط‌ های حساس‌ تر می‌ توان از روش‌ هایی مبتنی بر```Kerberos``` استفاده کرد. این روش‌ ها configuration بیشتری لازم دارند.

> 💡 نکته‌ ی مهم : NFS را نباید صرفاً به دلیل اینکه روی LAN اجرا می‌ شود ، خودکار امن فرض کرد.

اگر network قابل اعتماد نیست ، authentication و encryption باید به‌ صورت جدی طراحی شوند. کتاب نیز اشاره می‌ کند که بسیاری از قابلیت‌ های security در NFS به‌ صورت پیش‌ فرض فعال نیستند و امنیت قوی‌ تر نیازمند configuration بیشتری است.

#### 🔹 Automounter

یک مشکل مهم در network file system این است که filesystem remote ممکن است همیشه در دسترس نباشد. فرض کنید /etc/fstab طوری تنظیم شده که NFS server در boot بخواهد mount شود. اگر server down باشد یا network هنوز آماده نباشد ، boot ممکن است با delay یا failure همراه شود. برای همین automounter اهمیت پیدا می‌ کند و به جای اینکه filesystem را از همان boot همیشه mount کنیم :
```
boot
 ↓
mount NFS immediately
```

می‌ توان کاری کرد که :

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

برقرار شود. کتاب به ابزارهای سنتی automounting و سپس systemd mount/automount units اشاره می‌ کند و تأکید دارد که automount می‌ تواند وابستگی مستقیم boot به network storage را کاهش دهد.

#### 🔹 NFS Servers

راه‌ اندازی NFS server از client پیچیده‌ تر است. در سمت server باید component های مربوط به NFS را اجرا کرد و مشخص کرد چه directory هایی export شوند. یکی از configuration های مهم ```/etc/exports``` است. همچنین سرویس‌ های مربوط به nfsd و mountd نقش مهمی دارند.

اما کتاب عمداً وارد آموزش کامل ساخت NFS server نمی‌ شود. یکی از دلایل اصلی این است که در بسیاری از محیط‌ ها تهیه‌ ی یک NAS که خودش NFS server را مدیریت کند ، از ساختن و نگهداری infrastructure کامل NFS ساده‌ تر است. NAS های زیادی نیز در داخل خود از Linux استفاده می‌ کنند و قابلیت‌ هایی مانند RAID و NFS و SMB و Cloud backup و Management UI را آماده ارائه می‌ کنند.

---

### Cloud Storage

یک نوع دیگر storage شبکه‌ ای ، cloud storage است و نمونه‌ هایی که کتاب مطرح می‌ کند Amazon S3 و Google Cloud Storage هستند. این سیستم‌ ها از نظر performance معمولاً با local network storage رقابت نمی‌ کنند. اما دو مزیت بزرگ دارند Maintenance کم و زیرساخت storage و backup مدیریت‌ شده. یعنی شما لازم نیست خودتان تمام سخت‌افزار storage ، replication و بخشی از عملیات نگهداری را مدیریت کنید.

#### 🔹 Cloud Storage and Filesystem

معمولاً Cloud storage با API یا HTTP interface مخصوص خودش را دارد. اما گاهی برنامه‌ ای روی Linux انتظار دارد با یک directory و file system معمولی کار کند. اینجاست که یک لایه‌ ی مهم وارد می‌ شود به نام ```FUSE``` یا ```Filesystem in Userspace```  که FUSE اجازه می‌ دهد بخش قابل‌ توجهی از منطق file system در user space اجرا شود ، در حالی که kernel یک interface file system در اختیار آن برنامه قرار می‌ دهد. به‌ صورت مفهومی :

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

این معماری امکان می‌ دهد یک resource که ذاتاً filesystem سنتی نیست، از دید application شبیه filesystem به نظر برسد.

#### 🔹 FUSE for Cloud

برای cloud storage های مختلف implementation های مختلف FUSE وجود دارند مثلاً برای storage هایی مثل S3 ممکن است چند implementation متفاوت وجود داشته باشد. دلیل منطقی این کار این است که FUSE handler در user space قرار دارد و می‌ تواند کارهایی مثل :

- Authentication
- Protocol translation
- Encryption
- Caching
- Filename mapping

را بدون تغییر جدی در kernel انجام دهد این انعطاف‌ پذیری یکی از موضوعات مهم فصل است. کتاب وارد installation دقیق client های cloud storage نمی‌ شود ، چون implementation هر provider متفاوت است .

---

### The State of Network File Sharing

این بخش شاید از نظر فنی ساده‌ تر از بعضی قسمت‌ های دیگر به نظر برسد، اما از مهم‌ ترین بخش‌ های مفهومی فصل است. تا اینجا چندین روش مختلف دیدیم :

- Quick Copy
- rsync
- Samba/CIFS
- SSHFS
- NFS
- FUSE-based cloud filesystems

سؤال طبیعی این است که چرا هنوز یک روش واحد و کامل برای network file sharing نداریم ؟ جواب ساده‌ ای وجود ندارد. چون مسئله‌ ی network file sharing هم‌ زمان باید چند چیز دشوار را حل کند : 

- Performance
- Security
- Authentication
- Authorization
- Filesystem semantics
- Network failures
- Multiuser support
- Compatibility
- Scalability

هر راهکار بخشی از این مسئله را بهتر حل می‌ کند.

#### 🔹 NFS Security Limitations

همانطور که می دانید NFS یک technology قدیمی و قدرتمند است ، اما security آن به configuration وابستگی زیادی دارد. در حالت پایه ، ممکن است security خیلی قوی نباشد و برای رسیدن به سطح مناسب لازم باشد infrastructure بیشتری به آن اضافه کنیم. در مقابل ، بعضی نسخه‌ ها و implementation های مدرن CIFS/SMB قابلیت‌ های امنیتی قوی‌ تری در خود protocol دارند اما security تنها مشکل نیست. حتی اگر authentication و encryption درست باشند ، network latency همچنان باقی می‌ ماند و اگر remote file system موقتاً unreachable شود ، برنامه‌ ای که شدیداً به آن وابسته است ممکن است بسیار کند شود یا حتی در operation های فایل گیر کند.

#### 🔹 Andrew File System — AFS

یکی از تلاش‌ های مهم برای حل مشکلات network file sharing هم ```AFS``` یا Andrew File System بود. AFS از دهه‌ ی 1980 طراحی شد و هدفش ارائه‌ ی یک راه‌ حل جامع‌ تر برای distributed file access بود. اما چرا AFS همه‌ جا استفاده نمی‌ شود ؟ یکی از دلایل مهم ، پیچیدگی dependency های آن است. برای مثال ، security mechanism آن به ```Kerberos``` وابسته است. Kerberos قدرتمند است ، ولی راه‌ اندازی و نگهداری یک Kerberos infrastructure ساده نیست. یعنی برای استفاده از یک file-sharing technology باید ابتدا بخش‌ های دیگری از identity و authentication infrastructure را نیز آماده کنید. برای سازمان‌ هایی مانند Universities و Large institutions و Financial organizations این هزینه ممکن است قابل‌ قبول باشد اما برای یک کاربر یا network کوچک NFS یا CIFS یا SSHFS ممکن است انتخاب ساده‌ تری باشند. این مسئله نمونه‌ ی خوبی از trade-off بین :

یکی از تلاش‌ های مهم برای حل مشکلات network file sharing هم ```AFS``` یا Andrew File System بود. AFS از دهه‌ ی 1980 طراحی شد و هدفش ارائه‌ ی یک راه‌ حل جامع‌ تر برای distributed file access بود. اما چرا AFS همه‌ جا استفاده نمی‌ شود ؟ یکی از دلایل مهم ، پیچیدگی dependency های آن است. برای مثال ، security mechanism آن به ```Kerberos``` وابسته است. Kerberos قدرتمند است ، ولی راه‌ اندازی و نگهداری یک Kerberos infrastructure ساده نیست. یعنی برای استفاده از یک file-sharing technology باید ابتدا بخش‌ های دیگری از identity و authentication infrastructure را نیز آماده کنید. برای سازمان‌ هایی مانند Universities و Large institutions و Financial organizations این هزینه ممکن است قابل‌ قبول باشد اما برای یک کاربر یا network کوچک NFS یا CIFS یا SSHFS ممکن است انتخاب ساده‌ تری باشند. این مسئله نمونه‌ ی خوبی از trade-off بین :

```
Power
+
Security
+
Complexity
```

است.

#### 🔹 Kernel-based Network Filesystems

یکی دیگر از موضوعات مهم این بخش این است که بسیاری از network filesystem client های سنتی ، مخصوصاً NFS، به‌ شدت با kernel درگیر بوده‌ اند. در نگاه اول این موضوع منطقی به نظر می‌ رسد :

```
Filesystem
    ↓
Kernel
```

اما network file system ها بسیار پیچیده‌ تر از یک local filesystem هستند. Kernel باید با موضوعاتی درگیر شود مانند :
- network failures
- authentication
- encryption
- remote naming
- server state
- protocol differences

احراز هویت یا authentication به‌ خصوص چیزی نیست که به‌ سادگی بتوان همه‌ ی منطق آن را در kernel نگه داشت. علاوه بر آن ، هرچه client پیچیده‌ تر باشد ، توسعه‌ دهندگان کمتری می‌ توانند روی آن کار کنند ، چون برای توسعه به دانش kernel programming و زیرساخت مرتبط نیاز دارند. این موضوع دامنه‌ ی توسعه و نگهداری سیستم را محدود می‌ کند.

#### 🔹 Moving to User Space

اینجاست که FUSE و cloud storage اهمیت پیدا می‌ کنند. در معماری‌ های جدید تر می‌ توان بخش زیادی از client را در user space نگه داشت در نتیجه :

- Authentication
- Encryption
- Protocol handling
- Cloud API
- Filename handling

می‌ توانند بیرون از kernel انجام شوند. FUSE به kernel یک interface مشخص می‌ دهد، اما منطق واقعی filesystem می‌ تواند در یک process user-space قرار داشته باشد به‌ صورت مفهومی :

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

این طراحی انعطاف بیشتری ایجاد می‌ کند. اگر یک provider cloud protocol جدیدی داشته باشد ، لازم نیست برای هر تغییر کوچک kernel را تغییر دهید همچنین developer های بیشتری می‌ توانند در user space implementation بنویسند. کتاب همین موضوع را یکی از مسیرهای مهم آینده‌ ی network file sharing می‌ داند مثلا انتقال بیشتر functionality به user space و استفاده از security mechanism هایی مانند TLS به جای وابستگی شدید به infrastructure پیچیده‌ ی kernel یا Kerberos.

---

### Choosing the Right Tool for File Transfer and File Sharing

بعد از خواندن این فصل ، بهتر است این ابزار ها را با هم قاطی نکنیم.

#### 🔹 Quick Copy

وقتی فقط چند فایل را برای مدت کوتاه می‌ خواهید منتقل کنید ```Python HTTP server``` مناسب است. ویژگی اصلی سریع و ساده و موقت ، اما security آن محدود است.

#### 🔹 rsync

وقتی هدف Synchronization و Backup و Mirror و Repeated transfer باشد ، معمولاً انتخاب مناسب‌ تری است. نکته‌ های مهم : 

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

وقتی Linux و Windows باید با هم file sharing داشته باشند SMB + Samba انتخاب اصلی است. سمت Linux می‌ توان با smbclient کار کرد یا share را با CIFS به‌ صورت file system mount کرد.

#### 🔹 SSHFS

وقتی SSH از قبل روی server در دسترس است و یک روش ساده برای دسترسی به فایل‌ های remote می‌ خواهید ```SSHFS``` گزینه‌ ی بسیار خوبی است. مخصوصاً برای یک کاربر یا lab یا development یا دسترسی remote ساده .

#### 🔹 NFS

برای shared file system بین Unix/Linux و storage هایی مثل NAS هم ```NFS``` یکی از راه‌ حل‌ های سنتی و مهم است اما security ، network availability و automount را باید جدی گرفت.

#### 🔹 FUSE / Cloud Filesystems

وقتی resource اصلی :

- Object storage
- Cloud storage
- Remote API

باشد و بخواهیم آن را مثل filesystem ببینیم ، FUSE می‌ تواند bridge مناسبی ایجاد کند.

---

### Practical Tips

#### 🔹 Do not run `rsync --delete` without a dry run !

- ترتیب مناسب : ```rsync -an --delete source/ host:destination/``` .
- بررسی : چه چیزی قرار است حذف شود ؟  و بعد اجرای ```rsync -a --delete source/ host:destination/``` .

#### 🔹 In rsync, pay attention to the difference between `dir` and `dir/` !

این یکی از ساده‌ ترین جاهایی است که یک اشتباه کوچک می‌ تواند نتیجه‌ ی بزرگی ایجاد کند :


```
dir → directory itself
/dir → contents of directory
```


#### 🔹 Do not equate network storage with local storage !

اگر workload شما  random access و many small files و frequent metadata operations دارد ، latency شبکه می‌ تواند به‌ شدت اثرگذار باشد برای large sequential reads و streaming و centralized storage نیز network storage می‌ تواند مناسب‌ تر باشد.

#### 🔹 Authentication differs from encryption !

اینکه user login می‌ کند به این معنی نیست که traffic رمزنگاری شده است همیشه جداگانه بپرسید :

- Who are you?
- What can you access?
- Can someone read the traffic?

#### 🔹 Mounting a network filesystem at boot can be dangerous !

اگر remote server در دسترس نباشد ، dependency بین boot و network storage می‌ تواند مشکل‌ ساز شود. برای filesystem هایی که همیشه لازم نیستند ، automount می‌ تواند معماری مناسب‌ تری باشد.

---

### Summery

فصل دوازدهم نشان می‌ دهد برای انتقال یا اشتراک‌ گذاری فایل روی شبکه یک راه‌ حل واحد وجود ندارد. rsync برای synchronization و backup بسیار مناسب است ، Samba/CIFS برای ارتباط Linux و Windows کاربرد دارد ، SSHFS راهی ساده برای دسترسی به فایل‌ های remote از طریق SSH است و NFS برای shared filesystem های Unix/Linux و NAS ها اهمیت دارد. در طرف دیگر ، FUSE امکان می‌ دهد storage هایی مانند cloud object storage از دید برنامه‌ ها شبیه file system به نظر برسند.  مفهوم اصلی فصل این است که **هنگام انتخاب راهکار باید نوع workload ، performance ، امنیت ، سادگی و رفتار سیستم هنگام قطع شبکه را هم‌ زمان در نظر گرفت**.
