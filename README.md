# Home Assistant Core on Android with a chroot

Українська версія посібника: [README українською](uk/README.md).

A practical guide to running Home Assistant Core on rooted Android 16 through a native chroot. Commands were tested on a Xiaomi Redmi Note 8T (Snapdragon 665, 4 GB RAM, 64 GB storage) running LineageOS, Magisk, `chroot-distro`, and Debian. Results may differ on other devices and ROMs.

> [!CAUTION]
> Unlocking the bootloader, gaining root access, and formatting an SD card can erase data or prevent a device from booting. Back up all data and use firmware files for the exact device model.

> [!NOTE]
> This is an experimental, unofficial Home Assistant Core deployment method. Production support for Core installations ended with release 2025.12. The instance can still be updated, but official troubleshooting support is unavailable. For a supported deployment, use Home Assistant OS or Container on compatible hardware. See the [2025.12 release note](https://www.home-assistant.io/blog/2025/12/03/release-202512/) for the support change and the [original deprecation announcement](https://www.home-assistant.io/blog/2025/05/22/deprecating-core-and-supervised-installation-methods-and-32-bit-systems/) for background.

## Roadmap

| Step | Guide | Outcome | Status |
|---|---|---|---|
| 01 | [Device preparation](01_Device_Preparation.md) | Compatible Android device with root access and reliable booting | Required |
| 02 | [Android, Termux, and chroot](02_Android_Config_Termux.md) | Termux, `chroot-distro`, and Debian installation | Required |
| 03 | [Debian and Python](03_Debian_Python.md) | Debian packages and isolated Python environment | Required |
| 04 | [Home Assistant setup](04_Home_Assistant.md) | Manual Home Assistant start; SD-card storage and sensors are optional | Required |
| 05 | [Mosquitto MQTT broker](05_Mosquitto_MQTT.md) | Optional local MQTT broker | Optional |
| 06 | [Automatic startup and watchdog](06_Automatic_Startup.md) | Termux:Boot starts the verified Home Assistant instance | After step 04 |
| 07 | [Chrony NTP server](07_Chrony_NTP_Server.md) | Optional LAN time server in Termux | Optional |
| 08 | [ESPHome Device Builder](08_ESPHome_Builder.md) | Separate environment for building ESPHome firmware | Optional |
| 09 | [Termux:Widget](09_Termux_Widget.md) | Manual ESPHome controls and WatchDog maintenance buttons | Optional |
| 10 | [OpenBeken `autoexec.bat`](10_OpenBeken_Autoexec.md) | Standalone device schedule and local NTP synchronization | Device-specific |
| 11 | [Updates and cleanup](11_Update_cleanup.md) | Separate update and safe cleanup procedures | Reference |

## Device-level automation

OpenBeken smart-plug automation runs independently on the device firmware and is separate from the Home Assistant deployment stages. [Step 10 — OpenBeken `autoexec.bat`](10_OpenBeken_Autoexec.md) documents an autonomous schedule and local NTP synchronization through the Chrony server from [step 07](07_Chrony_NTP_Server.md). Additional OpenBeken features may be documented in that guide as they are added.

## Scope and options

- **Without an SD card:** Home Assistant configuration is stored in `/root/.homeassistant`.
- **With an SD card:** Home Assistant configuration (`/mnt/ha_native`) and the SQLite database are stored on an ext4 card through a bind mount. Automatic startup waits for the card.
- **Without Mosquitto:** Skip step 05 and keep `ENABLE_MOSQUITTO=0` in step 06.
- **Without Chrony:** Skip step 07 and keep `ENABLE_CHRONY=0` in step 06.
- **ESPHome:** Runs independently from Home Assistant. Install it after step 04 and leave it stopped until firmware needs to be built.

## Workflow

1. Enable automatic startup only after Home Assistant has completed a manual start and `check_config` has succeeded in step 04.
2. Follow [step 11](11_Update_cleanup.md) for updates and cleanup; update one layer at a time: Android/ROM, chroot, Debian packages, Home Assistant, or ESPHome.
3. Keep Home Assistant, SSH, MQTT, ESPHome, and NTP access on a trusted local network. Do not expose these services directly to the Internet.

## Troubleshooting

| Symptom | First check | Guide |
|---|---|---|
| Home Assistant does not open | `tail -f "$HA_CONFIG_DIR/home-assistant.log"` | Step 04 |
| Home Assistant does not start after reboot | `tail -n 100 ~/init.log` | Step 06 |
| SD card is unavailable | `/mnt/media_rw`, UUID, and bind mount | Steps 04 and 06 |
| Mosquitto does not start | `~/.local/share/mosquitto/mosquitto.log` | Step 05 |
| ESPHome does not start | `/root/esphome.log` in Debian | Steps 08 and 09 |
| Widget does not find scripts | `0700` permissions, `~/.shortcuts/`, and widget refresh | Step 09 |
| NTP clients do not receive time | Chrony configuration, LAN subnet, and UDP port 123 | Step 07 |

See [Notes.md](Notes.md) for additional checks, limitations, and a configuration log template.

[Experimental Home Assistant deployment video: errors and what not to do](https://www.youtube.com/watch?v=k6Uly6q1nxk)
