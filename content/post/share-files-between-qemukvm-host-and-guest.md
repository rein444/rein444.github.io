---
title: 'Share Files Between QEMU/KVM Host and Guest (SFTP + 9p-virtio + virtiofs + Samba + NFS)'
date: 2022-05-06 11:29:14
tags: [tech]
published: true
hideInList: false
feature: 
isTop: false
---
There is a variety of options to choose from for file sharing between the host and a QEMU/KVM virtual machine. This post is a collection of them.

---

## SFTP
If your goal is just to occasionally share a file with no need for it to be synchronized, SFTP would do the trick. Find your IP address through `ifconfig` or `ip addr` and make sure your ssh port is open. Make a connection and use `get` to download files or `put` to upload files like what you usually do.


## 9p-virtio
This is the easiest and most error-free option you can have but surprisingly, not much information can be found on the internet since 9p is almost obsolete. 9p, also known as Plan 9, is a network protocol for communicating between elements on a distributed system that can be used with its native virtio transport driver to access shared directories on the host machine.

It's pretty simple to setup. First create a folder you want to share, and then open virt-maneger (or any GUI you prefer since I don't like working with long commands in the terminal); locate the settings of the machine you want to share files with. Click Add Hardware -> Filesystem; choose virtio-9p for Driver, the folder you just created for Source path, and [a tag for your folder] (ex. "/share") for Target path. Now run the machine.

In the terminal, run the following command: 
```bash
mount -t 9p -o trans=virtio /tag /mount/path
```

To auto-mount the folder, write the following into your `/etc/fstab`:
```bash
/share	/mount/path	9p	trans=virtio,rw,_netdev	0	0
```
`_netdev` will delay the mount until 9p modules can be loaded since it is a network based filesystem.

One thing I like about this method is that it can be used in a CD live environment because 9p-vitio is part of the QEMU/KVM. One thing I don't mind but others might dislike is that it can't be used with Windows for the same reason.

## virtiofs
I stumbled upon [this web page](https://virtio-fs.gitlab.io/) when I was looking for information on 9p-virtio. Virtiofs is a shared file system that optimizes virtualization by using DAX through shared memory and FUSE instead of 9p, therefore theoretically it would be faster. 

In virt-manager, enable shared memory in the Memories tab, and then choose virtiofs for Driver in the Filesystem tab. Run the machine and type the following to mount it:
```bash
mount -t virtiofs /tag /mount/path
```

Using virtiofs with a Windows guest machine is apparently possible too but I wouldn't deal with all that fuss. Read this [guide](https://virtio-fs.gitlab.io/howto-windows.html) as a reference if you are curious. 

## Samba
I am not a big fan of Samba since sometimes it picks days on whether it wants to work or not. But still, it's a good way to share files, especially with Windows or if you have large files. Samba is an implementation of the SMB protocol for network sharing.

Install the samba suite from your distro's repository. Create a user by running the command `smbpasswd -a samba_user` and type a password for your samba user. Create a folder you want to share. Now go to the configuration file that's usually located at `/etc/samba/smb.conf` and enter the following:
```
[global]
workgroup = WORKGROUP
server string = Samba Server
server role = standalone server
hosts allow = 192.168.122. 127. //subnet and localhost
hosts deny = ALL
security = user
passdb backend = tdbsam
guest account = user //the user you just created
dns proxy = no

[share] //sharename
comment = share file
path = /path/to/shared_folder
guest ok = no
valid users = user //the user you just created
writable = yes
browsable = yes
```

You might need to open up some ports if you are using a firewall. For ufw, enter the following:
```bash
ufw allow CIFS
ufw allow Samba
```

For iptables, enter the following:
```bash
iptables -A INPUT -p udp --dport 137 -j ACCEPT
iptables -A INPUT -p udp --dport 138 -j ACCEPT
iptables -A INPUT -p tcp --dport 139 -j ACCEPT
iptables -A INPUT -p tcp --dport 445 -j ACCEPT
```

Use the command `testparm` to test your config file if needed, start the service with your system's init daemon, and run the guest machine. 

If you are running Windows, head over to the Network tab in File Explorer and type `\\[IP address/hostname]`. Enter your credentials to access the content. 

If you are running Linux with a desktop environment, you can use your default file explorer to access the folder. For Nautilus, go to File -> Connect to Server, choose Windows share from the list, and enter the IP address/hostname; for Dolphin, go to Network and type `smb://[IP address/hostname]`into the top bar. If your file explorer does not offer the option, use the following command:
```bash
mount -t cifs //host_IP/sharename /mount/path -o username=user,password=password
```

To auto-mount the folder, write the following into your `/etc/fstab`:
```bash
//host_IP/sharename	/mount/path	cifs  username=user,password=password,nofail,_netdev	0	0
```

## NFS
NFS is a distributed file protocol that relies on remote procedure calls to provide transparent file access. You could work with it on Windows by enabling the service. It is fairly easy to setup too (~wait didn't I say this for all of them~). 

Install the NFS util package on both the host and the client and create a shared directory.

Now open the config file `/etc/exports` and type the following into it to allow the directory to be shared:
```bash
/shared/directory	192.168.122.1/24(rw,fsid=0,sync,crossmnt)
```

`fsid=0` specifies the root directory, and `crossmnt` saves you some time for not having to mount each child export individually when you have multiple directories to share. You can omit `sync` if you wish to use `async`, the default option that guarantees better performance yet also higher chance of data corruption since it does not wait for data to be completely written to the disk. The address range limits accesses to the specific subnet our virtual machine is on.

Before starting the service, remember to enable the firewall. For ufw:
```bash
ufw allow nfs
```

For iptables:
```bash
iptables -A INPUT -p tcp -m tcp --dport 2049 -j ACCEPT
```

Now start the service and head over to the client. Your desktop environment's default file explorer should have an option for NFS and the process should be similar to that of Samba. To manually mount the folder, use the command:
```bash
mount host_IP:/shared/directory /mount/path
```

To auto-mount the folder, write the following into your `/etc/fstab`:
```bash
host_IP:/shared/directory	/mount/path	nfs	defaults,nofail,_netdev	0	0
```

---

Above is a summary of all the methods I know of on how to share folders. BTW I didn't really test my `fstab` examples but I think they will work. Trust me, okay?