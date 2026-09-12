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
- [Localhost](#localhost)
- [The Transport Layer](#the-transport-layer)
- [Understanding DHCP](#understanding-dhcp)
- [Automatic IPv6 Network Configuration](#automatic-ipv6-network-configuration)
- [Configuring Linux as a Router](#configuring-linux-as-a-router)
- [Private Networks](#private-networks)
- [Network Address Translation](#network-address-translation)
- [Routers and Linux](#routers-and-linux)
- [Firewalls](#firewalls)
- [Ethernet and IP and ARP and NDP](#ethernet-and-ip-and-arp-and-ndp)
- [Wireless Ethernet](#wireless-ethernet)
- [Tips](#tips)

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

### Resolving Hostnames

برای kernel و network stack  هم IP address مهم  است ، ولی application ها اغلب با hostname کار می‌ کنند. مثلاً برنامه به جای 192.0.2.20 ممکن است بخواهد به server.example.com متصل شود. این کار نیازمند name resolution است. DNS در application layer و user space قرار دارد و بسیاری از برنامه‌ ها مستقیماً خودشان DNS protocol را پیاده نمی‌ کنند. در عوض معمولاً از library ها و system APIs برای lookup استفاده می‌ کنند. 

برای kernel و network stack  هم IP address مهم  است ، ولی application ها اغلب با hostname کار می‌ کنند. مثلاً برنامه به جای 192.0.2.20 ممکن است بخواهد به server.example.com متصل شود. این کار نیازمند name resolution است. DNS در application layer و user space قرار دارد و بسیاری از برنامه‌ ها مستقیماً خودشان DNS protocol را پیاده نمی‌ کنند. در عوض معمولاً از library ها و system APIs برای lookup استفاده می‌ کنند. 

#### 🔹 /etc/hosts

فایل etc/hosts/ یک راه ساده و local برای تعریف mapping بین hostname و IP address است مثلاً : 

```
192.168.1.10    server1
192.168.1.11    server2
```
در چنین حالتی سیستم می‌ تواند بدون مراجعه به DNS  برای این نام‌ ها address پیدا کند. etc/hosts/ برای مواردی مثل local test environments و small networks و temporary overrides مفید است. اما برای شبکه‌ های بزرگ ، نگهداری دستی چنین mapping هایی معمولاً مناسب نیست.

#### 🔹 resolv.conf

فایل etc/resolv.conf/ اطلاعات مهمی درباره‌ ی name resolution ارائه می‌ دهد. یکی از مهم‌ ترین موارد آن nameserver است مثلاً ```nameserver 192.168.1.1```  یعنی سیستم ممکن است برای DNS query از آن resolver استفاده کند.

نکته‌ ی مهم این است که در سیستم‌ های مدرن /etc/resolv.conf ممکن است فایل واقعی نباشد و مثلاً به یک فایل تولید شده یا stub resolver اشاره کند. پس هنگام troubleshooting نباید صرفاً بر اساس ظاهر این فایل فرض کنید دقیقاً کدام daemon آن را تولید کرده است.

#### 🔹 Caching and Zero-Configuration DNS

معمولاً DNS query ها هزینه دارند و اگر یک hostname چند بار پشت سر هم resolve شود ، انجام query شبکه‌ ای برای هر درخواست منطقی نیست. به همین دلیل resolver ها و سرویس‌ های مربوط می‌ توانند پاسخ‌ ها را cache کنند. در Linux یکی از سرویس‌ هایی که ممکن است در این زمینه دیده شود ، ```systemd-resolved```است. این سرویس می‌ تواند DNS  response ها را cache کند و به application ها یک interface محلی برای resolution ارائه دهد. همچنین در شبکه‌ های محلی پروتکل‌ هایی مانند mDNS و LLMNR ممکن است برای name resolution بدون استفاده‌ ی کلاسیک از DNS server استفاده شوند. mDNS معمولاً برای local-name discovery اهمیت دارد. نکته‌ی مهم این است که استفاده از این سرویس‌ها universal نیست و توزیع ، نسخه و configuration سیستم تعیین می‌ کند چه resolver فعال باشد.

#### 🔹 /etc/nsswitch.conf

فایل etc/nsswitch.conf/ یکی از فایل‌ های مهم برای تعیین source های اطلاعات سیستم است. مثلاً system ممکن است برای hostname lookup چنین ترتیبی داشته باشد ```hosts: files dns``` یعنی اول /etc/hosts بعد DNS و NSS فقط برای hostname ها نیست. در سیستم Linux از همین مکانیزم می‌ توان برای lookup اطلاعات دیگری مثل users و groups نیز استفاده کرد. به این ترتیب برنامه‌ ها لازم نیست بدانند اطلاعات دقیقاً در کجا ذخیره شده‌ اند. برای مثال یک application می‌تواند از API استاندارد استفاده کند و NSS بعداً تصمیم بگیرد داده از etc/passwd/ یا LDAP یا SSSD یا source دیگری تأمین شود.

---

### Localhost

یکی از مهم‌ترین network interface های لینوکس lo است. این interface همان **loopback interface** است. هر داده‌ ای که به loopback ارسال شود به خود همان ماشین بر می‌گردد و از network hardware واقعی عبور نمی‌ کند.  در IPv4 address معروف 127.0.0.1 برای localhost استفاده می‌ شود. اما نکته‌ی مهم این است که کل بازه‌ ی  127.0.0.0/8 برای loopback رزرو شده است. بنابراین address هایی مانند 127.0.0.53 نیز می‌ توانند روی loopback استفاده شوند. در بعضی سیستم‌ ها مثلاً یک local DNS stub روی چنین address هم listen می‌ کند. در IPv6 آدرس اصلی loopback هم 1:: است. 

<img width="100%" height="242" alt="image" src="https://github.com/user-attachments/assets/9818baed-ae86-408b-97cb-d3438032f5e5" />

بنابراین IPv4 localhost برابر 127.0.0.1 و IPv6 localhost برابر با 1:: است. Loopback برای سرویس‌ هایی که فقط باید از همان ماشین قابل دسترسی باشند اهمیت زیادی دارد. مثلاً اگر یک application فقط روی 127.0.0.1:8080 ، listen کند ، سیستم‌ های دیگر معمولاً نمی‌ توانند مستقیماً به آن service متصل شوند ، حتی اگر ماشین روی شبکه‌ی بیرونی نیز address داشته باشد.

---

### The Transport Layer

تا اینجا IP تعیین می‌ کرد packet به کدام host برسد اما وقتی packet به یک host رسید ، یک سؤال دیگر داریم ، این داده مربوط به کدام application است؟ اینجا **port number** وارد می‌ شود. Transport layer بین application و IP قرار می‌ گیرد. دو پروتکل اصلی TCP و UDP هستند. TCP و UDP هر دو از port number استفاده می‌ کنند ، ولی مدل ارتباطی‌ شان متفاوت است.

#### 🔹 TCP Ports and Connections

یک host ممکن است هم‌ زمان چند سرویس داشته باشد مثلاً SSH و Web server و DNS server و Database همه‌ ی این سرویس‌ ها می‌ توانند از یک IP address استفاده کنند. Port number کمک می‌ کند ارتباط به application مناسب برسد. به‌ صورت مفهومی 192.168.1.10:22 ، 192.168.1.10:80 و 192.168.1.10:443 همگی ممکن است روی یک machine وجود داشته باشند ، ولی به سرویس‌ های متفاوت اشاره کنند.

برای مشاهده‌ ی socket ها و connection ها می‌ توان از ```ss``` استفاده کرد. netstat نیز ابزار شناخته‌ شده‌ ای است، ولی در سیستم‌ های جدید ```ss``` معمولاً انتخاب رایج‌ تری است.

وقتی یک client connection ایجاد می‌ کند ، معمولاً نیازی نیست خودش یک well-known port داشته باشد. Kernel می‌تواند یک port موقت به نام ephemeral port اختصاص دهد مثلاً :

```
Client:
192.168.1.50:52134

Server:
192.168.1.10:443
```

در این مثال 52134 یک client-side ephemeral port است در مقابل server معمولاً یک port شناخته‌ شده دارد.

برای برخی سرویس‌ ها شماره‌ های استاندارد وجود دارد مثلاً HTTPS → 443 . فهرست mapping های سرویس و port در etc/services/ قرار دارد. این فایل یک registry محلی برای نام سرویس‌ ها و port هاست البته application ها الزاماً مجبور نیستند از نام‌ های موجود در آن استفاده کنند.

در مدل سنتی Unix/Linux، پورت های  1023-1 هم privileged محسوب می‌ شوند. در حالت معمول ، bind کردن به چنین port هایی به امتیازهای خاصی نیاز دارد. در Linux این کار می‌ تواند با root یا capability مناسب مثل CAP_NET_BIND_SERVICE انجام شود. بنابراین جمله‌ی «فقط root می‌تواند port زیر 1024 را باز کند» دقیق نیست و capability ها نیز می‌ توانند نقش داشته باشند.

#### 🔹 TCP as a Byte Stream

پروتکل TCP مانند UDP پیام‌ های مستقل را تحویل application نمی‌ دهد  بلکه TCP یک byte stream ارائه می‌ کند. TCP داده را برای انتقال به segment ها تقسیم می‌ کند و در سمت مقصد با استفاده از sequence information آن را دوباره در قالب stream مناسب در اختیار application قرار می‌ دهد. ممکن است segment ها خارج از ترتیب دریافت شوند TCP این مسئله را مدیریت می‌ کند. در صورت نیاز retransmission نیز انجام می‌ شود در نتیجه application لازم نیست خودش packet های از دست‌ رفته را مجدداً مرتب و درخواست کند.

<img width="100%" height="701" alt="image" src="https://github.com/user-attachments/assets/d4597e48-5cfc-4e27-908b-ee10a2898499" />

#### 🔹 UDP

پروتکل UDP برخلاف TCP connection-oriented نیست. UDP برای ارسال datagram های مستقل طراحی شده و تضمین‌ های TCP را ندارد. به‌صورت کلی UDP تضمین نمی‌ کند که packet delivered و packet ordered و packet  retransmitted باشد. یعنی packet ممکن است lost و duplicated و delivered out of order شود و خود UDP این مشکل را مانند TCP حل  نمی‌کند.

این سادگی مزایایی دارد و UDP برای workload هایی مناسب است که simplicity و low overhead و low latency و application-level control اهمیت داشته باشند. NTP یکی از نمونه‌ های کلاسیک استفاده از UDP است. البته application هایی که از UDP استفاده می‌ کنند در صورت نیاز می‌ توانند خودشان reliability یا ordering را در لایه‌ ی application پیاده کنند.

---

### Understanding DHCP

در یک شبکه‌ ی ساده ممکن است IP address را به‌ صورت دستی برای هر machine تنظیم کنیم. اما اگر صد ها یا هزاران host داشته باشیم ، این روش عملی نیست. DHCP برای خودکار کردن بخشی از network configuration استفاده می‌ شود. وقتی host به‌ صورت DHCP client تنظیم شده باشد ، می‌ تواند از DHCP server اطلاعاتی مانند IP address و Subnet mask / prefix و Default gateway و DNS server دریافت کند. این اطلاعات معمولاً به‌صورت lease در اختیار client قرار می‌ گیرند یعنی lease برای همیشه تضمین نمی‌ شود و client در طول زمان باید آن را تمدید کند :
```
address assignment
        ↓
temporary lease
        ↓
renewal
```

#### 🔹 Linux DHCP Clients

در Linux نرم‌ افزارها و manager های مختلفی می‌ توانند DHCP client باشند یکی از ابزارهای سنتی dhclient است. در محیط‌ هایی که ```systemd-networkd``` مسئول network management باشد ، خود ```networkd``` نیز می‌ تواند نقش DHCP client را ایفا کند.  در سیستم‌ هایی که NetworkManager فعال است نیز DHCP می‌ تواند توسط خود NetworkManager یا component های مرتبط مدیریت شود. بنابراین «DHCP client در Linux» الزاماً به یک برنامه‌ ی واحد محدود نیست.

#### 🔹 Linux DHCP Servers

یک سیستم Linux فقط client نیست و می‌ تواند DHCP server نیز باشد. DHCP server می‌ تواند برای یک subnet address و configuration مورد نیاز client ها را ارائه کند.

---

### Automatic IPv6 Network Configuration

یکی از ویژگی‌های مهم IPv6، امکان پیکربندی stateless است و IPv6 فقط به DHCP وابسته نیست. در این مدل host می‌ تواند ابتدا یک link-local address ایجاد کند و سپس با استفاده از پیام‌ های router، prefix شبکه را دریافت کند.

#### 🔹 Router Advertisement

معمولاً Router ها می‌ توانند پیام‌ هایی به نام Router Advertisement یا RA ارسال کنند. این پیام‌ ها می‌ توانند اطلاعاتی درباره‌ ی prefix و ویژگی‌ های شبکه ارائه دهند. Host بر اساس این اطلاعات می‌ تواند address مناسب خودش را بسازد.

#### 🔹 Duplicate Address Detection

قبل از اینکه host address را به‌ صورت کامل استفاده  کند ، مکانیزم Duplicate Address Detection یا DAD برای بررسی duplicate نبودن address استفاده می‌ شود. به‌ صورت کلی :

```
Create tentative address
        ↓
Perform DAD
        ↓
Use address if no duplicate detected
```

این رویکرد با مدل کلاسیک DHCP متفاوت است. در IPv6 هنوز DHCPv6 نیز وجود دارد ، اما SLAAC و Router Advertisement می‌ توانند بخش مهمی از automatic configuration را انجام دهند.

---

### Configuring Linux as a Router

لینوکس می‌ تواند خودش نقش router را بازی کند. در ساده‌ ترین حالت یک router کامپیوتری است که حداقل دو مسیر یا interface شبکه دارد مثلاً :

```
Network A 192.168.1.0/24
       │
     eth0 → 192.168.1.1/24
     Linux
     eth1 → 192.168.2.1/24
       │
Network B 192.168.2.0/24
```
در این حالت Linux می‌ تواند بین دو subnet قرار بگیرد. هر interface باید address مناسب خودش را داشته باشد. اما صرف داشتن دو interface کافی نیست. به‌ صورت پیش‌ فرض kernel قرار نیست packet های ورودی از یک interface را به‌ طور خودکار از interface دیگر عبور دهد. برای فعال کردن IPv4 forwarding می‌توان از ```sysctl -w net.ipv4.ip_forward=1``` استفاده کرد. این تنظیم forwarding را در runtime فعال می‌ کند. برای پیکربندی دائمی معمولاً باید sysctl configuration مناسب نیز تنظیم شود و در IPv6 forwarding تنظیم جداگانه‌ ی مربوط به IPv6 وجود دارد.

<img width="100%" height="785" alt="image" src="https://github.com/user-attachments/assets/f9924d24-118e-4ecb-9e17-bcb87ca3403e" />

---

### Private Networks

فضای IPv4 محدود است. اگر قرار بود هر دستگاه داخل خانه یا شرکت یک public IPv4 address داشته باشد ، address space بسیار سریع مصرف می‌ شد. برای همین RFC 1918 سه بازه‌ ی معروف private IPv4 را مشخص می‌ کند 10.0.0.0/8 و 172.16.0.0/12 و 192.168.0.0/16 ، این address ها برای استفاده‌ ی داخلی شبکه‌ ها طراحی شده‌ اند و در اینترنت عمومی به‌ صورت عادی route نمی‌ شوند. مثلاً ممکن است شبکه‌ ی داخلی یک شرکت این باشد 10.0.0.0/8 و صدها host داخل آن از addressهای private استفاده کنند. این host ها برای دسترسی به اینترنت به mechanism دیگری نیاز دارند و رایج‌ ترین آن NAT است.

---

### Network Address Translation

نام NAT در Linux در سناریوی رایج خروجی شبکه **IP masquerading** نامیده می‌ شود. فرض کنید چند host داخلی داریم : 192.168.1.10 و 192.168.1.11 و 192.168.1.12 که همه باید از طریق یک public address به اینترنت دسترسی داشته باشند. Router می‌ تواند packet خروجی را تغییر دهد مثلاً :

```
Private source
192.168.1.10:50000

        ↓ NAT

Public source
203.0.113.10:40001
```

و Router اطلاعات مربوط به این translation را نگه می‌ دارد وقتی پاسخ برگردد ، router می‌ تواند آن را به host داخلی صحیح هدایت کند.

#### 🔹 Why is the port important ?

ممکن است چند host داخلی هم‌ زمان از port های مشابه استفاده کنند. بنابراین NAT می‌ تواند علاوه بر IP address، source port را نیز تغییر دهد و از ترکیب IP + Port برای tracking connection ها استفاده کند. به این مدل ترجمه در بسیاری از محیط‌های IPv4 اصطلاحاً PAT یا NAT overload نیز گفته می‌ شود.

در Linux می‌ توان NAT را با ابزارهای firewall stack مدیریت کرد. در محیط‌ های قدیمی iptables target  مانند MASQUERADE را ارائه می‌ کند. در سیستم‌ هایی که از nftables استفاده می‌ شود نیز می‌ توان NAT را در همان framework پیاده کرد.

> 💡 یک نکته‌ ی مهم ، NAT ذاتاً معادل firewall نیست. NAT می‌ تواند روی visibility و مسیر connection ها اثر بگذارد ، ولی سیاست امنیتی filtering موضوعی جداست. همچنین IPv6 به دلیل فضای address بسیار بزرگ برای ارتباط عادی host-to-host ذاتاً به NAT نیاز ندارد.

---

### Routers and Linux

بسیاری از router های تجاری در لایه‌ های پایین خود از Linux kernel یا سیستم‌ های مشابه Unix استفاده کرده‌ اند. سازنده‌ ی hardware می‌ تواند Linux kernel و Drivers و Networking features و Management software و Web interface را در یک محصول واحد قرار دهد. این موضوع باعث شده Linux در embedded networking کاربرد بسیار گسترده‌ ای داشته باشد. یکی از پروژه‌ های شناخته‌ شده در این حوزه OpenWRT است که برای اجرای یک سیستم Linux انعطاف‌ پذیر روی طیف بزرگی از hardware های router توسعه یافته است. چون این دستگاه‌ ها معمولاً resource محدودی دارند ، ابزارهای user space نیز ممکن است سبک باشند. اینجاست که BusyBox اهمیت پیدا می‌ کند. BusyBox یک باینری واحد است که مجموعه‌ ای از ابزارهای رایج Unix/Linux را در یک executable کوچک ارائه می‌ کند به همین دلیل در محیط‌ های embedded بسیار رایج است.

---

### Firewalls

یکی از مهم‌ ترین اجزای امنیت شبکه Firewall است. Firewall ترافیک را بررسی می‌ کند و بر اساس rule های مشخص تصمیم می‌ گیرد که packet یا connection را  accepted یا dropped یا rejected کند یا چه processing دیگری روی آن انجام گیرد. Firewall می‌تواند در جاهای مختلف قرار داشته باشد مثلاً Internet یا Firewall / Router یا Internal Network یا مستقیماً روی خود host مثلاً Network یا Linux Host یا Local Firewall یا Application قرار بگیرد. در حالت دوم معمولاً درباره‌ی host-based IP filtering صحبت می‌ کنیم.

#### 🔹 Linux Firewall Basics

در Linux یکی از ابزارهای مهم firewall  ، ابزار ```iptables``` بوده است. در سیستم‌ های جدید ```nftables``` معماری جدیدتر و ترجیحی kernel firewall framework است و iptables در بسیاری از سیستم‌ها به‌ صورت compatibility layer روی nftables یا در کنار آن دیده می‌ شود. در مدل کلاسیک iptables هم Rules و Chains و Tables داریم.

در table معروف filter سه chain اصلی عبارت‌اند از INPUT و OUTPUT و FORWARD : 

- **اول INPUT** : که packet هایی مقصد نهایی‌ شان خود host است.
- **دوم OUTPUT** : که Packet هایی از خود host خارج می‌ شوند.
- **سوم FORWARD** : که packet هایی که host را به‌ عنوان router عبور می‌ دهند و مقصد نهایی‌ شان خود host نیست.

<img width="100%" height="737" alt="image" src="https://github.com/user-attachments/assets/23f1cf67-554f-4cdb-9d0c-94925e5f4933" />

#### 🔹 Setting Firewall Rules

برای مشاهده‌ ی بعضی rule های iptables می‌توان از```iptables -L``` استفاده کرد. هر chain می‌ تواند یک default policy داشته باشد. دو مقدار بسیار مهم ACCEPT و DROP هستند.  مثلاً اگر policy یک chain روی DROP باشد و هیچ rule در packet را مجاز نکند ، packet در نهایت drop می‌ شود.

با ```iptables -A``` می‌توان rule را به انتهای chain اضافه کرد و با ```iptables -I``` می‌توان rule را در یک موقعیت مشخص insert کرد. ترتیب rule ها بسیار مهم است مثلاً اگر rule اول خیلی عمومی باشد و packet را ACCEPT کند ، ممکن است rule های پایین‌ تر هرگز packet را نبینند. 

باید بین دو مفهوم تمایز قائل شد ، Rule match و Final verdict . برخی target ها verdict نهایی ایجاد می‌ کنند و processing مربوط به chain را خاتمه می‌ دهند. برخی عملیات‌ ها مانند بعضی target های non-terminal الزاماً به معنی پایان کل processing نیستند. بنابراین جمله‌ ی «هر match باعث توقف chain می‌ شود» دقیق نیست.

#### 🔹 Firewall Strategies

یکی از استراتژی‌ های رایج و امن این است که Default = DROP و سپس فقط traffic مورد نیاز به‌ صورت صریح مجاز شود. به‌صورت مفهومی :

```
Incoming packet
       ↓
Is it trusted/required?
   ↙         ↘
 yes          no
  ↓            ↓
ACCEPT       DROP
```

در یک host عادی ممکن است rule های firewall شامل مواردی برای ICMP و Loopback و Established و connections و Related connections و DNS replies و SSH باشند. اما ترتیب دقیق و جزئیات rule ها باید متناسب با نقش واقعی سیستم باشد. مثلاً یک server عمومی ممکن است نیاز داشته باشد TCP 443 را قبول کند ، ولی host دیگری که فقط یک client است ممکن است اصلاً چنین rule را لازم نداشته باشد همچنین باید دقت کرد که rule های مربوط به ESTABLISHED و RELATED باعث می‌ شوند response مربوط به connection هایی که از داخل سیستم شروع شده‌ اند یا connection های related ، بدون نوشتن rule کامل و جداگانه برای هر response مدیریت شوند. Firewall خوب یعنی minimum required access نه اینکه فقط «همه‌چیز را ببندیم».

---

### Ethernet and IP and ARP and NDP

فرض کنید یک packet آماده‌ ی ارسال روی Ethernet است و destination IP آن را می‌ دانیم. هنوز یک سؤال داریم ، برای ارسال local frame چه MAC address باید استفاده شود ؟ در IPv4 این mapping با ARP انجام می‌شود.

#### 🔹 ARP

مخفف Address Resolution Protocol است. اگر host بداند که 192.168.1.20 اما MAC مربوط به این IP را نداند ، می‌ تواند یک ARP request ارسال کند. این request به شکل broadcast روی شبکه‌ ی local فرستاده می‌ شود. Host که آن IP را دارد پاسخ می‌ دهد : IP 192.168.1.20 و MAC aa:bb:cc:dd:ee:ff . این mapping معمولاً مدتی در cache نگهداری می‌ شود و برای مشاهده‌ ی neighbor information در لینوکس استفاده از ```ip neigh``` مفید است.

#### 🔹 IPv6 and NDP

هیچوقت IPv6 از ARP استفاده نمی‌ کند در IPv6 پروتکل NDP یا Neighbor Discovery Protocol این نقش را بر عهده دارد. NDP بخشی از ICMPv6 است و از پیام‌هایی مثل Neighbor Solicitation یا Neighbor Advertisement استفاده می‌ کند.

---

### Wireless Ethernet

از نظر مفهومی Wi-Fi هنوز در دنیای link-layer networking قرار دارد و از MAC address و frame استفاده می‌ کند ، اما medium دیگر کابل Ethernet نیست. در شبکه‌ ی بی‌ سیم با مفاهیمی مانند Frequency و Channel و SSID و Access Point و Authentication و Encryption سر و کار داریم. در Ethernet سیمی، دستگاه معمولاً با یک کابل و link فیزیکی مشخص ارتباط دارد. اما در Wi-Fi باید network discovery و association و authentication و key negotiation نیز مدیریت شوند به همین دلیل wireless networking از Ethernet سیمی پیچیده‌ تر است.

#### 🔹 iw

ابزار iw یکی از ابزارهای اصلی برای مشاهده و مدیریت wireless interface ها در Linux است. مثلاً برای مشاهده‌ ی interface های wireless می‌ توان از command های آن استفاده کرد. برای scan کردن شبکه‌ های اطراف نیز iw قابلیت‌ هایی ارائه می‌ دهد. در سیستم‌های managed ، کاربر ممکن است به‌ جای استفاده‌ ی مستقیم از iw از NetworkManager استفاده کند و NetworkManager در پشت صحنه بخش زیادی از این کارها را انجام دهد.

#### 🔹 Wireless Security

اتصال به یک Wi-Fi فقط به پیدا کردن SSID محدود نیست باید authentication و key management نیز انجام شود. یکی از daemon های مهم در لینوکس ```wpa_supplicant``` است. این daemon برای پیاده‌ سازی و مدیریت فرآیند های مرتبط با WPA و سازوکارهای authentication و encryption استفاده می‌ شود. در محیط‌ های مدرن ممکن است با استاندارد ها و mode های مختلفی مانند WPA2 و WPA3 روبرو شویم.

مدیریت مستقیم ```wpa_supplicant``` می‌ تواند پیچیده باشد ، چون تنظیم authentication ، interface و network profile ها نیازمند جزئیات زیادی است به همین دلیل ابزارهایی مثل ```NetworkManager``` یک لایه‌ ی ساده‌ تر برای کاربر ارائه می‌ دهند. بنابراین در یک سیستم Linux مدرن ممکن است شما فقط روی ```nmcli``` کار کنید ، در حالی که لایه‌ های پایین‌ تر مسئول جزئیات واقعی اتصال Wi-Fi باشند.

---

### Tips

در این بخش با پایه‌های شبکه در لینوکس آشنا شدیم. از packet ها ، لایه‌های شبکه ، IPv4 و IPv6 گرفته تا routing ، default gateway ، Ethernet ، network interface ، DNS و TCP/UDP. این بخش نشان داد که شبکه هم مثل بیشتر بخش‌ های لینوکس، بین kernel و user space تقسیم شده :

 کرنل وظایفی مثل packet forwarding ، routing و تعامل با network interface ها رو انجام میده و ابزارهای user space مثل NetworkManager ، DHCP client ها و Firewall tools مدیریت این زیرساخت رو ساده‌ تر می‌ کنن. فصل بعدی از این زیرساخت عبور می‌ کند و وارد application layer میشه. جایی که برنامه‌ های واقعی از شبکه برای برقراری ارتباط استفاده می‌ کنند.
