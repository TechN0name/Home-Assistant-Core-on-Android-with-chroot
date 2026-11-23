# 09 — Termux:Widget: manual control

Termux:Widget adds home-screen buttons for short operations. In this project, it provides start/stop/status controls for **ESPHome Device Builder** and pause/resume/status/log controls for the Home Assistant WatchDog. The WatchDog controls pause only its automatic checks; they do not stop Home Assistant or `00-init.sh`. The widget does **not** replace `00-init.sh` from [step 06](06_Automatic_Startup.md).

> [!NOTE]
> run every command in this step in regular Termux, not in Debian or a root shell. `Termux`, `Termux:Boot`, and `Termux:Widget` must come from the same signing source.

## 1. Prepare Termux:Widget

Install and open Termux:Widget once. In Termux, then create the foreground-script directory:

```sh
mkdir -p ~/.shortcuts
chmod 700 ~/.shortcuts
```

## 2. Add ESPHome buttons

Match the `chroot-distro` path used in step 02. If the path differs, change only `CHROOT_BIN` in each script below.

### Start ESPHome Device Builder

Open `nano` to create `~/.shortcuts/esphome-start.sh`:

```sh
nano ~/.shortcuts/esphome-start.sh
```

Paste the following script content:

```sh
#!/data/data/com.termux/files/usr/bin/sh
PATH="/data/data/com.termux/files/usr/bin:$PATH"
export PATH

SU_CMD="su -mm -c"
CHROOT_BIN="/system/bin/chroot-distro"
DISTRO="debian"

is_running() {
    $SU_CMD "$CHROOT_BIN command $DISTRO \"pgrep -f '[e]sphome-device-builder'\"" >/dev/null 2>&1
}

clear
echo "=== ESPHome Device Builder ==="

if is_running; then
    echo "Builder is already running."
else
    $SU_CMD "$CHROOT_BIN command $DISTRO \"cd /root && nohup /bin/bash /root/start-esphome.sh > /root/esphome.log 2>&1 &\""
    sleep 2

    if is_running; then
        echo "Builder started: http://<phone_ip>:6052"
        echo "Log: /root/esphome.log in Debian"
    else
        echo "Failed to start Builder. Error log:"
        echo "----------------------------------------"
        $SU_CMD "$CHROOT_BIN command $DISTRO \"cat /root/esphome.log 2>/dev/null\""
        echo "----------------------------------------"
    fi
fi

read -r -p "Press Enter to exit..." _
```

### Stop ESPHome Device Builder

Open `nano` to create `~/.shortcuts/esphome-stop.sh`:

```sh
nano ~/.shortcuts/esphome-stop.sh
```

Paste the following script content:

```sh
#!/data/data/com.termux/files/usr/bin/sh
PATH="/data/data/com.termux/files/usr/bin:$PATH"
export PATH

SU_CMD="su -mm -c"
CHROOT_BIN="/system/bin/chroot-distro"
DISTRO="debian"

clear
echo "=== Stopping ESPHome Device Builder ==="

if $SU_CMD "$CHROOT_BIN command $DISTRO \"pgrep -f '[e]sphome-device-builder'\"" >/dev/null 2>&1; then
    $SU_CMD "$CHROOT_BIN command $DISTRO \"pkill -TERM -f '[e]sphome-device-builder'\""
    echo "SIGTERM sent."
else
    echo "No active Builder found."
fi

read -r -p "Press Enter to exit..." _
```

### View status and log

Open `nano` to create `~/.shortcuts/esphome-status.sh`:

```sh
nano ~/.shortcuts/esphome-status.sh
```

Paste the following script content:

```sh
#!/data/data/com.termux/files/usr/bin/sh
PATH="/data/data/com.termux/files/usr/bin:$PATH"
export PATH

SU_CMD="su -mm -c"
CHROOT_BIN="/system/bin/chroot-distro"
DISTRO="debian"

clear
echo "=== ESPHome Device Builder: status ==="
$SU_CMD "$CHROOT_BIN command $DISTRO \"
    pgrep -af '[e]sphome-device-builder' || echo 'Builder is not running.'
    echo
    echo '--- Last 50 log lines ---'
    tail -n 50 /root/esphome.log 2>/dev/null || true
\""

read -r -p "Press Enter to exit..." _
```

### Pause or resume the Home Assistant WatchDog

Step [06](06_Automatic_Startup.md) already has a maintenance flag at `~/.ha-maintenance`. These buttons pause only the WatchDog checks; Home Assistant and `00-init.sh` keep running. In normal mode, the WatchDog reads the flag at the end of each check cycle (60 seconds by default); while paused, it checks every 10 seconds. The flag is cleared on reboot and expires after the configured `MAINTENANCE_MAX_MINUTES` (180 minutes by default). For the full update procedure, see [step 11](11_Update_cleanup.md).

