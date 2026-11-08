# Installing UM5 Admin on MikroTik (container)

🌐 **Install guide:** English · [فارسی](INSTALL-FA.md) · [Türkçe](INSTALL-TR.md)

Multilingual (Persian / English / Turkish) admin panel for MikroTik User Manager v5.
It is a single Go binary (~8 MB, no OS inside the image) and talks to the router
only over its **REST API**.

```
browser ──► router (dst-nat :8443) ──► container 172.17.0.2:8080 ──REST──► 172.17.0.1 (same router)
```

**Requirements:** RouterOS v7 with the `container` package, arch arm/arm64/x86.
Tested on CHR x86_64 and RouterOS 7.23.

> ⚠️ Run everything in **New Terminal**, not the GUI form. The "New Container"
> dialog in some WinBox builds rejects any value with `Error in Shm Size - expected!`.

> This guide assumes the subnet `172.17.0.0/24` (router `172.17.0.1`, container
> `172.17.0.2`). To use a different subnet, change those numbers consistently.

---

## 1) Enable containers (once)

```
/system/package/print
```
If `container` is missing: download the **Extra packages** archive matching your
RouterOS version from mikrotik.com, upload `container-x.y.npk` to **Files**, reboot.

Then turn container mode on:
```
/system/device-mode/update container=yes
```
This waits for **physical confirmation**:
- Hardware: press the reset button or power-cycle.
- CHR/VMware: **Power Off** and start again (a soft reboot is not enough).

Verify: `/system/device-mode/print` should show `container: yes`.

---

## 2) A dedicated API user

The panel connects with a **restricted** user, never `admin`. It may sign in only
from the container subnet:
```
/user/group/add name=um5-api policy=read,write,api,rest-api comment="UM5 Admin panel"
/user/add name=um5-panel group=um5-api address=172.17.0.0/24 password="a-strong-random-password"
```
The `www` service must be enabled (the `/um/` portal and REST both use it):
```
/ip/service/print where name=www
```

---

## 3) Container network

```
/interface/veth/add name=veth-um5 address=172.17.0.2/24 gateway=172.17.0.1
/interface/bridge/add name=containers
/ip/address/add address=172.17.0.1/24 interface=containers
/interface/bridge/port/add bridge=containers interface=veth-um5
```

Three things must match, or the log shows `no route to host`:
- the veth `gateway` = the router's IP = `172.17.0.1`
- that IP sits on the **bridge** (not on the veth itself)
- `ROUTER_URL` in step 4 = `http://172.17.0.1`

If your `input` firewall drops by default, allow the container to reach REST:
```
/ip/firewall/filter/add chain=input src-address=172.17.0.0/24 protocol=tcp dst-port=80 action=accept comment="UM5 panel to REST" place-before=0
```

> If you pull the image from ghcr.io (step 6, option A), the container needs
> internet + DNS **to pull the image**. If the veth has no route out, add a NAT rule:
> ```
> /ip/firewall/nat/add chain=srcnat src-address=172.17.0.0/24 action=masquerade comment="containers out"
> ```
> You can remove it after install if you do not expose the panel externally.

---

## 4) Environment variables

All variables go into one list named `um5`. Change the quoted values.
`ADMIN_USER` is the **username** and `ADMIN_PASS` the **password** of the first
admin; both must exist before the first start.

```
/container/envs/add list=um5 key=ROUTER_URL  value="http://172.17.0.1"
/container/envs/add list=um5 key=ROUTER_USER value="um5-panel"
/container/envs/add list=um5 key=ROUTER_PASS value="same-as-step-2"
/container/envs/add list=um5 key=ADMIN_USER  value="admin"
/container/envs/add list=um5 key=ADMIN_PASS  value="admin-password-8+chars"
/container/envs/add list=um5 key=TZ          value="Asia/Tehran"
/container/envs/add list=um5 key=CURRENCY    value="Toman"
/container/envs/add list=um5 key=PORTAL_URL  value="https://your-domain/um/"
```
Check: `/container/envs/print where list=um5`
(Full variable table at the bottom of this page.)

---

## 5) Data mount and temp directory

The panel's **own** data (operators, vouchers, sales, activity log) must live on a
mount so it survives container updates and removal. **User** data always stays on
User Manager itself, never here.

