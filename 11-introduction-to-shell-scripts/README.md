## از خط فرمان به اسکریپت
### 🐧 فصل یازدهم کتاب How Linux Works

تا اینجا بیشتر دستورها را مستقیماً در Shell اجرا می‌ کردیم. اما یکی از مهم‌ ترین قابلیت‌ های محیط Unix/Linux این است که تقریباً هر کاری را که بتوان در Shell انجام داد ، می‌ توان در یک فایل قرار داد تا Shell آن را به‌ صورت خودکار اجرا کند. به این فایل‌ ها **Shell Script** گفته می‌ شود.

فصل یازدهم کتاب How Linux Works با همین هدف وارد Shell scripting می‌ شود و از مباحث پایه‌ ای مثل ساختار یک script ، quoting و متغیرهای ویژه شروع می‌ کند و بعد به شرط‌ ها ، loop ها ، command substitution ، فایل‌ های موقت ، here document ، ابزارهایی مثل awk و sed ، subshell ، sourcing و گرفتن ورودی از کاربر می‌ رسد. 

نکته‌ ی مهم فصل این است که Shell فقط جایی برای تایپ چند command نیست بلکه خودش یک محیط برنامه‌ نویسی است. با این حال ، این زبان برای همه‌ ی مسائل مناسب نیست و یکی از مهارت‌ های مهم ، تشخیص مرز مناسب استفاده از Shell است.

---
📚 Table of Contents

