## مجازی سازی
### 🐧 فصل هفدهم کتاب How Linux Works

مجازی‌سازی در Linux فقط به ساختن یک **Virtual Machine** محدود نمی‌شود. در ساده‌ترین نگاه، virtualization یعنی ایجاد یک لایه‌ی میانی که منابع یا محیط زیرین را به‌شکل دیگری در اختیار یک یا چند مصرف‌کننده قرار می‌دهد. نمونه‌ای که پیش‌تر با آن آشنا شدیم **virtual memory** است؛ هر process می‌تواند فضای حافظه‌ی بزرگی را طوری ببیند که انگار متعلق به خودش است، در حالی که زیر آن یک سیستم مدیریت‌شده‌ی مشترک وجود دارد.

در این فصل همین ایده را در سطوح مختلف دنبال می‌کنیم: ابتدا **Virtual Machine** که یک سیستم‌عامل کامل را اجرا می‌کند، سپس **Container** که محیط اجرای مجموعه‌ای از processها را محدود و جدا می‌کند، و در پایان **Runtime-Based Virtualization** که جداسازی را در سطح محیط اجرای یک application انجام می‌دهد.

تمرکز فصل بیشتر روی درک مفاهیم، اصطلاحات و اجزای اصلی است تا جزئیات بسیار پایین‌سطح implementation.

---
📚 Table of Contents

