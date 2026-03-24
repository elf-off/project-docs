# ЗАДАЧА: Увеличить retry + fallback на OpenAI GPT-4o

## Проблема

При ошибке 529 (API перегружен) Claude API отказывает после 2 попыток. Сессия помечается `failed`, кандидат остаётся без ответа. В боевом режиме это недопустимо.

## Решение

Двухуровневая отказоустойчивость:
1. **Retry Claude** — до 5 попыток с экспоненциальной паузой
2. **Fallback на OpenAI GPT-4o** — если Claude так и не ответил

---

## Что менять

### Файл: `services/ai_claude.py`

Сначала прочитай текущий файл целиком (`cat services/ai_claude.py`).

#### Изменения в retry-логике:

```python
MAX_RETRIES = 5  # было 2
RETRY_DELAYS = [2, 5, 15, 30, 60]  # экспоненциальная пауза в секундах

# Коды для retry (не все ошибки стоит повторять)
RETRYABLE_STATUS_CODES = {429, 500, 502, 503, 529}
```

При каждой попытке:
- Проверить что status_code в RETRYABLE_STATUS_CODES
- Подождать RETRY_DELAYS[attempt] секунд
- Залогировать: `claude_retry attempt=N delay=Xs error=...`
- Если все 5 попыток провалились → перейти к fallback

#### Добавить fallback на OpenAI:

```python
OPENAI_FALLBACK_MODEL = "gpt-4o"

async def call_claude(system: str, user_message: str, max_tokens: int = 1024, temperature: float = 0.7, **kwargs) -> str:
    """
    Основная функция вызова LLM.
    1. Пробует Claude (до 5 попыток)
    2. Если Claude недоступен → fallback на OpenAI GPT-4o
    """
    
    # --- Попытки Claude ---
    last_error = None
    for attempt in range(MAX_RETRIES):
        try:
            response = await _call_claude_api(system, user_message, max_tokens, temperature, **kwargs)
            return response
        except Exception as e:
            last_error = e
            status_code = _extract_status_code(e)
            
            if status_code not in RETRYABLE_STATUS_CODES:
                # Не retryable ошибка (400, 401 и т.д.) — сразу fallback
                log.warning("claude_non_retryable", error=str(e), status_code=status_code)
                break
            
            if attempt < MAX_RETRIES - 1:
                delay = RETRY_DELAYS[attempt]
                log.warning("claude_retry", attempt=attempt, delay=delay, error=str(e))
                await asyncio.sleep(delay)
    
    # --- Fallback на OpenAI ---
    log.warning("claude_fallback_to_openai", claude_error=str(last_error))
    try:
        response = await _call_openai_fallback(system, user_message, max_tokens, temperature)
        log.info("openai_fallback_success")
        return response
    except Exception as e:
        log.error("openai_fallback_failed", error=str(e))
        raise last_error  # Бросаем оригинальную ошибку Claude
```

#### Функция OpenAI fallback:

```python
async def _call_openai_fallback(system: str, user_message: str, max_tokens: int, temperature: float) -> str:
    """Fallback вызов через OpenAI API."""
    import httpx
    
    # OpenAI API key уже должен быть в config (используется для embeddings)
    api_key = settings.openai_api_key
    if not api_key:
        raise ValueError("OPENAI_API_KEY not configured, fallback unavailable")
    
    async with httpx.AsyncClient(timeout=60.0) as client:
        resp = await client.post(
            "https://api.openai.com/v1/chat/completions",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
            },
            json={
                "model": OPENAI_FALLBACK_MODEL,
                "max_tokens": max_tokens,
                "temperature": temperature,
                "messages": [
                    {"role": "system", "content": system},
                    {"role": "user", "content": user_message},
                ],
            },
        )
        resp.raise_for_status()
        data = resp.json()
        return data["choices"][0]["message"]["content"]
```

