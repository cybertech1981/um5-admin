# نصب UM5 Admin روی میکروتیک (کانتینر)

🌐 **راهنمای نصب:** [English](INSTALL.md) · فارسی · [Türkçe](INSTALL-TR.md)

پنل مدیریت سه‌زبانه (فارسی / English / Türkçe) برای MikroTik User Manager v5.
برنامه یک فایل اجرایی Go است (بدون سیستم‌عامل داخل ایمیج، حدود ۸ مگابایت) و فقط از راه **REST API** با خود روتر حرف می‌زند.

```
مرورگر ──► روتر (dst-nat :8443) ──► کانتینر 172.17.0.2:8080 ──REST──► 172.17.0.1 (همین روتر)
```

**پیش‌نیازها:** RouterOS v7 با پکیج `container`، معماری arm/arm64/x86. روی CHR x86_64 و RouterOS 7.23 تست شده.

> ⚠️ همهٔ دستورها را در **New Terminal** بزنید، نه از فرم گرافیکی. فرم «New Container» در بعضی نسخه‌های WinBox با هر مقداری خطای `Error in Shm Size - expected!` می‌دهد.

> در این راهنما شبکهٔ `172.17.0.0/24` فرض شده (روتر `172.17.0.1`، کانتینر `172.17.0.2`). اگر شبکهٔ دیگری می‌خواهید، همین اعداد را همه‌جا یکدست عوض کنید.

---

## ۱) فعال‌کردن کانتینر (یک‌بار برای همیشه)

```
/system/package/print
```
اگر `container` در فهرست نیست: از mikrotik.com فایل **Extra packages** هم‌نسخهٔ روترتان را بگیرید، `container-x.y.npk` را در **Files** آپلود و روتر را ریبوت کنید.

بعد حالت کانتینر را روشن کنید:
```
/system/device-mode/update container=yes
```
این دستور منتظر **تأیید فیزیکی** می‌ماند:
- روی دستگاه سخت‌افزاری: دکمهٔ reset را بزنید یا برق را قطع و وصل کنید.
- روی CHR/VMware: ماشین را **Power Off** و دوباره روشن کنید (ریبوت نرم کافی نیست).

بررسی: `/system/device-mode/print` باید `container: yes` نشان دهد.

---

## ۲) کاربر API مخصوص پنل

پنل با یک کاربر **محدود** به روتر وصل می‌شود، نه با `admin`. این کاربر فقط از شبکهٔ کانتینر اجازهٔ ورود دارد:
```
/user/group/add name=um5-api policy=read,write,api,rest-api comment="UM5 Admin panel"
/user/add name=um5-panel group=um5-api address=172.17.0.0/24 password="یک-رمز-قوی-و-تصادفی"
```
سرویس `www` هم باید روشن باشد (پورتال `/um/` و REST از همین سرویس استفاده می‌کنند):
```
/ip/service/print where name=www
```

---

## ۳) شبکهٔ کانتینر

```
/interface/veth/add name=veth-um5 address=172.17.0.2/24 gateway=172.17.0.1
/interface/bridge/add name=containers
/ip/address/add address=172.17.0.1/24 interface=containers
/interface/bridge/port/add bridge=containers interface=veth-um5
```

سه چیز باید با هم جور باشند، وگرنه در لاگ خطای `no route to host` می‌بینید:
- `gateway` در veth = آدرس IP روتر = `172.17.0.1`
- آن آدرس روی **همان bridge** باشد (نه روی خود veth)
- `ROUTER_URL` در مرحلهٔ ۴ = `http://172.17.0.1`

اگر فایروال `input` به‌طور پیش‌فرض drop می‌کند، اجازهٔ دسترسی کانتینر به REST را بدهید:
```
/ip/firewall/filter/add chain=input src-address=172.17.0.0/24 protocol=tcp dst-port=80 action=accept comment="UM5 panel to REST" place-before=0
```

> اگر ایمیج را از ghcr.io می‌کشید (مرحلهٔ ۶، روش الف)، کانتینر برای **کشیدن ایمیج** به اینترنت و DNS نیاز دارد. اگر veth مسیر خروجی به اینترنت ندارد، یک قانون NAT بدهید:
> ```
> /ip/firewall/nat/add chain=srcnat src-address=172.17.0.0/24 action=masquerade comment="containers out"
> ```
> بعد از نصب، اگر پنل را از بیرون باز نمی‌کنید، این قانون لازم نیست و می‌توانید حذفش کنید.

