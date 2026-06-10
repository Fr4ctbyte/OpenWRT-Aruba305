# Hardware identification — Aruba IAP-305

## SoC and radios

| Component | Detail | Driver |
|---|---|---|
| SoC | Qualcomm IPQ4029 (4× ARMv7 Cortex-A7 @ ~717 MHz) | `qcom-ipq4019` |
| 2.4 GHz radio | IPQ4019 integrated, AHB @ `a000000.wifi` (chip_id `0xb`, codename Dakota) | `ath10k_ahb` |
| 5 GHz radio | **Qualcomm QCA9990 / AR900B hw2.0**, PCIe `0000:01:00.0` (devid `168c:0040`, chip_id `0x9`, codename Beeliner) — 3x3:3 MU-MIMO | `ath10k_pci` |
| Ethernet switch | Qualcomm Atheros QCA8K | `qca8k-ipq4019` |
| Ethernet PHY | Atheros AR8035-A (PoE input) | upstream |
| Bluetooth | Texas Instruments CC2540T on UART1 (BLSP1) | none in OpenWrt — declared but unused |
| TPM | Atmel AT97SC3203 on I2C @ 0x29 | declared, no driver |
| Temp sensor | Analog Devices AD7416 on I2C @ 0x48 | `hwmon-ad7418` |
| Power monitor | Intersil ISL28022 on I2C @ 0x40 | declared, no driver |

## Memory

- **RAM**: 512 MiB DDR3
- **NAND**: 128 MiB Macronix MX30LF1G18AC (SLC, 2048-byte pages, 128 KiB erase blocks)
- **NOR SPI**: 4 MiB Macronix MX25R3235F

## Flash layout

### NAND (128 MiB) — managed by Aruba/U-Boot

| mtd | Offset | Size | Name | Notes |
|---|---|---|---|---|
| mtd0 | 0x00000000 | 32 MiB | aos0 | Primary firmware partition (ArubaOS kernel+rootfs OR your OpenWrt) |
| mtd1 | 0x02000000 | 32 MiB | ubi (= aos1) | Secondary firmware partition (use this for OpenWrt sysupgrade) |
| mtd2 | 0x04000000 | 64 MiB | aruba-ubifs | Data partition (read-only by default in upstream OpenWrt DTS) |

### SPI NOR (4 MiB)

| mtd | Offset | Size | Name | Notes |
|---|---|---|---|---|
| mtd3 | 0x000000 | 256 KiB | sbl1 | Secondary boot loader (read-only) |
| mtd4 | 0x040000 | 128 KiB | mibib | MIBI boot (read-only) |
| mtd5 | 0x060000 | 384 KiB | qsee | Qualcomm Secure Execution Environment (read-only) |
| mtd6 | 0x0c0000 | 64 KiB | cdt | Config Data Table (read-only) |
| mtd7 | 0x0d0000 | 64 KiB | ddrparams | DDR timing parameters (read-only) |
| mtd8 | 0x0e0000 | 64 KiB | u-boot-env | APBoot environment (writable) |
| mtd9 | 0x0f0000 | 1024 KiB | appsbl | APBoot binary (read-only) |
| **mtd10** | **0x1f0000** | **64 KiB** | **ART** | **Radio calibration data** ⭐ |
| mtd11 | 0x200000 | 1.5 MiB | osss | OS Subset Storage (read-only) |
| mtd12 | 0x370000 | 64 KiB | pds | Persistent Device Settings (read-only) |
| mtd13 | 0x380000 | 64 KiB | apcd | AP Config Data (read-only) |
| mtd14 | 0x390000 | 64 KiB | mfginfo | Manufacturing info (MAC, serial) (read-only) |
| mtd15 | 0x3a0000 | 64 KiB | fcache | Flash cache (read-only) |
| mtd16 | 0x3b0000 | 320 KiB | osss1 | OS Subset Storage backup (read-only) |

## GPIO summary

Note: GPIO/LED mapping **Still Testing 2026-06-08** 

GPIOs of interest on the IPQ4019 TLMM controller:

| Pin | Direction | Purpose | Notes |
|---|---|---|---|
| 3 | out | **Hardware watchdog poke** (active-low toggle) | Differs from AP-365 (pin 41). Required. **Never toggle** |
| 6 / 7 | mux | MDIO / MDC | |
| 8 / 9 | mux | UART1 (BLE radio) | |
| 10 / 11 | mux | I2C0 (TPM, sensors) | |
| 12 | out | SPI0 chip-select (NOR flash) | |
| 13–15 | mux | SPI0 (NOR flash) | |
| 16 / 17 | mux | UART0 (console) | 9600n8 |
| 35 | out | **WD_LATCH_CLR_L** (watchdog) | **Never toggle** |
| 38 | out | PCIe PERST# (active low) | Reset for QCA9990 |
| 39 | out | "reset watchdog status flipflop" (Aruba) | driving it → **instant reboot** |
| 40 | out | "enable watchdog" (Aruba) | driving it → **instant reboot** |
| 41 | (legacy) | Watchdog poke on AP-365 — NOT on AP-305 | Don't use on AP-305 |
| 42 | out | **System LED amber** (active-high) | was wrongly labeled phy-reset & hogged → THE stuck-amber cause |
| 46 | — | **unused** on AP-305 | was wrongly labeled "system red" (toggling = no effect) |
| 47 | out | **PHY reset/enable** (active-high, held high) | the real phy-reset; driving low → **LAN drops** |
| 49 | — | **unused** on AP-305 | was wrongly labeled "system amber" (no effect) |
| 50 | in | PCIe wake / Reset button (dual purpose) | |
| 51 | out | **Wi-Fi LED green** (active-high) | |
| 52 | — | unused (no visible LED) | |
| 53–69 | mux | NAND pins | group over-broad: also lists 61 & 68, which are LEDs |
| 61 | out | **Wi-Fi LED amber** (active-high)  | leds-gpio reclaims it from the NAND group |
| 68 | out | **System LED green** (active-high) | reclaim from NAND group; verify on first boot |

## Bootloader

- **APBoot 2.4.0.8** (build 64221), built 2018-03-28
- U-Boot fork by Aruba — has custom commands like `netget` (TFTP fetch) and `apenv_backup`
- Default env: `boardname=Glenmorangie`, `bootcmd=run nandboot_openwrt`
- Console: 9600 8N1 on UART0 (`ttyMSM1` from Linux side)

## sysfs GPIO base mapping

Kernel 6.6 (OpenWrt 24.10.2) puts the TLMM gpiochip base at **512**. So:

| Sysfs path | TLMM offset | Pin role (this device) |
|---|---|---|
| `/sys/class/gpio/gpio515` | 3 | Watchdog poke |
| `/sys/class/gpio/gpio550` | 38 | PCIe PERST# |
| `/sys/class/gpio/gpio562` | 50 | Reset / PCIe wake |
| `/sys/class/gpio/gpio554` | 42 | System LED amber |
| `/sys/class/gpio/gpio580` | 68 | System LED green (inferred) |
| `/sys/class/gpio/gpio563` | 51 | Wi-Fi LED green |
| `/sys/class/gpio/gpio573` | 61 | Wi-Fi LED amber |

To map a desired TLMM pin to its sysfs ID: `sysfs_id = gpiochip_base + tlmm_offset`. The base depends on kernel version; verify with `cat /sys/class/gpio/gpiochip*/base`.
