# Zyxel EX5512-T0 OpenWrt Port Notes

Notes, patches, and install steps for running OpenWrt 24.10 on the Zyxel EX5512-T0.

The EX5512-T0 is a MediaTek MT7986A (Filogic 830) box closely related to the retail EX5601-T0, but with 512MB RAM instead of 1GB and a few wiring differences that will break things if you try running a vanilla EX5601 build.

Tested and working:
- Wi-Fi 6 (2.4GHz + 5GHz 4x4) via mt7915e with stock calibration
- 2.5G WAN (Realtek RTL8221B)
- 2.5G LAN (Realtek RTL8221B via MT7531 port 5)
- 3x 1G LAN (MT7531 switch)
- USB 3.0
- Front panel LEDs and buttons
- Stock dual-boot (Bank 1) and OpenWrt U-Boot layout (~400MB overlay)

---

## Hardware Quirks (EX5512 vs EX5601)

If you flash a generic EX5601 image, the router will boot but half the hardware will be broken:

1. **WAN PHY MDIO address:**
   The EX5601 has its Realtek RTL8221B WAN PHY at MDIO address 6. On the EX5512, Zyxel wired it to **MDIO address 2**. On an EX5601 build, the kernel throws `could not attach PHY: -22` and the blue 2.5G WAN port is dead.

2. **LEDs & VoIP pin clash:**
   On the EX5601, GPIO 30 and 31 are hooked up to an SPI bus for a Silicon Labs VoIP SLIC chip. The EX5512 has no VoIP hardware populated, and Zyxel rewired those pins to the front Internet LED (GPIO 30 green, GPIO 31 red). The EX5601 Internet LED was on GPIO 14, which is physically the WPS LED on the EX5512.
   The VoIP SPI bus must be disabled in DTS to free GPIO 30/31, otherwise the Internet LED will not work.

3. **2.5G RJ45 jack LEDs:**
   The RTL8221B PHYs control their own integrated jack LEDs over MDIO clause 45 registers. The kernel driver does not initialize these by default, so the link/activity LEDs on both 2.5G ports stay off unless you poke registers `0xd040`, `0xd036`, and `0xd044`. See `scripts/port_leds` for the init script.

---

## Step 1: Backup Your Calibrations (Do NOT skip this)

Before touching anything, SSH into the stock firmware as `supervisor` and back up your `Factory` and `ubootenv` partitions. `Factory` (mtd3) holds your Wi-Fi RF calibration data and MAC addresses. If you erase it without a backup, your Wi-Fi range and throughput will be permanently degraded.

```bash
cat /proc/mtd
dd if=/dev/mtd3 of=/tmp/Factory.bin
dd if=/dev/mtd2 of=/tmp/ubootenv.bin
```

Pull them to your PC:
```bash
scp supervisor@192.168.1.1:/tmp/Factory.bin .
```

---

## Method 1: Stock Dual-Boot (Safest)

This keeps the stock bootloader intact and flashes OpenWrt into Bank 1 (`mtd6`), leaving OEM firmware safe on Bank 2 (`mtd7`).

1. Copy `openwrt-*-stock-kernel.itb` and `openwrt-*-stock-root.squashfs` to `/tmp/`.
2. Attach Bank 1 and flash the volumes:
   ```bash
   ubiattach -m 6 /dev/ubi_ctrl
   ubiupdatevol /dev/ubi0_0 /tmp/openwrt-24.10.8-mediatek-filogic-zyxel_ex5512-t0-stock-kernel.itb
   ubiupdatevol /dev/ubi0_1 /tmp/openwrt-24.10.8-mediatek-filogic-zyxel_ex5512-t0-stock-root.squashfs
   ```
3. Set the bootloader to Bank 1 and reboot:
   ```bash
   fw_setenv boot_bank 1
   sync
   reboot
   ```

If you ever want to go back to stock, set `boot_bank 2`.

---

## Method 2: U-Boot Mod (Unlocks 400MB Overlay)

Replaces OEM BL2 and FIP with OpenWrt U-Boot. This removes the OEM dual-bank split and unifies `mtd5` into a single 474MB UBI partition, giving you roughly 382MB of free `/overlay` space.

Make sure you have a 3.3V USB-TTL serial cable handy in case anything goes sideways.

### 1. Flash the bootloader partitions
Copy `mtd-rw.ko`, `...preloader.bin`, and `...bl31-uboot.fip` to `/tmp/`:
```bash
cd /tmp
insmod mtd-rw.ko i_want_a_brick=1
mtd write openwrt-24.10.8-mediatek-filogic-zyxel_ex5512-t0-ubootmod-preloader.bin BL2
mtd verify openwrt-24.10.8-mediatek-filogic-zyxel_ex5512-t0-ubootmod-preloader.bin BL2
mtd write openwrt-24.10.8-mediatek-filogic-zyxel_ex5512-t0-ubootmod-bl31-uboot.fip FIP
mtd verify openwrt-24.10.8-mediatek-filogic-zyxel_ex5512-t0-ubootmod-bl31-uboot.fip FIP
sync
reboot
```

### 2. TFTP Recovery & Sysupgrade
After rebooting, U-Boot looks for a TFTP server at `192.168.1.254` on **LAN 2**:
1. Set your PC Ethernet to `192.168.1.254/24` plugged into LAN 2.
2. Host `openwrt-...-ubootmod-initramfs-recovery.itb` on your TFTP server.
3. Once booted into RAM OpenWrt (`192.168.1.1`), SSH in and format the unified UBI container:
   ```bash
   ubidetach -p /dev/mtd5 2>/dev/null || true
   ubiformat /dev/mtd5 -y
   ubiattach -p /dev/mtd5
   ubimkvol /dev/ubi0 -n 0 -N ubootenv -s 124KiB
   ubimkvol /dev/ubi0 -n 1 -N ubootenv2 -s 124KiB
   ubimkvol /dev/ubi0 -n 2 -N recovery -s 20MiB

   sysupgrade -n /tmp/openwrt-24.10.8-mediatek-filogic-zyxel_ex5512-t0-ubootmod-squashfs-sysupgrade.itb
   ```

---

## 2.5G Port LEDs

To get the rear RJ45 jack LEDs working on WAN and LAN 4, install `mdio-tools` and run:
```bash
# 2.5G WAN (PHY 2)
mdio mdio-bus mmd 2:0x1f raw 0xd040 0x0e00
mdio mdio-bus mmd 2:0x1f raw 0xd036 0x0027
mdio mdio-bus mmd 2:0x1f raw 0xd044 0x0040

# 2.5G LAN 4 (PHY 5)
mdio mdio-bus mmd 5:0x1f raw 0xd040 0x0e04
mdio mdio-bus mmd 5:0x1f raw 0xd036 0x0027
mdio mdio-bus mmd 5:0x1f raw 0xd044 0x0040
```
Copy `scripts/port_leds` to `/etc/init.d/port_leds` and run `/etc/init.d/port_leds enable` so it runs on boot.

---

## Patches

`patches/0001-target-mediatek-filogic-add-Zyxel-EX5512-T0-support.patch` is a clean, self-contained git patch against OpenWrt 24.10. It adds:
- `mt7986a-zyxel-ex5512-t0-stock.dts`
- `mt7986a-zyxel-ex5512-t0-ubootmod.dts`
- Board detection for first-boot network and LED setup
- `platform.sh` sysupgrade support
- Image build targets in `filogic.mk`
