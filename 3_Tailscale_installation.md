# Tailscale installation and setup guide

This guide walks through deploying Tailscale to the home server, to access its services from outside of the home LAN.

## 1. Host install vs Podman container

For the specific setup of this homelab the best option is to install Tailscale directly on the host. Here is why:

**1. You need to reach both your containers *and* your VM.**
Your containers are on the `proxy` bridge network and your VM is on `br0`. Tailscale on the host sits above all of these and can route to everything natively. A containerised Tailscale can reach the host from the tailnet, but the host cannot initiate connections *out* to other tailnet devices — because containers can't modify the host's `resolv.conf` or routing table. For a server that needs bidirectional access (SSH in, access VM, reach containers), that's a deal-breaker.

**2. Subnet routing requires host-level access.**
If you want to advertise your LAN subnet (so remote devices can reach your containers and VM by their internal IPs), exit node or subnet router functionality requires Tailscale installed directly on the host, or a privileged system container. Running a privileged system container mostly negates the isolation benefit anyway.

**3. Your compose-based workflow doesn't benefit from the containerised approach.**
The main appeal of a containerised Tailscale is the "sidecar" pattern, where each service gets its own tailnet node with its own MagicDNS hostname — great if you only intend to host a few services, but deploying a sidecar for every service can be taxing on your server, with the load increasing proportionally to the number of services you are exposing. You already have Nginx Proxy Manager handling routing and SSL, so you don't need per-service tailnet nodes.

**4. Tailscale is a first-class Fedora package.**
Most people running containers on a home server run Tailscale directly on the host — it's a clean, well-maintained approach. On Fedora (non-immutable), it integrates naturally as a regular `dnf` package + systemd service.

---

## 2. Installation instructions

Add the Tailscale repo and install. Use Tailscale's one-line install script which always pulls the latest stable build:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Enable and start the daemon:

```bash
sudo systemctl enable --now tailscaled
```

Bring Tailscale up and authenticate:

```bash
sudo tailscale up
```

* This prints a URL. Open it in a browser to authenticate with your Tailscale account. Once authenticated, your server will appear in your tailnet.

Advertise your LAN subnet:
* This is the key step that lets you reach your containers and VM by their internal IPs from any tailnet device:

```bash
# Replace with your actual LAN subnet
sudo tailscale up --advertise-routes=192.168.0.0/24
```

* Then go to your [Tailscale admin console](https://login.tailscale.com/admin/machines), find your server, click `…` → **Edit route settings**, and approve the advertised route.

Enable IP forwarding (required for subnet routing):

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' >> /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

Disable key expiry for a server:

* In the Tailscale admin console, find your server → `…` → **Disable key expiry**. For devices that should remain continuously connected, such as servers, you can disable key expiry to avoid unnecessary disruptions.

Check your Tailscale IP:

```bash
tailscale ip -4
```

* This gives you the `100.x.x.x` address you'll use to reach the server from other tailnet devices.

### Key takeaways

* **Nginx Proxy Manager** continues to handle SSL certificates and reverse proxy as-is. Nothing changes there. If you want to access NPM's admin UI (port 81) remotely, you can do so via the Tailscale IP.
* **Containers on the `proxy` network** are reachable via the host's Tailscale IP + their exposed ports, or via your LAN IPs if you enabled subnet routing.
* **The VM on `br0`** is reachable by its LAN IP once subnet routing is approved.

---

## 3. Activating Tailscale on a desktop computer

KTailctl is a dedicated GUI client for Tailscale designed for the KDE Plasma desktop. It allows you to monitor and manage your Tailscale network directly from the system tray, including toggling it on/off, changing exit nodes, and viewing connected devices.
