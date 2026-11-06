# UM5 Admin'in MikroTik'e kurulumu (container)

🌐 **Kurulum kılavuzu:** [English](INSTALL.md) · [فارسی](INSTALL-FA.md) · Türkçe

MikroTik User Manager v5 için çok dilli (Farsça / İngilizce / Türkçe) yönetim
paneli. Tek bir Go dosyasıdır (~8 MB, imaj içinde işletim sistemi yok) ve router
ile yalnızca **REST API** üzerinden konuşur.

```
tarayıcı ──► router (dst-nat :8443) ──► container 172.17.0.2:8080 ──REST──► 172.17.0.1 (aynı router)
```

**Gereksinimler:** `container` paketli RouterOS v7, arm/arm64/x86 mimari.
CHR x86_64 ve RouterOS 7.23 üzerinde test edildi.

> ⚠️ Her şeyi **New Terminal**'de çalıştırın, GUI formunda değil. Bazı WinBox
> sürümlerinde "New Container" formu her değere `Error in Shm Size - expected!` verir.

> Bu kılavuz `172.17.0.0/24` alt ağını varsayar (router `172.17.0.1`, container
> `172.17.0.2`). Farklı bir alt ağ için bu numaraları tutarlı şekilde değiştirin.

---

## 1) Container'ı etkinleştirme (bir kez)

```
/system/package/print
```
`container` yoksa: mikrotik.com'dan RouterOS sürümünüzle eşleşen **Extra packages**
arşivini indirin, `container-x.y.npk` dosyasını **Files**'a yükleyin, yeniden başlatın.

Sonra container modunu açın:
```
/system/device-mode/update container=yes
```
Bu komut **fiziksel onay** bekler:
- Donanım: reset düğmesine basın ya da gücü kesip açın.
- CHR/VMware: **Power Off** yapıp tekrar başlatın (yazılımsal reboot yetmez).

Doğrulama: `/system/device-mode/print` çıktısı `container: yes` olmalı.

---

## 2) Panele özel API kullanıcısı

Panel routera `admin` ile değil, **kısıtlı** bir kullanıcıyla bağlanır. Bu kullanıcı
yalnızca container alt ağından giriş yapabilir:
```
/user/group/add name=um5-api policy=read,write,api,rest-api comment="UM5 Admin panel"
/user/add name=um5-panel group=um5-api address=172.17.0.0/24 password="guclu-rastgele-sifre"
```
`www` servisi açık olmalı (`/um/` paneli ve REST bunu kullanır):
```
/ip/service/print where name=www
```

---

## 3) Container ağı

```
/interface/veth/add name=veth-um5 address=172.17.0.2/24 gateway=172.17.0.1
/interface/bridge/add name=containers
/ip/address/add address=172.17.0.1/24 interface=containers
/interface/bridge/port/add bridge=containers interface=veth-um5
```

