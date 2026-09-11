## شبکه از دید لینوکس
### 🐧 فصل دوم کتاب How Linux Works

فصل‌ های قبل عمد تاً داخل یک سیستم لینوکسی می‌ گذشتند و درباره‌ ی process ها ، resource ها ، kernel و نحوه‌ ی مدیریت آن‌ ها صحبت می‌ کردند. در این فصل از مرز یک ماشین عبور می‌ کنیم و وارد دنیای شبکه می‌ شویم یعنی بررسی می‌ کنیم یک سیستم Linux چطور داده را به مقصد می‌ رساند ، چطور داده‌ ی دریافتی را تشخیص می‌ دهد ، چه چیزی مسئول انتخاب مسیر است ، hostname ها چطور به IP تبدیل می‌ شوند و سرویس‌ هایی مثل DHCP، DNS، NAT و Firewall در این معماری چه نقشی دارند. 

دو سؤال اصلی که این فصل دنبال می‌ کند این‌ها هستند :

1. وقتی داده‌ ای را ارسال می‌ کنیم ، Linux از کجا می‌ فهمد packet باید کجا برود ؟
2. وقتی packet به سیستم می‌ رسد ، Linux از کجا می‌ فهمد این packet مربوط به کدام interface ، address ، port و application است ؟

برای جواب دادن به این سؤال‌ ها باید چند مفهوم را کنار هم ببینیم :

```
Application
    ↓
Transport
    ↓
IP / Network
    ↓
Link
    ↓
Physical / Wireless
```

در Linux  بخش مهمی از این مسیر توسط kernel مدیریت می‌ شود، در حالی که ابزارهای user space وظیفه‌ ی پیکربندی ، مشاهده ، مدیریت و ساده‌ کردن این زیرساخت را بر عهده دارند. این جداسازی در تمام فصل دیده می‌ شود.

---
📚 Table of Contents

