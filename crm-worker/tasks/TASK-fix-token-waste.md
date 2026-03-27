# ЗАДАЧА: Оптимизация расхода токенов при синке вакансий

## Контекст

Проект: `/opt/openai/crm-worker/`
БД: `2_kadry_4_crm_avito`

**Проблема:** Синк вакансий каждые 30 минут вызывает Claude для парсинга описания КАЖДОЙ вакансии, даже если текст не изменился. Результат — 5000+ вызовов/день, $11+/день на парсинг неизменённых вакансий. Фактические диалоги с кандидатами стоят всего $0.30/день.

**Цель:** Снизить расходы с $11/день до $0.30-0.50/день.

**ВАЖНО:** Перед изменением любого файла — сначала `cat` его полное содержимое!

---

## 1. Добавить колонку `description_hash` в таблицу `vacancies`

```sql
ALTER TABLE vacancies 
    ADD COLUMN description_hash VARCHAR(64) DEFAULT NULL 
    COMMENT 'SHA256 от raw_description для skip-перепарсинга';
```

---

## 2. Изменить `services/vacancy_sync.py` — не парсить неизменённые

Текущая логика в `ensure_vacancy()` (или `refresh_active_vacancies`):
1. Загружает вакансию из Avito API
2. Всегда вызывает `parse_vacancy_description()` через Claude
3. Сохраняет результат

**Новая логика:**

```python
import hashlib

def _compute_hash(text: str) -> str:
    """SHA256 от текста описания."""
    return hashlib.sha256((text or "").encode()).hexdigest()
```

В `ensure_vacancy()` (или аналогичной функции, которая обновляет вакансию):

1. Загрузить вакансию из Avito API
2. Вычислить `new_hash = _compute_hash(description)`
3. Загрузить текущую запись из БД: `SELECT description_hash FROM vacancies WHERE avito_vacancy_id = ...`
4. **Если `new_hash == existing_hash` → ПРОПУСТИТЬ парсинг.** Обновить только поля из API (title, salary и т.д.), НЕ вызывать Claude.
5. **Если хеш изменился или вакансия новая** → вызвать Claude для парсинга → сохранить `description_hash = new_hash`

Добавить лог:
```python
if new_hash == existing_hash:
    logger.debug("vacancy_skip_parse", vacancy_id=vid, reason="description_unchanged")
else:
    logger.info("vacancy_parse_needed", vacancy_id=vid, reason="new_or_changed")
    # ... вызов Claude
```

---

## 3. Использовать Haiku вместо Sonnet для парсинга вакансий

В `config.py` добавить:
```python
claude_parse_model: str = "claude-haiku-4-5-20251001"
```

В `services/vacancy_parser.py` (или где вызывается `call_claude` / `ask_claude` для парсинга):

Сейчас парсинг вакансий вызывает Claude через `call_claude()` который использует `settings.claude_model` (Sonnet).

Нужно передавать модель явно. Посмотри как устроен `call_claude` / `ask_claude` в `services/ai_claude.py`:

- Если функция принимает параметр `model` — передай `settings.claude_parse_model`
- Если НЕ принимает — **добавь** опциональный параметр `model: str = None` в `ask_claude()`:

```python
async def ask_claude(
    system: str,
    messages: list,
    max_tokens: int = 1000,
    temperature: float = 0.3,
    model: str = None,  # ← добавить
    session_id=None,
    application_id=None,
    dialog_stage=None,
) -> str:
    payload = {
        "model": model or settings.claude_model,  # ← использовать
        ...
    }
```

Аналогично в `call_claude()` — пробросить `model` в `ask_claude`.

Затем в `vacancy_parser.py`:
```python
response = await call_claude(
    system="Ты парсер вакансий. Отвечай ТОЛЬКО валидным JSON.",
    user_message=prompt,
    max_tokens=500,
    model=settings.claude_parse_model,  # ← Haiku вместо Sonnet
)
```

**ВАЖНО:** НЕ менять модель для диалогов (ai_agent.py). Только для парсинга вакансий.

---

## 4. Ограничить retry при сетевых ошибках

В `services/ai_claude.py` в функции `ask_claude()` есть retry-логика (до 5 попыток).

Добавить **circuit breaker** — если 3 ошибки подряд за последние 5 минут, прекратить попытки до следующего цикла:

```python
import time

_last_errors: list[float] = []
_CIRCUIT_BREAK_WINDOW = 300  # 5 минут
_CIRCUIT_BREAK_THRESHOLD = 3

def _check_circuit_breaker() -> bool:
    """True если circuit breaker сработал (слишком много ошибок)."""
    now = time.time()
    # Убрать старые ошибки
    _last_errors[:] = [t for t in _last_errors if now - t < _CIRCUIT_BREAK_WINDOW]
    return len(_last_errors) >= _CIRCUIT_BREAK_THRESHOLD

def _record_error():
    _last_errors.append(time.time())
```

В `ask_claude()`, перед retry-циклом:
```python
if _check_circuit_breaker():
    log.warning("circuit_breaker_open", errors_in_window=len(_last_errors))
    raise Exception("Circuit breaker open — too many API errors in last 5 minutes")
```

При каждой ошибке:
```python
except Exception as e:
    _record_error()
    last_exc = e
    ...
```

---

## 5. Добавить лог расходов в `refresh_active_vacancies`

В конце функции `refresh_active_vacancies` выводить итог:

```python
logger.info(
    "refresh_complete",
    total=len(ids),
    parsed=parsed_count,    # сколько реально вызвали Claude
    skipped=skipped_count,  # сколько пропустили (хеш совпал)
    errors=error_count,
)
```

Для этого `ensure_vacancy` (или эквивалент) должен возвращать статус: "parsed" / "skipped" / "error".

---

## 6. Заполнить hash для существующих вакансий

После деплоя — одноразовый скрипт, чтобы заполнить `description_hash` для всех вакансий, у которых он NULL:

```python
import hashlib
import asyncio
from sqlalchemy import select, update
from models.db import Vacancy, AsyncSessionFactory

async def backfill_hashes():
    async with AsyncSessionFactory() as session:
        result = await session.execute(
            select(Vacancy).where(Vacancy.description_hash == None)
        )
        vacancies = result.scalars().all()
        for v in vacancies:
            h = hashlib.sha256((v.raw_description or "").encode()).hexdigest()
            v.description_hash = h
        await session.commit()
        print(f"Updated {len(vacancies)} vacancies with hashes")

asyncio.run(backfill_hashes())
```

Выполнить один раз после деплоя:
```bash
cd /opt/openai/crm-worker
source venv/bin/activate
python3 -c "
import hashlib, asyncio
from sqlalchemy import select
from models.db import Vacancy, AsyncSessionFactory

async def backfill():
    async with AsyncSessionFactory() as session:
        result = await session.execute(select(Vacancy).where(Vacancy.description_hash == None))
        vacancies = result.scalars().all()
        for v in vacancies:
            v.description_hash = hashlib.sha256((v.raw_description or '').encode()).hexdigest()
        await session.commit()
        print(f'Updated {len(vacancies)} hashes')

asyncio.run(backfill())
"
```

---

## 7. Модель Vacancy — добавить поле

В `models/db.py` добавить поле в модель `Vacancy`:

```python
description_hash = Column(String(64), default=None, comment="SHA256 hash of raw_description")
```

---

## Порядок выполнения

1. `cat` все файлы, которые нужно менять: `models/db.py`, `services/vacancy_sync.py`, `services/vacancy_parser.py`, `services/ai_claude.py`, `config.py`
2. ALTER TABLE (пункт 1)
3. Модель db.py (пункт 7)
4. config.py — `claude_parse_model` (пункт 3)
5. ai_claude.py — параметр `model` + circuit breaker (пункты 3, 4)
6. vacancy_sync.py — хеш-проверка + счётчики (пункты 2, 5)
7. vacancy_parser.py — передать `model=settings.claude_parse_model` (пункт 3)
8. Перезапуск: `sudo systemctl restart k24-crm-worker`
9. Backfill хешей (пункт 6)
10. Проверка: `journalctl -u k24-crm-worker -f` — убедиться что при следующем синке все вакансии `vacancy_skip_parse`

---

## Проверка результата

После деплоя подождать один цикл синка (30 мин) и выполнить:

```sql
SELECT 
    DATE(created_at) day,
    dialog_stage,
    COUNT(*) calls,
    ROUND(SUM(cost_usd), 4) cost
FROM ai_prompts_log
WHERE created_at >= NOW() - INTERVAL 1 HOUR
GROUP BY day, dialog_stage;
```

Ожидание: 0 вызовов с `dialog_stage IS NULL` (если ни одна вакансия не изменилась).

---

## Чего НЕ делать

- НЕ менять модель для диалогов (ai_agent.py должен использовать Sonnet как раньше)
- НЕ менять логику диалога, промпты, стадии
- НЕ трогать webhooks.py, admin.py, admin_web.py
- НЕ менять интервал синка (оставить 30 мин)
