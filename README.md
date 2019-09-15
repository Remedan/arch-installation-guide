# Arch Installation Guide

I had two objectives during my last Arch install. Use LVM on LUKS for encryption and have working dual boot with Windows. Here are documented the basic steps to achieve that.

Sources:
* https://wiki.archlinux.org/index.php/Installation_guide
* https://wiki.archlinux.org/index.php/GRUB#Installation_2
* https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS
* https://github.com/ejmg/an-idiots-guide-to-installing-arch-on-a-lenovo-carbon-x1-gen-6

## Boot the Arch live USB

https://wiki.archlinux.org/index.php/Installation_guide#Boot_the_live_environment

## Set the keyboard layout

https://wiki.archlinux.org/index.php/Installation_guide#Set_the_keyboard_layout

## Update the system clock

https://wiki.archlinux.org/index.php/Installation_guide#Update_the_system_clock

## Partition the disk

The laptop had Windows preinstalled. I shrunk the Windows partition and created 2 new ones. One for unencrypted /boot and one for the Linux system featuring LVM on LUKS.

/dev/sda1 is a preexisting EFI partition  
/dev/sda2 is unencrypted /boot  
/dev/sda3 is LVM on LUKS

## Create an encrypted container

```
cryptsetup --type luks2 luksFormat /dev/sda3
cryptsetup --type luks2 open /dev/sda3 system
```

## Set LVM up

```
pvcreate /dev/mapper/system
vgcreate system_group /dev/mapper/system
lvcreate -L 8G system_group -n swap
lvcreate -l 100%FREE system_group -n root
```

## Create filesystems

```
mkfs.ext4 /dev/sda2
mkfs.ext4 /dev/mapper/system_group-root`
mkswap /dev/mapper/system_group-swap
```

## Mount the filesystems

```
mount /dev/mapper/system_group-root /mnt
mkdir /mnt/efi
mount /dev/sda1 /mnt/efi
mkdir /mnt/boot
mount /dev/sda2 /mnt/boot
swapon /dev/mapper/system_group-root
```

## Install system base

```
pacstrap /mnt base base-devel
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

## Set Time zone, Localization, and Network

* https://wiki.archlinux.org/index.php/Installation_guide#Time_zone
* https://wiki.archlinux.org/index.php/Installation_guide#Localization
* https://wiki.archlinux.org/index.php/Installation_guide#Network_configuration

## Install a bootloader

Edit `/etc/mkinitcpio.conf` and change the following line:
```
HOOKS=(base udev autodetect keyboard keymap modconf block encrypt lvm2 filesystems fsck)
```

Regenerate initramfs and install Grub:

```
mkinitcpio -p linux
pacman -S grub efibootmgr os-prober
```

Edit `/etc/default/grub` like this:  
(Find out the UUID of /dev/sda3 via `lsblk  -f`.)

```
GRUB_PRELOAD_MODULES="... lvm"
GRUB_CMDLINE_LINUX="cryptdevice=UUID=</dev/sda3 UUID>:system root=/dev/mapper/system_group-root"
```

Install grub:

```
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

This should be enough to have a basic Arch install that we can boot into. Next steps are to setup users, install a desktop environemnt and so on, depending on your preferences.
