# CRM Avito AI Worker -- Project Status Snapshot

**Date:** 2026-03-18
**Branch:** main (clean, commit 8e8210a)

---

## 1. File Tree

```
.
├── api/
│   ├── __init__.py
│   ├── admin.py
│   ├── admin_web.py
│   ├── oauth.py
│   └── webhooks.py
├── docs/
│   ├── SPEC.md
│   ├── audit/
│   │   └── audit-report.md
│   ├── dev/
│   │   ├── bitrix-line-deactivation-deploy.md
│   │   ├── final-two-bugs-deploy.md
│   │   ├── fix-ai-claude-retry-deploy.md
│   │   ├── fix-connection-reuse-deploy.md
│   │   ├── fix-debounce-and-cascade-deploy.md
│   │   ├── fix-duplicate-messages-deploy.md
│   │   ├── fix-time-deploy.md
│   │   ├── handover-delivery-deploy.md
│   │   ├── openai-proxy-deploy.md
│   │   ├── redis-webhook-buffer-deploy.md
│   │   ├── remove-socks-proxy-deploy.md
│   │   ├── retry-fallback-deploy.md
│   │   ├── telegram-fixes-deploy.md
│   │   └── webhook-bitrix-crm-toggle-deploy.md
│   └── tasks/
│       ├── TASK-admin-panel.md
│       ├── TASK-final-two-bugs.md
│       ├── TASK-fix-ai-claude-retry.md
│       ├── TASK-fix-connection-reuse.md
│       ├── TASK-fix-debounce-and-cascade.md
│       ├── TASK-fix-duplicate-messages.md
│       ├── TASK-fix-time.md
│       ├── TASK-handover-delivery.md
│       ├── TASK-multi-account.md
│       ├── TASK-openai-proxy.md
│       ├── TASK-redis-webhook-buffer.md
│       ├── TASK-remove-socks-proxy.md
│       ├── TASK-retry-fallback.md
│       ├── TASK-telegram-fixes.md
│       ├── TASK-webhook-bitrix-crm-toggle.md
│       └── task-bitrix-line-deactivation.md
├── migrations/
│   ├── 003_multi_account.sql
│   ├── 004_event_log.sql
│   ├── 005_telegram_topic.sql
│   ├── 006_redis_setup.md
│   ├── schema_v1.sql
│   ├── schema_v2_migration.sql
│   └── schema_v2_phase2_migration.sql
├── models/
│   ├── __init__.py
│   └── db.py
├── prompts/
│   ├── alternatives.txt
│   ├── booking.txt
│   ├── clarify.txt
│   ├── followup.txt
│   ├── greeting.txt
│   ├── objection.txt
│   ├── presentation.txt
│   ├── qualification.txt
│   └── system.txt
├── scripts/
│   ├── webhook_enable_bitrix.py
│   └── webhook_enable_ours.py
├── services/
│   ├── __init__.py
│   ├── add_refresh_token.py
│   ├── ai_agent.py
│   ├── ai_claude.py
│   ├── ai_rag.py
│   ├── avito_applications.py
│   ├── avito_auth.py
│   ├── avito_messenger.py
│   ├── handover.py
│   ├── message_scheduler.py
│   ├── redis_queue.py
│   ├── segmentation.py
│   ├── telegram_notifier.py
│   ├── vacancy_parser.py
│   └── vacancy_sync.py
├── templates/
│   ├── admin.html
│   └── login.html
├── utils/
│   ├── __init__.py
│   ├── event_logger.py
│   ├── logger.py
│   └── time_helpers.py
├── workers/
│   ├── __init__.py
│   ├── incoming_processor.py
│   ├── token_refresher.py
│   └── webhook_consumer.py
├── CHANGELOG-admin-panel.md
├── CHANGELOG-handover-delivery.md
├── CLAUDE.md
├── config.py
├── k24-crm-worker.service
├── main.py
└── requirements.txt
```

---

## 2. Key Files

### 2.1 main.py

```python
"""Tochka vkhoda. FastAPI app s lifespan."""
import asyncio
from contextlib import asynccontextmanager
from datetime import datetime
import httpx
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from fastapi import FastAPI

from api.webhooks import router as webhooks_router
from api.oauth import router as oauth_router
from api.admin import router as admin_router
from api.admin_web import router as admin_web_router
from config import settings
from models.db import engine
from utils.logger import get_logger, setup_logging
from workers.token_refresher import run_token_refresher
from services.message_scheduler import process_scheduled
from services.ai_rag import ensure_collection
from services.vacancy_sync import sync_all_vacancies
import logging
logging.getLogger("apscheduler").setLevel(logging.WARNING)

setup_logging()
log = get_logger(__name__)

scheduler = AsyncIOScheduler(timezone=settings.tz)


async def _cleanup_old_events() -> None:
    """Udalyaet sobytiya starshe 30 dney iz event_log."""
    try:
        from sqlalchemy import text as sa_text
        from models.db import AsyncSessionFactory
        async with AsyncSessionFactory() as session:
            await session.execute(
                sa_text("DELETE FROM event_log WHERE created_at < NOW() - INTERVAL 30 DAY")
            )
            await session.commit()
        log.info("event_log_cleanup_done")
    except Exception as exc:
        log.error("event_log_cleanup_failed", error=str(exc))


_webhook_consumer = None
_consumer_task = None


@asynccontextmanager
async def lifespan(app: FastAPI):
    global _webhook_consumer, _consumer_task
    log.info("crm_worker_starting", host=settings.webhook_host, port=settings.webhook_port)

    # Proverit soedinenie s BD
    try:
        from sqlalchemy import text
        async with engine.connect() as conn:
            await conn.execute(text("SELECT 1"))
        log.info("db_connection_ok")
    except Exception as exc:
        log.error("db_connection_failed", error=str(exc))

    # Proverit/sozdat' kollektsiyu Qdrant
    try:
        await ensure_collection()
        log.info("qdrant_collection_ok")
    except Exception as exc:
        log.error("qdrant_init_failed", error=str(exc))

    # Zapustit' planirovshhik
    scheduler.add_job(
        process_scheduled,
        "interval",
        seconds=5,
        id="message_scheduler",
        max_instances=1,
    )
    scheduler.add_job(
        run_token_refresher,
        "interval",
        minutes=30,
        id="token_refresher",
        max_instances=1,
    )
    scheduler.add_job(
        _refresh_vacancies_job,
        "interval",
        minutes=30,
        id="vacancies_refresh",
        max_instances=1,
        next_run_time=datetime.now(),
    )
    scheduler.add_job(
        _cleanup_old_events,
        "cron",
        hour=3,
        minute=0,
        id="event_log_cleanup",
        max_instances=1,
    )

    from services.telegram_notifier import send_morning_summary
    scheduler.add_job(
        send_morning_summary,
        "cron",
        hour=settings.telegram_morning_hour,
        minute=settings.telegram_morning_minute,
        timezone="Europe/Moscow",
        id="telegram_morning_summary",
        max_instances=1,
    )

    scheduler.start()
    log.info("scheduler_started")

    # Redis Streams consumer
    try:
        from services.redis_queue import ensure_consumer_groups, close_redis
        from workers.webhook_consumer import WebhookConsumer

        await ensure_consumer_groups()
        _webhook_consumer = WebhookConsumer(consumer_name="main-worker")
        _consumer_task = asyncio.create_task(_webhook_consumer.start())
        log.info("webhook_consumer_started")
    except Exception as exc:
        log.error("webhook_consumer_init_failed", error=str(exc))

    yield

    # Graceful shutdown
    if _webhook_consumer:
        await _webhook_consumer.stop()
    if _consumer_task:
        _consumer_task.cancel()
    try:
        from services.redis_queue import close_redis
        await close_redis()
    except Exception:
        pass

    scheduler.shutdown(wait=True)
    await engine.dispose()
    log.info("crm_worker_stopped")


async def _refresh_vacancies_job() -> None:
    """Sinkhronizuet vse vakansii so VSEKH aktivnykh akkauntov Avito kazhdye 30 min."""
    from sqlalchemy import select
    from models.db import AsyncSessionFactory, AvitoAccount

    try:
        async with AsyncSessionFactory() as session:
            result = await session.execute(
                select(AvitoAccount).where(AvitoAccount.is_active == True)
            )
            accounts = result.scalars().all()

        if not accounts:
            log.warning("vacancies_sync_no_active_accounts")
            return

        total = 0
        for account in accounts:
            try:
                count = await sync_all_vacancies(account_id=account.id)
                total += count
                log.info(
                    "vacancies_sync_account_done",
                    account_id=account.id,
                    account_name=account.account_name,
                    synced=count,
                )
            except Exception as exc:
                log.error(
                    "vacancies_sync_account_failed",
                    account_id=account.id,
                    error=str(exc),
                )

        log.info("vacancies_sync_all_done", total_synced=total, accounts=len(accounts))
    except Exception as exc:
        log.error("vacancies_sync_failed", error=str(exc))


app = FastAPI(
    title="CRM Avito AI Worker",
    version="1.0.0",
    lifespan=lifespan,
)

app.include_router(webhooks_router)
app.include_router(oauth_router)
app.include_router(admin_web_router)
app.include_router(admin_router)


@app.get("/health")
async def health():
    return {"status": "ok"}
```

---

### 2.2 config.py

```python
from pydantic_settings import BaseSettings
from pydantic import Field


class Settings(BaseSettings):
    # MariaDB
    db_host: str = "localhost"
    db_port: int = 3306
    db_name: str = "2_kadry_4_crm_avito"
    db_user: str = "crm_avito"
    db_password: str = "***"

    # Qdrant
    qdrant_host: str = "localhost"
    qdrant_port: int = 6333
    qdrant_collection: str = "vacancies"

    # Anthropic Claude
    anthropic_api_key: str = "***"
    claude_model: str = "claude-sonnet-4-6"
    claude_proxy: str = "socks5://127.0.0.1:1080"

    # OpenAI (embeddings + fallback)
    openai_api_key: str = "***"
    openai_embedding_model: str = "text-embedding-3-small"
    openai_embedding_dim: int = 1536
    openai_fallback_model: str = "gpt-5.4"

    # Claude retry
    claude_max_retries: int = 5

    # Redis
    redis_host: str = "127.0.0.1"
    redis_port: int = 6379
    redis_db: int = 0
    redis_password: str = "***"

    # Vacancy sync
    vacancy_cache_ttl_minutes: int = 30

    # Avito
    avito_default_account_id: int = 1

    # Admin
    admin_token: str = ""
    admin_login: str = "nimdo"
    admin_password: str = "***"
    admin_secret_key: str = "***"

    # Webhook
    webhook_secret: str = ""
    webhook_host: str = "127.0.0.1"
    webhook_port: int = 8800

    # AI Agent
    ai_work_start: str = "18:00"
    ai_work_end: str = "09:59"
    ai_pause_min_sec: int = 120
    ai_pause_max_sec: int = 360
    ai_max_followups: int = 2
    ai_followup_delay_min: int = 600
    ai_followup_delay_max: int = 900

    # Telegram
    telegram_bot_token: str = "***"
    telegram_chat_id: str = "-1003560902940_2"
    telegram_group_id: str = "-1003560902940"
    telegram_morning_hour: int = 10
    telegram_morning_minute: int = 0

    # Timezone
    tz: str = "Europe/Moscow"

    model_config = {"env_file": ".env", "extra": "ignore"}

    @property
    def db_url(self) -> str:
        return (
            f"mysql+aiomysql://{self.db_user}:{__import__('urllib.parse').parse.quote_plus(self.db_password)}"
            f"@{self.db_host}:{self.db_port}/{self.db_name}?charset=utf8mb4"
        )


settings = Settings()
```

> Note: secrets masked with `***` in this dump.

---

### 2.3 models/db.py

