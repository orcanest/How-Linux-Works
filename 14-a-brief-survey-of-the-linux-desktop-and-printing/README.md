## نگاهی سریع به دسکتاپ لینوکس و چاپگر
### 🐧 فصل چهاردهم کتاب How Linux Works



## A Brief Survey of the Linux Desktop and Printing
### 🌐 Chapter Fourteen of How Linux Works

فصل چهاردهم وارد بخشی از لینوکس می‌شود که نسبت به مباحثی مثل storage یا networking ساختار لایه‌ای و خطی کمتری دارد. در دسکتاپ لینوکس، معمولاً با مجموعه‌ای از componentهای مستقل روبه‌رو هستیم که هرکدام مسئولیت مشخصی دارند و با استفاده از protocolها، libraryها و IPC با یکدیگر ارتباط برقرار می‌کنند.

تمرکز اصلی این فصل روی شناخت اجزای اصلی یک desktop system است، نه آموزش کار با یک desktop environment خاص. فصل ابتدا مفهوم کلی نمایش گرافیکی و اجزای desktop را بررسی می‌کند، سپس وارد Wayland و X Window System می‌شود، ابزارهای تشخیصی ساده‌ای برای بررسی آن‌ها معرفی می‌کند، D-Bus را به‌عنوان یک مکانیزم مهم IPC بررسی می‌کند و در پایان معماری چاپ در Linux و CUPS را توضیح می‌دهد.

---
📚 Table of Contents

