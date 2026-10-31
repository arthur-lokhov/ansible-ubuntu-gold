# 📦 Полная таблица пакетов

Обоснование каждого пакета из базовой установки. Сгруппировано по функциональным блокам. Формат: **пакет** — что это — зачем нужен на этой станции.

Станция собирается под конкретное железо: Intel Core i5-11400 (Rocket Lake), NVIDIA RTX 3070 (без iGPU), Intel AX210 (Wi-Fi 6E + BT 5.4), Intel I219-V Ethernet, Logitech BRIO (UVC), TPM 2.0. Пакеты, не относящиеся к этому железу или к заявленному сценарию DevOps/NetOps, сознательно исключены.

---

## 🚀 Ядро и загрузка

| Пакет | Что это | Зачем нужен |
| :--- | :--- | :--- |
| `linux-generic-hwe-26.04` | Метапакет HWE-ядра | Ядро 7.0 с поддержкой новейшего железа + автообновления в рамках HWE |
| `grub-efi-amd64-signed` | Подписанный GRUB для UEFI | Загрузчик, совместимый с Secure Boot (подпись Microsoft) |
| `shim-signed` | Shim-загрузчик | Первичный бинарник, подписанный ключом Microsoft, доверяемый прошивкой. Через него грузится GRUB |
| `mokutil` | Утилита управления MOK | Регистрация собственного ключа для подписи сторонних модулей (Nvidia/DKMS) |
| `efibootmgr` | Управление EFI-записями NVRAM | Создание/удаление/переупорядочивание записей загрузки |
| `secureboot-db` | База отозванных сертификатов Secure Boot | Обновления dbx (revocation list) для SB |
| `os-prober` | Обнаружение других ОС | Находит Windows/Linux на других дисках для меню GRUB |

## 💽 Шифрование и файловые системы

| Пакет | Что это | Зачем нужен |
| :--- | :--- | :--- |
| `cryptsetup-initramfs` | LUKS в initramfs | Скрипты для разблокировки LUKS на раннем этапе загрузки |
| `lvm2` | LVM2 | Менеджер логических томов (`pvcreate`, `vgcreate`, `lvcreate`) |
| `thin-provisioning-tools` | Утилиты тонкого провижининга | Для thin pools LVM (если будете использовать) |
| `xfsprogs` | Утилиты XFS | `mkfs.xfs`, `xfs_repair`, `xfs_growfs` |
| `xfsdump` | Бэкап XFS | `xfsdump`/`xfsrestore` для образов ФС |
| `dosfstools` | Утилиты FAT | `mkfs.vfat` для EFI-раздела (ESP) |
| `attr` | Утилиты xattr | `getfattr`/`setfattr` для расширенных атрибутов |
| `initramfs-tools` | Генератор initramfs | Собирает initrd-образ при обновлении ядра |

## 🛠️ Базовый инструментарий

| Пакет | Что это | Зачем нужен |
| :--- | :--- | :--- |
| `opendoas` | Замена sudo | Минималистичный, аудируемый менеджер привилегий (меньше кода = проще аудит) |
| `neovim` | Текстовый редактор | Основной редактор для кода и конфигов |
| `wget` | HTTP/FTP-клиент | Скачивание файлов с докачкой |
| `curl` | HTTP-клиент | API-тестирование, REST, DevOps-скрипты |
| `unzip` | Распаковщик ZIP | Для `.zip`-архивов |
| `zip` | Упаковщик ZIP | Создание `.zip`-архивов |
| `zstd` | Zstandard-компрессор | Быстрое сжатие (используется в ядре, zram) |
| `pigz` | Параллельный gzip | Многопоточное сжатие gzip |
| `pbzip2` | Параллельный bzip2 | Многопоточный bzip2 |
| `pixz` | Параллельный xz | Многопоточный xz с индексом |
| `bzip2` | Классический bzip2 | Для `.bz2`-архивов |
| `lz4` | Быстрый компрессор LZ4 | Для быстрого сжатия (используется в zram) |
| `lzop` | Компрессор lzop | Для `.lzo`-архивов (kernel initrd) |
| `ca-certificates` | Корневые сертификаты | TLS-проверка для HTTPS |
| `bash-completion` | Автодополнение bash | TAB-автодополнение команд и путей |
| `less` | Пейджер | Чтение длинных файлов с навигацией |
| `patch` | Утилита patch | Наложение патчей (diff → apply) |
| `dialog` | TUI-инструмент для shell | Меню, input-боксы, file pickers в скриптах. Альтернатива `whiptail` с более широкими возможностями (gauge, calendar, checklist, form). Полезен для установочных и конфигурационных скриптов |