**ВАЖНО про OpenAI запросы:**
- OpenAI API НЕ заблокирован в РФ, но на всякий случай проверь — если нужен прокси, используй тот же SOCKS5 что и для Claude
- `openai_api_key` уже должен быть в config.py (используется для embeddings в ai_rag.py). Найди его: `grep -n "openai" config.py`
- Если ключ хранится под другим именем — используй то же имя

#### Вспомогательная функция:

```python
def _extract_status_code(error: Exception) -> int | None:
    """Извлечь HTTP status code из ошибки httpx."""
    error_str = str(error)
    for code in [429, 500, 502, 503, 529, 400, 401, 403]:
        if str(code) in error_str:
            return code
    if hasattr(error, 'response') and hasattr(error.response, 'status_code'):
        return error.response.status_code
    return None
```

---

### Файл: `services/ai_agent.py`

**НЕ помечать сессию failed при ошибке LLM**, если fallback тоже не сработал. Вместо этого — оставить сессию в текущем stage для повторной обработки.

Найди все места где ловится ошибка от Claude и ставится `status = "failed"` или `dialog_stage = "failed"`. Замени логику:

```python
# БЫЛО:
except Exception as e:
    ai_session.dialog_stage = "failed"
    ai_session.status = "failed"
    log.error("...", error=str(e))

# СТАЛО:
except Exception as e:
    # Не помечаем failed — оставляем в текущем stage для повторной обработки
    log.error("...", error=str(e))
    # Логируем событие
    try:
        from utils.event_logger import log_event
        await log_event(
            application.account_id if hasattr(application, 'account_id') else None,
            "session_error",
            f"Ошибка AI сессии {ai_session.id}: {str(e)[:200]}"
        )
    except Exception:
        pass
```

**Исключение:** Если ошибка НЕ связана с LLM (например, ошибка БД или парсинга) — по-прежнему ставить `failed`. Определять по типу ошибки:

```python
# Ошибки LLM (retry/fallback не помог) — НЕ ставить failed
LLM_ERROR_MARKERS = ["529", "502", "503", "429", "500", "anthropic", "openai", "timeout"]

def _is_llm_error(error: Exception) -> bool:
    error_str = str(error).lower()
    return any(marker in error_str for marker in LLM_ERROR_MARKERS)
```

---

### Файл: `config.py`

Убедись что есть:
```python
openai_api_key: str = ""  # или как он уже называется
openai_fallback_model: str = "gpt-4o"
claude_max_retries: int = 5
```

Если `openai_api_key` уже есть под другим именем — НЕ дублируй, используй существующий.

---

### Логирование

Добавь вызовы `log_event()` для мониторинга fallback:

| Событие | event_type | message |
|---------|-----------|---------|
| Claude retry | `claude_retry` | "Claude retry attempt {N}, delay {X}s" |
| Fallback на OpenAI | `claude_fallback` | "Claude недоступен, переключение на OpenAI GPT-4o" |
| OpenAI успех | `openai_fallback_ok` | "OpenAI fallback успешен для сессии {id}" |
| OpenAI тоже упал | `openai_fallback_fail` | "OpenAI fallback тоже не удался: {error}" |

---

## Порядок работы

1. Прочитай `services/ai_claude.py` — пойми текущую структуру
2. Прочитай `config.py` — найди openai_api_key
3. Обнови retry-логику в `ai_claude.py`
4. Добавь fallback-функцию
5. Обнови обработку ошибок в `ai_agent.py`
6. Проверь импорты

---

## ПОСЛЕ ВЫПОЛНЕНИЯ — ОБЯЗАТЕЛЬНЫЙ ОТЧЁТ

Создай `CHANGELOG-retry-fallback.md` в корне проекта:

1. **Таблица изменений** — каждый файл, что изменено
2. **Как проверить:**
```bash
# Должен работать
python -c "from services.ai_claude import call_claude; print('OK')"
```
3. **Логика retry/fallback:**
   - Перечислить: сколько попыток, какие паузы, при каких кодах
   - Когда включается fallback
   - Что происходит если и fallback не сработал
