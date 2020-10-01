---
title: "UEFI Install Guide for Arch Linux"
date: 2020-09-04T00:00:00-00:00
tags: [arch, linux]
---

I had a Dell Precision 5520 mobile workstation that I wanted to remove Windows from and install an open-source operating system to for a daily driver/travel laptop. I'd been using Debian and Ubuntu on top of Xen on the server side for my homelab for some time, but I was more interested in a minimal distribution such as [Gentoo](https://www.gentoo.org) or [Arch Linux](https://www.archlinux.org/). I like the idea of Gentoo -- everything is compiled from source, the kernel is compiled during the installation, and there are no pre-built binaries in sight. You can even do a multi-stage installation where the entire toolchain is built from source and then used to build the new operating system. However, in my testing on both bare metal and in a VM, the process was quite slow, even on good hardware. I don't have the time or patience to spend all day compiling, and my frustrations with Gentoo turned my eye towards Arch instead.

I had a few requirements in mind before I started the project. Namely:

- Encrypt the full system at rest (LVM over LUKS)
- Use GPG-encrypted keyfiles for the LUKS volume(s) instead of password-only (dual-factor protection for the system)
- Boot off of a USB key (kernel + encrypted keyfile will be stored on the USB key and will be required for boot)
- Use UEFI Secure Boot
- Use GNOME 3 on Wayland (and keep systemd)
- Suspend and Hibernate to an encrypted swap

I have three things to point out at the start of the guide:

1. I'll be following the [Arch Linux Install Guide](https://wiki.archlinux.org/index.php/Installation_guide) as closely as possible, making modifications where appropriate.
2. I won't be installing a fully-featured bootloader (like GRUB or Clover). Instead, I'll use `systemd-boot`, to keep the system minimal. If you're dual-booting with Windows 10 or another OS, you may consider using an alternative bootloader.
3. I'll be statically compiling `gpg` 1.4.23 (latest in the 1.4.x branch at the time of writing), as `gpg` 2.x has a requirement for pinentry which doesn't play nicely with early userspace initramfs.

With those things out of the way, let's begin!

## (Optional) Step 0: Backup your existing system
If you're dual-booting a system (for example, loading Arch alongside Windows 10), go ahead and take a full backup of your existing OS just in case anything goes wrong.

## Step 1: Gather prerequisites
I **highly** having a second PC handy. Once you boot the install medium, you can enable SSH and run all of your commands remotely instead of typing everything manually. If you're on Linux, you can just use your favorite terminal emulator. If you're using Windows, grab an SSH client (like PuTTY) and you'll be good to go.

Here's what else you'll need:

* A UEFI-enabled PC. I'm using a Dell Precision 5520 Mobile Workstation, but you can use another machine (or even a virtual machine). If you're on Windows and just want to test out what's going on here, install [VirtualBox](https://www.virtualbox.org/) and create a VM with your preferred settings -- just make sure to enable EFI. Go to the VM Settings > System and check "Enable EFI" under "Extended Features":

> Note: if you're using a VM, create a small (512MB or so) disk to attach to emulate the USB key you'd use on a machine. Here's what mine looks like:

* Two USB keys: one for the Arch Linux install disk (2GB or higher should be fine), and one for the `/efi` partition of the new system (initramfs + keyfile). You should be fine with 512MB or so; I'm using a 32GB drive that I had laying around.

* A working network with internet access. You'll want your helper PC and new machine to be able to communicate (SSH), and both to have internet access. Wireless is fine, too.

* `GnuPG` Installed on your helper PC (you technically don't have to do this, but it's a good idea to verify your downloads). On Linux, just use the `gpg` package. On Windows, install [GPG4Win](https://gnupg.org/download/index.html).

## (Optional) Step 2: Shrink Windows 10 disk on target machine

If you're dual-booting, shrink your existing OS disk to accommodate the new system. Right-click the start button and choose "Disk Management"

Right-click your OS disk and choose Shrink Volume...

Shrink it to whatever size is appropriate for you. I'm installing to bare metal on a NVMe SSD, so I'll be using the full disk. The root file system should take under 10GB when done, and then you'll want to plan for swap (equal to the amount of RAM), and space for any applications and files you'd like to store. 50GB+ should be good for a testing system, and if it's a daily driver, then only you know how much space you'll need for files such as photos, documents, etc.

## Step 3: Create the bootable Arch Linux disk

Using your helper PC, download the current Arch Linux installer disk and "burn" it to a USB stick. Go to the Arch download page: https://www.archlinux.org/download/ and then find your appropriate image. You can use a bittorrent download, or use an HTTP Direct Download if you don't have/don't want a bittorrent client on your computer.
Under "Checksums" on the download page, use the PGP signature and fingerprint to verify the download

On Windows, if you installed GPG4Win, launch Kleopatra and choose "Lookup on Server", and enter the fingerprint on the Arch Linux download page (without the 0x). Verify it matches what's listed on the Arch Linux Master Key page -- check the fingerprint it returns against the fingerprint on the website. If it does, click the "Certify" button.

Once you have downloaded the .sig file and the .iso file from the Downloads page, place them both in the same directory (Downloads, for example), and open up Kleopatra again. Choose "Decrypt/Verify" and navigate to the .iso file you downloaded. Kleopatra will search for the signature file and verify it with the fingerprint we downloaded and certified before:

If the download is complete and the signature verified, you're ready to burn it to a USB key! On Windows, download or install Rufus and insert your USB key. Choose the ISO file and the USB key, then click START. Once finished, insert the USB key into your computer and boot off of it.
If you're using a virtual machine, just choose to attach the ISO to the VM, and start the VM (or reset it if you've already started it).

