# Corporate Ethics Bot

## Что это

**Corporate Ethics Bot** — FastAPI-based Telegram webhook bot для автоматизации корпоративных процессов. Система обеспечивает:

- 🔍 **Мониторинг корпоративной этики** — автоматический анализ сообщений с помощью Claude AI
- 📊 **CEO Daily Planner** — двухцикловая система обратной связи по планёркам (v1.3)
- 📢 **Mention Tracking** — отслеживание @упоминаний с дедлайнами ответа
- 📝 **Report Reminders** — напоминания о ежедневных отчётах (РОП, ПЛК)
- 📅 **Weekly Planning** — недельное планирование менеджеров с AI-matching проектов

---

## Технический стек

### Backend
- **Python**: 3.11+
- **Web Framework**: FastAPI 0.109.0
- **ASGI Server**: Uvicorn 0.27.0
- **ORM**: SQLAlchemy 2.0 (async)

### Database
- **СУБД**: MariaDB 10.x
- **Driver**: aiomysql 0.2.0
- **Schema**: 21 таблица (см. [DB_SCHEMA.md](./DB_SCHEMA.md))

### AI & APIs
- **Primary AI**: Anthropic Claude API (claude-sonnet-4-20250514)
- **Telegram**: python-telegram-bot 20.7
- **HTTP Client**: httpx 0.26.0 (async)

### Infrastructure
- **Deployment**: Webhook-based (NO polling)
- **Process Manager**: systemd (ethics-bot.service)
- **Logging**: structlog 24.1.0 (structured JSON)
- **Config**: Pydantic Settings + per-chat JSON в БД

---

## Быстрый старт

### Требования

- Python 3.11+
- MariaDB 10.x
- Telegram Bot Token (от @BotFather)
- Anthropic API Key

### Установка

```bash
# 1. Клонирование/переход в директорию
cd /mnt/d/server_emul/opt/openai/tg_oleg_assist

# 2. Активация venv
source venv/bin/activate

# 3. Установка зависимостей
pip install -r requirements.txt

# 4. Настройка .env
cp .env.example .env
nano .env
# Заполните:
#   TELEGRAM_BOT_TOKEN=...
#   ANTHROPIC_API_KEY=...
#   DB_PASSWORD=...

# 5. Инициализация базы данных
python3 scripts/init_db.py

# 6. Установка webhook
python3 scripts/set_webhook.py

# 7. Запуск сервера
uvicorn app.main:app --reload
```

### Проверка

```bash
# Health check
curl http://localhost:8000/health
# Response: {"status": "ok"}

# Timer worker status
curl http://localhost:8000/timer/status
# Response: {"status": "running", "uptime": 123}

# Webhook status
python3 scripts/check_webhook.py
# Response: {"ok": true, "url": "https://ethics.kadry-24.ru/tg/webhook"}
```

---

## Архитектура

Corporate Ethics Bot построен на **webhook-based асинхронной архитектуре** с multi-chat конфигурацией и встроенным фоновым scheduler.

### Основные компоненты

```
Telegram → POST /tg/webhook
    ↓
MessageRouter (dispatch by mode)
    ├─ ChatContextResolver (load config from DB)
    ├─ EthicsAnalyzer (Claude AI analysis)
    ├─ ScoringEngine (FSM: NORMAL→TENSION→CONFLICT→ESCALATION)
    ├─ MentionService (@mention tracking)
    ├─ ReportService (report reminders)
    ├─ PlanningService (weekly manager plans)
    └─ PlannerService (CEO daily feedback)
```

### Диаграмма архитектуры

```mermaid
graph TB
    subgraph "Entry Point"
        TG[Telegram] --> WH[POST /tg/webhook]
    end

    subgraph "Routing Layer"
        WH --> MR[MessageRouter]
        MR --> CR[ChatContextResolver]
        CR --> DB[("Database\nchats table")]
    end

    subgraph "Business Logic Layer"
        MR --> EA[EthicsAnalyzer]
        MR --> MS[MentionService]
        MR --> RS[ReportService]
        MR --> PS[PlanningService]
        MR --> PL[PlannerService]

        EA --> SE[ScoringEngine]
        SE --> AS[ArbitrationService]
    end

    subgraph "Infrastructure Layer"
        EA --> AI[Claude API]
        PL --> AI
        PS --> AI

        AS --> NS[NotificationService]
        MS --> NS
        RS --> NS
        PL --> NS

        NS --> TG
    end

    subgraph "Background Jobs"
        TW[TimerWorker] --> PL
        TW --> MS
        TW --> RS
    end

    style MR fill:#e1f5ff
    style AI fill:#fff3cd
    style DB fill:#d4edda
```

