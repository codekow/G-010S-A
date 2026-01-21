# Backup / Restore Firmware

The following instructions will help you make a backup of critical data to restore your device to original state.

Anytime you are modifying firmware you run the risk of bricking your device. With the right skills and tools you should be able to restore this device from a brick.

## Backup / Capture Files From Device

```sh
ssh ONTUSER@192.168.1.10 # SUGAR2A041
```

`ONTUSER@SFP#`

```sh
# make backup dir
mkdir /tmp/dump
cd /tmp/dump

# capture state
cat /proc/mtd > mtd
cat /proc/cmdline > cmdline
uname -a > uname
cat /proc/cpuinfo > cpuinfo

# capture configs
fw_printenv > env.txt
grep goi_config env.txt > env.goi_config.txt

ritool dump > ritool.txt
sfp_i2c -r > eeprom.txt
sfp_ddm_tool dumpA0 > /tmp/dumpA0.txt
```

```sh
# save flash
for i in /dev/mtd*ro
do
  cp $i .
done

# hash flash files
md5sum mtd*ro > md5sum
```

Transfer files from device

```sh
mkdir -p dump/
scp -O ONTUSER@192.168.1.10:/tmp/dump/*  dump/ # SUGAR2A041
```

## Restore Firmware

NOTE: These instructions have **not** been tested on inexperienced users. They are to help those familiar with device hacking and development have a backup for [unbricking](UNBRICKING.md).

In short, good news! If you backed up your firmware, you should be able to recover it (you just might not know how to restore it).

### Run TFTP server

The following example is for running `dnsmasq` on Linux. Any TFTP server will work.

```sh
sudo systemctl stop firewalld
sudo dnsmasq \
  -d \
  -i eth0 \
  -a 0.0.0.0 \
  --port=0 \
  --enable-tftp \
  --tftp-root=$(pwd) \
  --tftp-no-fail \
  --tftp-secure \
  --tftp-no-blocksize
```

Default File List

```list
3FE46542AAAA_ri.bin
3FE46542AAAA_a0.bin
3FE46542AAAA_a2.bin
configfs.image
g010sa-squashfs.image
logfs.image
u-boot.env
```

Default Restore Functions

*Do not use to restore your backup*

```sh
setenv a0_image '3FE46542AAAA_a0.bin'
setenv a2_image '3FE46542AAAA_a2.bin'
setenv ri_image '3FE46542AAAA_ri.bin'
setenv image_name 'g010sa'

setenv kernel0_offs '0xC0000'
setenv kernel1_offs '0x6C0000'
setenv config_iaddr '0xCC0000'
setenv log_iaddr '0xDC0000'
setenv ri_hiaddr '0xFD0000'
setenv a2_iaddr '0xFE0000'
setenv ri_siaddr '0xFF0000'

setenv update_configfs 'tftpboot ${ram_addr} ${tftppath}configfs.image;sf probe 0;sf erase ${config_iaddr} +${filesize};sf write ${ram_addr} ${config_iaddr} ${filesize}'
setenv update_image0 'tftpboot ${ram_addr} ${tftppath}${image_name}-squashfs.image;sf probe 0;sf erase ${kernel0_offs} +${filesize};sf write ${ram_addr} ${kernel0_offs} ${filesize}'
setenv update_image1 'tftpboot ${ram_addr} ${tftppath}${image_name}-squashfs.image;sf probe 0;sf erase ${kernel1_offs} +${filesize};sf write ${ram_addr} ${kernel1_offs} ${filesize}'
setenv update_logfs 'tftpboot ${ram_addr} ${tftppath}logfs.image;sf probe 0;sf erase ${log_iaddr} +${filesize};sf write ${ram_addr} ${log_iaddr} ${filesize}'
setenv update_ri 'tftpboot ${ram_addr} ${tftppath}${ri_image};sf probe 0;sf erase ${ri_hiaddr} +${filesize};sf write ${ram_addr} ${ri_hiaddr} ${filesize};sf erase ${ri_siaddr} +${filesize};sf write ${ram_addr} ${ri_siaddr} ${filesize}'

setenv load_uboot_env 'tftpboot ${ram_addr} ${tftppath}u-boot.env'
setenv import_uboot_env 'env import -t ${ram_addr} ${filesize}'
setenv update_uboot_env 'run load_uboot_env && run import_uboot_env && saveenv'
```