---

## ۴) تعریف متغیرها (Environment)

همهٔ متغیرها در یک «لیست» به نام `um5` تعریف می‌شوند. مقادیر داخل گیومه را عوض کنید.
`ADMIN_USER` **نام کاربری** و `ADMIN_PASS` **رمز** مدیر اول است؛ هر دو باید قبل از اولین اجرا باشند.

```
/container/envs/add list=um5 key=ROUTER_URL  value="http://172.17.0.1"
/container/envs/add list=um5 key=ROUTER_USER value="um5-panel"
/container/envs/add list=um5 key=ROUTER_PASS value="همان-رمز-مرحلهٔ-۲"
/container/envs/add list=um5 key=ADMIN_USER  value="admin"
/container/envs/add list=um5 key=ADMIN_PASS  value="رمز-مدیر-حداقل-۸-کاراکتر"
/container/envs/add list=um5 key=TZ          value="Asia/Tehran"
/container/envs/add list=um5 key=CURRENCY    value="Toman"
/container/envs/add list=um5 key=PORTAL_URL  value="https://دامنهٔ-شما/um/"
```
بررسی: `/container/envs/print where list=um5`
(فهرست کامل متغیرها در جدول انتهای همین صفحه.)

---

## ۵) تعریف مانت داده و پوشهٔ موقت

داده‌های **خود پنل** (اپراتورها، ووچرها، فروش، رویدادها) باید روی یک مانت بمانند تا با به‌روزرسانی یا حذف کانتینر پاک نشوند. داده‌های **کاربران** همیشه روی خود User Manager است، نه اینجا.

```
/container/mounts/add name=um5-data src=um5/data dst=/data
/container/config/set tmpdir=um5/tmp
```
- `src=um5/data` = پوشه روی حافظهٔ روتر (خودش ساخته می‌شود)؛ `dst=/data` = مسیر داخل کانتینر.
- `tmpdir` = جایی که RouterOS موقع کشیدن/باز کردن ایمیج از آن استفاده می‌کند.

> اگر حافظهٔ خارجی (USB/NVMe) دارید، به‌جای `um5/...` از مسیر آن دیسک استفاده کنید؛ میکروتیک توصیه می‌کند کانتینر روی حافظهٔ خارجی باشد. روی CHR با دیسک مجازی کافی، همین حافظهٔ داخلی هم اوکی است.

---

## ۶) دریافت ایمیج و ساخت کانتینر

**فقط یکی از دو روش را انتخاب کنید.**

### روش الف) کشیدن مستقیم از ghcr.io (پیشنهادی — بدون آپلود فایل)
ایمیج عمومی است و به لاگین نیاز ندارد. روتر باید اینترنت و DNS داشته باشد.
```
/container/config/set registry-url=https://ghcr.io tmpdir=um5/tmp
/container/add remote-image=ghcr.io/cybertech1981/um5-admin:1.2.0 \
  interface=veth-um5 root-dir=um5/root mountlists=um5-data envlists=um5 \
  name=um5-admin start-on-boot=yes logging=yes
```

### روش ب) آپلود فایل
فایل `um5-admin-amd64.tar` را از بخش **Releases** مخزن بگیرید، در **Files** آپلود کنید، بعد:
```
/container/add file=um5-admin-amd64.tar \
  interface=veth-um5 root-dir=um5/root mountlists=um5-data envlists=um5 \
  name=um5-admin start-on-boot=yes logging=yes
```

نکته‌ها:
- در RouterOS قدیمی‌تر از 7.20 به‌جای `envlists`/`mountlists` بنویسید `envlist`/`mountlist`.
- `root-dir=um5/root` پوشه‌ای است که ایمیج آنجا باز می‌شود. موقع **به‌روزرسانی** یک اسم تازه بدهید (مثلاً `um5/root2`) تا با فایل‌های قبلی قاطی نشود.

---

## ۷) اجرا و بررسی

