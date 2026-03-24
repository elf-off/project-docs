# Project Summary -- CRM Avito AI Worker

Data: 2026-03-24

## Struktura proekta (derevo .py faylov)

```
crm-worker/                          6606 strok Python
|
|-- main.py                           198  FastAPI entry point, lifespan, APScheduler
|-- config.py                          93  Pydantic Settings (.env)
|-- requirements.txt                   13  zavisimosti
|
|-- api/
|   |-- __init__.py                     0
|   |-- webhooks.py                   554  Avito webhook endpoint (new_application / new_message)
|   |-- admin.py                      848  Admin REST API (applicants, chats, handover, emulator)
|   |-- admin_web.py                  120  Web UI (login, cookie-sessions, SPA)
|   |-- oauth.py                      101  OAuth2 token exchange endpoints
|
|-- models/
|   |-- __init__.py                     0
|   |-- db.py                         266  13 SQLAlchemy ORM modelej, engine, session factory
|
|-- services/
|   |-- __init__.py                     0
|   |-- ai_agent.py                  1315  Dialogovyj state machine (6 blokov, 9+ stages)
|   |-- ai_claude.py                  311  Claude API wrapper + retry 5x + fallback OpenAI
|   |-- ai_rag.py                     269  Qdrant vector search + OpenAI embeddings
|   |-- avito_auth.py                 233  OAuth2 token management per account
|   |-- avito_applications.py         213  Avito Applications API (fetch details, resume)
|   |-- avito_messenger.py            169  Send/receive messages via Avito Messenger
|   |-- redis_queue.py                203  Redis Streams wrapper (enqueue, consume, ack)
|   |-- vacancy_sync.py               297  Vacancy polling Avito API -> DB + Qdrant
|   |-- vacancy_parser.py             164  Address extraction (regexp + AI fallback)
|   |-- handover.py                   161  Handover card creation for operators
|   |-- telegram_notifier.py          156  Morning summary in Telegram (per account)
|   |-- segmentation.py               153  Block classification (1/2/3)
|   |-- message_scheduler.py          144  Delayed message queue with human-like pauses
|   |-- bitrix_agent.py                45  Bitrix CRM agent on/off toggle
|   |-- add_refresh_token.py           33  Utility: add refresh token to account
|
|-- workers/
|   |-- __init__.py                     0
|   |-- incoming_processor.py         148  Incoming message handler + 10s debounce
|   |-- webhook_consumer.py           133  Redis Streams consumer (async background)
|   |-- token_refresher.py            18   Background token refresh job (every 30 min)
|
|-- utils/
|   |-- __init__.py                     0
|   |-- time_helpers.py               106  Night window check, adaptive delays
|   |-- logger.py                      30  structlog config (JSON)
|   |-- event_logger.py                27  Event log helper
|
|-- scripts/
|   |-- webhook_enable_ours.py         49  Switch to our webhook + disable Bitrix
|   |-- webhook_enable_bitrix.py       49  Switch to Bitrix webhook + enable agents
|
|-- prompts/                          11 faylov, ~176 strok
|   |-- system.txt                        Master system prompt (persona Elena)
|   |-- greeting.txt                      Initial greeting
|   |-- qualification.txt                 Qualification stage
|   |-- presentation.txt                  Vacancy presentation
|   |-- alternatives.txt                  Alternative vacancies
|   |-- booking.txt                       Callback booking
|   |-- followup.txt                      Follow-up after silence
|   |-- objection.txt                     Objection handling
|   |-- clarify.txt                       Clarification
|   |-- asks_phone.txt                    Response when candidate asks for phone
|   |-- no_phone.txt                      Response when candidate refuses phone
|
|-- migrations/                       9 SQL faylov, ~391 strok
|   |-- schema_v1.sql                 177  Initial schema
|   |-- schema_v2_migration.sql        76  V2 migration
|   |-- schema_v2_phase2_migration.sql 63  V2 phase 2
|   |-- 003_multi_account.sql          15  Multi-account support
|   |-- 004_event_log.sql              12  Event audit table
|   |-- 005_telegram_topic.sql          4  telegram_topic_id column
|   |-- 007_ai_enabled.sql              8  AI enable flag
|   |-- 008_add_vacancy_accounts.sql    6  Vacancy-to-account link
|   |-- 009_alternatives_criteria_stage.sql 10  New dialog stage
|
|-- templates/
|   |-- admin.html                        SPA admin panel
|
|-- docs/                             60+ faylov dokumentacii
    |-- SPEC.md                           Full specification
    |-- tasks/TASK-01..TASK-27.md         Task specs
    |-- dev/*.md                          Deployment notes
    |-- audit/                            Audit reports
    |-- man/                              Status snapshots
```

