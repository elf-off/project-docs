# Audit Report: CEO Strategic Control Agent

**Audit date:** 2026-03-24
**Codebase:** 6,168 lines across 34 .py files
**Docs reviewed:** docs/API.md, docs/DEPLOYMENT.md, .env.example, plan.csv

---

## 1. Task / Feature Status Summary

Based on docs/API.md, docs/DEPLOYMENT.md, .env.example, and source code analysis:

| Feature | Documented | Implemented | Status | Notes |
|---|---|---|---|---|
| Data ingestion (shifts) | API.md | data_ingestion.py | OK | Working, snapshot-based |
| Risk score calculation | API.md | metrics.py | OK | 30-day rolling, 3-component formula |
| Risk classification (green/yellow/red) | API.md | metrics.py | OK | Thresholds from config |
| Incident detection | API.md | incident_handler.py | BUG | sync/async mismatch, see bug #1 |
| Manager Telegram notifications | API.md | incident_handler.py, bot.py | BUG | Fails inside running event loop |
| AI reason classification (Claude) | API.md | reason_analyzer.py | PARTIAL | Model hardcoded, limit_days ignored |
| Hypothesis generation | API.md | reason_analyzer.py | OK | 3-level weighted blending |
| Daily CEO digest | API.md | ceo_reports.py | OK | HTML via Telegram |
| Weekly TOP-5 ranking | API.md | weekly_ranking.py, ceo_reports.py | BUG | NameError in except block |
| Manual processing | API.md | orchestrator.py | OK | manual_process() |
| Scheduler (APScheduler) | DEPLOYMENT.md | orchestrator.py | WARN | Triple registration of same job |
| Systemd service | DEPLOYMENT.md | ceo-agent.service | OK | Sandbox configured |
| Docker deployment | DEPLOYMENT.md | -- | MISSING | Documented but no Dockerfile exists |
| pytest test suite | API.md | -- | MISSING | `pytest tests/` documented, no tests/ dir |
| Monthly exits report (XLSX) | -- | monthly_exits_by_le.py | OK | Undocumented feature |
| Weekly plan vs fact | -- | plan_fact_weekly.py | PARTIAL | CLI subparser `match` is a no-op |
| ALERT_MIN_RISK_SCORE filtering | .env.example | incident_handler.py | NOT IMPL | Config exists, code ignores it |
| ALERT_MIN_STREAK filtering | .env.example | incident_handler.py | NOT IMPL | Config exists, code ignores it |
| Sentry integration | .env.example | -- | NOT IMPL | SENTRY_DSN in config, no code uses it |
| Data retention (cleanup) | .env.example | -- | NOT IMPL | RETENTION_*_DAYS in config, no cleanup logic |
| DB SSL support | DEPLOYMENT.md | -- | NOT IMPL | Documented DB_SSL_* vars, not in settings.py |
| CEODecision feedback loop | -- | models.py | NOT IMPL | Model defined, no business logic |

---

## 2. Detailed Bug Report

### BUG #1 -- CRITICAL: sync/async mismatch in incident_handler.py

**File:** `src/core/incident_handler.py:120-138`

`_create_and_notify_incident()` is a synchronous method that calls `loop.run_until_complete()` to send Telegram messages. However, this method is invoked from `process_snapshot_incidents()`, which is called from `orchestrator._process_shift()` inside a running async event loop. Calling `run_until_complete()` from within a running loop raises `RuntimeError: This event loop is already running`.

**Impact:** Manager notifications silently fail in production.

**Fix:** Make `_create_and_notify_incident` async and use `await` instead of `run_until_complete`.

---

### BUG #2 -- HIGH: NameError in weekly_ranking.py except block

**File:** `src/analysis/weekly_ranking.py:244`

```python
except Exception as e:
    self.logger.error("metrics_calculation_failed",
                    project_id=project_id,  # <-- undefined variable
                    error=str(e))
```

The variable `project_id` does not exist in the local scope of `calculate_project_metrics()`. The correct variable is `entity_id`.

**Impact:** If metrics calculation fails, a secondary `NameError` is raised, masking the original error.

---

### BUG #3 -- HIGH: Duplicate TIMEZONE field in settings.py

**File:** `config/settings.py:14,32`

```python
TIMEZONE: str = "Europe/Amsterdam"   # line 14
# ...
TIMEZONE: str = "Europe/Moscow"      # line 32
```

The field is declared twice. Pydantic v2 uses the last declaration, so the effective default is `"Europe/Moscow"`, but the `"Europe/Amsterdam"` on line 14 is misleading dead code.

