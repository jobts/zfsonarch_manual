``` sh
# Еще один гид по zfs на Arch Linux
# навеян https://wiki.archlinux.org/index.php/Installing_Arch_Linux_on_ZFS
# и FreeBSD zfs мануалами

# Переход на zfs c работающей системы
# Соpдаем на новом диске разделы (с помощью fdisk неважно)
# Разметка gpt 
# Если ставить с нуля с CD диска то его надо сделать, вот так. 
# https://ramsdenj.com/2016/06/23/arch-linux-on-zfs-part-1-embed-zfs-in-archiso.html
# Работает проверено.

# загрузка bios grub 
# У кого Efi, можно вместо Bios boot сделать Efi раздел, побольше конечно
# В моем случае 2 раздела BIOS boot и Solaris root


#ВНИМАНИЕ
# Есть особенность про которую никто не рассказывает. Возможно такое поведение  grub,
# по-крайней мере на Freebsd такого точно не было.
# Я создаю файловые системы обычно так  в пуле tank


tank-
    |root    /
    |usr    /usr
    |tmp    /tmp
    |var    /var
    |home

# Особенность такова, что такая схема почему-то не работает, вываливается в initramfs shell с ошибкой:

# mounted successfully but /sbin/init does not exist

# Решение простое
# системные разделы создаем вложенные,  тогда все ок, не забываем хуки mkinitcpio.conf - zfs usr
# Работает c grub . Остальные разделы неважно где, возможно нужет только /usr вложенный.


tank-
    |root-      /
        |usr    /usr
        |tmp    /tmp
        |var    /var
        |home

# Можно использовать parted
# parted --script /dev/sdx mklabel gpt mkpart non-fs 0% 2 mkpart primary 2 100% set 1 bios_grub on set 2 boot on
# Но я размечаю gdisk или fdisk, схема по мануалу, т.к. не нашел Solaris Root тип раздела, команда выше bp мануала делает его EFI boot
# У gdisk все проще bf00 
Part     Size   Type
----     ----   -------------------------
   1       2M   BIOS boot partition (ef02)
   2     XXXG   Solaris Root (bf00)




# На рабочей системе ставлю из AUR. Кто хочет может из репозитория. 
# Если новая установка со сделаного CD, то не надо, потому коммент.
# Остальное на английском, итак понятно.


#yaourt -S zfs-linux


# zfs module load

modprobe zfs


# Создаем пул. Отключаем  неподдерживаемые опции grub и настраиваем производительность.

Безопасный вариант:

sudo zpool create -R /mnt -O mountpoint=none -O atime=off -O relatime=on -O compression=lz4 -o ashift=12   \
-o feature@multi_vdev_crash_dump=disabled \
-o feature@large_dnode=disabled           \
-o feature@sha512=disabled                \
-o feature@skein=disabled                 \
-o feature@edonr=disabled \
tank /dev/disk/by-id/ata-WDC_WDXXXXX-XXXXX_XX-XXXXXXXXX-part2

Мой вариант, который работает в системе, без отключенных фич
sudo zpool create -R /mnt -O mountpoint=none -O atime=off -O relatime=on -O compression=lz4 -o ashift=12   \
tank /dev/disk/by-id/ata-WDC_WDXXXXX-XXXXX_XX-XXXXXXXXX-part2 


# Создаем подтома, точка монтирования для root, остальные наследуется,
# но можно указать для несистемных другую 
zfs create -o mountpoint=/ tank/root
zfs create  tank/root/usr  
zfs create  tank/root/var 
zfs create  tank/root/tmp  
zfs create  tank/root/home
zpool set bootfs=tank/root tank


# Экспортируем пул, можно наверно и без этого, делаю как советует мануал
zpool export tank

# Импортируем пул /mnt altroot
zpool import -d /dev/disk/by-id  -R /mnt tank


# Копируем zpool кэш 
mkdir -p /mnt/etc/zfs
cp /etc/zfs/zpool.cache /mnt/etc/zfs/zpool.cache

# Или
#if zpool.cache not exist, set zpool cache file after install zfs package in arch-chroot


# Показываем точки монтирования
zfs mount

# Монтируем, если нет
zfs mount -a


#Установка системы

pacstrap /mnt base base-devel vim mc bash-completion sudo  git wget grub 

# Генерирую пример файла fstab. Он не используется в моем случае.
genfstab -U /mnt >> /mnt/etc/fstab.sample

arch-chroot /mnt bash

# --------------- Только для примера
# Setup from repo does not work. GPG keys error. You need set Trustall to the repo. 

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

# Создаем пользователя
useradd -m -s /bin/bash -G wheel myuser

# Пароли
passwd myuser
passwd root


# Sudo 

visudo

# ---------------
Uncomment
%wheel ALL=(ALL) NOPASSWD: ALL
# ---------------

# Установка yaourt 
su - myuser
cd /tmp
git clone https://aur.archlinux.org/package-query.git
git clone https://aur.archlinux.org/yaourt.git
cd /tmp/package-query
makepkg -si  --noconfirm
cd /tmp/yaourt
makepkg -si  --noconfirm

# Установка zfs в chroot
yaourt -S zfs-linux --noconfirm 

# Устанавливаем zfs кэш (если файла не было, см. выше). 
# zpool set cachefile=/etc/zfs/zpool.cache tank


# Создаем хуки для initramfs

vim /etc/mkinitcpio.conf

# Меняем
#----------------
HOOKS=(base udev autodetect modconf block keyboard zfs filesystems usr shutdown)
COMPRESSION="lz4"
# ---------------

# Генерируем образ initramfs
mkinitcpio -p linux

# --------------- Пример только для systemd initramfs
# Можнно использовать systemd initramfs вместо обычного, хуки другие:
# HOOKS=(base systemd autodetect modconf block keyboard filesystems sd-zfs) 
# Но надо ставить пакет
# yaourt -S mkinitcpio-sd-zfs

#vim /etc/default/grub

# Параметры для ядра  
# https://github.com/dasJ/sd-zfs
# may set root=zfs:AUTO  zfs_force=1 zfs_ignorecache=1
# 1. автопоиск root 2 принудительный импорт pool 3  игнорировать zfs cache
# GRUB_CMDLINE_LINUX_DEFAULT="quiet root=zfs:tank/root"
# ---------------


# ставим grub 

grub-install /dev/sda
# ошибка
# grub-install: error: failed to get canonical path of `/dev/ata-WDC_WDXXXXX-XXXXX_XX-XXXXXXXXX-part2`

# Исправление
ZPOOL_VDEV_NAME_PATH=1 grub-install /dev/sda

#Installing for i386-pc platform.
#Installation finished. No error reported.

# Но можно по-другому:
# ln -s /dev/sda3 /dev/ata-WDC_WDXXXXX-XXXXX_XX-XXXXXXXXX-part2
# затем 
# grub-install /dev/sda

# Пример генерации конфига grub /boot/grub/grub.cfg. Первый рабочий, остальное варианты.

vim /etc/grub.d/40_custom
 
# ---------------

# (0) Arch Linux zfs # Наш случай
menuentry "Arch Linux ZFS" {
    search -u UUID
    # Naming scheme: 
    # linux /zfspart1/zfspart2/@/bootdirectory/kernel
    # initrd /zfspart1/zfspart2/@/bootdirectory/initramfs-linux.img
    linux /root/@/boot/vmlinuz-linux zfs=tank/root rw
    initrd /root/@/boot/initramfs-linux.img
}

# (50) Arch Linux zfs  # Еще один пример
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

# (60) Arch Linux # Если отдельная партиция boot c ext4
menuentry "Arch Linux zfs3" {
    linux /boot/vmlinuz-linux zfs=tank/root rw
    initrd /boot/initramfs-linux.img
}
# ---------------

vim /etc/default/grub
# Коментируем приоритет 0 и выставляем название нашего menuentry
# GRUB_DEFAULT=0
GRUB_DEFAULT='Arch Linux ZFS'

# Генерируем Grub config

# grub-mkconfig -o /boot/grub/grub.cfg 
# ошибка 

# Решение
ZPOOL_VDEV_NAME_PATH=1 grub-mkconfig -o /boot/grub/grub.cfg


# Generating grub configuration file ...
# Found linux image: /boot/vmlinuz-linux
# Found initrd image(s) in /boot: initramfs-linux.img

# Устанавливаем время, etc
ln -f -s /usr/share/zoneinfo/Europe/Moscow /etc/localtime

# Локали
vim /etc/locale.gen

locale-gen
echo "LANG=en_US.UTF-8" >> locale.conf

# Важно!!! Разрешаем zfs службы 
systemctl enable zfs.target 
systemctl enable zfs-import-cache
systemctl enable zfs-mount


# Выходим из chroot
exit


# Отмонируем /mnt/boot (если  boot раздел legacy c ext4)

# Важно!!! Экспорт пула
zfs umount -a
zpool export tank

# Перезагрузка

reboot
```

