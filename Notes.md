# Notes, checks, and limitations

This is a reference file for diagnostics and personal observations. Exact power use is still to be measured. In a preliminary test with a 70% charge, the phone ran for seven days with the screen off and a small number of devices and sensors; observed consumption was below 1 W. Over several months of iterations, tests, reinstalls, and configuration changes, one reboot occurred while the screen was being activated during SD-card formatting and verification. It happened before Home Assistant and the startup script were installed, and no useful log was retained.

## Command context

| Environment | Used for | Example |
|---|---|---|
| Regular Termux | `pkg`, Termux:Boot, SSH, Mosquitto, widgets | `pkg install mosquitto` |
| Termux via `su -c '…'` | Android root, `chroot-distro`, bind mounts | `su -c id` |
| Debian chroot | `apt`, `uv`, `hass`, ESPHome | `apt update` |

We do not run `apt` in Termux or `pkg` in Debian. When a command is not found, we first confirm the current environment.

## Checkpoints

| After step | Minimum result |
|---|---|
| 01 | Android boots, root is stable, and networking works |
| 02 | `chroot-distro command debian` returns root and `aarch64`/`arm64` |
| 03 | `hass --script check_config` succeeds and the HA web interface opens manually |
| 04 | after reboot, `~/init.log` confirms startup |
| 05 | `esphome-device-builder --help` works and Builder is available on port `6052` |
| 06 | the widget finds and runs scripts from `~/.shortcuts/` |

## Known limitations

- Home Assistant Core in a virtual environment is no longer a supported production installation method. See the [official announcement](https://www.home-assistant.io/blog/2025/05/22/deprecating-core-and-supervised-installation-methods-and-32-bit-systems/).
- A native chroot shares the Android kernel and network namespace; it is neither a Docker container nor a virtual machine. We test `systemd`, cgroups, USB, Bluetooth, and multicast on the specific ROM.
- Root does not override SELinux policy. We do not install untrusted integrations or expose services to the WAN.
- Android can terminate Termux or Debian when RAM is low. Battery-optimization settings and a wake lock reduce the risk, but do not provide server-grade availability.
- An SD card is an additional point of failure. Without one, the step 04 script intentionally does not create an empty HA configuration in internal storage.

## Maintenance

- Before an update, we back up the HA configuration directory and record the versions of `hass`, Python, and installed packages. Updates can break a working setup.
- We update one layer at a time: Android/ROM → chroot → Debian → HA or ESPHome.
- For cache cleanup, we use the [script from step 06](06_Termux_Widget.md), not broad deletion of `/tmp` or system directories.
- Main logs: `$HA_CONFIG_DIR/home-assistant.log` (`/mnt/ha_native/home-assistant.log` for SD card or `/root/.homeassistant/home-assistant.log` for internal storage), `~/init.log` (in Termux), and `/root/esphome.log` (in Debian).

## Personal configuration log

```text
Date:
Model / codename:
Android / ROM:
Magisk, BusyBox NDK, chroot-distro:
Debian / Python / Home Assistant:
SD card (yes/no, UUID):
ESPHome:
Change, result, and log link:
```
