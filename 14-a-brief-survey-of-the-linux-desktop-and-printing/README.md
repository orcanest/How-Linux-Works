## نگاهی سریع به دسکتاپ لینوکس و چاپگر
### 🐧 فصل چهاردهم کتاب How Linux Works

فصل چهاردهم وارد بخشی از لینوکس می‌شود که نسبت به مباحثی مثل storage یا networking ساختار لایه‌ای و خطی کمتری دارد. در دسکتاپ لینوکس، معمولاً با مجموعه‌ای از componentهای مستقل روبه‌رو هستیم که هرکدام مسئولیت مشخصی دارند و با استفاده از protocolها، libraryها و IPC با یکدیگر ارتباط برقرار می‌کنند.

تمرکز اصلی این فصل روی شناخت اجزای اصلی یک desktop system است، نه آموزش کار با یک desktop environment خاص. فصل ابتدا مفهوم کلی نمایش گرافیکی و اجزای desktop را بررسی می‌کند، سپس وارد Wayland و X Window System می‌شود، ابزارهای تشخیصی ساده‌ای برای بررسی آن‌ها معرفی می‌کند، D-Bus را به‌عنوان یک مکانیزم مهم IPC بررسی می‌کند و در پایان معماری چاپ در Linux و CUPS را توضیح می‌دهد.

---
📚 Table of Contents

