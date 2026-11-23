# Home Assistant on Android with a native chroot: 04 — Home Assistant setup

Configure Home Assistant, verify a manual start, and optionally place configuration and database files on an SD card. Complete the manual verification before enabling automatic startup in [step 06](06_Automatic_Startup.md).

> [!WARNING]
> Home Assistant runs as root inside the chroot because Android restricts networking capabilities and device access. Custom integrations can therefore access privileged resources. Install trusted integrations only and keep the Home Assistant interface on a trusted network.

## Optional: SD card for configuration and database

### Purpose

This path formats **one selected** SD card as `ext4`; Android should then mount it automatically at `/mnt/media_rw/<UUID>`. The `ha_config` directory is subsequently bind-mounted into the Debian tree as `/mnt/ha_native`.

This bypasses Android’s FUSE layer for HA data. It is useful for SQLite but optional. `ext4` auto-mounting depends on the ROM and SELinux policy; it worked on the tested LineageOS 23.2 / Android 16 setup and must be verified on each device.

> [!CAUTION]
> The following commands erase the selected block device completely. Stop if the card’s size, model, or path is unexpected. `mmcblk1` is never treated as a universal value.

### Identify the device before formatting

In Debian, install the tools and inspect available devices:

```sh
apt install -y parted e2fsprogs util-linux
lsblk -o NAME,SIZE,MODEL,TRAN,FSTYPE,MOUNTPOINTS
cat /proc/partitions
ls -l /dev/block/mmcblk*
```

On the tested phone, the SD card was `/dev/block/mmcblk1`, but another phone or kernel can use a different path. Verify the card **size** first. If `lsblk` does not show the device in the chroot, leave Debian (`exit`) and inspect it from a Termux root shell:

```sh
su -c 'cat /proc/partitions; ls -l /dev/block/mmcblk*'
```

### Format only after exact verification

Below, `<SD_DEVICE>` is the verified path without a partition number (for example, `/dev/block/mmcblk1`). Use the verified path for the actual device.

The commands below show paths from the tested phone. Before running them, replace every `/dev/block/mmcblk1` and `/dev/block/mmcblk1p1` occurrence with the verified device and partition paths if yours differ. Do not run the formatting commands against an unverified path.

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
Create ext4 on the new partition in Debian. For example, if `<SD_DEVICE>` is `/dev/block/mmcblk1`, `<SD_PARTITION>` is `/dev/block/mmcblk1p1`; other device types can use a different partition name. Run the commands in order. Step 2 is optional; skip it to keep ext4 journaling enabled:

```sh
# 1. Create an ext4 filesystem with the HA_CONFIG label:
mkfs.ext4 -F -L HA_CONFIG /dev/block/mmcblk1p1

# 2. Optional: disable ext4 journaling to reduce SD-card writes.
#    This weakens recovery after an unclean shutdown; skip this command if data integrity is the priority.
tune2fs -O ^has_journal /dev/block/mmcblk1p1

# 3. Remove space reserved for the system root user (make 100% available for data):
tune2fs -m 0 /dev/block/mmcblk1p1

# 4. Check and display the UUID of the new partition:
blkid /dev/block/mmcblk1p1

# 5. Perform a final filesystem integrity check:
e2fsck -fn /dev/block/mmcblk1p1
```

If `parted` reports that the kernel still uses the partition table, reboot the phone before formatting and verify the path again.

### Verify Android auto-mounting

After formatting, reboot the phone. In Termux, check whether Android mounted the card automatically (custom ROMs commonly do so for ext4):

```sh
su -c 'ls -la /mnt/media_rw'
su -c 'mount | grep /mnt/media_rw'
```

Record the actual UUID from the directory name or `blkid`. If `/mnt/media_rw/<UUID>` is absent, **stop**: do not layer a manual mount of the same block device over Android’s mount. First, determine whether the ROM supports this card type and why `vold` did not mount it.

### Create and verify the Home Assistant configuration mount

Once the UUID is known, set the actual UUID variable in Termux:

```sh
export SD_UUID="paste_SD_UUID_here"
```

Replace `paste_SD_UUID_here` with the verified UUID. Then execute the directory creation and bind-mount commands using Magisk's `-mm` (`--mount-master`) option to operate in the global mount namespace:

```sh
# Create the HA configuration directory on the card:
su -mm -c "mkdir -p /mnt/media_rw/$SD_UUID/ha_config"

# Create the mount point inside the Debian chroot:
su -mm -c 'mkdir -p /data/local/chroot-distro/debian/mnt/ha_native'

# Expose the card directory to Debian (bind mount):
su -mm -c "mount --bind /mnt/media_rw/$SD_UUID/ha_config /data/local/chroot-distro/debian/mnt/ha_native"
```

Verify the bind mount both in the global namespace and from Debian. A successful `touch` alone is not sufficient: the mount must be visible from the chroot that runs HA.

```sh
su -mm -c 'mount | grep ha_native'
su -mm -c '/system/bin/chroot-distro command debian "mount | grep ha_native"'
su -mm -c '/system/bin/chroot-distro command debian "ls -la /mnt/ha_native"'
su -mm -c '/system/bin/chroot-distro command debian "touch /mnt/ha_native/.write-test && ls -la /mnt/ha_native"'
```

After the check, remove the test file:

```sh
su -mm -c '/system/bin/chroot-distro command debian "rm -f /mnt/ha_native/.write-test"'
```

> [!WARNING]
> **Bind mount is volatile.** The bind mount created above (`/mnt/ha_native`) exists only in kernel memory. It disappears when the phone reboots. The step 06 startup script restores it automatically, but **only** after it has been configured and started.
>
> If the phone reboots before completing the manual setup or activating step 06, repeat the bind mount in Termux before entering Debian:
>
> ```sh
> export SD_UUID='paste_SD_UUID_here'
> su -mm -c "mkdir -p /mnt/media_rw/$SD_UUID/ha_config"
> su -mm -c 'mkdir -p /data/local/chroot-distro/debian/mnt/ha_native'
> su -mm -c "mount --bind /mnt/media_rw/$SD_UUID/ha_config /data/local/chroot-distro/debian/mnt/ha_native"
> ```

#### Optional: migrate an existing HA configuration to the SD card

Perform this migration only when HA was already initialized with its configuration in internal storage, for example when `/root/.homeassistant/configuration.yaml` and `/root/.homeassistant/.storage` exist. Stop HA first. The commands below copy the known HA configuration files, state storage, and database from `/root/.homeassistant` to the mounted SD directory (`/mnt/ha_native`) without deleting the original files.

In Termux, enter Debian through the master mount namespace:

```sh
su -mm -c '/system/bin/chroot-distro login debian'
```

In Debian, run:

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

If the existing HA configuration is stored in another directory, copy that directory’s contents instead; do not copy the entire Debian `/root` directory blindly. This migration applies to an existing HA installation after the Python environment from step 03 is complete. Before enabling step 06, confirm that `check_config` succeeds and that the expected configuration and `.storage` are present on the SD-backed directory.

This mount disappears after a reboot. The step 06 script restores it **only** when `USE_SD_CARD=1` and the UUID is set.

## Manual Home Assistant start

### Pre-check: verify the SD card mount (SD card mode only)

If the SD card option was selected, confirm the bind mount is active before entering Debian. The mount is volatile and does not survive a reboot.

In Termux, run:

```sh
su -mm -c 'mount | grep ha_native'
```

- **Output is empty:** the mount is gone (the phone was rebooted). Restore it using the commands in the warning box in the SD-card section, then continue.
- **Output contains a bind-mount entry:** the mount is active and Debian can be entered.

Enter Debian from this Termux session. The `-mm` option keeps the master mount namespace visible, including the SD-card bind mount:

```sh
su -mm -c '/system/bin/chroot-distro login debian'
```

Run all commands below in the Debian root shell. Environment variables exported in Termux do not automatically carry into Debian.

Choose one of two configuration locations and set `HA_CONFIG_DIR` in this Debian shell:

**Option 1 — with an SD card.** Set the working directory inside Debian to the bind-mounted SD card location:
```sh
export HA_CONFIG_DIR="/mnt/ha_native"
```

**Option 2 — without an SD card / internal storage.** The following line is intentionally commented; remove only the leading `#` when using this option:

```sh
#export HA_CONFIG_DIR="/root/.homeassistant"
```

### Common working configuration path (`$HA_CONFIG_DIR`)

All subsequent commands in this Debian shell rely on the common working-directory variable `$HA_CONFIG_DIR` selected above. If you open a new Debian shell later, set and export it again.

Select the path that matches the installation mode:
- **Internal Storage (Mode A):** The real configuration directory is `/root/.homeassistant`.
- **SD Card (Mode B):** The working configuration directory inside Debian is `/mnt/ha_native`.

