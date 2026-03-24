# Project Summary -- tg_oleg_assist

**Data: 2026-03-24**

---

## 1. Struktura proekta (derevo .py faylov)

```
tg_oleg_assist/
|-- app/
|   |-- __init__.py
|   |-- main.py                          63 str  -- FastAPI entry point, lifespan, health check
|   |-- config.py                       116 str  -- Pydantic settings iz .env
|   |-- database.py                      23 str  -- SQLAlchemy async engine + session
|   |-- api/
|   |   |-- __init__.py
|   |   |-- webhook.py                   85 str  -- POST /tg/webhook, idempotency guard
|   |   |-- admin.py                      5 str  -- Admin API (stub)
|   |-- models/
|   |   |-- __init__.py                  45 str  -- Re-export vsekh modeley
|   |   |-- chat.py                      46 str  -- Chaty s mode/features/config JSON
|   |   |-- user.py                      35 str  -- Polzovateli s rolyami (Manager/Deputy/Director/HR)
|   |   |-- message.py                   24 str  -- Sokhranennye soobscheniya
|   |   |-- dialog_state.py              27 str  -- FSM sostoyaniye dialoga (NORMAL->ESCALATION)
|   |   |-- state_transition.py          23 str  -- Istoriya perekhodov sostoyaniy
|   |   |-- ethics_event.py              32 str  -- Sobyitiya eticheskogo analiza
|   |   |-- scoring_event.py             33 str  -- Sobyitiya skoringa
|   |   |-- notification.py              37 str  -- Log uvedomleniy
|   |   |-- mention.py                   42 str  -- Otslezhivanie @upominaniy
|   |   |-- daily_report.py              36 str  -- Ezhendnevnye otchety
|   |   |-- report_type.py              17 str  -- Tipy otchetov
|   |   |-- report_responsible.py        19 str  -- Otvetstvennye za otchety
|   |   |-- task.py                      38 str  -- Zadachi s dedlainami
|   |   |-- planning.py                 134 str  -- Planirovaniye: managers, submissions, entries
|   |   |-- planner.py                   92 str  -- Planner GD: states, feedbacks, analysis_logs
|   |   |-- bitrix_lead.py               27 str  -- Lidy Bitrix24
|   |   |-- growth_report.py             49 str  -- Otchety plan-fakt rosta
|   |   |-- org_unit.py                  16 str  -- Organizatsionnye edinitsy
|   |   |-- audit_log.py                 18 str  -- Audit log
|   |-- prompts/
|   |   |-- __init__.py                   7 str
|   |   |-- signal_detector.py          103 str  -- System prompt dlya detektsii eticheskikh narusheniy
|   |   |-- instruction.py               63 str  -- Prompt dlya detektsii instruktsiy
|   |-- rules/
|   |   |-- __init__.py                   9 str
|   |   |-- loader.py                   193 str  -- Zagruzchik YAML-pravil (S1-S7, C1+)
|   |   |-- rules.yaml                         -- Pravila etiki (kategorii, vesa, signaly)
|   |   |-- conflict_rules.yaml                -- Usloviya konfliktov, FSM, scoring config
|   |-- services/
|   |   |-- __init__.py                  24 str
|   |   |-- message_router.py           703 str  -- Centralnyy dispatcher soobscheniy po rezhimam
|   |   |-- chat_context.py             204 str  -- Resolver konteksta chata (mode/features/config)
|   |   |-- idempotency.py              131 str  -- In-memory dedup webhook retry po update_id
|   |   |-- ai_client.py               204 str  -- Anthropic Claude API wrapper (analyze/analyze_text)
|   |   |-- ethics_analyzer.py          172 str  -- Detektsiya narusheniy etiki (AI + fallback)
|   |   |-- rule_matcher.py             100 str  -- Keyword-matching fallback dlya pravil
|   |   |-- scoring_engine.py           322 str  -- FSM skoringa: NORMAL->TENSION->CONFLICT->ESCALATION
|   |   |-- arbitration.py              285 str  -- Opredelenie arbitrov po matrice rol x risk
|   |   |-- notification.py             618 str  -- Otpravka Telegram-uvedomleniy, preduprezhddeniy, blokirovok
|   |   |-- mention_service.py          256 str  -- Otslezhivanie @mentionov i dedlainov
|   |   |-- report_service.py           536 str  -- Obnaruzhenie i registratsiya otchetov, ROP validatsiya
|   |   |-- planning_service.py         750 str  -- Nedelnoe planirovanie menedzherov (#Planirovanie)
|   |   |-- bitrix_service.py           459 str  -- Sozdanie lidov v Bitrix24 (#Bitriks)
|   |   |-- rop_validator.py            631 str  -- Validatsiya ROP blok-otchetov
|   |   |-- instruction_detector.py     164 str  -- Detektsiya instruktsiy v soobscheniyakh
|   |   |-- timer_worker.py            1046 str  -- Fonovyy worker: mentiony, otchety, planner, plan-fakt
|   |   |-- monthly_exits_by_le.py      329 str  -- Ezhemesyachnyy otchet po vykhodam YuL
|   |   |-- old_ai_client.py            134 str  -- [DEPRECATED] Staryy AI client (OpenAI)
|   |   |-- old_rop_validator.py        371 str  -- [DEPRECATED] Staraya validatsiya ROP
|   |   |-- planner/
|   |   |   |-- __init__.py               5 str
|   |   |   |-- planner_service.py      631 str  -- CEO daily planner: FSM, callbacks, analitika
|   |   |   |-- planner_day_themes.py   539 str  -- Raspisanie nedeli, temy dney, chek-listy
|   |   |   |-- planner_formatter.py    218 str  -- Formatirovanie soobscheniy plannera
|   |   |   |-- planner_prompts.py      138 str  -- Prompty dlya Claude (analiz dnya)
|-- tests/
|   |-- __init__.py
|   |-- test_rules_loader.py            103 str  -- Testy zagruzki pravil
|   |-- test_scoring_logic.py           153 str  -- Testy logiki skoringa
|   |-- test_scoring_engine.py          179 str  -- Unit-testy scoring engine
|   |-- test_ethics_analyzer.py         134 str  -- Testy eticheskogo analiza
|   |-- test_message_router.py          258 str  -- Testy marshrutizatsii soobscheniy
|   |-- test_notifications.py           264 str  -- Testy uvedomleniy
|   |-- test_arbitration.py             199 str  -- Testy arbitratsii
|   |-- test_planner_service.py         636 str  -- Testy planner FSM
|   |-- test_planner_service_v13.py     362 str  -- Testy planner v1.3 (dva tsikla)
|   |-- test_planner_day_themes.py      364 str  -- Testy raspisaniya
|   |-- test_timer_worker.py            292 str  -- Testy timer worker
|   |-- test_manual_morning.py          154 str  -- Ruchnoy test utrennego soobscheniya
|-- scripts/
|   |-- __init__.py
|   |-- init_db.py                       49 str  -- Initsializatsiya tabilts BD
|   |-- set_webhook.py                   52 str  -- Ustanovka Telegram webhook
|   |-- set_webhook_simple.py           162 str  -- Webhook setup cherez httpx
|   |-- check_webhook.py               114 str  -- Proverka statusa webhook
|   |-- seed_data.py                    119 str  -- Zapolnenie BD testovymi dannymi
|   |-- test_rop_validator.py           240 str  -- Testovye scenarii ROP
|   |-- test_growth_plan_fact.py        474 str  -- Test plan-fakt na realnykh dannykh
|   |-- run_growth_report.py             42 str  -- Ruchnoy zapusk otcheta rosta
|-- migrations/
|   |-- 001_multichat_architecture.sql   92 str  -- Multi-chat config, features JSON
|   |-- 002_mentions_reports.sql         84 str  -- Mentiony, tipy otchetov
|   |-- 003_planning.sql                 60 str  -- Nedelnoe planirovanie
|   |-- 003_planning_patch_groups.sql    18 str  -- Dobavlenie group_id
|   |-- 004_assist.sql                  105 str  -- Planner GD tablitsy
|   |-- 005_planner_v1.3.sql             89 str  -- Dva tsikla v den (half)
|   |-- 006_bitrix_leads.sql             25 str  -- Bitrix24 lidy
|   |-- 007_growth_plan_fact.sql         53 str  -- Plan-fakt polya i tablitsa
|   |-- alter_leads_v3.sql                9 str  -- Flag is_autoclose
```

