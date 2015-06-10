<!-- -*- mode:markdown;tab-width:2;indent-tabs-mode:nil;coding:utf-8 -*-
     vim: ft=markdown syn=markdown fileencoding=utf-8 sw=2 ts=2 ai eol et si
     README.md: Dedibox Fully Encrypted Debian installation
     (c) 2011-15 Laurent Vallar <val@zbla.net>, WTFPL license v2 see below.

     This program is free software. It comes without any warranty, to
     the extent permitted by applicable law. You can redistribute it
     and/or modify it under the terms of the Do What The Fuck You Want
     To Public License, Version 2, as published by Sam Hocevar. See
     http://www.wtfpl.net/ for more details.
  -->

# Dedibox Fully Encrypted Debian Install

## TL;DR

1. Do a normal Debian server install but use swap partition as root filesystem.
2. After the first boot crypt other partitions but `/boot`.
3. Install new root filesystem on crypted target.
4. Do some stuff to get all workging after next reboot.
5. Reboot and log in with `ssh` while in boot process (`initramfs`+`dropbear`).
6. Decrypt all then log out.
7. Enjoy your fully encrypted Debian system.

## Final objectives

All partition but _/boot_ will be encrypted. Current partition layout is
based on disk space available for a Dedibox SC gen2, i.e. 1x500 Go SSHD.

| Type     | Filesystem | Mount point  | Size      | After encryption        |
|:---------|:-----------|:-------------|----------:|:------------------------|
| Primary  | Ext4       | /boot        |    524 Mo | /boot (_not encrypted_) |
| Primary  | _Swap_     | _none_       |   2064 Mo | _Swap_ (_encrypted_)    |
| Primary  | Ext4       | /            |   5240 Mo | / (_encrypted_)         |
| Primary  | Ext4       | /data        | 469108 Mo | /data (_encrypted_)     |

## Online.net classical install

### Server OS type

