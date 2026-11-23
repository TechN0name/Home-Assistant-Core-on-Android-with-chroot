# 09 — Termux:Widget: ручне керування

Англійська версія: [09_Termux_Widget.md](../09_Termux_Widget.md).


Termux:Widget додає на головний екран кнопки для коротких операцій. У цьому проєкті віджет має кнопки запуску, зупинки й статусу **ESPHome Device Builder**, а також паузи, відновлення, статусу й перегляду журналу WatchDog Home Assistant. Кнопки WatchDog призупиняють лише автоматичні перевірки; вони не зупиняють Home Assistant або `00-init.sh`. Віджет не замінює `00-init.sh` із [кроку 06](06_Automatic_Startup.md).

> [!NOTE]
> Усі команди цього кроку виконуйте у звичайному Termux, а не в Debian чи root-оболонці. `Termux`, `Termux:Boot` і `Termux:Widget` мають походити з одного джерела підпису.

## 1. Підготовка Termux:Widget

Установіть і один раз відкрийте Termux:Widget. Потім у Termux створіть каталог скриптів переднього плану:

```sh
mkdir -p ~/.shortcuts
chmod 700 ~/.shortcuts
```

## 2. Додавання кнопок ESPHome

Використовуйте той самий шлях до `chroot-distro`, що й на кроці 02. Якщо шлях інший, змініть лише `CHROOT_BIN` у кожному скрипті нижче.

### Запуск ESPHome Device Builder

Відкрийте `nano`, щоб створити `~/.shortcuts/esphome-start.sh`:

```sh
nano ~/.shortcuts/esphome-start.sh
```

Вставте такий вміст скрипту:

```sh
#!/data/data/com.termux/files/usr/bin/sh
PATH="/data/data/com.termux/files/usr/bin:$PATH"
export PATH

SU_CMD="su -mm -c"
CHROOT_BIN="/system/bin/chroot-distro"
DISTRO="debian"

is_running() {
    $SU_CMD "$CHROOT_BIN command $DISTRO \"pgrep -f '[e]sphome-device-builder'\"" >/dev/null 2>&1
}

clear
echo "=== ESPHome Device Builder ==="

if is_running; then
    echo "Builder уже запущено."
else
    $SU_CMD "$CHROOT_BIN command $DISTRO \"cd /root && nohup /bin/bash /root/start-esphome.sh > /root/esphome.log 2>&1 &\""
    sleep 2

    if is_running; then
        echo "Builder запущено: http://<phone_ip>:6052"
        echo "Журнал: /root/esphome.log у Debian"
    else
        echo "Не вдалося запустити Builder. Журнал помилок:"
        echo "----------------------------------------"
        $SU_CMD "$CHROOT_BIN command $DISTRO \"cat /root/esphome.log 2>/dev/null\""
        echo "----------------------------------------"
    fi
fi

read -r -p "Натисніть Enter, щоб вийти..." _
```

### Зупинка ESPHome Device Builder

Відкрийте `nano`, щоб створити `~/.shortcuts/esphome-stop.sh`:

```sh
nano ~/.shortcuts/esphome-stop.sh
```

Вставте такий вміст скрипту:

```sh
#!/data/data/com.termux/files/usr/bin/sh
PATH="/data/data/com.termux/files/usr/bin:$PATH"
export PATH

SU_CMD="su -mm -c"
CHROOT_BIN="/system/bin/chroot-distro"
DISTRO="debian"

clear
echo "=== Зупинка ESPHome Device Builder ==="

if $SU_CMD "$CHROOT_BIN command $DISTRO \"pgrep -f '[e]sphome-device-builder'\"" >/dev/null 2>&1; then
    $SU_CMD "$CHROOT_BIN command $DISTRO \"pkill -TERM -f '[e]sphome-device-builder'\""
    echo "Надіслано SIGTERM."
else
    echo "Активний Builder не знайдено."
fi

read -r -p "Натисніть Enter, щоб вийти..." _
```

### Перегляд стану та журналу

Відкрийте `nano`, щоб створити `~/.shortcuts/esphome-status.sh`:

```sh
nano ~/.shortcuts/esphome-status.sh
```

Вставте такий вміст скрипту:

```sh
#!/data/data/com.termux/files/usr/bin/sh
PATH="/data/data/com.termux/files/usr/bin:$PATH"
export PATH

SU_CMD="su -mm -c"
CHROOT_BIN="/system/bin/chroot-distro"
DISTRO="debian"

clear
echo "=== ESPHome Device Builder: стан ==="
$SU_CMD "$CHROOT_BIN command $DISTRO \"
    pgrep -af '[e]sphome-device-builder' || echo 'Builder не запущено.'
    echo
    echo '--- Останні 50 рядків журналу ---'
    tail -n 50 /root/esphome.log 2>/dev/null || true
\""

read -r -p "Натисніть Enter, щоб вийти..." _
```

### Призупинення та відновлення WatchDog Home Assistant

У кроці [06](06_Automatic_Startup.md) уже передбачено прапорець обслуговування `~/.ha-maintenance`. Ці кнопки призупиняють лише перевірки WatchDog; Home Assistant і `00-init.sh` продовжують працювати. У звичайному режимі WatchDog перевіряє прапорець наприкінці кожного циклу (за замовчуванням раз на 60 секунд); під час паузи перевірка відбувається раз на 10 секунд. Після перезавантаження прапорець видаляється, а його дія автоматично завершується після часу `MAINTENANCE_MAX_MINUTES` (за замовчуванням 180 хвилин). Повну процедуру оновлення дивіться у [кроці 11](11_Update_cleanup.md).