**Impact:** Confusing; .env.example says `Europe/Amsterdam`, code defaults to `Europe/Moscow`.

---

### BUG #4 -- HIGH: Wrong session for snapshot queries in ceo_reports.py

**File:** `src/reporting/ceo_reports.py:324`

`_get_recent_reasons()` or related methods query `agent_snapshots` using `get_main_session()` (read-only source session) instead of `get_agent_session()`. Since both sessions currently point to the same DB this works by accident, but will break if DBs are ever separated as the architecture intends.

---

### BUG #5 -- MEDIUM: asyncio.create_task in sync signal handler

**File:** `main.py:19`

```python
def signal_handler(sig, frame):
    asyncio.create_task(agent.stop())
```

`asyncio.create_task()` requires a running event loop in the current context. A signal handler runs in the main thread but outside of async context, causing `RuntimeError`.

---

### BUG #6 -- MEDIUM: Triple job registration in orchestrator.py

**File:** `src/orchestrator.py:74-113`

The `weekly_ranking_job` is registered 3 times with `replace_existing=True`. Only the last registration takes effect. The first two are dead code from incomplete refactoring.

---

### BUG #7 -- MEDIUM: limit_days parameter is ignored

**File:** `src/analysis/reason_analyzer.py:189`

`get_reason_statistics(limit_days=...)` accepts the parameter but never applies date filtering. All historical data is returned regardless of `limit_days`.

---

### BUG #8 -- MEDIUM: Duplicate function definitions in test_agent.py

**File:** `test_agent.py`

- `test_metrics_calculation` is defined twice (~lines 71 and 179). Second overwrites first.
- `test_weekly_ranking` is defined twice (~lines 122 and 299). Second overwrites first.

**Impact:** First versions of these test functions are unreachable.

---

### BUG #9 -- LOW: CLI subparser `match` is a no-op

**File:** `plan_fact_weekly.py:294`

`sub.add_parser("match")` registers the command but there is no `if args.cmd == "match"` handler. Running `python plan_fact_weekly.py match` silently does nothing.

---

### BUG #10 -- LOW: NameError in old_ceo_reports.py

**File:** `src/reporting/old_ceo_reports.py:170`

`timedelta` is used but not imported (only `date, datetime` are imported). Calling `_get_recent_reasons()` would raise `NameError`. File is dead code, so impact is nil.

---

### BUG #11 -- LOW: Bare except:pass swallowing errors

**Files:**
- `manual_daily_digest.py:67` -- cleanup errors silenced
- `src/orchestrator.py:135` -- startup alert errors silenced

---

## 3. Architectural / Code Quality Issues

### 3.1 Duplicate settings file

`config/settings.py` and `src/config/settings.py` are identical files. All imports use `config.settings` (the root one). `src/config/settings.py` is dead code that creates confusion risk.

**Recommendation:** Delete `src/config/settings.py` and `src/config/__init__.py`.

### 3.2 Dead code: old_ceo_reports.py

`src/reporting/old_ceo_reports.py` (349 lines) is not imported anywhere. It was replaced by `ceo_reports.py`.

**Recommendation:** Delete it.

### 3.3 Dead method: _send_weekly_ranking_report

`src/orchestrator.py` contains `_send_weekly_ranking_report()` (lines 217-243) which is never called. The functionality is inline in `process_weekly_ranking()`.

**Recommendation:** Remove the dead method.

### 3.4 Dead function: get_test_times in set_schedule.py

`set_schedule.py:10-27` defines `get_test_times()` which is never called.

### 3.5 N+1 query in metrics.py

`src/core/metrics.py:127-129` -- For each `SnapshotRecord`, a separate query fetches the parent `Snapshot` to get `shift_code`. With 1000 records this produces 1000 extra queries.

**Recommendation:** Use a JOIN or store `shift_code` directly on `SnapshotRecord`.

### 3.6 Hardcoded paths and DB names

- `analyze_filtering.py:6` -- hardcoded `/opt/ai-agents/ceo_agent`
- `monthly_exits_by_le.py:57-58` -- hardcoded DB names `2_kadry_4_finance`
- `plan_fact_weekly.py:57-58` -- same hardcoded DB names
- `reason_analyzer.py:69` -- hardcoded Claude model `claude-sonnet-4-20250514`

### 3.7 Deprecated APIs