```python
from datetime import datetime
from typing import Optional

from sqlalchemy import (
    BigInteger, Boolean, Column, DateTime, Enum, ForeignKey,
    Integer, JSON, Numeric, SmallInteger, String, Text, UniqueConstraint,
)
from sqlalchemy.ext.asyncio import AsyncAttrs, AsyncSession, create_async_engine, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase, relationship

from config import settings


class Base(AsyncAttrs, DeclarativeBase):
    pass


engine = create_async_engine(
    settings.db_url,
    pool_recycle=3600,
    pool_pre_ping=True,
    echo=False,
)

AsyncSessionFactory: async_sessionmaker[AsyncSession] = async_sessionmaker(
    engine, expire_on_commit=False
)


async def get_session() -> AsyncSession:
    async with AsyncSessionFactory() as session:
        yield session


class AvitoAccount(Base):
    __tablename__ = "avito_accounts"

    id               = Column(Integer, primary_key=True, autoincrement=True)
    client_id        = Column(String(100), nullable=False, unique=True)
    client_secret    = Column(String(255), nullable=False)
    avito_user_id    = Column(String(50), nullable=False)
    account_name     = Column(String(100))
    access_token     = Column(Text)
    refresh_token    = Column(Text)
    token_expires_at = Column(DateTime)
    telegram_topic_id   = Column(Integer, nullable=True)
    is_active           = Column(Boolean, default=True)
    webhook_registered  = Column(Boolean, default=False)
    created_at          = Column(DateTime, default=datetime.utcnow)
    updated_at          = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)


class Applicant(Base):
    __tablename__ = "applicants"

    id            = Column(Integer, primary_key=True, autoincrement=True)
    avito_user_id = Column(String(50))
    name          = Column(String(200))
    phone         = Column(String(20), unique=True)
    created_at    = Column(DateTime, default=datetime.utcnow)
    updated_at    = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    applications = relationship("Application", back_populates="applicant")


class Vacancy(Base):
    __tablename__ = "vacancies"

    id                = Column(Integer, primary_key=True, autoincrement=True)
    account_id        = Column(Integer, ForeignKey("avito_accounts.id"))
    avito_vacancy_id  = Column(BigInteger, unique=True, nullable=False)
    avito_uuid        = Column(String(100))
    title             = Column(String(200))
    raw_description   = Column(Text)
    avito_url         = Column(String(500))
    city              = Column(String(100))
    address           = Column(String(500))
    latitude          = Column(Numeric(10, 8))
    longitude         = Column(Numeric(11, 8))
    business_area     = Column(String(200))
    profession        = Column(String(200))
    schedule          = Column(String(100))
    employment        = Column(String(100))
    experience        = Column(String(100))
    salary_raw        = Column(Integer)
    salary_from       = Column(Integer)
    salary_to         = Column(Integer)
    paid_period       = Column(String(50))
    payout_frequency  = Column(String(100))
    facility_type     = Column(String(200))
    parsed_format     = Column(String(100))
    parsed_tasks      = Column(String(500))
    parsed_extra      = Column(Text)
    is_active         = Column(Boolean, default=True)
    embedding_indexed = Column(Boolean, default=False)
    last_synced_at    = Column(DateTime)
    created_at        = Column(DateTime, default=datetime.utcnow)
    updated_at        = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)


class Application(Base):
    __tablename__ = "applications"

    id                   = Column(Integer, primary_key=True, autoincrement=True)
    avito_application_id = Column(String(100), nullable=False, unique=True)
    applicant_id         = Column(Integer, ForeignKey("applicants.id"), nullable=False)
    account_id           = Column(Integer, ForeignKey("avito_accounts.id"), nullable=False)
    vacancy_id           = Column(Integer)
    avito_item_id        = Column(String(50))
    avito_vacancy_id     = Column(BigInteger)
    applicant_name       = Column(String(200))
    applicant_phone      = Column(String(20))
    chat_id              = Column(String(100))
    status               = Column(
        Enum("new", "ai_active", "ai_done", "operator", "closed"), default="new"
    )
    block          = Column(SmallInteger)
    callback_slot  = Column(String(50))
    created_at     = Column(DateTime, default=datetime.utcnow)
    updated_at     = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    applicant  = relationship("Applicant", back_populates="applications")
    account    = relationship("AvitoAccount")
    chats      = relationship("Chat", back_populates="application")
    ai_sessions = relationship("AISession", back_populates="application")


class Chat(Base):
    __tablename__ = "chats"

    id             = Column(Integer, primary_key=True, autoincrement=True)
    avito_chat_id  = Column(String(100), nullable=False, unique=True)
    application_id = Column(Integer, ForeignKey("applications.id"), nullable=False)
    account_id     = Column(Integer, ForeignKey("avito_accounts.id"), nullable=False)
    last_message_at = Column(DateTime)
    created_at     = Column(DateTime, default=datetime.utcnow)

    application = relationship("Application", back_populates="chats")
    account     = relationship("AvitoAccount")
    messages    = relationship("Message", back_populates="chat")
    ai_sessions = relationship("AISession", back_populates="chat")


class Message(Base):
    __tablename__ = "messages"

    id               = Column(Integer, primary_key=True, autoincrement=True)
    chat_id          = Column(Integer, ForeignKey("chats.id"), nullable=False)
    avito_message_id = Column(String(100))
    direction        = Column(Enum("incoming", "outgoing"), nullable=False)
    sender_type      = Column(Enum("applicant", "ai", "operator"), nullable=False)
    content          = Column(Text, nullable=False)
    scheduled_at     = Column(DateTime)
    delivered_at     = Column(DateTime)
    created_at       = Column(DateTime, default=datetime.utcnow)

    chat = relationship("Chat", back_populates="messages")


class AISession(Base):
    __tablename__ = "ai_sessions"

    id             = Column(Integer, primary_key=True, autoincrement=True)
    application_id = Column(Integer, ForeignKey("applications.id"), nullable=False)
    chat_id        = Column(Integer, ForeignKey("chats.id"), nullable=False)
    dialog_stage   = Column(
        Enum("greeting", "waiting_qualification", "presentation", "waiting_fork",
             "alternatives", "booking", "waiting_booking",
             "followup", "clarify", "handover", "done", "completed", "failed",
             "channel_choice", "qualification", "segmentation"),
        default="greeting"
    )
    assigned_block         = Column(SmallInteger)
    collected_age          = Column(String(10))
    collected_city         = Column(String(100))
    collected_citizenship  = Column(String(100))
    collected_metro        = Column(String(100))
    callback_slot          = Column(String(50))
    channel_choice         = Column(String(20))
    result                 = Column(String(50))
    followup_count         = Column(Integer, default=0)
    clarify_count          = Column(Integer, default=0)
    last_applicant_msg_at  = Column(DateTime)
    status                 = Column(Enum("active", "completed", "failed"), default="active")
    created_at             = Column(DateTime, default=datetime.utcnow)
    updated_at             = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    application = relationship("Application", back_populates="ai_sessions")
    chat        = relationship("Chat", back_populates="ai_sessions")


class HandoverCard(Base):
    __tablename__ = "handover_cards"

    id                = Column(Integer, primary_key=True, autoincrement=True)
    ai_session_id     = Column(Integer, ForeignKey("ai_sessions.id"), nullable=False)
    application_id    = Column(Integer, ForeignKey("applications.id"), nullable=False)
    avito_vacancy_id  = Column(BigInteger)
    vacancy_title     = Column(String(200))
    candidate_name    = Column(String(100))
    candidate_phone   = Column(String(20))
    candidate_city    = Column(String(100))
    candidate_metro   = Column(String(100))
    candidate_age     = Column(String(10))
    result            = Column(String(50))
    callback_slot     = Column(String(50))
    dialog_summary    = Column(Text)
    block             = Column(SmallInteger)
    messages_count    = Column(Integer, default=0)
    is_processed      = Column(Boolean, default=False)
    created_at        = Column(DateTime, default=datetime.utcnow)
    processed_at      = Column(DateTime)

    ai_session  = relationship("AISession")
    application = relationship("Application")


class WebhookLog(Base):
    __tablename__ = "webhook_log"

    id            = Column(Integer, primary_key=True, autoincrement=True)
    event_type    = Column(String(100))
    avito_user_id = Column(String(50))
    payload       = Column(Text)
    processed     = Column(Boolean, default=False)
    error         = Column(Text)
    created_at    = Column(DateTime, default=datetime.utcnow)


class EventLog(Base):
    __tablename__ = "event_log"

    id         = Column(Integer, primary_key=True, autoincrement=True)
    account_id = Column(Integer, ForeignKey("avito_accounts.id"), nullable=True)
    event_type = Column(String(50), nullable=False)
    message    = Column(Text)
    details    = Column(JSON, nullable=True)
    created_at = Column(DateTime, default=datetime.utcnow)

    account = relationship("AvitoAccount")


class AIPromptsLog(Base):
    __tablename__ = "ai_prompts_log"

    id                 = Column(Integer, primary_key=True, autoincrement=True)
    ai_session_id      = Column(Integer)
    application_id     = Column(Integer)
    dialog_stage       = Column(String(50))
    prompt_tokens      = Column(Integer)
    completion_tokens  = Column(Integer)
    total_tokens       = Column(Integer)
    cost_usd           = Column(Numeric(10, 6))
    response_ms        = Column(Integer)
    error              = Column(Text)
    created_at         = Column(DateTime, default=datetime.utcnow)
```

---

### 2.4 api/webhooks.py

