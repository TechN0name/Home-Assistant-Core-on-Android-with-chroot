# Home Assistant on Android with a native chroot: 11 — updates and cleanup

This step gathers the recurring update and cleanup guidance that was previously spread across steps 02–08 and `Notes.md`. It applies to the project’s Termux host, Debian chroot, Home Assistant and ESPHome environments.

> [!IMPORTANT]
> This project uses the Home Assistant Core Python-environment method. Home Assistant ended production support for this installation method with release 2025.12. It can still be updated, but there is no official support or guaranteed recovery path. Read the [2025.12 release note](https://www.home-assistant.io/blog/2025/12/03/release-202512/) for the support change and the [original deprecation announcement](https://www.home-assistant.io/blog/2025/05/22/deprecating-core-and-supervised-installation-methods-and-32-bit-systems/) before proceeding.

Run each command in the environment named in its subsection. Update one layer at a time; do not combine an Android/ROM change with Debian, Home Assistant or ESPHome package updates.

## 1. Update procedure

### Before an update

1. Choose one component to update and read its release notes. For Home Assistant, select an exact release and confirm that it supports the Python version already used by `/srv/homeassistant` in the [current development requirements](https://developers.home-assistant.io/docs/development_environment/).
2. Check free space before downloading packages or changing the Recorder database. Home Assistant documents that a database operation or schema change can need at least as much temporary free space as the database itself. Keep additional headroom when possible; check both the configuration filesystem and `/srv/homeassistant`.
3. Make a dated backup of the relevant configuration and record the installed package versions. Keep a copy of important backups off the phone. A package list is a recovery aid, not a substitute for a configuration/database backup.
4. If step 06 is running, pause the WatchDog before intentionally stopping Home Assistant. The step 09 widget is optional; its pause button is available only if you installed it. Without the widget, create the maintenance flag from regular Termux with `touch ~/.ha-maintenance`. Wait for `Maintenance mode ON; checks are paused.` in `~/init.log` before continuing. The flag takes effect at the end of the current check cycle (60 seconds by default). It pauses watchdog checks; it does not stop Home Assistant or `00-init.sh`.
5. If an update or validation fails, leave Home Assistant stopped and maintenance mode enabled. Keep the command output and backups, then diagnose the failure before restarting.

The step 06 maintenance flag expires after 180 minutes by default. If an operation may take longer, refresh it from regular Termux before it expires with `touch ~/.ha-maintenance`, then confirm the maintenance log entry again. A reboot clears the flag.

Home Assistant’s current Python version must remain compatible with the selected release. Do not try to change the Python version underneath an existing virtual environment. If a release requires a different Python major/minor version, stop here and plan a separate virtual-environment migration with its own backup and recovery steps.

### Termux packages

Run these commands in the regular Termux shell, not in Debian and not as root:

```sh
pkg update
pkg upgrade
```

Read the proposed transaction before accepting it; omit `-y`. Termux is a rolling-release environment, so use the complete `pkg upgrade` operation instead of upgrading isolated packages. It can update `sshd` and interrupt an SSH session. For a remote update, be ready to reconnect or use the local Termux screen. Keep Termux and its plug-ins from the same signing source as described in [step 02](02_Android_Config_Termux.md).

After the upgrade, check that Termux opens, `su -c id` works, and SSH reconnects. Do not combine this with an Android ROM, Magisk, or `chroot-distro` module update.

### Debian packages

Before entering Debian, run these checks in regular Termux (the Android host). The first command lists the distribution identifiers known to `chroot-distro` and their download/install status; it does not show Debian's release number. The second reads the current Debian stable release from Debian's official repository metadata. It may differ from the version in a `chroot-distro` rootfs image.

```sh
su -c '/system/bin/chroot-distro list'
curl -fsSL https://deb.debian.org/debian/dists/stable/Release |
  grep -E '^(Suite|Codename|Version):'
```

If Debian is already installed, check its actual release from Termux, then enter the installed chroot as root:

```sh
su -c '/system/bin/chroot-distro command debian "cat /etc/os-release; cat /etc/debian_version"'
su -c '/system/bin/chroot-distro login debian'
```

If Home Assistant is running, use the WatchDog pause above and stop HA as shown in the next subsection before changing Debian libraries. Run the commands below inside Debian.

```sh
apt update
apt list --upgradable
apt upgrade
dpkg --audit
```

Review the package plan and keep the normal `apt upgrade` behavior. Do not use `full-upgrade`/`dist-upgrade` as a routine update operation: those operations can remove packages to resolve dependency changes. A Debian release upgrade is a separate migration and is not covered by this procedure. After package updates, verify the chroot and Home Assistant manually or reboot the phone if step 06 is configured.

### Home Assistant Core

The steps below assume the existing `/srv/homeassistant` virtual environment, the `UV_LINK_MODE=copy` setting and the `HA_CONFIG_DIR` chosen in [step 04](04_Home_Assistant.md). Run the Debian commands as root.

First set the active configuration path: use `/mnt/ha_native` when the SD-card bind mount is active, or `/root/.homeassistant` for internal storage. Check space and record the current versions and configuration state:

```sh
export HA_CONFIG_DIR="/mnt/ha_native" # Or: /root/.homeassistant
STAMP="$(date +%Y%m%d-%H%M%S)"
umask 077

df -h "$HA_CONFIG_DIR" /srv/homeassistant
if [ -f "$HA_CONFIG_DIR/home-assistant_v2.db" ]; then
    du -h "$HA_CONFIG_DIR/home-assistant_v2.db"
fi

source /srv/homeassistant/bin/activate
export UV_LINK_MODE=copy
hass --version
/srv/homeassistant/bin/python --version
uv --version
uv pip check
hass --script check_config --config "$HA_CONFIG_DIR"
```

After the WatchDog reports maintenance mode ON, stop Home Assistant and confirm the process has exited:

```sh
pkill -TERM -f '[s]rv/homeassistant/bin/hass' || true
sleep 5
pgrep -af '[s]rv/homeassistant/bin/hass' || echo "Home Assistant is stopped."
```

If `pgrep` prints a Home Assistant process, stop here and wait or diagnose why it did not exit. Do not archive a live SQLite database. Once the process is stopped, create the dated snapshots:

```sh
tar -C "$HA_CONFIG_DIR" -czf "/root/ha-config-${STAMP}.tar.gz" .
uv pip freeze > "/root/homeassistant-requirements-before-${STAMP}.txt"
cp /srv/homeassistant/constraints.txt "/root/homeassistant-constraints-before-${STAMP}.txt"
chmod 600 "/root/ha-config-${STAMP}.tar.gz" \
          "/root/homeassistant-requirements-before-${STAMP}.txt" \
          "/root/homeassistant-constraints-before-${STAMP}.txt"
```

Copy the configuration archive off the phone before a major or risky upgrade. For an SD-card setup, the archive uses the active Debian bind mount; do not substitute Android’s host path `/mnt/media_rw/...`.

Choose the exact target version from the [Home Assistant releases](https://github.com/home-assistant/core/releases), replace the placeholder, and fetch that release’s constraints file **before** installing it:

```sh
TARGET_HA_VERSION="YYYY.MM.PATCH" # Replace with the exact release, for example 2026.11.2
NEW_CONSTRAINTS="/srv/homeassistant/constraints-${TARGET_HA_VERSION}.txt"

rm -f "${NEW_CONSTRAINTS}.tmp"
curl -fL \
  "https://raw.githubusercontent.com/home-assistant/core/${TARGET_HA_VERSION}/homeassistant/package_constraints.txt" \
  -o "${NEW_CONSTRAINTS}.tmp"
test -s "${NEW_CONSTRAINTS}.tmp" || { echo "Could not download the constraints file; stop here."; exit 1; }
mv "${NEW_CONSTRAINTS}.tmp" "$NEW_CONSTRAINTS" || { echo "Could not install the constraints file; stop here."; exit 1; }

export UV_CONSTRAINT="$NEW_CONSTRAINTS"
uv pip install "homeassistant==${TARGET_HA_VERSION}"
uv pip check
hass --version
hass --script check_config --config "$HA_CONFIG_DIR"
```

Proceed only if the install succeeds, `uv pip check` reports no broken requirements, `hass --version` shows the selected version, and `check_config` succeeds. Then make the release’s constraints the active file used by step 06:

```sh
cp "$NEW_CONSTRAINTS" /srv/homeassistant/constraints.txt.new
chmod 644 /srv/homeassistant/constraints.txt.new
mv /srv/homeassistant/constraints.txt.new /srv/homeassistant/constraints.txt
export UV_CONSTRAINT=/srv/homeassistant/constraints.txt
```

If any install or check fails, do not resume the WatchDog or reboot into the failed environment. Keep the snapshots, constraints files and error output. The saved `uv pip freeze` list can help reconstruct the previous package set, but restoring HA’s database/configuration may also be necessary if the new version was allowed to start.

After a successful update, the preferred project workflow is to reboot Android normally. Step 06 clears a leftover maintenance flag during boot and starts the services again. Confirm `~/init.log` reports successful startup and open the HA web interface. If not rebooting, start HA manually and verify it responds before removing the maintenance flag; resuming the WatchDog while HA is intentionally stopped makes it wait for HA.

### ESPHome Device Builder

Stop Builder with the `esphome-stop` button from [step 09](09_Termux_Widget.md), if you installed the optional widget. Otherwise, stop it from a Debian root shell and verify that the process has exited:

```sh
pkill -TERM -f '[e]sphome-device-builder' || true
sleep 2
pgrep -af '[e]sphome-device-builder' || echo "ESPHome Device Builder is stopped."
```

Then back up its YAML configuration and record the current package set before updating:

```sh
STAMP="$(date +%Y%m%d-%H%M%S)"
umask 077
tar -C /srv/esphome/config -czf "/root/esphome-config-${STAMP}.tar.gz" .
source /srv/esphome/bin/activate
uv pip freeze > "/root/esphome-requirements-before-${STAMP}.txt"
chmod 600 "/root/esphome-config-${STAMP}.tar.gz" \
          "/root/esphome-requirements-before-${STAMP}.txt"

uv pip install --upgrade "esphome-device-builder[esphome]"
uv pip check
esphome-device-builder --help
```

Copy the configuration archive off the phone if it contains the only copy of device YAML. After validation, start Builder from the widget and verify its authenticated web interface before flashing firmware.

Updating Builder and updating firmware on ESP devices are separate operations. Device firmware updates usually use OTA after the first USB flash; follow the device’s own update flow and keep source YAML in `/srv/esphome/config/`.

### Android, ROM, Magisk and chroot-distro

These layers are device- and ROM-specific. Follow the exact device instructions in [step 01](01_Device_Preparation.md) and [step 02](02_Android_Config_Termux.md), with a full device backup and a tested recovery path. Do not combine these changes with Debian, Home Assistant or ESPHome updates. After each reboot, verify root access, `chroot-distro list`, SSH and `~/init.log` before moving to another layer.

## 2. Cleanup procedure

Cleanup should free only known caches or data that the owning service can safely regenerate. First inspect usage; do not run a broad `rm -rf` against `/tmp`, `/root/.cache`, the chroot, or the Home Assistant configuration.

### Inspect space

In regular Termux:

```sh
df -h "$HOME" "$PREFIX"
du -sh "$HOME" "$PREFIX/var/cache/apt/archives" 2>/dev/null
```

In Debian, set `HA_CONFIG_DIR` as in step 04 and inspect the relevant filesystems and directories:

```sh
df -h / "$HA_CONFIG_DIR" /srv/homeassistant
if [ -d /srv/esphome ]; then
    df -h /srv/esphome
fi
du -sh /var/cache/apt/archives /root/.cache/uv \
  /srv/homeassistant "$HA_CONFIG_DIR" 2>/dev/null
if [ -d /srv/esphome ]; then
    du -sh /srv/esphome
fi
```

### Package and Python caches

In Termux, `pkg autoclean` removes outdated package archives; `pkg clean` removes all downloaded package archives. Neither command uninstalls packages. Use `pkg clean` only when you need the extra space because package files will need to be downloaded again.

In Debian:

```sh
apt autoclean
apt clean
apt-get -s autoremove
```

`apt clean` removes downloaded `.deb` archives, not installed packages. The last command is only a simulation. Review its full removal list; if it proposes removing something you recognize or need, stop. If the list contains only unneeded automatically installed dependencies and you agree, run `apt autoremove` interactively. Do not add `-y` or `--purge`.

For the shared `uv` cache, inspect its location and size, then prune only unused cache entries:

```sh
uv cache dir
du -sh "$(uv cache dir)"
uv cache prune
du -sh "$(uv cache dir)"
```

This project creates its active virtual environments explicitly at `/srv/homeassistant` and `/srv/esphome`; do not remove either environment as a cache cleanup. `uv cache prune` can also remove unused centralized uv project environments, so review its output if you use those elsewhere. Prefer it to `uv cache clean`, and never delete files inside the cache by hand.

### Downloaded `chroot-distro` rootfs

Run these commands in regular Termux, not inside Debian. `Downloaded: Yes` and `Installed: Yes` are separate states: the downloaded rootfs archive is only needed to install or reinstall the distribution. To free its space while keeping the installed Debian chroot, check the status, delete only the archive, then check again:

```sh
# Confirm Debian shows Downloaded: Yes and Installed: Yes
su -c '/system/bin/chroot-distro list'

# Delete only the downloaded Debian rootfs archive
su -c '/system/bin/chroot-distro delete debian'

# Confirm Debian now shows Downloaded: No and Installed: Yes
su -c '/system/bin/chroot-distro list'
```

The installed Debian remains usable. If you later need to install it again, download the rootfs again first. Do not use `chroot-distro remove debian` for this cleanup: `remove` deletes the downloaded archive, backup and installed distro directory. See the [`chroot-distro` command reference](https://github.com/Magisk-Modules-Alt-Repo/chroot-distro/blob/main/README.md).

### Home Assistant history and logs

Step 04 sets `recorder.purge_keep_days: 14`. Recorder automatically purges older history and, with its defaults enabled, periodically repacks the database. Purging old history alone may not shrink the SQLite file. Do not delete `home-assistant_v2.db`, edit it with SQLite tools while HA is running, or remove `.storage` to reclaim space.

To request an immediate purge and repack, use Home Assistant’s Recorder: Purge Recorder database action with:

```yaml
action: recorder.purge
data:
  keep_days: 14
  repack: true
```

Repacking is a heavy operation that can slow HA and temporarily needs extra disk space. Make a backup first and ensure free space is at least the database size; postpone the operation if space is tight. See the official [Recorder documentation](https://www.home-assistant.io/integrations/recorder/) and [purge action](https://www.home-assistant.io/actions/recorder.purge/).

The step 06 script rotates `~/init.log` above 1 MiB and retains `init.log.1`. Inspect logs with `tail`; do not truncate a log while its service is writing to it. The Builder log at `/root/esphome.log` is managed by its start script.

### Old backups

List the dated backups, verify that the newest successful archive can be read, and copy important data off the phone:

```sh
find /root -maxdepth 1 -type f \( \
  -name 'ha-config-*.tar.gz' -o \
  -name 'homeassistant-requirements-before-*.txt' -o \
  -name 'homeassistant-constraints-before-*.txt' -o \
  -name 'esphome-config-*.tar.gz' -o \
  -name 'esphome-requirements-before-*.txt' \
\) -exec ls -lh -- {} +

# Replace the timestamp with the exact archive name from the listing.
tar -tzf "/root/ha-config-YYYYMMDD-HHMMSS.tar.gz" >/dev/null
```

Keep the latest known-good configuration archive and package inventory until the next update has been verified. Remove older files only by their exact names, after checking them; do not use a wildcard deletion.

Do not remove `/srv/homeassistant`, `/srv/esphome`, `/root/.homeassistant`, `/mnt/ha_native`, active Termux boot scripts, or files inside the chroot as routine cleanup.

## References

- [Termux package management](https://github.com/termux/termux-packages/wiki/Package-Management)
- [Debian APT manual](https://manpages.debian.org/stable/apt/apt-get.8.en.html)
- [uv constraints and pip commands](https://docs.astral.sh/uv/reference/cli/#uv-pip-install)
- [uv cache management](https://docs.astral.sh/uv/concepts/cache/)
- [Home Assistant releases](https://github.com/home-assistant/core/releases) and versioned `homeassistant/package_constraints.txt` in the corresponding release tag
- [Home Assistant Recorder](https://www.home-assistant.io/integrations/recorder/)
