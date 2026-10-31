# 🛠️ Ubuntu Resolute Hardened Workstation

Данный документ описывает архитектуру и процесс сборки экспертной рабочей станции на базе **Ubuntu 26.04 LTS (Resolute Raccoon)**.

Система спроектирована для DevOps и NetOps-инженеров, с акцентом на:

* **Абсолютный детерминизм и воспроизводимость** (сборка с нуля через `debootstrap`).
* **Безопасность на уровне ядра и диска** (UEFI, Secure Boot, LUKS2).
* **Индустриальный стек хранения** (LVM поверх LUKS, файловая система XFS).
* **Строгую сегрегацию процессов**: четкое разделение зон ответственности.
* **Отказ от мусора**: полная блокировка Snap, неинтерактивный APT без `Recommends`, отказ от NetworkManager в пользу `systemd-networkd`.
* **Отказоустойчивость и восстановление**: автоматические резервные копии и возможность загрузки с предыдущего ядра.

---

## 🖥️ Этап 0: Инвентаризация аппаратной платформы

Прежде чем размечать диск, фиксируем реальную конфигурацию железа. Это критично: от неё зависит выбор firmware-пакетов и драйверов, а лишние «на всякий случай» пакеты в hardened-системе — это лишняя поверхность атаки и лишние мегабайты.

### Собранное железо (по `lspci`, `lsusb`, `/proc/asound/cards`)

| Компонент | Модель | Что даёт для сборки |
| :--- | :--- | :--- |
| **CPU** | Intel Core i5-11400 (Rocket Lake, 6C/12T) | `thermald`, `intel-microcode` (транзитивно). `amd64-microcode` не нужен, но приедет с ядром. |
| **iGPU** | Отсутствует (отключён в BIOS) | `va-driver-all` **не нужен**. `linux-firmware-intel-graphics` — тоже. |
| **GPU** | NVIDIA GeForce RTX 3070 (GA104, LHR) | Драйвер ставится отдельно. `linux-firmware-nvidia-graphics` **не нужен** — GSP-прошивка идёт в самом драйвере Nvidia. |
| **Wi-Fi + BT** | Intel AX210 (Wi-Fi 6E, BT 5.4) | `linux-firmware-intel-wireless` + `linux-firmware-intel-misc` (BT-прошивки). Интерфейс: `wlp3s0`. |
| **Ethernet** | Intel I219-V (`e1000e`) | Прошивка в ядре, `linux-firmware-realtek` **не нужен**. |
| **NVMe** | ADATA SX8100NP (контроллер Realtek RTS5762) | Прошивка в ядре, `linux-firmware-realtek` не нужен и для NVMe. |
| **Audio** | Intel HDA (ALC662) + NVIDIA HDMI + BRIO USB | `firmware-sof-signed` **нужен** (`snd_sof_intel_hda` реально загружен). |
| **Камера** | Logitech BRIO (USB, UVC) | Стандартный `uvcvideo`. `libcamera-ipa`, `pipewire-libcamera`, `gstreamer1.0-libcamera` **не нужны**. Нужен `v4l-utils` для диагностики. |
| **TPM** | TPM 2.0 (`/dev/tpm0` есть) | Опционально `tpm2-tools` для LUKS-разблокировки через `systemd-cryptenroll`. |

### Скрипт инвентаризации

Перед разметкой сохраните «слепок» железа — пригодится при отладке и для документирования:

```bash
{
  echo "===== CPU ====="            ; lscpu | grep -E 'Model name|Vendor|Thread|Core'
  echo "===== PCI ====="            ; lspci -nn
  echo "===== USB ====="            ; lsusb
  echo "===== Block devices ====="  ; lsblk -o NAME,SIZE,TYPE,TRAN,MODEL
  echo "===== Network ====="        ; ip -br link
  echo "===== Wi-Fi ====="          ; iw dev 2>/dev/null
  echo "===== Bluetooth ====="      ; hciconfig -a 2>/dev/null
  echo "===== Sound ====="          ; cat /proc/asound/cards 2>/dev/null
  echo "===== TPM ====="            ; ls /sys/class/tpm/ 2>/dev/null
  echo "===== Kernel modules =====" ; lsmod | grep -E '^(iwlwifi|e1000e|amdgpu|i915|nouveau|nvidia|snd_hda|btusb)'
} 2>&1 | tee /tmp/hw-info.txt
```

> [!question] Зачем это в документе?
> Данная руководство описывает шаги установки, которые справедливы для конкретно моей конфигурации.
> Читателю, который повторяет данное руководство, потребуется самостоятельно на основе своего железа адаптировать команды.
> Если позже Wi-Fi или BT перестанут работать — вы сможете точно знать, какой именно модуль ядра и какая прошивка за них отвечали.

### Сегрегация ответственности: Systemd vs S6

В основе системы лежит принцип строгой сегрегации ответственности между компонентами.

* **Уровень 1: Systemd (PID 1)**. Отвечает только за базовые вещи: `udev`, `logind` (критично для прав доступа к GPU и ввода), `cgroups`, файловые системы и **сеть через `systemd-networkd` + `systemd-resolved`**. NetworkManager **не используется** — см. Этап 3.2.
* **Уровень 2: Wayland + Qtile**. Занимается исключительно управлением окнами и тайлингом. Qtile не должен запускать фоновые демоны.
* **Уровень 3: S6 Supervisor**. Все пользовательские фоновые процессы (уведомления, буфер обмена, аудио, политики безопасности, keyring) запускаются и контролируются `s6-svscan` в рамках сессии пользователя. Это даёт микросекундный запуск, автоматический перезапуск упавших сервисов и чистые логи.

> [!question] Почему без NetworkManager?
> NM — удобная, но тяжёлая надстройка. Для стационарного ПК с одним Wi-Fi и одним Ethernet детерминированная связка `systemd-networkd` + `wpa_supplicant` + `systemd-resolved` покрывает 100% задач.
> Конфигурация — файлы в `/etc/systemd/network/`, которые можно версионировать.

---

## 💽 Этап 1: Разметка и шифрование (LUKS2 + LVM + XFS)

Используем GPT и три раздела. `/boot` выносится в отдельный незашифрованный раздел ext4 для предсказуемой работы GRUB и Secure Boot.

```bash
# Проверка UEFI (ожидаем UEFI)
[ -d /sys/firmware/efi ] && echo UEFI || echo BIOS

# Разметка: nvme0n1p1 — EFI 1G, nvme0n1p2 — /boot 1G, nvme0n1p3 — LUKS остаток
sudo cfdisk /dev/nvme0n1
```

Создаём контейнер LUKS2 с жёсткими параметрами безопасности:

```bash
sudo cryptsetup luksFormat \
  --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --hash sha512 \
  --iter-time 5000 \
  /dev/nvme0n1p3

sudo cryptsetup open /dev/nvme0n1p3 cryptroot
```

Создаём LVM-тома и форматируем их в **XFS** (стандарт RHEL, отлично работает с тяжелыми I/O нагрузками от ВМ и контейнеров):

```bash
sudo pvcreate /dev/mapper/cryptroot
sudo vgcreate vg0 /dev/mapper/cryptroot

# Под диск ~1TB
sudo lvcreate -L 100G -n root       vg0
sudo lvcreate -L 30G  -n var_log    vg0
sudo lvcreate -L 250G -n libvirt    vg0
sudo lvcreate -L 150G -n containers vg0
sudo lvcreate -l 100%FREE -n home   vg0

sudo mkfs.xfs /dev/vg0/root
sudo mkfs.xfs /dev/vg0/var_log
sudo mkfs.xfs /dev/vg0/libvirt
sudo mkfs.xfs /dev/vg0/containers
sudo mkfs.xfs /dev/vg0/home

sudo mkfs.ext4 /dev/nvme0n1p2
sudo mkfs.vfat -F32 /dev/nvme0n1p1
```

> [!question] Почему XFS, а не ext4?
> Для `var/lib/libvirt` и `var/lib/containers` XFS даёт лучшую параллельность и предсказуемое поведение при больших файлах (qcow2, слои образов). Для `/home` и `/` — тоже нормально.
> ext4 оставлен только под `/boot`, где важнее максимальная совместимость с GRUB.

---

## 📦 Этап 2: Базовая система через debootstrap

Монтируем тома в порядке иерархии (родительские раньше дочерних):

```bash
sudo mount /dev/vg0/root /mnt
sudo mkdir -p /mnt/{boot,home,var/log,var/lib/libvirt,var/lib/containers}
sudo mount /dev/nvme0n1p2 /mnt/boot
sudo mkdir -p /mnt/boot/efi
sudo mount /dev/nvme0n1p1 /mnt/boot/efi
sudo mount /dev/vg0/home /mnt/home
sudo mount /dev/vg0/var_log /mnt/var/log
sudo mount /dev/vg0/libvirt /mnt/var/lib/libvirt
sudo mount /dev/vg0/containers /mnt/var/lib/containers
```

Скачиваем и запускаем `debootstrap`:

```bash
sudo debootstrap \
  --arch=amd64 \
  --variant=minbase \
  --keyring=/usr/share/keyrings/ubuntu-archive-keyring.gpg \
  resolute /mnt http://mirror.yandex.ru/ubuntu
```

После завершения установки минимально рабочей системы, подключаем системные директории и входим в среду. Порядок монтирования важен: `/dev`, `/proc`, `/sys` и `/run` должны быть доступны внутри chroot, иначе `apt`, `systemd` и PAM не смогут нормально работать — а нам предстоит ставить пакеты, генерировать initramfs и настраивать сервисы.

```bash
sudo mount --bind /dev     /mnt/dev
sudo mount --bind /dev/pts /mnt/dev/pts
sudo mount --bind /proc    /mnt/proc
sudo mount --bind /sys     /mnt/sys
sudo mount --bind /run     /mnt/run
sudo chroot /mnt /bin/bash
```

> [!question] Зачем `/run`?
> Без него `systemctl` в chroot не сможет связаться с systemd (если вы планируете что-то запускать), а `apt` будет ругаться на отсутствующий `/run/lock`.
> В Debian/Ubuntu это классическая ошибка при сборке через `debootstrap`.

---

## 📦 Этап 3: Настройка APT и установка пакетов

Этот этап выполняется внутри `chroot` и состоит из трёх шагов: настройка источников (с решением проблемы «курицы и яйца» для TLS), политика APT (детерминизм и блокировка Snap) и единая команда установки пакетов.

### 🗂️ Этап 3.1: Источники APT в формате DEB822 (HTTPS + официальное зеркало)

Исторический формат APT — одна строка на репозиторий:

```
deb http://archive.ubuntu.com/ubuntu resolute main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu resolute-updates main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu resolute-security main restricted universe multiverse
```

У него три практических недостатка:

1. **Повторение.** Каждый suite — отдельная строка, URI и компоненты дублируются. Переход на другое зеркало — правка всех строк.
2. **Нет структуры для метаданных.** `Signed-By`, `Architectures`, `Enabled: no` нельзя привязать к конкретному репозиторию — только глобально через `apt.conf`.
3. **Хрупкость парсинга.** Пробелы, комментарии, продолжения строк — всё это регулярки, которые легко сломать.

**DEB822** (`.sources`-файлы, RFC822-подобный формат `Ключ: Значение`) решает все три проблемы:

- **Один блок = один репозиторий** со всеми свойствами.
- **`Suites:` принимает список** — один блок покрывает `resolute`, `resolute-updates` и `resolute-security`.
- **`Signed-By:`** — ключ проверки привязан **к конкретному источнику**, а не глобально. Это элемент безопасности: если позже добавите сторонний репозиторий (Mozilla APT для Firefox), он не сможет подписаться ключом Ubuntu.
- **`Enabled: no`** — можно временно отключить источник, не удаляя файл.

С Ubuntu 24.04 DEB822 — **рекомендуемый формат по умолчанию**, старый `sources.list` поддерживается только для совместимости. В 26.04 это норма.

> [!danger] Проблема «курицы и яйца»: сначала ca-certificates
> Я настоятельно рекомендую настраивать репозиторий на использование HTTPS, что я и демонстрирую ниже.
> Но `debootstrap --variant=minbase` **не устанавливает корневые сертификаты**, поэтому `apt update` по HTTPS сразу упадёт с ошибкой `certificate verify failed` — TLS-соединение не с чем проверить.
>
> Решение простое: `debootstrap` уже создал дефолтный `sources.list` (или `.sources`-файл), указывающий на **HTTP**-зеркало.
> Этого достаточно, чтобы поставить сертификаты штатным `apt` — без ручного `dpkg -i` и без временного отката схемы.
>
> ```bash
> apt update && apt install --no-install-recommends -y ca-certificates openssl
> ```
>
> После этого TLS работает, и можно использовать источник на HTTPS.

> [!question] Какое зеркало для Ubuntu выбрать?
> В Российской Федерации настоятельно рекомендую выберать страновое зеркало Ubuntu - `ru.archive.ubuntu.com`.
> Потому что зеркало Яндекса исторически имело проблемы с цепочкой сертификатов и известно ложными предупреждениями `Missing Signed-By` при работе по HTTPS.
> Официальное российское зеркало `ru.archive.ubuntu.com` лишено этих особенностей и использует стандартную инфраструктуру Canonical.

```bash
rm -f /etc/apt/sources.list
cat > /etc/apt/sources.list.d/ubuntu.sources <<'EOF'
Types: deb
URIs: https://ru.archive.ubuntu.com/ubuntu/
Suites: resolute resolute-updates resolute-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
EOF
```

| Поле | Значение | Почему так |
| :--- | :--- | :--- |
| `Types: deb` | Бинарные пакеты (не `deb-src`) | Исходники не нужны для эксплуатации. При необходимости добавите `Types: deb deb-src` для конкретного репозитория. |
| `URIs:` | `https://ru.archive.ubuntu.com/ubuntu/` | Официальное страновое зеркало с поддержкой HTTPS. Не зависит от особенностей сертификатной цепочки Яндекса. |
| `Suites:` | `resolute resolute-updates resolute-security` | **`resolute`** — базовый релиз (заморожен, security туда не попадает). **`resolute-updates`** — стабильные багфиксы. **`resolute-security`** — только security-патчи, приоритетный канал. **`resolute-backports`** сознательно не включён: backports — новые версии из будущего релиза, лишний риск для hardened-станции. |
| `Components:` | `main restricted universe multiverse` | **`main`** — свободное ПО с поддержкой Canonical. **`restricted`** — проприетарные драйверы. **`universe`** — свободное сообщества. **`multiverse`** — несвободное. Все четыре нужны. |
| `Signed-By:` | Путь к ключу | Зеркала подписывают пакеты оригинальными ключами Canonical — тот же ключ, что у `archive.ubuntu.com`. |

> [!info] Про настройку репозитория Ubuntu 
> - **`deb-src`** — исходники не нужны. При необходимости (пересборка пакета с патчем) — отдельный `.sources`-файл.
> - **`resolute-proposed`** — staging-канал, куда попадают обновления до стабилизации. **Никогда не включайте на рабочей станции** — сломает систему в самый неподходящий момент.
> - **`resolute-backports`** — новые версии из будущего релиза. Лишний риск несовместимости для hardened-станции.

Дополнительно зафиксируем политику проверки в отдельном файле — это не меняет поведение по умолчанию (проверка включена), но делает намерение явным и защищает от случайного отключения в будущем:

```bash
cat > /etc/apt/apt.conf.d/99-https-verify <<'EOF'
Acquire::https::Verify-Peer "true";
Acquire::https::Verify-Host "true";
EOF
```

| Опция | Что проверяет |
| :--- | :--- |
| `Acquire::https::Verify-Peer "true"` | Проверяет сертификат сервера против доверенных корневых сертификатов (`/etc/ssl/certs`). |
| `Acquire::https::Verify-Host "true"` | Проверяет, что Common Name (CN) или Subject Alternative Name (SAN) сертификата соответствует запрошенному хосту (`ru.archive.ubuntu.com`). |

Обе опции по умолчанию **уже `true`** — файл нужен как документация и страховка от случайного `false` в других конфигах.

### 🚫 Этап 3.2: Блокировка Snap через pinning

Этот подэтап определяет, какие пакеты APT имеет право устанавливать, и не даёт Snap просочиться в систему. Snap в 26.04 — это второй менеджер пакетов со своим демоном, автообновлением, loopback-устройствами, изолированными окружениями и привязкой к Canonical Store. Для минималистичной, воспроизводимой станции это антипаттерн: ещё один источник обновлений, ещё одна поверхность атаки и ещё одна точка, где поведение системы перестаёт быть предсказуемым. Блокировать его нужно на уровне APT — чтобы запрет работал до того, как пакет попадёт в дерево зависимостей.

> [!question] Почему Snap — это проблема, а не «просто ещё один формат пакетов»?
> Snap-демон ставит свои обновления в обход `apt`, в фоновом режиме, по расписанию Canonical.
> Это значит, что на станции появляется канал обновлений, который вы не контролируете ни через `unattended-upgrades`, ни через собственные пиннинги, ни через фиксацию версий.
> Для станции, которая собирается по фиксированному манифесту, это ломает саму идею воспроизводимости: два прогона сборки в разное время дают разное состояние.

Для блокировки используется механизм приоритетов — pinning. Каждому пакету из каждого репозитория APT присваивает приоритет, по умолчанию 500. Pinning позволяет понизить приоритет (пакет не будет выбран), повысить его (будет выбран вместо других) или **запретить установку** отрицательным приоритетом.

Шкала приоритетов выглядит так:

| Приоритет | Поведение |
| :--- | :--- |
| `1001+` | Установить даже при downgrade (опасно) |
| `990` | Установить, даже если это downgrade |
| `500` (default) | Обычная установка |
| `100` | Установить, только если нет другой версии |
| `0` | Не ставить автоматически, но можно явно |
| **`-1` и ниже** | **Никогда не устанавливать, даже явно** |
| `-10` | То же, но с запасом (старые версии APT трактовали `-1` нестрого) |