---

## 2. Razmer kodovoy bazy

| Kategoriya | Faylov | Strok |
|------------|--------|-------|
| app/ (production code) | 57 | 10,427 |
| tests/ | 13 | 3,098 |
| scripts/ | 9 | 1,252 |
| migrations/ (SQL) | 9 | 535 |
| **Vsego .py** | **81** | **15,504** |

### Top-10 po razmeru (app/)

| Fayl | Strok |
|------|-------|
| timer_worker.py | 1,046 |
| planning_service.py | 750 |
| message_router.py | 703 |
| rop_validator.py | 631 |
| planner_service.py | 631 |
| notification.py | 618 |
| planner_day_themes.py | 539 |
| report_service.py | 536 |
| bitrix_service.py | 459 |
| old_rop_validator.py | 371 |

---

## 3. Zavisimosti (requirements.txt)

| Paket | Versiya | Naznacheniye |
|-------|---------|-------------|
| fastapi | 0.109.0 | Web framework (webhook mode) |
| uvicorn[standard] | 0.27.0 | ASGI server |
| sqlalchemy[asyncio] | 2.0.25 | Async ORM |
| aiomysql | 0.2.0 | MySQL async driver |
| alembic | 1.13.1 | *Ne ispolzuetsya (migracii ruchnye SQL)* |
| openai | 1.12.0 | *Ne ispolzuetsya (tolko v old_ai_client.py)* |
| tenacity | 8.2.3 | *Ne ispolzuetsya (tolko v old_ai_client.py)* |
| python-dotenv | 1.0.1 | Zagruzka .env |
| pydantic-settings | 2.1.0 | Tipizovannaya konfiguratsiya |
| structlog | 24.1.0 | Strukturirovannoe logirovanie |
| pyyaml | 6.0.1 | Zagruzka pravil iz YAML |
| python-telegram-bot | 20.7 | Telegram Bot API |
| httpx | >=0.25.0,<1 | HTTP client (Bitrix24 API) |