### Restore From Captured Files

| Offset   | Length   | Backup | Description | Default Name          |
|----------|----------|--------|-------------|-----------------------|
| 0x000000 | 0x040000 | mtd1   | uboot       | n/a                   |
| 0x040000 | 0x080000 | mtd2   | uboot_env   | u-boot.env            |
| 0x0C0000 | 0x600000 | mtd3   | image0      | g010sa-squashfs.image |
| 0x6C0000 | 0x600000 | mtd4   | image1      | g010sa-squashfs.image |
| 0xCC0000 | 0x100000 | mtd5   | configfs    | configfs.image        |
| 0xDC0000 | 0x210000 | mtd6   | logfs       | logfs.image           |
| 0xFD0000 | 0x010000 | mtd7   | ri          | 3FE46542AAAA_ri.bin   |
| 0xFE0000 | 0x010000 | mtd8   | sfp         | 3FE46542AAAA_a2.bin   |
| 0xFF0000 | 0x010000 | mtd9   | ribackup    | 3FE46542AAAA_ri.bin   |

Set TFTP Path

```sh
setenv tftppath 'dump-3FE46541AADA/'
```

```sh
setenv mtd1 0x040000
setenv mtd2 0x0C0000
setenv mtd3 0x6C0000
setenv mtd6 0xCC0000
setenv mtd7 0xDC0000
setenv mtd8 0xFD0000
setenv mtd9 0xFE0000
setenv mtd10 0xFF0000

setenv ram_addr 80F00000

setenv restore_mtd1 'tftpboot ${ram_addr} ${tftppath}/mtd1ro;sf probe 0;sf erase ${mtd1} +${filesize};sf write ${ram_addr} ${mtd1} ${filesize}'
setenv restore_mtd2 'tftpboot ${ram_addr} ${tftppath}/mtd2ro;sf probe 0;sf erase ${mtd2} +${filesize};sf write ${ram_addr} ${mtd2} ${filesize}'
setenv restore_mtd3 'tftpboot ${ram_addr} ${tftppath}/mtd3ro;sf probe 0;sf erase ${mtd3} +${filesize};sf write ${ram_addr} ${mtd3} ${filesize}'
setenv restore_mtd6 'tftpboot ${ram_addr} ${tftppath}/mtd6ro;sf probe 0;sf erase ${mtd6} +${filesize};sf write ${ram_addr} ${mtd6} ${filesize}'

setenv restore_mtd7 'tftpboot ${ram_addr} ${tftppath}/mtd7ro;sf probe 0;sf erase ${mtd7} +${filesize};sf write ${ram_addr} ${mtd7} ${filesize}'
setenv restore_mtd8 'tftpboot ${ram_addr} ${tftppath}/mtd8ro;sf probe 0;sf erase ${mtd8} +${filesize};sf write ${ram_addr} ${mtd8} ${filesize}'
setenv restore_mtd9 'tftpboot ${ram_addr} ${tftppath}/mtd9ro;sf probe 0;sf erase ${mtd9} +${filesize};sf write ${ram_addr} ${mtd9} ${filesize}'
setenv restore_mtd10 'tftpboot ${ram_addr} ${tftppath}/mtd8ro;sf probe 0;sf erase ${mtd10} +${filesize};sf write ${ram_addr} ${mtd10} ${filesize}'

setenv restore_firmware 'run restore_mtd1 restore_mtd2 restore_mtd3 restore_mtd6 restore_mtd7 restore_mtd8 restore_mtd9 restore_mtd10'

run restore_firmware
```

```sh
reset
```

## Additional Info

- See [G-010S-A U-Boot Defaults](../convert/G-010S-A.env.txt)

- See [EEPROM Tech Sheet](https://cdn.hackaday.io/files/21599924091616/AN_2030_DDMI_for_SFP_Rev_E2.pdf)
