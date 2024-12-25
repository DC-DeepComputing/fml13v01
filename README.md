# DeepComputing FML13V01 SDK

This builds a complete RISC-V cross-compile toolchain for the `StarFiveTech` `JH7110` SoC. It also builds U-boot SPL, U-boot and a flattened image tree (FIT) image with a Opensbi binary, linux kernel, device tree, ramdisk image and rootfs image for the `DeepComputing FML13V01` board. Note this SDK is built with the linux kernel version `6.6`.

## Prerequisites

Recommend OS: Ubuntu 16.04/18.04/20.04/22.04 x86_64

Install required additional packages:

```
$ sudo apt update
$ sudo apt-get install build-essential automake libtool texinfo bison flex gawk
g++ git xxd curl wget gdisk gperf cpio bc screen texinfo unzip libgmp-dev
libmpfr-dev libmpc-dev libssl-dev libncurses-dev libglib2.0-dev libpixman-1-dev
libyaml-dev patchutils python3-pip zlib1g-dev device-tree-compiler dosfstools
mtools kpartx rsync
```

Additional packages for Git LFS support:

```
$ curl -s https://packagecloud.io/install/repositories/github/git-lfs/script.deb.sh | sudo bash
$ sudo apt-get install git-lfs
```

## Fetch Code Instructions ##

Checkout this repository  (e.g.: branch `fm7110-6.6`). Then checkout all of the linked submodules using:

	$ git clone https://github.com/DC-DeepComputing/fml13v01.git fml13v01-sdk
	$ cd fml13v01-sdk
	$ git submodule update --init --recursive

This will take some time and require around 12GB of disk space. Some modules may fail because certain dependencies don't have the best git hosting. The only solution is to wait and try again later (or ask someone for a copy of that source repository).

For developer, need to switch the 5 submodules `buildroot`, `u-boot`, `linux`, `opensbi`, `soft_3rdpart` to correct branch manually, or refer to `.gitmodule`

	$ git submodule foreach --recursive "git checkout origin/fm7110-6.6 -B fm7110-6.6"

## Quick Build Instructions

Below are the quick building for the initramfs image `image.fit`. The completed toolchain, `u-boot-spl.bin.normal.out`, `visionfive2_fw_payload.img`, `image.fit` will be generated under `work/` directory. The completed build tree will consume about 18G of disk space.

	$ make -j$(nproc)

Then the below target files will be generated:

```
work/
├── visionfive2_fw_payload.img
├── image.fit
├── initramfs.cpio.gz
├── u-boot-spl.bin.normal.out
├── linux/arch/riscv/boot
    ├── dts
    │   └── starfive
    │       └──jh7110-starfive-visionfive-2-framework.dtb
    └── Image.gz
```

Additional command to config buildroot, uboot, linux, busybox:

```
$ make buildroot_initramfs-menuconfig   # buildroot initramfs menuconfig
$ make buildroot_rootfs-menuconfig      # buildroot rootfs menuconfig
$ make uboot-menuconfig                 # uboot menuconfig
$ make linux-menuconfig                 # Kernel menuconfig
$ make -C ./work/buildroot_initramfs/ O=./work/buildroot_initramfs busybox-menuconfig  # for initramfs busybox menuconfig
$ make -C ./work/buildroot_rootfs/ O=./work/buildroot_rootfs busybox-menuconfig        # for rootfs busybox menuconfig
```

Additional command to build single package or module:

```
$ make vmlinux   # build linux kernel
$ make uboot     # build u-boot
$ make -C ./work/buildroot_rootfs/ O=./work/buildroot_rootfs busybox-rebuild   # build busybox package
$ make -C ./work/buildroot_rootfs/ O=./work/buildroot_rootfs ffmpeg-rebuild    # build ffmpeg package
```

## APPENDIX I: Generate Booting SD Card

Create a bootable TF card image with a default size of 16 GB. Note: This will erase all existing data on the target TF card. It is recommended to use a GPT partition table for the TF card. **NOTE THIS WILL DESTROY ALL EXISTING DATA** on the target TF card; The `GPT` Partition Table for the TF card is recommended.

#### Generate SD Card Image File

We could generate a sdcard image file by the below command. The sdcard image file could be copied to sd card or tf card through `dd` command, or `rpi-imager` or `balenaEtcher` tool

```
$ make -j$(nproc)
$ make buildroot_rootfs -j$(nproc)
$ make img
```

The output file `work/sdcard.img`  will be generated.

#### Copy Image File to SD Card

The `sdcard.img` can be copied to a tf card. e.g. through `dd` command as below

```
$ sudo dd if=work/sdcard.img of=/dev/sdX bs=4096
$ sync
```

Then, if the current card's capacity is much larger than the default TF card image (16 GB), you can expand the rootfs partition of the TF card. This can be achieved in the following two ways:

The first way could be done on Ubuntu host, need to install the below package:

```
$ sudo apt install cloud-guest-utils e2fsprogs
```

Then insert the tf card to Ubuntu host, run the below, note `/dev/sdX` is the tf card device.

```
$ sudo growpart /dev/sdX 4  # extend partition 4
$ sudo e2fsck -f /dev/sdX4
$ sudo resize2fs /dev/sdX4  # extend filesystem
$ sudo fsck.ext4 /dev/sdX4
```

The second way could be done on FML13V01 board, use fdisk and resize2fs command：

```
# fdisk /dev/mmcblk1
Welcome to fdisk (util-linux 2.37.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.
This disk is currently in use - repartitioning is probably a bad idea.
It's recommended to umount all file systems, and swapoff all swap
partitions on this disk.
Command (m for help): d
Partition number (1-4, default 4): 4
Partition 4 has been deleted.
Command (m for help): n
Partition number (4-128, default 4): 4
First sector (614400-62333918, default 614400):
): t sector, +/-sectors or +/-size{K,M,G,T,P} (614400-62333918, default 62333918)
Created a new partition 4 of type 'Linux filesystem' and of size 29.4 GiB.
Partition #4 contains a ext4 signature.
Do you want to remove the signature? [Y]es/[N]o: N
Command (m for help): w
The partition table has been altered.
Syncing disks.

# resize2fs /dev/mmcblk1p4
resize2fs 1.46.4 (18-Aug-2021)
Filesystem at /d[
111.756178] EXT4-fs (mmcblk1p4): resizing filesystem from 512000
to 30859756 blocks
ev/mmcblk1p4 is [
111.765203] EXT4-fs (mmcblk1p4): resizing filesystem from 512000
to 30859265 blocks
mounted on /; on-line resizing required
old_desc_blocks = 2, new_desc_blocks = 118
[ 112.141953] random: crng init done
[ 112.145369] random: 7 urandom warning(s) missed due to ratelimiting
[ 115.474184] EXT4-fs (mmcblk1p4): resized filesystem to 30859265
The filesystem on /dev/mmcblk1p4 is now 30859756 (1k) blocks long.
```

If you need to add a new partition, such as a swap partition (here we do set the rest of disk space to swap partition,
but normally swap partition size should be the same as DDR size or double of DDR size),
you can use the following shell script afer the image running on board:

```bash
#!bin/sh
sgdisk -e /dev/mmcblk0
disk=/dev/mmcblk0
gdisk $disk << EOF
p
n
5


8200
p
c
5
hibernation
w
y
EOF

mkswap /dev/mmcblk0p5
swapoff -a
swapon /dev/mmcblk0p5
```
