# Home Assistant на Android із нативним chroot: 11 — оновлення й очищення

У цьому розділі зібрано повторювані процедури оновлення та очищення, які раніше були розподілені між кроками 02–08 і `Notes.md`. Він охоплює Termux, Debian chroot, Home Assistant та ESPHome.

> [!IMPORTANT]
> У проєкті використовується спосіб встановлення Home Assistant Core у Python-середовище. Home Assistant припинив виробничу підтримку цього способу з випуском 2025.12. Екземпляр і далі можна оновлювати, але офіційної підтримки та гарантованого шляху відновлення немає. Перед продовженням прочитайте [нотатки до випуску 2025.12](https://www.home-assistant.io/blog/2025/12/03/release-202512/) про зміну підтримки та [початкове оголошення про припинення підтримки](https://www.home-assistant.io/blog/2025/05/22/deprecating-core-and-supervised-installation-methods-and-32-bit-systems/).

Виконуйте кожну команду в середовищі, зазначеному в підрозділі. Оновлюйте лише один рівень за раз; не поєднуйте зміну Android/ROM з оновленням Debian, Home Assistant чи ESPHome.

## 1. Процедура оновлення

### Перед оновленням

1. Виберіть один компонент і прочитайте його нотатки до випуску. Для Home Assistant виберіть точний випуск і перевірте, чи підтримує він версію Python, яка вже використовується в `/srv/homeassistant`, за [поточними вимогами розробників](https://developers.home-assistant.io/docs/development_environment/).
2. Перевірте вільне місце перед завантаженням пакетів або зміною бази Recorder. Home Assistant зазначає, що операції з базою чи зміна її схеми можуть тимчасово потребувати вільного місця розміром не менше самої бази. За можливості залиште додатковий запас; перевірте і файлову систему конфігурації, і `/srv/homeassistant`.
3. Створіть датовану резервну копію відповідної конфігурації та запишіть версії встановлених пакетів. Важливі копії збережіть поза телефоном. Перелік пакетів допомагає відновленню, але не замінює копію конфігурації та бази даних.
4. Якщо запущено крок 06, перед запланованою зупинкою Home Assistant призупиніть WatchDog. Віджет із кроку 09 необов’язковий; кнопка паузи доступна, лише якщо його встановлено. Без віджета створіть прапорець обслуговування у звичайному Termux командою `touch ~/.ha-maintenance`. Перед продовженням дочекайтеся в `~/init.log` запису `Maintenance mode ON; checks are paused.` Прапорець набуває чинності наприкінці поточного циклу перевірки (за замовчуванням до 60 секунд). Він призупиняє перевірки WatchDog, але не зупиняє Home Assistant або `00-init.sh`.
5. Якщо оновлення або перевірка завершилися помилкою, залиште Home Assistant зупиненим і режим обслуговування ввімкненим. Збережіть вивід команд і резервні копії та з’ясуйте причину до перезапуску.

За замовчуванням прапорець обслуговування кроку 06 спливає через 180 хвилин. Якщо операція може тривати довше, до завершення цього часу поновіть його зі звичайного Termux командою `touch ~/.ha-maintenance`, а потім знову перевірте запис у журналі. Після перезавантаження прапорець видаляється.

Версія Python Home Assistant має залишатися сумісною з обраним випуском. Не змінюйте Python під уже створеним віртуальним середовищем. Якщо випуск потребує іншої основної або мінорної версії Python, зупиніться та окремо сплануйте перенесення віртуального середовища з власною резервною копією й планом відновлення.

### Пакети Termux

Виконайте ці команди у звичайній оболонці Termux, не в Debian і не від root:

```sh
pkg update
pkg upgrade
```

Перегляньте запропоновану транзакцію перед підтвердженням; не додавайте `-y`. Termux використовує модель rolling release, тому запускайте повне `pkg upgrade`, а не оновлюйте окремі пакети. Оновлення може замінити `sshd` і перервати SSH-сеанс. Під час віддаленого оновлення будьте готові підключитися знову або скористайтеся локальним екраном Termux. Як описано на [кроці 02](02_Android_Config_Termux.md), Termux і його плагіни мають походити з одного джерела підпису.

Після оновлення перевірте, що Termux відкривається, `su -c id` працює, а SSH знову приймає підключення. Не поєднуйте це з оновленням Android ROM, Magisk або модуля `chroot-distro`.

### Пакети Debian

Перед входом у Debian виконайте ці перевірки у звичайному Termux (на Android). Перша команда показує назви дистрибутивів, відомі `chroot-distro`, та їхній стан завантаження/встановлення; вона не показує номер випуску Debian. Друга читає поточний stable-випуск із метаданих офіційного репозиторію Debian. Він може відрізнятися від версії образу, який завантажує `chroot-distro`.

```sh
su -c '/system/bin/chroot-distro list'
curl -fsSL https://deb.debian.org/debian/dists/stable/Release |
  grep -E '^(Suite|Codename|Version):'
```

Якщо Debian уже встановлено, перевірте фактичну версію цього chroot із Termux, а потім увійдіть до нього від root-користувача:

```sh
su -c '/system/bin/chroot-distro command debian "cat /etc/os-release; cat /etc/debian_version"'
su -c '/system/bin/chroot-distro login debian'
```

Якщо Home Assistant запущено, спершу призупиніть WatchDog і зупиніть HA за інструкціями в наступному підрозділі, перш ніж змінювати бібліотеки Debian. Наведені нижче команди виконуйте всередині Debian.

```sh
apt update
apt list --upgradable
apt upgrade
dpkg --audit
```

Перегляньте план пакетів і використовуйте звичайний режим `apt upgrade`. Не запускайте `full-upgrade`/`dist-upgrade` як звичайну процедуру оновлення: для розв’язання залежностей ці операції можуть видалити пакети. Перехід на новий випуск Debian — окрема міграція, не описана в цій процедурі. Після оновлення перевірте chroot і Home Assistant вручну або перезавантажте телефон, якщо налаштовано крок 06.

### Home Assistant Core

Інструкції нижче розраховані на наявне віртуальне середовище `/srv/homeassistant`, налаштування `UV_LINK_MODE=copy` та `HA_CONFIG_DIR`, вибраний на [кроці 04](04_Home_Assistant.md). Команди Debian виконуйте від root.

Спочатку встановіть активний шлях конфігурації: `/mnt/ha_native`, якщо працює bind-монтування SD-картки, або `/root/.homeassistant` для внутрішнього сховища. Перевірте вільне місце, запишіть поточні версії та перевірте конфігурацію:

```sh
export HA_CONFIG_DIR="/mnt/ha_native" # Або: /root/.homeassistant
STAMP="$(date +%Y%m%d-%H%M%S)"
umask 077

df -h "$HA_CONFIG_DIR" /srv/homeassistant
if [ -f "$HA_CONFIG_DIR/home-assistant_v2.db" ]; then
    du -h "$HA_CONFIG_DIR/home-assistant_v2.db"
fi

source /srv/homeassistant/bin/activate
export UV_LINK_MODE=copy
hass --version
/srv/homeassistant/bin/python --version
uv --version
uv pip check
hass --script check_config --config "$HA_CONFIG_DIR"
```

Після запису WatchDog про ввімкнення режиму обслуговування зупиніть Home Assistant і переконайтеся, що процес завершився:

```sh
pkill -TERM -f '[s]rv/homeassistant/bin/hass' || true
sleep 5
pgrep -af '[s]rv/homeassistant/bin/hass' || echo "Home Assistant зупинено."
```

Якщо `pgrep` показує процес Home Assistant, зупиніться й з’ясуйте, чому він не завершився. Не архівуйте активну базу SQLite. Після зупинки процесу створіть датовані копії:

```sh
tar -C "$HA_CONFIG_DIR" -czf "/root/ha-config-${STAMP}.tar.gz" .
uv pip freeze > "/root/homeassistant-requirements-before-${STAMP}.txt"
cp /srv/homeassistant/constraints.txt "/root/homeassistant-constraints-before-${STAMP}.txt"
chmod 600 "/root/ha-config-${STAMP}.tar.gz" \
          "/root/homeassistant-requirements-before-${STAMP}.txt" \
          "/root/homeassistant-constraints-before-${STAMP}.txt"
```

Перед значним або ризикованим оновленням скопіюйте архів конфігурації з телефона. Для налаштування на SD-картці архів створюється з активного bind-монтування Debian; не підставляйте шлях Android-хоста `/mnt/media_rw/...`.

Виберіть точну цільову версію зі [списку випусків Home Assistant](https://github.com/home-assistant/core/releases), замініть заповнювач і **до встановлення** завантажте файл обмежень саме цього випуску:

```sh
TARGET_HA_VERSION="YYYY.MM.PATCH" # Замініть на точний випуск, наприклад 2026.11.2
NEW_CONSTRAINTS="/srv/homeassistant/constraints-${TARGET_HA_VERSION}.txt"

rm -f "${NEW_CONSTRAINTS}.tmp"
curl -fL \
  "https://raw.githubusercontent.com/home-assistant/core/${TARGET_HA_VERSION}/homeassistant/package_constraints.txt" \
  -o "${NEW_CONSTRAINTS}.tmp"
test -s "${NEW_CONSTRAINTS}.tmp" || { echo "Не вдалося завантажити файл обмежень; зупиніться тут."; exit 1; }
mv "${NEW_CONSTRAINTS}.tmp" "$NEW_CONSTRAINTS" || { echo "Не вдалося встановити файл обмежень; зупиніться тут."; exit 1; }

export UV_CONSTRAINT="$NEW_CONSTRAINTS"
uv pip install "homeassistant==${TARGET_HA_VERSION}"
uv pip check
hass --version
hass --script check_config --config "$HA_CONFIG_DIR"
```

Продовжуйте лише якщо встановлення успішне, `uv pip check` не знаходить порушених залежностей, `hass --version` показує вибрану версію, а `check_config` завершується без помилок. Потім зробіть файл обмежень цього випуску активним для кроку 06:

```sh
cp "$NEW_CONSTRAINTS" /srv/homeassistant/constraints.txt.new
chmod 644 /srv/homeassistant/constraints.txt.new
mv /srv/homeassistant/constraints.txt.new /srv/homeassistant/constraints.txt
export UV_CONSTRAINT=/srv/homeassistant/constraints.txt
```

Якщо встановлення чи перевірка завершилися помилкою, не відновлюйте WatchDog і не перезавантажуйте телефон із несправним середовищем. Збережіть резервні копії, файли обмежень і текст помилки. Перелік `uv pip freeze` допоможе відновити попередній набір пакетів, але якщо нову версію вже запускали, може знадобитися також повернути конфігурацію та базу HA.

Після успішного оновлення рекомендований для цього проєкту спосіб — звичайно перезавантажити Android. Під час завантаження крок 06 видалить залишений прапорець обслуговування та запустить служби. Перевірте успішний запуск у `~/init.log` і відкрийте вебінтерфейс HA. Якщо не перезавантажуєте телефон, спершу запустіть HA вручну й переконайтеся, що він відповідає, і лише тоді приберіть прапорець обслуговування; відновлений WatchDog чекатиме на HA, якщо той зупинений.

### ESPHome Device Builder

Зупиніть Builder кнопкою `esphome-stop` із [кроку 09](09_Termux_Widget.md), якщо встановили необов’язковий віджет. Якщо віджета немає, зупиніть Builder з root-оболонки Debian і перевірте, що процес завершився:

```sh
pkill -TERM -f '[e]sphome-device-builder' || true
sleep 2
pgrep -af '[e]sphome-device-builder' || echo "ESPHome Device Builder зупинено."
```

Після цього створіть копію YAML-конфігурації та запишіть чинний набір пакетів:

```sh
STAMP="$(date +%Y%m%d-%H%M%S)"
umask 077
tar -C /srv/esphome/config -czf "/root/esphome-config-${STAMP}.tar.gz" .
source /srv/esphome/bin/activate
uv pip freeze > "/root/esphome-requirements-before-${STAMP}.txt"
chmod 600 "/root/esphome-config-${STAMP}.tar.gz" \
          "/root/esphome-requirements-before-${STAMP}.txt"

uv pip install --upgrade "esphome-device-builder[esphome]"
uv pip check
esphome-device-builder --help
```

Якщо архів містить єдину копію YAML пристроїв, збережіть його поза телефоном. Після перевірки запустіть Builder віджетом і переконайтеся, що його вебінтерфейс із паролем відкривається, перш ніж прошивати пристрої.

Оновлення Builder і прошивки ESP-пристроїв — різні операції. Після першого прошивання через USB подальші оновлення зазвичай надсилаються OTA; користуйтеся процедурою конкретного пристрою, а вихідні YAML зберігайте в `/srv/esphome/config/`.

### Android, ROM, Magisk і chroot-distro

Ці рівні залежать від моделі пристрою та ROM. Дотримуйтеся точних інструкцій для пристрою з [кроку 01](01_Device_Preparation.md) і [кроку 02](02_Android_Config_Termux.md), створіть повну резервну копію та перевірте шлях відновлення. Не поєднуйте ці зміни з оновленнями Debian, Home Assistant або ESPHome. Після кожного перезавантаження перевірте root-доступ, `chroot-distro list`, SSH і `~/init.log`, перш ніж переходити до наступного рівня.

## 2. Процедура очищення

Очищайте лише відомі кеші або дані, які відповідна служба може безпечно створити знову. Спершу оцініть використання сховища; не запускайте масове `rm -rf` для `/tmp`, `/root/.cache`, chroot або каталогу конфігурації Home Assistant.

### Перевірка вільного місця

У звичайному Termux:

```sh
df -h "$HOME" "$PREFIX"
du -sh "$HOME" "$PREFIX/var/cache/apt/archives" 2>/dev/null
```

У Debian встановіть `HA_CONFIG_DIR`, як на кроці 04, та перевірте файлові системи й важливі каталоги:

```sh
df -h / "$HA_CONFIG_DIR" /srv/homeassistant
if [ -d /srv/esphome ]; then
    df -h /srv/esphome
fi
du -sh /var/cache/apt/archives /root/.cache/uv \
  /srv/homeassistant "$HA_CONFIG_DIR" 2>/dev/null
if [ -d /srv/esphome ]; then
    du -sh /srv/esphome
fi
```

### Кеші пакетів і Python

У Termux команда `pkg autoclean` прибирає застарілі архіви пакетів; `pkg clean` видаляє всі завантажені архіви. Жодна з них не видаляє встановлені пакети. Використовуйте `pkg clean`, лише якщо потрібно звільнити більше місця: файли пакетів доведеться завантажити знову.

У Debian:

```sh
apt autoclean
apt clean
apt-get -s autoremove
```

`apt clean` видаляє завантажені архіви `.deb`, а не встановлені пакети. Остання команда лише імітує видалення. Перегляньте повний список; якщо там є знайомий або потрібний пакет, зупиніться. Якщо у списку лише непотрібні автоматично встановлені залежності й ви згодні з планом, запустіть `apt autoremove` інтерактивно. Не додавайте `-y` або `--purge`.

Для спільного кешу `uv` перевірте шлях і розмір, після чого видаліть лише застарілі невикористані записи:

```sh
uv cache dir
du -sh "$(uv cache dir)"
uv cache prune
du -sh "$(uv cache dir)"
```

У цьому проєкті активні віртуальні середовища створені безпосередньо в `/srv/homeassistant` і `/srv/esphome`; не видаляйте їх під час очищення кешу. `uv cache prune` також може видалити невикористані централізовані середовища `uv`, якщо ви зберігаєте їх в інших проєктах. Віддавайте перевагу цій команді замість `uv cache clean` і ніколи не видаляйте файли кешу вручну.

### Завантажений rootfs `chroot-distro`

Виконуйте ці команди у звичайному Termux, не в Debian. Стани `Downloaded: Yes` і `Installed: Yes` незалежні: завантажений архів rootfs потрібен лише для встановлення або повторного встановлення дистрибутива. Щоб звільнити його місце, залишивши встановлений Debian chroot, перевірте стани, видаліть лише архів і перевірте ще раз:

```sh
# Переконайтеся, що для Debian вказано Downloaded: Yes та Installed: Yes
su -c '/system/bin/chroot-distro list'

# Видаліть лише завантажений архів rootfs Debian
su -c '/system/bin/chroot-distro delete debian'

# Переконайтеся, що тепер вказано Downloaded: No та Installed: Yes
su -c '/system/bin/chroot-distro list'
```

Установлений Debian залишиться придатним до роботи. Якщо згодом знадобиться встановити його знову, спершу завантажте rootfs. Не використовуйте для такого очищення `chroot-distro remove debian`: команда `remove` видаляє завантажений архів, резервну копію та каталог установленого дистрибутива. Докладніше — у [довідці команд `chroot-distro`](https://github.com/Magisk-Modules-Alt-Repo/chroot-distro/blob/main/README.md).

### Історія Home Assistant і журнали

У кроці 04 задано `recorder.purge_keep_days: 14`. Recorder автоматично видаляє старішу історію й за ввімкнених стандартних параметрів періодично перепаковує базу. Після самого видалення старих записів файл SQLite може не стати меншим. Не видаляйте `home-assistant_v2.db`, не редагуйте її інструментами SQLite, поки працює HA, і не видаляйте `.storage` заради звільнення місця.

Щоб вручну запустити очищення й перепакування, скористайтеся дією Recorder: Purge Recorder database у Home Assistant:

```yaml
action: recorder.purge
data:
  keep_days: 14
  repack: true
```

Перепакування може сповільнити HA і тимчасово потребує додаткового місця. Спершу створіть резервну копію та переконайтеся, що вільного місця не менше за розмір бази; якщо його мало, відкладіть операцію. Докладніше — в офіційній [документації Recorder](https://www.home-assistant.io/integrations/recorder/) і [описі дії очищення](https://www.home-assistant.io/actions/recorder.purge/).

Скрипт кроку 06 ротює `~/init.log`, коли той перевищує 1 MiB, і зберігає попередню копію в `init.log.1`. Переглядайте журнали через `tail`; не обрізайте журнал, поки служба в нього пише. Журнал Builder `/root/esphome.log` обробляється скриптом його запуску.

### Старі резервні копії

Перегляньте датовані резервні копії, перевірте читабельність найновішого успішного архіву та збережіть важливі дані поза телефоном:

```sh
find /root -maxdepth 1 -type f \( \
  -name 'ha-config-*.tar.gz' -o \
  -name 'homeassistant-requirements-before-*.txt' -o \
  -name 'homeassistant-constraints-before-*.txt' -o \
  -name 'esphome-config-*.tar.gz' -o \
  -name 'esphome-requirements-before-*.txt' \
\) -exec ls -lh -- {} +

# Замініть часову позначку на точне ім'я архіву зі списку.
tar -tzf "/root/ha-config-YYYYMMDD-HHMMSS.tar.gz" >/dev/null
```

Залишайте найновіший перевірений архів конфігурації та перелік пакетів до перевірки наступного оновлення. Старі файли видаляйте лише за точними іменами після перевірки; не використовуйте підстановку `*` для видалення.

Не видаляйте `/srv/homeassistant`, `/srv/esphome`, `/root/.homeassistant`, `/mnt/ha_native`, активні скрипти Termux:Boot або файли в chroot як звичайну процедуру очищення.

## Джерела

- [Керування пакетами Termux](https://github.com/termux/termux-packages/wiki/Package-Management)
- [Посібник Debian APT](https://manpages.debian.org/stable/apt/apt-get.8.en.html)
- [Обмеження пакетів і команди pip в uv](https://docs.astral.sh/uv/reference/cli/#uv-pip-install)
- [Керування кешем uv](https://docs.astral.sh/uv/concepts/cache/)
- [Випуски Home Assistant](https://github.com/home-assistant/core/releases) і версійний файл `homeassistant/package_constraints.txt` у відповідному тегу
- [Recorder Home Assistant](https://www.home-assistant.io/integrations/recorder/)