### Ключевые паттерны

1. **Multi-Chat Configuration** — конфигурация в БД (не .env), per-chat режимы и фичи
2. **Feature Gates** — `ctx.has_feature('planner')` для включения/отключения модулей
3. **FSM in Database** — состояния диалогов в БД (planner_state, dialog_states)
4. **Async Everything** — весь I/O через async/await (DB, HTTP, API calls)
5. **Correlation ID Tracking** — UUID трекинг через весь стек (webhook → services → DB)
6. **Always 200 OK** — webhook не блокирует Telegram retries
7. **Two-Cycle Planner (v1.3)** — два параллельных FSM через `half` field

**Подробнее**: [PROJECT_MAP.md](./PROJECT_MAP.md) — полная карта проекта

---

## База данных

### Обзор

- **База**: `2_kadry_4_ethics_bot`
- **Таблиц**: 21
- **СУБД**: MariaDB 10.x
- **Connection**: `mysql+aiomysql://ethics_user:***@localhost/2_kadry_4_ethics_bot`

### Основные таблицы

| Таблица | Назначение | Ключевые поля |
|---------|-----------|--------------|
| `chats` | Multi-chat configuration | `mode`, `features` (JSON), `config` (JSON) |
| `users` | Пользователи с ролями | `role`, `curator_id` |
| `messages` | История сообщений | `tg_chat_id`, `text`, `correlation_id` |
| `ethics_events` | Нарушения этики | `severity`, `risk_level`, `arbiter_user_id` |
| `planner_feedback` | CEO feedback (v1.3) | UNIQUE(`date`, `user_id`, `half`) |
| `planner_state` | Planner FSM state | UNIQUE(`user_id`, `half`) |
| `mentions` | @mention tracking | `deadline`, `status` |
| `daily_reports` | Ежедневные отчёты | `report_type_id`, `report_date` |
| `dialog_states` | FSM scoring | `state` (NORMAL/TENSION/CONFLICT/ESCALATION) |
| `planning_submissions` | Недельные планы | `week_start`, `week_end` |

### ER-диаграмма (упрощённая)

```mermaid
erDiagram
    chats ||--o{ messages : "contains"
    chats ||--o{ ethics_events : "monitors"
    chats ||--o{ mentions : "tracks"
    chats ||--o{ daily_reports : "receives"

    users ||--o{ messages : "sends"
    users ||--o{ ethics_events : "violates/arbitrates"
    users ||--o{ planner_feedback : "provides"
    users ||--o{ planner_state : "has FSM state"
    users ||--o| users : "curator"

    messages ||--o{ ethics_events : "triggers"
    ethics_events ||--o{ scoring_events : "generates"
    scoring_events ||--o{ dialog_states : "updates"

    planner_feedback ||--o{ planner_analysis_log : "analyzed by AI"
```

**Подробнее**: [DB_SCHEMA.md](./DB_SCHEMA.md) — полная схема всех 21 таблиц

---

## Бизнес-логика

### Основные сценарии

#### 1. Мониторинг этики (compliance mode)

```
Сообщение → Claude AI анализ → Scoring Engine (FSM) → Arbitration Matrix → Уведомления
```

**Потоки**:
- LOW risk → Self-feedback
- MEDIUM risk → Curator notification
- HIGH risk → HR + Director alert

**FSM States**: NORMAL (0-30) → TENSION (30-60) → CONFLICT (60-80) → ESCALATION (80+)

#### 2. CEO Daily Planner (assist mode, v1.3)

**Дневной цикл** (4 точки):