Запись `Pin: release a=*` означает, что пин применяется **ко всем версиям** пакета из всех репозиториев: `a=*` — любой archive. Это важно: `snapd` присутствует в `main` и `universe`, а с течением времени может появиться в сторонних репозиториях. Пин по wildcard-маске закрывает все эти каналы сразу. Запрет оформляется так:

```bash
cat > /etc/apt/preferences.d/nosnap.pref <<'EOF'
Package: snapd
Pin: release a=*
Pin-Priority: -10
EOF
```

| Поле | Значение | Почему так |
| :--- | :--- | :--- |
| `Package: snapd` | Единственный пакет, к которому применяется пин | `snapd` — демон Snap. Остальные snap-пакеты (`snap-confine`, `snap-tool`) ставятся только как его зависимости, поэтому блокировки одного `snapd` достаточно. |
| `Pin: release a=*` | Любой archive | `a=*` — wildcard по полю archive. Подходит ко всем версиям из всех репозиториев: `main`, `restricted`, `universe`, будущих сторонних. |
| `Pin-Priority: -10` | Запрет | `-10` вместо `-1`: старые версии APT трактовали `-1` нестрого, а `-10` гарантирует отказ даже при явной команде `apt install snapd`. |

Проверить, что пин применился, можно так:

```bash
apt-cache policy snapd
```

В выводе `Installed:` и `Candidate:` должны показывать `(none)`, а версия пакета — приоритет `-10`. При попытке `apt install snapd` APT ответит `Package 'snapd' has no installation candidate` — это и есть ожидаемое поведение. Если позже понадобится поставить `snapd` осознанно (например, для тестирования), придётся временно повысить приоритет — то есть принять явное решение, а не получить Snap «по умолчанию».

### 🧭 Этап 3.3: Базовая политика APT

Файл базовой политики фиксирует правила APT на станции: минимум лишних пакетов, отсутствие интерактивных вопросов при обновлении, экономия трафика и автоочистка сирот. Без такого файла поведение APT зависит от того, помнит ли администратор флаг `--no-install-recommends` и `-y` в каждой команде; с файлом — становится свойством системы, а не привычкой человека. Это важно в том числе для `unattended-upgrades`, который работает без администратора и следует глобальной политике, а не отдельным флагам. Создаётся файл так:

```bash
cat > /etc/apt/apt.conf.d/00-base-policy <<'EOF'
APT::Install-Recommends "false";
APT::Install-Suggests "false";
Dpkg::Options { "--force-confdef"; "--force-confold"; };
Acquire::Languages "none";
APT::Get::AutomaticRemove "true";
EOF
```


Файлы в каталоге `/etc/apt/apt.conf.d/` читаются в лексикографическом порядке. Префикс `00-` гарантирует, что этот файл будет прочитан **раньше** остальных. Поэтому его значения становятся базой, которую другие файлы при необходимости могут переопределить.

| Параметр | Что делает | Почему так |
| :--- | :--- | :--- |
| `APT::Install-Recommends "false"` | Не ставит Recommended-пакеты | Recommended — «обычно нужно вместе с этим». Ставить их — значит тащить в систему десятки пакетов «на всякий случай». Осознанный минимум лучше, но требует ручного контроля критичных Recommended (см. ниже). |
| `APT::Install-Suggests "false"` | Не ставит Suggested-пакеты | Suggested — «может пригодиться». Почти всегда это мусор вроде `ghostscript`, `fonts-urw-base35`, `python3-tk`. Для минималистичной системы `false` — стандарт. |
| `Dpkg::Options { "--force-confdef"; "--force-confold"; }` | Конфликты конфигов — без интерактивных вопросов | В скриптовой сборке `dpkg` не должен спрашивать «оставить старый конфиг?». `confold` (а не `confnew`) — потому что ваши правки важнее новых дефолтов из пакета. |
| `Acquire::Languages "none"` | Не скачивает переводы описаний | По умолчанию APT качает `Translation-en`, `Translation-ru` и другие — десятки МБ при каждом `apt update`. `none` экономит трафик; английские описания для DevOps-станции достаточны. |
| `APT::Get::AutomaticRemove "true"` | Удаляет сирот при install/upgrade | Меньше мусора, меньше поверхность атаки: старые библиотеки с CVE не висят «на всякий случай». Риск — можно потерять пакет, помеченный `auto` вручную (см. ниже). |

> [!danger] Обратная сторона отключения Recommended
> Recommended — это не «совсем необязательные» пакеты, а «нужные в 90% случаев». Пример: `libpwquality1` рекомендует `cracklib-runtime` — без него ломается смена пароля через `passwd`.
> Поэтому критичные Recommended ставятся **явно, вручную**: `cracklib-runtime`, `publicsuffix`, `mesa-utils`, `xclip` и другие.
> Если пакет нужен, но не является жёсткой зависимостью, его отсутствие проявится в первом же прогоне сценария — это нормальная цена за минимализм.

> [!warning] Обратная сторона `confold`
> `confold` оставляет ваш конфиг при конфликте, но **не добавляет** в него новые опции, появившиеся в новой версии пакета.
> Пример: в новой версии `openssh-server` появилась опция `Include /etc/ssh/sshd_config.d/*.conf` — при `confold` её в старом конфиге не будет, и drop-in файлы перестанут подхватываться.
> Раз в полгода полезно сверять конфликтные файлы:
>
> ```bash
> dpkg-query -W -f='${Conffiles}\n' <package>
> ```
>
> Команда покажет все conffiles пакета; для каждого можно сравнить с версией из пакета через `apt-get download` и `dpkg-deb -x`.

> [!tip] Если нужно сохранить пакет от автоудаления
> `AutomaticRemove` удаляет всё, что помечено как `auto` и больше никому не нужно.
> Если вы ставили пакет как зависимость, а потом он стал нужен сам по себе — пометьте его как ручной, и автоудаление его не тронет:
>
> ```bash
> apt-mark manual <pkg>
> ```
>
> Обратная операция — `apt-mark auto <pkg>`: пакет снова становится кандидатом на автоудаление.

В итоге этот файл делает три вещи. Он фиксирует минимализм (Recommended и Suggested отключены), убирает интерактивность (`confold` вместо вопросов `dpkg`), экономит трафик (`Languages "none"`) и автоматически поддерживает чистоту (`AutomaticRemove`). Цена — ручной контроль критичных Recommended и периодическая сверка conffiles. Это осознанный компромисс: предсказуемость и воспроизводимость важнее удобства «поставилось само».

---

### 📥 Этап 3.4: Единая установка пакетов

Список ниже — результат примерно десяти итераций. На каждом шаге из Suggested/Recommended удалялось всё, что не нужно для заявленного железа и сценария, а затем проверялось, что ничего критичного не ушло. Итог — минимальный набор, в котором каждый пакет обоснован либо конкретным оборудованием (Intel AX210, NVIDIA RTX 3070, Logitech BRIO, TPM 2.0), либо заявленной ролью станции (DevOps/NetOps).

Принципы отбора формулируются так:

1. `--no-install-recommends` вместе с глобальной политикой `APT::Install-Recommends false` означает, что Recommended ставятся только явно.
2. Из Suggested/Recommended добавлено только критичное: `cracklib-runtime` (без него `libpwquality1` ломает смену пароля через `passwd`), `publicsuffix` (TLS-верификация через Public Suffix List), `mesa-utils` (диагностика GPU), `xclip` (X11-буфер для XWayland), `libengine-pkcs11-openssl`, `libsasl2-modules-gssapi-mit`.
3. Сознательно исключены: ImageMagick (тянулся транзитивно через `zbar-tools`), iOS-стек (`libimobiledevice-*`), `upower` (десктоп без батареи), `modemmanager` (нет модема), `tpm2daemon` (TPM не используется для GPG/SSH в базовой поставке), `zbar-tools` (QR-коды не нужны), `libcamera-ipa`/`pipewire-libcamera`/`gstreamer1.0-libcamera` (BRIO — стандартная UVC-камера, эти пакеты для ISP-пайплайнов ноутбучных камер).

Каждый пакет в списке имеет обоснование — полная таблица вынесена в отдельный файл `packages.md`.