```
/container/print
```
صبر کنید تا وضعیت از `extracting`/`downloading` به **`stopped`** برسد، بعد:
```
/container/start um5-admin
/log/print where topics~"container"
```
در لاگ باید این سه خط را ببینید:
```
created first admin "admin"
UM5 Admin 1.2.0 listening on http://:8080
router OK: RouterOS 7.23 ...
```
اگر `router OK` دیدید، پنل آماده است. آدرس موقت برای تست از داخل شبکه: `http://172.17.0.2:8080`

با `ADMIN_USER` / `ADMIN_PASS` وارد شوید.

---

## ۸) بعد از اولین ورود (امنیت — مهم)

1. رمز مدیر را از منوی بالای پنل (اسم شما ← «تغییر رمز من») عوض کنید.
2. متغیر `ADMIN_PASS` را پاک کنید (دیگر لازم نیست):
   ```
   /container/envs/remove [find list=um5 key=ADMIN_PASS]
   ```
3. RouterOS هنگام `start` همهٔ متغیرها — **از جمله رمزها** — را به‌صورت متن ساده در لاگ می‌نویسد. برای `ROUTER_PASS` یک رمز بلند و تصادفی بگذارید که جای دیگری استفاده نمی‌شود، و لاگ/اسکرین‌شات‌ها را پیش کسی نگذارید.

---

## ۹) HTTPS (پیشنهاد اکید)

پنل گواهی را از پوشهٔ `/data` می‌خواند و اگر فایل عوض شود، بدون ری‌استارت گواهی جدید را برمی‌دارد.

۱. اسکریپت خروجی‌گرفتن از گواهی روتر (نام گواهی را از `/certificate/print` بردارید):
```
/system/script/add name=um5-cert source={
  :do { /file/remove [find where name~"um5/data/tls"] } on-error={}
  /certificate/export-certificate "نام-گواهی" type=pem export-passphrase="رمز-کلید" file-name=um5/data/tls
}
/system/script/run um5-cert
/system/scheduler/add name=um5-cert interval=7d on-event=um5-cert
```
۲. متغیرهای TLS و ری‌استارت:
```
/container/envs/add list=um5 key=TLS_CERT     value="/data/tls.crt"
/container/envs/add list=um5 key=TLS_KEY      value="/data/tls.key"
/container/envs/add list=um5 key=TLS_KEY_PASS value="رمز-کلید"
/container/stop um5-admin
/container/start um5-admin
```

---

## ۱۰) دسترسی از بیرون

```
/ip/firewall/nat/add chain=dstnat protocol=tcp dst-port=8443 in-interface-list=WAN action=dst-nat to-addresses=172.17.0.2 to-ports=8080 comment="UM5 Admin"
```
اگر زنجیرهٔ `forward` پیش‌فرض drop می‌کند، یک accept با `connection-nat-state=dstnat` لازم است. برای امنیت بیشتر دسترسی را به IPهای مشخص محدود کنید (`src-address-list=`).
آدرس پنل: `https://دامنهٔ-شما:8443`

---

## به‌روزرسانی به نسخهٔ جدید

```
/container/stop um5-admin
/container/remove [find name=um5-admin]
```
بعد مرحلهٔ ۶ را با یک `root-dir` تازه (مثلاً `um5/root2`) دوباره اجرا کنید. چون داده روی مانت `um5-data` است، اپراتورها/ووچرها/رویدادها می‌مانند و `ADMIN_PASS` دیگر لازم نیست.
با روش ghcr کافی است شمارهٔ نسخه در `remote-image=...:1.2.1` را عوض کنید.

---

## عیب‌یابی

| پیام در لاگ | علت و راه‌حل |
|---|---|
| `Error in Shm Size - expected!` | باگ فرم WinBox؛ کانتینر را از **ترمینال** بسازید. |
| `layer ... missing, need repull` | فایل tar با فرمت قدیمی docker save؛ از `um5-admin-amd64.tar` این مخزن (OCI) استفاده کنید، یا از روش ghcr. |
| `first start: set ADMIN_PASS` | `ADMIN_PASS` در لیست نیست یا `envlists=um5` به کانتینر وصل نشده (`/container/print detail`). |
| `no route to host` | gateway در veth، IP روی bridge و `ROUTER_URL` با هم جور نیستند (مرحلهٔ ۳). |
| `router rejected the API login` | نام/رمز کاربر API، policy گروه (`rest-api`) یا `address` کاربر API را چک کنید (مرحلهٔ ۲). |
| هنگام pull: `dial tcp ... no route` یا `lookup ghcr.io` | کانتینر به اینترنت/DNS نمی‌رسد؛ قانون masquerade مرحلهٔ ۳ و `/ip/dns` را چک کنید. |
| `/container/shell` کار نمی‌کند | عادی است؛ ایمیج عمداً shell ندارد. از `/log/print` استفاده کنید. |

