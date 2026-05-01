# SMB share setup guide

This guide walks through deploying the Samba service to share data externally.

## 1. Initial assumptions

* Folder to be shared via the Samba protocol: `/mnt/usbdata`
* Samba user: `admin` 

## 2. Samba installation

Install Samba:

```
sudo dnf install samba
```

Start the daemon and enable it to run at boot:

```
sudo systemctl enable --now smb 
```

Check if the service is running correctly:

```
systemctl status smb 
```

Configure the firewall:
```
sudo firewall-cmd --permanent --add-service=samba
sudo firewall-cmd --reload
```

Note: Linux users are not automatically Samba users. Even if you use the same username, you must add the user to the Samba database.
Create a password for the generic Samba user 'admin' 

```
sudo smbpasswd -a admin
```

## 3. Samba share setup

Backup the original configuration file and configure a new one: 

```
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.orig
sudo nano /etc/samba/smb.conf
```

Add the following at the end of the global section:

`security = user`

Comment the whole `[homes]` section

Add the following to the bottom of file:
```
[Shared]
   path = /mnt/usbdata
   valid users = admin
   guest ok = no
   writable = yes
   browsable = yes
```

## 4. SELinux permission for Samba

Fedora uses SELinux by default, which blocks Samba from accessing folders it doesn't "own" or recognize. To tell SELinux that Samba can share folders and that a specific folder is allowed to be shared via Samba, run:

```
sudo setsebool -P samba_export_all_rw 1
```

```
sudo semanage fcontext -a -t samba_share_t "/mnt/usbdata(/.*)?"
sudo restorecon -Rv /mnt/usbdata
```

## 5. User permission for Samba

Note: the user logging into Samba must have permission to look inside the directory and all parent directories in the path. Ensure the user 'admin' owns the directory and has read/execute permissions.

```
sudo chown -R admin:admin /mnt/usbdata
sudo chmod -R 755 /mnt/usbdata
```

## 6. Samba service start

Check there are no sytax errors in smb.conf:

```
testparm
```

Restart the Samba service:

```
sudo systemctl restart smb
```

