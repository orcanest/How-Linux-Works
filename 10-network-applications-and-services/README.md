## وقتی سراغ لایه‌ ی Application می‌ رویم
### 🐧 فصل دهم کتاب How Linux Works

در فصل نهم ، بخش‌ های مختلف شبکه را از دید Linux بررسی کردیم. از IP و routing گرفته تا TCP ، UDP ، DNS ، DHCP ، firewall و network interface ها. در فصل دهم یک مرحله بالاتر می‌ رویم و وارد Application Layer می‌شویم جایی که برنامه‌ های واقعی از زیرساخت شبکه استفاده می‌ کنند. در اینجا دیگر مسئله فقط این نیست که packet به کدام host برسد. حالا باید ببینیم :

- کدام سرویس منتظر connection است ؟
- چه پروتکلی روی آن connection صحبت می‌شود ؟
- برنامه‌ی client چگونه با server ارتباط برقرار می‌کند ؟
- یک server چگونه connection های مختلف را مدیریت می‌کند ؟
- چگونه application از امکانات شبکه‌ ی kernel استفاده می‌ کند ؟

برای مثال ، وقتی در مرورگر یک URL باز می‌ کنیم ، از دید کاربر فقط یک صفحه‌ ی وب دیده می‌ شود؛ اما در زیر این اتفاق ، مجموعه‌ ای از لایه‌ ها با هم کار می‌ کنند :

```
Browser → HTTP → TCP → IP → Ethernet / Wi-Fi → Network
```

فصل دهم بیشتر روی بخش بالایی این زنجیره تمرکز دارد. یعنی service ها ، network server ها ، SSH ، ابزارهای  diagnostic ، RPC ، امنیت شبکه ، socket ها و Unix domain socket ها.

---
📚 Table of Contents

