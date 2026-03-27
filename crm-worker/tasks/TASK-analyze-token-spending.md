# ЗАДАЧА: Анализ расходов токенов Claude за 25-27 марта

## Контекст

Проект: `/opt/openai/crm-worker/`
БД: `2_kadry_4_crm_avito` (MySQL/MariaDB)
Таблица логов: `ai_prompts_log`

Наблюдается перерасход — $10 за вчера при 6 активных диалогах. Нужно найти, куда уходят токены.

**ВАЖНО:** Это задача ТОЛЬКО на анализ. Ничего не менять, не создавать, не удалять. Только SELECT-запросы и вывод результатов.

---

## Запросы для выполнения

### 1. Общие расходы по дням

```sql
SELECT 
    DATE(created_at) as day,
    COUNT(*) as calls,
    SUM(prompt_tokens) as input_tokens,
    SUM(completion_tokens) as output_tokens,
    SUM(total_tokens) as total_tokens,
    ROUND(SUM(cost_usd), 4) as total_cost_usd
FROM ai_prompts_log
WHERE created_at >= '2026-03-25 00:00:00'
GROUP BY DATE(created_at)
ORDER BY day;
```

### 2. Расходы по dialog_stage за каждый день

```sql
SELECT 
    DATE(created_at) as day,
    dialog_stage,
    COUNT(*) as calls,
    SUM(prompt_tokens) as input_tokens,
    SUM(completion_tokens) as output_tokens,
    ROUND(SUM(cost_usd), 4) as cost_usd,
    ROUND(AVG(prompt_tokens)) as avg_input,
    ROUND(AVG(completion_tokens)) as avg_output
FROM ai_prompts_log
WHERE created_at >= '2026-03-25 00:00:00'
GROUP BY DATE(created_at), dialog_stage
ORDER BY day, cost_usd DESC;
```

### 3. Расходы по аккаунтам (через application → avito_account)

```sql
SELECT 
    DATE(p.created_at) as day,
    COALESCE(aa.account_name, CONCAT('account_', a.account_id)) as account,
    COUNT(*) as calls,
    ROUND(SUM(p.cost_usd), 4) as cost_usd
FROM ai_prompts_log p
LEFT JOIN applications a ON a.id = p.application_id
LEFT JOIN avito_accounts aa ON aa.id = a.account_id
WHERE p.created_at >= '2026-03-25 00:00:00'
GROUP BY day, account
ORDER BY day, cost_usd DESC;
```

### 4. Топ-10 самых дорогих вызовов за вчера

```sql
SELECT 
    p.id,
    p.dialog_stage,
    p.prompt_tokens,
    p.completion_tokens,
    p.total_tokens,
    p.cost_usd,
    p.response_ms,
    p.created_at,
    p.application_id,
    p.ai_session_id
FROM ai_prompts_log p
WHERE p.created_at >= '2026-03-26 00:00:00'
ORDER BY p.cost_usd DESC
LIMIT 10;
```

### 5. Вызовы синка вакансий (dialog_stage вероятно NULL или 'vacancy_parse' или подобное)

```sql
SELECT 
    DATE(created_at) as day,
    dialog_stage,
    COUNT(*) as calls,
    ROUND(SUM(cost_usd), 4) as cost_usd
FROM ai_prompts_log
WHERE created_at >= '2026-03-25 00:00:00'
  AND (dialog_stage IS NULL 
       OR dialog_stage LIKE '%vacanc%' 
       OR dialog_stage LIKE '%parse%'
       OR dialog_stage LIKE '%sync%'
       OR application_id IS NULL)
GROUP BY day, dialog_stage
ORDER BY day;
```

### 6. Количество уникальных диалогов vs количество вызовов Claude

```sql
SELECT 
    DATE(p.created_at) as day,
    COUNT(DISTINCT p.ai_session_id) as unique_sessions,
    COUNT(DISTINCT p.application_id) as unique_applications,
    COUNT(*) as total_claude_calls,
    ROUND(COUNT(*) / NULLIF(COUNT(DISTINCT p.ai_session_id), 0), 1) as calls_per_session,
    ROUND(SUM(p.cost_usd), 4) as total_cost,
    ROUND(SUM(p.cost_usd) / NULLIF(COUNT(DISTINCT p.ai_session_id), 0), 4) as cost_per_session
FROM ai_prompts_log p
WHERE p.created_at >= '2026-03-25 00:00:00'
GROUP BY day
ORDER BY day;
```

### 7. Дубли — один application_id с несколькими ai_session

```sql
SELECT 
    application_id,
    COUNT(DISTINCT ai_session_id) as sessions_count,
    COUNT(*) as total_calls,
    ROUND(SUM(cost_usd), 4) as total_cost
FROM ai_prompts_log
WHERE created_at >= '2026-03-25 00:00:00'
  AND application_id IS NOT NULL
GROUP BY application_id
HAVING sessions_count > 1
ORDER BY total_cost DESC;
```

### 8. Почасовой расход за вчера (увидим пики)

```sql
SELECT 
    HOUR(created_at) as hour_utc,
    COUNT(*) as calls,
    ROUND(SUM(cost_usd), 4) as cost_usd
FROM ai_prompts_log
WHERE created_at >= '2026-03-26 00:00:00'
  AND created_at < '2026-03-27 00:00:00'
GROUP BY HOUR(created_at)
ORDER BY hour_utc;
```

### 9. Есть ли ошибки (retry → двойные расходы)?

```sql
SELECT 
    DATE(created_at) as day,
    COUNT(*) as error_calls,
    ROUND(SUM(cost_usd), 4) as wasted_cost,
    GROUP_CONCAT(DISTINCT LEFT(error, 80) SEPARATOR ' | ') as errors
FROM ai_prompts_log
WHERE created_at >= '2026-03-25 00:00:00'
  AND error IS NOT NULL
GROUP BY day;
```

### 10. Проверка: сколько раз запускался синк вакансий

```sql
SELECT 
    DATE(created_at) as day,
    event_type,
    COUNT(*) as count
FROM event_log
WHERE created_at >= '2026-03-25 00:00:00'
  AND (event_type LIKE '%vacanc%' OR event_type LIKE '%sync%' OR event_type LIKE '%refresh%')
GROUP BY day, event_type
ORDER BY day;
```

---

## Формат вывода

Выведи результаты ВСЕХ 10 запросов. Каждый запрос — заголовок + результат.
Если какая-то таблица не существует или запрос падает — выведи ошибку и иди дальше.

В конце — **СВОДКА** одним абзацем:
- Главный источник расходов (синк vs диалоги)
- Средняя стоимость диалога
- Есть ли аномалии (дубли, пики, ошибки)
- Рекомендации

---

## Выполнение

Подключись к БД и выполни все запросы:

```bash
mysql -u root -p 2_kadry_4_crm_avito -e "ЗАПРОС"
```

Или через Python:
```bash
cd /opt/openai/crm-worker
source venv/bin/activate
python3 -c "
import asyncio
from sqlalchemy import text
from models.db import AsyncSessionFactory
# ... выполни запросы
"
```
