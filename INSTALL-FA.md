# نصب UM5 Admin روی میکروتیک (کانتینر)

پنل مدیریت سه‌زبانه (فارسی / English / Türkçe) برای User Manager v5.
برنامه یک فایل اجرایی Go است (بدون سیستم‌عامل داخل ایمیج، حدود ۸ مگابایت) و فقط از راه REST API با خود روتر حرف می‌زند.

```
مرورگر ──► روتر (dst-nat :8443) ──► کانتینر 172.17.0.2:8080 ──REST──► 172.17.0.1 (همین روتر)
```

فایل ایمیج برای CHR / x86_64: `dist/um5-admin-amd64.tar` (فرمت OCI؛ روی RouterOS 7.23 تست شده)

> **نکته:** همهٔ دستورها را در **ترمینال** بزنید. فرم «New Container» در بعضی نسخه‌های WinBox با هر مقداری خطای `Error in Shm Size - expected!` می‌دهد.

---

## ۱. فعال‌کردن کانتینر (یک بار)

۱. پکیج container باید نصب باشد:
```
/system/package/print
```
اگر `container` در فهرست نیست، از mikrotik.com فایل «Extra packages» نسخهٔ 7.23 برای x86 را بگیرید، `container-7.23.npk` را در Files آپلود و روتر را ریبوت کنید.

۲. حالت کانتینر:
```
/system/device-mode/update container=yes
```
این دستور منتظر تأیید می‌ماند. روی CHR تأیید با **ریبوت سرد** است: ماشین مجازی را از VMware خاموش (Power Off) و دوباره روشن کنید. ریبوت معمولی کافی نیست.

---

## ۲. کاربر API مخصوص پنل

پنل با یک کاربر محدود به روتر وصل می‌شود، نه با admin. این کاربر فقط از شبکهٔ کانتینر اجازهٔ ورود دارد.
```
/user/group/add name=um5-api policy=read,write,api,rest-api comment="UM5 Admin panel"
/user/add name=um5-panel group=um5-api address=172.17.0.0/24 password="یک-رمز-قوی-و-طولانی"
```
سرویس `www` باید روشن باشد (پورتال `/um/` هم از همین سرویس استفاده می‌کند).

---

## ۳. شبکهٔ کانتینر
```
/interface/veth/add name=veth-um5 address=172.17.0.2/24 gateway=172.17.0.1
/interface/bridge/add name=containers
/ip/address/add address=172.17.0.1/24 interface=containers
/interface/bridge/port/add bridge=containers interface=veth-um5
```
سه چیز باید با هم جور باشد، وگرنه در لاگ خطای `no route to host` می‌بینید:
- `gateway` در veth
- آدرس IP روتر روی **همان bridge** (نه روی خود veth)
- `ROUTER_URL` در مرحلهٔ ۴

اگر شبکهٔ دیگری انتخاب کردید (مثلاً `192.168.151.0/24`)، هر سه را و `address` کاربر API در مرحلهٔ ۲ را با همان شبکه بنویسید.

اگر فایروال input به‌طور پیش‌فرض drop می‌کند، این قانون را بالای dropها بگذارید:
```
/ip/firewall/filter/add chain=input src-address=172.17.0.0/24 protocol=tcp dst-port=80 action=accept comment="UM5 panel to REST" place-before=0
```
پنل به اینترنت نیاز ندارد، پس قانون masquerade لازم نیست.

---

## ۴. دریافت ایمیج و تنظیمات

**ایمیج را به دو روش می‌توانید بدهید — فقط یکی را انتخاب کنید:**

**روش الف) کشیدن مستقیم از ghcr.io (ساده‌تر، نیازی به آپلود فایل نیست).**
روتر باید به اینترنت و DNS دسترسی داشته باشد. در مرحلهٔ ۴/۴ به‌جای `file=...` از `remote-image=` استفاده می‌کنید:
```
/container/config/set registry-url=https://ghcr.io tmpdir=um5/tmp
```
(ایمیج عمومی است و به لاگین نیاز ندارد.)