## 📖 Man-страницы

| Пакет | Что это | Зачем нужен |
| :--- | :--- | :--- |
| `man-db` | `man`, `mandb`, `apropos`, `whatis` | Сам `man` и индексация страниц. `debootstrap --variant=minbase` не ставит его, поэтому явно |
| `manpages` | Базовые man-страницы | Coreutils, bash, sed, awk и т.д. Без этого `man ls` → `No manual entry for ls` |
| `manpages-dev` | Системные вызовы и libc | `man 2 read`, `man 3 printf`. Тянет `manpages` по зависимостям. Критично для разработки |
| `manpages-posix-dev` | POSIX-стандарты | Спецификации POSIX для NetOps/DevOps, где регулярно нужны стандарты |

## 💻 Разработка

| Пакет | Что это | Зачем нужен |
| :--- | :--- | :--- |
| `git` | VCS | Управление исходным кодом |
| `make` | Сборщик | Сборка пакетов из исходников |
| `binutils` | Бинарные утилиты | `ld`, `as`, `objdump`, `nm`, `strip` |
| `gnutls-bin` | CLI GnuTLS | `certtool` для работы с сертификатами |
| `python3-venv` | venv для Python | Изолированные Python-окружения |
| `python3-rich` | Библиотека форматирования | Красивый вывод в консоли Python |

## 🔐 SSH

| Пакет | Что это | Зачем нужен |
| :--- | :--- | :--- |
| `openssh-client` | SSH-клиент | `ssh`, `scp`, `sftp`, `ssh-keygen` |
| `keychain` | Менеджер ssh-agent | Переиспользование ssh-agent между сессиями (не вводить пароль каждый раз) |
| `ssh-askpass` | GUI-запрос пароля SSH | Диалог для пароля при GUI-вызове ssh |

## 🌐 Сеть

| Пакет | Что это | Зачем нужен |
| :--- | :--- | :--- |
| `systemd-resolved` | DNS-резолвер | Stub на `127.0.0.53`, кэш, DNSSEC |
| `wpasupplicant` | Wi-Fi-аутентификация | WPA2/WPA3, roaming |
| `iw` | Утилиты Wi-Fi | Настройка беспроводных интерфейсов (замена `iwconfig`) |
| `libsasl2-modules` | Модули SASL | PLAIN, LOGIN, CRAM-MD5 для SMTP/LDAP |
| `libsasl2-modules-gssapi-mit` | GSSAPI/Kerberos для SASL | Аутентификация Kerberos |
| `nftables` | Файрвол | Современный netfilter (замена iptables) |
| `publicsuffix` | Public Suffix List | Обновляемый PSL для TLS-верификации (`libpsl`) |
| `dns-root-data` | Корневые DNS-ключи | Для DNSSEC в `systemd-resolved` |

## 🔊 Аудио (PipeWire)

| Пакет | Что это | Зачем нужен |
| :--- | :--- | :--- |
| `pipewire` | Аудио/видео сервер | Замена PulseAudio и JACK |
| `pipewire-pulse` | PulseAudio API в PipeWire | Совместимость со старыми приложениями |
| `pipewire-alsa` | ALSA-совместимость | Для программ, использующих ALSA напрямую |
| `pipewire-audio` | Метапакет аудио | Аудио-модули PipeWire |
| `wireplumber` | Session manager | Политики PipeWire |
| `libpipewire-0.3-modules-extra` | Доп. модули PipeWire | Фильтры, эффекты |
| `libpipewire-0.3-common` | Общие данные PipeWire | Конфиги, схемы |
| `alsa-utils` | Утилиты ALSA | `alsamixer`, `aplay`, `arecord` |
| `alsa-ucm-conf` | ALSA UCM | Use Case Manager (профили устройств) |
| `alsa-topology-conf` | ALSA topology | Описания DSP-топологий |
| `libasound2-plugins` | ALSA-плагины | Pulse, Jack, OSS-плагины |
| `libcanberra-pulse` | Звуки событий | Звуковые сигналы GUI (через Pulse) |
| `rtkit` | Realtime Kit | Даёт аудио-демонам приоритет реального времени |
| `firmware-sof-signed` | Прошивка Sound Open Firmware | Подписанная прошивка Intel SOF DSP (обязательна — `snd_sof_intel_hda` реально загружен на i5-11400) |