```bash
apt update && apt upgrade -y

apt install --no-install-recommends \
  linux-generic-hwe-26.04 grub-efi-amd64-signed shim-signed mokutil efibootmgr secureboot-db os-prober \
  cryptsetup-initramfs lvm2 thin-provisioning-tools xfsprogs xfsdump dosfstools attr initramfs-tools \
  opendoas neovim wget curl unzip zip zstd pigz pbzip2 pixz bzip2 lz4 lzop ca-certificates bash-completion less patch dialog \
  man-db manpages manpages-dev manpages-posix-dev \
  git make binutils gnutls-bin python3-venv python3-rich openssh-client keychain ssh-askpass \
  systemd-resolved wpasupplicant iw libsasl2-modules libsasl2-modules-gssapi-mit nftables publicsuffix dns-root-data \
  pipewire pipewire-pulse pipewire-alsa pipewire-audio wireplumber libpipewire-0.3-modules-extra libpipewire-0.3-common \
  alsa-utils alsa-ucm-conf alsa-topology-conf libasound2-plugins libcanberra-pulse rtkit firmware-sof-signed \
  dbus dbus-user-session libpam-systemd libpam-cap polkitd pkexec libpwquality1 cracklib-runtime \
  libsecret-1-0 keyutils libnss-systemd pass pass-otp oathtool qrencode dirmngr scdaemon pcscd libfido2-1 acl libengine-pkcs11-openssl \
  systemd-timesyncd systemd-oomd systemd-cryptsetup systemd-hwe-hwdb linux-sysctl-defaults \
  thermald lm-sensors i2c-tools fancontrol mesa-vulkan-drivers mesa-utils pciutils usbutils read-edid v4l-utils \
  plymouth plymouth-themes grub-theme-breeze \
  locales console-setup kbd xdg-user-dirs xdg-utils libglib2.0-data libfile-mimeinfo-perl \
  xauth x11-utils x11-xserver-utils xclip \
  librsvg2-common librsvg2-bin gstreamer1.0-plugins-good gstreamer1.0-tools gstreamer1.0-x opus-tools libnotify-bin \
  wl-clipboard fuse3 \
  bpftool linux-tools-7.0.0-42-generic linux-perf \
  unattended-upgrades apt-listchanges needrestart \
  restic zram-tools \
  linux-firmware-misc linux-firmware-intel-wireless linux-firmware-intel-misc
```

> [!warning] Важно про `systemd-networkd`
> Данного пакета тут нету, потому что это **не отдельный пакет** в Ubuntu 26.04. Бинарник и юниты поставляются в составе пакета `systemd`, который уже установлен.
> Попытка `apt install systemd-networkd` завершится ошибкой `Unable to locate package`. Активация выполняется только через `systemctl enable --now systemd-networkd`.

> [!warning] Важно про `linux-firmware`
> Метапакет `linux-firmware` и все его подпакеты приедут **транзитивно** через `linux-generic-hwe-26.04`, потому что в 26.04 firmware объявлен жёстким `Depends`.
> Явное перечисление `linux-firmware-misc`, `-intel-wireless`, `-intel-misc` в команде — это документирование намерения, а не экономия места.
> Бороться с метапакетом через pinning рискованно: сломает автообновления ядра, — и не рекомендуется.

> [!warning] Примечание про `linux-tools-7.0.0-42-generic`
> Пакет жёстко привязан к версии ядра. После обновления HWE-ядра он перестанет соответствовать новому `uname -r`, и `perf`/`bpftool` для нового ядра придётся доставлять отдельно.
> Более устойчивый вариант — метапакет `linux-tools-generic-hwe-26.04`, который тянет tools под актуальное ядро.
> Здесь оставлено явное указание версии, потому что оно соответствует текущему сборочному окружению; при долгой эксплуатации замените на метапакет.

> [!info] Полная таблица пакетов
> См. отдельный файл `packages.md`. В основном документе — только команда и принципы отбора, чтобы не раздуваться до размеров справочника.

---

## ⚙️ Этап 4: Базовая конфигурация системы

Этот этап отвечает на вопрос «как система идентифицирует себя»: какое окружение получает пользователь, как поднимается сеть, какие файловые системы монтируются при загрузке, как система управляет памятью и сетевым стеком. Подразделы независимы друг от друга: можно менять сетевой стек, не трогая fstab, и наоборот.

### 🌍 Этап 4.1: Переменные окружения

Переменные окружения задают базовый контракт между системой и графическими приложениями: в какой сессии они работают (Wayland), какой у них backend для Qt и Electron, как они должны вести себя с HiDPI, и какой `PATH` им доступен. Всё это фиксируется в одном системном файле, который PAM читает при логине.

`debootstrap` уже создал `/etc/environment` с дефолтным PATH из Ubuntu-образа, который включает `/snap/bin` и `/usr/games`. Для этой станции оба пути нежелательны: Snap заблокирован через pinning, а игр на DevOps/NetOps-станции нет. Поэтому файл перезаписывается целиком — с чистым PATH и Wayland-переменными:

```bash
cat > /etc/environment <<'EOF'
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
XDG_SESSION_TYPE=wayland
XDG_CURRENT_DESKTOP=qtile
QT_QPA_PLATFORM="wayland;xcb"
MOZ_ENABLE_WAYLAND=1
ELECTRON_OZONE_PLATFORM_HINT=auto
EOF
```

`/etc/environment` — **системный** файл переменных окружения, который PAM читает при логине (`pam_env.so`). Он применяется **до** запуска пользовательской сессии, поэтому переменные видны композитору, Qtile и всем приложениям сессии.

| Переменная | Значение | Почему именно так |
| :--- | :--- | :--- |
| `PATH` | `/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin` | `local/*` раньше системных — стандартный FHS-принцип: локально установленные бинарники переопределяют системные. `/sbin` после `bin` — стандарт Debian: обычные утилиты раньше root-инструментов. `/snap/bin` убран (Snap заблокирован), `/usr/games` убран (нет игр). |
| `XDG_SESSION_TYPE=wayland` | Тип сессии — Wayland | Сообщает приложениям (Firefox, Chromium, Qt, GTK), что они в Wayland-сессии. Без неё некоторые попытаются подключиться к `$DISPLAY` (X11), которого может не быть. |
| `XDG_CURRENT_DESKTOP=qtile` | Имя текущего DE/WM | `xdg-desktop-portal` выбирает backend по этой переменной; GTK-приложения меняют тему; Electron выбирает, какие диалоги показывать. |
| `QT_QPA_PLATFORM="wayland;xcb"` | Backend для Qt | `wayland` — нативный Wayland (правильные HiDPI, фракционный скейлинг). `xcb` — fallback на X11 через XWayland. Точка с запятой задаёт приоритет. |
| `MOZ_ENABLE_WAYLAND=1` | Firefox → Wayland | Firefox до сих пор может выбирать X11-backend. Переменная явно говорит «используй Wayland». |
| `ELECTRON_OZONE_PLATFORM_HINT=auto` | Electron → Wayland (авто) | Electron-приложения (VS Code, Slack, Discord) исторически шли через XWayland. `auto` означает «попробуй Wayland, если не получилось — X11». |

> [!question] Почему в `/etc/environment`, а не в `~/.profile` или конфиге Qtile?
> Во-первых, порядок применения: файл читается PAM **до** запуска сессии, поэтому переменные видны уже композитору и его потомкам. `~/.profile` появится только в шелле после логина.
> Во-вторых, системный уровень: для воспроизводимости это правильно — все пользователи получают одинаково настроенную Wayland-сессию.
> В-третьих, независимость от shell: `~/.profile` читается только login-shell'ом. Если Qtile запускается через `exec` из display manager'а без login-shell, `~/.profile` может вообще не прочитаться.
> `/etc/environment` работает гарантированно.
> В-четвёртых, `pam_env` выставляет `PATH` для приложений, запущенных не через shell (`.desktop`-файлы, systemd user units). Без явного `PATH` в `/etc/environment` они могут получить неполный PATH.

> [!danger] `>` для первого блока, `>>` для последующих
> Первый `cat >` создаёт файл с нуля (PATH + Wayland-переменные). Все последующие — `cat >>`, чтобы дописывать тему, раскладку и т.д.
> Если случайно использовать `>` во втором блоке — затрёте PATH и Wayland-переменные.

### ⌨️ Этап 4.2: Раскладка клавиатуры

Раскладка клавиатуры — это не уровень Qtile и не уровень X11. В Wayland-сессии композитор (Qtile) отвечает за **низкоуровневую** передачу скан-кодов, но **карта раскладки** (keymap) настраивается отдельно — через переменные окружения XKB, которые читают все приложения, поддерживающие этот протокол.

Для станции с двумя раскладками (us/ru) достаточно уровня XKB — сложные методы ввода (IME) не требуются. Добавляем в `/etc/environment`:

```bash
cat >> /etc/environment <<'EOF'
XKB_DEFAULT_LAYOUT=us,ru
XKB_DEFAULT_OPTIONS=grp:caps_toggle,caps:none
EOF
```

`XKB_DEFAULT_LAYOUT` — список раскладок через запятую (порядок важен: `us` первая, `ru` вторая). `XKB_DEFAULT_OPTIONS` — список опций через запятую. Здесь используются две:

- **`grp:caps_toggle`** — CapsLock переключает группу раскладок (us ↔ ru). Нажатие CapsLock меняет язык ввода.
- **`caps:none`** — отключает штатное поведение CapsLock (фиксацию верхнего регистра). Это нужно, потому что `grp:caps_toggle` уже «забирает» CapsLock себе — без `caps:none` CapsLock одновременно переключал бы раскладку **и** включал Caps Lock, что приводило бы к неожиданному поведению (например, при переключении на русский вы бы случайно получили «ЗАГЛАВНЫЕ» буквы).

