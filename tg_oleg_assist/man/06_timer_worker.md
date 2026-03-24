# Фоновый воркер (TimerWorker)

`app/services/timer_worker.py` — запускается при старте приложения как `asyncio.Task`, работает в бесконечном цикле с интервалом 30 секунд.

## Жизненный цикл

```python
# main.py
timer_worker = TimerWorker()
timer_task = asyncio.create_task(timer_worker.run())
```

При остановке приложения (SIGTERM/SIGINT или завершение lifespan) устанавливается `timer_worker.running = False`. Воркер ждёт 5 секунд на завершение текущей итерации, затем задача отменяется.

Обработчики сигналов (`SIGTERM`, `SIGINT`) также устанавливают `running = False` для корректного завершения при системном перезапуске.

## Главный цикл

```python
while self.running:
    now = datetime.now()
    now_msk = datetime.now(pytz.timezone('Europe/Moscow'))

    if is_workday():                        # только пн-пт
        await _check_overdue_mentions()
        await _check_report_reminders(now)
        await _check_planner_jobs(now)

        if (now_msk.weekday() == 0          # понедельник
                and time == growth_check_time
                and growth_report_chat_id != 0):
            await _check_growth_plan_fact(now_msk)

    await _check_tasks()                    # задачи — всегда

    sleep(30 секунд, с проверкой running каждую секунду)
```

---

## Проверка просроченных упоминаний

`_check_overdue_mentions()` — загружает все чаты с feature `mention_tracking` через `ChatContextResolver.resolve_all_with_feature()`, для каждого ищет `Mention` со статусом `WAITING` и `deadline_at < now`.

Для каждого просроченного упоминания:
- Статус → `OVERDUE`
- Личное уведомление упомянутому пользователю (если `mention_notify_private=True`)
- Публичное напоминание в чат (если `mention_notify_chat=True`)

Настройки берутся из `ctx.config` каждого чата.

---

## Напоминания об отчётах

`_check_report_reminders(now)` — загружает чаты с feature `report_reminders`. Для каждого чата читает `ctx.get_report_check_times()` — список пар `(hour, minute)` из `chats.config`.

**Защита от двойного срабатывания:** множество `_report_checks_done` содержит кортежи `(chat_id, date, time)`. Если ключ уже там — пропуск.

Для каждой временной точки (±60 секунд от текущего времени):
1. Проверить, сданы ли отчёты всех типов (`DailyReport.status == SUBMITTED`)
2. Если нет — найти ответственных (`ReportResponsible`)
3. Отправить напоминания (личка и/или чат, по настройкам)
4. Создать или обновить `DailyReport` со статусом `REMINDED`

---

## Задания планёра ГД

`_check_planner_jobs(now)` — загружает чаты с feature `planner`. Для каждого чата определяет часовой пояс (из `chats.config`) и проверяет четыре временные точки дня.

**Защита от двойного срабатывания:** множество `_planner_checks_done` с ключами `(chat_id, date, job_type)`.

**Точность срабатывания:** ±60 секунд от целевого времени. При интервале воркера 30 секунд это гарантирует попадание в окно.

| job_type | Время (по умолчанию) | Действие |
|---|---|---|
| `morning_info` | 09:00 | `PlannerService.send_morning_info()` |
| `feedback_morning` | 14:00 | `PlannerService.send_feedback_request('morning')` |
| `feedback_afternoon` | 20:30 | `PlannerService.send_feedback_request('afternoon')` |
| `eod` | 21:30 | `PlannerService.end_of_day()` |

---

## Plan-Fact отчёт

`_check_growth_plan_fact(now_msk)` — срабатывает по понедельникам в `growth_check_hour:growth_check_minute` (по умолчанию 10:00 МСК) при условии, что `GROWTH_REPORT_CHAT_ID != 0`.

**Защита от двойного срабатывания:** множество `_growth_checks_done` с ключами `(plan_year, plan_week)`.

Целевая неделя — прошлая ISO-неделя (воскресенье вчера → `.isocalendar()`).

Подробный алгоритм описан в [04_planning.md](./04_planning.md).

---

## Задачи (Tasks)

`_check_tasks()` — работает всегда (не только в рабочие дни). Обрабатывает две ситуации:

**Просроченные задачи** (`deadline_at < now`, `status=WAITING`):
- Статус → `OVERDUE`
- Личное уведомление исполнителю

**Предстоящие задачи** (дедлайн через ≤15 минут, ещё не напомнили):
- Поле `reminded_at` заполняется текущим временем (защита от повторного напоминания)
- Личное уведомление исполнителю с указанием оставшегося времени

---

## Важные особенности

**In-memory защита от двойного срабатывания** — множества `_report_checks_done`, `_planner_checks_done`, `_growth_checks_done` живут в памяти процесса. При перезапуске бота они сбрасываются. Это означает, что после перезапуска в правильное время задание может сработать повторно. Для plan-fact это некритично (проверяется `fact_checked_at`), для планёра — бот пришлёт повторное утреннее сообщение.

**Ошибки изолированы:** каждый `_check_*` метод обёрнут в `try/except`. Ошибка в одной проверке не останавливает остальные.

**Структура базы данных для отчётности:** все факты фиксируются в БД (`GrowthReport`, обновление `PlanningSubmission.fact_checked_at`), что позволяет восстановить историю даже после перезапуска.
