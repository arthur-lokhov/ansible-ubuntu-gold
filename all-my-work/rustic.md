### 💾 Этап 5.4: Автоматическое резервное копирование (restic)

Резервное копирование настраивается через `restic` — дедуплицирующий, шифрованный инструмент. Он не требует демона, работает из cron и сохраняет данные в репозиторий на внешнем диске или в удалённом хранилище. Скрипт бэкапа включает критичные директории, исключая кэши, и автоматически удаляет старые снимки, оставляя 7 дневных, 4 недельных и 6 месячных копий.

Создаём скрипт бэкапа:

```bash
cat > /usr/local/bin/backup.sh <<'EOF'
#!/bin/bash
set -euo pipefail

# Путь к репозиторию (внешний диск должен быть смонтирован)
export RESTIC_REPOSITORY="/mnt/backup/restic"
export RESTIC_PASSWORD_FILE="/etc/restic/password"

# Что бэкапим
BACKUP_PATHS=(/home /etc)

# Бэкап (с исключением кэшей)
restic backup "${BACKUP_PATHS[@]}" --exclude-caches

# Политика хранения
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
EOF
chmod +x /usr/local/bin/backup.sh
```

Пароль хранится в отдельном файле с правами `600`, чтобы не светить его в `ps` или истории команд:

```bash
mkdir -p /etc/restic
echo "ваш-надежный-пароль" > /etc/restic/password
chmod 600 /etc/restic/password
```

Инициализация репозитория (выполняется **один раз**, при смонтированном внешнем диске):

```bash
# ВАЖНО: /mnt/backup должен быть смонтирован до этой команды
restic init
```

Добавление задачи в `cron` (ежедневно в 3:00):

```bash
(crontab -l 2>/dev/null; echo "0 3 * * * /usr/local/bin/backup.sh >> /var/log/restic.log 2>&1") | crontab -
```

> [!warning] Проверка целостности
> `restic` не проверяет все данные при каждом запуске (это долго). Раз в месяц или после крупных обновлений полезно запускать `restic check`. Для критичных данных — `restic check --read-data` (читает все блоки, медленно, но выявляет битые сектора на диске).
