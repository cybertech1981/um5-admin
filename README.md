# UM5 Admin

![UM5 Admin](docs/banner.png)

**🌐 Languages:** English · [فارسی](README.fa.md) · [Türkçe](README.tr.md)  
**📖 Install guide:** [English](INSTALL.md) · [فارسی](INSTALL-FA.md) · [Türkçe](INSTALL-TR.md)

Multilingual (**Persian / English / Turkish**) admin panel for **MikroTik User
Manager v5**. It runs as a single tiny container (a static binary, ~8 MB, no OS
inside) and manages the router over its **REST API** — create users and vouchers,
renew plans, watch sessions, print cards, and read usage and sales reports.

Made by [Amir Echo Tech](https://amirechotech.ca/).

## What's in this repository

```
INSTALL.md / -FA / -TR   run it INSIDE RouterOS as a container (EN / FA / TR)
docker-compose.yml       run it on a normal Docker host
TERMS.md                 terms of use
docs/                    screenshots
```

The container image itself is **not** committed here — see **Get the image** below.

## Screenshots

| Persian (RTL) | English |
|---|---|
| ![Dashboard — Persian](docs/admin-fa.png) | ![Dashboard — English](docs/admin-en.png) |

| Turkish | English — dark |
|---|---|
| ![Dashboard — Turkish](docs/admin-tr.png) | ![Dashboard — dark](docs/admin-en-dark.png) |

<img src="docs/admin-tr-mobile.png" width="260" alt="Mobile — Turkish">

## Get the image

The container image is **not** in this repository (it is a binary). Download it from:

- **Releases** → `um5-admin-amd64.tar` (attached to the latest release), then
  `docker load -i um5-admin-amd64.tar`, or upload it to the router's **Files** and
  follow [INSTALL-FA.md](INSTALL-FA.md).
- or **GitHub Container Registry**:
  `docker pull ghcr.io/cybertech1981/um5-admin:1.2.0`

## Run inside RouterOS (recommended)

The full step-by-step — with every line explained, HTTPS, external access and
troubleshooting — is in **[INSTALL.md](INSTALL.md)**. The seven
steps:

**1. Enable containers (once).** With the `container` package installed:
```
/system/device-mode/update container=yes
```
Confirm with a cold reboot (Power Off/On on CHR).

**2. A restricted API user** (the panel signs in as this, never as `admin`):
```
/user/group/add name=um5-api policy=read,write,api,rest-api
/user/add name=um5-panel group=um5-api address=172.17.0.0/24 password="API-PASS"
```

**3. Container network:**
```
/interface/veth/add name=veth-um5 address=172.17.0.2/24 gateway=172.17.0.1
/interface/bridge/add name=containers
/ip/address/add address=172.17.0.1/24 interface=containers
/interface/bridge/port/add bridge=containers interface=veth-um5
/ip/firewall/nat/add chain=srcnat src-address=172.17.0.0/24 action=masquerade
```

**4. Environment variables:**
```
/container/envs/add list=um5 key=ROUTER_URL  value="http://172.17.0.1"
/container/envs/add list=um5 key=ROUTER_USER value="um5-panel"
/container/envs/add list=um5 key=ROUTER_PASS value="API-PASS"
/container/envs/add list=um5 key=ADMIN_USER  value="admin"
/container/envs/add list=um5 key=ADMIN_PASS  value="ADMIN-PASS-8+"
/container/envs/add list=um5 key=TZ          value="Asia/Tehran"
```

**5. Data mount** (operators, vouchers and the activity log live here):
```
/container/mounts/add name=um5-data src=um5/data dst=/data
```

**6. Create the container from ghcr.io:**
```
/container/config/set registry-url=https://ghcr.io tmpdir=um5/tmp
/container/add remote-image=ghcr.io/cybertech1981/um5-admin:1.2.0 interface=veth-um5 root-dir=um5/root mountlists=um5-data envlists=um5 name=um5-admin start-on-boot=yes logging=yes
```
(Or use `file=um5-admin-amd64.tar` from Releases instead of `remote-image`.)

**7. Start and check:**
```
/container/print            ;# wait until status = stopped
/container/start um5-admin
/log/print where topics~"container"   ;# expect "router OK"
```
Sign in with `ADMIN_USER` / `ADMIN_PASS`, then change the admin password in the
panel and remove `ADMIN_PASS`:
`/container/envs/remove [find list=um5 key=ADMIN_PASS]`

## Run on a Docker host

```bash
docker pull ghcr.io/cybertech1981/um5-admin:1.2.0
# edit docker-compose.yml: ROUTER_URL / ROUTER_USER / ROUTER_PASS / ADMIN_PASS
docker compose up -d
# open http://<host>:8080  — sign in with ADMIN_USER / ADMIN_PASS
```

After the first sign-in, change the admin password in the panel and remove
`ADMIN_USER` / `ADMIN_PASS` from the compose file.

## The router user it needs

Create a **restricted** RouterOS user for the panel (never use `admin`):

```
/user/group/add name=um5-api policy=read,write,api,rest-api
/user/add name=um5-panel group=um5-api address=<container-subnet> password="long-random"
```

Put that user in `ROUTER_USER` / `ROUTER_PASS`.

## Roles

- **Admin**: everything, including deleting users, managing operators, closing
  sessions, and the activity log.
- **Seller**: view users, create users and vouchers, renew plans, change
  passwords, disable users.

## License

Distributed binary image. See **[TERMS.md](TERMS.md)**. Free to use; the source
is not published and the author attribution may not be removed. For a commercial
redistribution or rebranding license, contact
[Amir Echo Tech](https://amirechotech.ca/).

---

**[Amir Echo Tech](https://amirechotech.ca/)** ·
[LinkedIn](https://www.linkedin.com/in/amir1360/) ·
[Telegram](https://t.me/AmirEchoTech)
