# Карта проекта Corporate Ethics Bot

## 1. Структура проекта

### 1.1 Дерево файлов

```
tg_oleg_assist/
├── app/                              # Основное приложение (FastAPI)
│   ├── main.py                       # FastAPI app + lifespan (Timer Worker) [359 строк]
│   ├── config.py                     # Pydantic Settings (из .env) [147 строк]
│   ├── database.py                   # SQLAlchemy async engine + sessions [51 строка]
│   │
│   ├── api/                          # REST API endpoints
│   │   ├── webhook.py                # POST /tg/webhook (главная точка входа) [111 строк]
│   │   └── admin.py                  # Admin endpoints (заглушка) [28 строк]
│   │
│   ├── models/                       # SQLAlchemy ORM модели (~692 строки)
│   │   ├── __init__.py              # Экспорт всех моделей
│   │   ├── chat.py                   # Чаты (mode + features + config JSON)
│   │   ├── user.py                   # Пользователи с ролями
│   │   ├── message.py                # История сообщений
│   │   ├── ethics_event.py           # Нарушения этики
│   │   ├── planner.py                # Planner v1.3 (Feedback + State + AnalysisLog)
│   │   ├── mention.py                # @mention tracking
│   │   ├── notification.py           # Логи отправленных уведомлений
│   │   ├── daily_report.py           # Ежедневные отчёты
│   │   ├── task.py                   # Задачи с дедлайнами
│   │   ├── scoring.py                # Scoring events
│   │   ├── dialog_state.py           # FSM состояния диалога
│   │   ├── state_transition.py       # История переходов состояний
│   │   ├── planning.py               # Недельное планирование менеджеров
│   │   ├── report_type.py            # Типы отчётов (РОП, ПЛК)
│   │   ├── org_unit.py               # Организационные единицы
│   │   └── audit_log.py              # Аудит событий
│   │
│   ├── schemas/                      # Pydantic request/response схемы
│   │   └── webhook.py                # Telegram Update schemas
│   │
│   ├── services/                     # Бизнес-логика (~4810 строк)
│   │   ├── message_router.py         # Главный диспетчер по chat mode [669 строк]
│   │   ├── chat_context.py           # ChatContext resolver [204 строки]
│   │   ├── ai_client.py              # Anthropic Claude API [204 строки]
│   │   ├── old_ai_client.py          # Legacy OpenAI (можно удалить) [189 строк]
│   │   ├── ethics_analyzer.py        # Анализ нарушений этики [172 строки]
│   │   ├── scoring_engine.py         # FSM scoring (NORMAL→ESCALATION) [322 строки]
│   │   ├── arbitration.py            # Определение арбитра по матрице [285 строк]
│   │   ├── notification.py           # Telegram sender [618 строк]
│   │   ├── mention_service.py        # @mention tracking [256 строк]
│   │   ├── report_service.py         # Report reminders [403 строки]
│   │   ├── planning_service.py       # Недельные планы менеджеров [626 строк]
│   │   ├── timer_worker.py           # Background scheduler [629 строк]
│   │   │
│   │   └── planner/                  # CEO Planner subsystem (~1531 строка)
│   │       ├── planner_service.py    # FSM + two-cycle logic [631 строка]
│   │       ├── planner_formatter.py  # HTML formatting [218 строк]
│   │       ├── planner_day_themes.py # Daily schedules [539 строк]
│   │       └── planner_prompts.py    # Claude prompts [138 строк]
│   │
│   ├── prompts/                      # AI prompts
│   │   └── ethics_analysis.yaml      # Ethics analysis prompt
│   │
│   └── rules/                        # Ethics rules в YAML
│       ├── 01_respect.yaml           # Правила уважения
│       ├── 02_communication.yaml     # Коммуникация
│       ├── 03_conflict.yaml          # Конфликты
│       ├── 04_work_process.yaml      # Рабочие процессы
│       └── 05_responsibility.yaml    # Ответственность
│
├── migrations/                       # SQL миграции (ручные, не Alembic)
│   ├── 001_initial_schema.sql        # Начальная схема
│   ├── 002_add_planner.sql           # Planner v1.0
│   ├── 003_planner_fixes.sql         # Planner fixes
│   ├── 004_planning_service.sql      # Planning service
│   └── 005_planner_v1.3.sql          # Planner v1.3 (two-cycle)
│
├── scripts/                          # Утилиты
│   ├── init_db.py                    # Инициализация БД
│   ├── set_webhook.py                # Установка Telegram webhook
│   ├── check_webhook.py              # Проверка webhook
│   └── test_*.py                     # Тестовые скрипты
│
├── tests/                            # Тесты (pytest)
│   ├── test_planner_service_v13.py   # Тесты Planner v1.3
│   └── ...
│
├── docs/                             # Документация
│   ├── SPEC_planner_integration.md   # Спецификация Planner v1.3
│   ├── PHASE6-ARCHITECTURE.md        # Multi-chat architecture
│   ├── DEPLOYMENT_CHECKLIST_v1.3.md  # Deployment steps
│   ├── PROJECT_MAP.md                # Этот файл
│   ├── DB_SCHEMA.md                  # Схема БД
│   ├── BUSINESS_LOGIC.md             # Бизнес-логика
│   └── README_PROJECT.md             # Итоговый обзор
│
├── CLAUDE.md                         # Инструкции для Claude Code
├── CLAUDE.local.md                   # Локальные настройки (не в git)
├── .env.example                      # Шаблон переменных окружения
├── .env                              # Реальные переменные (не в git)
├── requirements.txt                  # Python зависимости
├── alembic.ini                       # Alembic config (не используется)
├── .gitignore                        # Git ignore правила
└── venv/                             # Python virtual environment
```

