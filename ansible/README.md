# 🛠️ Ubuntu 26.04 LTS: Автоматизированная экспертная установка (Ansible)

Этот репозиторий превращает процесс ручной установки из документации в полностью воспроизводимый проект **Infrastructure as Code (IaC)** на базе Ansible. 

Это **не** обертка над стандартным установщиком Ubuntu. Процесс полностью контролируется: загружается LiveCD, утилита `debootstrap` создает минимальную целевую систему, поверх которой собирается стек хранения (LUKS2/LVM/XFS), а каждый конфигурационный файл записывается явно, без скрытых дефолтов.

> [!WARNING] Деструктивная операция.
> Плейбук установки может полностью уничтожить данные на целевом диске. Разметка диска произойдет только при передаче флага `-e disk_confirm_wipe=true`.
> Без этого флага плейбук попытается переиспользовать существующую разметку (режим возобновления после сбоя). Прочитайте документацию перед первым запуском.

---

## 📂 Структура проекта

```text
ansible/
├── ansible.cfg
├── inventory/
│   ├── hosts.yml                    # livecd + workstation (connection=local)
│   └── group_vars/all/
│       ├── main.yml                 # Основные настройки (без секретов)
│       ├── packages.yml             # Манифест устанавливаемых пакетов
│       └── secrets.yml.example      # Копируется в secrets.yml и шифруется Vault
├── playbooks/
│   ├── install.yml                  # Этап 1: выполняется из LiveCD
│   └── post-install.yml             # Этап 2: выполняется в установленной системе
├── roles/
│   ├── preflight/                   # Проверки UEFI, диска и переменных
│   ├── disk/                        # Разметка GPT, LUKS2, LVM, XFS
│   ├── debootstrap/                 # Установка минимальной базы
│   ├── chroot_mounts/               # Проброс /dev, /proc, /sys, /run
│   ├── apt_repo/                    # Настройка DEB822, TLS, блокировка Snap
│   ├── packages/                    # Установка базового списка пакетов
│   ├── base_config/                 # Настройка сети, fstab, пользователей
│   ├── boot/                        # GRUB, MOK, Plymouth
│   ├── finalize/                    # Очистка, размонтирование
│   ├── nvidia/                      # Установка драйверов NVIDIA (Post-install)
│   ├── libvirt/                     # Настройка KVM/QEMU (Post-install)
│   ├── containers/                  # Настройка containerd/nerdctl (Post-install)
│   └── restic/                      # Опциональное резервное копирование
├── docs/MAPPING.md                  # Таблица соответствия документации и ролей
└── artifacts/                       # Слепки железа, собранные с LiveCD
```

---

## 📋 Требования

*   LiveUSB с Ubuntu 26.04 LTS, загруженный в режиме **UEFI** (Secure Boot можно оставить включенным).
*   Целевой диск объемом не менее `disk_min_size_gb` (по умолчанию 560 ГиБ).
*   Наличие `ansible-core` в Live-среде. В LiveCD его нет, установите через `venv`:
    ```bash
    sudo apt update && sudo apt install -y python3-venv python3-apt git
    python3 -m venv .venv && source .venv/bin/activate
    pip install --upgrade pip ansible-core
    ```
*   Скопированный проект (через USB, `scp` или `git clone`).

*Примечание: Профиль оборудования (Intel i5-11400, AX210, RTX 3070) зашит в дефолтные значения. На другом железе потребуется изменить `target_disk`, список firmware-пакетов и `debootstrap_arch`.*

---

## 🚀 Быстрый старт

### 1. Подготовка LiveCD
```bash
# Опционально: доступ по SSH к Live-среде
sudo apt update && sudo apt install -y openssh-server
sudo passwd ubuntu
sudo systemctl start ssh
ip -br addr
```

### 2. Создание файла секретов
```bash
cp inventory/group_vars/all/secrets.yml.example \
   inventory/group_vars/all/secrets.yml
# Отредактируйте файл, указав пароли, затем зашифруйте его:
ansible-vault encrypt inventory/group_vars/all/secrets.yml
```