## 🚌 D-Bus, PAM, Polkit

| Пакет | Что это | Зачем нужен |
| :--- | :--- | :--- |
| `dbus` | Межпроцессная шина | System bus + user bus |
| `dbus-user-session` | Сессионная шина D-Bus | Автозапуск user bus |
| `libpam-systemd` | PAM-модуль systemd | Интеграция сессий с logind |
| `libpam-cap` | PAM-модуль capabilities | Управление capability при логине |
| `polkitd` | Демон polkit | Агент авторизации привилегированных действий |
| `pkexec` | Утилита pkexec | Запуск с полномочиями через polkit |
| `libpwquality1` | Проверка качества паролей | Оценка сложности паролей |
| `cracklib-runtime` | Словарь cracklib | **Критично**: без него `libpwquality1` не работает, `passwd` падает с `error loading dictionary` |

## 🔑 Секреты и крипто-железо

| Пакет | Что это | Зачем нужен |
| :--- | :--- | :--- |
| `libsecret-1-0` | Secret Service клиент | Библиотека для хранения секретов |
| `keyutils` | Управление kernel keyring | Работа с ядровыми ключами |
| `libnss-systemd` | NSS-модуль systemd | userdb, dynamic users |
| `pass` | Менеджер паролей | GPG-шифрованные пароли, интеграция с git |
| `pass-otp` | OTP для pass | TOTP/HOTP токены в pass |
| `oathtool` | Генератор OATH | TOTP/HOTP для 2FA |
| `qrencode` | Генератор QR | Кодирование `otpauth://`-URI в QR |
| `dirmngr` | Демон GnuPG | CRL/OCSP, WKD, keyserver для gpg |
| `scdaemon` | Смарт-карт-демон GnuPG | Работа с OpenPGP-картами |
| `pcscd` | Демон PC/SC | Смарт-карты, YubiKey PIV |
| `libfido2-1` | FIDO2/WebAuthn | Аппаратные ключи (YubiKey) |
| `acl` | POSIX ACL | Расширенные права доступа к файлам |
| `libengine-pkcs11-openssl` | PKCS#11 engine для OpenSSL | HSM/смарт-карты для TLS/SSH |

## ⏱️ Systemd-таймеры и хуки

| Пакет | Что это | Зачем нужен |
| :--- | :--- | :--- |
| `systemd-timesyncd` | NTP-клиент | Синхронизация времени |
| `systemd-oomd` | OOM-демон userspace | Проактивное убийство процессов при нехватке RAM |
| `systemd-cryptsetup` | LUKS в systemd | Разблокировка через `/etc/crypttab` |
| `systemd-hwe-hwdb` | HWE hwdb | Обновлённая база оборудования |
| `linux-sysctl-defaults` | Дефолтные sysctl | Настройки ядра по умолчанию |

## 🌡️ Мониторинг и железо

| Пакет | Что это | Зачем нужен |
| :--- | :--- | :--- |
| `thermald` | Intel thermal daemon | Управление троттлингом CPU (для Intel) |
| `lm-sensors` | Утилиты сенсоров | `sensors`, `pwmconfig` для вентиляторов |
| `i2c-tools` | Утилиты I2C | `i2cdetect`, `i2cget` (датчики, DDC) |
| `fancontrol` | Управление вентиляторами | Скрипт для pwm-fan |
| `mesa-vulkan-drivers` | Vulkan-драйверы Mesa | llvmpipe fallback (пока Nvidia не установлен) |
| `mesa-utils` | Утилиты Mesa | `glxinfo`, `eglinfo`, `vulkaninfo` |
| `pciutils` | Утилиты PCI | `lspci`, `setpci` |
| `usbutils` | Утилиты USB | `lsusb` |
| `read-edid` | Чтение EDID | Мониторы и их характеристики |
| `v4l-utils` | Утилиты Video4Linux | `v4l2-ctl` для камер (BRIO) |