```
/container/mounts/add name=um5-data src=um5/data dst=/data
/container/config/set tmpdir=um5/tmp
```
- `src=um5/data` = a folder on router storage (created automatically); `dst=/data`
  = the path inside the container.
- `tmpdir` = scratch space RouterOS uses while pulling/unpacking the image.

> With external storage (USB/NVMe), use that disk's path instead of `um5/...`;
> MikroTik recommends keeping containers on external storage. On a CHR with a big
> enough virtual disk, internal storage is fine.

---

## 6) Get the image and create the container

**Pick ONE of the two options.**

### Option A) Pull from ghcr.io (recommended — no file upload)
The image is public, no login needed. The router needs internet + DNS.
```
/container/config/set registry-url=https://ghcr.io tmpdir=um5/tmp
/container/add remote-image=ghcr.io/cybertech1981/um5-admin:1.2.0 \
  interface=veth-um5 root-dir=um5/root mountlists=um5-data envlists=um5 \
  name=um5-admin start-on-boot=yes logging=yes
```

### Option B) Upload the file
Get `um5-admin-amd64.tar` from the repo's **Releases**, upload it to **Files**, then:
```
/container/add file=um5-admin-amd64.tar \
  interface=veth-um5 root-dir=um5/root mountlists=um5-data envlists=um5 \
  name=um5-admin start-on-boot=yes logging=yes
```

Notes:
- On RouterOS older than 7.20 use `envlist`/`mountlist` instead of `envlists`/`mountlists`.
- `root-dir=um5/root` is where the image unpacks. On an **update**, give a fresh
  name (e.g. `um5/root2`) so it doesn't mix with the old files.

---

## 7) Start and verify

```
/container/print
```
Wait until the status goes from `extracting`/`downloading` to **`stopped`**, then:
```
/container/start um5-admin
/log/print where topics~"container"
```
The log should show these three lines:
```
created first admin "admin"
UM5 Admin 1.2.0 listening on http://:8080
router OK: RouterOS 7.23 ...
```
If you see `router OK`, the panel is ready. Test URL from the LAN:
`http://172.17.0.2:8080`

Sign in with `ADMIN_USER` / `ADMIN_PASS`.

---

## 8) After the first sign-in (security — important)

1. Change the admin password in the panel (your name → "Change my password").
2. Remove `ADMIN_PASS` (no longer needed):
   ```
   /container/envs/remove [find list=um5 key=ADMIN_PASS]
   ```
3. On `start`, RouterOS writes every variable — **including passwords** — as plain
   text in the log. Use a long random `ROUTER_PASS` that is not reused anywhere,
   and keep logs/screenshots private.

---

## 9) HTTPS (strongly recommended)

The panel reads the certificate from `/data` and reloads it without a restart if the
file changes.

1. A script to export the router's certificate (take the name from `/certificate/print`):
```
/system/script/add name=um5-cert source={
  :do { /file/remove [find where name~"um5/data/tls"] } on-error={}
  /certificate/export-certificate "cert-name" type=pem export-passphrase="key-pass" file-name=um5/data/tls
}
/system/script/run um5-cert
/system/scheduler/add name=um5-cert interval=7d on-event=um5-cert
```
2. TLS variables and restart:
```
/container/envs/add list=um5 key=TLS_CERT     value="/data/tls.crt"
/container/envs/add list=um5 key=TLS_KEY      value="/data/tls.key"
/container/envs/add list=um5 key=TLS_KEY_PASS value="key-pass"
/container/stop um5-admin
/container/start um5-admin
```

---

## 10) External access (VPS / CHR)

On a VPS or CHR the router itself holds the public IP — and it often also runs your
VPN — so you reach the panel over that public IP (or through the VPN). The container
sits on a private bridge, so exposing it needs **two** rules: a destination-NAT that
forwards a public port to the container, and a **forward**-chain accept for that
traffic.

Forward public port **8080** to the container's 8080:
```
/ip/firewall/nat/add chain=dstnat protocol=tcp dst-port=8080 in-interface-list=WAN action=dst-nat to-addresses=172.17.0.2 to-ports=8080 comment="UM5 Admin panel"
/ip/firewall/filter/add chain=forward protocol=tcp dst-address=172.17.0.2 dst-port=8080 action=accept comment="UM5 Admin panel" place-before=0
```
Panel URL: `http://<router-public-ip>:8080`

