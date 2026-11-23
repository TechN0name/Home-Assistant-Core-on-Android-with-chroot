# Home Assistant on Android with a native chroot: 03 — Debian and Python

Prepare Debian and install the isolated Python environment used by Home Assistant. Complete this step before configuring Home Assistant in [step 04](04_Home_Assistant.md).

## 1. Enter Debian and prepare the base system

In regular Termux, first check which distribution identifiers `chroot-distro` lists:

```sh
su -c '/system/bin/chroot-distro list'
```

This shows the distributions known to the installed module and their download/install status; it does not show a Debian release number. To check the current Debian stable release published by Debian, read the official repository metadata:

```sh
curl -fsSL https://deb.debian.org/debian/dists/stable/Release |
  grep -E '^(Suite|Codename|Version):'
```

This reports the current stable repository version, not necessarily the version in a `chroot-distro` rootfs image. If Debian is already installed, check the release in that chroot from Termux with:

```sh
su -c '/system/bin/chroot-distro command debian "cat /etc/os-release; cat /etc/debian_version"'
```

Then enter the installed Debian chroot as root:

```sh
su -c '/system/bin/chroot-distro login debian'
```

Until this subsection is complete, run every command **in Debian** as root:

```sh
# Refresh the package index before the initial package installation
apt update

# 2. Install base tools, build dependencies, media codecs, and C libraries
apt install -y \
  ca-certificates curl git nano tmux procps iproute2 sqlite3 \
  build-essential autoconf pkg-config \
  libssl-dev libxml2-dev libxslt1-dev libjpeg-dev libffi-dev libudev-dev zlib1g-dev \
  libavformat-dev libavcodec-dev libavdevice-dev libavutil-dev libswscale-dev \
  libswresample-dev libavfilter-dev ffmpeg \
  libturbojpeg0 libpcap-dev
```

For later Debian package updates, use [step 11](11_Update_cleanup.md). This command only refreshes the package index for the initial setup.

The library set follows the current Debian dependency list in the [Home Assistant development documentation](https://developers.home-assistant.io/docs/development_environment/). If a Python package build reports a `rustc` or `cargo` error, run `apt install -y rustc cargo` and repeat only the package-installation command.

## Install Python and Home Assistant

The tested environment uses Python 3.14.6. For a fresh installation, check the current Home Assistant Python requirement. Before an existing HA upgrade, follow [step 11](11_Update_cleanup.md); do not change the Python version underneath this virtual environment.

In Debian, run:

Choose an exact Home Assistant release from the [release list](https://github.com/home-assistant/core/releases). Replace the placeholder below and download that release's constraints file **before** installing Home Assistant so dependency resolution uses the matching constraints from the start.

```sh
# 1. Install the uv package manager
curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"

# 2. Install Python and create the virtual environment
uv python install 3.14.6
uv venv --python 3.14.6 /srv/homeassistant
source /srv/homeassistant/bin/activate

# 3. Set copy mode for chroot/Android
export UV_LINK_MODE=copy

# 4. Download constraints for the exact release before installing it
TARGET_HA_VERSION="YYYY.MM.PATCH" # Replace with the exact release selected above
rm -f /srv/homeassistant/constraints.txt.tmp
curl -fL "https://raw.githubusercontent.com/home-assistant/core/${TARGET_HA_VERSION}/homeassistant/package_constraints.txt" -o /srv/homeassistant/constraints.txt.tmp
test -s /srv/homeassistant/constraints.txt.tmp || { echo "Could not download the constraints file; stop here."; exit 1; }
mv /srv/homeassistant/constraints.txt.tmp /srv/homeassistant/constraints.txt || { echo "Could not install the constraints file; stop here."; exit 1; }
export UV_CONSTRAINT=/srv/homeassistant/constraints.txt

# 5. Install the selected Home Assistant release and base system modules
uv pip install "homeassistant==${TARGET_HA_VERSION}" pip wheel aioesphomeapi zlib-ng isal

# 6. Record the installed package set
uv pip freeze > /root/homeassistant-requirements-$(date +%F).txt
```

The virtual environment stays in internal storage (`/srv/homeassistant`); only the configuration directory and SQLite database are stored on the SD card or internal storage.

## Configure the `uv` profile

Python and packages are managed with `uv`. For stable operation in an Android/chroot environment, use the following variable profile:

```sh
export UV_LINK_MODE=copy
export UV_CONSTRAINT=/srv/homeassistant/constraints.txt
source /srv/homeassistant/bin/activate
```

`UV_LINK_MODE=copy` avoids hard links and symbolic links between the cache and isolated environment, which is important for Android filesystems.

`UV_CONSTRAINT` applies the version-compatible constraints file for the selected Home Assistant release and helps prevent dependency conflicts.

Do not use `UV_SYSTEM_PYTHON=1` in this project. It breaks venv isolation and Debian’s externally-managed protection blocks package installation.

If `test -s /srv/homeassistant/constraints.txt` fails, stop and check Internet access and the output of `hass --version`.