## 🎨 Загрузка и оформление

| Пакет | Что это | Зачем нужен |
| :--- | :--- | :--- |
| `plymouth` | Графический splash | Экран загрузки |
| `plymouth-themes` | Темы Plymouth | spinner, glow и др. |
| `grub-theme-breeze` | Тема GRUB Breeze | Визуальная тема меню |

## 🌍 Локали и MIME

| Пакет | Что это | Зачем нужен |
| :--- | :--- | :--- |
| `locales` | Генерация локалей | `locale-gen`, `update-locale` |
| `console-setup` | Настройка консоли | Шрифты, раскладка консоли |
| `kbd` | Утилиты клавиатуры | `loadkeys`, `setfont` для tty |
| `xdg-user-dirs` | XDG-каталоги | `~/Documents`, `~/Downloads` и т.д. |
| `xdg-utils` | XDG-утилиты | `xdg-open`, `xdg-mime` |
| `libglib2.0-data` | Данные GLib | Схемы GSettings, переводы |
| `libfile-mimeinfo-perl` | Определение MIME | `file-mimeinfo` для xdg-utils |

## 🖥️ X11 / XWayland

| Пакет | Что это | Зачем нужен |
| :--- | :--- | :--- |
| `xauth` | Авторизация X11 | `.Xauthority` для XWayland |
| `x11-utils` | Утилиты X11 | `xdpyinfo`, `xev`, `xwininfo` |
| `x11-xserver-utils` | Утилиты Xserver | `xrandr`, `xset` (для XWayland) |
| `xclip` | Буфер X11 | CLI-доступ к X11-clipboard |

## 🎬 Графика / GStreamer

| Пакет | Что это | Зачем нужен |
| :--- | :--- | :--- |
| `librsvg2-common` | SVG-рендеринг (GDK) | Иконки и SVG в GTK-приложениях |
| `librsvg2-bin` | Утилита rsvg-convert | CLI-конвертация SVG |
| `gstreamer1.0-plugins-good` | GStreamer good | `v4l2src`, `audioconvert` и др. |
| `gstreamer1.0-tools` | CLI GStreamer | `gst-launch`, `gst-inspect` |
| `gstreamer1.0-x` | GStreamer X11 | `ximagesink`, `xvimagesink` |
| `opus-tools` | Кодек Opus | `opusenc`, `opusdec` |
| `libnotify-bin` | CLI-уведомления | `notify-send` |

## 🌊 Wayland / FUSE

| Пакет | Что это | Зачем нужен |
| :--- | :--- | :--- |
| `wl-clipboard` | CLI-буфер Wayland | `wl-copy`, `wl-paste` |
| `fuse3` | FUSE 3 | Монтирование FUSE-ФС (AppImage, sshfs) |

## 🔬 Отладка и профилирование

| Пакет | Что это | Зачем нужен |
| :--- | :--- | :--- |
| `bpftool` | Утилита eBPF | Инспекция BPF-программ, карт |
| `linux-tools-7.0.0-42-generic` | Инструменты ядра | `perf`, `bpftool` под это ядро |
| `linux-perf` | perf (метапакет) | Профилирование |

## 🔄 Обслуживание

| Пакет | Что это | Зачем нужен |
| :--- | :--- | :--- |
| `unattended-upgrades` | Автообновления | Установка security-патчей |
| `apt-listchanges` | Changelog при обновлении | Показ изменений в пакетах |
| `needrestart` | Перезапуск сервисов | Автоматический рестарт после обновления библиотек |
| `restic` | Бэкапы | Дедуплицированный шифрованный бэкап |
| `zram-tools` | Настройка zram | `zram-generator` (сжатый swap в RAM) |

## 📡 Firmware

| Пакет | Что это | Зачем нужен |
| :--- | :--- | :--- |
| `linux-firmware-misc` | Прошивки разные | Catch-all для разнообразного |
| `linux-firmware-intel-wireless` | Wi-Fi Intel | Прошивки `iwlwifi` (AX210) |
| `linux-firmware-intel-misc` | Разное Intel | BT, чипсет |

> Остальные подпакеты (`-amd-graphics`, `-intel-graphics`, `-nvidia-graphics`, `-broadcom-wireless`, `-realtek`, `-mediatek`, `-marvell-*`, `-qlogic`, `-netronome`, `-mellanox-spectrum`, `-qualcomm-*`) приедут транзитивно через метапакет `linux-firmware`, который тянет `linux-generic-hwe-26.04`. Явно перечислять их бессмысленно — это документирование, а не экономия.

