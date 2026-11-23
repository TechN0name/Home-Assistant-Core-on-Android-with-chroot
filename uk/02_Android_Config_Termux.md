# Home Assistant на Android із нативним chroot: 02 — Android, Termux і chroot

Англійська версія: [02_Android_Config_Termux.md](../02_Android_Config_Termux.md).


На цьому кроці підготуємо Android-хост і Debian `chroot`. Home Assistant, SD-картку, Mosquitto та автоматичний запуск поки не налаштовуємо.

## 1. Модулі Magisk

Після успішного запуску Android відкрийте Magisk і на запит надайте Termux root-доступ.

Установіть кожен модуль і після цього перезавантажте телефон:

1. **BusyBox for Android NDK** — версія BusyBox, рекомендована `chroot-distro`.
2. **chroot-distro** з [репозиторію модуля](https://github.com/Magisk-Modules-Alt-Repo/chroot-distro/releases).

`chroot-distro` версії 2.x зберігає дистрибутиви в `/data/local/chroot-distro/` і потребує root-доступу. Не замінюйте його на `proot-distro`: це інший механізм з іншими шляхами й обмеженнями. Команди та вимоги наведено в [документації модуля](https://magisk.dev/modules/chroot-distro/).

## 2. Встановлення Termux і Termux:Boot

Установіть **Termux** і **Termux:Boot** з одного джерела підпису — наприклад, обидва з F-Droid або обидва з офіційних випусків GitHub. Не змішуйте APK з F-Droid і GitHub: ключі підпису різні, через що додатки-плагіни можуть не працювати.

Один раз відкрийте **Termux:Boot**, а потім закрийте його. Це потрібно, щоб Android міг надсилати застосунку події завантаження. Згодом скрипти зберігатимуться в `~/.termux/boot/`; офіційний порядок описано у [README Termux:Boot](https://github.com/termux/termux-boot/blob/master/README.md).

У системних налаштуваннях Android:

- дозвольте Termux і Termux:Boot працювати у фоновому режимі;
- вимкніть для обох застосунків оптимізацію або обмеження роботи акумулятора;
- дозвольте Termux доступ до мережі;
- за наявності забороніть системі агресивно закривати застосунки;
- встановіть ліміт заряджання 70%, щоб зменшити зношування акумулятора через тривалу роботу на максимальній напрузі.

У сучасних збірках LineageOS це налаштовується в **Battery → Charging Control**. Якщо в ROM такого пункту немає, скористайтеся модулем Magisk ACC.

Ці налаштування не гарантують, що Android ніколи не завершить процес через нестачу пам’яті, але значно зменшують таку ймовірність.

## 3. Базове середовище Termux

Відкрийте Termux як звичайний користувач і виконайте:

```sh
pkg update
pkg install -y openssh tmux nano curl wget git iproute2 procps
termux-wake-lock
```

Ці команди готують нове середовище Termux. Для подальшого оновлення пакетів та очищення кешу дивіться [крок 11](11_Update_cleanup.md).

Перевірте локальну IP-адресу телефона:

```sh
su -c 'ip -brief address'
```

Щоб підключатися з ПК, спершу інтерактивно задайте пароль користувача:

```sh
passwd
```

Після введення й підтвердження пароля запустіть SSH-сервер:

```sh
sshd
```

Зазвичай Termux OpenSSH слухає порт `8022`. Підключайтеся з ПК так:

```sh
ssh -p 8022 <termux_user>@<phone_ip>
```

Переконавшись, що підключення працює, замініть автентифікацію за паролем на SSH-ключ. Не відкривайте порт 8022 в Інтернет.

## 4. Перевірка root-доступу й модуля

У Termux перевірте root-доступ і модуль окремими командами:

```sh
su -c id
su -c '/system/bin/chroot-distro help'
su -c '/system/bin/chroot-distro env'
su -c '/system/bin/chroot-distro list'
```

Команда `id` має показати `uid=0(root)`. Якщо Magisk запитає дозвіл на root-доступ, надайте його Termux. Якщо `chroot-distro` відсутній у `/system/bin`, знайдіть його фактичний шлях командою `command -v chroot-distro` у root-оболонці й надалі використовуйте цей шлях.

## 5. Встановлення Debian

Решту команд у цьому розділі виконуйте з Termux через `su -c`:

```sh
su -c '/system/bin/chroot-distro download debian'
su -c '/system/bin/chroot-distro install debian'
```

Після встановлення виконайте діагностику без інтерактивної оболонки:

```sh
su -c '/system/bin/chroot-distro command debian "id; cat /etc/os-release; uname -m"'
```

У результаті мають бути `uid=0(root)`, Debian і 64-бітна архітектура ARM (`aarch64` або `arm64`). Потім увімкніть потрібні параметри модуля:

```sh
su -c '/system/bin/chroot-distro android-bind enable'
su -c '/system/bin/chroot-distro fix-suid enable'
su -c '/system/bin/chroot-distro ram-bind enable'
```

Команда може відповісти `enabled already` — це нормальний результат повторного виконання. `android-bind` надає chroot доступ до шляхів Android. Він не замінює окреме підключення SD-картки на кроці 04.

За бажанням інтерактивно перевірте середовище, увійшовши до Debian:

```sh
su -c '/system/bin/chroot-distro login debian'
```

> [!NOTE]
> **Контексти оболонки:**
> - **Звичайна оболонка Termux (запрошення `$`):** команди виконуються від непривілейованого користувача Termux в Android.
> - **Root-оболонка Termux (запрошення `#` через `su`):** команди виконуються від Android-користувача `root`.
> - **Оболонка Debian chroot (запрошення `root@localhost:~#` через `login debian`):** команди виконуються від `root` у середовищі Debian Linux.
>
> Щоб зберегти файл у `nano`, натисніть `Ctrl+O`, потім `Enter`; щоб вийти з редактора, натисніть `Ctrl+X`.

Завершіть сеанс командою `exit`. Далі в посібнику фраза «у Debian» означає цю root-оболонку.

## Контрольна точка

Переходьте до [кроку 03](03_Debian_Python.md), коли обидві перевірки успішні:

```sh
su -c id
su -c '/system/bin/chroot-distro command debian "id; cat /etc/os-release"'
```

Якщо будь-яка перевірка не пройшла, спершу виправте root-доступ або chroot. Наступні кроки не усунуть проблеми з простором монтування.

