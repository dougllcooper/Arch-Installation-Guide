# A Personal Arch Installation Guide

Had a text version of this I made years ago but found this and it's very well done and alot easier to read than an all text version.  Commands are easier to see.  Modifications include setting up a btrfs filesystem.  Also setup snapper and snap-pac-grub.  This allow automatic snapshots by snapper and pacman creates Pre and Post snapshots.  Snap-pac-grub also allows you to boot into the pacman created snapshots.  Also an additional hook that backs up the boot directory if you install a new kernel.

This is a personal guide so if you are lost and just found this guide from somewhere, I recommend you to read the official [`wiki`](https://wiki.archlinux.org/index.php/Installation_guide)!  This guide will focus on `systemd-boot`, `UEFI` and a guide if you want to encrypt your partition with `LUKS/LVM`. This guide exists so that I can remember a bunch of things when reinstalling `Archlinux`.

## Pre-installation

Before installing, make sure to:

+ Read the [official wiki](https://wiki.archlinux.org/index.php/installation_guide). It is advisable to read that instead. I wrote this guide for myself.
+ Acquire an installation image from [here](https://www.archlinux.org/download/).
+ Verify signature.
+ Prepare an installation medium.
+ Boot the live environment.

## Set the keyboard layout

The default console keymap is US. Available layouts can be listed with:

```
# ls /usr/share/kbd/keymaps/**/*.map.gz
```

To modify the layout, append a corresponding file name to loadkeys, omitting path and file extension. For example, to set a US keyboard layout:  

```
# loadkeys us
```

## Verify the boot mode

If UEFI mode is enabled on an UEFI motherboard, Archiso will boot Arch Linux accordingly via systemd-boot. To verify this, list the efivars directory:  

```
# ls /sys/firmware/efi/efivars
```

If the command shows the directory without error, then the system is booted in UEFI mode. If the directory does not exist, the system may be booted in **BIOS** (or **CSM**) mode.

## Connect to the internet

We need to make sure that we are connected to the internet to be able to install Arch Linux `base` and `linux` packages. Let’s see the names of our interfaces.

```
# ip link
```

You should see something like this:

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
		link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state DOWN mode DEFAULT group default qlen 1000
		link/ether 00:00:00:00:00:00 brd ff:ff:ff:ff:ff:ff
3: wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DORMANT group default qlen 1000
		link/ether 00:00:00:00:00:00 brd ff:ff:ff:ff:ff:ff permaddr 00:00:00:00:00:00
```

+ `enp0s0` is the wired interface  
+ `wlan0` is the wireless interface  

### Wired Connection

If you are on a wired connection, you can enable your wired interface by systemctl start `dhcpcd@<interface>`.  

```
# systemctl start dhcpcd@enp0s0
```

### Wireless Connection

If you are on a laptop, you can connect to a wireless access point using `iwctl` command from `iwd`. Note that it's already enabled by default. Also make sure the wireless card is not blocked with `rfkill`.

Scan for network.

```
# iwctl station wlan0 scan
```

Get the list of scanned networks by:

```
# iwctl station wlan0 get-networks
```

Connect to your network.

```
# iwctl -P "PASSPHRASE" station wlan0 connect "NETWORKNAME"
```

Ping archlinux website to make sure we are online:

```
# ping archlinux.org
``` 

If you receive Unknown host or Destination host unreachable response, means you are not online yet. Review your network configuration and redo the steps above.

## Update the system clock

Use `timedatectl` to ensure the system clock is accurate:

```
# timedatectl set-ntp true
```

To check the service status, use `timedatectl status`.

### Find fastest mirror

Use reflector to test mirrors:

```
reflector -c Canada,US -p https -f 10 --sort rate --save /etc/pacman.d/mirrorlist
```

## Partition the disks

When recognized by the live system, disks are assigned to a block device such as `/dev/sda`, `/dev/nvme0n1` or `/dev/mmcblk0`. To identify these devices, use lsblk or fdisk.  The most common main drive is **sda**.

```
# lsblk
```

Results ending in `rom`, `loop` or `airoot` may be ignored.

In this guide, I'll create a two different ways to partition a drive. One for a normal installation, the other one is setting up with an encryption(LUKS/LVM). Let's start with the unecrypted one:

### Btrfs filesystem setup

This section sets up a btrfs filesystem to install arch onto.  Setting up partitions for btrfs is easy.  You need an efi partition, a swap partition if you're going to use one, and everything else in one large partition:

```
fdisk /dev/sda
```
+ `g` to setup a new gpt partition table
+ `n` to create a new partition
+ `Enter` twice to accept defaults for partition # and starting sector
+ `+512M` to create a 512M efi partiton
+ `t` to select partition type
+ `1` type 1 is for an EFI partition
+ I usually have swap partiton on another drive or create one here
+ `n` for new partition
+ `Enter` twice
+ `+8G` or whatever for swap partition
+ `t` to select type
+ `19` is linux swap type
+ `n` for new partition
+ `Enter` three times
+ This creates a large partition for the rest of the drive
+ In this example, this would be /dev/sda3
+ We'll use this for the btrfs subvolumes

### Partitions example

In this example we'll use the following partition setup:
```
/dev/sda1 - efi
/dev/sda2 - swap
/dev/sda3 - btrfs filesystem
```

#### Formating partitions

#### EFI partition

EFI has to be formatted Fat32.
`mkfs.fat -F32 /dev/sda1`

#### Enable swap
Make and enable swap
```
mkswap /dev/sda2
swapon /dev/sda2
```

### Format btrfs partition, mount partition, create subvolumes, and umount

```
mkfs.btrfs /dev/sda3
mount /dev/sda3 /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@var
unmount /mnt
```

@ is used for root
@ home is home directory obviously
@snapshots is used by snapper for all snapshots
@var is the var directory.  Used to have separate subvolumes for /var/log, /var/tmp, and /var/cache but I guess that's not the in-thing to do anymore.  It's acceptable now to use one subvolume for var.  This is because if you do an update and it breaks something, you can boot up and rollback to a previous snapshot but it doesn't roll back your var directory.  System logs and stuff like that can be used to see what went wrong. Also holds pacman cache and tmp directory which is a waste of time and space to snapshot anyway.

### Now we mount the subvolumes

The whole idea here is that we mount the root subvolume, create subdirectories needed, and then mount the rest of the subvolumes.

```
mount -o rw,ssd,noatime,compress=zstd:3,space_cache=v2,discard=async,subvol=@ /dev/sda3 /mnt
```
####Mount options:
+ rw - mount read/write
+ ssd - use this if drive is an ssd
+ noatime - better for ssd's. Not as many writes.
+ compress=zstd:3 - Good speed and decent compression
+ space_cache=v2 - improves performance by caching empty blocks.
+ discard=async - use for ssd's

#### Create directories and mount rest of subvolumes

```
mkdir /mnt/{boot,home,.snapshots,var}
mount -o rw,ssd,noatime,compress=zstd:3,space_cache=v2,discard=async,subvol=@home /dev/sda3 /mnt/home
mount -o rw,ssd,noatime,compress=zstd:3,space_cache=v2,discard=async,subvol=@snapshots /dev/sda3 /mnt/.snapshots
mount -o rw,ssd,noatime,compress=zstd:3,space_cache=v2,discard=async,subvol=@var /dev/sda3 /mnt/var
```

#### Mount EFI partition on /mnt/boot

```
mount /dev/sda1 /mnt/boot
```

### If using btrfs, skip to `Installation` section in use pacstrap

### Unencrypted filesystem

+ Let’s clean up our main drive to create new partitions for our installation. And yeah, in this guide, we will use `/dev/sda` as our disk.

	```
	# gdisk /dev/sda
	```

+ Press <kbd>x</kbd> to enter **expert mode**. Then press <kbd>z</kbd> to *zap* our drive. Then hit <kbd>y</kbd> when prompted about wiping out GPT and blanking out MBR. Note that this will ***zap*** your entire drive so your data will be gone - reduced to atoms after doing this. THIS. CANNOT. BE. UNDONE.

+ Open `cgdisk` to start partitioning our filesystem

	```
	# cgdisk /dev/sda
	```

+ Press <kbd>Return</kbd> when warned about damaged GPT.

	Now we should be presented with our main drive showing the partition number, partition size, partition type, and partition name. If you see list of partitions, delete all those first.

+ Create the `boot` partition

	- Hit New from the options at the bottom.
	- Just hit enter to select the default option for the first sector.
	- Now the partion size - Arch wiki recommends 200-300 MB for the boot + size. Let’s make 1GiB in case we need to add more OS to our machine. I’m gonna assign mine with 1024MiB. Hit enter.
	- Set GUID to `EF00`. Hit enter.
	- Set name to `boot`. Hit enter.
	- Now you should see the new partition in the partitions list with a partition type of EFI System and a partition name of boot. You will also notice there is 1007KB above the created partition. That is the MBR. Don’t worry about that and just leave it there.

+ Create the `root` partition

	- Hit New again.
	- Hit enter to select the default option for the first sector.
	- Hit enter again to input your root size.
	- Also hit enter for the GUID to select default(`8300`).
	- Then set name of the partition to `root`.

+ Create the `root` partition

	- Hit New again.
	- Hit enter to select the default option for the first sector.
	- Hit enter again to use the remainder of the disk.
	- Also hit enter for the GUID to select default.
	- Then set name of the partition to `home`.

+ Lastly, hit `Write` at the bottom of the patitions list to *write the changes* to the disk. Type `yes` to *confirm* the write command. Now we are done partitioning the disk. Hit `Quit` *to exit cgdisk*. Go to the [next section](#formatting-partitions).

### Encrypted filesystem with `LUKS/LVM`

+ Let’s clean up our main drive to create new partitions for our installation. And yeah, in this guide, we will use `/dev/sda` as our disk.

	```
	# gdisk /dev/sda
	```

+ Press <kbd>x</kbd> to enter **expert mode**. Then press <kbd>z</kbd> to *zap* our drive. Then hit <kbd>y</kbd> when prompted about wiping out GPT and blanking out MBR. Note that this will ***zap*** your entire drive so your data will be gone - reduced to atoms after doing this. THIS. CANNOT. BE. UNDONE.

+ Create our partitions by running `cgdisk /dev/sda`

	```
	# cgdisk /dev/sda
	```

+ Just press <kbd>Return</kbd> when warned about damaged GPT.

	Now we should be presented with our main drive showing the partition number, partition size, partition type, and partition name. If you see list of partitions, delete all those first.

+ Create the `boot` partition

	- Hit New from the options at the bottom.
	- Just hit enter to select the default option for the first sector.
	- Now the partion size - Arch wiki recommends 200-300 MB for the boot + size. Let’s make 1GiB in case we need to add more OS to our machine. I’m gonna assign mine with 1024MiB. Hit enter.
	- Set GUID to `EF00`. Hit enter.
	- Set name to `boot`. Hit enter.
	- Now you should see the new partition in the partitions list with a partition type of EFI System and a partition name of boot.

+ Create the `LVM` partition

	- Hit New again.
	- Hit enter to select the default option for the first sector.
	- Hit enter again to use the remainder of the disk.
	- Set GUID to `8e00`. Hit enter.
	- Set name to `lvm`. Hit enter.

+ Lastly, hit `Write` at the bottom of the patitions list to *write the changes* to the disk. Type `yes` to *confirm* the write command. Now we are done partitioning the disk. Hit `Quit` *to exit cgdisk*. Go to the [next section](#formatting-partitions).
```
```

## Verifying the partitions

Use `lsblk` again to check the partitions we created. *We? I thought I'm doing this guide for myself lol*

```
# lsblk
```

You should see *something like this*:

### Unencrypted filesystem

| NAME | MAJ:MIN | RM | SIZE | RO | TYPE | MOUNTPOINT |
| --- | --- | --- | --- | --- | --- | --- |
| sda | 8:0 | 0 | 477G | 0 |   |   |
| sda1 | 8:1 | 0 | 1 | 0 | part |   |
| sda2 | 8:2 | 0 | 1 | 0 | part |   |
| sda3 | 8:3 | 0 | 175G | 0 | part |   |

**`sda`** is the main disk  
**`sda1`** is the boot partition  
**`sda2`** is the swap partition  
**`sda3`** is the home partition  

### Encrypted filesystem

| NAME | MAJ:MIN | RM | SIZE | RO | TYPE | MOUNTPOINT |
| --- | --- | --- | --- | --- | --- | --- |
| sda | 8:0 | 0 | 477G | 0 | disk |   |
| sda1 | 8:1 | 0 | 1 | 0 | part |   |
| sda2 | 8:2 | 0 | 1 | 0 | part |   |

**`sda`** is the main disk  
**`sda1`** is the boot partition  
**`sda2`** is the LVM partition

**Surprise! Surprise!** We will **not** encrypt the `/boot` partition.

## Format the partitions

### Unencrypted filesystem

+ Format `/dev/sda1` partition as `FAT32`. This will be our `/boot`.

	```
	# mkfs.fat -F32 /dev/sda1
	```

+ Format `/dev/sda3` and `/dev/sda4` partition as `EXT4`. This will be our `root` and `home`  partition.

	```
	# mkfs.ext4 /dev/sda3
	# mkfs.ext4 /dev/sda4
	```

### Encrypted filesystem

+ Format `/dev/sda1` partition as `FAT32`. This will be our `/boot`.

	```
	# mkfs.fat -F32 /dev/sda1
	```

+ Create the LUKS encrypted container.

	```
	# cryptsetup luksFormat /dev/sda2
	```

+ Enter your passphrase twice. Don't forget this!

+ Open the created container and name it whatever you want. In this guide I'll just use `cryptlvm`.

	```
	# cryptsetup open --type luks /dev/sda2 cryptlvm
	```

+ Enter your passphrase and verify it.

+ The decrypted container is now available at `/dev/mapper/cryptlvm`.

+ Create a physical volume on top of the opened LUKS container:

	```
	# pvcreate /dev/mapper/cryptlvm
	```

+ Create the volume group and name it `volume` (or whatever you want), adding the previously created physical volume to it:

	In this guide, I'll just use `volume` as the volume group name.

	```
	# vgcreate volume /dev/mapper/cryptlvm
	```

+ Create all your needed logical volumes on the volume group. We will create `root` and `home` logical volumes. Note that the `volume` is the name of the volume we just created.

	- Create our `root`. In this guide, I'll use 100GB.

		```
		# lvcreate -L 100G volume -n root
		```

		This will create `/dev/mapper/volume-root`.

	- Create our home sweet `home`. I'll just assign the remaining space to it.

		```
		# lvcreate -l 100%FREE volume -n home
		```

	This will create `/dev/mapper/volume-home`.

+ Format the logical partitions under the LVM volume.

	- Format our `root` and `home` partitions.

		```
		# mkfs.ext4 /dev/mapper/volume-root
		# mkfs.ext4 /dev/mapper/volume-home
		```

## Mount the filesystems

### Unencryped partition

+ Mount the `/dev/sda` partition to `/mnt`. This is our `/`:

	```
	# mount /dev/sda3 /mnt
	```

+ Create a `/boot` mountpoint:

	```
	# mkdir /mnt/boot  
	```

+ Mount `/dev/sda1` to `/mnt/boot` partition. This is will be our `/boot`:

	```
	# mount /dev/sda1 /mnt/boot
	```

+ Create a `/home` mountpoint:

	```
	# mkdir /mnt/home  
	```

+ Mount `/dev/sda4` to `/mnt/home` partition. This is will be our `/home`:

	```
	# mount /dev/sda1 /mnt/home
	```

### Encrypted partition

+ Mount the `/dev/mapper/volume-root` partition to `/mnt`. This is our `/`:

	```
	# mount /dev/mapper/volume-root /mnt
	```

+ Create a `/boot` mountpoint:

	```
	# mkdir /mnt/boot  
	```

+ Mount `/dev/sda1` to `/mnt/boot` partition. This is will be our `/boot`:

	```
	# mount /dev/sda1 /mnt/boot
	```

+ Create a `/home` mountpoint:

	```
	# mkdir /mnt/home  
	```

+ Mount `/dev/mapper/volume-home` to `/mnt/home` partition. This is will be our `/home`:

	```
	# mount /dev/mapper/volume-home /mnt/home
	```

	 We don’t need to mount `swap` since it is already enabled.
```
```
## Installation

Now let’s go ahead and install `base`, `linux`, `linux-firmware`, and `base-devel` packages into our system. 

+ linux, linux-lts, linux-zen:  Generally install one of these. I also install headers as well.

```
# pacstrap /mnt base base-devel linux linux-zen linux-lts linux-firmware linux-headers linux-lts-headers linux-zen-headers
```

I will install `linux-zen` since it has necessary modules for gaming.

Another option I usually use is `linux-lts`. This installs long-term-support kernel which gives a little more stability for archlinux.

This section also sets up zram.  If going this route you wouldn't need a swap file.  Didn't normally use zram, but maybe I'll give it a try. For swap, just use the appropriate sections. Swap partition, swap file, or zram.

The `base` package does not include all tools from the live installation, so installing other packages may be necessary for a fully functional base system. In particular, consider installing: 

+ software necessary for networking,

	- `dhcpcd`: RFC2131 compliant DHCP client daemon
	- `iwd`: Internet Wireless Daemon
	- `inetutils`: A collection of common network programs
	- `iputils`: Network monitoring tools, including `ping`
	- `wpa_supplicant`: always installed this just as a backup
	- `wireless_tools`: tools for wireless - go figure
	- `networkmanager`: I like networkmanager for internet connections

+ utilities for accessing `RAID` or `LVM` partitions,

	- `lvm2`: Logical Volume Manager 2 utilities (*if you are setting up an encrypted filesystem with LUKS/LVM, include this on pacstrap*)

+ btrfs filesystem,

	- `btrfs-progs`: programs for handling btrfs filesystems
	- `snapper`: might as well install now. Will be needed shortly
	- `grub2`: I use grub bootloader. Allows booting using pacman created snapshots
	- `efibootmgr`: for efi

+ Zram

	- `zram-generator`

+ a text editor(s),

	- `nano`
	- `vim`
	- `vi`
	- `neovim`

+ packages for accessing documentation in man and info pages,

	- `man-db`
	- `man-pages`

+ Microcode

	- `intel-ucode`/`amd-ucode`

+ tools:

	- `git`: the fast distributed version control system
	- `tmux`: A terminal multiplexer
	- `less`: A terminal based program for viewing text files
	- `usbutils`: USB Device Utilities
	- `bash-completion`: Programmable completion for the bash shell
	- `zsh`: I usually use zsh

+ userspace utilities for the management of file systems that will be used on the system,
	
	- `ntfs-3g`: NTFS filesystem driver and utilities
	- `unrar`: The RAR uncompression program
	- `unzip`: For extracting and viewing files in `.zip` archives
	- `p7zip`: Command-line file archiver with high compression ratio
	- `unarchiver`: `unar` and `lsar`: Objective-C tools for uncompressing archive files
	- `gvfs-mtp`: Virtual filesystem implementation for `GIO` (`MTP` backend; Android, media player)
	- `libmtp`: Library implementation of the Media Transfer Protocol
	- `android-udev`: Udev rules to connect Android devices to your linux box
	- `mtpfs`: A FUSE filesystem that supports reading and writing from any MTP devic
	- `xdg-user-dirs`: Manage user directories like `~/Desktop` and `~/Music`
	- `dosfstools`: for FAT filesystems
	- `mtools`: tools to handle ms-doc and win partitions without mounting them.

These tools will be useful later. So **future me**, install these.

## Generating the fstab

```
# genfstab -U /mnt >> /mnt/etc/fstab
```

Check the resulting `/mnt/etc/fstab` file, and edit it in case of errors. 

## Chroot

Now, change root into the newly installed system  

```
# arch-chroot /mnt /bin/bash
```

## Time zone

A selection of timezones can be found under `/usr/share/zoneinfo/`. Since I am in the Philippines, I will be using `/usr/share/zoneinfo/Asia/Manila`. Select the appropriate timezone for your country:

```
# ln -sf /usr/share/zoneinfo/America/Toronto /etc/localtime
```

Run `hwclockz` to generate `/etc/adjtime`: 

```
# hwclock --systohc
```

This command assumes the hardware clock is set to UTC.

## Localization

The `locale` defines which language the system uses, and other regional considerations such as currency denomination, numerology, and character sets. Possible values are listed in `/etc/locale.gen`. Uncomment `en_US.UTF-8`, as well as other needed localisations.

**Uncomment** `en_US.UTF-8 UTF-8` and other needed locales in `/etc/locale.gen`, **save**, and generate them with:  

```
# locale-gen
```

Create the `locale.conf` file, and set the LANG variable accordingly:  

```
# locale > /etc/locale.conf
```

If you set the keyboard layout earlier, make the changes persistent in `vconsole.conf`:

```
# echo "KEYMAP=us" > /etc/vconsole.conf
```

Not using `us` layout? Replace it, stoopid.

## Network configuration

Create the hostname file. In this guide I'll just use `MYHOSTNAME` as hostname. Hostname is the host name of the host. Every 60 seconds, a minute passes in Africa.

```
# echo "MYHOSTNAME" > /etc/hostname
```

Open `/etc/hosts` to add matching entries to `hosts`:

```
127.0.0.1    localhost  
::1          localhost  
127.0.1.1    MYHOSTNAME.localdomain	  MYHOSTNAME
```

If the system has a permanent IP address, it should be used instead of `127.0.1.1`.

## Initramfs  

Creating a new initramfs is usually not required, because mkinitcpio was run on installation of the kernel package with pacstrap. **This is important** if you are setting up a system with encryption!

### Unencrypted filesystem

	```
	# mkinitcpio -P
	```

	DO NOT FORGET TO RUN THIS BEFORE REBOOTING YOUR SYSTEM!

### Encrypted filesystem with LVM/LUKS

+ Open `/etc/mkinitcpio.conf` with an editor:

+ For Btrfs filesystem: at the top of mkinitcpio.conf is a line
	+ `MODULES=()`
	+ change this to
	+ `MODULES=(btrfs)`
+ Also for btrfs, remove fsck from HOOKS= line:
	+ Before
	+ `HOOKS=(base udev autodetect keyboard modconf block encrypt lvm2 filesystems fsck)`
	+ After
	+ `HOOKS=(base udev autodetect keyboard modconf block encrypt lvm2 filesystems)`
	+ You cannot do fsck on btrfs partition or it will corrupt data
+ 
+ In this guide, there are two ways to setting up initramfs, `udev` (default) and `systemd`. If you are planning to use `plymouth`(splashcreen), it is advisable to use a `systemd`-based initramfs.

	- udev-based initramfs (default).

		Find the `HOOKS` array, then change it to something like this:

		```
		HOOKS=(base udev autodetect keyboard modconf block encrypt lvm2 filesystems fsck)
		```

	- systemd-based initramfs.

		Find the `HOOKS` array, then change it to something like this:

		```
		HOOKS=(base systemd autodetect keyboard sd-vconsole modconf block sd-encrypt lvm2 filesystems fsck)
		```

	- Regenerate initramfs image:

		```
		# mkinitcpio -P
		```

		DO NOT FORGET TO RUN THIS BEFORE REBOOTING YOUR SYSTEM!

### Making Swap File and ZSwap

#### Time to create a swap file! I'll make two gigabytes swap file.

```
# dd if=/dev/zero of=/swapfile bs=1M count=2048 status=progress
```

Set the right permissions
```
# chmod 0600 /swapfile
```

After creating the correctly sized file, format it to swap:
```
# mkswap -U clear /swapfile
```

Activate the swap file
```
# swapon /swapfile
```

Finally, edit the fstab configuration to add an entry for the swap file in `/etc/fstab`:
```
/swapfile none swap defaults,pri=10 0 0
```

#### Install zram-generator:

```
# pacman -S zram-generator
```

Let's make a config file at `/etc/systemd/zram-generator.conf
!` I prefer having HALF of my TOTAL RAM as zswap size. My laptop have 4 cores, so I'll distribute it to FOUR zram devices. So I'll uthis config :

```
[zram0]
zram-size = ram/8
compression-algorithm = zstd
swap-priority = 100

[zram1]
zram-size = ram/8
compression-algorithm = zstd
swap-priority = 100

[zram2]
zram-size = ram/8
compression-algorithm = zstd
swap-priority = 100

[zram3]
zram-size = ram/8
compression-algorithm = zstd
swap-priority = 100
```

No need to enable/start anything, it will automatically initialize zram devices! Just reboot and run `swapon -s` to check the swap you have.

## Adding Repositories - `multilib` and `AUR`

Enable multilib and AUR repositories in `/etc/pacman.conf`. Open it with your editor of choice:

### Adding multilib repository

Uncomment `multilib` (remove # from the beginning of the lines). It should look like this:  

```
[multilib]
Include = /etc/pacman.d/mirrorlist
```

### Adding the AUR repository

Add the following lines at the end of your `/etc/pacman.conf` to enable the AUR repo:  

```
[archlinuxfr]
SigLevel = Never
Server = http://repo.archlinux.fr/$arch
```

### `pacman` goodies

You can enable the "easter-eggs" and goodies in `pacman`, the package manager of archlinux.

Open `/etc/pacman.conf`, then find `# Misc options`. 

To add colors to `pacman`, uncomment `Color`. Then add `Pac-Man` to `pacman` by adding `ILoveCandy` under the `Color` string. To enable parallel downloads, uncomment it too:

```
Color
ILoveCandy
ParallelDownloads = 3
```

### Update repositories and packages

To check if you successfully added the repositories and enable the easter-eggs, run:

```
# pacman -Syu
```

If updating returns an error, open the `pacman.conf` again and check for human errors. Yes, you f'ed up big time.

## Root password

Set the `root` password:  

```
# passwd
```

## Add a user account

Add a new user account. In this guide, I'll just use `MYUSERNAME` as the username of the new user aside from `root` account. (My phrasing seems redundant, eh?) Of course, change the example username with your own:  

```
# useradd -m -g users -G wheel,storage,power,video,audio,rfkill,input -s /bin/bash MYUSERNAME
```

This will create a new user and its `home` folder.

Set the password of user `MYUSERNAME`:  

```
# passwd MYUSERNAME
```

## Add the new user to sudoers:

If you want a root privilege in the future by using the `sudo` command, you should grant one yourself:

```
# EDITOR=nvim visudo
```

Uncomment the line (Remove #):

```
# %wheel ALL=(ALL) ALL
```

### Install boot loader if using btrfs

Using btrfs I use brub bootloader. This allow booting to snapshots that are created by snapper as well as pacman generated snapshots

```
grub2-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=arch
```

This should complete with no errors. Then make the config file:

`grub2-mkconfig -o /boot/grub/grub.mkconfig`

## Install the boot loader (skip for btrfs)

Yeah, this is where we install the bootloader. We will be using `systemd-boot`, so no need for `grub2`. 

+ Install bootloader:
	
	We will install it in `/boot` mountpoint (`/dev/sda1` partition).

	```
	# bootctl --path=/boot install
	```

+ Create a boot entry `/boot/loader/entries/arch.conf`, then add these lines:

### Unencrypted filesystem

	```
	title Arch Linux  
	linux /vmlinuz-linux  
	initrd  /initramfs-linux.img  
	options root=/dev/sda3 rw
	```

	If your `/` is not in `/dev/sda3`, make sure to change it. 

	Save and exit.

### Encrypted filesystem

Remember the two-types of initramfs earlier? Each type needs a specific kernel parameters. So there's also a two type of entries here. Remember that `volume` is the volume group name and `/dev/mapper/volume-root` is the path to `/`.

+ udev-based initramfs

	```
	title Arch Linux  
	linux /vmlinuz-linux  
	initrd  /initramfs-linux.img  
	options cryptdevice=UUID=/DEV/SDA2/UUID/HERE:volume root=/dev/mapper/volume-root rw
	```

	Replace `/DEV/SDA2/UUID/HERE` with the UUID of your `LVM` partition. You can check it by running `blkid /dev/sda2`. Note that `cryptdevice` parameter  is unsupported by plymouth so it's advisable to use systemd-based initramfs if you are planning to use it.

	Tip: If you are using `vim`, you can write the UUID easier by typing `:read ! blkid /dev/sda2` then hit enter. Then manipulate the output by using visual mode.

+ systemd-based initramfs

	```
	title Arch Linux
	linux /vmlinuz-linux
	initrd /intel-ucode.img
	initrd /initramfs-linux.img
	options rd.luks.name=/DEV/SDA2/UUID/HERE=volume root=/dev/mapper/volume-root rw
	```

	Replace `/DEV/SDA2/UUID/HERE` with the UUID of your `LVM` partition. You can check it by running `blkid /dev/sda2`.

	Tip: If you are using `vim`, you can write the UUID easier by typing `:read ! blkid /dev/sda2` then hit enter. Then manipulate the output by using visual mode.

### Update boot loader configuration

Update bootloader configuration

```
# vim /boot/loader/loader.conf
```

Delete all of its content, then replaced it by:

```
default arch.conf
timeout 0
console-mode max
editor no
```

#### Microcode (continue here with btrfs)

Processor manufacturers release stability and security updates to the processor microcode. These updates provide bug fixes that can be critical to the stability of your system. Without them, you may experience spurious crashes or unexpected system halts that can be difficult to track down. 

If you didn't install it using pacstrap, install microcode by:

For AMD processors:

```
# pacman -S amd-ucode
```

For Intel processors:

```
# pacman -S intel-ucode
```

If your Arch installation is on a removable drive that needs to have microcode for both manufacturer processors, install both packages. 

Load  microcode. For `systemd-boot`, use the `initrd` option to load the microcode, **before** the initial ramdisk, as follows:

```
# sudoedit /boot/loader/entries/entry.conf
```

```
title   Arch Linux
linux   /vmlinuz-linux
initrd  /CPU_MANUFACTURER-ucode.img
initrd  /initramfs-linux.img
...
```

Replace `CPU_MANUFACTURER` with either `amd` or `intel` depending on your processor.

### Enable systemd services

Enable services for network manager, fstrim for ssd, also scrub timers for btrfs. First scrub timer is for root subvolume. This sets a monthly timer.  This is standard for btrfs scrub. Use nmtui to start internet when you reboot.

```
systemctl enable NetworkManager
systemctl enable fstrim.timer
systemctl enable btrfs-scrub@-.timer
systemctl enable btrfs-scrub@home.timer
systemctl enable btrfs-scrub@snapshots.timer
systemctl enable btrfs-scrub@var.time
```
## Enable internet connection for the next boot

To enable the network daemons on your next reboot, you need to enable `dhcpcd.service` for wired connection and `iwd.service` for a wireless one. Not needed if using NetworkManager.

```
# systemctl enable dhcpcd iwd
```

## Exit chroot and reboot:  

Exit the chroot environment by typing `exit` or pressing <kbd>Ctrl + d</kbd>. You can also unmount all mounted partition after this. 

Finally, `reboot`.

##  Finale

If your installation is a success, then ***yay!!!*** If not, you should start questioning your own existence. Are your parents proud of you? 

## [[POST INSTALLATION]](./POST.md)		[[EXTRAS]](./EXTRAS.md)
