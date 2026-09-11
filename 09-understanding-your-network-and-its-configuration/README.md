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
- [Configuring Dual Stack Networks](#configuring-dual-stack-networks)
- [Basic ICMP and DNS Tools](#basic-icmp-and-dns-tools)
- [The Physical Layer and Ethernet](#the-physical-layer-and-ethernet)
- [Introduction to Network Interface Configuration](#introduction-to-network-interface-configuration)
- [Problems with Manual and Boot Activated Network Configuration](#problems-with-manual-and-boot-activated-network-configuration)
- [Network Configuration Managers](#network-configuration-managers)
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

---

### Configuring Dual Stack Networks

هر دو IPv4 و IPv6 ، دو protocol مستقل هستند. می‌ توان یک ماشین را طوری پیکربندی کرد که هم‌ زمان هر دو را داشته باشد. این حالت را ```dual-stack``` می‌ نامند. مثلاً یک interface ممکن است هم‌ زمان داشته باشد : 

```
IPv4: 192.168.1.20/24
IPv6: 2001:db8::20/64
```

و علاوه بر آن یک IPv6 link-local address نیز داشته باشد. در dual-stack ، application در صورت پشتیبانی می‌ تواند با IPv4 یا IPv6 ارتباط برقرار کند. این نکته مهم است که داشتن IPv6 باعث حذف IPv4 نمی‌شود بلکه در dual-stack هر دو stack به‌ صورت موازی وجود دارند.

---

### Basic ICMP and DNS Tools

 برای شروع troubleshooting شبکه معمولاً دو مسئله‌ ی بسیار رایج داریم.  آیا مقصد قابل دسترسی است؟  ، آیا نام host به address درست resolve می‌ شود؟ ping و host دو ابزار ساده ولی ارزشمند برای این کار هستند.

#### 🔹 ping

یکی از شناخته‌ شده‌ ترین ابزارهای عیب‌ یابی شبکه ping است. در حالت معمول ، ping از ICMP Echo Request استفاده می‌ کند و منتظر ICMP Echo Reply می‌ ماند مثلاً ```ping 8.8.8.8``` اگر reply دریافت شود ، اطلاعاتی مثل sequence و time و TTL ممکن است نمایش داده شوند. Sequence number کمک می‌ کند reply های مختلف از هم تشخیص داده شوند مثلاً اگر sequence های  1 و 2 و 3 و 5 را ببینید و 4 گم شده باشد ، احتمال packet loss یا مشکل در مسیر وجود دارد. 

اما ping باید با احتیاط تفسیر شود. عدم پاسخ ping الزاماً به معنی «host خاموش است» نیست. ممکن است firewall یا تنظیمات شبکه ICMP را فیلتر کنند بنابراین ```ping failure``` لزومی ندارد مساوی باشد با ```host unreachable``` همان‌ طور که ```ping success``` نیز لزوماً به این معنی نیست که یک service خاص روی آن host قابل دسترسی است. ping بیشتر کمک می‌ کند connectivity در یک سطح مشخص را بررسی کنید.

<img width="100%" height="207" alt="image" src="https://github.com/user-attachments/assets/2c75aadc-4e7b-4b02-a88c-0984af0b688c" />

#### 🔹 DNS and host 

برای ماشین‌ها IP address مناسب است ، اما برای انسان‌ ها حفظ کردن تعداد زیادی address دشوار است. به همین دلیل از hostname هایی مانند ```www.example.com``` استفاده می‌ کنیم. DNS وظیفه دارد نام‌ ها را به address ها و در بعضی شرایط address ها را به نام تبدیل کند. برای lookup ساده می‌ توان از ```host www.example.com``` استفاده کرد.

این دستور ممکن است IPv4 و IPv6 address هایی را که از DNS دریافت شده‌ اند نمایش دهد. برای reverse lookup نیز می‌توان address را در اختیار host قرار داد. DNS در application layer قرار دارد ، اما اطلاعاتی که از DNS دریافت می‌ شود در بخش‌ های پایین‌ تر شبکه برای برقراری ارتباط استفاده می‌ شود.

<img width="100%" height="107" alt="image" src="https://github.com/user-attachments/assets/b55ef011-bb6c-45cb-948a-ebd177532e6a" />

---

### The Physical Layer and Ethernet

تا اینجا بیشتر درباره‌ی IP و routing صحبت کردیم ، اما packet برای حرکت واقعی روی شبکه باید از یک link عبور کند. در Ethernet ، دستگاه‌ ها علاوه بر IP address ، یک MAC address یا hardware address نیز دارند. مثلاً یک MAC address می‌ تواند به شکلی شبیه 00:11:22:33:44:55 باشد. MAC address با IP address یک مفهوم نیست. به‌صورت ساده IP Address یک آدرس منطقی در لایه IP است ، در حالی که MAC Address یک آدرس در لایه Link است.

برای انتقال داده در شبکه‌ ی  local Ethernet، داده در قالب frame منتقل می‌ شود. Frame شامل اطلاعاتی مانند Source  MAC و Destination MAC و Payload است. IP packet درون این frame قرار می‌ گیرد اما Ethernet به‌ تنهایی برای عبور packet از چند شبکه کافی نیست. اگر destination روی شبکه‌ ای دیگر باشد ، frame local packet را تا router بعدی می‌ برد. سپس router packet را در یک frame جدید روی link بعدی قرار می‌ دهد یعنی :

```
IP packet
    ↓
Ethernet frame
    ↓
Router
    ↓
New Ethernet frame
    ↓
Next hop
```

به همین دلیل MAC address معمولاً برای یک link محلی اهمیت دارد ، در حالی که IP address در طول مسیر برای مشخص‌ کردن source و destination منطقی اهمیت خود را حفظ می‌ کند.

--- 

### Understanding Kernel Network Interfaces

کرنل برای ارتباط بین network stack و سخت‌ افزار از مفهوم **network interface** استفاده می‌ کند. interface می‌ تواند نماینده‌ ی یک دستگاه واقعی یا یک logical/virtual interface باشد. در سیستم‌ های مدرن ممکن است نام‌ هایی مثل ```enp0s31f6``` و ```ens33``` و ```wlp2s0``` ببینید.

این naming بر اساس مفهوم **predictable network interface names*** شکل گرفته تا نام‌ گذاری تا حد ممکن به سخت‌افزار و محل آن وابستگی قابل‌ پیش‌ بینی داشته باشد. در سیستم‌ های قدیمی‌ تر ممکن بود با نام‌ هایی مانند ```eth0```  و ```eth1``` و ```wlan0``` روبرو شوید. هر دو مدل هنوز در محیط‌ های مختلف دیده می‌ شوند.

برای مشاهده‌ ی interface ها دستور ```ip link show``` و برای دیدن address ها ```ip address show``` مناسب هستند. نکته‌ ی مهم این است که interface name با device driver و همچنین با IP address یکسان نیست. Interface در واقع یکی از نقاط اصلی اتصال kernel network stack به device یا virtual network layer است.

---

### Introduction to Network Interface Configuration

برای اینکه یک سیستم Linux واقعاً روی شبکه کار کند ، چند مرحله‌ ی کلی لازم است :

- **مرحله‌ ی اول سخت‌ افزار و driver** :  باید Kernel بتواند hardware را شناسایی کند و driver مناسب را در اختیار داشته باشد.
- **مرحله‌ ی دوم آماده‌ کردن link** : در Ethernet ممکن است link فیزیکی برقرار شود. در Wi-Fi علاوه بر آن باید به network مناسب متصل شویم و در صورت نیاز authentication انجام دهیم.
- **مرحله‌ ی سوم تنظیم IP** : باید Interface یک یا چند IP address داشته باشد و subnet مشخص شود مثلاً 192.168.1.20/24.
- **مرحله‌ ی چهارم routing** : سیستم باید route های لازم را بداند. این route ها می‌توانند شامل Local subnet و Default gateway و Specific network routes باشند.

بنابراین پیکربندی شبکه فقط به اختصاص دادن یک IP محدود نمی‌ شود به شکل مفهومی :

```
Hardware
   ↓
Interface
   ↓
IP address
   ↓
Subnet
   ↓
Routes
   ↓
Working network
```

#### 🔹 Manually Configuring Interfaces

برای آزمایش ، troubleshooting یا محیط‌ های خاص می‌توان interface را مستقیماً با ابزار ip پیکربندی کرد مثلاً  :

```ip address add 192.168.1.20/24 dev eth0```

این دستور یک address روی interface مشخص اضافه می‌ کند. همچنین می‌توان route اضافه کرد :

```ip route add 10.10.0.0/16 via 192.168.1.1```

این نوع پیکربندی معمولاً runtime است و ممکن است بعد از reboot باقی نماند مگر اینکه توسط سیستم configuration مدیریت شود. در محیط‌ های واقعی معمولاً configuration network را یک network manager یا service مناسب مدیریت می‌ کند.

#### 🔹 Manually Adding and Deleting Routes

همچنین route ها را نیز می‌ توان به‌ صورت دستی ایجاد و حذف کرد : 

- برای مشاهده ```ip route show``` استفاده می‌ شود.
- برای اضافه کردن ```ip route add``` استفاده می‌ شود.
- برای حذف ```ip route delete``` استفاده می‌ شود.

مثلاً ```ip route add 10.0.0.0/8 via 192.168.1.1``` ، اما باید همیشه توجه داشته باشید که route جدید ممکن است با route های موجود overlap داشته باشد. در این حالت باید longest prefix match و سایر ویژگی‌ های routing را در نظر گرفت.

---

### Boot-Activated Network Configuration

در سیستم‌ های قدیمی Unix/Linux ، تنظیمات شبکه معمولاً در زمان boot انجام می‌ شد. init یا  system startup اسکریپت‌ هایی اجرا می‌ کرد که interface ها را فعال و address ها و route ها را تنظیم می‌ کردند. ابزارهایی مثل ifup و  ifdown نمونه‌ ای از این رویکرد هستند.

در این مدل فرض بر این بود که configuration سیستم تا حد زیادی ثابت است اما شبکه‌ های مدرن بسیار پویا تر شده‌ اند به همین دلیل ابزارهای جدید تری برای مدیریت شبکه شکل گرفته‌ اند  :

```
Boot
 ↓
Configure interface
 ↓
Set IP
 ↓
Set routes
```

در برخی توزیع‌ ها ، **Netplan** یک لایه‌ ی configuration ارائه می‌ دهد که معمولاً فایل‌ های YAML را در etc/netplan/ قرار می‌ دهد. بعد این configuration به backend مناسب سپرده می‌ شود برای مثال NetworkManager و systemd-networkd . بنابراین Netplan خودش لزوماً network manager نهایی نیست بلکه بیشتر یک لایه‌ ی configuration و abstraction برای backend است.

---

### Problems with Manual and Boot Activated Network Configuration

مدل‌ های قدیمی در سیستم‌ هایی با شبکه‌ های ساده قابل‌ قبول بودند ، اما شبکه‌ های امروزی شرایط پیچیده‌ تری دارند. برای مثال، یک لپ‌ تاپ ممکن است در خانه به Wi-Fi A ، در محل کار به Wi-Fi B و در حال حرکت به شبکه موبایل متصل باشد.

در چنین محیطی فقط اجرای چند دستور در boot کافی نیست و مسائلی باید به‌ صورت dynamic مدیریت شوند مثل :

```
Which interface is available?
Which Wi-Fi should be used?
Was authentication successful?
Did DHCP succeed?
Did the link disappear?
Did the cable reconnect?
Did the address change?
```

به همین دلیل network configuration به سمت daemon ها و manager های هوشمند تر حرکت کرده است.

---

### Network Configuration Managers

یکی از network manager های بسیار رایج در لینوکس NetworkManager است. در برخی سیستم‌ ها ، به‌ خصوص server ها و محیط‌ هایی که configuration ساده و کنترل‌ شده‌ای دارند ، ممکن است systemd-networkd انتخاب شود. هر دو می‌ توانند بخش مهمی از lifecycle شبکه را مدیریت کنند.

#### 🔹 NetworkManager Operation

می‌تواند اطلاعات مختلفی را درباره‌ ی network device ها و connection ها مدیریت کند مثلاً Ethernet و Wi-Fi و DHCP و Static IP و Routes و DNS-related settings می‌ تواند در configuration آن قرار بگیرد.  در Wi-Fi ، هم NetworkManager می‌ تواند در ارتباط با سیستم‌ های پایین‌ تر شبکه و authentication عمل کند و connection profile مربوط به شبکه را مدیریت نماید. در نتیجه کاربر لازم نیست برای هر تغییر network به‌ صورت دستی command اجرا کند.

#### 🔹 NetworkManager Interaction

برای کار با NetworkManager ابزارهای مختلفی وجود دارند. مزیت این ابزارها این است که interface مدیریتی مناسبی برای کار با configuration شبکه در user space در اختیار قرار می‌ دهند. 

 در محیط graphical می‌ توان از applet های دسکتاپ استفاده کرد و در command line ابزار بسیار مهم ```nmcli```است مثلاً : 

-  دستور ```nmcli device``` برای مشاهده‌ ی وضعیت device ها مفید است.
- دستور ```nmcli connection``` برای مشاهده‌ ی connection profile ها کاربرد دارد.

ابزار nm-online نیز برای بررسی اینکه آیا NetworkManager وضعیت اتصال را به‌عنوان online تشخیص داده است یا نه  استفاده می‌ شود. 

#### 🔹 NetworkManager Configuration

دایرکتوری اصلی configuration مربوط به NetworkManager معمولاً در مسیر /etc/NetworkManager است. یکی از فایل‌ های شناخته‌ شده NetworkManager.conf است. بسته به توزیع و تنظیمات ، connection profile ها نیز در مکان‌ های مشخصی نگهداری می‌ شوند. NetworkManager قابلیت dispatcher نیز دارد و Dispatcher اجازه می‌ دهد هنگام رخداد هایی مثل interface up و interface down و connection change اسکریپت‌ هایی اجرا شوند.

این قابلیت برای واکنش دادن به تغییر وضعیت شبکه کاربردی است. مثلاً یک سیستم می‌ تواند هنگام برقرار شدن یک connection خاص ، تنظیم یا سرویس دیگری را فعال کند.

---

###



