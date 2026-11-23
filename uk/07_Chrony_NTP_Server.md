# Home Assistant на Android із нативним chroot: 07 — NTP-сервер Chrony

Англійська версія: [07_Chrony_NTP_Server.md](../07_Chrony_NTP_Server.md).


Якщо пристроям локальної мережі потрібен Network Time Protocol (NTP), зокрема під час зникнення зовнішнього Інтернету, налаштуйте Chrony у Termux. Пропустіть цей необов’язковий крок, якщо достатньо зовнішніх NTP-серверів.

## 1. Встановлення Chrony

У звичайному Termux установіть пакет і створіть захищені каталоги конфігурації, даних і виконання:

```sh
pkg install -y chrony
mkdir -p ~/.config/chrony ~/.local/share/chrony ~/.local/var/run/chrony
chmod 700 ~/.config/chrony ~/.local/share/chrony ~/.local/var/run/chrony
```

## 2. Налаштування NTP-сервера

Відкрийте файл конфігурації:

```sh
nano ~/.config/chrony/chrony.conf
```

`192.168.31.0/24` і `192.168.31.2` — це приклади, у вашій мережі значення можуть відрізнятися. Замініть підмережу на довірену локальну підмережу, а для NTP-сервера вкажіть фактичну LAN-адресу Android-хоста; виберіть відповідні зовнішні пули часу:

```conf
# Зовнішні джерела NTP, які використовуються за наявності Інтернету.
pool ua.pool.ntp.org iburst
pool pool.ntp.org iburst

# Зберігати дані виконання та похибки годинника у шляхах, доступних Termux.
pidfile /data/data/com.termux/files/home/.local/var/run/chrony/chronyd.pid
bindcmdaddress /data/data/com.termux/files/home/.local/var/run/chrony/chronyd.sock
driftfile /data/data/com.termux/files/home/.local/share/chrony/drift

# Дозволити NTP-клієнтів лише з довіреної локальної підмережі.
allow 192.168.31.0/24

# Використовувати годинник телефона як резервне джерело, коли зовнішні недоступні.
local stratum 10

# Порт служби NTP; потрібні привілеї root.
port 123
```

Обмежте підмережу в `allow` локальною мережею. Не відкривайте UDP-порт 123 у публічний Інтернет.

## 3. Запуск і перевірка Chrony

Апаратним годинником керує Android, а UDP-порт 123 є привілейованим. Запустіть `chronyd` від Android root з параметром `-x`, щоб заборонити зміну системного годинника. Для початкової діагностики запустіть службу в передньому плані:

```sh
su -c '/data/data/com.termux/files/usr/bin/chronyd -x -d -f /data/data/com.termux/files/home/.config/chrony/chrony.conf'
```

В іншому сеансі Termux перегляньте стан Chrony та опитайте NTP-службу з клієнта локальної мережі або іншого мережевого пристрою. Після перевірки зупиніть службу в передньому плані клавішами `Ctrl+C`:

```sh
chronyc -h 127.0.0.1 tracking
chronyd -Q 'server <phone_lan_ip> iburst'
```

З іншого пристрою в цій самій LAN перевірте відповідь телефона на UDP/123. У Windows PowerShell виконайте команду нижче; вона не змінює годинник ПК:

```powershell
w32tm /stripchart /computer:<phone_lan_ip> /dataonly /samples:5
```

Замініть `<phone_lan_ip>` LAN-адресою телефона. Успішні вибірки підтверджують доступність NTP-сервера з цього ПК; показане відхилення — різниця між годинником ПК і часом сервера.

Щоб запустити процес у фоновому режимі, скористайтеся командою нижче. Крок [06](06_Automatic_Startup.md) запускає Chrony за `ENABLE_CHRONY=1` і виконує необов’язкову локальну перевірку, якщо також встановлено `ENABLE_CHRONY_CHECK=1` (одразу після запуску, потім кожні 15 хвилин):

```sh
su -c '/data/data/com.termux/files/usr/bin/chronyd -x -f /data/data/com.termux/files/home/.config/chrony/chrony.conf'
```

## 4. Налаштування клієнтів локальної мережі

Вкажіть LAN-адресу Android-хоста як NTP-сервер на клієнтських пристроях, наприклад на OpenBeken у **Config → Configure General / Time**:

| Параметр | Значення |
|---|---|
| NTP-сервер | LAN-адреса Android-хоста (наприклад, `192.168.31.2`) |
| Port | `123` (UDP) |

Закріпіть за Android-хостом сталу LAN-адресу або налаштуйте DHCP-резервування. Під час зникнення зовнішніх серверів Chrony надає час за годинником телефона; точність тоді залежить від нього.

