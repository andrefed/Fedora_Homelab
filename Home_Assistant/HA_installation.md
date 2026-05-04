# Installing Home Assistant OS as a KVM VM on Fedora 44

The approach is: **CLI for setup and image prep** (for full control), and **Cockpit's Machines plugin** for day-to-day VM management afterwards.

Note: make sure the root partition has 32GB (at the OS image needs to be saved there)

---

## Step 1 — Install KVM/libvirt and the Cockpit Machines plugin

The required software prerequisites are `libvirt`, `cockpit`, and `cockpit-machines`. Since you already have Cockpit, you likely have some of these. Run this to make sure everything is in place:

```
sudo dnf install -y \
  libvirt \
  libvirt-daemon-kvm \
  qemu-kvm \
  virt-install \
  cockpit-machines \
  edk2-ovmf \
  xz
```

> `edk2-ovmf` provides the UEFI firmware — **required** for Home Assistant OS. `xz` is needed to decompress the image.

Now enable and start the services:

```
sudo systemctl enable --now libvirtd
sudo systemctl enable --now cockpit.socket
```

Verify KVM is working:

```bash
sudo virt-host-validate
```

You should see `PASS` next to the KVM and QEMU lines. If you see a WARN about IOMMU, it's not critical for this use case.

---

## Step 2 — Check virtualisation extensions

```bash
grep -Ec '(vmx|svm)' /proc/cpuinfo
```

Any number greater than 0 means your CPU supports virtualisation. `vmx` = Intel, `svm` = AMD.

---

## Step 3 — Download and prepare the Home Assistant OS image

