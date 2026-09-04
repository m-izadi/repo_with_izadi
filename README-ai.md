<div dir="rtl" lang="fa">

# تمدید سرتیفیکیت SSL رجیستری Nexus

این سند شرح کامل تمدید سرتیفیکیت `*.kavosh.org` روی `repo-nexus.kavosh.org` است: از تبدیل فایل PFX تا خطای Docker، بعد خطای کوبرنتیز روی vSphere، و راه‌حلی که **بدون دست زدن به ESXi/vCenter** مشکل را بست.

هدف این است که دفعهٔ بعد، با دیدن پیام خطا بفهمی مشکل از خود سرتیفیکیت است، از زنجیره است، یا از trust store کلاینت.

---

## ۱. معماری سرویس (کجا فایل می‌نشیند)

Nginx جلوی Nexus است و HTTPS را terminate می‌کند. تنظیم فعلی این است:

```nginx
ssl_certificate     /etc/ssl/certs/kvsh.crt;
ssl_certificate_key /etc/ssl/certs/kvsh.key;
```

این مسیرها **داخل کانتینر** `nexus-nginx` هستند. روی هاست، volume این‌طور مپ شده:

| روی سرور Nexus | داخل کانتینر |
| --- | --- |
| `/data/nexus/certs/kvsh.crt` | `/etc/ssl/certs/kvsh.crt` |
| `/data/nexus/certs/kvsh.key` | `/etc/ssl/certs/kvsh.key` |
| `/data/nexus/nginx-conf/conf.d/nexus.conf` | `/etc/nginx/conf.d/nexus.conf` |

نکتهٔ عملی خیلی مهم: ساختن فایلی به نام `kvsh2027.crt` روی لپ‌تاپ یا در `~/cert-kvsh2027/` هیچ اثری روی nginx ندارد، تا وقتی همان محتوا را روی **`/data/nexus/certs/kvsh.crt`** کپی نکنی و nginx را reload نکنی.

پورت‌های رجیستری Docker معمولاً `8110` و بقیهٔ گروه‌ها هستند؛ UI و بعضی pullها روی `443`. همهٔ این vhostها **همان دو فایل crt/key** را می‌خوانند. یک‌بار درست کردن `kvsh.crt` همه را با هم عوض می‌کند.

---

## ۲. زنجیرهٔ اعتماد چیست؟ (ریشهٔ همهٔ این ماجرا)

یک سرتیفیکیت دامنه به‌تنهایی کافی نیست. کلاینت باید بتواند از دامنه تا یک **روت مورد اعتماد خودش** یک زنجیره بسازد.

```text
سرتیفیکیت دامنه  (*.kavosh.org)
        ↓ امضا شده توسط
Intermediate / CA میانی
        ↓ امضا شده توسط
روت CA   ← این باید داخل trust store کلاینت باشد
```

سه قانون را همیشه به خاطر بسپار:

1. **nginx باید زنجیره را بفرستد**؛ نه فقط سرتیفیکیت دامنه. ترتیب فایل `ssl_certificate`: اول leaf (دامنه)، بعد intermediateها. روت را معمولاً نفرست (کلاینت باید روت را از قبل داشته باشد).
2. **مرورگر با کلاینت‌های Go/Docker/ESXi فرق دارد.** مرورگر اگر intermediate نباشد، اغلب با AIA از اینترنت دانلودش می‌کند. Docker و image service وی‌ام‌ور این کار را نمی‌کنند؛ فقط همان چیزی را که سرور فرستاده به‌علاوهٔ CAهای داخل سیستم خودشان بررسی می‌کنند.
3. **همهٔ کلاینت‌ها یک trust store ندارند.** لینوکس جدید و مرورگر، روت‌های تازه‌تر Certum را دارند. ESXi/vCenter معمولاً مجموعهٔ قدیمی‌تری دارند.

اگر فقط یکی از این سه تا رعایت نشود، ممکن است مرورگر سبز باشد و `docker pull` یا پاد کوبر قرمز.

---

## ۳. سرتیفیکیت قدیمی در مقابل جدید

هر دو از Certum بودند و هر دو برای `*.kavosh.org`، ولی **سلسله‌مراتب روت‌شان فرق داشت.** همین تفاوت، علت خطای کوبر بود.

### زنجیرهٔ قدیمی (تا ۳ سپتامبر ۲۰۲۶ کار می‌کرد)

```text
*.kavosh.org
    → Certum Domain Validation CA SHA2
        → Certum Trusted Network CA     ← روت قدیمی (Unizeto)
```

این روت قدیمی روی ESXiهای vSphere از قبل trusted بود. برای همین پادها با سرتیفیکیت قبلی image می‌کشیدند.

