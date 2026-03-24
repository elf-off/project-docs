# Доставка карточек менеджерам -- внедрение и запуск

## Что реализовано

### Измененные файлы

| Файл | Что сделано |
|------|-------------|
| `models/db.py` | Добавлено поле `telegram_topic_id` в модель `AvitoAccount` |
| `config.py` | Добавлен `telegram_group_id` (ID супергруппы с топиками) |
| `api/admin.py` | 4 эндпоинта карточек + `telegram_topic_id` в create/update аккаунтов + параметр `account_id` в тесте Telegram |
| `services/telegram_notifier.py` | Отправка по аккаунтам в их топики группы, поддержка `account_id` |
| `templates/admin.html` | Раздел "Карточки", модальное окно диалога, поле `telegram_topic_id` в форме аккаунтов |
| `main.py` | Scheduler-задача `telegram_morning_summary` (cron, МСК) |

### Новые файлы

| Файл | Назначение |
|------|------------|
| `services/telegram_notifier.py` | Модуль отправки уведомлений в Telegram (утренняя сводка по аккаунтам в топики) |

---

## Миграция БД

Выполнить SQL на MariaDB:

```sql
ALTER TABLE avito_accounts ADD COLUMN telegram_topic_id INT NULL;
```

---

## Настройка

### 1. Переменные окружения (.env)

Добавить в `.env`:

```bash
TELEGRAM_BOT_TOKEN=123456:ABC-DEF1234...    # токен от @BotFather
TELEGRAM_GROUP_ID=-1001234567890             # ID супергруппы с топиками
TELEGRAM_MORNING_HOUR=10                     # час отправки (МСК), по умолчанию 10
TELEGRAM_MORNING_MINUTE=0                    # минута отправки, по умолчанию 0
```

`TELEGRAM_CHAT_ID` -- сохранен для обратной совместимости, но основной параметр теперь `TELEGRAM_GROUP_ID`.

### 2. Создание Telegram-бота

1. Написать @BotFather в Telegram: `/newbot`
2. Скопировать токен в `TELEGRAM_BOT_TOKEN`
3. Создать супергруппу с топиками (Включить "Темы" в настройках группы)
4. Добавить бота в группу с правами администратора
5. Узнать group_id (можно через `https://api.telegram.org/bot<TOKEN>/getUpdates` после отправки сообщения в группу)
6. Создать топики для каждого аккаунта Avito

### 3. Заполнение telegram_topic_id для аккаунтов

Через админку: редактирование аккаунта -> поле "Telegram Topic ID".

Или через SQL:

```sql
UPDATE avito_accounts SET telegram_topic_id = <TOPIC_ID> WHERE id = <ACCOUNT_ID>;
```

Узнать topic_id: отправить сообщение в топик, затем вызвать `getUpdates` -- в объекте message будет `message_thread_id`.

Если `telegram_topic_id` не заполнен -- уведомления для этого аккаунта не отправляются.

### 4. Проверка доступности Telegram API

```bash
curl -s "https://api.telegram.org/bot<TOKEN>/getMe"
```

Если Telegram API заблокирован -- добавить SOCKS5-прокси в `telegram_notifier.py`:

```python
httpx.AsyncClient(timeout=15.0, proxy="socks5://127.0.0.1:1080")
```

---

## API-эндпоинты

### GET /admin/api/handover
Список карточек с фильтрами.

Параметры:
- `limit` (int, default=50)
- `offset` (int, default=0)
- `account_id` (int, optional) -- фильтр по аккаунту
- `result` (str, optional) -- booking / interested / not_interested / alternative
- `unprocessed_only` (bool, default=true)

### GET /admin/api/handover/{id}/messages
Полная переписка по карточке (chat messages).

### POST /admin/api/handover/{id}/process
Пометить карточку как обработанную (is_processed=1, processed_at=NOW).

### POST /admin/api/telegram/test-summary
Тестовая отправка утренней сводки.

Параметры:
- `account_id` (int, optional) -- только для конкретного аккаунта

Примеры:
```bash
# Все аккаунты
curl -X POST http://127.0.0.1:8800/admin/api/telegram/test-summary \
  -H "Cookie: admin_session=<SESSION>"

# Конкретный аккаунт
curl -X POST "http://127.0.0.1:8800/admin/api/telegram/test-summary?account_id=2" \
  -H "Cookie: admin_session=<SESSION>"
```

---

## Проверка после деплоя

### 1. Проверка импортов

```bash
python -c "from services.telegram_notifier import send_morning_summary; print('Telegram OK')"
python -c "from api.admin import router; print('Admin API OK')"
```

### 2. Тестовая отправка Telegram

Через админку: раздел "Карточки" -> кнопка "Тест Telegram".

### 3. Раздел карточек в админке

URL: `https://<HOST>/admin/` -> меню "Карточки"

Фильтры:
- По аккаунту (dropdown)
- По результату (booking / interested / not_interested / alternative)
- Только необработанные (checkbox, по умолчанию включен)

---

## Scheduler

Утренняя сводка отправляется автоматически каждый день в настроенное время (по умолчанию 10:00 МСК).
Задача: `telegram_morning_summary` (APScheduler, cron trigger).

Логика:
1. Выбирает активные аккаунты с заполненным `telegram_topic_id`
2. Для каждого аккаунта: находит необработанные карточки (`is_processed = 0`)
3. Отправляет в топик аккаунта: заголовок -> каждую карточку отдельным сообщением -> итог
4. Пауза 0.3с между сообщениями (rate limit Telegram)
5. Если Telegram не настроен (пустой token/group_id) -- пропускает без ошибки
6. Аккаунты без `telegram_topic_id` -- пропускаются