---

## جدول کامل متغیرها

| متغیر | پیش‌فرض | توضیح |
|---|---|---|
| `ROUTER_URL` | `http://172.17.0.1` | آدرس روتر از داخل کانتینر |
| `ROUTER_USER` / `ROUTER_PASS` | — | کاربر API مرحلهٔ ۲ (اجباری) |
| `ROUTER_INSECURE_TLS` | `true` | اگر `ROUTER_URL` با https و گواهی معتبر است، `false` کنید |
| `ADMIN_USER` / `ADMIN_PASS` | `admin` / — | فقط برای ساخت اولین مدیر |
| `LISTEN` | `:8080` | پورت داخلی پنل |
| `DATA_DIR` | `/data` | پوشهٔ دادهٔ پنل (همان مانت مرحلهٔ ۵) |
| `TLS_CERT` / `TLS_KEY` / `TLS_KEY_PASS` | — | فعال‌کردن HTTPS |
| `PORTAL_URL` | آدرس روتر + `/um/` | آدرسی که روی کارت ووچر چاپ می‌شود |
| `CURRENCY` | — | واحد پول کنار قیمت‌ها (مثلاً `Toman`) |
| `TZ` | `Asia/Tehran` | منطقهٔ زمانی |

---

## نقش‌ها

- **مدیر (admin):** همه‌چیز، از جمله حذف کاربر، مدیریت اپراتورها، بستن جلسه و دیدن رویدادها.
- **فروشنده (seller):** دیدن کاربران، ساخت کاربر و ووچر، تمدید بسته، تغییر رمز و غیرفعال‌کردن کاربر.

---

## خلاصهٔ کل نصب (کپی یک‌جا، روش ghcr)

مقادیر گیومه‌دار و شبکه را مطابق خودتان تنظیم کنید؛ بعد کل بلوک را در ترمینال بچسبانید. (مرحلهٔ `device-mode` و ریبوت سرد باید از قبل انجام شده باشد.)

```
/user/group/add name=um5-api policy=read,write,api,rest-api comment="UM5 Admin panel"
/user/add name=um5-panel group=um5-api address=172.17.0.0/24 password="API-PASS"

/interface/veth/add name=veth-um5 address=172.17.0.2/24 gateway=172.17.0.1
/interface/bridge/add name=containers
/ip/address/add address=172.17.0.1/24 interface=containers
/interface/bridge/port/add bridge=containers interface=veth-um5
/ip/firewall/nat/add chain=srcnat src-address=172.17.0.0/24 action=masquerade comment="containers out"

/container/envs/add list=um5 key=ROUTER_URL  value="http://172.17.0.1"
/container/envs/add list=um5 key=ROUTER_USER value="um5-panel"
/container/envs/add list=um5 key=ROUTER_PASS value="API-PASS"
/container/envs/add list=um5 key=ADMIN_USER  value="admin"
/container/envs/add list=um5 key=ADMIN_PASS  value="ADMIN-PASS-8+"
/container/envs/add list=um5 key=TZ          value="Asia/Tehran"
/container/envs/add list=um5 key=CURRENCY    value="Toman"

/container/mounts/add name=um5-data src=um5/data dst=/data
/container/config/set registry-url=https://ghcr.io tmpdir=um5/tmp
/container/add remote-image=ghcr.io/cybertech1981/um5-admin:1.2.0 interface=veth-um5 root-dir=um5/root mountlists=um5-data envlists=um5 name=um5-admin start-on-boot=yes logging=yes
```
بعد صبر تا `stopped`، سپس `/container/start um5-admin` و `/log/print where topics~"container"`.

---

## سازنده

طراحی و توسعه: امیر ماکوئی — [Amir Echo Tech](https://amirechotech.ca/) · [LinkedIn](https://www.linkedin.com/in/amir1360/) · [Telegram](https://t.me/AmirEchoTech)