Üç şey uyumlu olmalı, yoksa logda `no route to host` görürsünüz:
- veth `gateway` = routerın IP'si = `172.17.0.1`
- bu IP **bridge** üzerinde olmalı (veth'in kendisinde değil)
- 4. adımdaki `ROUTER_URL` = `http://172.17.0.1`

`input` firewall'unuz varsayılan olarak drop ediyorsa, container'ın REST'e
erişmesine izin verin:
```
/ip/firewall/filter/add chain=input src-address=172.17.0.0/24 protocol=tcp dst-port=80 action=accept comment="UM5 panel to REST" place-before=0
```

> İmajı ghcr.io'dan çekerseniz (adım 6, seçenek A), container **imajı çekmek için**
> internet + DNS ister. veth'in dışa çıkışı yoksa bir NAT kuralı ekleyin:
> ```
> /ip/firewall/nat/add chain=srcnat src-address=172.17.0.0/24 action=masquerade comment="containers out"
> ```
> Paneli dışarıya açmıyorsanız kurulumdan sonra bu kuralı kaldırabilirsiniz.

---

## 4) Ortam değişkenleri (Environment)

Tüm değişkenler `um5` adlı tek bir listeye girer. Tırnak içindeki değerleri
değiştirin. `ADMIN_USER` ilk yöneticinin **kullanıcı adı**, `ADMIN_PASS` ise
**şifresi**dir; ikisi de ilk başlatmadan önce var olmalı.

```
/container/envs/add list=um5 key=ROUTER_URL  value="http://172.17.0.1"
/container/envs/add list=um5 key=ROUTER_USER value="um5-panel"
/container/envs/add list=um5 key=ROUTER_PASS value="adim-2-ile-ayni"
/container/envs/add list=um5 key=ADMIN_USER  value="admin"
/container/envs/add list=um5 key=ADMIN_PASS  value="yonetici-sifresi-8+"
/container/envs/add list=um5 key=TZ          value="Asia/Tehran"
/container/envs/add list=um5 key=CURRENCY    value="TL"
/container/envs/add list=um5 key=PORTAL_URL  value="https://alan-adiniz/um/"
```
Kontrol: `/container/envs/print where list=um5`
(Tam değişken tablosu bu sayfanın sonunda.)

---

## 5) Veri mount'u ve geçici dizin

Panelin **kendi** verisi (operatörler, kuponlar, satışlar, etkinlik kaydı) container
güncellemesinden ve silinmesinden sağ çıkması için bir mount üzerinde durmalı.
**Kullanıcı** verisi her zaman User Manager'da kalır, burada değil.

```
/container/mounts/add name=um5-data src=um5/data dst=/data
/container/config/set tmpdir=um5/tmp
```
- `src=um5/data` = router depolamasındaki klasör (otomatik oluşur); `dst=/data` =
  container içindeki yol.
- `tmpdir` = RouterOS'un imajı çekerken/açarken kullandığı geçici alan.

> Harici depolama (USB/NVMe) varsa `um5/...` yerine o diskin yolunu kullanın;
> MikroTik container'ların harici depolamada tutulmasını önerir. Yeterli sanal diski
> olan bir CHR'de dahili depolama da olur.

---

## 6) İmajı alma ve container'ı oluşturma

**İki seçenekten BİRİNİ seçin.**

### Seçenek A) ghcr.io'dan çekme (önerilen — dosya yükleme yok)
İmaj herkese açık, giriş gerekmez. Router internet + DNS ister.
```
/container/config/set registry-url=https://ghcr.io tmpdir=um5/tmp
/container/add remote-image=ghcr.io/cybertech1981/um5-admin:1.2.0 \
  interface=veth-um5 root-dir=um5/root mountlists=um5-data envlists=um5 \
  name=um5-admin start-on-boot=yes logging=yes
```

### Seçenek B) Dosya yükleme
`um5-admin-amd64.tar` dosyasını deponun **Releases** bölümünden alın, **Files**'a
yükleyin, sonra:
```
/container/add file=um5-admin-amd64.tar \
  interface=veth-um5 root-dir=um5/root mountlists=um5-data envlists=um5 \
  name=um5-admin start-on-boot=yes logging=yes
```

Notlar:
- 7.20'den eski RouterOS'ta `envlists`/`mountlists` yerine `envlist`/`mountlist` yazın.
- `root-dir=um5/root` imajın açıldığı yerdir. **Güncellemede** yeni bir ad verin
  (örn. `um5/root2`) ki eski dosyalarla karışmasın.

---

## 7) Başlatma ve doğrulama

```
/container/print
```
Durum `extracting`/`downloading`'den **`stopped`**'a gelene kadar bekleyin, sonra:
```
/container/start um5-admin
/log/print where topics~"container"
```
Logda şu üç satırı görmelisiniz:
```
created first admin "admin"
UM5 Admin 1.2.0 listening on http://:8080
router OK: RouterOS 7.23 ...
```
`router OK` görürseniz panel hazırdır. LAN'dan test adresi: `http://172.17.0.2:8080`

`ADMIN_USER` / `ADMIN_PASS` ile giriş yapın.

---

## 8) İlk girişten sonra (güvenlik — önemli)

1. Panelden yönetici şifresini değiştirin (adınız → "Şifremi değiştir").
2. `ADMIN_PASS`'ı kaldırın (artık gerekmez):
   ```
   /container/envs/remove [find list=um5 key=ADMIN_PASS]
   ```
3. `start` sırasında RouterOS tüm değişkenleri — **şifreler dahil** — loga düz metin
   yazar. Başka yerde kullanılmayan uzun ve rastgele bir `ROUTER_PASS` kullanın,
   logları/ekran görüntülerini gizli tutun.

---

## 9) HTTPS (kesinlikle önerilir)

Panel sertifikayı `/data`'dan okur ve dosya değişirse yeniden başlatmadan yükler.

1. Routerın sertifikasını dışa aktaran bir script (adı `/certificate/print`'ten alın):
```
/system/script/add name=um5-cert source={
  :do { /file/remove [find where name~"um5/data/tls"] } on-error={}
  /certificate/export-certificate "sertifika-adi" type=pem export-passphrase="anahtar-sifresi" file-name=um5/data/tls
}
/system/script/run um5-cert
/system/scheduler/add name=um5-cert interval=7d on-event=um5-cert
```
2. TLS değişkenleri ve yeniden başlatma:
```
/container/envs/add list=um5 key=TLS_CERT     value="/data/tls.crt"
/container/envs/add list=um5 key=TLS_KEY      value="/data/tls.key"
/container/envs/add list=um5 key=TLS_KEY_PASS value="anahtar-sifresi"
/container/stop um5-admin
/container/start um5-admin
```

---

## 10) Dışarıdan erişim

```
/ip/firewall/nat/add chain=dstnat protocol=tcp dst-port=8443 in-interface-list=WAN action=dst-nat to-addresses=172.17.0.2 to-ports=8080 comment="UM5 Admin"
```
`forward` zinciriniz varsayılan drop ise `connection-nat-state=dstnat` ile bir accept
ekleyin. Güvenlik için erişimi bilinen IP'lerle sınırlayın (`src-address-list=`).
Panel adresi: `https://alan-adiniz:8443`

---

## Yeni sürüme güncelleme

```
/container/stop um5-admin
/container/remove [find name=um5-admin]
```
Sonra 6. adımı yeni bir `root-dir` ile (örn. `um5/root2`) tekrar çalıştırın. Veri
`um5-data` mount'unda olduğu için operatörler/kuponlar/etkinlik korunur ve
`ADMIN_PASS` artık gerekmez. ghcr ile sadece `remote-image=...:1.2.1` sürümünü artırın.

---

## Sorun giderme

| Log mesajı | Nedeni ve çözümü |
|---|---|
| `Error in Shm Size - expected!` | WinBox form hatası; container'ı **terminalden** oluşturun. |
| `layer ... missing, need repull` | Eski docker-save tar formatı; bu deponun `um5-admin-amd64.tar` (OCI) dosyasını veya ghcr yöntemini kullanın. |
| `first start: set ADMIN_PASS` | `ADMIN_PASS` listede yok ya da `envlists=um5` bağlı değil (`/container/print detail`). |
| `no route to host` | veth gateway, bridge IP ve `ROUTER_URL` uyuşmuyor (adım 3). |
| `router rejected the API login` | API kullanıcı adı/şifresi, grup policy'si (`rest-api`) ve kullanıcının `address`'ini kontrol edin (adım 2). |
| çekerken: `dial tcp ... no route` / `lookup ghcr.io` | Container internete/DNS'e ulaşamıyor; adım 3 masquerade kuralını ve `/ip/dns`'i kontrol edin. |
| `/container/shell` çalışmıyor | Normaldir; imajda bilinçli olarak shell yok. `/log/print` kullanın. |

---

## Tam değişken tablosu

| Değişken | Varsayılan | Anlamı |
|---|---|---|
| `ROUTER_URL` | `http://172.17.0.1` | Container içinden router adresi |
| `ROUTER_USER` / `ROUTER_PASS` | — | Adım 2'deki API kullanıcısı (zorunlu) |
| `ROUTER_INSECURE_TLS` | `true` | `ROUTER_URL` geçerli sertifikalı https ise `false` yapın |
| `ADMIN_USER` / `ADMIN_PASS` | `admin` / — | Yalnızca ilk yöneticiyi oluşturmak için |
| `LISTEN` | `:8080` | Panelin dahili portu |
| `DATA_DIR` | `/data` | Panel veri klasörü (adım 5 mount'u) |
| `TLS_CERT` / `TLS_KEY` / `TLS_KEY_PASS` | — | HTTPS'i etkinleştirir |
| `PORTAL_URL` | router + `/um/` | Kupon kartlarına yazılan adres |
| `CURRENCY` | — | Fiyatların yanındaki para birimi (örn. `TL`) |
| `TZ` | `Asia/Tehran` | Saat dilimi |

---

## Roller

- **Yönetici (admin):** her şey; kullanıcı silme, operatör yönetimi, oturum kapatma
  ve etkinlik kaydı dahil.
- **Satıcı (seller):** kullanıcıları görme, kullanıcı ve kupon oluşturma, paket
  yenileme, şifre değiştirme, kullanıcı devre dışı bırakma.

---

## Tek seferde kurulum (bir kez kopyala, ghcr yöntemi)

Tırnaklı değerleri ve alt ağı kendinize göre ayarlayın, sonra tüm bloğu yapıştırın.
(`device-mode` + soğuk reboot önceden yapılmış olmalı.)

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
/container/envs/add list=um5 key=CURRENCY    value="TL"

/container/mounts/add name=um5-data src=um5/data dst=/data
/container/config/set registry-url=https://ghcr.io tmpdir=um5/tmp
/container/add remote-image=ghcr.io/cybertech1981/um5-admin:1.2.0 interface=veth-um5 root-dir=um5/root mountlists=um5-data envlists=um5 name=um5-admin start-on-boot=yes logging=yes
```
Sonra `stopped`'ı bekleyin, `/container/start um5-admin` ve `/log/print where topics~"container"` çalıştırın.

---

## Geliştirici

Tasarım ve geliştirme: Amir Makouei — [Amir Echo Tech](https://amirechotech.ca/) · [LinkedIn](https://www.linkedin.com/in/amir1360/) · [Telegram](https://t.me/AmirEchoTech)
