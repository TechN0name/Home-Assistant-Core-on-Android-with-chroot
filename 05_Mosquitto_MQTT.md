# Home Assistant on Android with a native chroot: 05 — Optional Mosquitto MQTT broker

Configure a local Mosquitto broker in Termux when a local MQTT service is required. This step is optional and can be completed before enabling the broker in [step 06](06_Automatic_Startup.md).

## Configure the optional local Mosquitto broker

Use this section only when a **local MQTT broker** is required. If HA connects to an existing network broker or MQTT is not used, skip it.

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

Enter and confirm the password when prompted. Then edit the configuration file:

```sh
nano ~/.config/mosquitto/mosquitto.conf
```

Paste this configuration to listen on the phone and its local network. Devices connected through the same router can reach the broker at the Android host's LAN address. The `0.0.0.0` listener is intentional; do not configure router port forwarding that exposes port 1883 to the Internet.

> **Security:** MQTT on port 1883 does not encrypt the username or password. Use this setup only on a trusted LAN. For access beyond that LAN, use a VPN or configure TLS.

```conf
persistence true
persistence_location /data/data/com.termux/files/home/.local/share/mosquitto/

log_dest file /data/data/com.termux/files/home/.local/share/mosquitto/mosquitto.log

# Listen on all IPv4 interfaces so LAN devices can connect; use 127.0.0.1 for phone-only access
listener 1883 0.0.0.0

# Disallow connections without a password
allow_anonymous false
password_file /data/data/com.termux/files/home/.config/mosquitto/passwd
```

For the first diagnostic, optionally run the broker in the foreground in **Termux session 1**:

```sh
mosquitto -c ~/.config/mosquitto/mosquitto.conf -v
```

Keep the broker running. In **Termux session 2**, start a subscriber; `-C 1` makes it wait for one message:

```sh
mosquitto_sub -h 127.0.0.1 -p 1883 -u homeassistant -P '<password>' -t test/topic -C 1
```

While the subscriber is waiting, publish the test message from **Termux session 3**:

```sh
mosquitto_pub -h 127.0.0.1 -p 1883 -u homeassistant -P '<password>' -t test/topic -m ok
```

After the subscriber receives `ok`, stop the broker with `Ctrl+C` in session 1. These diagnostic commands use loopback to test the broker on the phone.

In **Settings → Devices & services → Add integration → MQTT**, use:

```sh
Broker: 127.0.0.1
Port: 1883
Username / password: the credentials created above.
```

For Home Assistant on this phone, use `127.0.0.1` as shown. Clients on other LAN devices should use the Android host's LAN address (for example, `192.168.31.2`; your address may differ).