| Время | Действие | Детали |
|-------|---------|--------|
| **09:00** | Morning Info | Полное расписание дня (без кнопок) |
| **14:00** | Morning Feedback Request | Блоки 10:00-14:00 + кнопки [✅⏳❌] |
| **20:30** | Afternoon Feedback Request | Блоки 15:00-19:00 + кнопки [✅⏳❌] |
| **21:30** | End of Day | Auto-close с NO_RESPONSE |

**FSM при нажатии "✅ Выполнено"**:
```
AWAITING_STATUS → AWAITING_QUALITY (1-10) → AWAITING_WORKED (что сработало) →
AWAITING_TO_ADD (что добавить) → AWAITING_PROBLEM (optional) → AWAITING_DECISION (optional) →
SAVE to planner_feedback → Claude AI analysis → Send report
```

**Два параллельных FSM**:
- `planner_state(user_id, half='morning')` — для утреннего цикла
- `planner_state(user_id, half='afternoon')` — для дневного цикла
- UNIQUE(user_id, half) обеспечивает независимость

#### 3. Mention Tracking

```
@username в сообщении → Создание Mention с deadline (+15 мин) →
Timer Worker проверяет каждые 60 сек → Напоминание при overdue
```

#### 4. Report Reminders

```
Детектирование маркера (РОП/ПЛК) → Сохранение DailyReport →
Timer Worker проверяет в check_times (10:45, 14:15, 19:15) → Напоминание при missing
```

#### 5. Weekly Planning

```
#Планирование + недельный план → Claude AI parsing → Project matching (external DB) →
Сохранение в planning_submissions + planning_entries → Подтверждение
```

### Диаграмма потока данных

```mermaid
sequenceDiagram
    participant User as Пользователь
    participant TG as Telegram
    participant Bot as Corporate Ethics Bot
    participant AI as Claude API
    participant DB as MariaDB

    User->>TG: Отправить сообщение
    TG->>Bot: POST /tg/webhook (Update)

    Bot->>DB: Загрузка ChatContext (mode + features)
    DB-->>Bot: ChatContext

    alt mode='compliance'
        Bot->>AI: Анализ этики
        AI-->>Bot: Нарушения
        Bot->>DB: Сохранение ethics_events
        Bot->>DB: Обновление scoring
        Bot->>TG: Уведомления (self/arbiter)
    end

    alt mode='assist' && feature='planner'
        Bot->>DB: Обновление planner_state (FSM)
        Bot->>AI: Анализ feedback (если завершён)
        AI-->>Bot: AI insights
        Bot->>DB: Сохранение planner_feedback
        Bot->>TG: Отправка отчёта CEO
    end

    Bot-->>TG: 200 OK (always)
```

**Подробнее**: [BUSINESS_LOGIC.md](./BUSINESS_LOGIC.md) — все сценарии и паттерны

---

## Конфигурация

### Multi-Chat Configuration System

**Критично**: Конфигурация хранится **В БАЗЕ ДАННЫХ** (таблица `chats`), НЕ в `.env`.

#### Структура конфигурации

```python
ChatContext:
  mode: str                    # viewer | compliance | operator | evaluation | planning | assist
  features: dict               # Feature flags: {"planner": true, "ethics_analysis": true}
  config: dict                 # Parameters: {"mention_deadline_minutes": 15, ...}
```

#### Пример записи в БД

```json
{
  "tg_chat_id": -1001234567890,
  "mode": "compliance",
  "features": {
    "ethics_analysis": true,
    "mention_tracking": true,
    "report_reminders": true,
    "planner": false
  },
  "config": {
    "mention_deadline_minutes": 15,
    "report_check_times": [
      {"hour": 10, "minute": 45},
      {"hour": 14, "minute": 15},
      {"hour": 19, "minute": 15}
    ],
    "ethics_threshold": 0.7
  }
}
```

#### Для CEO Planner чата

```json
{
  "tg_chat_id": -1009876543210,
  "mode": "assist",
  "features": {
    "planner": true
  },
  "config": {
    "planner_ceo_user_id": 987654321,
    "planner_timezone": "Europe/Moscow",
    "planner_eod_hour": 21,
    "planner_eod_minute": 30
  }
}
```

