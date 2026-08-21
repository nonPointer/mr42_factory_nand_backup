# Meraki MR42 — Factory NAND Backup

Cisco Meraki **MR42**（Qualcomm IPQ8064 / ipq806x）刷 OpenWrt 之前的**出厂 NAND 完整备份**。

- 备份日期：2026-08-20
- 出厂固件：`RNAQ-MR1 v0.3.a` / QSDK `IPQ806X.LN.1.3.4-CSu2(r00057.1)` / Linux 3.4.103
- NAND：128 MiB，page 2048 B，erase 128 KiB，OOB 64 B（`nandid 1580a101`）
- 全部文件经 `xz` 压缩；`SHA256SUMS.txt` 记录的是**解压后 `.bin`** 的校验值

```sh
xz -d factory-nand/*.xz extras/*.xz
shasum -a256 -c SHA256SUMS.txt
```

## `factory-nand/` — 出厂 13 个分区（完整）

在 cryptid OpenWrt initramfs 下 dump（此时全部分区才可见；Meraki 诊断模式只暴露 4 个）。

| 文件 | 分区 | 大小 | 说明 |
|---|---|---|---|
| `mtd00-sbl1.bin` | sbl1 | 256 KiB | 一级 bootloader |
| `mtd01-mibib.bin` | mibib | 1.25 MiB | 分区表 |
| `mtd02-sbl2.bin` | sbl2 | 1.25 MiB | 二级 bootloader |
| `mtd03-sbl3.bin` | sbl3 | 2.5 MiB | 三级 bootloader |
| `mtd04-ddrconfig.bin` | ddrconfig | 1.125 MiB | DDR 参数 |
| `mtd05-ssd.bin` | ssd | 1.125 MiB | — |
| `mtd06-tz.bin` | tz | 2.5 MiB | TrustZone |
| `mtd07-rpm.bin` | rpm | 2.5 MiB | RPM 固件 |
| **`mtd08-u-boot.bin`** | u-boot | 1.5 MiB | **出厂 u-boot（恢复原厂的关键）**※ |
| `mtd09-bootkernel1.bin` | bootkernel1 | 10.5 MiB | Meraki 内核槽 1 |
| `mtd10-bootkernel2.bin` | bootkernel2 | 10.5 MiB | Meraki 内核槽 2 |
| `mtd11-ubi.bin` | ubi | 70.75 MiB | Meraki rootfs（UBI：diagnostic1 / part.safe / part.old / storage） |
| `mtd12-art.bin` | art | 2 MiB | **射频校准 + MAC，绝不可丢** |

※ `mtd08-u-boot.bin` 取自**换 u-boot 之前**在 Meraki 诊断模式下的 dump（诊断模式里它是 `mtd1`）。
其余 12 个分区在 initramfs 下 dump，彼时 mtd8 已被 cryptid u-boot 覆盖 —— 故单独回填，
保证本目录是一份**纯出厂**镜像。

## `extras/`

| 文件 | 说明 |
|---|---|
| `spi-m25p80.bin` | 板载 SPI flash（pm25lv512，64 KiB）。**只有诊断模式能 dump**，正常模式未映射 |
| `ubi1-ART-volume.bin` | `ubi1` 上 ART 卷的逻辑内容（1.21 MiB）。`mtd12-art.bin` 是含 UBI 元数据的整分区 |
| `mtd08-u-boot-AFTER-cryptid-flash.bin` | 刷入后的 **cryptid 网络版 u-boot**，仅作留档对照，**不是**出厂件 |

## 已知的分区命名差异（踩过的坑）

Meraki 诊断模式的 `/proc/mtd` 与正常/OpenWrt 下**编号和命名都不同**：

| 诊断模式 | 正常模式 | 备注 |
|---|---|---|
| `mtd0 "cal"` | `mtd12 "art"` | **同一块**（md5 `68a42620…` 完全一致），只是叫法不同 |
| `mtd1 "u-boot"` | `mtd8 "u-boot"` | 刷 u-boot 时选错编号会写坏别的分区 |
| `mtd2 "m25p80"` | —（未映射） | SPI flash |
| `mtd3 "ART"` | ≈ `ubi1` 的 ART 卷 | 逻辑卷内容，非整分区 |

## 恢复出厂 u-boot（示例）

Meraki 诊断模式下（此时 u-boot = `mtd1`）：

```sh
echo 1 > /sys/devices/platform/msm_nand/boot_layout
flash_erase /dev/mtd1 0 0          # 用 flash_erase；本机上 `mtd erase` 会触发内核 Oops
nandwrite -pam /dev/mtd1 mtd08-u-boot.bin
echo 0 > /sys/devices/platform/msm_nand/boot_layout
```

> ⚠️ 擦写期间**只能有一个会话操作 NAND**。并发访问（如另开 telnet 跑 `nanddump` 看进度）
> 会在 `part_fill_badblockstats` 触发内核 Oops，把 NAND 控制器整个冻住，只能断电恢复。

## 免责声明

仅为本人所有设备的备份留档。`art` / `cal` 含本机唯一的射频校准与 MAC，
**不要**将其写入他人设备。

## 取证分析

见 [`FORENSICS.md`](FORENSICS.md) —— 对这份 NAND 的完整逆向：Meraki 代号映射（Yowie=MR42）、RSA 签名锁、OpenWrt 血统、免拆机后门根因、LED/EEPROM 硬件全貌。