---

## 🚫 Сознательно исключённые пакеты

Эти пакеты были в Suggested/Recommended или транзитивных зависимостях предыдущих итераций, но исключены осознанно — с обоснованием.

| Пакет | Почему исключён |
| :--- | :--- |
| `network-manager` | Заменён на `systemd-networkd` + `wpa_supplicant` + `systemd-resolved`. Тяжёлая надстройка для стационарного ПК с одним Wi-Fi и одним Ethernet. |
| `libmm-glib0` | Жёсткая зависимость NetworkManager. Уходит вместе с ним. |
| `dnsmasq-base` | Тянулся для NetworkManager. `systemd-resolved` занимает порт 53 и конфликтует с `dnsmasq`. |
| `modemmanager` | Для 3G/4G-модемов. У станции их нет. |
| `upower` | Демон батареи. У стационарного ПК батареи нет. `libupower-glib3` приезжает как библиотека — это нормально. |
| `imagemagick-7-common`, `libmagickcore-*` | Тянулись транзитивно через `zbar-tools`. Удаление `zbar-tools` убирает всю цепочку (~40 МБ). |
| `zbar-tools` | QR-коды не нужны. `pass-otp` умеет добавлять токены из `otpauth://`-URI напрямую, без QR. |
| `libimobiledevice-*`, `libusbmuxd-*`, `libplist-*` | iOS-стек. У станции нет iOS-устройств. Уходили через `upower`/`usbmuxd`. |
| `tpm2daemon` | TPM 2.0 есть, но не используется для GPG/SSH в базовой поставке. При необходимости — `tpm2-tools` отдельно. |
| `libcamera-ipa`, `pipewire-libcamera`, `gstreamer1.0-libcamera`, `libspa-0.2-libcamera` | BRIO — стандартная UVC-камера (`uvcvideo`), ей эти пакеты не нужны. Они для ISP-пайплайнов ноутбучных камер (Intel IPU6 и т.п.). |
| `va-driver-all` | У станции нет Intel iGPU (отключён в BIOS) и нет AMD GPU. VA-API для Nvidia ставится отдельно (`nvidia-vaapi-driver`). |
| `linux-firmware-intel-graphics` | Intel iGPU отключён в BIOS — в `lspci` только NVIDIA. |
| `linux-firmware-amd-graphics` | Нет AMD GPU. |
| `linux-firmware-nvidia-graphics` | GSP-прошивка для RTX 3070 идёт в самом драйвере Nvidia. Отдельный firmware-пакет не нужен. |
| `linux-firmware-broadcom-wireless` | Wi-Fi — Intel AX210. |
| `linux-firmware-realtek` | Нет Realtek-сети; NVMe-контроллер RTS5762 прошивку не требует. |
| `gnome-keyring` | Есть `pass` + `keychain` + `libsecret-1-0` (клиентская библиотека). Полный Secret Service при необходимости доставляется отдельно. |
| `netplan.io`, `networkd-dispatcher` | Netplan — прослойка над NM/networkd; при прямых `.network`-файлах бесполезна. `networkd-dispatcher` — для хуков на события, в базовой поставке не нужен. |
| `ubuntu-kernel-accessories` | Метапакет из Recommended. Модули `v4l2loopback` и `zfs` уже приехали через `linux-main-modules-*` в составе HWE-ядра. |
| `speex` (бинарник) | Кодек Speex. `libspeex1` и `libspeexdsp1` приезжают с PipeWire как библиотеки — это нормально. |
| `monkeysphere` | Управление SSH-ключами через OpenPGP. Ниша, не нужна. |
| `ghostscript`, `fonts-urw-base35` | Из ImageMagick. Уходят вместе с `zbar-tools`. |
| `python3-gi`, `python3-tk` | GTK-биндинги и Tkinter. Не нужны. |
| `libjs-jquery`, `libjs-sphinxdoc`, `sphinx-rtd-theme-common` | JS-библиотеки и темы документации. Не нужны. |
| `network-manager-pptp`, `ppp` | PPTP устарел, PPP для модемов. Не нужны. |
