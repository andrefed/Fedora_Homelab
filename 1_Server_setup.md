# Home server installation and configuration guide

This guide walks through deploying a home server with Fedora 44 to run various services.

## 1. Initial assumptions
* Fedora server 44 installed (see partition recommendation)
* system user: `admin`
* static IP set for the server (For example 192.168.0.3). Access the web inteface of your router and reserve an addess for the server
* static IP set for the Virtual Machine running Home assistant (for example 192.168.0.4). Note that the MAC address can be found in Cockpit only once the VM is installed.
* podman-compose installed (run `sudo dnf install podman-compose` if not installed)
* all containers are stored in `/home/admin/server` (run `mkdir ~/server` if the repository does not exist)
* data storage resides in an external drive
* the server web interface can be accesses via Cockpit at port 9090 (for example: `http://192.168.0.3:9090`)

---

## 2. Partitions

In Fedora, the root partition (/) often contains /var. System logs, package caches, old kernel versions and container data are all stored in /var, making it susceptible to filling up from system operations. If there is no separation between root (/) and /var the system is at risk of crashing when /var becomes full.

To avoid this 'disk full' scenario we install Fedora server with custom partions.

Homelab services typically fall into two categories: things that generate large data blobs (Immich photos, Ollama models, Nextcloud files, backups) versus things that need fast, bounded storage (databases, container layers, logs, system state). Since we're offloading bulk data to an external drive, our root partition just needs to stay lean for the OS and container metadata. 

Here's the rationale for each choice, based on a 512 GB (477 GiB) drive:

**`/boot/efi`  1 GiB (EFI partition - FAT32 ).** Fedora's default is 600 MB, but 1 GB gives you room if you ever dual-boot or accumulate shim/grub updates. No reason to go larger.

**`/boot`  2 GiB (ext4).** Must stay ext4 because GRUB can't read Btrfs reliably. 2 GB comfortably holds 4–5 kernels with their initramfs images, which is more than dnf's default retention. Don't put this on Btrfs.

**`/var`  150 GiB (Btrfs).** Podman stores all image layers and container writable layers in `/var/lib/containers`; the HAOS VM disk lives in `/var/lib/libvirt/images`; the Immich Postgres database grows with your photo library. Separating `/var` means none of this can ever starve your OS of space. Use Btrfs here so you can snapshot before major container updates.

**`/`  300 GiB (Btrfs).** After /var is split out, root only needs to hold: the OS itself (~4–5 GB), dnf package cache, `/home` service directories with configs and small data files. Use the standard Fedora Btrfs subvolume layout: `@` for root and `@home` as a subvolume, plus `@snapshots`.

**`swap`  ~24 GiB.** Common practice for server is "half of RAM, minimum 4 GB". Enable zram as well (`systemctl enable --now systemd-zram-setup@zram0`) for in-memory compression — it handles short-lived spikes before hitting disk swap.

**Key configuration tips:**

Set up Btrfs snapshots with snapper or Timeshift — snapshot before every `dnf upgrade` so you can roll back if a kernel or container runtime update breaks things. 

Configure systemd-journald to cap log size in `/etc/systemd/journald.conf` with `SystemMaxUse=2G`. For Ollama specifically, set `OLLAMA_MODELS=/mnt/data/ollama` in the service environment so models over ~3–4 GB land on the external disk rather than root. 

For Podman containers that write large data (Immich, Nextcloud), bind-mount their data directories from the external drives rather than letting them accumulate in `/var/lib/containers`.

---

## 3. Priviledged port setup

On Fedora privileged ports are those numbered below 1024 (e.g., 80, 443, 22). By default, only the root user can bind services to these ports. As Podman runs rootless, we need to allow it to use priviledged ports, preventing the error `rootlessport cannot expose privileged port 80` for example.

Open `/etc/sysctl.conf`:

```
sudo nano /etc/sysctl.conf
```

add:

```
net.ipv4.ip_unprivileged_port_start=80
```

*⚠️ Reboot after this change*


Update the firewall configuration:

```
sudo firewall-cmd --permanent --add-port={80,81,443}/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

---

## 4. Managing service restarts (Systemd)

Podman provides a systemd unit file (podman-restart.service) to handle container restarts after a system reboot. 

Enable automatic restart on boot (User):

```
systemctl --user enable --now podman-restart.service
```

Start the service now:

```
systemctl --user start podman-restart.service
```

Check logs for restart service:

```
journalctl --user -u podman-restart.service
```
---

## 5. Enable lingering

Enabling lingering in Podman allows rootless containers to start at boot and continue running after the user logs out

```
loginctl enable-linger $USER   
```

---

 ## 6. Server domain

Obtain a domain for your server, for example from `www.duckdns.org` and point `yourdomain.duckdns.org` to your local address of the server (for example 192.168.0.3). This will be used to manage SSL certificate via a Let's Encrypt DNS challenge. 

If you have already or buy a new domain, you can set the nameservers of your provider to point to services like `www.dynu.com` and set there your local IP address.

---

## 7. USB drive setup

Create a USB mount. List all drives and their UUID. Locate the USB UUID.

Create a persistent mount point:

```bash
sudo mkdir -p /mnt/usbdata
```

List all drives and their UUID:

```bash
lsblk -f
```

Set mounting at boot:

```bash
sudo cp /etc/fstab /etc/fstab.bak #Backup current fstab
sudo nano /etc/fstab
```

Add the line below to the fstab (replace with the actual UUID of the USB drive):

```
UUID=YOUR-UUID-HERE /mnt/famdata ext4 defaults,nofail,x-systemd.automount 0 0
```

Test the mount before applying the changes or rebooting:

```bash
sudo mount -a #Test the mount
```
Apply the changes:

```bash
sudo systemctl daemon-reload #Reload so that systemd can use the modified fstab
```

_Note: since this is a USB drive, the system might be trying to mount it before the USB bus has fully finished "settling" or probing the hardware. By adding x-systemd.automount, you tell Fedora: "Don't worry about it at the exact millisecond of boot; just mount it the very first time someone tries to access the folder."_

Ensure the `admin` user has appropriate rights for the USB drive:

```bash
sudo chown -R admin:admin /mnt/usbdata #Ensure the user 'admin' owns this folder and all its parents
sudo chmod -R 775 /mnt/usbdata
```

_Note: for best Linux performance, format the USB drive as ext4_


### Optional: Hard disk spin down
Spinning down a USB hard drive on a Fedora Server when idle can be achieved using hdparm to set a standby timer, or hd-idle for more precise control. The process involves identifying the drive, testing the spin-down command, and then automating it. 

**Recommended: hd-idle**

`hd-idle` is often more reliable for USB drives that do not respond well to `hdparm` 

 - Install hd-idle
```
sudo dnf install hd-idle
```
 - Verify that the configuration file `/etc/sysconfig/hd-idle` has `HD_IDLE_OPTS="-i 600"` uncommented. (600 seconds = 10 minutes)
 - Enable Service:
```
sudo systemctl enable --now hd-idle
```

**Proper way to detach/shutdown**

If you want to spin the drive down safely before unplugging it, use udisksctl: 
```
udisksctl unmount -b /dev/sdX1
udisksctl power-off -b /dev/sdX
```
This forces the drive to park its heads and power down. 

**Important notes:**

 - Waking up: The drive will automatically spin back up when accessed, which may introduce a small delay.
 - Enclosure Support: Some USB-to-SATA enclosures do not support passing spin-down commands (SAT - SCSI-ATA Command Translation). If hdparm doesn't work, the enclosure firmware may be the bottleneck.
 - Over-spinning: Do not set the idle time too low (e.g., less than 3-5 minutes) as frequent spin-up/spin-down cycles can shorten drive lifespan.

---

## 8. SELinux considerations

| Volume label    | Meaning         |
| --------------- | --------------- |
| `:Z`            | Private relabel |
| `:z`            | Shared relabel  |

Verify labels with:

```bash
ls -lZ /home/admin/server/nextcloud/nextcloud-data
ls -lZ /home/admin/server/nextcloud/db-data
```

Expected:

```
system_u:object_r:container_file_t:s0
```

Fix if needed:

```bash
sudo chcon -Rt container_file_t /home/admin/server/nextcloud/nextcloud-data
sudo chcon -Rt container_file_t /home/admin/server/nextcloud/db-data
sudo chcon -Rt container_file_t /home/admin/server/nextcloud/redis-data
```

---