Створіть `~/.shortcuts/watchdog-pause.sh`:

```sh
#!/data/data/com.termux/files/usr/bin/sh
MAINT_FLAG="$HOME/.ha-maintenance"
umask 077

echo "=== Обслуговування WatchDog: пауза ==="
if [ -e "$MAINT_FLAG" ]; then
    echo "Пауза вже запитана."
elif : > "$MAINT_FLAG"; then
    echo "Пауза запитана; WatchDog призупинить перевірки наприкінці поточного циклу (за замовчуванням до 60 секунд)."
    echo "Home Assistant і 00-init.sh продовжують працювати."
else
    echo "Не вдалося створити $MAINT_FLAG"
fi
printf 'Натисніть Enter, щоб вийти...'
IFS= read -r _
```

Створіть `~/.shortcuts/watchdog-resume.sh`:

```sh
#!/data/data/com.termux/files/usr/bin/sh
MAINT_FLAG="$HOME/.ha-maintenance"

echo "=== Обслуговування WatchDog: відновлення ==="
if [ ! -e "$MAINT_FLAG" ]; then
    echo "Перевірки WatchDog уже ввімкнені."
elif rm -f "$MAINT_FLAG"; then
    echo "Запитано відновлення; перевірки продовжаться після підтвердження відповіді Home Assistant."
else
    echo "Не вдалося видалити $MAINT_FLAG"
fi
printf 'Натисніть Enter, щоб вийти...'
IFS= read -r _
```

Створіть `~/.shortcuts/watchdog-status.sh`:

```sh
#!/data/data/com.termux/files/usr/bin/sh
MAINT_FLAG="$HOME/.ha-maintenance"

echo "=== Стан WatchDog Home Assistant ==="
if [ -e "$MAINT_FLAG" ]; then
    echo "Прапорець обслуговування: є; паузу WatchDog запитано. Підтвердження перевірте в журналі нижче."
    stat -c 'Паузу запитано: %y' "$MAINT_FLAG" 2>/dev/null || true
else
    echo "Прапорець обслуговування: відсутній; перевірки WatchDog увімкнені."
fi
echo
echo "--- Останні записи журналу WatchDog ---"
grep -E 'Watchdog started|Maintenance mode (ON|OFF)' "$HOME/init.log" 2>/dev/null | tail -n 8 || true
printf 'Натисніть Enter, щоб вийти...'
IFS= read -r _
```

### Перегляд журналу WatchDog і запуску

Створіть `~/.shortcuts/watchdog-log.sh`:

```sh
nano ~/.shortcuts/watchdog-log.sh
```

Вставте цей скрипт:

```sh
#!/data/data/com.termux/files/usr/bin/sh
LOG_FILE="$HOME/init.log"

echo "=== Журнал 00-init.sh / WatchDog: останні 100 рядків ==="
if [ -r "$LOG_FILE" ]; then
    tail -n 100 "$LOG_FILE"
else
    echo "Журнал не знайдено або немає доступу: $LOG_FILE"
fi
printf 'Натисніть Enter, щоб вийти...'
IFS= read -r _
```

Ця кнопка показує файл Termux `~/init.log` із повідомленнями запуску й WatchDog. Журнал самого Home Assistant зберігається окремо в активному каталозі конфігурації, описаному в [Notes](Notes.md).

Надайте права на виконання всім семи скриптам:

```sh
chmod 700 ~/.shortcuts/*.sh
```

### Дозвіл Termux:Widget відкривати термінальні сеанси

Якщо після натискання кнопки переднього плану з’являється повідомлення, що Termux потрібен дозвіл **«Показ поверх інших програм»**, надайте його саме **Termux** (пакет `com.termux`), а не Termux:Widget.

**Через налаштування:** Якщо під час спроби ввімкнути перемикач з’явиться повідомлення «App was denied access», закрийте його. Потім відкрийте **Налаштування → Програми → Termux → ⋮ → Дозволити обмежені налаштування** та підтвердьте дію, якщо Android попросить. Поверніться до **Спеціальний доступ → Показ поверх інших програм → Termux** і ввімкніть дозвіл. Назви пунктів можуть різнитися залежно від ROM. В Android 16 цей дозвіл вважається обмеженим для застосунків, позначених як установлені з локального APK. Див. [довідку Android про обмежені налаштування](https://support.google.com/android/answer/12623953?hl=en) та [вимоги сумісності Android 16](https://source.android.com/docs/compatibility/16/android-16-cdd).

**Запасний спосіб із root:** Якщо Android не показує підтвердження в налаштуваннях або відмовляє в ньому, виконайте ці команди у звичайній оболонці Termux (не в Debian). Вони встановлюють і перевіряють app-op лише для Termux:

```sh
su -mm -c '/system/bin/appops set com.termux SYSTEM_ALERT_WINDOW allow'
su -mm -c '/system/bin/appops get com.termux SYSTEM_ALERT_WINDOW'
```

Очікуваний результат перевірки містить `SYSTEM_ALERT_WINDOW: allow`. Це надає Termux можливість показувати вікна поверх інших програм, щоб віджет міг відкривати термінальні сеанси; використовуйте команду лише для довіреної установки Termux.

## 3. Додавання віджета

На головному екрані Android затисніть порожнє місце → **Widgets** → **Termux:Widget** → перетягніть віджет на екран. Якщо нові скрипти одразу не з’явилися, натисніть кнопку оновлення віджета.

