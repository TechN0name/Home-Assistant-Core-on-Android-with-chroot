# Home Assistant на Android із нативним chroot: 04 — Налаштування Home Assistant

Англійська версія: [04_Home_Assistant.md](../04_Home_Assistant.md).


Налаштуйте Home Assistant, перевірте ручний запуск і за бажанням розмістіть файли конфігурації та базу даних на SD-картці. Завершіть ручну перевірку перед увімкненням автоматичного запуску на [кроці 06](06_Automatic_Startup.md).

> [!WARNING]
> Home Assistant працює від root-користувача всередині chroot, оскільки Android обмежує мережеві можливості та доступ до пристроїв. Тому власні інтеграції можуть отримати доступ до привілейованих ресурсів. Установлюйте лише довірені інтеграції та залишайте інтерфейс Home Assistant у довіреній мережі.

## Необов’язково: SD-картка для конфігурації та бази даних

### Призначення

У цьому варіанті **вибрана** SD-картка форматується у `ext4`; після цього Android має автоматично підключити її до `/mnt/media_rw/<UUID>`. Далі каталог `ha_config` підключається bind-монтуванням до дерева Debian як `/mnt/ha_native`.

Такий підхід обходить шар FUSE Android для даних HA. Це корисно для SQLite, але не обов’язково. Автоматичне монтування `ext4` залежить від ROM і політики SELinux; воно працювало на перевіреній конфігурації LineageOS 23.2 / Android 16, але має бути перевірене на цільовому пристрої.

> [!CAUTION]
> Наступні команди повністю стирають вибраний блоковий пристрій. Зупиніться, якщо розмір, модель або шлях картки неочікувані. Значення `mmcblk1` не можна вважати універсальним.

### Визначте пристрій до форматування

У Debian установіть потрібні утиліти та перегляньте доступні пристрої:

```sh
apt install -y parted e2fsprogs util-linux
lsblk -o NAME,SIZE,MODEL,TRAN,FSTYPE,MOUNTPOINTS
cat /proc/partitions
ls -l /dev/block/mmcblk*
```

На перевіреному телефоні SD-картка була `/dev/block/mmcblk1`, але на іншому телефоні чи ядрі шлях може відрізнятися. Спершу перевірте **розмір** картки. Якщо `lsblk` не показує пристрій у chroot, вийдіть із Debian (`exit`) і перегляньте його з root-оболонки Termux:

```sh
su -c 'cat /proc/partitions; ls -l /dev/block/mmcblk*'
```

### Форматуйте лише після точної перевірки

Нижче `<SD_DEVICE>` — перевірений шлях без номера розділу (наприклад, `/dev/block/mmcblk1`). Для реального пристрою використовуйте саме перевірений шлях.

Наведені нижче команди містять шляхи з перевіреного телефона. Перед запуском замініть усі входження `/dev/block/mmcblk1` і `/dev/block/mmcblk1p1` на перевірені шляхи пристрою та розділу, якщо у вас вони інші. Не запускайте команди форматування з неперевіреним шляхом.

**Крок 1. Запустіть інтерактивний `parted` у Debian:**

```sh
parted /dev/block/mmcblk1
```

У запрошенні `(parted)` вводьте команди послідовно:

```text
mklabel gpt
mkpart primary ext4 1MiB 100%
print
quit
```

**Крок 2. Відмонтуйте пристрій, якщо його змонтовано:**
У налаштуваннях файлового менеджера Android або з root-оболонки Termux виконайте:

```sh
su -c 'umount -l /dev/block/mmcblk1*'
```

**Крок 3. Створіть і налаштуйте файлову систему ext4:**
Створіть ext4 на новому розділі в Debian. Наприклад, якщо `<SD_DEVICE>` відповідає `/dev/block/mmcblk1`, то `<SD_PARTITION>` буде `/dev/block/mmcblk1p1`; для інших типів пристроїв назва розділу може відрізнятися. Виконуйте команди по порядку. Крок 2 необов’язковий; пропустіть його, щоб залишити журналювання ext4 увімкненим:

