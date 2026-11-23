# Home Assistant на Android із нативним chroot: 03 — Debian і Python

Англійська версія: [03_Debian_Python.md](../03_Debian_Python.md).


Підготуйте Debian та встановіть ізольоване середовище Python для Home Assistant. Завершіть цей крок перед налаштуванням Home Assistant на [кроці 04](04_Home_Assistant.md).

## 1. Увійдіть у Debian і підготуйте базову систему

У звичайному Termux спочатку перевірте, які назви дистрибутивів показує `chroot-distro`:

```sh
su -c '/system/bin/chroot-distro list'
```

Команда показує дистрибутиви, відомі встановленому модулю, та їхній стан завантаження/встановлення, але не номер випуску Debian. Щоб перевірити поточний stable-випуск, опублікований Debian, прочитайте метадані офіційного репозиторію:

```sh
curl -fsSL https://deb.debian.org/debian/dists/stable/Release |
  grep -E '^(Suite|Codename|Version):'
```

Це показує версію поточного stable-репозиторію, а не обов’язково версію образу, який завантажує `chroot-distro`. Якщо Debian уже встановлено, перевірте версію саме цього chroot із Termux:

```sh
su -c '/system/bin/chroot-distro command debian "cat /etc/os-release; cat /etc/debian_version"'
```

Після цього увійдіть у встановлений Debian chroot від root-користувача:

```sh
su -c '/system/bin/chroot-distro login debian'
```

До завершення цього підрозділу виконуйте всі команди **у Debian** від root-користувача:

```sh
# Оновлення індексу пакетів перед початковим встановленням
apt update

# 2. Встановлення базових утиліт, залежностей для збирання, медіакодеків і бібліотек C
apt install -y \
  ca-certificates curl git nano tmux procps iproute2 sqlite3 \
  build-essential autoconf pkg-config \
  libssl-dev libxml2-dev libxslt1-dev libjpeg-dev libffi-dev libudev-dev zlib1g-dev \
  libavformat-dev libavcodec-dev libavdevice-dev libavutil-dev libswscale-dev \
  libswresample-dev libavfilter-dev ffmpeg \
  libturbojpeg0 libpcap-dev
```

Для подальшого оновлення пакетів Debian дивіться [крок 11](11_Update_cleanup.md). Тут команда лише оновлює індекс перед початковим налаштуванням.

Набір бібліотек відповідає поточному переліку залежностей Debian у [документації розробників Home Assistant](https://developers.home-assistant.io/docs/development_environment/). Якщо під час збирання пакета Python з’явиться помилка `rustc` або `cargo`, виконайте `apt install -y rustc cargo` і повторіть лише команду встановлення пакетів.

## Встановіть Python і Home Assistant

У перевіреному середовищі використовується Python 3.14.6. Для нового встановлення звіртеся з поточними вимогами Home Assistant до Python. Перед оновленням наявного HA виконайте [крок 11](11_Update_cleanup.md); не змінюйте версію Python під цим віртуальним середовищем.

У Debian виконайте:

Виберіть точний випуск Home Assistant зі [списку випусків](https://github.com/home-assistant/core/releases). Замініть заповнювач нижче та завантажте файл обмежень цього випуску **до** встановлення Home Assistant, щоб залежності від початку визначалися з відповідними обмеженнями.

```sh
# 1. Встановлення менеджера пакетів uv
curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"

# 2. Встановлення Python і створення віртуального середовища
uv python install 3.14.6
uv venv --python 3.14.6 /srv/homeassistant
source /srv/homeassistant/bin/activate

# 3. Увімкнення режиму копіювання для chroot/Android
export UV_LINK_MODE=copy

# 4. Завантаження обмежень для точного випуску перед його встановленням
TARGET_HA_VERSION="YYYY.MM.PATCH" # Замініть точним випуском, вибраним вище
rm -f /srv/homeassistant/constraints.txt.tmp
curl -fL "https://raw.githubusercontent.com/home-assistant/core/${TARGET_HA_VERSION}/homeassistant/package_constraints.txt" -o /srv/homeassistant/constraints.txt.tmp
test -s /srv/homeassistant/constraints.txt.tmp || { echo "Не вдалося завантажити файл обмежень; зупиніться тут."; exit 1; }
mv /srv/homeassistant/constraints.txt.tmp /srv/homeassistant/constraints.txt || { echo "Не вдалося встановити файл обмежень; зупиніться тут."; exit 1; }
export UV_CONSTRAINT=/srv/homeassistant/constraints.txt

# 5. Встановлення вибраного випуску Home Assistant і базових системних модулів
uv pip install "homeassistant==${TARGET_HA_VERSION}" pip wheel aioesphomeapi zlib-ng isal

# 6. Збереження переліку встановлених пакетів
uv pip freeze > /root/homeassistant-requirements-$(date +%F).txt
```

Віртуальне середовище зберігатиметься у внутрішній пам’яті (`/srv/homeassistant`); лише каталог конфігурації та база SQLite зберігатимуться на SD-картці або у внутрішній пам’яті.

## Налаштуйте профіль `uv`

Python і пакети керуються через `uv`. Для стабільної роботи в середовищі Android/chroot використовуйте такий набір змінних:

```sh
export UV_LINK_MODE=copy
export UV_CONSTRAINT=/srv/homeassistant/constraints.txt
source /srv/homeassistant/bin/activate
```

`UV_LINK_MODE=copy` уникає жорстких і символічних посилань між кешем та ізольованим середовищем, що важливо для файлових систем Android.

`UV_CONSTRAINT` застосовує сумісний файл обмежень для вибраного випуску Home Assistant і допомагає уникнути конфліктів залежностей.

Не використовуйте в цьому проєкті `UV_SYSTEM_PYTHON=1`. Цей параметр порушує ізоляцію venv, а захист Debian від зовнішнього керування пакетами блокує їх установлення.

Якщо `test -s /srv/homeassistant/constraints.txt` завершується помилкою, зупиніться та перевірте доступ до Інтернету й результат `hass --version`.

