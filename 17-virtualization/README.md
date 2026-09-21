## مجازی سازی
### 🐧 فصل هفدهم کتاب How Linux Works

مجازی‌ سازی در Linux فقط به ساختن یک **Virtual Machine** محدود نمی‌ شود. در ساده‌ ترین نگاه ، virtualization یعنی ایجاد یک لایه‌ ی میانی که منابع یا محیط زیرین را به‌ شکل دیگری در اختیار یک یا چند مصرف‌ کننده قرار می‌ دهد. نمونه‌ ای که پیش‌ تر با آن آشنا شدیم **virtual memory** است. هر process می‌ تواند فضای حافظه‌ ی بزرگی را طوری ببیند که انگار متعلق به خودش است ، در حالی که زیر آن یک سیستم مدیریت‌ شده‌ ی مشترک وجود دارد.

در این فصل همین ایده را در سطوح مختلف دنبال می‌کنیم : ابتدا **Virtual Machine** که یک سیستم‌عامل کامل را اجرا می‌ کند ، سپس **Container** که محیط اجرای مجموعه‌ ای از process ها را محدود و جدا می‌ کند و در پایان **Runtime-Based Virtualization** که جدا سازی را در سطح محیط اجرای یک application انجام می‌ دهد. تمرکز فصل بیشتر روی درک مفاهیم ، اصطلاحات و اجزای اصلی است تا جزئیات بسیار پایین‌ سطح implementation.

---
📚 Table of Contents

