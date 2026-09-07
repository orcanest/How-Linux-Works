## وقتی Kernel کارش تموم میشه ، User Space شروع میشه
### 🐧 فصل ششم کتاب How Linux Works

فصل پنجم توضیح داد که Linux Kernel چگونه از طریق firmware و boot loader وارد حافظه می‌ شود، سخت‌ افزار و driver ها را آماده می‌ کند ، filesystem هم ``/`` را در دسترس قرار می‌ دهد و در نهایت اولین process مربوط به user space را اجرا می‌ کند. نقطه‌ ای که Kernel اولین process را اجرا می‌ کند، یکی از مهم‌ ترین نقاط در فرآیند boot است. تا قبل از آن، مسیر اجرای سیستم عمدتاً درون Kernel و تحت کنترل مستقیم آن قرار دارد. اما از اینجا به بعد ، بخش بزرگی از رفتار سیستم توسط برنامه‌ های user space ، سرویس‌ ها ، daemon ها ، configuration ها و ابزارهای مدیریتی تعیین می‌ شود.

به همین دلیل ، برای فهم Linux فقط دانستن اینکه Kernel چگونه boot می‌ شود کافی نیست. باید بدانیم بعد از اجرای init چه اتفاقی می‌ افتد ، سرویس‌ ها چگونه شروع می‌ شوند ، dependency ها چگونه مدیریت می‌ شوند ، systemd چگونه process ها را دنبال می‌ کند ، سیستم‌ های قدیمی‌ تر مانند System V init چگونه کار می‌کردند و در شرایط اضطراری چگونه می‌ توان سیستم را بالا آورد. به‌ صورت کلی ، شروع user space را می‌ توان چنین دید:

1. اجرای init
2. راه‌ اندازی سرویس‌ های پایه و ضروری
3. آماده‌ سازی شبکه
4. راه‌ اندازی سرویس‌ های سطح میانی و بالاتر
5. ارائه login prompt ، محیط گرافیکی و سرویس‌ های سطح بالا

این ترتیب یک تصویر مفهومی است و در سیستم‌ های مختلف ممکن است جزئیات آن متفاوت باشد.

---
📚 Table of Contents

