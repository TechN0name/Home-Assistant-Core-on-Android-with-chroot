# Home Assistant on Android with a native chroot: 06 — Automatic startup and watchdog

Before this step, Home Assistant must have started successfully at least once in [step 04](04_Home_Assistant.md); for the SD-card option, the bind mount must also be verified. Otherwise, repeated attempts only mask a startup error. Review the **User settings** block before permanent use.

Termux:Boot runs files from `~/.termux/boot/` alphabetically after a reboot, provided that its app has been opened at least once. This behavior is documented in the [official Termux:Boot README](https://github.com/termux/termux-boot/blob/master/README.md).

Keep only scripts that should run automatically in this directory. Termux:Boot can execute backup copies there too, even if their filenames have suffixes such as `.bak`. Store backups outside the boot directory, for example in `~/boot-backups/`.

## 1. Create the script

In regular Termux (not in Debian or a `su` shell), create the boot directory:

```sh
mkdir -p ~/.termux/boot
```

Then open the initialization script in `nano`:

```sh
nano ~/.termux/boot/00-init.sh
```

Set the local configuration values. An SD UUID is needed only when `USE_SD_CARD=1`.

```sh
#!/data/data/com.termux/files/usr/bin/sh

# Initialize services in the native chroot through Termux:Boot using the global mount namespace.
#
# What this script does:
#   1. Starts the optional services (sshd, Mosquitto, Chrony) and prepares the storage.
#   2. Starts Home Assistant in the Debian chroot and waits until its web interface answers.
#   3. Runs the watchdog loop: checks HA over HTTP and (optionally) the Wi-Fi/LAN connection.
#      Network recovery restarts Wi-Fi; network-triggered phone reboots are disabled during staged
#      testing. HA HTTP failures can still reboot, limited by MAX_REBOOT_ATTEMPTS_PER_WINDOW.
#
# Maintenance (pauses all checks, nothing is restarted; ideal for Termux:Widget buttons):
#   touch ~/.ha-maintenance      # pause the watchdog
#   rm -f ~/.ha-maintenance      # resume; the script waits for HA to answer before it counts failures
# Stop this script completely (HA itself keeps running):
#   kill "$(cat ~/.00-init.pid)"

PATH="/data/data/com.termux/files/usr/bin:$PATH"
export PATH

# ---------- User settings ----------
SU_CMD="su -mm -c"  # Use the Master Mount Namespace to avoid mount isolation
CHROOT_BIN="/system/bin/chroot-distro"
# Fallback for Magisk installs where the module launcher exists but its /system/bin overlay is absent.
CHROOT_BIN_FALLBACK="/data/adb/modules/chroot-distro/system/bin/chroot-distro"
DISTRO="debian"

# 0 = HA directory in internal storage (/root/.homeassistant).
# 1 = HA directory on an ext4 SD card through a bind mount.
USE_SD_CARD=0
SD_UUID="" # Required only when USE_SD_CARD=1.

# Enable only after verifying Mosquitto in step 05.
ENABLE_MOSQUITTO=0
MOSQUITTO_CONFIG="$HOME/.config/mosquitto/mosquitto.conf"

# Set both Chrony switches to 0 to disable it completely. The health check can also be disabled
# independently while keeping the service running by setting ENABLE_CHRONY_CHECK=0.
# Chrony serves LAN clients on UDP port 123 and uses the phone clock as a fallback.
ENABLE_CHRONY=0
CHRONY_BIN="/data/data/com.termux/files/usr/bin/chronyd"
CHRONY_CONFIG="$HOME/.config/chrony/chrony.conf"
TIMEOUT_BIN="/data/data/com.termux/files/usr/bin/timeout"
# Optional health check: set to 1 only when ENABLE_CHRONY=1 and step 07 is configured.
# Runs once at startup, then every 15 minutes. Checks the daemon and local chronyc control interface
# (not LAN client reachability), with a 5-second command timeout. Failures are logged and may trigger
# a Chrony start attempt, but never reboot HA/the phone.
ENABLE_CHRONY_CHECK=0
CHRONY_CHECK_INTERVAL=900 # seconds between checks after the initial startup check
CHRONYC_BIN="/data/data/com.termux/files/usr/bin/chronyc"
# Set by the optional Chrony check below; state changes are logged once to avoid log spam.
CHRONY_CHECK_STATE="unknown"
CHRONY_LAST_CHECK=0

# Enable only when SSH is protected with keys or a password and is actually needed.
ENABLE_SSHD=1

# --- Home Assistant availability check ---
# The check is a real HTTP request, not a process lookup. Any HTTP answer with a status below 500
# (200, redirects, 401, 403, 404 ...) proves that the HA web server is alive; no answer, a timeout
# or a 5xx status means it is not. List several URLs so that the check survives a change of port or
# SSL (the port may be omitted when HA listens on 80/443). The URL that answered last is remembered
# and tried first. Self-signed certificates are accepted. Make sure nothing else listens on these
# ports. If you set "server_host" in the HA http: section, add that address here, for example
# http://192.168.1.5:8123/ .
HA_URLS="http://127.0.0.1:8123/ https://127.0.0.1:8123/ http://127.0.0.1/ https://127.0.0.1/"

# 1 = run curl as Android root through $SU_CMD (the setup that worked so far).
# 0 = run curl as the normal Termux user (no su needed); switch only after the test commands pass.
HEALTH_CHECK_VIA_SU=1
# While HEALTH_CHECK_VIA_SU=1, also send the same request as the Termux user and log state changes.
# Test this with the screen off and on battery before switching to user-only mode.
# Keep HEALTH_CHECK_VIA_SU=1 until a sustained test passes.
HEALTH_CHECK_SHADOW=1
# Under su the PATH has no Termux directory, so root uses the Android system curl.
CURL_SU_BIN="/system/bin/curl"
# Used when HEALTH_CHECK_VIA_SU=0.
CURL_BIN="/data/data/com.termux/files/usr/bin/curl"

# Regular expression that matches only the real hass process (the [h] trick stops pgrep from matching
# itself). Used to avoid a second HA instance and to notice a crash during startup.
# Check it with:  su -c "ps -A -o PID,ARGS" | grep '[h]ass'
HA_PROC_PATTERN='/srv/homeassistant/bin/[h]ass'

# --- Watchdog timing ---
# After launching HA, poll every STARTUP_POLL_INTERVAL seconds for up to STARTUP_TIMEOUT seconds.
STARTUP_TIMEOUT=900
STARTUP_POLL_INTERVAL=10
# Afterwards check every CHECK_INTERVAL seconds; reboot after MAX_CONSECUTIVE_FAILURES failed checks in a row.
CHECK_INTERVAL=60
MAX_CONSECUTIVE_FAILURES=5

# --- Reboot limiter ---
# At most MAX_REBOOT_ATTEMPTS_PER_WINDOW reboot-command attempts per REBOOT_WINDOW_SECONDS.
# The attempt is recorded before calling Android: reboot may interrupt the shell before it can write.
# Failed command attempts count too, deliberately limiting repeated requests if Android rejects reboot.
MAX_REBOOT_ATTEMPTS_PER_WINDOW=3
REBOOT_WINDOW_SECONDS=3600
REBOOT_ATTEMPTS_LOG="$HOME/.ha-reboot-attempts"

# --- Maintenance mode ---
# While this file exists, the watchdog does nothing. It is removed at boot, and a flag older than
# MAINTENANCE_MAX_MINUTES is ignored so that a forgotten flag cannot disable the watchdog forever
# (0 = never expires).
MAINT_FLAG="$HOME/.ha-maintenance"
MAINTENANCE_MAX_MINUTES=180

# --- Wi-Fi / LAN check ---
# Recovery order: restart Wi-Fi first. A phone reboot for a network outage is controlled separately
# by NETWORK_REBOOT_ENABLED. HA failures are not counted while Wi-Fi recovery is in progress.
ENABLE_NETWORK_CHECK=1
WIFI_IFACE="wlan0"
# What to ping: "" = the default gateway of WIFI_IFACE (detected automatically), an IP/host name =
# that address (for example the router: 192.168.1.1), "none" = only check that WIFI_IFACE has an IPv4
# address (use it if your router does not answer ping).
NET_CHECK_HOST=""
NET_MAX_FAILURES=3          # failed checks in a row before the first Wi-Fi restart
WIFI_RESTART_ATTEMPTS=2     # Wi-Fi restarts before the phone is rebooted
NET_RECOVERY_WAIT=60        # seconds to wait for the network after each Wi-Fi restart
# Keep 0 during staged network-recovery testing: the script may restart Wi-Fi, but a network outage
# alone will not reboot the phone. HA health failures can still request a phone reboot.
NETWORK_REBOOT_ENABLED=0

# --- Logging ---
LOG_FILE="$HOME/init.log"
LOG_MAX_BYTES=1048576       # init.log is rotated to init.log.1 when it grows above this size
PID_FILE="$HOME/.00-init.pid"

# ---------- Internal state (do not edit) ----------
HA_URL_ACTIVE=""            # HA URL that answered last
FAILURES=0                  # failed HA checks in a row
NET_FAILURES=0              # failed network checks in a row
WIFI_RESTARTS=0             # Wi-Fi restarts done during the current network outage
NET_ARMED=0                 # 1 after the network check has succeeded once since start
NET_GW=""                   # cached gateway address
NET_GW_WARNED=0
NET_GIVEUP_LOGGED=0
SHADOW_STATE=""              # result of the last shadow check: ok / bad
IN_MAINTENANCE=0
REBOOT_REASON=""            # set by the checks when a phone reboot is wanted
REBOOT_BLOCK_LOGGED=0

# ---------- Helpers ----------
log() {
    printf '%s %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$*" >> "$LOG_FILE"
}

# Sleep that a TERM signal (kill) interrupts at once; a plain foreground sleep would delay it.
nap() {
    sleep "$1" &
    wait "$!"
}

# Run one command inside Debian as Android root through the Master Namespace.
chroot_run() {
    if ! $SU_CMD "test -x '$CHROOT_BIN'"; then
        if $SU_CMD "test -x '$CHROOT_BIN_FALLBACK'"; then
            CHROOT_BIN="$CHROOT_BIN_FALLBACK"
        else
            return 127
        fi
    fi
    $SU_CMD "$CHROOT_BIN command $DISTRO \"$1\""
}

# True when Android root works at all (used to tell "su is broken" from "HA/network is down").
su_works() {
    $SU_CMD "true" >/dev/null 2>&1
}

# Rotate init.log when it is too large. The file is truncated in place (not renamed), so the
# running HA process, which appends to it, keeps writing to the same file.
trim_log() {
    [ -f "$LOG_FILE" ] || return 0
    LOG_SIZE=$(stat -c %s "$LOG_FILE" 2>/dev/null) || return 0
    if [ "$LOG_SIZE" -gt "$LOG_MAX_BYTES" ]; then
        cp "$LOG_FILE" "$LOG_FILE.1" 2>/dev/null
        : > "$LOG_FILE"
        log "Log rotated; the previous content is in $LOG_FILE.1"
    fi
}

# Allow only one copy of this script. A stale PID file (after a reboot or a crash) is ignored.
acquire_lock() {
    if [ -f "$PID_FILE" ]; then
        OLD_PID=$(cat "$PID_FILE" 2>/dev/null)
        if [ -n "$OLD_PID" ] && [ "$OLD_PID" != "$$" ] && kill -0 "$OLD_PID" 2>/dev/null &&
           tr '\0' ' ' < "/proc/$OLD_PID/cmdline" 2>/dev/null | grep -q '00-init.sh'; then
            log "Another 00-init.sh instance is already running (pid $OLD_PID); exiting."
            return 1
        fi
    fi
    echo "$$" > "$PID_FILE"
}

# True while the maintenance flag exists and has not expired.
maintenance_active() {
    [ -e "$MAINT_FLAG" ] || return 1
    [ "$MAINTENANCE_MAX_MINUTES" -gt 0 ] || return 0
    FLAG_TIME=$(stat -c %Y "$MAINT_FLAG" 2>/dev/null) || return 0
    FLAG_AGE=$(( ($(date +%s) - FLAG_TIME) / 60 ))
    if [ "$FLAG_AGE" -ge "$MAINTENANCE_MAX_MINUTES" ]; then
        log "Maintenance flag is older than ${MAINTENANCE_MAX_MINUTES} min; ignoring and removing it."
        rm -f "$MAINT_FLAG"
        return 1
    fi
    return 0
}

# ---------- Reboot with limiter ----------
# Counts reboot-command attempts, not confirmed device restarts.
recent_reboot_attempts() {
    [ -f "$REBOOT_ATTEMPTS_LOG" ] || { echo 0; return 0; }
    awk -v now="$1" -v win="$REBOOT_WINDOW_SECONDS" '$1 > now - win { n++ } END { print n + 0 }' "$REBOOT_ATTEMPTS_LOG"
}

# Persist the attempt before calling Android; the phone may reboot before the shell can write again.
# Attempts whose reboot commands fail also count, so repeated failures cannot bypass the limiter.
record_reboot_attempt() {
    {
        [ -f "$REBOOT_ATTEMPTS_LOG" ] && awk -v now="$1" -v win="$REBOOT_WINDOW_SECONDS" '$1 > now - win' "$REBOOT_ATTEMPTS_LOG"
        echo "$1"
    } > "$REBOOT_ATTEMPTS_LOG.tmp" 2>/dev/null && mv "$REBOOT_ATTEMPTS_LOG.tmp" "$REBOOT_ATTEMPTS_LOG"
}

# Reboot the phone unless the attempt limiter forbids it. Returns 0 when Android accepts a command.
reboot_phone() {
    REASON="$1"
    NOW=$(date +%s)
    COUNT=$(recent_reboot_attempts "$NOW")
    if [ "$COUNT" -ge "$MAX_REBOOT_ATTEMPTS_PER_WINDOW" ]; then
        if [ "$REBOOT_BLOCK_LOGGED" -eq 0 ]; then
            log "Reboot suppressed ($REASON): $COUNT reboot attempt(s) already recorded within ${REBOOT_WINDOW_SECONDS}s. Fix the cause manually; monitoring continues."
            REBOOT_BLOCK_LOGGED=1
        fi
        return 1
    fi

    if ! record_reboot_attempt "$NOW"; then
        log "Cannot update $REBOOT_ATTEMPTS_LOG; automatic reboot cancelled so the attempt limiter remains effective."
        return 1
    fi
    log "Rebooting the phone: $REASON"
    sync
    if $SU_CMD 'setprop sys.powerctl reboot' >> "$LOG_FILE" 2>&1 || $SU_CMD 'svc power reboot' >> "$LOG_FILE" 2>&1 || $SU_CMD reboot >> "$LOG_FILE" 2>&1; then
        return 0
    fi
    log "Phone reboot command failed; the watchdog will retry after the next check."
    return 1
}

# ---------- Home Assistant checks ----------
# Print the HTTP status code of one URL. "000" = no connection; empty output = curl/su could not run.
http_status() {
    if [ "$HEALTH_CHECK_VIA_SU" -eq 1 ]; then
        $SU_CMD "'$CURL_SU_BIN' -k -s -o /dev/null -w '%{http_code}' --connect-timeout 5 --max-time 10 '$1'" 2>/dev/null
    else
        "$CURL_BIN" -k -s -o /dev/null -w '%{http_code}' --connect-timeout 5 --max-time 10 "$1" 2>/dev/null
    fi
}

# Return 0 = HA answers, 1 = HA does not answer, 2 = the check itself could not run (not an HA failure).
ha_healthy() {
    TRIED=" "
    CHECK_RAN=0
    for URL in $HA_URL_ACTIVE $HA_URLS; do
        case "$TRIED" in *" $URL "*) continue ;; esac
        TRIED="$TRIED$URL "
        CODE=$(http_status "$URL")
        [ -n "$CODE" ] && CHECK_RAN=1
        case "$CODE" in
            [1-4][0-9][0-9])
                if [ "$HA_URL_ACTIVE" != "$URL" ]; then
                    log "HA answers at $URL (HTTP $CODE); this URL is used for the checks."
                    HA_URL_ACTIVE="$URL"
                fi
                return 0
                ;;
        esac
    done
    [ "$CHECK_RAN" -eq 1 ] || return 2
    return 1
}

# Shadow check: the same request as the Termux user, only logged, never used for decisions.
shadow_check() {
    [ "$HEALTH_CHECK_VIA_SU" -eq 1 ] && [ "$HEALTH_CHECK_SHADOW" -eq 1 ] || return 0
    SH_CODE=$("$CURL_BIN" -k -s -o /dev/null -w '%{http_code}' --connect-timeout 5 --max-time 10 "$HA_URL_ACTIVE" 2>/dev/null)
    SH_RC=$?
    case "$SH_CODE" in
        [1-4][0-9][0-9])
            if [ "$SHADOW_STATE" != "ok" ]; then
                log "SHADOW: curl without su works too (HTTP $SH_CODE); su is probably not needed."
                SHADOW_STATE="ok"
            fi
            ;;
        *)
            if [ "$SHADOW_STATE" != "bad" ]; then
                log "SHADOW: curl without su FAILED (code [$SH_CODE], rc=$SH_RC) while the su check passed; keep HEALTH_CHECK_VIA_SU=1."
                SHADOW_STATE="bad"
            fi
            ;;
    esac
}

ha_process_running() {
    $SU_CMD "pgrep -f '$HA_PROC_PATTERN'" >/dev/null 2>&1
}

# Wait until HA answers. Gives up after STARTUP_TIMEOUT seconds, or earlier when a hass process that
# was seen running disappears (crash). A missing process is only a warning, because a wrong
# HA_PROC_PATTERN must not shorten the wait.
wait_for_ha() {
    WAITED=0
    SEEN=0
    WARNED=0
    CHECK_WARNED=0
    while [ "$WAITED" -lt "$STARTUP_TIMEOUT" ]; do
        ha_healthy
        HA_RC=$?
        if [ "$HA_RC" -eq 0 ]; then
            log "HA is up and responding (waited ${WAITED}s)."
            shadow_check
            if [ "$SEEN" -eq 0 ] && ! ha_process_running; then
                log "WARNING: HA answers, but HA_PROC_PATTERN matches no process; fix the pattern (see the settings comment)."
            fi
            return 0
        fi
        if [ "$HA_RC" -eq 2 ] && [ "$CHECK_WARNED" -eq 0 ]; then
            log "WARNING: the HTTP check cannot run (curl or su problem); see HEALTH_CHECK_VIA_SU, CURL_SU_BIN and CURL_BIN."
            CHECK_WARNED=1
        fi

        if ha_process_running; then
            SEEN=1
        elif [ "$SEEN" -eq 1 ]; then
            log "The hass process exited during startup; see the traceback above in $LOG_FILE."
            return 1
        elif [ "$WAITED" -ge 60 ] && [ "$WARNED" -eq 0 ]; then
            log "WARNING: no hass process found ${WAITED}s after launch. Check $LOG_FILE and HA_PROC_PATTERN."
            WARNED=1
        fi

        nap "$STARTUP_POLL_INTERVAL"
        WAITED=$((WAITED + STARTUP_POLL_INTERVAL))
    done
    log "HA did not respond within ${STARTUP_TIMEOUT}s after launch; the watchdog takes over."
    return 1
}

# True while a network outage is being handled by Wi-Fi restarts.
network_recovering() {
    [ "$ENABLE_NETWORK_CHECK" -eq 1 ] && [ "$NET_FAILURES" -gt 0 ] && [ "$WIFI_RESTARTS" -lt "$WIFI_RESTART_ATTEMPTS" ]
}

# One watchdog step for HA: updates FAILURES and sets REBOOT_REASON when the limit is reached.
ha_cycle() {
    ha_healthy
    HA_RC=$?

    if [ "$HA_RC" -eq 0 ]; then
        if [ "$FAILURES" -ne 0 ]; then
            log "HA is responding again; failure counter reset."
        fi
        FAILURES=0
        shadow_check
        return 0
    fi

    if [ "$HA_RC" -eq 2 ]; then
        log "The HTTP check could not run (curl or su problem); not counted as an HA failure."
        return 0
    fi

    if network_recovering; then
        FAILURES=0
        log "HA check failed while the network is being recovered; not counted."
        return 0
    fi

    FAILURES=$((FAILURES + 1))
    # Past the limit (reboot suppressed by the limiter) log only every 10th failure to keep the log short.
    if [ "$FAILURES" -le "$MAX_CONSECUTIVE_FAILURES" ] || [ $((FAILURES % 10)) -eq 0 ]; then
        log "HA did not respond to the HTTP check: $FAILURES/$MAX_CONSECUTIVE_FAILURES."
    fi
    if [ "$FAILURES" -ge "$MAX_CONSECUTIVE_FAILURES" ]; then
        REBOOT_REASON="HA failed $FAILURES consecutive HTTP checks"
    fi
}

# ---------- Wi-Fi / LAN checks ----------
# Print the IPv4 default gateway of WIFI_IFACE (Android keeps default routes in per-interface tables).
detect_gateway() {
    $SU_CMD "ip -4 route show table all" 2>/dev/null | awk -v dev="$WIFI_IFACE" '
        $1 == "default" {
            gw = ""; ifc = ""
            for (i = 2; i < NF; i++) {
                if ($i == "via") gw = $(i + 1)
                if ($i == "dev") ifc = $(i + 1)
            }
            if (ifc == dev && gw != "") { print gw; exit }
        }'
}

ping_host() {
    $SU_CMD "ping -c 2 -W 3 '$1'" >/dev/null 2>&1
}

wifi_has_ip() {
    $SU_CMD "ip -4 addr show dev '$WIFI_IFACE'" 2>/dev/null | grep -q 'inet '
}

# Return 0 when the Wi-Fi connection works. Checks the router (L2/L3 reachability) because HA on
# loopback keeps answering even when the phone has lost Wi-Fi.
network_ok() {
    if [ "$NET_CHECK_HOST" != "none" ]; then
        TARGET="$NET_CHECK_HOST"
        [ -n "$TARGET" ] || TARGET="$NET_GW"
        if [ -z "$TARGET" ]; then
            NET_GW=$(detect_gateway)
            TARGET="$NET_GW"
        fi
        if [ -n "$TARGET" ]; then
            ping_host "$TARGET" && return 0
            # DHCP may have given the router a new address: detect it again once before giving up.
            if [ -z "$NET_CHECK_HOST" ]; then
                NEW_GW=$(detect_gateway)
                if [ -n "$NEW_GW" ] && [ "$NEW_GW" != "$TARGET" ]; then
                    NET_GW="$NEW_GW"
                    ping_host "$NET_GW" && return 0
                fi
            fi
            return 1
        fi
        # No gateway known: fall back to the interface check below.
        if [ "$NET_GW_WARNED" -eq 0 ]; then
            log "WARNING: no default gateway found for $WIFI_IFACE; only the interface address is checked. Set NET_CHECK_HOST to the router address to ping it."
            NET_GW_WARNED=1
        fi
    fi
    wifi_has_ip
}

# Turn Wi-Fi off and on, then wait for the network. Returns 0 if the network came back.
restart_wifi() {
    log "Network is down; restarting Wi-Fi ($1)."
    $SU_CMD 'svc wifi disable' >> "$LOG_FILE" 2>&1
    nap 5
    $SU_CMD 'svc wifi enable' >> "$LOG_FILE" 2>&1
    WIFI_WAITED=0
    while [ "$WIFI_WAITED" -lt "$NET_RECOVERY_WAIT" ]; do
        nap 5
        WIFI_WAITED=$((WIFI_WAITED + 5))
        if network_ok; then
            log "Network is back after the Wi-Fi restart (${WIFI_WAITED}s)."
            NET_FAILURES=0
            WIFI_RESTARTS=0
            return 0
        fi
    done
    log "Network is still down after the Wi-Fi restart."
    return 1
}

# One watchdog step for the network: updates counters, restarts Wi-Fi, and sets REBOOT_REASON only
# when the Wi-Fi restarts did not help.
network_cycle() {
    [ "$ENABLE_NETWORK_CHECK" -eq 1 ] || return 0

    if network_ok; then
        if [ "$NET_FAILURES" -gt 0 ]; then
            log "Network is OK again."
        fi
        if [ "$NET_ARMED" -eq 0 ]; then
            NET_ARMED=1
            if [ "$NETWORK_REBOOT_ENABLED" -eq 1 ]; then
                log "Network check works; network-triggered phone reboot is armed."
            else
                log "Network check works; network-triggered phone reboot remains disabled."
            fi
        fi
        NET_FAILURES=0
        WIFI_RESTARTS=0
        NET_GIVEUP_LOGGED=0
        return 0
    fi

    if ! su_works; then
        log "su is not responding; the network check is skipped."
        return 0
    fi

    NET_FAILURES=$((NET_FAILURES + 1))
    # Once the recovery steps are used up, log only every 10th failure to keep the log short.
    if [ "$NET_FAILURES" -le $((NET_MAX_FAILURES + WIFI_RESTART_ATTEMPTS + 1)) ] || [ $((NET_FAILURES % 10)) -eq 0 ]; then
        log "Network check failed ($NET_FAILURES in a row)."
    fi
    [ "$NET_FAILURES" -ge "$NET_MAX_FAILURES" ] || return 0

    if [ "$WIFI_RESTARTS" -lt "$WIFI_RESTART_ATTEMPTS" ]; then
        WIFI_RESTARTS=$((WIFI_RESTARTS + 1))
        restart_wifi "$WIFI_RESTARTS/$WIFI_RESTART_ATTEMPTS"
        return 0
    fi

    if [ "$NETWORK_REBOOT_ENABLED" -eq 1 ] && [ "$NET_ARMED" -eq 1 ]; then
        REBOOT_REASON="network is still down after $WIFI_RESTARTS Wi-Fi restart(s)"
    elif [ "$NET_GIVEUP_LOGGED" -eq 0 ]; then
        log "Network is still down after $WIFI_RESTARTS Wi-Fi restart(s); no reboot (disabled, or the network check never succeeded since start)."
        NET_GIVEUP_LOGGED=1
    fi
}

# ---------- Services ----------
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

chrony_running() {
    $SU_CMD "pgrep -f '[c]hronyd.*chrony.conf'" >/dev/null 2>&1
}

start_chrony() {
    [ "$ENABLE_CHRONY" -eq 1 ] || return 0

    if [ ! -x "$CHRONY_BIN" ] || [ ! -f "$CHRONY_CONFIG" ]; then
        log "Chrony is enabled, but the daemon or configuration file was not found. Complete step 07 first."
        return 1
    fi

    if chrony_running; then
        return 0
    fi

    # Run Chrony as Android root for UDP port 123; -x prevents system clock adjustments.
    # The command is started in the background, so its exit status says nothing about chronyd:
    # check that the process really exists a moment later.
    $SU_CMD "nohup '$CHRONY_BIN' -x -f '$CHRONY_CONFIG' >/dev/null 2>&1 </dev/null &"
    sleep 2
    if chrony_running; then
        log "Chrony started."
    else
        log "Chrony did not stay running; check $CHRONY_CONFIG. Home Assistant will continue without the NTP service."
        return 1
    fi
}

chrony_health_check() {
    [ "$ENABLE_CHRONY_CHECK" -eq 1 ] || return 0
    [ "$ENABLE_CHRONY" -eq 1 ] || return 0

    CHRONY_NOW=$(date +%s)
    if [ "$CHRONY_LAST_CHECK" -gt 0 ] &&
       [ $((CHRONY_NOW - CHRONY_LAST_CHECK)) -lt "$CHRONY_CHECK_INTERVAL" ]; then
        return 0
    fi
    CHRONY_LAST_CHECK="$CHRONY_NOW"

    if [ ! -x "$CHRONY_BIN" ] || [ ! -f "$CHRONY_CONFIG" ] || [ ! -x "$CHRONYC_BIN" ] || [ ! -x "$TIMEOUT_BIN" ]; then
        if [ "$CHRONY_CHECK_STATE" != "bad" ]; then
            log "Chrony check is enabled, but a required binary or configuration file is missing."
        fi
        CHRONY_CHECK_STATE="bad"
        return 1
    fi

    if ! chrony_running; then
        if [ "$CHRONY_CHECK_STATE" != "missing" ]; then
            log "Chrony daemon is not running; attempting to start it."
        fi
        CHRONY_CHECK_STATE="missing"
        start_chrony || return 1
    fi

    if $SU_CMD "'$TIMEOUT_BIN' 5 '$CHRONYC_BIN' -h 127.0.0.1 tracking" >/dev/null 2>&1; then
        if [ "$CHRONY_CHECK_STATE" != "ok" ]; then
            log "Chrony control check is responding."
        fi
        CHRONY_CHECK_STATE="ok"
        return 0
    fi

    if [ "$CHRONY_CHECK_STATE" != "bad" ]; then
        log "Chrony control check failed; no phone reboot will be requested."
    fi
    CHRONY_CHECK_STATE="bad"
    return 1
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
        log "The SD card or ha_config did not appear within 150 seconds. HA will not start to avoid creating an empty configuration in internal storage."
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

start_ha() {
    # Checked first: a chroot-distro command is never run while HA is already running.
    if ha_process_running; then
        log "A hass process is already running; not starting a second instance."
        return 0
    fi

    if ! chroot_run "test -s /srv/homeassistant/constraints.txt" >/dev/null 2>&1; then
        # Tell a broken chroot/su apart from a missing file: the message points to the right fix.
        if chroot_run "true" >/dev/null 2>&1; then
            log "/srv/homeassistant/constraints.txt was not found. Restore the uv file; HA startup cancelled."
        else
            log "Cannot run commands in the $DISTRO chroot (su denied, chroot-distro failed, or the rootfs is unavailable); HA startup cancelled."
        fi
        return 1
    fi

    log "Starting Home Assistant from $HA_CONFIG_IN_CHROOT (activate)"
    chroot_run "
        export UV_LINK_MODE=copy &&
        export UV_CONSTRAINT=/srv/homeassistant/constraints.txt &&
        cd /root &&
        . /srv/homeassistant/bin/activate &&
        exec hass --config $HA_CONFIG_IN_CHROOT
    " >> "$LOG_FILE" 2>&1 &
    return 0
}

# ---------- Main ----------
acquire_lock || exit 0
trap 'rm -f "$PID_FILE"' EXIT
trap 'exit 1' HUP INT TERM
# Acquire the singleton lock before rotating the shared log so duplicate boot launches cannot race.
trim_log

# A maintenance flag must not survive a reboot.
if [ -e "$MAINT_FLAG" ]; then
    rm -f "$MAINT_FLAG"
    log "Removed a leftover maintenance flag."
fi

termux-wake-lock >> "$LOG_FILE" 2>&1 || log "Failed to acquire the Termux wake lock."
log "===== 00-init.sh start ====="

if [ "$ENABLE_SSHD" -eq 1 ] && ! pgrep -x sshd >/dev/null 2>&1; then
    sshd >> "$LOG_FILE" 2>&1 || log "Failed to start sshd."
fi

prepare_storage || exit 1
start_mosquitto || true
start_chrony || true
chrony_health_check || true

# Give Android time to finish boot-time initialization before entering the chroot.
log "Waiting 10 seconds before starting the chroot environment..."
sleep 10

start_ha || exit 1

log "Waiting for HA to answer (up to ${STARTUP_TIMEOUT} s)."
wait_for_ha

log "Watchdog started: check every ${CHECK_INTERVAL} s, reboot after ${MAX_CONSECUTIVE_FAILURES} failed HA checks in a row."
while :; do
    if maintenance_active; then
        if [ "$IN_MAINTENANCE" -eq 0 ]; then
            log "Maintenance mode ON; checks are paused."
            IN_MAINTENANCE=1
        fi
        FAILURES=0
        NET_FAILURES=0
        WIFI_RESTARTS=0
        nap 10
        continue
    fi

    if [ "$IN_MAINTENANCE" -eq 1 ]; then
        IN_MAINTENANCE=0
        log "Maintenance mode OFF; waiting for HA before the checks resume."
        wait_for_ha
    fi

    trim_log
    start_mosquitto || true
    chrony_health_check || true

    REBOOT_REASON=""
    network_cycle
    ha_cycle

    if [ -n "$REBOOT_REASON" ]; then
        if reboot_phone "$REBOOT_REASON"; then
            exit 0
        fi
    else
        REBOOT_BLOCK_LOGGED=0
    fi

    nap "$CHECK_INTERVAL"
done
```

In nano, `Ctrl+O`, then Enter saves; `Ctrl+X` exits. Set its permissions and validate syntax without starting services:

```sh
chmod 700 ~/.termux/boot/00-init.sh
sh -n ~/.termux/boot/00-init.sh
```

## 2. Select the correct configuration

### Without an SD card

Before saving the script, use:

```sh
USE_SD_CARD=0
SD_UUID=""
```

In this mode, the script does not check `/mnt/media_rw`, does not perform `mount --bind`, and starts HA from `/root/.homeassistant`, even when no SD card is inserted. Do not add SD-card settings “just in case”.

### With an SD card

Only after step 04 has formatted the card, Android mounts it automatically, and the manual bind mount is verified, use:

```sh
USE_SD_CARD=1
SD_UUID="the-actual-uuid"
```

The script waits up to 150 seconds for `/mnt/media_rw/<UUID>/ha_config`. If the card is not ready, HA intentionally does not start; this prevents an accidental launch with a new empty internal-storage directory.

### Mosquitto

If step 05 was skipped, keep:

```sh
ENABLE_MOSQUITTO=0
```

After the broker has been manually verified, `~/.config/mosquitto/mosquitto.conf` exists, and HA has connected to it, set the value to `1`. The script attempts to start Mosquitto during boot and health checks; MQTT unavailability by itself does not reboot the phone.

### Chrony

If step 07 was skipped, keep both switches disabled:

```sh
ENABLE_CHRONY=0
ENABLE_CHRONY_CHECK=0
```

After Chrony has been configured and verified, set both values to `1` to start and monitor it. The script checks Chrony once at startup and then every 15 minutes. Each `chronyc tracking` request is limited to five seconds. Set `ENABLE_CHRONY_CHECK=0` to keep the service running without periodic checks; set both switches to `0` to disable Chrony in the startup script.

The health check verifies the daemon and its local control interface. It does not test whether another device can reach UDP port 123. From Windows PowerShell on the same LAN, test the NTP service with:

```powershell
w32tm /stripchart /computer:<phone_lan_ip> /dataonly /samples:5
```

Replace `<phone_lan_ip>` with the phone's address. Successful samples confirm the LAN NTP response; the displayed offset is the difference between the Windows clock and the phone's NTP time.

## 3. First test and reboot

Before rebooting, review the settings once more, especially the UUID. After the reboot, allow up to 15 minutes for storage mounting and HA startup, then connect over SSH and check:

```sh
tail -n 100 ~/init.log
su -c 'curl --head http://127.0.0.1:8123/'
```

With an SD card, additionally check:

```sh
su -c 'mount | grep ha_native'
su -c '/system/bin/chroot-distro command debian "ls -la /mnt/ha_native"'
```

### Startup log

The script writes its status messages and Home Assistant startup output to `~/init.log`.

## Watchdog behavior and limits

The script below documents the tested startup logic and uses safe defaults for a new setup:

- **Storage:** `USE_SD_CARD=0` selects `/root/.homeassistant` and skips SD-card checks and bind mounts. This works when no card is inserted. With `USE_SD_CARD=1`, the script waits up to 150 seconds for the UUID-specific `ha_config` directory and cancels startup if the card or directory is missing; it never silently falls back to a new empty internal configuration.
- **Home Assistant startup:** The process check prevents a second `hass` instance and helps detect a crash during startup. The HTTP check remains the health decision. Startup polling can continue for up to 900 seconds.
- **HTTP watchdog:** It checks every 60 seconds by default. Any HTTP status from 100 through 499 counts as an answering web server; connection errors and 5xx responses count as failures. After five consecutive HA failures, the script can reboot the phone. A limiter records each reboot attempt before issuing the command and allows at most three attempts per hour; failed reboot commands count too.
- **Wi-Fi recovery:** After three network-check failures, the script restarts Wi-Fi, up to two attempts, and waits up to 60 seconds after each attempt. `NETWORK_REBOOT_ENABLED=0` disables a reboot triggered directly by the network check. HA HTTP failures can still request a reboot after their own threshold.
- **Chrony:** The daemon starts at boot when enabled. Its optional health check runs at startup and every 900 seconds, with a five-second timeout. Check the LAN NTP response separately from a client as shown above.
- **Maintenance and logs:** `touch ~/.ha-maintenance` pauses watchdog checks; `rm ~/.ha-maintenance` resumes them. Startup status and HA console output go to `~/init.log`, which rotates above 1 MiB. Use [step 11](11_Update_cleanup.md) for the full update and cleanup procedures.
- **Termux:Boot:** Keep only scripts intended to start automatically in `~/.termux/boot/`. Move backups outside that folder; filename suffixes do not prevent Termux:Boot from running an executable backup.

To temporarily disable automatic startup, run:

```sh
mv ~/.termux/boot/00-init.sh ~/00-init.sh.bak
```

To restore it, run:

```sh
mv ~/00-init.sh.bak ~/.termux/boot/00-init.sh
```
