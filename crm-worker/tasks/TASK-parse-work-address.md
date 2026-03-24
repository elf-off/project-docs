# TASK: Парсинг адреса места работы из текста объявления (v2)

**Дата:** 2026-03-19
**Приоритет:** Высокий
**Затрагиваемые файлы:** `services/vacancy_parser.py`, `services/vacancy_sync.py`

---

## Проблема

Адрес места работы (`address`) берётся из поля **{размещение}** Avito API. Это адрес размещения объявления, а не реальное место работы. Нужно брать адрес **только из текста объявления** (`raw_description`).

---

## Подход: Regexp + AI fallback, БЕЗ адреса из размещения

**Адрес из поля размещения Avito НЕ используем вообще.**

1. **Regexp** — ищем маркеры в тексте описания
2. **AI fallback** — если маркер не найден, извлекаем через LLM
3. **Если ничего не нашли** — поле `address` остаётся пустым (`None`), НЕ подставляем адрес размещения

---

## Что нужно сделать

### 1. Regexp-парсинг в `services/vacancy_parser.py`

**Ключевые фразы (регистронезависимо, в порядке приоритета):**
- `Адрес объекта:`
- `Адрес Места работы:`
- `Адрес места работы:`
- `Адрес:`

**Логика:**
1. Ищем вхождение фраз от более специфичной к менее
2. Берём текст после фразы до конца строки (`\n`)
3. Strip
4. Если непусто — возвращаем
5. Если пусто — `None`

```python
def parse_work_address_regexp(description: str) -> str | None:
    """Извлекает адрес места работы из текста по ключевым маркерам."""
```

### 2. AI-парсинг (fallback) в `services/vacancy_parser.py`

Вызывается только если regexp не нашёл.

```python
async def parse_work_address_ai(description: str) -> str | None:
    """Извлекает адрес места работы через AI (fallback)."""
```

**Промпт:**
```
Извлеки из текста вакансии адрес МЕСТА РАБОТЫ (не адрес размещения объявления).
Если адрес места работы указан — верни его.
Если не указан — верни null.

Текст вакансии:
{description}

Ответь ТОЛЬКО JSON: {"address": "..." или null}
```

- `temperature=0`, `max_tokens=150`
- Обязательно try/except — ошибка AI не ломает синхронизацию
- Обрезать description до ~2000 символов
- НЕ логировать в `ai_prompts_log`

### 3. Оркестратор

```python
async def parse_work_address(description: str) -> str | None:
    """Regexp -> AI fallback -> None. Без адреса размещения."""
    address = parse_work_address_regexp(description)
    if address:
        return address
    return await parse_work_address_ai(description)
```

### 4. Изменение в `services/vacancy_sync.py`

**Убрать** использование адреса из размещения для поля `address`. Заменить на:

```python
from services.vacancy_parser import parse_work_address

# Адрес ТОЛЬКО из описания, НЕ из размещения
parsed_address = await parse_work_address(raw_description)
vacancy.address = parsed_address  # может быть None — это ок
```

**Адрес из размещения Avito (`location`, `address` из API) — НЕ записывать в поле `address` вообще.**

Логирование:
```python
log.info("work_address_parsed", vacancy_id=..., address=parsed_address, source="regexp|ai|none")
```

### 5. Очистка базы и пересинхронизация

После деплоя выполнить:

```sql
-- Очистить адреса всех вакансий чтобы пересинхронизировать
UPDATE vacancies SET address = NULL, last_synced_at = NULL;
```

Сброс `last_synced_at = NULL` заставит `vacancy_sync` заново обработать все вакансии и спарсить адреса из описаний.

**Альтернативно** (если sync проверяет по другому условию):
```sql
-- Полная очистка и пересинхронизация
TRUNCATE TABLE vacancies;
```

После этого перезапустить сервис — синхронизация подхватит все вакансии заново через 30 минут (или вызвать вручную через admin API).

---

## Важные детали

- Поиск фраз — **регистронезависимый**
- Адрес — текст от маркера до `\n`
- Пустая строка после маркера — пропускаем, ищем следующий
- Специфичные фразы раньше общих
- **Адрес из размещения НЕ используется как fallback** — если ничего не найдено, `address = None`
- AI fallback в try/except
- После деплоя — очистить `address` и `last_synced_at` в таблице `vacancies`, перезапустить сервис

---

## Тестирование

**Пример 1 (regexp):**
```
Адрес объекта: г. Москва, ул. Ленина, д. 15
```
→ `г. Москва, ул. Ленина, д. 15` ✅

**Пример 2 (regexp):**
```
Адрес Места работы: Санкт-Петербург, Невский проспект, 100
```
→ `Санкт-Петербург, Невский проспект, 100` ✅

**Пример 3 (AI fallback):**
```
Ищем уборщицу в ТЦ "Мега Химки", Ленинградское шоссе, 5
```
→ AI: `Ленинградское шоссе, 5` ✅

**Пример 4 (нет адреса):**
```
Описание вакансии без адреса
```
→ `address = None` (НЕ подставляем размещение)

---

## Файлы для изменения

1. **`services/vacancy_parser.py`** — добавить `parse_work_address_regexp()`, `parse_work_address_ai()`, `parse_work_address()`
2. **`services/vacancy_sync.py`** — адрес только из `parse_work_address()`, убрать адрес из размещения

## После деплоя

1. Очистить адреса и сбросить синхронизацию:
```sql
UPDATE vacancies SET address = NULL, last_synced_at = NULL, embedding_indexed = FALSE;
```

2. Очистить коллекцию Qdrant (старые эмбеддинги содержат неправильные адреса):
```python
# Вариант А: пересоздать коллекцию (рекомендуется)
from qdrant_client import QdrantClient
client = QdrantClient(host="localhost", port=6333)
client.delete_collection("vacancies")
# При старте сервиса ensure_collection() создаст её заново

# Вариант Б: через API
# curl -X DELETE http://localhost:6333/collections/vacancies
```

3. Перезапустить сервис:
```bash
sudo systemctl restart k24-crm-worker
```

Сервис при старте:
- `ensure_collection()` — пересоздаст коллекцию Qdrant
- `vacancy_sync` (через 30 мин) — подхватит все вакансии, спарсит адреса из описаний, переиндексирует в Qdrant