**Why `/mnt/ha_native` is used for Mode B:**
Inside the Debian chroot, `/mnt/ha_native` is the active bind mount where the SD card's `ha_config` folder is mounted. This is the exact path used by Home Assistant itself after startup (`hass --config /mnt/ha_native`). Physical host storage at `/mnt/media_rw/<UUID>` exists only in the Android host mount namespace and must **never** be used for normal Home Assistant maintenance or administration inside Debian.

By using `$HA_CONFIG_DIR`, all administration steps—editing `configuration.yaml`, running `check_config`, checking `home-assistant.log`, inspecting SQLite, or performing backups—remain identical regardless of storage mode.

Create the configuration directory:
```sh
install -d -m 0700 "$HA_CONFIG_DIR"
```
The first start creates the base file structure and downloads required integration components. Stop it with `Ctrl+C` only after the logs report a successful startup and the web interface is ready on port 8123:

```sh
export UV_LINK_MODE=copy
export UV_CONSTRAINT=/srv/homeassistant/constraints.txt
source /srv/homeassistant/bin/activate

hass --config "$HA_CONFIG_DIR"
```

Open `http://<phone_ip>:8123` in a browser, complete the onboarding flow, and create an account. After the first successful sign-in, stop the terminal process with `Ctrl+C` and validate the configuration:

```sh
export UV_LINK_MODE=copy
export UV_CONSTRAINT=/srv/homeassistant/constraints.txt
source /srv/homeassistant/bin/activate

hass --script check_config --config "$HA_CONFIG_DIR"
```

### Reduce writes to an SD card or internal flash

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

In nano, press `Ctrl+O`, then Enter to save; press `Ctrl+X` to exit.

Validate the configuration after every `.yaml` edit:

```sh
hass --script check_config --config "$HA_CONFIG_DIR"
```


Manually start Home Assistant and confirm that the web interface opens after a restart. Continue to step 06 only after this check succeeds.

## Optional: Android hardware sensors via `command_line`

This section adds device-specific sensors that read hardware telemetry directly from the Linux kernel's sysfs interface (`/sys/class/`). All commands run inside the Debian chroot as root, which has access to Android kernel interfaces through the native chroot.

> [!NOTE]
> These sensors are specific to the Redmi Note 8T (Snapdragon 665) and the tested kernel. The thermal zone indices and sysfs paths may differ on other devices. Before adding a sensor, verify the path exists:
>
> ```sh
> cat /sys/class/power_supply/battery/capacity
> ```
>
> If the file is absent or unreadable, the sensor is not available on this device.

Edit `configuration.yaml` and add a `command_line:` block. If `command_line:` already exists in the file, append the new entries under the existing key; do not add the key twice.

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

In nano, press `Ctrl+O`, then Enter to save; press `Ctrl+X` to exit.

> [!IMPORTANT]
> **Full Home Assistant Restart Required:**
> Adding new `command_line` sensors or changing existing sensor unique IDs requires a **full Home Assistant restart** (**Settings → System → Restart** or restarting the service). Reloading YAML configuration alone is insufficient for newly added entities to appear in Home Assistant.

Validate the configuration before restarting:

```sh
hass --script check_config --config "$HA_CONFIG_DIR"
```

After a successful check, restart Home Assistant. The sensors appear under **Settings → Devices & services → Entities** with the names defined above.

> [!NOTE]
> The dynamic thermal zone lookup (`grep -l … | grep -oE '[0-9]+'`) resolves the correct zone index at every poll. If two thermal zones match the same type string on a different kernel, the command returns multiple lines and the sensor reports an error. In that case, identify the correct fixed index with `grep -r 'cpuss-0-usr' /sys/class/thermal/` and replace the dynamic command with a hardcoded path such as `cat /sys/class/thermal/thermal_zone4/temp`.

### Optional: Advanced system and resource monitoring

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

## Logs

Use [step 11](11_Update_cleanup.md) for the Home Assistant update and backup procedure. The main logs are:

```sh
# Home Assistant log in Debian (uses the common working directory):
tail -f "$HA_CONFIG_DIR/home-assistant.log"

# In Termux after installing the step 06 script:
tail -f ~/init.log

# Only when Mosquitto is enabled:
tail -f ~/.local/share/mosquitto/mosquitto.log
```
