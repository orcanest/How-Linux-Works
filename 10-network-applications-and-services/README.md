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
- [Remote Procedure Calls](#remote-procedure-calls)
- [Looking Forward](#looking-forward)
- [Network Sockets](#network-sockets)
- [Unix Domain Sockets](#unix-domain-sockets)
- [Tips](#tips)

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

همان‌ طور که در فصل هشتم دیدیم ، lsof می‌ تواند resource های باز process ها را نشان دهد. برای network نیز می‌توان از ```lsof -i```استفاده کرد. این command می‌ تواند socket های شبکه‌ ای مرتبط با process ها را نشان دهد. مثلاً ```lsof -iTCP یا lsof -iTCP:443 ``` می‌ تواند خروجی را به connection های TCP یا یک port خاص محدود کند. این ابزار برای سؤال‌ هایی مانند ، چه process ای به یک connection خاص متصل است؟ یا این socket مربوط به کدام PID است؟ بسیار مفید است. برای مثال ممکن است ببینید :

```nginx   1234   root   ... TCP *:443 (LISTEN)```

و فوراً متوجه شوید که port 443 توسط nginx در اختیار گرفته شده است.

#### 🔹 tcpdump

گاهی دانستن اینکه یک process روی port گوش می‌ دهد کافی نیست. باید ببینیم واقعاً چه packet هایی روی network حرکت می‌ کنند. ابزار اصلی برای این کار```tcpdump``` است. tcpdump می‌ تواند packet هایی را که یک interface می‌بیند capture کند و اطلاعات مختلفی از آن‌ها را نمایش دهد. مثلاً ```tcpdump``` برای capture اولیه یا ```tcpdump -i eth0``` برای مشخص کردن interface.

#### 🔹 Filtering

یکی از ویژگی‌ های مهم tcpdump ، filter expression ها هستند. مثلاً ```tcpdump tcp``` فقط traffic مربوط به TCP را هدف می‌ گیرد یا ```tcpdump udp``` برای UDP همچنین ```tcpdump port 80``` برای traffic مربوط به port 80 می‌توان filter های پیچیده‌ تر نیز ساخت برای مثال ```tcpdump 'tcp port 443'``` یا ```tcpdump 'host 192.168.1.10'``` در troubleshooting باید توجه داشت که tcpdump traffic را در نقطه‌ ای capture می‌ کند که interface مورد نظر آن را می‌ بیند. بنابراین capture نکردن یک packet لزوماً به معنی این نیست که packet هیچ‌ جا وجود نداشته  ممکن است از interface دیگری عبور کرده باشد ،  توسط offload ها یا filtering های مختلف به شکل متفاوتی دیده شود یا اصلاً به host نرسیده باشد.

#### 🔹 netcat

یکی از ابزارهای بسیار ساده و در عین حال انعطاف‌ پذیر در شبکه ```netcat``` یا ```nc``` است. netcat می‌ تواند هم client باشد و هم server. مثلاً برای اتصال به یک TCP service دستور ```nc host.example.com 80``` و بعد می‌توانید یک HTTP request ارسال کنید. از طرف دیگر می‌توان یک listener ساده ایجاد کرد.

تفاوت مهم netcat با telnet این است که netcat فقط به یک کاربرد خاص یا TCP محدود نیست و می‌ تواند برای TCP و UDP و سناریوهای مختلف testing استفاده شود. در troubleshooting می‌ توان با nc بررسی کرد که آیا یک port reachable است ؟  یا connection TCP برقرار می‌شود؟  ، البته موفق شدن TCP connection به این معنی نیست که application protocol نیز درست کار می‌ کند.

#### 🔹 Port Scanning

برای بررسی مجموعه‌ ای از port ها و سرویس‌ های قابل دسترسی ، ابزار معروف ```Nmap``` است. Nmap می‌ تواند برای شناسایی وضعیت port ها و در بعضی mode ها اطلاعات بیشتری درباره‌ ی service های فعال جمع‌ آوری کند. یک scan ساده ممکن است مشخص کند که port هایی مانند 22 ،  80 و 443 قابل دسترسی هستند.

از Nmap می‌ توان برای Network inventory و Troubleshooting و Security auditing و Service discovery استفاده کرد. اما یک اصل مهم وجود دارد ، اسکن کردن سیستم‌ ها یا شبکه‌ هایی که مالک آن‌ ها نیستید یا اجازه‌ ی بررسی آن‌ ها را ندارید ، می‌ تواند غیرقانونی باشد بنابراین Nmap باید در شبکه‌ های خودتان ، آزمایشگاه شخصی یا محیط‌ هایی که مجوز صریح دارید استفاده شود.

---

### Remote Procedure Calls

مخفف Remote Procedure Call مفهومی است که به برنامه اجازه می‌ دهد یک operation را روی سیستم دیگری اجرا کند، به شکلی که از دید programmer تا حدی شبیه فراخوانی یک local function باشد. 

```
Application A
    ↓
call function()
    ↓
RPC layer
    ↓
Network
    ↓
RPC server
    ↓
Execute operation
```

هدف RPC این است که جزئیات ارتباط شبکه ای تا حدی از application logic جدا شود. البته در عمل ، RPC واقعاً یک local function call نیست. latency ، failure ، serialization ، authentication و network errors همگی وجود دارند.

#### 🔹  rpcbind

در سیستم‌ های Unix سرویسی به نام ```rpcbind``` نقش مهمی در پیدا کردن سرویس‌ های RPC داشت. RPC service ها معمولاً یک program number دارند. rpcbind این شناسه را با یک transport  و port mapping مرتبط می‌ کند. client می‌ تواند از rpcbind بپرسد که برنامه‌ ی RPC مورد نظر روی چه port قرار دارد این مدل مخصوصاً در  سرویس‌ هایی مانند ```NFS``` اهمیت تاریخی و عملی داشته است. البته دنیای modern RPC بسیار متنوع‌ تر از مدل ONC RPC است و framework هایی مثل gRPC و JSON-RPC و XML-RPC هم وجود دارند.

اما مفهوم پایه همچنان همان است application یک operation را برای remote system درخواست می‌ کند و زیرساخت RPC جزئیات ارتباط و encoding/transport را مدیریت می‌ کند.

#### 🔹  Network Security

وقتی یک service را روی شبکه قرار می‌ دهیم ، دیگر مسئله فقط «کار کردن» نیست. هر port باز و هر service فعال ، بخشی از attack surface سیستم است به همین دلیل چند اصل ساده اهمیت زیادی دارند.

#### 🔹 Minimum number of services

هر سرویسی که لازم ندارید بهتر است اجرا نشود. مثلاً اگر machine فقط برای SSH استفاده می‌ شود، نیازی نیست چندین daemon شبکه‌ ای غیرضروری روی آن فعال باشند. این رویکرد با اصل Minimize attack surface مطابقت دارد.

#### 🔹 Firewall

فقط service هایی را که واقعاً لازم هستند در معرض شبکه قرار دهید یعنی اگر application فقط از localhost استفاده می‌ کند ، نباید الزاماً روی همه‌ی interface ها listen کند برای مثال 127.0.0.1:8080 با 0.0.0.0:8080 از نظر exposure کاملاً متفاوت است.

#### 🔹 Update

سرویس‌ هایی که روی شبکه در معرض attack هستند باید مرتب به‌ روزرسانی شوند. این موضوع مخصوصاً برای vulnerability های شناخته‌ شده اهمیت دارد.

#### 🔹 Supported versions

استفاده از release های دارای پشتیبانی بلند مدت می‌ تواند مدیریت امنیت را ساده‌ تر کند ، چون patch های امنیتی برای مدت مشخصی ارائه می‌ شوند.

#### 🔹 unnecessary accounts

ایجاد user account بدون دلیل ، attack surface را افزایش می‌ دهد. اگر یک account دیگر استفاده نمی‌ شود ، بهتر است حذف یا غیرفعال شود.

#### 🔹 Three general categories of attacks

از دید ساده‌ ی این فصل می‌ توان چند هدف اصلی حملات را در نظر گرفت.

#### 🔹 Full Compromise

هدف مهاجم به دست آوردن کنترل گسترده روی سیستم است بدترین حالت می‌تواند به دست آوردن دسترسی ```root``` یا سطحی معادل آن باشد. در این وضعیت مهاجم می‌ تواند بخش بزرگی از سیستم را کنترل کند.

#### 🔹 Denial of Service

 در DoS هدف لزوماً کنترل سیستم نیست. هدف می‌ تواند این باشد که Service unavailable یا System overloaded یا Resource exhausted شود. مثلاً مهاجم ممکن است CPU ، memory ، connection slots یا bandwidth را مصرف کند.

#### 🔹 Malware

دسته‌ی بزرگی از software های مخرب است که می‌ توانند برای Data theft و Remote control و Persistence و Destruction و Cryptomining و Spying استفاده شوند.

#### 🔹 Typical Vulnerabilities

دو دسته‌ ی مهم از مشکلات امنیتی که در چنین محیط‌ هایی باید به آن‌ ها توجه کرد عبارت‌ اند از **ضعف در خود برنامه و ارسال یا نگهداری ناامن اطلاعات**.

#### 🔹 Direct Attacks

در این حالت مهاجم از یک ضعف مستقیم در برنامه یا سرویس سوء استفاده می‌ کند. یکی از مثال‌ های کلاسیک buffer overflow است. اگر یک برنامه بدون بررسی مناسب مقدار داده را در buffer قرار دهد ، ممکن است مهاجم بتواند از memory corruption ایجاد شده سوء استفاده کند. اما modern operating systems و compiler ها چندین mechanism دفاعی دارند.

یکی از مهم‌ ترین آن‌ها ASLR یا Address Space Layout Randomization است. ASLR باعث می‌ شود location برخی بخش‌ های مهم memory در اجرا ها قابل‌ پیش‌ بینی نباشد و در نتیجه برخی exploit ها دشوارتر شوند. البته ASLR به‌ تنهایی یک buffer overflow را اصلاح نمی‌ کند فقط یکی از لایه‌ های دفاعی است. سازوکارهای دیگری مانند NX / DEP و Stack canaries و PIE و Control-flow defenses و Memory-safe languages نیز می‌ توانند نقش مهمی داشته باشند.

#### 🔹 Cleartext Data

دسته‌ ی دیگر، ارسال اطلاعات حساس بدون رمزنگاری است. اگر protocol ارتباطی plaintext باشد ، افراد یا دستگاه‌ هایی که در مسیر قرار دارند ممکن است بتوانند اطلاعات را مشاهده کنند. نمونه‌های کلاسیک telnet و ftp هستند. این ابزارها برای داده‌ های حساس مناسب نیستند ، چون حفاظت رمزنگاری‌ شده‌ ی مدرن SSH یا TLS را ندارند. در مقابل ، باید از protocol هایی استفاده کرد که confidentiality و integrity را فراهم می‌ کنند. برای مثال : SSH و HTTPS و TLS در سناریوهای مناسب.

#### 🔹 Security Resources

امنیت شبکه حوزه‌ ی بسیار گسترده‌ ای است و تنها با خواندن یک فصل نمی‌ توان به تسلط کامل رسید. برای مطالعه‌ ی بیشتر، منابع و سازمان‌ های امنیتی مختلفی وجود دارند. از جمله SANS Institute و CERT این منابع می‌ توانند برای مطالعه‌ ی Vulnerabilities و Incident response و Security practices و Network security مفید باشند.

یکی از موضوعات مهم برای ادامه‌ ی مسیرTLS است. TLS سازوکاری برای ایجاد ارتباط امن روی شبکه است. باید توجه کرد که SSL نام خانواده‌ ی قدیمی‌ تر همین technology است و نسخه‌ های قدیمی SSL امروزه منسوخ و ناامن محسوب می‌ شوند. در عمل ، وقتی امروز از secure HTTPS صحبت می‌ کنیم ، اساساً درباره‌ ی TLS صحبت می‌کنیم ، نه SSL قدیمی.

---

### Looking Forward

بعد از یادگیری مفاهیم اولیه‌ ی network service ، بهترین راه ادامه دادن این است که با server های واقعی کار کنیم. مثلاً Apache یا nginx یا Postfix هرکدام نمونه‌ ی خوبی برای دیدن یک application-layer service واقعی هستند. با نصب یک server می‌توانید زنجیره‌ ی مفاهیم فصل‌ های قبل را هم‌ زمان ببینید :

```
Application
    ↓
Socket
    ↓
TCP
    ↓
IP
    ↓
Network interface
    ↓
Kernel
```

مثلاً در مورد یک web server می‌ توانید بررسی کنید ```ss -lntp``` کدام process روی port گوش می‌ دهد. بعد ```lsof -i``` را بررسی کنید سپس ```tcpdump``` را اجرا کنید تا packet های واقعی را ببینید و در نهایت ```curl``` را برای صحبت با application اجرا کنید. این روش باعث می‌ شود چند فصل مختلف کتاب به یکدیگر متصل شوند.

#### 🔹 The Importance of Firewalls and NAT in Testing

یکی از بهترین روش‌ های مطالعه‌ی این موضوع ، قرار دادن server در محیطی است که access آن تحت کنترل شما باشد. مثلاً : 

```
Internet
   ↓
Firewall / NAT
   ↓
Lab machine
   ↓
Apache / nginx / sshd
```


این ساختار اجازه می‌ دهد رفتار واقعی service ها را بررسی کنید ، بدون اینکه مستقیماً یک machine آزمایشی را بدون حفاظت مناسب در معرض اینترنت قرار دهید.

---

### Network Sockets

از این بخش به بعد ، کتاب وارد سطح فنی‌ تری می‌ شود که برای programmer ها اهمیت بیشتری دارد. تا اینجا گفتیم application از TCP یا UDP استفاده می‌ کند. اما برنامه دقیقاً چطور با این protocol ها کار می‌ کند ؟ جواب socket است. Socket یک abstraction و programming interface است که application از طریق آن با communication subsystem سیستم‌عامل کار می‌کند.

```
Application
    ↓
Socket API
    ↓
Kernel
    ↓
TCP / UDP / IP
    ↓
Network Device
```

یعنی application معمولاً لازم نیست packet های Ethernet را خودش بسازد. در عوض با API های socket کار می‌ کند.

#### 🔹 Socket Types

دو نوع مهم socket در networking هم Stream socket و Datagram socket هستند. Stream socket معمولاً برای TCP استفاده می‌ شود و TCP byte stream ارائه می‌ کند و Application می‌ تواند داده را بنویسد و دریافت کند . همچنین Datagram socket معمولاً برای UDP استفاده می‌ شود. در این مدل message ها به‌ صورت datagram های جداگانه ارسال می‌ شوند. در Unix/Linux خانواده‌ ی دیگری از socket ها نیز وجود دارد که کمی جلوتر درباره‌ ی آن‌ها صحبت می‌ کنیم Unix domain sockets .

#### 🔹 Server Socket Lifecycle

یک TCP server معمولاً lifecycle مشخصی دارد. ابتدا application یک socket ایجاد می‌ کند بعد socket را به address و port خاصی bind می‌ کند مثلاً ```192.168.1.10:8080``` سپس آن را وارد وضعیت listening می‌ کند. وقتی یک client connection برقرار می‌ کند ، server از()accept استفاده می‌ کند.

اینجا یک نکته‌ ی بسیار مهم وجود دارد ، socket که برای listen استفاده می‌ شود با socket که برای یک connection مشخص به client استفاده می‌شود یکی نیست : 


```
Listening Socket
       │
       ├── accept() → Connection Socket A
       ├── accept() → Connection Socket B
       └── accept() → Connection Socket C
```

همچنان Listening socket وظیفه‌ ی گرفتن connection های جدید را دارد. هر connection یک socket مخصوص خودش دریافت می‌ کند پس ```listen socket``` برای پذیرش connection های جدید است و ```connected socket``` برای ارتباط واقعی با یک client مشخص.

#### 🔹 Handling Connections with fork()

در معماری‌ های قدیمی‌ تر، server می‌توانست بعد از()accept یک Child process ایجاد کند :

```
Parent
  │
  ├── accept()
  │
  ├── fork()
  │      ↓
  │   Child handles client
  │
  └── continues accepting
```

در این مدل Parent وظیفه connection management و Child وظیفه client request را بر عهده می‌گیرد. با افزایش تعداد connection ها ، process های بیشتری ساخته می‌ شوند. اما ایجاد process برای هر connection همیشه بهترین روش نیست. به همین دلیل معماری‌ های دیگر نیز وجود دارند مانند Threads و Worker pools و Event loops و Async I/O و Multiplexing . مثلاً سروری مانند nginx می‌ تواند تعداد زیادی connection را با معماری event-driven مدیریت کند.

#### 🔹 File Descriptor and Socket

از دید برنامه ، socket معمولاً مانند یک **file descriptor** در اختیار process قرار می‌ گیرد. این یک ایده‌ ی بسیار مهم Unix است یعنی همان abstraction عمومی که برای file و pipe و terminal و socket داریم ، در بسیاری از موارد از descriptor ها استفاده می‌ کند مثلاً process ممکن است : 

```
fd 0 → stdin
fd 1 → stdout
fd 2 → stderr
fd 3 → socket
```

داشته باشد. پس وقتی برنامه روی socket عملیات read/write انجام می‌ دهد ، در نهایت از همان interface عمومی descriptorها استفاده می‌ کند.

<img width="100%" height="478" alt="image" src="https://github.com/user-attachments/assets/e273ec92-07ed-405a-8cef-35302b3053b2" />

---

### Unix Domain Sockets

شبکه تنها زمانی نیست که دو process باید با یکدیگر ارتباط برقرار کنند گاهی هر دو process روی یک machine قرار دارند مثلاً Process A به Process B و بالعکس . در این وضعیت اگر فقط نیاز به ارتباط local داشته باشیم ، لازم نیست داده را از network interface واقعی عبور دهیم. برای چنین کاربرد هایی Unix domain socket بسیار مناسب است. Unix domain socket از نظر programming model بسیار شبیه network socket است ، اما endpoint ها به جای IP/port از local namespace استفاده می‌ کنند. یک نمونه‌ی معمول ```/tmp/my-service.sock``` یا socket که در مسیرهای runtime مانند run/ قرار گرفته باشد.

#### 🔹 Network Socket vs Unix Domain Socket

به شکل ساده :

```
Network socket
→ communication through network stack
→ IP address / port

vs

Unix domain socket
→ local IPC
→ filesystem path یا abstract namespace
```

این بدان معنی نیست که Unix domain socket هیچ‌ وقت وارد kernel networking نمی‌شود اتفاقاً communication همچنان توسط kernel مدیریت می‌ شود. منظور این است که ارتباط به network protocol های معمول IP/TCP/UDP وابسته نیست و در همان host انجام می‌ شود.

#### 🔹 Advantages of Unix Domain Sockets

یکی از مزیت‌ های Unix domain socket این است که در حالت pathname-based می‌ توان از permission های filesystem برای کنترل دسترسی استفاده کرد و ownership مناسب روی socket وجود داشته باشد در نتیجه دسترسی process ها به socket می‌ تواند با مدل permission های Unix هماهنگ شود. علاوه بر آن، Unix domain sockets اطلاعاتی مثل peer credentials را نیز در اختیار سازوکارهای مناسب قرار می‌ دهند که می‌ تواند برای authorization محلی مفید باشد.

#### 🔹 Performance

چون ارتباط local است ، نیازی به عبور واقعی از Ethernet یا Wi-Fi یا Router یا IP network نیست. در نتیجه برای بسیاری از ارتباطات  local مثل  Unix domain socket می‌ تواند  سربار کمتری نسبت به ارتباط network داشته باشد. اما نباید این را به یک قانون مطلق تبدیل کرد. performance واقعی به application ، message size ، kernel behavior و implementation بستگی دارد.

نمونه‌ های استفاده Unix domain sockets در سرویس‌ های مختلف بسیار رایج هستند. مثلاً در بعضی deployment ها MySQL / MariaDB می‌ توانند برای  local connection از Unix socket استفاده کنند. همچنین سرویس‌ هایی مانند D-Bus از مکانیزم‌ های ارتباط local بر پایه‌ ی Unix domain sockets استفاده می‌کنند. در چنین طراحی‌ ای به جای ```127.0.0.1:3306``` ممکن است application از socket path local استفاده کند. این مدل هم performance و هم مدل امنیتی متفاوتی ارائه می‌ دهد.

#### 🔹 View Unix Domain Sockets

برای دیدن Unix domain socket ها می‌ توان از ```lsof -U``` استفاده کرد همچنین ابزارهایی مانند ```ss -x``` برای مشاهده‌ ی Unix sockets مفید هستند. این ابزارها کمک می‌کنند بفهمیم چه process ای socket را ایجاد کرده ؟ یا socket در کجا قرار دارد ؟ یا چه process هایی از آن استفاده می‌کنند ؟ در نتیجه مفهوم socket دوباره به فصل هشتم و lsof برمی‌گردد.

---

### Tips 

در این فصل وارد لایه‌ ی Application شدیم و دیدیم سرویس‌ هایی مثل HTTP و SSH چگونه روی زیرساخت شبکه کار می‌کنند. سپس ابزارهایی مثل curl، lsof، tcpdump، netcat و nmap را برای بررسی و عیب‌ یابی سرویس‌ های شبکه دیدیم.

در ادامه با معماری server ها ، inetd و xinetd ، مفاهیم پایه‌ی امنیت شبکه و در نهایت socket آشنا شدیم. همچنین دیدیم که Unix domain socket چگونه برای ارتباط بین process های روی یک ماشین استفاده می‌ شود.