#### Использование в коде

```python
# Проверка feature flag
if ctx.has_feature('planner'):
    await planner_service.process()

# Получение конфига с fallback
deadline_minutes = ctx.get_config(
    'mention_deadline_minutes',
    settings.mention_deadline_minutes  # Fallback на .env
)
```

### Глобальная конфигурация (.env)

```bash
# Telegram
TELEGRAM_BOT_TOKEN=7824716583:AAFmECVBZ...
WEBHOOK_URL=https://ethics.kadry-24.ru/tg/webhook
WEBHOOK_SECRET=your-secret-123

# Database
DB_HOST=localhost
DB_PORT=3306
DB_NAME=2_kadry_4_ethics_bot
DB_USER=ethics_user
DB_PASSWORD=***

# AI
ANTHROPIC_API_KEY=sk-ant-api03-...
ANTHROPIC_MODEL=claude-sonnet-4-20250514
ANTHROPIC_TIMEOUT=30

# Logging
LOG_LEVEL=INFO
```

---

## Development

### Запуск в dev режиме

```bash
# Активация venv
source venv/bin/activate

# Запуск с hot reload
uvicorn app.main:app --reload

# Проверка
curl http://localhost:8000/health
curl http://localhost:8000/timer/status
```

### Тестирование

```bash
# Unit tests (pytest)
python3 -m pytest tests/ -v

# Specific test file
python3 -m pytest tests/test_planner_service_v13.py -v

# Manual tests (когда pytest недоступен)
python3 manual_test_v13.py
python3 test_manual_morning.py

# Syntax check
python3 -m py_compile app/services/planner/planner_service.py
```

### Миграции БД

```bash
# Создать migration
nano migrations/006_description.sql

# Применить вручную
mysql -u root -p 2_kadry_4_ethics_bot < migrations/006_description.sql

# Проверить
mysql -u root -p 2_kadry_4_ethics_bot -e "SHOW TABLES;"
mysql -u root -p 2_kadry_4_ethics_bot -e "DESCRIBE planner_state;"
```

### Работа с webhook

```bash
# Установить webhook
python3 scripts/set_webhook.py

# Проверить webhook
python3 scripts/check_webhook.py

# Удалить webhook (для тестирования)
curl -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/deleteWebhook"
```

---

## Production Deployment

### Systemd Service

```bash
# Статус сервиса
sudo systemctl status ethics-bot

# Перезапуск
sudo systemctl restart ethics-bot

# Логи
sudo journalctl -u ethics-bot -f

# Включить автозапуск
sudo systemctl enable ethics-bot
```

### Deployment Checklist

1. **Остановить сервис**
   ```bash
   sudo systemctl stop ethics-bot
   ```

2. **Применить миграцию БД** (если есть)
   ```bash
   mysql -u root -p 2_kadry_4_ethics_bot < migrations/00X_migration.sql
   ```

3. **Обновить код**
   ```bash
   cd /opt/openai/tg_oleg_assist
   git pull origin main
   ```

4. **Активировать venv и обновить зависимости**
   ```bash
   source venv/bin/activate
   pip install -r requirements.txt
   ```

5. **Запустить сервис**
   ```bash
   sudo systemctl start ethics-bot
   ```

6. **Проверить статус**
   ```bash
   sudo systemctl status ethics-bot
   curl https://ethics.kadry-24.ru/health
   ```

7. **Проверить webhook**
   ```bash
   python3 scripts/check_webhook.py
   ```

8. **Мониторинг логов** (5-10 минут)
   ```bash
   sudo journalctl -u ethics-bot -f
   ```

**Полный checklist**: См. `docs/DEPLOYMENT_CHECKLIST_v1.3.md`

---

## Документация

### Основные документы (этот проект)

- **[PROJECT_MAP.md](./PROJECT_MAP.md)** — Полная карта проекта (файлы + зависимости + модули + конфигурация)
- **[DB_SCHEMA.md](./DB_SCHEMA.md)** — Схема БД (21 таблица + ER-диаграмма + индексы)
- **[BUSINESS_LOGIC.md](./BUSINESS_LOGIC.md)** — Бизнес-логика (5 сценариев + потоки + паттерны)
- **[README_PROJECT.md](./README_PROJECT.md)** — Этот файл (итоговый обзор)