```python
"""Webhook endpoint dlya priyoma sobytiy ot Avito."""
import asyncio
import json
from datetime import datetime

import httpx
from fastapi import APIRouter, BackgroundTasks, HTTPException, Request
from sqlalchemy import select

from config import settings
from models.db import (
    AsyncSessionFactory, WebhookLog, Application, Applicant,
    Chat, AISession, AvitoAccount, Vacancy,
)
from services.avito_applications import get_application_details, _auth_headers
from utils.logger import get_logger
from utils.time_helpers import is_night_window

log = get_logger(__name__)

router = APIRouter()

# Zashchita ot parallel'noy obrabotki odnogo otklika
_processing_ids: set[str] = set()


async def _resolve_account_by_user_id(user_id) -> AvitoAccount | None:
    """Nakhodit aktivnyy akkaunt po avito_user_id."""
    if user_id is None:
        return None
    async with AsyncSessionFactory() as session:
        result = await session.execute(
            select(AvitoAccount).where(
                AvitoAccount.avito_user_id == str(user_id),
                AvitoAccount.is_active == True,
            )
        )
        return result.scalar_one_or_none()


async def _resolve_account_for_application(payload: dict) -> int | None:
    """
    Opredelyaet account_id dlya novogo otklika.
    1. Esli v payload est' user_id -> ishchem akkaunt
    2. Esli est' vacancy_id -> ishchem vakansiyu s account_id
    3. Fallback -> perebiraem aktivnye akkaunty
    """
    # 1. Po user_id iz payload
    user_id = payload.get("user_id")
    if user_id:
        account = await _resolve_account_by_user_id(user_id)
        if account:
            log.info("account_resolved_by_user_id", user_id=user_id, account_id=account.id)
            return account.id

    # 2. Po vacancy_id cherez tablitsu vacancies
    vacancy_id = payload.get("vacancyId") or payload.get("vacancy_id") or payload.get("item_id")
    if vacancy_id:
        async with AsyncSessionFactory() as session:
            result = await session.execute(
                select(Vacancy.account_id).where(
                    Vacancy.avito_vacancy_id == int(vacancy_id),
                    Vacancy.account_id != None,
                )
            )
            row = result.first()
            if row and row[0]:
                log.info("account_resolved_by_vacancy", vacancy_id=vacancy_id, account_id=row[0])
                return row[0]

    # 3. Fallback -- esli odin aktivnyy akkaunt, ispol'zuem ego
    async with AsyncSessionFactory() as session:
        result = await session.execute(
            select(AvitoAccount).where(AvitoAccount.is_active == True)
        )
        accounts = result.scalars().all()

    if len(accounts) == 1:
        log.info("account_resolved_single_active", account_id=accounts[0].id)
        return accounts[0].id

    # 4. Perebiraem aktivnye akkaunty -- probuem poluchit' dannye otklika
    avito_application_id = payload.get("applyId") or str(
        payload.get("application_id") or payload.get("id") or payload.get("object_id", "")
    )
    if avito_application_id and accounts:
        for account in accounts:
            try:
                details = await get_application_details(account.id, avito_application_id)
                if details.get("chat_id") or details.get("phone"):
                    log.info(
                        "account_resolved_by_trial",
                        account_id=account.id,
                        account_name=account.account_name,
                    )
                    return account.id
            except Exception:
                continue

    # Sovsem fallback -- defoltnyy akkaunt
    log.warning("account_resolved_fallback_default")
    return settings.avito_default_account_id


async def _get_all_our_user_ids() -> set[str]:
    """Vozvrashchaet set vsekh avito_user_id aktivnykh akkauntov."""
    async with AsyncSessionFactory() as session:
        result = await session.execute(
            select(AvitoAccount.avito_user_id).where(AvitoAccount.is_active == True)
        )
        return {str(row[0]) for row in result.fetchall()}


# -- Diagnostika --

@router.get("/debug/application/{application_id}")
async def debug_application(application_id: str, account_id: int = None):
    """Pryamoy zapros k Avito API -- pokazyvaet syroy otvet."""
    from services.avito_auth import get_valid_token

    if account_id is None:
        account_id = settings.avito_default_account_id
    token = await get_valid_token(account_id)
    url = "https://api.avito.ru/job/v1/applications/get_by_ids"
    body = {"ids": [application_id]}

    results = {}
    async with httpx.AsyncClient(timeout=30) as client:
        resp1 = await client.post(
            url,
            headers={"Authorization": f"Bearer {token}"},
            json=body,
        )
        results["without_employee_header"] = {
            "status": resp1.status_code,
            "body": resp1.json(),
        }

        resp2 = await client.post(
            url,
            headers=_auth_headers(token),
            json=body,
        )
        results["with_employee_header"] = {
            "status": resp2.status_code,
            "body": resp2.json(),
        }

    return {
        "account_id": account_id,
        "application_id": application_id,
        "token_preview": token[:20] + "...",
        "results": results,
    }


# -- Klassifikatsiya vebkhukov --

def _classify_webhook(payload: dict) -> tuple[str, dict]:
    """
    Opredelyaet tip sobytiya i izvlekaet dannye iz payload.

    Avito shlyot dva formata:

    1) Job webhook (otkliki):
       {"applyId": "xxx", ...}

    2) Messenger V3 webhook (soobshcheniya):
       {
         "id": "webhook_msg_id",
         "payload": {
           "type": "message",
           "value": {
             "author_id": 123,
             "chat_id": "u2i-xxx",
             "content": {"text": "Privet"},
             "created": 1700000000,
             "id": "avito_msg_id"
           }
         },
         "timestamp": 1700000000,
         "version": "v1.1"
       }
    """
    # --- Job: novyy otklik ---
    if "applyId" in payload:
        return "new_application", payload

    # --- Messenger V3: soobshchenie ---
    inner_payload = payload.get("payload", {})
    if isinstance(inner_payload, dict) and "type" in inner_payload:
        msg_type = inner_payload.get("type", "")
        value = inner_payload.get("value", {})

        if msg_type == "message" and isinstance(value, dict):
            content = value.get("content", {})
            text = ""
            if isinstance(content, dict):
                text = content.get("text", "")
            elif isinstance(content, str):
                text = content

            normalized = {
                "message_id": value.get("id", ""),
                "chat_id": value.get("chat_id", ""),
                "author_id": value.get("author_id"),
                "user_id": payload.get("user_id"),
                "text": text,
                "chat_type": value.get("chat_type", ""),
                "created": value.get("created"),
            }
            return "new_message", normalized

    # --- Fallback ---
    event_type = payload.get("event_type") or payload.get("type", "unknown")
    return event_type, payload


# -- Opredelenie strima --

def _determine_stream(payload: dict) -> str:
    """Opredelyaet v kakoy Redis stream polozhit' vebkhuk."""
    from services.redis_queue import STREAM_MESSENGER, STREAM_APPLICATIONS

    if "applyId" in payload:
        return STREAM_APPLICATIONS

    inner = payload.get("payload", {})
    if isinstance(inner, dict) and inner.get("type") == "message":
        return STREAM_MESSENGER

    return STREAM_MESSENGER


# -- Osnovnoy endpoint --

@router.post("/webhooks/avito")
async def avito_webhook(request: Request, background_tasks: BackgroundTasks):
    body = await request.body()

    try:
        payload = json.loads(body)
    except (json.JSONDecodeError, ValueError):
        log.info("webhook_ping", body=body[:100])
        return {"status": "ok"}

    if settings.webhook_secret:
        received_secret = request.headers.get("x-secret", "")
        if received_secret != settings.webhook_secret:
            log.warning("webhook_invalid_secret")
            raise HTTPException(status_code=403, detail="Invalid secret")

    event_type, _ = _classify_webhook(payload)

    log.info(
        "webhook_received",
        event_type=event_type,
        payload_preview=str(payload)[:300],
    )

    background_tasks.add_task(
        _save_webhook_log, event_type=event_type, payload=body.decode()
    )

    stream = _determine_stream(payload)
    try:
        from services.redis_queue import enqueue_webhook
        source_ip = request.client.host if request.client else ""
        msg_id = await enqueue_webhook(stream, payload, source_ip=source_ip)
        log.info("webhook_enqueued", stream=stream, message_id=msg_id)
        return {"ok": True}

    except Exception as redis_exc:
        log.error("redis_unavailable_fallback_sync", error=str(redis_exc))

    # Fallback: obrabotat' sinkhronno kak ran'she (Redis nedostupen)
    if event_type == "new_application":
        apply_id = payload.get("applyId", "")
        if apply_id in _processing_ids:
            log.info("webhook_already_processing", apply_id=apply_id)
            return {"ok": True}
        background_tasks.add_task(_process_new_application, payload)

    elif event_type == "new_message":
        _, normalized = _classify_webhook(payload)
        background_tasks.add_task(_process_new_message, normalized)

    else:
        log.debug("webhook_unknown_event", event_type=event_type)

    return {"ok": True}


# -- Vnutrennie obrabotchiki --

async def _log_webhook_event(event_type: str, account_id: int | None, message: str) -> None:
    try:
        from utils.event_logger import log_event
        await log_event(account_id, event_type, message)
    except Exception:
        pass


async def _save_webhook_log(event_type: str, payload: str) -> None:
    try:
        async with AsyncSessionFactory() as session:
            log_entry = WebhookLog(
                event_type=event_type, payload=payload, processed=False
            )
            session.add(log_entry)
            await session.commit()
    except Exception as exc:
        log.error("webhook_log_save_failed", error=str(exc))


async def _process_new_application(payload: dict) -> None:
    avito_application_id = payload.get("applyId") or str(
        payload.get("application_id")
        or payload.get("id")
        or payload.get("object_id", "")
    )
    if not avito_application_id:
        log.error("webhook_no_application_id", payload=str(payload)[:200])
        return

    if avito_application_id in _processing_ids:
        log.info("webhook_skip_duplicate", apply_id=avito_application_id)
        return
    _processing_ids.add(avito_application_id)

    try:
        async with AsyncSessionFactory() as session:
            existing = await session.execute(
                select(Application).where(
                    Application.avito_application_id == avito_application_id
                )
            )
            if existing.scalar_one_or_none():
                log.debug(
                    "webhook_duplicate_application",
                    avito_application_id=avito_application_id,
                )
                return

        account_id = await _resolve_account_for_application(payload)
        details = await get_application_details(account_id, avito_application_id)

        applicant_name = details.get("name", "")
        applicant_phone = details.get("phone", "")
        avito_item_id = str(details.get("item_id", ""))
        avito_chat_id = str(details.get("chat_id", ""))

        log.info(
            "application_details_result",
            avito_application_id=avito_application_id,
            name=applicant_name,
            has_phone=bool(applicant_phone),
            chat_id=avito_chat_id,
        )

        async with AsyncSessionFactory() as session:
            existing = await session.execute(
                select(Application).where(
                    Application.avito_application_id == avito_application_id
                )
            )
            if existing.scalar_one_or_none():
                return

            applicant = None
            if applicant_phone:
                app_result = await session.execute(
                    select(Applicant).where(Applicant.phone == applicant_phone)
                )
                applicant = app_result.scalar_one_or_none()

            if not applicant:
                applicant = Applicant(
                    name=applicant_name or None, phone=applicant_phone or None
                )
                session.add(applicant)
                await session.flush()

            application = Application(
                avito_application_id=avito_application_id,
                applicant_id=applicant.id,
                account_id=account_id,
                avito_item_id=avito_item_id or None,
                avito_vacancy_id=int(avito_item_id) if avito_item_id else None,
                applicant_name=applicant_name or None,
                applicant_phone=applicant_phone or None,
                chat_id=avito_chat_id or None,
                status="new",
            )
            session.add(application)
            await session.flush()

            chat = None
            if avito_chat_id:
                chat = Chat(
                    avito_chat_id=avito_chat_id,
                    application_id=application.id,
                    account_id=account_id,
                )
                session.add(chat)
                await session.flush()

            ai_session_id = None
            if is_night_window() and chat:
                application.status = "ai_active"
                ai_session = AISession(
                    application_id=application.id,
                    chat_id=chat.id,
                    dialog_stage="greeting",
                )
                session.add(ai_session)
                await session.flush()
                ai_session_id = ai_session.id

            await session.commit()

            log.info(
                "application_created",
                application_id=application.id,
                avito_application_id=avito_application_id,
                applicant_id=applicant.id,
                account_id=account_id,
                has_chat=bool(chat),
                night_window=is_night_window(),
                ai_session_id=ai_session_id,
            )

        try:
            from utils.event_logger import log_event
            vacancy_id = application.avito_vacancy_id or ""
            await log_event(
                account_id, "webhook_application",
                f"Novyy otklik {avito_application_id}, vakansiya {vacancy_id}",
            )
        except Exception:
            pass

        if is_night_window() and chat:
            from services.ai_agent import process_new_application

            await asyncio.sleep(2)
            await process_new_application(application.id)

    except Exception as exc:
        log.error(
            "process_new_application_failed", error=str(exc), exc_info=True
        )
    finally:
        _processing_ids.discard(avito_application_id)


async def _process_new_message(data: dict) -> None:
    """Obrabotka vkhodayshchego soobshcheniya."""
    try:
        avito_message_id = str(data.get("message_id", ""))
        avito_chat_id = str(data.get("chat_id", ""))
        author_id = data.get("author_id")
        text = data.get("text", "")

        log.info(
            "processing_incoming_message",
            avito_message_id=avito_message_id,
            avito_chat_id=avito_chat_id,
            text_preview=text[:100] if text else "",
        )

        if not avito_chat_id:
            log.warning("message_no_chat_id", data=str(data)[:200])
            return

        if author_id is not None and int(author_id) == 0:
            log.debug("message_skip_system", avito_message_id=avito_message_id)
            return

        our_user_ids = await _get_all_our_user_ids()

        async with AsyncSessionFactory() as session:
            chat_result = await session.execute(
                select(Chat).where(Chat.avito_chat_id == avito_chat_id)
            )
            chat = chat_result.scalar_one_or_none()
            if not chat:
                log.debug("webhook_unknown_chat", avito_chat_id=avito_chat_id)
                return

            if author_id is not None and str(author_id) in our_user_ids:
                log.debug(
                    "message_skip_own",
                    avito_message_id=avito_message_id,
                    author_id=author_id,
                )
                return

            if avito_message_id:
                from models.db import Message

                dup = await session.execute(
                    select(Message).where(
                        Message.avito_message_id == avito_message_id
                    )
                )
                if dup.scalar_one_or_none():
                    log.debug(
                        "message_duplicate", avito_message_id=avito_message_id
                    )
                    return

            from models.db import Message

            msg = Message(
                chat_id=chat.id,
                avito_message_id=avito_message_id or None,
                direction="incoming",
                sender_type="applicant",
                content=text or "",
            )
            session.add(msg)
            chat.last_message_at = datetime.utcnow()
            await session.commit()
            await session.refresh(msg)

        log.info("message_saved", message_id=msg.id, chat_id=chat.id)

        from workers.incoming_processor import handle_incoming

        await handle_incoming(msg.id)

    except Exception as exc:
        log.error("process_new_message_failed", error=str(exc), exc_info=True)
```

---

### 2.5 api/admin.py