| Ключ в Vault | Назначение |
| :--- | :--- |
| `vault_luks_passphrase` | Пароль от LUKS2 (минимум 8 символов) |
| `vault_user_password` | Пароль для ежедневного пользователя |
| `vault_wifi_psk` | Пароль от Wi-Fi сети |
| `vault_mok_password` | Одноразовый пароль для MOK Manager |
| `vault_restic_password` | Пароль для бэкапов (если `restic_enabled=true`) |

*Имя Wi-Fi сети (SSID) не является секретом и настраивается в `main.yml` (переменная `wifi_ssid`).*

### 3. Настройка переменных
Проверьте в `inventory/group_vars/all/main.yml` как минимум:
```yaml
target_disk: /dev/nvme0n1
workstation_hostname: ustation
workstation_user: lohovap
wifi_ssid: "Ваш_SSID"
timezone: Europe/Moscow
```
Имена сетевых интерфейсов (`wifi_interface` / `ethernet_interface`) определяются автоматически, но их можно переопределить вручную.

### 4. Запуск установки из LiveCD
```bash
# Первый запуск (уничтожает данные на диске)
ansible-playbook playbooks/install.yml \
  -e disk_confirm_wipe=true \
  --ask-vault-pass
```

**Что произойдет (по шагам):**
1. Проверка UEFI, прав, наличия секретов и целевого диска.
2. Создание таблицы GPT (ESP 1G, /boot 1G, LUKS остаток).
3. `cryptsetup luksFormat` (AES-XTS-512), создание LVM `vg0`, XFS и FAT32.
4. Запуск `debootstrap --arch=amd64v3 --variant=minbase`.
5. Проброс системных директорий (`/dev`, `/proc`, `/sys`, `/run`) в `chroot`.
6. Настройка HTTPS DEB822, блокировка Snap, установка 650+ пакетов.
7. Написание всех конфигов (сеть, `doas`, fstab, sysctl, ZRAM, пользователи).
8. Установка GRUB, генерация MOK-ключа, Plymouth.
9. Очистка кэша и размонтирование.

> [!IMPORTANT]  
> Плейбук **не перезагружает систему автоматически**. Если произошел сбой — просто запустите плейбук снова, но с флагом `-e disk_confirm_wipe=false`. Ansible продолжит с места остановки, не форматируя диск заново.

### 5. Первая загрузка (Критический этап!)
1. Отключите LiveUSB и перезагрузитесь.
2. В синем меню **MOK Manager** выберите `Enroll MOK` → `Continue` → `Yes`, введите пароль MOK и нажмите `Reboot`.
3. Введите пароль LUKS (в консольной раскладке `us`).
4. Войдите под пользователем `workstation_user` (вход под root заблокирован).

**Проверка работоспособности:**
```bash
doas id                                         # uid=0(root)
ip a && ping -c 3 1.1.1.1
sysctl net.ipv4.tcp_congestion_control          # bbr
zramctl                                         # /dev/zram0
```

### 6. Post-Install (Опционально)
Поскольку `doas` не поддерживается в Ansible как `become_method` из коробки, второй плейбук запускается от имени `root` с сохранением текущего окружения:

```bash
# На установленной системе:
doas env HOME=/root PATH="$PATH" ansible-playbook \
  playbooks/post-install.yml --ask-vault-pass
```

Вы можете запускать отдельные роли через теги:
```bash
... --tags nvidia       # Установка nvidia-driver-610 и настройка DKMS
... --tags libvirt      # Установка KVM/QEMU
... --tags containers   # Установка nerdctl, containerd, buildkit
```

---

## ⚙️ Ключевые переменные