- [Desktop Components](#desktop-components)
- [Are You Running Wayland or X?](#are-you-running-wayland-or-x)
- [A Closer Look at Wayland](#a-closer-look-at-wayland)
- [A Closer Look at the X Window System](#a-closer-look-at-the-x-window-system)
- [D-Bus](#d-bus)
- [Printing](#printing)
- [Other Desktop Topics](#other-desktop-topics)
- [Desktop Architecture and Summary](#desktop-architecture-and-summary)

---

###  Desktop Components

یکی از ویژگی‌های مهم Linux desktop انعطاف‌ پذیری آن است. آنچه کاربر در نهایت به‌ عنوان ظاهر و رفتار desktop می‌ بیند ، حاصل همکاری چندین component مختلف است و الزاماً یک نرم‌افزار واحد مسئول همه چیز نیست. برخلاف بعضی قسمت‌ های سیستم‌ عامل که می‌ توان آن‌ ها را به‌ صورت یک زنجیره نسبتاً مشخص از لایه‌ ها تصور کرد ، desktop بیشتر مجموعه‌ ای از اجزای مستقل است. بعضی از این اجزا library های مشترکی دارند و از طریق همین building block های مشترک با هم هماهنگ می‌ شوند ، اما architecture کلی بسیار آزاد تر است. در نسخه‌ های جدید تر Linux ، این حوزه در وضعیت گذار میان X Window System و Wayland قرار گرفته است. X برای سال‌ های طولانی پایه‌ ی اصلی desktop های Linux بود ، ولی بسیاری از distribution ها به سمت سیستم‌ های مبتنی بر Wayland حرکت کرده‌ اند.

#### 🔹 Framebuffers

در پایین‌ ترین سطح یک سیستم نمایش گرافیکی، چیزی به نام framebuffer قرار دارد. Framebuffer را می‌ توان بخشی از حافظه در نظر گرفت که اطلاعات مربوط به تصویر در آن ذخیره می‌ شود و سخت‌ افزار گرافیکی آن اطلاعات را می‌ خواند و برای نمایش روی صفحه استفاده می‌ کند. در ساده‌ ترین مدل ذهنی ، هر قسمت از حافظه‌ ی framebuffer متناظر با اطلاعات مربوط به یک یا چند pixel است. بنابراین اگر قرار باشد ظاهر تصویر تغییر کند ، در نهایت باید مقادیر مناسب در framebuffer تغییر کنند. اما همین‌ جا یکی از مسائل اصلی windowing system ها مطرح می‌ شود. فرض کنید چند process مختلف داریم و هر process صاحب یک یا چند window است. هر برنامه به‌ صورت مستقل بخش گرافیکی خودش را تولید می‌ کند. کاربر هم باید بتواند پنجره‌ ها را حرکت دهد ، آن‌ ها را روی هم بیندازد ، بعضی را جلوتر از بقیه بیاورد و بخشی از صفحه را پوشش دهد. در این شرایط دو مسئله اساسی وجود دارد:

- هر برنامه باید بداند دقیقاً کجا باید محتوای window خودش را رسم کند.
- یک برنامه نباید بتواند محتوای گرافیکی متعلق به window برنامه‌ ی دیگر را خراب یا overwrite کند.

بنابراین یکی از مسئولیت‌ های اصلی هر windowing system این است که بین buffer های مربوط به برنامه‌ های مختلف هماهنگی ایجاد کند و مشخص کند چه چیزی در نهایت باید روی framebuffer نمایش داده شود.

#### 🔹 The X Window System

مدل X Window System برای حل این مسئله یک X server مرکزی دارد. X server را می‌ توان تقریباً به‌ عنوان هسته‌ ی قدیمی desktop در نظر گرفت. این server مسئول کارهایی مانند مدیریت نمایش ، rendering ، تنظیم display و دریافت input از device هایی مثل keyboard و mouse است. برنامه‌ هایی مثل terminal window یا web browser در این مدل X client هستند. client به X server متصل می‌ شود و درخواست‌ هایی برای ساخت و نمایش window ارسال می‌ کند. X server در پاسخ ، مسئولیت بخشی از کارهای مربوط به نمایش را بر عهده می‌ گیرد :

- مشخص می‌ کند window ها در کجا قرار بگیرند.
- مشخص می‌ کند خروجی client در چه بخشی از نمایش render شود.
- داده‌ ی گرافیکی را در مسیر رسیدن به framebuffer مدیریت می‌ کند.
- همچنین event های input را به client مناسب می‌ رساند.

> نکته‌ ی مهم این است که X server به‌ تنهایی ظاهر یک برنامه را تعیین نمی‌ کند. این خود X client ها هستند که UI برنامه را می‌ سازند. بنابراین X server بیشتر زیرساختی است که client ها روی آن کار می‌ کنند.

این architecture در زمان خودش بسیار انعطاف‌پذیر بود ، اما از آنجا که X واسطه‌ ی بسیار زیادی از کارهاست ، می‌ تواند یک bottleneck ایجاد کند. علاوه بر این ، X سیستم بسیار قدیمی‌ ای است که ریشه‌ ی آن به دهه‌ ی ۱۹۸۰ برمی‌ گردد و طی سال‌ ها قابلیت‌ های زیادی به آن اضافه شده است ؛ در نتیجه بخش‌هایی از آن شامل functionality هایی است که دیگر در desktop های مدرن اهمیت گذشته را ندارند.

#### 🔹 Wayland

برای حل همین مسائل Wayland با architecture متفاوتی طراحی شده است. در Wayland یک display server مرکزی مشابه مدل X وجود ندارد که همه‌ ی framebuffer operations را برای تمام client ها انجام دهد. در عوض ، هر graphical client معمولاً buffer خودش را برای window خودش در اختیار دارد سپس یک component به نام compositor buffer های مختلف client ها را با یکدیگر ترکیب می‌ کند و نتیجه‌ ی نهایی را برای نمایش روی framebuffer آماده می‌ کند. از آنجا که بسیاری از سیستم‌ های گرافیکی امروزی از hardware acceleration استفاده می‌ کنند ، این مرحله می‌ تواند بسیار کارآمد انجام شود. از یک دید دیگر، بخشی از این ایده کاملاً جدید نیست. بسیاری از X client های مدرن سال‌ هاست که بخش زیادی از تصویر خودشان را به‌ صورت bitmap تولید می‌ کنند و سپس آن را در اختیار X قرار می‌ دهند. X نیز برای همین مدل ، extension های compositing را در طول زمان اضافه کرده است. بنابراین یکی از تفاوت‌ های اصلی این است که در Wayland compositing از ابتدا بخشی اساسی از architecture محسوب می‌ شود ، در حالی که X این قابلیت را در طول زمان به سیستم خود اضافه کرده است. برای مسیر input نیز بسیاری از سیستم‌ های Wayland و حتی بسیاری از پیاده‌ سازی‌های X از libinput استفاده می‌ کنند تا event های device های ورودی به شکلی استاندارد به بخش مناسب سیستم منتقل شوند. libinput جزئی اجباری از protocol خود Wayland نیست ، ولی در desktop های امروزی بسیار رایج است.

#### 🔹 Window Managers

مسئول بخش مهمی از تجربه‌ی کاربر در دسکتاپ Window manager است ، زیرا مشخص می‌ کند پنجره‌ ها چگونه روی صفحه قرار بگیرند و چه رفتاری داشته باشند. در X هم window manager خودش یک client است که به X server متصل می‌شود. وظایفی مثل این‌ ها را انجام می‌ دهد :

- تنظیم position پنجره‌ ها
- جا به‌ جا کردن پنجره‌ ها
- مدیریت title bar
- مدیریت دکمه‌ هایی مثل close
- واکنش به event های مربوط به decoration
- درخواست از X server برای جا به‌ جایی window ها

به عبارت دیگر، X server و window manager دو component جدا هستند. در Wayland ، این تفکیک تا حد زیادی از بین می‌ رود. compositor عملاً همان بخش مرکزی‌ ای است که نقش window manager را هم بر عهده دارد. compositor buffer های client ها را ترکیب می‌ کند ، آن‌ ها را در framebuffer نهایی قرار می‌ دهد و event های input را نیز به مقصد مناسب هدایت می‌ کند. در نتیجه compositor در Wayland مسئولیت بیشتری نسبت به window manager سنتی X دارد. در هر دو سیستم window manager های متعددی وجود دارند ، ولی X به‌ دلیل قدمت بسیار بیشتر، تنوع بسیار بیشتری پیدا کرده است. همچنین پروژه‌ هایی مانند Mutter در GNOME و KWin در KDE به‌ گونه‌ای توسعه داده شده‌ اند که از compositing در Wayland هم پشتیبانی کنند. با وجود این تنوع ، انتظار وجود یک استاندارد واحد برای window manager در Linux منطقی نیست ، چون انتخاب window manager تا حد زیادی به نیاز و سلیقه‌ ی کاربر بستگی دارد.

#### 🔹 Toolkits

برنامه‌ های desktop معمولاً element های مشترکی مانند button و menu و textbox و widget دارند. اگر هر برنامه بخواهد همه‌ ی این اجزا را از صفر پیاده‌ سازی کند ، هم توسعه دشوار می‌ شود و هم ظاهر برنامه‌ ها یکدست نخواهد بود. به همین دلیل از graphical toolkit استفاده می‌ شود. Toolkit مجموعه‌ ای از component ها ، library ها و support file ها را فراهم می‌ کند تا توسعه‌ دهنده بتواند UI برنامه را سریع‌ تر بسازد. در Linux دو toolkit بسیار مهم عبارت‌ اند از GTK+ و Qt . معمولاً Toolkit ها فقط مجموعه‌ ای از library های کدنویسی نیستند و می‌ توانند شامل فایل‌ های کمکی مانند image ها و اطلاعات مربوط به theme ها نیز باشند.

#### 🔹 Desktop Environments

همچنین Toolkit به‌ تنهایی همه‌ ی نیازهای یک desktop را پوشش نمی‌ دهد. گاهی چند برنامه باید با هم همکاری کنند. مثلاً :

- یک برنامه بخواهد اطلاعاتی را با برنامه‌ ی دیگری به اشتراک بگذارد.
- برنامه‌ ها باید با notification های desktop هماهنگ شوند.
- منوها و title ها باید بر اساس convention های مشترک ظاهر شوند.
- برنامه‌ ها باید رفتار مشخصی نسبت به event های سیستم داشته باشند.

برای ایجاد این هماهنگی ، toolkit ها و library های دیگر در مجموعه‌ های بزرگ‌ تری به نام desktop environment قرار می‌ گیرند. نمونه‌ های رایج desktop environment عبارت‌اند از GNOME و KDE و Xfce . همچنین Desktop environment فقط toolkit نیست. معمولاً مجموعه‌ ای از موارد زیر را نیز شامل می‌ شود مثل icon ها ، configuration ها ، theme ها ، design convention ها ، قوانین مربوط به ظاهر application ها و رفتار مشترک application ها در برابر event های سیستم. به این ترتیب desktop environment تلاش م ی‌کند مجموعه‌ ای از برنامه‌ ها را از نظر تجربه‌ ی کاربری تا حد ممکن یکپارچه کند.

#### 🔹 Applications

در بالا ترین سطح desktop ، خود application ها قرار دارند مثل web browser و terminal emulator و file  manager و office suite و utility های مختلف. این application ها معمولاً برنامه‌ های مستقلی هستند ، اما در عمل باید نسبت به event های مهم سیستم و desktop آگاه باشند. مثلاً یک application ممکن است بخواهد هنگام وقوع یکی از این event ها واکنش نشان دهد :

- اتصال یک storage device جدید
- دریافت email
- دریافت instant message
- تغییر وضعیت یک سرویس

برای این نوع ارتباط ، application ها معمولاً از interprocess communication استفاده می‌ کنند و یکی از مهم‌ ترین مکانیزم‌ ها در این حوزه D-Bus است.

---

### Are You Running Wayland or X ?

برای بررسی اینکه session گرافیکی شما با Wayland اجرا می‌ شود یا X ، کتاب یک روش بسیار ساده ارائه می‌ کند و باید مقدار environment variable مربوط به Wayland را بررسی کنید.

```echo $WAYLAND_DISPLAY```

اگر چیزی شبیه wayland-0 ببینید ، احتمالاً session شما روی Wayland قرار دارد. اگر این variable تنظیم نشده باشد ، در محیط مورد بحث کتاب معمولاً می‌ توان نتیجه گرفت که session شما X است البته این آزمایش استثناهایی دارد و یک تشخیص مطلق نیست. 

> نکته‌ ی مهم این است که Wayland و X الزاماً دو سیستم کاملاً مجزا و mutually exclusive نیستند.

اگر سیستم شما از Wayland استفاده کند ، بسیار محتمل است که در کنار آن یک X compatibility server نیز در حال اجرا باشد تا application های قدیمی X بتوانند همچنان کار کنند. از طرف دیگر حتی می‌ توان یک Wayland compositor را داخل یک X window اجرا کرد. با این حال چنین ترکیبی ممکن است باعث رفتارهای گیج‌ کننده شود ، چون برنامه‌ هایی که از هر دو protocol پشتیبانی می‌ کنند ممکن است compositor موجود را پیدا کنند و به‌ جای session اصلی X به آن متصل شوند.

---

### A Closer Look at Wayland

در زمان نگارش کتاب Wayland به‌ عنوان فناوری در حال گسترش و جایگزینی تدریجی X معرفی می‌ شود و در بسیاری از   distribution ها به‌ صورت پیش‌ فرض مورد استفاده قرار گرفته است. با این حال یکی از تفاوت‌ های مهم از نظر troubleshooting این است که در X ابزارهای بسیار بیشتری برای مشاهده و بررسی اجزای داخلی وجود دارد. Wayland طراحی جدید تری دارد و ابزارهای تشخیصی command-line آن به گستردگی ابزارهای X نیستند. اولین نکته‌ ای که باید درباره Wayland روشن شود این است که Wayland نام یک desktop environment یا یک server بزرگ به سبک X نیست. Wayland در اصل یک communications protocol میان یک graphical client و یک compositing window manager است بنابراین نباید انتظار داشته باشید با نصب چیزی به نام "Wayland core" یک desktop کامل در اختیار داشته باشید. آنچه معمولاً در سیستم پیدا می‌ کنید شامل library هایی برای صحبت کردن با protocol و مجموعه‌ ای از component های مرتبط است. در کنار آن Weston قرار دارد که یک reference compositing window manager برای Wayland است. Reference بودن Weston به این معنا نیست که قرار است مثل GNOME یا KDE یک desktop environment عمومی باشد. Weston interface بسیار ساده‌ ای دارد و بیشتر برای این منظور ساخته شده که توسعه‌ دهندگان بتوانند implementation های اصلی compositor را ببینند و از آن برای درک نحوه‌ ی پیاده‌ سازی بخش‌ های مهم Wayland استفاده کنند.

#### 🔹 The Compositing Window Manager

در Wayland همیشه از ظاهر desktop نمی‌ توان فهمید دقیقاً چه compositor در حال اجراست. برخلاف برخی سیستم‌ های دیگر، یک محل استاندارد و ساده وجود ندارد که فقط با نگاه کردن به یک فایل مشخص بتوان نام compositor را پیدا کرد. یکی از روش‌ های قابل اتکاتر این است که Unix domain socket مربوط به Wayland را پیدا کنیم. نام این socket از مقدار environment variable زیر مشخص می‌ شود :

```echo $WAYLAND_DISPLAY```

مقدار معمول ```wayland-0``` است و این socket معمولاً در مسیری مشابه ```/run/user/<uid>/wayland-0``` قرار دارد و اگر محل runtime directory را ندانید ، می‌توانید مقدار ```echo $XDG_RUNTIME_DIR```  را بررسی کنید سپس از طریق ss می‌ توان process ای را که روی socket هم listen می‌ کند پیدا کرد ```ss -xlp | grep wayland```. در یک نمونه‌ ی مطرح‌ شده در کتاب ، خروجی نشان می‌ دهد که process با نام gnome-shell و یک PID مشخص روی socket مربوط به Wayland در حال listening است. اما اینجا یک لایه‌ ی دیگر نیز وجود دارد. در GNOME هم gnome-shell خودش implementation مستقل کامل compositor نیست بلکه از Mutter به‌ عنوان compositing window manager استفاده می‌ کند. یعنی GNOME Shell در این معماری از Mutter به‌ صورت library بهره می‌ گیرد.

#### 🔹 Window Decorations in Wayland

یکی از تفاوت‌ های جالب X و Wayland در نحوه‌ ی مدیریت window decorations است. در X ، window manager به‌ طور سنتی decoration هایی مانند title bar را کنترل می‌ کند. در implementation های ابتدایی Wayland ، این مسئولیت بیشتر به خود application ها سپرده شده بود و این موضوع می‌ توانست باعث شود چند نوع متفاوت title bar و decoration روی یک desktop دیده شود. بعد ها protocol به نام XDG-Decoration اضافه شد تا client و window manager بتوانند درباره‌ ی این موضوع با یکدیگر مذاکره کنند. در نتیجه client می‌ تواند متوجه شود که آیا compositor حاضر است decoration مربوط به window را خودش رسم کند یا نه.

#### 🔹 Displays and Multiple Compositors

در Wayland می‌ توان display را همان فضای قابل مشاهده‌ ای در نظر گرفت که در نهایت با framebuffer نمایش داده می‌ شود. یک display می‌ تواند چند monitor را شامل شود همچنین در موارد خاص می‌ توان بیش از یک compositor را هم‌ زمان اجرا کرد ، مثلاً اگر compositor ها روی virtual terminal های متفاوت قرار گرفته باشند. در چنین حالتی معمولاً نام display ها به‌ ترتیب چیزی شبیه wayland-0 و wayland-1 خواهد بود. برای به‌ دست آوردن اطلاعاتی درباره‌ ی interface هایی که compositor ارائه می‌ کند می‌ توان از ```weston-info``` استفاده کرد. با این حال نباید انتظار داشت این ابزار یک debugger کامل برای compositor باشد. اطلاعاتی که به‌ دست می‌آورید بیشتر مربوط به display و input  device هاست.

#### 🔹 libinput

برای آنکه input از device های فیزیکی مثل keyboard یا mouse از kernel به graphical client برسد ، compositor به مکانیزمی نیاز دارد که event ها را از device دریافت و استاندارد کند. این وظیفه را در بسیاری از سیستم‌ های جدید libinput انجام می‌ دهد. libinput به device های ورودی kernel که در مسیرهایی مانند ```/dev/input/```  دیده می‌ شوند دسترسی دارد و event های خام آن‌ ها را پردازش می‌ کند. در Wayland هم compositor معمولاً event خام kernel را بدون تغییر مستقیماً به client نمی‌ فرستد. بلکه event را دریافت می‌ کند ، آن را به شکل مناسب درمی‌ آورد و سپس آن را از طریق Wayland protocol به client مربوطه منتقل می‌ کند. یکی از مزیت‌ های مهم libinput این است که فقط یک library نیست و utility های command-line نیز در اختیار کاربر قرار می‌ دهد. برای دیدن device های ورودی از ```libinput list-devices``` استفاده می شود و ممکن است خروجی شامل device هایی مثل :

```
Device: Cypress USB Keyboard
Kernel: /dev/input/event3
Group: 6
Seat: seat0, default
Capabilities: keyboard
```

باشد و این خروجی نشان می‌ دهد که kernel device مثلاً یک keyboard را در مسیر /dev/input/event3 می‌ شناسد. برای مشاهده‌ ی event های نیز می‌ توان از```libinput debug-events --show-keycodes``` استفاده کرد. بعد از اجرای آن ، با حرکت دادن mouse یا فشردن کلید های keyboard ، event هایی مانند device addition ، press و release را خواهید دید. این ابزار فقط مخصوص Wayland نیست. libinput یک سیستم برای دریافت و پردازش kernel input event هاست و بنابراین در بسیاری از محیط‌ های X نیز مورد استفاده قرار می‌ گیرد.

#### 🔹 X Compatibility in Wayland

یکی از موانع مهم مهاجرت از X به Wayland ، تعداد بسیار زیاد application های قدیمی X است. اگر قرار بود با ورود Wayland همه‌ ی این برنامه‌ ها کنار گذاشته شوند ، انتقال به سیستم جدید بسیار دشوار می‌ شد. برای همین دو رویکرد هم‌ زمان شکل گرفته است.

#### 🔹 Native Wayland Applications

روش اول این است که application مستقیماً از Wayland پشتیبانی کند. بسیاری از برنامه‌ های گرافیکی قدیمی X از toolkit هایی مثل toolkit های GNOME و KDE استفاده می‌ کنند. وقتی خود toolkit از Wayland پشتیبانی کند ، انتقال application به حالت native Wayland آسان‌ تر می‌ شود. در این حالت توسعه‌ دهنده باید بخش‌ هایی مثل window decoration و input configuration و وابستگی‌ های باقی‌ مانده به کتابخانه‌ های X را بررسی کند. بسیاری از application های مهم این مسیر را طی کرده‌ اند.

#### 🔹 Xwayland

روش دوم اجرای application های X از طریق یک compatibility layer است. این لایه Xwayland نام دارد. Xwayland در واقع یک X server کامل است که به‌ عنوان Wayland client اجرا می‌ شود در نتیجه application های قدیمی X می‌ توانند همچنان تصور کنند که با یک X server کار می‌ کنند ، در حالی که خود X server در لایه‌ ی پایین‌ تر به compositor Wayland متصل شده است. Xwayland باید input event ها را ترجمه کند و buffer های مربوط به window های X را نیز به‌ شکل جداگانه مدیریت کند. وجود این لایه می‌ تواند کمی overhead ایجاد کند ، ولی برای بیشتر application ها این هزینه در عمل ناچیز است.

#### 🔹 Running Wayland Inside X

حرکت برعکس به این سادگی نیست. یک Wayland client را نمی‌ توان دقیقاً به همان روش مستقیماً روی X اجرا کرد. با این حال می‌ توان یک compositor مانند ```weston``` را داخل یک X window اجرا کرد ، در این حالت می‌ توانید  application های Wayland را داخل compositor اجرا کنید و حتی با تنظیم درست Xwayland ، application های X را نیز داخل آن داشته باشید. مشکل زمانی ظاهر می‌ شود که چنین compositor  را باز بگذارید و دوباره به session معمول X برگردید. Application هایی که هم X و هم Wayland را پشتیبانی می‌ کنند ممکن است compositor Wayland را پیدا کنند و به آن متصل شوند بنابراین ممکن است برنامه‌ ای که انتظار دارید در X باز شود ، داخل compositor دیگر نمایش داده شود. برای جلوگیری از این نوع رفتارهای عجیب ، توصیه‌ ی اصلی این است که به‌ صورت هم‌ زمان یک compositor داخلی Wayland را در کنار X session معمولی اجرا نکنید.

---

### A Closer Look at the X Window System

در مقایسه با Wayland هم ، X Window System از نظر تاریخی مجموعه‌ ی بسیار بزرگ‌ تری از component ها را شامل می‌ شد. در یک X system مجموعه‌ ای از این اجزا وجود داشت مثل X server و client libraries و X clients و ابزارهای جانبی و window manager ها. با ظهور desktop environment هایی مانند GNOME و KDE ، نقش X به‌ مرور محدود تر و مشخص‌ تر شد و تمرکز آن بیشتر روی بخش‌ های مرکزی مانند rendering و مدیریت input قرار گرفت.

#### 🔹 Identifying the X Server

معمولاً X server با نام‌ هایی مانند X یا Xorg دیده می‌ شود با بررسی process ها ممکن است command line  شبیه این ببینید : 

```
Xorg -core :0 -seat seat0 -auth /var/run/lightdm/root/:0 \
-nolisten tcp vt7 -novtswitch
```

در اینجا 0: یک X display identifier است. یک display می‌ تواند یک یا چند monitor را نمایندگی کند یعنی چند monitor می‌ توانند در یک display مشترک قرار داشته باشند و از keyboard و mouse مشترک استفاده کنند. در process های یک X session نیز این variable هم```echo $DISPLAY``` معمولاً به همین display identifier اشاره می‌ کند مثلاً ```0:``` از نظر تئوری display می‌ تواند به screen های جداگانه مانند این تقسیم شود :

```
:0.0
:0.1
```

ولی این مدل امروزه چندان متد اول نیست ، زیرا extension هایی مثل RandR می‌ توانند چند monitor را در قالب یک فضای مجازی بزرگ مدیریت کنند. در Linux ، یک X server روی virtual terminal اجرا می‌ شود. مثلاً اگر در process arguments عبارت vt7 را ببینید ، منظور virtual terminal مربوط به /dev/tty7 است همچنین می‌ توان بیش از یک X server داشت کافی است آن‌ ها روی virtual terminal های جداگانه اجرا شوند و هر کدام display identifier مخصوص خودشان را داشته باشند. برای جا به‌ جایی بین virtual terminal ها نیز می‌ توان از ترکیب کلید های CTRL-ALT-FN یا utility مثل chvt استفاده کرد.

#### 🔹 Display Managers

اگر X server را به‌ تنهایی اجرا کنید ، لزوماً desktop قابل استفاده‌ ای در اختیار نخواهید داشت. X server فقط زیرساخت display را فراهم می‌ کند و نمی‌ داند چه client هایی باید اجرا شوند بنابراین اگر X را بدون client های لازم بالا بیاورید ، ممکن است فقط یک صفحه‌ ی خالی ببینید. راه رایج‌ تر استفاده از display manager است. display manager معمولاً X server را راه‌ اندازی  و  صفحه‌ی login را نمایش می‌ دهد پس از login مجموعه‌ ای از client ها را شروع می‌ کند و session گرافیکی مورد نظر کاربر را راه‌ اندازی می‌ کند. از نمونه‌ های مطرح‌ شده در کتاب gdm و kdm و lightdm 
هستند. مثلاً lightdm یک display manager عمومی‌ تر است که می‌ تواند session های مختلفی مثل GNOME یا KDE را راه‌ اندازی کند. اگر نخواهید از display manager استفاده کنید ، می‌ توانید از virtual console با ابزارهایی مانند ```startx``` یا ```xinit``` یک X session را شروع کنید. با این حال session حاصل معمولاً بسیار ساده‌ تر از session خواهد بود که از طریق display manager ایجاد می‌ شود ، چون startup mechanics و startup file های این دو روش یکسان نیستند.

#### 🔹 Network Transparency

یکی از ویژگی‌ های تاریخی و مهم X هم network transparency است. در X هم client و server از طریق یک protocol با یکد یگر صحبت می‌ کنند. بنابراین در این حالت می‌ توان client را روی یک machine و X server را روی machine دیگری اجرا کرد. در این مدل ، X server می‌ توانست به 6000 TCP port هم listen کند سپس client راه دور به آن متصل می‌ شد و window های خود را روی display آن machine نمایش می‌ داد. مشکل این روش امنیت است این ارتباط در شکل قدیمی معمولاً encryption مناسبی نداشت. به همین دلیل بسیاری از distribution ها listening مستقیم X روی شبکه را غیرفعال کرده‌ اند و X server را با option مانند ```nolisten -tcp``` اجرا می‌ کنند. با این حال remote X application همچنان می‌ تواند از طریق SSH tunneling مورد استفاده قرار گیرد. در این روش SSH مسیر امنی را برای اتصال فراهم می‌ کند و امکان اجرای client راه دور را بدون باز کردن مستقیم TCP port مربوط به X فراهم می‌ سازد. Wayland در این زمینه با X تفاوت مهمی دارد. در Wayland هر client buffer خودش را مدیریت می‌ کند و compositor باید به آن buffer ها دسترسی داشته باشد بنابراین model ساده‌ ی remote X را نمی‌ توان مستقیماً روی Wayland تکرار کرد. سیستم‌ های دیگری مثل RDP می‌ توانند با compositor همکاری کنند و remote desktop functionality ایجاد کنند ، ولی این مسئله از نظر معماری با network transparency قدیمی X متفاوت است.


#### 🔹 Ways of Exploring X Clients

یکی از بخش‌ های جالب X این است که حتی از command line هم می‌ توانید وضعیت window ها و client ها را بررسی کنید.
#### 🔹 xwininfo

یکی از ساده‌ ترین utility ها ```xwininfo``` است. اگر آن را بدون argument اجرا کنید ، از شما می‌ خواهد یک window را با mouse انتخاب کنید. بعد از انتخاب window اطلاعاتی مانند position و size آن نمایش داده می‌ شود. نمونه‌ ای از اطلاعات مهم :

```
xwininfo: Window id: 0x5400024 "xterm"
Absolute upper-left X: 1075
Absolute upper-left Y: 594
```

در اینجا window ID اهمیت زیادی دارد. X server و window manager از این شناسه برای شناسایی windowها استفاده می‌ کنند. برای دریافت فهرستی از X client ها و window های موجود می‌ توانید از ```xlsclients -l``` استفاده کنید.

#### 🔹 X Events

معمولاً X client ها برای دریافت input و اطلاع از تغییر وضعیت server از event ها استفاده می‌ کنند. این event ها  synchronous هستند یعنی client لازم نیست دائماً وضعیت input device را poll کند. X server event را از source دریافت می‌ کند و آن را برای client هایی که به آن علاقه‌ مند هستند ارسال می‌ کند. این مدل شباهت مفهومی زیادی با event   system هایی مانند udev events و D-Bus events دارد. برای مشاهده‌ ی event های X می‌ توانید از ```xev``` استفاده کنید. اجرای xev یک window باز می‌ کند. اگر mouse را داخل آن حرکت دهید ، click کنید یا key بزنید ، خروجی command line شامل event های دریافتی نمایش داده می‌ شود. برای مثال event مربوط به حرکت mouse اطلاعاتی درباره‌ ی coordinate ها دارد ```MotionNotify event``` . دو جفت مختصات در این خروجی مهم هستند که مختصات pointer در داخل window و مختصات pointer نسبت به کل display . برای مثال :

```
(47,174)
root:(1692,486)
```

مختصات اول مربوط به window و مختصات دوم مربوط به کل display است. X event ها فقط حرکت mouse نیستند. event هایی مثل :

- key press
- button click
- mouse entering window
- mouse leaving window
- window gaining focus
- window losing focus

نیز وجود دارند. یکی از کاربرد های مهم xev استخراج اطلاعات keyboard است مثلاً با فشردن یک کلید می‌ توانید چیزی شبیه این ببینید :

```
KeyPress event ...
keycode 46 (keysym 0x6c, l)
```

در اینجا keycode شناسه‌ ی مربوط به کلید در X است و keysym مشخص می‌ کند آن کلید از نظر معنایی چه کاراکتر یا symbol را نشان می‌ دهد همچنین می‌ توانید یک window موجود را با id- هدف بگیرید مانند ```xev -id 0x5400024``` شناسه‌ ی 0x را می‌ توانید از xwininfo به‌ دست آورید.

#### 🔹 X Input and Preference Settings

یکی از گیج‌ کننده‌ ترین بخش‌ های X این است که برای یک تنظیم مشخص ممکن است چند روش مختلف وجود داشته باشد و همه‌ ی این روش‌ها لزوماً در هر desktop environment به یک شکل کار نکنند. مثلاً برای تغییر رفتار CAPS LOCK و تبدیل آن به CTRL ، روش‌ های مختلفی وجود دارد. علت این پیچیدگی این است که چند لایه‌ ی مختلف ممکن است در تنظیمات input دخالت داشته باشند :

- X server
- X Input Extension
- XKB
- xmodmap
- desktop environment

پس قبل از تغییر تنظیم باید بدانید دقیقاً کدام component مسئول آن feature است ، چون desktop environment ممکن است تنظیمات خودش را اعمال یا override کند.

#### 🔹 Input Devices (General)

برای مدیریت input device های مختلف ، X از X Input Extension استفاده می‌ کند. در سطح پایه ، دو نوع device اصلی مطرح می‌ شوند مانند keyboard و pointer . می‌ توانید چند device از یک نوع به سیستم متصل کنید و برای هماهنگ کردن چند keyboard یا چند pointer هم ، X مفهوم virtual core device را ارائه می‌ کند. برای مشاهده‌ ی device ها ```xinput --list``` خروجی ممکن است چیزی شبیه این باشد :

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

در این مدل master device نقش virtual core device را دارد و slave device های واقعی هستند که input را تولید می‌ کنند. مثلاً ID های 2 و 3 در این نمونه device های core هستند و device هایی مانند keyboard واقعی ID های دیگر دارند. نکته‌ ی مهم این است که power button خود machine نیز از دید X می‌ تواند یک input device محسوب شود. بیشتر X client ها فقط به core device هم listen می‌ دهند و لازم نیست بدانند event دقیقاً از کدام keyboard یا mouse فیزیکی آمده است ولی یک application می‌ تواند از X Input Extension استفاده کند تا یک device خاص را انتخاب کند.

#### 🔹 Device Properties

هر input device مجموعه‌ ای از property ها دارد. برای مشاهده‌ ی property های یک device از ```xinput --list-props 8 ``` می‌توانید property هایی مثل این را مشاهده کنید :

```
Device Enabled
Coordinate Transformation Matrix
Device Accel Profile
Device Accel Constant Deceleration
Device Accel Adaptive Deceleration
Device Accel Velocity Scaling
```

برخی از این property ها را می‌ توان با ```xinput --set-prop``` تغییر داد در نتیجه xinput فقط ابزار نمایش device ها نیست و برای تغییر بعضی تنظیمات runtime نیز استفاده می‌ شود.

#### 🔹 Mouse

برای mouse و pointer option های مختلفی وجود دارد. می‌ توان بعضی property ها را مستقیماً تغییر داد ، اما برای بعضی کارهای متداول ، option های اختصاصی‌ تر راحت‌ تر هستند. مثلاً برای تغییر mapping دکمه‌ های mouse :

```xinput --set-button-map dev 3 2 1```

در نمونه‌ ی کتاب این کار ترتیب سه دکمه‌ ی mouse را معکوس می‌ کند و می‌ تواند برای استفاده‌ ی left-handed ها مفید باشد.

#### 🔹 Keyboard and XKB

معمولاًKeyboard configuration در X پیچیده‌ تر است ، مخصوصاً چون layout های بین‌المللی بسیار متنوع‌ اند. X از ابتدا قابلیت mapping داخلی برای keyboard داشت و utility قدیمی‌ تر ```xmodmap``` برای تغییر mapping استفاده می‌ شد. 
اما در سیستم‌ های نسبتاً جدید تر، XKB (X Keyboard Extension) روش اصلی و قدرتمند تر برای مدیریت keyboard mapping است. XKB از نظر ساختاری پیچیده است ، به همین دلیل حتی در سیستم‌ های جدید نیز بعضی کاربران برای تغییرات سریع هنوز سراغ xmodmap می‌ روند. ایده‌ ی اصلی XKB این است که یک keyboard map تعریف کنید ، آن را با ```xkbcomp``` هم compile کنید و سپس با ```setxkbmap``` آن را در X server فعال کنید. یکی از ویژگی‌ های جالب XKB این است که لازم نیست همیشه کل keyboard map را از صفر تعریف کنید. می‌ توانید یک partial map بسازید که روی mapping موجود اعمال شود. این قابلیت برای تغییراتی مثل تبدیل CAPS LOCK به CTRL مفید است و بسیاری از graphical keyboard preference utility ها در desktop environment ها از همین ایده استفاده می‌ کنند. ویژگی دیگر این است که می‌ توانید برای keyboard های مختلف mapping های جداگانه داشته باشید.

#### 🔹 Desktop Background

در X هم root window در اصل background اصلی display محسوب می‌ شود. ابزار قدیمی ```xsetroot``` می‌ تواند ویژگی‌هایی مثل رنگ background مربوط به root window را تغییر دهد. اما در desktop های مدرن، root window معمولاً مستقیماً دیده نمی‌ شود. بسیاری از desktop environment ها یک window بزرگ در پشت سایر window ها قرار می‌دهند تا قابلیت‌هایی مثل wallpaper پویا و desktop file browsing و interaction های مربوط به desktop را پیاده کنند. به همین دلیل تغییر root window با xsetroot ممکن است روی ظاهر واقعی desktop تأثیر قابل‌ مشاهده‌ ای نداشته باشد. در بعضی محیط‌ ها می‌ توان از ابزارهایی مثل gsettings برای تغییر background استفاده کرد ، ولی این موضوع دیگر به زیرساخت قدیمی root window محدود نیست.

#### 🔹 xset

یکی از فناوری‌ های مهمی که در Linux desktop رشد کرده D-Bus یا Desktop Bus است. D-Bus یک message-passing system است که برای IPC استفاده می‌ شود. یعنی process ها می‌ توانند از طریق آن به یکدیگر پیام بدهند و application ها می‌ توانند از event های مربوط به سیستم یا برنامه‌ های دیگر مطلع شوند. برای مثال ، وقتی یک USB storage device متصل می‌ شود ، یک component می‌ تواند event مربوط به آن را تشخیص دهد و process های دیگری که به آن event علاقه دارند مطلع شوند. در نگاه ساده ، خود D-Bus شامل library و protocol است که ارتباط بین process ها را استاندارد می‌ کند. بدون بخش مرکزی D-Bus ، این قابلیت تقریباً چیزی شبیه یک IPC پیشرفته‌ تر بر پایه‌ ی mechanism هایی مثل Unix domain socket خواهد بود و قسمت مهم ماجرا dbus-daemon است. dbus-daemon مانند یک central hub عمل می‌ کند که process ها به آن متصل می‌ شوند و client ها مشخص می‌ کنند به چه نوع event هایی علاقه دارند سپس process های دیگر event ایجاد می‌ کنند و dbus-daemon پیام را به گیرنده‌ های مناسب منتقل می‌ کند. یک مثال مهم در کتاب udisks-daemon است. این process event های مربوط به disk را از udev دریافت می‌ کند و سپس آن‌ ها را از طریق D-Bus به application هایی که به چنین event هایی علاقه‌ مند هستند منتقل می‌ کند. به همین دلیل D-Bus فقط یک روش ساده برای اجرای remote procedure نیست بلکه به یک backbone مهم برای communication بین قسمت‌ های مختلف desktop و حتی بخش‌ هایی از system تبدیل شده است.

#### 🔹 System and Session Instances

در D-Bus دیگر فقط یک ابزار مربوط به desktop نیست و در بخش‌ های مختلف Linux system مورد استفاده قرار گرفته است. حتی service هایی مثل systemd و در نسخه‌ های قدیمی‌ تر Upstart نیز می‌ توانند از D-Bus برای ارتباط استفاده کنند اما یک design principle مهم مطرح می‌ شود که **اجزای core system نباید بدون دلیل به ابزارهای خاص desktop وابسته شوند** ، برای همین D-Bus معمولاً به دو نوع instance تقسیم می‌ شود.

#### 🔹 System Instance

معمولاً System bus در سطح کل سیستم فعالیت می‌ کند. این instance هنگام boot و توسط init system راه‌ اندازی می‌ شود و در کتاب با option معرفی شده است ```system--```. این instance معمولاً با یک user مخصوص D-Bus اجرا می‌شود و فایل configuration آن معمولاً در مسیر ```etc/dbus-1/system.conf``` قرار دارد و کتاب توصیه می‌ کند بدون دلیل آن را تغییر ندهید. ارتباط با system bus از طریق Unix domain socket مانند ```/var/run/dbus/system_bus_socket``` انجام می‌ شود. این bus مربوط به یک user خاص یا یک desktop session خاص نیست و برای سرویس‌ های system-wide کاربرد دارد.

#### 🔹 Session Instance

در کنار system bus ، یک session bus نیز وجود دارد. Session bus مستقل از system instance است و هنگام شروع یک desktop session ایجاد می‌ شود. Application های desktop مربوط به یک session به این bus متصل می‌ شوند بنابراین به‌ صورت مفهومی :

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

این جدا سازی باعث می‌ شود communication مربوط به desktop کاربر با communication سطح system یکی نشود.

#### 🔹 D-Bus Message Monitoring

یکی از بهترین راه‌ ها برای درک تفاوت system bus و session bus مشاهده‌ ی event های واقعی روی آن‌ هاست. برای  system bus :

```dbus-monitor --system```

بعد از اجرای این command ممکن است پیام‌ هایی شبیه این ببینید : 
```
signal sender=org.freedesktop.DBus
-> dest=:1.952
serial=2
path=/org/freedesktop/DBus
interface=org.freedesktop.DBus
member=NameAcquired
```

پیام ابتدایی بیشتر به خود monitor مربوط است که به bus متصل شده و یک name دریافت کرده است. اگر فقط همین command را اجرا کنید ، ممکن است برای مدتی activity زیادی نبینید ، چون system bus معمولاً خیلی شلوغ نیست. برای تولید event واقعی می‌ توانید مثلاً یک USB storage device متصل کنید و فعالیت bus را مشاهده کنید در مقابل ، session bus معمولاً بسیار پرتحرک‌ تر است :

```dbus-monitor --session```

حالا اگر یک desktop application مثل file manager را باز یا استفاده کنید ، در یک desktop D-Bus-aware احتمالاً مجموعه‌ ی زیادی از پیام‌ ها را خواهید دید. البته همه‌ ی application ها الزاماً برای هر نوع operation پیام D-Bus تولید نمی‌ کنند.

---

### 14.6 Printing

چاپ در Linux یک عملیات یک‌ مرحله‌ ای نیست. وقتی از یک application درخواست print می‌ کنید ، معمولاً document از چند مرحله عبور می‌ کند تا در نهایت به printer برسد. مسیر کلی که کتاب توضیح می‌ دهد شامل این مراحل است :


1. اول application در صورت نیاز document را به PostScript تبدیل می‌ کند.
2. بعد document به print server ارسال می‌ شود.
3. بعد print server آن را داخل print queue قرار می‌ دهد.
4. وقتی نوبت document برسد ، print server آن را به print filter می‌ فرستد.
5. اگر document در قالب PostScript نباشد ، filter می‌ تواند آن را convert کند.
6. اگر printer مستقیماً PostScript را پشتیبانی نکند ، printer driver document را به format مناسب printer تبدیل می‌ کند.
7. سپس driver می‌ تواند instruction هایی مثل paper tray و duplexing را به job اضافه کند.
8. در نهایت print server از یک backend برای ارسال document به printer استفاده می‌ کند.

بنابراین مسیر ساده‌ شده چنین است :

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

البته همه‌ ی document ها الزاماً از تمام این مراحل عبور نمی‌ کنند. مثلاً تبدیل به PostScript در مرحله‌ ی اول اختیاری است و بسته به application و workflow می‌ تواند متفاوت باشد.

#### 🔹 PostScript

یکی از قسمت‌ های عجیب معماری قدیمی چاپ این است که چرا PostScript تا این اندازه مهم است. PostScript فقط یک data format ساده نیست بلکه در واقع یک programming language است. یعنی وقتی یک document در قالب PostScript ارسال می‌ شود ، در اصل یک برنامه به مقصد ارسال شده که سیستم چاپ یا printer آن را پردازش می‌ کند تا تصویر نهایی صفحه تولید شود. PostScript در Unix-like systems نقش یک استاندارد تاریخی برای چاپ داشته است ، همان‌ طور که format هایی مثل tar. در زمینه‌ ی archive ها یک استاندارد رایج هستند. در سیستم‌ های جدید تر بعضی application ها از PDF به‌ عنوان خروجی استفاده می‌ کنند. با این حال تبدیل PDF به مسیرهای مناسب چاپ نسبتاً ساده است و به همین دلیل PDF نیز به‌ خوبی در این ecosystem قرار می‌ گیرد.

#### 🔹 CUPS

سرویس CUPS مخفف Common UNIX Printing System است و سیستم چاپ استاندارد در بسیاری از سیستم‌ های Linux و دیگر Unix-like ها محسوب می‌ شود. CUPS همچنین در macOS استفاده شده است و daemon اصلی CUPS هم ```cupsd``` نام دارد. برای ارسال ساده‌ ی فایل می‌ توان از client مثل ```lpr``` استفاده کرد. یکی از ویژگی‌ های مهم CUPS پشتیبانی از IPP (Internet Printing Protocol) است. IPP امکان انجام transaction های شبیه HTTP میان client و server را فراهم می‌ کند و در مدل مطرح‌ شده در کتاب روی این TCP port کار می‌ کند ```631``` . اگر CUPS روی سیستم شما در حال اجرا باشد، می‌ توانید معمولاً interface وب local آن را از طریق ```http://localhost:631```
ببینید. این interface می‌ تواند برای مشاهده‌ ی configuration ، printer ها و job های چاپ مفید باشد. بسیاری از network printer ها و print server ها نیز از IPP پشتیبانی می‌ کنند و همین موضوع configuration printer های راه دور را آسان‌ تر می‌ کند. با این حال کتاب هشدار می‌ دهد که نباید فرض کنید interface وب CUPS همیشه برای administration مناسب است. configuration پیش‌ فرض لزوماً با این هدف طراحی نشده که remote administration امن و بدون تغییر در اختیار همه باشد. به‌همین دلیل distribution معمولاً یک graphical settings interface برای افزودن و تغییر printer ها ارائه می‌ کند. این ابزار ها configuration های CUPS را مدیریت می‌ کنند و فایل‌ های آن‌ ها معمولاً در
```etc/cups``` قرار دارند. از آنجا که configuration چاپ می‌ تواند پیچیده باشد ، بهتر است حتی در صورت نیاز به configuration دستی ، ابتدا printer را از طریق ابزار گرافیکی distribution بسازید تا یک configuration پایه و قابل بررسی داشته باشید.

#### 🔹 Format Conversion and Print Filters

همه‌ ی printer ها نمی‌ توانند مستقیماً PostScript یا PDF را پردازش کنند. خصوصاً بسیاری از printer های ارزان‌ تر، format نهایی خاص خودشان را نیاز دارند. در چنین شرایطی سیستم چاپ Linux باید document را به format تبدیل کند که printer بتواند آن را بفهمد. CUPS در این مسیر از Raster Image Processor (RIP) استفاده می‌ کند. RIP document را به شکل bitmap مناسب برای چاپ آماده می‌ کند. در معماری توضیح‌ داده‌ شده در کتاب ، RIP معمولاً از Ghostscript استفاده می‌ کند ```gs``` و بخش قابل‌ توجهی از conversion واقعی را انجام می‌ دهد. اما ایجاد bitmap به‌ تنهایی کافی نیست و bitmap نهایی باید مطابق قابلیت‌ ها و محدودیت‌ های printer باشد. اینجاست که PPD اهمیت پیدا می‌ کند. PPD مخفف ```PostScript Printer Definition``` است. PPD اطلاعات مربوط به ویژگی‌ ها و قابلیت‌ های printer را در اختیار driver قرار می‌ دهد ، از جمله مواردی مانند resolution و paper size و تنظیمات چاپ و قابلیت‌های خاص  printer ، بنابراین workflow ساده‌ شده این بخش می‌ تواند چنین در نظر گرفته شود :

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

در این میان PPD نقش metadata مربوط به قابلیت‌ های printer را دارد و به driver کمک می‌ کند conversion را مطابق device مقصد انجام دهد.

---

### Other Desktop Topics

یکی از خصوصیات جالب Linux desktop این است که مجبور نیستید همه‌ ی اجزای desktop را به‌ عنوان یک مجموعه‌ ی غیرقابل‌ تفکیک انتخاب کنید. می‌ توانید بخش‌ های مختلف را از پروژه‌ ها و component های گوناگون انتخاب کنید ، چیزهایی را که دوست دارید نگه دارید و اجزایی را که نمی‌ پسندید جایگزین کنید. این انعطاف‌ پذیری یکی از دلایل تنوع زیاد پروژه‌ های desktop در Linux است. کتاب برای آشنایی با پروژه‌ ها و ecosystem های مختلف desktop به منابع و جامعه‌ ی freedesktop.org اشاره می‌ کند. جایی که پروژه‌ های مرتبط با desktop Linux ، protocol ها و infrastructure های مشترک زیادی را می‌ توان دنبال کرد

#### 🔹 Chromium OS and Chrome OS

یکی دیگر از تحولات مهمی که کتاب به آن اشاره می‌ کند Chromium OS و نسخه‌ ی تجاری آن یعنی Chrome OS است. این سیستم نیز یک Linux system است و بخش زیادی از technology های desktop Linux را استفاده می‌ کند ، اما تجربه‌ ی کاربری آن ب ه‌شکل متفاوتی سازمان‌ دهی شده است. در این مدل ، محور اصلی محیط کاربری Chromium / Chrome browser است. بخش قابل‌ توجهی از چیزهایی که در یک traditional desktop Linux می‌ بینید در Chrome OS حذف یا ساده شده‌ اند. در نتیجه می‌ توان Chrome OS را نمونه‌ ای از این دانست که چگونه همان زیرساخت Linux می‌ تواند در قالب یک desktop بسیار متفاوت از GNOME یا KDE استفاده شود.

---

### Desktop Architecture and Summary

فصل چهاردهم در نهایت یک نکته‌ ی معماری مهم را روشن می‌ کند: Linux desktop یک برنامه‌ ی واحد نیست ، بلکه مجموعه‌ ای از component های مستقل است که هر کدام مسئولیت مشخصی دارند و از طریق interface ها ، protocol ها ، library ها و IPC با یکدیگر همکاری می‌ کنند ، در مسیر زیر می‌ توان معماری را به‌ شکل زیر در نظر گرفت : 

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

در مدل X ، یک X server مرکزی نقش اصلی را در مدیریت display ، rendering و input دارد و application ها به‌ عنوان client با آن ارتباط برقرار می‌ کنند. در مدل Wayland ، خود protocol مسئول ایجاد یک desktop کامل نیست.  graphical client ها buffer های خودشان را دارند و compositor آن‌ ها را ترکیب می‌ کند و خروجی نهایی را برای display آماده می‌ سازد. به همین دلیل در Wayland ، compositor نقش بسیار مرکزی‌ تری دارد. در مسیر input ، تصویر متفاوتی داریم :

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

در اینجا libinput می‌ تواند event های input device های kernel را جمع‌ آوری و استاندارد کند ، در حالی که در X ابزارهایی مانند X Input Extension ، xinput ، XKB و xmodmap برای مشاهده و تنظیم input مورد استفاده قرار می‌ گیرند. در سطح application ، toolkit هایی مانند GTK+ و Qt امکانات لازم برای ساخت widget های گرافیکی را فراهم می‌کنند. سپس desktop environment هایی مانند GNOME ، KDE و Xfce مجموعه‌ ی بزرگ‌ تری از toolkit ها ، library ها ، theme ها ، icon ها و convention ها را در کنار هم قرار می‌ دهند تا application ها رفتار و ظاهر هماهنگ‌ تری داشته باشند. برای ارتباط بین process ها و D-Bus یک مسیر جداگانه فراهم می‌ کند :

```
Process
    |
    ↕
D-Bus
    |
    ↕
Process
```

در این معماری dbus-daemon نقش hub را دارد و event ها و پیام‌ ها را میان process های علاقه‌ مند جا به‌ جا می‌ کند. System bus برای communication سطح سیستم و session bus برای communication مربوط به desktop session استفاده می‌ شوند ، مسیر چاپ نیز یک pipeline جداگانه دارد :

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

در این مسیر CUPS زیرساخت اصلی چاپ است ، cupsd daemon آن را اجرا می‌ کند و IPP برای communication با client ها ، printer ها و print server ها کاربرد دارد. اگر printer مستقیماً format ورودی را پشتیبانی نکند ، component هایی مانند Ghostscript ، RIP و PPD در تبدیل document و تطبیق آن با قابلیت‌ های printer نقش پیدا می‌ کنند. به همین دلیل هنگام troubleshooting بهتر است desktop را به‌ عنوان یک component واحد در نظر نگیریم. اگر مشکل مربوط به window placement یا نحوه‌ ی ترکیب window ها باشد ، باید window manager یا compositor را بررسی کنیم. اگر مشکل مربوط به input باشد ، مسیر device ، kernel input ، libinput یا X Input اهمیت پیدا می‌ کند. اگر یک X application در محیط Wayland رفتار غیرمنتظره‌ ای داشته باشد ، باید وجود و عملکرد Xwayland را در نظر گرفت. اگر مشکل مربوط به ارتباط بین application ها یا event های system باشد ، D-Bus یکی از component های اصلی برای بررسی است و اگر مشکل مربوط به printing باشد ، باید queue ، filter ، RIP ، driver و backend را به‌ عنوان مرحله‌ های جداگانه در نظر گرفت بنابراین معماری کلی فصل را می‌ توان بدون تکرار مطالب قبلی به این شکل خلاصه کرد :

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

در نتیجه ، چیزی که باید از این فصل در ذهن بماند این نیست که **Linux desktop یک سیستم بزرگ و یکپارچه است** بلکه برعکس ، باید آن را **مجموعه‌ ای از component های مستقل ، قابل‌ تعویض و قابل‌ بررسی دید**. همین نگاه معماری است که عیب‌یابی را ساده می‌ کند ، به‌جای جست‌ وجو در کل desktop ، ابتدا مسیر مربوط به مشکل را مشخص می‌ کنیم و سپس component مسئول همان مرحله را بررسی می‌ کنیم.
