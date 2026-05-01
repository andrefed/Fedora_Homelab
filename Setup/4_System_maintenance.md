# System maintenance

## 1. Updating a container stack manually
Unlike Docker, Podman containers are "immutable" once created. To update a container to the latest version of its image, you must pull the new image, stop/remove the old container, and start a new one.

Using Podman Compose makes this process much easier because the YAML file tracks all your volume mappings, environment variables, and network settings for you. You don't have to worry about re-typing long `podman run` commands.

To update your entire stack to the latest images, you generally follow a three-step workflow, running the commands listed below from the directory where your compose.yaml file is located.

1. Download the newest versions of the images defined in your file without stopping your current containers:

```bash
podman compose pull
```

2. This command compares the newly pulled images with the running containers. If an image has changed, Podman will stop the old container, remove it, and start a new one using the same configuration.

```bash
podman compose up -d
```

3. After updating, you will likely have "dangling" images (the old versions) taking up disk space. You can clear them out with:

```bash
podman image prune
```

### Important considerations

**Data persistence**

⚠️  Updating a container effectively **destroys the old container** and creates a new one. 

**Note:** Always ensure your important data is stored in **Volumes** or **Bind Mounts** defined in your compose file. If data is stored inside the container's internal writable layer, it **will be lost** during the update.

**Version tags**

If your compose file uses specific version tags (e.g., `image: nextcloud:28`), `podman-compose pull` will not update you to version 29. You must manually edit the version number in your `.yml` file first, then run the `up -d` command. If you use the `:latest` tag, the commands above will work automatically.

---

## 2. System cleanup

Your root directory (/) in Fedora Server 43 is likely growing due to accumulating system logs, package caches, old kernel versions, or container data (Docker/Podman/Flatpak) stored in /var. In Fedora, the root partition often contains /var, making it susceptible to filling up from system operations. 

You can mitigate this risk by installing Fedora with a customised partion (see the server setup section).


Here are the most common culprits and how to fix them:

1. Large Systemd Journal Logs
* Systemd logs can grow uncontrollably, especially if a service is crashing or logging excessive debug information.
* Check usage: `journalctl --disk-usage`
* Fix: clear logs older than 3 days: `sudo journalctl --vacuum-time=3d`

2. DNF Package Cache 
* Every time you update your server, DNF downloads packages and stores them.
* Fix: run `sudo dnf clean all` to remove cached package data. 

3. Old Kernel Versions 
* Fedora keeps multiple kernel versions for safety, but these take up significant space in `/boot` and `/root`. 
* Fix: remove old kernels, keeping only the latest two: `sudo dnf remove $(dnf repoquery --installonly --latest-limit=-2 -q)`

4. Container & Image Data (Docker/Podman) 
* If you run Podman, images, volumes, and container logs can eat up all available space in `/var/lib`.
* Fix: run `podman system prune -a` to clean up unused data. 

5. Btrfs Snapshots 
* If you are using Btrfs, automatic snapshots (e.g., via Timeshift) can accumulate, especially if they are backing up your home directory. 
* Fix: check and remove old snapshots in your backup utility. 


**How to Identify the Culprit* - run this command to see which directory is consuming the most space: 

```bash
sudo du -sh /* | sort -h
```

Or use the ncdu tool for an interactive view:

```bash
sudo dnf install ncdu
sudo ncdu -x /
```
Quick cleanup Commands

```bash
sudo dnf clean all # Clean DNF cache
sudo journalctl --vacuum-size=100M # Vacuum Journals (Limit to 100MB)
sudo dnf autoremove # Autoremove unused packages
sudo dnf remove $(dnf repoquery --installonly --latest-limit=-2 -q) # Remove old kernels
```