- [14.1 Desktop Components](#141-desktop-components)
  - [14.1.1 Framebuffers](#1411-framebuffers)
  - [14.1.2 The X Window System](#1412-the-x-window-system)
  - [14.1.3 Wayland](#1413-wayland)
  - [14.1.4 Window Managers](#1414-window-managers)
  - [14.1.5 Toolkits](#1415-toolkits)
  - [14.1.6 Desktop Environments](#1416-desktop-environments)
  - [14.1.7 Applications](#1417-applications)
- [14.2 Are You Running Wayland or X?](#142-are-you-running-wayland-or-x)
- [14.3 A Closer Look at Wayland](#143-a-closer-look-at-wayland)
  - [14.3.1 The Compositing Window Manager](#1431-the-compositing-window-manager)
    - [Window Decorations in Wayland](#window-decorations-in-wayland)
    - [Displays and Multiple Compositors](#displays-and-multiple-compositors)
  - [14.3.2 libinput](#1432-libinput)
  - [14.3.3 X Compatibility in Wayland](#1433-x-compatibility-in-wayland)
    - [Native Wayland Applications](#native-wayland-applications)
    - [Xwayland](#xwayland)
    - [Running Wayland Inside X](#running-wayland-inside-x)
- [14.4 A Closer Look at the X Window System](#144-a-closer-look-at-the-x-window-system)
  - [Identifying the X Server](#identifying-the-x-server)
  - [14.4.1 Display Managers](#1441-display-managers)
  - [14.4.2 Network Transparency](#1442-network-transparency)
  - [14.4.3 Ways of Exploring X Clients](#1443-ways-of-exploring-x-clients)
    - [xwininfo](#xwininfo)
  - [14.4.4 X Events](#1444-x-events)
  - [14.4.5 X Input and Preference Settings](#1445-x-input-and-preference-settings)
    - [Input Devices (General)](#input-devices-general)
    - [Device Properties](#device-properties)
    - [Mouse](#mouse)
    - [Keyboard and XKB](#keyboard-and-xkb)
    - [Desktop Background](#desktop-background)
    - [xset](#xset)
- [14.5 D-Bus](#145-d-bus)
  - [14.5.1 System and Session Instances](#1451-system-and-session-instances)
    - [System Instance](#system-instance)
    - [Session Instance](#session-instance)
  - [14.5.2 D-Bus Message Monitoring](#1452-d-bus-message-monitoring)
- [14.6 Printing](#146-printing)
  - [PostScript](#postscript)
  - [14.6.1 CUPS](#1461-cups)
  - [14.6.2 Format Conversion and Print Filters](#1462-format-conversion-and-print-filters)
- [14.7 Other Desktop Topics](#147-other-desktop-topics)
  - [Chromium OS and Chrome OS](#chromium-os-and-chrome-os)
- [Desktop Architecture and Summary](#desktop-architecture-and-summary)

---

### 14.1 Desktop Components

یکی از ویژگی‌های مهم Linux desktop انعطاف‌پذیری آن است. آنچه کاربر در نهایت به‌عنوان ظاهر و رفتار desktop می‌بیند، حاصل همکاری چندین component مختلف است و الزاماً یک نرم‌افزار واحد مسئول همه چیز نیست.

برخلاف بعضی قسمت‌های سیستم‌عامل که می‌توان آن‌ها را به‌صورت یک زنجیره نسبتاً مشخص از لایه‌ها تصور کرد، desktop بیشتر مجموعه‌ای از اجزای مستقل است. بعضی از این اجزا libraryهای مشترکی دارند و از طریق همین building blockهای مشترک با هم هماهنگ می‌شوند، اما architecture کلی بسیار آزادتر است.

در نسخه‌های جدیدتر Linux، این حوزه در وضعیت گذار میان X Window System و Wayland قرار گرفته است. X برای سال‌های طولانی پایه‌ی اصلی desktopهای Linux بود، ولی بسیاری از distributionها به سمت سیستم‌های مبتنی بر Wayland حرکت کرده‌اند.

#### 14.1.1 Framebuffers

در پایین‌ترین سطح یک سیستم نمایش گرافیکی، چیزی به نام framebuffer قرار دارد.

Framebuffer را می‌توان بخشی از حافظه در نظر گرفت که اطلاعات مربوط به تصویر در آن ذخیره می‌شود و سخت‌افزار گرافیکی آن اطلاعات را می‌خواند و برای نمایش روی صفحه استفاده می‌کند. در ساده‌ترین مدل ذهنی، هر قسمت از حافظه‌ی framebuffer متناظر با اطلاعات مربوط به یک یا چند pixel است. بنابراین اگر قرار باشد ظاهر تصویر تغییر کند، در نهایت باید مقادیر مناسب در framebuffer تغییر کنند.

اما همین‌جا یکی از مسائل اصلی windowing systemها مطرح می‌شود.

فرض کنید چند process مختلف داریم و هر process صاحب یک یا چند window است. هر برنامه به‌صورت مستقل بخش گرافیکی خودش را تولید می‌کند. کاربر هم باید بتواند پنجره‌ها را حرکت دهد، آن‌ها را روی هم بیندازد، بعضی را جلوتر از بقیه بیاورد و بخشی از صفحه را پوشش دهد.

در این شرایط دو مسئله اساسی وجود دارد:

- هر برنامه باید بداند دقیقاً کجا باید محتوای window خودش را رسم کند.
- یک برنامه نباید بتواند محتوای گرافیکی متعلق به window برنامه‌ی دیگر را خراب یا overwrite کند.

بنابراین یکی از مسئولیت‌های اصلی هر windowing system این است که بین bufferهای مربوط به برنامه‌های مختلف هماهنگی ایجاد کند و مشخص کند چه چیزی در نهایت باید روی framebuffer نمایش داده شود.

#### 14.1.2 The X Window System

مدل X Window System برای حل این مسئله یک X server مرکزی دارد.

X server را می‌توان تقریباً به‌عنوان هسته‌ی قدیمی desktop در نظر گرفت. این server مسئول کارهایی مانند مدیریت نمایش، rendering، تنظیم display و دریافت input از deviceهایی مثل keyboard و mouse است.

برنامه‌هایی مثل terminal window یا web browser در این مدل X client هستند. client به X server متصل می‌شود و درخواست‌هایی برای ساخت و نمایش window ارسال می‌کند.

X server در پاسخ، مسئولیت بخشی از کارهای مربوط به نمایش را بر عهده می‌گیرد:

- مشخص می‌کند windowها در کجا قرار بگیرند.
- مشخص می‌کند خروجی client در چه بخشی از نمایش render شود.
- داده‌ی گرافیکی را در مسیر رسیدن به framebuffer مدیریت می‌کند.
- eventهای input را به client مناسب می‌رساند.

نکته‌ی مهم این است که X server به‌تنهایی ظاهر یک برنامه را تعیین نمی‌کند. این خود X clientها هستند که UI برنامه را می‌سازند. بنابراین X server بیشتر زیرساختی است که clientها روی آن کار می‌کنند.

این architecture در زمان خودش بسیار انعطاف‌پذیر بود، اما از آنجا که X واسطه‌ی بسیار زیادی از کارهاست، می‌تواند یک bottleneck ایجاد کند. علاوه بر این، X سیستم بسیار قدیمی‌ای است که ریشه‌ی آن به دهه‌ی ۱۹۸۰ برمی‌گردد و طی سال‌ها قابلیت‌های زیادی به آن اضافه شده است؛ در نتیجه بخش‌هایی از آن شامل functionalityهایی است که دیگر در desktopهای مدرن اهمیت گذشته را ندارند.

#### 14.1.3 Wayland

Wayland برای حل همین مسائل با architecture متفاوتی طراحی شده است.

در Wayland یک display server مرکزی مشابه مدل X وجود ندارد که همه‌ی framebuffer operations را برای تمام clientها انجام دهد. در عوض، هر graphical client معمولاً buffer خودش را برای window خودش در اختیار دارد.

سپس یک component به نام compositor bufferهای مختلف clientها را با یکدیگر ترکیب می‌کند و نتیجه‌ی نهایی را برای نمایش روی framebuffer آماده می‌کند.

از آنجا که بسیاری از سیستم‌های گرافیکی امروزی از hardware acceleration استفاده می‌کنند، این مرحله می‌تواند بسیار کارآمد انجام شود.

از یک دید دیگر، بخشی از این ایده کاملاً جدید نیست. بسیاری از X clientهای مدرن سال‌هاست که بخش زیادی از تصویر خودشان را به‌صورت bitmap تولید می‌کنند و سپس آن را در اختیار X قرار می‌دهند. X نیز برای همین مدل، extensionهای compositing را در طول زمان اضافه کرده است.

بنابراین یکی از تفاوت‌های اصلی این است که در Wayland compositing از ابتدا بخشی اساسی از architecture محسوب می‌شود، در حالی که X این قابلیت را در طول زمان به سیستم خود اضافه کرده است.

برای مسیر input نیز بسیاری از سیستم‌های Wayland و حتی بسیاری از پیاده‌سازی‌های X از libinput استفاده می‌کنند تا eventهای deviceهای ورودی به شکلی استاندارد به بخش مناسب سیستم منتقل شوند. libinput جزئی اجباری از protocol خود Wayland نیست، ولی در desktopهای امروزی بسیار رایج است.

#### 14.1.4 Window Managers

Window manager مسئول بخش مهمی از تجربه‌ی کاربر در desktop است، زیرا مشخص می‌کند پنجره‌ها چگونه روی صفحه قرار بگیرند و چه رفتاری داشته باشند.

در X، window manager خودش یک client است که به X server متصل می‌شود. وظایفی مثل این‌ها را انجام می‌دهد:

- تنظیم position پنجره‌ها
- جابه‌جا کردن پنجره‌ها
- مدیریت title bar
- مدیریت دکمه‌هایی مثل close
- واکنش به eventهای مربوط به decoration
- درخواست از X server برای جابه‌جایی windowها

به عبارت دیگر، X server و window manager دو component جدا هستند.

در Wayland، این تفکیک تا حد زیادی از بین می‌رود. compositor عملاً همان بخش مرکزی‌ای است که نقش window manager را هم بر عهده دارد. compositor bufferهای clientها را ترکیب می‌کند، آن‌ها را در framebuffer نهایی قرار می‌دهد و eventهای input را نیز به مقصد مناسب هدایت می‌کند.

در نتیجه compositor در Wayland مسئولیت بیشتری نسبت به window manager سنتی X دارد.

در هر دو سیستم window managerهای متعددی وجود دارند، ولی X به‌دلیل قدمت بسیار بیشتر، تنوع بسیار بیشتری پیدا کرده است. همچنین پروژه‌هایی مانند Mutter در GNOME و KWin در KDE به‌گونه‌ای توسعه داده شده‌اند که از compositing در Wayland هم پشتیبانی کنند.

با وجود این تنوع، انتظار وجود یک استاندارد واحد برای window manager در Linux منطقی نیست، چون انتخاب window manager تا حد زیادی به نیاز و سلیقه‌ی کاربر بستگی دارد.

#### 14.1.5 Toolkits

برنامه‌های desktop معمولاً elementهای مشترکی مانند:

- button
- menu
- textbox
- widget

دارند.

اگر هر برنامه بخواهد همه‌ی این اجزا را از صفر پیاده‌سازی کند، هم توسعه دشوار می‌شود و هم ظاهر برنامه‌ها یکدست نخواهد بود.

به همین دلیل از graphical toolkit استفاده می‌شود. Toolkit مجموعه‌ای از componentها، libraryها و support fileها را فراهم می‌کند تا توسعه‌دهنده بتواند UI برنامه را سریع‌تر بسازد.

در Linux دو toolkit بسیار مهم عبارت‌اند از:

- GTK+
- Qt

Toolkitها معمولاً فقط مجموعه‌ای از libraryهای کدنویسی نیستند و می‌توانند شامل فایل‌های کمکی مانند imageها و اطلاعات مربوط به themeها نیز باشند.

#### 14.1.6 Desktop Environments

Toolkit به‌تنهایی همه‌ی نیازهای یک desktop را پوشش نمی‌دهد.

گاهی چند برنامه باید با هم همکاری کنند. مثلاً:

- یک برنامه بخواهد اطلاعاتی را با برنامه‌ی دیگری به اشتراک بگذارد.
- برنامه‌ها باید با notificationهای desktop هماهنگ شوند.
- منوها و titleها باید بر اساس conventionهای مشترک ظاهر شوند.
- برنامه‌ها باید رفتار مشخصی نسبت به eventهای سیستم داشته باشند.

برای ایجاد این هماهنگی، toolkitها و libraryهای دیگر در مجموعه‌های بزرگ‌تری به نام desktop environment قرار می‌گیرند.

نمونه‌های رایج desktop environment عبارت‌اند از:

- GNOME
- KDE
- Xfce

Desktop environment فقط toolkit نیست. معمولاً مجموعه‌ای از موارد زیر را نیز شامل می‌شود:

- iconها
- configurationها
- themeها
- design conventionها
- قوانین مربوط به ظاهر applicationها
- رفتار مشترک applicationها در برابر eventهای سیستم

به این ترتیب desktop environment تلاش می‌کند مجموعه‌ای از برنامه‌ها را از نظر تجربه‌ی کاربری تا حد ممکن یکپارچه کند.

#### 14.1.7 Applications

در بالاترین سطح desktop، خود applicationها قرار دارند.

نمونه‌ها:

- web browser
- terminal emulator
- file manager
- office suite
- utilityهای مختلف

این applicationها معمولاً برنامه‌های مستقلی هستند، اما در عمل باید نسبت به eventهای مهم سیستم و desktop آگاه باشند.

مثلاً یک application ممکن است بخواهد هنگام وقوع یکی از این eventها واکنش نشان دهد:

- اتصال یک storage device جدید
- دریافت email
- دریافت instant message
- تغییر وضعیت یک سرویس

برای این نوع ارتباط، applicationها معمولاً از interprocess communication استفاده می‌کنند و یکی از مهم‌ترین مکانیزم‌ها در این حوزه D-Bus است.

---

### 14.2 Are You Running Wayland or X?

برای بررسی اینکه session گرافیکی شما با Wayland اجرا می‌شود یا X، کتاب یک روش بسیار ساده ارائه می‌کند: مقدار environment variable مربوط به Wayland را بررسی کنید.

```
echo $WAYLAND_DISPLAY
```

اگر چیزی شبیه این ببینید:

```
wayland-0
```

احتمالاً session شما روی Wayland قرار دارد.

اگر این variable تنظیم نشده باشد، در محیط مورد بحث کتاب معمولاً می‌توان نتیجه گرفت که session شما X است؛ البته این آزمایش استثناهایی دارد و یک تشخیص مطلق نیست.

نکته‌ی مهم این است که Wayland و X الزاماً دو سیستم کاملاً مجزا و mutually exclusive نیستند.

اگر سیستم شما از Wayland استفاده کند، بسیار محتمل است که در کنار آن یک X compatibility server نیز در حال اجرا باشد تا applicationهای قدیمی X بتوانند همچنان کار کنند.

از طرف دیگر حتی می‌توان یک Wayland compositor را داخل یک X window اجرا کرد. با این حال چنین ترکیبی ممکن است باعث رفتارهای گیج‌کننده شود، چون برنامه‌هایی که از هر دو protocol پشتیبانی می‌کنند ممکن است compositor موجود را پیدا کنند و به‌جای session اصلی X به آن متصل شوند.

---

### 14.3 A Closer Look at Wayland

Wayland در زمان نگارش کتاب به‌عنوان فناوری در حال گسترش و جایگزینی تدریجی X معرفی می‌شود و در بسیاری از distributionها به‌صورت پیش‌فرض مورد استفاده قرار گرفته است.

با این حال یکی از تفاوت‌های مهم از نظر troubleshooting این است که در X ابزارهای بسیار بیشتری برای مشاهده و بررسی اجزای داخلی وجود دارد. Wayland طراحی جدیدتری دارد و ابزارهای تشخیصی command-line آن به گستردگی ابزارهای X نیستند.

اولین نکته‌ای که باید درباره Wayland روشن شود این است که Wayland نام یک desktop environment یا یک server بزرگ به سبک X نیست.

Wayland در اصل یک communications protocol میان یک graphical client و یک compositing window manager است.

بنابراین نباید انتظار داشته باشید با نصب چیزی به نام "Wayland core" یک desktop کامل در اختیار داشته باشید.

آنچه معمولاً در سیستم پیدا می‌کنید شامل libraryهایی برای صحبت کردن با protocol و مجموعه‌ای از componentهای مرتبط است.

در کنار آن Weston قرار دارد که یک reference compositing window manager برای Wayland است.

Reference بودن Weston به این معنا نیست که قرار است مثل GNOME یا KDE یک desktop environment عمومی باشد. Weston interface بسیار ساده‌ای دارد و بیشتر برای این منظور ساخته شده که توسعه‌دهندگان بتوانند implementationهای اصلی compositor را ببینند و از آن برای درک نحوه‌ی پیاده‌سازی بخش‌های مهم Wayland استفاده کنند.

#### 14.3.1 The Compositing Window Manager

در Wayland همیشه از ظاهر desktop نمی‌توان فهمید دقیقاً چه compositorی در حال اجراست.

برخلاف برخی سیستم‌های دیگر، یک محل استاندارد و ساده وجود ندارد که فقط با نگاه کردن به یک فایل مشخص بتوان نام compositor را پیدا کرد.

یکی از روش‌های قابل اتکاتر این است که Unix domain socket مربوط به Wayland را پیدا کنیم.

نام این socket از مقدار environment variable زیر مشخص می‌شود:

```
echo $WAYLAND_DISPLAY
```

مقدار معمول:

```
wayland-0
```

این socket معمولاً در مسیری مشابه این قرار دارد:

```
/run/user/<uid>/wayland-0
```

و اگر محل runtime directory را ندانید، می‌توانید مقدار زیر را بررسی کنید:

```
echo $XDG_RUNTIME_DIR
```

سپس از طریق ss می‌توان processای را که روی socket گوش می‌دهد پیدا کرد:

```
ss -xlp | grep wayland
```

در یک نمونه‌ی مطرح‌شده در کتاب، خروجی نشان می‌دهد که processای با نام gnome-shell و یک PID مشخص روی socket مربوط به Wayland در حال listening است.

اما اینجا یک لایه‌ی دیگر نیز وجود دارد. در GNOME، gnome-shell خودش implementation مستقل کامل compositor نیست؛ بلکه از Mutter به‌عنوان compositing window manager استفاده می‌کند. یعنی GNOME Shell در این معماری از Mutter به‌صورت library بهره می‌گیرد.

##### Window Decorations in Wayland

یکی از تفاوت‌های جالب X و Wayland در نحوه‌ی مدیریت window decorations است.

در X، window manager به‌طور سنتی decorationهایی مانند title bar را کنترل می‌کند.

در implementationهای ابتدایی Wayland، این مسئولیت بیشتر به خود applicationها سپرده شده بود و این موضوع می‌توانست باعث شود چند نوع متفاوت title bar و decoration روی یک desktop دیده شود.

بعدها protocolای به نام XDG-Decoration اضافه شد تا client و window manager بتوانند درباره‌ی این موضوع با یکدیگر مذاکره کنند.

در نتیجه client می‌تواند متوجه شود که آیا compositor حاضر است decoration مربوط به window را خودش رسم کند یا نه.

##### Displays and Multiple Compositors

در Wayland می‌توان display را همان فضای قابل مشاهده‌ای در نظر گرفت که در نهایت با framebuffer نمایش داده می‌شود.

یک display می‌تواند چند monitor را شامل شود.

همچنین در موارد خاص می‌توان بیش از یک compositor را هم‌زمان اجرا کرد، مثلاً اگر compositorها روی virtual terminalهای متفاوت قرار گرفته باشند. در چنین حالتی معمولاً نام displayها به‌ترتیب چیزی شبیه این خواهد بود:

```
wayland-0
wayland-1
```

برای به‌دست آوردن اطلاعاتی درباره‌ی interfaceهایی که compositor ارائه می‌کند می‌توان از:

```
weston-info
```

استفاده کرد.

با این حال نباید انتظار داشت این ابزار یک debugger کامل برای compositor باشد. اطلاعاتی که به‌دست می‌آورید بیشتر مربوط به display و input deviceهاست.

#### 14.3.2 libinput

برای آنکه input از deviceهای فیزیکی مثل keyboard یا mouse از kernel به graphical client برسد، compositor به مکانیزمی نیاز دارد که eventها را از device دریافت و استاندارد کند.

این وظیفه را در بسیاری از سیستم‌های جدید libinput انجام می‌دهد.

libinput به deviceهای ورودی kernel که در مسیرهایی مانند زیر دیده می‌شوند دسترسی دارد:

```
/dev/input/
```

و eventهای خام آن‌ها را پردازش می‌کند.

در Wayland، compositor معمولاً event خام kernel را بدون تغییر مستقیماً به client نمی‌فرستد. بلکه event را دریافت می‌کند، آن را به شکل مناسب درمی‌آورد و سپس آن را از طریق Wayland protocol به client مربوطه منتقل می‌کند.

یکی از مزیت‌های مهم libinput این است که فقط یک library نیست و utilityهای command-line نیز در اختیار کاربر قرار می‌دهد.

برای دیدن deviceهای ورودی:

```
libinput list-devices
```

ممکن است خروجی شامل deviceهایی مثل:

```
Device: Cypress USB Keyboard
Kernel: /dev/input/event3
Group: 6
Seat: seat0, default
Capabilities: keyboard
```

باشد.

این خروجی نشان می‌دهد که kernel device مثلاً یک keyboard را در مسیر /dev/input/event3 می‌شناسد.

برای مشاهده‌ی eventهای زنده نیز می‌توان از:

```
libinput debug-events --show-keycodes
```

استفاده کرد.

بعد از اجرای آن، با حرکت دادن mouse یا فشردن کلیدهای keyboard، eventهایی مانند device addition، press و release را خواهید دید.

این ابزار فقط مخصوص Wayland نیست. libinput یک سیستم برای دریافت و پردازش kernel input eventهاست و بنابراین در بسیاری از محیط‌های X نیز مورد استفاده قرار می‌گیرد.

#### 14.3.3 X Compatibility in Wayland

یکی از موانع مهم مهاجرت از X به Wayland، تعداد بسیار زیاد applicationهای قدیمی X است.

اگر قرار بود با ورود Wayland همه‌ی این برنامه‌ها کنار گذاشته شوند، انتقال به سیستم جدید بسیار دشوار می‌شد. برای همین دو رویکرد هم‌زمان شکل گرفته است.

##### Native Wayland Applications

روش اول این است که application مستقیماً از Wayland پشتیبانی کند.

بسیاری از برنامه‌های گرافیکی قدیمی X از toolkitهایی مثل toolkitهای GNOME و KDE استفاده می‌کنند. وقتی خود toolkit از Wayland پشتیبانی کند، انتقال application به حالت native Wayland آسان‌تر می‌شود.

در این حالت توسعه‌دهنده باید بخش‌هایی مثل:

- window decoration
- input configuration
- وابستگی‌های باقی‌مانده به کتابخانه‌های X

را بررسی کند.

بسیاری از applicationهای مهم این مسیر را طی کرده‌اند.

##### Xwayland

روش دوم اجرای applicationهای X از طریق یک compatibility layer است.

این لایه Xwayland نام دارد.

Xwayland در واقع یک X server کامل است که به‌عنوان Wayland client اجرا می‌شود.

در نتیجه applicationهای قدیمی X می‌توانند همچنان تصور کنند که با یک X server کار می‌کنند، در حالی که خود X server در لایه‌ی پایین‌تر به compositor Wayland متصل شده است.

Xwayland باید input eventها را ترجمه کند و bufferهای مربوط به windowهای X را نیز به‌شکل جداگانه مدیریت کند.

وجود این لایه می‌تواند کمی overhead ایجاد کند، ولی برای بیشتر applicationها این هزینه در عمل ناچیز است.

##### Running Wayland Inside X

حرکت برعکس به این سادگی نیست. یک Wayland client را نمی‌توان دقیقاً به همان روش مستقیماً روی X اجرا کرد.

با این حال می‌توان یک compositor مانند Weston را داخل یک X window اجرا کرد:

```
weston
```

در این حالت می‌توانید applicationهای Wayland را داخل compositor اجرا کنید و حتی با تنظیم درست Xwayland، applicationهای X را نیز داخل آن داشته باشید.

مشکل زمانی ظاهر می‌شود که چنین compositorای را باز بگذارید و دوباره به session معمول X برگردید.

Applicationهایی که هم X و هم Wayland را پشتیبانی می‌کنند ممکن است compositor Wayland را پیدا کنند و به آن متصل شوند.

بنابراین ممکن است برنامه‌ای که انتظار دارید در X باز شود، داخل compositor دیگر نمایش داده شود.

برای جلوگیری از این نوع رفتارهای عجیب، توصیه‌ی اصلی این است که به‌صورت هم‌زمان یک compositor داخلی Wayland را در کنار X session معمولی اجرا نکنید.

---

### 14.4 A Closer Look at the X Window System

در مقایسه با Wayland، X Window System از نظر تاریخی مجموعه‌ی بسیار بزرگ‌تری از componentها را شامل می‌شد.

در یک X system سنتی، مجموعه‌ای از این اجزا وجود داشت:

- X server
- client libraries
- X clients
- ابزارهای جانبی
- window managerها

با ظهور desktop environmentهایی مانند GNOME و KDE، نقش X به‌مرور محدودتر و مشخص‌تر شد و تمرکز آن بیشتر روی بخش‌های مرکزی مانند rendering و مدیریت input قرار گرفت.

#### Identifying the X Server

X server معمولاً با نام‌هایی مانند:

- X
- Xorg

دیده می‌شود.

با بررسی processها ممکن است command lineای شبیه این ببینید:

```
Xorg -core :0 -seat seat0 -auth /var/run/lightdm/root/:0 \
-nolisten tcp vt7 -novtswitch
```

در اینجا :0 یک X display identifier است.

یک display می‌تواند یک یا چند monitor را نمایندگی کند؛ یعنی چند monitor می‌توانند در یک display مشترک قرار داشته باشند و از keyboard و mouse مشترک استفاده کنند.

در processهای یک X session نیز variable زیر معمولاً به همین display identifier اشاره می‌کند:

```
echo $DISPLAY
```

مثلاً:

```
:0
```

از نظر تئوری display می‌تواند به screenهای جداگانه مانند این تقسیم شود:

```
:0.0
:0.1
```

ولی این مدل امروزه چندان متداول نیست، زیرا extensionهایی مثل RandR می‌توانند چند monitor را در قالب یک فضای مجازی بزرگ مدیریت کنند.

در Linux، یک X server روی virtual terminal اجرا می‌شود. مثلاً اگر در process arguments عبارت vt7 را ببینید، منظور virtual terminal مربوط به /dev/tty7 است.

همچنین می‌توان بیش از یک X server داشت؛ کافی است آن‌ها روی virtual terminalهای جداگانه اجرا شوند و هرکدام display identifier مخصوص خودشان را داشته باشند.

برای جابه‌جایی بین virtual terminalها نیز می‌توان از ترکیب کلیدهای CTRL-ALT-FN یا utilityای مثل chvt استفاده کرد.

#### 14.4.1 Display Managers

اگر X server را به‌تنهایی اجرا کنید، لزوماً desktop قابل استفاده‌ای در اختیار نخواهید داشت.

X server فقط زیرساخت display را فراهم می‌کند و نمی‌داند چه clientهایی باید اجرا شوند.

بنابراین اگر X را بدون clientهای لازم بالا بیاورید، ممکن است فقط یک صفحه‌ی خالی ببینید.

راه رایج‌تر استفاده از display manager است.

Display manager معمولاً:

- X server را راه‌اندازی می‌کند.
- صفحه‌ی login را نمایش می‌دهد.
- پس از login مجموعه‌ای از clientها را شروع می‌کند.
- session گرافیکی موردنظر کاربر را راه‌اندازی می‌کند.

از نمونه‌های مطرح‌شده در کتاب:

- gdm
- kdm
- lightdm

هستند.

مثلاً lightdm یک display manager عمومی‌تر است که می‌تواند sessionهای مختلفی مثل GNOME یا KDE را راه‌اندازی کند.

اگر نخواهید از display manager استفاده کنید، می‌توانید از virtual console با ابزارهایی مانند:

```
startx
```

یا:

```
xinit
```

یک X session را شروع کنید.

با این حال session حاصل معمولاً بسیار ساده‌تر از sessionای خواهد بود که از طریق display manager ایجاد می‌شود، چون startup mechanics و startup fileهای این دو روش یکسان نیستند.

#### 14.4.2 Network Transparency

یکی از ویژگی‌های تاریخی و مهم X، network transparency است.

در X، client و server از طریق یک protocol با یکدیگر صحبت می‌کنند. بنابراین در حالت سنتی می‌توان client را روی یک machine و X server را روی machine دیگری اجرا کرد.

در این مدل، X server می‌توانست به TCP port زیر گوش دهد:

```
6000
```

سپس client راه دور به آن متصل می‌شد و windowهای خود را روی display آن machine نمایش می‌داد.

مشکل این روش امنیت است.

این ارتباط در شکل سنتی معمولاً encryption مناسبی نداشت. به همین دلیل بسیاری از distributionها listening مستقیم X روی شبکه را غیرفعال کرده‌اند و X server را با optionای مانند:

```
-nolisten tcp
```

اجرا می‌کنند.

با این حال remote X application همچنان می‌تواند از طریق SSH tunneling مورد استفاده قرار گیرد. در این روش SSH مسیر امنی را برای اتصال فراهم می‌کند و امکان اجرای client راه دور را بدون باز کردن مستقیم TCP port مربوط به X فراهم می‌سازد.

Wayland در این زمینه با X تفاوت مهمی دارد. در Wayland هر client buffer خودش را مدیریت می‌کند و compositor باید به آن bufferها دسترسی داشته باشد؛ بنابراین model ساده‌ی remote X را نمی‌توان مستقیماً روی Wayland تکرار کرد.

سیستم‌های دیگری مثل RDP می‌توانند با compositor همکاری کنند و remote desktop functionality ایجاد کنند، ولی این مسئله از نظر معماری با network transparency سنتی X متفاوت است.

#### 14.4.3 Ways of Exploring X Clients

یکی از بخش‌های جالب X این است که حتی از command line هم می‌توانید وضعیت windowها و clientها را بررسی کنید.

##### xwininfo

یکی از ساده‌ترین utilityها:

```
xwininfo
```

است.

اگر آن را بدون argument اجرا کنید، از شما می‌خواهد یک window را با mouse انتخاب کنید.

بعد از انتخاب window اطلاعاتی مانند position و size آن نمایش داده می‌شود.

نمونه‌ای از اطلاعات مهم:

```
xwininfo: Window id: 0x5400024 "xterm"
Absolute upper-left X: 1075
Absolute upper-left Y: 594
```

در اینجا window ID اهمیت زیادی دارد.

X server و window manager از این شناسه برای شناسایی windowها استفاده می‌کنند.

برای دریافت فهرستی از X clientها و windowهای موجود می‌توانید از:

```
xlsclients -l
```

استفاده کنید.

#### 14.4.4 X Events

X clientها برای دریافت input و اطلاع از تغییر وضعیت server از eventها استفاده می‌کنند.

این eventها asynchronous هستند؛ یعنی client لازم نیست دائماً وضعیت input device را poll کند. X server event را از source دریافت می‌کند و آن را برای clientهایی که به آن علاقه‌مند هستند ارسال می‌کند.

این مدل شباهت مفهومی زیادی با event systemهایی مانند:

- udev events
- D-Bus events

دارد.

برای مشاهده‌ی eventهای X می‌توانید از:

```
xev
```

استفاده کنید.

اجرای xev یک window باز می‌کند. اگر mouse را داخل آن حرکت دهید، click کنید یا key بزنید، خروجی command line شامل eventهای دریافتی نمایش داده می‌شود.

برای مثال event مربوط به حرکت mouse اطلاعاتی درباره‌ی coordinateها دارد:

```
MotionNotify event ...
```

دو جفت مختصات در این خروجی مهم هستند:

- مختصات pointer در داخل window
- مختصات pointer نسبت به کل display

برای مثال:

```
(47,174)
root:(1692,486)
```

مختصات اول مربوط به window و مختصات دوم مربوط به کل display است.

X eventها فقط حرکت mouse نیستند. eventهایی مثل:

- key press
- button click
- ورود mouse به window
- خروج mouse از window
- focus گرفتن window
- focus از دست دادن window

نیز وجود دارند.

یکی از کاربردهای مهم xev استخراج اطلاعات keyboard است.

مثلاً با فشردن یک کلید می‌توانید چیزی شبیه این ببینید:

```
KeyPress event ...
keycode 46 (keysym 0x6c, l)
```

در اینجا keycode شناسه‌ی مربوط به کلید در X است و keysym مشخص می‌کند آن کلید از نظر معنایی چه کاراکتر یا symbolای را نشان می‌دهد.

همچنین می‌توانید یک window موجود را با -id هدف بگیرید:

```
xev -id 0x5400024
```

شناسه‌ی 0x... را می‌توانید از xwininfo به‌دست آورید.

#### 14.4.5 X Input and Preference Settings

یکی از گیج‌کننده‌ترین بخش‌های X این است که برای یک تنظیم مشخص ممکن است چند روش مختلف وجود داشته باشد و همه‌ی این روش‌ها لزوماً در هر desktop environment به یک شکل کار نکنند.

مثلاً برای تغییر رفتار CAPS LOCK و تبدیل آن به CTRL، روش‌های مختلفی وجود دارد.

علت این پیچیدگی این است که چند لایه‌ی مختلف ممکن است در تنظیمات input دخالت داشته باشند:

- X server
- X Input Extension
- XKB
- xmodmap
- desktop environment

پس قبل از تغییر تنظیم باید بدانید دقیقاً کدام component مسئول آن feature است، چون desktop environment ممکن است تنظیمات خودش را اعمال یا override کند.

##### Input Devices (General)

X از X Input Extension برای مدیریت input deviceهای مختلف استفاده می‌کند.

در سطح پایه، دو نوع device اصلی مطرح می‌شوند:

- keyboard
- pointer

می‌توانید چند device از یک نوع به سیستم متصل کنید.

برای هماهنگ کردن چند keyboard یا چند pointer، X مفهوم virtual core device را ارائه می‌کند.

برای مشاهده‌ی deviceها:

```
xinput --list
```

خروجی ممکن است چیزی شبیه این باشد:

```
Virtual core pointer id=2 [master pointer (3)]
  ↳ Virtual core XTEST pointer id=4 [slave pointer (2)]
  ↳ Logitech Unifying Device id=8 [slave pointer (2)]

Virtual core keyboard id=3 [master keyboard (2)]
  ↳ Virtual core XTEST keyboard id=5 [slave keyboard (3)]
  ↳ Power Button id=6 [slave keyboard (3)]
  ↳ Power Button id=7 [slave keyboard (3)]
  ↳ Cypress USB Keyboard id=9 [slave keyboard (3)]
```

در این مدل:

- master device نقش virtual core device را دارد.
- slave deviceهای واقعی هستند که input را تولید می‌کنند.

مثلاً IDهای 2 و 3 در این نمونه deviceهای core هستند و deviceهایی مانند keyboard واقعی IDهای دیگر دارند.

نکته‌ی مهم این است که power button خود machine نیز از دید X می‌تواند یک input device محسوب شود.

بیشتر X clientها فقط به core device گوش می‌دهند و لازم نیست بدانند event دقیقاً از کدام keyboard یا mouse فیزیکی آمده است.

ولی یک application می‌تواند از X Input Extension استفاده کند تا یک device خاص را انتخاب کند.

##### Device Properties

هر input device مجموعه‌ای از propertyها دارد.

برای مشاهده‌ی propertyهای یک device:

```
xinput --list-props 8
```

می‌توانید propertyهایی مثل این را مشاهده کنید:

```
Device Enabled
Coordinate Transformation Matrix
Device Accel Profile
Device Accel Constant Deceleration
Device Accel Adaptive Deceleration
Device Accel Velocity Scaling
```

برخی از این propertyها را می‌توان با:

```
xinput --set-prop
```

تغییر داد.

در نتیجه xinput فقط ابزار نمایش deviceها نیست و برای تغییر بعضی تنظیمات runtime نیز استفاده می‌شود.

##### Mouse

برای mouse و pointer optionهای مختلفی وجود دارد.

می‌توان بعضی propertyها را مستقیماً تغییر داد، اما برای بعضی کارهای متداول، optionهای اختصاصی‌تر راحت‌تر هستند.

مثلاً برای تغییر mapping دکمه‌های mouse:

```
xinput --set-button-map dev 3 2 1
```

در نمونه‌ی کتاب این کار ترتیب سه دکمه‌ی mouse را معکوس می‌کند و می‌تواند برای استفاده‌ی left-handedها مفید باشد.

##### Keyboard and XKB

Keyboard configuration در X پیچیده‌تر است، مخصوصاً چون layoutهای بین‌المللی بسیار متنوع‌اند.

X از ابتدا قابلیت mapping داخلی برای keyboard داشت و utility قدیمی‌تر:

```
xmodmap
```

برای تغییر mapping استفاده می‌شد.

اما در سیستم‌های نسبتاً جدیدتر، XKB (X Keyboard Extension) روش اصلی و قدرتمندتر برای مدیریت keyboard mapping است.

XKB از نظر ساختاری پیچیده است، به همین دلیل حتی در سیستم‌های جدید نیز بعضی کاربران برای تغییرات سریع هنوز سراغ xmodmap می‌روند.

ایده‌ی اصلی XKB این است که یک keyboard map تعریف کنید، آن را با:

```
xkbcomp
```

compile کنید و سپس با:

```
setxkbmap
```

آن را در X server فعال کنید.

یکی از ویژگی‌های جالب XKB این است که لازم نیست همیشه کل keyboard map را از صفر تعریف کنید. می‌توانید یک partial map بسازید که روی mapping موجود اعمال شود.

این قابلیت برای تغییراتی مثل تبدیل CAPS LOCK به CTRL مفید است و بسیاری از graphical keyboard preference utilityها در desktop environmentها از همین ایده استفاده می‌کنند.

ویژگی دیگر این است که می‌توانید برای keyboardهای مختلف mappingهای جداگانه داشته باشید.

##### Desktop Background

در X، root window در اصل background اصلی display محسوب می‌شود.

ابزار قدیمی:

```
xsetroot
```

می‌تواند ویژگی‌هایی مثل رنگ background مربوط به root window را تغییر دهد.

اما در desktopهای مدرن، root window معمولاً مستقیماً دیده نمی‌شود. بسیاری از desktop environmentها یک window بزرگ در پشت سایر windowها قرار می‌دهند تا قابلیت‌هایی مثل:

- wallpaper پویا
- desktop file browsing
- interactionهای مربوط به desktop

را پیاده کنند.

به همین دلیل تغییر root window با xsetroot ممکن است روی ظاهر واقعی desktop تأثیر قابل‌مشاهده‌ای نداشته باشد.

در بعضی محیط‌ها می‌توان از ابزارهایی مثل gsettings برای تغییر background استفاده کرد، ولی این موضوع دیگر به زیرساخت قدیمی root window محدود نیست.

##### xset

یکی دیگر از ابزارهای قدیمی X:

```
xset
```

است.

این utility امروزه کمتر استفاده می‌شود، ولی می‌توانید برای دیدن وضعیت بعضی تنظیمات از:

```
xset q
```

استفاده کنید.

یکی از بخش‌های مفید آن تنظیمات مربوط به:

- screen saver
- DPMS (Display Power Management Signaling)

است.

---

### 14.5 D-Bus

یکی از فناوری‌های مهمی که در Linux desktop رشد کرده D-Bus یا Desktop Bus است.

D-Bus یک message-passing system است که برای IPC استفاده می‌شود.

یعنی processها می‌توانند از طریق آن به یکدیگر پیام بدهند و applicationها می‌توانند از eventهای مربوط به سیستم یا برنامه‌های دیگر مطلع شوند.

برای مثال، وقتی یک USB storage device متصل می‌شود، یک component می‌تواند event مربوط به آن را تشخیص دهد و processهای دیگری که به آن event علاقه دارند مطلع شوند.

در نگاه ساده، خود D-Bus شامل library و protocolای است که ارتباط بین processها را استاندارد می‌کند. بدون بخش مرکزی D-Bus، این قابلیت تقریباً چیزی شبیه یک IPC پیشرفته‌تر بر پایه‌ی mechanismهایی مثل Unix domain socket خواهد بود.

قسمت مهم ماجرا dbus-daemon است.

dbus-daemon مانند یک central hub عمل می‌کند:

- processها به آن متصل می‌شوند.
- clientها مشخص می‌کنند به چه نوع eventهایی علاقه دارند.
- processهای دیگر event ایجاد می‌کنند.
- dbus-daemon پیام را به گیرنده‌های مناسب منتقل می‌کند.

یک مثال مهم در کتاب udisks-daemon است. این process eventهای مربوط به disk را از udev دریافت می‌کند و سپس آن‌ها را از طریق D-Bus به applicationهایی که به چنین eventهایی علاقه‌مند هستند منتقل می‌کند.

به همین دلیل D-Bus فقط یک روش ساده برای اجرای remote procedure نیست، بلکه به یک backbone مهم برای communication بین قسمت‌های مختلف desktop و حتی بخش‌هایی از system تبدیل شده است.

#### 14.5.1 System and Session Instances

D-Bus دیگر فقط یک ابزار مربوط به desktop نیست و در بخش‌های مختلف Linux system مورد استفاده قرار گرفته است.

حتی serviceهایی مثل systemd و در نسخه‌های قدیمی‌تر Upstart نیز می‌توانند از D-Bus برای ارتباط استفاده کنند.

اما یک design principle مهم مطرح می‌شود:

> اجزای core system نباید بدون دلیل به ابزارهای خاص desktop وابسته شوند.

برای همین D-Bus معمولاً به دو نوع instance تقسیم می‌شود.

##### System Instance

System bus در سطح کل سیستم فعالیت می‌کند.

این instance هنگام boot و توسط init system راه‌اندازی می‌شود و در کتاب با option زیر معرفی شده است:

```
--system
```

این instance معمولاً با یک user مخصوص D-Bus اجرا می‌شود.

فایل configuration آن معمولاً در این مسیر قرار دارد:

```
/etc/dbus-1/system.conf
```

و کتاب توصیه می‌کند بدون دلیل آن را تغییر ندهید.

ارتباط با system bus از طریق Unix domain socketای مانند:

```
/var/run/dbus/system_bus_socket
```

انجام می‌شود.

این bus مربوط به یک user خاص یا یک desktop session خاص نیست و برای سرویس‌های system-wide کاربرد دارد.

##### Session Instance

در کنار system bus، یک session bus نیز وجود دارد.

Session bus مستقل از system instance است و هنگام شروع یک desktop session ایجاد می‌شود.

Applicationهای desktop مربوط به یک session به این bus متصل می‌شوند.

بنابراین به‌صورت مفهومی:

```
System Bus
    |
    +-- System-wide services
    +-- Hardware/system events
    +-- Services shared by the whole machine


Session Bus
    |
    +-- Desktop applications
    +-- User-session events
    +-- Per-user desktop communication
```

این جداسازی باعث می‌شود communication مربوط به desktop کاربر با communication سطح system یکی نشود.

#### 14.5.2 D-Bus Message Monitoring

یکی از بهترین راه‌ها برای درک تفاوت system bus و session bus مشاهده‌ی eventهای واقعی روی آن‌هاست.

برای system bus:

```
dbus-monitor --system
```

بعد از اجرای این command ممکن است پیام‌هایی شبیه این ببینید:

```
signal sender=org.freedesktop.DBus
-> dest=:1.952
serial=2
path=/org/freedesktop/DBus
interface=org.freedesktop.DBus
member=NameAcquired
```

پیام ابتدایی بیشتر به خود monitor مربوط است که به bus متصل شده و یک name دریافت کرده است.

اگر فقط همین command را اجرا کنید، ممکن است برای مدتی activity زیادی نبینید، چون system bus معمولاً خیلی شلوغ نیست.

برای تولید event واقعی می‌توانید مثلاً یک USB storage device متصل کنید و فعالیت bus را مشاهده کنید.

در مقابل، session bus معمولاً بسیار پرتحرک‌تر است:

```
dbus-monitor --session
```

حالا اگر یک desktop application مثل file manager را باز یا استفاده کنید، در یک desktop D-Bus-aware احتمالاً مجموعه‌ی زیادی از پیام‌ها را خواهید دید.

البته همه‌ی applicationها الزاماً برای هر نوع operation پیام D-Bus تولید نمی‌کنند.

---

### 14.6 Printing

چاپ در Linux یک عملیات یک‌مرحله‌ای نیست.

وقتی از یک application درخواست print می‌کنید، معمولاً سند از چند مرحله عبور می‌کند تا در نهایت به printer برسد.

مسیر کلی که کتاب توضیح می‌دهد شامل این مراحل است:

1. application در صورت نیاز document را به PostScript تبدیل می‌کند.
2. document به print server ارسال می‌شود.
3. print server آن را داخل print queue قرار می‌دهد.
4. وقتی نوبت document برسد، print server آن را به print filter می‌فرستد.
5. اگر document در قالب PostScript نباشد، filter می‌تواند آن را convert کند.
6. اگر printer مستقیماً PostScript را پشتیبانی نکند، printer driver document را به format مناسب printer تبدیل می‌کند.
7. driver می‌تواند instructionهایی مثل paper tray و duplexing را به job اضافه کند.
8. در نهایت print server از یک backend برای ارسال document به printer استفاده می‌کند.

بنابراین مسیر ساده‌شده چنین است:

```
Application
    |
    v
Document
    |
    v
Print Server
    |
    v
Print Queue
    |
    v
Print Filter
    |
    v
Printer Driver
    |
    v
Backend
    |
    v
Printer
```

البته همه‌ی documentها الزاماً از تمام این مراحل عبور نمی‌کنند. مثلاً تبدیل به PostScript در مرحله‌ی اول اختیاری است و بسته به application و workflow می‌تواند متفاوت باشد.

#### PostScript

یکی از قسمت‌های عجیب معماری سنتی چاپ این است که چرا PostScript تا این اندازه مهم است.

PostScript فقط یک data format ساده نیست؛ در واقع یک programming language است.

یعنی وقتی یک document در قالب PostScript ارسال می‌شود، در اصل یک برنامه به مقصد ارسال شده که سیستم چاپ یا printer آن را پردازش می‌کند تا تصویر نهایی صفحه تولید شود.

PostScript در Unix-like systems نقش یک استاندارد تاریخی برای چاپ داشته است، همان‌طور که formatهایی مثل .tar در زمینه‌ی archiveها یک استاندارد رایج هستند.

در سیستم‌های جدیدتر بعضی applicationها از PDF به‌عنوان خروجی استفاده می‌کنند. با این حال تبدیل PDF به مسیرهای مناسب چاپ نسبتاً ساده است و به همین دلیل PDF نیز به‌خوبی در این ecosystem قرار می‌گیرد.

#### 14.6.1 CUPS

CUPS مخفف Common UNIX Printing System است و سیستم چاپ استاندارد در بسیاری از سیستم‌های Linux و دیگر Unix-likeها محسوب می‌شود.

CUPS همچنین در macOS استفاده شده است.

daemon اصلی CUPS:

```
cupsd
```

نام دارد.

برای ارسال ساده‌ی فایل می‌توان از clientای مثل:

```
lpr
```

استفاده کرد.

یکی از ویژگی‌های مهم CUPS پشتیبانی از IPP (Internet Printing Protocol) است.

IPP امکان انجام transactionهای شبیه HTTP میان client و server را فراهم می‌کند و در مدل مطرح‌شده در کتاب روی این TCP port کار می‌کند:

```
631
```

اگر CUPS روی سیستم شما در حال اجرا باشد، می‌توانید معمولاً interface وب محلی آن را از طریق:

```
http://localhost:631/
```

ببینید.

این interface می‌تواند برای مشاهده‌ی configuration، printerها و jobهای چاپ مفید باشد.

بسیاری از network printerها و print serverها نیز از IPP پشتیبانی می‌کنند و همین موضوع configuration printerهای راه دور را آسان‌تر می‌کند.

با این حال کتاب هشدار می‌دهد که نباید فرض کنید interface وب CUPS همیشه برای administration مناسب است. configuration پیش‌فرض لزوماً با این هدف طراحی نشده که remote administration امن و بدون تغییر در اختیار همه باشد.

به‌همین دلیل distribution معمولاً یک graphical settings interface برای افزودن و تغییر printerها ارائه می‌کند.

این ابزارها configurationهای CUPS را مدیریت می‌کنند و فایل‌های آن‌ها معمولاً در:

```
/etc/cups
```

قرار دارند.

از آنجا که configuration چاپ می‌تواند پیچیده باشد، بهتر است حتی در صورت نیاز به configuration دستی، ابتدا printer را از طریق ابزار گرافیکی distribution بسازید تا یک configuration پایه و قابل بررسی داشته باشید.

#### 14.6.2 Format Conversion and Print Filters

همه‌ی printerها نمی‌توانند مستقیماً PostScript یا PDF را پردازش کنند.

خصوصاً بسیاری از printerهای ارزان‌تر، format نهایی خاص خودشان را نیاز دارند.

در چنین شرایطی سیستم چاپ Linux باید document را به formatی تبدیل کند که printer بتواند آن را بفهمد.

CUPS در این مسیر از Raster Image Processor (RIP) استفاده می‌کند.

RIP document را به شکل bitmap مناسب برای چاپ آماده می‌کند.

در معماری توضیح‌داده‌شده در کتاب، RIP معمولاً از Ghostscript استفاده می‌کند:

```
gs
```

Ghostscript بخش قابل‌توجهی از conversion واقعی را انجام می‌دهد.

اما ایجاد bitmap به‌تنهایی کافی نیست؛ bitmap نهایی باید مطابق قابلیت‌ها و محدودیت‌های printer باشد.

اینجاست که PPD اهمیت پیدا می‌کند.

PPD مخفف:

```
PostScript Printer Definition
```

است.

PPD اطلاعات مربوط به ویژگی‌ها و قابلیت‌های printer را در اختیار driver قرار می‌دهد، از جمله مواردی مانند:

- resolution
- paper size
- تنظیمات چاپ
- قابلیت‌های خاص printer

بنابراین workflow ساده‌شده این بخش می‌تواند چنین در نظر گرفته شود:

```
Document
   |
   v
Print Filter
   |
   v
RIP / Ghostscript
   |
   v
Bitmap / Printer-oriented data
   |
   v
Printer Driver
   |
   v
Printer Backend
```

در این میان PPD نقش metadata مربوط به قابلیت‌های printer را دارد و به driver کمک می‌کند conversion را مطابق device مقصد انجام دهد.

---

### 14.7 Other Desktop Topics

یکی از خصوصیات جالب Linux desktop این است که مجبور نیستید همه‌ی اجزای desktop را به‌عنوان یک مجموعه‌ی غیرقابل‌تفکیک انتخاب کنید.

می‌توانید بخش‌های مختلف را از پروژه‌ها و componentهای گوناگون انتخاب کنید، چیزهایی را که دوست دارید نگه دارید و اجزایی را که نمی‌پسندید جایگزین کنید.

این انعطاف‌پذیری یکی از دلایل تنوع زیاد پروژه‌های desktop در Linux است.

کتاب برای آشنایی با پروژه‌ها و ecosystemهای مختلف desktop به منابع و جامعه‌ی freedesktop.org اشاره می‌کند؛ جایی که پروژه‌های مرتبط با desktop Linux، protocolها و infrastructureهای مشترک زیادی را می‌توان دنبال کرد.

#### Chromium OS and Chrome OS

یکی دیگر از تحولات مهمی که کتاب به آن اشاره می‌کند Chromium OS و نسخه‌ی تجاری آن یعنی Chrome OS است.

این سیستم نیز یک Linux system است و بخش زیادی از technologyهای desktop Linux را استفاده می‌کند، اما تجربه‌ی کاربری آن به‌شکل متفاوتی سازمان‌دهی شده است.

در این مدل، محور اصلی محیط کاربری:

```
Chromium / Chrome browser
```

است.

بخش قابل‌توجهی از چیزهایی که در یک traditional desktop Linux می‌بینید در Chrome OS حذف یا ساده شده‌اند.

در نتیجه می‌توان Chrome OS را نمونه‌ای از این دانست که چگونه همان زیرساخت Linux می‌تواند در قالب یک desktop بسیار متفاوت از GNOME یا KDE استفاده شود.

---

### Desktop Architecture and Summary

فصل چهاردهم در نهایت یک نکته‌ی معماری مهم را روشن می‌کند: Linux desktop یک برنامه‌ی واحد نیست، بلکه مجموعه‌ای از componentهای مستقل است که هرکدام مسئولیت مشخصی دارند و از طریق interfaceها، protocolها، libraryها و IPC با یکدیگر همکاری می‌کنند.

در مسیر نمایش، می‌توان معماری را به‌شکل زیر در نظر گرفت:

```
Application
    |
    v
Toolkit
    |
    v
Wayland / X Client
    |
    v
Compositor / X Server
    |
    v
Framebuffer
    |
    v
Display Hardware
```

در مدل X، یک X server مرکزی نقش اصلی را در مدیریت display، rendering و input دارد و applicationها به‌عنوان client با آن ارتباط برقرار می‌کنند.

در مدل Wayland، خود protocol مسئول ایجاد یک desktop کامل نیست. graphical clientها bufferهای خودشان را دارند و compositor آن‌ها را ترکیب می‌کند و خروجی نهایی را برای display آماده می‌سازد. به همین دلیل در Wayland، compositor نقش بسیار مرکزی‌تری دارد.

در مسیر input، تصویر متفاوتی داریم:

```
Keyboard / Mouse
        |
        v
Kernel Input Subsystem
        |
        v
libinput / X Input
        |
        v
Compositor / X Server
        |
        v
Application
```

در اینجا libinput می‌تواند eventهای input deviceهای kernel را جمع‌آوری و استاندارد کند، در حالی که در X ابزارهایی مانند X Input Extension، xinput، XKB و xmodmap برای مشاهده و تنظیم input مورد استفاده قرار می‌گیرند.

در سطح application، toolkitهایی مانند GTK+ و Qt امکانات لازم برای ساخت widgetهای گرافیکی را فراهم می‌کنند. سپس desktop environmentهایی مانند GNOME، KDE و Xfce مجموعه‌ی بزرگ‌تری از toolkitها، libraryها، themeها، iconها و conventionها را در کنار هم قرار می‌دهند تا applicationها رفتار و ظاهر هماهنگ‌تری داشته باشند.

برای ارتباط بین processها، D-Bus یک مسیر جداگانه فراهم می‌کند:

```
Process
    |
    ↕
D-Bus
    |
    ↕
Process
```

در این معماری dbus-daemon نقش hub را دارد و eventها و پیام‌ها را میان processهای علاقه‌مند جابه‌جا می‌کند. System bus برای communication سطح سیستم و session bus برای communication مربوط به desktop session استفاده می‌شوند.

مسیر چاپ نیز یک pipeline جداگانه دارد:

```
Application
    |
    v
Print Server
    |
    v
Print Queue
    |
    v
Print Filter
    |
    v
RIP / Printer Driver
    |
    v
Backend
    |
    v
Printer
```

در این مسیر CUPS زیرساخت اصلی چاپ است، cupsd daemon آن را اجرا می‌کند و IPP برای communication با clientها، printerها و print serverها کاربرد دارد. اگر printer مستقیماً format ورودی را پشتیبانی نکند، componentهایی مانند Ghostscript، RIP و PPD در تبدیل document و تطبیق آن با قابلیت‌های printer نقش پیدا می‌کنند.

به همین دلیل هنگام troubleshooting بهتر است desktop را به‌عنوان یک component واحد در نظر نگیریم.

اگر مشکل مربوط به window placement یا نحوه‌ی ترکیب windowها باشد، باید window manager یا compositor را بررسی کنیم.

اگر مشکل مربوط به input باشد، مسیر device، kernel input، libinput یا X Input اهمیت پیدا می‌کند.

اگر یک X application در محیط Wayland رفتار غیرمنتظره‌ای داشته باشد، باید وجود و عملکرد Xwayland را در نظر گرفت.

اگر مشکل مربوط به ارتباط بین applicationها یا eventهای system باشد، D-Bus یکی از componentهای اصلی برای بررسی است.

و اگر مشکل مربوط به printing باشد، باید queue، filter، RIP، driver و backend را به‌عنوان مرحله‌های جداگانه در نظر گرفت.

بنابراین معماری کلی فصل را می‌توان بدون تکرار مطالب قبلی به این شکل خلاصه کرد:

```
                      Linux Desktop
                           |
        +------------------+------------------+
        |                  |                  |
        v                  v                  v
      Display              IPC             Printing
        |                  |                  |
   +----+----+           D-Bus              CUPS
   |         |              |                  |
   v         v              v                  v
   X       Wayland      Processes            Queue
   |         |                                 |
X Server  Compositor                         Filter
   |         |                                 |
   +----+----+                                 v
        |                                  RIP / Driver
        v                                      |
   Framebuffer                                 v
        |                                    Backend
        v                                      |
      Display                                  v
                                             Printer
```

در نتیجه، چیزی که باید از این فصل در ذهن بماند این نیست که «Linux desktop یک سیستم بزرگ و یکپارچه است»، بلکه برعکس، باید آن را مجموعه‌ای از componentهای مستقل، قابل‌تعویض و قابل‌بررسی دید.

همین نگاه معماری است که عیب‌یابی را ساده می‌کند: به‌جای جست‌وجو در کل desktop، ابتدا مسیر مربوط به مشکل را مشخص می‌کنیم و سپس component مسئول همان مرحله را بررسی می‌کنیم.