Другие популярные варианты для `grp:` — `grp:alt_shift_toggle` (Alt+Shift), `grp:win_space_toggle` (Win+Space). Если вам привычнее Alt+Shift, замените `grp:caps_toggle` на `grp:alt_shift_toggle` и уберите `caps:none` (CapsLock вернётся к штатному поведению).

> [!info] Если Qtile игнорирует `XKB_DEFAULT_LAYOUT`
> Некоторые версии Qtile на Wayland считывают layout из своего конфига, а не из переменных окружения. В этом случае добавьте в `~/.config/qtile/config.py`:
> ```python
> from libqtile.backend.wayland import InputConfig
> wl_input_rules = {
>     "type:keyboard": InputConfig(
>         kb_layout="us,ru",
>         kb_options="grp:caps_toggle,caps:none",
>     ),
> }
> ```
> Переменные из `/etc/environment` при этом остаются как fallback для приложений, которые не читают конфиг Qtile.
>
> **Обратите внимание:** список опций должен совпадать в обоих местах — иначе поведение будет разным для Qtile-нативных окон и для XWayland-приложений.

### 🌐 Этап 4.3: Сетевой стек без NetworkManager

Вместо NetworkManager я предпочитаю пользоваться детерминированной связкой `systemd-networkd` + `wpa_supplicant` + `systemd-resolved`. Конфигурация — это файлы в `/etc/systemd/network/` и `/etc/wpa_supplicant/`, которые можно версионировать и которые не меняются «самим собой».

Ключевой момент, который нужно усвоить сразу: **`networkctl` не умеет подключаться к Wi-Fi**. Это инструмент мониторинга и управления линками для `systemd-networkd`. Сам Wi-Fi-аутентификацией занимается `wpa_supplicant`, а интерактивным клиентом к нему служит `wpa_cli`.

| Слой | Компонент | Что делает |
| :--- | :--- | :--- |
| Аутентификация Wi-Fi | `wpa_supplicant` | Ассоциация с точкой доступа, WPA2/WPA3, roaming. |
| IP-адресация | `systemd-networkd` | DHCP-клиент, маршруты, линки. |
| DNS | `systemd-resolved` | Stub-resolver на `127.0.0.53`, кэш, DNSSEC. |
| Мониторинг | `networkctl` | `networkctl status`, `networkctl list`. |
| Интерактивное подключение | `wpa_cli` | `scan`, `add_network`, `set_network`, `enable_network`. |

Сначала включаем службы и настраиваем симлинк для резолвера:

```bash
systemctl enable systemd-networkd
systemctl enable systemd-resolved
systemctl enable wpa_supplicant@wlp3s0.service
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

Имя интерфейса `wlp3s0` взято из инвентаризации Этапа 0. Если у вас другой Wi-Fi адаптер, имя будет отличаться — проверьте через `networkctl list` или `ip -br link`.

Конфигурация `systemd-networkd` — два файла: Wi-Fi и Ethernet. Ethernet-файл нужен на случай подключения кабеля.

```bash
# Wi-Fi (wlp3s0)
cat > /etc/systemd/network/25-wireless.network <<'EOF'
[Match]
Name=wlp3s0

[Network]
DHCP=yes

[DHCPv4]
RouteMetric=600
EOF

# Ethernet (eno1) — с более высоким приоритетом
cat > /etc/systemd/network/20-wired.network <<'EOF'
[Match]
Name=eno1

[Network]
DHCP=yes

[DHCPv4]
RouteMetric=100
EOF
```

`RouteMetric=100` для Ethernet и `600` для Wi-Fi — при одновременном подключении трафик пойдёт через кабель.

Конфигурация `wpa_supplicant` лежит в файле, имя которого соответствует имени сервиса: `wpa_supplicant@wlp3s0.service` читает `/etc/wpa_supplicant/wpa_supplicant-wlp3s0.conf`:

```bash
cat > /etc/wpa_supplicant/wpa_supplicant-wlp3s0.conf <<'EOF'
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=RU

