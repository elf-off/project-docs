# Deploy: Парсинг адреса места работы из текста объявления (v2)

**Дата:** 2026-03-19

## Что сделано

Адрес места работы теперь берется **только из текста объявления**, а не из поля размещения Avito API. Двухуровневый парсинг: regexp + AI fallback. Если адрес не найден -- поле остается пустым (NULL).

### Изменения

| Файл | Что изменено |
|------|-------------|
| `services/vacancy_parser.py` | 3 функции: `parse_work_address_regexp()`, `parse_work_address_ai()`, `parse_work_address()` |
| `services/vacancy_sync.py` | Убран адрес из размещения в `map_avito_to_db`, адрес только из `parse_work_address()` |

### Стратегия парсинга

1. **Regexp** (быстро, бесплатно) -- ищет маркеры: "Адрес объекта:", "Адрес места работы:", "Адрес:"
2. **AI fallback** -- Claude с `temperature=0`, обрезка до 2000 символов, try/except
3. **Если ничего не найдено** -- `address = NULL` (адрес размещения НЕ подставляется)

## Порядок деплоя

### 1. Обновить код

```bash
cd /opt/crm-worker
git pull
```

### 2. Очистить адреса, сбросить синхронизацию и индексацию

```sql
UPDATE vacancies SET address = NULL, last_synced_at = NULL, embedding_indexed = FALSE;
```

### 3. Очистить коллекцию Qdrant (старые эмбеддинги содержат неправильные адреса)

```bash
curl -X DELETE http://localhost:6333/collections/vacancies
```

При старте сервиса `ensure_collection()` создаст коллекцию заново.

### 4. Перезапустить сервис

```bash
sudo systemctl restart k24-crm-worker
```

Сервис при старте:
- `ensure_collection()` -- пересоздаст коллекцию Qdrant
- `vacancy_sync` (через 30 мин) -- подхватит все вакансии, спарсит адреса из описаний, переиндексирует в Qdrant

## Проверка

1. Дождаться синка вакансий (до 30 мин)
2. В логах: `work_address_parsed_regexp` или `work_address_parsed_ai`
3. Проверить в админке: вакансии с "Адрес объекта:" в описании должны иметь корректный адрес
4. Вакансии без адреса в описании должны иметь `address = NULL`
5. `SELECT title, address FROM vacancies WHERE address IS NOT NULL LIMIT 10;`
6. Проверить что Qdrant заново проиндексировал: `SELECT COUNT(*) FROM vacancies WHERE embedding_indexed = TRUE;`
