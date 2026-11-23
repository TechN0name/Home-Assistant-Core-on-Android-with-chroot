# 08 — ESPHome Device Builder у Debian chroot

Англійська версія: [08_ESPHome_Builder.md](../08_ESPHome_Builder.md).


На цьому кроці в Debian chroot встановлюється **[ESPHome Device Builder](https://github.com/esphome/device-builder/)** як окрема служба. Це не класичний `esphome dashboard`: у посібнику використовується пакет `esphome-device-builder[esphome]` і його командний інтерфейс.

> [!NOTE]
> Для цього кроку потрібен [крок 03](03_Debian_Python.md); середовище Python Home Assistant він не змінює.

## 1. Створення ізольованого середовища

У Termux увійдіть у Debian від root:

```sh
su -c '/system/bin/chroot-distro login debian'
```

У Debian chroot створіть середовище й установіть ESPHome Device Builder:

```sh
mkdir -p /srv/esphome/config
uv venv /srv/esphome
source /srv/esphome/bin/activate
uv pip install "esphome-device-builder[esphome]"
esphome-device-builder --help
```

Якщо довідка з’явилася без помилок, створіть `/root/start-esphome.sh` і виконайте перший тестовий запуск.

## 2. Створення скрипту запуску

У Debian відкрийте `nano`, щоб створити скрипт запуску:

```sh
nano /root/start-esphome.sh
```

Задайте пароль і вставте такий вміст:

```sh
#!/bin/bash

CONFIG_DIR="/srv/esphome/config"

# Створити каталог конфігурації, якщо його ще немає
mkdir -p "$CONFIG_DIR"

export ESPHOME_USERNAME="admin"
export ESPHOME_PASSWORD="replace-with-the-password"

# Запустити безпосередньо виконуваний файл із середовища
exec /srv/esphome/bin/esphome-device-builder "$CONFIG_DIR" --host 0.0.0.0 --port 6052
```

У `nano` натисніть `Ctrl+O`, потім Enter, щоб зберегти; натисніть `Ctrl+X`, щоб вийти. Обмежте доступ до файла, оскільки пароль зберігається у відкритому вигляді:

```sh
chmod 700 /root/start-esphome.sh
```

> [!IMPORTANT]
> Для автономної роботи Builder потрібні обидві змінні `ESPHOME_USERNAME` і `ESPHOME_PASSWORD`. На відміну від параметра `--password`, змінні середовища не показують пароль у списку процесів.

## 3. Запуск і перевірка

У Debian запустіть скрипт у передньому плані, щоб перевірити роботу:

```sh
/root/start-esphome.sh
```

Після запуску Builder доступний у локальній мережі за адресою `http://<phone_ip>:6052`. Перевіривши доступ через браузер, зупиніть процес клавішами `Ctrl+C` у терміналі.

Builder не потрібно постійно тримати запущеним. На [кроці 09](09_Termux_Widget.md) створюються кнопки для ручного запуску, зупинки та перегляду журналу.

> [!WARNING]
> Автономний Builder за замовчуванням доступний із локальної мережі. Не перенаправляйте порт `6052` у глобальну мережу. Задайте надійний пароль. Класичний `esphome dashboard` — це інший інтерфейс і процес.

## Оновлення та прошивки

Про оновлення пакета Builder і безпечне резервне копіювання дивіться у [кроці 11](11_Update_cleanup.md). Для першого прошивання нового ESP-пристрою потрібен USB-кабель із передаванням даних; подальші оновлення прошивки зазвичай надсилаються мережею (OTA). Зберігайте YAML-файли лише в `/srv/esphome/config/`.

