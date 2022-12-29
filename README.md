This Repo is deprecated, support is integrated into my uboot-repo

https://github.com/frank-w/u-boot/tree/mtk-atf

# ATF for Bananapi-R64

<img src="bpi-r64-storage.svg">

## requirements

On a x86/x64-host you need cross compile tools for the arm64 architecture

(package list maybe incomplete)

```sh
sudo apt install gcc-aarch64-linux-gnu make gcc
```

## build

- change build.conf to match desired device (emmc/sd) and upload-settings
- copy 64bit uboot.bin to this source directory
- run ./build.sh

built binaries:

```
./build/mt7622/release/bl31.bin
./build/mt7622/release/bl2.bin
./build/mt7622/release/bl2.img
./build/mt7622/release/fip.bin
```
- run ./build.sh upload

## GPT-Layout

new GPT with BPI-BOOT/BPI-ROOT (headless)
```
root@bpi-r64:~# gdisk /dev/mmcblk0

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048            6143   2.0 MiB     8300  fip #BL31+uboot 0x800 (len 0x1000)
   2            6144            7167   512.0 KiB   8300  config #uboot-env 0x1800 (len 0x400), unused
   3            7168            8191   512.0 KiB   8300  rf #calibration data, unused
   4            8192          212991   100.0 MiB   0700  kernel #BPI_BOOT for kernel/dtb/FIT
   5          212992        15269854   7.2 GiB     8300  root #BPI-ROOT for rootfs

/dev/mmcblk0: PTUUID="2bd17853-102b-4500-aa1a-8a21d4d7984d" PTTYPE="gpt"
/dev/mmcblk0p1: PARTLABEL="fip" PARTUUID="18de6587-4f17-4e08-a6c9-d9d3d424f4c5"
/dev/mmcblk0p2: PARTLABEL="config" PARTUUID="19a4763a-6b19-4a4b-a0c4-8cc34f4c2ab9"
/dev/mmcblk0p3: PARTLABEL="rf" PARTUUID="8142c1b2-1697-41d9-b1bf-a88d76c7213f"
/dev/mmcblk0p4: PARTLABEL="kernel" PARTUUID="971f7556-ef1a-44cd-8b28-0cf8100b9c7e"
/dev/mmcblk0p5: PARTLABEL="root" PARTUUID="c0cb3022-76d5-42d6-a370-fd1c5594f2b2"
```
## Install

### EMMC with upstream uboot

needs option CONFIG_SUPPORT_EMMC_BOOT=y in installed uboot (sdcard) for "mmc partconf"!

```sh
#flash bl2
setenv ipaddr 192.168.0.19
setenv serverip 192.168.0.11
setenv loadaddr 0x44000000
mmc dev 0
mmc partconf 0 1 1 1
if tftp ${loadaddr} ${serverip}:bl2_emmc.img;then mmc erase 0x0 0x400;mmc write ${loadaddr} 0x0 0x400;fi
mmc partconf 0 1 1 0
```
```sh
mmc dev 0
#mmc partconf 0 1 1 0
setenv loadaddr 0x44000000
setenv serverip 192.168.0.10
#write emmc_header
if tftp ${loadaddr} ${serverip}:header_emmc.bin ;then mmc erase 0x0 0x400;mmc write ${loadaddr} 0x0 0x1;fi
#write gpt
if tftp ${loadaddr} ${serverip}:bpi-r64_headless.gpt;then mmc write ${loadaddr} 0x1 0x3FF; fi
#write fip
if tftp ${loadaddr} ${serverip}:fip_emmc.bin;then mmc erase 0x800 0x1000;mmc write ${loadaddr} 0x800 0x1000;fi
```

### SD via dd

## build and install img
```sh
#backup old bootchain
sudo dd if=/dev/sdb of=atf-backup.img bs=512 count=6144
#build new one
./build.sh
./build.sh rename
./build.sh img
#install new bootchain
sudo dd if=bpi-r64_sdmmc.img of=/dev/sdb
sync
```

## manual install parts
```sh
target=/dev/sdb
sudo dd of=$target if=header_sdmmc.bin bs=512 seek=0
sudo dd of=$target if=bpi-r64_headless.gpt bs=512 seek=1
sudo dd of=$target if=./bl2_sdmmc.img bs=512 seek=1024
sudo dd of=$target if=./fip_sdmmc.bin bs=512 seek=2048
```

## strip own gpt

without first 512 bytes (gpt only, like bpi-r64_headless.bin):

```sh
npart=4;sudo dd if=/dev/sdb of=bpi-r64_headless.gpt bs=1 skip=512 count=$(( $npart*128 +1024 ))
```
change npart to number of partitions in your GPT

## booting linux kernel

### booting fit-image

my kernel-repo creates for r64 (arch=arm64) also an FIT-Image (*.itb) which needs to be uploaded to tftp

```sh
setenv serverip 192.168.0.10
setenv loadaddr 0x44000000
tftp ${loadaddr} bpi-r64-5.4.77-main.itb
bootm ${loadaddr}
```

as there is no rootfs yet you need a initramfs to create filesystem on emmc

### create filesystem from running system

```sh
root@bpi-r64:~# mkfs.vfat -n "BPI-BOOT" /dev/mmcblk0p4
root@bpi-r64:~# mkfs.ext4 -L "BPI-ROOT" /dev/mmcblk0p5
```

current u-boot binary supports variable "fit" in uEnv.txt, so put your itb in

bananapi/bpi-r64/linux/ of /dev/mmcblk0p4

and set this in this uEnv.txt:

```
fit=bpi-r64-5.4.77-main.itb
```
