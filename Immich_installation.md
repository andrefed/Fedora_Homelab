# Immich installation and configuration guide

## 1. Overview and prerequisites

This guide walks through deploying Immich on a Fedora 44 server using Podman compose, with Nginx Proxy Manager as a reverse proxy.

Assumptions: 

* all containers are stored under `/home/admin/server/`
* all volumes use `:z` or `:Z`for SELinux compatibility
* data storage resides in an external drive mounted at `/mnt/usbdata`
* `podman-compose` is installed (run `sudo dnf install -y podman-compose` to install)
* all containers run in a shared `proxy` Podman network (verify with `podman network ls | grep proxy`, create with `podman network create proxy`)

---

## 2. Installation

 *⚠️ Directories must exist before starting containers*

Create the directory structure:

```
mkdir -p ~/server/immich/library/data ~/server/immich/postgres
```

Create the repository for Immich data:

```bash
mkdir /mnt/usbdata/immich-data
```

_Note: Immich offers the option to attach external libraries, but these cannot be used to edit photos (delete duplicates and adding comments or geotagging, for example). For this reason we are using the external storage as a container volume_


Create the compose and .env files:


```
cd ~/server/immich
```

```
nano compose.yaml # Use or customise the compose file from this repository
```

```
nano .env # Use or customise the env file from this repository
```

## 3. Starting the stack

Start the containers:

```bash
cd ~/server/immich
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
| Domain | immich.yourdomain.com     |
| Scheme | http                      |
| Host   | immich_server             |
| Port   | 2283                      |

* Enable:
  * Web sockets
  * Block common exploits

**SSL Tab**

* Choose the certificate you created earlier (see Nginx installation part)
* Enable:
  * Force SSL
  * HTTP/2

**Advanced config** (Gear icon)

```nginx
client_max_body_size 50000M;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
proxy_set_header Host $host;
```
---

## 5. Post-installation

**First access**

Create your admin user and login to Immich: `http://immich.yourdomain.com`

_Note: the first account created in Immich is by default an admin: it's important to create the access for the `admin` user first_

--- 

Reference:  https://docs.immich.app/install/docker-compose/