```sh
# 1. Створити файлову систему ext4 з міткою HA_CONFIG:
mkfs.ext4 -F -L HA_CONFIG /dev/block/mmcblk1p1

# 2. Необов’язково: вимкнути журналювання ext4, щоб зменшити кількість записів на SD-картку.
#    Це погіршує відновлення після аварійного вимкнення; пропустіть команду, якщо цілісність даних важливіша.
tune2fs -O ^has_journal /dev/block/mmcblk1p1

# 3. Звільнити місце, зарезервоване для системного root-користувача (віддати 100% місця для даних):
tune2fs -m 0 /dev/block/mmcblk1p1

# 4. Переглянути UUID нового розділу:
blkid /dev/block/mmcblk1p1

# 5. Виконати остаточну перевірку цілісності файлової системи:
e2fsck -fn /dev/block/mmcblk1p1
```

Якщо `parted` повідомляє, що ядро досі використовує таблицю розділів, перезавантажте телефон перед форматуванням і ще раз перевірте шлях.

### Перевірте автоматичне монтування Android

Після форматування перезавантажте телефон. У Termux перевірте, чи Android автоматично підключив картку (власні ROM зазвичай роблять це для ext4):

```sh
su -c 'ls -la /mnt/media_rw'
su -c 'mount | grep /mnt/media_rw'
```

Запишіть справжній UUID із назви каталогу або з `blkid`. Якщо `/mnt/media_rw/<UUID>` відсутній, **зупиніться**: не накладайте ручне монтування того самого блокового пристрою поверх монтування Android. Спершу визначте, чи підтримує ROM цей тип картки та чому `vold` її не підключив.

### Створіть і перевірте точку монтування конфігурації Home Assistant

Дізнавшись UUID, установіть фактичне значення змінної в Termux:

```sh
export SD_UUID="paste_SD_UUID_here"
```

Замініть `paste_SD_UUID_here` на перевірений UUID. Потім створіть каталоги та виконайте bind-монтування з параметром Magisk `-mm` (`--mount-master`), щоб працювати у глобальному просторі монтування:

```sh
# Створити каталог конфігурації HA на картці:
su -mm -c "mkdir -p /mnt/media_rw/$SD_UUID/ha_config"

# Створити точку монтування всередині Debian chroot:
su -mm -c 'mkdir -p /data/local/chroot-distro/debian/mnt/ha_native'

# Відкрити каталог картки для Debian через bind-монтування:
su -mm -c "mount --bind /mnt/media_rw/$SD_UUID/ha_config /data/local/chroot-distro/debian/mnt/ha_native"
```

Перевірте bind-монтування і в глобальному просторі імен, і з Debian. Самого успішного `touch` недостатньо: монтування має бути видимим із chroot, у якому працює HA.

```sh
su -mm -c 'mount | grep ha_native'
su -mm -c '/system/bin/chroot-distro command debian "mount | grep ha_native"'
su -mm -c '/system/bin/chroot-distro command debian "ls -la /mnt/ha_native"'
su -mm -c '/system/bin/chroot-distro command debian "touch /mnt/ha_native/.write-test && ls -la /mnt/ha_native"'
```

Після перевірки видаліть тестовий файл:

```sh
su -mm -c '/system/bin/chroot-distro command debian "rm -f /mnt/ha_native/.write-test"'
```

> [!WARNING]
> **Bind-монтування тимчасове.** Створене вище монтування (`/mnt/ha_native`) існує лише в пам’яті ядра й зникає після перезавантаження телефона. Скрипт запуску з кроку 06 відновлює його автоматично, але лише після налаштування та запуску цього скрипту.
>
> Якщо телефон перезавантажився до завершення ручного налаштування або активації кроку 06, перед входом у Debian повторіть у Termux:
>
> ```sh
> export SD_UUID='paste_SD_UUID_here'
> su -mm -c "mkdir -p /mnt/media_rw/$SD_UUID/ha_config"
> su -mm -c 'mkdir -p /data/local/chroot-distro/debian/mnt/ha_native'
> su -mm -c "mount --bind /mnt/media_rw/$SD_UUID/ha_config /data/local/chroot-distro/debian/mnt/ha_native"
> ```

#### Необов’язково: перенесення наявної конфігурації HA на SD-картку