- [Shell Script Basics](#shell-script-basics)
- [Quoting and Literals](#quoting-and-literals)
- [Special Variables](#special-variables)
- [Exit Codes](#exit-codes)
- [Conditionals](#conditionals)
- [Loops](#loops)
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

### Shell Script Basics

یک Shell Script در ساده‌ ترین تعریف ، مجموعه‌ ای از command هاست که در یک فایل نوشته شده‌ اند. همان‌ طور که در terminal می‌ توانیم بنویسیم : 

```
echo hello
pwd
ls
```

می‌توانیم همین دستورها را داخل فایلی مثل ```script.sh``` قرار دهیم. بعد Shell فایل را می‌ خواند و command های آن را تقریباً همان‌ طور که انگار از terminal وارد شده‌ اند ، اجرا می‌ کند. یک Bourne Shell Script معمولاً با خطی شبیه این شروع می‌ شود ```bin/sh/!#```این خط shebang نام دارد. قسمت ```!#``` به سیستم می‌ گوید که فایل executable باید توسط interpreter مشخص‌شده اجرا شود. در این مثال interpreter هم bin/sh/ است در نتیجه اگر فایل executable باشد به صورت ```script.sh/.``` سیستم هنگام اجرای آن ، interpreter مشخص‌ شده را برای پردازش script به کار می‌ گیرد. Shebang محدود به Shell نیست برای مثال ممکن است script پایتون با چیزی شبیه ```usr/bin/python3/!# شروع شود. به همین دلیل shebang در واقع یک مکانیزم عمومی برای مشخص کردن interpreter یک فایل اجرایی متنی است.

#### 🔹 Executing the script

فایل script برای اجرای مستقیم معمولاً باید permission مناسب داشته باشد مثلاً ```chmod +x script.sh``` و سپس ```script.sh/.``` البته اگر interpreter را مستقیماً call کنید ```sh script.sh```  وجود execute permission برای خود فایل دیگر شرط اجرای مستقیم نیست. این تفاوت مهم است ، برای اجرای script.sh/. سیستم باید اجازه‌ ی execute فایل را داشته باشد و برای ```sh script.sh```  فایل به عنوان input به sh داده می‌ شود همچنین interpreter و permission های آن interpreter نیز در نهایت اهمیت دارند.

#### 🔹 Why /bin/sh ?

عبارت sh در Unix یک نام تاریخی و استاندارد برای Bourne-style shell است. در بسیاری از Linux distribution های مدرن ، bin/sh/ ممکن است symlink به shell دیگری باشد ، مثلاً dash یا bash در mode سازگاری sh. بنابراین script که با ```bin/sh/!#``` شروع می‌ شود باید تا حد ممکن فقط از syntax و قابلیت‌ های قابل اتکا در محیط sh استفاده کند. اگر script عمداً به قابلیت‌ های اختصاصی Bash وابسته است ، بهتر است interpreter را صریحاً Bash قرار دهد ```bin/bash/#!``` در غیر این صورت ممکن است script روی سیستمی که bin/sh/ آن Bash نیست ، رفتار متفاوتی داشته باشد.

#### 🔹 Limitations of Shell Scripts

قابلیت Shell scripting برای automation و ترکیب command های Unix بسیار قدرتمند است ، اما این قدرت به معنی مناسب بودن Shell برای هر کاری نیست. نقطه‌ی قوت Shell این است که command های مختلف را به هم متصل کنیم و از امکاناتی مانند pipes و redirection و environment variables و exit codes و loops و conditionals استفاده کنیم. اما اگر script به سمت کارهایی مانند محاسبات عددی سنگین یا پردازش پیچیده‌ ی رشته‌ ها یا ساختمان‌ های داده‌ ی پیچیده یا منطق برنامه‌ نویسی گسترده یا ساختارهای بزرگ و چند لایه برود ، Shell به‌ سرعت ناخوشایند می‌ شود. مشکل فقط performance نیست. خوانایی ، maintainability و احتمال bug نیز مطرح هستند. مثلاً Shell برای این نوع کار ``` find /var/log -name '*.log' | gzip```  فوق‌ العاده است اما اگر قرار باشد الگوریتمی با state پیچیده ، parsing فراوان و ده‌ ها حالت مختلف بنویسیم ، زبان‌ هایی مثل Python یا Perl انتخاب طبیعی‌ تری هستند. awk نیز در بعضی از این سناریوها انتخاب بسیار خوبی است، مخصوصاً برای پردازش داده‌ های متنی و ساختاریافته. 

> 💡 پیام اصلی این بخش این است که Shell را به خاطر قدرتش به یک زبان هم ه‌کاره تبدیل نکنید. هرچه script بزرگ‌ تر و پیچیده‌ تر شود ، نقطه‌ای می‌ رسد که انتخاب یک زبان مناسب‌ تر هزینه‌ ی نگهداری را بسیار پایین‌ تر می‌آورد.

---

### Quoting and Literals

یکی از مهم‌ ترین و در عین حال گیج‌ کنند ه‌ ترین قسمت‌ های Shell scripting هم **quoting** است. Shell قبل از اینکه argument ها را به command بدهد ، آن‌ ها را در چند مرحله تفسیر می‌ کند. کارهایی مانند variable expansion و command substitution و globbing و word splitting و quote removal ، می‌ توانند روی متن وارد شده اثر بگذارند. به همین دلیل اگر بخواهید متنی را همان‌طور که هست به یک command بدهید ، باید بدانید چه چیزهایی را Shell تفسیر می‌کند. مثلاً این ```echo *.txt``` به این معنی نیست که echo دقیقاً متن *.txt را دریافت می‌ کند. اگر در directory فایل‌ هایی مانند a.txt و b.txt و c.txt وجود داشته باشد ، shell ممکن است *.txt را به این فهرست تبدیل کند. این همان **pathname expansion** یا globbing است. اگر بخواهید خود رشته‌ ی *.txt را به command بدهید ، quoting به کار می‌ آید مثلا ```'echo '*.txt``` . پس quoting فقط برای زیبایی syntax نیست بلکه بخشی از کنترل نحوه‌ ی تفسیر command توسط Shell است.

#### 🔹 Literals

یعنی مقداری که می‌ خواهیم Shell آن را تا حد ممکن به همان شکل به command بدهد. در Shell تعداد زیادی character دارای معنی ویژه هستند مانند ``` * ? $ ; & | < > ( ) ``` و بسیاری موارد دیگر مثلاً ```* echo``` با ``` '*' echo ``` رفتار یکسانی ندارد. در حالت اول * می‌ تواند توسط globbing گسترش پیدا کند  و در حالت دوم * literal باقی می‌ ماند. همین مسئله برای regex ها نیز مهم است. مثلاً اگر regex شما شامل character هایی باشد که Shell آن‌ ها را ویژه می‌ داند ، باید مشخص کنید کدام لایه قرار است آن character ها را تفسیر کند. این اشتباه بسیار رایج است که کاربر regex را درست می‌ نویسد ولی Shell قبل از رسیدن regex به برنامه ، بخشی از آن را تغییر می‌ دهد.

#### 🔹 Single Quotes

ساده‌ ترین و قوی‌ ترین نوع quoting معمولی در Shell است مثلاً  :

```echo '$HOME' → $HOME```  

خروجی HOME$ خواهد بود ، نه مقدار متغیر. در داخل single quotes نیز variable expansion و command substitution انجام نمی‌ شوند. همچنین بسیاری از character های ویژه‌ ی Shell literal در نظر گرفته می‌ شوند. مثلاً :

```echo '* $HOME $(date)'```

کل رشته را تقریباً همان‌ طور که نوشته شده دریافت می‌ کند.

#### 🔹 Single quote limitation

یک single quote نمی‌ تواند مستقیماً داخل single-quoted string قرار بگیرد. مثلاً 'echo 'it's این syntax معتبر نیست زیرا Shell اولین ' را شروع string و ' بعد از it را پایان آن در نظر می‌ گیرد برای این کار باید quoting را ترکیب کنیم.

#### 🔹 Double Quotes

همچنین Double quotes نیز بسیاری از تفسیرهای Shell را مهار می‌ کنند، اما همه‌ چیز را متوقف نمی‌ کنند. مثلاً :

```echo "$HOME"```  

متغیر HOME$ همچنان expand می‌ شود. همچنین command substitution کار می‌ کند :

```echo "$(date)"```

اما globbing مانند حالت بدون quote انجام نمی‌ شود. این ویژگی باعث می‌شود double quotes یکی از مهم‌ ترین ابزارهای Shell scripting باشد مثلاً :

```
name="John Doe"
echo "$name"
```

در این حالت کل مقدار John Doe به‌ عنوان یک argument واحد به echo داده می‌ شود. در مقابل echo $name ممکن است به دلیل word splitting به دو argument تبدیل شود. به همین دلیل یک قاعده‌ ی بسیار مهم در Shell scripting این است ، **وقتی مطمئن نیستید splitting مورد نیاز است ، expansion متغیر را quote کنید**. این موضوع مخصوصاً برای path هایی که ممکن است space داشته باشند بسیار مهم است.

#### 🔹 Literal Single Quotes

اگر واقعاً بخواهیم یک ' را در متنی که با single quotes نوشته شده وارد کنیم ، باید quoting را ببندیم ، character را در یک context دیگر اضافه کنیم و دوباره quoting را ادامه دهیم مثلاً :

```echo 'It'\''s fine'```

در این عبارت 'It' بخش اول string است بعد '\ و single quote literal را وارد می‌ کند سپس quoting دوباره ادامه پیدا می‌ کند. راه دیگر استفاده از double quotes برای آن بخش است :

```echo "It's fine"```

البته انتخاب روش مناسب به context بستگی دارد. نکته‌ ی مهم این است که single quote داخل single-quoted string راه ساده‌ ی مستقیمی ندارد.

---

### Special Variables

در Shell چند متغیر ویژه وجود دارد که در script ها بسیار مهم‌ اند. این متغیرها از قبل توسط Shell مدیریت می‌ شوند و اطلاعاتی درباره‌ی script ، argument های ورودی، process و status اجرای command در اختیار script قرار می‌ دهند. هرکدام از متغیر ها معنی مخصوص خودشان را دارند و مهم‌ ترین‌ ها عبارت اند از : 

``` $1 $2 ... $# $@ $0 $$ $?```

#### 🔹 Individual Arguments: $1, $2, and So On

وقتی script را با argument اجرا می‌ کنیم ```script.sh one two three/. شل این argument ها را به script می‌ دهد. در script :

``` $1 → one , $2 → two , $3 → three ```

و به همین ترتیب این یکی از مهم‌ ترین روش‌ های parameterization در Shell است مثلاً :

```echo "Hello $1"```

با اجرای script.sh example/. می‌تواند example را به عنوان argument اول دریافت کند. اگر argument وجود نداشته باشد ، parameter مربوط معمولاً مقدار خالی خواهد داشت. همین موضوع دلیل اهمیت quoting را دوباره نشان می‌ دهد :

``` [ "$1" = "hi" ] Better than [ $1 = hi ]```

چون حالت خالی بودن $1 می‌تواند syntax را خراب کند.

#### 🔹 Number of Arguments: $#

متغیر#$ تعداد positional argument ها را مشخص می‌ کند مثلاً ```script.sh one two three/.```باعث می‌ شود 3 = #$ شود. این برای بررسی تعداد argument های مورد انتظار بسیار کاربردی است مثلاً :

```
if [ "$#" -ne 2 ]; then
    echo "Usage: $0 source destination"
    exit 1
fi
```

در چنین script ابتدا تعداد argument ها بررسی می‌ شود.

#### 🔹 All Arguments: $@

همچنین @$ به کل positional parameter های script اشاره می‌ کند اما نحوه‌ ی quote کردن آن بسیار مهم است. در حالت ``` "@$"``` هر argument اصلی به‌ صورت جداگانه حفظ می‌ شود. مثلاً اگر script با ```script.sh "hello world" test/.``` اجرا شود ، استفاده از ``` "@$"``` دو argument را نگه می‌ دارد : hello world و test اما @$ در بسیاری از context ها ممکن است بعداً تحت word splitting و دیگر قواعد قرار گیرد. به همین دلیل در script های حرفه‌ ای وقتی می‌ خواهید argument های دریافتی را همان‌ طور که هستند به command دیگری منتقل کنید ، الگوی مهم این ``` "@$"``` است این یکی از تفاوت‌ های ظریف و بسیار مهم Shell است.

#### 🔹 Script Name: $0

متغیر 0$ به command name یا path اشاره می‌ کند که Shell برای script در اختیار دارد. مثلاً اگر اجرا کنیم script.sh/. معمولاً 0$ مقداری مانند script.sh/. خواهد داشت. این متغیر در پیام‌ های usage و error بسیار مفید است :

```echo "Usage: $0 file"```

چون اگر نام script عوض شود، پیام نیز خودکار متناسب با invocation ساخته می‌ شود. نباید آن را با path مطلق یا نام واقعی فایل روی دیسک اشتباه گرفت مقدار 0$ به نحوه‌ ی فراخوانی بستگی دارد.

#### 🔹 Process ID: $$

این $$ شناسه‌ ی process شلی است که script را اجرا می‌ کند مثلاً "$$" echo می‌تواند PID آن Shell را نشان دهد. در گذشته و در بعضی script های ساده از $$ برای ساختن نامی نسبتاً یکتا استفاده می‌ شد. مثلاً $$./tmp/file اما این روش همیشه امن و مناسب نیست. PID می‌ تواند دوباره استفاده شود و مشکلات race condition ایجاد شوند. برای ایجاد فایل موقت، mktemp راه مناسب‌ تری است که در ادامه می‌ بینیم.

#### 🔹 Exit Code: $?

متغیر ?$ می تواند exit status آخرین command اجرا شده را در اختیار بگذارد مثلاً :

```
ls /tmp
echo "$?"
=> If ls succeeds, you usually see 0, If it fails, you might receive another amount.
```

این متغیر پایه‌ ی بسیاری از تصمیم‌ گیری‌های Shell script است مثلاً :

```
some_command

if [ "$?" -eq 0 ]; then
    echo "success"
fi
```

البته معمولاً این را می‌ توان مستقیم و خواناتر نوشت :

```
if some_command; then
    echo "success"
fi
```

چون if در Shell مستقیماً exit status یک command را بررسی می‌ کند. نکته‌ ی مهم این است که اجرای هر command دیگری "?$" را تغییر می‌ دهد. پس اگر می‌ خواهید status یک command را بررسی کنید ، نباید بین آن command و بررسی ?$ یک command دیگر اجرا کنید.

---

### Exit Codes

یکی از ویژگی‌ های اساسی Unix ، استفاده از exit status برای گزارش نتیجه‌ ی execution یک process است به‌طور معمول 0 نشانه‌ی موفقیت است. مقادیر غیر صفر معمولاً نشان‌ دهنده‌ی failure یا وضعیت دیگری هستند اما « غیر صفر = خطای واقعی» یک قانون مطلق نیست. بعضی برنامه‌ ها از exit status های غیر صفر برای وضعیت‌ هایی استفاده می‌ کنند که از دید برنامه کاملاً معمولی هستند. مثلاً grep foo file.txt اگر foo پیدا نشود ، grep معمولاً exit status 1 برمی‌گرداند. این اتفاق خطای واقعی به معنای خراب شدن grep نیست فقط یعنی pattern پیدا نشده است. همین موضوع درباره‌ی diff نیز مهم است تفاوت داشتن دو فایل ممکن است با exit status خاصی گزارش شود ، نه اینکه command با مشکل اجرایی مواجه شده باشد. بنابراین هنگام استفاده از exit status باید مستندات command مورد نظر را بشناسید.

#### 🔹 Exit status in the shell

اگر script خودش exit 0 اجرا کند ، status نهایی آن 0 خواهد بود یا مثلاً exit 1 می‌تواند failure را گزارش کند. به همین دلیل Shell script نیز مثل هر executable دیگر می‌ تواند نتیجه‌ ی کار خود را به caller برگرداند.

---

### Conditionals

برای تصمیم‌ گیری ، Shell  ساختارهای شرطی مختلفی دارد مهم‌ ترین‌ ها if و elif و else و case هستند. نکته‌ ی مهم در Shell این است که if مستقیماً یک عبارت منطقی به سبک بعضی زبان‌ های programming را evaluate نمی‌ کند. Shell یک command را اجرا می‌ کند و** exit status** آن را بررسی می‌ کند. اگر status برابر 0 باشد، شرط موفق در نظر گرفته می‌ شود. اگر nonzero باشد ، شرط شکست‌ خورده محسوب می‌ شود.

```
if [ "$1" = "hi" ]; then
    echo 'The first argument was "hi"'
else
    echo 'The first argument was not "hi"'
fi
```

در اینجا if ساختار شرط را آغاز می‌کند و ] در واقع command معروفی است که با نام test نیز شناخته می‌ شود و then شروع branch موفق است و else هم branch شکست است و fi پایان ساختار if است.

#### 🔹 [ ... ]

[ ... ] در واقع command است این نکته بسیار مهم است [ یک builtin یا در بسیاری از سیستم‌ ها حتی executable مستقل نیز دارد و syntax مربوط به test را پیاده می‌ کند یعنی :

```[ "$1" = hi ]```

عملاً یک command اجرا می‌ کند و این command یک exit status می‌ دهد مثلاً :

```
0 → condition true
nonzero → condition false
```

پس Shell چیزی شبیه این انجام می‌ دهد :

```
execute test
      ↓
inspect exit status
      ↓
choose branch
```

#### 🔹 ; in the if structure

وقتی بنویسیم : ```if [ "$1" = "hi" ]; then``` از ; قبل از then استفاده شده است. این ; جادوی مخصوص if نیست بلکه علامت پایان command در syntax شل است در نتیجه :

```
if [ "$1" = "hi" ]
then
    ...
fi
```

نیز معتبر است و دیگر به ; نیاز نداریم. پس ; فقط به parser می‌ گوید command قبلی تمام شده است.

#### 🔹 A Workaround for Empty Parameter Lists

یکی از مثال‌ های معروف Shell syntax این است : 

```[ $1 = hi ]```

اگر 1$ خالی باشد ، عبارت بعد از expansion می‌تواند به چیزی شبیه [ hi =   ] تبدیل شود. این syntax چیزی نیست که انتظار داشتیم راه صحیح‌ تر : 

```[ "$1" = hi ]```

است. در این حالت اگر 1$ خالی باشد [ hi = '' ''  ] داریم و syntax همچنان سالم است. این مثال نشان می‌ دهد quoting فقط درباره‌ ی space نیست بلکه برای حفظ ساختار argument ها نیز ضروری است.

#### 🔹 Other Commands for Tests

فقط if محدود به [ ... ] نیست هر command که exit status معناداری دارد می‌ تواند مستقیماً در if استفاده شود. مثلاً :

```
if grep -q root /etc/passwd; then
    echo "root exists"
fi
```

در اینجا اصلاً ] نداریم و خود grep شرط را فراهم می‌ کند. این یکی از قابلیت‌ های جالب Shell است ، چون command های معمول Unix از طریق exit status می‌ توانند مستقیماً بخشی از منطق برنامه شوند.

#### 🔹 elif

وقتی چند حالت جایگزین داریم ، elif به ما اجازه می‌ دهد شرط‌ ها را پشت سر هم قرار دهیم مثلاً :

```
if [ "$1" = "hi" ]; then
    echo "hello"
elif [ "$2" = "bye" ]; then
    echo "goodbye"
else
    echo "something else"
fi
```

ابتدا Shell شرط if را بررسی می‌ کند. اگر موفق شد ، فقط همان branch اجرا می‌ شود و اگر موفق نشد ، سراغ elif می‌ رود. این روند ادامه پیدا می‌ کند تا اولین شرط موفق پیدا شود و اگر هیچ شرطی موفق نشود و else وجود داشته باشد ، else اجرا می‌ شود. مثلاً اگر script با script.sh hi bye/. اجرا شود ، هر دو شرط ظاهراً درست هستند ، اما فقط branch اول اجرا می‌ شود. بنابراین elif برای حالت‌ هایی مناسب است که mutually exclusive هستند و باید فقط اولین match انتخاب شود. اگر تعداد case ها زیاد شود ، case معمولاً خوانا تر است.

#### 🔹 Logical Constructs

در Shell علاوه بر if و case ، عملگرهای منطقی مهمی برای ترکیب commandها وجود دارد دو مورد بسیار مهم && و || هستند. && اگر command سمت چپ موفق شود، command سمت راست اجرا می‌ شود مثلاً : 

```mkdir test && cd test```

اگر ساخت directory موفق باشد ، cd اجرا می‌ شود. اگر mkdir fail شود ، cd اجرا نمی‌ شود. || اگر command سمت چپ fail شود ، command سمت راست اجرا می‌ شود. مثلاً : 

```mkdir test || echo "Could not create directory"```

پس می‌ توان از این دو برای ساختن منطق کوتاه و idiomatic در Shell استفاده کرد. البته با پیچیده‌ تر شدن منطق ، if اغلب خوانا تر می‌ شود.

#### 🔹 Negation

در context های مختلف می‌ توان از ! برای منفی کردن نتیجه استفاده کرد مثلاً :

```
if ! grep -q foo file.txt; then
    echo "foo was not found"
fi
```

در این حالت موفقیت grep به failure و failure آن به success برای command compound تبدیل می‌ شود.

#### 🔹 Logical tests in test

در test و ] نیز syntax هایی برای ترکیب condition ها دارند اما syntax های پیچیده‌ی a- و o- در test قدیمی و مبهم هستند. برای script های portable و readable ، استفاده از چند test با && ، || یا پرانتزهای مناسب در syntax پشتیبانی‌ شده معمولاً واضح‌ تر است.

#### 🔹 Testing Conditions

کاربرد ] برای تست کردن condition ها به کار می‌ رود مثلاً :

```[ "$x" = "$y" ]```

اگر دو رشته برابر باشند ، exit status برابر 0 می‌ شود. Condition های مهم را می‌ توان در چند گروه دید File Tests و String Tests و Arithmetic Tests.

- **گروه اول File Tests** : یکی از کاربردهای مهم ] بررسی فایل‌ هاست مثلاً :

```[ -e "$file" ]```

بررسی می‌کند path وجود دارد.

```[ -f "$file" ]```

بررسی می‌ کند موارد ،  مورد نظر یک regular file باشد.

```[ -d "$dir" ]```

برای directory و موارد دیگر :

```
-h → symbolic link
-b → block device
-c → character device
-p → named pipe
-S → socket
```

این تست‌ها در script های system administration بسیار کاربردی هستند.

#### 🔹 Permission Tests

مواردی مثل r- و w- و x- به ترتیب برای بررسی read و write و execute permission قابل استفاده‌ اند. همچنین u- و g- و k- برای بررسی ویژگی‌هایی مثل setuid و setgid و sticky bit استفاده می‌ شوند. این تست‌ها **فقط permission bit روی فایل** را به‌ صورت انتزاعی گزارش نمی‌ کنند بلکه باید توجه داشت که مثلاً موفقیت واقعی در دسترسی به فایل به identity و قوانین permission سیستم نیز وابسته است.

#### 🔹 Comparing two files

سه تست مهم :  nt- بررسی می‌ کند فایل اول جدید تر باشد  و ot- برای قدیمی‌ تر بودن استفاده می‌ شود همچنین ef-  بررسی می‌ کند دو path به یک file در filesystem یکسان اشاره کنند. این معمولاً برای تشخیص link های یکسان مفید است مثلاً دو hard link به یک inode می‌ توانند این test را موفق کنند.

#### 🔹 String Tests

برای String ها می‌ توان از = و =! و z- و n- استفاده کرد مثلاً :

```[ "$name" = "john" ]```

برابری را بررسی می‌ کند.

```[ "$name" != "john" ]```

نابرابری را بررسی می‌ کند.

```[ -z "$name" ]```

یعنی string خالی است و 

```[ -n "$name" ]```

یعنی string خالی نیست. 

نکته‌ ی مهم z- و n- را معمولاً برای بررسی مستقیم خالی بودن یا نبودن رشته استفاده می‌ کنیم و به همین دلیل بسیار خوانا هستند.

#### 🔹 Arithmetic Tests

یکی از اشتباهات رایج این است که برای مقایسه‌ ی عددی از = استفاده شود مثلاً [ 1 = 01 ] از نظر string comparison شکست می‌خورد ، چون :

```"01" != "1"```

اما 

```[ 01 -eq 1 ]```

موفق است، چون comparison عددی انجام می‌ شود.

عملگرهای مهم عددی :
```
-eq → equal 
-ne → not equal 
-lt → less than 
-gt → greater than
-le → less than or equal
-ge → greater than or equal
```

پس این تفاوت باید کاملاً روشن باشد :

```
=   → string comparison
-eq → numeric comparison
```

در Shell های رایج از جمله Bash هم test و ] معمولاً به‌ صورت builtin وجود دارند. بنابراین اجرای یک test لزوماً به معنی ایجاد یک process جداگانه نیست. همچنین ] شکل مخصوص syntax شرط در سطح زبان نیست بلکه یک command است که Shell آن را در context مناسب اجرا می‌ کند.

#### 🔹 case

برای زمانی case  بسیار مناسب است که بخواهیم یک مقدار را با چند pattern مقایسه کنیم. ساختار کلی :

```
case "$1" in
    pattern1)
        commands
        ;;
    pattern2)
        commands
        ;;
    *)
        default_commands
        ;;
esac
```

برای مثال :

```
case "$1" in
    bye)
        echo "Fine, bye."
        ;;
    hi|hello)
        echo "Nice to see you."
        ;;
    "what"?)
        echo "Whatever."
        ;;
    *)
        echo "Huh?"
        ;;