- [The Basics of Services](#the-basics-of-services)
- [A Closer Look](#a-closer-look)
- [Network Servers](#network-servers)
- [Pre systemd Network Connection Servers](#pre-systemd-network-connection-servers)
- [Diagnostic Tools](#diagnostic-tools)
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

### The Basics of Services

برای اینکه مفهوم network service را بهتر درک کنیم ، یکی از بهترین روش‌ ها این است که مستقیماً با یک service ارتباط برقرار کنیم. فرض کنید یک web server روی TCP port 80 در حال گوش دادن است در گذشته می‌توانستیم با ابزار telnet به آن وصل شویم مثلاً ```telnet example.com 80```. در این حالت telnet خودش web server نیست. فقط یک **TCP client ساده** است که connection را برقرار می‌ کند و اجازه می‌ دهد داده را به‌ صورت دستی روی connection ارسال و دریافت کنیم. بعد می‌توانیم یک HTTP request خام بفرستیم. برای HTTP/1.1 باید Host header را نیز مشخص کنیم ```GET / HTTP/1.1``` و ```Host: example.com``` خط خالی پایانی اهمیت دارد ، چون پایان header section را مشخص می‌ کند. ممکن است server پاسخی شبیه این ارسال کند :

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: ...

<html>
...
</html>
```

از این مثال چند نکته‌ ی مهم مشخص می‌ شود. در سمت remote یک process یا مجموعه‌ ای از process ها وجود دارد که یک **TCP listening socket** دارد و منتظر connection های جدید است. در سمت local نیز telnet به‌ عنوان client عمل می‌ کند یعنی : 

```
telnet
   │
   │ TCP connection
   ▼
server
   │
   └── listening on port 80
```

اما نکته‌ ی مهم‌ تر این است که TCP فقط یک stream از bytes را حمل می‌ کند. چیزی درباره‌ ی اینکه این bytes متعلق به HTTP هستند یا چیز دیگری ، ذاتاً در خود TCP وجود ندارد. این application است که تصمیم می‌ گیرد این bytes را به عنوان HTTP تفسیر کند در نتیجه اگر روی port 80 یک سرویس کاملاً متفاوت هم قرار داده شود ، TCP همچنان فقط connection را منتقل می‌ کند و معنی داده را application تعیین می‌ کند. این همان نقطه‌ ای است که مفهوم service از صرفاً «یک port باز» مهم‌ تر می‌ شود. یک service در عمل ترکیبی از چند چیز است : Process / Program و  Listening socket و Protocol و Application logic .

---

### A Closer Look

بعد از دیدن یک HTTP request خام ، ابزارهای مناسب‌ تری برای مشاهده‌ ی ارتباط application و transport داریم. یکی از مهم‌ ترین ابزارها curl است. curl نسبت به telnet برای کار با HTTP بسیار مناسب‌ تر است، چون خودش جزئیات زیادی از protocol را می‌ فهمد. برای مشاهده‌ ی جزئیات exchange می‌ توان از ```-curl --trace-ascii``` استفاده کرد. مثلاً ```curl --trace-ascii - http://example.com``` در چنین حالتی می‌ توانید ببینید چه bytes هایی از طرف client ارسال و چه bytes هایی دریافت می‌ شوند. این موضوع به فهم یک مرز مهم کمک می‌ کند ، TCP برای انتقال byte stream و HTTP برای تفسیر آن byte stream است.

از دید  TCP، چیزی به نام HTTP header و HTTP body وجود ندارد. TCP فقط stream از bytes دارد. این خود HTTP است که structure را تعریف می‌ کند.  مثلاً در HTTP/1.x معمولاً header ها با یک خط خالی از body جدا می‌شوند :

```
Header 1: ...
Header 2: ...
Header 3: ...

Body...
```

یک client مثل curl بر اساس قوانین HTTP می‌ فهمد که خط خالی به معنی پایان بخش header است. بنابراین این تفاوت برای debugging بسیار مهم است : 

```
Network Layer → packet

Transport Layer → TCP stream

Application Layer → HTTP message structure
```

اگر TCP connection کاملاً سالم باشد ، ولی application داده را اشتباه تفسیر کند ، مشکل می‌ تواند در application protocol باشد، نه در Underlying network.

---

### Network Servers

بسیاری از server های شبکه ، در اصل daemon هایی هستند که یک کار اضافه هم انجام می‌ دهند مثلاً روی یک network socket کارشان listen کردن است . نمونه‌ های شناخته‌ شده Apache و nginx و sshd و Postfix . البته معماری هر server می‌ تواند متفاوت باشد ، ولی یک الگوی بسیار رایج این است :

```
                    ┌── Worker
                    │
Listening socket ───┼── Worker
                    │
                    └── Worker
```

یک process یا thread وظیفه‌ ی دریافت connection های جدید را دارد و worker های دیگر actual request ها را پردازش می‌ کنند.

#### 🔹 Why multiple processes or workers?

چون server ممکن است هم‌ زمان connection های زیادی داشته باشد. اگر یک process تنها مسئول همه‌ ی connection ها باشد ممکن است یک request کند request های دیگر را نیز متوقف کند. به همین دلیل معماری‌های مختلفی شکل گرفته‌اند :

- One process per connection
- Thread per connection
- Pre-fork workers
- Thread pools
- Event-driven workers
- Async I/O

برخی server ها worker را فقط وقتی که لازم است ایجاد می‌ کنند. برخی دیگر تعدادی worker را از قبل ایجاد می‌ کنند. ایجاد worker از قبل می‌ تواند هزینه‌ ی ایجاد process جدید در زمان رسیدن هر connection را کاهش دهد. در مقابل ، معماری event-driven می‌ تواند تعداد زیادی connection را با تعداد محدودی worker مدیریت کند. بنابراین هیچ مدل واحدی برای همه‌ ی server ها وجود ندارد.

#### 🔹 Secure Shell

یکی از مهم‌ ترین service های شبکه در Unix/Linux نیز SSH است. SSH جایگزین امنی برای ابزارهای قدیمی مانند telnet و rlogin و rcp شد که برای ارتباطات حساس امنیت کافی نداشتند. SSH چند کار را هم‌ زمان انجام می‌دهد :

- Remote login
- Authentication
- Encrypted communication
- Secure file transfer
- Port forwarding

یکی از مهم‌ ترین ویژگی‌ های SSH این است که ارتباط بین client و server به‌ صورت رمزنگاری‌ شده برقرار می‌ شود. بنابراین اطلاعاتی مانند password و commands و command output و files به شکل plaintext روی شبکه ارسال نمی‌ شوند.

#### 🔹 Public-Key Authentication

سرویس SSH از public-key cryptography نیز برای authentication پشتیبانی می‌ کند و private key نزد client باقی می‌ ماند و نباید در اختیار دیگران قرار گیرد . به‌ صورت مفهومی  :

```
Client
 ├── private key
 └── public key

Server
 └── authorized public key
```

در بسیاری از configuration ها ، public key در فایلی مانند ```ssh/authorized_keys/~.``` در سمت account مقصد قرار می‌ گیرد.

#### 🔹 SSH Tunneling

سرویس SSH فقط برای shell login نیست. می‌ تواند connection های دیگر را نیز از داخل یک کانال SSH عبور دهد. یکی از نمونه‌ های تاریخی آن X11 forwarding است. همچنین SSH از port forwarding های مختلف پشتیبانی می‌ کند که برای دسترسی امن به سرویس‌ های داخلی یا ایجاد tunnel بسیار کاربردی هستند.

#### 🔹 The sshd Server

سمت server توسط daemon مربوط به SSH مدیریت می‌ شود که معمولاً sshd نام دارد. Configuration اصلی آن معمولاً /etc/ssh/sshd_config است برای نمونه ، گزینه‌ هایی مانند PermitRootLogin و X11Forwarding و PasswordAuthentication و PubkeyAuthentication می‌توانند behavior مربوط به SSH را کنترل کنند  البته مجموعه‌ ی option های قابل استفاده به نسخه و configuration سیستم بستگی دارد.

#### 🔹 Host Keys

سرویس SSH server از host key ها برای شناسایی cryptographic هویت server استفاده می‌ کند. معمولاً بیش از یک نوع host key روی سیستم وجود دارد تا الگوریتم‌ های مختلف پشتیبانی شوند. این key ها جفت هستند یعنی Private key و Public key که private host key باید فقط در اختیار خود server و administrator های مجاز باشد. کلیدهای host معمولاً در مسیرهایی مانند /etc/ssh/ قرار دارند. ابزاری که برای ساخت SSH key ها استفاده می‌ شود ```ssh-keygen``` است. 

> 💡 نکته‌ ی امنیتی بسیار مهم ، private key نباید در repository عمومی ، فایل backup ناامن ، کانال ارتباطی ناامن یا اختیار دیگران قرار گیرد. اگر private key لو برود ، امنیت هویتی که آن key نمایندگی می‌ کند می‌ تواند به خطر بیفتد.

#### 🔹 fail2ban

وقتی SSH یا سرویس دیگری را مستقیماً روی اینترنت قرار می‌ دهید ، احتمال مشاهده‌ ی تلاش‌ های نا موفق authentication بسیار زیاد است. این تلاش‌ ها ممکن است از طرف automated scanners یا credential stuffing یا brute-force  attempts باشند. ابزاری مثل ```fail2ban``` برای کمک به مقابله با چنین الگوهایی طراحی شده است. fail2ban به این صورت کار می کند :

```
Log files
   ↓
Detect repeated failures
   ↓
Trigger action
   ↓
Firewall rule / ban
```

مثلاً ممکن است بعد از تعداد مشخصی تلاش login ناموفق از یک IP، آن address برای مدت مشخصی block شود. 

> 💡 نکته‌ ی مهم این است که fail2ban جایگزین authentication امن یا firewall مناسب نیست. 

همچنین block کردن یک IP به‌ تنهایی راه‌ حل کامل brute-force نیست ، چون مهاجم می‌ تواند address های مختلف داشته باشد استفاده از strong authentication و SSH keys و  MFA where available و rate limiting و firewall  policy در کنار fail2ban امنیت بیشتری ایجاد می‌ کند.

#### 🔹 The SSH Client

برای برقراری SSH connection معمولاً ```ssh user@host``` استفاده می‌ کنیم. وقتی client برای اولین بار به server متصل می‌ شود ، SSH host key server را به کاربر نشان می‌ دهد و معمولاً پس از تأیید ، آن را در فایلی مانند ```ssh/known_hosts/~. ``` ذخیره می‌ کند. دفعه‌ی بعد ، client می‌ تواند key دریافت‌ شده را با key قبلی مقایسه کند و اگر key server ناگهان تغییر کرده باشد ، SSH هشدار می‌ دهد. این تغییر می‌تواند دلایل مختلفی داشته باشد :
```
- Server reinstalled
- Host keys regenerated
- DNS points to another machine
- Legitimate infrastructure change
- Potential man-in-the-middle attack
```

بنابراین نباید صرفاً warning را نادیده گرفت. باید مشخص شود که آیا تغییر واقعاً از طرف administrator یا تغییر زیرساخت بوده است یا خیر.

#### 🔹 File Transfer

برای انتقال امن فایل ، SSH ecosystem ابزارهایی مثل scp و sftp را ارائه می‌ دهد. این ابزارها جایگزین امن‌ تری برای  ابزارهای قدیمی مانند rcp و  ftp هستند هرچند scp و sftp از نظر protocol و behavior یکسان نیستند. SFTP یک protocol مخصوص file transfer روی SSH است، در حالی که scp در اصل یک روش copy مبتنی بر SSH است.

---

### Pre systemd Network Connection Servers

قبل از اینکه service manager هایی مانند systemd به شکل امروزی رایج  شوند ، Unix های زیادی از مفهومی به نام ```inetd``` استفاده می‌ کردند. ```inetd``` را می‌ توان یک **super-server** در نظر گرفت. به جای اینکه هر service کوچک دائماً یک process جداگانه داشته باشد که روی port خودش listen کند، ```inetd``` می‌ توانست socket های مختلف را مدیریت کند مثلاً :

```
Network
   ↓
inetd
   ↓
Service A
Service B
Service C
```

وقتی connection روی یک port مشخص می‌ رسید،  inetd process مربوط به آن service را اجرا می‌ کرد و connection را در اختیار آن قرار می‌ داد. این روش یک مزیت مهم داشت لازم نبود هر سرویس کم‌ مصرف دائماً در حال اجرا باشد اما هزینه‌ها و محدودیت‌هایی نیز داشت. xinetd نسخه‌ ی توسعه‌ یافته‌ تر همین ایده بود و قابلیت‌ های بیشتری برای policy و کنترل connection ارائه می‌ کرد. در سیستم‌ های مدرن ، **systemd socket activation** می‌ تواند ایده‌ ای مشابه را به شکلی یکپارچه‌ تر پیاده کند.

در systemd می‌ توان ```socket.``` را برای listening socket تعریف کرد و یک ```service.``` را هنگام نیاز فعال کرد. به‌صورت مفهومی :

```
Client
  ↓
systemd socket
  ↓
service activation
  ↓
server process
```

این ایده نمونه‌ ی خوبی از این است که چگونه یک قابلیت قدیمی Unix در معماری‌ های جدید نیز به شکل دیگری ادامه پیدا کرده است.

---

### Diagnostic Tools

وقتی یک application network  درست کار نمی‌ کند، باید بدانیم مشکل دقیقاً کجاست. ممکن است :

```
- Process not running
- Port not listening
- Firewall blocking
- DNS incorrect
- TCP connection failing
- Application protocol failing
```

باشد. چند ابزار ساده اما بسیار قدرتمند برای چنین troubleshooting هایی وجود دارند مانند  ```lsof``` و ```tcpdump``` و ```netcat``` و ```nmap```.

#### 🔹 lsof