Виконуйте перенесення лише якщо HA вже ініціалізовано з конфігурацією у внутрішній пам’яті, наприклад якщо існують `/root/.homeassistant/configuration.yaml` і `/root/.homeassistant/.storage`. Спершу зупиніть HA. Наведені команди копіюють відомі файли конфігурації HA, сховище стану та базу даних із `/root/.homeassistant` до змонтованого каталогу SD-картки (`/mnt/ha_native`), не видаляючи оригіналів.

У Termux увійдіть у Debian через головний простір монтування:

```sh
su -mm -c '/system/bin/chroot-distro login debian'
```

У Debian виконайте:

```sh
pkill -TERM -f '[s]rv/homeassistant/bin/hass' || true
cp -a /root/.homeassistant/configuration.yaml /root/.homeassistant/.storage /mnt/ha_native/
for item in automations.yaml scripts.yaml scenes.yaml secrets.yaml custom_components www blueprints packages themes home-assistant_v2.db home-assistant_v2.db-shm home-assistant_v2.db-wal; do
    [ -e "/root/.homeassistant/$item" ] && cp -a "/root/.homeassistant/$item" /mnt/ha_native/
done
ls -la /mnt/ha_native
export UV_LINK_MODE=copy
export UV_CONSTRAINT=/srv/homeassistant/constraints.txt
source /srv/homeassistant/bin/activate
hass --script check_config --config /mnt/ha_native
exit
```

Якщо наявна конфігурація HA зберігається в іншому каталозі, скопіюйте вміст саме цього каталогу; не копіюйте навмання весь каталог Debian `/root`. Перенесення стосується наявного встановлення HA після завершення кроку 03 зі створення середовища Python. Перед увімкненням кроку 06 переконайтеся, що `check_config` завершується успішно, а очікувані файли конфігурації та `.storage` є в каталозі на SD-картці.

Після перезавантаження це монтування зникає. Скрипт кроку 06 відновить його, лише якщо задано `USE_SD_CARD=1` і UUID.

## Ручний запуск Home Assistant

### Попередня перевірка: перевірте монтування SD-картки (лише для режиму SD-картки)

Якщо ви обрали SD-картку, перед входом у Debian переконайтеся, що bind-монтування активне. Воно тимчасове й не переживає перезавантаження.

У Termux виконайте:

```sh
su -mm -c 'mount | grep ha_native'
```

- **Вивід порожній:** монтування зникло (телефон перезавантажували). Відновіть його за командами з блока-попередження в розділі про SD-картку, а потім продовжуйте.
- **Є рядок bind-монтування:** монтування активне, можна входити в Debian.

Із цього сеансу Termux увійдіть у Debian. Параметр `-mm` залишає видимим головний простір монтування, зокрема bind-монтування SD-картки:

```sh
su -mm -c '/system/bin/chroot-distro login debian'
```

Усі наведені нижче команди виконуйте в оболонці Debian від root-користувача. Змінні середовища, експортовані в Termux, автоматично до Debian не передаються.

Виберіть одне з двох місць для конфігурації та задайте `HA_CONFIG_DIR` у цій оболонці Debian:

**Варіант 1 — із SD-карткою.** Задайте в Debian робочий каталог на змонтованій SD-картці:
```sh
export HA_CONFIG_DIR="/mnt/ha_native"
```

**Варіант 2 — без SD-картки / у внутрішній пам’яті.** Наведений рядок навмисно закоментований; для цього варіанта видаліть лише початковий символ `#`:

```sh
#export HA_CONFIG_DIR="/root/.homeassistant"
```

### Спільний шлях до робочої конфігурації (`$HA_CONFIG_DIR`)

Усі подальші команди в цій оболонці Debian використовують спільну змінну робочого каталогу `$HA_CONFIG_DIR`, вибрану вище. Якщо пізніше відкриєте нову оболонку Debian, задайте й експортуйте її знову.

Виберіть шлях відповідно до режиму встановлення:
- **Внутрішня пам’ять (режим A):** фактичний каталог конфігурації — `/root/.homeassistant`.
- **SD-картка (режим B):** робочий каталог конфігурації в Debian — `/mnt/ha_native`.

