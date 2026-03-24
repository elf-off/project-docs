# TASK: Убрать SOCKS5-прокси из ai_claude.py

## Причина

Маршрутизация трафика к api.anthropic.com и api.openai.com теперь настроена на уровне MikroTik-роутера (policy routing через VPN-туннель). SOCKS5-прокси (microsocks) больше не нужен — трафик идёт напрямую через роутер.

## Файл
`/opt/openai/crm-worker/services/ai_claude.py`

## Что сделать

### 1. В `_claude_client()` убрать proxy

**Было:**
```python
def _claude_client() -> httpx.AsyncClient:
    proxy_url = settings.claude_proxy
    return httpx.AsyncClient(
        proxy=proxy_url,
        timeout=httpx.Timeout(...),
        limits=httpx.Limits(...),
        http1=True,
        http2=False,
    )
```

**Стало:**
```python
def _claude_client() -> httpx.AsyncClient:
    return httpx.AsyncClient(
        timeout=httpx.Timeout(
            connect=CONNECT_TIMEOUT,
            read=READ_TIMEOUT,
            write=WRITE_TIMEOUT,
            pool=CONNECT_TIMEOUT,
        ),
    )
```

Убрать: `proxy`, `limits`, `http1`, `http2` — они были нужны для обхода проблем с SOCKS5.

### 2. В `_call_openai_fallback()` убрать proxy

**Было:**
```python
async with httpx.AsyncClient(proxy=settings.claude_proxy, timeout=timeout, limits=..., http1=True, http2=False) as client:
```

**Стало:**
```python
async with httpx.AsyncClient(timeout=timeout) as client:
```

### 3. Убрать импорт/использование `settings.claude_proxy`

Проверить что нигде в файле не осталось обращений к `settings.claude_proxy`.

## Что НЕ менять

- Retry-логику (5 попыток Claude + 3 попытки OpenAI) — оставить
- `_is_retryable()` и `_RETRYABLE_EXCEPTIONS` — оставить
- Таймауты (CONNECT_TIMEOUT, READ_TIMEOUT, WRITE_TIMEOUT) — оставить
- `_log_prompt()`, `call_claude()`, `ask_claude()` — логику не трогать
- Расчёт cost и usage — не трогать
- Формат ошибок `f"{type(exc).__name__}: {exc!s}"` — оставить

## Также проверить

Файл `/opt/openai/crm-worker/services/ai_rag.py` — там тоже может быть прокси для OpenAI embeddings:
```bash
grep -n "proxy\|claude_proxy" /opt/openai/crm-worker/services/ai_rag.py
```
Если есть — тоже убрать прокси, оставить обычный httpx.AsyncClient.

## Проверка

```bash
sudo systemctl restart k24-crm-worker
journalctl -u k24-crm-worker -f --no-pager | grep -E "claude_retry|claude_response|vacancy_parse|disconnected"
```

Ожидание: запросы к Claude и OpenAI проходят без ошибок "Server disconnected".