esac
```

در case چند نکته مهم وجود دارد :

در Shell ، دستور case مقدار را با pattern ها مقایسه می‌ کند و به همین دلیل فقط برای بررسی برابری ساده نیست بلکه می‌ توان از patternهای glob مانند * و ? ، همچنین character class ها و ساختارهای دیگر استفاده کرد. برای مثال :

```
case "$1" in
 *.txt)
 echo "text file"
 ;;
 *.jpg|*.png)
 echo "image file"
 ;;
 *)
 echo "unknown file"
 ;;
esac
```

در این مثال ، .txt* فایل‌ های متنی را match می‌کند و jpg|*.png.* با استفاده از | دو pattern جایگزین را برای یک branch تعریف می‌ کند. در انتهای case معمولاً از (* به‌عنوان حالت پیش‌ فرض استفاده می‌ شود تا هر مقداری که با pattern های قبلی match نشده است ، وارد این branch شود. هر branch معمولاً با ;; پایان می‌یابد و ;; مشخص می‌ کند که دستورات مربوط به branch فعلی تمام شده‌ اند و پس از اجرای آن‌ ها ، Shell باید از ساختار case خارج شود و ادامه اسکریپت را اجرا کند. در واقع ;; باعث می‌ شود Shell پس از match شدن یک branch ،  سراغ pattern های بعدی نرود. در نهایت ، case با esac بسته می‌ شود و زمانی که بخواهیم حالت‌ های مختلف یک متغیر را بررسی کنیم ، معمولاً خوانا تر از زنجیره‌ های طولانی if/elif است.

---

### Loops




















