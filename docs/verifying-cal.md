# Verifying calibration is read correctly

This document helps you confirm that the AP-305 OpenWrt build is reading the radio calibration from your device's ART partition correctly.

## On a booted OpenWrt

### 1. Check your version

```sh
cat /sys/firmware/devicetree/base/compatible
```

Expected: `aruba,ap-305` (possibly followed by a fallback compatible string). If you see `aruba,ap-365`, you're still on the previous build.

### 2. dmesg

```sh
dmesg | grep -E "ath10k|nvmem|pre-cal"
```

Expected (key lines):

```
ath10k_ahb a000000.wifi: qca4019 hw1.0 ...
ath10k_ahb a000000.wifi: pre-calibration data from nvmem-cells: found
ath10k_ahb a000000.wifi: firmware ver 10.4-3.x-... features ...
ath10k_ahb a000000.wifi: 10.4 wmi init: vdevs: 16 peers: 48 ...

ath10k_pci 0000:01:00.0: qca99x0 hw2.0 target 0x01000000 chip_id 0x003900ff
ath10k_pci 0000:01:00.0: pre-calibration data from nvmem-cells: found
ath10k_pci 0000:01:00.0: firmware ver 10.4-3.x-... features ...
ath10k_pci 0000:01:00.0: 10.4 wmi init: vdevs: 16 peers: 48 ...
```

Critical line for the QCA9990: `ath10k_pci 0000:01:00.0: pre-calibration data from nvmem-cells: found`.

If you see `failed to fetch board data` or `could not find pre-cal data`, the nvmem path is wrong. Check that your ART is at offset `0x1f0000` in SPI NOR (see step 3 below).

### 3. Inspect your ART manually

```sh
cat /proc/mtd | grep ART
# Should show:  mtd10: 00010000 00010000 "ART"
# (size 64 KiB, name "ART")
```

If the partition name or offset differs:

```sh
# Dump the ART:
dd if=/dev/mtd10 of=/tmp/art.bin bs=1k count=64

# Look for the cal blocks at known offsets:
xxd /tmp/art.bin | grep -E "^00001000:|^00009000:" | head -2
```

Expected output (the checksum will be different on each device, but the structure is the same):

```
00001000: 202f XXYY 0101 ...   ← length 0x2F20, checksum 0xYYXX (LE), version 1.1
00009000: 202f XXYY 0101 ...
```

If these offsets show all `ff ff ff ff` (empty), your AP-305 doesn't store cal at these offsets — possible variants:
- Cal at `0x5000` instead (= AP-365 layout, in case Aruba moved the cal during production)
- Cal at `0x9000` only (single-radio cal)
- Different magic byte sequence

In that case, adjust the `nvmem-layout` in the DTS to match what you actually see, and rebuild.

### 4. Inspect cal_data via ath10k debugfs

```sh
mount -t debugfs none /sys/kernel/debug 2>/dev/null
ls /sys/kernel/debug/ieee80211/phy1/ath10k/
xxd /sys/kernel/debug/ieee80211/phy1/ath10k/cal_data | head -4
```

The first 8 bytes should match the ART header you saw earlier (`20 2f XX YY 01 01 XX YY`). If they do, ath10k is using your cal correctly.

### 5. RF performance check

A quick sanity test (requires a client device with 5 GHz):

```sh
# On the AP, bring up a temporary AP on channel 36:
uci set wireless.radio1.disabled='0'
uci set wireless.radio1.channel='36'
uci set wireless.radio1.htmode='HE80'  # or VHT80
uci set wireless.@wifi-iface[1].disabled='0'
uci set wireless.@wifi-iface[1].ssid='AP305-test'
uci set wireless.@wifi-iface[1].encryption='none'
uci commit wireless
wifi

# On a client, connect and check signal strength:
# It should be normal (e.g., -40 to -65 dBm at close range), not severely degraded.

# Run iperf3 from a client:
iperf3 -s   # on a different machine, plugged into the AP
iperf3 -c <ap-lan-ip> -t 10 -P 4
# AP-305 with QCA9990 80 MHz 3x3 should reach ~400-700 Mbps depending on conditions
```

If signal is much weaker than expected or TX power is very low, the cal might be wrong. Try the legacy aruba_ap-365 build for comparison (without nvmem-cells, ath10k falls back on OTP cal which is generic but should still be reasonable).

## Verify or copy the cal blocks

> It's not tested yet but it should theoretically work.

If you just want to verify the cal blocks from an ART dump on your PC :

```sh
# After dd'ing /dev/mtd10 from any working OS (ArubaOS or OpenWrt):
xxd art.bin | grep "^00001000\|^00009000"
```

Both lines should start with `202f` (the length 0x2F20 in LE).

You can also extract the blocks as standalone files to verify size:

```sh
dd if=art.bin of=cal-24g.bin bs=1 skip=4096  count=12064
dd if=art.bin of=cal-5g.bin  bs=1 skip=36864 count=12064

ls -la cal-*.bin
# Both should be exactly 12064 bytes
```

These per-device files are not needed for the OpenWrt build (the build reads cal from the device's ART at runtime via nvmem-cells), but they're useful if you want to:
- Back up your calibration before any flash operation
- Restore calibration if the ART gets corrupted
- Inject the cal manually via `/lib/firmware/ath10k/cal-pci-0000:01:00.0.bin` and `cal-ahb-a000000.wifi.bin` (older OpenWrt mechanism, still works as fallback)

⚠️ **The per-device cal files contain your AP's factory radio fingerprint. Do not redistribute publicly.**