## Razmer kodovoj bazy

| Metrika | Znachenie |
|---------|-----------|
| Python fajlov (bez venv) | 35 |
| Strok Python koda | 6 606 |
| SQL migracij | 9 fajlov, ~391 strok |
| Prompt templates | 11 fajlov, ~176 strok |
| HTML templates | 1 (SPA admin panel) |
| Dokumentaciya | 60+ fajlov |
| Obshchij razmer | ~7 200 strok koda |

## Zavisimosti (requirements.txt)

| Paket | Versiya | Naznachenie |
|-------|---------|-------------|
| fastapi | >=0.111.0 | ASGI web framework |
| uvicorn[standard] | >=0.29.0 | ASGI server |
| httpx[socks] | >=0.27.0 | HTTP client s SOCKS5 proxy |
| sqlalchemy | >=2.0.30 | ORM (async) |
| aiomysql | >=0.2.0 | MariaDB async driver |
| apscheduler | >=3.10.4 | Background scheduler |
| qdrant-client | >=1.9.0 | Vector DB client |
| structlog | >=24.1.0 | Structured JSON logging |
| pydantic-settings | >=2.2.1 | Config from .env |
| pytz | >=2024.1 | Timezone handling |
| jinja2 | >=3.1.0 | HTML templates |
| python-multipart | >=0.0.9 | Form data parsing |
| redis[hiredis] | >=5.0.0 | Redis Streams + cache |

## Kratkoe opisanie modulej (1 stroka)

| Modul | Opisanie |
|-------|----------|
| `main.py` | FastAPI app, lifespan, APScheduler (5s messages, 30m tokens, 30m vacancies, 3:00 cleanup, 9:00 telegram) |
| `config.py` | Pydantic Settings: DB, API keys, night window, delays, Telegram, Bitrix |
| `api/webhooks.py` | Avito webhook receiver: classify event, enqueue to Redis or process sync |
| `api/admin.py` | REST API: applicants, chats, handover cards, emulator, segmentation stats, export |
| `api/admin_web.py` | Web UI: login/logout, signed cookie sessions (HMAC, 24h), HTML pages |
| `api/oauth.py` | OAuth2 code -> token exchange for Avito accounts |
| `models/db.py` | 13 ORM modelej: AvitoAccount, Applicant, Vacancy, Application, Chat, Message, AISession, HandoverCard, WebhookLog, EventLog, AIPromptsLog + Base, engine |
| `services/ai_agent.py` | Dialog state machine: greeting -> qualification -> presentation -> fork -> booking/alternatives -> handover |
| `services/ai_claude.py` | Claude API: retry 5x exponential (2,5,15,30,60s), fallback OpenAI 3x (3,10,30s), cost tracking |
| `services/ai_rag.py` | Qdrant vector search + OpenAI text-embedding-3-small, vacancy matching by description+skills |
| `services/avito_auth.py` | OAuth2 per-account token refresh, expiration checking |
| `services/avito_applications.py` | Avito Applications API: fetch details, resume, phone extraction |
| `services/avito_messenger.py` | Avito Messenger: send/receive messages, test chat interception, skip_db flag |
| `services/redis_queue.py` | Redis Streams: enqueue, consume, ack, pending recovery (5 min idle) |
| `services/vacancy_sync.py` | Sync vacancies from Avito API -> DB + Qdrant, auto-deactivation disabled |
| `services/vacancy_parser.py` | Address extraction: regexp markers + Claude AI fallback |
| `services/handover.py` | Create handover cards with dialog summary, candidate data, callback slot |
| `services/telegram_notifier.py` | Morning summary: cards za noch (21:00-09:00) per account per Telegram topic |
| `services/segmentation.py` | Block classification (1=high priority, 2=warm, 3=not suitable) + callback slot parsing |
| `services/message_scheduler.py` | Delayed queue: schedule -> poll 5s -> send, session status check, skip_db |
| `services/bitrix_agent.py` | Bitrix CRM agent toggle on/off via HTTP |
| `workers/incoming_processor.py` | Incoming message handler: 10s debounce per chat, merge fast messages, night window check |
| `workers/webhook_consumer.py` | Redis Streams consumer: read, process, ack, pending recovery |
| `workers/token_refresher.py` | Background job: refresh all active account tokens |
| `utils/time_helpers.py` | Night window check (18:00-09:59), adaptive delay, greeting delay, followup delay |
| `utils/logger.py` | structlog config: JSON format, context fields |
| `utils/event_logger.py` | Helper: log_event(account_id, event_type, message) -> event_log table |
