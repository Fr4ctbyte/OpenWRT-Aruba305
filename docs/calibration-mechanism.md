# How Aruba stores radio calibration (and how we reuse it)

## ArubaOS's mechanism

In ArubaOS, the proprietary `umac.ko` driver reads radio calibration **directly from the MTD ART partition** via custom code (`ol_transfer_bin_file()`), not through the standard Linux `request_firmware()` API. Confirmed by `dmesg` on a stock ArubaOS device:

```
[124.619] ol_transfer_bin_file: flash data file defined
[124.619] Cal location [0]: 00008000
[124.619] Wifi0 NAND FLASH Select OFFSET 0x9000
[124.638] mtd_read status:0 flash cal data len 12064 should equal EEPROM:12064
[124.638] qc98xx_verify_checksum: flash checksum passed: 0x1cdc
[124.638] Download Flash data len 12064
...
[124.696] wifi0 Select filename AP30X_boardData_AR900B_CUS239_5G_v2_001.bin
[124.696] Board Data File download to address=0xc0000
```

The filename `AP30X_boardData_AR900B_CUS239_5G_v2_001.bin` printed in the log is **purely cosmetic** — it's a logical label only. Verified by extracting the entire ArubaOS rootfs initramfs from `ArubaInstant_Ursa_8.6.0.25_90367`:

- `/lib/firmware/` is **empty**
- `/aruba/bin/` contains `athwlan_AR900B_v2.codeswap.bin` (WLAN firmware, ~148 KB) but **no `boardData_*.bin` file**
- The actual cal blob is the 12 064 bytes read from MTD ART

## Cal data layout in the ART partition

| ART offset | Size | Radio | Header checksum |
|---|---|---|---|
| `0x1000` | 12 064 B | 2.4 GHz IPQ4019 (wifi0) | `0x5b7f` |
| `0x9000` | 12 064 B | 5 GHz QCA9990 (wifi2)  | `0x1cdc` |

Both blocks use the standard Qualcomm **`qc98xx` calibration format**:

```
Offset  Bytes  Description
+0      2      Length (LE u16) = 0x2F20 = 12 064
+2      2      Checksum (LE u16) — matches ArubaOS `qc98xx_verify_checksum` log
+4      2      Version (LE u16) = 0x0101
+6      2      Flags
+8      ...    Calibration payload (12 056 bytes)
```

You can verify on your own device:

```sh
dd if=/dev/mtd10 of=/tmp/art.bin bs=1k count=64
xxd /tmp/art.bin | grep -E "^00001000:|^00009000:" | head -2
# Expected output:
#   00001000: 202f XXXX 0101 ...   ← 2.4G header (XXXX = checksum)
#   00009000: 202f XXXX 0101 ...   ← 5G header
```

Replace `/dev/mtd10` with your ART partition device — it's the SPI NOR partition at offset `0x1f0000`, 64 KiB long. Find it with `cat /proc/mtd | grep ART`.

## ath10k

The QCA `qc98xx` format **is directly consumable by ath10k upstream** via the standard `nvmem-cells` mechanism. No extraction, no conversion, no per-device file in `/lib/firmware/`. ath10k reads the blob from your device's ART at probe time.

### DTS configuration that uses this

```dts
&blsp1_spi1 {
    flash@0 {
        partitions {
            partition@1f0000 {
                label = "ART";
                reg = <0x1f0000 0x10000>;
                read-only;

                nvmem-layout {
                    compatible = "fixed-layout";
                    #address-cells = <1>;
                    #size-cells = <1>;

                    precal_art_1000: precal@1000 {
                        reg = <0x1000 0x2f20>;
                    };
                    precal_art_9000: precal@9000 {
                        reg = <0x9000 0x2f20>;
                    };
                };
            };
        };
    };
};

&wifi0 {
    nvmem-cell-names = "pre-calibration", "mac-address";
    nvmem-cells = <&precal_art_1000>, <&macaddr_mfginfo_1d 0>;
};

&pcie0 {
    bridge@0,0 {
        wifi2: wifi@1,0 {
            compatible = "qcom,ath10k";
            nvmem-cell-names = "pre-calibration", "mac-address";
            nvmem-cells = <&precal_art_9000>, <&macaddr_mfginfo_1d 2>;
        };
    };
};
```

At probe time, `ath10k_pci` (or `ath10k_ahb` for the integrated radio) reads the cal blob from the referenced nvmem cell via the `pre-calibration` consumer key and passes it to the firmware as the calibration data, same as it would for a stock QCA reference design.

## Verification on a booted OpenWrt

After flashing the AP-305 build:

```sh
dmesg | grep -E "ath10k_pci|pre-cal|nvmem"
# Look for:
#   ath10k_pci 0000:01:00.0: pre-calibration data from nvmem-cells: found
#   ath10k_pci 0000:01:00.0: board_file api 2 bmi_id 0:NN crc32 ... cal pre-cal-nvmem

cat /sys/kernel/debug/ieee80211/phy1/ath10k/cal_data | xxd | head
# Shows the actual 12064 bytes loaded as calibration
```

If you see `pre-calibration data from nvmem-cells: found` followed by a successful firmware init, the QCA9990 is running on your factory-calibrated cal.

If you see `failed to fetch board data` or `no calibration data found`, the offset / size / format may differ on your device — check that:
1. Your ART partition is at offset `0x1f0000` (you can read another path if not)
2. Your cal data is at offsets `0x1000` / `0x9000` (might differ if Aruba used different offsets on your batch)
3. The first 8 bytes of each block match `20 2f XX XX 01 01 XX XX` pattern

## No cal-*.bin required on the AP-305

> This took way too long to figure out.

The build does not include any `cal-*.bin` files in the rootfs. The DTS only tells ath10k *where to find* the cal on your device — the cal itself lives in *your* ART partition, written there at the factory.

This makes the build:
- **Portable** between AP-305 devices (each AP uses its own calibration)
- **Legal** to redistribute (no third-party RF data baked in)
- **Robust** to firmware upgrades (you don't lose your cal when reflashing)

## References

- ath10k pre-calibration mechanism: see kernel `drivers/net/wireless/ath/ath10k/core.c` function `ath10k_core_pre_cal_download`
- Example of similar approach: [`qcom-ipq4019-meraki-mr33.dts`](https://github.com/openwrt/openwrt/blob/main/target/linux/ipq40xx/files/arch/arm/boot/dts/qcom-ipq4019-meraki-mr33.dts) — Meraki MR33 uses the same idiom
- QCA `qc98xx` format: see ArubaOS `umac.ko` (proprietary), or extract via `strings` for the magic constants
