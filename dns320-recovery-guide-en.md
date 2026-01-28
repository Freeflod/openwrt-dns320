# DNS-320 Rev A - Unbricking and OpenWrt Flash Guide

## Required Hardware

- D-Link DNS-320 Rev A
- Serial console (UART) on JP4: TX, RX, GND (3.3V, 115200 baud)
- USB-TTL adapter (FTDI FT232 or equivalent)
- USB stick formatted in **FAT32** with MBR partition table
- Linux PC with `u-boot-tools` and `tio` installed

### Serial Console Access

```bash
# Install tio
sudo apt install tio

# Connect to NAS
tio /dev/ttyUSB0

# (baudrate 115200 is auto-detected)
# To quit: CTRL+T then Q
```

**Cable test:** When booting the NAS, you should see text scrolling. If nothing appears, check:
- TX/RX swapped (NAS TX → adapter RX)
- GND connected
- Adapter set to 3.3V (not 5V!)

## Required Files

Download from [APCCV/openwrt-dns320](https://github.com/APCCV/openwrt-dns320/releases):
- `u-boot.kwb`
- `env.bin`
- `ubi.bin`

---

## ⚠️ How I Bricked the NAS

**Ignored "No partition table" error**

```bash
=> fatload usb 0:1 0x1000000 u-boot.kwb
** No partition table - usb 0 **
```

❌ **I ignored the error and continued** → USB stick not properly recognized, file not loaded.

**Possible cause:** Partition created without MBR table, or incompatible USB stick.

**Erased NAND without verification**

```bash
=> nand erase 0x0 0x0e0000
=> nand write 0x1000000 0x0 0x0e0000
```

No error displayed → I thought it worked.  
**Result:** NAND erased with nothing in it → **NAS bricked** 🧱

**Lesson:** Always check `printenv filesize` after `fatload`!

---

## 🛠️ Recovery via kwboot

**Brick symptom:** Serial console completely silent on boot.

### Procedure

```bash
# In one terminal: serial connection
tio /dev/ttyUSB0

# In another terminal: kwboot
sudo apt install u-boot-tools

# Unplug NAS power

# Start kwboot
sudo kwboot -t -B 115200 /dev/ttyUSB0 -b u-boot.kwb

# Plug power back in while kwboot is running
```

**Note:** During kwboot, `tio` will display weird characters (normal, it's the transfer). Wait for completion.

### ✅ Expected result

```
Sending boot message. Please reboot the target...
Sending boot image...
[=====================] 100%
Done finishing transfer
```

Then u-boot prompt appears: `=>`

**If you see this** → u-boot loaded in RAM (temporary).  
**If nothing** → serial wiring issue or bad u-boot.kwb file.

---

## ✅ Permanent Installation

### USB Stick Preparation

**FAT32 formatting:**

```bash
# Create MBR partition table
sudo fdisk /dev/sdX
# o (MBR table), n (new partition), p (primary), 1, ENTER, ENTER
# t, c (FAT32 LBA), w (write)

# Format as FAT32
sudo mkfs.vfat -F 32 /dev/sdX1

# Mount and copy files
sudo mount /dev/sdX1 /mnt
cp u-boot.kwb env.bin ubi.bin /mnt/
sudo umount /mnt
```

---

## Step 1: Flash Permanent u-boot

**In u-boot:**

```bash
usb reset
usb storage
```

### ✅ Expected result

```
Bus ehci@50000: USB EHCI 1.00
scanning bus ehci@50000 for devices... 2 USB Device(s) found
       scanning usb for storage devices... 1 Storage Device(s) found

  Device 0: Vendor: ... Prod: ... Rev: ...
            Type: Removable Hard Disk
            Capacity: XXXX MB
```

**If you see this** → USB stick detected.  
**If "0 Storage Device(s) found"** → unplug/replug stick, retry `usb reset`.

---

```bash
fatload usb 0:1 0x1000000 u-boot.kwb
```

### ✅ Expected result

```
590156 bytes read in 45 ms (12.5 MiB/s)
```

**If you see this** → file loaded OK.  
**If "No partition table" or timeout** → unplug/replug USB stick, `usb reset`, retry.

---

```bash
printenv filesize
```

### ✅ Expected result

```
filesize=90156
```

**Size in decimal:** 590156 bytes (matches u-boot.kwb file size)

**If empty or 0** → file NOT loaded, do NOT continue!  
**If different value** → file partially loaded or corrupted, retry.

---

```bash
nand erase 0x0 0x0e0000
nand write 0x1000000 0x0 0x0e0000
saveenv
reset
```

### ✅ Expected result after reset

NAS reboots and displays:

```
U-Boot 2020.04 (Dec 17 2025 - 21:08:22 +0000)
D-Link DNS-320
...
Hit any key to stop autoboot:
=>
```

**If you see this WITHOUT kwboot** → permanent u-boot OK! 🎉  
**If nothing** → u-boot not flashed, start over with kwboot.

---

## Step 2: Load APCCV Environment

```bash
usb reset
fatload usb 0:1 0x1000000 env.bin
env import -b 0x1000000 ${filesize}
```

### ✅ Expected result

```
[several lines of imported variables]
```

**If error** → env.bin file corrupted or not loaded.

---

```bash
setenv ethaddr 'XX:XX:XX:XX:XX:XX'  # Your MAC
printenv mtdparts
```

### ✅ Expected result

```
mtdparts=orion_nand:0x0e0000@0x0(uboot),0x20000@0x0e0000(ubootenv),0x7f00000@0x100000(ubi)
```

**If you see this** → environment correct.  
**If different** → env.bin loaded incorrectly, retry.

---

```bash
saveenv
```

---

## Step 3: Flash OpenWrt (ubi.bin)

```bash
nand erase 0x100000 0x7f00000
```

### ✅ Expected result

```
NAND erase: device 0 offset 0x100000, size 0x7f00000
Skipping bad block at  0xXXXXXXXX  (may appear)
Erasing at 0x7fe0000 -- 100% complete.
OK
```

**If you see this** → UBI partition erased.  
**If error** → hardware NAND issue (rare).

---

```bash
usb reset
fatload usb 0:1 0x1000000 ubi.bin
```

### ✅ Expected result

```
11010048 bytes read in 456 ms (23 MiB/s)
```

**If you see this** → file loaded.  
**If timeout `EHCI timed out on TD`** → unplug/replug USB stick, `usb reset`, retry (may take 2-3 attempts).

---

```bash
printenv filesize
```

### ✅ Expected result

```
filesize=a80000
```

**Size in decimal:** 11010048 bytes (matches ubi.bin file size)

**If empty** → not loaded, retry!  
**If different value** → file partially loaded, retry `fatload`.

---

```bash
nand write 0x1000000 0x100000 ${filesize}
```

### ✅ Expected result

```
NAND write: device 0 offset 0x100000, size 0xa80000
 11010048 bytes written: OK
```

**If you see this** → OpenWrt flashed! 🎉  
**If error** → NAND issue.

---

```bash
reset
```

---

## Successful OpenWrt Boot

### ✅ Expected result

```
U-Boot 2020.04 (Dec 17 2025 - 21:08:22 +0000)
...
ubi0: attaching mtd2
ubi0: scanning is finished
ubi0: attached mtd2 (name "ubi", size 127 MiB)
...
[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 6.6.x ...
...

Please press Enter to activate this console
login[758]: root login on 'ttyS0'


BusyBox v1.36.1 (2025-12-17 21:08:22 UTC) built-in shell (ash)

  _______                     ________        __
 |       |.-----.-----.-----.|  |  |  |.----.|  |_
 |   -   ||  _  |  -__|     ||  |  |  ||   _||   _|
 |_______||   __|_____|__|__||________||__|  |____|
          |__| W I R E L E S S   F R E E D O M
 -----------------------------------------------------
 OpenWrt 24.10.5, r29087-d9c5716d1d
 -----------------------------------------------------

root@(none):~# 
```

**If you see this** → ✅ **Installation successful!** 🎉

**Login:**
Next, Openwrt invite you to change root's password. **Do it!**



---

## ❌ Possible Issues

### "No partition table" with FAT32

```bash
=> fatload usb 0:1 0x1000000 u-boot.kwb
** No partition table - usb 0 **
```

**Cause:** USB stick without MBR partition table, or directly formatted partition.  
**Solution:** Recreate stick with fdisk (see USB Stick Preparation section):
- Create MBR table (`o` in fdisk)
- Create primary partition (`n`, `p`, `1`)
- Set FAT32 LBA type (`t`, `c`)
- Format (`mkfs.vfat -F 32`)

After this, `fatload` should work.

### "ubi0 error: the layout volume was not found"

```
ubi0 error: the layout volume was not found
Kernel panic - not syncing: VFS: Unable to mount root fs
```

**Cause:** Wrong `bootargs` (incorrect ubi.mtd).  
**Solution:** Reload `env.bin` with `env import`.

### Repeated USB timeouts

```
EHCI timed out on TD - token=0x...
```

**Cause:** Combination of USB stick + old Kirkwood controller.  
**Solution:** Unplug/replug stick, `usb reset`, retry (patience!).

### Nothing on boot after flash

**Cause:** u-boot or ubi.bin incorrectly flashed.  
**Solution:** Start over with kwboot from beginning.

---

## 📋 Key Points Summary

### ✅ What works

- **FAT32 works** with `fatload` (no need for ext4)
- **Check `filesize`** after each `fatload` (crucial!)
  - The `filesize` value must **match exactly** the loaded file size
  - If different → file partially loaded or corrupted
- **Serial console essential** for recovery
- **APCCV's env.bin** avoids configuration errors
- **Retry USB loads** if timeout (normal on old hardware)

### ❌ To avoid

- Ignoring `No partition table` → create MBR table with fdisk
- Erasing NAND without checking `filesize`
- Not verifying file is loaded before flashing

---

## Final Specifications

**Software:**
- U-Boot: 2020.04 (APCCV)
- OpenWrt: 24.10.5-r1
- Kernel: Linux 6.6.x LTS

**NAND Partitioning:**
```
0x000000 - 0x0e0000 : u-boot     (896KB)
0x0e0000 - 0x100000 : u-boot-env (128KB)
0x100000 - 0x8000000 : ubi       (127MB)
```

---

## 🙏 Acknowledgments

Thanks to **APCCV** for the DNS-320 OpenWrt port!

**Project:** https://github.com/APCCV/openwrt-dns320  
**Date:** January 2026

