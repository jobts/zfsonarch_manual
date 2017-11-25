``` sh
# Another guide zfs on Arch Linux
# inspired https://wiki.archlinux.org/index.php/Installing_Arch_Linux_on_ZFS
# and FreeBSD zfs manuals

# Transition to zfs from a runing system, but  
# if you want make a new installation, you need boot CD with zfs modules. Read this:
# https://ramsdenj.com/2016/06/23/arch-linux-on-zfs-part-1-embed-zfs-in-archiso.html
# Make partitions on new disk with gpt partitioning structure.
# fdisk or parted, other tool. It doesn't matter
# I use grub bootloader with Bios boot partition.
# If you have Efi, you can make larger size Efi partition.
# In my case
# 2 partion BIOS boot and Solaris root



# Attention, was a problem
# I  usually use below zfs strusture in FreeBSD. But in linux it doesn't work. A cause may be  grub.
tank-
    |root    /
    |usr    /usr
    |tmp    /tmp
    |var    /var
    |home


# Initramfs fall down to shell with error:

# mounted successfully but /sbin/init does not exist


# A solution is simple: nested system volumes in root. Use  stucture below. Keep in mind hooks in mkinitcpio.conf - zfs usr
# Maybe need only usr, but I include all system volumes.  tank/root/usr
tank-
    |root-      /
        |usr    /usr
        |tmp    /tmp
        |var    /var
        |home


# Partition. May use parted
# parted --script /dev/sdx mklabel gpt mkpart non-fs 0% 2 mkpart primary 2 100% set 1 bios_grub on set 2 boot on
# Вut I make partition with gdisk or fdisk according scheme in the manual. I can not find Solaris Root type partition in parted. Higher command make EFI boot partition.

Part     Size   Type
----     ----   -------------------------
   1       2M   BIOS boot partition (ef02)
   2     XXXG   Solaris Root (bf00)


# On working system install zfs package. If new installation from CD skip this step.

#yaourt -S zfs-linux


# zfs module load

modprobe zfs


# create pool. Disable grub unsupported features and make some tunes

sudo zpool create -R /mnt -O mountpoint=none -O atime=off -O relatime=on -O compression=lz4 -o ashift=12   \
-o feature@multi_vdev_crash_dump=disabled \
-o feature@large_dnode=disabled           \
-o feature@sha512=disabled                \
-o feature@skein=disabled                 \
-o feature@edonr=disabled \
tank /dev/disk/by-id/ata-WDC_WDXXXXX-XXXXX_XX-XXXXXXXXX-part2

# Create fs. Set mountpoint=/ tank/root, other inhirit mountpoints
zfs create -o mountpoint=/ tank/root
zfs create  tank/root/usr  
zfs create  tank/root/var 
zfs create  tank/root/tmp  
zfs create  tank/root/home
zpool set bootfs=tank/root tank

# May be not need, as in manual
zpool export tank

# Import pool to /mnt altroot
zpool import -d /dev/disk/by-id  -R /mnt tank



mkdir -p /mnt/etc/zfs
cp /etc/zfs/zpool.cache /mnt/etc/zfs/zpool.cache

#or
#if zpool.cache not exist, set zpool cache file after installing package zfs in arch-chroot


# Show mounts
sudo zfs mount

# if did not mount
sudo zfs mount -a


#System install

pacstrap /mnt base base-devel vim mc bash-completion sudo  git wget grub 

# Generate sample fstab 
genfstab -U /mnt >> /mnt/etc/fstab.sample

arch-chroot /mnt bash



# --------------- Example only 
# Setup from repo does not work. GPG keys error.

# vim /etc/pacman.conf
# # add lines

# [archzfs]
# Server = http://archzfs.com/$repo/x86_64
# ---------------
# # add unofficial repo key  check  
# # to solve keyserver problems time, etc go https://wiki.archlinux.org/index.php/Pacman/Package_signing
# gpg --recv-keys  5E1ABF240EE7A126
# pacman-key -r 5E1ABF240EE7A126
# ---------------

# user add
useradd -m -s /bin/bash -G wheel myuser

# Set passwords
passwd myuser
passwd root

visudo
Uncomment
%wheel ALL=(ALL) NOPASSWD: ALL

# yaourt install 
su - myuser
cd /tmp
git clone https://aur.archlinux.org/package-query.git
git clone https://aur.archlinux.org/yaourt.git
cd /tmp/package-query
makepkg -si  --noconfirm
cd /tmp/yaourt
makepkg -si  --noconfirm

# install zfs in chrooted system
yaourt -S zfs-linux --noconfirm 

# Set zfs cache if not exist (higher in this text). 
# zpool set cachefile=/etc/zfs/zpool.cache tank

# Edit

vim /etc/mkinitcpio.conf

# set
#----------------
HOOKS=(base udev autodetect modconf block keyboard zfs filesystems usr shutdown)
COMPRESSION="lz4"
# ---------------

# Make initramfs
mkinitcpio -p linux

# --------------- only for systemd initramfs
# if you want systemd initramfs:
# HOOKS=(base systemd autodetect modconf block keyboard filesystems sd-zfs) 
# yaourt -S mkinitcpio-sd-zfs

#vim /etc/default/grub

# set kernel parameter  
# https://github.com/dasJ/sd-zfs
# may set root=zfs:AUTO  zfs_force=1 zfs_ignorecache=1
# 1. autosearch root 2 force import pool 3  ignore zfs cache
# GRUB_CMDLINE_LINUX_DEFAULT="quiet root=zfs:tank/root"
# ---------------


# grub install

grub-install /dev/sda
# error
# grub-install: error: failed to get canonical path of `/dev/ata-WDC_WDXXXXX-XXXXX_XX-XXXXXXXXX-part2`

#workaround
ZPOOL_VDEV_NAME_PATH=1 grub-install /dev/sda

#Installing for i386-pc platform.
#Installation finished. No error reported.

#but you may use:
#ln -s /dev/sda3 /dev/ata-WDC_WDXXXXX-XXXXX_XX-XXXXXXXXX-part2
# then 
# grub-install /dev/sda


# Everything ok

# /boot/grub/grub.cfg

# New config for grub
vim /etc/grub.d/05_grub
 
# (0) Arch Linux zfs # My case
menuentry "Arch Linux ZFS" {
    # My config worked without search
    # search -u UUID
    # Naming scheme: 
    # linux /zfspart1/zfspart2/@/bootdirectory/kernel
    # initrd /zfspart1/zfspart2/@/bootdirectory/initramfs-linux.img
    linux /root/@/boot/vmlinuz-linux zfs=tank/root rw
    initrd /root/@/boot/initramfs-linux.img
}

# Example
vim /etc/grub.d/40_custum

# (50) Arch Linux zfs  # Another example
menuentry "Arch zfs2" {
   load_video
   set gfxpayload=keep
   insmod gzio
   insmod part_gpt
   insmod zfs
   search -u 111111111111111111111000
   linux /root/@/boot/vmlinuz-linux root=ZFS=tank/root rw zfs=tank/root zfs_force=1 
   initrd /root/@/boot/initramfs-linux.img
}

# (60) Arch Linux # if boot on separate partition with ext4
menuentry "Arch Linux zfs3" {
    linux /boot/vmlinuz-linux zfs=tank/root rw
    initrd /boot/initramfs-linux.img
}
# ---------------

vim /etc/default/grub
# Comment and add
# GRUB_DEFAULT=0
GRUB_DEFAULT='Arch Linux ZFS'


# Grub config

# grub-mkconfig -o /boot/grub/grub.cfg 
# error 

# workaround
ZPOOL_VDEV_NAME_PATH=1 grub-mkconfig -o /boot/grub/grub.cfg


# Generating grub configuration file ...
# Found linux image: /boot/vmlinuz-linux
# Found initrd image(s) in /boot: initramfs-linux.img

# Set time, etc
ln -f -s /usr/share/zoneinfo/Europe/Moscow /etc/localtime

# Set locale
vim /etc/locale.gen

locale-gen
echo "LANG=en_US.UTF-8" >> /etc/locale.conf

# Important!!! Enable zfs services 
systemctl enable zfs.target 
systemctl enable zfs-import-cache
systemctl enable zfs-mount


# Exit from chroot
exit


# umount /mnt/boot (if you have a legacy boot partition)

# Important!!! export pool
zfs umount -a
zpool export tank


reboot
```