## Step 4: Enter the Installation Environment

Now that you're booted into the environment, you should have a prompt like this:

```console
To install Arch Linux follow the installation guide:
https://wiki.archlinux.org/index.php/Installation_guide

For Wi-Fi, authenticate to the wireless network using the iwctl utility.
Ethernet and Wi-Fi connections using DHCP should work automatically.

After connecting to the internet, the installation guide can be accessed
via the convenience script Installation_guide.

Last login: Wed Sep  2 18:45:20 2020
root@archiso ~ # 
```

First things first. Make sure you're connected to the network. If you're using Ethernet and you've got a DHCP server on your network, you should already be connected. If you need to connect wireless, run `iwctl` and set up wireless:

```console
# iwctl
[iwd]# device list
[iwd]# station "wlan0" connect "SSID"
Enter passphrase: <Enter passphrase for wireless>
[iwd]# quit
```

Once you're connected, get (and note) your IP address so you can connect over SSH in a moment. Issue `ip a | grep inet` to get output like this:

```console
# ip a | grep inet
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host 
    inet 192.168.42.234/24 brd 192.168.42.255 scope global dynamic enp0s3
```

In this example, my machine is available at `192.168.42.234`. Next, set a root password (this is just a temporary password for the live installer, not for the installed system), and start a SSH server so we can access this machine from our helper machine.

```console
# passwd
# systemctl start sshd.service
```

With that, connect from your helper PC to the arch installation using `ssh` (PuTTY or similar on Windows, and `ssh` on Linux/macOS). At the terminal, verify internet connectivity and update the system time:

```console
# ping -c3 archlinux.org
PING archlinux.org(apollo.archlinux.org (2a01:4f8:172:1d86::1)) 56 data bytes
64 bytes from apollo.archlinux.org (2a01:4f8:172:1d86::1): icmp_seq=1 ttl=48 time=116 ms
64 bytes from apollo.archlinux.org (2a01:4f8:172:1d86::1): icmp_seq=2 ttl=48 time=186 ms
64 bytes from apollo.archlinux.org (2a01:4f8:172:1d86::1): icmp_seq=3 ttl=48 time=209 ms

--- archlinux.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 115.539/169.941/208.586/39.586 ms

# timedatectl set-ntp true
```

Now we can actually start the installation...Let's partition!

## Step 5: Partition the Disks

I'll be creating the `/efi` partition on the USB disk, and creating a LUKS container on our main disk to hold the `swap`, `root`, and `/home` partitions. First, identify your disks:

```console
# lsblk
sda           8:0    0 223.6G  0 disk 
sdb           8:16   1   1.9G  0 disk 
sdc           8:32   1  14.5G  0 disk /run/archiso/bootmnt
├─sdc1        8:33   1   671M  0 part 
└─sdc2        8:34   1    64M  0 part 
nvme0n1     259:0    0 238.5G  0 disk 
```

