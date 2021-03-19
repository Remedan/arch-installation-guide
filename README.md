# Arch Installation Guide

I noted all the steps during my last Arch installation. They are sufficient for a bootable system.

(There is also a different [guide with LVM on LUKS](lvm-on-luks.md).)

## Verify the boot mode

I want to boot in EFI mode. Let's verify.

```
ls /sys/firmware/efi/efivars
```

## Update the system clock

```
timedatectl set-ntp true
```

## Partition the disks

Partition with `cfdisk /dev/sda`, use GPT and the following scheme.

| mount | fs    | size    |
|-------| ------|---------|
| /efi  | FAT32 | 512 MiB |
| /boot | ext2  | 512 MiB |
| swap  | swap  |         |
| /     | ext4  |         |

## Format the partitions

```
mkfs.fat -F32 /dev/sda1
mkfs.ext2 /dev/sda2
mkswap /dev/sda3
mkfs.ext4 /dev/sda4
```

## Mount the file systems

```
swapon /dev/sda3
mount /dev/sda4 /mnt
mkdir /mnt/{efi,boot}
mount /dev/sda1 /mnt/efi
mount /dev/sda2 /mnt/boot
```

## Install essential packages

```
pacstrap /mnt base linux linux-firmware
```

## Fstab

```
genfstab -U /mnt >> /mnt/etc/fstab
```

## Chroot

```
arch-chroot /mnt
```

## Time zone

Set the time zone to `Europe/Prague`.

```
ln -sf /usr/share/zoneinfo/Europe/Prague /etc/localtime
hwclock --systohc
```

## Localization

Install `vim` and uncomment `en_US.UTF-8 UTF-8` and `cs_CZ.UTF-8 UTF-8` in `/etc/locale.gen`

```
locale-gen
echo 'LANG=cs_CZ.UTF-8' > /etc/locale.conf
```

## Hostname

The hostname here is _arch_.

```
echo 'arch' > /etc/hostname
cat >> /etc/hosts << EOF
127.0.0.1	localhost
::1		localhost
127.0.0.1	arch.localdomain	arch
EOF
```

## Initramfs

```
mkinitcpio -P
```

## Root password

```
passwd
```

## Boot loader

```
pacman -S grub efibootmgr os-prober
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

## DHCP

I need to have a DHCP client available after reboot.

```
pacman -S dhcpcd
```

## Reboot

First exit the chroot environment then reboot (and remove the installation medium).

```
exit
reboot
```