![dedibox os_type](dedibox_os_type.png){: style="margin: 0px auto;
display: block; max-width: 90%; border: 2px solid gray;
border-radius: 10px 10px"}

### Distribution

![dedibox os_choice](dedibox_os_choice.png){: style="margin: 0px auto;
display: block; max-width: 90%; border: 2px solid gray;
border-radius: 10px 10px"}

### Online.net installation console screenshot

_Online.net_ installation partitioning is a little buggy : if free space is left
swap partition will get all remaining space. That's why I created an
fourth primary partition to handle properly available free space.

![dedibox partitioning](dedibox_partitioning.png){: style="margin: 0px auto;
display: block; max-width: 90%; border: 2px solid gray;
border-radius: 10px 10px"}

## Prepare encryption

### Log in as <configured_user> and set SSH authorized key

<!--
# cat ~/.ssh/id_rsa.pub | ssh root@<your_dedibox_IP_or_FQDN> \
#   -o StrictHostKeyChecking=no -o PasswordAuthentication=yes \
#   "( install -m 700 -d /root/.ssh && cat >> /root/.ssh/authorized_keys )"
  -->

~~~~~
ssh-keyscan -H <your_dedibox_IP_or_FQDN> >> ~/.ssh/known_hosts
ssh-copy-id <user>@<your_dedibox_IP_or_FQDN>
~~~~~

### Copy SSH authorized key to root account and continue as root

~~~~~
su -
cp -a ~<user>/.ssh /root
chown root.root -R /root/.ssh
# Change passwords
passwd <user>
passwd
~~~~~

### Cleanup Online.net installation & shutdown all services

~~~~~
apt-get remove -y --purge bind9 bind9utils
rm -rf /var/cache/bind
for daemon in exim4 openntpd acpid udev atd cron rsyslog dbus; do
  systemctl stop $daemon
done
~~~~~

### Disable swap & umount configured /data

~~~~~
swapoff -a
umount /data
~~~~~

### Save actual partitioning

~~~~
sfdisk -d /dev/sda > sda.sfdisk
~~~~

### Change encrypted partition types


~~~~
perl -pe 's|^(.*, Id=)8[23]$|\1da|g' -i sda.sfdisk
~~~~

### Apply partition types modification

~~~~~
sfdisk --no-reread --force /dev/sda < sda.sfdisk
rm -f sda.sfdisk
~~~~~

### Encrypt future root and use /data mount point temporarily

~~~~~
/bin/echo -e "console-setup\tconsole-setup/charmap47\tselect\tUTF-8" |\
debconf-set-selections
apt-get install -y cryptsetup
cryptsetup luksFormat /dev/sda3
cryptsetup luksOpen /dev/sda3 sda3_crypt
mkfs.ext4 -j -m 0.1 -L ROOT \
  -O dir_index,extent,filetype,sparse_super,huge_file /dev/mapper/sda3_crypt
mount /dev/mapper/sda3_crypt /data
~~~~~

## Install new system

### Do a fresh install on new encrypted root

~~~~~
debian_mirror=http://http.debian.net/debian
debian_codename=jessie # change with target distribution
debootstrap_base_url=${debian_mirror}/pool/main/d/debootstrap
debootstrap_version=\
$(wget ${debootstrap_base_url} -q -O - |\
  egrep 'debootstrap_.*_all.deb' |\
  sed -e 's/^.*>debootstrap_\(.*\)_all\.deb<.*$/\1/g' |\
  sort |tail -n 1)
wget -c -q ${debootstrap_base_url}/debootstrap_${debootstrap_version}_all.deb
dpkg -i debootstrap_*_all.deb
debootstrap ${debian_codename} /data ${debian_mirror}
~~~~~

### Prepare SSH root connection for later use

~~~~~
cp -a /root/.ssh /data/root/.
~~~~~

## Make new system bootable

### Chroot into encrypted root

~~~~~
umount /boot
mount -o bind /sys /data/sys
mount -o bind /proc /data/proc
mount -o bind /dev /data/dev
mount -o bind /dev/pts /data/dev/pts
mount -o bind /run /data/run
mount -o bind /run/lock /data/run/lock
mount -o bind /run/shm /data/run/shm
chroot /data
export TERM=xterm-color
export LANG=C.UTF-8
export LC_ALL=C.UTF-8
~~~~~

### Configure Apt

~~~~~
cat <<EOF> /etc/apt/sources.list
deb http://ftp.fr.debian.org/debian/ jessie main contrib non-free
deb-src http://ftp.fr.debian.org/debian/ jessie main contrib non-free

deb http://security.debian.org/ jessie/updates main contrib non-free
deb-src http://security.debian.org/ jessie/updates main contrib non-free

# jessie-updates, previously known as 'volatile'
deb http://ftp.fr.debian.org/debian/ jessie-updates main contrib non-free
deb-src http://ftp.fr.debian.org/debian/ jessie-updates main contrib non-free

# jessie-backports, previously on backports.debian.org
deb http://ftp.fr.debian.org/debian/ jessie-backports main contrib non-free
deb-src http://ftp.fr.debian.org/debian/ jessie-backports main contrib non-free
EOF
cat <<EOF> /etc/apt/apt.conf.d/30disable-recommends-and-suggests
APT::Install-Recommends "0";
APT::Install-Suggests "0";
EOF
cat <<EOF> /etc/apt/apt.conf.d/99disable-translations
APT::Acquire::Languages "none";
EOF
~~~~~

### Install cryptsetup inside new root

~~~~~
/bin/echo -e "console-setup\tconsole-setup/codeset47\tselect\t"\
"# Latin1 and Latin5 - western Europe and Turkic languages\n"\
"console-setup\tconsole-setup/codesetcode\tstring\tLat15\n"\
"console-setup\tconsole-setup/charmap47\tselect\tUTF-8" |\
debconf-set-selections
apt-get update
apt-get -y install cryptsetup
apt-get -y autoremove
apt-get clean
~~~~~

### Encrypt future /data and configure crypttab

~~~~~
cryptsetup luksFormat /dev/sda4
cryptsetup luksOpen /dev/sda4 sda4_crypt
mkfs.ext4 -j -m 0 -L DATA \
  -O dir_index,extent,filetype,sparse_super,huge_file /dev/mapper/sda4_crypt
cat <<EOF>> /etc/crypttab
# encrypted swap
sda2_crypt /dev/sda2 /dev/urandom cipher=aes-cbc-essiv:sha256,size=256,swap
# encrypted root
sda3_crypt UUID=`cryptsetup luksUUID /dev/sda3` none luks
# encrypted /data
sda4_crypt UUID=`cryptsetup luksUUID /dev/sda4` none luks
EOF
~~~~~

### Create fstab

~~~~~
cat <<EOF> /etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>

# /boot on /dev/sda1
UUID=`blkid -s UUID -o value /dev/sda1` /boot ext4 defaults 0 2

# swap on /dev/mapper/sda2_crypt (encrypted /dev/sda2)
/dev/mapper/sda2_crypt none swap sw 0 0

# root on /dev/mapper/sda3_crypt (encrypted /dev/sda3)
UUID=`blkid -s UUID -o value /dev/mapper/sda4_crypt` / ext4 errors=remount-ro \
0 1

# data on /dev/mapper/sda4_crypt (encrypted /dev/sda4)
UUID=`blkid -s UUID -o value /dev/mapper/sda4_crypt` /data ext4 defaults 0 2

EOF
~~~~~

### Configure Network Interfaces

~~~~~
cat <<EOF> /etc/network/interfaces.d/00_loopback
# The loopback network interface
auto lo
iface lo inet loopback
EOF
cat <<EOF> /etc/network/interfaces.d/01_primary
# The primary network interface
auto eth0
iface eth0 inet dhcp
EOF
~~~~~

### Install kernel and grub

~~~~~
mount /boot
rm -rf /boot/*
grub_disk=$(ls /dev/disk/by-id/ |egrep -v '(part[0-9]+|crypt)$' |grep ata)
/bin/echo -e \
"grub-pc\tgrub-pc/install_devices\tmultiselect\t/dev/disk/by-id/${grub_disk}" |\
debconf-set-selections
apt-get install -y linux-image-amd64 grub-pc
update-grub
grub-install /dev/sda
~~~~~

### Make previous root partition a little unregognizable (optional)

~~~~~
dd if=/dev/urandom of=/dev/sda2 bs=1M count=10
~~~~~

### Configure Time Zone (optional)

~~~~~
/bin/echo -e "tzdata\ttzdata/Areas\tselect\tEurope\n\
tzdata\ttzdata/Zones/Europe\tselect\tParis\n\
tzdata\ttzdata/Zones/Etc\tselect\tUTC\n" |\
debconf-set-selections
echo "Europe/Paris" > /etc/timezone
dpkg-reconfigure -f noninteractive tzdata
~~~~~

### Configure Locale (optional)

~~~~~
/bin/echo -e \
"locales\tlocales/locales_to_be_generated\tmultiselect\ten_US.UTF-8 UTF-8\n"\
"locales\tlocales/default_environment_locale\tselect\tC.UTF-8\n"\
"localepurge\tlocalepurge/use-dpkg-feature\tboolean\ttrue\n"\
"localepurge\tlocalepurge/nopurge\tmultiselect\ten_US.UTF-8" |\
debconf-set-selections
apt-get install -y locales localepurge
~~~~~

### Install SSH

~~~~~
apt-get install -y openssh-server
~~~~~

### Install dropbear

~~~~~
apt-get install -y dropbear busybox
~~~~~

### Copy your SSH public key to initramfs root account

~~~~
rm -rf /etc/initramfs-tools/root/.ssh/*
cp -a /root/.ssh/authorized_keys /etc/initramfs-tools/root/.ssh
~~~~

### Make SSH keys dropbear's ones

~~~~~
rm -f /etc/initramfs-tools/etc/dropbear/*
/usr/lib/dropbear/dropbearconvert openssh dropbear \
  /etc/ssh/ssh_host_rsa_key  \
  /etc/initramfs-tools/etc/dropbear/dropbear_rsa_host_key
/usr/lib/dropbear/dropbearconvert openssh dropbear \
  /etc/ssh/ssh_host_dsa_key \
  /etc/initramfs-tools/etc/dropbear/dropbear_dss_host_key
/usr/lib/dropbear/dropbearconvert openssh dropbear \
  /etc/ssh/ssh_host_ecdsa_key \
  /etc/initramfs-tools/etc/dropbear/dropbear_ecdsa_host_key
rm -f /etc/dropbear/dropbear_{rsa,dss,ecdsa}_host_key
cp -a /etc/initramfs-tools/etc/dropbear/dropbear_{rsa,dss,ecdsa}_host_key \
  /etc/dropbear
~~~~~
  
### Change SSH listen port if needed (optional)

~~~~
ssh_port=<your SSH listen port>
# Ensure package update will not lost anything
dpkg-divert --add --rename --divert \
  /usr/lib/dropbear/initramfs-tools_script_init-premount_dropbear.real \
  /usr/share/initramfs-tools/scripts/init-premount/dropbear
# Add optional listen port
perl -pe 's|^(/sbin/dropbear).*$|\1 -s -p '${ssh_port}'|' \
  /usr/lib/dropbear/initramfs-tools_script_init-premount_dropbear.real \
  > /usr/share/initramfs-tools/scripts/init-premount/dropbear
# Make script executable
chmod 700 /usr/share/initramfs-tools/scripts/init-premount/dropbear
# Change SSH listen port
perl -pe 's/^(Port) \d+$/\1 '${ssh_port}'/g' -i /etc/ssh/sshd_config
~~~~

### Configure initramfs

~~~~~
# Add eth0 as listen interface
perl -pe 's/^(DEVICE=)$/\1eth0/' -i /etc/initramfs-tools/initramfs.conf
# Add dropbear on boot
cat <<EOF>> /etc/initramfs-tools/initramfs.conf
# Launch dropbear at boot time
DROPBEAR=y
EOF
~~~~~

### Install `start_dm_crypt` boot time helper script

~~~~
cat <<EOF> /etc/initramfs-tools/start_dm_crypt
#!/bin/sh

PATH=/sbin:/usr/sbin:/bin:/usr/bin
export PATH

SLEEP_TIME=3

sep()
{
  echo \
'------------------------------------------------------------------------------'
}

start_dm_crypt()
{
  sep; echo 'unlocking /data'
  /sbin/cryptsetup luksOpen /dev/sda4 sda4_crypt

  cat <<EOS> /conf/conf.d/cryptroot
target=sda3_crypt,source=UUID=`cryptsetup luksUUID /dev/sda3`,key=none,rootdev
EOS

  sep; echo 'unlocking / & exit'
  /scripts/local-top/cryptroot
  
  sep; read -p 'quit & continue boot? (Y/n)' resp
  if [ "\$resp" == "Y" -o "\$resp" == "y" -o "\$resp" == "" ]
  then
    for i in 1 2 3
    do
      sep; echo "sending return on passfifo (try #\${i})"
      echo '' > /lib/cryptsetup/passfifo
      sep; echo "sleeping \${SLEEP_TIME}s..."
      sleep \$SLEEP_TIME
    done
  fi
}

start_dm_crypt

EOF
chmod a+x /etc/initramfs-tools/start_dm_crypt
~~~~

### Install `install_start_dm_crypt` initramfs hook script

~~~~
cat <<EOF> /etc/initramfs-tools/hooks/install_start_dm_crypt
#!/bin/sh
PREREQ=''
prereqs()
{
  echo "\$PREREQ"
}

case \$1 in
prereqs)
  prereqs
  exit 0
  ;;
esac

. /usr/share/initramfs-tools/hook-functions

copy_exec /etc/initramfs-tools/start_dm_crypt /sbin/start_dm_crypt

/usr/lib/dropbear/dropbearconvert openssh dropbear \\
  /etc/ssh/ssh_host_rsa_key \\
  /etc/initramfs-tools/etc/dropbear/dropbear_rsa_host_key

/usr/lib/dropbear/dropbearconvert openssh dropbear \\
  /etc/ssh/ssh_host_dsa_key \\
  /etc/initramfs-tools/etc/dropbear/dropbear_dss_host_key

/usr/lib/dropbear/dropbearconvert openssh dropbear \\
  /etc/ssh/ssh_host_ecdsa_key \\
  /etc/initramfs-tools/etc/dropbear/dropbear_ecdsa_host_key

EOF
chmod a+x /etc/initramfs-tools/hooks/install_start_dm_crypt
~~~~

### Update initramfs

~~~~
update-initramfs -u -kall
~~~~

### Install some usefull stuff (optional)

~~~~
apt-get install -y vim bash-completion
sed -e '/^#if ! shopt -oq posix; then/,/^#fi/ s/^#\(.*\)/\1/g' \
  -i /etc/bash.bashrc
~~~~

### Sync disk & exit

~~~~
sync
exit
exit
~~~~

## Force reboot and finish

Go to _online.net_ console and do a forced (electrical) reboot.
On first login using SSH type `/sbin/start_dm_crypt` then logout, full system
will boot up and be available shortly.
