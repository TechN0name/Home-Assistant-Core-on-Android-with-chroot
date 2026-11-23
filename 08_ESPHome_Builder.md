# 08 — ESPHome Device Builder in a Debian chroot

This step installs **[ESPHome Device Builder](https://github.com/esphome/device-builder/)** as a standalone service in the Debian chroot. It is not the classic `esphome dashboard`: this guide uses the `esphome-device-builder[esphome]` package and its command-line entry point.

> [!NOTE]
> This step requires [step 03](03_Debian_Python.md), but does not change the Home Assistant Python environment.

## 1. Create an isolated environment

In Termux, enter Debian as root:

```sh
su -c '/system/bin/chroot-distro login debian'
```

Inside the Debian chroot, create the environment and install ESPHome Device Builder:

```sh
mkdir -p /srv/esphome/config
uv venv /srv/esphome
source /srv/esphome/bin/activate
uv pip install "esphome-device-builder[esphome]"
esphome-device-builder --help
```

If the help text appears without errors, create `/root/start-esphome.sh` and perform the first test run.

## 2. Create the startup script

In Debian, open `nano` to create the startup script:

```sh
nano /root/start-esphome.sh
```

Set a password and paste the following:

```sh
#!/bin/bash

CONFIG_DIR="/srv/esphome/config"

# Create the configuration directory if it does not exist
mkdir -p "$CONFIG_DIR"

export ESPHOME_USERNAME="admin"
export ESPHOME_PASSWORD="replace-with-the-password"

# Run directly through the environment binary
exec /srv/esphome/bin/esphome-device-builder "$CONFIG_DIR" --host 0.0.0.0 --port 6052
```

In nano, `Ctrl+O`, then Enter saves; `Ctrl+X` exits. Restrict access because the password is stored in plain text:

```sh
chmod 700 /root/start-esphome.sh
```

> [!IMPORTANT]
> Standalone Builder requires both `ESPHOME_USERNAME` and `ESPHOME_PASSWORD`. Environment variables, unlike `--password`, do not expose the password in the process list.

## 3. Start and verify

In Debian, run the startup script in the foreground to verify functionality:

```sh
/root/start-esphome.sh
```

After startup, Builder is available on the local network at `http://<phone_ip>:6052`. After verifying access in a browser, stop the foreground process by pressing `Ctrl+C` in the terminal.

Builder does not need to run continuously. Step [09](09_Termux_Widget.md) provides manual buttons for starting, stopping, and viewing logs.

> [!WARNING]
> Standalone Builder is accessible from the LAN by default. Do not forward port `6052` to the WAN. Set a strong password. The classic `esphome dashboard` is a different interface and process.

## Updates and firmware

For Builder package updates and safe backups, follow [step 11](11_Update_cleanup.md). The first flash of a new ESP device requires a USB data cable; subsequent firmware updates are usually sent over the air (OTA). Keep YAML files only in `/srv/esphome/config/`.