Create `~/.shortcuts/watchdog-pause.sh`:

```sh
#!/data/data/com.termux/files/usr/bin/sh
MAINT_FLAG="$HOME/.ha-maintenance"
umask 077

echo "=== WatchDog maintenance: pause ==="
if [ -e "$MAINT_FLAG" ]; then
    echo "Pause is already requested."
elif : > "$MAINT_FLAG"; then
    echo "Pause requested; WatchDog pauses at the end of its current check cycle (60 seconds by default)."
    echo "Home Assistant and 00-init.sh keep running."
else
    echo "Could not create $MAINT_FLAG"
fi
printf 'Press Enter to exit...'
IFS= read -r _
```

Create `~/.shortcuts/watchdog-resume.sh`:

```sh
#!/data/data/com.termux/files/usr/bin/sh
MAINT_FLAG="$HOME/.ha-maintenance"

echo "=== WatchDog maintenance: resume ==="
if [ ! -e "$MAINT_FLAG" ]; then
    echo "WatchDog checks are already enabled."
elif rm -f "$MAINT_FLAG"; then
    echo "Resume requested; checks continue after the WatchDog confirms Home Assistant is responding."
else
    echo "Could not remove $MAINT_FLAG"
fi
printf 'Press Enter to exit...'
IFS= read -r _
```

Create `~/.shortcuts/watchdog-status.sh`:

```sh
#!/data/data/com.termux/files/usr/bin/sh
MAINT_FLAG="$HOME/.ha-maintenance"

echo "=== Home Assistant WatchDog status ==="
if [ -e "$MAINT_FLAG" ]; then
    echo "Maintenance flag: present; WatchDog pause is requested. Check the log below for confirmation."
    stat -c 'Pause requested: %y' "$MAINT_FLAG" 2>/dev/null || true
else
    echo "Maintenance flag: absent; WatchDog checks are enabled."
fi
echo
echo "--- Recent WatchDog maintenance log entries ---"
grep -E 'Watchdog started|Maintenance mode (ON|OFF)' "$HOME/init.log" 2>/dev/null | tail -n 8 || true
printf 'Press Enter to exit...'
IFS= read -r _
```

### Read the recent WatchDog and startup log

Create `~/.shortcuts/watchdog-log.sh`:

```sh
nano ~/.shortcuts/watchdog-log.sh
```

Paste this script:

```sh
#!/data/data/com.termux/files/usr/bin/sh
LOG_FILE="$HOME/init.log"

echo "=== 00-init.sh / WatchDog log: last 100 lines ==="
if [ -r "$LOG_FILE" ]; then
    tail -n 100 "$LOG_FILE"
else
    echo "Log not found or unreadable: $LOG_FILE"
fi
printf 'Press Enter to exit...'
IFS= read -r _
```

This button reads the Termux `~/init.log` file, which contains startup and WatchDog messages. The HA application log is separate and is under the active configuration directory described in [Notes](Notes.md).

Grant execute permission to all seven scripts:

```sh
chmod 700 ~/.shortcuts/*.sh
```

### Allow Termux:Widget to open terminal sessions

If tapping a foreground shortcut says Termux needs **Display over other apps**, grant the permission to **Termux** (package `com.termux`), not to Termux:Widget.

**Settings method:** If the “App was denied access” notice appears while you are trying the overlay switch, close it. Then open **Settings → Apps → Termux → ⋮ → Allow restricted settings** and confirm if Android asks. Return to **Special app access → Display over other apps → Termux** and enable the permission. Menu names vary by ROM. Android 16 classifies this overlay permission as restricted for apps marked as installed from a local APK. See [Android restricted settings](https://support.google.com/android/answer/12623953?hl=en) and the [Android 16 compatibility definition](https://source.android.com/docs/compatibility/16/android-16-cdd).

**Root fallback:** If Android does not offer or accept that Settings confirmation, run these commands in the regular Termux shell (not Debian). They set and then verify the overlay app-op for Termux only:

```sh
su -mm -c '/system/bin/appops set com.termux SYSTEM_ALERT_WINDOW allow'
su -mm -c '/system/bin/appops get com.termux SYSTEM_ALERT_WINDOW'
```

The expected verification output includes `SYSTEM_ALERT_WINDOW: allow`. This grants Termux the ability to draw over other apps so its widget can open terminal sessions; use it only for the trusted Termux installation.

## 3. Add the widget

On the Android home screen, hold an empty area → **Widgets** → **Termux:Widget** → drag the widget onto the screen. Tap the widget’s refresh button if new scripts do not appear immediately.
