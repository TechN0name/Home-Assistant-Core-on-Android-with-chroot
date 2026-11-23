# Home Assistant on Android with a native chroot: 07 — Chrony NTP server

Configure Chrony in Termux when LAN devices require a local Network Time Protocol (NTP) server, including during upstream Internet outages. Skip this optional step when external NTP servers are sufficient.

## 1. Install Chrony

In regular Termux, install the package and create restricted configuration, data, and runtime directories:

```sh
pkg install -y chrony
mkdir -p ~/.config/chrony ~/.local/share/chrony ~/.local/var/run/chrony
chmod 700 ~/.config/chrony ~/.local/share/chrony ~/.local/var/run/chrony
```

## 2. Configure the NTP server

Open the configuration file:

```sh
nano ~/.config/chrony/chrony.conf
```

The values `192.168.31.0/24` and `192.168.31.2` are examples and may differ on your network. Replace the subnet with the trusted LAN subnet and use the Android host's actual LAN address for the NTP server; select upstream pools appropriate for the deployment:

```conf
# Upstream NTP sources used while Internet access is available.
pool ua.pool.ntp.org iburst
pool pool.ntp.org iburst

# Store runtime and drift data in Termux-writable paths.
pidfile /data/data/com.termux/files/home/.local/var/run/chrony/chronyd.pid
bindcmdaddress /data/data/com.termux/files/home/.local/var/run/chrony/chronyd.sock
driftfile /data/data/com.termux/files/home/.local/share/chrony/drift

# Permit NTP clients only from the trusted local subnet.
allow 192.168.31.0/24

# Serve the phone clock as a fallback when upstream sources are unavailable.
local stratum 10

# NTP service port; root privileges are required.
port 123
```

Restrict the `allow` subnet to the local network. Do not expose UDP port 123 to the public Internet.

## 3. Run and verify Chrony

Android manages the hardware clock, and UDP port 123 is privileged. Start `chronyd` as Android root with `-x` to prevent system clock adjustments. For the initial diagnostic, run the daemon in the foreground:

```sh
su -c '/data/data/com.termux/files/usr/bin/chronyd -x -d -f /data/data/com.termux/files/home/.config/chrony/chrony.conf'
```

In a second Termux session, inspect Chrony status and query the NTP listener from a LAN client or a separate network device. Stop the foreground daemon with `Ctrl+C` after verification:

```sh
chronyc -h 127.0.0.1 tracking
chronyd -Q 'server <phone_lan_ip> iburst'
```

From Windows PowerShell on another device in the same LAN, verify the phone's UDP/123 response without changing the PC clock:

```powershell
w32tm /stripchart /computer:<phone_lan_ip> /dataonly /samples:5
```

Replace `<phone_lan_ip>` with the phone's LAN address. Successful samples confirm that the NTP server is reachable from that PC; the reported offset is the difference between the PC clock and the server time.

For a persistent background process, use the following command. [Step 06](06_Automatic_Startup.md) starts Chrony when `ENABLE_CHRONY=1` is set and runs its optional local health check when `ENABLE_CHRONY_CHECK=1` is also set (initially and every 15 minutes):

```sh
su -c '/data/data/com.termux/files/usr/bin/chronyd -x -f /data/data/com.termux/files/home/.config/chrony/chrony.conf'
```

## 4. Configure LAN clients

Set the Android host LAN address as the NTP server on client devices, such as OpenBeken under **Config → Configure General / Time**:

| Setting | Value |
|---|---|
| NTP server | Android host LAN IP address (for example, `192.168.31.2`) |
| Port | `123` (UDP) |

Keep the Android host on a stable LAN address or configure a DHCP reservation. Chrony provides the phone clock during upstream outages; time accuracy then depends on the phone clock.