سرتیفیکیت دامنهٔ قدیمی در **۳ سپتامبر ۲۰۲۶ منقضی شد**. امروز دیگر قابل استفاده نیست؛ حتی اگر زنجیره‌اش روی ESXi عالی باشد.

### زنجیرهٔ جدید (۲۰۲۶–۲۰۲۷)

```text
*.kavosh.org
    → Certum DV TLS G2 R39 CA
        → Certum Trusted Root CA        ← روت جدید (Asseco)
```

Certum محصولات DV را به روت جدید منتقل کرده. لینوکس جدید و مرورگر این روت را دارند. **ESXi این روت را ندارد.**

---

## ۴. مرحلهٔ اول: تبدیل PFX به چیزی که nginx می‌فهمد

فایل `.pfx` (PKCS#12) بستهٔ ویندوزی است: سرتیفیکیت + کلید + (گاهی) زنجیره، با رمز. Nginx PFX نمی‌خواند؛ PEM می‌خواهد: یک `crt` و یک `key` بدون passphrase.

اگر OpenSSL 3 خطا داد، `-legacy` لازم است (الگوریتم قدیمی داخل PFX).

```bash
openssl pkcs12 -in kavosh2027.pfx -nocerts -nodes -out kvsh2027.key -legacy
openssl pkcs12 -in kavosh2027.pfx -clcerts -nokeys -out server2027.crt -legacy
openssl pkcs12 -in kavosh2027.pfx -cacerts -nokeys -out ca-chain2027.crt -legacy
```

- `-nodes` یعنی کلید را بدون رمز بگذار؛ وگرنه nginx موقع استارت گیر می‌کند.
- `ca-chain2027.crt` در این تمدید **خالی از آب درآمد (۰ بایت)**. یعنی PFX زنجیره را همراه نداشت. `cat server2027.crt ca-chain2027.crt` عملاً فقط leaf ساخت. این اولین تله بود.

چک دامنه، تاریخ، و جفت بودن کلید:

```bash
openssl x509 -in server2027.crt -noout -subject -dates -ext subjectAltName
openssl x509 -noout -modulus -in server2027.crt | openssl md5
openssl rsa  -noout -modulus -in kvsh2027.key     | openssl md5
```

دو hash باید یکی باشند. کلید را در git نگذار؛ فقط `crt` را در `Cert/` نگه داشتیم.

روی سرور Nexus:

```bash
sudo cp /data/nexus/certs/kvsh.crt /data/nexus/certs/kvsh.crt.bak
sudo cp /data/nexus/certs/kvsh.key /data/nexus/certs/kvsh.key.bak
sudo cp kvsh2027.crt /data/nexus/certs/kvsh.crt
sudo cp kvsh2027.key /data/nexus/certs/kvsh.key
sudo chmod 644 /data/nexus/certs/kvsh.crt
sudo chmod 600 /data/nexus/certs/kvsh.key
docker exec nexus-nginx nginx -t && docker exec nexus-nginx nginx -s reload
```

---

## ۵. خطای اول: مرورگر OK، Docker نه

### پیام

```text
docker pull repo-nexus.kavosh.org/nginx
tls: failed to verify certificate: x509: certificate signed by unknown authority
```

### علت

nginx فقط **یک** سرتیفیکیت می‌فرستاد: leaf دامنه. صادرکننده‌اش `Certum DV TLS G2 R39 CA` بود، ولی خود این CA میانی در پاسخ TLS نبود.

- مرورگر با AIA intermediate را از سایت Certum گرفت و صفحه سبز شد.
- Docker (کتابخانهٔ TLS زبان Go) AIA نمی‌گیرد → «unknown authority».

تأیید از بیرون:

```bash
echo | openssl s_client -connect repo-nexus.kavosh.org:443 \
  -servername repo-nexus.kavosh.org -showcerts 2>/dev/null \
  | grep -E 'BEGIN CERTIFICATE|^ [0-9] s:|Verify return code'
```

آن موقع خروجی فقط **یک** `BEGIN CERTIFICATE` بود و `Verify return code` غیر صفر.

### درست شدن

intermediate رسمی را از AIA همان سرتیفیکیت گرفتیم و به leaf چسباندیم:

```text
http://certumdvtlsg2r39ca.repository.certum.pl/certumdvtlsg2r39ca.cer
```

```bash
openssl x509 -in certum-kvsh2027.cer -inform DER -out certum-kvsh2027.pem
cat server2027.crt certum-kvsh2027.pem > kvsh2027.crt
```

بعد **حتماً** این فایل را روی `/data/nexus/certs/kvsh.crt` گذاشتیم و nginx reload شد. تا قبل از این کپی، فایل محلی درست بود ولی سرور هنوز leaf-only سرو می‌کرد.

بعد از reload باید **دو** سرتیفیکیت در زنجیره دیده شود و روی لینوکس جدید:

```text
Verify return code: 0 (ok)
```

از این جا به بعد `docker pull` روی ماشینی که Docker و CA bundle به‌روز داشت درست شد.

---

## ۶. خطای دوم: Docker OK، پاد کوبرنتیز نه

### پیام (از `kubectl describe po nginx -n cp-tools-dev`)

```text
Failed to resolve on node esx-r13-03.kavosh.org
Head "https://repo-nexus.kavosh.org/v2/nginx/manifests/latest"
tls: failed to verify certificate: x509: certificate signed by unknown authority
```

مانیفست تست:

```yaml
image: repo-nexus.kavosh.org/nginx
```

مسیر فایل: `/data/Project/Docs/Kuber/Test/nginx.yml`

### علت — این‌بار nginx مقصر نبود

زنجیره روی سرور کامل بود (۲ سرتیفیکیت، verify روی لینوکس صفر). مشکل از **کلاینت** بود.

این کلاستر **vSphere Supervisor** است، نه نود لینوکسی معمولی:

- پاد به نودهایی با اسم ESXi زمان‌بندی می‌شود (`esx-r12-08.kavosh.org` و مشابه).
- کامپوننت `image-controller` ایمیج را **روی خود ESXi** از رجیستری می‌کشد، نه با Docker روی لینوکس.
- trust store وی‌ام‌ور روت **`Certum Trusted Network CA`** (روت قدیمی) را دارد.
- روت جدید **`Certum Trusted Root CA`** را ندارد.

پس لینوکس می‌تواند زنجیرهٔ جدید را تا روت جدید تمام کند؛ ESXi به دیوار می‌خورد: intermediate را می‌بیند، ولی صادرکنندهٔ آن را trust ندارد.

اضافه کردن CA به vCenter از نظر امنیتی «تغییر روی هایپروایزرهای پروداکشن» است و در این سناریو لازم نبود.

### چرا سرتیفیکیت قدیمی را قاطی نکردیم؟

- leaf قدیمی **منقضی** شده بود.
- intermediate قدیمی (`Domain Validation CA SHA2`) سرتیفیکیت جدید را امضا نکرده؛ چسباندنش به leaf جدید از نظر رمزنگاری بی‌معنی است و verify را خراب می‌کند.

چیزی که از دنیای قدیم هنوز باارزش بود، **خود روت قدیمی روی ESXi** بود، نه فایل crt منقضی.

---

## ۷. راه‌حل نهایی: cross-certificate (فقط nginx)

Certum برای مهاجرت روت، یک سرتیفیکیت واسط منتشر کرده:

```text
https://repository.certum.pl/ctnca-ctrca.pem
```

این فایل می‌گوید:

```text
subject = Certum Trusted Root CA          ← همان روت جدید
issuer  = Certum Trusted Network CA       ← همان روت قدیمی که ESXi دارد
اعتبار  ≈ ۲۰۲۳ تا ۲۰۲۸
```

یعنی روت جدید را با کلید روت قدیمی امضا کرده‌اند. اگر nginx این فایل را **بعد از** intermediate بفرستد، زنجیره برای ESXi این می‌شود:

```text
*.kavosh.org
    → Certum DV TLS G2 R39 CA
        → Certum Trusted Root CA   (نسخهٔ cross-sign، نه self-signed)
            → Certum Trusted Network CA   ← داخل trust store وی‌ام‌ور
```

کلاینت‌های جدید همان‌طور که قبلاً روت جدید را مستقیم trust داشتند، همچنان OK می‌مانند. **هیچ تغییری روی vCenter/ESXi لازم نیست.** فقط `kvsh.crt` سه تکه می‌شود.

فایل آماده در مخزن:

```text
Cert/kvsh2027-esxi.crt
```

ترتیب داخلش:

1. `server2027.crt` — دامنه
2. `certum-kvsh2027.pem` — CA میانی R39
3. `ctnca-ctrca.pem` — cross-sign

ساخت مجدد اگر لازم شد:

```bash
curl -fsSL -o Cert/ctnca-ctrca.pem https://repository.certum.pl/ctnca-ctrca.pem
cat Cert/server2027.crt Cert/certum-kvsh2027.pem Cert/ctnca-ctrca.pem \
    > Cert/kvsh2027-esxi.crt
grep -c 'BEGIN CERTIFICATE' Cert/kvsh2027-esxi.crt   # باید 3 باشد
```

اعمال روی Nexus (کلید را عوض نکن):

```bash
scp -P22 Cert/kvsh2027-esxi.crt ubuntu@repo-nexus.kavosh.org:~/cert-kvsh2027/

# روی سرور:
grep -c 'BEGIN CERTIFICATE' ~/cert-kvsh2027/kvsh2027-esxi.crt
sudo cp /data/nexus/certs/kvsh.crt /data/nexus/certs/kvsh.crt.bak
sudo cp ~/cert-kvsh2027/kvsh2027-esxi.crt /data/nexus/certs/kvsh.crt
sudo chmod 644 /data/nexus/certs/kvsh.crt
docker exec nexus-nginx nginx -t && docker exec nexus-nginx nginx -s reload
```

تأیید:

```bash
echo | openssl s_client -connect repo-nexus.kavosh.org:443 \
  -servername repo-nexus.kavosh.org -showcerts 2>/dev/null \
  | grep -E 'BEGIN CERTIFICATE|^ [0-9] s:|^   i:|Verify return code'
```

باید سه سرتیفیکیت و `Verify return code: 0 (ok)` باشد.

بعد پاد:

```bash
kubectl delete po nginx -n cp-tools-dev
kubectl apply -f /data/Project/Docs/Kuber/Test/nginx.yml
kubectl describe po nginx -n cp-tools-dev
```

اسم پاد `nginx` است، نه `nginx.yml`.

---

## ۸. نقشهٔ خطاها (برای دفعهٔ بعد)

```text
مرورگر خراب، openssl هم خراب
    → فایل روی nginx اشتباه است، یا کلید جفت نیست، یا reload نشده

مرورگر OK ، openssl تعداد cert = 1 ، Docker خراب
    → زنجیره ناقص است؛ intermediate را بچسبان
    → یادت باشد فایل را روی /data/nexus/certs/kvsh.crt بگذاری

openssl روی لینوکس Verify 0 ، Docker لینوکس OK ، کوبر/ESXi خراب
    → روت جدید در trust store وی‌ام‌ور نیست
    → یا cross-sign به nginx اضافه کن (کم‌خطر)
    → یا روت را در vCenter Trusted Root Certificates بگذار (تغییر زیرساخت)

همه TLS سبز، باز هم ErrImagePull
    → اسم ایمیج/پورت/ریپوی Docker اشتباه است (مثلاً :8110)، نه سرتیفیکیت
```

---

## ۹. فایل‌های پوشهٔ `Cert/`

| فایل | چیست |
| --- | --- |
| `server2027.crt` | فقط سرتیفیکیت دامنه (leaf) |
| `certum-kvsh2027.pem` | intermediate جدید (R39)، نسخهٔ PEM |
| `certum-kvsh2027.cer` | همان intermediate، DER خام از Certum |
| `ctnca-ctrca.pem` | cross-sign روت جدید با روت قدیمی |
| `kvsh2027.crt` | leaf + R39 (برای کلاینت لینوکس/مرورگر کافی بود) |
| `kvsh2027-esxi.crt` | leaf + R39 + cross-sign — **فایل نهایی nginx** |
| `kvsh2027.crt.leaf-only.bak` | بکاپ leaf تنها |
| `ca-chain2027.crt` | خروجی خالی `-cacerts` از PFX؛ به آن اعتماد نکن |

کلید خصوصی (`kvsh2027.key`) نباید داخل این پوشه در git باشد.

---

## ۱۰. کارهایی که نباید تکرار شوند

- گذاشتن passphrase روی کلید nginx.
- فرستادن روت self-signed به‌جای cross-sign؛ کلاینت روت را از store خودش می‌خواهد، نه لزوماً از سرور.
- چسباندن زنجیرهٔ سرتیفیکیت **منقضی** به سرتیفیکیت جدید.
- `insecure-registries` یا خاموش کردن verify روی کلاستر، وقتی با یک فایل crt روی nginx حل می‌شود.
- آپلود CA روی همهٔ ESXi وقتی Certum خودش پل (cross-sign) داده است.
- فراموش کردن reload بعد از کپی فایل؛ `nginx -t` را قبل از reload بزن.

---

## ۱۱. منبع Certum دربارهٔ روت جدید

Certum رسماً گفته روت DV از `Trusted Network CA` به `Trusted Root CA` عوض شده و برای سیستم‌های قدیمی باید cross-certificate روی **سرور** نصب شود، نه روی هر کلاینت:

- توضیح: [Certum implements new Root CAs](https://support.certum.eu/en/technical-news/certum-implements-new-root-cas/)
- فایل پل: `https://repository.certum.pl/ctnca-ctrca.pem`

</div>
