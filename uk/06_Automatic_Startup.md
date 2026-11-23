# Home Assistant на Android із нативним chroot: 06 — Автоматичний запуск і watchdog

Англійська версія: [06_Automatic_Startup.md](../06_Automatic_Startup.md).


Перш ніж виконувати цей крок, Home Assistant має хоча б один раз успішно запуститися на [кроці 04](04_Home_Assistant.md); для варіанта із SD-карткою також потрібно перевірити bind-монтування. Інакше повторні спроби лише приховуватимуть помилку запуску. Перед постійним використанням перегляньте блок **Налаштування користувача**.

Після перезавантаження Termux:Boot запускає файли з `~/.termux/boot/` в алфавітному порядку, якщо застосунок відкривали принаймні один раз. Цю поведінку описано в [офіційному README Termux:Boot](https://github.com/termux/termux-boot/blob/master/README.md).

У цій папці залишайте лише скрипти, які мають запускатися автоматично. Termux:Boot може виконати й резервні копії з розширеннями на кшталт `.bak`. Зберігайте копії поза boot-каталогом, наприклад у `~/boot-backups/`.

## 1. Створіть скрипт

У звичайному Termux (не в Debian і не в оболонці `su`) створіть каталог завантажувальних скриптів:

```sh
mkdir -p ~/.termux/boot
```

Потім відкрийте скрипт ініціалізації в `nano`:

```sh
nano ~/.termux/boot/00-init.sh
```

Укажіть локальні значення конфігурації. UUID SD-картки потрібен лише за `USE_SD_CARD=1`.

```sh
#!/data/data/com.termux/files/usr/bin/sh

# Запуск служб у нативному chroot через Termux:Boot у глобальному просторі монтування.
#
# Що виконує цей скрипт:
#   1. Запускає необов'язкові служби (sshd, Mosquitto, Chrony) і готує сховище.
#   2. Запускає Home Assistant у Debian chroot і чекає на HTTP-відповідь вебінтерфейсу.
#   3. Запускає watchdog: перевіряє HA через HTTP і (за потреби) з'єднання Wi-Fi/LAN.
#      Відновлення мережі перезапускає Wi-Fi; перезавантаження телефона через збій мережі вимкнене під час тестування.
#      Помилки HTTP-перевірки HA все ще можуть перезавантажити телефон, з обмеженням MAX_REBOOT_ATTEMPTS_PER_WINDOW.
#
# Режим обслуговування (призупиняє перевірки; нічого не перезапускається; зручно для кнопок Termux:Widget):
#   touch ~/.ha-maintenance      # призупинити watchdog
#   rm -f ~/.ha-maintenance      # продовжити; перед підрахунком збоїв скрипт чекає на відповідь HA
# Повністю зупинити цей скрипт (сам HA продовжить працювати):
#   kill "$(cat ~/.00-init.pid)"

PATH="/data/data/com.termux/files/usr/bin:$PATH"
export PATH

# ---------- Налаштування користувача ----------
SU_CMD="su -mm -c"  # Використати головний простір монтування, щоб уникнути ізоляції монтувань
CHROOT_BIN="/system/bin/chroot-distro"
# Резервний шлях, якщо launcher модуля є, але його overlay у /system/bin відсутній.
CHROOT_BIN_FALLBACK="/data/adb/modules/chroot-distro/system/bin/chroot-distro"
DISTRO="debian"

# 0 = каталог HA у внутрішній пам'яті (/root/.homeassistant).
# 1 = каталог HA на SD-картці ext4 через bind-монтування.
USE_SD_CARD=0
SD_UUID="" # Потрібно лише якщо USE_SD_CARD=1.

# Увімкніть лише після перевірки Mosquitto на кроці 05.
ENABLE_MOSQUITTO=0
MOSQUITTO_CONFIG="$HOME/.config/mosquitto/mosquitto.conf"

# Встановіть обидва перемикачі Chrony у 0, щоб повністю його вимкнути. Перевірку справності можна вимкнути окремо,
# залишивши службу запущеною: для цього задайте ENABLE_CHRONY_CHECK=0.
# Chrony обслуговує клієнтів LAN через UDP-порт 123 і використовує годинник телефона як резерв.
ENABLE_CHRONY=0
CHRONY_BIN="/data/data/com.termux/files/usr/bin/chronyd"
CHRONY_CONFIG="$HOME/.config/chrony/chrony.conf"
TIMEOUT_BIN="/data/data/com.termux/files/usr/bin/timeout"
# Необов'язкова перевірка: задавайте 1 лише за ENABLE_CHRONY=1 і налаштованого кроку 07.
# Виконується під час запуску, а потім кожні 15 хвилин. Перевіряє процес і локальний інтерфейс керування chronyc
# (але не доступність із клієнта LAN); тайм-аут команди — 5 секунд. Збої записуються в журнал і можуть спричинити
# спробу запуску Chrony, але не перезавантажують HA чи телефон.
ENABLE_CHRONY_CHECK=0
CHRONY_CHECK_INTERVAL=900 # секунд між перевірками після першої під час запуску
CHRONYC_BIN="/data/data/com.termux/files/usr/bin/chronyc"
# Внутрішній стан необов'язкової перевірки Chrony; зміни стану записуються без повторного спаму журналу.
CHRONY_CHECK_STATE="unknown"
CHRONY_LAST_CHECK=0

# Увімкніть, лише якщо SSH потрібен і захищений ключами або паролем.
ENABLE_SSHD=1

# --- Перевірка доступності Home Assistant ---
# Перевірка надсилає HTTP-запит, а не шукає процес. Будь-яка відповідь зі статусом нижче 500
# (200, redirect, 401, 403, 404 тощо) означає, що вебсервер HA відповідає; відсутність відповіді,
# тайм-аут або статус 5xx означає збій. Кілька URL допомагають пережити зміну порту або SSL
# (порт можна не вказувати, якщо HA слухає 80/443). Останній URL, що відповів, перевіряється першим.
# Самопідписані сертифікати приймаються. Переконайтеся, що інша служба не слухає ці порти. Якщо в
# секції http HA задано "server_host", додайте цю адресу, наприклад http://192.168.1.5:8123/.
HA_URLS="http://127.0.0.1:8123/ https://127.0.0.1:8123/ http://127.0.0.1/ https://127.0.0.1/"

# 1 = запускати curl від Android root через $SU_CMD (поточний перевірений варіант).
# 0 = запускати curl від звичайного користувача Termux; перемикайте лише після успішних тестів.
HEALTH_CHECK_VIA_SU=1
# За HEALTH_CHECK_VIA_SU=1 скрипт також виконує такий самий запит від Termux і записує зміну стану.
# Перед переходом на перевірку лише від Termux протестуйте її з вимкненим екраном і від батареї.
# Залишайте HEALTH_CHECK_VIA_SU=1, доки тривалий тест не пройде успішно.
HEALTH_CHECK_SHADOW=1
# У PATH root через su немає каталогу Termux, тому використовується системний curl Android.
CURL_SU_BIN="/system/bin/curl"
# Використовується за HEALTH_CHECK_VIA_SU=0.
CURL_BIN="/data/data/com.termux/files/usr/bin/curl"

# Регулярний вираз знаходить справжній процес hass (шаблон [h] не дає pgrep знайти сам себе).
# Він запобігає запуску другого HA та допомагає виявити аварію під час старту.
# Перевірка: su -c "ps -A -o PID,ARGS" | grep '[h]ass'
HA_PROC_PATTERN='/srv/homeassistant/bin/[h]ass'

# --- Інтервали watchdog ---
# Після запуску HA перевіряти його кожні STARTUP_POLL_INTERVAL секунд, до STARTUP_TIMEOUT секунд.
STARTUP_TIMEOUT=900
STARTUP_POLL_INTERVAL=10
# Далі перевіряти кожні CHECK_INTERVAL секунд; перезавантажити після MAX_CONSECUTIVE_FAILURES збоїв поспіль.
CHECK_INTERVAL=60
MAX_CONSECUTIVE_FAILURES=5

# --- Обмежувач перезавантажень ---
# Не більше MAX_REBOOT_ATTEMPTS_PER_WINDOW спроб команди reboot за REBOOT_WINDOW_SECONDS.
# Спроба записується до команди Android: reboot може перервати оболонку раніше за запис.
# Невдалі команди також рахуються, щоб обмежити повтори, якщо Android відхиляє reboot.
MAX_REBOOT_ATTEMPTS_PER_WINDOW=3
REBOOT_WINDOW_SECONDS=3600
REBOOT_ATTEMPTS_LOG="$HOME/.ha-reboot-attempts"

# --- Режим обслуговування ---
# Поки файл існує, watchdog призупинено. Після reboot його видаляють; прапорець старший за
# MAINTENANCE_MAX_MINUTES ігнорується, щоб забутий файл не вимкнув watchdog назавжди (0 = без строку).
MAINT_FLAG="$HOME/.ha-maintenance"
MAINTENANCE_MAX_MINUTES=180

# --- Перевірка Wi-Fi / LAN ---
# Спершу скрипт перезапускає Wi-Fi. Перезавантаження телефона через мережевий збій керується окремо
# параметром NETWORK_REBOOT_ENABLED. Під час відновлення Wi-Fi збої HA не рахуються.
ENABLE_NETWORK_CHECK=1
WIFI_IFACE="wlan0"
# Що перевіряти ping: "" = автоматично знайдений шлюз WIFI_IFACE; IP/ім'я вузла = задана адреса
# (наприклад, роутер 192.168.1.1); "none" = лише перевірити IPv4-адресу WIFI_IFACE, якщо роутер
# не відповідає на ping.
NET_CHECK_HOST=""
NET_MAX_FAILURES=3          # невдалі перевірки поспіль до першого перезапуску Wi-Fi
WIFI_RESTART_ATTEMPTS=2     # кількість спроб перезапуску Wi-Fi
NET_RECOVERY_WAIT=60        # секунд очікування мережі після кожного перезапуску Wi-Fi
# Залишайте 0 під час тестування: скрипт перезапускає Wi-Fi, але збій мережі сам не запускає reboot.
# Помилки перевірки HA все ще можуть призвести до перезавантаження телефона.
NETWORK_REBOOT_ENABLED=0

# --- Журнал ---
LOG_FILE="$HOME/init.log"
LOG_MAX_BYTES=1048576       # після перевищення цього розміру init.log ротують у init.log.1
PID_FILE="$HOME/.00-init.pid"

# ---------- Внутрішній стан (не редагуйте) ----------
# Останній URL HA, що відповів
HA_URL_ACTIVE=""
# Лічильники поточного періоду
FAILURES=0                  # невдалі перевірки HA поспіль
NET_FAILURES=0              # невдалі перевірки мережі поспіль
WIFI_RESTARTS=0             # перезапуски Wi-Fi під час поточного збою мережі
NET_ARMED=0                 # 1 після першої успішної перевірки мережі
NET_GW=""                   # кешована адреса шлюзу
NET_GW_WARNED=0
NET_GIVEUP_LOGGED=0
SHADOW_STATE=""              # результат останньої діагностичної перевірки: ok / bad
IN_MAINTENANCE=0
REBOOT_REASON=""            # перевірки задають причину, якщо потрібно перезавантажити телефон
REBOOT_BLOCK_LOGGED=0

# ---------- Допоміжні функції ----------
log() {
    printf '%s %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$*" >> "$LOG_FILE"
}

# Очікування, яке сигнал TERM (kill) перериває одразу; звичайний sleep у передньому плані затримав би вихід.
nap() {
    sleep "$1" &
    wait "$!"
}

# Виконати команду в Debian від Android root через головний простір монтування.
chroot_run() {
    if ! $SU_CMD "test -x '$CHROOT_BIN'"; then
        if $SU_CMD "test -x '$CHROOT_BIN_FALLBACK'"; then
            CHROOT_BIN="$CHROOT_BIN_FALLBACK"
        else
            return 127
        fi
    fi
    $SU_CMD "$CHROOT_BIN command $DISTRO \"$1\""
}

# Повертає істину, якщо root-доступ Android загалом працює (допомагає відрізнити несправний su від збою HA/мережі).
su_works() {
    $SU_CMD "true" >/dev/null 2>&1
}

# Обрізати init.log, якщо він завеликий. Файл очищається на місці, а не перейменовується,
# тому запущений процес HA і далі додає записи до того самого файлу.
trim_log() {
    [ -f "$LOG_FILE" ] || return 0
    LOG_SIZE=$(stat -c %s "$LOG_FILE" 2>/dev/null) || return 0
    if [ "$LOG_SIZE" -gt "$LOG_MAX_BYTES" ]; then
        cp "$LOG_FILE" "$LOG_FILE.1" 2>/dev/null
        : > "$LOG_FILE"
        log "Log rotated; the previous content is in $LOG_FILE.1"
    fi
}

# Дозволити лише один екземпляр скрипту. Застарілий PID-файл після reboot або аварії ігнорується.
acquire_lock() {
    if [ -f "$PID_FILE" ]; then
        OLD_PID=$(cat "$PID_FILE" 2>/dev/null)
        if [ -n "$OLD_PID" ] && [ "$OLD_PID" != "$$" ] && kill -0 "$OLD_PID" 2>/dev/null &&
           tr '\0' ' ' < "/proc/$OLD_PID/cmdline" 2>/dev/null | grep -q '00-init.sh'; then
            log "Another 00-init.sh instance is already running (pid $OLD_PID); exiting."
            return 1
        fi
    fi
    echo "$$" > "$PID_FILE"
}

# Повертає істину, поки прапорець обслуговування існує та не сплив.
maintenance_active() {
    [ -e "$MAINT_FLAG" ] || return 1
    [ "$MAINTENANCE_MAX_MINUTES" -gt 0 ] || return 0
    FLAG_TIME=$(stat -c %Y "$MAINT_FLAG" 2>/dev/null) || return 0
    FLAG_AGE=$(( ($(date +%s) - FLAG_TIME) / 60 ))
    if [ "$FLAG_AGE" -ge "$MAINTENANCE_MAX_MINUTES" ]; then
        log "Maintenance flag is older than ${MAINTENANCE_MAX_MINUTES} min; ignoring and removing it."
        rm -f "$MAINT_FLAG"
        return 1
    fi
    return 0
}

# ---------- Перезавантаження з обмежувачем ----------
# Рахує спроби виконати команду перезавантаження, а не підтверджені перезапуски пристрою.
recent_reboot_attempts() {
    [ -f "$REBOOT_ATTEMPTS_LOG" ] || { echo 0; return 0; }
    awk -v now="$1" -v win="$REBOOT_WINDOW_SECONDS" '$1 > now - win { n++ } END { print n + 0 }' "$REBOOT_ATTEMPTS_LOG"
}

# Записати спробу до виклику Android: телефон може перезавантажитися раніше, ніж оболонка встигне записати її.
# Невдалі команди перезавантаження також рахуються, щоб повторні збої не обходили обмежувач.
record_reboot_attempt() {
    {
        [ -f "$REBOOT_ATTEMPTS_LOG" ] && awk -v now="$1" -v win="$REBOOT_WINDOW_SECONDS" '$1 > now - win' "$REBOOT_ATTEMPTS_LOG"
        echo "$1"
    } > "$REBOOT_ATTEMPTS_LOG.tmp" 2>/dev/null && mv "$REBOOT_ATTEMPTS_LOG.tmp" "$REBOOT_ATTEMPTS_LOG"
}

# Перезавантажити телефон, якщо обмежувач спроб це дозволяє. Повертає 0, якщо Android прийняв команду.
reboot_phone() {
    REASON="$1"
    NOW=$(date +%s)
    COUNT=$(recent_reboot_attempts "$NOW")
    if [ "$COUNT" -ge "$MAX_REBOOT_ATTEMPTS_PER_WINDOW" ]; then
        if [ "$REBOOT_BLOCK_LOGGED" -eq 0 ]; then
            log "Reboot suppressed ($REASON): $COUNT reboot attempt(s) already recorded within ${REBOOT_WINDOW_SECONDS}s. Fix the cause manually; monitoring continues."
            REBOOT_BLOCK_LOGGED=1
        fi
        return 1
    fi

    if ! record_reboot_attempt "$NOW"; then
        log "Cannot update $REBOOT_ATTEMPTS_LOG; automatic reboot cancelled so the attempt limiter remains effective."
        return 1
    fi
    log "Rebooting the phone: $REASON"
    sync
    if $SU_CMD 'setprop sys.powerctl reboot' >> "$LOG_FILE" 2>&1 || $SU_CMD 'svc power reboot' >> "$LOG_FILE" 2>&1 || $SU_CMD reboot >> "$LOG_FILE" 2>&1; then
        return 0
    fi
    log "Phone reboot command failed; the watchdog will retry after the next check."
    return 1
}

# ---------- Перевірки Home Assistant ----------
# Вивести HTTP-статус для URL. «000» означає відсутність з'єднання; порожній вивід — curl/su не запустився.
http_status() {
    if [ "$HEALTH_CHECK_VIA_SU" -eq 1 ]; then
        $SU_CMD "'$CURL_SU_BIN' -k -s -o /dev/null -w '%{http_code}' --connect-timeout 5 --max-time 10 '$1'" 2>/dev/null
    else
        "$CURL_BIN" -k -s -o /dev/null -w '%{http_code}' --connect-timeout 5 --max-time 10 "$1" 2>/dev/null
    fi
}

# Повертає 0, якщо HA відповідає, 1 — якщо не відповідає, 2 — якщо не вдалося виконати перевірку (це не збій HA).
ha_healthy() {
    TRIED=" "
    CHECK_RAN=0
    for URL in $HA_URL_ACTIVE $HA_URLS; do
        case "$TRIED" in *" $URL "*) continue ;; esac
        TRIED="$TRIED$URL "
        CODE=$(http_status "$URL")
        [ -n "$CODE" ] && CHECK_RAN=1
        case "$CODE" in
            [1-4][0-9][0-9])
                if [ "$HA_URL_ACTIVE" != "$URL" ]; then
                    log "HA answers at $URL (HTTP $CODE); this URL is used for the checks."
                    HA_URL_ACTIVE="$URL"
                fi
                return 0
                ;;
        esac
    done
    [ "$CHECK_RAN" -eq 1 ] || return 2
    return 1
}

# Діагностична перевірка: той самий запит від користувача Termux; результат лише записується в журнал і не впливає на рішення.
shadow_check() {
    [ "$HEALTH_CHECK_VIA_SU" -eq 1 ] && [ "$HEALTH_CHECK_SHADOW" -eq 1 ] || return 0
    SH_CODE=$("$CURL_BIN" -k -s -o /dev/null -w '%{http_code}' --connect-timeout 5 --max-time 10 "$HA_URL_ACTIVE" 2>/dev/null)
    SH_RC=$?
    case "$SH_CODE" in
        [1-4][0-9][0-9])
            if [ "$SHADOW_STATE" != "ok" ]; then
                log "SHADOW: curl without su works too (HTTP $SH_CODE); su is probably not needed."
                SHADOW_STATE="ok"
            fi
            ;;
        *)
            if [ "$SHADOW_STATE" != "bad" ]; then
                log "SHADOW: curl without su FAILED (code [$SH_CODE], rc=$SH_RC) while the su check passed; keep HEALTH_CHECK_VIA_SU=1."
                SHADOW_STATE="bad"
            fi
            ;;
    esac
}

ha_process_running() {
    $SU_CMD "pgrep -f '$HA_PROC_PATTERN'" >/dev/null 2>&1
}

# Чекати на відповідь HA не довше STARTUP_TIMEOUT секунд або менше, якщо процес hass, який уже запускався,
# зникне (аварійне завершення). Відсутність процесу — лише попередження: помилковий
# HA_PROC_PATTERN не має скорочувати час очікування.
wait_for_ha() {
    WAITED=0
    SEEN=0
    WARNED=0
    CHECK_WARNED=0
    while [ "$WAITED" -lt "$STARTUP_TIMEOUT" ]; do
        ha_healthy
        HA_RC=$?
        if [ "$HA_RC" -eq 0 ]; then
            log "HA is up and responding (waited ${WAITED}s)."
            shadow_check
            if [ "$SEEN" -eq 0 ] && ! ha_process_running; then
                log "WARNING: HA answers, but HA_PROC_PATTERN matches no process; fix the pattern (see the settings comment)."
            fi
            return 0
        fi
        if [ "$HA_RC" -eq 2 ] && [ "$CHECK_WARNED" -eq 0 ]; then
            log "WARNING: the HTTP check cannot run (curl or su problem); see HEALTH_CHECK_VIA_SU, CURL_SU_BIN and CURL_BIN."
            CHECK_WARNED=1
        fi

        if ha_process_running; then
            SEEN=1
        elif [ "$SEEN" -eq 1 ]; then
            log "The hass process exited during startup; see the traceback above in $LOG_FILE."
            return 1
        elif [ "$WAITED" -ge 60 ] && [ "$WARNED" -eq 0 ]; then
            log "WARNING: no hass process found ${WAITED}s after launch. Check $LOG_FILE and HA_PROC_PATTERN."
            WARNED=1
        fi

        nap "$STARTUP_POLL_INTERVAL"
        WAITED=$((WAITED + STARTUP_POLL_INTERVAL))
    done
    log "HA did not respond within ${STARTUP_TIMEOUT}s after launch; the watchdog takes over."
    return 1
}

# Повертає істину, поки збій мережі усувається перезапусками Wi-Fi.
network_recovering() {
    [ "$ENABLE_NETWORK_CHECK" -eq 1 ] && [ "$NET_FAILURES" -gt 0 ] && [ "$WIFI_RESTARTS" -lt "$WIFI_RESTART_ATTEMPTS" ]
}

# Один цикл WatchDog для HA: оновлює FAILURES і задає REBOOT_REASON після досягнення порога.
ha_cycle() {
    ha_healthy
    HA_RC=$?

    if [ "$HA_RC" -eq 0 ]; then
        if [ "$FAILURES" -ne 0 ]; then
            log "HA is responding again; failure counter reset."
        fi
        FAILURES=0
        shadow_check
        return 0
    fi

    if [ "$HA_RC" -eq 2 ]; then
        log "The HTTP check could not run (curl or su problem); not counted as an HA failure."
        return 0
    fi

    if network_recovering; then
        FAILURES=0
        log "HA check failed while the network is being recovered; not counted."
        return 0
    fi

    FAILURES=$((FAILURES + 1))
    # Після порога (перезавантаження заблоковане обмежувачем) записувати лише кожен десятий збій, щоб журнал не розростався.
    if [ "$FAILURES" -le "$MAX_CONSECUTIVE_FAILURES" ] || [ $((FAILURES % 10)) -eq 0 ]; then
        log "HA did not respond to the HTTP check: $FAILURES/$MAX_CONSECUTIVE_FAILURES."
    fi
    if [ "$FAILURES" -ge "$MAX_CONSECUTIVE_FAILURES" ]; then
        REBOOT_REASON="HA failed $FAILURES consecutive HTTP checks"
    fi
}

# ---------- Перевірки Wi-Fi / LAN ----------
# Вивести IPv4-шлюз за замовчуванням для WIFI_IFACE (Android зберігає маршрути за замовчуванням у таблицях інтерфейсів).
detect_gateway() {
    $SU_CMD "ip -4 route show table all" 2>/dev/null | awk -v dev="$WIFI_IFACE" '
        $1 == "default" {
            gw = ""; ifc = ""
            for (i = 2; i < NF; i++) {
                if ($i == "via") gw = $(i + 1)
                if ($i == "dev") ifc = $(i + 1)
            }
            if (ifc == dev && gw != "") { print gw; exit }
        }'
}

ping_host() {
    $SU_CMD "ping -c 2 -W 3 '$1'" >/dev/null 2>&1
}

wifi_has_ip() {
    $SU_CMD "ip -4 addr show dev '$WIFI_IFACE'" 2>/dev/null | grep -q 'inet '
}

# Повертає 0, якщо Wi-Fi-з'єднання працює. Перевіряє доступність роутера (рівні L2/L3), бо HA на
# loopback продовжує відповідати, навіть якщо телефон утратив Wi-Fi.
network_ok() {
    if [ "$NET_CHECK_HOST" != "none" ]; then
        TARGET="$NET_CHECK_HOST"
        [ -n "$TARGET" ] || TARGET="$NET_GW"
        if [ -z "$TARGET" ]; then
            NET_GW=$(detect_gateway)
            TARGET="$NET_GW"
        fi
        if [ -n "$TARGET" ]; then
            ping_host "$TARGET" && return 0
            # DHCP міг призначити роутеру нову адресу: перед завершенням ще раз визначити її.
            if [ -z "$NET_CHECK_HOST" ]; then
                NEW_GW=$(detect_gateway)
                if [ -n "$NEW_GW" ] && [ "$NEW_GW" != "$TARGET" ]; then
                    NET_GW="$NEW_GW"
                    ping_host "$NET_GW" && return 0
                fi
            fi
            return 1
        fi
        # Якщо шлюз невідомий, перейти до перевірки інтерфейсу нижче.
        if [ "$NET_GW_WARNED" -eq 0 ]; then
            log "WARNING: no default gateway found for $WIFI_IFACE; only the interface address is checked. Set NET_CHECK_HOST to the router address to ping it."
            NET_GW_WARNED=1
        fi
    fi
    wifi_has_ip
}

# Вимкнути й знову ввімкнути Wi-Fi, а потім дочекатися мережі. Повертає 0, якщо мережа відновилася.
restart_wifi() {
    log "Network is down; restarting Wi-Fi ($1)."
    $SU_CMD 'svc wifi disable' >> "$LOG_FILE" 2>&1
    nap 5
    $SU_CMD 'svc wifi enable' >> "$LOG_FILE" 2>&1
    WIFI_WAITED=0
    while [ "$WIFI_WAITED" -lt "$NET_RECOVERY_WAIT" ]; do
        nap 5
        WIFI_WAITED=$((WIFI_WAITED + 5))
        if network_ok; then
            log "Network is back after the Wi-Fi restart (${WIFI_WAITED}s)."
            NET_FAILURES=0
            WIFI_RESTARTS=0
            return 0
        fi
    done
    log "Network is still down after the Wi-Fi restart."
    return 1
}

# Один цикл WatchDog для мережі: оновлює лічильники, перезапускає Wi-Fi та задає REBOOT_REASON,
# лише якщо перезапуски Wi-Fi не допомогли.
network_cycle() {
    [ "$ENABLE_NETWORK_CHECK" -eq 1 ] || return 0

    if network_ok; then
        if [ "$NET_FAILURES" -gt 0 ]; then
            log "Network is OK again."
        fi
        if [ "$NET_ARMED" -eq 0 ]; then
            NET_ARMED=1
            if [ "$NETWORK_REBOOT_ENABLED" -eq 1 ]; then
                log "Network check works; network-triggered phone reboot is armed."
            else
                log "Network check works; network-triggered phone reboot remains disabled."
            fi
        fi
        NET_FAILURES=0
        WIFI_RESTARTS=0
        NET_GIVEUP_LOGGED=0
        return 0
    fi

    if ! su_works; then
        log "su is not responding; the network check is skipped."
        return 0
    fi

    NET_FAILURES=$((NET_FAILURES + 1))
    # Після використання всіх способів відновлення записувати лише кожен десятий збій, щоб журнал не розростався.
    if [ "$NET_FAILURES" -le $((NET_MAX_FAILURES + WIFI_RESTART_ATTEMPTS + 1)) ] || [ $((NET_FAILURES % 10)) -eq 0 ]; then
        log "Network check failed ($NET_FAILURES in a row)."
    fi
    [ "$NET_FAILURES" -ge "$NET_MAX_FAILURES" ] || return 0

    if [ "$WIFI_RESTARTS" -lt "$WIFI_RESTART_ATTEMPTS" ]; then
        WIFI_RESTARTS=$((WIFI_RESTARTS + 1))
        restart_wifi "$WIFI_RESTARTS/$WIFI_RESTART_ATTEMPTS"
        return 0
    fi

    if [ "$NETWORK_REBOOT_ENABLED" -eq 1 ] && [ "$NET_ARMED" -eq 1 ]; then
        REBOOT_REASON="network is still down after $WIFI_RESTARTS Wi-Fi restart(s)"
    elif [ "$NET_GIVEUP_LOGGED" -eq 0 ]; then
        log "Network is still down after $WIFI_RESTARTS Wi-Fi restart(s); no reboot (disabled, or the network check never succeeded since start)."
        NET_GIVEUP_LOGGED=1
    fi
}

# ---------- Служби ----------
start_mosquitto() {
    [ "$ENABLE_MOSQUITTO" -eq 1 ] || return 0

    if [ ! -f "$MOSQUITTO_CONFIG" ]; then
        log "Mosquitto is enabled, but the configuration was not found: $MOSQUITTO_CONFIG"
        return 1
    fi

    if pgrep -x mosquitto >/dev/null 2>&1; then
        return 0
    fi

    if mosquitto -c "$MOSQUITTO_CONFIG" -d; then
        log "Mosquitto started."
    else
        log "Failed to start Mosquitto; HA will continue without it."
        return 1
    fi
}

chrony_running() {
    $SU_CMD "pgrep -f '[c]hronyd.*chrony.conf'" >/dev/null 2>&1
}

start_chrony() {
    [ "$ENABLE_CHRONY" -eq 1 ] || return 0

    if [ ! -x "$CHRONY_BIN" ] || [ ! -f "$CHRONY_CONFIG" ]; then
        log "Chrony is enabled, but the daemon or configuration file was not found. Complete step 07 first."
        return 1
    fi

    if chrony_running; then
        return 0
    fi

    # Запустити Chrony від root-користувача Android для UDP-порту 123; параметр -x забороняє змінювати системний годинник.
    # Команда запускається у фоновому режимі, тому її код завершення нічого не говорить про chronyd:
    # трохи згодом перевірити, чи процес справді працює.
    $SU_CMD "nohup '$CHRONY_BIN' -x -f '$CHRONY_CONFIG' >/dev/null 2>&1 </dev/null &"
    sleep 2
    if chrony_running; then
        log "Chrony started."
    else
        log "Chrony did not stay running; check $CHRONY_CONFIG. Home Assistant will continue without the NTP service."
        return 1
    fi
}

chrony_health_check() {
    [ "$ENABLE_CHRONY_CHECK" -eq 1 ] || return 0
    [ "$ENABLE_CHRONY" -eq 1 ] || return 0

    CHRONY_NOW=$(date +%s)
    if [ "$CHRONY_LAST_CHECK" -gt 0 ] &&
       [ $((CHRONY_NOW - CHRONY_LAST_CHECK)) -lt "$CHRONY_CHECK_INTERVAL" ]; then
        return 0
    fi
    CHRONY_LAST_CHECK="$CHRONY_NOW"

    if [ ! -x "$CHRONY_BIN" ] || [ ! -f "$CHRONY_CONFIG" ] || [ ! -x "$CHRONYC_BIN" ] || [ ! -x "$TIMEOUT_BIN" ]; then
        if [ "$CHRONY_CHECK_STATE" != "bad" ]; then
            log "Chrony check is enabled, but a required binary or configuration file is missing."
        fi
        CHRONY_CHECK_STATE="bad"
        return 1
    fi

    if ! chrony_running; then
        if [ "$CHRONY_CHECK_STATE" != "missing" ]; then
            log "Chrony daemon is not running; attempting to start it."
        fi
        CHRONY_CHECK_STATE="missing"
        start_chrony || return 1
    fi

    if $SU_CMD "'$TIMEOUT_BIN' 5 '$CHRONYC_BIN' -h 127.0.0.1 tracking" >/dev/null 2>&1; then
        if [ "$CHRONY_CHECK_STATE" != "ok" ]; then
            log "Chrony control check is responding."
        fi
        CHRONY_CHECK_STATE="ok"
        return 0
    fi

    if [ "$CHRONY_CHECK_STATE" != "bad" ]; then
        log "Chrony control check failed; no phone reboot will be requested."
    fi
    CHRONY_CHECK_STATE="bad"
    return 1
}

prepare_storage() {
    if [ "$USE_SD_CARD" -eq 0 ]; then
        HA_CONFIG_IN_CHROOT="/root/.homeassistant"
        return 0
    fi

    if [ -z "$SD_UUID" ]; then
        log "USE_SD_CARD=1, but SD_UUID is empty. Startup cancelled."
        return 1
    fi

    CARD_SOURCE="/mnt/media_rw/$SD_UUID/ha_config"
    CHROOT_TARGET="/data/local/chroot-distro/$DISTRO/mnt/ha_native"
    ATTEMPT=0

    # Android vold може змонтувати картку із затримкою після завантаження.
    while [ "$ATTEMPT" -lt 30 ]; do
        if $SU_CMD "test -d '$CARD_SOURCE'"; then
            break
        fi
        ATTEMPT=$((ATTEMPT + 1))
        log "Waiting for the SD card: attempt $ATTEMPT/30."
        sleep 2
    done

    if ! $SU_CMD "test -d '$CARD_SOURCE'"; then
        log "The SD card or ha_config did not appear within 150 seconds. HA will not start to avoid creating an empty configuration in internal storage."
        return 1
    fi

    if ! $SU_CMD "mkdir -p '$CHROOT_TARGET'"; then
        log "Failed to create the bind-mount target: $CHROOT_TARGET"
        return 1
    fi

    # Перевірити наявність активного монтування в головному просторі монтування.
    if ! $SU_CMD "grep -qs ' $CHROOT_TARGET ' /proc/mounts"; then
        if ! $SU_CMD "mount --bind '$CARD_SOURCE' '$CHROOT_TARGET'"; then
            log "Failed to bind-mount $CARD_SOURCE to $CHROOT_TARGET"
            return 1
        fi
    fi

    HA_CONFIG_IN_CHROOT="/mnt/ha_native"
    log "SD card is ready (Master Mount); HA configuration: $HA_CONFIG_IN_CHROOT"
    return 0
}

start_ha() {
    # Спершу перевірити: команда chroot-distro не запускається, якщо HA вже працює.
    if ha_process_running; then
        log "A hass process is already running; not starting a second instance."
        return 0
    fi

    if ! chroot_run "test -s /srv/homeassistant/constraints.txt" >/dev/null 2>&1; then
        # Відрізнити несправний chroot/su від відсутнього файла: повідомлення підкаже правильний спосіб виправлення.
        if chroot_run "true" >/dev/null 2>&1; then
            log "/srv/homeassistant/constraints.txt was not found. Restore the uv file; HA startup cancelled."
        else
            log "Cannot run commands in the $DISTRO chroot (su denied, chroot-distro failed, or the rootfs is unavailable); HA startup cancelled."
        fi
        return 1
    fi

    log "Starting Home Assistant from $HA_CONFIG_IN_CHROOT (activate)"
    chroot_run "
        export UV_LINK_MODE=copy &&
        export UV_CONSTRAINT=/srv/homeassistant/constraints.txt &&
        cd /root &&
        . /srv/homeassistant/bin/activate &&
        exec hass --config $HA_CONFIG_IN_CHROOT
    " >> "$LOG_FILE" 2>&1 &
    return 0
}

# ---------- Основний блок ----------
acquire_lock || exit 0
trap 'rm -f "$PID_FILE"' EXIT
trap 'exit 1' HUP INT TERM
# Отримати блокування єдиного екземпляра до очищення спільного журналу, щоб паралельні запуски не конфліктували.
trim_log

# Прапорець обслуговування не має зберігатися після перезавантаження.
if [ -e "$MAINT_FLAG" ]; then
    rm -f "$MAINT_FLAG"
    log "Removed a leftover maintenance flag."
fi

termux-wake-lock >> "$LOG_FILE" 2>&1 || log "Failed to acquire the Termux wake lock."
log "===== 00-init.sh start ====="

if [ "$ENABLE_SSHD" -eq 1 ] && ! pgrep -x sshd >/dev/null 2>&1; then
    sshd >> "$LOG_FILE" 2>&1 || log "Failed to start sshd."
fi

prepare_storage || exit 1
start_mosquitto || true
start_chrony || true
chrony_health_check || true

# Дати Android час завершити ініціалізацію перед запуском chroot.
log "Waiting 10 seconds before starting the chroot environment..."
sleep 10

start_ha || exit 1

log "Waiting for HA to answer (up to ${STARTUP_TIMEOUT} s)."
wait_for_ha

log "Watchdog started: check every ${CHECK_INTERVAL} s, reboot after ${MAX_CONSECUTIVE_FAILURES} failed HA checks in a row."
while :; do
    if maintenance_active; then
        if [ "$IN_MAINTENANCE" -eq 0 ]; then
            log "Maintenance mode ON; checks are paused."
            IN_MAINTENANCE=1
        fi
        FAILURES=0
        NET_FAILURES=0
        WIFI_RESTARTS=0
        nap 10
        continue
    fi

    if [ "$IN_MAINTENANCE" -eq 1 ]; then
        IN_MAINTENANCE=0
        log "Maintenance mode OFF; waiting for HA before the checks resume."
        wait_for_ha
    fi

    trim_log
    start_mosquitto || true
    chrony_health_check || true

    REBOOT_REASON=""
    network_cycle
    ha_cycle

    if [ -n "$REBOOT_REASON" ]; then
        if reboot_phone "$REBOOT_REASON"; then
            exit 0
        fi
    else
        REBOOT_BLOCK_LOGGED=0
    fi

    nap "$CHECK_INTERVAL"
done
```

У `nano` натисніть `Ctrl+O`, потім Enter, щоб зберегти; натисніть `Ctrl+X`, щоб вийти. Надайте скрипту права та перевірте синтаксис, не запускаючи служби:

```sh
chmod 700 ~/.termux/boot/00-init.sh
sh -n ~/.termux/boot/00-init.sh
```

## 2. Виберіть правильну конфігурацію

### Без SD-картки

Перед збереженням скрипту вкажіть:

```sh
USE_SD_CARD=0
SD_UUID=""
```

У цьому режимі скрипт не перевіряє `/mnt/media_rw`, не виконує `mount --bind` і запускає HA з `/root/.homeassistant`, навіть якщо SD-картку не вставлено. Не додавайте параметри SD-картки «про всяк випадок».

### Із SD-карткою

Лише після того, як на кроці 04 картку відформатовано, Android автоматично її підключив і ручне bind-монтування перевірено, укажіть:

```sh
USE_SD_CARD=1
SD_UUID="the-actual-uuid"
```

Скрипт чекатиме до 150 секунд на `/mnt/media_rw/<UUID>/ha_config`. Якщо картка не готова, HA навмисно не запускається; це запобігає випадковому старту з новим порожнім каталогом у внутрішній пам’яті.

### Mosquitto

Якщо крок 05 пропущено, залиште:

```sh
ENABLE_MOSQUITTO=0
```

Після ручної перевірки брокера, створення `~/.config/mosquitto/mosquitto.conf` і підключення HA до нього встановіть значення `1`. Скрипт намагається запускати Mosquitto під час завантаження та HTTP-перевірок; сама недоступність MQTT не перезавантажує телефон.

### Chrony

Якщо крок 07 пропущено, залиште обидва перемикачі вимкненими:

```sh
ENABLE_CHRONY=0
ENABLE_CHRONY_CHECK=0
```

Після налаштування та перевірки Chrony задайте обом параметрам `1`, щоб запускати й контролювати службу. Скрипт перевіряє Chrony одразу під час запуску, а потім кожні 15 хвилин. Запит `chronyc tracking` має тайм-аут 5 секунд. Задайте `ENABLE_CHRONY_CHECK=0`, щоб залишити службу запущеною без періодичних перевірок; для вимкнення Chrony у скрипті задайте обом параметрам `0`.

Перевірка справності контролює процес і локальний інтерфейс Chrony. Вона не підтверджує доступність UDP-порту 123 для інших пристроїв. Перевірте NTP із Windows PowerShell на ПК у тій самій LAN:

```powershell
w32tm /stripchart /computer:<phone_lan_ip> /dataonly /samples:5
```

Замініть `<phone_lan_ip>` LAN-адресою телефона. Успішні вибірки підтверджують, що NTP-сервер відповідає цьому ПК; показане відхилення — різниця між годинником ПК і часом сервера.

## 3. Перша перевірка та перезавантаження

Перед перезавантаженням ще раз перегляньте налаштування, особливо UUID. Потім перезавантажте телефон.

Через 2–3 хвилини підключіться через SSH і перевірте:

```sh
tail -n 100 ~/init.log
su -c 'curl --head http://127.0.0.1:8123/'
```

Якщо використовується SD-картка, додатково перевірте:

```sh
su -c 'mount | grep ha_native'
su -c '/system/bin/chroot-distro command debian "ls -la /mnt/ha_native"'
```

### Журнал скрипту

Скрипт записує повідомлення про стан і вивід запуску Home Assistant у `~/init.log`.

## Поведінка watchdog та обмеження

Наведений нижче скрипт описує перевірену логіку запуску та має безпечні початкові значення:

- **Сховище:** `USE_SD_CARD=0` вибирає `/root/.homeassistant` і пропускає перевірку та bind-монтування SD-картки. Цей режим працює без картки в телефоні. За `USE_SD_CARD=1` скрипт чекає до 150 секунд на каталог `ha_config` для вказаного UUID і скасовує запуск, якщо картки або каталогу немає; автоматичного переходу на нову порожню конфігурацію у внутрішній пам'яті немає.
- **Запуск Home Assistant:** перевірка процесу не допускає запуску другого екземпляра `hass` і допомагає виявити аварійне завершення під час старту. Поточна доступність визначається HTTP-перевіркою. Очікування старту триває до 900 секунд.
- **HTTP-watchdog:** стандартно перевіряє HA кожні 60 секунд. Відповіді HTTP зі статусом від 100 до 499 вважаються доступним вебсервером; помилки з'єднання та відповіді 5xx — збоями. Після п'яти невдалих перевірок HA поспіль скрипт може перезавантажити телефон. Обмежувач записує спробу до команди й дозволяє не більше трьох спроб за годину; невдалі команди теж рахуються.
- **Відновлення Wi-Fi:** після трьох мережевих збоїв скрипт перезапускає Wi-Fi, до двох спроб, і чекає до 60 секунд після кожної. `NETWORK_REBOOT_ENABLED=0` вимикає перезавантаження, яке ініціює сама мережева перевірка. Помилки HTTP-перевірки HA все ще можуть спричинити reboot після досягнення власного порога.
- **Chrony:** якщо параметр увімкнено, демон запускається під час завантаження. Необов'язкова перевірка справності виконується під час запуску та кожні 900 секунд із тайм-аутом 5 секунд. Доступність NTP у LAN перевіряйте окремо з клієнтського пристрою.
- **Обслуговування та журнал:** `touch ~/.ha-maintenance` призупиняє перевірки watchdog; `rm ~/.ha-maintenance` відновлює їх. Статус запуску та консольний вивід HA записуються у `~/init.log`; при розмірі понад 1 MiB файл ротують. Повну процедуру оновлення й очищення дивіться у [кроці 11](11_Update_cleanup.md).
- **Termux:Boot:** залишайте в `~/.termux/boot/` лише файли, які мають запускатися автоматично. Резервні копії переміщуйте за межі цієї папки: розширення імені не заважає Termux:Boot запускати виконуваний файл.

Щоб тимчасово вимкнути автоматичний запуск, виконайте:

```sh
mv ~/.termux/boot/00-init.sh ~/00-init.sh.bak
```

Щоб відновити його, виконайте:

```sh
mv ~/00-init.sh.bak ~/.termux/boot/00-init.sh
```