- [Introduction to init](#introduction-to-init)
- [Identifying Your init](#identifying-your-init)
- [systemd](#systemd)
- [Units and Unit Types](#units-and-unit-types)
- [Booting and Unit Dependency Graphs](#booting-and-unit-dependency-graphs)
- [systemd Configuration](#systemd-configuration)
- [systemd Operation](#systemd-operation)
- [systemd Process Tracking and Synchronization](#systemd-process-tracking-and-synchronization)
- [systemd Dependencies](#systemd-dependencies)
- [systemd on demand and resource parallelized startup](#systemd-on-demand-and-resource-parallelized-startup)
- [systemd Auxiliary Components](#systemd-auxiliary-components)
- [system V Runlevels](#system-v-runlevels)
- [system V init](#system-v-init)
- [system V init startup command sequence](#system-v-init-startup-command-sequence)
- [system V init link farm](#system-v-init-link-farm)
- [run parts](#run-parts)
- [system V init control](#system-v-init-control)
- [systemd system V compatibility](#systemd-system-V-compatibility)
- [shutting down your system](#shutting-down-your-system)
- [initial RAM filesystem](#initial-ram-filesystem)
- [emergency booting and single user mode](#emergency-booting-and-single-user-mode)
- [Tips](#tips)

---

### Introduction to init

برنامه init ، یک برنامه معمولی در user space است ، یعنی بخشی از Kernel نیست. نقش اصلی init این است که بعد از آماده شدن Kernel ، فرآیند راه‌ اندازی باقی سیستم را بر عهده بگیرد و سرویس‌ های مورد نیاز را شروع و در زمان مناسب متوقف کند. در سیستم‌های Linux مدرن، systemd رایج‌ ترین implementation برای init است ، اما Linux فقط یک نوع init ندارد. پیاده‌ سازی‌ های دیگری نیز وجود داشته‌ اند یا هنوز در بعضی سیستم‌ ها استفاده می‌ شوند:


- systemd
- System V init
- Upstart
- runit
- Special implementations in some embedded systems
- Android-specific init

کتاب برای توضیح اصلی این فصل روی systemd تمرکز می‌ کند ، چون در بسیاری از توزیع‌ های اصلی Linux به implementation غالب init تبدیل شده است. در عین حال شناخت System V init همچنان مهم است ، چون تعداد زیادی script و package قدیمی بر اساس مدل آن ساخته شده‌ اند.

#### 🔹 problem with System V init model

مدل System V init اساساً مجموعه‌ ای از script هاست که init آن‌ ها را به ترتیب اجرا می‌ کند. هر script معمولاً وظیفه‌ای را انجام می‌دهد مانند:

- Starting a daemon
- Preparing a subsystem
- Mounting a filesystem
- Configuring a service
- Executing part of the boot process

این روش مزایایی دارد. مثلاً dependency ها را می‌ توان با تعیین ترتیب اجرای script ها مدیریت کرد و administrator می‌تواند script ها را برای شرایط خاص تغییر دهد اما چند مشکل مهم دارد : 

- **نبود parallel startup مناسب** : در مدل ترتیبی ، دو بخش مستقل از فرآیند boot معمولاً نمی‌ توانند هم‌ زمان اجرا شوند. مثلاً اگر سرویس A و سرویس B هیچ وابستگی به یکدیگر نداشته باشند، در یک سیستم کاملاً ترتیبی ممکن است ابتدا A اجرا شود ، منتظر پایان آن بمانیم و سپس B را اجرا کنیم. این مسئله می‌ تواند زمان boot را افزایش دهد.

- **سخت بودن مدیریت process های سرویس** : معمولاً Script یک daemon را اجرا می‌ کند و بعد administrator باید somehow بفهمد daemon مربوطه دقیقاً کدام PID را دارد. ممکن است ```ps``` استفاده شود یا خود سرویس PID را در فایلی مانند ```var/run/myservice.pid/``` ذخیره کند. در نتیجه یک روش استاندارد و عمومی برای tracking تمام process های متعلق به یک سرویس وجود ندارد.

- **تکرار زیاد در startup script ها** : بسیاری از  scriptهای System V شامل مقدار زیادی boilerplate هستند بخش‌ هایی از این کد ها برای موارد زیر استفاده می‌شوند در نتیجه script ها می‌توانند طولانی شوند :

```
- Check service status
- Find PID
- Start
- Stop
- Restart
- Reload
- Check for errors
```

4. **نبود مفهوم قدرتمند on-demand service** : در مدل قدیمی بسیاری از سرویس‌ ها در زمان boot شروع می‌ شوند ، حتی اگر بعداً هیچ‌ وقت مورد استفاده قرار نگیرند. مدل on-demand می‌ تواند اجازه دهد سرویس فقط وقتی واقعاً لازم شد اجرا شود. ابزارهایی مانند inetd در گذشته بخشی از این ایده را برای سرویس‌ های شبکه ارائه می‌ کردند.

---

### Identifying Your init

برای فهمیدن اینکه سیستم از چه init استفاده می‌ کند،  یکی از راه‌های ساده بررسی init(1) است. در سیستم‌ های مختلف می‌توان به filesystem نیز نگاه کرد. مثلاً وجود ```usr/lib/systemd/``` یا ```etc/systemd/``` نشانه‌ ای قوی از وجود systemd است.

وجود ```etc/init/``` همراه با چند فایل conf. می‌ تواند نشانه Upstart باشد و وجود ```etc/inittab/``` در سیستمی که systemd یا Upstart ندارد ، می‌تواند نشان‌ دهنده System V init باشد. 

یک روش عملی‌ تر در سیستم‌ های Linux امروزی این است که PID 1 را بررسی کنیم چون init اولین process اصلی user space است و PID آن معمولاً 1 است :

```ps -p 1 -o pid,comm,args```

---

### systemd

یکی از نسل جدید init system های Linux است. هدف systemd فقط اجرای چند script در زمان boot نیست بلکه systemd تلاش می‌کند مدیریت گسترده‌ تری از سرویس‌ها و منابع سیستم ارائه کند. systemd از ایده‌ های سیستم‌های دیگری مانند launchd نیز الهام گرفته است ، از جمله ایده‌ هایی که در طراحی آن دیده می‌ شود :

- Service management
- Process tracking
- Dependency management
- Parallel startup
- Socket activation
- Management of some resources
- Timers
- Mounts
- Integration with components such as udev and logging

تفاوت مهم systemd با init این است که systemd به جای اینکه صرفاً یک sequence ثابت از script ها را اجرا کند ، سیستم را به مجموعه‌ ای از unit ها ، dependency ها و ordering ها تبدیل می‌ کند. در این مدل، unit ها زمانی که شرایط لازم برایشان فراهم شود فعال می‌ شوند و unit های مستقل می‌ توانند تا حد امکان به‌ صورت موازی اجرا شوند. systemd حتی بعد از boot نیز می‌ تواند در واکنش به event های سیستم ، unit های دیگری را فعال کند.

---

### Units and Unit Types

مفهوم مرکزی systemd، unit است. هر unit نماینده یک نوع resource یا task تحت مدیریت systemd است. برخلاف init سنتی که تمرکز اصلی‌ اش روی process و daemon بود ، systemd می‌ تواند چیزهای بیشتری را مدیریت کند.

#### 🔹 Service Unit

از مهم‌ ترین unit type ها **Service Unit** است. فایل‌ هایی با پسوند  ```service.``` برای مدیریت daemon ها و سرویس‌ ها استفاده می‌ شوند. مثلاً ```sshd.service``` می‌ تواند سرویس SSH را کنترل کند.

#### 🔹 Target Unit

مورد بعدی **Target Unit** هستش فایل‌ هایی با پسوند ```target.``` بیشتر برای grouping و تعریف یک وضعیت یا هدف سیستم استفاده می‌ شوند. Target معمولاً خودش daemon نیست و می‌ تواند در ساختار boot نقش داشته باشند برای مثال:

- default.target
- multi-user.target
- basic.target
- sysinit.target

#### 🔹 Socket Unit

فایل‌ هایی با پسوند ```socket.``` برای نمایش و مدیریت یک endpoint مانند socket شبکه استفاده می‌ شوند. یکی از کاربرد های مهم آن‌ها socket activation است.

#### 🔹 Mount Unit

فایل‌ هایی با پسوند ```mount.``` برای مدیریت mount شدن filesystem ها استفاده می‌ شوند. بنابراین systemd را نباید فقط یک process launcher در نظر گرفت مدل آن بسیار general تر است.

---

### Booting and Unit Dependency Graphs

هنگام boot ، systemd یک unit پیش‌ فرض را فعال می‌ کند این unit معمولاً ```default.target``` است. ```default.target``` مجموعه‌ ای از dependency ها را به سیستم معرفی می‌ کند. ممکن است این target به target های دیگری وابسته باشد و در نهایت سرویس‌ ها و resource های مختلف نیز وارد این graph شوند مانند:

- multi-user.target
- basic.target
- sysinit.target

نکته مهم این است که dependency های systemd یک tree ساده نیستند. ساختار واقعی یک graph است یعنی ممکن است یک unit به چند unit دیگر وابسته باشد یا چند مسیر مختلف در نهایت به یک unit مشترک برسند و Unit هایی که در بخش‌ های مختلف boot قرار دارند دوباره به یکدیگر متصل شوند برای مشاهده graph می‌توان از  ```systemd-analyze dot``` استفاده کرد. در یک سیستم واقعی graph بسیار بزرگ خواهد بود و خواندن تمام آن ساده نیست بنابراین معمولاً باید بخش خاصی از آن را بررسی کرد.

<img width="100%" height="367" alt="image" src="https://github.com/user-attachments/assets/f739d134-c631-4f5f-b07f-4af53e819d46" />

---

### systemd Configuration

فایل‌ های configuration مربوط به systemd در چند directory قرار دارند. دو مسیر مهم عبارت‌ اند از ```usr/lib/systemd/system/``` و ```lib/systemd/system/ که محل unit های ارائه‌ شده توسط سیستم و package هاست و ```etc/systemd/system/``` که محل configuration های محلی administrator است.

قاعده مهم ، تغییرات local را در etc/ انجام دهید و فایل‌ های ارائه‌ شده توسط distribution را مستقیماً تغییر ندهید. چون فایل‌ های داخل usr/ یا lib/ ممکن است هنگام update package ها تغییر کنند یا overwrite شوند. برای مشاهده مسیرهای جستجوی systemd می‌توان استفاده کرد:

```systemctl -p UnitPath show```

همچنین برای دیدن مسیرهای مربوط به unit ها و configuration استفاده می‌ شوند :

```pkg-config systemd --variable=systemdsystemunitdir```

```pkg-config systemd --variable=systemdsystemconfdir```

#### 🔹 Unit File

 ساختار Unit file شبیه فایل‌ های configuration مبتنی بر section دارد مثلاً:

```
[Unit]
Description=Example Service

[Service]
ExecStart=/usr/bin/example
```
بخش [Unit] معمولاً شامل Description و dependencies و relationships است و [Service] اطلاعات مربوط به نحوه اجرای سرویس را مشخص می‌ کند. option هایی مانند ```ExecStart``` و ```ExecReload``` در این قسمت قرار می‌گیرند.

#### 🔹 Variables

در unit file می‌توان variable هایی با $ داشت مثلاً ```MAINPID$``` می‌ تواند PID اصلی process تحت tracking سرویس را نشان دهد یک نمونه :

```ExecReload=/bin/kill -HUP $MAINPID```

یعنی هنگام reload، signal مناسب به process اصلی سرویس ارسال شود. Variable هایی مانند OPTIONS$ یا CRYPTO_POLICY$ ممکن است از Environment File گرفته شوند.

#### 🔹 Specifiers

 معمولاً Specifier ها با % شروع می‌ شوند مثلاً ```n%``` برای نام unit جاری و ```H%``` برای hostname استفاده می‌ شود. یکی از کاربرد های مهم specifier ها ، ساخت instance های متعدد از یک unit template است مثلاً ```getty@.service``` می‌تواند template باشد که instance های زیر از آن ساخته می‌شوند مانند ```getty@tty1``` و ```getty@tty2``` ، قسمت بعد از @ همان instance است. در template ممکن است از ```i%``` و ```I%``` برای دریافت instance استفاده شود.

---

### systemd Operation

#### 🔹 systemctl

ابزار اصلی برای تعامل با systemd ، ابزار ```systemctl``` است.

- برای دیدن unit های فعال از ```systemctl list-units``` استفاده می شود در واقع ```systemctl``` به‌ طور پیش‌ فرض نیز رفتار مشابهی دارد.
- برای دیدن نام کامل unit ها ```systemctl list-units --full``` استفاده می شود.
- برای دیدن unit هایی که active نیستند نیز می‌توان ```systemctl list-units --all``` را استفاده کرد.
- برای دیدن وضعیت یک unit مثلاً ```systemctl status sshd.service``` می‌تواند اطلاعاتی را نمایش دهد مانند :
```
- Unit status
- Start time
- Main PID
- Number of tasks
- Cgroup
- Service related processes
- Logs
```

- برای دیدن log های مربوط به یک unit از ```journalctl --unit=sshd.service``` استفاده می‌ شود.

#### 🔹 Start / Stop / Restart

این عملیات با مفهوم job در systemd مرتبط هستند :

- برای start از ```systemctl start unit``` استفاده می‌ شود.
- برای stop از ```systemctl stop unit``` استفاده می‌ شود.
- برای restart از ```systemctl restart unit``` استفاده می‌ شود.

#### 🔹 reload

دو مفهوم متفاوت وجود دارد. برای reload configuration خود سرویس ```systemctl reload unit``` و برای اینکه خود systemd unit file های جدید یا تغییرکرده را دوباره بخواند ```systemctl daemon-reload``` استفاده می شود که این دو را نباید با یکدیگر اشتباه گرفت.

#### 🔹 View jobs

برای دیدن job ها از ```systemctl list-jobs``` استفاده می شود. Job ها درخواست‌ های فعال برای تغییر state unit ها هستند. در زمان boot ممکن است تعداد زیادی job در وضعیت waiting یا running دیده شود. وقتی dependency اصلی آماده شود ، job های منتظر نیز می‌توانند ادامه پیدا کنند و نکته مهم اینکه job در systemd با job control مربوط به shell یکی نیست.

#### 🔹 Add a new unit

معمولاً unit محلی را در ```etc/systemd/system/``` قرار می‌ دهیم. بعد می‌ توان آن را start کرد :

```systemctl start myservice.service```

اگر unit دارای [Install] باشد و بخواهیم برای boot فعال شود :

```systemctl enable myservice.service```

نکته بسیار مهم start و enable دو مفهوم متفاوت هستند. start یعنی همین الان unit را فعال کن و enable یعنی configuration لازم را ایجاد کن تا unit در شرایط مشخص ، مثلاً هنگام boot ، به‌ صورت خودکار فعال شود. بنابراین :

```systemctl enable foo.service```

به‌ تنهایی لزوماً سرویس را همین لحظه start نمی‌ کند.

برای حذف از stop و disable استفاده و سپس در صورت نیاز unit file حذف می‌ شود :

```systemctl stop foo.service```

```systemctl disable foo.service```

---

### systemd Process Tracking and Synchronization

یکی از مشکلات مهم سیستم‌ های init قدیمی tracking process هاست : 

- یک سرویس ممکن است process اصلی را ایجاد کند.
- یک سرویس ممکن است ()fork کند.
- یک سرویس ممکن است daemonize شود.
- یک سرویس ممکن است process اولیه را terminate کند.
- یک سرویس ممکن است child process های زیادی بسازد.

بنابراین صرفاً نگه داشتن PID اولیه برای مدیریت سرویس کافی نیست .

قابلیت systemd برای tracking بهتر از قابلیت Kernel به نام ```cgroups``` استفاده می‌ کند. cgroup به systemd اجازه می‌ دهد process های مرتبط با یک سرویس را در یک گروه منطقی دنبال کند.

#### 🔹 Type=simple

در این مدل process سرویس fork نمی‌ کند و process اصلی همان process سرویس باقی می‌ ماند. 

```
[Service]
Type=simple
ExecStart=/usr/bin/example
```

مشکل این روش این است که systemd الزاماً نمی‌ داند process چه زمانی واقعاً initialization خودش را کامل کرده است. ممکن است process شروع شده باشد اما هنوز برای استفاده سرویس‌ های دیگر آماده نباشد.

#### 🔹 Type=forking

در این مدل سرویس fork می‌ کند و process اولیه terminate می‌ شود. systemd انتظار دارد process اولیه کنار برود و process daemon باقی بماند. ```Type=forking``` در این حالت systemd پایان process اولیه را به‌ عنوان نشانه آماده شدن سرویس در نظر می‌ گیرد.

#### 🔹 Type=notify

در این مدل خود سرویس زمانی که آماده شد به systemd اطلاع می‌ دهد و ```Type=notify``` این روش زمانی مفید است که لحظه واقعی آماده شدن سرویس اهمیت داشته باشد.

#### 🔹 Type=dbus

سرویس زمانی ready در نظر گرفته می‌شود که روی D-Bus ثبت شود.
#### 🔹 Type=oneshot

در oneshot ، process سرویس اجرا می‌ شود و در نهایت تمام می‌ شود. تفاوت مهم این است که systemd سرویس را تا زمان پایان process ، started در نظر نمی‌ گیرد. در این حالت RemainAfterExit=yes نیز می‌تواند باعث شود unit بعد از پایان process همچنان active در نظر گرفته شود.

#### 🔹 Type=idle

این حالت شبیه simple است ، اما systemd اجرای سرویس را تا پایان job های فعال دیگر به تأخیر می‌ اندازد. هدف اصلی این است که خروجی سرویس‌ های مختلف در زمان startup روی هم نیفتد و سرویس مورد نظر بعد از سایر startup job ها اجرا شود.

---

### systemd Dependencies

اگر dependency ها بیش از حد سخت‌ گیرانه تعریف شوند ، خرابی یک سرویس می‌ تواند باعث خرابی زنجیره‌ ای سایر بخش‌ ها شود چون dependency ها قلب مدل systemd هستند. مثلاً اگر login prompt به‌ شکل سخت‌ گیرانه به یک database وابسته شود و database خراب شود ، ممکن است حتی login کردن برای administrator هم مشکل پیدا کند به همین دلیل systemd چند نوع dependency مختلف دارد.

#### 🔹 Requires

این dependency سخت‌ گیرانه مثلا ```Requires=foo.service``` هنگام فعال کردن unit ، systemd تلاش می‌ کند dependency را نیز فعال کند. اگر dependency fail شود ، unit وابسته نیز می‌ تواند deactivate شود.

#### 🔹 Wants

مثلاً ```Wants=foo.service``` که systemd تلاش می‌ کند dependency را فعال کند ، اما failure آن معمولاً باعث شکست unit اصلی نمی‌ شود. این مدل برای بسیاری از dependency ها انعطاف‌ پذیرتر است. یکی از مزیت‌ های اصلی Wants این است که failure یک component غیر ضروری لزوماً کل startup را خراب نمی‌کند.

#### 🔹 Requisite

این dependency مثلاً ```Requisite=foo.service``` یعنی dependency باید از قبل active باشد. systemd در این حالت ابتدا وضعیت dependency را بررسی می‌ کند. اگر فعال نباشد ، activation unit اصلی fail می‌ شود.

#### 🔹 Conflicts

این dependency نوعی رابطه منفی است ```Conflicts=foo.service``` یعنی دو unit نباید هم‌ زمان فعال باشند. اگر یکی فعال شود ، systemd می‌ تواند دیگری را deactivate کند.

#### 🔹 View dependency

می‌توان dependency های مختلف را با systemctl بررسی کرد ، مثلاً ```systemctl show -p Wants unit``` یا ```systemctl show -p Requires unit```.

#### 🔹 Dependency is not the same as Ordering

این نکته بسیار مهم است وقتی می‌گوییم ```Wants=foo.service``` لزومی ندارد منظورمان این باشد که اول foo اجرا شود ، بعد unit فعلی اجرا شود. Dependency و ordering دو مفهوم جدا هستند برای ordering از ```Before``` و ```After``` استفاده می‌ شود مثلاً ```After=network.target``` یعنی unit فعلی باید بعد از unit مشخص‌ شده در ordering اجرا شود و ```Before=foo.service``` یعنی unit فعلی باید قبل از foo.service قرار بگیرد در نتیجه می‌ توان چیزی شبیه این داشت ```Wants=foo.service``` و ```After=foo.service``` که هم dependency ایجاد می‌ کند و هم ordering مشخصی تعریف می‌ کند.

#### 🔹 Default Dependencies

همه dependency ها الزاماً مستقیماً داخل unit file نوشته نشده‌ اند. systemd برای بعضی unit ها به‌ صورت خودکار dependency های پیش‌ فرض ایجاد می‌ کند. این dependency های خودکار بر اساس نوع unit متفاوت‌ اند. برای مثال target ها و service ها دقیقاً dependency های پیش‌ فرض یکسانی ندارند. این dependency ها در زمان کار systemd محاسبه می‌ شوند و لزوماً به شکل یک خط واضح داخل فایل unit دیده نمی‌ شوند. در صورت نیاز با ```DefaultDependencies=no``` می‌توان dependency های پیش‌ فرض را غیرفعال کرد.

#### 🔹 Conditional Dependencies

همچنین systemd علاوه بر dependency روی unit ها ، می‌تواند condition نیز بررسی کند مثلاً :

```ConditionPathExists=/some/path```

یعنی فقط اگر path وجود داشت unit قابل activation باشد. نمونه‌ های دیگر :

```ConditionPathIsDirectory=/some/path```

```ConditionFileNotEmpty=/some/file```

اگر condition برقرار نباشد ، خود unit فعال نمی‌ شود. نکته مهم این است که false بودن condition یک unit الزاماً جلوی activation سایر dependency های آن را نمی‌ گیرد.

#### 🔹 [Install] Section

روش دیگری برای مشخص کردن relationship بین unit ها استفاده از ```[Install]``` است مثلاً :

```
[Install]
WantedBy=multi-user.target
```

یعنی هنگام enable کردن unit، systemd می‌تواند link مناسبی در ساختار dependency ایجاد کند مثلاً :

```systemctl enable example.service```

ممکن است باعث ایجاد چیزی شبیه :

```/etc/systemd/system/multi-user.target.wants/example.service```

این link باعث می‌ شود dependency مورد نظر در boot بعدی نیز وجود داشته باشد. نکته مهم اینکه enable کردن unit آن را همان لحظه اجرا نمی‌ کند. اثر اصلی enable ایجاد configuration/link لازم برای activation های آینده است.

---

### systemd on demand and resource parallelized startup

یکی از قابلیت‌ های مهم systemd این است که می‌ تواند startup یک سرویس را تا زمانی که واقعاً لازم نشده ، به تأخیر بیندازد ایده کلی :

- اول Unit A سرویس را ارائه می‌ دهد.
- بعد Unit R resource مورد استفاده سرویس را نمایندگی می‌ کند.
- بعد systemdf ابتدا Unit R را در اختیار می‌ گیرد.
- وقتی کسی به resource مراجعه کرد ، systemd Unit A را فعال می‌ کند.
- بعد از آماده شدن Unit A ، resource به سرویس تحویل داده می‌ شود.

این resource می‌تواند چیزهایی مانند network socket و path و device باشد. این مدل شباهت زیادی به ایده‌ هایی دارد که قبلاً در ابزارهایی مانند inetd و xinetd و automount دیده می‌ شد.

#### 🔹 Socket Activation Example

فرض کنیم یک سرویس ساده echo روی TCP port 22222 داریم و یک socket unit می‌ تواند چیزی شبیه این باشد :

```
[Unit]
Description=echo socket

[Socket]
ListenStream=22222
Accept=true
```

اینجا socket resource نماینده port است سپس service unit متناظر ```echo@.service``` است مثلاً:

```
[Unit]
Description=echo service

[Service]
ExecStart=/bin/cat
StandardInput=socket
```

قابلیت systemd  ، به‌ دلیل naming convention میتواند بفهمد ```echo.socket``` با ```echo@.service``` ارتباط دارد. پس وقتی روی socket فعالیتی اتفاق بیفتد ، systemd می‌ تواند instance سرویس را ایجاد کند. برای فعال کردن socket :

```systemctl start echo.socket```

سپس می‌توان با ابزاری مانند ```telnet localhost 22222``` به آن متصل شد و داده‌ ای که client می‌ فرستد توسط سرویس echo برگردانده می‌ شود.

#### 🔹 Why is @ important ?

وجود ```echo@.service``` نشان می‌ دهد این unit یک template برای instance های متعدد است. اگر چند client هم‌ زمان متصل شوند ، systemd می‌ تواند برای هر connection یک instance ایجاد کند. مثلاً conceptually :

```
echo@instance1
echo@instance2
echo@instance3
```

در این مثال Accept=true باعث می‌ شود systemd connection را قبول کند و آن را به instance مناسب سرویس تحویل دهد البته این مدل برای تمام سرویس‌ های شبکه مناسب نیست بلکه بسیاری از network daemon ها منطق پیچیده‌ تری برای قبول و مدیریت connection دارند.

#### 🔹 Resource-Parallelized Startup

همین ایده فقط برای on-demand service استفاده نمی‌ شود. systemd می‌ تواند از resource unit برای **parallel کردن boot** نیز استفاده کند. فرض کنیم Service E یک resource مهم R را فراهم می‌ کند. در روش قدیمی :

```
Service E
   ↓
Service A
   ↓
Service B
   ↓
Service C
```

اگر A ، B و C همگی منتظر E باشند ، startup ترتیبی زمان زیادی می‌ برد. در این مدل :

```
Service E
    ↓
Resource R
 ↙   ↓   ↘
A    B    C
```
همچنین systemd می‌ تواند resource interface را سریع‌ تر آماده کند و هم‌ زمان E ، A ، B و C را شروع کند. بعد از آماده شدن E ، خود Service E کنترل resource را در اختیار می‌ گیرد. این باعث می‌ شود dependency ها الزاماً به یک sequence طولانی تبدیل نشوند. البته parallelization همیشه رایگان نیست و اگر تعداد زیادی unit هم‌ زمان شروع شوند ، ممکن است برای مدت کوتاهی CPU ، disk یا سایر منابع تحت فشار قرار گیرند.

<img width="100%" height="305" alt="image" src="https://github.com/user-attachments/assets/54cac2a7-b955-4fcd-ad8b-b5d042796c36" />

<img width="100%" height="377" alt="image" src="https://github.com/user-attachments/assets/0939a016-2949-4889-b0e0-c5bab4893970" />

---

### systemd Auxiliary Components

به مرور systemd از یک init system ساده فراتر رفته و بخش‌ های مختلف دیگری نیز در ecosystem آن قرار گرفته‌ اند. در مسیرهایی مانند ```lib/systemd/``` یا مسیرهای معادل در distribution های مختلف ، executable های مرتبط با این functionality ها دیده می‌ شوند. چند مورد مهم :

#### 🔹 systemd-udevd

نسخه integrated مربوط به udevd است که در مدیریت device ها نقش دارد.

#### 🔹 systemd-journald

سرویس logging مربوط به systemd است. journald پیام‌ های logging مختلف را دریافت و مدیریت می‌ کند و می‌ تواند با سیستم‌ های logging سنتی نیز تعامل داشته باشد. جزئیات journal در فصل‌ های بعدی بررسی می‌ شود.

#### 🔹 systemd-resolved

سرویسی برای name resolution و caching مربوط به DNS است البته استفاده از آن در همه distribution ها و همه installation ها یکسان نیست.

#### 🔹 systemd-fsck

برخی executable های systemd در واقع wrapper یا helper هایی هستند که ابزارهای سنتی سیستم را اجرا می‌کنند و نتیجه را به systemd گزارش می‌ دهند. برای مثال systemd-fsck می‌تواند filesystem check را در چارچوب systemd مدیریت کند.

---

### system V Runlevels

قبل از systemd ، یکی از مفاهیم مهم در System V init چیزی به نام ```runlevel``` بود. Runlevel وضعیت کلی سیستم را با عددی بین 0 تا 6 نمایش می‌ داد. سیستم معمولاً بیشتر زمان خود را در یک runlevel مشخص می‌ گذراند. هنگام shutdown نیز init می‌ توانست به runlevel دیگری تغییر وضعیت دهد تا سرویس‌ ها به‌ شکل مناسب متوقف شوند. برای مشاهده runlevel با استفاده از ```who -r``` مثلاً ```run-level 5``` را ممکن است ببینید.

معنای دقیق runlevel ها در distribution های مختلف همیشه یکسان نیست ، اما به‌ طور کلی :

- بعضی runlevel ها برای text console
- یک runlevel برای single-user
- برخی runlevel های خاص برای shutdown/reboot
- در بعضی سیستم‌ ها runlevel 5 برای graphical login

استفاده می‌ شد. در systemd ، runlevel ها عمدتاً legacy محسوب می‌ شوند و target ها مدل اصلی هستند.

---

### system V init

یکی از قدیمی‌ ترین مدل‌ های init در لینوکس System V init است. امروزه در بسیاری از desktop و server های جدید کمتر دیده می‌ شود ، اما دانستن آن همچنان مهم است چون سیستم‌ های قدیمی هنوز وجود دارند بعضی embedded system ها از آن استفاده می‌ کنند و package های قدیمی ممکن است script های System V داشته باشند و systemd برای compatibility از آن‌ها پشتیبانی می‌ کند.

ساختار اصلی System V init دو قسمت دارد ، یکی configuration مرکزی و دیگری مجموعه‌ ای از startup script ها و symbolic link ها است . فایل مرکزی معمولاً ```etc/inittab/``` است.

#### 🔹 /etc/inittab

هر entry در inittab چهار قسمت اصلی دارد ```id:runlevels:action:command``` مثلاً ```id:5:initdefault``` یعنی runlevel پیش‌ فرض ```5``` باشد. چهار field عبارت‌ اند از :

- Identifier
- Related runlevels
- Action
- Command Optional

#### 🔹 Important actions

1. **اکشن initdefault** : که runlevel پیش‌فرض را مشخص می‌کند.
2. **اکشن wait** : یک command را اجرا می‌کند و init برای ادامه کار منتظر پایان آن می‌ ماند.
3. **اکشن respawn** : اگر process تمام شود، دوباره آن را اجرا می‌ کند. برای login prompt روی virtual console بسیار مهم است. مثلاً: ```1:2345:respawn:/sbin/mingetty tty1``` باعث می‌ شود mingetty روی console مربوطه اجرا شود و بعد از logout مجدداً login prompt ایجاد شود.
4. **اکشن ctrlaltdel** : رفتار سیستم در برابر ```CTRL-ALT-DEL``` را مشخص می‌ کند.
5. **اکشن sysinit** : برای command هایی است که init باید در ابتدای startup اجرا کند ، قبل از ورود به runlevel ها.

---

### system V init startup command sequence

یک خط مهم در inittab ممکن است شبیه این باشد ```l5:5:wait:/etc/rc.d/rc 5``` قسمت ```5``` نشان می‌دهد runlevel مورد نظر 5 است. سپس rc به سراغ directory مربوط به runlevel می‌ رود. ممکن است این directory در ```etc/rc5.d/``` یا ```etc/rc.d/rc5.d/``` باشد.

در این directory entry هایی مانند ```S10sysklogd``` و ```S20ppp``` و ```S99httpd``` وجود دارند. حرف ```S``` یعنی start و عدد 00 تا 99 ترتیب اجرای script را مشخص می‌ کند. مثلاً S10 و S20 و S99 به همین ترتیب اجرا می‌ شوند. در مقابل ، K نشان‌ دهنده kill/stop است مثلا ```K20service``` و script با argument مناسب برای stop اجرا می‌ شود.

---

### system V init link farm

فایل‌ های داخل rc*.d معمولاً خود script واقعی نیستند. آن‌ ها symbolic link هایی به ```init.d``` هستند. مثلاً ```S99httpd -> ../init.d/httpd``` یعنی script اصلی ```init.d/httpd``` است. وجود تعداد زیادی symbolic link در چند directory مختلف را اصطلاحاً **link farm** می‌ نامند. مزیت این ساختار این است که یک script واحد می‌ تواند در runlevel های مختلف استفاده شود.

#### 🔹 Manually starting and stopping the service

به‌ جای دستکاری مستقیم link ها ، معمولاً script اصلی داخل init.d مورد استفاده قرار می‌ گیرد. مثلاً ```etc/init.d/httpd start/``` یا ```etc/init.d/httpd stop/``` البته مسیر دقیق و نام سرویس در distribution های مختلف ممکن است متفاوت باشد.

#### 🔹 Change boot sequence

در System V معمولاً تغییر boot sequence با تغییر link farm انجام می‌ شود. برای غیرفعال کردن یک سرویس ، یکی از روش‌ ها تغییر نام link است. مثلاً ```mv S99httpd _S99httpd``` چون دیگر filename با S یا K شروع نمی‌ شود ، rc آن را در sequence عادی اجرا نمی‌ کند ، اما نام اصلی هنوز مشخص می‌ کند این link قبلاً برای چه چیزی بوده است.

برای اضافه کردن سرویس نیز باید script سرویس ایجاد شود و در init.d قرار گیرد سپس link مناسب در runlevel مورد نظر ساخته شود و شماره مناسبی برای ordering انتخاب شود. برای سرویس‌ های غیرضروری ، قرار دادن آن‌ ها در بخش انتهایی sequence می‌تواند باعث شود بعد از سرویس‌ های اصلی سیستم اجرا شوند.

---

### run parts

ابزار run-parts یک utility است که مجموعه‌ ای از executable ها را از یک directory اجرا می‌ کند. ایده آن ساده است :

```
directory
   ↓
find executable files
   ↓
execute in predictable order
```

رفتار دقیق و امکانات run-parts بین distribution ها متفاوت است. برخی implementation ها بسیار ساده هستند و برخی امکان انتخاب فایل‌ ها با regular expression و ارسال argument به script ها را دارند. مثلاً ممکن است از pattern شبیه ```S[0-9]{2}``` برای انتخاب startup script ها استفاده شود.

> 💡 نکته اصلی : run-parts خودش یک init system نیست بلکه فقط ابزاری برای اجرای مجموعه‌ ای از program ها در یک directory است.

---

### system V init control

برای کنترل System V init ابزار ```telinit``` استفاده می‌ شود. برای تغییر runlevel از ```telinit 3``` و برای reload کردن configuration از ```telinit q``` و برای single-user mode از ```telinit s``` استفاده می‌ شود. تغییر runlevel می‌ تواند باعث terminate شدن process هایی شود که برای runlevel جدید تعریف نشده‌اند ، بنابراین باید با احتیاط انجام شود.

---

### systemd system V compatibility

در systemd برای اجرای script های قدیمی System V یک compatibility layer دارد. ایده کلی این است که systemd target مربوط به runlevel را فعال می‌ کند و link های موجود در rc<N>.d بررسی می‌ شوند سپس script متناظر در init.d پیدا می‌ شود و script به یک service unit نسبت داده می‌ شود بعد systemd script را با start یا stop اجرا می‌ کند و در نهایت systemd تلاش می‌ کند process های حاصل از script را نیز با service unit مرتبط کند.

مثلاً ```etc/init.d/foo/``` می‌ تواند به مفهومی مانند ```foo.service``` تبدیل شود در نتیجه می‌ توان از ```systemctl status foo.service``` برای مشاهده وضعیت استفاده کرد. اما compatibility به این معنا نیست که script قدیمی ناگهان تمام مزایای native systemd را به دست می‌ آورد. برای مثال  script های System V همچنان محدودیت‌ های مدل ترتیبی خودشان را دارند.

---

### shutting down your system

خاموش کردن سیستم نیز بخشی از وظایف init است. روش استاندارد برای shutdown استفاده از دستور ```shutdown``` است. برای halt فوری از ```shutdown -h now``` و برای reboot از ```shutdown -r now``` استفاده می شود پارامتر زمانی باید مشخص شود مثلاً ```shutdown -r +10``` یعنی reboot بعد از 10 دقیقه. در shutdown زمان‌ دار ، سیستم می‌ تواند login های جدید را نیز محدود کند.

#### 🔹 General shutdown steps

جزئیات implementation می‌ تواند متفاوت باشد ، اما روند کلی شامل این مراحل است :

1. **درخواست shutdown تمیز از process ها** : init به process ها فرصت می‌ دهد به‌ صورت مناسب terminate شوند.
2. **ارسال SIGTERM** : برای process هایی که terminate نشده‌ اند ```SIGTERM``` ارسال می‌ شود.
3. **ارسال SIGKILL** : اگر process هنوز باقی مانده باشد ، در نهایت ```SIGKILL``` استفاده می‌ شود.
4. **آماده‌ سازی filesystem ها** : سیستم وضعیت فایل‌ ها و resource ها را برای shutdown آماده می‌ کند.
5. **بعد unmount کردن filesystem ها** : filesystem های غیر از root هم unmount می‌ شوند.
6. **بعد remount کردن root به read-only** : فایل سیستم root به حالت read-only منتقل می‌ شود.
7. **بعد flush کردن داده‌ های buffer شده** : با ```sync``` داده‌ های buffer شده روی storage نوشته می‌ شوند.
8. **درخواست نهایی از Kernel** : در نهایت Kernel از طریق ```reboot(2)``` به reboot یا halt هدایت می‌ شود.

#### 🔹 reboot and halt and poweroff

این command ها بسته به وضعیت سیستم و نحوه فراخوانی ممکن است از مسیرهای متفاوتی عبور کنند. در حالت عادی معمولاً shutdown procedure را دنبال می‌ کنند. گزینه force مانند ```f-``` برای شرایط اضطراری وجود دارد ، اما باید با احتیاط بسیار زیاد استفاده شود چون bypass کردن clean shutdown می‌ تواند باعث از دست رفتن داده یا آسیب filesystem شود.

---

### initial RAM filesystem

یکی از قسمت‌ های مهم boot ، initramfs است. initramfs در واقع یک محیط user space موقت است که قبل از user space واقعی قرار می‌ گیرد. مشکل اصلی Kernel برای mount کردن root filesystem به driver مربوط به storage نیاز دارد. مثلاً ممکن است root filesystem روی RAID یا controller خاص یا storage پیچیده یا LVM یا hardware باشد که driver آن به‌ صورت module ارائه شده باشد. اما module خودش یک file است. اگر هنوز فایل سیستمی mount نشده باشد ، Kernel نمی‌ تواند به سادگی module مربوط به storage را از filesystem واقعی بخواند. این یک مشکل circular dependency ایجاد می‌ کند :

```
To mount root
↓
A driver is required
↓
The driver is a module on the filesystem
↓
To read the module, we must have a filesystem
↓
To mount the filesystem, we must have a driver initramfs solves this problem
```

#### 🔹 How does initramfs work ?

در Boot loader قبل از اجرای Kernel ، archive مربوط به initramfs را نیز در memory قرار می‌ دهد. سپس کرنل archive را دریافت می‌ کند و آن را به‌ عنوان filesystem موقت در RAM آماده می‌ کند و محیط initramfs را به‌ عنوان root موقت استفاده می‌ کند و init مربوط به initramfs را اجرا می‌ کند و  driver های لازم را load می‌ کند و storage واقعی را پیدا می‌ کند و root filesystem واقعی را mount می‌ کند و از محیط موقت به root واقعی منتقل می‌ شود سپس init واقعی سیستم را اجرا می‌ کند به این ترتیب circular dependency حل می‌ شود.

#### 🔹 init inside initramfs

برخی implementation ها متفاوت‌ اند. در بعضی سیستم‌ ها initramfs شامل یک shell script نسبتاً ساده است که udevd را اجرا می‌ کند و driver ها را load می‌ کند و root filesystem را پیدا می‌ کند آن را mount می‌ کند سپس init واقعی را اجرا می‌ کند در سیستم‌ هایی که systemd استفاده می‌ کنند ، ممکن است محیط initramfs پیچیده‌ تر باشد و اجزای بیشتری از systemd در آن وجود داشته باشد.

#### 🔹 Is initramfs always necessary ?

خیر ، initramfs همیشه لازم نیست اگر Kernel تمام driver های لازم برای دسترسی به root filesystem را مستقیماً در خود داشته باشد ، ممکن است بدون initramfs نیز بتواند root filesystem را mount کند. در چنین شرایطی می‌توان initramfs را از boot configuration حذف کرد. اما این کار در سیستم‌های واقعی همیشه ساده نیست و ویژگی‌ هایی مانند root روی storage پیچیده و RAID و LVM و  encrypted root و پیدا کردن root بر اساس UUID می‌ توانند نیاز به initramfs را افزایش دهند.

#### 🔹 Check initramfs

برای بررسی محتوای archive می‌ توان از ```unmkinitramfs``` استفاده کرد. ابزارهای رایج برای ساخت image نیز شامل ```mkinitramfs``` و ```dracut``` هستند. بسته به distribution ، ابزار و روش دقیق ساخت initramfs متفاوت است.

#### 🔹 initramfs vs initrd

 دو اصطلاح را باید از هم تشخیص داد. initramfs معمولاً بر پایه یک archive مانند ```cpio``` است و initrd روش قدیمی‌ تر است که بر پایه یک disk image برای initial RAM disk کار می‌ کرد. در عمل امروزه واژه initrd هنوز بسیار دیده می‌ شود حتی زمانی که فایل واقعی یک cpio-based initramfs است به همین دلیل ممکن است نام فایل یا configuration همچنان شامل initrd باشد.

#### 🔹 Moving from temporary root to real root

نزدیک پایان کار initramfs ، باید محیط موقت کنار گذاشته شود و root واقعی جایگزین شود. این مرحله معمولاً شامل یک نوع ```switch_root``` یا مکانیزم مشابه است. هدفش انتقال به filesystem واقعی و آزاد کردن محیط موقت و کاهش مصرف RAM و ادامه اجرای سیستم با root واقعی است.

---

### emergency booting and single user mode

وقتی سیستم به‌ درستی boot نمی‌ شود ، یکی از بهترین روش‌ ها استفاده از یک Live Image یا Rescue Image است. Live system می‌ تواند بدون استفاده از installation موجود روی disk ، Linux را اجرا کند. بسیاری از installation image های distribution ها قابلیت live environment نیز دارند.

#### 🔹 Things you can do in the Rescue/Live Environment

کارهایی که در Rescue/Live Environment می‌توان انجام داد از جمله : 

1. **بررسی *filesystem** : بعد از crash می‌ توان filesystem را بررسی و در صورت نیاز repair کرد.
2. **ریست کردن password** : اگر password فراموش شده باشد ، می‌توان از محیط نجات به filesystem نصب‌ شده دسترسی پیدا کرد و مشکل را اصلاح کرد. اصلاح فایل‌ های حیاتی مثلاً ```/etc/fstab``` و ```/etc/passwd``` و سایر configuration های ضروری.
3. **بازیابی backup** : اگر filesystem یا installation آسیب جدی دیده باشد ، می‌توان backup را بازیابی کرد.

#### 🔹 Single-User Mode

راه دیگر این است که سیستم را در حالت single-user بالا بیاوریم. هدف این حالت این است که سیستم با حداقل سرویس‌ ها به یک root shell برسد. در System V init ، single-user mode معمولاً ```runlevel 1``` است. در systemd ، مفهوم نزدیک به آن ```rescue.target``` است. برای ورود به این حالت می‌ توان پارامتر ```s-``` را در boot configuration به init منتقل کرد. در بعضی شرایط سیستم ممکن است برای ورود به single-user mode password کاربر root را درخواست کند.

#### 🔹 Single-user mode limitations

محیط Single-user mode برای تعمیر سریع مفید است ، اما محیط کاملی در اختیار شما قرار نمی‌ دهد. ممکن است network آماده نباشد یا GUI وجود نداشته باشد یا terminal محدود باشد یا سرویس‌ های معمول سیستم اجرا نشده باشند. به همین دلیل برای بسیاری از عملیات recovery ، Live/Rescue Image گزینه مناسب‌ تری است.

---
### Tips

تا اینجا دو مرحله بزرگ از زندگی یک Linux system را دیدیم :

```
Firmware
   ↓
Boot Loader
   ↓
Linux Kernel
   ↓
Kernel Initialization
   ↓
init
   ↓
User Space
```

در فصل پنجم تمرکز روی boot شدن Kernel بود. در این فصل تمرکز روی اتفاقاتی بود که بعد از اجرای اولین user-space process رخ می‌ دهند. مهم‌ ترین مفاهیمی که در این فصل دیدیم :

- init
- systemd
- System V init
- unit
- target
- service
- socket
- mount
- dependency
- ordering
- cgroup
- process tracking
- socket activation
- parallel startup
- runlevel
- inittab
- rc*.d
- init.d
- run-parts
- telinit
- shutdown
- initramfs
- single user mode
- rescue environment

مهم‌ ترین نکته این فصل این است که **شروع user space یک نقطه پایان برای boot نیست بلکه در واقع نقطه‌ ای است که سیستم از یک مسیر عمدتاً Kernel-controlled وارد یک ساختار بزرگ از process ها ، service ها ، configuration ها و dependency ها می‌ شود**.

در سیستم‌ های جدید ، systemd تلاش می‌ کند این ساختار را به‌ صورت یک graph از unit ها مدیریت کند ، process های سرویس‌ ها را دنبال کند ، dependency ها را کنترل کند و در صورت امکان startup را موازی و on-demand انجام دهد. در کنار آن ، شناخت System V init و initramfs همچنان برای troubleshooting و فهم سیستم‌ های قدیمی و recovery بسیار مهم است.
