# OpenWrt support for Aruba IAP-305

Patches and prebuilt images to run **OpenWrt 24.10.2** on the **Aruba Instant AP-305** (IAP-305-RW), with both radios functional including the external 5 GHz Qualcomm QCA9990.

> **Notes**
> - Community port, not an official OpenWrt target — **use at your own risk**.
> - I haven't submitted to the OpenWRT project. Treat it as documentation and sharing, not a maintained fork.
> - Work in progress: I've been tinkering with this AP for over a year.
> - Expect some bugs. I've tested a lot, but a few things still need polish, and I might be wrong on some non-critical details.

## Hardware

| Component | Detail |
|---|---|
| Model | Aruba IAP-305-RW |
| Internal codename | Glenmorangie / AP-30x family |
| SoC | Qualcomm IPQ4029 (ARMv7 quad-core Cortex-A7) |
| 2.4 GHz radio | IPQ4019 integrated, AHB `a000000.wifi` (chip_id `0xb`) |
| 5 GHz radio | **Qualcomm QCA9990 / AR900B hw2.0** on PCIe `0000:01:00.0` (devid `168c:0040`, chip_id `0x9`) |
| RAM | 512 MiB |
| NAND | 128 MiB (Macronix MX30LF1G18AC) |
| SPI NOR | 4 MiB (Macronix MX25R3235F) |
| Bootloader | Aruba APBoot 2.4.0.8 (U-Boot fork) |

The key distinction vs the IAP-303 (which uses both IPQ4019 internal radios) is the **external QCA9990 PCIe 5 GHz radio** — the same topology as the Meraki MR42.

## What works

- Boot from NAND (kernel 6.6.93)
- Ethernet (gigabit via QCA8K switch)
- 2.4 GHz Wi-Fi (IPQ4019 AHB, phy0/wlan0) with factory calibration
- **5 GHz Wi-Fi (QCA9990 PCIe, phy1/wlan1) with factory calibration**
- Hardware watchdog (GPIO-toggle, TLMM 3)
- TPM (Atmel AT97SC3203, declared but no driver)
- Temperature sensor (AD7416)
- Power monitor (ISL28022, declared but no driver)

## Quick install

This assumes OpenWrt is already running on the AP (any flavor: `aruba_ap-303`, `aruba_ap-365`, or an earlier build of this one).

### First time on a stock ArubaOS AP