The current major release is Home Assistant OS 17.x. Download the latest stable `.qcow2` image (check the [releases page](https://github.com/home-assistant/operating-system/releases) for the exact latest version number — substitute it below):

```bash
# Create a directory for your VM images
sudo mkdir -p /var/lib/libvirt/images

# Download the compressed qcow2 image (replace 17.1 with the latest version)
sudo curl -L \
  https://github.com/home-assistant/operating-system/releases/download/17.1/haos_ova-17.1.qcow2.xz \
  -o /var/lib/libvirt/images/haos_ova-17.1.qcow2.xz

# Decompress it
sudo xz --decompress /var/lib/libvirt/images/haos_ova-17.1.qcow2.xz

# Confirm it looks right
sudo qemu-img info /var/lib/libvirt/images/haos_ova-17.1.qcow2
```

You should see it reported as a `QCOW2` image, roughly 32–34 GB virtual size.

---

## Step 4 — Create the VM with `virt-install`

The official Home Assistant installation command for KVM is:

```bash
sudo virt-install \
  --name haos \
  --description "Home Assistant OS" \
  --os-variant=generic \
  --ram=4096 \
  --vcpus=2 \
  --disk /var/lib/libvirt/images/haos_ova-17.1.qcow2,bus=scsi \
  --controller type=scsi,model=virtio-scsi \
  --import \
  --graphics none \
  --boot uefi \
  --network network=default \
  --noautoconsole
```

Key notes:
- `--boot uefi` is **mandatory** — a common failure is booting the VM in BIOS mode instead of UEFI, which prevents it from starting. Make sure you have `edk2-ovmf` installed (Step 1).
- `--ram=4096` gives it 4 GB. Home Assistant officially recommends 2 GB minimum, but 4 GB gives comfortable headroom for add-ons.
- `--noautoconsole` lets the command return immediately; you'll monitor it separately.

---

## Step 5 — Monitor boot and find the IP address

Watch the VM boot via the serial console:

```bash
sudo virsh console haos
```

You'll see the Home Assistant boot log. Once it settles (1–2 minutes), press `Enter` — you'll get the HA OS prompt (very limited shell, this is normal). Type:

```
login
```

Username is `root`, no password. Then:

```bash
network info
```

This shows the VM's IP address. Note it down. Press `Ctrl+]` to exit the console.

Alternatively, ask libvirt for the IP:

```bash
sudo virsh domifaddr haos
```

---

## Step 6 — Set the VM to autostart

```bash
sudo virsh autostart haos
```

This ensures Home Assistant starts automatically every time your Fedora server boots.

---

## Step 7 — Create a bridge network in Cockpit

A libvirt/KVM NAT network IP is only reachable from the host machine or from inside the VM network. It is NOT reachable from your LAN or the internet.

Instead of NAT (192.168.122.x), give the VM an IP on your LAN (192.168.1.x). This requires changing VM network config to a bridge network (libvirt bridge).

Networking → Add Interface → Choose Bridge

Fill in:
 - Name: `br0`
 - Interfaces: select your main NIC (e.g. `enp3s0`, `eth0`)
 - IPv4: choose Automatic (DHCP) (simplest)
 - Set MAC address and static IP (For example 52:54:00:94:48:65 and 192.168.0.4)

Note: when you create the bridge your host may briefly lose network. This is normal (it’s moving the IP from NIC → bridge)

Verify bridge is up. You should now see:

`br0` with an IP (e.g. `192.168.1.x`)

Attach VM to bridge

Virtual Machines → Edit → Network Interface

Change source to `br0`

Restart the VM

Get the VM’s new IP (Something like 192.168.1.xxx)

Test from your browser
`http://192.168.1.xxx:8123`

---

To confirm host accessibility from your remote machine:

```
curl http://192.168.1.xxx:8123
```

If that fails open the firewall on host:
```
sudo firewall-cmd --add-port=8123/tcp --permanent
sudo firewall-cmd --reload
```
---

## Step 8 — Access Home Assistant

Open a browser and go to:

```
http://<VM-IP-ADDRESS>:8123
```

or try the mDNS hostname:

```
http://homeassistant.local:8123
```

You'll land on the **Home Assistant onboarding screen**. Create your admin account and follow the setup wizard.

First access to Home Assistant:
 - `http://192.168.0.4:8123`

---

## 9. Nginx Proxy Manager setup

Open `https://proxy.yourdomain.com`

**Add a proxy host**

*Hosts → Proxy Hosts → Add Proxy Host*

| Field  | Value                     |
| ------ | ------------------------- |
| Domain | hass.yourdomain.com       |
| Scheme | http                      |
| Host   | VM.IP.address             |
| Port   | 8123                      |

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
proxy_buffering off;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header Host $host;
```
---

## Step 10 — Update HA configuration

Open Home Assistant and go to Settings > Apps and install File Editor.

Click the Add-on Store button in the bottom right.

Once installed toggle on Show in sidebar, then start.

Click File Editor in your left-hand sidebar, click the Folder icon at the top left, and select `configuration.yaml`.

Add the following code:
```
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 192.168.0.3  # The IP of your Fedora Server running NPM
```
**Important**: once you add the code, do not just restart HA. 
1. Go to Developer Tools > YAML.
2. Click Check Configuration.
3. Only if it says "Configuration will not prevent Home Assistant from starting", then click Restart


## HACS setup

### Prerequisites

 - A GitHub account.
 - Home Assistant OS installed.
   
### Step-by-Step Installation Guide

Enable Advanced Mode: Go to your profile (bottom left), scroll down, and toggle Advanced Mode on.

Install SSH Add-on:
 - Navigate to Settings > Apps > Install App.
 - Search for "Terminal & SSH" and install it.
 - Start the add-on and ensure "Show in sidebar" is enabled.

Run HACS Download Script:
 - Open the Terminal from the sidebar.
 - Paste the following command and press Enter:
```
wget -q -O - https://install.hacs.xyz | bash -
```
 - Wait for the command to complete, which will indicate it has successfully downloaded the files.

Restart Home Assistant:
 - Navigate to Settings > System and click the power icon in the top right to restart.

Configure HACS Integration:
 - Once restarted, go to Settings > Devices & Services.
 - Click Add Integration in the bottom right.
 - Search for "HACS" and select it.
 - Follow the on-screen prompts, agree to the requirements, and authorize with your GitHub account (copy the code provided and use the link to enter it).
 - Finalize: Assign HACS to an area and click finish. The HACS icon will now appear in your sidebar. If it does not, clear your browser cache or force a refresh (Ctrl+F5). 