In this example, I have `sda`, a 240GB SATA SSD (which for this example, I won't be setting up); `sdb`, the smaller USB key that will hold the `/efi` partition and the encryption keys; and `nvme0n1`, the main NVMe SSD, which will be fully encrypted and store the system.

Using `parted`, create the partition for the `/efi` partition:

```console
# parted -a optimal /dev/sdb -s mklabel gpt
# parted -a optimal /dev/sdb -s mkpart primary 1mib 513mib
# parted -a optimal /dev/sdb -s set 1 esp on
```

Create a mountpoint at `/tmp/usb`, then create a filesystem (FAT32) and mount the partition in the installation environment:

```console
# mkdir /tmp/usb
# mkfs.vfat -F32 -n "EFI" /dev/sdb1 
# mount -v -t vfat /dev/sdb1 /tmp/usb
```

### Creating the keyfile for LUKS

Now, it's time to create an encrypted keyfile for protecting the LUKS data. We'll create a keyfile using `/dev/urandom`, pass it into `gpg`, and store the encrypted file directly to the USB. In this way, the plaintext keyfile is never stored outside of memory. Also, we need to set `GPG_TTY` because `pinentry` won't work if we don't.

The keyfile will be one byte less than 8MiB (due to a bug in `cryptsetup`), and we'll use AES256 to encrypt the keyfile.

```console
# export GPG_TTY=$(tty)
# dd if=/dev/urandom bs=8388607 count=1 | gpg --symmetric --cipher-algo AES256 --output /tmp/usb/luks-key.gpg
```

> DO NOT LOSE THIS FILE. IF YOU DO, YOUR DATA IS NOT RECOVERABLE

You'll be asked for a passphrase. I'd suggest coming up with a short sentence to remember and use that for your passphrase. It's unlikely you'll forget it, and it's typically more secure than the stereotypical "uppercase, lowercase, number and a symbol" rule of passwords. Use punctuation and capitalization rules to maximize entropy.

### Create the LUKS container on the main drive

Remember how I said to back up your files on your main drive earlier? Make sure that's done now, as this next step can cause data loss if you mess it up.

I'm using the entire disk, but if you're not (just installing alongside Windows, for example), check your free sectors on the disk you got from `lsblk` above:

```console
# parted -a optimal /dev/sdc
(parted) unit s
(parted) print free

Number  Start     End       Size      File system  Name     Flags
        34s       2047s     2014s     Free Space
 1      2048s     1050624s  1048577s  fat32        primary  boot, esp
        1050625s  3911646s  2861022s  Free Space
```

This is the output of `print free` on the USB disk I just created; you would use it on a different disk in your system. Note the start and end sectors for the free space at the end of the drive; your command will look like this:

```console
(parted) mkpart primary 1050625s 3911646s
(parted) q
```

If you're using the whole disk, the command is:

```console
(parted) mkpart primary 0% 100%
(parted) q
```

Run `lsblk` again to check out your new disk layout:

```console
# lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sdb           8:16   1   1.9G  0 disk 
└─sdb1        8:17   1   512M  0 part /tmp/usb
nvme0n1     259:0    0 238.5G  0 disk 
└─nvme0n1p1 259:2    0 238.5G  0 part 
```

Time to make that new partition a LUKS container, using our encrypted keyfile as the key. You don't have to use the `--cipher`, `--key-size`, or `--hash` arguments, but I'm choosing these based on very secure yet performant options. If you leave them out, you can check `cryptsetup --help` to see what you'll get:

```console
# cryptsetup --help
Default compiled-in device cipher parameters:
	loop-AES: aes, Key 256 bits
	plain: aes-cbc-essiv:sha256, Key: 256 bits, Password hashing: ripemd160
	LUKS: aes-xts-plain64, Key: 256 bits, LUKS header hashing: sha256, RNG: /dev/urandom
	LUKS: Default keysize with XTS mode (two internal keys) will be doubled.
```

Serpent is more secure but slightly slower than the Rijndael cipher, but since I'm on a NVMe SSD, I don't care. You might want to use `aes-xts-plain64` instead. If so, just don't enter the `--cipher` or `--key-size parameters`. I do strongly suggest using whirlpool for your hashing algorithm; it's considerably harder to brute force than SHA256.

Double-check your disk paths (I'm using `/dev/nvme0n1p1` in my example, you may be using something like `/dev/sda7`) and enter this command:

```console
# gpg --decrypt /tmp/usb/luks-key.gpg | cryptsetup --cipher serpent-xts-plain64 --key-size 512 --hash whirlpool --key-file - luksFormat /dev/nvme0n1p1 
```

Verify your LUKS container:

```console
# cryptsetup luksDump /dev/nvme0n1p1
```

And, to be safe, let's backup the LUKS header to our USB disk. This will help protect from damage to the header in the future.

```console
# cryptsetup luksHeaderBackup /dev/nvme0n1p1 --header-backup-file /tmp/usb/luks-header.img
```

### Creating the LVM Structure

We're still partitioning! Now it's time to create the LVM groups for your system. This isn't a primer on LVM, in fact, I'm hoping you're familiar with LVM already. We're going to create a single physical volume (PV), with a single volume group (VG) inside it. Then, we'll create three logical volumes (LV) inside that to hold our swap, root, and home partitions. You can adjust as your needs dictate.

Check your RAM size (so we create the swap space appropriately):

```console
# grep MemTotal /proc/meminfo
MemTotal:       32602072 kB
```

I've got 32GB of RAM in this system, so I'll create a 36G swap space.

First, open the LUKS container and put it at `/dev/mapper/arch`:

```console
# gpg --decrypt /tmp/usb/luks-key.gpg | cryptsetup --key-file - luksOpen /dev/nvme0n1p1 arch
# ls /dev/mapper
arch  control
```

Create the Physical Volume:

```console
# pvcreate /dev/mapper/arch
```

Create the LVs on the new VG:

```console
# lvcreate --size 36G --name swap vg0
# lvcreate --size 50G --name root vg0
# lvcreate -l +100%FREE --name home vg0
```

Take a look at the LVM info:

```console
# pvs
# vgs
# lvs
```

Activate the Volume Group so the LVs become available in the device mapper:

```console
# vgchange --available y
# ls /dev/mapper
arch  control  vg0-home  vg0-root  vg0-swap
```

Now, create the filesystems. I'm using `XFS`, but you're welcome to use `ext4` or any other system you'd like (such as `btrfs`).

```console
# mkswap -L "swap" /dev/mapper/vg0-swap
# mkfs.xfs -L "root" /dev/mapper/vg0-root
# mkfs.xfs -L "home" /dev/mapper/vg0-home
```

Activate the swap and mount the root partition:

```console
# swapon -v /dev/mapper/vg0-swap
# mount -v /dev/mapper/vg0-root /mnt
```

Create the `/home` and `/efi` directories, unmount the USB from `/tmp/usb` and mount the `/home` and `/efi` partitions:

```console
# umount -v /tmp/usb
# mkdir -v /mnt/{home,efi}
# mount -v /dev/mapper/vg0-home /mnt/home
# mount -v /dev/sdb1 /mnt/efi
```

Quick sanity check to make sure everything's mounted appropriately:

```console
# lsblk
NAME           MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sdb              8:16   1   1.9G  0 disk  
└─sdb1           8:17   1   512M  0 part  /mnt/efi
sdc              8:32   1  14.5G  0 disk  /run/archiso/bootmnt
├─sdc1           8:33   1   671M  0 part  
└─sdc2           8:34   1    64M  0 part  
nvme0n1        259:0    0 238.5G  0 disk  
└─nvme0n1p1    259:2    0 238.5G  0 part  
  └─arch       254:0    0 238.5G  0 crypt 
    ├─vg0-swap 254:1    0    36G  0 lvm   [SWAP]
    ├─vg0-root 254:2    0    50G  0 lvm   /mnt
    └─vg0-home 254:3    0 152.5G  0 lvm   /mnt/home
```

You can see here that my USB key `sdb` is mounted on `/mnt/efi`, and my `vg0` LVM entries are mounted at `/mnt` and `/mnt/home`. Now it's time to install the system!

## Step 6: Install Arch Linux

Installation is actually quite simple, regardless of what you've heard! First, we'll use `pacstrap` to install the system and some related packages, then we'll do some small configurations.

### Installing the base system

Install the base system and firmware as follows:

```console
# pacstrap /mnt base linux linux-firmware
```

> Note: If you're on a virtual machine, you can ignore the `linux-firmware` package, as you won't need it. Also, add in `base-devel` if you'd like a toolchain for development and compilation later.

You'll also want some packages for your filesystem, LVM, man pages, ZShell, and text editing. You can use [this page](https://wiki.archlinux.org/index.php/File_systems) to determine your file system packages.

```console
# pacstrap /mnt xfsprogs dosfstools e2fsprogs lvm2 vim nano man-db man-pages texinfo zsh
```

To plan for our GNOME 3 install later, go ahead and install NetworkManager as well:

```console
# pacstrap /mnt networkmanager
```

### Configuring the system

Generate an fstab file and edit anything you may need to edit. If you're using an SSD, consider changing relatime to noatime on any filesystems, and remove the /boot mountpoint from the fstab file, as you won't need that file booted once the system is booted.

```console
# genfstab -U /mnt >> /mnt/etc/fstab
```

Change root into the new Arch system:

```console
# arch-chroot /mnt
```

> Note: From this point, anything you do is done inside the new Arch installation.

Enable NetworkManager:

```console
# systemctl enable NetworkManager.service
```

Set your time zone and sync the clock:
```console
# ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
# hwclock --systohc
```

> Note: You can issue `ls /usr/share/zoneinfo` to find the regions available, and then `ls /usr/share/zoneinfo/<Region>` to find the Cities available.

Edit `/etc/locale.gen` and uncomment your needed locales (such as `en_US.UTF-8 UTF-8` and `en_US ISO-8859-1`), then generate locales:

```console
# vim /etc/locale.gen
# locale-gen
```

Set your hostname and edit hosts file:

```console
# echo arch-laptop > /etc/hostname
# vim /etc/hosts
127.0.0.1	localhost
::1	        localhost
127.0.1.1	arch-laptop.localdomain	arch-laptop
```

### Statically compiling GNUPG 1.4.x

Since we're using GPG-encrypted keyfiles, we need to include a statically-compiled `gpg` in the 1.4.x branch, as the version of `gpg` that ships with Arch Linux is now in the 2.x branch. As noted above, that causes issues in early userspace. Thankfully, this part is pretty easy.

First, install the `base-devel` group:

```console
# pacman -S base-devel
```

Change to `/tmp`, check GnuPG's site for the latest code in the 1.4.x branch, and download it, along with it's signature:

```console
# cd /tmp
# curl -OL https://gnupg.org/ftp/gcrypt/gnupg/gnupg-1.4.23.tar.bz2
# curl -OL https://gnupg.org/ftp/gcrypt/gnupg/gnupg-1.4.23.tar.bz2.sig
```

Again, it's wise to verify the integrity of the download by using the `.sig` file provided at GnuPG's site. Head to the [signing keys page](https://gnupg.org/signature_key.html) and get the fingerprint for the current signing keys. Import to your GPG keyring:

```console
# gpg --keyserver pgp.mit.edu --recv-key 4F25E3B6
```

And verify the download:

```console
# gpg --verify gnupg-1.4.23.tar.bz2{.sig,}
gpg: Signature made Mon 11 Jun 2018 05:00:34 AM EDT
gpg:                using RSA key D8692123C4065DEA5E0F3AB5249B39D24F25E3B6
gpg: Good signature from "Werner Koch (dist sig)" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: D869 2123 C406 5DEA 5E0F  3AB5 249B 39D2 4F25 E3B6
```

It's OK (*in this instance*) that the key isn't certified; we are running under root in our new system, and we haven't created a trust database yet. The signature is good; that's what we're looking for.

Unpack the archive and set some compilation flags:

```console
# tar xjf gnupg-1.4.23.tar.bz2
# cd gnupg-1.4.23
# CC=gcc LDFLAGS=-static CFLAGS="-g0 -fcommon" ./configure
# make && make install
```

> Note: you need to set the `-fcommon` flag here, or the code won't compile. You can check out the note at [the GCC documentation](https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html) to read what `-fcommon` does. In short, uninitialized global variables are places in a common block; the default `-fno-common` places them in the BSS section of the object file, which doesn't allow merging of the same variable if it's defined more than once. GCC 10 will throw a multiple definition error if you don't include this flag; you can either install the `gcc9` package (which sets `-fcommon` by default) or include the `-fcommon` flag.

Setting `-static` for `LDFLAGS` instructs the linker to statically link everything, so we have no dynamic dependencies for the compiled binary; this will be critical to ensure the early userspace `gpg` doesn't require any external dependencies. Setting `-g0` instructs GCC to not include debugging symbols.

If you have a slow machine/VM, this may (but probably won't) take a little while. When it's done, your new gpg will be at `/usr/local/bin/gpg`. You can check that it's statically linked, and that the version is correct:

```console
# file /usr/local/bin/gpg
/usr/local/bin/gpg: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, for GNU/Linux 3.2.0, not stripped
# /usr/local/bin/gpg --version | head -n1
gpg (GnuPG) 1.4.23
```

Perfect! Now it's time to insert this module into the initramfs with `mkinitcpio`. Edit `/etc/mkinitcpio.conf` and make a few changes:

* Add `vfat` to the `MODULES=(...)` section to ensure the USB key will be mountable
* Add `/usr/local/bin/gpg` to `BINARIES=(...)` to place our new statically-linked `gpg` in the image
* Add `gpgcrypt lvm2` to `HOOKS` after `autodetect` and before `filesystems`.

```console
vim /etc/mkinitcpio.conf
```

Next, we need to create the `gpgcrypt` hook we just told `mkinitcpio` to use. You can copy the current `encrypt` hook and modify it as needed, or use the one I've included here. Copy this content to `/lib/initcpio/hooks/gpgcrypt`:

{{< expandable label="File /lib/initcpio/hooks/gpgcrypt" level="1" >}}
```bash
#!/usr/bin/ash

run_hook() {
    /sbin/modprobe -a -q dm-crypt >/dev/null 2>&1
    if [ -e "/sys/class/misc/device-mapper" ]; then
        if [ ! -e "/dev/mapper/control" ]; then
            /bin/mknod "/dev/mapper/control" c $(cat /sys/class/misc/device-mapper/dev | sed 's|:| |')
        fi
        [ "${quiet}" = "y" ] && CSQUIET=">/dev/null"

        # Get keyfile if specified
        ckeyfile="/crypto_keyfile"
        usegpg="n"
        if [ "x${cryptkey}" != "x" ]; then
            ckdev="$(echo "${cryptkey}" | cut -d: -f1)"
            ckarg1="$(echo "${cryptkey}" | cut -d: -f2)"
            ckarg2="$(echo "${cryptkey}" | cut -d: -f3)"
            if poll_device "${ckdev}" ${rootdelay}; then
                case ${ckarg1} in
                    *[!0-9]*)
                        # Use a file on the device
                        # ckarg1 is not numeric: ckarg1=filesystem, ckarg2=path
                        if [ "${ckarg2#*.}" = "gpg" ]; then
                            ckeyfile="${ckeyfile}.gpg"
                            usegpg="y"
                        fi
                        mkdir /ckey
                        mount -r -t ${ckarg1} ${ckdev} /ckey
                        dd if=/ckey/${ckarg2} of=${ckeyfile} >/dev/null 2>&1
                        umount /ckey
                        ;;
                    *)
                        # Read raw data from the block device
                        # ckarg1 is numeric: ckarg1=offset, ckarg2=length
                        dd if=${ckdev} of=${ckeyfile} bs=1 skip=${ckarg1} count=${ckarg2} >/dev/null 2>&1
                        ;;
                esac
            fi
            [ ! -f ${ckeyfile} ] && echo "Keyfile could not be opened. Reverting to passphrase."
        fi
        if [ -n "${cryptdevice}" ]; then
            DEPRECATED_CRYPT=0
            cryptdev="$(echo "${cryptdevice}" | cut -d: -f1)"
            cryptname="$(echo "${cryptdevice}" | cut -d: -f2)"
        else
            DEPRECATED_CRYPT=1
            cryptdev="${root}"
            cryptname="root"
        fi

        warn_deprecated() {
            echo "The syntax 'root=${root}' where '${root}' is an encrypted volume is deprecated"
            echo "Use 'cryptdevice=${root}:root root=/dev/mapper/root' instead."
        }

        if  poll_device "${cryptdev}" ${rootdelay}; then
            if /sbin/cryptsetup isLuks ${cryptdev} >/dev/null 2>&1; then
                [ ${DEPRECATED_CRYPT} -eq 1 ] && warn_deprecated
                dopassphrase=1
                # If keyfile exists, try to use that
                if [ -f ${ckeyfile} ]; then
                    if [ "${usegpg}" = "y" ]; then
                        # gpg tty fixup
                        if [ -e /dev/tty ]; then mv /dev/tty /dev/tty.backup; fi
                        cp -a /dev/console /dev/tty
                        while [ ! -e /dev/mapper/${cryptname} ];
                        do
                            sleep 2
                            /usr/local/bin/gpg -d "${ckeyfile}" 2>/dev/null | cryptsetup --key-file=- luksOpen ${cryptdev} ${cryptname} ${CSQUIET}
                            dopassphrase=0
                        done
                        rm /dev/tty
                        if [ -e /dev/tty.backup ]; then mv /dev/tty.backup /dev/tty; fi
                    else
                        if eval /sbin/cryptsetup --key-file ${ckeyfile} luksOpen ${cryptdev} ${cryptname} ${CSQUIET}; then
                            dopassphrase=0
                        else
                            echo "Invalid keyfile. Reverting to passphrase."
                        fi
                    fi
                fi
                # Ask for a passphrase
                if [ ${dopassphrase} -gt 0 ]; then
                    echo ""
                    echo "A password is required to access the ${cryptname} volume:"

                    #loop until we get a real password
                    while ! eval /sbin/cryptsetup luksOpen ${cryptdev} ${cryptname} ${CSQUIET}; do
                        sleep 2;
                    done
                fi
                if [ -e "/dev/mapper/${cryptname}" ]; then
                    if [ ${DEPRECATED_CRYPT} -eq 1 ]; then
                        export root="/dev/mapper/root"
                    fi
                else
                    err "Password succeeded, but ${cryptname} creation failed, aborting..."
                    exit 1
                fi
            elif [ -n "${crypto}" ]; then
                [ ${DEPRECATED_CRYPT} -eq 1 ] && warn_deprecated
                msg "Non-LUKS encrypted device found..."
                if [ $# -ne 5 ]; then
                    err "Verify parameter format: crypto=hash:cipher:keysize:offset:skip"
                    err "Non-LUKS decryption not attempted..."
                    return 1
                fi
                exe="/sbin/cryptsetup create ${cryptname} ${cryptdev}"
                tmp=$(echo "${crypto}" | cut -d: -f1)
                [ -n "${tmp}" ] && exe="${exe} --hash \"${tmp}\""
                tmp=$(echo "${crypto}" | cut -d: -f2)
                [ -n "${tmp}" ] && exe="${exe} --cipher \"${tmp}\""
                tmp=$(echo "${crypto}" | cut -d: -f3)
                [ -n "${tmp}" ] && exe="${exe} --key-size \"${tmp}\""
                tmp=$(echo "${crypto}" | cut -d: -f4)
                [ -n "${tmp}" ] && exe="${exe} --offset \"${tmp}\""
                tmp=$(echo "${crypto}" | cut -d: -f5)
                [ -n "${tmp}" ] && exe="${exe} --skip \"${tmp}\""
                if [ -f ${ckeyfile} ]; then
                    exe="${exe} --key-file ${ckeyfile}"
                else
                    exe="${exe} --verify-passphrase"
                    echo ""
                    echo "A password is required to access the ${cryptname} volume:"
                fi
                eval "${exe} ${CSQUIET}"

                if [ $? -ne 0 ]; then
                    err "Non-LUKS device decryption failed. verify format: "
                    err "      crypto=hash:cipher:keysize:offset:skip"
                    exit 1
                fi
                if [ -e "/dev/mapper/${cryptname}" ]; then
                    if [ ${DEPRECATED_CRYPT} -eq 1 ]; then
                        export root="/dev/mapper/root"
                    fi
                else
                    err "Password succeeded, but ${cryptname} creation failed, aborting..."
                    exit 1
                fi
            else
                err "Failed to open encryption mapping: The device ${cryptdev} is not a LUKS volume and the crypto= paramater was not specified."
            fi
        fi
        rm -f ${ckeyfile}
    fi
}
# vim: set ft=sh ts=4 sw=4 et:
```
{{< /expandable >}}

{{< expandable label="File /lib/initcpio/install/gpgcrypt" level="1" >}}
```bash
#!/bin/bash

build() {
    local mod

    add_module "dm-crypt"
    add_module "dm-integrity"
    if [[ $CRYPTO_MODULES ]]; then
        for mod in $CRYPTO_MODULES; do
            add_module "$mod"
        done
    else
        add_all_modules "/crypto/"
    fi

    add_dir "/dev/mapper"
    add_binary "cryptsetup"
    add_binary "dmsetup"
    add_binary "/usr/local/bin/gpg"
    add_file "/usr/lib/udev/rules.d/10-dm.rules"
    add_file "/usr/lib/udev/rules.d/13-dm-disk.rules"
    add_file "/usr/lib/udev/rules.d/95-dm-notify.rules"
    add_file "/usr/lib/initcpio/udev/11-dm-initramfs.rules" "/usr/lib/udev/rules.d/11-dm-initramfs.rules"

    # cryptsetup calls pthread_create(), which dlopen()s libgcc_s.so.1
    add_binary "/usr/lib/libgcc_s.so.1"

    add_runscript
}

help() {
    cat <<HELPEOF
This hook allows for an encrypted root device. Users should specify the device
to be unlocked using 'cryptdevice=device:dmname' on the kernel command line,
where 'device' is the path to the raw device, and 'dmname' is the name given to
the device after unlocking, and will be available as /dev/mapper/dmname.

For unlocking via keyfile, 'cryptkey=device:fstype:path' should be specified on
the kernel cmdline, where 'device' represents the raw block device where the key
exists, 'fstype' is the filesystem type of 'device' (or auto), and 'path' is
the absolute path of the keyfile within the device.

Without specifying a keyfile, you will be prompted for the password at runtime.
This means you must have a keyboard available to input it, and you may need
the keymap hook as well to ensure that the keyboard is using the layout you
expect.
HELPEOF
}

# vim: set ft=sh ts=4 sw=4 et:
```
{{< /expandable >}}

The only changes of this file compared to the original `encrypt` hook is that we're injecting the `/usr/local/bin/gpg` binary in the install (line 19), and using it to unlock the LUKS container in the execution of the hook (line 69)

Finally, regenerate your initramfs:

```console
# mkinitcpio -P
```

If you're on a physical machine, install microcode for your CPU. Adjust to amd or intel based on your hardware:

```console
# pacman -S intel-ucode
```

Set the root password (note: this is the root password of the installed system)

```console
# passwd
```

Create a new user and set the password:

```console
# useradd -m -G wheel -s /bin/zsh tom
# passwd tom
```

> Note: you can set another shell like /bin/bash instead of /bin/zsh

### Setup systemd-boot

Setting the system to boot is quite simple. Make sure your USB key is mounted on `/efi` and that the `boot` and `esp` flags are set:

```console
# df -hT
Filesystem           Type      Size  Used Avail Use% Mounted on
/dev/mapper/vg0-root xfs        50G  2.6G   48G   6% /
/dev/mapper/vg0-home xfs       153G  1.1G  152G   1% /home
/dev/sdb1            vfat      511M   98M  414M  20% /efi

# parted /dev/sdb -s print
Number  Start   End    Size   File system  Name     Flags
 1      1049kB  538MB  537MB  fat32        primary  boot, esp
```

Use `bootctl` to install the EFI executable and make a new entry the default:

```console
# bootctl install --esp-path=/efi
# echo "default arch" >> /efi/loader/loader.conf
```

Find the ID for your /efi disk and the UUID for the LUKS container:

```console
# ls -l /dev/disk/by-id | grep sdb1
lrwxrwxrwx 1 root root 10 Sep  3 11:19 usb-SMI_USB_DISK_AA000000000000000810-0:0-part1 -> ../../sdb1

# blkid | grep LUKS
/dev/nvme0n1p1: UUID="b42f37a1-d721-4770-b0cd-96ec67635897" TYPE="crypto_LUKS" PARTLABEL="primary" PARTUUID="6a409d43-3af5-4734-a7f7-2af247debba1"
```

Create a new entry .conf file for Arch Linux at `/efi/loader/entries/arch.conf`:

```conf
title Arch Linux
linux /vmlinuz-linux
initrd /intel-code.img
initrd /initramfs-linux.img
options root=/dev/mapper/vg0-root resume=/dev/mapper/vg0-swap cryptdevice=/dev/disk/by-uuid/b42f37a1-d721-4770-b0cd-96ec67635897:arch cryptkey=/dev/disk/by-id/usb-SMI_USB_DISK_AA000000000000000810-0\:0-part1:vfat:/luks-key.gpg rw intel_pstate=no_hwp
```

Change `intel-ucode` to `amd-ucode` if you're using AMD microcode. The other important thing here is the options line. What we're setting is:
* `root` - this is the root device. For our purpose, it's the root LV in the `vg0` volume group we created earlier
* `resume` - this is the device to use to write RAM to when hibernating. Again, we're writing to the swap LV so that the contents are encrypted.
* `cryptdevice` - this is the actual block device and LUKS device name to map to. Using the output from `lsblk` above, the path is `/dev/disk/by-uuid/<UUID>`, and the mapper name is the one we created earlier - `arch`.
* `cryptkey` - similar to `cryptdevice`, this is the disk device, filesystem, and keyfile location relative to the root of the device. The `gpgcrypt` hook we installed in the initramfs earlier will mount the device we specify here with the filesystem we specify here, and use the key we specify here to decrypt the LUKS container from `cryptdevice`.

> A very important note: If your USB key has a `:` in the device path, you need to escape the colon. I've done this in the above example for convenience. See [this page](https://wiki.archlinux.org/index.php/Dm-crypt/System_configuration#Boot_loader) on the Arch Wiki.

## Miscellaneous Setup and reboot into new machine

Now, you should install the `sudo` package and enable the `wheel` group to run commands. Also install an SSH server and enable it at startup:

```console
# pacman -S openssh sudo
# systemctl enable sshd.service
# visudo
(find this line for wheel and uncomment)
%wheel        ALL=(ALL)       ALL
```

Let's go ahead and install GNOME 3 before we reboot. The gnome package includes the display manager `gdm` (which we'll enable so it starts on next boot), the display server `wayland` (X applications will be started on XWayland).

```console
# pacman -S gnome
# systemctl enable gdm.service
```

At this point, you should have a bootable system. To try it out, exit the arch system, turn off the swap, unmount the file systems, and reboot:

```console
[root@archiso /] # exit
root@archiso ~ # swapoff /dev/mapper/vg0-swap
root@archiso ~ # umount -R /mnt
root@archiso ~ # reboot
```

Make sure to remove the USB key with the Arch Installer on it.