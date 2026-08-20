# MR42 Factory NAND Backup

Meraki MR42（IPQ806x / IPQ8064）出厂固件在 **Diagnostic Mode**（免拆机诊断模式，
telnet root @ 192.168.1.1）下 dump 出的 flash 分区备份，刷 OpenWrt 前留存。

- 备份日期：2026-08-20
- 来源固件：`RNAQ-MR1 v0.3.a` / QSDK `enterprise_ap160` / Linux 3.4.103
- 方法：诊断模式 telnet 内 `dd if=/dev/mtdN`，经 nc 传回并校验 md5（与设备端逐字节一致）

## ⚠️ 关于「完整 NAND」的重要说明

诊断模式的精简 initramfs **只把下列 4 个分区映射成 mtd 设备**，
整片 128MB NAND 里的 `bootkernel1/2`、~70MB `ubi` rootfs 等**并未暴露**，
因此**无法从诊断模式 dump 整片 NAND**。真正每台唯一、丢失不可复原的是
`cal`（射频校准 + MAC）与 `ART`——这两个已完整备份。完整整片 dump 需拆机接
UART，在正常/OpenWrt initramfs 下才能看到全部 13 个分区。

## 文件（诊断模式 `/proc/mtd`）

| 文件 | mtd | 大小 | erasesize | 载体 | 内容 |
|---|---|---|---|---|---|
| `mr42-mtd0-cal.bin`    | mtd0 | 0x200000 (2 MiB)    | 0x20000 | NAND | UBI 卷：射频校准 + MAC（重要） |
| `mr42-mtd1-uboot.bin`  | mtd1 | 0x180000 (1.5 MiB)  | 0x20000 | NAND | 出厂 u-boot（刷机写 mtd1 即此分区） |
| `mr42-mtd2-m25p80.bin` | mtd2 | 0x10000 (64 KiB)    | 0x1000  | SPI  | m25p80 (pm25lv512) |
| `mr42-mtd3-ART.bin`    | mtd3 | 0x136000 (~1.2 MiB) | 0x1f000 | -    | ART（无线校准，切勿丢失） |

md5 见 `MD5SUMS.txt`。

## 恢复（回原厂 u-boot，示例）

在诊断模式 telnet：
```sh
echo 1 > /sys/devices/platform/msm_nand/boot_layout
mtd erase /dev/mtd1
nandwrite -pam /dev/mtd1 mr42-mtd1-uboot.bin
echo 0 > /sys/devices/platform/msm_nand/boot_layout
```
`cal` / `ART` 同理写回对应 mtd（非必要不要动）。
