# Home Assistant Core on Android with chroot

A practical record of deploying Home Assistant Core on rooted Android 16 through chroot. This is not a universal guide: every command was tested only on a Xiaomi Redmi Note 8T (Snapdragon 665, 4 GB RAM, 64 GB storage) running LineageOS, Magisk, `chroot-distro`, and Debian. It documents a working configuration of that setup.

> Unlocking the bootloader, gaining root access, and formatting an SD card can erase data or leave a device unable to boot. We back up all data and use firmware files only for the exact device model.

> [!NOTE]
> This is an experimental, unofficial way to run Home Assistant Core in a Python environment. Home Assistant ended support for production Core installations after release 2025.12. The instance can still be updated technically, but official troubleshooting support is unavailable. For a supported deployment, we choose Home Assistant OS or Container on compatible hardware. See the [Home Assistant announcement](https://www.home-assistant.io/blog/2025/05/22/deprecating-core-and-supervised-installation-methods-and-32-bit-systems/).

## Roadmap

| Step | Outcome | Status |
|---|---|---|
| [01 — Device preparation](01_Device_Preparation.md) | Compatible Android with root and reliable booting | Required |
| [02 — Android, Termux, and chroot](02_Android_Config_Termux.md) | Termux, `chroot-distro`, and a working Debian install | Required |
| [03 — Debian, Python, HA, and Mosquitto](03_Debian_Python_HA_Mo.md) | Manual Home Assistant startup; SD card and Mosquitto are optional | Required |
| [04 — Automatic startup and watchdog](04_automation_script.md) | Termux:Boot starts the verified HA instance | After step 03 |
| [05 — ESPHome Device Builder](05_ESPHome_Builder.md) | Separate environment for creating and building ESPHome firmware | Optional |
| [06 — Termux:Widget](06_Termux_Widget.md) | Manual buttons for ESPHome and maintenance tasks | Optional |

## Scope and options

- **Without an SD card:** HA configuration is stored in `/root/.homeassistant`.
- **With an SD card:** Home Assistant configuration (`/mnt/ha_native`) and SQLite database are stored on an ext4 card through a bind mount; automatic startup does not start HA until the card is available.
- **Without Mosquitto:** we skip the MQTT block in step 03 and keep `ENABLE_MOSQUITTO=0` in step 04.
- **ESPHome:** an independent service from Home Assistant. We can install it after step 03 and leave it stopped until firmware needs to be built.

## Workflow

1. We do not enable automatic startup until Home Assistant has completed a manual start and `check_config` in step 03.
2. We update only one layer at a time: Android/ROM, then chroot, Debian packages, Home Assistant, or ESPHome.
3. We do not expose HA, SSH, MQTT, or ESPHome directly to the Internet. We use them only on a trusted local network or through separately configured secure access.

## Troubleshooting

| Symptom | First check | Where to fix it |
|---|---|---|
| HA does not open | `tail -f "$HA_CONFIG_DIR/home-assistant.log"` | Step 03 |
| HA does not start after reboot | `tail -n 100 ~/init.log` | Step 04 |
| SD card is unavailable | `/mnt/media_rw`, UUID, and bind mount | Step 03/04 |
| ESPHome does not start | `/root/esphome.log` in Debian | Step 05/06 |
| Widget does not find scripts | `0700` permissions, `~/.shortcuts/`, and widget refresh | Step 06 |

Additional checks, limitations, and a log template are in [Notes.md](Notes.md).

[Experimental Home Assistant deployment video: errors and what not to do](https://www.youtube.com/watch?v=k6Uly6q1nxk)
