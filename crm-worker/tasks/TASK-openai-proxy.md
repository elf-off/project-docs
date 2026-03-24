# ЗАДАЧА: Добавить прокси в OpenAI fallback

## Проблема

В `services/ai_claude.py` функция `_call_openai_fallback()` создаёт httpx.AsyncClient БЕЗ прокси. OpenAI API заблокирован на сервере → 403 Forbidden. Fallback не работает.

Claude API уже использует SOCKS5-прокси через `settings.claude_proxy` — нужно сделать то же самое для OpenAI.

## Решение

**Файл: `services/ai_claude.py`**

Найди функцию `_call_openai_fallback()`. В ней строка:

```python
async with httpx.AsyncClient(timeout=60.0) as client:
```

Замени на:

```python
async with httpx.AsyncClient(timeout=60.0, proxy=settings.claude_proxy) as client:
```

Больше ничего не меняй.

## Проверка

```bash
grep -n "proxy" services/ai_claude.py
```

Должно показать прокси и в `_claude_client()`, и в `_call_openai_fallback()`.
