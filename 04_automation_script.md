# Home Assistant on Android with a native chroot: 04 — Automatic startup and watchdog

Before this step, Home Assistant must have started successfully at least once in step 03; for the SD-card option, the bind mount must also be verified. Otherwise, the script only masks a startup error with repeated attempts. The timings and switches match the tested setup, so before permanent use we review the **User settings** block.

Termux:Boot runs files from `~/.termux/boot/` alphabetically after a reboot, provided that its app has been opened at least once. This behavior is documented in the [official Termux:Boot README](https://github.com/termux/termux-boot/blob/master/README.md).

## 1. Create the script

In regular Termux (not in Debian or a `su` shell), create the boot directory:

```sh
mkdir -p ~/.termux/boot
```

Then open the initialization script in `nano`:

```sh
nano ~/.termux/boot/00-init.sh
```

We set the values for the local configuration, including the actual **SD UUID and HA access token** if HA-log entries through the interface are needed.

```sh
#!/data/data/com.termux/files/usr/bin/sh

# Initialize services in the native chroot through Termux:Boot using the global mount namespace.

PATH="/data/data/com.termux/files/usr/bin:$PATH"
export PATH

# ---------- User settings ----------
SU_CMD="su -mm -c"  # Use the Master Mount Namespace to avoid mount isolation
CHROOT_BIN="/system/bin/chroot-distro"
DISTRO="debian"

# 0 = HA directory in internal storage (/root/.homeassistant).
# 1 = HA directory on an ext4 SD card through a bind mount.
USE_SD_CARD=0
SD_UUID="24b7606d-01b1-45ca-9202-126624675927" # Example card UUID; replace it with the actual value.

# Enable only after verifying Mosquitto in step 03.
ENABLE_MOSQUITTO=0
MOSQUITTO_CONFIG="$HOME/.config/mosquitto/mosquitto.conf"

# Enable only when SSH is protected with keys or a password and is actually needed.
ENABLE_SSHD=1

# The watchdog restarts only HA first.
CHECK_INTERVAL=300
MAX_CONSECUTIVE_FAILURES=2
INITIAL_GRACE_SECONDS=90
REBOOT_AFTER_RESTARTS=1
MAX_RESTARTS_BEFORE_REBOOT=2

# ---------- Logging settings ----------
# 0 = Save storage writes (write to HA; create $LOG_FILE only on an ERROR/failure).
# 1 = Duplicate all logs to the local $LOG_FILE.
ENABLE_LOCAL_LOG=1

# Home Assistant long-lived access token (Profile → Long-Lived Access Tokens); the current value is an example.
HA_TOKEN="eyJh1GciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJiYzViYW1kYzM2ZTg0ODBlOWFhZGE2ZmU5OGE4YzgwYiIsImlhdCI6MTc4NTA3MDc2NiwiZXhwIjoyMTAwNDMwNzY2fQ.iVakEzcD_LpTt7Ft8LQ7Kitip9dbFkn8JToZIUweCzk"
LOGGER_NAME="termux.init"
# -----------------------------------------------

LOG_FILE="$HOME/init.log"
HA_URL="http://127.0.0.1:8123/"
RESTART_COUNT=0

# Output destinations for standard system-command streams
if [ "$ENABLE_LOCAL_LOG" -eq 1 ]; then
    OUT_DEV="$LOG_FILE"
else
    OUT_DEV="/dev/null"
fi

log() {
    MSG="$*"
    TIMESTAMP="$(date '+%Y-%m-%d %H:%M:%S.000')"

    # Automatically determine the HA log level
    LEVEL="INFO"
    case "$MSG" in
        *"WARNING"*|*"Failed"*|*"did not respond"*|*"Waiting"*)
            LEVEL="WARNING"
            ;;
    esac
    case "$MSG" in
        *"ERROR"*|*"cancelled"*|*"Error"*|*"did not appear"*|*"exhausted"*)
            LEVEL="ERROR"
            ;;
    esac

    FORMATTED_ENTRY="$TIMESTAMP $LEVEL (MainThread) [$LOGGER_NAME] $MSG"

    # 1. Local init.log file (when ENABLE_LOCAL_LOG=1 OR an ERROR occurs)
    if [ "$ENABLE_LOCAL_LOG" -eq 1 ] || [ "$LEVEL" = "ERROR" ]; then
        printf '%s\n' "$FORMATTED_ENTRY" >> "$LOG_FILE"
    fi

    # 2. Write directly to home-assistant.log (when the HA configuration is mounted)
    if [ -n "$HA_CONFIG_IN_CHROOT" ]; then
        REAL_HA_LOG_PATH="/data/local/chroot-distro/$DISTRO${HA_CONFIG_IN_CHROOT}/home-assistant.log"
        $SU_CMD "test -d '$(dirname "$REAL_HA_LOG_PATH")' && printf '%s\n' '$FORMATTED_ENTRY' >> '$REAL_HA_LOG_PATH'" >/dev/null 2>&1 || true
    fi

    # 3. Send to the HA system log through the REST API (when HA runs and a token is set)
    if [ -n "$HA_TOKEN" ] && [ "$HA_TOKEN" != "PASTE_YOUR_HOME_ASSISTANT_TOKEN_HERE" ]; then
        API_LEVEL=$(echo "$LEVEL" | tr '[:upper:]' '[:lower:]')
        API_URL="${HA_URL%/}/api/services/system_log/write"
        $SU_CMD "curl -s -X POST \
            -H 'Authorization: Bearer $HA_TOKEN' \
            -H 'Content-Type: application/json' \
            -d '{\"level\": \"$API_LEVEL\", \"message\": \"$MSG\", \"logger\": \"$LOGGER_NAME\"}' \
            '$API_URL'" >/dev/null 2>&1 &
    fi
}

# Run one command inside Debian as Android root through the Master Namespace.
chroot_run() {
    $SU_CMD "$CHROOT_BIN command $DISTRO \"$1\""
}

start_mosquitto() {
    [ "$ENABLE_MOSQUITTO" -eq 1 ] || return 0

    if [ ! -f "$MOSQUITTO_CONFIG" ]; then
        log "Mosquitto is enabled, but the configuration was not found: $MOSQUITTO_CONFIG"
        return 1
    fi

    if pgrep -x mosquitto >/dev/null 2>&1; then
        return 0
    fi

    if mosquitto -c "$MOSQUITTO_CONFIG" -d; then
        log "Mosquitto started."
    else
        log "Failed to start Mosquitto; HA will continue without it."
        return 1
    fi
}

prepare_storage() {
    if [ "$USE_SD_CARD" -eq 0 ]; then
        HA_CONFIG_IN_CHROOT="/root/.homeassistant"
        return 0
    fi

    if [ -z "$SD_UUID" ]; then
        log "USE_SD_CARD=1, but SD_UUID is empty. Startup cancelled."
        return 1
    fi

    CARD_SOURCE="/mnt/media_rw/$SD_UUID/ha_config"
    CHROOT_TARGET="/data/local/chroot-distro/$DISTRO/mnt/ha_native"
    ATTEMPT=0

    # Android vold can mount the card with a delay after boot.
    while [ "$ATTEMPT" -lt 30 ]; do
        if $SU_CMD "test -d '$CARD_SOURCE'"; then
            break
        fi
        ATTEMPT=$((ATTEMPT + 1))
        log "Waiting for the SD card: attempt $ATTEMPT/30."
        sleep 2
    done

    if ! $SU_CMD "test -d '$CARD_SOURCE'"; then
        log "The SD card or ha_config did not appear within 60 seconds. HA will not start to avoid creating an empty configuration in internal storage."
        return 1
    fi

    if ! $SU_CMD "mkdir -p '$CHROOT_TARGET'"; then
        log "Failed to create the bind-mount target: $CHROOT_TARGET"
        return 1
    fi

    # Check for an active mount in the master namespace
    if ! $SU_CMD "grep -qs ' $CHROOT_TARGET ' /proc/mounts"; then
        if ! $SU_CMD "mount --bind '$CARD_SOURCE' '$CHROOT_TARGET'"; then
            log "Failed to bind-mount $CARD_SOURCE to $CHROOT_TARGET"
            return 1
        fi
    fi

    HA_CONFIG_IN_CHROOT="/mnt/ha_native"
    log "SD card is ready (Master Mount); HA configuration: $HA_CONFIG_IN_CHROOT"
    return 0
}

# CHECK: Look for a default gateway only on the Wi-Fi interface (wlan0)
is_network_up() {
    $SU_CMD "ip route show table all" | grep "default" | grep -q "wlan0"
}

# Wait for a network connection before starting HA
wait_for_network() {
    log "Waiting for the device to connect to the network..."
    NET_ATTEMPT=0
    while [ "$NET_ATTEMPT" -lt 30 ]; do
        if is_network_up; then
            log "Network connection is active. Continuing."
            return 0
        fi
        NET_ATTEMPT=$((NET_ATTEMPT + 1))
        sleep 2
    done
    log "WARNING: Network not found within 60 seconds. Starting HA in offline mode."
    return 1
}

ha_is_running() {
    # The [s]rv pattern avoids falsely matching the grep process
    chroot_run "pgrep -f '[s]rv/homeassistant/bin/hass' >/dev/null" >/dev/null 2>&1
}

start_ha() {
    if ha_is_running; then
        log "Home Assistant is already running."
        return 0
    fi

    if ! chroot_run "test -s /srv/homeassistant/constraints.txt" >/dev/null 2>&1; then
        log "/srv/homeassistant/constraints.txt was not found. Restore the uv file; HA startup cancelled."
        return 1
    fi

    log "Starting Home Assistant from $HA_CONFIG_IN_CHROOT (activate)"
    chroot_run "
        export UV_LINK_MODE=copy &&
        export UV_CONSTRAINT=/srv/homeassistant/constraints.txt &&
        cd /root &&
        . /srv/homeassistant/bin/activate &&
        exec hass --config $HA_CONFIG_IN_CHROOT
    " >> "$OUT_DEV" 2>&1 &
    return 0
}

stop_ha() {
    chroot_run "pkill -TERM -f '[s]rv/homeassistant/bin/hass' || true" >> "$OUT_DEV" 2>&1
}

ha_healthy() {
    # This check must run as Android root in the Master Namespace
    $SU_CMD "curl -fsS --connect-timeout 5 --max-time 10 '$HA_URL'" >/dev/null 2>&1
}

termux-wake-lock >> "$OUT_DEV" 2>&1 || log "Failed to acquire the Termux wake lock."
log "===== 00-init.sh start ====="

if [ "$ENABLE_SSHD" -eq 1 ] && ! pgrep -x sshd >/dev/null 2>&1; then
    sshd >> "$OUT_DEV" 2>&1 || log "Failed to start sshd."
fi

prepare_storage || exit 1
wait_for_network
start_mosquitto || true

# Give Android time to finish boot-time initialization before entering the chroot.
log "Waiting 10 seconds before starting the chroot environment..."
sleep 10

start_ha || exit 1

log "Waiting for initial HA startup: ${INITIAL_GRACE_SECONDS} s."
sleep "$INITIAL_GRACE_SECONDS"

FAILURES=0
while :; do
    if ! is_network_up; then
        log "Network unavailable (Wi-Fi is disabled). Skipping the HA health check to avoid a false reboot."
        FAILURES=0 
    else
        start_mosquitto || true

        if ha_healthy; then
            if [ "$FAILURES" -ne 0 ]; then
                log "HA is responding again; failure counter reset."
            fi
            FAILURES=0
        else
            FAILURES=$((FAILURES + 1))
            log "HA did not respond to the HTTP check: $FAILURES/$MAX_CONSECUTIVE_FAILURES."
        fi

        if [ "$FAILURES" -ge "$MAX_CONSECUTIVE_FAILURES" ]; then
            # Check whether the soft-restart limit is exhausted before making another attempt
            if [ "$REBOOT_AFTER_RESTARTS" -eq 1 ] && [ "$RESTART_COUNT" -ge "$MAX_RESTARTS_BEFORE_REBOOT" ]; then
                log "Soft-restart limit ($RESTART_COUNT) exhausted; phone reboot is enabled."
                $SU_CMD 'setprop sys.powerctl reboot' >> "$OUT_DEV" 2>&1 || $SU_CMD 'svc power reboot' >> "$OUT_DEV" 2>&1 || $SU_CMD reboot >> "$OUT_DEV" 2>&1
                exit 0
            fi

            RESTART_COUNT=$((RESTART_COUNT + 1))
            log "Restarting HA, attempt $RESTART_COUNT/$MAX_RESTARTS_BEFORE_REBOOT."
            stop_ha
            sleep 20
            start_ha
            FAILURES=0

            # Give HA time to initialize before resuming checks
            log "Waiting for HA to start after restart: ${INITIAL_GRACE_SECONDS} s."
            sleep "$INITIAL_GRACE_SECONDS"
        fi
    fi

    sleep "$CHECK_INTERVAL"
done
```

In nano, `Ctrl+O`, then Enter saves; `Ctrl+X` exits. We set its permissions and validate syntax without starting services:

```sh
chmod 700 ~/.termux/boot/00-init.sh
sh -n ~/.termux/boot/00-init.sh
```

## 2. Select the correct configuration

### Without an SD card

Before saving the script, we use:

```sh
USE_SD_CARD=0
SD_UUID=""
```

In this mode, the script does not check `/mnt/media_rw`, does not perform `mount --bind`, and starts HA from `/root/.homeassistant`. We do not add SD-card settings “just in case”.

### With an SD card

Only after step 03 has formatted the card, Android mounts it automatically, and the manual bind mount is verified, we use:

```sh
USE_SD_CARD=1
SD_UUID="your-actual-uuid"
```

The script waits up to 60 seconds for `/mnt/media_rw/<UUID>/ha_config`. If the card is not ready, HA intentionally does not start; this prevents an accidental launch with a new empty internal-storage directory.

### Mosquitto

If the Mosquitto section in step 03 was skipped, we keep:

```sh
ENABLE_MOSQUITTO=0
```

After the broker has been manually verified, `~/.config/mosquitto/mosquitto.conf` exists, and HA has connected to it, we set the value to `1`. The watchdog restarts only the broker; MQTT unavailability does not reboot the phone.

## 3. First test and reboot

Before rebooting, we review the settings once more, especially the UUID. Then we reboot the phone.

After 2–3 minutes, we connect over SSH and check:

```sh
tail -n 100 ~/init.log
su -c 'curl -I http://127.0.0.1:8123/'
```

With an SD card, we additionally check:

```sh
su -c 'mount | grep ha_native'
su -c '/system/bin/chroot-distro command debian "ls -la /mnt/ha_native"'
```

### Integrated logging

The initialization script implements multi-destination logging:
1. **Local init log:** Writes status and watchdog events to `~/init.log` when `ENABLE_LOCAL_LOG=1` or when an `ERROR` occurs.
2. **Home Assistant log file:** Directly appends formatted entries into `home-assistant.log` inside the active Home Assistant configuration directory (`/root/.homeassistant/home-assistant.log` or `/mnt/ha_native/home-assistant.log`).
3. **Home Assistant REST API:** When `HA_TOKEN` is configured, events are transmitted to Home Assistant via the `/api/services/system_log/write` REST endpoint.

Because of this integration, startup script messages and watchdog alerts appear directly inside the Home Assistant UI under **Settings → System → Logs**. This behavior is intentional and provides unified logging; do not disable or remove this integration.

## Watchdog behavior and limits

The watchdog loop operates according to the precise logic implemented in `00-init.sh`:

- **SD card detection & startup cancellation:** When `USE_SD_CARD=1`, the script waits up to 60 seconds (30 attempts at 2-second intervals) for the physical SD storage (`/mnt/media_rw/$SD_UUID/ha_config`). If the card or directory is not found within 60 seconds, startup is automatically cancelled to prevent Home Assistant from launching with a new empty configuration in internal storage (`/root/.homeassistant`).
- **Initial grace period:** Home Assistant is allowed an initial grace period of `INITIAL_GRACE_SECONDS` (90 seconds) to complete startup before the watchdog loop begins health checks.
- **Network loss & skipped health checks:** The script checks whether active Wi-Fi connectivity (`wlan0` default gateway route) is present. If the network interface is down during a watchdog check, health checks are skipped and the `FAILURES` counter is reset to `0` to prevent false positive restarts or reboots while offline.
- **HTTP health check & restart threshold:** Checks are performed at `CHECK_INTERVAL` (300 seconds by default) using an HTTP GET request to `http://127.0.0.1:8123/`. If Home Assistant fails `MAX_CONSECUTIVE_FAILURES` (2) consecutive HTTP checks, the script sends `SIGTERM` (`stop_ha`), waits 20 seconds, and restarts Home Assistant (`start_ha`), allowing another 90-second grace period.
- **Reboot threshold:** If `REBOOT_AFTER_RESTARTS=1` and Home Assistant fails again after `MAX_RESTARTS_BEFORE_REBOOT` (2) soft restarts, the script triggers an automatic system reboot (`setprop sys.powerctl reboot`). Disable system reboots only deliberately: an unhandled configuration error could otherwise result in a reboot loop.
- The script does not replace backups, Android updates, or power management. Android can still kill Termux under severe memory pressure. In practice, this was not reproduced with stress tests, including with `termux-wake-lock` disabled.

To temporarily disable automatic startup, we run:

```sh
mv ~/.termux/boot/00-init.sh ~/00-init.sh.bak
```

To restore it, we run:

```sh
mv ~/00-init.sh.bak ~/.termux/boot/00-init.sh
```