```python
"""Admin-endpointy dlya upravleniya akkauntami Avito."""
from datetime import datetime, date
from typing import Optional

import httpx
from fastapi import APIRouter, Depends, HTTPException, Header, Request
from pydantic import BaseModel
from sqlalchemy import select, func, text

from config import settings
from models.db import (
    AsyncSessionFactory, AvitoAccount, EventLog, Application, Message,
    AISession, HandoverCard, Chat,
)
from services.avito_auth import get_valid_token
from utils.logger import get_logger

log = get_logger(__name__)

router = APIRouter(prefix="/admin", tags=["admin"])


def _verify_admin(request: Request):
    """Proverka avtorizatsii: cookie ILI Bearer token."""
    from api.admin_web import verify_admin
    if not verify_admin(request):
        raise HTTPException(status_code=401, detail="Unauthorized")


class CreateAccountRequest(BaseModel):
    name: str
    client_id: str
    client_secret: str
    avito_user_id: str
    telegram_topic_id: Optional[int] = None


class UpdateAccountRequest(BaseModel):
    name: str
    client_id: str
    client_secret: Optional[str] = None
    avito_user_id: str
    telegram_topic_id: Optional[int] = None


@router.get("/accounts", dependencies=[Depends(_verify_admin)])
async def list_accounts():
    """Spisok vsekh akkauntov."""
    async with AsyncSessionFactory() as session:
        result = await session.execute(select(AvitoAccount))
        accounts = result.scalars().all()

    return [
        {
            "id": a.id,
            "name": a.account_name,
            "client_id": a.client_id,
            "avito_user_id": a.avito_user_id,
            "is_active": a.is_active,
            "webhook_registered": a.webhook_registered,
            "telegram_topic_id": a.telegram_topic_id,
            "has_token": bool(a.access_token),
            "token_expires_at": str(a.token_expires_at) if a.token_expires_at else None,
            "created_at": str(a.created_at),
        }
        for a in accounts
    ]


@router.post("/accounts", dependencies=[Depends(_verify_admin)])
async def create_account(req: CreateAccountRequest):
    async with AsyncSessionFactory() as session:
        existing = await session.execute(
            select(AvitoAccount).where(AvitoAccount.client_id == req.client_id)
        )
        if existing.scalar_one_or_none():
            raise HTTPException(status_code=409, detail="Account with this client_id already exists")

        account = AvitoAccount(
            account_name=req.name,
            client_id=req.client_id,
            client_secret=req.client_secret,
            avito_user_id=req.avito_user_id,
            telegram_topic_id=req.telegram_topic_id,
            is_active=True,
        )
        session.add(account)
        await session.commit()
        await session.refresh(account)
        account_id = account.id

    log.info("admin_account_created", account_id=account_id, name=req.name)

    try:
        from utils.event_logger import log_event
        await log_event(account_id, "account_updated", f"Akkaunt '{req.name}' sozdan")
    except Exception:
        pass

    token_ok = False
    try:
        token = await get_valid_token(account_id)
        token_ok = bool(token)
        log.info("admin_account_token_ok", account_id=account_id)
    except Exception as exc:
        log.warning("admin_account_token_failed", account_id=account_id, error=str(exc))

    webhooks_ok = False
    if token_ok:
        try:
            webhooks_ok = await _register_webhooks_for_account(account_id)
        except Exception as exc:
            log.warning("admin_account_webhooks_failed", account_id=account_id, error=str(exc))

    return {
        "status": "ok",
        "account_id": account_id,
        "token_ok": token_ok,
        "webhooks_registered": webhooks_ok,
    }


@router.put("/accounts/{account_id}", dependencies=[Depends(_verify_admin)])
async def update_account(account_id: int, req: UpdateAccountRequest):
    async with AsyncSessionFactory() as session:
        account = await session.get(AvitoAccount, account_id)
        if not account:
            raise HTTPException(status_code=404, detail="Account not found")

        account.account_name = req.name
        account.client_id = req.client_id
        account.avito_user_id = req.avito_user_id
        account.telegram_topic_id = req.telegram_topic_id
        if req.client_secret:
            account.client_secret = req.client_secret
        await session.commit()

    log.info("admin_account_updated", account_id=account_id, name=req.name)

    try:
        from utils.event_logger import log_event
        await log_event(account_id, "account_updated", f"Akkaunt '{req.name}' obnovlyon")
    except Exception:
        pass

    return {"status": "ok", "account_id": account_id}


@router.post("/accounts/{account_id}/register-webhooks", dependencies=[Depends(_verify_admin)])
async def register_webhooks(account_id: int):
    async with AsyncSessionFactory() as session:
        account = await session.get(AvitoAccount, account_id)
        if not account:
            raise HTTPException(status_code=404, detail="Account not found")
        account_name = account.account_name

    try:
        ok = await _register_webhooks_for_account(account_id)

        try:
            from utils.event_logger import log_event
            await log_event(account_id, "webhook_registered", f"Vebkhuki zaregistrirovany dlya '{account_name}'")
        except Exception:
            pass

        return {"status": "ok" if ok else "partial", "account_id": account_id}
    except Exception as exc:
        raise HTTPException(status_code=500, detail=str(exc))


@router.post("/accounts/{account_id}/toggle", dependencies=[Depends(_verify_admin)])
async def toggle_account(account_id: int, is_active: bool = True):
    async with AsyncSessionFactory() as session:
        account = await session.get(AvitoAccount, account_id)
        if not account:
            raise HTTPException(status_code=404, detail="Account not found")
        account.is_active = is_active
        await session.commit()

    log.info("admin_account_toggled", account_id=account_id, is_active=is_active)
    return {"status": "ok", "account_id": account_id, "is_active": is_active}


@router.get("/api/events", dependencies=[Depends(_verify_admin)])
async def get_events(limit: int = 100, account_id: int = None, event_type: str = None):
    async with AsyncSessionFactory() as session:
        query = (
            select(EventLog, AvitoAccount.account_name)
            .outerjoin(AvitoAccount, EventLog.account_id == AvitoAccount.id)
            .order_by(EventLog.created_at.desc())
            .limit(min(limit, 500))
        )
        if account_id:
            query = query.where(EventLog.account_id == account_id)
        if event_type:
            query = query.where(EventLog.event_type == event_type)

        result = await session.execute(query)
        rows = result.all()

    return [
        {
            "id": ev.id,
            "account_id": ev.account_id,
            "account_name": acct_name or "",
            "event_type": ev.event_type,
            "message": ev.message,
            "details": ev.details,
            "created_at": str(ev.created_at),
        }
        for ev, acct_name in rows
    ]


@router.get("/api/stats", dependencies=[Depends(_verify_admin)])
async def get_stats():
    today_start = datetime.combine(date.today(), datetime.min.time())

    async with AsyncSessionFactory() as session:
        total = await session.execute(select(func.count(AvitoAccount.id)))
        active = await session.execute(
            select(func.count(AvitoAccount.id)).where(AvitoAccount.is_active == True)
        )

        apps_today = await session.execute(
            select(func.count(Application.id)).where(Application.created_at >= today_start)
        )

        msgs_today = await session.execute(
            select(func.count(Message.id)).where(
                Message.created_at >= today_start,
                Message.direction == "outgoing",
            )
        )

        errors_today = await session.execute(
            select(func.count(AISession.id)).where(
                AISession.status == "failed",
                AISession.updated_at >= today_start,
            )
        )

    return {
        "total_accounts": total.scalar() or 0,
        "active_accounts": active.scalar() or 0,
        "applications_today": apps_today.scalar() or 0,
        "messages_today": msgs_today.scalar() or 0,
        "errors_today": errors_today.scalar() or 0,
    }


@router.get("/api/handover", dependencies=[Depends(_verify_admin)])
async def list_handover_cards(
    limit: int = 50,
    offset: int = 0,
    account_id: Optional[int] = None,
    result: Optional[str] = None,
    unprocessed_only: bool = True,
):
    async with AsyncSessionFactory() as session:
        query = (
            select(HandoverCard, AvitoAccount.account_name)
            .join(Application, HandoverCard.application_id == Application.id)
            .join(AvitoAccount, Application.account_id == AvitoAccount.id)
            .order_by(HandoverCard.created_at.desc())
        )
        count_query = (
            select(func.count(HandoverCard.id))
            .join(Application, HandoverCard.application_id == Application.id)
        )

        if unprocessed_only:
            query = query.where(HandoverCard.is_processed == False)
            count_query = count_query.where(HandoverCard.is_processed == False)
        if account_id:
            query = query.where(Application.account_id == account_id)
            count_query = count_query.where(Application.account_id == account_id)
        if result:
            query = query.where(HandoverCard.result == result)
            count_query = count_query.where(HandoverCard.result == result)

        total_result = await session.execute(count_query)
        total = total_result.scalar() or 0

        query = query.limit(min(limit, 200)).offset(offset)
        rows = await session.execute(query)
        cards_rows = rows.all()

    return {
        "total": total,
        "cards": [
            {
                "id": card.id,
                "candidate_name": card.candidate_name,
                "candidate_phone": card.candidate_phone,
                "candidate_city": card.candidate_city,
                "candidate_metro": card.candidate_metro,
                "candidate_age": card.candidate_age,
                "vacancy_title": card.vacancy_title,
                "result": card.result,
                "callback_slot": card.callback_slot,
                "dialog_summary": card.dialog_summary,
                "messages_count": card.messages_count,
                "is_processed": card.is_processed,
                "created_at": str(card.created_at) if card.created_at else None,
                "account_name": acct_name or "",
            }
            for card, acct_name in cards_rows
        ],
    }


@router.get("/api/handover/{card_id}/messages", dependencies=[Depends(_verify_admin)])
async def get_handover_messages(card_id: int):
    async with AsyncSessionFactory() as session:
        card = await session.get(HandoverCard, card_id)
        if not card:
            raise HTTPException(status_code=404, detail="Card not found")

        application = await session.get(Application, card.application_id)
        if not application:
            raise HTTPException(status_code=404, detail="Application not found")

        chat_result = await session.execute(
            select(Chat).where(Chat.application_id == application.id).limit(1)
        )
        chat = chat_result.scalar_one_or_none()
        if not chat:
            return {"card_id": card_id, "candidate_name": card.candidate_name, "messages": []}

        msg_result = await session.execute(
            select(Message)
            .where(Message.chat_id == chat.id)
            .order_by(Message.created_at.asc())
        )
        messages = msg_result.scalars().all()

    return {
        "card_id": card_id,
        "candidate_name": card.candidate_name,
        "messages": [
            {
                "sender": m.sender_type,
                "text": m.content,
                "sent_at": str(m.delivered_at or m.created_at) if (m.delivered_at or m.created_at) else None,
            }
            for m in messages
        ],
    }


@router.post("/api/handover/{card_id}/process", dependencies=[Depends(_verify_admin)])
async def process_handover_card(card_id: int):
    async with AsyncSessionFactory() as session:
        card = await session.get(HandoverCard, card_id)
        if not card:
            raise HTTPException(status_code=404, detail="Card not found")
        card.is_processed = True
        card.processed_at = datetime.utcnow()
        await session.commit()

    return {"status": "ok", "card_id": card_id}


@router.post("/api/telegram/test-summary", dependencies=[Depends(_verify_admin)])
async def test_telegram_summary(account_id: Optional[int] = None):
    from services.telegram_notifier import send_morning_summary
    sent = await send_morning_summary(account_id=account_id)
    return {"status": "ok", "sent": sent}


WEBHOOK_URL = "https://babito.kadry-24.ru/webhooks/avito"


async def _register_webhooks_for_account(account_id: int) -> bool:
    token = await get_valid_token(account_id)

    messenger_ok = False
    job_ok = False

    async with httpx.AsyncClient(timeout=30) as client:
        try:
            resp = await client.post(
                "https://api.avito.ru/messenger/v3/webhook",
                headers={"Authorization": f"Bearer {token}"},
                json={"url": WEBHOOK_URL},
            )
            messenger_ok = resp.status_code in (200, 201)
            log.info(
                "webhook_messenger_registered",
                account_id=account_id,
                status=resp.status_code,
                body=resp.text[:200],
            )
        except Exception as exc:
            log.error("webhook_messenger_register_failed", account_id=account_id, error=str(exc))

        try:
            resp = await client.put(
                "https://api.avito.ru/job/v1/applications/webhook",
                headers={"Authorization": f"Bearer {token}"},
                json={"url": WEBHOOK_URL},
            )
            job_ok = resp.status_code in (200, 201)
            log.info(
                "webhook_job_registered",
                account_id=account_id,
                status=resp.status_code,
                body=resp.text[:200],
            )
        except Exception as exc:
            log.error("webhook_job_register_failed", account_id=account_id, error=str(exc))

    if messenger_ok or job_ok:
        async with AsyncSessionFactory() as session:
            account = await session.get(AvitoAccount, account_id)
            if account:
                account.webhook_registered = True
                await session.commit()

    return messenger_ok and job_ok


@router.get("/api/queues", dependencies=[Depends(_verify_admin)])
async def get_queue_status():
    try:
        from services.redis_queue import (
            get_stream_info, STREAM_MESSENGER, STREAM_APPLICATIONS,
        )
        messenger = await get_stream_info(STREAM_MESSENGER)
        applications = await get_stream_info(STREAM_APPLICATIONS)
        return {
            "messenger": messenger,
            "applications": applications,
        }
    except Exception as exc:
        return {
            "messenger": {"error": str(exc)},
            "applications": {"error": str(exc)},
        }
```

---

### 2.6 api/admin_web.py

```python
"""Veb-interfeys adminki -- HTML-routy, login/logout, cookie-sessii."""
import hashlib
import hmac
import time

from fastapi import APIRouter, Request, Form, Depends
from fastapi.responses import HTMLResponse, RedirectResponse
from fastapi.templating import Jinja2Templates

from config import settings
from utils.logger import get_logger

log = get_logger(__name__)

router = APIRouter(prefix="/admin", tags=["admin-web"])
templates = Jinja2Templates(directory="templates")

COOKIE_NAME = "admin_session"
COOKIE_MAX_AGE = 86400  # 24 hours


def _sign_cookie(timestamp: str) -> str:
    return hmac.new(
        settings.admin_secret_key.encode(),
        timestamp.encode(),
        hashlib.sha256,
    ).hexdigest()


def _create_session_cookie() -> str:
    ts = str(int(time.time()))
    sig = _sign_cookie(ts)
    return f"{ts}:{sig}"


def _validate_cookie(value: str) -> bool:
    try:
        parts = value.split(":", 1)
        if len(parts) != 2:
            return False
        ts_str, sig = parts
        ts = int(ts_str)
        if time.time() - ts > COOKIE_MAX_AGE:
            return False
        expected = _sign_cookie(ts_str)
        return hmac.compare_digest(sig, expected)
    except Exception:
        return False


def verify_admin_cookie(request: Request) -> bool:
    cookie = request.cookies.get(COOKIE_NAME)
    if cookie and _validate_cookie(cookie):
        return True
    return False


def verify_admin(request: Request) -> bool:
    if verify_admin_cookie(request):
        return True
    auth = request.headers.get("Authorization", "")
    if settings.admin_token and auth == f"Bearer {settings.admin_token}":
        return True
    return False


@router.get("/login", response_class=HTMLResponse)
async def login_page(request: Request, error: str = ""):
    return templates.TemplateResponse("login.html", {
        "request": request,
        "error": error,
    })


@router.post("/login")
async def login_submit(
    request: Request,
    username: str = Form(...),
    password: str = Form(...),
):
    if username == settings.admin_login and password == settings.admin_password:
        response = RedirectResponse(url="/admin/", status_code=302)
        response.set_cookie(
            key=COOKIE_NAME,
            value=_create_session_cookie(),
            max_age=COOKIE_MAX_AGE,
            httponly=True,
            samesite="lax",
        )
        log.info("admin_login_success", username=username)
        return response

    log.warning("admin_login_failed", username=username)
    return templates.TemplateResponse("login.html", {
        "request": request,
        "error": "Nevernyy login ili parol'",
    })


@router.get("/logout")
async def logout(request: Request):
    response = RedirectResponse(url="/admin/login", status_code=302)
    response.delete_cookie(COOKIE_NAME)
    return response


@router.get("/", response_class=HTMLResponse)
async def admin_page(request: Request):
    if not verify_admin_cookie(request):
        return RedirectResponse(url="/admin/login", status_code=302)
    return templates.TemplateResponse("admin.html", {"request": request})
```

---

### 2.7 services/ai_agent.py

