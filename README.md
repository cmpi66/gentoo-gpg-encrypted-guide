# Gentoo GPG Encrypted Yubiey UEFI install

A simple guide of the major commands to run to install an Encrypted Gentoo system that can unlock with an opengpg configured Yubikey. We're also using Wayland so there will be some Wayland specific commands.


## partition the disks 

First we'll securely wipe our drive, this is recommended even if your drives are brand new. Although it will take time even on an NVME drive. The recommended amount of times to pass it through would be three but you decide how many times you want to do it. The reason why we're doing this is to basically write over the free storage space with one's and zero's to make the data unrecoverable for rogue individuals.

```
shred --verbose --random-source /dev/urandom --iterations 3 /dev/nvme0n1
```


Use fdisk to create three partitions; root, extended boot and efi:

```
fdisk /dev/nvme0n1
```

Use 'n' for new partition and give efi and boot 1G each to be on the safe side. Then press 't' for type and give them the following types:


- efi type 1
- boot type 136
- root type 23

Probe for cryptsetup:

```
modprobe dm-crypt
```

## Password ecnrytped


```
cryptsetup luksFormat --key-size 512 /dev/nvme0n1p1
```

backup img: 

```
cryptsetup luksHeaderBackup /dev/nvme0n1p3 --header-backup-file root_headers.img
```

Open drive:

```
cryptsetup luksOpen /dev/nvme0n1p3 root
```

**Create file system AFTER encrypted**

```
mkfs.vfat -F32 /dev/nvme0n1p1
mkfs.ext4 -L boot /dev/nvme0n1p2
mkfs.btrfs -L rootfs /dev/mapper/root
```


## GPG encrypted 

```
export GPG_TTY=$(tty)
```

Import your public GPG key file and trust it ultimately

Create the GPG file with your keys email as recipient:

```
dd bs=8388608 count=1 if=/dev/urandom | gpg --recipient myemail@example.com --output crypt_key.luks.gpg --encrypt
```
Format the drive with the key:

```
gpg --decrypt crypt_key.luks.gpg | cryptsetup luksFormat --key-size 512 /dev/nvme0n1p3 -
```

OPEN up the drive:

```
gpg --decrypt crypt_key.luks.gpg | cryptsetup --key-file - open /dev/nvme0n1p3 root
```

**Backup header file:** This is crucial, it literally saved when I accidentally reformatted my drive, save it somewhere secure and back it up to multiple places.

```
cryptsetup luksHeaderBackup /dev/nvme0n1p3 --header-backup-file crypt_headers.img
```

## Gentoo installation begins


```
mkdir /mnt/gentoo
cd /mnt/gentoo
mount --label rootfs /mnt/gentoo
```

Download the Gentoo tarball:

```
links https://www.gentoo.org/downloads/mirrors/
```

Extract the tarball:

```
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
```


Bring your make.conf if you have one already:

```
vi /mnt/gentoo/etc/portage/make.conf 
```

**ESSENTIAL USE FLAGS**

- Grub: device mapper 
- Gnupg: usb
- make.conf crypt cryptsetup 

Configure Mirrors:

``` 
mirrorselect -i -o >> /mnt/gentoo/etc/portage/make.conf
```

Add the repo:

```
mkdir --parents /mnt/gentoo/etc/portage/repos.conf
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
```

## Chroot into system


```
arch-chroot /mnt/gentoo
source /etc/profile
export PS1="(chroot) ${PS1}"
```

**MAKE SURE BOOT IS MOUNTED**:


```
mkdir /efi
mount /dev/nvme0n1p1 /efi
mount /dev/nvme0n1p2 /boot
```

Sync the system for up to date packages:

```
emerge-webrsync
emerge --sync
```

Emerge Git and configure your CPU flags:

```
emerge -av vim eix app-eselect/eselect-repository dev-vcs/git
```

```
emerge --ask app-portage/cpuid2cpuflags
```

```
echo "*/* $(cpuid2cpuflags)" > /etc/portage/package.use/00cpu-flags
```

## Update system and install generic kernel for now


```
emerge -uvDNa @world
```

```
echo "America/New_York" > /etc/timezone
emerge --config sys-libs/timezone-data
```

```
vim  /etc/locale.gen
locale-gen
```

```
eselect locale list
```

```
env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
```

**DIST KERNEL AT THE BEGINING**

```
emerge --ask --autounmask sys-kernel/linux-firmware sys-kernel/gentoo-kernel-bin 
```

```
eselect kernel set 1
```

Make sure efi 64 is in make.conf before emerging grub:

```
emerge -av grub gentoolkit btrfs-progs
```


Edit fstab:

## Encrypted

You would obviously get your UUID:

```
UUID=044502df-6f                                /efi            vfat            noauto,noatime  1 2
LABEL=boot                                      /boot           ext4            defaults        0 2
LABEL=rootfs                                    /               btrfs           defaults        0 1
```

Edit Dracut:

```
vim dracut.conf
```


```
add_dracutmodules+=" crypt crypt-gpg btrfs dm rootfs-block "
```

Here we're unlocking the root drive and we're also saving the crypt_key.luks.gpg file we created earlier in our boot drive. The first UUID would be the unlocked root partition and the second would be the crypt partition:

```
kernel_cmdline+=" root=UUID=31d9edbd-643f-4bec-9d60-9eaae271d5e9  rd.luks.uuid=ec787b20-fb88-4a49-87ae-012417b1b865 rd.luks.key=/crypt_key.luks.gpg:UUID=20E3-B825 "
```

```
mkdir /etc/dracut.conf.d/
```

Copy public GGP to dracut directory: 

```
/etc/dracut.conf.d/crypt-public-key.gpg
```
Rebuild dracut image with your current kernel verison it will most likely be different from the live image:

```
dracut --kver=
```


## System and services configuration

```
echo YOURHOSTNAME > /etc/hostname
```

Networking:

```
emerge --ask net-misc/dhcpcd 
```

```
rc-update add dhcpcd default
rc-service dhcpcd start
```


You would uncomment this from the /etc/conf.d/net file or whatever your NIC's name is: config_eth0="dhcp"

```
emerge --ask --noreplace net-misc/netifrc
vim /etc/conf.d/net
```

```
cd /etc/init.d
ln -s net.lo net.eth0
rc-update add net.eth0 default
```

Configure Hosts:
```
vim /etc/hosts
```

Add a password for root:

```
passwd root 
```

Emerge a few more packages

```
emerge --ask sys-process/cronie app-admin/syslog-ng emerge --ask sys-apps/mlocate app-shells/bash-completion net-misc/chrony sys-block/io-scheduler-udev-rules
```


```
rc-update add cronie default
rc-update add sshd default
rc-update add seatd
rc-update add dbus
rc-update add elogind
rc-update add smartd
rc-update add pcscd
rc-update add syslog-ng
rc-update add zfs-share
rc-update add libvirtd
rc-update add libvirtd-guests
```

## Grub and user configuration


```
grub-install --target=x86_64-efi --efi-directory=/efi
```
```
grub-mkconfig -o /boot/grub/grub.cfg
```
```
useradd -m -G users,wheel,audio,video,seat,pipewire USER
```

```
exit
```

```
umount -l /mnt/gentoo/dev{/shm,/pts,}
umount -R /mnt/gentoo
```

```
reboot
```