**روش ب) آپلود فایل.** فایل `um5-admin-amd64.tar` را از بخش Releases مخزن بگیرید و در WinBox به بخش Files بکشید.

۲. متغیرها (مقادیر داخل گیومه را عوض کنید). دقت کنید `ADMIN_USER` **نام کاربری** است و `ADMIN_PASS` **رمز**؛ هر دو باید قبل از اولین اجرا وجود داشته باشند:
```
/container/envs/add list=um5 key=ROUTER_URL  value="http://172.17.0.1"
/container/envs/add list=um5 key=ROUTER_USER value="um5-panel"
/container/envs/add list=um5 key=ROUTER_PASS value="همان-رمز-مرحله-۲"
/container/envs/add list=um5 key=ADMIN_USER  value="admin"
/container/envs/add list=um5 key=ADMIN_PASS  value="رمز-اولین-مدیر-پنل-حداقل-۸-کاراکتر"
/container/envs/add list=um5 key=TZ          value="Asia/Tehran"
/container/envs/add list=um5 key=PORTAL_URL  value="https://دامنه-شما/um/"
```
اختیاری: `CURRENCY` (مثلاً `Toman`) برای نمایش واحد پول کنار قیمت‌ها.

۳. پوشهٔ دادهٔ پنل (اپراتورها، ووچرها، فروش و رویدادها اینجا می‌مانند؛ داده‌های کاربران همیشه روی خود User Manager است):
```
/container/mounts/add list=um5-data src=um5/data dst=/data
/container/config/set tmpdir=um5/tmp
```

۴. ساخت و اجرای کانتینر:
```
# روش ب (فایل آپلودشده):
/container/add file=um5-admin-amd64.tar interface=veth-um5 root-dir=um5/root envlists=um5 mountlists=um5-data name=um5-admin start-on-boot=yes logging=yes
# یا روش الف (کشیدن از ghcr.io؛ به‌جای خط بالا):
/container/add remote-image=ghcr.io/cybertech1981/um5-admin:1.2.0 interface=veth-um5 root-dir=um5/root envlists=um5 mountlists=um5-data name=um5-admin start-on-boot=yes logging=yes
/container/print
```
در نسخه‌های قدیمی‌تر از 7.20 به‌جای `envlists` بنویسید `envlist`.
صبر کنید تا وضعیت از `extracting` به `stopped` برسد، بعد:
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

۵. بعد از اولین ورود:
- رمز مدیر را از منوی بالای پنل عوض کنید.
- متغیر `ADMIN_PASS` را پاک کنید: `/container/envs/remove [find list=um5 key=ADMIN_PASS]`
- RouterOS هنگام start همهٔ متغیرها، از جمله رمزها، را **متن ساده در لاگ** می‌نویسد. از لاگ و اسکرین‌شات‌ها مراقبت کنید و برای `ROUTER_PASS` رمزی بلند و تصادفی بگذارید که جای دیگری استفاده نمی‌شود.

---

## ۵. HTTPS (پیشنهاد اکید)

پنل گواهی را از پوشهٔ `/data` می‌خواند و اگر فایل عوض شود، بدون ری‌استارت گواهی جدید را برمی‌دارد.

۱. اسکریپت خروجی گرفتن از گواهی ACME روتر (نام گواهی را از `/certificate/print` بردارید):
```
/system/script/add name=um5-cert source={
  :do { /file/remove [find where name~"um5/data/tls"] } on-error={}
  /certificate/export-certificate "نام-گواهی" type=pem export-passphrase="یک-رمز-برای-کلید" file-name=um5/data/tls
}
/system/script/run um5-cert
/system/scheduler/add name=um5-cert interval=7d on-event=um5-cert
```

۲. متغیرهای TLS و ری‌استارت کانتینر:
```
/container/envs/add list=um5 key=TLS_CERT     value="/data/tls.crt"
/container/envs/add list=um5 key=TLS_KEY      value="/data/tls.key"
/container/envs/add list=um5 key=TLS_KEY_PASS value="همان-رمز-کلید"
/container/stop um5-admin
/container/start um5-admin
```

---