Boot the **`aruba_ap-303`** image first. The OpenWrt wiki has a [working install procedure for AP-303](https://openwrt.org/toh/aruba/ap-303), and the same procedure applies to the AP-305 (same Glenmorangie codename, same bootloader, same NAND layout). Don't worry that 5 GHz won't come up and the watchdog will reboot you every ~60 s.

You need to tickle the watchdog a little to keep the AP from going into a reboot loop:

```sh
echo 515 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio515/direction
while true; do
  echo 0 > /sys/class/gpio/gpio515/value; sleep 1
  echo 1 > /sys/class/gpio/gpio515/value; sleep 1
done &
```

If the script fails on `export` (Device or resource busy), the sysfs base may differ on your kernel — see [docs/hardware.md](docs/hardware.md#sysfs-gpio-base-mapping) for the math.

### From any running OpenWrt → flash the AP-305 image

```sh
wget https://github.com/Fr4ctbyte/OpenWRT-Aruba305/raw/main/images/openwrt-24.10.2-aruba_ap-305-squashfs-sysupgrade.bin -O /tmp/sysupgrade.bin
sysupgrade -F -n /tmp/sysupgrade.bin
# -F : required because the compatible string changes (e.g. aruba,ap-365 → aruba,ap-305)
# -n : do not preserve config (target differs)
```

## Verification after reboot

```sh
cat /sys/firmware/devicetree/base/compatible
# Expected: aruba,ap-305

dmesg | grep -E "ath10k_pci|qca99x0|wifi[012]"
# Expected:
#   ath10k_pci 0000:01:00.0: qca99x0 hw2.0 target 0x01000000 ...
#   ath10k_pci 0000:01:00.0: pre-calibration data from nvmem-cells: found
#   ath10k_pci 0000:01:00.0: 10.4 wmi init: vdevs: 16 peers: 48 ...

iw phy
# Should list phy0 (2.4 GHz) and phy1 (5 GHz)
```

## Repository layout

```
.
├── README.md                       this file
├── LICENSE                         GPL-2.0-only (same as OpenWrt)
├── patches/
│   ├── 0001-aruba-ap-305-support.patch  combined patch (apply with `git am` or `git apply`)
│   └── qcom-ipq4029-ap-305.dts          standalone DTS for reference
├── images/                         prebuilt images for direct flash
│   ├── openwrt-24.10.2-aruba_ap-305-squashfs-sysupgrade.bin
│   ├── openwrt-24.10.2-aruba_ap-305-initramfs-uImage.itb
│   └── packages.manifest                list of included packages
└── docs/
    ├── hardware.md                 detailed hardware identification
    ├── calibration-mechanism.md    how Aruba stores radio calibration in ART
    ├── installation.md             flashing procedure (long form)
    └── verifying-cal.md            how to verify your device's calibration is read
```

## Build from source

If you prefer building yourself (recommended for security — why would you trust a random repo on the internet anyway?):

```sh
git clone --branch v24.10.2 https://github.com/openwrt/openwrt.git
cd openwrt
./scripts/feeds update -a && ./scripts/feeds install -a

# Apply this patch:
git apply /path/to/patches/0001-aruba-ap-305-support.patch

# Configure for AP-305:
make defconfig
echo "CONFIG_TARGET_PROFILE=\"DEVICE_aruba_ap-305\"" >> .config
echo "CONFIG_TARGET_ipq40xx_generic_DEVICE_aruba_ap-305=y" >> .config
make defconfig

# Build:
make -j$(nproc)

# Output:
ls bin/targets/ipq40xx/generic/openwrt-*-aruba_ap-305-*
```

## Key DTS modifications

The patch creates `qcom-ipq4029-ap-305.dts` derived from `qcom-ipq4029-ap-365.dts` with three changes:

### 1. Watchdog GPIO

On the AP-365, OpenWrt uses `<&tlmm 41>`. On the AP-305 the watchdog flip-flop is wired to **`<&tlmm 3>`**. Without this fix the board reboots every ~60 s.

### 2. wifi1 disabled, PCIe enabled

The AP-305 does not wire the IPQ4019's integrated 5 GHz radio (`a800000.wifi`). 5 GHz comes from the external QCA9990 on PCIe instead. So:

- `&wifi1 { status = "disabled"; };`
- `&pcie0 { status = "okay"; ... wifi2: wifi@1,0 { ... }; };`

### 3. Calibration via `nvmem-cells`

Aruba's proprietary driver reads cal data **directly from the MTD ART partition** at fixed offsets:

| Radio | ART offset | Size | Format |
|---|---|---|---|
| 2.4 GHz (IPQ4019) | `0x1000` | 12 064 B | QCA `qc98xx` |
| 5 GHz (QCA9990) | `0x9000` | 12 064 B | QCA `qc98xx` |

ath10k supports the exact same format via `nvmem-cells`, with no extraction or per-device file needed. The patch adds the appropriate `precal_art_*` cells to the ART partition's `nvmem-layout` and references them from the wifi nodes.

See [docs/calibration-mechanism.md](docs/calibration-mechanism.md) for the full reverse-engineering walkthrough.

## Important caveats

- **The calibration is per-device.** Each AP-305 has its own factory-calibrated cal data in its own ART. The build itself contains no calibration — it only declares the nvmem-cell paths. ath10k reads the cal from your device's ART at boot. **Don't redistribute another device's ART dump.**
- **Watchdog GPIO 3 is empirical.** Confirmed working on my IAP-305-RW. If your AP reboots every minute despite this fix, the wiring may differ — try toggling other TLMM pins from userspace to find the right one.
- **Compatible string change** requires `sysupgrade -F` at first flash.
- **LEDs**: GPIOs 46/49/61 (red/amber/green system) are inherited from the AP-365 DTS. If some LEDs stay dark or show the wrong color on your AP-305, adjust the `leds` block in the DTS. Mine ends up orange — I plan to fix it someday :)

## Credits

- Forum thread that started this: https://forum.openwrt.org/t/openwrt-support-for-aruba-ap-305/63218
- Existing AP-303 / AP-365 OpenWrt support (the base): work from the OpenWrt community
- ath10k & ath10k-ct projects for the Wi-Fi stack
- Thanks to idovitz for the GPIO findings on the IAP-305

## License

GPL-2.0-only, matching the OpenWrt project license. See [LICENSE](LICENSE).

