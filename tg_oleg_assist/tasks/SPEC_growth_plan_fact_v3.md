# SPEC: Анализ План-Факт роста по проектам (v2)

## Цель

Добавить автоматическую проверку факта к уже существующему сбору планов роста.

**Что уже работает**: модуль `#Планирование` сохраняет `change_percent` по проектам в `planning_entries`.

**Что нужно добавить**: в понедельник автоматически проверять факт по `exits_wa` и отправлять отчёт в чат.

---

## Актуальная структура БД (из дампа 27.02.2026)

### planning_entries (ТЕКУЩАЯ)

```sql
CREATE TABLE planning_entries (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  submission_id BIGINT NOT NULL,        -- FK → planning_submissions
  project_id INT UNSIGNED DEFAULT NULL,  -- FK → 2_kadry_4.ref_projects_wa
  group_id INT UNSIGNED DEFAULT NULL,    -- FK → 2_kadry_4.ref_project_groups
  change_percent DECIMAL(10,2) NOT NULL  -- План роста: +10.00, -5.00
);
```

**Уже есть**: `change_percent` = плановый рост, `project_id` / `group_id` = привязка к проекту/группе.

### planning_submissions (ТЕКУЩАЯ)

```sql
CREATE TABLE planning_submissions (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  chat_id BIGINT NOT NULL,               -- FK → chats
  manager_id BIGINT NOT NULL,            -- FK → planning_managers
  tg_message_id BIGINT NOT NULL,
  plan_year SMALLINT NOT NULL,           -- Год плана (2026)
  plan_week TINYINT NOT NULL,            -- Номер недели ISO (1-53)
  submitted_at DATETIME NOT NULL,
  is_accepted TINYINT(1) DEFAULT 1,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

**Ключевое**: неделя хранится как `plan_year` + `plan_week` (ISO week number).

### planning_managers (ТЕКУЩАЯ)

```sql
CREATE TABLE planning_managers (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  chat_id BIGINT NOT NULL,          -- FK → chats
  tg_user_id BIGINT NOT NULL,
  full_name VARCHAR(128),
  username VARCHAR(64),
  is_active TINYINT(1) DEFAULT 1,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY (chat_id, tg_user_id)
);
```

### exits_wa (ВНЕШНЯЯ БД: 2_kadry_4)

```sql
CREATE TABLE exits_wa (
  id INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  project_id INT UNSIGNED NOT NULL,    -- FK → ref_projects_wa
  date DATE NOT NULL,
  shift_id TINYINT UNSIGNED NOT NULL,  -- FK → ref_shifts
  zayavka INT DEFAULT NULL,            -- Заявка (план на смену)
  vykhod INT DEFAULT NULL,             -- Факт выходов
  UNIQUE KEY (project_id, date, shift_id)
);
```

### project_group_members (ВНЕШНЯЯ БД: 2_kadry_4)

```sql
CREATE TABLE project_group_members (
  id INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  group_id INT UNSIGNED NOT NULL,     -- FK → ref_project_groups
  project_id INT UNSIGNED NOT NULL,   -- FK → ref_projects_wa
  UNIQUE KEY (group_id, project_id)
);
```

---

## Бизнес-логика

### Цикл

```
Пятница недели N:
  Менеджер пишет:
  ────────────────────────────────────────
  #Планирование
  Фамилия К24  +10%
  Kari -5%
  ────────────────────────────────────────

  Бот (УЖЕ РАБОТАЕТ):
  1. Парсит проекты → matching с ref_projects_wa
  2. Сохраняет planning_submission (plan_year, plan_week = неделя N+1)
  3. Сохраняет planning_entries (change_percent = +10, -5)

  Бот (НУЖНО ДОБАВИТЬ — при сохранении):
  4. Для каждого entry запрашивает БАЗУ:
     → SUM(vykhod) из exits_wa за неделю N
     → Сохраняет как baseline_value

Понедельник недели N+2, 10:00 (TimerWorker):
  Бот (НУЖНО ДОБАВИТЬ):
  1. Находит submissions где plan_week = прошлая неделя И fact_checked = FALSE
  2. Для каждого entry:
     → fact_value = SUM(vykhod) за целевую неделю (plan_week)
     → fact_growth = (fact - baseline) / baseline × 100%
     → plan_status = achieved / partial / missed / no_data
  3. Формирует отчёт
  4. Отправляет в ОТДЕЛЬНЫЙ чат руководства (из .env: GROWTH_REPORT_CHAT_ID)
  5. Отмечает submission.fact_checked = TRUE
