# TASK: Исправить retry и таймауты в ai_claude.py

## Файл
`/opt/openai/crm-worker/services/ai_claude.py`

## Проблема

В логах постоянные ошибки:
```
Claude недоступен, переключение на OpenAI GPT-4o: Server disconnected without sending a response.
OpenAI fallback тоже не удался: Server disconnected without sending a response.
```

Обе ошибки повторяются каждые ~15 секунд. Причины:

1. **`_is_retryable()` не распознаёт сетевые ошибки.** Она проверяет только HTTP-статусы (429, 500...), но "Server disconnected without sending a response" — это `httpx.RemoteProtocolError` (сетевая ошибка, не HTTP-ответ). Результат: `_is_retryable` возвращает `False` → `break` → **всего 1 попытка Claude** вместо 5, затем сразу fallback.

2. **Таймаут `timeout=60` — единый для connect+read.** Для LLM-генерации 60 секунд на read может быть мало (длинные промпты, медленная генерация). Нужны раздельные таймауты: connect=15с, read=120с.

3. **Нет retry для OpenAI fallback** — одна попытка, и если не удалась — всё, ошибка.

## Что сделать

### 1. Исправить `_is_retryable()` — добавить сетевые ошибки

Заменить текущую функцию на:

```python
# Ошибки сети, при которых стоит повторить
_RETRYABLE_EXCEPTIONS = (
    httpx.ConnectError,
    httpx.ConnectTimeout,
    httpx.ReadTimeout,
    httpx.WriteTimeout,
    httpx.RemoteProtocolError,
    httpx.CloseError,
    ConnectionError,
    ConnectionResetError,
    ConnectionAbortedError,
    OSError,
)

def _is_retryable(exc: Exception) -> bool:
    """Проверяет, стоит ли повторять запрос при данной ошибке."""
    # Сетевые ошибки — всегда повторяем
    if isinstance(exc, _RETRYABLE_EXCEPTIONS):
        return True

    # Ключевые фразы в тексте ошибки
    error_str = str(exc).lower()
    retryable_phrases = [
        "disconnected", "connection reset", "connection closed",
        "connection aborted", "broken pipe", "timed out", "timeout",
        "eof occurred", "incomplete read", "server disconnected",
    ]
    for phrase in retryable_phrases:
        if phrase in error_str:
            return True

    # HTTP-статусы (429, 500, 502, 503, 529)
    if hasattr(exc, 'response') and hasattr(exc.response, 'status_code'):
        return exc.response.status_code in RETRYABLE_STATUS_CODES
    for code in RETRYABLE_STATUS_CODES:
        if str(code) in str(exc):
            return True

    return False
```

### 2. Раздельные таймауты

Заменить `timeout=60` на раздельные значения. Добавить константы:

```python
CONNECT_TIMEOUT = 15.0   # подключение к серверу
READ_TIMEOUT = 120.0     # ожидание ответа LLM
WRITE_TIMEOUT = 30.0     # отправка запроса
```

В `_claude_client()` заменить:
```python
# Было:
timeout=60,

# Стало:
timeout=httpx.Timeout(
    connect=CONNECT_TIMEOUT,
    read=READ_TIMEOUT,
    write=WRITE_TIMEOUT,
    pool=CONNECT_TIMEOUT,
),
```

### 3. OpenAI fallback — всегда через прокси, с раздельными таймаутами

Заменить `_call_openai_fallback()`:

```python
async def _call_openai_fallback(
    system: str,
    messages: list[dict],
    max_tokens: int,
    temperature: float,
) -> str:
    """Fallback через OpenAI GPT-4o (через SOCKS5-прокси)."""
    api_key = settings.openai_api_key
    if not api_key:
        raise ValueError("OPENAI_API_KEY not configured, fallback unavailable")

    openai_messages = [{"role": "system", "content": system}] + messages
    request_json = {
        "model": OPENAI_FALLBACK_MODEL,
        "max_tokens": max_tokens,
        "temperature": temperature,
        "messages": openai_messages,
    }
    request_headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
    }
    timeout = httpx.Timeout(
        connect=CONNECT_TIMEOUT,
        read=READ_TIMEOUT,
        write=WRITE_TIMEOUT,
        pool=CONNECT_TIMEOUT,
    )

    async with httpx.AsyncClient(proxy=settings.claude_proxy, timeout=timeout) as client:
        resp = await client.post(OPENAI_API_URL, headers=request_headers, json=request_json)
        resp.raise_for_status()
        data = resp.json()
        return data["choices"][0]["message"]["content"]
```

### 4. Retry для OpenAI fallback в `ask_claude()`

В секции "Fallback на OpenAI" добавить 3 попытки с паузами:

```python
    # --- Fallback на OpenAI ---
    log.warning("claude_fallback_to_openai", claude_error=str(last_exc)[:200])
    try:
        from utils.event_logger import log_event
        await log_event(None, "claude_fallback", f"Claude недоступен, переключение на OpenAI GPT-4o: {str(last_exc)[:150]}")
    except Exception:
        pass

    openai_retries = 3
    openai_delays = [3, 10, 30]
    for oai_attempt in range(openai_retries):
        try:
            start_ms = int(_time.monotonic() * 1000)
            content = await _call_openai_fallback(system, messages, max_tokens, temperature)
            elapsed_ms = int(_time.monotonic() * 1000) - start_ms

            log.info("openai_fallback_success", ms=elapsed_ms, length=len(content), attempt=oai_attempt + 1)
            await _log_prompt(
                session_id=session_id,
                application_id=application_id,
                dialog_stage=f"{dialog_stage}_fallback" if dialog_stage else "fallback",
                response_ms=elapsed_ms,
            )
            try:
                from utils.event_logger import log_event
                await log_event(None, "openai_fallback_ok", f"OpenAI fallback успешен, {elapsed_ms}ms")
            except Exception:
                pass
            return content

        except Exception as fallback_exc:
            log.warning("openai_fallback_attempt_failed", attempt=oai_attempt + 1, error=str(fallback_exc)[:150])
            if oai_attempt < openai_retries - 1:
                await asyncio.sleep(openai_delays[oai_attempt])
            else:
                log.error("openai_fallback_exhausted", attempts=openai_retries, error=str(fallback_exc)[:150])
                try:
                    from utils.event_logger import log_event
                    await log_event(None, "openai_fallback_fail", f"OpenAI fallback тоже не удался после {openai_retries} попыток: {str(fallback_exc)[:100]}")
                except Exception:
                    pass
                raise last_exc or fallback_exc
```

## Что НЕ менять

- Функцию `call_claude()` (упрощённая обёртка) — не трогать
- Функцию `_log_prompt()` — не трогать
- Константы стоимости `_COST_INPUT_PER_M`, `_COST_OUTPUT_PER_M` — не трогать
- Логику расчёта cost в `ask_claude()` — не трогать
- Импорты `from utils.event_logger import log_event` внутри try/except — оставить как есть (ленивый импорт)

## Проверка

После применения:
```bash
sudo systemctl restart k24-crm-worker
journalctl -u k24-crm-worker -f --no-pager | grep -E "claude_retry|openai|fallback|disconnected"
```

Ожидаемое поведение:
- При "Server disconnected" → retry (до 5 попыток Claude с паузами 2, 5, 15, 30, 60 сек)
- Если все 5 не удались → OpenAI через прокси (до 3 попыток с паузами 3, 10, 30 сек)
- Если и OpenAI не удался → ошибка

## Итог изменений

| Что | Было | Стало |
|-----|------|-------|
| Retry при "Server disconnected" | 1 попытка (break) | 5 попыток с паузами |
| Таймаут read | 60с (общий) | 120с (раздельный) |
| Retry OpenAI | 1 попытка | 3 попытки с паузами |
| Таймауты OpenAI | 60с (общий) | 120с read (раздельный) |
