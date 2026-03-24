# ЗАДАЧА: 4 фикса в Telegram-уведомлениях

## КРИТИЧНО — все 4 пункта связаны с telegram_notifier.py

---

## ФИКС 1: Утренняя сводка — только карточки за ночь

### Проблема
`send_morning_summary()` отправляет ВСЕ необработанные карточки (`is_processed=0`). Должна отправлять только карточки созданные за прошедшую ночь (с 21:00 вчера до 09:00 сегодня).

### Решение

**Файл: `services/telegram_notifier.py`**

В `send_morning_summary()` добавь фильтр по времени:

```python
from datetime import datetime, timedelta
import pytz

moscow_tz = pytz.timezone("Europe/Moscow")
now_moscow = datetime.now(moscow_tz)
# Ночное окно: вчера 21:00 → сегодня 09:00
night_start = now_moscow.replace(hour=21, minute=0, second=0, microsecond=0) - timedelta(days=1)
night_end = now_moscow.replace(hour=9, minute=0, second=0, microsecond=0)

# Конвертируем в UTC для сравнения с БД
night_start_utc = night_start.astimezone(pytz.utc).replace(tzinfo=None)
night_end_utc = night_end.astimezone(pytz.utc).replace(tzinfo=None)
```

Добавь в запрос:
```python
.where(HandoverCard.created_at >= night_start_utc)
.where(HandoverCard.created_at <= night_end_utc)
```

Полный фильтр должен быть:
```python
select(HandoverCard)
.join(Application, HandoverCard.application_id == Application.id)
.where(
    Application.account_id == account.id,
    HandoverCard.is_processed == False,
    HandoverCard.created_at >= night_start_utc,
    HandoverCard.created_at <= night_end_utc,
)
.order_by(HandoverCard.created_at.desc())
```

---

## ФИКС 2: Отправка в правильный топик (message_thread_id)

### Проблема
Сообщения уходят в общий чат группы, а не в топик аккаунта.

### Решение

**Файл: `services/telegram_notifier.py`**

Проверь функцию `send_telegram_message()`. Убедись что `message_thread_id` передаётся корректно:

```python
async def send_telegram_message(topic_id: int | None, text: str, parse_mode: str = "HTML") -> bool:
    if not settings.telegram_bot_token or not settings.telegram_group_id:
        log.warning("telegram_not_configured")
        return False

    try:
        api_url = f"https://api.telegram.org/bot{settings.telegram_bot_token}"
        payload = {
            "chat_id": int(settings.telegram_group_id),   # Убедись что int, не str
            "text": text,
            "parse_mode": parse_mode,
        }
        # Топик добавляем ТОЛЬКО если задан
        if topic_id is not None and topic_id > 0:
            payload["message_thread_id"] = int(topic_id)

        async with httpx.AsyncClient(
            timeout=15.0,
            proxy=settings.claude_proxy,  # ФИКС 3: прокси
        ) as client:
            resp = await client.post(f"{api_url}/sendMessage", json=payload)
            resp.raise_for_status()
            return True
    except Exception as e:
        log.error("telegram_send_error", topic_id=topic_id, error=str(e))
        return False
```

**Ключевые моменты:**
- `chat_id` должен быть `int`, не `str` — Telegram API для суперграмм с топиками требует числовой ID
- `message_thread_id` добавлять только если `topic_id` задан и > 0
- Проверь что `telegram_group_id` в config.py хранится как строка, а при отправке конвертируется в int

**Также проверь** что в `send_morning_summary()` при вызове `send_telegram_message()` передаётся `account.telegram_topic_id`:

```python
await send_telegram_message(account.telegram_topic_id, header)
await send_telegram_message(account.telegram_topic_id, text)
await send_telegram_message(account.telegram_topic_id, footer)
```

---

## ФИКС 3: Прокси для Telegram API

### Проблема
Telegram API может быть заблокирован в РФ. Нужно использовать тот же SOCKS5-прокси что для Claude.

### Решение

**Файл: `services/telegram_notifier.py`**

В `send_telegram_message()` добавь прокси в httpx.AsyncClient:

```python
async with httpx.AsyncClient(
    timeout=15.0,
    proxy=settings.claude_proxy,  # socks5://127.0.0.1:1080
) as client:
```

`settings.claude_proxy` уже настроен и используется в `ai_claude.py`. Тот же прокси будет работать для Telegram.

---

## ФИКС 4: Баг test-summary endpoint

### Проблема (из аудита)
`api/admin.py` строка ~390: `send_morning_summary(account_id=account_id)` — функция не принимает этот параметр → TypeError.

### Решение

**Вариант А:** Добавить параметр `account_id` в `send_morning_summary()`:

```python
async def send_morning_summary(account_id: int | None = None) -> int:
```

Если `account_id` задан — фильтровать только по этому аккаунту:
```python
if account_id:
    accounts_query = select(AvitoAccount).where(
        AvitoAccount.id == account_id,
        AvitoAccount.is_active == True,
        AvitoAccount.telegram_topic_id != None,
    )
else:
    accounts_query = select(AvitoAccount).where(
        AvitoAccount.is_active == True,
        AvitoAccount.telegram_topic_id != None,
    )
```

---

## ПОРЯДОК РАБОТЫ

1. Прочитай `services/telegram_notifier.py` целиком
2. Прочитай вызов в `api/admin.py` (строка ~390)
3. Примени все 4 фикса
4. Проверь импорты

## ПОСЛЕ ВЫПОЛНЕНИЯ

```bash
# Проверка
python -c "from services.telegram_notifier import send_morning_summary, send_telegram_message; print('OK')"

# Проверить что прокси используется
grep -n "proxy\|claude_proxy" services/telegram_notifier.py

# Проверить что topic_id передаётся
grep -n "message_thread_id\|topic_id" services/telegram_notifier.py

# Проверить что фильтр по времени есть
grep -n "night_start\|night_end\|21\|09" services/telegram_notifier.py

# Проверить что account_id принимается
grep -n "def send_morning_summary" services/telegram_notifier.py
```