```

### Расчёт дат из plan_year + plan_week

```python
from datetime import date, timedelta
import datetime

def week_to_dates(plan_year: int, plan_week: int) -> tuple[date, date]:
    """
    ISO week → (понедельник, воскресенье).
    plan_year=2026, plan_week=7 → (2026-02-09, 2026-02-15)
    """
    # ISO week: Monday is day 1
    monday = datetime.date.fromisocalendar(plan_year, plan_week, 1)
    sunday = monday + timedelta(days=6)
    return monday, sunday

def previous_week(plan_year: int, plan_week: int) -> tuple[int, int]:
    """Возвращает (year, week) предыдущей недели."""
    d = datetime.date.fromisocalendar(plan_year, plan_week, 1) - timedelta(days=7)
    iso = d.isocalendar()
    return iso[0], iso[1]
```

### Пример расчёта

```
Менеджер написал в пятницу 07.02: «Фамилия К24 +10%»
→ planning_submission: plan_year=2026, plan_week=7  (неделя 09.02-15.02)
→ planning_entry: project_id=9, change_percent=10.00

БАЗА (неделя 6 = 02.02-08.02):
  SUM(vykhod) FROM exits_wa WHERE project_id=9 AND date BETWEEN '2026-02-02' AND '2026-02-08'
  → baseline_value = 140

ФАКТ (неделя 7 = 09.02-15.02) — проверяется в понедельник 16.02:
  SUM(vykhod) FROM exits_wa WHERE project_id=9 AND date BETWEEN '2026-02-09' AND '2026-02-15'
  → fact_value = 161

fact_growth = (161 - 140) / 140 × 100% = +15.0%
plan = +10%
Результат: ✅ Выполнено (факт +15% при плане +10%)
```

### Обработка group_id (группа проектов)

Если `planning_entry.group_id IS NOT NULL` (менеджер указал группу, а не конкретный проект):

```sql
-- Получить сумму выходов по ВСЕМ проектам группы
SELECT COALESCE(SUM(e.vykhod), 0) AS total
FROM 2_kadry_4.exits_wa e
JOIN 2_kadry_4.project_group_members pgm ON e.project_id = pgm.project_id
WHERE pgm.group_id = :group_id
  AND e.date BETWEEN :week_start AND :week_end;
```

---

## Изменения в БД

### Миграция 007_growth_plan_fact.sql

```sql
-- =====================================================
-- Migration 007: Growth Plan-Fact Analysis
-- Date: 2026-02-27
-- Description: Add fact-checking fields to planning tables
-- =====================================================

-- 1. Добавить поля факта в planning_entries
ALTER TABLE planning_entries
  ADD COLUMN baseline_value INT DEFAULT NULL
    COMMENT 'SUM(vykhod) за базовую неделю (N), заполняется при создании плана',
  ADD COLUMN fact_value INT DEFAULT NULL
    COMMENT 'SUM(vykhod) за целевую неделю (plan_week), заполняется при проверке',
  ADD COLUMN fact_growth_percent DECIMAL(10,2) DEFAULT NULL
    COMMENT 'Фактический % роста = (fact - baseline) / baseline * 100',
  ADD COLUMN plan_status VARCHAR(20) DEFAULT NULL
    COMMENT 'achieved | partial | missed | no_data';

ALTER TABLE planning_entries
  ADD INDEX idx_plan_status (plan_status);

-- 2. Добавить флаг проверки в planning_submissions
ALTER TABLE planning_submissions
  ADD COLUMN fact_checked_at DATETIME DEFAULT NULL
    COMMENT 'Когда бот проверил план-факт (NULL = ещё не проверял)',
  ADD COLUMN fact_report_sent TINYINT(1) DEFAULT 0
    COMMENT 'Отправлен ли отчёт в чат';

ALTER TABLE planning_submissions
  ADD INDEX idx_fact_checked (fact_checked_at);

