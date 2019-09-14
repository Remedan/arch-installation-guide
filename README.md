# Arch Installation Guide

* https://wiki.archlinux.org/index.php/Installation_guide
* https://wiki.archlinux.org/index.php/GRUB#Installation_2
* https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS
* https://github.com/ejmg/an-idiots-guide-to-installing-arch-on-a-lenovo-carbon-x1-gen-6

## Set the keyboard layout

https://wiki.archlinux.org/index.php/Installation_guide#Set_the_keyboard_layout

## Update the system clock

https://wiki.archlinux.org/index.php/Installation_guide#Update_the_system_clock

## Partition the disk

/dev/sda1 is efi  
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
mkfs.fat -F32 /dev/sda1
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
pacstrap /mnt base
genfstab -p /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

## Set Time zone, Localization, and Network

* https://wiki.archlinux.org/index.php/Installation_guide#Time_zone
* https://wiki.archlinux.org/index.php/Installation_guide#Localization
* https://wiki.archlinux.org/index.php/Installation_guide#Network_configuration

## Install a bootloader

/etc/mkinitcpio.conf  
`HOOKS=(base udev autodetect keyboard keymap modconf block encrypt lvm2 filesystems fsck)`

`mkinitcpio -p linux`

`pacman -S grub efibootmgr`

/etc/default/grub  
`GRUB_PRELOAD_MODULES="... lvm"`  
`GRUB_CMDLINE_LINUX="cryptdevice=UUID=</dev/sda3 UUID>:system root=/dev/mapper/system_group-root"`

```
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```