- [17.1 Virtual Machines](#171-virtual-machines)
  - [17.1.1 Hypervisors](#1711-hypervisors)
    - [Type 2 Hypervisor](#type-2-hypervisor)
    - [Type 1 Hypervisor](#type-1-hypervisor)
  - [17.1.2 Hardware in a Virtual Machine](#1712-hardware-in-a-virtual-machine)
    - [Virtual Machine CPU Modes](#virtual-machine-cpu-modes)
  - [17.1.3 Common Uses of Virtual Machines](#1713-common-uses-of-virtual-machines)
    - [Testing and Trials](#testing-and-trials)
    - [Application Compatibility](#application-compatibility)
    - [Servers and Cloud Services](#servers-and-cloud-services)
  - [17.1.4 Drawbacks of Virtual Machines](#1714-drawbacks-of-virtual-machines)
- [17.2 Containers](#172-containers)
  - [17.2.1 Docker, Podman, and Privileges](#1721-docker-podman-and-privileges)
  - [17.2.2 A Docker Example](#1722-a-docker-example)
    - [Dockerfile](#dockerfile)
    - [Intermediate Images](#intermediate-images)
    - [RUN and Intermediate Containers](#run-and-intermediate-containers)
    - [CMD](#cmd)
    - [Running Docker Containers](#running-docker-containers)
    - [Inspecting Processes Inside the Container](#inspecting-processes-inside-the-container)
    - [Overlay Filesystems](#overlay-filesystems)
    - [`lowerdir`](#lowerdir)
    - [`upperdir`](#upperdir)
    - [`workdir`](#workdir)
    - [Networking](#networking)
    - [Docker Operation](#docker-operation)
    - [Docker Images](#docker-images)
    - [Docker Service Process Models](#docker-service-process-models)
  - [17.2.3 LXC](#1723-lxc)
  - [17.2.4 Kubernetes](#1724-kubernetes)
  - [17.2.5 Pitfalls of Containers](#1725-pitfalls-of-containers)
    - [Storage Overhead](#storage-overhead)
    - [Resource Usage](#resource-usage)
    - [Runtime Data and Logs](#runtime-data-and-logs)
    - [Container-Centric Operations](#container-centric-operations)
    - [Configuration Problems](#configuration-problems)
    - [Versioning Problems](#versioning-problems)
    - [Trust and Image Sources](#trust-and-image-sources)
- [17.3 Runtime-Based Virtualization](#173-runtime-based-virtualization)
  - [Activating a Virtual Environment](#activating-a-virtual-environment)
- [Summary](#summary)

---

### 17.1 Virtual Machines

**Virtual Machine** را می‌توان نسخه‌ی مجازی‌شده‌ی یک کامپیوتر در نظر گرفت. در این مدل، به کمک software یک machine جدید ایجاد می‌شود که می‌تواند processor، memory، I/O interface و سایر اجزای موردنیاز را داشته باشد.

تفاوت مهم آن با یک process معمولی این است که در یک **system virtual machine**، یک operating system کامل داخل VM اجرا می‌شود و آن operating system، kernel خودش را نیز دارد.

به‌همین دلیل guest درون VM می‌تواند همانند یک سیستم واقعی با hardware مجازی‌شده کار کند.

این مفهوم قدمت زیادی دارد. برای نمونه، mainframeهای IBM سال‌ها از system virtual machine برای ایجاد محیط‌های جداگانه و multiuser استفاده کرده‌اند.

یک VM می‌تواند کاملاً توسط software شبیه‌سازی شود که در این صورت معمولاً با مفهوم **emulator** روبه‌رو هستیم، یا می‌تواند تا جای ممکن از hardware واقعی سیستم میزبان استفاده کند. روش دوم به‌خاطر performance بهتر برای کاربردهای معمول Linux مهم‌تر است.

در عمل، ابزارهایی مانند VirtualBox ایجاد و اجرای VM را ساده می‌کنند و حتی می‌توان فرآیند مدیریت VM را با ابزارهای command-line یا interfaceهای ابری automation کرد. بنابراین برای کاربر معمولی، اهمیت اصلی درک مدل VM و اصطلاحات آن است، نه پیاده‌سازی جزئی آن.

#### 17.1.1 Hypervisors

نرم‌افزاری که مسئول مدیریت یک یا چند Virtual Machine است، **Hypervisor** یا **Virtual Machine Monitor (VMM)** نام دارد.

Hypervisor از نظر نقش شبیه یک لایه‌ی مدیریتی است که منابع سیستم را میان VMها قرار می‌دهد و دسترسی آن‌ها به hardware را کنترل می‌کند.

دو نوع اصلی hypervisor وجود دارد.

##### Type 2 Hypervisor

**Type 2** روی یک operating system معمولی اجرا می‌شود. در این مدل، Linux یا سیستم‌عامل دیگری ابتدا روی hardware اجرا شده و سپس hypervisor در همان محیط اجرا می‌شود.

نمونه‌ی شناخته‌شده‌ی آن:

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

VirtualBox نمونه‌ای از Type 2 hypervisor است. این نوع hypervisor برای desktop و آزمایش سیستم‌های مختلف بسیار قابل‌دسترس است.

##### Type 1 Hypervisor

**Type 1** مستقیماً برای اجرای VMها طراحی شده و بیشتر شبیه یک operating system مخصوص virtualization عمل می‌کند.

ممکن است یک سیستم کمکی مانند Linux برای management در کنار آن قرار بگیرد، اما لایه‌ی اصلی که VMها را اجرا می‌کند خود hypervisor است.

بخش زیادی از زیرساخت cloud بر پایه‌ی همین مدل ساخته شده است. وقتی یک cloud provider به شما instance یک operating system می‌دهد، در عمل یک Virtual Machine را روی زیرساختی اجرا می‌کند که hypervisor نوع ۱ آن را مدیریت می‌کند.

در این مدل، اصطلاحات مهم عبارت‌اند از:

**Guest:** خود Virtual Machine همراه با operating system داخلی آن.

**Host:** سیستمی که hypervisor را اجرا می‌کند. در Type 2 معمولاً همان operating system اصلی شماست؛ در Type 1، host در معنای گسترده‌تر به hypervisor و محیط مدیریتی آن اشاره می‌کند.

#### 17.1.2 Hardware in a Virtual Machine

در نگاه اول می‌توان تصور کرد که hypervisor باید تمام hardware را به‌صورت کامل emulate کند. مثلاً برای ساخت یک virtual disk می‌توان روی host یک فایل بزرگ ایجاد کرد و آن را به guest مانند یک disk device واقعی نشان داد.

این مدل، نوعی **hardware virtualization** کاملاً شبیه‌سازی‌شده است، اما از نظر performance همیشه بهترین روش نیست.

در عمل، یکی از مهم‌ترین تکنیک‌ها استفاده‌ی مستقیم‌تر guest از منابع host است.

به این رویکرد **paravirtualization** گفته می‌شود. در این حالت به‌جای اینکه guest برای همه‌چیز با یک virtual hardware کاملاً شبیه‌سازی‌شده صحبت کند، ممکن است از interfaceها یا driverهایی استفاده کند که ارتباط مستقیم‌تری با hypervisor برقرار می‌کنند.

**Network interface** و **block device** از مهم‌ترین جاهایی هستند که چنین روش‌هایی کاربرد دارند.

برای مثال، در یک cloud environment ممکن است deviceای مانند:

```text
/dev/xvd...
```

نمایان شود که در واقع یک virtual disk وابسته به Xen است و driver مربوطه می‌تواند مستقیماً با hypervisor ارتباط برقرار کند.

Paravirtualization همیشه فقط برای performance نیست. گاهی برای راحت‌تر شدن interaction با host استفاده می‌شود؛ برای مثال هماهنگی mouse میان محیط guest و host در محیط‌هایی مانند VirtualBox.

هدف نهایی این abstraction این است که guest تا حد امکان virtual hardware را مثل hardware عادی ببیند. برای یک Linux guest، بهتر است virtual disk در نهایت یک **block device** معمولی باشد تا بتوان با ابزارهای عادی روی آن partition و filesystem ایجاد کرد.

##### Virtual Machine CPU Modes

برای فهمیدن یکی از مشکلات اصلی virtualization باید تفاوت میان **user mode** و **kernel mode** را به یاد داشته باشیم.

CPU در kernel mode اجازه‌ی اجرای operationهای بیشتری دارد، در حالی که user mode محدودتر است و بعضی instructionها یا دسترسی‌های memory در آن مجاز نیستند.

مشکل Virtual Machineهای اولیه روی معماری x86 این بود که operating system داخل guest انتظار داشت بتواند در **kernel mode** کار کند، در حالی که خود guest در شرایطی مانند user mode اجرا می‌شد.

راه‌حل این بود که hypervisor instructionهای restricted را تشخیص دهد و آن‌ها را **trap** کند. سپس hypervisor آن operation را به‌شکل مناسب emulate می‌کرد تا guest بتواند رفتار موردانتظار kernel mode را داشته باشد.

از آنجا که همه‌ی instructionهای kernel restricted نیستند، بسیاری از operationها مستقیماً اجرا می‌شدند و فقط بخش‌هایی که نیاز به intervention داشتند هزینه‌ی اضافی ایجاد می‌کردند.

بعدها Intel و AMD قابلیت‌های سخت‌افزاری مخصوص virtualization ارائه کردند:

```text
Intel → VT-x
AMD   → AMD-V
```

این featureها به processor اجازه می‌دهند hypervisor در مدیریت guestها کارآمدتر عمل کند و در بعضی محیط‌ها حتی وجود آن‌ها ضروری است.

#### 17.1.3 Common Uses of Virtual Machines

Virtual Machineها در Linux کاربردهای مختلفی دارند و کتاب چند مورد اصلی را برجسته می‌کند.

##### Testing and Trials

یکی از مهم‌ترین کاربردها، **test و آزمایش** است.

هنگام توسعه‌ی software بهتر است software جدید را روی همان سیستمی که محیط اصلی developer است آزمایش نکنید. VM امکان می‌دهد یک محیط جداگانه و نسبتاً disposable ایجاد کنید.

همین موضوع برای امتحان کردن distributionهای مختلف نیز کاربرد دارد. می‌توان یک Linux جدید را داخل VM اجرا کرد بدون اینکه hardware جدیدی خریداری شود یا سیستم اصلی دست‌کاری شود.

##### Application Compatibility

گاهی یک application فقط روی operating system خاصی اجرا می‌شود. اگر سیستم اصلی شما آن operating system را نداشته باشد، VM می‌تواند محیط موردنیاز را درون همان سیستم ایجاد کند.

##### Servers and Cloud Services

در محیط‌های server و cloud نیز VMها بسیار رایج‌اند.

اگر بخواهید یک web server یا server application را روی اینترنت در دسترس قرار دهید، یکی از ساده‌ترین روش‌ها استفاده از یک cloud instance است که در اصل یک Virtual Machine است.

Cloud providerها علاوه بر VM خام، serverهای از قبل پیکربندی‌شده مانند database environment نیز ارائه می‌دهند که باز هم روی زیرساخت VM اجرا می‌شوند.

#### 17.1.4 Drawbacks of Virtual Machines

VMها isolation بسیار مفیدی ایجاد می‌کنند، اما هزینه‌هایی نیز دارند.

اولین مسئله، **setup و configuration** است. آماده کردن یک operating system کامل و application environment زمان می‌برد. حتی اگر این کار با automation tools ساده‌تر شود، راه‌اندازی یک سیستم از صفر همچنان نسبت به یک environment سبک‌تر زمان‌بر است.

دومین مشکل، **boot و reboot** است. یک VM باید یک Linux system کامل را بالا بیاورد و بنابراین startup آن نسبت به چیزی مثل یک process یا container کندتر است.

سومین هزینه، **maintenance** است. هر VM یک operating system کامل دارد و باید update و security patch شود. ابزارهایی مانند `systemd` و `sshd` و libraryها و packageهای موردنیاز application نیز بخشی از همین maintenance هستند.

مشکل دیگر dependencyها هستند. ممکن است application شما با softwareهای استانداردی که روی VM نصب شده‌اند ناسازگار باشد. حتی libraryهایی که با یک system upgrade تغییر می‌کنند، می‌توانند باعث شوند applicationای که قبلاً کار می‌کرد دیگر درست اجرا نشود.

موضوع بعدی هزینه‌ی **resource isolation** است. جدا کردن هر service در یک VM می‌تواند از نظر مدیریت خوب باشد، اما اگر تعداد serviceها زیاد باشد مصرف resource و هزینه‌ی cloud نیز افزایش پیدا می‌کند.

این مشکلات برای سیستم‌های کوچک الزاماً جدی نیستند، اما هرچه تعداد serviceها بیشتر شود، زمان و هزینه‌ی نگهداری بیشتر نمایان می‌شود. اینجا است که containerها به‌عنوان یک alternative سبک‌تر اهمیت پیدا می‌کنند.

---

### 17.2 Containers

Virtual Machine برای جداسازی یک operating system کامل بسیار مناسب است، اما گاهی لازم نیست برای اجرای یک application کل operating system دیگری را boot کنیم.

**Container** برای چنین شرایطی یک محیط محدودتر و سبک‌تر فراهم می‌کند.

ریشه‌ی این ایده را می‌توان در تکنیک‌های قدیمی‌تر مانند `chroot()` و resource limitation دید.

با `chroot()` می‌توان root directoryای متفاوت برای یک process تعیین کرد تا process، در حالت عادی، محیط filesystem دیگری را به‌عنوان root ببیند.

مثلاً:

```text
/var/spool/my_service
```

می‌تواند برای یک service به‌عنوان root جدید انتخاب شود.

به همین دلیل اصطلاح **chroot jail** نیز رایج است.

Kernel همچنین قابلیت‌هایی برای محدود کردن منابع process دارد؛ یکی از این‌ها **rlimit** است که می‌تواند محدودیت‌هایی روی resourceهایی مانند CPU time یا اندازه‌ی file اعمال کند.

Container از ترکیب همین ایده‌ها و featureهای مختلف kernel شکل می‌گیرد.

در یک تعریف کلی، container یک **restricted runtime environment** برای مجموعه‌ای از processهاست، به‌گونه‌ای که این processها نتوانند آزادانه به بخش‌های خارج از محیط خود دسترسی داشته باشند.

این نوع virtualization معمولاً **operating system-level virtualization** نامیده می‌شود.

نکته‌ی مهم این است که یک machine که چند container روی آن اجرا می‌شوند همچنان فقط **یک Linux kernel زیرین** دارد.

یعنی:

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

هر container می‌تواند user-space متفاوتی داشته باشد، حتی اگر distribution داخل آن با distribution سیستم اصلی فرق کند، اما kernel جداگانه‌ای ندارد.

Isolation در container از ترکیب چند قابلیت kernel ساخته می‌شود. processهای داخل container می‌توانند ویژگی‌هایی مانند این‌ها داشته باشند:

* cgroupهای مخصوص خود
* deviceها و filesystemهای مخصوص
* عدم مشاهده یا تعامل مستقیم با processهای خارج از container
* network interfaceهای جداگانه

جمع کردن همه‌ی این محدودیت‌ها به‌صورت دستی دشوار است؛ به‌خصوص مدیریت cgroupها و namespaceها در سطح پایین کار ساده‌ای نیست.

به همین دلیل ابزارهایی مانند **Docker** و **LXC** برای ساخت و مدیریت containerها توسعه داده شده‌اند.

#### 17.2.1 Docker, Podman, and Privileges

در این فصل Docker به‌عنوان ابزار اصلی مثال‌ها استفاده می‌شود، اما **Podman** نیز معرفی می‌شود.

تفاوت مهم این دو ابزار در مدل اجرای آن‌هاست.

Docker معمولاً به یک server process نیاز دارد و در معماری متداول خود از:

```text
dockerd
```

استفاده می‌کند.

در بسیاری از configurationهای Docker، دسترسی به featureهای kernel موردنیاز containerها با privilegeهای بالاتر انجام می‌شود و `dockerd` بخش زیادی از این کار را بر عهده دارد.

Podman می‌تواند بدون server مرکزی اجرا شود و یکی از قابلیت‌های مهم آن **rootless operation** است؛ یعنی کاربر معمولی می‌تواند container را بدون اجرای آن با privilegeهای root مدیریت کند.

Podman همچنین می‌تواند به‌صورت root اجرا شود و در آن حالت از برخی تکنیک‌های isolation مشابه Docker استفاده کند.

از طرف دیگر، نسخه‌های جدیدتر Docker نیز می‌توانند rootless mode را پشتیبانی کنند.

از نظر command-line، Podman تا حد زیادی با Docker سازگار است. بنابراین بسیاری از مثال‌های Docker را می‌توان با جایگزین کردن:

```text
docker
```

با:

```text
podman
```

اجرا کرد.

البته implementation آن‌ها یکسان نیست و به‌خصوص در rootless mode تفاوت‌هایی در نحوه‌ی isolation وجود دارد.

#### 17.2.2 A Docker Example

برای درک container بهتر است مستقیماً با یک نمونه‌ی واقعی کار کنیم.

اول باید تفاوت **image** و **container** را روشن کنیم.

یک **image** شامل filesystem و اطلاعات لازم برای ایجاد container است. معمولاً image از یک image آماده که از یک registry دریافت شده شروع می‌شود.

یک مدل ساده برای فکر کردن درباره‌ی آن این است:

```text
Image
  ↓
Container
  ↓
Processes
```

processها داخل image اجرا نمی‌شوند؛ داخل container اجرا می‌شوند.

این مدل کاملاً دقیق نیست، اما برای شروع مفید است. به‌خصوص باید به یاد داشت تغییراتی که هنگام اجرای یک container روی filesystem ایجاد می‌کنید، image اصلی را تغییر نمی‌دهند.

##### Dockerfile

در مثال کتاب، یک image ساده بر پایه‌ی Alpine ساخته می‌شود.

فایل `Dockerfile` شامل این دستورهاست:

```dockerfile
FROM alpine:latest
RUN apk add bash
CMD ["/bin/bash"]
```

در این مثال:

```text
FROM
```

image پایه را مشخص می‌کند.

```text
RUN
```

دستوری است که هنگام **build کردن image** اجرا می‌شود.

و:

```text
CMD
```

رفتار پیش‌فرضی را تعیین می‌کند که هنگام **اجرای container** مورد استفاده قرار می‌گیرد.

برای ساخت image:

```bash
docker build -t hlw_test .
```

در این command:

```text
-t hlw_test
```

یک tag برای image تعیین می‌کند و:

```text
.
```

به directory جاری به‌عنوان build context اشاره دارد.

##### Intermediate Images

در زمان build، Docker معمولاً از imageهای مرحله‌ای عبور می‌کند.

در مثال:

```text
FROM alpine:latest
```

ابتدا image پایه‌ی Alpine دریافت می‌شود.

Docker با digestهای SHA256 و identifierهای کوتاه‌تر، اجزای مختلف image را دنبال می‌کند.

imageی که هنوز محصول نهایی نیست و قرار است لایه‌های بیشتری روی آن ساخته شود، **intermediate image** نام دارد.

##### RUN and Intermediate Containers

در مرحله‌ی بعد:

```dockerfile
RUN apk add bash
```

Docker یک **intermediate container** ایجاد می‌کند و command `apk add bash` را داخل آن اجرا می‌کند.

این موضوع مهم است، چون build یک Docker image همیشه به معنی اجرای commandها مستقیماً روی host نیست.

مدل ساده‌شده‌ی این مرحله چنین است:

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

پس دو مفهوم را باید از هم جدا کنیم:

**Intermediate Image:** بعد از build step می‌تواند باقی بماند و مبنای لایه‌ی بعدی باشد.

**Intermediate Container:** container موقتی است که برای اجرای یک build step ایجاد می‌شود و بعد از اتمام کار حذف می‌شود.

##### CMD

بعد از اجرای `RUN`، نوبت:

```dockerfile
CMD ["/bin/bash"]
```

است.

این command در زمان build، `/bin/bash` را برای اجرا نمی‌کند. `CMD` به **container runtime** می‌گوید وقتی container از image ساخته‌شده اجرا شد، command پیش‌فرض چه باشد.

به همین دلیل `RUN` و `CMD` نقش کاملاً متفاوتی دارند.

در پایان build، image نهایی یک ID دارد و tag نیز می‌گیرد:

```text
hlw_test
```

یا:

```text
hlw_test:latest
```

برای مشاهده‌ی imageها:

```bash
docker images
```

استفاده می‌شود.

##### Running Docker Containers

حالا می‌توان از image ساخته‌شده یک container اجرا کرد:

```bash
docker run -it hlw_test
```

گزینه‌ی `-i` محیط را interactive نگه می‌دارد و `-t` یک terminal متصل می‌کند.

اگر این گزینه‌ها حذف شوند، در سناریوی این مثال ممکن است shell شما prompt دریافت نکند و container تقریباً بلافاصله terminate شود.

در نتیجه:

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

در shell ایجادشده در این مثال، process با privilege مربوط به root داخل container اجرا می‌شود.

##### Inspecting Processes Inside the Container

بعد از ورود به container، با commandهایی مانند `mount` و `ps` می‌توان محیط را بررسی کرد.

اگر processهای container را مشاهده کنید، ممکن است چیزی شبیه این ببینید:

```text
PID USER COMMAND
1   root /bin/bash
6   root ps aux
```

در یک Linux system معمولی، PID 1 متعلق به `init` یا process اصلی system است، اما در این container، shell شما PID 1 را دارد.

دلیل این رفتار استفاده از **PID namespace** است.

یک process می‌تواند namespace جدیدی برای PIDها ایجاد کند که از داخل آن، processهای container شماره‌گذاری مخصوص خودشان را داشته باشند و از processهای خارج از namespace اطلاع نداشته باشند.

از طرف دیگر، همان processها در host همچنان process واقعی سیستم هستند و می‌توان آن‌ها را در process list host نیز پیدا کرد؛ فقط PIDی که host برای آن‌ها می‌بیند با PIDی که container می‌بیند یکسان نیست.

##### Overlay Filesystems

filesystem داخل container نیز با یک Linux system معمولی تفاوت مهمی دارد.

در Docker معمولاً می‌توان یک **overlay filesystem** را مشاهده کرد.

در این مدل، چند directory به‌عنوان layer روی هم قرار می‌گیرند و تغییرات در لایه‌ی بالایی نگهداری می‌شوند.

در output مربوط به mount معمولاً با سه مفهوم مهم روبه‌رو می‌شوید:

```text
lowerdir
upperdir
workdir
```

##### `lowerdir`

`lowerdir` مجموعه‌ای از directoryهای پایه است که layerهای پایین‌تر filesystem را تشکیل می‌دهند.

این مسیر می‌تواند چند directory جدا داشته باشد که به‌ترتیب روی هم stack می‌شوند.

##### `upperdir`

`upperdir` لایه‌ی بالایی است و تغییرات جدید filesystem در آن ظاهر می‌شوند.

برای container معمولاً این همان جایی است که تغییرات runtime به‌صورت writable ثبت می‌شوند.

##### `workdir`

`workdir` برای عملیات داخلی filesystem driver استفاده می‌شود و قبل از mount باید وضعیت مناسب داشته باشد.

در نتیجه، ساختار کلی overlay را می‌توان چنین تصور کرد:

```text
lower layer
lower layer
lower layer
-------------
upper writable layer
```

این مدل باعث می‌شود چند container یا image بتوانند بخش‌های مشترک پایینی را share کنند.

اما imageهایی که build stepهای زیادی دارند می‌توانند layerهای زیادی نیز ایجاد کنند. به همین دلیل strategyهایی مانند ترکیب بعضی `RUN` commandها یا **multistage build** می‌توانند تعداد layerها را کاهش دهند.

در rootless mode، Podman ممکن است از نسخه‌ی FUSE-based مربوط به overlay استفاده کند و اطلاعات mount متفاوتی روی host نشان دهد. در آن وضعیت می‌توان processهای `fuse-overlayfs` را بررسی کرد.

##### Networking

containerها می‌توانند در بعضی configurationها از network host استفاده کنند، اما حالت رایج‌تر ایجاد یک network namespace جدا است.

Docker به‌صورت معمول از **bridge network** استفاده می‌کند.

ابتدا interfaceای مانند:

```text
docker0
```

روی host ایجاد می‌شود که معمولاً در یک private subnet قرار دارد.

بعد برای container یک **network namespace** تازه ساخته می‌شود.

در ابتدا این namespace تقریباً خالی است و حداقل یک loopback interface مانند:

```text
lo
```

دارد.

سپس Docker یک جفت virtual interface ایجاد می‌کند که دو طرف یک ارتباط مجازی را تشکیل می‌دهند. یک طرف روی host باقی می‌ماند و طرف دیگر وارد network namespace مربوط به container می‌شود.

در container می‌توانید interfaceای مانند `eth0` ببینید، در حالی که host نیز ممکن است interfaceای با نام دیگری یا حتی مشابه داشته باشد؛ چون interfaceها در namespaceهای جدا قرار دارند.

این container معمولاً روی subnet خصوصی Docker قرار می‌گیرد. اگر قرار باشد container به شبکه‌ی بیرونی دسترسی داشته باشد، host باید traffic آن را با **NAT** به شبکه‌ی خارج منتقل کند.

پس مسیر کلی چنین است:

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

این معماری همان چیزی است که باعث می‌شود container هم isolation شبکه داشته باشد و هم بتواند به بیرون دسترسی پیدا کند.

یک نکته‌ی عملی این است که private subnet مورد استفاده‌ی Docker ممکن است با subnetی که router یا تجهیزات شبکه‌ی دیگر به کار می‌برند تداخل داشته باشد؛ بنابراین هنگام troubleshooting networking باید به address rangeها توجه کرد.

در **rootless Podman** ساخت network interface مجازی متفاوت است، زیرا ایجاد interfaceهای موردنیاز معمولاً privilege بالاتری می‌خواهد. Podman در این حالت می‌تواند از network namespace جدید به‌همراه **TAP interface** و ابزار forwarding مانند `slirp4netns` استفاده کند. این روش محدودیت‌هایی نسبت به مدل معمول Docker دارد؛ برای مثال ارتباط مستقیم containerها با یکدیگر به همان شکل وجود ندارد.

موضوعاتی مانند port exposure و routing جزئیات بیشتری دارند، اما مهم‌ترین چیزی که باید از این بخش باقی بماند، شناخت topology شبکه و نقش network namespace، virtual interface، bridge و NAT است.

**Figure 17-1: Bridge network in Docker.**

##### Docker Operation

Docker علاوه بر ایجاد isolation، باید lifecycle تمام resourceهای مربوط به container را نیز مدیریت کند.

Docker یک container را تا زمانی **running** در نظر می‌گیرد که حداقل یک process در آن در حال اجرا باشد.

برای دیدن containerهای در حال اجرا:

```bash
docker ps
```

استفاده می‌شود.

وقتی تمام processهای container terminate شوند، Docker container را از بین نمی‌برد، بلکه آن را به حالت exited منتقل می‌کند؛ مگر اینکه container را با گزینه‌ی:

```text
--rm
```

اجرا کرده باشید.

پس از توقف، container می‌تواند تغییرات filesystem خود را نیز نگه دارد و می‌توان filesystem آن را با ابزارهایی مانند:

```bash
docker export
```

دریافت کرد.

نکته‌ی مهم اینجاست که:

```bash
docker ps
```

به‌صورت پیش‌فرض فقط containerهای running را نشان می‌دهد.

برای مشاهده‌ی exited containerها نیز باید از:

```bash
docker ps -a
```

استفاده کرد.

این موضوع از نظر storage مهم است. ممکن است تعداد زیادی container متوقف‌شده روی سیستم باقی بمانند و اگر application داخل آن‌ها داده‌ی زیادی تولید کرده باشد، disk space به‌مرور مصرف شود.

برای حذف container تمام‌شده می‌توان از:

```bash
docker rm <container>
```

استفاده کرد.

##### Docker Images

همین مسئله درباره‌ی imageها نیز وجود دارد.

ساخت image معمولاً یک فرآیند تکراری است. اگر یک image جدید با همان tag قبلی ساخته شود، Docker image قدیمی را الزاماً حذف نمی‌کند؛ ممکن است tag از image قبلی جدا شود و آن image بدون tag باقی بماند.

برای مشاهده‌ی imageها:

```bash
docker images
```

استفاده می‌شود و ممکن است در خروجی imageهایی با:

```text
REPOSITORY   <none>
TAG          <none>
```

ببینید.

این imageها می‌توانند به‌مرور فضای زیادی مصرف کنند.

برای حذف image:

```bash
docker rmi <image>
```

استفاده می‌شود.

در بسیاری از موارد حذف image می‌تواند intermediate imageهایی را که دیگر موردنیاز نیستند نیز حذف کند.

بنابراین مدیریت Docker فقط ساخت و اجرای container نیست؛ imageها، containerهای متوقف‌شده و layerهای قدیمی نیز بخشی از operational maintenance هستند.

##### Docker Service Process Models

یکی از نکات ظریف containerها به **lifecycle processها** مربوط می‌شود.

وقتی یک child process terminate می‌شود، parent باید status خروج آن را با `wait()` جمع‌آوری یا به‌اصطلاح **reap** کند.

اگر این کار انجام نشود، processهای مرده می‌توانند در سیستم به شکل **zombie process** باقی بمانند.

در container نیز ممکن است همین مسئله اتفاق بیفتد؛ مخصوصاً وقتی process tree پیچیده باشد و process با PID 1 داخل container به‌صورت مستقیم از child processها آگاه نباشد.

این موضوع به یک تصور اشتباه منجر شده است که گویی نباید چند process یا service را داخل یک container اجرا کرد. کتاب صراحتاً تأکید می‌کند که داشتن چند process داخل container ذاتاً مشکل نیست.

مسئله اصلی این است که process parent باید lifecycle child processهای خود را درست مدیریت کند.

اگر یک service ساده دارید که processهای دیگری ایجاد می‌کند و در پایان container processهای باقی‌مانده می‌بینید، Docker گزینه‌ای به نام:

```bash
docker run --init ...
```

ارائه می‌دهد.

این option یک init بسیار ساده را به‌عنوان PID 1 در container اجرا می‌کند تا در reap کردن child processها کمک کند.

اگر container شما چند service یا task مختلف دارد، می‌توان به‌جای یک startup script ساده از process managerهایی مانند **Supervisor (`supervisord`)** برای start و monitor کردن processها استفاده کرد.

#### 17.2.3 LXC

**LXC** یکی از قدیمی‌ترین سیستم‌های Linux container است. حتی نسخه‌های اولیه‌ی Docker نیز بر پایه‌ی LXC ساخته شده بودند.

اصطلاح LXC گاهی برای مجموعه‌ی kernel featureهایی که containerها را ممکن می‌کنند استفاده می‌شود، اما معمولاً منظور از LXC یک library و مجموعه‌ای از utilityها برای ایجاد و مدیریت Linux containerهاست.

تفاوت مهم LXC با Docker در میزان abstraction و مقدار setup دستی است.

Docker بسیاری از جزئیات را پنهان می‌کند و با imageهای آماده کار را سریع می‌کند، اما LXC کنترل بیشتری در اختیار کاربر قرار می‌دهد و در نتیجه ممکن است نیاز باشد برخی اجزای محیط را خودتان تنظیم کنید.

برای مثال در LXC ممکن است خودتان مسئول ایجاد network interface مناسب یا تنظیم **user ID mapping** باشید.

از ابتدا LXC تا حد زیادی با این هدف طراحی شده بود که یک Linux system نسبتاً کامل داخل container داشته باشید، حتی تا سطح `init`.

در نتیجه می‌توانید یک distribution مخصوص container نصب کنید و softwareهای موردنیاز خود را داخل آن قرار دهید.

LXC به‌صورت پیش‌فرض از همان overlay filesystem که در مثال Docker دیدیم استفاده نمی‌کند، هرچند امکان اضافه کردن آن وجود دارد.

همچنین LXC یک C API در اختیار می‌گذارد و همین موضوع باعث می‌شود برنامه‌ی دیگری بتواند در سطح پایین‌تر مستقیماً با functionality آن کار کند.

در کنار LXC، پروژه‌ی **LXD** برای ساده‌تر کردن بعضی قسمت‌های management ارائه شده است. LXD می‌تواند کارهایی مانند network creation و image management را ساده‌تر کند و علاوه بر C API، یک **REST API** نیز در اختیار قرار دهد.

در نتیجه می‌توان گفت Docker بیشتر روی ساده‌سازی workflow و image-based operation تأکید دارد، در حالی که LXC کنترل و انعطاف بیشتری در سطح پایین در اختیار کاربر قرار می‌دهد.

#### 17.2.4 Kubernetes

Containerها وقتی تعدادشان زیاد شود، به‌سرعت به یک مسئله‌ی management تبدیل می‌شوند.

فرض کنید تعداد زیادی container روی چند machine دارید. دیگر فقط اجرای یک container کافی نیست؛ باید مواردی مانند این‌ها را مدیریت کنید:

* مشخص شود کدام machine توانایی اجرای container را دارد.
* containerها start شوند.
* وضعیتشان monitor شود.
* در صورت failure دوباره restart شوند.
* startup آن‌ها پیکربندی شود.
* network آن‌ها تنظیم شود.
* نسخه‌های جدید image منتشر و به‌صورت کنترل‌شده روی containerهای در حال اجرا اعمال شوند.

همین مسئله باعث ظهور سیستم‌های **container orchestration** شد.

یکی از مهم‌ترین نمونه‌ها **Kubernetes** است.

Kubernetes برای مدیریت مجموعه‌ای از containerها روی چند machine طراحی شده و از نظر مفهومی می‌توان آن را به دو بخش کلی server و client تقسیم کرد.

در سمت server، machineهایی قرار دارند که containerها روی آن‌ها اجرا می‌شوند.

در سمت client، مجموعه‌ای از command-line utilityها و configurationها قرار دارند که برای ایجاد و مدیریت گروه‌های container استفاده می‌شوند.

Configurationهای Kubernetes می‌توانند بسیار بزرگ و پیچیده شوند و بخش قابل‌توجهی از کار عملی Kubernetes در نوشتن configuration صحیح برای workloadهاست.

برای آزمایش Kubernetes بدون راه‌اندازی کامل server infrastructure می‌توان از **Minikube** استفاده کرد که یک Kubernetes cluster را در قالب یک Virtual Machine روی سیستم شخصی ایجاد می‌کند.

Kubernetes در این فصل بیشتر به‌عنوان مرحله‌ی بعدی مدیریت containerها معرفی می‌شود؛ یعنی زمانی که دیگر مسئله فقط «چطور یک container را اجرا کنم؟» نیست، بلکه «چطور تعداد زیادی container را روی چند machine مدیریت کنم؟» مطرح است.

#### 17.2.5 Pitfalls of Containers

Containerها بسیاری از مشکلات VM را ساده می‌کنند، اما مشکل را به‌طور کامل حذف نمی‌کنند.

در هر container infrastructure همچنان به حداقل یک یا چند machine نیاز دارید و آن machine باید یک Linux system کامل باشد؛ چه روی hardware واقعی باشد و چه خودش یک Virtual Machine باشد.

بنابراین هنوز هزینه‌ی infrastructure، hardware، hosting و maintenance وجود دارد.

اگر infrastructure را خودتان مدیریت کنید، هزینه‌ی عملیاتی و زمانی بالاست. اگر از container service یا managed Kubernetes استفاده کنید، همان هزینه در قالب هزینه‌ی monetary به provider منتقل می‌شود.

##### Storage Overhead

Containerها سبک‌تر از VM هستند، اما application داخل container همچنان به libraryها و بخش‌هایی از user-space یک Linux system نیاز دارد.

در نتیجه اگر base image مناسب انتخاب نشود، imageها می‌توانند بزرگ شوند.

Overlay filesystem می‌تواند وقتی چند container از base layerهای مشترک استفاده می‌کنند، مقداری از این هزینه را کاهش دهد؛ چون فایل‌های مشترک لازم نیست برای هر container به‌صورت مستقل ذخیره شوند.

اما اگر application هنگام runtime داده‌ی زیادی تولید کند، writable upper layer هر container نیز می‌تواند رشد کند.

##### Resource Usage

Container isolation به این معنی نیست که CPU و سایر resourceها بی‌نهایت می‌شوند.

همچنان یک kernel و یک مجموعه‌ی زیرین از resources وجود دارد.

می‌توانید برای containerها limit تعریف کنید، اما ظرفیت machine واقعی همچنان محدود است.

اگر workload بیش از توان host باشد، هم containerها و هم سیستم زیرین می‌توانند تحت فشار قرار بگیرند.

##### Runtime Data and Logs

یک مسئله‌ی مهم دیگر محل نگهداری data است.

در سیستم‌هایی که overlay filesystem دارند، تغییراتی که application روی filesystem container در زمان runtime ایجاد می‌کند ممکن است با lifecycle همان container مرتبط باشند و پس از terminate شدن processها دیگر در آن شکل باقی نمانند.

در بسیاری از applicationها، user data داخل database ذخیره می‌شود و مدیریت data تا حدی به database administration منتقل می‌شود.

اما **logs** همچنان مسئله‌ی مهمی هستند. یک server application برای troubleshooting و operation به log نیاز دارد و در محیط‌های بزرگ، معمولاً باید logها در یک سیستم جداگانه نگهداری و مدیریت شوند.

##### Container-Centric Operations

بخش قابل‌توجهی از container tooling و operational modelها برای web serverها طراحی شده‌اند.

در web workloads، ابزارها و documentation فراوانی برای اجرای container وجود دارد و سیستم‌هایی مانند Kubernetes قابلیت‌های زیادی برای جلوگیری از runaway workloadها ارائه می‌دهند.

اما اگر بخواهید service متفاوتی را در همان مدل قرار دهید، ممکن است بعضی abstractionها دقیقاً با نیاز واقعی آن service هماهنگ نباشند.

##### Configuration Problems

Container یک محیط ایزوله ایجاد می‌کند، اما شما را از configuration mistake نجات نمی‌دهد.

یک container می‌تواند به‌دلیل configuration نادرست، dependency اشتباه یا design ضعیف دچار مشکل شود.

حتی اگر مجبور نباشید با بخش‌هایی از یک Linux system کامل مانند `systemd` سروکار داشته باشید، اجزای زیادی همچنان وجود دارند که می‌توانند اشتباه پیکربندی شوند.

یکی از خطاهای رایج هنگام troubleshooting این است که کاربر بدون درک علت مشکل، چیزهای بیشتری به environment اضافه می‌کند تا موقتاً مشکل برطرف شود. این کار می‌تواند environment را بسیار پیچیده و شکننده کند.

بهتر است هر تغییر را با درک دقیق اثر آن انجام دهید.

##### Versioning Problems

در مثال‌های کتاب از tag:

```text
latest
```

استفاده می‌شود، اما این روش یک ریسک مهم دارد.

وقتی image جدیدی با `latest` منتشر می‌شود، ممکن است چیزی در base distribution یا dependencyهای زیرین تغییر کرده باشد و application شما دیگر همان رفتار قبلی را نداشته باشد.

برای جلوگیری از وابستگی غیرقابل‌پیش‌بینی به تغییرات آینده، یکی از روش‌های معمول استفاده از **version tag مشخص** برای base image است.

در نتیجه این تفاوت مهم است:

```text
latest
```

یک reference متحرک است و ممکن است به نسخه‌ی دیگری در آینده اشاره کند.

در حالی که یک version مشخص، build environment قابل‌پیش‌بینی‌تری ایجاد می‌کند.

##### Trust and Image Sources

موضوع دیگری که نباید فراموش شود **trust** است.

وقتی container خود را بر پایه‌ی image موجود در یک public repository می‌سازید، در واقع به یک لایه‌ی اضافی از software distribution اعتماد می‌کنید.

باید فرض کنید که imageی که دریافت می‌کنید همواره همان چیزی است که انتظار دارید و همچنین در آینده در دسترس باقی می‌ماند.

این یکی از تفاوت‌های مهمی است که کتاب میان مدل image-based Docker و رویکرد LXC مطرح می‌کند؛ در LXC کاربر بیشتر به سمت ساخت و کنترل محیط خودش هدایت می‌شود.

در نهایت، این محدودیت‌ها به معنی بد بودن container نیستند. تقریباً هر مدل دیگری نیز شکل دیگری از همین مشکلات را دارد.

Container قرار نیست همه‌ی مسائل infrastructure را حل کند.

اگر application روی یک Linux system عادی زمان زیادی برای startup نیاز داشته باشد، container کردن آن به‌خودی‌خود startup را جادویی سریع نمی‌کند.

Container فقط layerهای خاصی از محیط را ساده‌تر و قابل‌کنترل‌تر می‌کند.

---

### 17.3 Runtime-Based Virtualization

نوع دیگری از virtualization در سطح **application runtime** اتفاق می‌افتد.

در این مدل برخلاف VM و container، هدف ایجاد یک machine یا system environment جداگانه نیست.

هدف، جدا کردن environment مربوط به یک application خاص است.

این موضوع زمانی اهمیت پیدا می‌کند که چند application از یک programming language و packageهای مشترک استفاده می‌کنند.

Python مثال خوبی برای این مسئله است.

ممکن است خود Linux distribution از Python استفاده کند و packageهای مختلفی نیز روی آن نصب شده باشد. اگر application شما به version متفاوتی از یکی از این packageها نیاز داشته باشد، استفاده‌ی مستقیم از system Python می‌تواند باعث conflict شود.

راه‌حل Python، ایجاد یک **virtual environment** است.

ابتدا یک directory برای environment ایجاد می‌کنید:

```bash
python3 -m venv test-venv
```

در این directory می‌توانید ساختاری شبیه یک environment مستقل ببینید:

```text
test-venv/
├── bin/
├── include/
└── lib/
```

این محیط به شما اجازه می‌دهد package library جداگانه‌ای داشته باشید بدون اینکه packageهای system Python را تغییر دهید.

#### Activating a Virtual Environment

برای فعال کردن محیط، باید script زیر را **source** کنید:

```bash
. test-venv/bin/activate
```

این نکته مهم است که script را نباید مثل یک executable معمولی اجرا کنید.

دلیل آن این است که activation در اصل با تغییر **environment variableهای shell فعلی** انجام می‌شود.

اگر script را به‌صورت process مستقل اجرا کنید، تغییر environment آن process به shell اصلی منتقل نمی‌شود.

بعد از activation، command `python` به Python مربوط به environment جدید اشاره می‌کند. در ساختار معمول این environment، executable مربوطه ممکن است خودش یک symbolic link باشد.

همچنین environment variable زیر تنظیم می‌شود:

```text
VIRTUAL_ENV
```

که base directory محیط فعال را مشخص می‌کند.

برای خارج شدن از environment:

```bash
deactivate
```

استفاده می‌شود.

بعد از activation، packageهایی که داخل environment نصب می‌کنید به library جداگانه‌ی آن environment می‌روند، نه library اصلی system.

در نتیجه چند پروژه می‌توانند dependencyهای متفاوتی داشته باشند بدون اینکه packageهای یکدیگر را خراب کنند.

اینجا اصطلاح **virtualization** معنای متفاوتی نسبت به VM و container دارد. دیگر کل machine یا operating system را جدا نکرده‌ایم؛ فقط environment لازم برای یک application یا programming language را از سایر applicationها جدا کرده‌ایم.

---

### Summary

فصل هفدهم سه سطح متفاوت از virtualization را بررسی می‌کند.

در **Virtual Machine**، یک machine کامل و یک operating system مستقل داخل محیط مجازی اجرا می‌شود. Hypervisor منابع hardware را مدیریت می‌کند و بسته به نوع آن می‌تواند Type 1 یا Type 2 باشد. VMها برای testing، application compatibility و cloud workloads بسیار مناسب‌اند، اما به‌دلیل داشتن operating system کامل، هزینه‌ی بیشتری برای startup، resource و maintenance دارند.

در **Container**، دیگر kernel جداگانه‌ای برای هر environment وجود ندارد. چند container روی یک Linux kernel مشترک اجرا می‌شوند و با ترکیب قابلیت‌هایی مانند **namespaces، cgroups، filesystem isolation و network isolation** محیط‌های جداگانه برای processها ساخته می‌شود. Docker و Podman workflow این کار را ساده می‌کنند و LXC کنترل بیشتری در سطح پایین ارائه می‌دهد. وقتی تعداد containerها زیاد شود، Kubernetes مسئله‌ی orchestration را مدیریت می‌کند.

در **Runtime-Based Virtualization**، جداسازی حتی محدودتر می‌شود و فقط environment مربوط به یک application یا programming language مستقل می‌گردد؛ Python `venv` نمونه‌ی مشخص آن است.

بنابراین virtualization در Linux یک تکنولوژی واحد نیست؛ مجموعه‌ای از روش‌ها برای ایجاد **isolation، کنترل منابع و جداسازی محیط‌های اجرا** است و هر سطح، مقدار متفاوتی از isolation و overhead ایجاد می‌کند.