```python
"""Osnovnaya logika AI-dialoga -- 6 blokov (v2).

Flow: greeting -> waiting_qualification -> presentation+waiting_fork ->
      booking/alternatives -> waiting_booking -> handover
"""
import json
from datetime import datetime
from pathlib import Path
from typing import Optional

from sqlalchemy import select, func

from models.db import (
    AsyncSessionFactory, AISession, Application, Applicant,
    Chat, Message, Vacancy,
)
from services.ai_claude import ask_claude, call_claude
from services.ai_rag import search_vacancies
from services.handover import create_handover_card
from services.message_scheduler import schedule_message
from services.vacancy_sync import ensure_vacancy
from utils.logger import get_logger
from utils.time_helpers import calc_greeting_delay, calc_message_delay

log = get_logger(__name__)

PROMPTS_DIR = Path(__file__).parent.parent / "prompts"

AGENT_NAME = "Elena"
COMPANY_NAME = "Kadry 24"


def _load_prompt(name: str) -> str:
    path = PROMPTS_DIR / f"{name}.txt"
    if path.exists():
        return path.read_text(encoding="utf-8").strip()
    return ""


QUALIFICATION_PARSE_PROMPT = """Izvleki iz soobshcheniya soiskatelya:
- city (gorod, stroka ili null)
- metro (stantsiya metro ili rayon, stroka ili null)
- age (vozrast, chislo ili null)

Soobshchenie: "{message}"

Otvet' TOL'KO JSON: {{"city": "...", "metro": "...", "age": ...}}"""

INTENT_PARSE_PROMPT = """Opredeli namerenie soiskatelya.
Soobshchenie: "{message}"

Varianty:
- "ok" -- soglasen, khochet zapisat'sya
- "no" -- ne soglasen, khochet drugie varianty
- "unclear" -- neponyatno

Otvet' TOL'KO JSON: {{"intent": "ok|no|unclear"}}"""

BOOKING_PARSE_PROMPT = """Izvleki iz soobshcheniya kandidata udobnoe vremya dlya zvonka.
Soobshchenie: "{message}"
Esli kandidat ukazal LYUBOE vremya, interval ili period -- verni kak est'.
Esli kandidat NE ukazal vremya ili otkazalsya -- verni null.
Otvet' TOL'KO JSON: {{"callback_slot": "..."}}"""


def _extract_json(text: str) -> dict:
    import re
    text = text.strip()
    md_match = re.search(r"```(?:json)?\s*([\s\S]*?)```", text)
    if md_match:
        text = md_match.group(1).strip()
    try:
        return json.loads(text)
    except (json.JSONDecodeError, ValueError):
        pass
    brace_match = re.search(r"\{[^{}]*\}", text)
    if brace_match:
        try:
            return json.loads(brace_match.group())
        except (json.JSONDecodeError, ValueError):
            pass
    raise ValueError(f"Cannot extract JSON from: {text[:200]}")


async def _get_dialog_history(chat_db_id: int, limit: int = 10) -> list[dict]:
    async with AsyncSessionFactory() as session:
        result = await session.execute(
            select(Message)
            .where(Message.chat_id == chat_db_id)
            .order_by(Message.created_at.desc())
            .limit(limit)
        )
        messages = list(reversed(result.scalars().all()))
    return [
        {"role": "user" if m.sender_type == "applicant" else "assistant", "content": m.content}
        for m in messages
    ]


async def _calc_candidate_response_sec(chat_db_id: int) -> int | None:
    async with AsyncSessionFactory() as session:
        result = await session.execute(
            select(Message)
            .where(Message.chat_id == chat_db_id, Message.direction == "outgoing")
            .order_by(Message.created_at.desc())
            .limit(1)
        )
        last_outgoing = result.scalar_one_or_none()

        result = await session.execute(
            select(Message)
            .where(Message.chat_id == chat_db_id, Message.direction == "incoming")
            .order_by(Message.created_at.desc())
            .limit(1)
        )
        last_incoming = result.scalar_one_or_none()

    if not last_outgoing or not last_incoming:
        return None

    bot_time = last_outgoing.delivered_at or last_outgoing.created_at
    candidate_time = last_incoming.created_at

    if candidate_time <= bot_time:
        return None

    diff = (candidate_time - bot_time).total_seconds()
    return int(diff) if diff > 0 else None


async def _build_system_prompt(
    ai_session: AISession,
    vacancy_data: dict | None = None,
) -> str:
    template = _load_prompt("system")

    vacancy_title = (vacancy_data or {}).get("title", "ne ukazana")
    city = (vacancy_data or {}).get("city", "")

    vacancy_details = ""
    if vacancy_data:
        parts = []
        if vacancy_data.get("business_area"):
            parts.append(f"Sfera: {vacancy_data['business_area']}")
        if vacancy_data.get("profession"):
            parts.append(f"Professiya: {vacancy_data['profession']}")
        if vacancy_data.get("schedule"):
            parts.append(f"Grafik: {vacancy_data['schedule']}")
        if vacancy_data.get("salary_raw"):
            parts.append(f"Oplata: {vacancy_data['salary_raw']}")
        if vacancy_data.get("parsed_tasks"):
            parts.append(f"Zadachi: {vacancy_data['parsed_tasks']}")
        vacancy_details = "\n".join(parts)

    collected = (
        f"gorod={ai_session.collected_city or '?'}, "
        f"metro={ai_session.collected_metro or '?'}, "
        f"vozrast={ai_session.collected_age or '?'}"
    )

    return (
        template
        .replace("{vacancy_title}", vacancy_title)
        .replace("{city}", city)
        .replace("{vacancy_details}", vacancy_details)
        .replace("{dialog_stage}", ai_session.dialog_stage or "")
        .replace("{collected_data}", collected)
    )


async def _get_vacancy_data(application: Application) -> dict | None:
    avito_vid = application.avito_vacancy_id
    if not avito_vid:
        return None
    try:
        return await ensure_vacancy(int(avito_vid), account_id=application.account_id)
    except Exception as exc:
        log.error("ensure_vacancy_failed", vacancy_id=avito_vid, error=str(exc))
        return None


# -- Tochki vkhoda --

async def process_new_application(application_id: int) -> None:
    """Blok 1: Privetstvie + voprosy (gorod/metro, vozrast)."""
    async with AsyncSessionFactory() as session:
        application = await session.get(Application, application_id)
        if not application:
            log.error("application_not_found", application_id=application_id)
            return

        result = await session.execute(
            select(AISession).where(
                AISession.application_id == application_id,
                AISession.status == "active",
            )
        )
        ai_session = result.scalar_one_or_none()
        if not ai_session:
            log.error("ai_session_not_found", application_id=application_id)
            return

        chat = await session.get(Chat, ai_session.chat_id)
        if not chat:
            log.error("chat_not_found", application_id=application_id)
            return

    vacancy_data = await _get_vacancy_data(application)
    vacancy_title = (vacancy_data or {}).get("title", "eta vakansiya")

    greeting_prompt = _load_prompt("greeting").replace("{vacancy_title}", vacancy_title)
    system = await _build_system_prompt(ai_session, vacancy_data)

    try:
        greeting_text = await ask_claude(
            system=system,
            messages=[{"role": "user", "content": greeting_prompt}],
            session_id=ai_session.id,
            application_id=application_id,
            dialog_stage="greeting",
        )

        delay = calc_greeting_delay()
        await schedule_message(chat.id, greeting_text, delay)

        async with AsyncSessionFactory() as session:
            sess = await session.get(AISession, ai_session.id)
            if sess:
                sess.dialog_stage = "waiting_qualification"
                await session.commit()

        log.info("greeting_scheduled", ai_session_id=ai_session.id, delay=delay)

    except Exception as exc:
        log.error("greeting_failed", error=str(exc))
        try:
            from utils.event_logger import log_event
            await log_event(application.account_id, "session_error", f"Oshibka AI sessii {ai_session.id}: {exc}")
        except Exception:
            pass
        await _mark_session_failed(ai_session.id, error=exc)


async def process_incoming_message(message_id: int) -> None:
    """Marshrutizator vkhodayshchikh soobshcheniy po dialog_stage."""
    async with AsyncSessionFactory() as session:
        message = await session.get(Message, message_id)
        if not message:
            return

        chat = await session.get(Chat, message.chat_id)
        if not chat:
            return

        result = await session.execute(
            select(AISession).where(
                AISession.chat_id == chat.id,
                AISession.status == "active",
            )
        )
        ai_session = result.scalar_one_or_none()
        if not ai_session:
            return

        application = await session.get(Application, ai_session.application_id)
        if not application:
            return

        ai_session.last_applicant_msg_at = datetime.utcnow()
        await session.commit()

    candidate_speed = await _calc_candidate_response_sec(chat.id)
    if candidate_speed:
        log.info("candidate_response_speed", seconds=candidate_speed, chat_id=chat.id)

    stage = ai_session.dialog_stage
    log.info("processing_stage", stage=stage, message_id=message_id)

    if stage == "waiting_qualification":
        await _handle_qualification(ai_session, application, chat, message, candidate_speed)
    elif stage == "waiting_fork":
        await _handle_fork(ai_session, application, chat, message, candidate_speed)
    elif stage == "alternatives":
        await _handle_alternatives_response(ai_session, application, chat, message, candidate_speed)
    elif stage == "waiting_booking":
        await _handle_booking_response(ai_session, application, chat, message, candidate_speed)
    elif stage == "clarify":
        await _handle_clarify_response(ai_session, application, chat, message, candidate_speed)
    elif stage == "followup":
        await _handle_followup_response(ai_session, application, chat, message, candidate_speed)
    else:
        log.debug("message_ignored_stage", stage=stage, msg_id=message_id)


async def process_followup(ai_session_id: int) -> None:
    """Blok 5: Follow-up pri molchanii."""
    from config import settings

    async with AsyncSessionFactory() as session:
        ai_session = await session.get(AISession, ai_session_id)
        if not ai_session or ai_session.status != "active":
            return

        if ai_session.followup_count >= settings.ai_max_followups:
            ai_session.result = "no_response"
            await session.commit()
            await create_handover_card(ai_session_id, result="no_response")
            return

        application = await session.get(Application, ai_session.application_id)
        chat = await session.get(Chat, ai_session.chat_id)

        ai_session.followup_count += 1
        ai_session.dialog_stage = "followup"
        await session.commit()

    vacancy_data = await _get_vacancy_data(application) if application else None
    vacancy_title = (vacancy_data or {}).get("title", "vakansiya")

    followup_prompt = (
        _load_prompt("followup")
        .replace("{followup_number}", str(ai_session.followup_count))
        .replace("{vacancy_title}", vacancy_title)
    )

    system = await _build_system_prompt(ai_session, vacancy_data)
    history = await _get_dialog_history(chat.id)
    messages = history + [{"role": "user", "content": followup_prompt}]

    try:
        reply = await ask_claude(
            system=system,
            messages=messages,
            session_id=ai_session_id,
            application_id=application.id if application else None,
            dialog_stage="followup",
        )
        delay = calc_message_delay(candidate_response_sec=candidate_speed)
        await schedule_message(chat.id, reply, delay)
    except Exception as exc:
        log.error("followup_failed", ai_session_id=ai_session_id, error=str(exc))
        await _mark_session_failed(ai_session_id, error=exc)


# -- Stage handlers (qualification, presentation, fork, booking, alternatives, clarify, followup) --
# ... (full implementation ~700 lines, see services/ai_agent.py)


_LLM_ERROR_MARKERS = ["529", "502", "503", "429", "500", "anthropic", "openai", "timeout", "fallback"]


def _is_llm_error(error) -> bool:
    error_str = str(error).lower()
    return any(marker in error_str for marker in _LLM_ERROR_MARKERS)


async def _mark_session_failed(ai_session_id: int, error=None) -> None:
    if error and _is_llm_error(error):
        log.warning(
            "session_llm_error_skip_failed",
            ai_session_id=ai_session_id,
            error=str(error)[:200],
        )
        try:
            from utils.event_logger import log_event
            await log_event(None, "session_llm_skip", f"Sessiya {ai_session_id}: LLM-oshibka, NE pomechena failed")
        except Exception:
            pass
        return

    async with AsyncSessionFactory() as session:
        sess = await session.get(AISession, ai_session_id)
        if sess:
            sess.status = "failed"
            sess.dialog_stage = "failed"
            await session.commit()

            try:
                from utils.event_logger import log_event
                app = await session.get(Application, sess.application_id)
                account_id = app.account_id if app else None
                await log_event(account_id, "session_error", f"AI sessiya {ai_session_id} zavershena s oshibkoy")
            except Exception:
                pass

    log.error("session_marked_failed", ai_session_id=ai_session_id)
```

> Note: Full file is 1077 lines. Stage handlers (_handle_qualification, _send_presentation_and_fork, _handle_fork, _send_booking, _handle_booking_response, _send_alternatives, _handle_alternatives_response, _send_clarify, _handle_clarify_response, _handle_followup_response) omitted for brevity -- see source file for full implementation.

---

### 2.8 services/avito_messenger.py