## ۶. دسترسی از بیرون
```
/ip/firewall/nat/add chain=dstnat protocol=tcp dst-port=8443 in-interface-list=WAN action=dst-nat to-addresses=172.17.0.2 to-ports=8080 comment="UM5 Admin"
```
اگر زنجیرهٔ forward شما به‌طور پیش‌فرض drop می‌کند، یک accept با `connection-nat-state=dstnat` لازم است.
برای امنیت بیشتر، دسترسی را به IPهای مشخص محدود کنید (`src-address-list=`).

آدرس پنل: `https://دامنه-شما:8443`

---

## به‌روزرسانی
```
/container/stop um5-admin
/container/remove [find name=um5-admin]
```
فایل tar جدید را آپلود و دستور `/container/add` مرحلهٔ ۴ را دوباره اجرا کنید. اپراتورها، ووچرها و رویدادها در `um5/data` می‌مانند و `ADMIN_PASS` دیگر لازم نیست.

## عیب‌یابی
| پیام | علت و راه‌حل |
|---|---|
| `Error in Shm Size - expected!` | باگ فرم WinBox؛ کانتینر را از ترمینال بسازید. |
| `layer ... missing, need repull` | فایل tar قدیمی (فرمت docker save قدیم)؛ از `um5-admin-amd64.tar` همین بسته استفاده کنید. |
| `first start: set ADMIN_PASS` | متغیر `ADMIN_PASS` در لیست نیست یا envlists به کانتینر وصل نیست (`/container/print detail`). |
| `no route to host` | gateway در veth، IP روی bridge و `ROUTER_URL` با هم جور نیستند (مرحلهٔ ۳). |
| `router rejected the API login` | نام کاربری/رمز API، policy گروه (`rest-api`) یا `address` کاربر API را بررسی کنید. |
| `container/shell` کار نمی‌کند | عادی است؛ ایمیج عمداً shell ندارد. از لاگ استفاده کنید. |

## نقش‌ها
- **مدیر**: همه‌چیز، از جمله حذف کاربر، مدیریت اپراتورها، بستن جلسه و دیدن رویدادها.
- **فروشنده**: دیدن کاربران، ساخت کاربر و ووچر، تمدید بسته، تغییر رمز و غیرفعال‌کردن کاربر.

## متغیرها
| متغیر | پیش‌فرض | توضیح |
|---|---|---|
| `ROUTER_URL` | `http://172.17.0.1` | آدرس روتر از داخل کانتینر |
| `ROUTER_USER` / `ROUTER_PASS` | — | کاربر API مرحلهٔ ۲ |
| `ROUTER_INSECURE_TLS` | `true` | اگر `ROUTER_URL` با https است و گواهی معتبر دارد، `false` کنید |
| `ADMIN_USER` / `ADMIN_PASS` | `admin` / — | فقط برای ساخت اولین مدیر |
| `LISTEN` | `:8080` | پورت داخلی پنل |
| `DATA_DIR` | `/data` | پوشهٔ داده |
| `TLS_CERT` / `TLS_KEY` / `TLS_KEY_PASS` | — | فعال‌کردن HTTPS |
| `PORTAL_URL` | آدرس روتر + `/um/` | آدرسی که روی کارت ووچر چاپ می‌شود |
| `CURRENCY` | — | واحد پول کنار قیمت‌ها |
| `TZ` | `Asia/Tehran` | منطقهٔ زمانی |

## سازنده
طراحی و توسعه: امیر ماکوئی — [Amir Echo Tech](https://amirechotech.ca/) · [LinkedIn](https://www.linkedin.com/in/amir1360/) · [Telegram](https://t.me/AmirEchoTech)

## ساخت دوباره از سورس
```
docker buildx build --platform linux/amd64 -t um5-admin:1.2.0 --output type=docker .
docker save um5-admin:1.2.0 -o um5-admin-amd64.tar
```
با Docker 25 یا جدیدتر بسازید؛ نسخه‌های قدیمی‌تر فرمت قدیمی می‌سازند که RouterOS 7.23 لایه‌اش را پیدا نمی‌کند.
برای روترهای ARM64 به‌جای `linux/amd64` از `linux/arm64` استفاده کنید.