- `datetime.utcnow` used as column defaults in `models.py` (deprecated Python 3.12+)
- `declarative_base()` from `sqlalchemy.ext.declarative` (deprecated since SA 1.4)
- `session.query(Model).get(id)` pattern in `incident_handler.py` (deprecated SA 2.0)
- `asyncio.get_event_loop().run_until_complete()` in `monthly_exits_by_le.py:269`, `plan_fact_weekly.py:248`

### 3.8 Unused imports

| File | Unused import |
|---|---|
| `main.py:7` | `from datetime import date` |
| `src/orchestrator.py:5` | `time` from datetime |
| `src/connectors/data_ingestion.py` | `func` from sqlalchemy |
| `src/core/incident_handler.py` | `date` from datetime |
| `src/core/metrics.py` | `func` from sqlalchemy |
| `src/storage/database.py:4` | `event` from sqlalchemy, `NullPool` from sqlalchemy.pool |
| `src/telegram/bot.py:6` | `Dict` from typing |
| `src/reporting/ceo_reports.py` | `TelegramMessage` from models |
| `src/reporting/old_ceo_reports.py` | `TelegramMessage` from models |
| `src/analysis/reason_analyzer.py:12` | `TelegramMessage` from models |

### 3.9 Inconsistent Risk Score formulas

`ceo_reports.py` uses a simplified single-day Risk Score formula (lines 425-435) that differs from the 30-day rolling formula in `metrics.py`. Both produce `risk_score` values that appear in different contexts, potentially confusing the CEO.

### 3.10 No test_create_tables.py guard

`test_create_tables.py` has no `if __name__ == "__main__"` guard. Importing the module as a library will execute table creation.

### 3.11 Config parameters defined but not implemented

| Parameter | Defined in | Not used in |
|---|---|---|
| `ALERT_MIN_RISK_SCORE` | settings.py, .env.example | incident_handler.py (always True if ALERT_ON_MISS) |
| `ALERT_MIN_STREAK` | settings.py, .env.example | incident_handler.py |
| `SENTRY_DSN` | settings.py, .env.example | nowhere |
| `RETENTION_SNAPSHOTS_DAYS` | settings.py, .env.example | nowhere (no cleanup job) |
| `RETENTION_LOGS_DAYS` | settings.py, .env.example | nowhere |
| `DB_SSL_CA/CERT/KEY` | DEPLOYMENT.md | not in settings.py or database.py |

---

## 4. Documentation vs Implementation Gaps

| Doc claim | Reality |
|---|---|
| DEPLOYMENT.md: Docker/docker-compose deployment | No Dockerfile or docker-compose.yml in repo |
| API.md: `pytest tests/` with `--cov=src` | No `tests/` directory exists; only interactive `test_agent.py` |
| .env.example: `TIMEZONE=Europe/Amsterdam` | Code defaults to `Europe/Moscow` (second declaration wins) |
| DEPLOYMENT.md: DB_SSL_CA/CERT/KEY support | Not implemented in settings or database.py |
| API.md: `IncidentHandler` notifies on risk score / streak thresholds | Always notifies on any miss (thresholds ignored) |

---

## 5. Open TODO / FIXME

**None found.** No `TODO`, `FIXME`, `HACK`, `XXX`, or `WORKAROUND` markers exist anywhere in the Python codebase.

However, there are multiple features that are partially implemented or stubbed (see section 1, "NOT IMPL" rows), which effectively serve as implicit TODOs:

1. Implement `ALERT_MIN_RISK_SCORE` and `ALERT_MIN_STREAK` filtering in `incident_handler.py`
2. Implement data retention cleanup job using `RETENTION_*_DAYS` config
3. Implement Sentry integration
4. Implement DB SSL support
5. Implement `CEODecision` feedback loop
6. Create Dockerfile / docker-compose.yml (or remove from docs)
7. Create actual pytest test suite in `tests/` directory

---

## 6. Priority Recommendations

1. **Fix sync/async mismatch in incident_handler.py** -- This is likely causing silent failures in production right now
2. **Fix NameError in weekly_ranking.py:244** -- Will mask real errors when they occur
3. **Remove duplicate TIMEZONE declaration in settings.py** -- Decide Amsterdam vs Moscow
4. **Delete dead code** -- `src/config/`, `old_ceo_reports.py`, triple job registration
5. **Add `if __name__` guards** to utility scripts
6. **Implement ALERT_MIN_RISK_SCORE/STREAK filtering** or remove from config to avoid confusion
7. **Fix N+1 query in metrics.py** -- Performance issue at scale
8. **Create proper pytest suite** replacing interactive test_agent.py
