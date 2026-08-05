# 06 — Termux:Widget: manual control

Termux:Widget adds home-screen buttons for short operations. In this project, it starts and stops **ESPHome Device Builder** and displays its log. The widget does **not** replace `00-init.sh` from [step 04](04_automation_script.md).

> [!NOTE]
> We run every command in this step in regular Termux, not in Debian or a root shell. `Termux`, `Termux:Boot`, and `Termux:Widget` must come from the same signing source.

## 1. Prepare Termux:Widget

We install and open Termux:Widget once. In Termux, we then create the foreground-script directory:

```sh
mkdir -p ~/.shortcuts
chmod 700 ~/.shortcuts
```

## 2. Add ESPHome buttons

The `chroot-distro` path must match step 02. If the path differs, we change only `CHROOT_BIN` in each script below.

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
        echo "Builder started: http://<phone_IP>:6052"
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

We grant execute permission to all three scripts:

```sh
chmod 700 ~/.shortcuts/*.sh
```

## 3. Add the widget

On the Android home screen, we hold an empty area → **Widgets** → **Termux:Widget** → drag the widget onto the screen. We tap the widget’s refresh button if new scripts do not appear immediately.
