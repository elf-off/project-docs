# Project Summary: CEO Strategic Control Agent

**Audit date:** 2026-03-24

## Overview

AI-agent for strategic control of staffing operations. Monitors shift fulfillment across projects, calculates risk scores, detects incidents, notifies managers via Telegram, and generates daily/weekly reports for CEO.

## Project tree (.py files)

```
ceo_agent/
|-- main.py                          # 85 lines  - Entry point, event loop, signal handling
|-- config/
|   |-- __init__.py                  # 0 lines
|   |-- settings.py                  # 88 lines  - Pydantic settings (primary, used by all modules)
|-- src/
|   |-- __init__.py                  # 0 lines
|   |-- orchestrator.py              # 446 lines - Main coordinator, APScheduler, lifecycle
|   |-- config/
|   |   |-- __init__.py              # 0 lines
|   |   |-- settings.py              # 88 lines  - DUPLICATE of config/settings.py (dead code)
|   |-- connectors/
|   |   |-- __init__.py              # 0 lines
|   |   |-- data_ingestion.py        # 284 lines - Fetches shift data from main DB, creates snapshots
|   |-- core/
|   |   |-- __init__.py              # 0 lines
|   |   |-- incident_handler.py      # 223 lines - Detects unfulfilled requests, notifies managers
|   |   |-- metrics.py               # 312 lines - Risk score calculation (30-day rolling window)
|   |-- analysis/
|   |   |-- __init__.py              # 0 lines
|   |   |-- reason_analyzer.py       # 355 lines - AI-powered reason classification (Claude API)
|   |   |-- weekly_ranking.py        # 475 lines - Weekly TOP-5 ranking engine
|   |-- reporting/
|   |   |-- __init__.py              # 0 lines
|   |   |-- ceo_reports.py           # 536 lines - HTML report generator (daily digest, weekly ranking)
|   |   |-- old_ceo_reports.py       # 349 lines - DEPRECATED old report generator (dead code)
|   |-- storage/
|   |   |-- __init__.py              # 0 lines
|   |   |-- database.py              # 122 lines - DB connection manager (main + agent sessions)
|   |   |-- models.py                # 239 lines - SQLAlchemy ORM models (ref tables + agent_* tables)
|   |-- telegram/
|   |   |-- __init__.py              # 0 lines
|   |   |-- bot.py                   # 162 lines - Telegram bot (outgoing messages only)
|-- diagnose.py                      # 303 lines - Diagnostic script (env, DB, Telegram checks)
|-- setup_db.py                      # 199 lines - Interactive DB setup + project-manager mapping
|-- test_agent.py                    # 443 lines - Interactive test menu (7 test cases)
|-- test_create_tables.py            # 11 lines  - One-off table creation script
|-- analyze_filtering.py             # 116 lines - Project filtering analysis utility
|-- check_timezone.py                # 128 lines - Timezone diagnostic utility
|-- manual_daily_digest.py           # 78 lines  - Manual daily digest trigger
|-- monthly_exits_by_le.py           # 329 lines - Monthly XLSX report by legal entity
|-- plan_fact_weekly.py              # 333 lines - Weekly plan vs fact comparison
|-- patch_shifts.py                  # 166 lines - One-off patch: fix RefShift model
|-- patch_sqlalchemy.py              # 164 lines - One-off patch: SQLAlchemy 2.0 compat
|-- set_schedule.py                  # 134 lines - Interactive schedule configuration
```

## Codebase size

| Metric | Value |
|---|---|
| Total .py files | 34 |
| Source files (non-empty) | 25 |
| Total lines of Python | 6,168 |
| Core application (src/) | 3,594 lines |
| Utility/scripts (root) | 2,574 lines |

## Dependencies (requirements.txt)

| Package | Version | Purpose |
|---|---|---|
| python-dotenv | 1.0.0 | .env file loading |
| pydantic | 2.5.0 | Data validation |
| pydantic-settings | 2.1.0 | Settings via Pydantic |
| pymysql | 1.1.0 | MySQL/MariaDB driver |
| sqlalchemy | 2.0.23 | ORM |
| python-telegram-bot | 20.7 | Telegram Bot API |
| APScheduler | 3.10.4 | Task scheduler |
| pytz | 2023.3 | Timezone handling |
| anthropic | 0.8.1 | Claude AI API (reason analysis) |
| structlog | 23.2.0 | Structured logging |
| pytest | 7.4.3 | Testing (dev) |
| pytest-asyncio | 0.21.1 | Async test support (dev) |
| tenacity | 8.2.3 | Retry logic |

## Module descriptions (1 line each)

| Module | Description |
|---|---|
| `config/settings.py` | Pydantic-based settings loaded from .env with defaults for all parameters |
| `src/orchestrator.py` | Main agent class with APScheduler; coordinates data ingestion, metrics, incidents, reports |
| `src/connectors/data_ingestion.py` | Reads shift data from source DB (exits_wa), normalizes and saves immutable snapshots |
| `src/core/metrics.py` | Calculates 30-day rolling risk scores per project (frequency, scale, streak components) |
| `src/core/incident_handler.py` | Detects unfulfilled shifts, creates incidents, notifies responsible managers via Telegram |
| `src/analysis/reason_analyzer.py` | Classifies manager responses using Claude API (or keyword fallback), builds hypotheses |
| `src/analysis/weekly_ranking.py` | Weekly TOP-5 ranking: volume, fulfillment rate, trend via linear regression |
| `src/reporting/ceo_reports.py` | Generates HTML reports (daily digest + weekly ranking) for Telegram delivery to CEO |
| `src/reporting/old_ceo_reports.py` | Deprecated old report generator, not imported anywhere |
| `src/storage/database.py` | Dual-session DB manager: read-only for source data, read-write for agent tables |
| `src/storage/models.py` | All SQLAlchemy models: 5 source tables (ref_*) + 10 agent tables (agent_*) |
| `src/telegram/bot.py` | Outgoing Telegram messages: incident alerts, CEO digests, admin notifications |
| `diagnose.py` | Full environment diagnostic: checks DB, Telegram, deps, config, processes |
| `setup_db.py` | Interactive setup: creates agent tables, manages project-manager mappings |
| `monthly_exits_by_le.py` | Monthly XLSX report of actual exits grouped by legal entity |
| `plan_fact_weekly.py` | Weekly plan-vs-fact comparison by legal entity with Telegram delivery |
