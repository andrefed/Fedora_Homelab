# Nextcloud installation and configuration guide

## 1. Overview and prerequisites

This guide walks through deploying Nextcloud (standard edition) on a Fedora 44 server using Podman compose, with Nginx Proxy Manager as a reverse proxy.

Assumptions: 

* all containers are stored under `/home/admin/server/`
* the setup uses standard `nextcloud:stable-apache`, not AIO (not Podman-compatible)
* all volumes use `:z` for SELinux compatibility
* external data storage mount: `/mnt/usbdata` 
* `podman-compose` is installed (run `sudo dnf install -y podman-compose` to install)
* all containers run in a shared `proxy` Podman network (verify with `podman network ls | grep proxy`, create with `podman network create proxy`)

---

## 2. Installation

Create the directory structure:

 *⚠️ Directories must exist before starting containers*

```bash
mkdir -p /home/admin/server/nextcloud
cd /home/admin/server/nextcloud

mkdir nextcloud-data db-data redis-data
```

Create the compose file:

*⚠️ Replace all `CHANGE_ME_*` values before starting*

```bash
cd /home/admin/server/nextcloud
nano compose.yaml # Use or customise the compose file in this repository
```

---

## 3. Starting the stack

Start the containers:

```bash
cd /home/admin/server/nextcloud
podman compose up -d
```

Verify:

```bash
podman ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

---

## 4. Nginx Proxy Manager setup

Open `https://proxy.yourdomain.com`

**Add a proxy host**

*Hosts → Proxy Hosts → Add Proxy Host*

| Field  | Value                     |
| ------ | ------------------------- |
| Domain | nextcloud.yourdomain.com  |
| Scheme | http                      |
| Host   | nextcloud-app             |
| Port   | 80                        |

**SSL Tab**

* Choose the certificate you created earlier (see Nginx installation part)
* Enable:
  * Force SSL
  * HTTP/2

**Advanced config** (Gear icon)

```nginx
client_max_body_size 10G;
client_body_timeout 300s;

add_header Strict-Transport-Security "max-age=15552000; includeSubDomains" always;
add_header Referrer-Policy "no-referrer" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Download-Options "noopen" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Permitted-Cross-Domain-Policies "none" always;
add_header X-Robots-Tag "noindex, nofollow" always;
add_header X-XSS-Protection "1; mode=block" always;

fastcgi_hide_header X-Powered-By;

proxy_connect_timeout 300;
proxy_send_timeout 300;
proxy_read_timeout 300;
send_timeout 300;
```

---

## 5. Post-installation

Access Nextcloud and create your fist user:

```
https://nextcloud.yourdomain.com
```

Set Cron:

Settings → Basic settings → Background jobs → Cron


### Permissions for the USB drive

Fix permissions for the USB drive (crucial for rootless Podman): if running Podman in rootless mode, the user running the container (often UID 1000) does not match the user in the container (www-data, UID 33) or the root user on the host. 

Change ownership to the Nextcloud user - use podman unshare to chown the data to the correct user within the container's namespace:

```
podman unshare chown -R 33:33 /mnt/usbdata
```
Ensure appropriate permissions:

```
chmod -R 750 /mnt/usbdata
```