### Дополнительная документация

- **[SPEC_planner_integration.md](./SPEC_planner_integration.md)** — Спецификация CEO Planner v1.3
- **[PHASE6-ARCHITECTURE.md](./PHASE6-ARCHITECTURE.md)** — Multi-chat architecture overview
- **[DEPLOYMENT_CHECKLIST_v1.3.md](../DEPLOYMENT_CHECKLIST_v1.3.md)** — Deployment steps для v1.3
- **[CLAUDE.md](../CLAUDE.md)** — Инструкции для разработчика (Claude Code)

---

## Известные ограничения

### Неиспользуемый код

- ✂️ `app/services/old_ai_client.py` — OpenAI legacy (можно удалить)
- ✂️ `alembic.ini` + `alembic/` — Alembic настроен, но миграции ручные
- ✂️ `app/api/admin.py` — Заглушка, не реализован

### Частично реализованные модули

- ⚠️ **TaskService** — упоминается в timer_worker, но не полностью реализован
- ⚠️ **pytest** — используется в tests/, но не в requirements.txt

### Технические ограничения

1. **Нет idempotency** для webhook → риск duplicate notifications при Telegram retries
2. **AI API без fallback** → если Claude недоступен, операция fails
3. **Mention tracking только @username** → не работает для users без username
4. **TimerWorker interval=60 sec** → точность ±60 секунд для scheduled jobs
5. **Webhook Secret не проверяется** → риск spam/injection

---

## Архитектурные решения

### Почему webhook, а не polling?

✅ **Webhook**:
- Real-time processing (нет задержки)
- Масштабируемость (не держит соединение)
- Lower latency

❌ **Polling**:
- Задержка до 1-2 секунд
- Постоянная нагрузка на Telegram API
- Проблемы с масштабированием

### Почему конфигурация в БД, а не .env?

✅ **БД (chats table)**:
- Per-chat isolation
- Динамическая настройка без деплоя
- Feature toggle per chat
- Audit trail (config history)

❌ **.env**:
- Глобальный (не multi-tenant)
- Требует перезапуска
- Нет версионирования

### Почему FSM в БД, а не Redis?

✅ **БД (planner_state table)**:
- Persistent state (survives restarts)
- ACID transactions
- No external dependencies
- Easy debugging (SQL queries)

❌ **Redis**:
- Extra dependency
- Сложнее debugging
- Нужен отдельный backup

### Почему Always 200 OK?

✅ **Always 200 OK**:
- Prevent Telegram retries для unrecoverable errors
- Graceful degradation
- Simpler error handling

❌ **Return errors**:
- Telegram будет повторять → duplicate processing
- Сложнее обеспечить idempotency

---

## Производительность

### Метрики (оценочные)

| Метрика | Значение | Комментарий |
|---------|---------|-------------|
| Обработка webhook | 50-200 ms | Без AI calls |
| Обработка с AI analysis | 1-3 сек | Claude API latency |
| Concurrent requests | 100+ req/sec | Async architecture |
| DB queries per request | 2-5 | С учётом caching в ChatContext |
| Timer Worker interval | 60 sec | Можно уменьшить до 10 sec |

### Bottlenecks

1. **Claude API** — 1-3 sec latency
   - Решение: Async batch processing, caching для repeated prompts
2. **DB queries** — 10-50 ms per query
   - Решение: Connection pooling (уже используется), индексы
3. **Telegram API** — 100-500 ms per message
   - Решение: Bulk operations где возможно

---

## Мониторинг и логирование

### Structured Logging (structlog)

```json
{
  "event": "Ethics violation detected",
  "correlation_id": "abc-123",
  "user_id": 12345678,
  "chat_id": -1001234567890,
  "severity": "MEDIUM",
  "risk_level": "MEDIUM",
  "timestamp": "2026-02-11T15:30:00Z"
}
```

### Ключевые метрики для мониторинга

