# Home Assistant on Android with a native chroot: 03 — Debian, Python, Home Assistant, and Mosquitto

In this step, we create a working Home Assistant instance, verify a manual start, and optionally configure an SD card and MQTT broker. We proceed to automatic startup in step 04 only after the manual verification succeeds.

## Before starting: two independent choices

| Feature | Required? | What it changes |
|---|---:|---|
| SD card for `/config` and SQLite | No | Adds formatting, a bind mount, and a startup wait for the card |
| Mosquitto MQTT broker | No | Adds a Termux package, password, configuration, and optional watchdog startup |

Without an SD card, we skip section **A** and use `/root/.homeassistant` as the configuration directory. Without MQTT, we skip section **D** and keep `ENABLE_MOSQUITTO=0` in the step 04 script.

> [!WARNING]
> **Root context.** This guide runs HA as root *inside the chroot* because Android restricts networking capabilities and device access. Every custom integration is therefore potentially privileged. We install only trusted integrations and do not expose the HA interface to the Internet.

## 1. Enter Debian and update the base system

In Termux, we run:

```sh
su -c '/system/bin/chroot-distro login debian'
```

Until this subsection is complete, we run every command **in Debian** as root:

```sh
# 1. Update Debian system packages
apt update && apt upgrade -y

# 2. Install base tools, build dependencies, media codecs, and C libraries
apt install -y \
  ca-certificates curl git nano tmux procps iproute2 sqlite3 \
  build-essential autoconf pkg-config \
  libssl-dev libxml2-dev libxslt1-dev libjpeg-dev libffi-dev libudev-dev zlib1g-dev \
  libavformat-dev libavcodec-dev libavdevice-dev libavutil-dev libswscale-dev \
  libswresample-dev libavfilter-dev ffmpeg \
  libturbojpeg0 libpcap-dev
```