### 1.2 Размеры ключевых модулей

| Модуль | Строк кода | Назначение |
|--------|-----------|-----------|
| `app/services/message_router.py` | 669 | Главный диспетчер |
| `app/services/planner/planner_service.py` | 631 | CEO Planner FSM |
| `app/services/timer_worker.py` | 629 | Фоновый scheduler |
| `app/services/planning_service.py` | 626 | Недельное планирование |
| `app/services/notification.py` | 618 | Telegram sender |
| `app/services/planner/planner_day_themes.py` | 539 | Daily schedules |
| `app/services/report_service.py` | 403 | Report reminders |
| `app/main.py` | 359 | FastAPI app |
| `app/services/scoring_engine.py` | 322 | FSM scoring |
| `app/services/arbitration.py` | 285 | Arbitration logic |
| `app/services/mention_service.py` | 256 | Mention tracking |
| `app/services/planner/planner_formatter.py` | 218 | HTML formatting |
| `app/services/chat_context.py` | 204 | Context resolver |
| `app/services/ai_client.py` | 204 | Claude API |
| `app/services/ethics_analyzer.py` | 172 | Ethics analysis |
| `app/services/planner/planner_prompts.py` | 138 | AI prompts |
| **Всего services/** | **~4810** | |
| **Всего models/** | **~692** | |

---

## 2. Зависимости

### 2.1 Основные зависимости (requirements.txt)

```
# Web Framework
fastapi==0.109.0              # Async web framework
uvicorn[standard]==0.27.0     # ASGI server
pydantic-settings==2.1.0      # Settings management

# Database
sqlalchemy==2.0.25            # Async ORM
aiomysql==0.2.0              # MySQL async driver
pymysql==1.1.0               # MySQL sync (для скриптов)

# AI Providers
anthropic>=0.42.0            # Claude API (основной)
openai>=1.0.0                # OpenAI API (legacy)

# Telegram
python-telegram-bot==20.7    # Telegram Bot API

# HTTP Client
httpx==0.26.0                # Async HTTP client
tenacity==8.2.3              # Retry with backoff

# Utils
structlog==24.1.0            # Structured logging
pyyaml==6.0.1                # YAML parsing (rules)
python-dotenv==1.0.0         # .env loader
pytz                         # Timezone handling
```

### 2.2 Dev зависимости (используются, но не в requirements.txt)

```
pytest                       # Unit testing
pytest-asyncio               # Async test support
```

### 2.3 Системные зависимости

- **Python**: 3.11+
- **MariaDB**: 10.x
- **OS**: Linux (WSL2 в dev, Ubuntu в production)
- **Systemd**: для production services

---

## 3. Точки входа

### 3.1 Главная точка входа

**Файл**: `app/main.py`

**Запуск**:
```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

**Production (systemd)**:
```ini
# /etc/systemd/system/ethics-bot.service
[Service]
ExecStart=/opt/openai/tg_oleg_assist/venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8000
```

### 3.2 API Endpoints

| Endpoint | Метод | Назначение |
|----------|-------|-----------|
| `/tg/webhook` | POST | Получение Telegram updates (главный вход) |
| `/health` | GET | Health check (200 OK) |
| `/timer/status` | GET | Статус Timer Worker |
| `/admin/*` | * | Admin endpoints (заглушка) |

### 3.3 FastAPI Lifespan

Timer Worker запускается в lifespan context:

```python
# app/main.py
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: запуск Timer Worker
    timer_worker = TimerWorker()
    timer_task = asyncio.create_task(timer_worker.run())

    yield  # Приложение работает

    # Shutdown: остановка Timer Worker
    timer_worker.stop()
    await timer_task
```

### 3.4 Обработка запросов

```
HTTP Request → Uvicorn (ASGI)
    ↓
FastAPI routing
    ↓
POST /tg/webhook → webhook.py:handle_telegram_update()
    ↓
MessageRouter.process(update)
    ↓
Dispatch по chat mode:
    - viewer: логирование
    - compliance: ethics + scoring + mention + report
    - operator: (зарезервировано)
    - evaluation: (зарезервировано)
    - planning: weekly plans
    - assist: CEO planner
    ↓
Response 200 OK (всегда, даже при ошибках)
```

---

## 4. Модули и связи

### 4.1 Архитектурная схема

**Часть 1: Слои и компоненты**

```mermaid
graph TB
    subgraph "API Layer"
        WH["webhook.py\nPOST /tg/webhook"]
        ADMIN["admin.py\nAdmin API"]
    end

    subgraph "Core"
        MAIN["main.py\nFastAPI app + lifespan"]
        CONFIG["config.py\nSettings"]
        DB["database.py\nSQLAlchemy engine"]
    end

    subgraph "Routing & Context"
        ROUTER["MessageRouter\nDispatch by mode"]
        RESOLVER["ChatContextResolver\nLoad config from DB"]
        CTX["ChatContext\nmode + features + config"]
    end

    subgraph "Services - Ethics & Scoring"
        ETHICS["EthicsAnalyzer\nClaude API"]
        SCORING["ScoringEngine\nFSM scoring"]
        ARBITER["ArbitrationService\nArbiter selection"]
        RULES["RulesLoader\nYAML rules"]
    end

    subgraph "Services - Tracking"
        MENTION["MentionService\n@mention tracking"]
        REPORT["ReportService\nReport reminders"]
        PLANNING["PlanningService\nWeekly plans"]
    end

    subgraph "Services - CEO Planner"
        PLANNER["PlannerService\nFSM two-cycle"]
        FORMATTER["PlannerFormatter\nHTML format"]
        THEMES["PlannerDayThemes\nSchedules"]
        PROMPTS["PlannerPrompts\nAI prompts"]
    end

    subgraph "Services - Infrastructure"
        NOTIF["NotificationService\nTelegram sender"]
        AI["AIClient\nAnthropic API"]
        TIMER["TimerWorker\nBackground scheduler"]
    end

    style ROUTER fill:#e1f5ff
    style AI fill:#fff3cd
    style DB fill:#d4edda
```

**Часть 2: Связи между компонентами**

```mermaid
graph TB
    WH["webhook.py"]
    ROUTER["MessageRouter"]
    RESOLVER["ChatContextResolver"]
    CTX["ChatContext"]

    ETHICS["EthicsAnalyzer"]
    SCORING["ScoringEngine"]
    ARBITER["ArbitrationService"]
    RULES["RulesLoader"]
    MENTION["MentionService"]
    REPORT["ReportService"]
    PLANNING["PlanningService"]
    PLANNER["PlannerService"]

    NOTIF["NotificationService"]
    AI["AIClient"]
    TIMER["TimerWorker"]

    M_CHAT[("Chat")]
    M_USER[("User")]
    M_MSG[("Message")]
    M_ETHICS[("EthicsEvent")]
    M_PLANNER[("PlannerFeedback\nPlannerState")]
    M_MENTION[("Mention")]
    M_REPORT[("DailyReport")]

    MAIN["main.py"]
    DB["database.py"]
    TG["Telegram"]

    WH --> ROUTER
    ROUTER --> RESOLVER
    RESOLVER --> M_CHAT
    RESOLVER --> CTX

    ROUTER --> ETHICS
    ROUTER --> SCORING
    ROUTER --> MENTION
    ROUTER --> REPORT
    ROUTER --> PLANNING
    ROUTER --> PLANNER

    ETHICS --> AI
    ETHICS --> RULES
    ETHICS --> SCORING
    SCORING --> ARBITER
    ARBITER --> NOTIF

    PLANNER --> AI
    PLANNER --> NOTIF
    PLANNER --> M_PLANNER

    MENTION --> NOTIF
    MENTION --> M_MENTION
    REPORT --> NOTIF
    REPORT --> M_REPORT

    NOTIF --> TG

    TIMER --> PLANNER
    TIMER --> MENTION
    TIMER --> REPORT

    MAIN --> TIMER
    MAIN --> DB
    DB --> M_CHAT
    DB --> M_USER
    DB --> M_MSG
```

### 4.2 Граф зависимостей модулей

```mermaid
graph LR
    subgraph "Entry Point"
        WH[webhook.py]
    end

    subgraph "Dispatcher"
        MR[MessageRouter]
        CR[ChatContextResolver]
    end

    subgraph "Business Logic"
        EA[EthicsAnalyzer]
        SE[ScoringEngine]
        AS[ArbitrationService]
        MS[MentionService]
        RS[ReportService]
        PS[PlanningService]
        PL[PlannerService]
    end

    subgraph "Infrastructure"
        NS[NotificationService]
        AI[AIClient]
        TW[TimerWorker]
    end

    WH --> MR
    MR --> CR
    CR --> |load config| DB[(Database)]

    MR --> EA
    MR --> SE
    MR --> MS
    MR --> RS
    MR --> PS
    MR --> PL

    EA --> AI
    EA --> SE
    SE --> AS
    AS --> NS

    PL --> AI
    PL --> NS

    MS --> NS
    RS --> NS
    PS --> NS

    TW --> PL
    TW --> MS
    TW --> RS

    NS --> |HTTP| TG[Telegram API]
    AI --> |HTTP| CLAUDE[Claude API]
```

### 4.3 Взаимодействие сервисов

**MessageRouter** — центральный диспетчер:
- Получает Telegram Update
- Резолвит ChatContext через ChatContextResolver
- Диспетчеризует по `ctx.mode`:
  - `viewer`: только логирование
  - `compliance`: ethics + scoring + mention + report
  - `operator`: (зарезервировано)
  - `evaluation`: (зарезервировано)
  - `planning`: weekly plans
  - `assist`: CEO planner
- Проверяет feature flags: `ctx.has_feature('planner')`

**ChatContext** — конфигурация per-chat:
```python
@dataclass
class ChatContext:
    tg_chat_id: int
    mode: str                    # viewer | compliance | ...
    features: dict               # {"planner": true, ...}
    config: dict                 # {"planner_eod_hour": 21, ...}
    org_unit_id: Optional[int]

    def has_feature(self, name: str) -> bool:
        return self.features.get(name, False)

    def get_config(self, key: str, default=None):
        return self.config.get(key, default)
```

**Сервисы используют ChatContext**:
```python
# В каждом сервисе:
class PlannerService:
    def __init__(self, ctx: ChatContext, db_session, correlation_id):
        self.ctx = ctx
        self.db = db_session
        self.cid = correlation_id

        # Получение конфига:
        self.eod_hour = ctx.get_config('planner_eod_hour', 21)
        self.ceo_id = ctx.get_config('planner_ceo_user_id')
```

### 4.4 Паттерн обработки сообщения

```python
# Упрощённый псевдокод
async def process_message(update: Update):
    # 1. Extract data
    chat_id = update.message.chat.id
    user_id = update.message.from_user.id
    text = update.message.text

    # 2. Resolve context
    ctx = await ChatContextResolver.resolve(chat_id, db)

    # 3. Save message to DB
    await save_message(chat_id, user_id, text, db)

    # 4. Dispatch by mode
    if ctx.mode == 'compliance':
        if ctx.has_feature('ethics_analysis'):
            await EthicsAnalyzer(ctx, db, cid).analyze(text)
        if ctx.has_feature('mention_tracking'):
            await MentionService(ctx, db, cid).process(update)
        if ctx.has_feature('report_reminders'):
            await ReportService(ctx, db, cid).process(update)

    elif ctx.mode == 'assist':
        if ctx.has_feature('planner'):
            await PlannerService(ctx, db, cid).handle_message(update)

    elif ctx.mode == 'planning':
        await PlanningService(ctx, db, cid).process(update)
```

---

## 5. Конфигурация

### 5.1 Приоритет конфигурации

1. **Global defaults** — `app/config.py` (из `.env`)
2. **Per-chat config** — `chats.config` JSON в БД
3. **Feature flags** — `chats.features` JSON в БД

### 5.2 Глобальная конфигурация (.env)

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
DB_PASSWORD=CC7zX1FhQ7Xe7xgRoUFA

# AI
ANTHROPIC_API_KEY=sk-ant-api03-...
ANTHROPIC_MODEL=claude-sonnet-4-20250514
ANTHROPIC_TIMEOUT=30

# Legacy (можно удалить)
OPENAI_API_KEY=sk-proj-...
OPENAI_MODEL=gpt-4o

# Logging
LOG_LEVEL=INFO
```

### 5.3 Per-Chat конфигурация (в БД)

**Таблица `chats`**:

```sql
CREATE TABLE chats (
    id BIGINT PRIMARY KEY,
    tg_chat_id BIGINT UNIQUE,
    mode VARCHAR(20) DEFAULT 'viewer',  -- viewer|compliance|operator|evaluation|planning|assist
    features JSON,                      -- Feature flags
    config JSON,                        -- Per-chat parameters
    org_unit_id INT,
    is_active BOOLEAN DEFAULT TRUE
);
```

**Пример записи**:

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

**Для CEO Planner чата**:

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

### 5.4 Использование конфигурации в коде

```python
# В любом сервисе
class SomeService:
    def __init__(self, ctx: ChatContext, db_session, correlation_id):
        # Получение с fallback на глобальный default
        deadline = ctx.get_config(
            'mention_deadline_minutes',
            settings.mention_deadline_minutes  # из .env
        )

        # Проверка feature flag
        if ctx.has_feature('planner'):
            await self._run_planner_logic()
```

---

## 6. Ключевые паттерны

### 6.1 Async Everything

❌ **НИКОГДА**:
```python
import requests
import pymysql
import time

response = requests.get(url)         # Блокирующий вызов
db = pymysql.connect(...)            # Блокирующий
time.sleep(60)                       # Блокирующий
```

✅ **ВСЕГДА**:
```python
import httpx
import aiomysql
import asyncio

async with httpx.AsyncClient() as client:
    response = await client.get(url)

async with get_db_session() as db:
    result = await db.execute(query)

await asyncio.sleep(60)
```

### 6.2 Correlation ID Tracking

```python
# Генерация в webhook.py
correlation_id = str(uuid.uuid4())
log = logger.bind(correlation_id=correlation_id)

# Передача через весь стек
router = MessageRouter(db, correlation_id)
service = EthicsAnalyzer(ctx, db, correlation_id)

# Логирование
self.log.info("Event occurred", user_id=123, event_type="violation")
# Output: {"event": "Event occurred", "correlation_id": "abc-123", "user_id": 123, ...}

# Сохранение в БД
ethics_event.correlation_id = correlation_id
```

### 6.3 Always 200 OK to Telegram

```python
@router.post("/tg/webhook")
async def handle_telegram_update(request: Request):
    try:
        update = await request.json()
        await process_update(update)
    except Exception as e:
        logger.error("Processing failed", error=str(e), exc_info=True)
        # НЕ raise! Telegram будет повторять запрос

    return {"ok": True}  # ВСЕГДА 200 OK
```

### 6.4 Feature-Gated Services

```python
# В MessageRouter
if ctx.has_feature('mention_tracking'):
    await MentionService(ctx, db, cid).process(update)

if ctx.has_feature('planner'):
    await PlannerService(ctx, db, cid).handle_callback(callback_query)
```

### 6.5 Session Management

```python
# Правильно: context manager
async with get_db_session() as db:
    result = await db.execute(query)
    await db.commit()
# Автоматический rollback при ошибке

# Неправильно: ручное управление (риск утечек)
db = await get_session()
try:
    result = await db.execute(query)
    await db.commit()
finally:
    await db.close()
```

### 6.6 Structured Logging

```python
import structlog

logger = structlog.get_logger()
log = logger.bind(correlation_id=cid, user_id=user_id)

log.info("Processing started", chat_id=chat_id, mode=ctx.mode)
log.warning("Deadline exceeded", mention_id=123, deadline=datetime)
log.error("API call failed", provider="anthropic", status_code=500)
```

---

## 7. Внешние интеграции

### 7.1 Telegram Bot API

**Библиотека**: `python-telegram-bot 20.7`

**Основные операции**:

```python
from telegram import Bot, InlineKeyboardMarkup, InlineKeyboardButton

bot = Bot(token=settings.telegram_bot_token)

# Отправка сообщения
await bot.send_message(
    chat_id=chat_id,
    text="Текст сообщения",
    parse_mode="HTML"
)

# Отправка с кнопками
keyboard = InlineKeyboardMarkup([
    [InlineKeyboardButton("✅ Да", callback_data="yes")],
    [InlineKeyboardButton("❌ Нет", callback_data="no")]
])
await bot.send_message(chat_id, text, reply_markup=keyboard)

# Ответ на callback
await bot.answer_callback_query(callback_query_id)
await bot.edit_message_text(chat_id, message_id, new_text)
```

### 7.2 Anthropic Claude API

**Библиотека**: `anthropic >= 0.42.0`

**Модель**: `claude-sonnet-4-20250514`

**Использование через AIClient**:

```python
# app/services/ai_client.py
from anthropic import AsyncAnthropic

class AIClient:
    def __init__(self):
        self.client = AsyncAnthropic(api_key=settings.anthropic_api_key)

    async def analyze(
        self,
        system_prompt: str,
        user_prompt: str,
        response_format: Optional[dict] = None,
        temperature: float = 0.7,
        max_tokens: int = 2000
    ) -> str:
        messages = [{"role": "user", "content": user_prompt}]

        response = await self.client.messages.create(
            model=settings.anthropic_model,
            system=system_prompt,
            messages=messages,
            temperature=temperature,
            max_tokens=max_tokens,
            timeout=settings.anthropic_timeout
        )

        return response.content[0].text
```

**Use cases**:
1. **EthicsAnalyzer** — анализ нарушений этики
2. **PlannerService** — анализ feedback CEO
3. **PlanningService** — matching проектов

### 7.3 MariaDB

**База**: `2_kadry_4_ethics_bot`

**Драйвер**: `aiomysql` (async)

**ORM**: SQLAlchemy 2.0

**Connection string**:
```
mysql+aiomysql://ethics_user:CC7zX1FhQ7Xe7xgRoUFA@localhost/2_kadry_4_ethics_bot
```

**Session pattern**:
```python
async with get_db_session() as db:
    result = await db.execute(
        select(Chat).where(Chat.tg_chat_id == chat_id)
    )
    chat = result.scalar_one_or_none()
```

---

## 8. Известные проблемы и TODO

### 8.1 Legacy код (можно удалить)

- ✂️ `app/services/old_ai_client.py` — OpenAI integration (заменён на Anthropic)
- ✂️ `alembic.ini` + `alembic/` — Alembic настроен, но миграции ручные (SQL файлы)

### 8.2 Незавершённые модули

- ⚠️ `app/api/admin.py` — заглушка, не реализован
- ⚠️ TaskService — упоминается в timer_worker, но не полностью реализован

### 8.3 Технический долг

- 📋 pytest используется в `tests/`, но не в `requirements.txt`
- 📋 Systemd services упоминаются в CLAUDE.md, но файлы не найдены в проекте
- 📋 Некоторые migration файлы содержат IF EXISTS, но MariaDB не всегда поддерживает

### 8.4 Документация

✅ **Актуально**:
- `CLAUDE.md` — Полные инструкции для разработчика
- `docs/SPEC_planner_integration.md` — Spec Planner v1.3
- `docs/PHASE6-ARCHITECTURE.md` — Multi-chat architecture

📝 **Создано в рамках этого анализа**:
- `docs/PROJECT_MAP.md` — Этот файл
- `docs/DB_SCHEMA.md` — Схема БД
- `docs/BUSINESS_LOGIC.md` — Бизнес-логика
- `docs/README_PROJECT.md` — Итоговый обзор

---

## 9. Deployment

### 9.1 Production Environment

- **OS**: Ubuntu 22.04 LTS
- **Python**: 3.11+
- **MariaDB**: 10.x
- **Web Server**: Uvicorn (за Nginx reverse proxy)
- **Process Manager**: systemd

### 9.2 Systemd Services

**ethics-bot.service** (основной сервис):
```ini
[Unit]
Description=Ethics Bot Webhook Server
After=network.target mariadb.service

[Service]
Type=simple
User=ethics
WorkingDirectory=/opt/openai/tg_oleg_assist
Environment="PATH=/opt/openai/tg_oleg_assist/venv/bin"
ExecStart=/opt/openai/tg_oleg_assist/venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8000
Restart=always

[Install]
WantedBy=multi-user.target
```

### 9.3 Deployment Checklist

См. `docs/DEPLOYMENT_CHECKLIST_v1.3.md` для полного чеклиста.

**Основные шаги**:
1. Остановить systemd service
2. Применить SQL migration
3. Обновить код (git pull)
4. Перезапустить service
5. Проверить логи
6. Проверить webhook status

---

## 10. Development Workflow

### 10.1 Локальная разработка

```bash
# Активация venv
source venv/bin/activate

# Запуск с hot reload
uvicorn app.main:app --reload

# Проверка
curl http://localhost:8000/health
curl http://localhost:8000/timer/status

# Проверка webhook
python3 scripts/check_webhook.py
```

### 10.2 Тестирование

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

### 10.3 Миграции БД

```bash
# Создать migration
nano migrations/006_description.sql

# Применить
mysql -u root -p 2_kadry_4_ethics_bot < migrations/006_description.sql

# Проверить
mysql -u root -p 2_kadry_4_ethics_bot -e "SHOW TABLES;"
mysql -u root -p 2_kadry_4_ethics_bot -e "DESCRIBE planner_state;"
```

### 10.4 Git Workflow

```bash
# Feature branch
git checkout -b feature/new-feature

# Commit
git add .
git commit -m "feat: add new feature"

# Push
git push origin feature/new-feature

# После merge в main
cd /opt/openai/tg_oleg_assist
git pull origin main
sudo systemctl restart ethics-bot
```

---

## Заключение

Corporate Ethics Bot — сложная система с чёткой модульной архитектурой. Основные принципы:

1. **Async Everything** — весь I/O через async/await
2. **Multi-Chat Config** — конфигурация в БД, не в .env
3. **Feature Gates** — модули включаются per-chat
4. **Correlation ID** — трекинг через весь стек
5. **Always 200 OK** — webhook не блокирует Telegram

Код хорошо структурирован, с чёткой separation of concerns и минимальным техническим долгом.
