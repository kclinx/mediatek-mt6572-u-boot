# Mainline U-Boot on MediaTek mt6572

Works: uart.

I ported this to run on a fake "S21 Ultra" that was produced by some obscure company.  On mine, the TX and RX pins for uart0 were marked on the board.

## Build
```
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- mt6572_alps_obscure_defconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- -jx
```
## Installing in "secondary" bootloader mode
For this step, you will need a mediatek "mkimage" tool.  This is NOT the u-boot mkimage tool.  It can be found precompiled [here](https://forum.xda-developers.com/t/guide-building-mediatek-boot-img-appending-headers.2753788/#post-52707851).

```
# Create a dummy initrd so that the stock bootloader will accept it
dd if=/dev/random of=ramdisk bs=2048 count=9

# Append a MediaTek header so UBoot will pass a check
# Once again: this step is NOT using the u-boot mkimage

mkimage ramdisk ROOTFS > ramdisk.img.mtk
mkimage u-boot.bin KERNEL > u-boot.bin.mtk
```
Finally, create the android bootimg. You may need to select other parameters if you have a different device.
```
cat <<EOF >bootimg.cfg
bootsize = 0x600000
pagesize = 0x800
kerneladdr = 0x10008000
ramdiskaddr = 0x11000000
secondaddr = 0x10f00000
tagsaddr = 0x10000100
name = C88QKFD001
cmdline =
EOF

abootimg --create u-boot-mt6572.img -f bootimg.cfg -k u-boot.bin.mtk -r ramdisk.img.mtk
```
Then use bkerler's [mtkclient](https://github.com/bkerler/mtkclient) to flash to the BOOTIMG partition.

```
mtk w BOOTIMG u-boot-mt6572.img
```
## Installing in "first" bootloader mode (untested)

Find the u-boot-mtk.bin file and flash it with the SP Flash Tool to the "UBOOT" section.

ATTENTION: In this mode, the screen will not work, you can see the work of u-boot only by uart.

# Usage

## Running test builds of u-boot.bin by uart
Install ckermit and write a script
```
cat uartboot-mt6582-uboot.script
#!/usr/bin/ckermit

set port /dev/ttyUSB0
set speed 921600
set carrier-watch off
set flow-control none
set prefixing all

echo {loading u-boot}
PAUSE 1

OUTPUT loadb 0x81e00000 921600\{13}
send /home/username/build/../u-boot/u-boot.bin
PAUSE 1
OUTPUT go 0x81e00000\{13}
c
```
## Download and run linux kernel by uart
Write a script for ckermit
```
cat uartboot-mt6582-linux.script
#!/usr/bin/ckermit

set port /dev/ttyUSB0
set speed 921600
set carrier-watch off
set flow-control none
set prefixing all

OUTPUT setenv initrd_high 0x85000000\{13}

echo {loading zImage}
PAUSE 1

OUTPUT loadb ${loadaddr} 921600\{13}
send /home/username/build/linux/../zImage

echo {loading fdt}
PAUSE 1

OUTPUT loadb ${fdt_addr_r} 921600\{13}
send /home/username/build/linux/../mt6582-device.dtb

echo {loading ramdisk}
PAUSE 1

OUTPUT loadb 0x85000000 921600\{13}
send uInitrd
PAUSE 1
OUTPUT bootz ${loadaddr} 0x85000000 ${fdt_addr_r}\{13}
c
```

## Booting linux kernel from sdcard
Write **boot.cmd** as below
```
env set initrd_high 0x85000000
load mmc 1:1 0xac000000 mt6582-prestigio-pmt5008-3g.dtb
load mmc 1:1 0x84000000 zImage
load mmc 1:1 0x85000000 uInitrd
bootz 0x84000000 0x85000000 0xac000000
```

Sign it with mkimage for U-Boot
`mkimage -C none -A arm -T script -d boot.cmd boot.scr`
And put the **boot.scr** file in the root of the sdcard.

Don't forget to include other files too!

Note: uInitrd is created by the command `mkimage -A arm -T ramdisk -C none -n uInitrd -d your_ramdisk  uInitrd`

