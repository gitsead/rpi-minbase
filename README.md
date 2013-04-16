rpi-minbase
===========
Introduction
------------
This guide is designed to help you to manually create Debian Linux (wheezy) base system (minbase) SD card images for the Raspberry Pi platform. Advanced knowledge of the Linux Operating System and of using the Linux command line is required and recommended.  
Summary of Approach
-------------------
The following is a summary of the approach to create a Debian Linux SD card image:
- Create a SD card image file and mount it via loopback
- Create Partition table on the image file using fdisk
 - Boot Partition: VFAT32 (64 MB)
 - Root Partition: EXT4
- Create Linux file systems on the image file using mkfs
- Building QEMU (qemu-rpi) with support for the "arm1176" cpu
- Bootstraping the Debian (wheezy) base system (minbase) for "armel" architecture
- Configure base system settings

Requirements
------------
This guide was tested on a Debain Linux "wheezy" (i386) base system (minbase) with Standard system utilities (tasksel) running inside VMWare Workstation. The following list of packages are required and must have been installed (apt-get):
```
apt-get install build-essential git pkg-config zlib1g-dev libglib2.0-dev libpixman-1-0
apt-get install binfmt-support debootstrap kpartx lvm2 dosfstools
```
Getting started...
------------------
###Building of QEMU
The bootstrapping process requires QEMU to emulate armel target architecture. The latest version of QEMU from the Debian package repository may have problems to emulate armel and armhf architectures. It's recommended to use the qemu-rpi repository that already includes all neccessary patches. qemu-rpi was created by Gregory Estrade and is available at https://github.com/Torlus/qemu-rpi. It's only required to build the static-compiled QEMU binaries for the arm target architecture (make install is not required).
```
git clone https://github.com/Torlus/qemu.git
cd qemu
./configure --target-list=arm-linux-user --static
<strong>make</strong>
```

The required staticly linked QEMU binary is located at `./qemu/arm-linux/qemu-arm`.
###Preparing the SD card image
A empyy file is created and set up as loop device. The required Boot and Root partitions are created with fdisk and formated with the corresponding filesystems VFAT32 and EXT4.
```
dd if=/dev/zero of=rpi-minbase.img bs=1MB count=1000
losetup -f --show rpi-minbase.img
fdisk /dev/loop0
```

Create the Boot partition (FAT32) using fdisk:
```
Command (m for help): n
Command action
   e   extended
   p   primary partition (1-4)
p
Partition number (1-4): 1
First cylinder (1-121, default 1): 1
Last cylinder, +cylinders or +size{K,M,G} (1-121, default 121): +64M

Command (m for help): t
Selected partition 1
Hex code (type L to list codes): c
Changed system type of partition 1 to c (W95 FAT32 (LBA))
```

Create the Root parition (Linux) using fdisk:
```
Command (m for help): n
Command action
   e   extended
   p   primary partition (1-4)
p
Partition number (1-4): 2
First cylinder (10-121, default 10): 10
Last cylinder, +cylinders or +size{K,M,G} (10-121, default 121): 121
Command (m for help): w
The partition table has been altered!
```

Create device maps for the artition table of the file image (loop device)
```
modprobe dm_mod
kpartx -va ./rpi-minbase.img
```

Build FAT32 and EXT4 file systems for the created Boot and Root partitions.
```
mkfs.vfat /dev/loop1p1
mkfs.ext4 /dev/loop1p2

mkdir ./rpi_rootfs
mount /dev/loop1p2 ./rpi_rootfs
```

###Bootstraping the base system (first stage)
The bootstrappping process is divided into two stages. In the first stage the Debian packages to be installed are only initialy unpacked but not fully installed. In the second stage the packages are finaly installed. The different stages are required because the new bootstrapped system will use a different target architecture (armel).
```
debootstrap --foreign --arch armel wheezy ./rpi_rootfs http://ftp.debian.org/debian/
```
###Bootstraping the base system (second stage)
The second stage of the bootstrapping proccess is performed within chroot enviroment. QEMU is used to emulate the required armel target architecture. The static compiled qemu-arm binary created earlier (-> "Building of QEMU") needs to be copied inside of the chroot enviroment to be accessable after the chroot enviroment has started.
```
cp ./qemu/arm-linux/qemu-arm ./rpi_rootfs/usr/bin
LANG=C chroot ./rpi_rootfs /debootstrap/debootstrap --second-stage
```
###Inital configuration within the base system (chroot)
```
mount /dev/loop1p1 ./boot
```

Create file `/boot/cmdline.txt`:
```
dwc_otg.lpm_enable=0 console=ttyAMA0,115200 kgdboc=ttyAMA0,115200 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 rootwait
```

Create file `/etc/fstab`:
```
proc            /proc           proc    defaults        0       0
/dev/mmcblk0p1  /boot           vfat    defaults        0       0
```

Create file `/etc/hostname`:
```
rpiminbase
```

Create file `/etc/network/interfaces`:
```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
```

Set password of the root user:
```
passwd root
```

###Initial configuration of APT
Create file `/etc/apt/sources.list`:
```
deb http://ftp.de.debian.org/debian wheezy main contrib non-free
deb http://ftp.de.debian.org/debian wheezy-updates main contrib non-free
deb http://security.debian.org/ wheezy/updates main contrib non-free
```

Update APT collection
```
apt-get update
```
###Inital of Console and Timezone
Configure locales:
```
apt-get install locales
dpkg-reconfigure locales
```

Configure keyboard:
```
apt-get install console-data
```

Configure tzdata:
```
dpkg-reconfigure tzdata
```

###Installation of basic packages
```
apt-get install git-core ca-certificates binutils iputils-ping dnsutils
```

###Updating and bootstapping the Rasberry PI Firmware
```
wget http://goo.gl/1BOfJ -O /usr/bin/rpi-update
chmod +x /usr/bin/rpi-update
mkdir -p /lib/modules/3.1.9+
touch /boot/start.elf
rpi-update
```

###Cleanup
```
aptitude update
aptitude clean
apt-get clean
```