- [Virtual Machines](#virtual-machines)
- [Containers](#containers)
- [Runtime-Based Virtualization](#runtime-based-virtualization)
- [Summary](#summary)

---

### Virtual Machines

می‌ توان **Virtual Machine** را نسخه‌ ی مجازی‌ شده‌ ی یک کامپیوتر در نظر گرفت. در این مدل ، به کمک software یک machine جدید ایجاد می‌ شود که می‌ تواند processor و memory و I/O interface و سایر اجزای مورد نیاز را داشته باشد. تفاوت مهم آن با یک process معمولی این است که در یک **system virtual machine* ،  یک operating system کامل داخل VM اجرا می‌ شود و آن operating system ، kernel خودش را نیز دارد. به‌همین دلیل guest درون VM می‌ تواند همانند یک سیستم واقعی با hardware مجازی‌ شده کار کند. این مفهوم قدمت زیادی دارد. برای نمونه ، mainframe های IBM سال‌ ها از system virtual machine برای ایجاد محیط‌ های جداگانه و multiuser استفاده کرده‌ اند. یک VM می‌ تواند کاملاً توسط software شبیه‌ سازی شود که در این صورت معمولاً با مفهوم **emulator** رو به‌ رو هستیم ،  یا می‌ تواند تا جای ممکن از hardware واقعی سیستم میزبان استفاده کند. روش دوم به‌خاطر performance بهتر برای کاربرد های معمول Linux مهم‌ تر است. در عمل ، ابزارهایی مانند VirtualBox ایجاد و اجرای VM را ساده می‌ کنند و حتی می‌ توان فرآیند مدیریت VM را با ابزارهای command-line یا interface های ابری automation کرد. بنابراین برای کاربر معمولی ، اهمیت اصلی درک مدل VM و اصطلاحات آن است ، نه پیاده‌ سازی جزئی آن.

#### 🔹 Hypervisors

نرم‌افزاری که مسئول مدیریت یک یا چند Virtual Machine است ، **Hypervisor** یا **Virtual Machine Monitor (VMM)** نام دارد. Hypervisor از نظر نقش شبیه یک لایه‌ ی مدیریتی است که منابع سیستم را میان VMها قرار می‌ دهد و دسترسی آن‌ ها به hardware را کنترل می‌کند و دو نوع اصلی hypervisor وجود دارد به نام type 1 و type 2.

#### 🔹 Type 2 Hypervisor

روی یک operating system معمولی اجرا می‌ شود. در این مدل ، Linux یا سیستم‌ عامل دیگری ابتدا روی hardware اجرا شده و سپس hypervisor در همان محیط اجرا می‌ شود. VirtualBox نمونه‌ ای از Type 2 hypervisor است. این نوع hypervisor برای desktop و آزمایش سیستم‌ های مختلف بسیار قابل‌ دسترس است. نمونه‌ ی شناخته‌ شده‌ ی Type 2  :

```text
Hardware
    |
    v
Host Operating System
    |
    v
Type 2 Hypervisor
    |
    v
Virtual Machine
```

#### 🔹 Type 1 Hypervisor

مستقیماً برای اجرای VM ها طراحی شده و بیشتر شبیه یک operating system مخصوص virtualization عمل می‌ کند. ممکن است یک سیستم کمکی مانند Linux برای management در کنار آن قرار بگیرد ، اما لایه‌ ی اصلی که VM ها را اجرا می‌ کند خود hypervisor است. بخش زیادی از زیرساخت cloud بر پایه‌ ی همین مدل ساخته شده است. وقتی یک cloud provider به شما instance یک operating system می‌ دهد ، در عمل یک Virtual Machine را روی زیر ساختی اجرا می‌ کند که hypervisor نوع 1 آن را مدیریت می‌ کند. در این مدل ، اصطلاحات مهم عبارت‌ اند از :

- اولین اصطلاح **Guest:** خود Virtual Machine همراه با operating system داخلی آن.
- دومین اصطلاح **Host:** سیستمی که hypervisor را اجرا می‌کند. در Type 2 معمولاً همان operating system اصلی شماست. درType 1 هم host در معنای گسترده‌ تر به hypervisor و محیط مدیریتی آن اشاره می‌ کند.

#### 🔹 Hardware in a Virtual Machine

در نگاه اول می‌ توان تصور کرد که hypervisor باید تمام hardware را به‌ صورت کامل emulate کند. مثلاً برای ساخت یک virtual disk می‌ توان روی host یک فایل بزرگ ایجاد کرد و آن را به guest مانند یک disk device واقعی نشان داد. این مدل ، نوعی **hardware virtualization** کاملاً شبیه‌ سازی‌ شده است ، اما از نظر performance همیشه بهترین روش نیست. در عمل ، یکی از مهم‌ ترین تکنیک‌ ها استفاده‌ ی مستقیم‌ تر guest از منابع host است. به این رویکرد **paravirtualization** گفته می‌ شود. در این حالت به‌ جای اینکه guest برای همه‌ چیز با یک virtual hardware کاملاً شبیه‌ سازی‌ شده صحبت کند ، ممکن است از interface ها یا driver هایی استفاده کند که ارتباط مستقیم‌ تری با hypervisor برقرار می‌ کنند. **Network interface** و **block device** از مهم‌ ترین جاهایی هستند که چنین روش‌ هایی کاربرد دارند. برای مثال در یک cloud environment ممکن است device مانند ```dev/xvd/``` ، نمایان شود که در واقع یک virtual disk وابسته به Xen است و driver مربوطه می‌ تواند مستقیماً با hypervisor ارتباط برقرار کند. paravirtualization همیشه فقط برای performance نیست. گاهی برای راحت‌ تر شدن interaction با host استفاده می‌شود برای مثال هماهنگی mouse میان محیط guest و host در محیط‌ هایی مانند VirtualBox. هدف نهایی این abstraction این است که guest تا حد امکان virtual hardware را مثل hardware عادی ببیند. برای یک Linux guest بهتر است virtual disk در نهایت یک **block device** معمولی باشد تا بتوان با ابزار های عادی روی آن partition و filesystem ایجاد کرد.
#### 🔹 Virtual Machine CPU Modes

برای فهمیدن یکی از مشکلات اصلی virtualization باید تفاوت میان **user mode** و **kernel mode** را به یاد داشته باشیم. CPU در kernel mode اجازه‌ ی اجرای operation های بیشتری دارد ، در حالی که user mode محدود تر است و بعضی instruction ها یا دسترسی‌ های memory در آن مجاز نیستند. مشکل Virtual Machine های اولیه روی معماری x86 این بود که operating system داخل guest انتظار داشت بتواند در **kernel mode** کار کند ، در حالی که خود guest در شرایطی مانند user mode اجرا می‌ شد. راه‌ حل این بود که hypervisor instruction های restricted را تشخیص دهد و آن‌ ها را **trap** کند. سپس hypervisor آن operation را به‌ شکل مناسب emulate می‌ کرد تا guest بتواند رفتار مورد انتظار kernel mode را داشته باشد. از آنجا که همه‌ ی instruction های kernel restricted نیستند ، بسیاری از operation ها مستقیماً اجرا می‌ شدند و فقط بخش‌ هایی که نیاز به intervention داشتند هزینه‌ ی اضافی ایجاد می‌ کردند. بعد ها Intel و AMD قابلیت‌ های سخت‌افزاری مخصوص virtualization ارائه کردند :

```text
Intel → VT-x
AMD   → AMD-V
```
این feature ها به processor اجازه می‌ دهند hypervisor در مدیریت guest ها کارآمدتر عمل کند و در بعضی محیط‌ ها حتی وجود آن‌ ها ضروری است.

#### 🔹  Testing and Trials

یکی از مهم‌ ترین کاربرد ها ، **test و آزمایش** است. هنگام توسعه‌ ی software بهتر است software جدید را روی همان سیستمی که محیط اصلی developer است آزمایش نکنید. VM امکان می‌ دهد یک محیط جداگانه و نسبتاً disposable ایجاد کنید. همین موضوع برای امتحان کردن distribution های مختلف نیز کاربرد دارد. می‌ توان یک Linux جدید را داخل VM اجرا کرد بدون اینکه hardware جدیدی خریداری شود یا سیستم اصلی دست‌ کاری شود.

#### 🔹 Application Compatibility

گاهی یک application فقط روی operating system خاصی اجرا می‌ شود. اگر سیستم اصلی شما آن operating system را نداشته باشد ، VM می‌ تواند محیط مورد نیاز را درون همان سیستم ایجاد کند.

#### 🔹 Servers and Cloud Services

در محیط‌ های server و cloud نیز VM ها بسیار رایج‌ اند. اگر بخواهید یک web server یا server application را روی اینترنت در دسترس قرار دهید ، یکی از ساده‌ ترین روش‌ ها استفاده از یک cloud instance است که در اصل یک Virtual Machine است. Cloud provider ها علاوه بر VM خام ، server های از قبل پیکربندی‌ شده مانند database environment نیز ارائه می‌ دهند که باز هم روی زیرساخت VM اجرا می‌ شوند.

#### 🔹 Drawbacks of Virtual Machines

معمولاً VM ها isolation بسیار مفیدی ایجاد می‌ کنند ، اما هزینه‌ هایی نیز دارند :
اولین مسئله **setup و configuration** است. آماده کردن یک operating system کامل و application environment زمان می‌ برد. حتی اگر این کار با automation tools ساده‌ تر شود ، راه‌ اندازی یک سیستم از صفر همچنان نسبت به یک environment سبک‌ تر زمان‌ بر است.

دومین مشکل **boot و reboot** است. یک VM باید یک Linux system کامل را بالا بیاورد و بنابراین startup آن نسبت به چیزی مثل یک process یا container کند تر است.

سومین هزینه ، **maintenance** است. هر VM یک operating system کامل دارد و باید update و security patch شود. ابزارهایی مانند `systemd` و `sshd` و library ها و package های مورد نیاز application نیز بخشی از همین maintenance هستند.

مشکل دیگر dependency ها هستند. ممکن است application شما با software های استانداردی که روی VM نصب شده‌ اند ناسازگار باشد. حتی library هایی که با یک system upgrade تغییر می‌ کنند ، می‌ توانند باعث شوند application که قبلاً کار می‌ کرد دیگر درست اجرا نشود.

موضوع بعدی هزینه‌ ی **resource isolation** است. جدا کردن هر service در یک VM می‌ تواند از نظر مدیریت خوب باشد ، اما اگر تعداد service ها زیاد باشد مصرف resource و هزینه‌ ی cloud نیز افزایش پیدا می‌ کند.

این مشکلات برای سیستم‌ های کوچک الزاماً جدی نیستند ، اما هرچه تعداد service ها بیشتر شود ، زمان و هزینه‌ ی نگهداری بیشتر نمایان می‌ شود. اینجا است که container ها به‌ عنوان یک alternative سبک‌ تر اهمیت پیدا می‌ کنند.

---

### Containers

قابلیت Virtual Machine برای جدا سازی یک operating system کامل بسیار مناسب است ، اما گاهی لازم نیست برای اجرای یک application کل operating system دیگری را boot کنیم. **Container** برای چنین شرایطی یک محیط محدود تر و سبک‌ تر فراهم می‌ کند. ریشه‌ ی این ایده را می‌ توان در تکنیک‌ های قدیمی‌ تر مانند `()chroot` و resource limitation دید. با `()chroot` می‌توان root directory متفاوت برای یک process تعیین کرد تا process در حالت عادی محیط filesystem دیگری را به‌ عنوان root ببیند مثلاً ```var/spool/my_service/``` ، می‌ تواند برای یک service به‌ عنوان root جدید انتخاب شود به همین دلیل اصطلاح **chroot jail** نیز رایج است. Kernel همچنین قابلیت‌ هایی برای محدود کردن منابع process دارد یکی از این‌ ها **rlimit** است که می‌ تواند محدودیت‌ هایی روی resource هایی مانند CPU time یا اندازه‌ی file اعمال کند. Container از ترکیب همین ایده‌ ها و feature های مختلف kernel شکل می‌ گیرد. در یک تعریف کلی container یک **restricted runtime environment** برای مجموعه‌ ای از process هاست ، به‌ گونه‌ ای که این process ها نتوانند آزادانه به بخش‌ های خارج از محیط خود دسترسی داشته باشند. این نوع virtualization معمولاً **operating system-level virtualization** نامیده می‌ شود. نکته‌ ی مهم این است که یک machine که چند container روی آن اجرا می‌ شوند همچنان فقط **یک Linux kernel زیرین** دارد یعنی :

```text
Host
  |
  +-- Container A
  |
  +-- Container B
  |
  +-- Container C
  |
  +-- One Linux Kernel
```

هر container می‌ تواند user-space متفاوتی داشته باشد ، حتی اگر distribution داخل آن با distribution سیستم اصلی فرق کند ، اما kernel جداگانه‌ ای ندارد. Isolation در container از ترکیب چند قابلیت kernel ساخته می‌ شود. process های داخل container می‌ توانند ویژگی‌ هایی مانند cgroup ها و device ها و filesystem های مخصوص و عدم مشاهده یا تعامل مستقیم با process های خارج از container همچنین network interface های جداگانه داشته باشند. جمع کردن همه‌ ی این محدودیت‌ ها به‌ صورت دستی دشوار است به‌ خصوص مدیریت cgroup ها و namespace ها در سطح پایین کار ساده‌ ای نیست. به همین دلیل ابزارهایی مانند **Docker** و **LXC** برای ساخت و مدیریت container ها توسعه داده شده‌ اند.

#### 🔹 Docker, Podman, and Privileges

در این فصل Docker به‌ عنوان ابزار اصلی مثال‌ ها استفاده می‌ شود ، اما **Podman** نیز معرفی می‌ شود. تفاوت مهم این دو ابزار در مدل اجرای آن‌ هاست. docker معمولاً به یک server process نیاز دارد و در معماری متداول خود از ```dockerd``` استفاده می‌ کند. در بسیاری از configuration های docker ، دسترسی به feature های kernel مورد نیاز container ها با privilege های بالاتر انجام می‌ شود و `dockerd` بخش زیادی از این کار را بر عهده دارد. podman می‌ تواند بدون server مرکزی اجرا شود و یکی از قابلیت‌ های مهم آن **rootless operation** است یعنی کاربر معمولی می‌ تواند container را بدون اجرای آن با privilege های root مدیریت کند. podman همچنین می‌تواند به‌صورت root اجرا شود و در آن حالت از برخی تکنیک‌های isolation مشابه docker استفاده کند از طرف دیگر، نسخه‌ های جدید تر docker نیز می‌ توانند rootless mode را پشتیبانی کنند. از نظر command-line هم podman تا حد زیادی با docker سازگار است. بنابراین بسیاری از مثال‌ های docker را می‌ توان با جایگزین کردن ```docker``` با ```podman``` اجرا کرد. البته implementation آن‌ ها یکسان نیست و به‌ خصوص در rootless mode تفاوت‌ هایی در نحوه‌ ی isolation وجود دارد.

#### 🔹 A Docker Example

برای درک container بهتر است مستقیماً با یک نمونه‌ ی واقعی کار کنیم. اول باید تفاوت **image** و **container** را روشن کنیم. یک **image** شامل filesystem و اطلاعات لازم برای ایجاد container است. معمولاً image از یک image آماده که از یک registry دریافت شده شروع می‌ شود. یک مدل ساده برای فکر کردن درباره‌ ی آن این است :

```text
Image
  ↓
Container
  ↓
Processes
```

برای درک container بهتر است مستقیماً با یک نمونه‌ ی واقعی کار کنیم. اول باید تفاوت **image** و **container** را روشن کنیم. یک **image** شامل filesystem و اطلاعات لازم برای ایجاد container است. معمولاً image از یک image آماده که از یک registry دریافت شده شروع می‌ شود.  در مدل ساده زیر process ها داخل image اجرا نمی‌ شوند بلکه داخل container اجرا می‌ شوند. این مدل کاملاً دقیق نیست ، اما برای شروع مفید است. به‌ خصوص باید به یاد داشت تغییراتی که هنگام اجرای یک container روی filesystem ایجاد می‌ کنید ، image اصلی را تغییر نمی‌ دهند.

#### 🔹 Dockerfile

در مثال کتاب ، یک image ساده بر پایه‌ ی Alpine ساخته می‌ شود. فایل `Dockerfile` شامل این دستورهاست :

```dockerfile
FROM alpine:latest
RUN apk add bash
CMD ["/bin/bash"]
```

 در این مثال ```FROM``` کارش image پایه را مشخص می‌ کند و ```RUN``` دستوری است که هنگام **build کردن image** اجرا می‌ شود و ```CMD``` هم رفتار پیش‌ فرضی را تعیین می‌ کند که هنگام **اجرای container** مورد استفاده قرار می‌گیرد. برای ساخت image :

```bash
docker build -t hlw_test .
```

در این command هم ```t hlw_test-``` یک tag برای image تعیین می‌کند و ```.``` به directory جاری به‌ عنوان build context اشاره دارد.

#### 🔹 Intermediate Images

در زمان build ، Docker معمولاً از image های مرحله‌ ای عبور می‌ کند در مثال ```FROM alpine:latest``` ابتدا image پایه‌ ی Alpine دریافت می‌ شود و Docker با digest های SHA256 و identifier های کوتاه‌ تر، اجزای مختلف image را دنبال می‌ کند و image که هنوز محصول نهایی نیست و قرار است لایه‌ های بیشتری روی آن ساخته شود ، **intermediate image** نام دارد.

#### 🔹 RUN and Intermediate Containers

در مرحله‌ ی بعد Docker یک **intermediate container** ایجاد می‌کند و command `apk add bash` را داخل آن اجرا می‌ کند. این موضوع مهم است ، چون build یک Docker image همیشه به معنی اجرای command ها مستقیماً روی host نیست.

```dockerfile
RUN apk add bash
```

مدل ساده‌ شده‌ ی این مرحله چنین است :

```text
Base Image
    |
    v
Temporary Container
    |
    v
RUN command
    |
    v
New Intermediate Image
```

پس دو مفهوم را باید از هم جدا کنیم  ، **Intermediate Image** بعد از build step می‌ تواند باقی بماند و مبنای لایه‌ ی بعدی باشد و **Intermediate Container** هم container  موقتی است که برای اجرای یک build step ایجاد می‌ شود و بعد از اتمام کار حذف می‌ شود.

#### 🔹 CMD

بعد از اجرای `RUN`، نوبت :

```dockerfile
CMD ["/bin/bash"]
```

است. این command در زمان build هم `bin/bash/` را اجرا نمی‌ کند. `CMD` به **container runtime** می‌ گوید وقتی container از image ساخته‌ شده اجرا شد ، command پیش‌ فرض چه باشد به همین دلیل `RUN` و `CMD` نقش کاملاً متفاوتی دارند در پایان build هم image نهایی یک ID دارد و tag نیز می‌ گیرد با دستور ```hlw_test``` یا ```hlw_test:latest ``` و برای مشاهده‌ ی image ها با دستور ```docker images``` استفاده می‌ شود.

#### 🔹 Running Docker Containers

حالا می‌ توان از image ساخته‌ شده یک container اجرا کرد :
```bash
docker run -it hlw_test
```

گزینه‌ی `i-` محیط را interactive نگه می‌ دارد و `t-` یک terminal متصل می‌ کند. اگر این گزینه‌ ها حذف شوند ، در سناریوی این مثال ممکن است shell شما prompt دریافت نکند و container تقریباً بلافاصله terminate شود در نتیجه :

```text
Image
   |
   v
docker run
   |
   v
Container
   |
   v
/bin/bash
```

در shell ایجاد شده در این مثال ، process با privilege مربوط به root داخل container اجرا می‌ شود.

#### 🔹 Inspecting Processes Inside the Container

بعد از ورود به container ، با command هایی مانند `mount` و `ps` می‌ توان محیط را بررسی کرد. اگر process های container را مشاهده کنید ، ممکن است چیزی شبیه این ببینید :

```text
PID USER COMMAND
1   root /bin/bash
6   root ps aux
```

در یک Linux system معمولی ، PID 1 متعلق به `init` یا process اصلی system است ، اما در این container ، shell شما PID 1 را دارد که دلیل این رفتار استفاده از **PID namespace** است. یک process می‌ تواند namespace جدیدی برای PID ها ایجاد کند که از داخل آن ، process های container شماره‌ گذاری مخصوص خودشان را داشته باشند و از process های خارج از namespace اطلاع نداشته باشند. از طرف دیگر، همان process ها در host همچنان process واقعی سیستم هستند و می‌ توان آن‌ ها را در process list host نیز پیدا کرد. فقط PID که host برای آن‌ ها می‌ بیند با PID که container می‌ بیند یکسان نیست.


#### 🔹 Overlay Filesystems

فایل سیستم داخل container نیز با یک Linux system معمولی تفاوت مهمی دارد. در Docker معمولاً می‌ توان یک **overlay filesystem** را مشاهده کرد. در این مدل ، چند directory به‌ عنوان layer روی هم قرار می‌ گیرند و تغییرات در لایه‌ ی بالایی نگهداری می‌ شوند ، در output مربوط به mount معمولاً با سه مفهوم مهم lowerdir و upperdir و workdir ، رو به‌ رو می‌ شوید .

#### 🔹 lowerdir

مجموعه‌ ای از directory های پایه است که layer های پایین‌ تر filesystem را تشکیل می‌ دهند. این مسیر می‌ تواند چند directory جدا داشته باشد که به‌ ترتیب روی هم stack می‌ شوند.

#### 🔹 upperdir

لایه‌ ی بالایی است و تغییرات جدید filesystem در آن ظاهر می‌ شوند. برای container معمولاً این همان جایی است که تغییرات runtime به‌ صورت writable ثبت می‌ شوند.

#### 🔹 workdir

برای عملیات داخلی filesystem driver استفاده می‌ شود و قبل از mount باید وضعیت مناسب داشته باشد در نتیجه ، ساختار کلی overlay را می‌ توان چنین تصور کرد :

```text
lower layer
lower layer
lower layer
-------------
upper writable layer
```

این مدل باعث می‌ شود چند container یا image بتوانند بخش‌ های مشترک پایینی را share کنند. اما image هایی که build step های زیادی دارند می‌ توانند layer های زیادی نیز ایجاد کنند. به همین دلیل strategy هایی مانند ترکیب بعضی `RUN` command ها یا **multistage build** می‌ توانند تعداد layer ها را کاهش دهند. در rootless mode هم Podman ممکن است از نسخه‌ ی FUSE-based مربوط به overlay استفاده کند و اطلاعات mount متفاوتی روی host نشان دهد. در آن وضعیت می‌ توان process های `fuse-overlayfs` را بررسی کرد.

#### 🔹 Networking

معمولاً container ها می‌توانند در بعضی configuration ها از network host استفاده کنند ، اما حالت رایج‌ تر ایجاد یک network namespace جدا است. Docker به‌ صورت معمول از **bridge network** استفاده می‌ کند. ابتدا interface مانند ```docker0``` روی host ایجاد می‌ شود که معمولاً در یک private subnet قرار دارد بعد برای container یک **network namespace** تازه ساخته می‌ شود در ابتدا این namespace تقریباً خالی است و حداقل یک loopback interface مانند ```lo``` دارد سپس Docker یک جفت virtual interface ایجاد می‌ کند که دو طرف یک ارتباط مجازی را تشکیل می‌ دهند. یک طرف روی host باقی می‌ ماند و طرف دیگر وارد network namespace مربوط به container می‌شود. در container می‌توانید interface مانند `eth0` ببینید ، در حالی که host نیز ممکن است interface با نام دیگری یا حتی مشابه داشته باشد چون interface ها در namespace های جدا قرار دارند. این container معمولاً روی subnet خصوصی Docker قرار می‌ گیرد. اگر قرار باشد container به شبکه‌ ی بیرونی دسترسی داشته باشد ، host باید traffic آن را با **NAT** به شبکه‌ ی خارج منتقل کند پس مسیر کلی چنین است :

```text
Container
    |
    v
Network Namespace
    |
    v
Virtual Interface Pair
    |
    v
docker0
    |
    v
NAT
    |
    v
External Network
```

این معماری همان چیزی است که باعث می‌ شود container هم isolation شبکه داشته باشد و هم بتواند به بیرون دسترسی پیدا کند. یک نکته‌ی عملی این است که private subnet مورد استفاده‌ ی Docker ممکن است با subnet که router یا تجهیزات شبکه‌ ی دیگر به کار می‌ برند تداخل داشته باشد بنابراین هنگام troubleshooting networking باید به address range ها توجه کرد. در **rootless Podman** ساخت network interface مجازی متفاوت است ، زیرا ایجاد interface های مورد نیاز معمولاً privilege بالاتری می‌ خواهد. Podman در این حالت می‌ تواند از network namespace جدید به‌ همراه **TAP interface** و ابزار forwarding مانند `slirp4netns` استفاده کند. این روش محدودیت‌هایی نسبت به مدل معمول Docker دارد برای مثال ارتباط مستقیم container ها با یکدیگر به همان شکل وجود ندارد. موضوعاتی مانند port exposure و routing جزئیات بیشتری دارند ، اما مهم‌ ترین چیزی که باید از این بخش باقی بماند ، شناخت topology شبکه و نقش network namespace ، virtual interface ، bridge و NAT است.

<img width="100%" height="427" alt="image" src="https://github.com/user-attachments/assets/d812f541-dffa-436e-a046-8f78247aaf77" />

#### 🔹 Docker Operation

همچنین Docker علاوه بر ایجاد isolation ، باید lifecycle تمام resource های مربوط به container را نیز مدیریت کند. Docker یک container را تا زمانی **running** در نظر می‌ گیرد که حداقل یک process در آن در حال اجرا باشد برای دیدن container های در حال اجرا دستور ```docker ps``` استفاده می‌ شود. وقتی تمام process های container terminate شوند ، Docker container را از بین نمی‌ برد بلکه آن را به حالت exited منتقل می‌کند مگر اینکه container را با گزینه‌ ی ```rm--``` اجرا کرده باشید. پس از توقف ، container می‌ تواند تغییرات filesystem خود را نیز نگه دارد و می‌توان filesystem آن را با ابزارهایی مانند ```docker export``` دریافت کرد.

نکته‌ی مهم اینجاست که ```docker ps``` به‌ صورت پیش‌ فرض فقط container های running را نشان می‌ دهد و برای مشاهده‌ی exited container ها نیز باید از ```docker ps -a``` استفاده کرد. این موضوع از نظر storage مهم است ، ممکن است تعداد زیادی container متوقف‌ شده روی سیستم باقی بمانند و اگر application داخل آن‌ ها داده‌ ی زیادی تولید کرده باشد ، disk space به‌ مرور مصرف شود. برای حذف container تمام‌ شده می‌ توان از ```<docker rm container>``` استفاده کرد.

#### 🔹 Docker Images

همین مسئله درباره‌ی image ها نیز وجود دارد. ساخت image معمولاً یک فرآیند تکراری است. اگر یک image جدید با همان tag قبلی ساخته شود ، Docker image قدیمی را الزاماً حذف نمی‌کند و ممکن است tag از image قبلی جدا شود و آن image بدون tag باقی بماند برای مشاهده‌ ی image ها با دستور ```docker images``` استفاده می‌ شود و ممکن است در خروجی image هایی به صورت زیر ببینید :

```text
REPOSITORY   <none>
TAG          <none>
```

این image ها می‌ توانند به‌ مرور فضای زیادی مصرف کنند برای حذف image از دستور ```<docker rmi image>``` استفاده می‌ شود. در بسیاری از موارد حذف image می‌ تواند intermediate image هایی را که دیگر مورد نیاز نیستند نیز حذف کند بنابراین مدیریت Docker فقط ساخت و اجرای container نیست بلکه image ها ، container های متوقف‌ شده و layer های قدیمی نیز بخشی از operational maintenance هستند.

#### 🔹 Docker Service Process Models

یکی از نکات ظریف container ها به **lifecycle process ها** مربوط می‌ شود. وقتی یک child process terminate می‌ شود ، parent باید status خروج آن را با `()wait` جمع‌ آوری یا به‌ اصطلاح **reap** کند. اگر این کار انجام نشود ، process های مرده می‌ توانند در سیستم به شکل **zombie process** باقی بمانند. در container نیز ممکن است همین مسئله اتفاق بیفتد مخصوصاً وقتی process tree پیچیده باشد و process با PID 1 داخل container به‌ صورت مستقیم از child process ها آگاه نباشد. این موضوع به یک تصور اشتباه منجر شده است که گویی نباید چند process یا service را داخل یک container اجرا کرد. کتاب صراحتاً تأکید می‌ کند که داشتن چند process داخل container ذاتاً مشکل نیست. مسئله اصلی این است که process parent باید lifecycle child process های خود را درست مدیریت کند. اگر یک service ساده دارید که process های دیگری ایجاد می‌ کند و در پایان container process های باقی‌مانده می‌ بینید ، Docker گزینه‌ ای به نام ```docker run --init``` ارائه می‌ دهد. این option یک init بسیار ساده را به‌ عنوان PID 1 در container اجرا می‌ کند تا در reap کردن child process ها کمک کند. اگر container شما چند service یا task مختلف دارد ، می‌ توان به‌ جای یک startup script ساده از process manager هایی مانند **Supervisor (`supervisord`)** برای start و monitor کردن process ها استفاده کرد.

#### 🔹 LXC

یکی از قدیمی‌ ترین سیستم‌ های Linux container هم **LXC**  است. حتی نسخه‌ های اولیه‌ ی Docker نیز بر پایه‌ ی LXC ساخته شده بودند. اصطلاح LXC گاهی برای مجموعه‌ ی kernel feature هایی که container ها را ممکن می‌ کنند استفاده می‌ شود ، اما معمولاً منظور از LXC یک library  و مجموعه‌ ای از utility ها برای ایجاد و مدیریت Linux container هاست. تفاوت مهم LXC با Docker در میزان abstraction و مقدار setup دستی است. Docker بسیاری از جزئیات را پنهان می‌ کند و با image های آماده کار را سریع می‌ کند ، اما LXC کنترل بیشتری در اختیار کاربر قرار می‌ دهد و در نتیجه ممکن است نیاز باشد برخی اجزای محیط را خودتان تنظیم کنید. برای مثال در LXC ممکن است خودتان مسئول ایجاد network interface مناسب یا تنظیم **user ID mapping** باشید. از ابتدا LXC تا حد زیادی با این هدف طراحی شده بود که یک Linux system نسبتاً کامل داخل container داشته باشید ، حتی تا سطح `init` در نتیجه می‌ توانید یک distribution مخصوص container نصب کنید و software های مورد نیاز خود را داخل آن قرار دهید. LXC به‌ صورت پیش‌ فرض از همان overlay filesystem که در مثال Docker دیدیم استفاده نمی‌ کند ، هرچند امکان اضافه کردن آن وجود دارد همچنین LXC یک C API در اختیار می‌ گذارد و همین موضوع باعث می‌ شود برنامه‌ ی دیگری بتواند در سطح پایین‌ تر مستقیماً با functionality آن کار کند. در کنار LXC ، پروژه‌ ی **LXD** برای ساده‌ تر کردن بعضی قسمت‌ های management ارائه شده است. LXD می‌ تواند کارهایی مانند network creation و image management را ساده‌ تر کند و علاوه بر C API ، یک **REST API** نیز در اختیار قرار دهد در نتیجه می‌ توان گفت Docker بیشتر روی ساده‌ سازی workflow و image-based operation تأکید دارد ، در حالی که LXC کنترل و انعطاف بیشتری در سطح پایین در اختیار کاربر قرار می‌ دهد.

#### 🔹 Kubernetes

وقتی Container ها تعدادشان زیاد شود ، به‌ سرعت به یک مسئله‌ی management تبدیل می‌ شوند. فرض کنید تعداد زیادی container روی چند machine دارید. دیگر فقط اجرای یک container کافی نیست باید مواردی را مدیریت کنید ، 

- مشخص شود کدام machine توانایی اجرای container را دارد.
- بعد container ها start شوند.
- وضعیتشان monitor شود.
- در صورت failure دوباره restart شوند.
- بعد startup آن‌ ها پیکربندی شود.
- بعد network آن‌ ها تنظیم شود.
- نسخه‌ های جدید image منتشر و به‌ صورت کنترل‌ شده روی container های در حال اجرا اعمال شوند. 

همین مسئله باعث ظهور سیستم‌ های **container orchestration** شد ، یکی از مهم‌ ترین نمونه‌ ها **Kubernetes** است. Kubernetes برای مدیریت مجموعه‌ ای از container ها روی چند machine طراحی شده و از نظر مفهومی می‌ توان آن را به دو بخش کلی server و client تقسیم کرد. در سمت server  هم machine هایی قرار دارند که container ها روی آن‌ ها اجرا می‌ شوند. در سمت client مجموعه‌ ای از command-line utility ها و configuration ها قرار دارند که برای ایجاد و مدیریت گروه‌ های container استفاده می‌ شوند. Configuration های Kubernetes می‌ توانند بسیار بزرگ و پیچیده شوند و بخش قابل‌ توجهی از کار عملی Kubernetes در نوشتن configuration صحیح برای workload هاست. برای آزمایش Kubernetes بدون راه‌ اندازی کامل server infrastructure می‌ توان از **Minikube** استفاده کرد که یک Kubernetes cluster را در قالب یک Virtual Machine روی سیستم شخصی ایجاد می‌ کند. Kubernetes در این فصل بیشتر به‌ عنوان مرحله‌ ی بعدی مدیریت container ها معرفی می‌ شود یعنی زمانی که دیگر مسئله فقط چطور یک container را اجرا کنم؟ نیست ، بلکه چطور تعداد زیادی container را روی چند machine مدیریت کنم؟ مطرح است.

#### 🔹 Pitfalls of Containers

قابلیت Container ها بسیاری از مشکلات VM را ساده می‌ کنند ، اما مشکل را به‌ طور کامل حذف نمی‌ کنند. در هر container infrastructure همچنان به حداقل یک یا چند machine نیاز دارید و آن machine باید یک Linux system کامل باشد چه روی hardware واقعی باشد و چه خودش یک Virtual Machine باشد بنابراین هنوز هزینه‌ ی infrastructure ، hardware ، hosting و maintenance وجود دارد. اگر infrastructure را خودتان مدیریت کنید ، هزینه‌ ی عملیاتی و زمانی بالاست. اگر از container service یا managed Kubernetes استفاده کنید ، همان هزینه در قالب هزینه‌ ی monetary به provider منتقل می‌ شود.

#### 🔹 Storage Overhead

معمولاً Container ها سبک‌ تر از VM هستند ، اما application داخل container همچنان به library ها و بخش‌ هایی از user-space یک Linux system نیاز دارد در نتیجه اگر base image مناسب انتخاب نشود ، image ها می‌ توانند بزرگ شوند. Overlay filesystem می‌ تواند وقتی چند container از base layer های مشترک استفاده می‌ کنند ، مقداری از این هزینه را کاهش دهد چون فایل‌ های مشترک لازم نیست برای هر container به‌ صورت مستقل ذخیره شوند اما اگر application هنگام runtime داده‌ ی زیادی تولید کند ، writable upper layer هر container نیز می‌ تواند رشد کند.

#### 🔹 Resource Usage

معنای Container isolation به این معنی نیست که CPU و سایر resource ها بی‌نهایت می‌ شوند. همچنان یک kernel و یک مجموعه‌ ی زیرین از resources وجود دارد و می‌توانید برای container ها limit تعریف کنید ، اما ظرفیت machine واقعی همچنان محدود است. اگر workload بیش از توان host باشد ، هم container ها و هم سیستم زیرین می‌ توانند تحت فشار قرار بگیرند.

#### 🔹 Runtime Data and Logs

یک مسئله‌ ی مهم دیگر محل نگهداری data است. در سیستم‌ هایی که overlay filesystem دارند ، تغییراتی که application روی filesystem container در زمان runtime ایجاد می‌ کند ممکن است با lifecycle همان container مرتبط باشند و پس از terminate شدن process ها دیگر در آن شکل باقی نمانند. در بسیاری از application ها ، user data داخل database ذخیره می‌ شود و مدیریت data تا حدی به database administration منتقل می‌ شود. اما **logs** همچنان مسئله‌ ی مهمی هستند. یک server application برای troubleshooting و operation به log نیاز دارد و در محیط‌ های بزرگ ، معمولاً باید log ها در یک سیستم جداگانه نگهداری و مدیریت شوند.

#### 🔹 Container-Centric Operations

بخش قابل‌ توجهی از container tooling و operational model ها برای web server ها طراحی شده‌ اند. در web  workloads ، ابزارها و documentation فراوانی برای اجرای container وجود دارد و سیستم‌ هایی مانند Kubernetes قابلیت‌ های زیادی برای جلوگیری از runaway workload ها ارائه می‌ دهند. اما اگر بخواهید service متفاوتی را در همان مدل قرار دهید ، ممکن است بعضی abstraction ها دقیقاً با نیاز واقعی آن service هماهنگ نباشند.

#### 🔹 Configuration Problems

معمولاً Container یک محیط ایزوله ایجاد می‌ کند ، اما شما را از configuration mistake نجات نمی‌ دهد. یک container می‌ تواند به‌ دلیل configuration نادرست ، dependency اشتباه یا design ضعیف دچار مشکل شود. حتی اگر مجبور نباشید با بخش‌ هایی از یک Linux system کامل مانند `systemd` سروکار داشته باشید ، اجزای زیادی همچنان وجود دارند که می‌ توانند اشتباه پیکربندی شوند. یکی از خطا های رایج هنگام troubleshooting این است که کاربر بدون درک علت مشکل ، چیزهای بیشتری به environment اضافه می‌ کند تا موقتاً مشکل برطرف شود. این کار می‌ تواند environment را بسیار پیچیده و شکننده کند بهتر است هر تغییر را با درک دقیق اثر آن انجام دهید.

#### 🔹 Versioning Problems

در مثال‌ های کتاب از tag به نام ```latest``` استفاده می‌ شود ، اما این روش یک ریسک مهم دارد. وقتی image جدیدی با `latest` منتشر می‌ شود ، ممکن است چیزی در base distribution یا dependency های زیرین تغییر کرده باشد و application شما دیگر همان رفتار قبلی را نداشته باشد. برای جلوگیری از وابستگی غیرقابل‌ پیش‌ بینی به تغییرات آینده یکی از روش‌ های معمول استفاده از **version tag**  مشخص برای base image است در نتیجه این تفاوت مهم است ```latest``` یک reference متحرک است و ممکن است به نسخه‌ ی دیگری در آینده اشاره کند در حالی که یک version مشخص ، build environment قابل‌ پیش‌ بینی‌ تری ایجاد می‌ کند.

#### 🔹 Trust and Image Sources

موضوع دیگری که نباید فراموش شود **trust** است. وقتی container خود را بر پایه‌ ی image موجود در یک public repository می‌ سازید ، در واقع به یک لایه‌ ی اضافی از software distribution اعتماد می‌ کنید. باید فرض کنید که imageی که دریافت می‌ کنید همواره همان چیزی است که انتظار دارید و همچنین در آینده در دسترس باقی می‌ ماند. این یکی از تفاوت‌ های مهمی است که کتاب میان مدل image-based Docker و رویکرد LXC مطرح می‌ کند ، در LXC کاربر بیشتر به سمت ساخت و کنترل محیط خودش هدایت می‌ شود. در نهایت ، این محدودیت‌ ها به معنی بد بودن container نیستند. تقریباً هر مدل دیگری نیز شکل دیگری از همین مشکلات را دارد. container قرار نیست همه‌ ی مسائل infrastructure را حل کند. اگر application روی یک Linux system عادی زمان زیادی برای startup نیاز داشته باشد ، container کردن آن به‌ خودی‌ خود startup را جادویی سریع نمی‌ کند. container فقط layer های خاصی از محیط را ساده‌ تر و قابل‌ کنترل‌ تر می‌ کند.

---

### Runtime-Based Virtualization

نوع دیگری از virtualization در سطح **application runtime** اتفاق می‌ افتد. در این مدل برخلاف VM و container ، هدف ایجاد یک machine یا system environment جداگانه نیست. هدف ، جدا کردن environment مربوط به یک application خاص است. این موضوع زمانی اهمیت پیدا می‌ کند که چند application از یک programming language و package های مشترک استفاده می‌ کنند. زبان Python مثال خوبی برای این مسئله است. ممکن است خود Linux distribution از Python استفاده کند و package های مختلفی نیز روی آن نصب شده باشد. اگر application شما به version متفاوتی از یکی از این package ها نیاز داشته باشد ، استفاده‌ ی مستقیم از system Python می‌ تواند باعث conflict شود و راه‌ حل Python ، ایجاد یک **virtual environment** است که ابتدا یک directory برای environment ایجاد می‌ کنید :

```bash
python3 -m venv test-venv
```

در این directory می‌ توانید ساختاری شبیه یک environment مستقل ببینید :
```text
test-venv/
├── bin/
├── include/
└── lib/
```

این محیط به شما اجازه می‌ دهد package library جدا گانه‌ ای داشته باشید بدون اینکه packageهای system Python را تغییر دهید.

#### 🔹 Activating a Virtual Environment

برای فعال کردن محیط ، باید script زیر را **source** کنید :

```bash
. test-venv/bin/activate
```

این نکته مهم است که script را نباید مثل یک executable معمولی اجرا کنید دلیل آن این است که activation در اصل با تغییر **environment variable های shell فعلی** انجام می‌ شود. اگر script را به‌ صورت process مستقل اجرا کنید ، تغییر environment آن process به shell اصلی منتقل نمی‌ شود. بعد از activation و  command هم `python` به Python مربوط به environment جدید اشاره می‌ کند. در ساختار معمول این environment هم executable مربوطه ممکن است خودش یک symbolic link باشد همچنین environment variable زیر تنظیم می‌ شود :


```text
VIRTUAL_ENV
```
که base directory محیط فعال را مشخص می‌ کند و برای خارج شدن از environment :

```bash
deactivate
```

استفاده می‌ شود. بعد از activation و package هایی که داخل environment نصب می‌ کنید به library جدا گانه‌ ی آن environment می‌روند ، نه library اصلی system در نتیجه چند پروژه می‌ توانند dependency های متفاوتی داشته باشند بدون اینکه package های یکدیگر را خراب کنند. اینجا اصطلاح **virtualization** معنای متفاوتی نسبت به VM و container دارد. دیگر کل machine یا operating system را جدا نکرده‌ ایم فقط environment لازم برای یک application یا programming language را از سایر application ها جدا کرده‌ ایم.

---

### Summary

فصل هفدهم سه سطح متفاوت از virtualization را بررسی می‌ کند :

در **Virtual Machine** یک machine کامل و یک operating system مستقل داخل محیط مجازی اجرا می‌ شود و Hypervisor منابع hardware را مدیریت می‌ کند و بسته به نوع آن می‌ تواند Type 1 یا Type 2 باشد. VM ها برای   testing و application compatibility و cloud workloads بسیار مناسب‌اند ، اما به‌ دلیل داشتن operating system کامل ، هزینه‌ ی بیشتری برای startup ، resource و maintenance دارند. 

در **Container** دیگر kernel جداگانه‌ ای برای هر environment وجود ندارد. چند container روی یک Linux kernel مشترک اجرا می‌ شوند و با ترکیب قابلیت‌ هایی مانند **namespaces و cgroups و filesystem isolation و network isolation** محیط‌ های جداگانه برای process ها ساخته می‌ شود. Docker و Podman workflow این کار را ساده می‌ کنند و LXC کنترل بیشتری در سطح پایین ارائه می‌ دهد. وقتی تعداد container ها زیاد شود ، Kubernetes مسئله‌ ی orchestration را مدیریت می‌ کند.

در **Runtime-Based Virtualization** ، جدا سازی حتی محدود تر می‌ شود و فقط environment مربوط به یک application یا programming language مستقل می‌ گردد و `venv`  در Python نمونه‌ ی مشخص آن است.

بنابراین virtualization در Linux یک تکنولوژی واحد نیست بلکه مجموعه‌ ای از روش‌ ها برای ایجاد **isolation و کنترل منابع و جدا سازی محیط‌ های اجرا** است و هر سطح ، مقدار متفاوتی از isolation و overhead ایجاد می‌ کند.
