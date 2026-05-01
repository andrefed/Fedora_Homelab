# Nginx reverse proxy installation and configuration guide

## 1. Overview and prerequisites

This guide walks through deploying Nginx Reverse Proxy Manager on a Fedora 44 server using Podman compose.

Assumptions: 

* all containers are stored under `/home/admin/server/`
* all volumes use `:Z` for SELinux compatibility
* `podman-compose` is installed (run `sudo dnf install -y podman-compose` to install)

---

## 2. Installation

Create the directory structure:

 *⚠️ Directories must exist before starting containers*

```bash
mkdir -p ~/server/nginx/npm/data ~/server/nginx/npm/letsencrypt
cd ~/server/nginx
```

Create the network `proxy` for the containers to communicate:

```bash
podman network create proxy
```

Create the compose.yaml file: use or customise the yaml file from this repository

```bash
nano compose.yaml
```

---

## 3. Starting the stack

Start the container:
```
cd ~/server/nginx
podman compose up -d
```

## 4. Post-installation

**First access**

Create your admin user and login to Nginx: `http://your-IP:81` (For example: `http://192.168.0.3:81`). 

**Create a certificate**

*Certificates → Add Certificates → Let's Encrypt via DNS*

 * add domains: `yourdomain.com` and `*.yourdomain.com`
 * select provider: (For example `DuckDNS`)
 * add token from from your provider (For example `duckdns.org`)
 * set propagation seconds to `120` and be patient! You might need to repeat this a few times or increase the propagation time


**Add a proxy host**

*Hosts → Proxy Hosts → Add Proxy Host*

| Field  | Value                                   |
| ------ | --------------------------------------- |
| Domain | proxy.yourdomain.com                    |
| Scheme | http                                    |
| Host   | your.local.IP (For example 192.168.0.3) |
| Port   | 81                                      |

* Enable:
  * Web sockets
  * Block common exploits

**SSL Tab**

* Choose the certificate you created earlier 
* Enable:
  * Force SSL
  * HTTP/2

---

Reference: https://nginxproxymanager.com/setup/
