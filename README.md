# Arch Installation Guide

An opinionated guide to a complete Arch Linux installation. There are two paths, one that includes full-disk encryption and one that doesn't.

Sources:
- https://wiki.archlinux.org/title/Installation_guide
- https://wiki.archlinux.org/title/GRUB#Installation_2
- https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS
- https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Encrypted_boot_partition_(GRUB)
- https://github.com/ejmg/an-idiots-guide-to-installing-arch-on-a-lenovo-carbon-x1-gen-6

# 1. Preparation

## Boot the Arch live USB

https://wiki.archlinux.org/title/Installation_guide#Boot_the_live_environment

## Connect to the internet

If ethernet with DHCP is connected, it should come up automatically.

WiFi may be operated via `iwctl`.

```
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect <SSID>
```

## Verify the boot mode

I want to boot in EFI mode. Let's verify.

```bash
ls /sys/firmware/efi/efivars
```

## Update the system clock

The live CD should take care of this once a connection to the internet is established. Verify that time is correct via:

```shell
timedatectl status
```

# 2A. Unencrypted disk

If you want full-disk encryption, go to [2B. Full disk encryption](#2b-full-disk-encryption) instead.

## Partition the disks

Partition with `cfdisk /dev/sda`, use GPT and the following scheme.

| mount | fs     | size    |
|-------| -------|---------|
| /efi  | FAT32  | 512 MiB |
| /boot | ext4   | 512 MiB |
| swap  | swap   |         |
| /     | btrfs  |         |

## Format the partitions

```bash
mkfs.fat -F32 /dev/sda1
mkfs.ext4 /dev/sda2
mkswap /dev/sda3
mkfs.btrfs /dev/sda4
```

## Mount the file systems

```bash
swapon /dev/sda3
mount /dev/sda4 /mnt
mkdir /mnt/{efi,boot}
mount /dev/sda1 /mnt/efi
mount /dev/sda2 /mnt/boot
```

## Install system base and chroot into it

```bash
pacstrap /mnt base base-devel linux linux-firmware
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

## Install a bootloader

```bash
mkinitcpio -P
pacman -S grub efibootmgr os-prober
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

Now go to [3. Final configuration](#3-final-configuration).

# 2B. Full-disk encryption

We will encrypt the entire drive (including `/boot`) using the LVM on LUKS scheme. The only unencrypted partition is `/efi`. Since GRUB supports encrypted boot partition with LUKS1 only, we cannot use LUKS2.

## Partition the disk

Partition with `cfdisk /dev/sda`, use GPT and the following scheme.

| partition | purpose     | size     |
|-----------| ------------|----------|
| /dev/sda1 | /efi        | 512 MiB  |
| /dev/sda2 | LVM on LUKS | the rest |

## Create an encrypted container

```bash
cryptsetup luksFormat --type luks1 /dev/sda2
cryptsetup open /dev/sda2 system
```

## Set LVM up

```bash
pvcreate /dev/mapper/system
vgcreate system_group /dev/mapper/system
lvcreate -L 8G system_group -n swap
lvcreate -l 100%FREE system_group -n root
```

## Create filesystems

```bash
mkfs.fat -F32 /dev/sda1
mkfs.btrfs /dev/mapper/system_group-root`
mkswap /dev/mapper/system_group-swap
```

## Mount the filesystems

```bash
mount /dev/mapper/system_group-root /mnt
mkdir /mnt/efi
mount /dev/sda1 /mnt/efi
swapon /dev/mapper/system_group-swap
```

## Install system base and chroot into it

```bash
pacstrap /mnt base base-devel linux linux-firmware
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

## Install a bootloader

Make sure the `lvm2` package is installed and add the `keyboard`, `keymap`, `encrypt` and `lvm2` hooks to [mkinitcpio.conf](https://wiki.archlinux.org/title/Mkinitcpio.conf "Mkinitcpio.conf"):

```bash
HOOKS=(base udev autodetect keyboard keymap consolefont modconf block encrypt lvm2 filesystems fsck)
```

Regenerate initramfs and install Grub:

```bash
mkinitcpio -p linux
pacman -S grub efibootmgr
```

Edit `/etc/default/grub` like this:  
(Find out the UUID of `/dev/sda2` via `lsblk -f`, this may only work outside of chroot.)

```bash
GRUB_ENABLE_CRYPTODISK=y
GRUB_CMDLINE_LINUX="cryptdevice=UUID=</dev/sda2 UUID>:system"
```

Install grub:

```bash
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB --recheck
grub-mkconfig -o /boot/grub/grub.cfg
```

## Avoid having to enter the passphrase twice

https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Avoiding_having_to_enter_the_passphrase_twice

The way things are set up so far would work, but we would need to enter the LUKS password twice on each boot. Once for grub and a second time for the kernel.

Instead, we may create a keyfile that will be read by the kernel automatically. This is secure, since the keyfile itself lives on the encrypted filesystem and is accessible only after entering the LUKS password.

Create the keyfile:

```shell
dd bs=512 count=4 if=/dev/random of=/root/system.keyfile iflag=fullblock
chmod 000 /root/system.keyfile
cryptsetup -v luksAddKey /dev/sda2 /root/system.keyfile
```

Edit `/etc/mkinitcpio.conf`:

```shell
FILES=(/root/system.keyfile)
```

Recreate the initramfs image and secure the embedded keyfile:

```shell
mkinitcpio -p linux
chmod 600 /boot/initramfs-linux*
```

Edit `/etc/default/grub`:

```shell
GRUB_CMDLINE_LINUX="... cryptkey=rootfs:/root/system.keyfile"
```

Regenerate grub config:

```shell
grub-mkconfig -o /boot/grub/grub.cfg
```

# 3. Final configuration

## Time zone

```shell
ln -sf /usr/share/zoneinfo/Europe/Prague /etc/localtime
hwclock --systohc
```

## Localization

Edit `/etc/locale.gen` and uncomment `en_US.UTF-8 UTF-8` and other needed [locales](https://wiki.archlinux.org/title/Locale "Locale"). Generate the locales by running:

```shell
locale-gen
```

Create the `/etc/locale.conf` file, and set the `LANG` variable accordingly:

```bash
LANG=en_US.UTF-8
```

## Network

Create the `/etc/hostname` file with the desired hostname.

The simplest way to get wired network to work is to install the `dhcpcd` package and enable dhcp for the network interface (here `enp1s0` for example).

```shell
systemctl enable dhcpcd@enp1s0
```

WiFi and more:
https://wiki.archlinux.org/title/Network_configuration

## Root password

```
passwd
```

## Reboot

First exit the chroot and then reboot. If everything went well we should see Grub with the option to boot into Arch. The Arch install still needs a lot of setup, e.g. users, desktop environment, common software.

# 4. Further setup

## Create user

```shell
useradd -m -U -G wheel -s /usr/bin/zsh <user>
```

`-m` - create home
`-U` - create a group with the same name as the user
`-G` - additional groups
`-s` - login shell

## PipeWire

```shell
pacman -S pipewire wireplumber pipewire-pulse pipewire-alsa
```

## Touchpad

https://wiki.archlinux.org/title/Libinput#Via_xinput

Install `xf86-input-libinput` and make sure that `xf86-input-synaptics` is **not** installed.

## Backlight

https://wiki.archlinux.org/index.php/Backlight#ACPI

I had to add udev rules so that users that are in the video group can change brightness level.
