---
title: 'Gentoo Installation (Encryption + Btrfs Subvolume + Multi Boot)'
date: 2021-06-19 20:59:14
tags: [tech]
published: true
hideInList: false
feature: 
isTop: false
---
I have been using Arch as my daily drive for a while and decided to venture a little bit by trying out Gentoo. My first Gentoo installation went semi-successfully, so I want to record my installation process here in case I encounter the same problem in the future. Therefore, this post serves a purpose more of a personal record than a guide. With that being said, you are welcome to follow it.

---

## Before Installation
There are three drives on my computer. One ssd (dev/sda) for Windows and Arch, one hdd (dev/sdb) for shared storage, and a nvme ssd (dev/nvme0n1) with two partitions linked to Windows. I am going to install a **luks encrypted Gentoo with btrfs subvolumes as root** to my nvme drive.   

Since I already have a linux running, there is no need for me to use any additional installation medium. But if you only have Windows, download the [Ubuntu iso](https://ubuntu.com/download/desktop) or any other distro that can access a GUI based browser without installing the system and create a bootable usb stick using [Rufus](https://rufus.ie/en_US/).

---

## Preparing the Disk
Change the user to root for your own convenience. This step is optional.
```bash
sudo -i
```
Open your favorite disk management tool on your Linux and be ready to partition your disk. For me it's fdisk.
```bash
fdisk /dev/nvme0n1 #change it to your disk
```
If you don't know how to use fdisk, follow [this page](https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/Disks#Partitioning_the_disk_with_GPT_for_UEFI). Please notice that I am using **GPT/UEFI** here. After finishing adding partition, you should have a layout similar to the one below. Check your partition layout by using the command `fdisk -l /dev/nvme0n1`
```bash
Device            Start       End   Sectors  Size Type
/dev/nvme0n1p1       34     32767     32734   16M Microsoft reserved
/dev/nvme0n1p2    32768  40992767  40960000 19.5G Microsoft basic data
/dev/nvme0n1p3 40992768  41402367    409600  200M EFI System
/dev/nvme0n1p4 41402368  42426367   1024000  500M Linux filesystem
/dev/nvme0n1p5 42426368 566714367 524288000  250G Linux filesystem
```
`/dev/nvme0n1p3` is the EFI partition required by UEFI boot; `/dev/nvme0n1p4` is the boot partition required by an encrypted root; and `/dev/nvme0n1p5` is where we are going to put the root and home.
You are now ready to encrypt your system. Run the following command.
```bash
cryptsetup luksFormat /dev/nvme0n1p5 #change it to your desired partition
```
Type the password and open it.
```bash
cryptsetup luksOpen /dev/nvme0n1p5 cryptgentoo #or name it anything you want
```
Format your system accordingly.
```bash
mkfs.vfat -F 32 /dev/nvme0n1p3 #change it to your EFI partition
mkfs.ext4 /dev/nvme0n1p4 #change it to your boot partition
mkfs.btrfs /dev/mapper/cryptgentoo #change cryptgentoo to the name you gave earlier
```
Create subvolumes for root and home. You might have to manually create the folder and mount the top volume first. After you have created the subvolume, you can safely unmount.
```
mkdir -p /mnt/gentoo
mount /dev/mapper/cryptgentoo /mnt/gentoo
btrfs subvolume create /mnt/gentoo/gentoo
btrfs subvolume create /mnt/gentoo/gentoo/home
mkdir /mnt/gentoo/home
umount /dev/mapper/cryptgentoo
mount -o subvol=gentoo /dev/mapper/cryptgentoo /mnt/gentoo
```

---

## Installing Gentoo Installation Files
If you are inside the live environment of Ubuntu, the clock should have been already set to UTC time. If not, double check if you have the correct time and adjust it manually if needed.
```bash
date MMDDhhmmYYYY
```
Now you have to make a decision on which stage tarball to choose. I recommend multilib with openrc for most people due to its flexibility. Simply go to [this page](https://www.gentoo.org/downloads/) and click download, or use your favorite tool like `wget` or `lynx`.

The handbook suggests you to verify and validate the stage file. You should do it but I am going to skip this step. Just directly go to the directory and unpack the tarball.
```bash
cd /mnt/gentoo
tar xpvf /path/to/stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
```
In order to optimize gentoo, you need to edit the make.conf. We are not going to make tons of changes here right now since you can always edit it later and activate the changes by `emerge --ask --changed-use --deep @world`. Open `/mnt/gentoo/etc/portage/make.conf` with a text editor and make the following changes.
```bash
COMMON_FLAGS="-march=native -O2 -pipe"
MAKEOPTS="-j<number>" #number of cores plus one
GENTOO_MIRRORS="http://www.gtlib.gatech.edu/pub/gentoo rsync://rsync.gtlib.gatech.edu/gentoo"
VIDEO_CARDS="<your_card>" #intel, nvidia, radeon, vesa
ACCEPT_KEYWORDS="~amd64" #ignore this for faster compiling time
```
Change the mirrors if you are not living in the US. To use `mirrorselect`, you have to chroot to the new environment unless you are using the minimal gentoo installation CD which I adviced against earlier. Find a location that's close to you [here](https://www.gentoo.org/downloads/mirrors/).

We still have a few important things to do before entering the new environment. For a detailed explanation, please resort to the handbook. I am only going to copy the commands.
```bash
mkdir --parents /mnt/gentoo/etc/portage/repos.conf
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
test -L /dev/shm && rm /dev/shm && mkdir /dev/shm
mount --types tmpfs --options nosuid,nodev,noexec shm /dev/shm
chmod 1777 /dev/shm
```
Be sure that none of the commands above returns anything. You will encounter more errors in the Gentoo environment if it does.

Now you can enter the new environment. The last command will add a `(chroot)` label to your hostname. Remember to always stay inside the chroot environment while doing the following steps in this post by checking the label.
```bash
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"
```
Mount the boot and the efi partition. **Do not do it before chrooting**.
```bash
mount /dev/nvme0n1p4 /boot #change it to your boot partition
mount /dev/nvme0n1p3 /boot/efi #change it to your efi partition
```
Install a snapshot of the Gentoo ebuild repository and update it.
```bash
emerge-websync
emerge --sync
```
It's time to choose a profile. First check all the available profiles through `eselect profile list`, then select the profile by `eselect profile set <number>`. For now you can just select the default stable one for a faster compilation time. Otherwise select the `desktop` version.

Update the @world set.
```bash
emerge -auvDN @world
```
Here comes the most Gentoo part of this Gentoo installation post, editing the USE flags. It is not required for you to do all the editing right now, so no worries if you don't know what you want to enable or disable. If you are using anything other than gnome or kde for desktop environment, I would encourage you to disable them. A full list of USE flags can be found [here](https://www.gentoo.org/support/use-flags/).

I am skipping the license part for now. Next you need to set the timezone and locales.
```bash
ls /usr/share/zoneinfo
echo "your/timezone" > /etc/timezone
emerge --config sys-libs/timezone-data
echo en_US.UTF-8 UTF-8 > /etc/locale.gen #add more if you like
locale-gen
```
Set the system-wide locale settings by first checking the available targets using `eselect locale list`, then choose the locale by `eselect locale set <number>`. Reload the environment after you finish.
```bash
env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
```

---

## Configuring the Kernel

For the full Gentoo experience, you are welcome to configure the kernel manually. But keep in mind that it is a time-consuming process to find all your hardwares and the corresponding option. So for a compromised solution, I'm going to use genkernel with the `--menuconfig` option. This allows me to enjoy the benefits of an automatically configured kernel while not having to enable every possible option. Lovely.

Emerge all the required packages. The second line emerges packages that might be useful later.
```bash
emerge --ask sys-kernel/gentoo-sources genkernel
emerge --ask cryptsetup lvm2 btrfs-progs ntfs3g
```
If the output asks you to review the changed config files, do so by running `dispatch-conf` then retyping the commands above. To create a symbolic link, run the commands below.
```bash
eselect kernel list
eselect kernel set 1
```
By then you should have a symbolic link called `linux` pointing to the kernel source. Check by listing it then changing your directory to it..
```bash
ls -l /usr/src/linux
cd /usr/src/linux
```
Before generating the kernel, there are two things to do. First add your boot partition to `/etc/fstab`. I recommend using the UUID and it should be like this. You can find the UUID by running the command `blkid`.
```bash
#/dev/nvme0n1p4
UUID=<UUID>	/boot	ext4	defaults,noatime	0 2
```
Next, be sure that **you have the corresponding package for the kernel compression mode you are going to use**. Let's take lz4 as an example. Check it by running emerge with the pretend option.
```bash
emerge -p lz4
```
If it is marked with an `N`, you need to install it. You don't need to do anything if it is marked with a `R`, which means replacing.

Now run genkernel with all the options below.
```bash
genkernel --lvm --luks --btrfs --menuconfig --save-config all
```
I know we are not using lvm here, but you might have to add it and also enable it in your kernel in order to unlock your luks device when you boot. The last option will save your configuration and use that file the next time you run genkernel.

When the menu shows up, do the following changes. **Remember that M and * are different**.
```bash
#btrfs support
File systems  --->
    <*> Btrfs filesystem

#luks support
[*] Enable loadable module support
Device Drivers --->
    [*] Multiple devices driver support (RAID and LVM) --->
        <*> Device mapper support
        <*>   Crypt target support
[*] Cryptographic API --->
    <*> XTS support
    <*> SHA224 and SHA256 digest algorithm
    <*> AES cipher algorithms
    <*> AES cipher algorithms (x86_64)
    <*> User-space interface for hash algorithms
    <*> User-space interface for symmetric key cipher algorithms
General setup  --->
    [*] Initial RAM filesystem and RAM disk (initramfs/initrd) support

#lvm support
Device Drivers  --->
   Multiple devices driver support (RAID and LVM)  --->
       <*> Device mapper support
           <*> Crypt target support
           <*> Snapshot target
           <*> Mirror target
           <*> Multipath target
               <*> I/O Path Selector based on the number of in-flight I/Os
               <*> I/O Path Selector based on the service time

#ntfs support
File systems  --->
    DOS/FAT/NT Filesystems  --->
        <*> NTFS file system support
        <*>   NTFS write support
		<*> FUSE (Filesystem in Userspace) support

#nvme support
Device Drivers →
  NVME Support →
    <*> NVM Express block device
    [*] NVMe multipath support
    [*] NVMe hardware monitoring
    <M> NVM Express over Fabrics FC host driver
    <M> NVM Express over Fabrics TCP host driver
    <M> NVMe Target support
          [*]   NVMe Target Passthrough support
        <M>   NVMe loopback device support
        <M>   NVMe over Fabrics FC target driver
        < >     NVMe over Fabrics FC Transport Loopback Test driver (NEW)
        <M>   NVMe over Fabrics TCP target support
```
For optimization, check out [Mental Outlaw's video](https://youtu.be/NVWVHiLx1sU). After finished editing, save and exit. Genkernel will do the rest for you.

The handbook masked it optional but I recommend you to install the firmware now.
```bash
emerge --ask sys-kernel/linux-firmware
```

## Configure the System

Add a password to the root account by simply typing `passwd`. To add a new user, do the following
```bash
useradd -m -G users,wheel,audio,video,tty -s /bin/bash <user>
passwd <user>
```
For sudo usage, install the sudo package and uncomment the required line in the configuration file. I'm not elaborating here.

In order for your system to automatically mount the necessary partitions, everything has to be specified in `/etc/fstab`. I will give mine as an example.
```bash
#/dev/nvme0n1p4
UUID=<UUID>	/boot	ext4	defaults,noatime	0 2
#/dev/mapper/cryptgentoo
UUID=<UUID>	/	btrfs	subvol=gentoo,subvolid=<subvolid>,relatime,rw   0 1
#/dev/mapper/cryptgentoo
UUID=<UUID>	/home	btrfs	subvol=/gentoo/home,subvolid=<subvolid>,relatime,rw 0 2
#/dev/sdb3
UUID=<UUID>	/data_shared	ntfs-3g	gid=<gid>,uid=<uid>,dmask=022,fmask=133,big_writes,windows_names	0 0
#/dev/nvme0n1p3
UUID=<UUID>	/boot/efi	vfat	defaults	0	0
```
It's hella messy, but you get the point. `/data_shared` is for my shared ntfs drive. Find the id by typing `id <user>`. `subvolid` is not required, but you can find it through `btrfs subvolume list /mnt`.

Change the hostname in `/etc/conf.d/hostname` and edit the `/etc/hosts` file as followed.
```bash
# IPv4 and IPv6 localhost aliases
127.0.0.1	<hostname>.localdomain localhost
::1		localhost	<hostname>
```

If you are using ethernet, follow [this part](https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/System#Automatically_start_networking_at_boot) of the handbook. If you are using wifi, you have some additional steps to do. I installed wpa_supplicant and networkmanager just in case. The guides for them are clear enough. **Both ethernet and wifi users need the net-misc/dhcpcd package in most cases**.

Install a system logger and a cron daemon if you like.
```bash
emerge -a syslog-ng cronie
rc-update add syslog-ng default
rc-update add cronie default
```

---

## Configuring the Bootloader

Echo the following to the corresponding file and add dmcrypt to openRC before emerging grub.
```bash
echo 'GRUB_PLATFORMS="efi-64"' >> /etc/portage/make.conf
echo "sys-boot/grub:2 device-mapper" >> /etc/portage/package.use/sys-boot
rc-update add dmcrypt boot
emerge -a sys-boot/grub:2
```
Edit the grub configuration file so your luks with btrfs subvolume setup works. Open `/etc/default/grub` and add this line. The first UUID is your raw partition (e.g. /dev/nvme0n1p5), and the second UUID is your opened luks device (e.g. /dev/mapper/cryptgentoo).
```bash
GRUB_CMDLINE_LINUX="crypt_root=UUID=<UUID_of_your_luks_partition> real_root=UUID=<UUID_of_your_decrypted_luks_device> rootflags=subvol=gentoo rootfstype=btrfs dolvm dobtrfs quiet"
```
Also you might want to uncomment `GRUB_TIMEOUT=` and add a number (in seconds). This way it gives you more time to choose a system before the screen flashes to the first boot entry.

Install grub. Make sure that your EFI partition is mounted. **Do not forget the `--removable` option**, otherwise it might not appear on your boot menu.
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --removable /dev/nvme0n1 #change it to your disk
```
Generate the configuration. Ignore the os-prober warning unless you need it to detect Windows that's on the same disk.
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```
Time to reboot! Yeahhhhhhh
```bash
exit
cd
umount -l /mnt/gentoo/dev{/shm,/pts,}
umount -R /mnt/gentoo
reboot
```
Don't forget to remove your usb device if you are using one.

---

## Troubleshooting

**Problem**: *Block device is not a valid root device*
**Solution**: Double check that you have all the options related to luks and lvm enabled in your kernel, the `--lvm` option is passed when generating an initramfs, dmcrypt is added to rc-service, and your `/etc/default/grub` file is like what I have above. If mine doesn't work, try some alternating options like `root=UUID=<UUID>`, `rd.luks.uuid=<UUID>`, etc. Keep trying until you get a different error.

**Problem**: *Failed to open LUKS device /dev/xxx*
**Solution**: Try everything above. If nothing works, try removing the cryptsetup package from the @world using the command below but do not depclean. Really weird and dumb but it worked in my case. Re-emerge it later after you manage to enter the system.
```bash
emerge --deselect cryptsetup @world
```

**Problem**: *Invalid EFI path*
**Solution**: This should not happen at all. Are you sure you are on the UEFI mode? If you get this for booting Windows, did you manually add the BIOS mode chainloader entry instead of using os-prober? Or if you are trying to chainload your Gentoo machine to the grub on the other disk or vise versa but it doesn't work? For the last scenario, let me present to you a simpler method. Just copy the entry on the `grub.cfg` (should be right after `### BEGIN /etc/grub.d/10_linux ###`) and paste it in `/etc/grub.d/40_custom` for the other grub. Regenerate grub afterwards.

---

I have been using Gentoo for a week or so after I finished writing this. My thoughts? Fuck /g/entoo I'm back to sweet home arch. It is indeed a good distro if you are all for minimalism and customization, but sorry I am not really interested in turning my room into an airport when compiling firefox. 
