# TASK: Пересчёт baseline при проверке план-факт

## Проблема

Baseline (`planning_entries.baseline_value`) считается **в момент подачи плана** (пятница), но базовая неделя ещё не закончилась. Например:

- Менеджер пишет `#Планирование` в **пятницу 06.03** (ISO неделя 10)
- `_get_next_week()` → `plan_week = 11`
- `_get_baseline_week(11)` → неделя 10 = **02.03–08.03**
- Baseline считается сразу = SUM(vykhod) за 02.03–06.03 = **124** (неполная неделя, нет Сб+Вс)
- В понедельник при проверке реальная сумма за полную неделю = **180**
- Результат: вместо реального +5% бот показывает +45%, потому что сравнивает факт с заниженным baseline

## Что исправить

### 1. `app/services/timer_worker.py` — метод `_collect_submission_fact` (или `_process_submission_fact`)

**Перед расчётом fact_value для каждого entry**, пересчитать baseline из exits_wa:

```python
# Пересчитать baseline — при подаче плана неделя могла быть неполной
baseline_monday = target_monday - timedelta(days=7)
baseline_sunday = baseline_monday + timedelta(days=6)

fresh_baseline = await planning_svc._get_weekly_exits(
    project_id=entry.project_id,
    group_id=entry.group_id,
    week_start=baseline_monday,
    week_end=baseline_sunday
)
if fresh_baseline is not None:
    entry.baseline_value = fresh_baseline
```

Где:
- `target_monday` — понедельник целевой недели (plan_week), уже вычисляется в методе
- `planning_svc._get_weekly_exits()` — существующий метод в `app/services/planning_service.py`, умеет работать с project_id и group_id
- После пересчёта baseline, расчёт fact_growth_percent пойдёт с правильными числами

### 2. Порядок операций в цикле по entries

Для каждого entry должно быть:
1. Пересчитать `baseline_value` за полную базовую неделю (N-1)
2. Получить `fact_value` за целевую неделю (N)
3. Рассчитать `fact_growth_percent = (fact - baseline) / baseline * 100`
4. Определить `plan_status`

### 3. НЕ менять `planning_service.py`

Сохранение baseline при подаче плана (`_save_submission_and_entries`) оставить как есть — это полезно для превью, но финальный расчёт должен пересчитывать baseline в timer_worker.

## Как проверить

### Автоматическая проверка

```bash
python3 -m py_compile app/services/timer_worker.py
python3 -m py_compile app/services/planning_service.py
```

### Проверка логики на реальных данных

1. Сбросить факт для недели 11 (чтобы бот пересчитал):

```sql
UPDATE planning_submissions
SET fact_checked_at = NULL, fact_report_sent = 0
WHERE plan_year = 2026 AND plan_week = 11;

UPDATE planning_entries pe
JOIN planning_submissions ps ON pe.submission_id = ps.id
SET pe.fact_value = NULL, pe.fact_growth_percent = NULL, pe.plan_status = NULL
WHERE ps.plan_year = 2026 AND ps.plan_week = 11;
```

2. После деплоя — вручную проверить что baseline обновился:

```sql
-- Пример: project_id=190, baseline должен стать ~180 (полная неделя 10)
SELECT pe.id, pe.project_id, pe.baseline_value, pe.fact_value, pe.fact_growth_percent
FROM planning_entries pe
JOIN planning_submissions ps ON pe.submission_id = ps.id
WHERE ps.plan_year = 2026 AND ps.plan_week = 11 AND pe.project_id = 190;
```

3. Эталонный расчёт для сверки (project_id=190):

```sql
-- Baseline (неделя 10: 02.03-08.03) — должно быть ~180
SELECT SUM(vykhod) FROM 2_kadry_4.exits_wa
WHERE project_id = 190 AND date BETWEEN '2026-03-02' AND '2026-03-08';

-- Fact (неделя 11: 09.03-15.03)
SELECT SUM(vykhod) FROM 2_kadry_4.exits_wa
WHERE project_id = 190 AND date BETWEEN '2026-03-09' AND '2026-03-15';
```

Правильный рост = (fact - 180) / 180 × 100%

## Файлы

| Файл | Действие |
|------|----------|
| `app/services/timer_worker.py` | EDIT — добавить пересчёт baseline перед расчётом факта |

## Контекст

- `_get_weekly_exits()` находится в `app/services/planning_service.py` (строки ~701-740)
- Принимает `project_id`, `group_id`, `week_start`, `week_end`
- Возвращает `Optional[int]` — SUM(vykhod) из `2_kadry_4.exits_wa`
- Для group_id суммирует через `project_group_members`
- Метод async, нужен await