- `ethics_events` — количество нарушений per day
- `dialog_states` — распределение состояний (NORMAL/TENSION/CONFLICT/ESCALATION)
- `mentions` — overdue rate
- `daily_reports` — submission rate
- `planner_feedback` — completion rate (held vs no_response)
- `notifications` — success rate

### Алерты (рекомендуемые)

- ⚠️ `dialog_states.state = 'ESCALATION'` → immediate alert
- ⚠️ `planner_feedback.meeting_status = 'no_response'` → daily summary
- ⚠️ `mentions overdue > 5` в чате → alert to admin
- ⚠️ API errors > 10% → alert

---

## FAQ

### Как добавить новый чат?

```sql
INSERT INTO chats (tg_chat_id, mode, features, config, is_active)
VALUES (
  -1001234567890,
  'compliance',
  '{"ethics_analysis": true, "mention_tracking": true}',
  '{"mention_deadline_minutes": 15}',
  true
);
```

### Как изменить режим чата?

```sql
UPDATE chats SET mode = 'assist' WHERE tg_chat_id = -1001234567890;
```

### Как включить/отключить feature?

```sql
UPDATE chats
SET features = JSON_SET(features, '$.planner', true)
WHERE tg_chat_id = -1001234567890;
```

### Как посмотреть логи конкретного запроса?

```sql
SELECT * FROM audit_log WHERE correlation_id = 'abc-123';
SELECT * FROM ethics_events WHERE correlation_id = 'abc-123';
SELECT * FROM mentions WHERE correlation_id = 'abc-123';
```

### Как очистить старые данные?

```sql
-- Messages старше 90 дней
DELETE FROM messages WHERE sent_at < DATE_SUB(NOW(), INTERVAL 90 DAY);

-- Audit log старше 180 дней
DELETE FROM audit_log WHERE created_at < DATE_SUB(NOW(), INTERVAL 180 DAY);

-- Expired planner states
DELETE FROM planner_state WHERE expires_at < NOW();
```

### Как протестировать planner локально?

```bash
# Manual test script
python3 manual_test_v13.py

# Unit tests
python3 -m pytest tests/test_planner_service_v13.py -v
```

---

## Контакты и поддержка

### Полезные ссылки

- **GitHub Issues**: (добавить ссылку на репозиторий)
- **Documentation**: `/mnt/d/server_emul/opt/openai/tg_oleg_assist/docs/`
- **Production URL**: https://ethics.kadry-24.ru/

### Команда

- **Разработка**: (указать контакт)
- **Deployment**: (указать контакт)
- **Database**: (указать контакт)

---

## История версий

### v1.3 (11.02.2026) — Planner Two-Cycle

**Изменения**:
- ✨ Добавлено поле `half` в `planner_state` и `planner_feedback`
- ✨ UNIQUE constraints изменены: `(user_id, half)` и `(date, user_id, half)`
- ✨ Два параллельных FSM для morning и afternoon
- ✨ Новые кнопки: "✅ Выполнено / ⏳ Частично / ❌ Не удалось"
- ✨ Убрано 17:30 reminder, добавлено 09:00 informational message
- ✨ EOD перенесён с 20:00 на 21:30

**Migration**: `migrations/005_planner_v1.3.sql`

### v1.2 (январь 2026) — Planning Service

- ✨ Добавлено недельное планирование менеджеров
- ✨ AI-matching проектов с внешней БД
- ✨ Таблицы: `planning_managers`, `planning_submissions`, `planning_entries`

### v1.1 (декабрь 2025) — Report Reminders

- ✨ Детектирование отчётов (РОП, ПЛК)
- ✨ Напоминания в check_times
- ✨ Таблицы: `report_types`, `report_responsible`, `daily_reports`

### v1.0 (ноябрь 2025) — Initial Release

- ✨ Multi-chat configuration
- ✨ Ethics monitoring с Claude AI
- ✨ Mention tracking
- ✨ CEO Planner v1.0 (single cycle)
- ✨ Scoring FSM (NORMAL→ESCALATION)

---

## Лицензия

(Указать лицензию проекта)

---

**Документация актуальна на**: 11.02.2026

**Версия**: v1.3

**Статус**: ✅ Production Ready
