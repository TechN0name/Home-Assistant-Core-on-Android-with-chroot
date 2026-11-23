# Notes, checks, and limitations

This reference file records diagnostics, limitations, and observed behavior. Exact power use is still to be measured. In a preliminary test with a 70% charge, the phone ran for seven days with the screen off and a small number of devices and sensors; observed consumption was below 1 W. Over several months of iterations, tests, reinstalls, and configuration changes, one reboot occurred while the screen was being activated during SD-card formatting and verification. The reboot happened before Home Assistant and the startup script were installed, and no useful log was retained.

## Command context

| Environment | Used for | Example |
|---|---|---|
| Regular Termux | `pkg`, Termux:Boot, SSH, Mosquitto, widgets | `pkg install mosquitto` |
| Termux via `su -c '…'` | Android root, `chroot-distro`, bind mounts | `su -c id` |
| Debian chroot | `apt`, `uv`, `hass`, ESPHome | `apt update` |

Do not run `apt` in Termux or `pkg` in Debian. When a command is not found, first confirm the current environment.

## Checkpoints

| After step | Minimum result |
|---|---|
| 01 | Android boots, root is stable, and networking works |
| 02 | `chroot-distro command debian` returns root and `aarch64`/`arm64` |
| 03 | Python environment is active and `hass --version` returns a version |
| 04 | `hass --script check_config` succeeds and the HA web interface opens manually |
| 05 | Optional: Mosquitto accepts authenticated local MQTT connections |
| 06 | After reboot, `~/init.log` confirms automatic startup |
| 07 | Optional: Chrony responds to NTP requests on UDP port `123` |
| 08 | `esphome-device-builder --help` works and Builder is available on port `6052` |
| 09 | The widget finds and runs scripts from `~/.shortcuts/` |
| 10 | OpenBeken devices synchronize with the local NTP server and run the configured autonomous schedule |
| 11 | Updates have dated backups and cleanup is limited to reviewed caches and service-owned history |

## Known limitations

- Home Assistant Core in a virtual environment is no longer a supported production installation method. See the [official announcement](https://www.home-assistant.io/blog/2025/05/22/deprecating-core-and-supervised-installation-methods-and-32-bit-systems/).
- A native chroot shares the Android kernel and network namespace; it is neither a Docker container nor a virtual machine. Test `systemd`, cgroups, USB, Bluetooth, and multicast on the specific ROM.
- Root does not override SELinux policy. Do not install untrusted integrations or expose services to the WAN.
- Android can terminate Termux or Debian when RAM is low. Battery-optimization settings and a wake lock reduce the risk, but do not provide server-grade availability.
- An SD card is an additional point of failure. Without one, the step 06 script intentionally does not create an empty HA configuration in internal storage.

## Maintenance

- Follow [step 11](11_Update_cleanup.md) for the complete update and cleanup procedures. They include dated backups, package-version records, the WatchDog maintenance flag, safe cache cleanup, and Home Assistant Recorder guidance.
- Main logs: `$HA_CONFIG_DIR/home-assistant.log` (`/mnt/ha_native/home-assistant.log` for SD card or `/root/.homeassistant/home-assistant.log` for internal storage), `~/init.log` (in Termux), and `/root/esphome.log` (in Debian).

## Future guide additions

Extend [step 10 — OpenBeken `autoexec.bat`](10_OpenBeken_Autoexec.md) with future device-level features, including power-saving options and other autonomous smart-plug settings. Keep the local Chrony NTP dependency documented when relevant.

## Configuration log

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
