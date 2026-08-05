# Home Assistant on Android with a native chroot: 02 — Android, Termux, and chroot

This step prepares the Android host and Debian `chroot`. Home Assistant, the SD card, Mosquitto, and automatic startup are **not** configured yet.

## 1. Magisk modules

After Android has booted successfully, we open Magisk and grant Termux root access when requested.

We install each module and reboot the phone afterwards:

1. **BusyBox for Android NDK** — the BusyBox recommended by `chroot-distro`.
2. **chroot-distro** from the [module repository](https://github.com/Magisk-Modules-Alt-Repo/chroot-distro/releases).

`chroot-distro` v2.x stores distributions in `/data/local/chroot-distro/` and requires root. We do not substitute `proot-distro`: it is a different mechanism with different paths and limitations. Commands and requirements are documented in the [module documentation](https://magisk.dev/modules/chroot-distro/).

## 2. Install Termux and Termux:Boot

We install **Termux** and **Termux:Boot** from the same signing source—for example, both from F-Droid or both from the official GitHub releases. We do not mix F-Droid and GitHub APKs: their signing keys differ, which can prevent plug-ins from working.

We open **Termux:Boot** once and then close it. This is required before Android can deliver boot events to the app. Later, scripts are stored in `~/.termux/boot/`; the official workflow is in the [Termux:Boot README](https://github.com/termux/termux-boot/blob/master/README.md).

In Android system settings, we:

- allow Termux and Termux:Boot to run in the background;
- disable battery optimization or battery restrictions for both apps;
- allow network access for Termux;
- if available, prevent the system from aggressively closing the apps;
- set a 70% charging limit to reduce battery wear from continuous maximum voltage.

Modern LineageOS builds provide this under **Battery → Charging Control**. If the ROM lacks it, we use the Magisk ACC module.

This does not guarantee that Android will never kill a process under memory pressure, but it substantially reduces the likelihood.

## 3. Basic Termux environment

We open Termux as the regular user and run:

```sh
pkg update && pkg upgrade -y
pkg install -y openssh tmux nano curl wget git iproute2 procps
termux-wake-lock
```

`termux-wake-lock` keeps the device awake while Termux holds the wake lock. It increases battery use, so a permanent server benefits from stable power and temperature monitoring.

We check the phone’s local IP address:

```sh
su -c 'ip -brief address'
```

For PC access, set a user password interactively first:

```sh
passwd
```

After entering and confirming the password, start the SSH server:

```sh
sshd
```

Termux OpenSSH usually listens on port `8022`. From the PC, we connect with:

```sh
ssh -p 8022 <termux_user>@<phone_IP>
```

After confirming access, we replace password authentication with an SSH key. Port 8022 is not exposed to the Internet.

## 4. Verify root and the module

In Termux, we check root and the module with one-off commands:

```sh
su -c id
su -c '/system/bin/chroot-distro help'
su -c '/system/bin/chroot-distro env'
su -c '/system/bin/chroot-distro list'
```

`id` should report `uid=0(root)`. If Magisk requests root access, we grant it to Termux. If `chroot-distro` is not in `/system/bin`, we obtain its actual path with `command -v chroot-distro` in a root shell and use that path throughout the guide.

## 5. Install Debian

We run the remaining commands in this section from Termux through `su -c`:

```sh
su -c '/system/bin/chroot-distro download debian'
su -c '/system/bin/chroot-distro install debian'
```

After installation, we run a non-interactive diagnostic:

```sh
su -c '/system/bin/chroot-distro command debian "id; cat /etc/os-release; uname -m"'
```

The output should include `uid=0(root)`, Debian, and a 64-bit ARM architecture (`aarch64` or `arm64`). We then enable the required module options:

```sh
su -c '/system/bin/chroot-distro android-bind enable'
su -c '/system/bin/chroot-distro fix-suid enable'
su -c '/system/bin/chroot-distro ram-bind enable'
```

The command may report `enabled already`; this is a normal idempotent result. `android-bind` grants the chroot access to Android paths. It does not replace the separate SD-card bind mount in step 03.

For an optional interactive check, we enter Debian with:

```sh
su -c '/system/bin/chroot-distro login debian'
```

> [!NOTE]
> **Understanding Shell Contexts:**
> - **Regular Termux shell (`$` prompt):** Commands run as the unprivileged Termux user on Android.
> - **Termux root shell (`#` prompt via `su`):** Commands run as Android `root`.
> - **Debian chroot shell (`root@localhost:~#` prompt via `login debian`):** Commands run inside the Debian Linux environment as `root`.
>
> When editing files with `nano` in any shell: press `Ctrl+O`, then `Enter` to save changes; press `Ctrl+X` to exit the editor.

We leave the session with `exit`. In the rest of this guide, “in Debian” means this root session.

## Checkpoint

We continue to `03_Debian_Python_HA_Mo.md` when both checks work:

```sh
su -c id
su -c '/system/bin/chroot-distro command debian "id; cat /etc/os-release"'
```

If either check fails, we fix root or chroot before installing Python: later steps cannot work around a broken mount namespace.