```python
"""Otpravka i poluchenie soobshcheniy cherez Avito Messenger API."""
import asyncio
from datetime import datetime
from typing import Optional

import httpx
from sqlalchemy import select

from models.db import AsyncSessionFactory, AvitoAccount, Chat, Message
from services.avito_auth import get_valid_token
from utils.logger import get_logger

log = get_logger(__name__)

MESSENGER_BASE = "https://api.avito.ru/messenger/v1/accounts"


def _avito_client() -> httpx.AsyncClient:
    return httpx.AsyncClient(timeout=30)


async def _get_avito_user_id(account_id: int) -> str:
    async with AsyncSessionFactory() as session:
        account = await session.get(AvitoAccount, account_id)
        if not account:
            raise ValueError(f"Account {account_id} not found")
        return account.avito_user_id


async def send_message(account_id: int, chat_id: str, text: str, skip_db: bool = False) -> dict:
    token = await get_valid_token(account_id)
    user_id = await _get_avito_user_id(account_id)

    url = f"{MESSENGER_BASE}/{user_id}/chats/{chat_id}/messages"
    payload = {"message": {"text": text}, "type": "text"}

    result = await _request_with_retry(
        method="POST",
        url=url,
        token=token,
        json=payload,
    )

    if not skip_db:
        async with AsyncSessionFactory() as session:
            chat_row = await session.execute(
                select(Chat).where(Chat.avito_chat_id == chat_id)
            )
            chat_obj = chat_row.scalar_one_or_none()
            if chat_obj:
                msg = Message(
                    chat_id=chat_obj.id,
                    avito_message_id=result.get("id"),
                    direction="outgoing",
                    sender_type="ai",
                    content=text,
                    delivered_at=datetime.utcnow(),
                )
                session.add(msg)
                chat_obj.last_message_at = datetime.utcnow()
                await session.commit()

    log.info("message_sent", account_id=account_id, chat_id=chat_id, length=len(text))

    try:
        from utils.event_logger import log_event
        await log_event(account_id, "message_sent", f"Soobshchenie otpravleno v chat {chat_id}")
    except Exception:
        pass

    return result


async def get_messages(account_id: int, chat_id: str) -> list[dict]:
    token = await get_valid_token(account_id)
    user_id = await _get_avito_user_id(account_id)

    url = f"{MESSENGER_BASE}/{user_id}/chats/{chat_id}/messages"
    result = await _request_with_retry(method="GET", url=url, token=token)
    return result.get("messages", [])


async def get_chat_info(account_id: int, chat_id: str) -> dict:
    token = await get_valid_token(account_id)
    user_id = await _get_avito_user_id(account_id)

    url = f"{MESSENGER_BASE}/{user_id}/chats/{chat_id}"
    return await _request_with_retry(method="GET", url=url, token=token)


async def _request_with_retry(
    method: str,
    url: str,
    token: str,
    json: Optional[dict] = None,
    max_retries: int = 3,
) -> dict:
    headers = {"Authorization": f"Bearer {token}"}
    last_exc: Optional[Exception] = None

    async with _avito_client() as client:
        for attempt in range(max_retries):
            try:
                if method == "GET":
                    resp = await client.get(url, headers=headers)
                else:
                    resp = await client.post(url, headers=headers, json=json)

                if resp.status_code == 401:
                    raise httpx.HTTPStatusError(
                        "401 Unauthorized", request=resp.request, response=resp
                    )

                if resp.status_code == 429:
                    log.warning("avito_rate_limit", attempt=attempt)
                    await asyncio.sleep(30)
                    continue

                if resp.status_code >= 500:
                    log.warning("avito_server_error", status=resp.status_code, attempt=attempt)
                    await asyncio.sleep(10)
                    continue

                resp.raise_for_status()
                return resp.json()

            except httpx.HTTPStatusError as exc:
                last_exc = exc
                if attempt < max_retries - 1:
                    await asyncio.sleep(10)
            except Exception as exc:
                last_exc = exc
                if attempt < max_retries - 1:
                    await asyncio.sleep(5)

    raise last_exc or RuntimeError(f"Failed after {max_retries} attempts: {url}")
```

---

### 2.9 services/message_scheduler.py

```python
"""Planirovshchik otlozhennoy otpravki soobshcheniy s imitatsiey pauz zhivogo cheloveka."""
import asyncio
from datetime import datetime, timedelta
from typing import Optional

from sqlalchemy import select

from models.db import AsyncSessionFactory, Message, Chat, AISession
from services.avito_messenger import send_message
from utils.logger import get_logger
from utils.time_helpers import calc_message_delay

log = get_logger(__name__)


async def schedule_message(
    chat_db_id: int,
    text: str,
    delay_sec: int,
    sender_type: str = "ai",
    account_id: Optional[int] = None,
) -> int:
    scheduled_at = datetime.utcnow() + timedelta(seconds=delay_sec)

    async with AsyncSessionFactory() as session:
        msg = Message(
            chat_id=chat_db_id,
            direction="outgoing",
            sender_type=sender_type,
            content=text,
            scheduled_at=scheduled_at,
        )
        session.add(msg)
        await session.commit()
        await session.refresh(msg)
        log.debug("message_scheduled", chat_db_id=chat_db_id, delay_sec=delay_sec, msg_id=msg.id)
        return msg.id


async def schedule_message_for_chat(
    avito_chat_id: str,
    text: str,
    incoming_text: str = "",
    sender_type: str = "ai",
) -> Optional[int]:
    delay = calc_message_delay(incoming_text)

    async with AsyncSessionFactory() as session:
        result = await session.execute(
            select(Chat).where(Chat.avito_chat_id == avito_chat_id)
        )
        chat = result.scalar_one_or_none()
        if not chat:
            log.error("chat_not_found_for_schedule", avito_chat_id=avito_chat_id)
            return None

    return await schedule_message(chat.id, text, delay, sender_type)


async def has_pending_messages(chat_db_id: int) -> bool:
    async with AsyncSessionFactory() as session:
        result = await session.execute(
            select(Message).where(
                Message.chat_id == chat_db_id,
                Message.direction == "outgoing",
                Message.delivered_at == None,
                Message.scheduled_at != None,
            )
        )
        return result.scalar_one_or_none() is not None


async def process_scheduled() -> None:
    """Proveryaet ochered' otlozhennykh soobshcheniy i otpravlyaet gotovye. Kazhdye 5 sek."""
    now = datetime.utcnow()

    async with AsyncSessionFactory() as session:
        result = await session.execute(
            select(Message)
            .where(
                Message.direction == "outgoing",
                Message.scheduled_at <= now,
                Message.delivered_at == None,
            )
            .limit(50)
        )
        messages = result.scalars().all()

    for msg in messages:
        try:
            async with AsyncSessionFactory() as session:
                chat = await session.get(Chat, msg.chat_id)
                if not chat:
                    log.error("chat_not_found", msg_id=msg.id)
                    continue

                result = await session.execute(
                    select(AISession).where(
                        AISession.chat_id == chat.id,
                    ).order_by(AISession.id.desc()).limit(1)
                )
                ai_session = result.scalar_one_or_none()

                if ai_session and ai_session.status != "active":
                    db_msg = await session.get(Message, msg.id)
                    if db_msg:
                        db_msg.delivered_at = now
                        await session.commit()
                    log.info(
                        "scheduled_message_skipped_inactive",
                        msg_id=msg.id,
                        chat_id=msg.chat_id,
                        session_status=ai_session.status,
                    )
                    continue

                avito_chat_id = chat.avito_chat_id
                account_id = chat.account_id

            await send_message(account_id, avito_chat_id, msg.content, skip_db=True)

            async with AsyncSessionFactory() as session:
                db_msg = await session.get(Message, msg.id)
                if db_msg:
                    db_msg.delivered_at = datetime.utcnow()
                    await session.commit()

            log.info("scheduled_message_delivered", msg_id=msg.id, chat_id=msg.chat_id)

        except Exception as exc:
            log.error("scheduled_message_failed", msg_id=msg.id, error=str(exc))
```

---

### 2.10 services/handover.py

```python
"""Formirovanie kartochek peredachi soiskateeley menedzheram (v2)."""
from datetime import datetime

from sqlalchemy import select, func

from models.db import (
    AsyncSessionFactory, AISession, Application, Applicant,
    Message, Chat, Vacancy, HandoverCard,
)
from services.ai_claude import ask_claude
from utils.logger import get_logger

log = get_logger(__name__)


async def create_handover_card(ai_session_id: int, result: str = "unknown") -> int:
    async with AsyncSessionFactory() as session:
        ai_session = await session.get(AISession, ai_session_id)
        if not ai_session:
            raise ValueError(f"AISession {ai_session_id} not found")

        application = await session.get(Application, ai_session.application_id)
        applicant = await session.get(Applicant, application.applicant_id) if application else None
        chat = await session.get(Chat, ai_session.chat_id)

        vacancy = None
        if application and application.avito_vacancy_id:
            vac_result = await session.execute(
                select(Vacancy).where(
                    Vacancy.avito_vacancy_id == application.avito_vacancy_id
                )
            )
            vacancy = vac_result.scalar_one_or_none()

        msg_result = await session.execute(
            select(Message)
            .where(Message.chat_id == ai_session.chat_id)
            .order_by(Message.created_at.asc())
        )
        messages = msg_result.scalars().all()

        messages_count = len(messages)

    dialog_lines = []
    for m in messages:
        sender = "Soiskatel'" if m.sender_type == "applicant" else "Elena"
        dialog_lines.append(f"{sender}: {m.content}")
    dialog_text = "\n".join(dialog_lines[-20:])

    candidate_name = ""
    candidate_phone = ""
    if application:
        candidate_name = application.applicant_name or ""
        candidate_phone = application.applicant_phone or ""
    if not candidate_name and applicant:
        candidate_name = applicant.name or ""
    if not candidate_phone and applicant:
        candidate_phone = applicant.phone or ""

    collected = (
        f"Imya: {candidate_name or 'ne ukazano'}, "
        f"Telefon: {candidate_phone or 'ne ukazan'}, "
        f"Gorod: {ai_session.collected_city or 'ne ukazan'}, "
        f"Metro: {ai_session.collected_metro or 'ne ukazano'}, "
        f"Vozrast: {ai_session.collected_age or 'ne ukazan'}"
    )
    callback_info = f"Slot zvonka: {ai_session.callback_slot}" if ai_session.callback_slot else ""
    result_info = f"Rezul'tat: {result}"

    summary_prompt = (
        f"Dannye soiskatelya: {collected}\n"
        f"{result_info}. {callback_info}\n\n"
        f"Dialog:\n{dialog_text}\n\n"
        "Napishi kratkoe rezyume (2-3 predlozheniya): kto soiskatel', chto obsuzhdali, itog."
    )

    try:
        dialog_summary = await ask_claude(
            system="Ty -- HR-assistent. Pishi lakonichno i po delu.",
            messages=[{"role": "user", "content": summary_prompt}],
            session_id=ai_session_id,
            application_id=ai_session.application_id,
            dialog_stage="handover",
            temperature=0.3,
            max_tokens=300,
        )
    except Exception as exc:
        log.error("handover_summary_failed", error=str(exc))
        dialog_summary = collected

    async with AsyncSessionFactory() as session:
        card = HandoverCard(
            ai_session_id=ai_session_id,
            application_id=ai_session.application_id,
            avito_vacancy_id=application.avito_vacancy_id if application else None,
            vacancy_title=vacancy.title if vacancy else None,
            candidate_name=candidate_name or None,
            candidate_phone=candidate_phone or None,
            candidate_city=ai_session.collected_city,
            candidate_metro=ai_session.collected_metro,
            candidate_age=ai_session.collected_age,
            result=result,
            callback_slot=ai_session.callback_slot,
            dialog_summary=dialog_summary,
            block=ai_session.assigned_block,
            messages_count=messages_count,
            is_processed=False,
        )
        session.add(card)

        app = await session.get(Application, ai_session.application_id)
        if app:
            app.status = "ai_done"

        sess = await session.get(AISession, ai_session_id)
        if sess:
            sess.dialog_stage = "completed"
            sess.status = "completed"
            sess.result = result

        await session.commit()
        await session.refresh(card)

    log.info(
        "handover_card_created",
        card_id=card.id,
        ai_session_id=ai_session_id,
        result=result,
        callback_slot=ai_session.callback_slot,
    )

    try:
        from utils.event_logger import log_event
        account_id = application.account_id if application else None
        await log_event(
            account_id, "session_completed",
            f"Sessiya {ai_session_id} zavershena, rezul'tat: {result}",
        )
    except Exception:
        pass

    return card.id
```

---

### 2.11 services/telegram_notifier.py