**Neispolzuemye zavisimosti:** alembic, openai, tenacity (mogut byt udaleny)

---

## 4. Kratkoe opisanie moduley (1 stroka)

### Core
| Modul | Opisanie |
|-------|----------|
| `main.py` | FastAPI app s lifespan, TimerWorker, health check |
| `config.py` | Pydantic settings iz .env (40+ parametrov) |
| `database.py` | SQLAlchemy async engine + context manager sessiy |

### API
| Modul | Opisanie |
|-------|----------|
| `webhook.py` | Priyom Telegram webhook s secret token + idempotency |
| `admin.py` | Stub dlya admin API |

### Services
| Modul | Opisanie |
|-------|----------|
| `message_router.py` | Centralnyy dispatcher: chat mode -> handler pipeline |
| `chat_context.py` | Resolver chata: tg_chat_id -> mode/features/config s keshem |
| `idempotency.py` | In-memory TTL cache dedup webhook retries po update_id |
| `ai_client.py` | Anthropic Claude wrapper: analyze() -> JSON, analyze_text() -> string |
| `ethics_analyzer.py` | Detektsiya narusheniy etiki cherez Claude + fallback na rules |
| `rule_matcher.py` | Keyword-matching fallback po YAML-pravilam (S1-S7) |
| `scoring_engine.py` | FSM skoringa: nakoplenie ballov, perehody sostoyaniy, skol. okno |
| `arbitration.py` | Matrica arbitrov: rol x uroven riska -> kto razrulivalyet |
| `notification.py` | Otpravka TG soobscheniy: preduprezhdeniya, blokirovki, broadcast |
| `mention_service.py` | Otslezhivanie @mentionov s dedlainami i napominaniyami |
| `report_service.py` | Detektsiya otchetov, ROP validatsiya, AI-reformatirovanie |
| `planning_service.py` | Nedelnoe planirovanie: #Planirovanie, AI-matching proektov |
| `bitrix_service.py` | Sozdanie lidov v Bitrix24: regex + AI parsing, REST API |
| `rop_validator.py` | Validatsiya ROP blok-otchetov: format + AI-proverka |
| `instruction_detector.py` | Detektsiya instruktsiy/porucheniy v soobscheniyakh |
| `timer_worker.py` | Fonovyy async worker (30s loop): mentiony, otchety, planner, plan-fakt |
| `monthly_exits_by_le.py` | Ezhemesyachnyy otchet po vykhodam v razreze YuL |
| `old_ai_client.py` | [DEPRECATED] OpenAI client s retry |
| `old_rop_validator.py` | [DEPRECATED] Staraya versiya ROP validatsii |

