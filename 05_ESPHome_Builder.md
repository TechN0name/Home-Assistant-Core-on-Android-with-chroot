# 05 — ESPHome Device Builder in a Debian chroot

This step installs **[ESPHome Device Builder](https://github.com/esphome/device-builder/)** as a standalone service in the Debian chroot. It is not the classic `esphome dashboard`: this guide uses the `esphome-device-builder[esphome]` package and its command-line entry point.

> [!NOTE]
> This step requires [03 — Debian, Python, Home Assistant, and Mosquitto](03_Debian_Python_HA_Mo.md), but does not change the Home Assistant Python environment.

## 1. Create an isolated environment

We enter Debian:

```sh
su -c '/system/bin/chroot-distro login debian'
```

```sh
mkdir -p /srv/esphome/config
uv venv /srv/esphome
source /srv/esphome/bin/activate
uv pip install "esphome-device-builder[esphome]"
esphome-device-builder --help
```

If the help text appears without errors, we create `/root/start-esphome.sh` and perform the first test run.

## 2. Create the startup script

In Debian, we create the script:

```sh
nano /root/start-esphome.sh
```

We set a password and paste the following:

```sh
#!/bin/bash

CONFIG_DIR="/srv/esphome/config"

# Create the configuration directory if it does not exist
mkdir -p "$CONFIG_DIR"

export ESPHOME_USERNAME="admin"
export ESPHOME_PASSWORD="replace-with-your-password"

# Run directly through the environment binary
exec /srv/esphome/bin/esphome-device-builder "$CONFIG_DIR" --host 0.0.0.0 --port 6052
```

In nano, `Ctrl+O`, then Enter saves; `Ctrl+X` exits. We restrict access because the password is stored in plain text:

```sh
chmod 700 /root/start-esphome.sh
```

> [!IMPORTANT]
> Standalone Builder requires both `ESPHOME_USERNAME` and `ESPHOME_PASSWORD`. Environment variables, unlike `--password`, do not expose the password in the process list.

## 3. Start and verify

In Debian, we run:

```sh
/root/start-esphome.sh
```

After startup, Builder is available on the local network at `http://<phone_IP>:6052`. We stop the foreground process with `Ctrl+C` in Termux.

Builder does not need to run continuously. Step [06](06_Termux_Widget.md) provides manual buttons for starting, stopping, viewing logs, and clearing caches.

> [!WARNING]
> Standalone Builder is accessible from the LAN by default. We do not forward port `6052` to the WAN and use a strong password. We do not replace it with the classic `esphome dashboard`: it is a different interface and process.

## Updates and maintenance

Before updating, we stop Builder. In Debian, we run:

```sh
source /srv/esphome/bin/activate
uv pip install --upgrade "esphome-device-builder[esphome]"
esphome-device-builder --help
```

The first flash of a new ESP device requires a USB data cable. After a successful first installation, subsequent updates are usually sent over the air (OTA). We keep YAML files only in `/srv/esphome/config/`.