network={
    ssid="Ваш_SSID"
    psk="Ваш_пароль"
    key_mgmt=WPA-PSK
}
EOF
chmod 600 /etc/wpa_supplicant/wpa_supplicant-wlp3s0.conf
```

> [!tip] Альтернатива — не хранить пароль в открытом виде
> ```bash
> wpa_passphrase "Ваш_SSID" "Ваш_пароль" >> /etc/wpa_supplicant/wpa_supplicant-wlp3s0.conf
> ```
> Утилита создаст блок `network={ ssid=... #psk="..." psk=<hex> }`, где пароль в открытом виде закомментирован, а рабочий `psk` хранится в hex.

Проверка:

```bash
systemctl restart systemd-networkd wpa_supplicant@wlp3s0
networkctl status wlp3s0
# Ожидаем: State=routable, Address=<IPv4>
ip -4 a show wlp3s0
resolvectl status
```

> [!warning] Если что-то не так
> `journalctl -u systemd-networkd -u wpa_supplicant@wlp3s0 -b` — покажет, на каком этапе отвалилось.
> `wpa_cli -i wlp3s0 status` — состояние ассоциации (`wpa_state=COMPLETED` — успех).
> `networkctl status` — что видит `systemd-networkd` со своей стороны.

### 🚀 Этап 4.4: Сетевой sysctl-тюнинг (BBR, fq, low-latency)

Для стационарного ПК, особенно при игре и работе с сетевыми сервисами, стандартный CUBIC + `fq_codel` — не оптимальный выбор. TCP BBR даёт заметно меньшую задержку и лучшую пропускную способность на потерях, а `fq` (fair queue) — правильный qdisc для BBR. Все параметры фиксируются в drop-in файле, который читается при загрузке.

Создаём drop-in в `/etc/sysctl.d/`:

```bash
cat > /etc/sysctl.d/99-network-performance.conf <<'EOF'
# BBR congestion control + fair queue
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr

# Low-latency: не буферизовать «лишние» байты в сокете
net.ipv4.tcp_notsent_lowat = 16384

# Буферы сокетов (для стационарного ПК с 32+ ГБ RAM — разумный компромисс)
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# Быстрое переиспользование TIME_WAIT (для активного серфинга/игр)
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1

# Очередь на входящие соединения
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 8192
EOF
```

> [!info] Почему именно эти параметры
> `net.core.default_qdisc = fq` и `net.ipv4.tcp_congestion_control = bbr` — базовая связка для BBR. `fq` даёт fair queueing, который BBR использует для оценки пропускной способности. Без `fq` BBR работает, но менее эффективно.
> `tcp_notsent_lowat = 16384` — снижает задержку, не позволяя TCP-стеку держать в сокете больше 16 КБ непереданных данных. Полезно для интерактивных приложений (SSH, игры, HTTP).
> `tcp_tw_reuse = 1` — позволяет переиспользовать сокеты в состоянии TIME_WAIT для новых исходящих соединений. Безопасно на клиентской машине, заметно снижает задержку при активном серфинге.
> Буферы `rmem_max`/`wmem_max` на 16 МБ — разумный потолок для домашней сети; больше — только для 10G+ или BDP-тяжёлых каналов.

Применяем и проверяем:

```bash
sysctl --system
sysctl net.ipv4.tcp_congestion_control   # ожидаем bbr
sysctl net.core.default_qdisc            # ожидаем fq
```

> [!warning] Если `bbr` недоступен
> Проверьте: `sysctl net.ipv4.tcp_available_congestion_control`. Если `bbr` в списке нет — ядро собрано без него (маловероятно для Ubuntu 26.04, но возможно). В этом случае оставьте `fq_codel` как fallback: `net.core.default_qdisc = fq_codel`, `net.ipv4.tcp_congestion_control = cubic`.

> [!tip] Модуль `tcp_bbr`
> В некоторых ядрах `tcp_bbr` собран как модуль. Если `bbr` не появляется, добавьте `tcp_bbr` в `/etc/modules-load.d/bbr.conf`:
> ```bash
> echo tcp_bbr > /etc/modules-load.d/bbr.conf
> ```
> И перезагрузитесь.

### 🗂️ Этап 4.5: fstab и crypttab — эталонные опции монтирования

Два файла задают идентичность дисковой подсистемы: `/etc/crypttab` описывает LUKS-контейнеры, которые нужно разблокировать при загрузке, `/etc/fstab` — что и куда монтировать и с какими опциями. Оба файла напрямую влияют на производительность и безопасность, поэтому опции подобраны осознанно, а не скопированы из `defaults`.

#### crypttab

`crypttab` генерируется автоматически из UUID раздела. Основное решение здесь — опция `discard`, которая разрешает проброс TRIM-команд к SSD. Это нужно для сохранения производительности NVMe со временем: без TRIM контроллер SSD не знает, какие блоки свободны, и постепенно деградирует. Минус — небольшая утечка информации: по паттерну TRIM-запросов теоретически можно судить о том, какие блоки освободились. Для домашней станции это приемлемый компромисс.

```bash
LUKS_UUID=$(cryptsetup luksUUID /dev/nvme0n1p3)
cat > /etc/crypttab <<EOF
cryptroot UUID=${LUKS_UUID} none luks,discard
EOF
```

> [!info] Если позже добавите TPM2 как второй ключ
> `crypttab` можно расширить опцией `tpm2-device=auto` (см. Этап 8.3). Тогда `systemd-cryptsetup` попробует сначала TPM, а при неудаче — пароль. Строка будет выглядеть так:
> ```
> cryptroot UUID=... none luks,discard,tpm2-device=auto
> ```
> Но это делается **после** `systemd-cryptenroll --tpm2-device=auto ...`, иначе загрузка зависнет в ожидании несуществующего токена.

#### fstab с эталонными опциями

`fstab` собирается вручную из UUID, которые выдаёт `blkid`. Помимо самих точек монтирования, важны опции: `noatime` убирает лишние записи метаданных при чтении, `mode=1777` и `nosuid,nodev` на `/tmp` — базовая мера для hardened-системы. Опция `async` на XFS сознательно **не** добавляется: она даёт прирост скорости ценой потери данных при отключении питания.

```bash
cat > /etc/fstab <<'EOF'
# <file system>                           <mount point>            <type>  <options>                                         <dump> <pass>
UUID=<UUID-root>                          /                        xfs     defaults,noatime                                  0      1
UUID=<UUID-boot>                          /boot                    ext4    defaults,noatime                                  0      2
UUID=<UUID-efi>                           /boot/efi                vfat    umask=0077,noatime                                0      1
UUID=<UUID-home>                          /home                    xfs     defaults,noatime                                  0      2
UUID=<UUID-var_log>                       /var/log                 xfs     defaults,noatime                                  0      2
UUID=<UUID-libvirt>                       /var/lib/libvirt         xfs     defaults,noatime                                  0      2
UUID=<UUID-containers>                    /var/lib/containers      xfs     defaults,noatime                                  0      2
tmpfs                                     /tmp                     tmpfs   defaults,nosuid,nodev,noatime,mode=1777           0      0
EOF
```

| Опция | Где применяется | Почему так |
| :--- | :--- | :--- |
| `noatime` | XFS, ext4 | Отключает обновление времени доступа при чтении файла. Для XFS по умолчанию используется `relatime`, который почти так же эффективен, но `noatime` полностью убирает лишние записи метаданных при чтении. Для десктопа это безопасно: программы, которым действительно нужен `atime` (например, `mutt`), используют `O_NOATIME` или явно включают его. |
| `noatime` на `/boot/efi` | vfat | ESP почти не читается после загрузки, но `noatime` убирает единственную причину для записи при чтении. |
| `mode=1777` на `/tmp` | tmpfs | Стандартные права для `/tmp`: все могут писать, но удалять — только владелец (sticky bit). `tmpfs` не сохраняет права при пересоздании, поэтому `mode=1777` нужно указать явно. |
| `nosuid,nodev` на `/tmp` | tmpfs | Запрещает SUID-бинарники и device-файлы в `/tmp`. Базовая мера для hardened-системы. |
| `async` на XFS | — | **Не добавляйте.** `async` означает «игнорировать fsync», что даёт прирост скорости ценой потери данных при отключении питания. Для станции с LUKS и бэкапами это неоправданный риск. XFS и так агрессивно кэширует записи. |

> [!tip] Про `logbufs` и `logbsize`
> На XFS можно увеличить `logbufs=8` и `logbsize=256k` для лучшей производительности журнала при интенсивной записи (ВМ, контейнеры, сборки). Но это имеет смысл только при подтверждённой нагрузке. Начните с `defaults,noatime`; если увидите узкое место в журнале XFS (`xfs_info` и `iostat -x` покажут), добавите:
> ```
> defaults,noatime,logbufs=8,logbsize=256k
> ```

### 💾 Этап 4.6: ZRAM (сжатый swap в RAM)

ZRAM создаёт блочное устройство в оперативной памяти, которое сжимает страницы. Это даёт «swap», который в разы быстрее диска и не изнашивает SSD. Для станции с 32+ ГБ RAM это скорее страховка от редких пиков, чем необходимость, но настроить стоит — цена нулевая, а выигрыш при неожиданном пике потребления памяти заметный.

```bash
cat > /etc/systemd/zram-generator.conf <<'EOF'
[zram0]
zram-size = min(ram / 2, 16384)
compression-algorithm = zstd
swap-priority = 100
EOF
```

| Параметр | Значение | Почему так |
| :--- | :--- | :--- |
| `zram-size = min(ram / 2, 16384)` | Половина RAM, но не больше 16 ГБ | Для 32 ГБ RAM это даст 16 ГБ ZRAM — с запасом. `min()` защищает от абсурдных значений на машинах с 128+ ГБ RAM: ZRAM там не нужен в размере 64 ГБ. |
| `compression-algorithm = zstd` | zstd | Лучший баланс степень сжатия / скорость. `lzo-rle` быстрее, но сжимает хуже; `zstd` на современных CPU (у вас i5-11400 с AVX-512) даёт хороший результат при минимальной нагрузке. |
| `swap-priority = 100` | Высокий приоритет | Ядро будет предпочитать ZRAM дисковому swap (если он вообще появится). Это правильно: ZRAM быстрее. |

> [!info] Нужен ли дисковый swap
> С ZRAM на 16 ГБ и 32 ГБ RAM дисковый swap, скорее всего, не понадобится. Но если вы хотите подстраховаться (например, для гибернации), можно добавить swap-файл на LUKS-разделе. Это делается отдельно и не входит в базовую поставку.

Применение:

```bash
systemctl daemon-reload
systemctl start systemd-zram-setup@zram0.service
zramctl
# Ожидаем: /dev/zram0 с размером ~16G и алгоритмом zstd
```

### 🔑 Этап 4.7: Настройка opendoas

`opendoas` — минималистичная замена `sudo` (около 1000 строк кода против десятков тысяч у sudo). В Ubuntu 26.04 пакет `opendoas` не создаёт конфигурационный файл автоматически — без `/etc/doas.conf` команда `doas` отказывает всем без объяснений. Поэтому конфиг создаётся вручную, сразу после установки пакетов.

Базовый конфиг выглядит так:

```bash
cat > /etc/doas.conf <<'EOF'
permit persist keepenv :wheel
EOF
chmod 0400 /etc/doas.conf
```

| Директива | Значение | Почему так |
| :--- | :--- | :--- |
| `permit` | Разрешение | Основная директива: «разрешить выполнение». |
| `persist` | Не спрашивать пароль повторно | Аналог sudo `timestamp_timeout` — в течение нескольких минут повторные `doas` не требуют пароля. Удобно при серии команд. |
| `keepenv` | Сохранять окружение | По умолчанию `doas` очищает env, оставляя минимум (`PATH`, `HOME`, `USER`, `SHELL`). Для `apt install` с `DEBIAN_FRONTEND=noninteractive` или для `doas` в скриптах, где важны переменные, `keepenv` нужен. |
| `:wheel` | Группа wheel | Все члены группы `wheel` получают право на `doas`. Группа `wheel` — исторический стандарт в BSD; в Debian/Ubuntu её нет по умолчанию, но её легко создать. |

Права `0400` на `/etc/doas.conf` обязательны: `doas` откажется работать с более широкими правами, потому что конфиг содержит правила безопасности.

Группа `wheel` создаётся так:

```bash
groupadd -f wheel
```

После создания пользователя (этап создания пользователей в документе пока не описан, но он будет) его нужно добавить в группу:

```bash
usermod -aG wheel <username>
```

Проверка, что `doas` работает:

```bash
doas -C /etc/doas.conf        # проверка синтаксиса
doas id                        # должно вернуть uid=0(root) или ошибку, если пользователь не в wheel
```

> [!warning] `doas` — не `sudo`
> Синтаксис почти совпадает, но есть отличия: нет `doas -i` (аналог `sudo -i`), нет `doas -s` (шелл), нет `doas -E` (сохранить env — для этого `keepenv` в конфиге), нет `doasedit` (аналог `sudoedit`). Если в скриптах используется `sudo -i` или `sudoedit`, `doas` их не заменит — придётся либо адаптировать скрипты, либо оставить `sudo` дополнительно.

> [!tip] Если нужна прозрачная замена `sudo`
> Можно создать симлинк:
> ```bash
> ln -s "$(command -v doas)" /usr/local/bin/sudo
> ```
> Но это работает только для базового `sudo <command>`. Все флаги (`-i`, `-s`, `-E`, `-u`, `-g`, `-H`, `-k`, `-K`) либо отсутствуют, либо ведут себя иначе. Симлинк оправдан только в простых окружениях; для станции с разнообразными скриптами лучше явно использовать `doas` и не маскировать его под `sudo`.

---

## 🌐 Этап 3.2: Сетевой стек без NetworkManager

### Что вы получаете и чего не получаете

Ключевой момент: **`networkctl` не умеет подключаться к Wi-Fi**. Это инструмент мониторинга и управления линками для `systemd-networkd`. Сам Wi-Fi-аутентификацией занимается **`wpa_supplicant`**, а интерактивным клиентом к нему служит **`wpa_cli`**.

| Слой | Компонент | Что делает |
| :--- | :--- | :--- |
| Аутентификация Wi-Fi | `wpa_supplicant` | Ассоциация с AP, WPA2/WPA3, roaming. |
| IP-адресация | `systemd-networkd` | DHCP-клиент, маршруты, линки. |
| DNS | `systemd-resolved` | Stub-resolver на `127.0.0.53`, кэш, DNSSEC. |
| Мониторинг | `networkctl` | `networkctl status`, `networkctl list`. |
| Интерактивное подключение | `wpa_cli` | `scan`, `add_network`, `set_network`, `enable_network`. |

> **Аналог `nmcli`.** Прямого аналога «одной командой подключиться к Wi-Fi» в связке `systemd-networkd` + `wpa_supplicant` нет. Ближайший эквивалент — `wpa_cli` (интерактивно) или `wpa_passphrase` + правка `wpa_supplicant.conf` (декларативно). Для стационарного ПК это даже плюс: конфигурация версионируется и не меняется «самим собой».

### Включение служб

```bash
systemctl enable systemd-networkd
systemctl enable systemd-resolved
systemctl enable wpa_supplicant@wlp3s0.service    # интерфейс из Этапа 0
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

### Конфигурация `systemd-networkd`

```bash
# Wi-Fi (wlp3s0)
cat > /etc/systemd/network/25-wireless.network <<'EOF'
[Match]
Name=wlp3s0

[Network]
DHCP=yes

[DHCPv4]
RouteMetric=600
EOF

# Ethernet (eno1) — с более высоким приоритетом
cat > /etc/systemd/network/20-wired.network <<'EOF'
[Match]
Name=eno1

[Network]
DHCP=yes

[DHCPv4]
RouteMetric=100
EOF
```

> `RouteMetric=100` для Ethernet и `600` для Wi-Fi — при одновременном подключении трафик пойдёт через кабель. Классическая схема для стационарного ПК.

### Конфигурация `wpa_supplicant`

```bash
cat > /etc/wpa_supplicant/wpa_supplicant-wlp3s0.conf <<'EOF'
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=RU

network={
    ssid="Ваш_SSID"
    psk="Ваш_пароль"
    key_mgmt=WPA-PSK
}
EOF
chmod 600 /etc/wpa_supplicant/wpa_supplicant-wlp3s0.conf
```

> **Альтернатива — интерактивно.** Если не хотите хранить пароль в открытом виде:
>
> ```bash
> wpa_passphrase "Ваш_SSID" "Ваш_пароль" >> /etc/wpa_supplicant/wpa_supplicant-wlp3s0.conf
> ```
>
> Это создаст `network={ ssid=... #psk="..." psk=<hex> }` — пароль в открытом виде закомментирован. Или полностью интерактивно через `wpa_cli` (см. ниже).

### Интерактивное подключение через `wpa_cli`

```bash
wpa_cli -i wlp3s0
# внутри:
#   scan
#   scan_results
#   add_network                     → вернёт ID, например 0
#   set_network 0 ssid "Ваш_SSID"
#   set_network 0 psk "Ваш_пароль"
#   enable_network 0
#   save_config
#   quit
```

### Проверка

```bash
systemctl restart systemd-networkd wpa_supplicant@wlp3s0
networkctl status wlp3s0
# Ожидаем: State=routable, Address=<IPv4>
ip -4 a show wlp3s0
resolvectl status
```

> **Если что-то не так.** `journalctl -u systemd-networkd -u wpa_supplicant@wlp3s0 -b` — первым делом. `wpa_cli -i wlp3s0 status` — состояние ассоциации. `networkctl status` — что видит `systemd-networkd`.

---

## 🗂️ Этап 3.3: fstab и crypttab

```bash
LUKS_UUID=$(cryptsetup luksUUID /dev/nvme0n1p3)
echo "cryptroot UUID=${LUKS_UUID} none luks,discard" > /etc/crypttab

# fstab — подставьте UUID из `blkid`
cat > /etc/fstab <<'EOF'
UUID=<UUID-root>       /                       xfs     defaults                0 1
UUID=<UUID-boot>       /boot                   ext4    defaults                0 2
UUID=<UUID-efi>        /boot/efi               vfat    umask=0077              0 1
UUID=<UUID-home>       /home                   xfs     defaults                0 2
UUID=<UUID-var_log>    /var/log                xfs     defaults                0 2
UUID=<UUID-libvirt>    /var/lib/libvirt        xfs     defaults                0 2
UUID=<UUID-containers> /var/lib/containers     xfs     defaults                0 2
tmpfs                  /tmp                    tmpfs   defaults,nosuid,nodev   0 0
EOF
```

> **`tmpfs` на `/tmp`** — небольшая, но полезная деталь для hardened-системы: временные файлы не переживают перезагрузку. Учтите, что некоторым сборкам (например, `go build`) может не хватить RAM — при необходимости поднимите `size=` или откажитесь от этой строки.---

## 🎨 Этап 4: GRUB, Plymouth, Secure Boot MOK и резервное копирование

### 🖼️ 4.1. Красивый и информативный GRUB

```bash
# Установка темы
apt install -y grub-theme-breeze

cat > /etc/default/grub <<'EOF'
GRUB_DEFAULT=saved
GRUB_SAVEDEFAULT=true
GRUB_TIMEOUT=10
GRUB_TIMEOUT_STYLE=menu
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
GRUB_CMDLINE_LINUX=""
GRUB_DISABLE_RECOVERY=true
GRUB_THEME="/boot/grub/themes/breeze/theme.txt"
EOF
```

**Пояснение ключевых параметров:**

* `GRUB_DEFAULT=saved` + `GRUB_SAVEDEFAULT=true`: GRUB запоминает последнее успешно загруженное ядро. Если новое окажется проблемным — вы всегда сможете выбрать старое один раз, и оно станет дефолтным.
* `GRUB_TIMEOUT=10`: 10 секунд на выбор — разумный запас для админской станции.
* `GRUB_DISABLE_RECOVERY=true`: убирает лишние пункты «recovery mode» (у нас есть полноценный live-USB, recovery-режим Ubuntu избыточен).

```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu --uefi-secure-boot
update-grub
update-initramfs -u -k all
```

### 🔐 4.2. Secure Boot и MOK (критично для Nvidia/DKMS)

Secure Boot включён, но сторонние модули (в первую очередь Nvidia и VirtualBox) не подписаны ключами Microsoft. Их нужно подписать собственным MOK-ключом и зарегистрировать его в прошивке.

```bash
# Генерация MOK-ключа (делается один раз)
mokutil --generate-key
# Создаст /var/lib/shim-signed/mok/MOK.priv и MOK.der

# Регистрация ключа в прошивке (потребует пароль и перезагрузку)
mokutil --import /var/lib/shim-signed/mok/MOK.der
```

При следующей загрузке появится **MOK Manager** (синий экран). Нужно: `Enroll MOK` → `Continue` → `Yes` → ввести пароль, заданный при `--import` → `Reboot`.

> **Зачем это в базовой сборке.** Даже если Nvidia вы поставите позже, ключ MOK должен быть зарегистрирован до первой загрузки с DKMS-модулем. Иначе `nvidia.ko` не загрузится, и вы получите llvmpipe/чёрный экран вместо драйвера. Регистрация MOK на этапе сборки — самый безболезненный момент.

### 🔄 4.3. Сохранение двух ядер

Ubuntu по умолчанию хранит несколько последних ядер. `GRUB_DEFAULT=saved` + `GRUB_SAVEDEFAULT=true` уже дают вам возможность откатиться. Для явного управления:

```bash
# Посмотреть список доступных ядер
grep -E "menuentry 'Ubuntu" /boot/grub/grub.cfg

# Установить конкретное ядро по умолчанию
# sudo grub-set-default "1>2"
```

Файл `/etc/apt/apt.conf.d/01autoremove-kernels` в Ubuntu по умолчанию настроен так, чтобы не удалять ядро, с которого currently running. Дополнительная настройка не требуется.

### 📸 4.4. Информативный экран загрузки (Plymouth)

```bash
update-alternatives --config default.plymouth   # выбрать spinner или glow
update-initramfs -u
```

> **Про `splash` и LUKS.** Plymouth не скрывает запрос пароля LUKS — он появляется поверх темы. Если хочется видеть больше диагностики при загрузке (например, при отладке), на время уберите `splash` из `GRUB_CMDLINE_LINUX_DEFAULT`.

### 💾 4.5. Автоматическое резервное копирование

```bash
cat > /usr/local/bin/backup.sh <<'EOF'
#!/bin/bash
set -euo pipefail
export RESTIC_REPOSITORY="/mnt/backup/restic"
export RESTIC_PASSWORD_FILE="/etc/restic/password"
restic backup /home /etc /var/lib/libvirt /var/lib/containers --exclude-caches
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
EOF
chmod +x /usr/local/bin/backup.sh

mkdir -p /etc/restic
echo "your-strong-password" > /etc/restic/password
chmod 600 /etc/restic/password

# Инициализация репозитория (один раз, при смонтированном /mnt/backup)
restic init

# Cron: ежедневно в 3:00
(crontab -l 2>/dev/null; echo "0 3 * * * /usr/local/bin/backup.sh") | crontab -
```

> **Проверка целостности.** Раз в месяц (или после каждого крупного обновления) — `restic check`. Для критичных данных — `restic check --read-data` (читает все данные, медленно, но надёжно).

---

## 🧩 Этап 5: Архитектура пользовательской сессии (Qtile + s6)

Разделение ролей продолжается на уровне пользователя:

1. **Qtile** отвечает только за тайлинг окон и панели (включая кастомный дашборд CVE).
2. **S6** (устанавливается через `pipx` или из исходников) запускается как дочерний процесс Qtile и руководит фоновыми службами.

### 🗂️ Карта пользовательских сервисов (под управлением S6)

| Компонент | Роль | Ответственность |
| :--- | :--- | :--- |
| **dunst** | Демон уведомлений | Всплывающие уведомления, уровни срочности |
| **pipewire** | Аудиосервер | Маршрутизация звука, устройства (замена PulseAudio) |
| **wireplumber** | Session manager | Политики PipeWire |
| **polkit agent** | Агент аутентификации | Запросы прав для привилегированных операций |
| **gnome-keyring-daemon** | Хранилище секретов | Пароли приложений, `libsecret`-сервис |
| **thunar --daemon** | Файловый менеджер | Автомонтирование, корзина |
| **fcitx5** | Ввод текста | Раскладки и методы ввода |
| **Qtile widget** | Wi-Fi индикатор | Статус `wlp3s0` через `networkctl`/`wpa_cli` (вместо `nm-applet`) |

> **Что делать с `nm-applet`.** Он удалён вместе с NetworkManager. Если нужен трей-индикатор Wi-Fi, проще всего написать Qtile-виджет, который парсит `networkctl status wlp3s0` и `wpa_cli status`. Альтернативы: `iwgtk` (для `iwd`, не для `wpa_supplicant`), `wpa_gui` (Qt, требует `wpagui`, но в `--no-install-recommends` может потянуть Qt-зависимости).

### 🔐 Сервисы системного уровня (Systemd)

| Сервис | Роль |
| :--- | :--- |
| **systemd-networkd** | Управление сетью |
| **systemd-resolved** | DNS-резолвер |
| **wpa_supplicant@wlp3s0** | Wi-Fi-аутентификация |
| **libvirtd** | Управление KVM/QEMU |
| **systemd-timesyncd** | Синхронизация времени |
| **fstrim.timer** | Периодическая очистка SSD |

---

## 🧰 Этап 6: Курируемый стек приложений (DevOps & DevNet)

| Категория | Инструменты | Назначение |
| :--- | :--- | :--- |
| **Терминал** | **Kitty** | GPU-ускоренный эмулятор с лигатурами. |
| **Оболочка** | **Zsh** | Интерактивная оболочка + Oh My Zsh. |
| **Редактор** | **Neovim**, **Freelens** | Код (Neovim); UI для K8s (Freelens). |
| **VCS** | **Git** | Управление исходным кодом. |
| **Контейнеры** | **nerdctl**, **kubectl**, **helm**, **kubectx** | containerd, K8s, контексты. |
| **IaC** | **Terraform**, **OpenTofu**, **Ansible** | Провижининг и конфигурация. |
| **Сеть / API** | **curl**, **wget** | HTTP-запросы. |
| **Безопасность** | **GPG**, **pass**, **Ente Auth** | Шифрование, менеджер паролей, TOTP. |
| **Виртуализация** | **virt-manager**, **virt-viewer** | KVM/QEMU, SPICE/VNC. |
| **Мониторинг** | **btop**, **fastfetch** | Процессы/ресурсы, инфо о системе. |
| **Файлы** | **Thunar** | Лёгкий графический FM. |
| **Документы** | **Zathura** | PDF/PS с Vim-биндингами. |
| **Изображения** | **feh** | Просмотр и обои. |
| **Медиа** | **mpv** | Минималистичный плеер. |
| **Браузер** | **Firefox** | Из Mozilla APT или Flatpak, **не Snap**. |

---

## 🧹 Этап 7: Финализация

```bash
# Ограничение журналов
cat > /etc/systemd/journald.conf.d/99-storage-limit.conf <<'EOF'
[Journal]
SystemMaxUse=2G
SystemMaxFileSize=100M
EOF

# ZRAM (сжатый swap в памяти)
cat > /etc/systemd/zram-generator.conf <<'EOF'
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
EOF

systemctl enable systemd-networkd
systemctl enable systemd-resolved
systemctl enable wpa_supplicant@wlp3s0.service
systemctl enable fstrim.timer
systemctl enable systemd-timesyncd

apt clean
apt autoremove --purge -y
exit
sudo umount -R /mnt
sudo cryptsetup close cryptroot
sudo reboot
```

---

## 🚀 Этап 8: Пост-инсталляция (после первой загрузки)

### 8.1. Проверка, что нежелательное не прилипло

```bash
for p in network-manager libmm-glib0 upower modemmanager imagemagick-7-common zbar-tools \
         libimobiledevice-1.0-6 libcamera-ipa pipewire-libcamera systemd-resolved; do
  if [ "$p" = "systemd-resolved" ]; then
    dpkg -s "$p" 2>/dev/null | grep -q '^Status: install ok installed' && echo "✅ $p (должен быть)" || echo "❌ $p ОТСУТСТВУЕТ"
  else
    dpkg -s "$p" 2>/dev/null | grep -q '^Status: install ok installed' && echo "❌ $p УСТАНОВЛЕН" || echo "✅ $p отсутствует"
  fi
done
```

### 8.2. Установка Nvidia

```bash
# Определить рекомендуемый драйвер
ubuntu-drivers devices

# Установить (пример для 550-й ветки, актуальную смотрите в выводе выше)
apt install nvidia-driver-550

# VA-API для аппаратного декодирования видео (Firefox, mpv)
apt install nvidia-vaapi-driver
```

> **DKMS + Secure Boot.** Nvidia-драйвер собирается через DKMS. Модуль будет автоматически подписан вашим MOK-ключом (Этап 4.2). После установки — перезагрузка. Проверка: `nvidia-smi`, `lsmod | grep nvidia`. Если модуль не загрузился — `dmesg | grep -i 'module verification'` покажет, что MOK не зарегистрирован.

### 8.3. TPM (опционально)

```bash
apt install tpm2-tools

# Посмотреть PCR и TPM-информацию
tpm2_getcap properties-fixed

# Опционально: добавить TPM как второй ключ к LUKS
# systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7 /dev/nvme0n1p3
```

> **Осторожно с `--tpm2-pcrs`.** PCR 7 привязан к Secure Boot state, PCR 0 — к прошивке. Неудачная комбинация может заблокировать автоматическую разблокировку после обновления BIOS. Всегда оставляйте пароль LUKS как fallback.

### 8.4. BRIO: проверка

```bash
v4l2-ctl --list-devices
# Ожидаем: Logitech BRIO (usb-...): /dev/video0, /dev/video1 (второй — метаданные)

v4l2-ctl --device=/dev/video0 --list-formats-ext | head -40
wpctl status | grep -i brio
```

### 8.5. Wi-Fi: проверка через `networkctl`

```bash
networkctl status wlp3s0       # State: routable, Address: 192.168.x.x
wpa_cli -i wlp3s0 status       # wpa_state=COMPLETED
resolvectl status               # Current DNS Server
ping -c1 1.1.1.1 && ping -c1 ya.ru
```

---

## 💎 Итог

Вы получаете гибридную систему:

* **Фундаментальная стабильность Ubuntu LTS** с жёстким контролем над пакетами (без Snap, без Recommends, только security-обновления).
* **Индустриальное хранилище**: LUKS2 + LVM + XFS с раздельными томами под ВМ и контейнеры.
* **Чистая архитектура сети**: `systemd-networkd` + `wpa_supplicant` + `systemd-resolved` — без NetworkManager, без `libmm-glib0`, без `modemmanager`. Конфигурация — версионируемые файлы.
* **Чистая архитектура пользовательской сессии**: Qtile — только окна, S6 — только фоновые сервисы.
* **Отказоустойчивая конфигурация загрузки**: GRUB с сохранением последнего ядра, Plymouth, Secure Boot + MOK.
* **Автоматизированное резервное копирование** на базе `restic`.
* **Готовность к Nvidia**: MOK зарегистрирован заранее, `mesa-vulkan-drivers` даёт fallback-рендеринг до установки проприетарного драйвера.

Система воспроизводима, аудируема и не содержит «пакетов на всякий случай» — каждый элемент либо продиктован железом (Intel AX210, BRIO, RTX 3070, TPM 2.0), либо входит в заявленный DevOps/DevNet-стек.
