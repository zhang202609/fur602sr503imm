# HONOR FUR-602 / FUR-603 设备树（DTS）说明

## 文件

| 文件 | 适用 | 说明 |
|---|---|---|
| `mt7981b-honor-fur-602.dts` | **ImmortalWrt 25.12（内核 6.12）** | 新语法：`mt7981b.dtsi`、`bias-disable`、`MTK_DRIVE_8mA` |
| `mt7981b-honor-fur-602_24.10.dts` | ImmortalWrt 24.10（内核 6.6） | 旧语法：`mt7981.dtsi`、`mediatek,pull-up-adv`、`drive-strength = <8>` |

两份文件功能完全相同，只有内核 6.12 的 pinctrl/DTS 语法差异。按你的源码分支选对应版本。

## 放置路径

```
<源码根>/target/linux/mediatek/dts/mt7981b-honor-fur-602.dts
```

通常还需同步这几处（否则设备不出现在 menuconfig）：

1. `target/linux/mediatek/image/filogic.mk` — 追加 `honor_fur-602` 设备定义（见下方）
2. `target/linux/mediatek/filogic/base-files/etc/board.d/02_network` — LAN/WAN 接口分组加 `honor,fur-602`
3. `target/linux/mediatek/filogic/base-files/lib/upgrade/platform.sh` — 两处 case 列表（`platform_do_upgrade` / `platform_check_image`）加 `honor,fur-602`
4. `package/boot/uboot-tools/uboot-envtools/files/mediatek_filogic` — 加 `honor,fur-602`（旧树路径为 `package/boot/uboot-envtools/files/mediatek_filogic`）
5. `package/mtk/applications/mtk-smp/files/smp.sh` — 设备列表加 `honor,fur-602`（走 MT7981_whnat 分支）

filogic.mk 设备定义：

```makefile
define Device/honor_fur-602
  DEVICE_VENDOR := HONOR
  DEVICE_MODEL := FUR-602
  DEVICE_DTS := mt7981b-honor-fur-602
  DEVICE_DTS_DIR := ../dts
  SUPPORTED_DEVICES := honor,fur-602
  UBINIZE_OPTS := -E 5
  BLOCKSIZE := 128k
  PAGESIZE := 2048
  IMAGE_SIZE := 116736k
  KERNEL_IN_UBI := 1
  IMAGES += factory.bin
  IMAGE/factory.bin := append-ubi | check-size $$$$(IMAGE_SIZE)
  IMAGE/sysupgrade.bin := sysupgrade-tar | append-metadata
  KERNEL = kernel-bin | lzma | \
	fit lzma $$(KDIR)/image-$$(firstword $$(DEVICE_DTS)).dtb
  KERNEL_INITRAMFS = kernel-bin | lzma | \
	fit lzma $$(KDIR)/image-$$(firstword $$(DEVICE_DTS)).dtb with-initrd
endef
TARGET_DEVICES += honor_fur-602
```

## 硬件要点（DTS 中的关键映射）

**SoC**：MT7981（filogic），compatible = `honor,fur-602`

**LED**（低电平点亮）
- 绿灯 GPIO8 → `led-running` / `led-upgrade`
- 红灯 GPIO13 → `led-boot` / `led-failsafe`

**按键**
- reset：GPIO1 → `KEY_RESTART`
- mesh：GPIO0 → `BTN_9`（`EV_SW` 类型，注意不是普通按键域）

**网络**
- `gmac0`：2500base-x，固定链路，MAC 取自 Factory `0x2a`
- MT7531 交换机：reset GPIO39，IRQ GPIO38
- 端口映射：**port@0=WAN**（MAC 取 Factory `0x24`）、port@1=lan3、port@2=lan2、port@3=lan1、port@6=CPU 口 2500base-x

**SPI-NAND**（128MB，52MHz，4bit）
- 启用 `mediatek,nmbm`（坏块管理），`bmt-max-ratio=1`、`bmt-max-reserved-blocks=64`
- 分区表：

| 偏移 | 大小 | 标签 | 说明 |
|---|---|---|---|
| 0x000000 | 0x100000 | BL2 | 引导 |
| 0x100000 | 0x80000 | u-boot-env | 环境变量 |
| 0x180000 | 0x1e0000 | Factory | **WiFi 校准数据，只读，切勿擦除** |
| 0x360000 | 0x20000 | Trace | 只读 |
| 0x380000 | 0x200000 | FIP | 只读 |
| 0x580000 | 0x7200000 | ubi | OpenWrt 根文件系统 |

**Factory 分区 nvmem 布局**（mac-base 类型，支持偏移运算）
- `eeprom@0`（0x0，0x1000）→ WiFi EEPROM（`&wifi` 的 eeprom）
- `macaddr@a`（0xa，6 字节）→ 5G 频段 MAC（`band@1`）
- `macaddr@24`（0x24，6 字节）→ WAN MAC
- `macaddr@2a`（0x2a，6 字节）→ gmac0（LAN）MAC

**硬件加速**：`&hnat` 节点开启，`mtketh-wan = "wan"`、`mtketh-lan = "eth0"`、`mtketh-max-gmac = <1>`

## 说明

这份 DTS 是参照 25.12 树中同为 MTK 闭源 WiFi 方案的 `cmcc-a10.dtsi` 布局移植的，WiFi 部分走 mtk 闭源驱动（`nvmem-cells` 指向 Factory eeprom），**不是** mainline mt76 方案，因此不包含 `mediatek,mtd-eeprom` 那种写法，而是 `band@1` + `nvmem-cell-names = "mac-address"`。

编译验证：该 DTS 已通过 dtc 编译并由内核构建使用，无语法错误。