-- 3. Таблица логов отчётов
CREATE TABLE IF NOT EXISTS growth_reports (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    submission_id BIGINT NOT NULL
      COMMENT 'FK → planning_submissions',
    manager_id BIGINT NOT NULL
      COMMENT 'FK → planning_managers',
    tg_chat_id BIGINT NOT NULL,
    plan_year SMALLINT NOT NULL,
    plan_week TINYINT NOT NULL,
    total_projects INT NOT NULL DEFAULT 0,
    achieved_count INT NOT NULL DEFAULT 0,
    report_text TEXT,
    sent_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    correlation_id VARCHAR(36),

    INDEX idx_submission_id (submission_id),
    INDEX idx_manager_id (manager_id),
    INDEX idx_plan_week (plan_year, plan_week),
    FOREIGN KEY (submission_id) REFERENCES planning_submissions(id) ON DELETE CASCADE,
    FOREIGN KEY (manager_id) REFERENCES planning_managers(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci
COMMENT='Логи отправленных план-факт отчётов по росту';
```

---

## Изменения в коде

### 1. planning_service.py — сохранение baseline при создании плана

При сохранении `planning_entry` с `project_id` или `group_id`, сразу запросить базу:

```python
# После matching проекта и создания entry:
entry = PlanningEntry(
    submission_id=submission.id,
    project_id=matched_project_id,   # или None
    group_id=matched_group_id,       # или None
    change_percent=growth_percent,   # +10.00, -5.00
)

# НОВОЕ: запросить baseline
baseline_week_start, baseline_week_end = self._get_baseline_week(
    submission.plan_year, submission.plan_week
)
entry.baseline_value = await self._get_weekly_exits(
    project_id=entry.project_id,
    group_id=entry.group_id,
    week_start=baseline_week_start,
    week_end=baseline_week_end
)
```

### 2. planning_service.py — новые методы

```python
def _get_baseline_week(self, plan_year: int, plan_week: int) -> tuple[date, date]:
    """
    Базовая неделя = неделя ДО целевой (plan_week - 1).
    plan_year=2026, plan_week=7 → baseline = неделя 6 (02.02-08.02)
    """
    import datetime
    target_monday = datetime.date.fromisocalendar(plan_year, plan_week, 1)
    baseline_monday = target_monday - timedelta(days=7)
    baseline_sunday = baseline_monday + timedelta(days=6)
    return baseline_monday, baseline_sunday


async def _get_weekly_exits(
    self,
    project_id: Optional[int],
    group_id: Optional[int],
    week_start: date,
    week_end: date
) -> Optional[int]:
    """
    SUM(vykhod) из 2_kadry_4.exits_wa за неделю.

    Если project_id — сумма по одному проекту.
    Если group_id — сумма по всем проектам группы (через project_group_members).

    ВАЖНО: запрос в ДРУГУЮ БД (2_kadry_4), не в 2_kadry_4_ethics_bot.
    Пользователь ethics_user должен иметь SELECT-доступ.
    """
    if project_id:
        query = text("""
            SELECT COALESCE(SUM(vykhod), 0) AS total
            FROM 2_kadry_4.exits_wa
            WHERE project_id = :project_id
              AND date BETWEEN :week_start AND :week_end
        """)
        params = {
            'project_id': project_id,
            'week_start': week_start,
            'week_end': week_end
        }
    elif group_id:
        query = text("""
            SELECT COALESCE(SUM(e.vykhod), 0) AS total
            FROM 2_kadry_4.exits_wa e
            JOIN 2_kadry_4.project_group_members pgm
              ON e.project_id = pgm.project_id
            WHERE pgm.group_id = :group_id
              AND e.date BETWEEN :week_start AND :week_end
        """)
        params = {
            'group_id': group_id,
            'week_start': week_start,
            'week_end': week_end
        }
    else:
        return None

    result = await self.db.execute(query, params)
    row = result.fetchone()
    return row.total if row else 0
```

### 3. timer_worker.py — проверка план-факт по понедельникам

В метод `run()` добавить:

```python
# В основном цикле TimerWorker, после существующих проверок:
from app.config import settings

if (now_msk.weekday() == 0
    and now_msk.hour == settings.growth_check_hour
    and 0 <= now_msk.minute < 1
    and settings.growth_report_chat_id):
    # Понедельник, настроенное время MSK — проверяем план-факт
    await self._check_growth_plan_fact(now_msk)
```

Метод проверки:

```python
async def _check_growth_plan_fact(self, now: datetime):
    """
    Проверяет план-факт для всех submissions прошлой недели.

    Логика:
    - Сегодня понедельник недели N+2
    - Целевая неделя = прошлая неделя (N+1)
    - Ищем submissions с plan_week = N+1, fact_checked_at IS NULL
    """
    # Определяем прошлую неделю (целевую для проверки)
    yesterday = (now - timedelta(days=1)).date()  # Воскресенье
    iso = yesterday.isocalendar()
    target_year, target_week = iso[0], iso[1]

    self.log.info(
        "Checking growth plan-fact",
        target_year=target_year,
        target_week=target_week
    )

    async with get_db_session() as db:
        # Найти непроверенные submissions за целевую неделю
        query = (
            select(PlanningSubmission)
            .where(
                PlanningSubmission.plan_year == target_year,
                PlanningSubmission.plan_week == target_week,
                PlanningSubmission.fact_checked_at.is_(None)
            )
        )
        result = await db.execute(query)
        submissions = result.scalars().all()

        if not submissions:
            self.log.info("No unchecked submissions for target week")
            return

        for submission in submissions:
            await self._process_submission_fact(db, submission, target_year, target_week)


async def _process_submission_fact(
    self, db, submission, target_year: int, target_week: int
):
    """Обработка одного submission: проверка факта и отправка отчёта."""

    # Даты целевой недели
    target_monday = datetime.date.fromisocalendar(target_year, target_week, 1)
    target_sunday = target_monday + timedelta(days=6)

    # Загрузить entries этого submission
    entries_query = (
        select(PlanningEntry)
        .where(PlanningEntry.submission_id == submission.id)
    )
    entries_result = await db.execute(entries_query)
    entries = entries_result.scalars().all()

    if not entries:
        return

    planning_svc = PlanningService(ctx=None, db=db, correlation_id=str(uuid.uuid4()))

    for entry in entries:
        if entry.project_id is None and entry.group_id is None:
            entry.plan_status = 'no_data'
            continue

        # Получить факт за целевую неделю
        fact_value = await planning_svc._get_weekly_exits(
            project_id=entry.project_id,
            group_id=entry.group_id,
            week_start=target_monday,
            week_end=target_sunday
        )
        entry.fact_value = fact_value

        # Рассчитать факт роста
        if entry.baseline_value and entry.baseline_value > 0 and fact_value is not None:
            entry.fact_growth_percent = round(
                (fact_value - entry.baseline_value) / entry.baseline_value * 100,
                2
            )
            # Определить статус
            plan = float(entry.change_percent)
            fact = float(entry.fact_growth_percent)
            if fact >= plan:
                entry.plan_status = 'achieved'
            elif fact >= plan - 10:
                entry.plan_status = 'partial'
            else:
                entry.plan_status = 'missed'
        else:
            entry.plan_status = 'no_data'
            entry.fact_growth_percent = None

    await db.flush()

    # Сформировать и отправить отчёт
    report_text = self._format_growth_report(submission, entries, target_year, target_week)

    # Отправить в чат руководства (из .env)
    from app.config import settings
    if settings.growth_report_chat_id:
        await self.notifier.send_to_chat(
            chat_tg_id=settings.growth_report_chat_id,
            text=report_text,
            parse_mode='HTML'
        )
    else:
        self.log.warning(
            "GROWTH_REPORT_CHAT_ID not set, skipping report",
            submission_id=submission.id
        )

    # Отметить submission как проверенный
    submission.fact_checked_at = datetime.datetime.now()
    submission.fact_report_sent = True

    # Сохранить лог
    manager = await db.get(PlanningManager, submission.manager_id)
    achieved = sum(1 for e in entries if e.plan_status == 'achieved')
    growth_report = GrowthReport(
        submission_id=submission.id,
        manager_id=submission.manager_id,
        tg_chat_id=settings.growth_report_chat_id or 0,
        plan_year=target_year,
        plan_week=target_week,
        total_projects=len([e for e in entries if e.plan_status is not None]),
        achieved_count=achieved,
        report_text=report_text,
        correlation_id=str(uuid.uuid4())
    )
    db.add(growth_report)
    await db.commit()

    self.log.info(
        "Growth report sent",
        manager_id=submission.manager_id,
        achieved=achieved,
        total=len(entries)
    )
```

### 4. Форматирование отчёта

```python
def _format_growth_report(
    self,
    submission,
    entries: list,
    plan_year: int,
    plan_week: int
) -> str:
    """Форматирует HTML-отчёт план-факт для Telegram."""
    import datetime

    monday = datetime.date.fromisocalendar(plan_year, plan_week, 1)
    sunday = monday + timedelta(days=6)
    week_label = f"{monday:%d.%m}-{sunday:%d.%m.%Y}"

    # Имя менеджера (из planning_managers)
    display_name = f"@{submission.manager.username}" if submission.manager.username else submission.manager.full_name

    lines = [f"<b>📊 Отчёт план-факт за неделю {week_label}</b>"]
    lines.append("")
    lines.append(f"<b>Менеджер: {display_name}</b>")
    lines.append("")

    achieved = 0
    total = 0

    for entry in entries:
        if entry.plan_status is None:
            continue

        total += 1
        # Название проекта
        project_name = self._get_entry_name(entry)

        plan_pct = float(entry.change_percent)

        if entry.plan_status == 'no_data':
            lines.append(f"⚠️ <b>{project_name}</b>")
            if entry.baseline_value is None or entry.baseline_value == 0:
                lines.append(f"   План: {plan_pct:+.0f}% | Нет базовых данных за пред. неделю")
            else:
                lines.append(f"   План: {plan_pct:+.0f}% | Нет данных о выходах за целевую неделю")

        elif entry.plan_status == 'achieved':
            achieved += 1
            fact_pct = float(entry.fact_growth_percent)
            diff = fact_pct - plan_pct
            lines.append(f"✅ <b>{project_name}</b>")
            lines.append(
                f"   План: {plan_pct:+.0f}% | Факт: {fact_pct:+.1f}% | "
                f"База: {entry.baseline_value} → Факт: {entry.fact_value}"
            )
            if diff > 0.5:
                lines.append(f"   <i>Перевыполнение на {diff:.1f}%</i>")
            else:
                lines.append(f"   <i>Выполнено</i>")

        elif entry.plan_status == 'partial':
            fact_pct = float(entry.fact_growth_percent)
            diff = plan_pct - fact_pct
            lines.append(f"⚠️ <b>{project_name}</b>")
            lines.append(
                f"   План: {plan_pct:+.0f}% | Факт: {fact_pct:+.1f}% | "
                f"База: {entry.baseline_value} → Факт: {entry.fact_value}"
            )
            lines.append(f"   <i>Недовыполнение на {diff:.1f}%</i>")

        else:  # missed
            fact_pct = float(entry.fact_growth_percent)
            diff = plan_pct - fact_pct
            lines.append(f"❌ <b>{project_name}</b>")
            lines.append(
                f"   План: {plan_pct:+.0f}% | Факт: {fact_pct:+.1f}% | "
                f"База: {entry.baseline_value} → Факт: {entry.fact_value}"
            )
            lines.append(f"   <i>Не выполнено (отклонение {diff:.1f}%)</i>")

        lines.append("")

    lines.append("━━━━━━━━━━━━━━━━━━━━━━")
    pct = round(achieved / total * 100) if total > 0 else 0
    lines.append(f"<b>Итого: {achieved} из {total} выполнено ({pct}%)</b>")

    return "\n".join(lines)


def _get_entry_name(self, entry) -> str:
    """Получить название проекта или группы для отображения."""
    # Через кеш или запрос к ref_projects_wa / ref_project_groups
    if entry.project_id:
        # SELECT name FROM 2_kadry_4.ref_projects_wa WHERE id = entry.project_id
        return entry._project_name or f"Проект #{entry.project_id}"
    elif entry.group_id:
        # SELECT title FROM 2_kadry_4.ref_project_groups WHERE id = entry.group_id
        return entry._group_title or f"Группа #{entry.group_id}"
    return "Неизвестный проект"
```

### 5. models/planning.py — добавить новые поля

```python
# В модели PlanningEntry добавить:
baseline_value = Column(Integer, nullable=True, comment='SUM(vykhod) за базовую неделю')
fact_value = Column(Integer, nullable=True, comment='SUM(vykhod) за целевую неделю')
fact_growth_percent = Column(Numeric(10, 2), nullable=True, comment='Фактический % роста')
plan_status = Column(String(20), nullable=True, comment='achieved|partial|missed|no_data')

# В модели PlanningSubmission добавить:
fact_checked_at = Column(DateTime, nullable=True, comment='Когда проверен план-факт')
fact_report_sent = Column(Boolean, default=False, comment='Отправлен ли отчёт')
```

### 6. models/growth_report.py — новая модель

```python
class GrowthReport(Base):
    __tablename__ = 'growth_reports'

    id = Column(BigInteger, primary_key=True, autoincrement=True)
    submission_id = Column(BigInteger, ForeignKey('planning_submissions.id'), nullable=False)
    manager_id = Column(BigInteger, ForeignKey('planning_managers.id'), nullable=False)
    tg_chat_id = Column(BigInteger, nullable=False)
    plan_year = Column(SmallInteger, nullable=False)
    plan_week = Column(SmallInteger, nullable=False)
    total_projects = Column(Integer, default=0)
    achieved_count = Column(Integer, default=0)
    report_text = Column(Text, nullable=True)
    sent_at = Column(DateTime, server_default=func.now())
    correlation_id = Column(String(36), nullable=True)
```

---

## Конфигурация

### .env — глобальные параметры

```bash
# Куда отправлять план-факт отчёты (tg_chat_id чата руководства)
GROWTH_REPORT_CHAT_ID=-1009876543210

# Время проверки (MSK, понедельник)
GROWTH_CHECK_HOUR=10
GROWTH_CHECK_MINUTE=0
```

### config.py — Pydantic Settings

```python
# Добавить в класс Settings:
growth_report_chat_id: int = 0          # 0 = отчёт не отправляется
growth_check_hour: int = 10
growth_check_minute: int = 0
```

### Использование в коде

```python
from app.config import settings

# В TimerWorker:
if settings.growth_report_chat_id:
    await self.notifier.send_to_chat(
        chat_tg_id=settings.growth_report_chat_id,
        text=report_text,
        parse_mode='HTML'
    )
else:
    self.log.warning("GROWTH_REPORT_CHAT_ID not set, skipping report")
```

**ВАЖНО**: Отчёт отправляется НЕ в тот чат, где менеджер пишет `#Планирование`,
а в ОТДЕЛЬНЫЙ чат руководства. Если `GROWTH_REPORT_CHAT_ID=0` — отчёт не
отправляется, только логирование.

### chats.features — feature flag (опционально)

Включение/выключение план-факт проверки per-chat:
```json
{"growth_check": true}
```

Проверяется как `ctx.has_feature('growth_check')` при сохранении baseline.
Если не нужна granularity per-chat — можно обойтись без этого, всё управлять через .env.

### Права доступа к внешней БД

```sql
-- Выполнить на сервере:
GRANT SELECT ON 2_kadry_4.exits_wa TO 'ethics_user'@'localhost';
GRANT SELECT ON 2_kadry_4.ref_projects_wa TO 'ethics_user'@'localhost';
GRANT SELECT ON 2_kadry_4.project_group_members TO 'ethics_user'@'localhost';
GRANT SELECT ON 2_kadry_4.ref_project_groups TO 'ethics_user'@'localhost';
FLUSH PRIVILEGES;
```

---

## Файлы для создания/изменения

| Действие | Файл | Описание |
|----------|------|----------|
| CREATE | `migrations/007_growth_plan_fact.sql` | ALTER planning_entries + planning_submissions, CREATE growth_reports |
| CREATE | `app/models/growth_report.py` | SQLAlchemy модель GrowthReport |
| EDIT | `app/models/__init__.py` | Добавить `from .growth_report import GrowthReport` |
| EDIT | `app/models/planning.py` | Добавить поля: baseline_value, fact_value, fact_growth_percent, plan_status в PlanningEntry; fact_checked_at, fact_report_sent в PlanningSubmission |
| EDIT | `app/config.py` | Добавить: `growth_report_chat_id: int = 0`, `growth_check_hour: int = 10`, `growth_check_minute: int = 0` |
| EDIT | `.env` | Добавить: `GROWTH_REPORT_CHAT_ID`, `GROWTH_CHECK_HOUR`, `GROWTH_CHECK_MINUTE` |
| EDIT | `.env.example` | Добавить те же переменные с комментариями |
| EDIT | `app/services/planning_service.py` | Добавить методы: `_get_weekly_exits()`, `_get_baseline_week()`. Вызывать при сохранении entries |
| EDIT | `app/services/timer_worker.py` | Добавить `_check_growth_plan_fact()`, `_process_submission_fact()`, `_format_growth_report()`, `_get_entry_name()`. Вызов в понедельник по settings.growth_check_hour |

---

## Edge Cases

### 1. entry.project_id IS NULL AND entry.group_id IS NULL
- Проект не заматчен → `plan_status = 'no_data'`
- В отчёте: `⚠️ Проект не привязан к справочнику`

### 2. baseline_value = 0 (новый проект)
- Невозможно рассчитать % роста (деление на 0)
- `plan_status = 'no_data'`
- В отчёте: `⚠️ Нет базовых данных за предыдущую неделю`

### 3. fact_value = 0 при baseline > 0
- `fact_growth = -100%`
- Рассчитывается нормально, скорее всего `missed`

### 4. group_id вместо project_id
- Суммируем vykhod по ВСЕМ проектам группы через project_group_members
- Название берём из ref_project_groups.title

### 5. Несколько submissions за одну неделю (повторная отправка)
- Текущая логика: DELETE старых + создание новых (CASCADE на entries)
- Baseline пересчитывается при повторной отправке

### 6. Данные exits_wa ещё не внесены в понедельник
- `growth_check_hour = 10` даёт запас
- Если данных нет → `fact_value = 0` → `no_data` или `missed`
- Можно добавить retry в 14:00 для `no_data` записей (опционально, v2)

### 7. Переход года (неделя 52/53 → неделя 1)
- ISO week корректно обрабатывает через `fromisocalendar()`

### 8. Смены (shift_id)
- Суммируем ВСЕ смены: `SUM(vykhod)` без фильтра по shift_id

---

## Обратная совместимость

- Новые поля `baseline_value`, `fact_value` и т.д. = NULL для старых записей
- TimerWorker проверяет только submissions с `fact_checked_at IS NULL`
- Старые entries без `change_percent` не затрагиваются (change_percent уже NOT NULL в текущей схеме)
- Нет breaking changes

---

## Проверка после внедрения

1. Добавить в `.env`:
   ```bash
   GROWTH_REPORT_CHAT_ID=-100XXXXXXXXXX  # tg_chat_id чата руководства
   GROWTH_CHECK_HOUR=10
   GROWTH_CHECK_MINUTE=0
   ```

2. Синтаксис:
   ```bash
   python3 -m py_compile app/services/planning_service.py
   python3 -m py_compile app/services/timer_worker.py
   python3 -m py_compile app/models/planning.py
   python3 -m py_compile app/models/growth_report.py
   python3 -m py_compile app/config.py
   ```

3. Миграция:
   ```bash
   mysql -u root -p 2_kadry_4_ethics_bot < migrations/007_growth_plan_fact.sql
   ```

4. Права:
   ```bash
   mysql -u root -p -e "GRANT SELECT ON 2_kadry_4.exits_wa TO 'ethics_user'@'localhost'; GRANT SELECT ON 2_kadry_4.ref_projects_wa TO 'ethics_user'@'localhost'; GRANT SELECT ON 2_kadry_4.project_group_members TO 'ethics_user'@'localhost'; GRANT SELECT ON 2_kadry_4.ref_project_groups TO 'ethics_user'@'localhost'; FLUSH PRIVILEGES;"
   ```

5. Перезапуск:
   ```bash
   sudo systemctl restart ethics-bot
   ```

6. Тест сохранения baseline — отправить #Планирование и проверить:
   ```sql
   SELECT pe.*, ps.plan_year, ps.plan_week
   FROM planning_entries pe
   JOIN planning_submissions ps ON pe.submission_id = ps.id
   ORDER BY ps.id DESC LIMIT 10;
   ```
   Проверить что `baseline_value` заполнен.

7. Тест отчёта — вручную вызвать проверку или дождаться понедельника 10:00.
