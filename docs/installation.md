# Installation guide — Aruba IAP-305 → OpenWrt 24.10.2

## Prerequisites

- Aruba IAP-305-RW (or compatible variant)
- USB-TTL serial cable, 3.3V (3 wires: GND, RX, TX) — for console access during the initial install
- A TFTP server reachable from the same VLAN as the AP
- Basic familiarity with `scp`, `wget`, and OpenWrt's `sysupgrade`

## Step 1 — Open serial console

Open the AP case. The serial header is a 4-pin TTL on the PCB (search the OpenWrt wiki for your specific board photo). Pinout:

- GND
- TX (from AP, to your USB-TTL RX)
- RX (from AP, to your USB-TTL TX)
- Don't connect VCC unless you know what you're doing

Connect at **9600 8N1** (e.g., `screen /dev/ttyUSB0 9600` or PuTTY → Serial).

Power the AP — you should see APBoot output. Hit Enter to get a prompt:

```
apboot>
```

## Step 2 — Configure U-Boot for TFTP

On a fresh AP, set up the ramboot variable:

```
apboot> dhcp
apboot> setenv ramboot_openwrt "setenv ipaddr <ap-ip>; setenv serverip <tftp-server-ip>; netget; set fdt_high 0x87000000; bootm"
apboot> setenv bootfile ipq40xx.ari
apboot> setenv bootcmd "run ramboot_openwrt"
apboot> saveenv
```

Replace `<ap-ip>` with the static IP you want the AP to use (e.g., `192.168.1.250`), and `<tftp-server-ip>` with your TFTP server's IP. Both must be on the same subnet. The `bootfile` is the filename APBoot will TFTP — `ipq40xx.ari` is convention.

## Step 3 — TFTP-boot a bootstrap OpenWrt

The goal here is just to get *some* OpenWrt running on the AP — any flavor — so you can use `sysupgrade` in step 5. The bootstrap image doesn't need to support the QCA9990 5 GHz radio: Ethernet and `sysupgrade` are enough.

The easiest bootstrap is the upstream **`aruba_ap-303`** initramfs (same Glenmorangie codename, same bootloader, same NAND layout — the install procedure documented in the [OpenWrt AP-303 wiki](https://openwrt.org/toh/aruba/ap-303) works on the AP-305 too).

On your TFTP server, place the bootstrap initramfs as the name APBoot will fetch:

```sh
cp openwrt-...-aruba_ap-303-initramfs-uImage.itb /srv/tftp/ipq40xx.ari
```

Then in APBoot:

```
apboot> run ramboot_openwrt
```

The AP boots OpenWrt entirely in RAM, NAND untouched. The 5 GHz won't come up and the watchdog will likely reboot you every ~60 s — that's expected at this stage, and harmless.

If ramboot doesn't work for some reason, you can also write a sysupgrade directly to NAND from APBoot (riskier, requires manual size handling):

```
apboot> tftpboot 0x84000000 openwrt-...-aruba_ap-303-squashfs-sysupgrade.bin
apboot> mtdparts default
apboot> nand erase.part ubi
apboot> nand write 0x84000000 ubi <filesize>
```

## Step 4 — Once OpenWrt is running

Default OpenWrt has SSH disabled until you set a root password (it only listens on LAN). Connect via Ethernet, browse to `http://192.168.1.1` (the default OpenWrt IP), and set a password.

Ethernet should be up. Depending on which OpenWrt bootstrap image you used, 5 GHz may or may not be functional yet — but it doesn't matter, the next step replaces the firmware.

## Step 5 — Flash the AP-305 specific image

```sh
# Transfer the AP-305 sysupgrade.bin:
scp images/openwrt-24.10.2-aruba_ap-305-squashfs-sysupgrade.bin root@192.168.1.1:/tmp/sysupgrade.bin
ssh root@192.168.1.1

# Verify checksum (computed at release time, see SHA256SUMS in /images):
sha256sum /tmp/sysupgrade.bin

# Flash:
sysupgrade -F -n /tmp/sysupgrade.bin
# -F : force, required because compatible string changes (aruba,ap-365 → aruba,ap-305)
# -n : do not preserve config (target differs)
```

The AP reboots after ~30 s. Wait 2 full minutes before reconnecting.

## Step 6 — Configure U-Boot for permanent NAND boot

```
apboot> setenv bootargs console=ttyMSM1,9600n8
apboot> setenv nandboot_openwrt "ubi part aos1; ubi read 0x85000000 kernel; set fdt_high 0x87000000; bootm 0x85000000"
apboot> setenv bootcmd "run nandboot_openwrt"
apboot> saveenv
```

Now power-cycle the AP. It should boot straight into the new AP-305 OpenWrt.

## Step 7 — Verify everything works

After reboot, via serial console or ssh:

```sh
# Confirm we're running the AP-305 image:
cat /sys/firmware/devicetree/base/compatible
# Expected: aruba,ap-305

# Confirm both radios are up:
iw phy
# phy0 (2.4 GHz) and phy1 (5 GHz)

# Confirm cal data is being read from ART:
dmesg | grep "pre-calibration data"
# Expected: 2 lines, one for AHB (2.4G), one for PCIe (5G)
# Each should say: "pre-calibration data from nvmem-cells: found"

# Test wireless:
iw dev wlan0 info     # 2.4G
iw dev wlan1 info     # 5G QCA9990
```

## Recovery

If the AP doesn't come back after flashing, or you lose network access:

1. Power off
2. Connect the serial console
3. Power on and interrupt the boot to land in APBoot (Ctrl+C at the prompt)
4. Recover via ramboot with a known-good image:

   ```
   apboot> setenv bootfile recovery.ari
   apboot> run ramboot_openwrt
   ```

5. Once OpenWrt is back up in RAM, flash whatever image you want to NAND.

## Hardware watchdog warning

The AP-305 has a hardware watchdog wired to **GPIO TLMM 3** that resets the board every ~60 s if not kicked. The DTS in this build configures Linux's `gpio-wdt` driver to kick it automatically (`hw_margin_ms = 1000`).

If your AP reboots periodically anyway, the pin might be different on your specific batch — see [docs/hardware.md](hardware.md) for alternatives (try TLMM 41, the AP-365 default, as fallback).