| Переменная | По умолчанию | Описание |
| :--- | :--- | :--- |
| `target_disk` | `/dev/nvme0n1` | Целевой диск для разметки. |
| `disk_confirm_wipe` | `false` | Защита от случайного удаления данных. |
| `disk_min_size_gb` | `560` | Минимальный размер диска для дефолтной схемы LVM. |
| `lvm_volumes` | См. `main.yml` | Размеры и точки монтирования LVM. Последний том должен быть `100%FREE`. |
| `debootstrap_arch` | `amd64v3` | Замените на `amd64` для процессоров старше Haswell. |
| `wifi_enabled` | `true` | Установите `false` для машин только с Ethernet. |
| `plymouth_theme` | `text` | Тема вывода загрузки (вместо логотипа). |
| `nvidia_driver_package` | `nvidia-driver-610` | Транзитивный пакет, тянущий ветку 615 с патчами. |

---

## ⚠️ Отклонения от оригинального документа

В оригинальном тексте были внутренние противоречия. Плейбук принимает следующие решения:
1.  **Список пакетов:** `manpages-posix-dev`, `qrencode`, `xfsdump`, `python3-rich`, `linux-perf` и версионные `linux-tools` исключены. Заменены на `linux-tools-generic-hwe-26.04`. Темы `grub-theme-breeze` и `plymouth-themes` не ставятся (GRUB использует текстовое меню, Plymouth использует базовую тему `text`).
2.  **ZRAM:** Документ ставит `zram-tools`, но пишет конфиг от `systemd-zram-generator`. В плейбуке ставится только `systemd-zram-generator`, а `zram-tools` удаляется во избежание конфликтов.
3.  **fstab:** Генерируется нативными модулями Ansible из фактов UUID, а оригинальный bash-скрипт устанавливается в систему как `/usr/local/sbin/gen-fstab` для будущего обслуживания.
4.  **Генерация MOK:** Команда `mokutil --generate-key` устарела. Ключ создается через `openssl req` с параметрами `rsa:2048`.

---

## 🛑 Что нельзя автоматизировать

Эти шаги намеренно оставлены человеку:
*   **Подтверждение MOK Manager**: Secure Boot требует физического нажатия кнопок при следующей загрузке. Плейбук только создает запрос `mokutil --import`.
*   **Перезагрузка**: Автоматическая перезагрузка свежеустановленной системы с шифрованием — плохая идея.
*   **Ввод пароля LUKS**: Вводится с консоли. Плейбук хранит пароль в Vault только для форматирования и открытия контейнера во время установки.
*   **Проверка Wi-Fi**: Ассоциация с точкой доступа проверяется уже после первой загрузки через `wpa_cli status`.

---

## 🛠️ Решение проблем (Troubleshooting)

| Симптом | Причина / Решение |
| :--- | :--- |
| `does not contain the expected three partitions` | Первый запуск: добавьте `-e disk_confirm_wipe=true`. |
| `unable to open pty` или падение Ansible (`process object is closed`) | Баг Python 3.14. Используйте `ansible-core` 2.20+ в `venv` и добавьте `forks = 1` в `ansible.cfg`. |
| `apt-get install` не находит пакет | Зеркало Ubuntu 26.04 может отличаться. Отредактируйте `inventory/group_vars/all/packages.yml`. |
| `nvidia-smi` не работает после установки | Перезагрузитесь, пройдите MOK Manager. Проверьте `dkms status` и `/etc/dkms/framework.conf`. |
| `nerdctl` не подключается | Проверьте `systemctl status containerd buildkit` и наличие сокета `/run/containerd/containerd.sock`. |

---

## 🔒 Заметки по безопасности

*   Вход под `root` заблокирован. Единственный путь получения привилегий — группа `wheel` и утилита `doas`.
*   Секреты хранятся только в зашифрованном Ansible Vault.
*   Флаг `disk_confirm_wipe` — это осознанная защита от случайного уничтожения данных.
*   В целевой системе не хранятся пароли LUKS, только заголовок и ключевые слоты.
*   Использование Rootful-контейнеров (`nerdctl`) — задокументированный компромисс. Не запускайте непроверенные образы.
