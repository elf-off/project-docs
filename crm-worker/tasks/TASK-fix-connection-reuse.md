# TASK: Исправить переиспользование соединений в ai_claude.py

## Файл
`/opt/openai/crm-worker/services/ai_claude.py`

## Проблема

Retry работает (5 попыток Claude + 3 попытки OpenAI), но ВСЕ попытки падают с одной и той же ошибкой "Server disconnected without sending a response".

Лог:
```
18:01:38 HTTP Request: POST https://api.anthropic.com/v1/messages "HTTP/1.1 200 OK"   ← первый запрос ПРОШЁЛ
18:01:53 claude_retry attempt=1 error='Server disconnected without sending a response.'
18:02:10 claude_retry attempt=2 error='Server disconnected without sending a response.'
18:02:30 claude_retry attempt=3 error='Server disconnected without sending a response.'
18:03:00 claude_retry attempt=4 error='Server disconnected without sending a response.'
18:03:46 claude_all_retries_exhausted attempts=5
18:04:01 openai_fallback_attempt_failed attempt=1 error='Server disconnected...'
18:04:19 openai_fallback_attempt_failed attempt=2 error='Server disconnected...'
18:04:44 openai_fallback_attempt_failed attempt=3 error='Server disconnected...'
```

**Причина:** `_claude_client()` создаётся ОДИН РАЗ на все 5 попыток:

```python
async with _claude_client() as client:        # ← один клиент
    for attempt in range(MAX_RETRIES):         # ← все retry через него
        resp = await client.post(...)          # ← broken connection
```

Сервер закрыл TCP-соединение после первого запроса. httpx пытается переиспользовать мёртвое соединение → все retry бьют в одну закрытую трубу.

## Что сделать

### 1. Перенести создание клиента ВНУТРЬ цикла retry

В функции `ask_claude()` найти блок с retry и перенести `async with _claude_client()` внутрь `for`:

**Было:**
```python
async with _claude_client() as client:
    for attempt in range(MAX_RETRIES):
        start_ms = int(_time.monotonic() * 1000)
        try:
            resp = await client.post(CLAUDE_API_URL, json=payload, headers=headers)
            ...
```

**Стало:**
```python
for attempt in range(MAX_RETRIES):
    start_ms = int(_time.monotonic() * 1000)
    try:
        async with _claude_client() as client:
            resp = await client.post(CLAUDE_API_URL, json=payload, headers=headers)
            ...
```

ВАЖНО: весь блок обработки ответа (`resp.raise_for_status()`, парсинг `data`, расчёт cost, вызов `_log_prompt`, `return content`) должен остаться ВНУТРИ `async with _claude_client() as client:`. Только `except`, `_log_prompt(error=...)` и `asyncio.sleep(delay)` — СНАРУЖИ.

Итоговая структура:
```python
for attempt in range(MAX_RETRIES):
    start_ms = int(_time.monotonic() * 1000)
    try:
        async with _claude_client() as client:
            resp = await client.post(CLAUDE_API_URL, json=payload, headers=headers)
            elapsed_ms = int(_time.monotonic() * 1000) - start_ms
            resp.raise_for_status()
            data = resp.json()
            content = data["content"][0]["text"]
            # ... расчёт usage, cost, _log_prompt, return content ...
            return content
    except Exception as exc:
        elapsed_ms = int(_time.monotonic() * 1000) - start_ms
        last_exc = exc
        # ... _log_prompt(error=...), _is_retryable проверка, sleep ...
```

### 2. Проверить _call_openai_fallback

Убедиться, что в `_call_openai_fallback()` клиент создаётся внутри функции (при каждом вызове — новый). Это уже должно быть так (`async with httpx.AsyncClient(...) as client:`), но проверить.

## Что НЕ менять

- Логику `_is_retryable()` — не трогать, она уже исправлена
- Таймауты — не трогать, уже раздельные
- `_log_prompt()`, `call_claude()` — не трогать
- Расчёт cost и usage — не трогать
- Секцию fallback на OpenAI — не трогать (там клиент уже создаётся при каждом вызове)

## Проверка

```bash
sudo systemctl restart k24-crm-worker
journalctl -u k24-crm-worker -f --no-pager | grep -E "claude_retry|claude_response|openai|fallback|200 OK"
```

Ожидание: если первая попытка упала — вторая должна пройти (свежее соединение), а не падать с той же ошибкой.
