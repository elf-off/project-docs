# Руководство по развёртыванию CEO Agent

## Содержание
1. [Требования](#требования)
2. [Установка на bare metal](#установка-на-bare-metal)
3. [Установка через Docker](#установка-через-docker)
4. [Настройка Telegram бота](#настройка-telegram-бота)
5. [Настройка БД](#настройка-бд)
6. [Мониторинг и логи](#мониторинг-и-логи)

## Требования

### Минимальные

- Python 3.11+
- MariaDB/MySQL 5.7+
- 1GB RAM
- 10GB диск

### Рекомендуемые

- Python 3.11+
- MariaDB 10.11+
- 2GB RAM
- 20GB диск
- SSD

## Установка на bare metal

### 1. Подготовка системы

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install python3.11 python3.11-venv python3-pip
sudo apt install default-libmysqlclient-dev build-essential

# CentOS/RHEL
sudo yum install python311 python311-devel
sudo yum install mariadb-devel gcc
```

### 2. Создание пользователя

```bash
sudo useradd -r -m -s /bin/bash ceo-agent
sudo mkdir -p /opt/ceo_agent
sudo chown ceo-agent:ceo-agent /opt/ceo_agent
```

### 3. Установка приложения

```bash
# От пользователя ceo-agent
sudo su - ceo-agent
cd /opt/ceo_agent

# Клонирование/копирование кода
# ... скопируйте файлы проекта ...

# Создание venv
python3.11 -m venv venv
source venv/bin/activate

# Установка зависимостей
pip install -r requirements.txt
```

### 4. Конфигурация

```bash
cp .env.example .env
nano .env
```

Заполните все обязательные параметры:
```env
DB_HOST=localhost
DB_NAME=your_db
DB_USER=readonly_user
DB_PASSWORD=secure_password

TELEGRAM_BOT_TOKEN=123456:ABC-DEF...
TELEGRAM_CEO_CHAT_ID=123456789
TELEGRAM_ADMIN_CHAT_ID=987654321

ANTHROPIC_API_KEY=sk-ant-...
```

### 5. Инициализация БД

```bash
python setup_db.py
```

### 6. Тестирование

```bash
python test_agent.py
```

Проверьте:
1. Подключение к БД
2. Подключение к Telegram
3. Расчёт метрик
4. Обработку тестовой смены

### 7. Установка systemd service

```bash
# От root
sudo cp ceo-agent.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable ceo-agent
sudo systemctl start ceo-agent

# Проверка статуса
sudo systemctl status ceo-agent

# Просмотр логов
journalctl -u ceo-agent -f
```

## Установка через Docker

### 1. Подготовка

```bash
# Установка Docker и Docker Compose
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

sudo apt install docker-compose-plugin
```

### 2. Конфигурация

```bash
cp .env.example .env
nano .env
```

### 3. Сборка и запуск

```bash
# Сборка
docker-compose build

# Запуск
docker-compose up -d

# Просмотр логов
docker-compose logs -f ceo-agent
```

### 4. Инициализация БД

```bash
# Выполнение setup_db.py внутри контейнера
docker-compose exec ceo-agent python setup_db.py
```

## Настройка Telegram бота

### 1. Создание бота

1. Откройте Telegram и найдите @BotFather
2. Отправьте команду `/newbot`
3. Следуйте инструкциям
4. Сохраните токен

### 2. Получение Chat ID

#### Для CEO:
1. Создайте бота (см. выше)
2. CEO должен написать боту `/start`
3. Получите chat_id через API:

```bash
curl https://api.telegram.org/bot<TOKEN>/getUpdates
```

Или используйте специального бота @userinfobot

#### Для менеджеров:
То же самое - каждый менеджер должен написать боту `/start`

### 3. Настройка маппинга

```bash
python setup_db.py
```

Выберите "Добавить менеджера" и введите:
- ID проекта
- Имя менеджера
- Telegram Chat ID

## Настройка БД

### 1. Создание read-only пользователя

```sql
-- Подключение к MySQL/MariaDB
mysql -u root -p

-- Создание пользователя
CREATE USER 'ceo_agent_ro'@'%' IDENTIFIED BY 'secure_password';

-- Права только на чтение существующих таблиц
GRANT SELECT ON your_database.ref_projects_wa TO 'ceo_agent_ro'@'%';
GRANT SELECT ON your_database.ref_shifts TO 'ceo_agent_ro'@'%';
GRANT SELECT ON your_database.exits_wa TO 'ceo_agent_ro'@'%';

-- Права на создание и запись в таблицы агента
GRANT ALL PRIVILEGES ON your_database.agent_* TO 'ceo_agent_ro'@'%';

FLUSH PRIVILEGES;
```

### 2. Проверка подключения

```bash
mysql -h localhost -u ceo_agent_ro -p -D your_database

# Проверка read-only
SELECT * FROM ref_projects_wa LIMIT 1;  -- OK
INSERT INTO ref_projects_wa ...;         -- ERROR (read-only)
```

### 3. Оптимизация

```sql
-- Индексы для производительности
CREATE INDEX idx_exits_project_date ON exits_wa(project_id, date, shift_id);
CREATE INDEX idx_exits_date_shift ON exits_wa(date, shift_id);
```

## Мониторинг и логи

### Просмотр логов

```bash
# Systemd
journalctl -u ceo-agent -f
journalctl -u ceo-agent --since today

# Docker
docker-compose logs -f ceo-agent
docker-compose logs --tail=100 ceo-agent

# Файлы (если настроено)
tail -f /opt/ceo_agent/logs/agent.log
```

### Метрики

Агент логирует структурированные JSON логи. Пример:

```json
{
  "timestamp": "2026-01-29T11:30:00+01:00",
  "level": "info",
  "module": "orchestrator",
  "action": "shift_processed",
  "snapshot_id": 123,
  "duration": 2.5
}
```

### Алерты

Агент отправляет алерты админу при:
- Ошибках подключения к БД
- Ошибках обработки смены
- Критических ошибках

Все алерты также логируются в `agent_audit_logs`.

### Проверка здоровья

```bash
# Ручная проверка
python test_agent.py

# Проверка через БД
mysql -u ceo_agent_ro -p -e "SELECT * FROM agent_audit_logs ORDER BY timestamp DESC LIMIT 10;"

# Проверка последнего снапшота
mysql -u ceo_agent_ro -p -e "SELECT * FROM agent_snapshots ORDER BY captured_at DESC LIMIT 1;"
```

## Резервное копирование

### Backup БД

```bash
# Автоматический backup (добавьте в cron)
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
mysqldump -u backup_user -p your_database \
  agent_snapshots \
  agent_snapshot_records \
  agent_incidents \
  agent_telegram_messages \
  agent_reasons \
  agent_risk_scores \
  agent_ceo_decisions \
  agent_audit_logs \
  agent_project_managers \
  > /backup/ceo_agent_$DATE.sql

# Сжатие
gzip /backup/ceo_agent_$DATE.sql

# Удаление старых (> 30 дней)
find /backup -name "ceo_agent_*.sql.gz" -mtime +30 -delete
```

### Cron задача

```bash
# Добавьте в crontab
crontab -e

# Backup каждый день в 23:00
0 23 * * * /path/to/backup_script.sh
```

## Обновление

### Обновление кода

```bash
# Остановка сервиса
sudo systemctl stop ceo-agent

# От пользователя ceo-agent
cd /opt/ceo_agent
git pull  # или скопируйте новые файлы

# Обновление зависимостей
source venv/bin/activate
pip install -r requirements.txt --upgrade

# Миграции БД (если есть)
# python migrate.py

# Запуск
sudo systemctl start ceo-agent
sudo systemctl status ceo-agent
```

### Откат версии

```bash
sudo systemctl stop ceo-agent
cd /opt/ceo_agent
git checkout <previous-version>
pip install -r requirements.txt
sudo systemctl start ceo-agent
```

## Troubleshooting

### Агент не запускается

```bash
# Проверка логов
journalctl -u ceo-agent -n 50

# Проверка окружения
sudo -u ceo-agent bash
cd /opt/ceo_agent
source venv/bin/activate
python main.py  # Запуск вручную
```

### Ошибки БД

```bash
# Проверка подключения
mysql -h <DB_HOST> -u <DB_USER> -p

# Проверка существования таблиц
SHOW TABLES LIKE 'agent_%';

# Проверка прав
SHOW GRANTS FOR 'ceo_agent_ro'@'%';
```

### Telegram не работает

```bash
# Проверка токена
curl https://api.telegram.org/bot<TOKEN>/getMe

# Проверка chat_id
curl https://api.telegram.org/bot<TOKEN>/getUpdates

# Тест отправки
python test_agent.py
# Выберите: 2. Тест подключения к Telegram
```

### Высокая нагрузка

```bash
# Проверка процессов
top -u ceo-agent

# Проверка БД запросов
# В MySQL
SHOW FULL PROCESSLIST;

# Оптимизация - уменьшите period_days в конфиге
# или добавьте индексы в БД
```

## Безопасность

### Рекомендации

1. **Используйте read-only пользователя** для основной БД
2. **Храните токены в .env**, не коммитьте в git
3. **Регулярно обновляйте** зависимости
4. **Настройте firewall** для БД
5. **Используйте SSL** для подключения к БД
6. **Мониторьте логи** на подозрительную активность

### SSL для БД

```env
# В .env добавьте
DB_SSL_CA=/path/to/ca-cert.pem
DB_SSL_CERT=/path/to/client-cert.pem
DB_SSL_KEY=/path/to/client-key.pem
```

## Поддержка

При возникновении проблем:

1. Проверьте логи
2. Запустите тесты
3. Проверьте аудит в БД
4. Откройте issue в репозитории

---

**Документация обновлена:** 29.01.2026
