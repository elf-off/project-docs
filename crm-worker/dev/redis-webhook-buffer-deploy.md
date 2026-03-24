# Redis Streams: буфер для входящих вебхуков

## Описание

Webhook endpoint теперь мгновенно отвечает `{"ok": true}`, а обработку выполняет фоновый consumer из Redis Streams. При недоступности Redis - fallback на синхронную обработку (как было раньше).

## Измененные файлы

| Файл | Что изменено |
|------|-------------|
| `config.py` | Добавлены `redis_host`, `redis_port`, `redis_db`, `redis_password` |
| `requirements.txt` | Добавлен `redis[hiredis]>=5.0.0` |
| `api/webhooks.py` | Endpoint: enqueue в Redis, fallback на sync при ошибке Redis |
| `main.py` | Startup: ensure_consumer_groups + запуск WebhookConsumer, shutdown: stop + close_redis |
| `api/admin.py` | Добавлен `GET /admin/api/queues` для мониторинга |

## Новые файлы

| Файл | Назначение |
|------|-----------|
| `services/redis_queue.py` | Обертка над Redis Streams (enqueue, consume, ack, recover) |
| `workers/webhook_consumer.py` | Consumer: читает из стримов, вызывает бизнес-логику |
| `migrations/006_redis_setup.md` | Инструкция по установке Redis |

## Архитектура

```
Avito webhook -> POST /webhooks/avito
                    |
                    v
            enqueue_webhook(stream, payload)
                    |
           +--------+--------+
           |                  |
  webhooks:messenger   webhooks:applications
           |                  |
           +--------+---------+
                    |
            WebhookConsumer._consume_loop()
                    |
           _process_messenger()  /  _process_application()
                    |
          (существующая бизнес-логика)
```

## Два стрима

- `webhooks:messenger` - сообщения из чатов Avito
- `webhooks:applications` - отклики на вакансии

## Поведение при ошибках

| Ситуация | Поведение |
|----------|-----------|
| Redis недоступен при enqueue | Fallback: синхронная обработка (как было) |
| Redis недоступен при consume | Worker ждет 10 сек, пробует снова |
| Ошибка в handler | Нет ack, сообщение в pending |
| Pending > 5 мин | recovery_loop перехватывает и повторяет |
| > 3 retry | ack + логируем как failed |

## Переменные окружения

```env
# Дефолты работают для локального Redis без пароля
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_DB=0
REDIS_PASSWORD=
```

## Деплой

### 1. Установить Redis
```bash
sudo apt update && sudo apt install redis-server -y
sudo systemctl enable redis-server
sudo systemctl start redis-server
redis-cli ping  # PONG
```

### 2. Установить зависимости
```bash
pip install -r requirements.txt
```

### 3. Перезапустить сервис
```bash
sudo systemctl restart k24-crm-worker
```

### 4. Проверить
```bash
# Логи: должен быть redis_group_created + webhook_consumer_started
journalctl -u k24-crm-worker -f | grep -E "redis|consumer"

# Стримы созданы
redis-cli XINFO STREAM webhooks:messenger
redis-cli XINFO STREAM webhooks:applications

# Тестовый вебхук
curl -X POST http://localhost:8800/webhooks/avito \
  -H "Content-Type: application/json" \
  -d '{"test": true}'

# Длина очереди
redis-cli XLEN webhooks:messenger

# Мониторинг через админку
curl http://localhost:8800/admin/api/queues -H "Cookie: admin_session=..."
```

### 5. Проверить fallback (опционально)
```bash
# Остановить Redis
sudo systemctl stop redis-server

# Отправить вебхук - должен обработаться синхронно
# В логах: redis_unavailable_fallback_sync

# Вернуть Redis
sudo systemctl start redis-server
```
