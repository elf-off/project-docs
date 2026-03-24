# FIX: Пересчет baseline при проверке план-факт

## Проблема

Baseline (`planning_entries.baseline_value`) считался в момент подачи плана (пятница), когда базовая неделя еще не завершилась. Это приводило к заниженному baseline и искаженным процентам роста.

Пример: менеджер подает план в пятницу (02.03-08.03 -- неделя 10), baseline = SUM за 02.03-06.03 = 124.
В понедельник реальная сумма за полную неделю = 180. Бот показывал +45% вместо реальных +5%.

## Что сделано

Файл: `app/services/timer_worker.py`, метод `_collect_submission_fact`

Добавлен пересчет baseline за полную базовую неделю (N-1) перед расчетом fact_growth_percent.

### Порядок операций для каждого entry (после исправления)

1. Получить `fact_value` за целевую неделю N (`target_monday` .. `target_sunday`)
2. Пересчитать `baseline_value` за полную базовую неделю N-1 (`target_monday - 7` .. `target_monday - 1`)
3. Рассчитать `fact_growth_percent = (fact - baseline) / baseline * 100`
4. Определить `plan_status` (achieved / partial / missed / no_data)

### Детали реализации

- Используется существующий метод `_get_weekly_exits_db()` (тот же, что и для получения fact)
- Если пересчет baseline не удался (exception), используется исходное значение (graceful fallback)
- `planning_service.py` не изменен -- baseline при подаче плана сохраняется как есть (полезно для превью)

## Развертывание

### 1. Деплой кода

```bash
cd /opt/openai/tg_oleg_assist
git pull origin main
systemctl restart k24-oleg-bot.service
systemctl status k24-oleg-bot.service
```

### 2. Сброс факта для пересчета (при необходимости)

Если нужно пересчитать уже проверенные недели:

```sql
-- Сбросить факт для конкретной недели (пример: неделя 11, 2026)
UPDATE planning_submissions
SET fact_checked_at = NULL, fact_report_sent = 0
WHERE plan_year = 2026 AND plan_week = 11;

UPDATE planning_entries pe
JOIN planning_submissions ps ON pe.submission_id = ps.id
SET pe.fact_value = NULL, pe.fact_growth_percent = NULL, pe.plan_status = NULL
WHERE ps.plan_year = 2026 AND ps.plan_week = 11;
```

После рестарта TimerWorker автоматически пересчитает при следующем цикле проверки.

### 3. Проверка результата

```sql
-- Проверить что baseline обновился (пример: project_id=190)
SELECT pe.id, pe.project_id, pe.baseline_value, pe.fact_value, pe.fact_growth_percent
FROM planning_entries pe
JOIN planning_submissions ps ON pe.submission_id = ps.id
WHERE ps.plan_year = 2026 AND ps.plan_week = 11 AND pe.project_id = 190;
```

Эталонные запросы для ручной сверки:

```sql
-- Baseline (неделя 10: 02.03-08.03) -- должно быть ~180
SELECT SUM(vykhod) FROM 2_kadry_4.exits_wa
WHERE project_id = 190 AND date BETWEEN '2026-03-02' AND '2026-03-08';

-- Fact (неделя 11: 09.03-15.03)
SELECT SUM(vykhod) FROM 2_kadry_4.exits_wa
WHERE project_id = 190 AND date BETWEEN '2026-03-09' AND '2026-03-15';
```

## Проверка кода

```bash
python3 -m py_compile app/services/timer_worker.py   # OK
python3 -m py_compile app/services/planning_service.py  # OK (не менялся)
python tests/test_rules_loader.py                       # OK
python tests/test_scoring_logic.py                      # OK
```
