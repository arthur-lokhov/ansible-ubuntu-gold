# Ubuntu 26.04 LTS — экспертная установка рабочей станции

Воспроизводимая сборка зашифрованной рабочей станции Ubuntu 26.04 LTS (Resolute Raccoon) на Ansible: GPT + LUKS2 + LVM + XFS, UEFI Secure Boot, `systemd-networkd`, NVIDIA/DKMS, libvirt/KVM и rootful containerd/nerdctl/BuildKit.

Проект вырос из документа [`docs/expert-install.md`](docs/expert-install.md) — это спецификация и объяснение каждого шага. Ansible-реализация находится в [`ansible/`](ansible/), подробная инструкция по запуску — в [`ansible/README.md`](ansible/README.md).

> [!warning] Playbook установки может полностью уничтожить целевой диск
> Переразметка выполняется только с явным флагом `-e disk_confirm_wipe=true`.

## Состав репозитория

| Путь | Назначение |
| --- | --- |
| `ansible/` | Playbook, 13 ролей, inventory, шаблоны |
| `ansible/playbooks/install.yml` | Этап 1: запуск с LiveCD, сборка системы в `/mnt` |
| `ansible/playbooks/post-install.yml` | Этап 2: NVIDIA, libvirt, контейнеры, restic |
| `ansible/README.md` | Полная инструкция: требования, секреты, запуск, troubleshooting |
| `ansible/docs/MAPPING.md` | Карта «раздел документа → роль/задача» |
| `docs/expert-install.md` | Исходная спецификация |
| `docs/packages.md` | Обоснование каждого пакета |
| `docs/restic.md` | Рецепт резервного копирования (роль `restic`) |

## Требования

* Ubuntu 26.04 LTS LiveUSB, загруженный в **UEFI** (Secure Boot можно оставить
  включённым).
* Целевой диск не меньше `disk_min_size_gb` (по умолчанию 560 GiB).
* `ansible-core` и `python3-apt` в Live-среде.
* Сеть на LiveCD — для клонирования репозитория и загрузки пакетов.

## Быстрый старт на LiveCD

### 1. Установить инструменты и скачать репозиторий

```bash
sudo apt update
sudo apt install -y git ansible-core python3-apt

# замените <account> на свой GitHub/GitLab-аккаунт
git clone https://github.com/<account>/ubuntu-gold.git
cd ubuntu-gold/ansible
```

Если `git` недоступен или удобнее скачать архив:

```bash
curl -L https://github.com/<account>/ubuntu-gold/archive/refs/heads/main.tar.gz \
  | tar -xz
cd ubuntu-gold-main/ansible
```

Можно также скопировать каталог на USB-накопитель и работать с него: внешние
зависимости проекту не нужны, кроме `ansible-core`.

### 2. Создать и зашифровать секреты

```bash
cp inventory/group_vars/all/secrets.yml.example \
   inventory/group_vars/all/secrets.yml
$EDITOR inventory/group_vars/all/secrets.yml
ansible-vault encrypt inventory/group_vars/all/secrets.yml
```

`inventory/group_vars/all/secrets.yml` уже в `.gitignore`. **Никогда не
коммитьте его** и не публикуйте значения из него.

### 3. Проверить настройки

```bash
$EDITOR inventory/group_vars/all/main.yml
```

Как минимум: `target_disk`, `workstation_hostname`, `workstation_user`,
`wifi_ssid`, `timezone`.

### 4. Запустить установку

```bash
ansible-playbook playbooks/install.yml \
  -e disk_confirm_wipe=true \
  --ask-vault-pass
```

Playbook не перезагружает машину. После завершения нужно вручную
перезагрузиться, пройти MOK Manager и ввести пароль LUKS. Точные шаги и
проверки — в [`ansible/README.md`](ansible/README.md).

### 5. Post-install (после первого входа)

```bash
doas apt install -y ansible-core
doas env HOME=/root ansible-playbook \
  playbooks/post-install.yml --ask-vault-pass
```

## Публикация репозитория

Репозиторий уже инициализирован (ветка `main`), но remote не настроен:

```bash
# GitHub CLI
gh repo create ubuntu-gold --public --source=. --remote=origin --push

# или вручную
git remote add origin git@github.com:<account>/ubuntu-gold.git
git push -u origin main
```

Перед публикацией проверьте, что в индекс не попали секреты:

```bash
git status
git ls-files | grep -i secret   # должен остаться только secrets.yml.example
```

Лицензия не добавлена: выберите её сами (MIT, Apache-2.0, GPL-3.0 и т.п.),
если планируете, чтобы код могли переиспользовать.

## Документация

* [`ansible/README.md`](ansible/README.md) — эксплуатация, переменные, ограничения.
* [`docs/expert-install.md`](docs/expert-install.md) — полная спецификация.
* [`ansible/docs/MAPPING.md`](ansible/docs/MAPPING.md) — соответствие документу.

## Статус

Playbook использует только `ansible-core` и встроенные модули, без Galaxy-
коллекций и внешних ролей. Перед боевым запуском проверьте сборку на тестовом
диске: установка необратимо переразмечает целевой носитель.