### Planner (CEO daily assistant)
| Modul | Opisanie |
|-------|----------|
| `planner_service.py` | CEO planner: FSM obratnoy svyazi, 2 tsikla v den, Claude analiz |
| `planner_day_themes.py` | Nedelnoe raspisanie, temy dney, utrennie chek-listy |
| `planner_formatter.py` | Formatirovanie soobscheniy plannera (HTML) |
| `planner_prompts.py` | System prompty dlya Claude (analiz rabochego dnya) |

### Prompts
| Modul | Opisanie |
|-------|----------|
| `signal_detector.py` | System prompt: detektsiya 7 kategoriy eticheskikh narusheniy |
| `instruction.py` | System prompt: detektsiya instruktsiy/porucheniy |

### Rules
| Modul | Opisanie |
|-------|----------|
| `loader.py` | Zagruzchik YAML pravil: rules S1-S7, conflict conditions, scoring config |

### Models (20 faylov)
| Modul | Opisanie |
|-------|----------|
| `chat.py` | Chat s mode/features/config (multi-chat arkhitektura) |
| `user.py` | Polzovatel s rolyami i kuratorom |
| `message.py` | Sokhranennoe soobscheniye |
| `dialog_state.py` | FSM sostoyaniye dialoga |
| `state_transition.py` | Istoriya perekhodov |
| `ethics_event.py` | Sobytie eticheskogo analiza |
| `scoring_event.py` | Sobytie skoringa |
| `notification.py` | Log uvedomleniy |
| `mention.py` | @Upominaniye s dedlainom |
| `daily_report.py` | Ezhednevnyy otchet |
| `report_type.py` | Tip otcheta |
| `report_responsible.py` | Otvetstvennyy za otchet |
| `task.py` | Zadacha s dedlainom |
| `planning.py` | Planirovanie: managers, submissions, entries |
| `planner.py` | Planner: states, feedbacks, analysis_logs |
| `bitrix_lead.py` | Lid Bitrix24 |
| `growth_report.py` | Otchet plan-fakt rosta |
| `org_unit.py` | Organizatsionnaya edinitsa |
| `audit_log.py` | Audit log |

---

## 5. Vneshnie integratsii

| Sistema | Ispolzovanie |
|---------|-------------|
| Telegram Bot API | Webhook priom/otpravka soobscheniy, inline knopki, restrikcii |
| Anthropic Claude API | Eticheskiy analiz, ROP validatsiya, planirovanie, planner analitika |
| MariaDB/MySQL | Osnovnaya BD (21 tablitsa) |
| 2_kadry_4 (vneshnyaya BD) | ref_projects_wa, ref_project_groups, exits_wa, exits_wa_stajers |
| Bitrix24 REST API | Sozdanie lidov cherez webhook URL |

---

## 6. Rezhimy chatov

| Rezhim | Opisanie | Klyuchevye servisy |
|--------|----------|-------------------|
| `viewer` | Ignorirovat vse soobscheniya (default) | -- |
| `compliance` | Etika + mentiony + otchety | EthicsAnalyzer, ScoringEngine, MentionService, ReportService |
| `planning` | Nedelnoe planirovanie menedzherov | PlanningService |
| `assist` | CEO daily planner (lichnye chaty) | PlannerService |
| `operator` | Stub dlya budushchey fazy | -- |
| `evaluation` | Stub dlya budushchey fazy | -- |