```python
"""Otpravka kartochek menedzheram v Telegram."""
import asyncio
from datetime import datetime, timedelta

import httpx
import pytz
from config import settings
from utils.logger import get_logger

log = get_logger(__name__)


async def send_telegram_message(topic_id: int | None, text: str, parse_mode: str = "HTML") -> bool:
    if not settings.telegram_bot_token or not settings.telegram_group_id:
        log.warning("telegram_not_configured")
        return False

    api_url = f"https://api.telegram.org/bot{settings.telegram_bot_token}/sendMessage"

    try:
        payload = {
            "chat_id": int(settings.telegram_group_id),
            "text": text,
            "parse_mode": parse_mode,
        }
        if topic_id is not None and topic_id > 0:
            payload["message_thread_id"] = int(topic_id)

        async with httpx.AsyncClient(timeout=15.0) as client:
            resp = await client.post(api_url, json=payload)
            resp.raise_for_status()
            return True
    except Exception as e:
        log.error("telegram_send_error", topic_id=topic_id, error=str(e))
        return False


def format_card_for_telegram(card) -> str:
    result_labels = {
        "booking": "[Zapisan na zvonok]",
        "interested": "[Zainteresovan]",
        "not_interested": "[Ne interesno]",
        "alternative": "[Nuzhny alternativy]",
    }
    label = result_labels.get(card.result, card.result or "--")

    slot = f"\nSlot: {card.callback_slot}" if card.callback_slot else ""
    metro = f" (m. {card.candidate_metro})" if card.candidate_metro else ""
    age = f", {card.candidate_age} let" if card.candidate_age else ""

    return (
        f"<b>{card.candidate_name}{age}</b>\n"
        f"Tel: <code>{card.candidate_phone}</code>\n"
        f"Gorod: {card.candidate_city or '--'}{metro}\n"
        f"Vakansiya: {card.vacancy_title}{slot}\n"
        f"Rezultat: {label}\n"
        f"\n{card.dialog_summary or ''}"
    )


async def send_morning_summary(account_id: int | None = None) -> int:
    if not settings.telegram_bot_token or not settings.telegram_group_id:
        log.warning("telegram_morning_skip_not_configured")
        return 0

    from sqlalchemy import select
    from models.db import AsyncSessionFactory, HandoverCard, Application, AvitoAccount

    moscow_tz = pytz.timezone("Europe/Moscow")
    now_moscow = datetime.now(moscow_tz)
    night_start = now_moscow.replace(hour=21, minute=0, second=0, microsecond=0) - timedelta(days=1)
    night_end = now_moscow.replace(hour=9, minute=0, second=0, microsecond=0)

    night_start_utc = night_start.astimezone(pytz.utc).replace(tzinfo=None)
    night_end_utc = night_end.astimezone(pytz.utc).replace(tzinfo=None)

    async with AsyncSessionFactory() as session:
        if account_id:
            accounts_query = select(AvitoAccount).where(
                AvitoAccount.id == account_id,
                AvitoAccount.is_active == True,
                AvitoAccount.telegram_topic_id != None,
            )
        else:
            accounts_query = select(AvitoAccount).where(
                AvitoAccount.is_active == True,
                AvitoAccount.telegram_topic_id != None,
            )

        accounts_result = await session.execute(accounts_query)
        accounts = accounts_result.scalars().all()

    total_sent = 0

    for account in accounts:
        async with AsyncSessionFactory() as session:
            result = await session.execute(
                select(HandoverCard)
                .join(Application, HandoverCard.application_id == Application.id)
                .where(
                    Application.account_id == account.id,
                    HandoverCard.is_processed == False,
                    HandoverCard.created_at >= night_start_utc,
                    HandoverCard.created_at <= night_end_utc,
                )
                .order_by(HandoverCard.created_at.desc())
            )
            cards = result.scalars().all()

        if not cards:
            await send_telegram_message(
                account.telegram_topic_id,
                "<b>Utrennyaya svodka</b>\n\nNovykh kartochek za noch net.",
            )
            continue

        header = (
            f"<b>Utrennyaya svodka -- {len(cards)} kartochek</b>\n"
            f"{'--' * 30}\n"
        )
        await send_telegram_message(account.telegram_topic_id, header)

        sent = 0
        for card in cards:
            text = format_card_for_telegram(card)
            success = await send_telegram_message(account.telegram_topic_id, text)
            if success:
                sent += 1
            await asyncio.sleep(0.3)

        booking_count = sum(1 for c in cards if c.result == "booking")
        footer = f"\n{'--' * 30}\nVsego: {len(cards)} | Zapisi: {booking_count}"
        await send_telegram_message(account.telegram_topic_id, footer)

        total_sent += sent

    log.info("telegram_morning_sent", total=total_sent)

    try:
        from utils.event_logger import log_event
        await log_event(
            None, "telegram_summary",
            f"Utrennyaya svodka: {total_sent} kartochek otpravleno",
        )
    except Exception:
        pass

    return total_sent
```

---

### 2.12 templates/admin.html

> Full file: 828 lines. See source. Key JS section handles:
> - UTC->Moscow timezone conversion (`toMoscow()`)
> - Account CRUD with modal forms
> - Handover cards with dialog viewer modal
> - Event log with auto-refresh (10s)
> - Stats polling (30s)

---

### 2.13 scripts/webhook_enable_ours.py

```python
import sys; sys.path.insert(0, "/opt/openai/crm-worker")
"""Ubrat' Bitriks, vklyuchit' nash vebkhuk dlya akkaunta Lavka (id=2)."""
import asyncio, subprocess, warnings
warnings.filterwarnings('ignore', 'Exception ignored')

import httpx
from services.avito_auth import get_valid_token
from models.db import engine

BITRIX_URL = "https://b24.kadry-24.ru/local/source/avito/avito_new_chat2.php"
OUR_URL = "https://babito.kadry-24.ru/webhooks/avito"
ACCOUNT_ID = 2
BITRIX_REST = "https://b24.kadry-24.ru/rest/8539/3j61afi1yqk890du"
BITRIX_LINE_ID = 12

async def main():
    token = await get_valid_token(ACCOUNT_ID)
    headers = {"Authorization": f"Bearer {token}"}
    async with httpx.AsyncClient(timeout=15) as client:
        r1 = await client.post(
            "https://api.avito.ru/messenger/v1/webhook/unsubscribe",
            headers=headers, json={"url": BITRIX_URL},
        )
        print(f"Unsubscribe Bitrix: {r1.status_code} {r1.text}")

        r2 = await client.post(
            "https://api.avito.ru/messenger/v3/webhook",
            headers=headers, json={"url": OUR_URL},
        )
        print(f"Subscribe ours: {r2.status_code} {r2.text}")

    r3 = subprocess.run([
        "curl", "-s", "-X", "POST",
        f"{BITRIX_REST}/imopenlines.config.update",
        "-H", "Content-Type: application/json",
        "-d", '{"CONFIG_ID": ' + str(BITRIX_LINE_ID) + ', "PARAMS": {"ACTIVE": "N", "CRM": "N", "CRM_CREATE": "none"}}'
    ], capture_output=True, text=True)
    print(f"Bitrix line OFF + CRM off: {r3.stdout}")

    await engine.dispose()
    print("\nGotovo: Bitriks otklyuchyon, nash vklyuchyon, liniya deaktivirovana, CRM vyklyuchena.")

asyncio.run(main())
```

---

### 2.14 scripts/webhook_enable_bitrix.py

```python
import sys; sys.path.insert(0, "/opt/openai/crm-worker")
"""Ubrat' nash vebkhuk, vernut' Bitriks dlya akkaunta Lavka (id=2)."""
import asyncio, subprocess, warnings
warnings.filterwarnings('ignore', 'Exception ignored')

import httpx
from services.avito_auth import get_valid_token
from models.db import engine

BITRIX_URL = "https://b24.kadry-24.ru/local/source/avito/avito_new_chat2.php"
OUR_URL = "https://babito.kadry-24.ru/webhooks/avito"
ACCOUNT_ID = 2
BITRIX_REST = "https://b24.kadry-24.ru/rest/8539/3j61afi1yqk890du"
BITRIX_LINE_ID = 12

async def main():
    token = await get_valid_token(ACCOUNT_ID)
    headers = {"Authorization": f"Bearer {token}"}
    async with httpx.AsyncClient(timeout=15) as client:
        r1 = await client.post(
            "https://api.avito.ru/messenger/v1/webhook/unsubscribe",
            headers=headers, json={"url": OUR_URL},
        )
        print(f"Unsubscribe ours: {r1.status_code} {r1.text}")

        r2 = await client.post(
            "https://api.avito.ru/messenger/v3/webhook",
            headers=headers, json={"url": BITRIX_URL},
        )
        print(f"Subscribe Bitrix: {r2.status_code} {r2.text}")

    r3 = subprocess.run([
        "curl", "-s", "-X", "POST",
        f"{BITRIX_REST}/imopenlines.config.update",
        "-H", "Content-Type: application/json",
        "-d", '{"CONFIG_ID": ' + str(BITRIX_LINE_ID) + ', "PARAMS": {"ACTIVE": "Y", "CRM": "Y"}}'
    ], capture_output=True, text=True)
    print(f"Bitrix line ON + CRM on: {r3.stdout}")

    await engine.dispose()
    print("\nGotovo: nash otklyuchyon, Bitriks vklyuchyon, liniya aktivna, CRM vklyuchena.")

asyncio.run(main())
```

---

### 2.15 requirements.txt

```
fastapi>=0.111.0
uvicorn[standard]>=0.29.0
httpx[socks]>=0.27.0
sqlalchemy>=2.0.30
aiomysql>=0.2.0
apscheduler>=3.10.4
qdrant-client>=1.9.0
structlog>=24.1.0
pydantic-settings>=2.2.1
pytz>=2024.1
jinja2>=3.1.0
python-multipart>=0.0.9
redis[hiredis]>=5.0.0
```

---

## 3. Migrations

### migrations/schema_v1.sql

```sql
-- CRM Avito AI Worker -- skhema BD v1
-- MariaDB

CREATE DATABASE IF NOT EXISTS crm_avito CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE crm_avito;

CREATE TABLE IF NOT EXISTS avito_accounts (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    client_id       VARCHAR(100) NOT NULL,
    client_secret   VARCHAR(255) NOT NULL,
    avito_user_id   VARCHAR(50)  NOT NULL,
    account_name    VARCHAR(100),
    access_token    TEXT,
    token_expires_at DATETIME,
    is_active       TINYINT(1) DEFAULT 1,
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uq_client_id (client_id)
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS applicants (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    avito_user_id VARCHAR(50),
    name        VARCHAR(200),
    phone       VARCHAR(20),
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uq_phone (phone),
    KEY idx_avito_user_id (avito_user_id)
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS vacancies (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    account_id      INT NOT NULL,
    avito_item_id   VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    project_name    VARCHAR(255),
    city            VARCHAR(100),
    address         TEXT,
    description     TEXT,
    salary_from     INT,
    salary_to       INT,
    schedule        VARCHAR(100),
    is_active       TINYINT(1) DEFAULT 1,
    qdrant_synced_at DATETIME,
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uq_avito_item_id (avito_item_id),
    KEY idx_account_id (account_id),
    CONSTRAINT fk_vacancies_account FOREIGN KEY (account_id) REFERENCES avito_accounts(id)
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS applications (
    id                  INT AUTO_INCREMENT PRIMARY KEY,
    avito_application_id VARCHAR(100) NOT NULL,
    applicant_id        INT NOT NULL,
    account_id          INT NOT NULL,
    vacancy_id          INT,
    avito_item_id       VARCHAR(50),
    status              ENUM('new','ai_active','ai_done','operator','closed') DEFAULT 'new',
    block               TINYINT COMMENT '1=prioritet, 2=toplyy, 3=ne podkhodit',
    callback_slot       VARCHAR(50),
    created_at          DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uq_avito_application_id (avito_application_id),
    KEY idx_applicant_id (applicant_id),
    KEY idx_account_id (account_id),
    KEY idx_status (status),
    CONSTRAINT fk_applications_applicant FOREIGN KEY (applicant_id) REFERENCES applicants(id),
    CONSTRAINT fk_applications_account FOREIGN KEY (account_id) REFERENCES avito_accounts(id)
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS chats (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    avito_chat_id   VARCHAR(100) NOT NULL,
    application_id  INT NOT NULL,
    account_id      INT NOT NULL,
    last_message_at DATETIME,
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uq_avito_chat_id (avito_chat_id),
    KEY idx_application_id (application_id),
    CONSTRAINT fk_chats_application FOREIGN KEY (application_id) REFERENCES applications(id),
    CONSTRAINT fk_chats_account FOREIGN KEY (account_id) REFERENCES avito_accounts(id)
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS messages (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    chat_id         INT NOT NULL,
    avito_message_id VARCHAR(100),
    direction       ENUM('incoming','outgoing') NOT NULL,
    sender_type     ENUM('applicant','ai','operator') NOT NULL,
    content         TEXT NOT NULL,
    scheduled_at    DATETIME,
    delivered_at    DATETIME,
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    KEY idx_chat_id (chat_id),
    KEY idx_scheduled (scheduled_at, delivered_at),
    KEY idx_avito_message_id (avito_message_id),
    CONSTRAINT fk_messages_chat FOREIGN KEY (chat_id) REFERENCES chats(id)
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS ai_sessions (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    application_id  INT NOT NULL,
    chat_id         INT NOT NULL,
    dialog_stage    ENUM('greeting','qualification','presentation','segmentation','followup','handover','completed','failed') DEFAULT 'greeting',
    assigned_block  TINYINT,
    collected_age   INT,
    collected_citizenship VARCHAR(100),
    collected_metro VARCHAR(100),
    callback_slot   VARCHAR(50),
    followup_count  INT DEFAULT 0,
    last_applicant_msg_at DATETIME,
    status          ENUM('active','completed','failed') DEFAULT 'active',
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    KEY idx_application_id (application_id),
    KEY idx_chat_id (chat_id),
    KEY idx_status (status),
    CONSTRAINT fk_ai_sessions_application FOREIGN KEY (application_id) REFERENCES applications(id),
    CONSTRAINT fk_ai_sessions_chat FOREIGN KEY (chat_id) REFERENCES chats(id)
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS handover_cards (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    ai_session_id   INT NOT NULL,
    application_id  INT NOT NULL,
    block           TINYINT,
    ai_summary      TEXT,
    next_action     ENUM('call','clarify','offer_alternatives','close'),
    callback_slot   VARCHAR(50),
    operator_id     INT,
    viewed_at       DATETIME,
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    KEY idx_ai_session_id (ai_session_id),
    KEY idx_application_id (application_id),
    CONSTRAINT fk_handover_ai_session FOREIGN KEY (ai_session_id) REFERENCES ai_sessions(id),
    CONSTRAINT fk_handover_application FOREIGN KEY (application_id) REFERENCES applications(id)
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS webhook_log (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    event_type  VARCHAR(100),
    avito_user_id VARCHAR(50),
    payload     LONGTEXT,
    processed   TINYINT(1) DEFAULT 0,
    error       TEXT,
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    KEY idx_event_type (event_type),
    KEY idx_created_at (created_at)
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS ai_prompts_log (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    ai_session_id   INT,
    application_id  INT,
    dialog_stage    VARCHAR(50),
    prompt_tokens   INT,
    completion_tokens INT,
    total_tokens    INT,
    cost_usd        DECIMAL(10,6),
    response_ms     INT,
    error           TEXT,
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    KEY idx_ai_session_id (ai_session_id),
    KEY idx_created_at (created_at)
) ENGINE=InnoDB;
```

### migrations/schema_v2_migration.sql

