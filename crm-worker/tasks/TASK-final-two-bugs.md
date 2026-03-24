# ЗАДАЧА: Два оставшихся бага из аудита

## КРИТИЧНО — выполнить оба пункта. Без них система работает с ошибками.

---

## БАГ 1: Дублирование сообщений в БД

### Проблема

При отправке AI-сообщения создаются ДВЕ записи в таблице `messages`:
- Первая: в `message_scheduler.py` → `schedule_message()` (scheduled_at заполнен)
- Вторая: в `avito_messenger.py` → `send_message()` (scheduled_at = NULL)

Это приводит к тому, что каждое сообщение бота показывается в интерфейсе дважды.

### Диагностика

```bash
grep -rn "send_message" services/ --include="*.py" | grep -v "__pycache__"
```

Определи: вызывается ли `send_message` из `avito_messenger.py` ТОЛЬКО из `message_scheduler.py`, или ещё откуда-то напрямую.

### Решение

**Файл: `services/avito_messenger.py`**

Добавь параметр `skip_db: bool = False` в функцию `send_message()`.

Найди блок где создаётся Message (примерно строки 56-64):
```python
msg = Message(...)
session.add(msg)
```

Оберни в условие:
```python
if not skip_db:
    msg = Message(...)
    session.add(msg)
```

Сигнатура функции:
```python
async def send_message(account_id: int, chat_id: str, text: str, skip_db: bool = False) -> dict:
```

**Файл: `services/message_scheduler.py`**

В функции `process_scheduled()` найди вызов `send_message()` и добавь `skip_db=True`:

```python
# БЫЛО (примерно):
await send_message(account_id, avito_chat_id, msg.content)

# СТАЛО:
await send_message(account_id, avito_chat_id, msg.content, skip_db=True)
```

Запись в messages уже создана при планировании — дублировать не нужно.

### Проверка

После изменения:
```bash
python -c "from services.avito_messenger import send_message; print('OK')"
python -c "from services.message_scheduler import process_scheduled; print('OK')"
```

---

## БАГ 2: LLM-ошибки помечают сессию failed

### Проблема

Когда Claude API возвращает 529 (перегружен), сессия помечается `failed` и кандидат навсегда остаётся без ответа. Даже если через минуту API восстановится — сессия уже мёртвая.

`ai_claude.py` уже имеет retry + fallback, но `ai_agent.py` всё равно ставит `failed` при любой ошибке.

### Решение

**Файл: `services/ai_agent.py`**

#### Шаг 1: Добавь функцию проверки типа ошибки (перед `_mark_session_failed`):

```python
_LLM_ERROR_MARKERS = ["529", "502", "503", "429", "500", "anthropic", "openai", "timeout", "fallback"]

def _is_llm_error(error) -> bool:
    """Проверяет, является ли ошибка временной LLM-ошибкой."""
    error_str = str(error).lower()
    return any(marker in error_str for marker in _LLM_ERROR_MARKERS)
```

#### Шаг 2: Измени `_mark_session_failed` — добавь параметр `error`:

Найди текущую функцию (примерно строка 988):
```python
async def _mark_session_failed(ai_session_id: int) -> None:
```

Замени на:
```python
async def _mark_session_failed(ai_session_id: int, error=None) -> None:
    """Помечает сессию failed, НО ТОЛЬКО если ошибка не связана с LLM."""
    if error and _is_llm_error(error):
        log.warning(
            "session_llm_error_skip_failed",
            ai_session_id=ai_session_id,
            error=str(error)[:200],
        )
        try:
            from utils.event_logger import log_event
            await log_event(None, "session_llm_skip", f"Сессия {ai_session_id}: LLM-ошибка, НЕ помечена failed")
        except Exception:
            pass
        return

    async with AsyncSessionFactory() as session:
        sess = await session.get(AISession, ai_session_id)
        if sess:
            sess.status = "failed"
            sess.dialog_stage = "failed"
            await session.commit()
    log.error("session_marked_failed", ai_session_id=ai_session_id)
```

#### Шаг 3: Обнови ВСЕ вызовы `_mark_session_failed` — передавай error

Найди все вызовы:
```bash
grep -n "_mark_session_failed" services/ai_agent.py
```

Каждый вызов находится внутри блока `except Exception as exc:`. Замени:

```python
# БЫЛО:
await _mark_session_failed(ai_session.id)

# СТАЛО:
await _mark_session_failed(ai_session.id, error=exc)
```

Вот конкретные строки (номера могут отличаться — проверь grep-ом):

1. После `greeting_failed` → `_mark_session_failed(ai_session.id)` → добавить `error=exc`
2. После `followup_failed` → `_mark_session_failed(ai_session_id)` → добавить `error=exc`
3. После `presentation_failed` → `_mark_session_failed(ai_session.id)` → добавить `error=exc`
4. После `booking_failed` → `_mark_session_failed(ai_session.id)` → добавить `error=exc`
5. После `alternatives_failed` → `_mark_session_failed(ai_session.id)` → добавить `error=exc`

**ВАЖНО:** Убедись что в каждом блоке except переменная ошибки называется именно `exc`. Если где-то `e` — используй `e`.

### Проверка

```bash
python -c "from services.ai_agent import _mark_session_failed, _is_llm_error; print('OK')"
```

---

## ПОРЯДОК РАБОТЫ

1. Прочитай `services/avito_messenger.py` — найди где создаётся Message
2. Прочитай `services/message_scheduler.py` — найди где вызывается send_message
3. Примени фикс дублей (skip_db)
4. Прочитай `services/ai_agent.py` — найди _mark_session_failed и все её вызовы
5. Добавь _is_llm_error()
6. Измени _mark_session_failed (добавь error=)
7. Обнови все 5 вызовов (добавь error=exc)
8. Проверь импорты

## ПОСЛЕ ВЫПОЛНЕНИЯ

Покажи:
- `grep -n "skip_db" services/avito_messenger.py services/message_scheduler.py`
- `grep -n "_is_llm_error\|error=exc\|error=e" services/ai_agent.py`
- Результат проверки импортов