**Чому для режиму B використовується `/mnt/ha_native`:**
У Debian chroot `/mnt/ha_native` — це активне bind-монтування каталогу SD-картки `ha_config`. Саме цей шлях використовує Home Assistant під час запуску (`hass --config /mnt/ha_native`). Фізичне сховище хоста в `/mnt/media_rw/<UUID>` існує лише у просторі монтування Android і його **не можна** використовувати для звичайного обслуговування чи адміністрування Home Assistant у Debian.

Завдяки `$HA_CONFIG_DIR` усі адміністративні дії — редагування `configuration.yaml`, запуск `check_config`, перегляд `home-assistant.log`, перевірка SQLite чи створення резервних копій — однакові для обох варіантів сховища.

Створіть каталог конфігурації:
```sh
install -d -m 0700 "$HA_CONFIG_DIR"
```
Під час першого запуску буде створено основну структуру файлів і завантажено потрібні компоненти інтеграцій. Зупиняйте процес клавішами `Ctrl+C`, лише коли в журналі з’явиться повідомлення про успішний запуск, а вебінтерфейс буде готовий на порту 8123:

```sh
export UV_LINK_MODE=copy
export UV_CONSTRAINT=/srv/homeassistant/constraints.txt
source /srv/homeassistant/bin/activate

hass --config "$HA_CONFIG_DIR"
```

У браузері відкрийте `http://<phone_ip>:8123`, завершіть початкове налаштування й створіть обліковий запис. Після першого успішного входу зупиніть процес у терміналі клавішами `Ctrl+C` і перевірте конфігурацію:

```sh
export UV_LINK_MODE=copy
export UV_CONSTRAINT=/srv/homeassistant/constraints.txt
source /srv/homeassistant/bin/activate

hass --script check_config --config "$HA_CONFIG_DIR"
```

### Зменшення кількості записів на SD-картку або внутрішню пам’ять

```sh
nano "$HA_CONFIG_DIR/configuration.yaml"
```

```yaml
default_config:

frontend:
  themes: !include_dir_merge_named themes

automation: !include automations.yaml
script: !include scripts.yaml
scene: !include scenes.yaml

# Зменшити частоту запису журналу, щоб берегти накопичувач
recorder:
  commit_interval: 60
  purge_keep_days: 14
```

У `nano` натисніть `Ctrl+O`, потім Enter, щоб зберегти; натисніть `Ctrl+X`, щоб вийти.

Перевіряйте конфігурацію після кожної зміни `.yaml`:

```sh
hass --script check_config --config "$HA_CONFIG_DIR"
```


Запустіть Home Assistant вручну й переконайтеся, що після перезапуску вебінтерфейс відкривається. Переходьте до кроку 06 лише після успішної перевірки.

## Необов’язково: датчики обладнання Android через `command_line`

У цьому розділі додаються датчики для конкретного пристрою, які читають дані обладнання безпосередньо з інтерфейсу ядра Linux sysfs (`/sys/class/`). Усі команди виконуйте в Debian chroot від root; нативний chroot має доступ до інтерфейсів ядра Android.

> [!NOTE]
> Ці датчики призначені для Redmi Note 8T (Snapdragon 665) і перевіреного ядра. Індекси термозон і шляхи sysfs на інших пристроях можуть відрізнятися. Перш ніж додавати датчик, перевірте наявність шляху:
>
> ```sh
> cat /sys/class/power_supply/battery/capacity
> ```
>
> Якщо файл відсутній або недоступний для читання, цей датчик на пристрої недоступний.

Відредагуйте `configuration.yaml` і додайте блок `command_line:`. Якщо ключ `command_line:` уже є у файлі, додайте нові елементи під наявним ключем; не додавайте ключ удруге.

```sh
nano "$HA_CONFIG_DIR/configuration.yaml"
```