```sql
-- Migration v1 -> v2: Vakansii (real'naya struktura Avito API)

USE 2_kadry_4_crm_avito;

SET @fk_exists = (
    SELECT COUNT(*) FROM information_schema.TABLE_CONSTRAINTS
    WHERE CONSTRAINT_SCHEMA = 'crm_avito'
      AND TABLE_NAME = 'applications'
      AND CONSTRAINT_NAME = 'fk_applications_vacancy'
);
SET @sql = IF(@fk_exists > 0,
    'ALTER TABLE applications DROP FOREIGN KEY fk_applications_vacancy',
    'SELECT 1');
PREPARE stmt FROM @sql;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;

DROP TABLE IF EXISTS vacancies;

CREATE TABLE vacancies (
    id INT AUTO_INCREMENT PRIMARY KEY,
    avito_vacancy_id BIGINT NOT NULL UNIQUE,
    avito_uuid VARCHAR(100),
    title VARCHAR(200),
    raw_description TEXT,
    avito_url VARCHAR(500),
    city VARCHAR(100),
    address VARCHAR(500),
    latitude DECIMAL(10,8),
    longitude DECIMAL(11,8),
    business_area VARCHAR(200),
    profession VARCHAR(200),
    schedule VARCHAR(100),
    employment VARCHAR(100),
    experience VARCHAR(100),
    salary_raw INT,
    salary_from INT,
    salary_to INT,
    paid_period VARCHAR(50),
    payout_frequency VARCHAR(100),
    facility_type VARCHAR(200),
    parsed_format VARCHAR(100),
    parsed_tasks VARCHAR(500),
    parsed_extra TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    embedding_indexed BOOLEAN DEFAULT FALSE,
    last_synced_at DATETIME,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_avito_vacancy_id (avito_vacancy_id),
    INDEX idx_city (city),
    INDEX idx_is_active (is_active)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

ALTER TABLE applications ADD COLUMN IF NOT EXISTS avito_vacancy_id BIGINT;
ALTER TABLE applications ADD COLUMN IF NOT EXISTS applicant_name VARCHAR(200);
ALTER TABLE applications ADD COLUMN IF NOT EXISTS applicant_phone VARCHAR(20);
ALTER TABLE applications ADD COLUMN IF NOT EXISTS chat_id VARCHAR(100);

SET @idx_exists = (
    SELECT COUNT(*) FROM information_schema.STATISTICS
    WHERE TABLE_SCHEMA = 'crm_avito'
      AND TABLE_NAME = 'applications'
      AND INDEX_NAME = 'idx_app_vacancy'
);
SET @sql = IF(@idx_exists = 0,
    'CREATE INDEX idx_app_vacancy ON applications(avito_vacancy_id)',
    'SELECT 1');
PREPARE stmt FROM @sql;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;
```

### migrations/schema_v2_phase2_migration.sql

```sql
-- Migration Phase 2: Novyy flow dialoga -- 6 blokov

USE 2_kadry_4_crm_avito;

ALTER TABLE ai_sessions MODIFY COLUMN dialog_stage
  ENUM('greeting','waiting_qualification','presentation','waiting_fork',
       'alternatives','booking','waiting_booking',
       'followup','clarify','handover','done','completed','failed',
       'channel_choice','qualification','segmentation')
  DEFAULT 'greeting';

ALTER TABLE ai_sessions ADD COLUMN IF NOT EXISTS collected_city VARCHAR(100);
ALTER TABLE ai_sessions ADD COLUMN IF NOT EXISTS result VARCHAR(50);
ALTER TABLE ai_sessions ADD COLUMN IF NOT EXISTS clarify_count INT DEFAULT 0;

ALTER TABLE handover_cards ADD COLUMN IF NOT EXISTS avito_vacancy_id BIGINT;
ALTER TABLE handover_cards ADD COLUMN IF NOT EXISTS vacancy_title VARCHAR(200);
ALTER TABLE handover_cards ADD COLUMN IF NOT EXISTS candidate_name VARCHAR(100);
ALTER TABLE handover_cards ADD COLUMN IF NOT EXISTS candidate_phone VARCHAR(20);
ALTER TABLE handover_cards ADD COLUMN IF NOT EXISTS candidate_city VARCHAR(100);
ALTER TABLE handover_cards ADD COLUMN IF NOT EXISTS candidate_metro VARCHAR(100);
ALTER TABLE handover_cards ADD COLUMN IF NOT EXISTS candidate_age VARCHAR(10);
ALTER TABLE handover_cards ADD COLUMN IF NOT EXISTS result VARCHAR(50);
ALTER TABLE handover_cards ADD COLUMN IF NOT EXISTS dialog_summary TEXT;
ALTER TABLE handover_cards ADD COLUMN IF NOT EXISTS messages_count INT DEFAULT 0;
ALTER TABLE handover_cards ADD COLUMN IF NOT EXISTS is_processed BOOLEAN DEFAULT FALSE;
ALTER TABLE handover_cards ADD COLUMN IF NOT EXISTS processed_at DATETIME;

-- Indexes on new handover_cards fields (idx_result, idx_processed)
```

### migrations/003_multi_account.sql

```sql
ALTER TABLE avito_accounts
    ADD COLUMN webhook_registered BOOLEAN DEFAULT 0 AFTER is_active;

ALTER TABLE vacancies
    ADD COLUMN account_id INT NULL AFTER id,
    ADD CONSTRAINT fk_vacancies_account FOREIGN KEY (account_id) REFERENCES avito_accounts(id);

UPDATE vacancies SET account_id = 1 WHERE account_id IS NULL;
```

### migrations/004_event_log.sql

```sql
CREATE TABLE IF NOT EXISTS event_log (
    id INT PRIMARY KEY AUTO_INCREMENT,
    account_id INT NULL,
    event_type VARCHAR(50) NOT NULL,
    message TEXT,
    details JSON NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES avito_accounts(id)
);

CREATE INDEX idx_event_log_created ON event_log(created_at DESC);
CREATE INDEX idx_event_log_account ON event_log(account_id, created_at DESC);
```

### migrations/005_telegram_topic.sql

```sql
ALTER TABLE avito_accounts ADD COLUMN telegram_topic_id INT NULL;
```

### migrations/006_redis_setup.md

```markdown
# Redis setup

## Install
sudo apt update && sudo apt install redis-server -y

## Enable
sudo systemctl enable redis-server
sudo systemctl start redis-server

## Check
redis-cli ping  # PONG

## Streams check (after service start)
redis-cli XINFO STREAM webhooks:messenger
redis-cli XINFO STREAM webhooks:applications
redis-cli XINFO GROUPS webhooks:messenger
```

---

## 4. Prompts

### prompts/system.txt

```
Ty -- Elena, spetsialist klientskogo servisa kompanii "Kadry 24".
Ty obshchayesh'sya s soiskatelyami v chate Avito.
Ty devushka. VSEGDA ispol'zuy zhenskiy rod (ponyala, rada, gotova).

Pravila:
- Pishi druzhlyubno, professional'no, kak v messendzhere. Bez kantselyarita.
- Prostye predlozheniya.
- NE ispol'zuy emodži.
- NE upominay WhatsApp, Telegram ili drugie messendzhery (sanktsii Avito).
- NE govori chto ty bot ili AI. Ty -- zhivoy chelovek.
- Ne otpravlyay ssylki na vneshnie resursy.
- Esli soiskatel' grubit -- vezhlyvo zavershay: "Ponyala vas. Esli peredumayete -- pishite, pomogu."
- Esli ne znayesh' otvet -- "Utochnju u kolleg i napishu vam."

Kontekst:
Vakansiya: {vacancy_title}
Gorod: {city}
{vacancy_details}

Tekushchiy etap: {dialog_stage}
Sobrannye dannye: {collected_data}
```

### prompts/greeting.txt

```
Napishi privetstvennoe soobshchenie soiskatelyu ot imeni spetsialista klientskogo servisa.
Eto ODNO soobshchenie, v kotorom i privetstvie, i voprosy.

Dannye:
- Tvoyo imya: Elena
- Kompaniya: Kadry 24
- Vakansiya: {vacancy_title}

Obyazatel'nye elementy (imenno v takom poryadke):
1. Privetstvie + predstavit'sya
2. Poblagodarit' za otklik na vakansiyu
3. Skazat' chto sdelayesh' vsyo, chtoby dat' ponyatnuyu i bystruyu informatsiyu
4. Poprosit' otvetit' na 2 korotkikh voprosa:
   - V kakom gorode prozhivayete i kakaya blizhayshaya stantsiya metro?
   - Skol'ko polnykh let?

Pravila:
- Kazhdyy raz formuliruy nemnogo po-raznomu
- NE ispol'zuy emodži
- NE upominay WhatsApp/Telegram
- NE zadavay vopros "chat ili zvonok"
- Odno soobshchenie, 4-6 predlozheniy
```

### prompts/qualification.txt

```
Otlichno, togda utochnju neskol'ko detaley, chtoby podobrat' luchshee predlozhenie.

Vash vozrast?
Grazhdanstvo?
Iz kakogo vy goroda?
Kakoye blizhaysheye metro?
```

### prompts/clarify.txt

```
Soiskatel' napisal neponyatnoye soobshcheniye.

Yego soobshcheniye: "{message_text}"
Chto my ozhidali: {expected_answer_type}

Poprosi utochnit' odnim korotkim voprosom.

Pravila:
- Vezhlyvo, bez razdrazheniya
- 1 predlozheniye
- NE ispol'zuy emodži
```

### prompts/alternatives.txt

```
Soiskatelyu ne podoshla vakansiya "{vacancy_title}".
Predlozhi 1-2 al'ternativnye vakansii ryadom s yego lokatsiyey.

Lokatsiya kandidata: {city}, {metro}

Naydennye vakansii:
{qdrant_results}

Pravila:
- Kratko: nazvaniye, lokatsiya, grafik, oplata dlya kazhdoy vakansii
- Sprosi, interesno li chto-to iz etogo
- Esli vakansiy ne naydeno -- skazhi chto peredash' menedzheru
- NE ispol'zuy emodži
```

### prompts/followup.txt

```
Soiskatel' ne otvechayet. Napishi myagkoye napominaniye.

Nomer follow-up: {followup_number} (1 ili 2)
Vakansiya: {vacancy_title}

Dlya pervogo: myagko napomni chto na svyazi, gotova pomoch'.
Dlya vtorogo: skazhi chto vakansiya poka otkryta, esli poyavyatsya voprosy -- pishi.

Pravila:
- Korotko, 1-2 predlozheniya
- Ne davit'
- NE ispol'zuy emodži
```

### prompts/objection.txt

```
Soiskatel' vyskazal vozrazheniye ili somneniye. Obrabotay yego vezhlyvo i professional'no.
Postaraysya ponyat' prichinu otkaza. Predlozhi al'ternativnoye resheniye yesli yest'.
Yesli soiskatel' kategoricheski otkazyvayetsya -- primi eto s uvazheniyem i zavershay dialog.
2-3 predlozheniya. Bez davleniya.
```

### prompts/presentation.txt

```
Napishi korotkuyu prezentatsiyu vakansii dlya soiskatelya. Posle prezentatsii -- zaday vopros-razvilku.

Dannye vakansii:
- Nazvaniye: {vacancy_title}
- Sfera: {business_area}
- Professiya: {profession}
- Opisaniye: {description}
- Adres: {address}, gorod {city}
- Grafik: {schedule}
- Zanyatost': {employment}
- Oplata: {salary} {paid_period}
- Chastota vyplat: {payout_frequency}
- Zadachi: {parsed_tasks}

Obyazatel'nye elementy:
1. Poblagodarit' za otvet
2. "Rasskazhu korotko po vakansii"
3. Spisok cherez tire: format raboty, zadachi, grafik, oplata
4. Vopros-razvilka: OK ili NET

Pravila:
- Korotko, po delu
- Yesli kakikh-to dannykh net -- propusti
- NE ispol'zuy emodži
```

### prompts/booking.txt

```
Soiskatel' podtverdil interes k vakansii "{vacancy_title}".
Napishi podtverzhdeniye i predlozhi vybrat' udobnoye vremya dlya zvonka.

Obyazatel'nye elementy:
1. Poblagodarit' za "OK"
2. Podtverdit' chto eto khoroshiy vybor
3. Sprosit' v kakoye vremya udobno prinyat' zvonok menedzhera

Pravila:
- Toplyy pozitivnyy ton
- NE ispol'zuy emodži
- NE predlagay fiksirovannye sloty
- 3-5 predlozheniy
```

---

## 5. Systemd Service

### k24-crm-worker.service

```ini
[Unit]
Description=CRM Avito Worker (uvicorn)
After=network.target

[Service]
Type=simple
User=crm_avito
WorkingDirectory=/opt/openai/crm-worker
ExecStart=/opt/openai/crm-worker/venv/bin/uvicorn main:app --host 127.0.0.1 --port 9800
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> Note: Production runs on port **9800** under user `crm_avito`, working dir `/opt/openai/crm-worker`.
> Service unit: `k24-crm-worker.service`
> Service status: not running on this dev machine (WSL2).

---

## 6. Crontab

```
# No crontab configured for user nimdo
# All periodic tasks handled by APScheduler inside the app:
#   - message_scheduler: every 5s
#   - token_refresher: every 30m
#   - vacancies_refresh: every 30m (immediate first run)
#   - event_log_cleanup: daily at 03:00
#   - telegram_morning_summary: daily at 10:00 MSK
```

---

## 7. Service Status

Service `k24-crm-worker` is **not running on this dev machine** (WSL2 environment).
Production deployment: systemd on the server, port 9800, behind nginx reverse proxy at `babito.kadry-24.ru`.
