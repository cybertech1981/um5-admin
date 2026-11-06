# پنل ادمین UM5

![UM5 Admin](docs/banner.png)

**🌐 زبان‌ها:** [English](README.md) · فارسی · [Türkçe](README.tr.md)  
**📖 راهنمای نصب:** [English](INSTALL.md) · [فارسی](INSTALL-FA.md) · [Türkçe](INSTALL-TR.md)

<div dir="rtl">

پنل مدیریت **چندزبانه (فارسی / انگلیسی / ترکی)** برای **MikroTik User Manager
نسخهٔ ۵**. به‌صورت یک کانتینر کوچک (یک فایل اجرایی حدود ۸ مگابایت، بدون سیستم‌عامل)
اجرا می‌شود و روتر را از طریق **REST API** مدیریت می‌کند: ساخت کاربر و ووچر، تمدید
بسته، مشاهدهٔ اتصال‌ها، چاپ کارت، و گزارش مصرف و فروش.

ساختهٔ [Amir Echo Tech](https://amirechotech.ca/).

## محتوای این مخزن

```
INSTALL.md / -FA / -TR   اجرا داخل RouterOS به‌صورت کانتینر (انگلیسی / فارسی / ترکی)
docker-compose.yml       اجرا روی یک هاست Docker معمولی
TERMS.md                 شرایط استفاده
docs/                    اسکرین‌شات‌ها
```

خودِ ایمیج کانتینر در مخزن نیست — بخش «دریافت ایمیج» را ببینید.

## تصاویر

| فارسی (راست‌چین) | انگلیسی |
|---|---|
| ![داشبورد — فارسی](docs/admin-fa.png) | ![داشبورد — انگلیسی](docs/admin-en.png) |

| ترکی | انگلیسی — حالت تیره |
|---|---|
| ![داشبورد — ترکی](docs/admin-tr.png) | ![حالت تیره](docs/admin-en-dark.png) |

<img src="docs/admin-tr-mobile.png" width="260" alt="موبایل — ترکی">

## دریافت ایمیج

ایمیج کانتینر **در این مخزن نیست** (فایل باینری است). از این‌جا بگیرید:

- بخش **Releases** ← فایل `um5-admin-amd64.tar`، سپس `docker load -i um5-admin-amd64.tar`
  یا آپلود در **Files** روتر و دنبال‌کردن [INSTALL-FA.md](INSTALL-FA.md).
- یا از **GitHub Container Registry**:
  `docker pull ghcr.io/cybertech1981/um5-admin:1.2.0`

## اجرا داخل RouterOS (پیشنهادی)

نصب در ۷ گام زیر است. راهنمای کامل با توضیح هر خط، HTTPS، دسترسی از بیرون و
عیب‌یابی در **[INSTALL-FA.md](INSTALL-FA.md)** است.

**۱. فعال‌کردن کانتینر (یک‌بار).** پکیج `container` نصب باشد، بعد:
```
/system/device-mode/update container=yes
```
با **ریبوت سرد** (Power Off/On روی CHR) تأیید کنید.

**۲. کاربر API محدود** (پنل با این وصل می‌شود، نه admin):
```
/user/group/add name=um5-api policy=read,write,api,rest-api
/user/add name=um5-panel group=um5-api address=172.17.0.0/24 password="API-PASS"
```

**۳. شبکهٔ کانتینر:**
```
/interface/veth/add name=veth-um5 address=172.17.0.2/24 gateway=172.17.0.1
/interface/bridge/add name=containers
/ip/address/add address=172.17.0.1/24 interface=containers
/interface/bridge/port/add bridge=containers interface=veth-um5
/ip/firewall/nat/add chain=srcnat src-address=172.17.0.0/24 action=masquerade
```

**۴. متغیرها (Environment):**
```
/container/envs/add list=um5 key=ROUTER_URL  value="http://172.17.0.1"
/container/envs/add list=um5 key=ROUTER_USER value="um5-panel"
/container/envs/add list=um5 key=ROUTER_PASS value="API-PASS"
/container/envs/add list=um5 key=ADMIN_USER  value="admin"
/container/envs/add list=um5 key=ADMIN_PASS  value="ADMIN-PASS-8+"
/container/envs/add list=um5 key=TZ          value="Asia/Tehran"
/container/envs/add list=um5 key=CURRENCY    value="Toman"
```

**۵. مانت داده (اپراتورها/ووچرها/رویدادها اینجا می‌مانند):**
```
/container/mounts/add name=um5-data src=um5/data dst=/data
```

**۶. ساخت کانتینر از ghcr.io:**
```
/container/config/set registry-url=https://ghcr.io tmpdir=um5/tmp
/container/add remote-image=ghcr.io/cybertech1981/um5-admin:1.2.0 interface=veth-um5 root-dir=um5/root mountlists=um5-data envlists=um5 name=um5-admin start-on-boot=yes logging=yes
```
(یا به‌جای `remote-image`، فایل `um5-admin-amd64.tar` از Releases را با `file=...` بدهید.)

**۷. اجرا و بررسی:**
```
/container/print            ;# صبر تا وضعیت = stopped
/container/start um5-admin
/log/print where topics~"container"   ;# باید «router OK» ببینید
```
با `ADMIN_USER` / `ADMIN_PASS` وارد شوید. بعد رمز مدیر را در پنل عوض و
`ADMIN_PASS` را حذف کنید:
`/container/envs/remove [find list=um5 key=ADMIN_PASS]`

## اجرا روی هاست Docker

```bash
docker pull ghcr.io/cybertech1981/um5-admin:1.2.0
# در docker-compose.yml مقادیر ROUTER_URL / ROUTER_USER / ROUTER_PASS / ADMIN_PASS را بگذارید
docker compose up -d
# آدرس http://<host>:8080 را باز کنید
```

بعد از اولین ورود، رمز مدیر را در پنل عوض کنید و `ADMIN_USER` و `ADMIN_PASS` را
از compose حذف کنید.

## کاربر API روتر

برای پنل یک کاربر **محدود** بسازید (هرگز `admin` نه):

```
/user/group/add name=um5-api policy=read,write,api,rest-api
/user/add name=um5-panel group=um5-api address=<شبکهٔ-کانتینر> password="رمز-طولانی-تصادفی"
```

## نقش‌ها

- **مدیر**: همه‌چیز، از جمله حذف کاربر، اپراتورها، بستن جلسه و رویدادها.
- **فروشنده**: دیدن کاربران، ساخت کاربر و ووچر، تمدید، تغییر رمز و غیرفعال‌کردن.

## لایسنس

ایمیج باینری توزیع‌شده. **[TERMS.md](TERMS.md)** را ببینید. استفاده رایگان است؛
سورس منتشر نشده و نام سازنده نباید حذف شود. برای مجوز انتشار مجدد تجاری یا تغییر
برند با [Amir Echo Tech](https://amirechotech.ca/) تماس بگیرید.

**[Amir Echo Tech](https://amirechotech.ca/)** ·
[لینکدین](https://www.linkedin.com/in/amir1360/) ·
[تلگرام](https://t.me/AmirEchoTech)

</div>
