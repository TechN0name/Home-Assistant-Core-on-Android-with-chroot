# Home Assistant on Android with a native chroot: 01 — Device preparation

This guide uses a native `chroot` on rooted Android, not `proot`. A native chroot is useful when direct access to networking, USB, Bluetooth, or an SD card matters. An error while flashing or formatting can erase data or leave the phone unable to boot.

## Tested setup

The tested device is a **Xiaomi Redmi Note 8T** with Snapdragon 665, 4 GB RAM, 64 GB storage, an unlocked bootloader, LineageOS, and Magisk. Xiaomi device names and ROM codenames are easy to confuse, so we verify the **exact model and codename** against the source for the intended build before downloading recovery, firmware, or ROM files. We never flash files selected only by a similar phone name. Recent HA versions may not run on 32-bit ROMs because of `cryptography` limitations.

After installing Termux, we confirm the CPU architecture:

```sh
getprop ro.product.cpu.abi
```

Expected output:

```text
arm64-v8a
```

```sh
uname -m
```

Expected output:

```text
aarch64
```

After an installation, we record the following in an issue or README:

- model, codename, RAM, and storage capacity;
- Android / LineageOS version and build date;
- firmware version;
- Magisk, `chroot-distro`, and BusyBox NDK versions;
- Home Assistant and Python versions;
- whether an SD card is used and its filesystem.

## Prerequisites

- A PC with working `adb` and `fastboot` that detects the phone.
- A reliable USB cable and at least 60% battery charge.
- A backup of all phone data. Unlocking the bootloader and formatting `data` erase it.
- The official installation guide, recovery, and firmware for the exact LineageOS variant.
- An SD card, if needed. Formatting it in step 03 **erases all of its data**.
- For USB devices such as Zigbee or Ethernet adapters, a USB-C hub with Power Delivery (PD) is needed to power the phone and use peripherals simultaneously.
- Stable chroot startup often requires SELinux to run in permissive mode (`setenforce 0` as root during boot). We account for this security trade-off: a rooted phone is not used for banking apps or sensitive personal data, and SSH, MQTT, and the Home Assistant web interface are not exposed on untrusted networks.

## Flashing sequence

For the Xiaomi Redmi Note 8T, we use the [official builds](https://download.lineageos.org/devices/ginkgo/builds) and follow the [official installation guide](https://wiki.lineageos.org/devices/ginkgo/install/variant2/).

The summary below is intentionally abbreviated. We enter Fastboot mode by holding **Volume Down + Power**, then open a command prompt or terminal in the `flashing` directory on the PC.

First, verify that Fastboot detects the device:

```sh
cd C:\flashing 
fastboot devices
```

If the device is listed, flash the partition images:

```sh
fastboot flash vbmeta vbmeta.img
fastboot flash dtbo dtbo.img
fastboot flash boot boot.img
fastboot flash recovery recovery.img
```

In Lineage Recovery, we select **Apply update → Apply from ADB** (the screen displays “Now send the package…”). We then check the ADB connection from the PC:

```sh
adb devices
```

Firmware, ROM, and root packages must be sideloaded sequentially. Each sideload requires selecting **Apply update → Apply from ADB** in Lineage Recovery before executing the corresponding `adb sideload` command on the PC:

1. In recovery, select **Apply update → Apply from ADB**, then run:

```sh
adb sideload fw.zip
```

2. After sideloading completes, select **Apply update → Apply from ADB** again, then run:

```sh
adb sideload lineage.zip
```

3. Select **Apply update → Apply from ADB** once more, then run:

```sh
adb sideload magisk.zip
```

## Before continuing

We continue to `02_Android_Config_Termux.md` only when every item below is true:

```text
[ ] Android boots without a reboot loop.
[ ] Magisk reports a valid root status.
[ ] Wi-Fi connects and the phone has Internet access.
[ ] Root persists after a reboot.
[ ] The recovery and Fastboot entry methods, and the recovery procedure, are known.
```
