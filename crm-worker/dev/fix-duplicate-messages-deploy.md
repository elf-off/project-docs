# Внедрение: Исправление дублирования сообщений

## Что изменилось

### `services/avito_messenger.py`

Удалён блок создания записи `Message` в БД из функции `send_message()`.
Раньше при каждой отправке создавалась дублирующая запись (первая создавалась
при планировании в `message_scheduler.py`, вторая — при отправке в `avito_messenger.py`).

Теперь `send_message()` только отправляет сообщение через Avito API и возвращает результат.

### `services/message_scheduler.py`

В `process_scheduled()` добавлено:
- Сохранение `avito_message_id` из ответа API в существующую запись `Message`
- Обновление `last_message_at` в таблице `Chat` (раньше это делалось в `avito_messenger.py`)

## Проверка после деплоя

### 1. Проверка импортов

```bash
python -c "from services.avito_messenger import send_message; print('OK')"
python -c "from services.message_scheduler import process_scheduled; print('OK')"
```

### 2. Проверка отсутствия дублей

После деплоя дождаться отправки нескольких AI-сообщений и проверить в БД:

```sql
SELECT chat_id, content, COUNT(*) as cnt
FROM messages
WHERE direction = 'outgoing' AND sender_type = 'ai'
  AND created_at > NOW() - INTERVAL 1 HOUR
GROUP BY chat_id, content
HAVING cnt > 1;
```

Результат должен быть пустым (нет дублей).

### 3. Проверка заполнения avito_message_id

```sql
SELECT id, avito_message_id, delivered_at
FROM messages
WHERE direction = 'outgoing' AND sender_type = 'ai'
  AND delivered_at > NOW() - INTERVAL 1 HOUR
ORDER BY delivered_at DESC
LIMIT 10;
```

У доставленных сообщений должен быть заполнен `avito_message_id`.

## Откат

Если нужно откатить, восстановить предыдущую версию двух файлов:
- `services/avito_messenger.py`
- `services/message_scheduler.py`

```bash
git revert <commit-hash>
sudo systemctl restart k24-crm-worker
```

## Перезапуск сервиса

```bash
sudo systemctl restart k24-crm-worker
sudo systemctl status k24-crm-worker
```