- [Network Basics](#network-basics)
- [Packets](#packets)
- [Network Layers](#network-layers)
- [The Internet Layer](#the-internet-layer)
- [Subnets](#subnets)
- [CIDR Notation](#cidr-notation)
- [Routes and the Kernel Routing Table](#routes-and-the-kernel-routing-table)
- [The Default Gateway](#the-default-gateway)
- [IPv6 Addresses and Networks](#ipv6-addresses-and-networks)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)
- [](#)

---

### Network Basics

یک شبکه‌ ی ساده‌ ی خانگی یا اداری را می‌ توان با چند host تصور کرد که به یک router متصل شده‌ اند مثلاً :

```
Host A ─┐
Host B ─┼── Router ─── Internet
Host C ─┘
```

هر host می‌ تواند یک کامپیوتر، سرور، تلفن یا دستگاه دیگر ی باشد. Router نقش اتصال شبکه‌ ی محلی به شبکه‌ های دیگر را بر عهده دارد و معمولاً مسیر خروجی به اینترنت را هم فراهم می‌ کند. اما زمانی که یک برنامه روی Host A می‌ خواهد داده‌ ای برای Host B ارسال کند ، داده به‌ صورت یک بلوک بزرگ و یکپارچه روی شبکه حرکت نمی‌ کند. داده در مسیر شبکه به واحد های کوچک‌ تری به نام packet تقسیم می‌ شود. Packet معمولاً دو بخش مفهومی اصلی دارد : Header و Payload.

#### 🔹 Header

اطلاعات کنترلی packet را نگه می‌ دارد. بسته به پروتکل ، ممکن است اطلاعاتی درباره‌ ی  Source و Destination و Protocol و Length و Flags و Sequence information در header وجود داشته باشد .

#### 🔹 Payload

بخش داده‌ ای است که packet واقعاً حمل می‌ کند به‌ صورت ساده :

```
+------------------+----------------------+
| Header           | Payload              |
+------------------+----------------------+
```

هر لایه می‌تواند header مخصوص خودش را به داده اضافه کند. این طراحی باعث می‌ شود پروتکل‌ های مختلف بتوانند روی یک زیرساخت مشترک کار کنند و هر لایه وظیفه‌ی مشخص خودش را انجام دهد. برای مثال IP بیشتر با آدرس‌ دهی و routing سروکار دارد ،  در حالی که TCP مسئله‌ ی ترتیب داده ، reliability و ارتباط بین endpoint ها را مدیریت می‌ کند.

---

### Packets

یکی از مفاهیم بنیادی شبکه Packet  است. وقتی application یک مقدار داده ارسال می‌ کند، آن داده از چند لایه عبور می‌ کند و هر لایه اطلاعات مورد نیاز خودش را به آن اضافه می‌ کند. به شکل مفهومی :

```
Application Data
         ↓
Transport Header + Data
         ↓
IP Header + Transport Segment
         ↓
Link Header + IP Packet
```

به همین دلیل واژه‌ ی packet در همه‌ ی سطوح دقیقاً به یک چیز اشاره نمی‌ کند. مثلاً TCP بیشتر با segment و UDP با datagram صحبت می‌ کند ، در حالی که IP واحدی را حمل می‌ کند که معمولاً IP packet نامیده می‌ شود. یک packet می‌تواند در طول مسیر از چند router عبور کند. Router ممکن است packet را از یک interface دریافت کند ، header مربوط به IP را بررسی کند ، routing decision بگیرد و packet را از interface دیگری ارسال کند. این یعنی packet لزوماً یک‌ باره از فرستنده به مقصد نمی‌ رسد.

---

### Network Layers

شبکه از چند لایه تشکیل شده است و هر لایه وظیفه‌ ی مشخصی دارد. یکی از مدل‌ های ساده برای درک این ساختار :

```
Application Layer
    ↓
Transport Layer
    ↓
Internet / Network Layer
    ↓
Link Layer
    ↓
Physical Layer
```

#### 🔹 Application Layer

در این سطح پروتکل‌ هایی قرار می‌ گیرند که مستقیماً برای application ها معنا دارند مثلاً HTTP و DNS و SSH. این پروتکل‌ ها معمولاً در user space و در context برنامه‌ ها استفاده می‌ شوند.

#### 🔹 Transport Layer

دو پروتکل بسیار مهم در این لایه TCP و UDP هستند. TCP امکان ارتباط connection-oriented و قابل‌اعتماد را فراهم می‌کند ، در حالی که UDP مدل ساده‌ تری دارد و تضمین‌های TCP را ارائه نمی‌ دهد.

#### 🔹 Internet / Network Layer

این لایه بیشتر با IP addressing و Routing و Packet forwarding سر و کار دارد. در این سطح packet باید بتواند از یک شبکه به شبکه‌ ی دیگر حرکت کند. 

#### 🔹 Link Layer

این لایه مسئول انتقال داده در یک شبکه‌ ی local یا یک لینک مشخص است. Ethernet و Wi-Fi نمونه‌ های مهم این سطح هستند. در این لایه با مفاهیمی مانند MAC address و frame سروکار داریم.

#### 🔹 Physical Layer

در پایین‌ ترین سطح، خود media انتقال قرار می‌ گیرد مانند : Copper و Fiber و Radio.

> 💡 در Linux ، بخش عمده‌ ی transport و لایه‌ های پایین‌تر توسط kernel و network stack مدیریت می‌ شود ، در حالی که application ها معمولاً از API های socket استفاده می‌ کنند و مجبور نیستند مستقیماً جزئیات این لایه‌ ها را مدیریت کنند.

---

### The Internet Layer

در لایه‌ ی Internet ، مفهوم مهمی به نام **IP address** داریم. هر host یا network interface می‌ تواند یک یا چند IP address داشته باشد. در IPv4 ، آدرس از 32 بیت تشکیل شده و معمولاً به شکل a.b.c.d که چهار عدد اعشاری نوشته می‌ شود مثلاً 10.23.2.37 .

برای مشاهده‌ ی  address های interface های سیستم Linux می‌ توانید ازدستور ```ip address show``` یا شکل کوتاه‌ تر ```ip addr``` استفاده کنید. 

<img width="100%" height="179" alt="image" src="https://github.com/user-attachments/assets/ea7c8b84-bdf8-4f40-b6de-73b39455c6e1" />


این دستور فقط IP address را نشان نمی‌ دهد بلکه اطلاعاتی درباره‌ ی interface و بعضی metadata های مربوط به آن نیز ارائه می‌ کند. در خروجی معمولاً باید به interface هایی مثل lo و enp0s31f6 و eth0 و wlan0 و address های اختصاص‌ یافته به آن‌ ها توجه کنید. نکته‌ ی مهم این است که یک ماشین می‌ تواند چند interface و هر interface نیز بسته به پیکربندی چند address داشته باشد.

<img width="100%" height="428" alt="image" src="https://github.com/user-attachments/assets/d32535ce-f774-4785-8622-14ee5f8f1d6d" />

<img width="100%" height="482" alt="image" src="https://github.com/user-attachments/assets/5d93b0d5-2252-4e03-926d-efbbdeb8dd35" />

---

### Subnets

داشتن IP address به‌ تنهایی برای تصمیم‌ گیری درباره‌ ی routing کافی نیست. سیستم باید بداند کدام address ها بخشی از یک local subnet  هستند و کدام destination ها در شبکه‌ های دیگر قرار دارند. یک subnet مجموعه‌ ای از address هاست که یک بخش مشترک از address space را دارند.  به‌صورت مفهومی IP address به دو قسمت تقسیم می‌ شود Network Prefix و Host Portion . مثلاً اگر subnet این 192.168.1.0/24 باشد ، بخش 24/ مشخص می‌ کند که 24 بیت اول برای network prefix استفاده می‌ شوند.

در یک subnet با 24/ ، address هایی که در بخش اول مشترک هستند معمولاً در همان subnet قرار می‌ گیرند. Subnet mask متناظر با 24/ عبارت است از 255.255.255.0 و Subnetting به سیستم‌ ها کمک می‌ کند تشخیص دهند یک destination باید مستقیماً روی شبکه‌ ی local پیدا شود یا از یک router عبور کند.

<img width="100%" height="74" alt="image" src="https://github.com/user-attachments/assets/76365e1f-ff94-48fa-82b3-230e350de011" />

---

### CIDR Notation

نوشتن subnet mask به شکل کامل همیشه لازم نیست به جای 192.168.1.0 و 255.255.255.0 می‌توان از 192.168.1.0/24 استفاده کرد. عدد بعد از / تعداد بیت‌ های ابتدایی mask را نشان می‌ دهد مثلاً 8/ یعنی 8 بیت اول مربوط به prefix هستند و 16/ ، 24/ و /32 نیز به همین صورت که یعنی تمام 32 بیت IPv4 address مشخص شده‌ اند و معمولاً یک host route را نشان می‌ دهد. برای IPv6 نیز CIDR همین مفهوم را دارد ، با این تفاوت که address دارای 128 بیت است.

<img width="100%" height="316" alt="image" src="https://github.com/user-attachments/assets/928ac470-d0f8-4c62-b890-68a32c94e5aa" />


---

### Routes and the Kernel Routing Table

بعد از اینکه مقصد را شناختیم ، سؤال بعدی این است که packet از کدام مسیر باید ارسال شود؟ این کار را routing انجام می‌ دهد. Kernel یک **routing table** دارد که در آن مشخص می‌ شود برای destination های مختلف باید از چه route استفاده شود. برای مشاهده‌ی این جدول دستور ```ip route show``` یا ```ip route```را اجرا کنید. خروجی ممکن است چیزی شبیه این باشد :

<img width="100%" height="107" alt="image" src="https://github.com/user-attachments/assets/fda3216e-c8c6-4541-b495-cac98c5d4ff4" />


خط دوم می‌ گوید شبکه‌ ی 192.168.1.0/24 از طریق local  interface  قابل دسترسی است. اما برای destination هایی که در این subnet قرار ندارند ، معمولاً packet به router فرستاده می‌ شود.

#### 🔹 Direct Route

اگر destination در همان local subnet باشد ، host می‌ تواند بدون عبور از router اصلی به آن destination دسترسی پیدا کند. البته برای ارسال روی link باید MAC مقصد نیز مشخص شود که در IPv4 با ARP و در IPv6 با NDP انجام می‌ شود.

#### 🔹 Indirect Route

اگر مقصد در subnet دیگری باشد، host معمولاً packet را به router یا next hop مناسب می‌ دهد.

بنابراین routing table حلقه‌ ی اتصال بین Destination IP و Route و Next Hop / Interface است.

---

### The Default Gateway

در routing table معمولاً route با نام default وجود دارد. این route برای destination هایی استفاده می‌ شود که route مشخص‌ تر و مناسب‌ تری برایشان وجود ندارد مثلاً ```default via 192.168.1.1 dev eth0``` یعنی برای مقصد هایی که route دقیق‌ تری ندارند ، packet از طریق gateway به 192.168.1.1 ارسال شود.

این gateway معمولاً همان router شبکه‌ ی local است. اما نکته‌ ی بسیار مهمی درباره‌ ی routing وجود دارد ، **kernel صرفاً اولین route موجود را انتخاب نمی‌ کند**. اگر چند route با مقصد یکسان یا overlapping وجود داشته باشند ، اصل مهم **longest prefix match** است.

مثلاً 10.0.0.0/8 و 10.20.0.0/16 و 10.20.30.0/24 اگر destination این باشد 10.20.30.50 ، route /24 از 16/ و 8/ مشخص‌ تر است ، بنابراین در حالت عادی route با طولانی‌ ترین prefix انتخاب می‌ شود در نتیجه : 

```
more specific route
        ↓
priority over less specific route
```

در واقع default با 0.0.0.0/0 کم‌ اختصاصی‌ ترین route است .

---

### IPv6 Addresses and Networks

در IPv4 فقط 32 بیت address در اختیار می‌ گذارد و همین محدودیت باعث شد استفاده از IPv6 اهمیت پیدا کند. IPv6 دارای  address های 128bit است. به همین دلیل تعداد address های قابل استفاده در IPv6 بسیار بزرگ‌ تر از IPv4 است. یک IPv6 address معمولاً به شکل هگزادسیمال و در چند گروه نوشته می‌ شود مثلاً ```2001:db8:1234:5678::1``` صفر های متوالی را می‌ توان با ```::```کوتاه کرد.

#### 🔹 Link-Local Addresses

در IPv6 یک مفهوم مهم، **link-local address** است این address ها معمولاً از بازه‌ ی ```fe80::/10``` استفاده می‌ کنند. Link-local address برای ارتباط روی همان link بسیار مهم است و در بسیاری از پیکربندی‌ های IPv6 حتی بدون داشتن global address نیز وجود دارد.

#### 🔹 Global Unicast

در صورت نیاز، interface می‌ تواند یک global unicast address نیز داشته باشد که برای ارتباط در شبکه‌ های گسترده‌ تر استفاده می‌ شود. برای مشاهده‌ ی address های IPv6 دستور ```ip -6 address show``` را اجرا کنید.

یکی از تفاوت‌ های مهم IPv6 این است که network configuration به اندازه‌ ی IPv4 وابسته به یک روش واحد نیست و مکانیزم‌ هایی مثل SLAAC می‌ توانند نقش مهمی داشته باشند که در ادامه به آن‌ ها برمی‌ گردیم.

<img width="100%" height="215" alt="image" src="https://github.com/user-attachments/assets/f7eefc0f-f7d6-435d-9506-2a4812aece60" />

