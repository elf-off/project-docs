# Деплой: Два финальных бага из аудита

## Что изменено

### Баг 1: Дублирование сообщений в БД

**Проблема:** Каждое AI-сообщение создавало ДВЕ записи в `messages` — одну при планировании (`schedule_message`), вторую при отправке (`send_message`).

**Файлы:**
- `services/avito_messenger.py` — добавлен параметр `skip_db: bool = False` в `send_message()`. Когда `skip_db=True`, запись в БД пропускается.
- `services/message_scheduler.py` — в `process_scheduled()` вызов `send_message()` теперь передает `skip_db=True`, т.к. запись уже создана планировщиком.

### Баг 2: LLM-ошибки помечают сессию failed

**Проблема:** При временных ошибках Claude API (529, 502, 503, таймаут) сессия помечалась `failed` навсегда, кандидат оставался без ответа.

**Файл:** `services/ai_agent.py`
- Добавлена функция `_is_llm_error(error)` — проверяет, является ли ошибка временной LLM-ошибкой по маркерам: `529`, `502`, `503`, `429`, `500`, `anthropic`, `openai`, `timeout`, `fallback`.
- Функция `_mark_session_failed()` теперь принимает параметр `error=None`. Если ошибка LLM — сессия НЕ помечается failed, только логируется warning.
- Все 5 вызовов `_mark_session_failed` обновлены: передается `error=exc`.

## Затронутые вызовы _mark_session_failed

| Место | Этап | Строка (примерно) |
|---|---|---|
| `process_new_application` | greeting | 275 |
| `process_followup` | followup | 378 |
| `_send_presentation_and_fork` | presentation | 550 |
| `_send_booking` | booking | 630 |
| `_send_alternatives` | alternatives | 781 |

## Деплой

```bash
# 1. Обновить код
cd /opt/crm-worker
git pull

# 2. Перезапустить сервис
sudo systemctl restart k24-crm-worker

# 3. Проверить логи
sudo journalctl -u k24-crm-worker -f --no-pager | head -50
```

## Проверка после деплоя

### Баг 1 (дубли)
```sql
-- Проверить что новые сообщения не дублируются
-- (у scheduled сообщений scheduled_at != NULL, delivered_at заполняется при отправке)
SELECT id, chat_id, content, scheduled_at, delivered_at, created_at
FROM messages
WHERE direction = 'outgoing' AND sender_type = 'ai'
ORDER BY created_at DESC
LIMIT 20;

-- Не должно быть пар с одинаковым content и chat_id в пределах минуты
SELECT chat_id, content, COUNT(*) as cnt
FROM messages
WHERE direction = 'outgoing' AND sender_type = 'ai'
  AND created_at > NOW() - INTERVAL 1 HOUR
GROUP BY chat_id, content
HAVING cnt > 1;
```

### Баг 2 (LLM failed)
```bash
# Должны появляться логи session_llm_error_skip_failed вместо session_marked_failed
# при временных ошибках API
sudo journalctl -u k24-crm-worker --since "1 hour ago" | grep -E "session_llm_error_skip|session_marked_failed"
```

## Откат

Изменения обратно совместимы. При необходимости откат через `git revert`.