```yaml
command_line:

  # --- Датчики температури ---

  - sensor:
      name: "RN8T: темп. процесора"
      unique_id: rn8t_cpu_temperature
      # Динамічно визначає індекс термозони основного кластера процесора.
      # grep знаходить зону, у файлі type якої міститься 'cpuss-0-usr'.
      command: "cat /sys/class/thermal/thermal_zone$(grep -l 'cpuss-0-usr' /sys/class/thermal/thermal_zone*/type | grep -oE '[0-9]+')/temp"
      unit_of_measurement: "°C"
      device_class: temperature
      state_class: measurement
      # Ядро повертає температуру в міліградусах Цельсія; ділимо на 1000.
      value_template: "{{ (value | float / 1000) | round(1) }}"
      scan_interval: 30

  - sensor:
      name: "RN8T: темп. акумулятора"
      unique_id: rn8t_battery_temperature
      command: "cat /sys/class/power_supply/battery/temp"
      unit_of_measurement: "°C"
      device_class: temperature
      state_class: measurement
      # Драйвер акумулятора повертає температуру в десятих частках градуса; ділимо на 10.
      value_template: "{{ (value | float / 10) | round(1) }}"
      scan_interval: 30

  - sensor:
      name: "RN8T: темп. eMMC"
      unique_id: rn8t_emmc_temperature
      # Динамічно визначає індекс термозони датчика eMMC/UFS.
      command: "cat /sys/class/thermal/thermal_zone$(grep -l 'emmc-ufs-therm-adc' /sys/class/thermal/thermal_zone*/type | grep -oE '[0-9]+')/temp"
      unit_of_measurement: "°C"
      device_class: temperature
      state_class: measurement
      value_template: "{{ (value | float / 1000) | round(1) }}"
      # Температура сховища змінюється повільно; опитування раз на 60 секунд достатньо.
      scan_interval: 60

  # --- Датчики акумулятора та живлення ---

  - sensor:
      name: "RN8T: рівень заряду акумулятора"
      unique_id: rn8t_battery_level
      command: "cat /sys/class/power_supply/battery/capacity"
      unit_of_measurement: "%"
      device_class: battery
      state_class: measurement
      value_template: "{{ value | int }}"
      # Якщо ліміт заряджання встановлено на 70% (крок 02), це значення зупиниться на 70.
      scan_interval: 60

  - sensor:
      name: "RN8T: струм акумулятора"
      unique_id: rn8t_battery_current
      command: "cat /sys/class/power_supply/battery/current_now"
      unit_of_measurement: "mA"
      device_class: current
      state_class: measurement
      # Драйвер Qualcomm повертає струм у мікроамперах (µA); ділимо на 1000 для отримання мА.
      # Додатне значення означає заряджання; від’ємне — розряджання (споживання).
      value_template: "{{ (value | float / 1000) | round(1) }}"
      # Короткий інтервал забезпечує точніші графіки заряджання та розряджання.
      scan_interval: 10

  - sensor:
      name: "RN8T: кількість циклів акумулятора"
      unique_id: rn8t_battery_cycle_count
      command: "cat /sys/class/power_supply/battery/cycle_count"
      icon: mdi:battery-sync
      # total_increasing: значення збільшується протягом усього строку служби акумулятора.
      state_class: total_increasing
      value_template: "{{ value | int }}"
      # Кількість циклів змінюється рідко; погодинне опитування зменшує зайві записи.
      scan_interval: 3600
```

У `nano` натисніть `Ctrl+O`, потім Enter, щоб зберегти; натисніть `Ctrl+X`, щоб вийти.

> [!IMPORTANT]
> **Потрібен повний перезапуск Home Assistant:**
> Після додавання нових датчиків `command_line` або зміни унікальних ID наявних датчиків потрібен **повний перезапуск Home Assistant** (**Settings → System → Restart** або перезапуск служби). Простого перезавантаження YAML-конфігурації недостатньо, щоб нові сутності з’явилися в Home Assistant.

Перед перезапуском перевірте конфігурацію:

```sh
hass --script check_config --config "$HA_CONFIG_DIR"
```

Після успішної перевірки перезапустіть Home Assistant. Датчики з указаними назвами з’являться в **Settings → Devices & services → Entities**.

