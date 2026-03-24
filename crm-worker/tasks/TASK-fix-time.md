# ЗАДАЧА: Исправить дублирование сообщений в БД

## Проблема

При отправке AI-сообщения через scheduler создаются **две записи** в таблице `messages`:

1. `message_scheduler.py` строка ~30 — создаёт Message при планировании (`scheduled_at` заполнен)
2. `avito_messenger.py` строка ~56 — создаёт ещё одну Message при отправке (`scheduled_at = NULL`)

В результате в интерфейсе каждое сообщение бота показывается дважды.

## Диагностика

Сначала проверь, откуда вызывается `send_message`:

```bash
grep -rn "send_message" /opt/openai/crm-worker/services/ /opt/openai/crm-worker/api/ --include="*.py"
```

Возможны два сценария:

### Сценарий А: ВСЕ отправки идут через schedule_message

Если `send_message` вызывается ТОЛЬКО из `message_scheduler.py` (process_scheduled), то:
- **Убрать** создание Message из `avito_messenger.py` полностью
- Запись уже создаётся при планировании в `message_scheduler.py`
- В `process_scheduled` уже обновляется `delivered_at`

### Сценарий Б: Есть прямые вызовы send_message (не через scheduler)

Если `send_message` вызывается ещё откуда-то напрямую (не через scheduler), то:
- **Добавить** параметр `skip_db: bool = False` в `send_message()`
- Если `skip_db=True` — НЕ создавать запись в messages
- В `process_scheduled()` при вызове `send_message` передавать `skip_db=True`
- Все остальные вызовы останутся без изменений

## Что менять

### Файл: `services/avito_messenger.py`

В зависимости от сценария:
- **А:** Убрать блок создания Message (строки ~56-64)
- **Б:** Добавить `skip_db: bool = False` в сигнатуру `send_message()`, обернуть создание Message в `if not skip_db:`

### Файл: `services/message_scheduler.py`

Только для сценария Б:
- В `process_scheduled()` где вызывается `send_message` — добавить `skip_db=True`

## Больше ничего не менять

Не трогать: ai_agent.py, webhooks.py, промпты, шаблоны, админку.

---

# ЗАДАЧА 2: Время в админке — показывать по Москве (UTC+3)

## Проблема

В БД всё время хранится в UTC. В админке (`templates/admin.html`) время отображается как есть — без поправки на часовой пояс. Нужно показывать по Москве (UTC+3).

## Что менять

### Файл: `templates/admin.html`

Добавь JS-функцию для конвертации UTC → Moscow:

```javascript
function toMoscow(utcString) {
    if (!utcString) return '—';
    const utc = new Date(utcString + 'Z');  // Явно указываем что это UTC
    return utc.toLocaleString('ru-RU', {
        timeZone: 'Europe/Moscow',
        day: '2-digit',
        month: '2-digit',
        hour: '2-digit',
        minute: '2-digit',
        second: '2-digit'
    });
}
```

Применить эту функцию ко ВСЕМ местам где отображается время:
- Раздел "Аккаунты" — столбец `token_expires_at`
- Раздел "Логи" — время событий
- Раздел "Карточки" — `created_at`, `processed_at`
- Модалка диалога — время сообщений

Найди все места где выводится дата/время и оберни в `toMoscow()`.

### Файл: `templates/login.html`

Не трогать.

## После исправления

Проверь что импорты работают:
```bash
python -c "from services.avito_messenger import send_message; print('OK')"
python -c "from services.message_scheduler import process_scheduled; print('OK')"
```