- No `WAN` interface list? Use `in-interface=ether1` (your public interface) instead
  of `in-interface-list=WAN`.
- With **HTTPS** (step 9), forward a different port to 8080, e.g. 8443:
  ```
  /ip/firewall/nat/add chain=dstnat protocol=tcp dst-port=8443 in-interface-list=WAN action=dst-nat to-addresses=172.17.0.2 to-ports=8080 comment="UM5 Admin (https)"
  ```
  then open `https://<your-domain>:8443`.
- **Prefer VPN-only:** instead of exposing it to the whole internet, reach it through
  your VPN — drop the dst-nat and just allow the VPN subnet to the container:
  ```
  /ip/firewall/filter/add chain=forward protocol=tcp src-address=<vpn-subnet> dst-address=172.17.0.2 dst-port=8080 action=accept place-before=0
  ```
  then browse to `http://172.17.0.2:8080` over the VPN.

**Security:** anyone who can reach this port can try to log in. Restrict it with
`src-address=` / `src-address-list=`, prefer HTTPS, and ideally keep it VPN-only.

---

## Updating to a new version

```
/container/stop um5-admin
/container/remove [find name=um5-admin]
```
Then re-run step 6 with a fresh `root-dir` (e.g. `um5/root2`). Because the data is on
the `um5-data` mount, operators/vouchers/activity are kept and `ADMIN_PASS` is no
longer needed. With ghcr, just bump the version in `remote-image=...:1.2.1`.

---

## Troubleshooting

| Log message | Cause and fix |
|---|---|
| `Error in Shm Size - expected!` | WinBox form bug; build the container from the **terminal**. |
| `layer ... missing, need repull` | Old docker-save tar format; use this repo's `um5-admin-amd64.tar` (OCI) or the ghcr method. |
| `first start: set ADMIN_PASS` | `ADMIN_PASS` not in the list, or `envlists=um5` not attached (`/container/print detail`). |
| `no route to host` | veth gateway, bridge IP and `ROUTER_URL` don't match (step 3). |
| `router rejected the API login` | Check the API user name/password, the group policy (`rest-api`), and the user's `address` (step 2). |
| on pull: `dial tcp ... no route` / `lookup ghcr.io` | Container can't reach internet/DNS; check the step-3 masquerade rule and `/ip/dns`. |
| `/container/shell` doesn't work | Expected; the image has no shell by design. Use `/log/print`. |

---

## Full variable table

| Variable | Default | Meaning |
|---|---|---|
| `ROUTER_URL` | `http://172.17.0.1` | Router address from inside the container |
| `ROUTER_USER` / `ROUTER_PASS` | — | API user from step 2 (required) |
| `ROUTER_INSECURE_TLS` | `true` | Set `false` if `ROUTER_URL` is https with a valid cert |
| `ADMIN_USER` / `ADMIN_PASS` | `admin` / — | Only to create the first admin |
| `LISTEN` | `:8080` | Panel's internal port |
| `DATA_DIR` | `/data` | Panel data folder (the step-5 mount) |
| `TLS_CERT` / `TLS_KEY` / `TLS_KEY_PASS` | — | Enable HTTPS |
| `PORTAL_URL` | router + `/um/` | URL printed on voucher cards |
| `CURRENCY` | — | Currency shown next to prices (e.g. `Toman`) |
| `TZ` | `Asia/Tehran` | Time zone |

---

## Roles

- **Admin:** everything, including deleting users, managing operators, closing
  sessions, and the activity log.
- **Seller:** view users, create users and vouchers, renew plans, change passwords,
  disable users.

---

## All-in-one install (copy once, ghcr method)

Set the quoted values and subnet to match your setup, then paste the whole block.
(`device-mode` + cold reboot must already be done.)

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
Then wait for `stopped`, run `/container/start um5-admin` and `/log/print where topics~"container"`.

---

## Author

Designed and developed by Amir Makouei — [Amir Echo Tech](https://amirechotech.ca/) · [LinkedIn](https://www.linkedin.com/in/amir1360/) · [Telegram](https://t.me/AmirEchoTech)
