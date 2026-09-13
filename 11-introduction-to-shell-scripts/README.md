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
- [Command Substitution](#command-substitution)
- [Temporary File Management](#temporary-file-management)
- [Here Documents](#here-documents)
- [Important Shell Script Utilities](#important-shell-script-utilities)
- [Subshells](#subshells)
- [Including Other Files in Scripts](#including-other-files-in-scripts)
- [Reading User Input](#reading-user-input)
- [When Not to Use Shell Scripts](#when-not-to-use-shell-scripts)
- [Tips](#tips)

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

شل علاوه بر branch ها ، امکان تکرار command ها را نیز فراهم می‌ کند. دو loop اصلی در شل for و while هستند و until نیز loop دیگری است که منطق آن تقریباً معکوس while عمل می‌ کند.

#### 🔹 for Loops

حلقه for در Bourne-style شل بیشتر شبیه for-each loop است تا for عددی در زبان‌ هایی مانند C. در این ساختار، یک متغیر در هر iteration مقدار بعدی را از یک فهرست دریافت  می‌کند :

```
for str in one two three four; do
    echo "$str"
done
```

در هر iteration ، مقدار str به‌ ترتیب one ، two ، three و four می‌ شود. بنابراین for می‌ تواند روی یک فهرست از value ها پیمایش کند. این فهرست می‌تواند نتیجه globbing نیز باشد :

```
for file in *.txt; do
    echo "$file"
done
```

در اینجا ابتدا globbing الگوی txt.* را به مجموعه‌ ای از filename ها تبدیل می‌ کند و سپس loop روی این filename ها حرکت می‌ کند.
#### 🔹 The importance of quoting

در script های واقعی ، هنگام استفاده از متغیرهایی که ممکن است شامل space باشند ، باید به quote کردن توجه کنیم. برای مثال : 

```echo "$file"```

امن‌تر از echo $file است زیرا در حالت دوم ممکن است word splitting اتفاق بیفتد و مقدار متغیر به چند word تقسیم شود.

#### 🔹 Script arguments

یک الگوی بسیار رایج برای پیمایش تمام آرگومان‌ های script استفاده از "@$" است :

```
for arg in "$@"; do
    echo "argument: $arg"
done
```
در این حالت، "@$" هر آرگومان را به‌ صورت جداگانه حفظ می‌ کند بنابراین ساختار آرگومان‌ ها ، حتی اگر شامل space باشند ، از بین نمی‌ رود.

#### 🔹 while Loops

حلقه while نیز بر اساس exit status یک command عمل می‌ کند . ساختار کلی آن به شکل زیر است:

```
while some_command; do
    ...
done
```

تا زمانی که some_command با status صفر تمام شود ، loop ادامه پیدا می‌ کند. به محض اینکه command یک status غیرصفر برگرداند ، شرط while برقرار نیست و loop پایان می‌ یابد.

#### 🔹 break

با استفاده از break می‌ توان بدون منتظر ماندن برای false شدن شرط ، زودتر از loop خارج  شد. برای مثال :

```
while true; do
    read answer

    if [ "$answer" = "quit" ]; then
        break
    fi
done
```

در این مثال ، while true به‌ صورت عادی همیشه ادامه پیدا می‌ کند ، اما وقتی کاربر مقدار quit را وارد کند ، دستور break اجرا می‌ شود و Shell بلافاصله از loop خارج می‌ شود.

#### 🔹 until

 از نظر منطق شرط until ، تقریباً معکوس while است :
```
until some_command; do
    ...
done
````

در until ، loop تا زمانی ادامه پیدا می‌ کند که command status غیرصفر برگرداند. وقتی command با status صفر تمام شود، loop پایان می‌ یابد. بنابراین می‌ توان منطق این دو را به شکل زیر خلاصه کرد :

```
while → continue while status == 0
until → continue while status != 0
```

در نهایت ، وجود loop های زیاد و منطق بسیار پیچیده در یک Shell script می‌ تواند نشانه‌ ای باشد که مسئله از محدوده مناسب Shell خارج شده و بهتر است با زبان یا ابزار مناسب‌ تری پیاده‌ سازی شود.

---

### Command Substitution

یکی از قابلیت‌ های بسیار مهم Shell ، استفاده از خروجی یک command به‌ عنوان بخشی از command دیگر است. syntax مدرن (command)$ است مثلاً :

```
today=$(date)
echo "$today"
```

ابتدا date اجرا می‌ شود و خروجی آن جمع‌ آوری می‌ شود و بعد در assignment قرار می‌ گیرد.
#### 🔹 Pipeline in command substitution

مثلاً :

```FLAGS=$(grep '^flags' /proc/cpuinfo | sed 's/.*://' | head -1)```

از چند command تشکیل شده است ابتدا grep داده را پیدا می‌ کند بعد sed آن را پردازش می‌ کند و head مقدار مورد نیاز را انتخاب می‌ کند. در پایان نتیجه داخل FLAGS قرار می‌گیرد. Command substitution برای ترکیب قدرت command های Unix با منطق script بسیار مهم است.

#### 🔹 Legacy syntax

در Shell های قدیمی می‌ توان از backtick نیز استفاده کرد`command` اما (command)$ خوانا تر است و امکان nesting آن نیز بهتر است در حالی که nesting backtick ها بسیار دشوار و گیج‌ کننده می‌ شود  مثلاً :

#### 🔹 trailing newlines

یکی از جزئیات مهم command substitution این است که trailing newline های output در هنگام substitution حذف می‌ شوند. این موضوع در بعضی پردازش‌ های متنی اهمیت دارد. همچنین اگر خروجی command شامل چند خط باشد ، کل خروجی وارد context substitution می‌ شود و بعد بسته به استفاده‌ ی آن ممکن است تحت splitting قرار گیرد پس مثلاً : 

```files=$(find . -type f)```

همیشه بهترین روش انتقال لیست فایل‌ ها نیست ، مخصوصاً اگر filename ها بتوانند شامل newline یا کاراکترهای عجیب باشند. این همان دلیلی است که در کار با file list ها گاهی بهتر است از pipe مستقیم ، find -exec یا find -print0 استفاده کنیم.

---

### Temporary File Management

گاهی script نیاز دارد داده‌ ای را موقتاً در یک فایل نگهداری کند مثلاً : 

```
command A
     ↓
temporary file
     ↓
command B
```
یکی از بد ترین روش‌ ها این است که یک نام ثابت و قابل‌ پیش‌بینی بسازیم tmp/myfile/ چون ممکن است  race condition یا حتی مشکل امنیتی ایجاد شود :

```
- file already exists
- another process creates it first
- another user can manipulate it
```
#### 🔹 Using PID

روشی قدیمی‌ تر استفاده از $$./tmp/file است. چون $$ معمولاً PID shell را در بر دارد و احتمال برخورد را کاهش می‌ دهد. اما این روش تضمین امنیتی کافی ندارد.

#### 🔹 mktemp

روش مناسب‌ ترmktemp است مثلاً :

```TMPFILE=$(mktemp /tmp/example.XXXXXX)```

با XXXXXX توسط mktemp یک الگوی مناسب و unique جایگزین می‌ شود و فایل را نیز ایجاد می‌ کند. این نکته خیلی مهم است **فقط انتخاب یک نام تصادفی کافی نیست باید creation هم به شکلی امن انجام شود**. mktemp دقیقاً برای همین سناریو طراحی شده است.

#### 🔹 cleanup

فرض کنید script با Ctrl+C قطع شود. در این حالت ممکن است temporary file باقی بماند برای cleanup می‌ توان از ```trap``` استفاده کرد مثلاً : 

```
TMPFILE=$(mktemp)

cleanup() {
    rm -f "$TMPFILE"
}

trap cleanup EXIT
```

در این الگو cleanup به پایان script متصل می‌ شود استفاده از trap باعث می‌ شود cleanup فقط به یک خط موفقیت‌ آمیز در انتهای script وابسته نباشد.

---

### Here Documents

اگر بخواهیم حجم نسبتاً بزرگی از متن را به standard input یک command بدهیم ، مجبور نیستیم چندین echo بنویسیم. Shell مکانیزمی به نام here document دارد ساختار کلی :

```
command <<EOF
line 1
line 2
line 3
EOF
```

در اینجا >>EOF به Shell می‌ گوید خطوط بعدی را به standard input command بدهد. وقتی Shell به خطی برسد که فقط شامل EOF است، here document پایان می‌ یابد. EOF نام خاص و جادویی نیست می‌توان نام دیگری انتخاب کرد مانند >>END ، مهم این است که marker ابتدا و پایان یکسان باشد.

#### 🔹 Variable Expansion

در حالت معمول ، متغیرها داخل here document expand می‌ شوند مثلاً :
```
DATE=$(date)

cat <<EOF
Date: $DATE
The output above is from the Unix date command.
EOF
```

در اینجا DATE$ قبل از اینکه متن به cat داده شود ، توسط Shell جایگزین می‌ شود.
#### 🔹 Quoted Delimiter

اگر delimiter را quote کنیم Shell expansion را انجام نمی‌ دهد یعنی محتوا تقریباً literal باقی می‌ ماند  :

```
cat <<'EOF'
$HOME
$(date)
EOF
```

کاربرد Here document برای مواردی مثل  configuration generation و SQL input و multi-line messages و scripted commands و report generation می باشد.

---

### Important Shell Script Utilities

شل خودش امکاناتی برای منطق برنامه فراهم می‌ کند ، اما قدرت واقعی آن وقتی مشخص می‌ شود که command های Unix را کنار هم قرار دهید. این بخش چند ابزار مهم را معرفی می‌ کند : basename و awk و sed و xargs و expr و exec . بعضی از این‌ ها مثل basename utility های ساده‌ اند و بعضی دیگر مثل awk تقریباً یک زبان برنامه‌ نویسی مستقل هستند.

#### 🔹 basename

برای استخراج بخش نهایی یک pathname استفاده می‌ شود مثلاً ```basename /usr/local/bin/example``` نتیجه example خواهد بود. یعنی component های directory حذف می‌ شوند. همچنین می‌ توان suffix مشخصی را نیز حذف کرد ```basename example.html .html``` که نتیجه example است. این utility در script ها برای جدا کردن filename از path بسیار رایج است مثلاً : 

```
file="/var/log/app.log"
name=$(basename "$file")
```

اکنون name=app.log است. البته برای بعضی عملیات path در  scriptهای پیچیده‌ تر، ابزارهایی مثل dirname نیز اهمیت دارند.

#### 🔹 awk

نباید awk را صرفاً یک command ساده در نظر گرفت. awk در واقع یک زبان پردازش متن و برنامه‌ نویسی است. یکی از کاربردهای بسیار رایج آن استخراج field هاست مثلاً :

```ls -l | awk '{print $5}'```

در یک output کلاسیک ls -l، این command field پنجم را چاپ می‌ کند. مفاهیم مهم awk شامل records و fields و patterns و actions و variables و conditions و loops هستند می‌ توان نوشت :

```awk '$5 > 1000 {print $9}' file```

یعنی برای record هایی که field پنجم شان از مقدار مشخصی بیشتر است ، field دیگری را نمایش بده.

> نکته‌ ی مهم درباره‌ ی ls ، استفاده از ```ls -l | awk``` برای آموزش مفهوم field مفید است، اما برای automation دقیق روی filename ها همیشه انتخاب مطمئنی نیست ، چون ls برای machine parsing طراحی نشده است. برای  script های robust معمولاً بهتر است از ابزارهایی استفاده شود که خروجی ساختاریافته‌ تری دارند.

#### 🔹 sed

مخفف stream editor است. این ابزار input را به‌ صورت stream می‌ گیرد و می‌ تواند بر اساس pattern ها آن را تغییر دهد .دو operation بسیار رایج s و d هستند.

#### 🔹 sed - Substitute

مثلاً ```sed 's/:/%/g' /etc/passwd هر : را به % تبدیل می‌ کند. ساختار کلی به صورت s/old/new/  است و g در پایان یعنی تمام occurrence های مورد نظر در هر خط جایگزین شوند ، نه فقط اولین occurrence.

#### 🔹 sed - Delete

با d می‌ توان line هایی را حذف کرد بر اساس range یا pattern مثلاً ```sed '/pattern/d' file```.

#### 🔹 sed - Regular Expressions

به‌ طور گسترده sed از pattern matching و regular expressions برای انتخاب line ها استفاده می‌ کند در نتیجه می‌توان بر اساس line number و range و regex تصمیم گرفت چه چیزی تغییر کند. sed مخصوصاً برای transformation های متنی سریع بسیار مناسب است.

#### 🔹 xargs

فرض کنید command A یک فهرست طولانی از argument تولید می‌ کند و command B باید روی آن‌ ها اجرا شود. xargs می‌ تواند output را به argument تبدیل کند و command مورد نظر را با آن‌ ها اجرا کند. مثال کلاسیک :

```find . -name '*.gif' -print | xargs file```

اما این روش یک مشکل جدی دارد. اگر filename شامل space یا newline یا quotes باشد ، parsing ساده‌ ی xargs می‌ تواند خروجی را خراب کند. حتی در بعضی context ها می‌ تواند security problem ایجاد کند.

#### 🔹 xargs - A safer way for filenames

استفاده از null delimiter :

```find . -name '*.gif' -print0 | xargs -0 file```

در این مدل print0- هر path را با byte صفر جدا می‌کند و 0- در xargs می‌ گوید همین روش parsing را استفاده کند. این روش می‌تواند filename هایی با space یا newline را نیز بهتر مدیریت کند.

#### 🔹 xargs - find -exec

گاهی اصلاً xargs لازم نیست خود find می‌تواند از :

```find . -name '*.gif' -exec file {} \;```

استفاده کند. یا برای کاهش تعداد invocation ها در حالت‌ های مناسب :

```find . -name '*.gif' -exec file {} +```

این روش در بسیاری از script ها ساده‌ تر و مطمئن‌ تر است.

#### 🔹 expr

برای expression های ساده استفاده می‌ شود مثلاً expr 1 + 2 نتیجه 3 است. می‌ توان از آن برای عملیات عددی و بعضی عملیات string نیز استفاده کرد. اما syntax آن قدیمی است در Shell های امروزی ، برای محاسبات ساده معمولاً روش‌ های دیگری وجود دارند. مثلاً Bash امکانات arithmetic expansion دارد : ```echo $((1 + 2))``` . در script های sh نیز بسته به نیاز می‌ توان از ابزارها یا زبان دیگری استفاده کرد. بنابراین expr بیشتر برای شناخت محیط کلاسیک Unix اهمیت دارد.

#### 🔹 exec

یک Shell builtin بسیار مهم است کار آن این است که process فعلی Shell را با برنامه‌ ی جدید جایگزین کند مثلاً ```exec command``` باعث می‌ شود Shell process دیگر command قبلی را به‌ عنوان child اجرا نکند بلکه خودش جای آن command قرار بگیرد. از دید مفهومی، در اجرای معمولی یک command ، ساختار به این صورت است :

```
Shell process
     ↓
 run command
     ↓
New program
```

در این حالت ، Shell به اجرای خود ادامه می‌ دهد و در صورت نیاز منتظر پایان command می‌ ماند. اما با استفاده از exec ، خود process مربوط به Shell با برنامه جدید جایگزین می‌ شود :

```
Shell process
     ↓
    exec
     ↓
New program
```

پس بعد از اجرای exec command ، شل قبلی دیگر به‌ عنوان یک process جداگانه وجود ندارد همان process که قبلاً Shell بود ، اکنون برنامه جدید را اجرا می‌ کند. بنابراین exec command با اجرای معمولی command تفاوت اساسی دارد : در حالت معمول ، Shell باقی می‌ ماند و command را اجرا یا برای آن wait می‌ کند ، اما در حالت exec command ، خود Shell با command جایگزین می‌ شود.

---

### Subshells

گاهی می‌خواهیم تعدادی command را در یک محیط Shell موقت اجرا کنیم، بدون اینکه تغییرات آن‌ ها به Shell فعلی برگردد. برای این کار می‌ توان command ها را داخل پرانتز قرار داد ```(cd uglydir; uglyprogram)``` این ساختار در یک **subshell environment** اجرا می‌ شود. اگر داخل آن cd uglydir انجام دهیم ، تغییر directory فقط برای همان subshell است و بعد از پایان آن ، Shell اصلی هنوز در directory قبلی خواهد بود.

#### 🔹 Environment Variable

همین موضوع برای environment variable نیز کاربرد دارد ```(PATH=/usr/confusing:$PATH; uglyprogram)``` در این حالت مقدار جدید PATH فقط برای آن execution context استفاده می‌ شود. یک syntax بسیار ساده‌ تر نیز وجود دارد ```PATH=/usr/confusing:$PATH uglyprogram``` این مقدار environment را فقط برای اجرای همان command تنظیم می‌ کند. این روش نیازی به subshell صریح ندارد.

#### 🔹 Subshell and Pipeline

همچنین Pipeline ها نیز می‌ توانند باعث ایجاد execution environment های جداگانه شوند. به همین دلیل بعضی تغییرات variable در یک بخش pipeline الزاماً در Shell اصلی قابل مشاهده نیستند. این جزئیات یکی از دلایل تفاوت Shell scripting با زبان‌هایی مثل Python است. process model و environment باید همیشه در ذهن برنامه‌ نویس باشد.

---

### Including Other Files in Scripts

گاهی لازم است چند script یا configuration مشترک داشته باشند. به جای اینکه محتویات را کپی کنیم ، می‌ توان فایل دیگری را در Shell فعلی وارد کرد عملگر '.' برای این کار استفاده می‌ شود مثلاً ```config.sh . ``` این عملیات را **sourcing** می‌ گویند. در Bash می‌ توان شکل دیگری هم دید ```source config.sh``` اما '.' روش استاندارد تر و قابل‌ حمل‌ تر در Bourne-style shells است.

#### 🔹 Difference between source and standard script execution

فرض کنید config.sh یک variable تعریف کند. اگر آن script را به‌عنوان یک command معمولی اجرا کنیم ، تغییر environment آن معمولاً به Shell والد برنمی‌ گردد ولی اگر ```config.sh .``` اجرا کنیم ، command های فایل در همان Shell فعلی اجرا می‌ شوند. در نتیجه : 

```
. config.sh
echo "$MY_VARIABLE"
```

می‌تواند variable ای را که داخل config.sh تعریف شده ، در Shell فعلی در اختیار داشته باشد این برای shared configuration و environment variables و function definitions و common shell code بسیار مفید است.

---

### Reading User Input

فقط Shell script می‌تواند argument دریافت کند. می‌ تواند هنگام اجرا نیز از user ورودی بگیرد و builtin read برای این کار استفاده می‌ شود و Shell یک line از standard input می‌خواند و آن را در متغیر قرار می‌ دهد مثلاً  : 

```
read var
echo "You entered: $var"
```

#### 🔹 User interaction

می‌ توان برای تعامل با user  یک prompt ساخت :

```
printf "Enter your name: "
read name
echo "Hello, $name"
```

یا قبل از انجام عملیات حساس تأیید گرفت بعد می‌ توان با case یا if پاسخ را بررسی کرد :

```
printf "Are you sure? [y/N] "
read answer
```

#### 🔹 Limiting input

محدود کردن ورودی بسته به Shell می‌ توان برای read گزینه‌های مختلفی داشت. مثلاً Bash امکاناتی برای timeout و delimiter و silent input و multiple variables ارائه می‌ دهد. اما اگر هدف portability به bin/sh/ است ، باید فقط به قابلیت‌ هایی تکیه کرد که توسط shell هدف تضمین شده‌ اند.

---

### When Not to Use Shell Scripts

این بخش یکی از مهم‌ ترین قسمت‌ های فصل است ، چون به جای آموزش یک syntax جدید ، درباره‌ ی **انتخاب ابزار مناسب** صحبت می‌ کند Shell برای این کارها عالی است :

```
- Executing commands
- Combining commands
- Pipelines
- File management
- Connecting Unix tools
- Simple automation
- System administration
```

مثلاً چنین چیزی کاملاً در حوزه‌ ی طبیعی Shell است :

```
find /var/log -name '*.log' |
    grep error |
    sort |
    head
```

در اینجا Shell به‌ عنوان چسبی میان چند ابزار تخصصی عمل می‌ کند. اما وقتی script تبدیل شود به یک برنامه‌ی بزرگ با state پیچیده و ساختمان داده و الگوریتم‌ های پیچیده و پردازش سنگین و error handling گسترده همراه با بخش‌ های زیاد و وابسته ، دیگر مزیت Shell کمتر می‌ شود. یکی از نشانه‌ های خوب این است اگر بیشتر زمانتان صرف جنگیدن با syntax و quoting و word splitting می‌ شود تا حل خود مسئله ، احتمالاً باید ابزار دیگری انتخاب کنید.

#### 🔹 Select another language

بسته به مسئله ممکن است انتخاب‌ های مناسب‌ تر این‌ ها باشند ، Python و Perl و awk . مثلاً :

```
Shell → Orchestration and system commands
awk → Text processing and structured records
Python → Complex logic and data structures
```

این به معنی بد بودن Shell نیست برعکس، Shell یکی از مهم‌ ترین ابزارهای Unix است. اما قدرت واقعی یک programmer یا administrator این نیست که هر کاری را با Shell انجام دهد بلکه این است که تشخیص دهد **چه زمانی Shell ابزار مناسب است و چه زمانی مناسب نیست**.

---

### Tips

فصل یازدهم Shell را از یک محیط اجرای command به یک ابزار واقعی برای automation و scripting تبدیل می‌ کند. در این فصل ساختار script ، shebang ، quoting ، پارامترها ، exit code ها ، شرط‌ ها ، case، loop ها، command substitution ، فایل‌ های موقت ، here document ، ابزارهایی مثل awk و sed ، exec ، subshell ، sourcing و read را بررسی کردیم.
مهم‌ تر از syntax ، فصل تأکید می‌کند که باید مرز استفاده از Shell را بشناسیم: **برای ترکیب command ها و automation عالی است ، اما برای منطق پیچیده بهتر است سراغ زبان مناسب‌ تری برویم**.
