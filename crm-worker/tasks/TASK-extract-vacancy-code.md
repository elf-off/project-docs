# TASK: Извлечение vacancy_code из описания объявления

**Дата:** 2026-03-31
**Приоритет:** Высокий (нужен для связки CRM → портал откликов)
**Проект:** `/opt/openai/crm-worker/`

---

## КОНТЕКСТ

Каждое объявление на Avito содержит код вакансии в квадратных скобках в конце текста описания, например `[НК0101]`, `[ЯЛ2601]`, `[ЯЛ2303]`. Этот код нужен для привязки карточки кандидата к региону в портале Kadry 24 (через таблицу `vacancy_code_regions` в `2_kadry_4_maps`).

---

## 1. МИГРАЦИЯ БД

Создать файл `migrations/006_vacancy_code.sql`:

```sql
-- Добавить vacancy_code в vacancies
ALTER TABLE vacancies ADD COLUMN vacancy_code VARCHAR(20) DEFAULT NULL AFTER title;
ALTER TABLE vacancies ADD INDEX idx_vacancy_code (vacancy_code);

-- Добавить vacancy_code в handover_cards
ALTER TABLE handover_cards ADD COLUMN vacancy_code VARCHAR(20) DEFAULT NULL AFTER avito_vacancy_id;
ALTER TABLE handover_cards ADD INDEX idx_hc_vacancy_code (vacancy_code);

-- Заполнить vacancy_code из raw_description для существующих вакансий
UPDATE vacancies
SET vacancy_code = TRIM(BOTH ' ' FROM SUBSTRING(
    raw_description,
    LOCATE('[', raw_description, LENGTH(raw_description) - 30) + 1,
    LOCATE(']', raw_description, LENGTH(raw_description) - 30) - LOCATE('[', raw_description, LENGTH(raw_description) - 30) - 1
))
WHERE raw_description REGEXP '\\[[А-Яа-яЁё0-9]+\\]$'
  AND vacancy_code IS NULL;

-- Заполнить vacancy_code в существующих handover_cards из vacancies через avito_vacancy_id
UPDATE handover_cards hc
JOIN vacancies v ON v.avito_vacancy_id = hc.avito_vacancy_id
SET hc.vacancy_code = v.vacancy_code
WHERE v.vacancy_code IS NOT NULL
  AND hc.vacancy_code IS NULL;
```

**Выполнить на сервере:**
```bash
mysql -u root -p 2_kadry_4_crm_avito < /opt/openai/crm-worker/migrations/006_vacancy_code.sql
```

---

## 2. МОДЕЛЬ БД

### Файл: `models/db.py`

Добавить поле в модель `Vacancy`:
```python
vacancy_code = Column(String(20), nullable=True, index=True)
```

Добавить поле в модель `HandoverCard`:
```python
vacancy_code = Column(String(20), nullable=True, index=True)
```

---

## 3. ПАРСИНГ VACANCY_CODE ПРИ СИНКЕ ВАКАНСИЙ

### Файл: `services/avito_vacancies.py` (или где происходит парсинг/сохранение вакансии)

Добавить функцию извлечения кода:

```python
import re

def extract_vacancy_code(description: str) -> str | None:
    """Извлекает код вакансии из квадратных скобок в конце описания, например [НК0101]"""
    if not description:
        return None
    match = re.search(r'\[([А-ЯЁа-яё0-9]+)\]\s*$', description)
    return match.group(1) if match else None
```

В месте где вакансия сохраняется/обновляется в БД — вызвать эту функцию и записать результат в `vacancy_code`:

```python
vacancy_code = extract_vacancy_code(raw_description)
# При INSERT/UPDATE вакансии добавить: vacancy_code=vacancy_code
```

Найти нужное место:
```bash
grep -rn "raw_description\|INSERT.*vacancies\|vacancy.*save\|vacancy.*update\|vacancy.*create" /opt/openai/crm-worker/services/ --include="*.py" | head -20
```

---

## 4. КОПИРОВАНИЕ В HANDOVER_CARD

### Файл: `services/handover.py` (или где создаётся handover_card)

При создании handover_card — подтянуть vacancy_code из вакансии:

```python
# Найти vacancy_code по avito_vacancy_id
vacancy = await get_vacancy_by_avito_id(avito_vacancy_id)
vacancy_code = vacancy.vacancy_code if vacancy else None

# Добавить в INSERT handover_cards
# ..., vacancy_code=vacancy_code, ...
```

Найти нужное место:
```bash
grep -rn "handover_card\|HandoverCard\|INSERT.*handover" /opt/openai/crm-worker/services/ --include="*.py" | head -20
```

---

## 5. ПРОВЕРКА

```bash
# 1. Проверить что vacancy_code заполнился для существующих вакансий:
mysql -u root -p -e "SELECT avito_vacancy_id, vacancy_code, title FROM 2_kadry_4_crm_avito.vacancies WHERE vacancy_code IS NOT NULL LIMIT 10;"

# 2. Проверить что handover_cards тоже заполнились:
mysql -u root -p -e "SELECT id, vacancy_code, vacancy_title, candidate_name FROM 2_kadry_4_crm_avito.handover_cards WHERE vacancy_code IS NOT NULL LIMIT 10;"

# 3. Проверить что все вакансии с описанием получили код:
mysql -u root -p -e "SELECT COUNT(*) as total, SUM(vacancy_code IS NOT NULL) as with_code, SUM(vacancy_code IS NULL) as without_code FROM 2_kadry_4_crm_avito.vacancies WHERE is_active=1;"

# 4. Перезапустить сервис:
sudo systemctl restart k24-crm-worker
```

---

## ⚠️ ВАЖНО

- Regex `\[([А-ЯЁа-яё0-9]+)\]\s*$` — ищет квадратные скобки с кириллицей/цифрами **в конце** текста
- Не ломать существующий vacancy sync (hash-based deduplication) — vacancy_code добавляется параллельно
- Если описание не содержит кода — `vacancy_code = NULL`, это нормально