> [!NOTE]
> Динамічний пошук термозони (`grep -l … | grep -oE '[0-9]+'`) щоразу визначає правильний індекс зони. Якщо в іншому ядрі одному рядку типу відповідають кілька термозон, команда поверне кілька рядків, і датчик видаватиме помилку. У такому разі знайдіть правильний фіксований індекс командою `grep -r 'cpuss-0-usr' /sys/class/thermal/` і замініть динамічну команду на конкретний шлях, наприклад `cat /sys/class/thermal/thermal_zone4/temp`.

### Необов’язково: розширений моніторинг системи та ресурсів

Наведені нижче необов’язкові датчики надають корисні дані про систему Home Assistant на пристроях Android. Додайте ці записи під розділом `command_line:` у `configuration.yaml`:

```yaml
command_line:

  # --- Розширений моніторинг системи та ресурсів (необов’язково) ---

  - sensor:
      name: "RN8T: доступна оперативна пам’ять"
      unique_id: rn8t_available_ram
      command: "awk '/MemAvailable/ {print int($2/1024)}' /proc/meminfo"
      unit_of_measurement: "MB"
      device_class: data_size
      state_class: measurement
      value_template: "{{ value | int }}"
      scan_interval: 30

  - sensor:
      name: "RN8T: завантаження процесора (1 хв)"
      unique_id: rn8t_cpu_load_1m
      command: "cat /proc/loadavg | awk '{print $1}'"
      icon: mdi:chip
      state_class: measurement
      value_template: "{{ value | float | round(2) }}"
      scan_interval: 15

  - sensor:
      name: "RN8T: частота процесора"
      unique_id: rn8t_cpu_frequency
      command: "cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq 2>/dev/null || echo 0"
      unit_of_measurement: "MHz"
      state_class: measurement
      value_template: "{{ (value | float / 1000) | round(0) | int }}"
      scan_interval: 15

  - sensor:
      name: "RN8T: вільне місце у внутрішній пам’яті"
      unique_id: rn8t_internal_storage_free
      command: "df -m /data | awk 'NR==2 {print $4}'"
      unit_of_measurement: "MB"
      device_class: data_size
      state_class: measurement
      value_template: "{{ value | int }}"
      scan_interval: 300

  - sensor:
      name: "RN8T: розмір бази даних HA"
      unique_id: rn8t_ha_database_size
      command: "[ -f /mnt/ha_native/home-assistant_v2.db ] && du -m /mnt/ha_native/home-assistant_v2.db | awk '{print $1}' || du -m /root/.homeassistant/home-assistant_v2.db 2>/dev/null | awk '{print $1}'"
      unit_of_measurement: "MB"
      device_class: data_size
      state_class: measurement
      value_template: "{{ value | int }}"
      scan_interval: 300

  - sensor:
      name: "RN8T: стан монтування сховища"
      unique_id: rn8t_storage_mount_status
      command: "mountpoint -q /mnt/ha_native && echo 'Змонтовано (SD)' || echo 'Внутрішнє сховище'"
      icon: mdi:harddisk
      scan_interval: 60

  - sensor:
      name: "RN8T: час роботи системи"
      unique_id: rn8t_system_uptime
      command: "cat /proc/uptime | awk '{print int($1)}'"
      unit_of_measurement: "s"
      device_class: duration
      state_class: total_increasing
      value_template: "{{ value | int }}"
      scan_interval: 60
```

> [!NOTE]
> Підтримка обладнання та шляхи файлів sysfs залежать від моделі пристрою Android і збірки ядра. Наприклад, доступність частоти процесора залежить від драйверів ядра `cpufreq`. Якщо будь-який датчик повертає недоступне значення чи помилку, перед додаванням перевірте базову команду в терміналі Debian chroot. Пам’ятайте, що після додавання датчиків потрібен повний перезапуск Home Assistant.

## Журнали

Процедуру оновлення Home Assistant і створення резервної копії дивіться у [кроці 11](11_Update_cleanup.md). Основні журнали:

```sh
# Журнал Home Assistant у Debian (використовує спільний робочий каталог):
tail -f "$HA_CONFIG_DIR/home-assistant.log"

# У Termux після встановлення скрипту з кроку 06:
tail -f ~/init.log

# Лише якщо ввімкнено Mosquitto:
tail -f ~/.local/share/mosquitto/mosquitto.log
```