The library set follows the current Debian dependency list in the [Home Assistant development documentation](https://developers.home-assistant.io/docs/development_environment/). If a Python package build reports a `rustc` or `cargo` error, we run `apt install -y rustc cargo` and repeat only the package-installation command.

## A. Optional: SD card for configuration and database

### A.1. What this does

This path formats **one selected** SD card as `ext4`; Android should then mount it automatically at `/mnt/media_rw/<UUID>`. The `ha_config` directory is subsequently bind-mounted into the Debian tree as `/mnt/ha_native`.

This bypasses Android’s FUSE layer for HA data. It is useful for SQLite but optional. `ext4` auto-mounting depends on the ROM and SELinux policy; it worked on the tested LineageOS 23.2 / Android 16 setup and must be verified on each device.

> [!CAUTION]
> The following commands erase the selected block device completely. We stop if the card’s size, model, or path is unexpected. `mmcblk1` is never treated as a universal value.

### A.2. Identify the device before formatting

In Debian, we install the tools and inspect available devices:

```sh
apt install -y parted e2fsprogs util-linux
lsblk -o NAME,SIZE,MODEL,TRAN,FSTYPE,MOUNTPOINTS
cat /proc/partitions
ls -l /dev/block/mmcblk*
```

On the tested phone, the SD card was `/dev/block/mmcblk1`, but another phone or kernel can use a different path. We verify the card **size** first. If `lsblk` does not show the device in the chroot, we leave Debian (`exit`) and inspect it from a Termux root shell:

```sh
su -c 'cat /proc/partitions; ls -l /dev/block/mmcblk*'
```

### A.3. Format only after exact verification

Below, `<SD_DEVICE>` is the verified path without a partition number (for example, `/dev/block/mmcblk1`). We use the path for the actual device.

**Step 1. Run interactive `parted` in Debian:**

```sh
parted /dev/block/mmcblk1
```

At the interactive `(parted)` prompt, enter each command sequentially:

```text
mklabel gpt
mkpart primary ext4 1MiB 100%
print
quit
```

**Step 2. Unmount the device if mounted:**
In Android file-manager settings or from a Termux root shell:

```sh
su -c 'umount -l /dev/block/mmcblk1*'
```

**Step 3. Create and configure ext4 filesystem:**
We then create ext4 on the new partition in Debian. For example, if `<SD_DEVICE>` is `/dev/block/mmcblk1`, `<SD_PARTITION>` is `/dev/block/mmcblk1p1`; other device types can use a different partition name. Execute each command sequentially:

```sh
# 1. Create an ext4 filesystem with the HA_CONFIG label:
mkfs.ext4 -F -L HA_CONFIG /dev/block/mmcblk1p1

# 2. Disable ext4 journaling to reduce SD-card wear:
tune2fs -O ^has_journal /dev/block/mmcblk1p1

# 3. Remove space reserved for the system root user (make 100% available for data):
tune2fs -m 0 /dev/block/mmcblk1p1

# 4. Check and display the UUID of the new partition:
blkid /dev/block/mmcblk1p1

# 5. Perform a final filesystem integrity check:
e2fsck -fn /dev/block/mmcblk1p1
```

If `parted` reports that the kernel still uses the partition table, we reboot the phone before formatting and verify the path again.

### A.4. Verify Android auto-mounting

After formatting, we reboot the phone. In Termux, we check whether Android mounted the card automatically (custom ROMs commonly do so for ext4):

```sh
su -c 'ls -la /mnt/media_rw'
su -c 'mount | grep /mnt/media_rw'
```

We record the actual UUID from the directory name or `blkid`. If `/mnt/media_rw/<UUID>` is absent, **we stop**: we do not layer a manual mount of the same block device over Android’s mount. First, we determine whether the ROM supports this card type and why `vold` did not mount it.

### A.5. Create and verify the HA configuration mount

Once the UUID is known, set your actual UUID variable in Termux:

```sh
export SD_UUID="paste_SD_UUID_here"
```

Replace `paste_SD_UUID_here` with your verified UUID. Then execute the directory creation and bind-mount commands using Magisk's `-mm` (`--mount-master`) option to operate in the global mount namespace:

```sh
# Create the HA configuration directory on the card:
su -mm -c "mkdir -p /mnt/media_rw/$SD_UUID/ha_config"

# Create the mount point inside the Debian chroot:
su -mm -c 'mkdir -p /data/local/chroot-distro/debian/mnt/ha_native'

# Expose the card directory to Debian (bind mount):
su -mm -c "mount --bind /mnt/media_rw/$SD_UUID/ha_config /data/local/chroot-distro/debian/mnt/ha_native"
```

We verify the bind mount both in the global namespace and from Debian. A successful `touch` alone is not sufficient: the mount must be visible from the chroot that runs HA.

```sh
su -mm -c 'mount | grep ha_native'
su -mm -c '/system/bin/chroot-distro command debian "mount | grep ha_native"'
su -mm -c '/system/bin/chroot-distro command debian "ls -la /mnt/ha_native"'
su -mm -c '/system/bin/chroot-distro command debian "touch /mnt/ha_native/.write-test && ls -la /mnt/ha_native"'
```

After the check, we remove the test file:

```sh
su -mm -c '/system/bin/chroot-distro command debian "rm -f /mnt/ha_native/.write-test"'
```

> [!WARNING]
> **Bind mount is volatile.** The bind mount created above (`/mnt/ha_native`) exists only in kernel memory. It disappears when the phone reboots. The step 04 startup script restores it automatically, but **only** after it has been configured and started.
>
> If you reboot before completing section C or before step 04 is active, re-run the bind mount in Termux before entering Debian:
>
> ```sh
> export SD_UUID='paste_SD_UUID_here'
> su -mm -c "mkdir -p /mnt/media_rw/$SD_UUID/ha_config"
> su -mm -c 'mkdir -p /data/local/chroot-distro/debian/mnt/ha_native'
> su -mm -c "mount --bind /mnt/media_rw/$SD_UUID/ha_config /data/local/chroot-distro/debian/mnt/ha_native"
> ```

#### Optional: migrate an existing HA configuration to the SD card

We perform this migration only when HA was already initialized with its configuration in internal storage, for example when `/root/.homeassistant/configuration.yaml` and `/root/.homeassistant/.storage` exist. We stop HA first. The commands below copy the known HA configuration files, state storage, and database from `/root/.homeassistant` to the mounted SD directory (`/mnt/ha_native`) without deleting the original files.

In Termux, we enter Debian through the master mount namespace:

```sh
su -mm -c '/system/bin/chroot-distro login debian'
```

In Debian, we run:

```sh
pkill -TERM -f '[s]rv/homeassistant/bin/hass' || true
cp -a /root/.homeassistant/configuration.yaml /root/.homeassistant/.storage /mnt/ha_native/
for item in automations.yaml scripts.yaml scenes.yaml secrets.yaml custom_components www blueprints packages themes home-assistant_v2.db home-assistant_v2.db-shm home-assistant_v2.db-wal; do
    [ -e "/root/.homeassistant/$item" ] && cp -a "/root/.homeassistant/$item" /mnt/ha_native/
done
ls -la /mnt/ha_native
export UV_LINK_MODE=copy
export UV_CONSTRAINT=/srv/homeassistant/constraints.txt
source /srv/homeassistant/bin/activate
hass --script check_config --config /mnt/ha_native
exit
```

If the existing HA configuration is stored in another directory, we copy that directory’s contents instead; we do not copy the entire Debian `/root` directory blindly. This migration applies to an existing HA installation after the Python environment from section B is available. Before enabling step 04, we confirm that `check_config` succeeds and that the expected configuration and `.storage` are present on the SD-backed directory.

This mount disappears after a reboot. The step 04 script restores it **only** when `USE_SD_CARD=1` and the UUID is set.

We return to Debian to install HA:

```sh
su -c '/system/bin/chroot-distro login debian'
```

## B. Python and Home Assistant

At the time of testing, Home Assistant development documentation required Python **3.14.2 or newer**. Before updating HA, we check the [current requirement](https://developers.home-assistant.io/docs/development_environment/).

In Debian, we run:

```sh
# 1. Install the uv package manager
curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"

# 2. Install Python and create the virtual environment
uv python install 3.14.6
uv venv --python 3.14.6 /srv/homeassistant
source /srv/homeassistant/bin/activate

# 3. Set copy mode for chroot/Android
export UV_LINK_MODE=copy

# 4. Install Home Assistant and base system modules
uv pip install --upgrade homeassistant pip wheel aioesphomeapi zlib-ng isal

# 5. Download constraints for the installed version
HA_VERSION="$(hass --version)"
curl -fL "https://raw.githubusercontent.com/home-assistant/core/${HA_VERSION}/homeassistant/package_constraints.txt" -o /srv/homeassistant/constraints.txt
test -s /srv/homeassistant/constraints.txt && export UV_CONSTRAINT=/srv/homeassistant/constraints.txt

# 6. Record the installed package set
uv pip freeze > /root/homeassistant-requirements-$(date +%F).txt
```

The virtual environment stays in internal storage (`/srv/homeassistant`); only the configuration directory and SQLite database are stored on the SD card or internal storage.

## Project uv profile

Python and packages are managed with `uv`. For stable operation in an Android/chroot environment, we use the following variable profile:

```sh
export UV_LINK_MODE=copy
export UV_CONSTRAINT=/srv/homeassistant/constraints.txt
source /srv/homeassistant/bin/activate
```

`UV_LINK_MODE=copy` avoids hard links and symbolic links between the cache and isolated environment, which is important for Android filesystems.

`UV_CONSTRAINT` applies the compatible constraints file for the Home Assistant version and helps prevent dependency breakage during updates.

We do not use `UV_SYSTEM_PYTHON=1` in this project. It breaks venv isolation and Debian’s externally-managed protection blocks package installation.

If `test -s /srv/homeassistant/constraints.txt` fails, we stop and check Internet access and the output of `hass --version`.

## C. First manual Home Assistant start

### Pre-check: verify the SD card mount (SD card mode only)

If the SD card option was chosen in section A, confirm the bind mount is active before entering Debian. The mount is volatile and does not survive a reboot.

In Termux, run:

```sh
su -mm -c 'mount | grep ha_native'
```

- **Output is empty:** the mount is gone (the phone was rebooted). Restore it using the commands in the warning box at the end of section A.5, then continue.
- **Output contains a bind-mount entry:** the mount is active and Debian can be entered.

We choose one of two configuration locations:

**Option 1 — with an SD card.** We set the working directory inside Debian to the bind-mounted SD card location:
```sh
export HA_CONFIG_DIR="/mnt/ha_native"
```

**Option 2 — without an SD card / internal storage.** The following line is intentionally commented; remove only the leading `#` when using this option:

```sh
#export HA_CONFIG_DIR="/root/.homeassistant"
```

### Common working configuration path (`$HA_CONFIG_DIR`)

All subsequent maintenance and configuration commands in this guide rely on the single common working directory environment variable `$HA_CONFIG_DIR` selected above.

Depending on your installation mode:
- **Internal Storage (Mode A):** The real configuration directory is `/root/.homeassistant`.
- **SD Card (Mode B):** The working configuration directory inside Debian is `/mnt/ha_native`.

**Why `/mnt/ha_native` is used for Mode B:**
Inside the Debian chroot, `/mnt/ha_native` is the active bind mount where the SD card's `ha_config` folder is mounted. This is the exact path used by Home Assistant itself after startup (`hass --config /mnt/ha_native`). Physical host storage at `/mnt/media_rw/<UUID>` exists only in the Android host mount namespace and must **never** be used for normal Home Assistant maintenance or administration inside Debian.

By using `$HA_CONFIG_DIR`, all administration steps—editing `configuration.yaml`, running `check_config`, checking `home-assistant.log`, inspecting SQLite, or performing backups—remain identical regardless of storage mode.

We create the configuration directory:
```sh
install -d -m 0700 "$HA_CONFIG_DIR"
```
The first start creates the base file structure and downloads required integration components. We stop it with `Ctrl+C` only after the logs report a successful startup and the web interface is ready on port 8123:

```sh
export UV_LINK_MODE=copy
export UV_CONSTRAINT=/srv/homeassistant/constraints.txt
source /srv/homeassistant/bin/activate

hass --config "$HA_CONFIG_DIR"
```

We open `http://<phone-IP>:8123` in a browser, complete the onboarding flow, and create an account. After the first successful sign-in, we stop the terminal process with `Ctrl+C` and validate the configuration:

```sh
export UV_LINK_MODE=copy
export UV_CONSTRAINT=/srv/homeassistant/constraints.txt
source /srv/homeassistant/bin/activate

hass --script check_config --config "$HA_CONFIG_DIR"
```

### To reduce writes to an SD card or internal flash, we edit configuration.yaml:

```sh
nano "$HA_CONFIG_DIR/configuration.yaml"
```

```yaml
default_config:

frontend:
  themes: !include_dir_merge_named themes

automation: !include automations.yaml
script: !include scripts.yaml
scene: !include scenes.yaml

# Reduce log flush frequency to protect the storage device
recorder:
  commit_interval: 60
  purge_keep_days: 14
```

In nano, `Ctrl+O`, then Enter saves; `Ctrl+X` exits.

We validate the configuration after every `.yaml` edit:

```sh
hass --script check_config --config "$HA_CONFIG_DIR"
```


We manually start HA and confirm that the web interface opens after a restart. We do not continue to step 04 until this is confirmed.

## E. Optional: Android hardware sensors via `command_line`

This section adds device-specific sensors that read hardware telemetry directly from the Linux kernel's sysfs interface (`/sys/class/`). All commands run inside the Debian chroot as root, which has access to Android kernel interfaces through the native chroot.

> [!NOTE]
> These sensors are specific to the Redmi Note 8T (Snapdragon 665) and the tested kernel. The thermal zone indices and sysfs paths may differ on other devices. Before adding a sensor, verify the path exists:
>
> ```sh
> cat /sys/class/power_supply/battery/capacity
> ```
>
> If the file is absent or unreadable, the sensor is not available on this device.

We edit `configuration.yaml` and add a `command_line:` block. If `command_line:` already exists in the file, we append the new entries under the existing key; we do not add the key twice.

```sh
nano "$HA_CONFIG_DIR/configuration.yaml"
```

```yaml
command_line:

  # --- Temperature sensors ---

  - sensor:
      name: "RN8T CPU Temperature"
      unique_id: rn8t_cpu_temperature
      # Dynamically resolves the thermal zone index for the primary CPU cluster.
      # The grep finds the zone whose type file contains 'cpuss-0-usr'.
      command: "cat /sys/class/thermal/thermal_zone$(grep -l 'cpuss-0-usr' /sys/class/thermal/thermal_zone*/type | grep -oE '[0-9]+')/temp"
      unit_of_measurement: "°C"
      device_class: temperature
      state_class: measurement
      # The kernel reports temperature in millidegrees Celsius; divide by 1000.
      value_template: "{{ (value | float / 1000) | round(1) }}"
      scan_interval: 30

  - sensor:
      name: "RN8T Battery Temperature"
      unique_id: rn8t_battery_temperature
      command: "cat /sys/class/power_supply/battery/temp"
      unit_of_measurement: "°C"
      device_class: temperature
      state_class: measurement
      # The battery driver reports temperature in tenths of a degree; divide by 10.
      value_template: "{{ (value | float / 10) | round(1) }}"
      scan_interval: 30

  - sensor:
      name: "RN8T eMMC Temperature"
      unique_id: rn8t_emmc_temperature
      # Dynamically resolves the thermal zone index for the eMMC/UFS sensor.
      command: "cat /sys/class/thermal/thermal_zone$(grep -l 'emmc-ufs-therm-adc' /sys/class/thermal/thermal_zone*/type | grep -oE '[0-9]+')/temp"
      unit_of_measurement: "°C"
      device_class: temperature
      state_class: measurement
      value_template: "{{ (value | float / 1000) | round(1) }}"
      # Storage temperature changes slowly; 60-second polling is sufficient.
      scan_interval: 60

  # --- Battery and power sensors ---

  - sensor:
      name: "RN8T Battery Level"
      unique_id: rn8t_battery_level
      command: "cat /sys/class/power_supply/battery/capacity"
      unit_of_measurement: "%"
      device_class: battery
      state_class: measurement
      value_template: "{{ value | int }}"
      # If the charging limit is set to 70 % (step 02), this value stabilises at 70.
      scan_interval: 60

  - sensor:
      name: "RN8T Battery Current"
      unique_id: rn8t_battery_current
      command: "cat /sys/class/power_supply/battery/current_now"
      unit_of_measurement: "mA"
      device_class: current
      state_class: measurement
      # The Qualcomm driver reports current in microamperes (µA); divide by 1000 for mA.
      # Positive values indicate charging; negative values indicate discharge (consumption).
      value_template: "{{ (value | float / 1000) | round(1) }}"
      # Short interval for accurate charge/discharge graphs.
      scan_interval: 10

  - sensor:
      name: "RN8T Battery Cycle Count"
      unique_id: rn8t_battery_cycle_count
      command: "cat /sys/class/power_supply/battery/cycle_count"
      icon: mdi:battery-sync
      # total_increasing: the value only grows over the battery's lifetime.
      state_class: total_increasing
      value_template: "{{ value | int }}"
      # Cycle count changes rarely; hourly polling avoids unnecessary writes.
      scan_interval: 3600
```

In nano, `Ctrl+O`, then Enter saves; `Ctrl+X` exits.

> [!IMPORTANT]
> **Full Home Assistant Restart Required:**
> Adding new `command_line` sensors or changing existing sensor unique IDs requires a **full Home Assistant restart** (**Settings → System → Restart** or restarting the service). Reloading YAML configuration alone is insufficient for newly added entities to appear in Home Assistant.

We validate the configuration before restarting:

```sh
hass --script check_config --config "$HA_CONFIG_DIR"
```

After a successful check, we restart Home Assistant. The sensors appear under **Settings → Devices & services → Entities** with the names defined above.

> [!NOTE]
> The dynamic thermal zone lookup (`grep -l … | grep -oE '[0-9]+'`) resolves the correct zone index at every poll. If two thermal zones match the same type string on a different kernel, the command returns multiple lines and the sensor reports an error. In that case, identify the correct fixed index with `grep -r 'cpuss-0-usr' /sys/class/thermal/` and replace the dynamic command with a hardcoded path such as `cat /sys/class/thermal/thermal_zone4/temp`.

### E.2. Optional: Advanced system and resource monitoring

The optional sensors below provide useful system telemetry for Home Assistant running on Android devices. Add these entries under the `command_line:` section in `configuration.yaml`:

```yaml
command_line:

  # --- Advanced system and resource monitoring (Optional) ---

  - sensor:
      name: "RN8T Available RAM"
      unique_id: rn8t_available_ram
      command: "awk '/MemAvailable/ {print int($2/1024)}' /proc/meminfo"
      unit_of_measurement: "MB"
      device_class: data_size
      state_class: measurement
      value_template: "{{ value | int }}"
      scan_interval: 30

  - sensor:
      name: "RN8T CPU Load (1m)"
      unique_id: rn8t_cpu_load_1m
      command: "cat /proc/loadavg | awk '{print $1}'"
      icon: mdi:chip
      state_class: measurement
      value_template: "{{ value | float | round(2) }}"
      scan_interval: 15

  - sensor:
      name: "RN8T CPU Frequency"
      unique_id: rn8t_cpu_frequency
      command: "cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq 2>/dev/null || echo 0"
      unit_of_measurement: "MHz"
      state_class: measurement
      value_template: "{{ (value | float / 1000) | round(0) | int }}"
      scan_interval: 15

  - sensor:
      name: "RN8T Internal Storage Free"
      unique_id: rn8t_internal_storage_free
      command: "df -m /data | awk 'NR==2 {print $4}'"
      unit_of_measurement: "MB"
      device_class: data_size
      state_class: measurement
      value_template: "{{ value | int }}"
      scan_interval: 300

  - sensor:
      name: "RN8T HA Database Size"
      unique_id: rn8t_ha_database_size
      command: "[ -f /mnt/ha_native/home-assistant_v2.db ] && du -m /mnt/ha_native/home-assistant_v2.db | awk '{print $1}' || du -m /root/.homeassistant/home-assistant_v2.db 2>/dev/null | awk '{print $1}'"
      unit_of_measurement: "MB"
      device_class: data_size
      state_class: measurement
      value_template: "{{ value | int }}"
      scan_interval: 300

  - sensor:
      name: "RN8T Storage Mount Status"
      unique_id: rn8t_storage_mount_status
      command: "mountpoint -q /mnt/ha_native && echo 'Mounted (SD)' || echo 'Internal Storage'"
      icon: mdi:harddisk
      scan_interval: 60

  - sensor:
      name: "RN8T System Uptime"
      unique_id: rn8t_system_uptime
      command: "cat /proc/uptime | awk '{print int($1)}'"
      unit_of_measurement: "s"
      device_class: duration
      state_class: total_increasing
      value_template: "{{ value | int }}"
      scan_interval: 60
```

> [!NOTE]
> Hardware support and sysfs file paths vary by Android device model and kernel build. For example, CPU frequency exposure depends on kernel `cpufreq` drivers. If any sensor returns an unreadable value or error, test the underlying command in the Debian chroot terminal before adding it to `configuration.yaml`. Remember that adding new sensors requires a full Home Assistant restart.

## D. Optional: local Mosquitto in Termux

We use this section only when a **local MQTT broker** is required. If HA connects to an existing network broker or MQTT is not used, we skip it.

First, install Mosquitto and create configuration directories:

```sh
pkg install -y mosquitto
mkdir -p ~/.config/mosquitto ~/.local/share/mosquitto
chmod 700 ~/.config/mosquitto ~/.local/share/mosquitto
```

Next, interactively create a password file for the `homeassistant` user:

```sh
mosquitto_passwd -c ~/.config/mosquitto/passwd homeassistant
```

Enter and confirm your password when prompted. Then edit the configuration file:

```sh
nano ~/.config/mosquitto/mosquitto.conf
```

We paste this configuration for access **only from the phone**:

```conf
persistence true
persistence_location /data/data/com.termux/files/home/.local/share/mosquitto/

log_dest file /data/data/com.termux/files/home/.local/share/mosquitto/mosquitto.log

# Listen on the LAN; for phone-only access, use 127.0.0.1 instead
listener 1883 0.0.0.0

# Disallow connections without a password
allow_anonymous false
password_file /data/data/com.termux/files/home/.config/mosquitto/passwd
```

For the first diagnostic, we optionally run the broker in the foreground:

```sh
mosquitto -c ~/.config/mosquitto/mosquitto.conf -v
```

In a second Termux session, we verify it; we then stop the first session with `Ctrl+C`:

```sh
mosquitto_sub -h 127.0.0.1 -p 1883 -u homeassistant -P '<your_password>' -t test/topic -C 1
```

```sh
mosquitto_pub -h 127.0.0.1 -p 1883 -u homeassistant -P '<your_password>' -t test/topic -m ok
```

In **Settings → Devices & services → Add integration → MQTT**, we use:

```sh
Broker: 127.0.0.1
Port: 1883
Username / password: the credentials created above.
```

## Updates, logs, and backups

We run all Home Assistant commands below in Debian. Before updating, we back up the configuration directory; for an SD card, we keep a copy of the archive off the phone as well.

```sh
tar -C "$HA_CONFIG_DIR" -czf /root/ha-config-$(date +%F).tar.gz .
source /srv/homeassistant/bin/activate
export UV_LINK_MODE=copy
uv pip freeze > /root/homeassistant-requirements-before-update-$(date +%F).txt
uv pip install --upgrade homeassistant
HA_VERSION="$(hass --version)"
curl -fL "https://raw.githubusercontent.com/home-assistant/core/${HA_VERSION}/homeassistant/package_constraints.txt" -o /srv/homeassistant/constraints.txt
test -s /srv/homeassistant/constraints.txt
export UV_CONSTRAINT=/srv/homeassistant/constraints.txt
hass --script check_config --config "$HA_CONFIG_DIR"
```

Main logs:

```sh
# Home Assistant log in Debian (uses the common working directory):
tail -f "$HA_CONFIG_DIR/home-assistant.log"

# In Termux after installing the step 04 script:
tail -f ~/init.log

# Only when Mosquitto is enabled:
tail -f ~/.local/share/mosquitto/mosquitto.log
```
