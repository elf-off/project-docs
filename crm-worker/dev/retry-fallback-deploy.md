# Retry + Fallback на OpenAI GPT-4o

## Описание

Двухуровневая отказоустойчивость при вызове LLM:
1. Retry Claude - до 5 попыток с экспоненциальной паузой
2. Fallback на OpenAI GPT-4o - если Claude не ответил

## Измененные файлы

| Файл | Что изменено |
|------|-------------|
| `config.py` | Добавлены `openai_fallback_model` (gpt-4o), `claude_max_retries` (5) |
| `services/ai_claude.py` | Retry 5 попыток, fallback на OpenAI, логирование событий |
| `services/ai_agent.py` | LLM-ошибки не помечают сессию failed, сессия остается активной |

## Логика retry

- **Количество попыток:** 5
- **Паузы между попытками:** 2s, 5s, 15s, 30s, 60s
- **Retryable коды:** 429, 500, 502, 503, 529
- **Non-retryable коды** (400, 401, 403): сразу переход к fallback

## Логика fallback

1. Все 5 попыток Claude провалились (или non-retryable ошибка)
2. Формируется запрос к OpenAI GPT-4o с теми же system/messages
3. Если OpenAI тоже упал - бросается оригинальная ошибка Claude

## Обработка ошибок в ai_agent.py

- LLM-ошибки (содержат маркеры: 529, 502, 503, 429, 500, anthropic, openai, timeout):
  сессия НЕ помечается failed, остается в текущем stage для повторной обработки
- Прочие ошибки (БД, парсинг): по-прежнему помечаются failed

## Логирование

| Событие | event_type | Описание |
|---------|-----------|----------|
| Claude retry | `claude_retry` | Каждая повторная попытка с номером и задержкой |
| Fallback | `claude_fallback` | Переключение на OpenAI |
| OpenAI OK | `openai_fallback_ok` | Fallback успешен |
| OpenAI fail | `openai_fallback_fail` | Fallback тоже не удался |
| LLM ошибка | `session_error` | Сессия оставлена активной при LLM-ошибке |

## Переменные окружения

```env
# Уже существующие (используются):
OPENAI_API_KEY=sk-proj-...

# Новые (опциональные, есть дефолты):
OPENAI_FALLBACK_MODEL=gpt-4o      # дефолт: gpt-4o
CLAUDE_MAX_RETRIES=5               # дефолт: 5
```

## Проверка

```bash
# Импорт работает
python -c "from services.ai_claude import call_claude; print('OK')"

# Проверка конфига
python -c "from config import settings; print(f'retries={settings.claude_max_retries}, fallback={settings.openai_fallback_model}')"
```

## Деплой

1. Никаких миграций БД не требуется
2. `.env` менять не нужно (дефолты достаточны)
3. Перезапустить сервис:
```bash
sudo systemctl restart k24-crm-worker
```
4. Проверить логи на наличие событий `claude_retry` / `claude_fallback`:
```bash
journalctl -u k24-crm-worker -f | grep -E "claude_retry|fallback"
```
