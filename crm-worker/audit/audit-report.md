# Audit Report -- CRM Avito AI Worker

Data audita: 2026-03-24

---

## 1. Svodnaya tablica zadach i ikh statusov

| # | Task | Prioritet | Status | Kommentarij |
|---|------|-----------|--------|-------------|
| 1 | Vacancy-Only Accounts (ai_enabled) | HIGH | DONE | ai_enabled pole v DB, migracija 007/008, proverka v webhooks/incoming_processor |
| 2 | Web Admin Panel | MEDIUM | DONE | admin_web.py + admin.py + templates/admin.html, cookie auth, event_log |
| 3 | Vacancy List in Admin | MEDIUM | DONE | GET /admin/api/vacancies + stats endpoint v admin.py |
| 4 | Bitrix Agent Toggle | MEDIUM | DONE | services/bitrix_agent.py sozdana, config vars dobavleny |
| 5 | Bitrix Line Deactivation (curl) | MEDIUM | PARTIAL | Scripty obnovleny, no CRM toggle cherez curl ne podtverzhden |
| 6 | Dialog Edge Cases (phone, today) | HIGH | DONE | prompts/no_phone.txt, asks_phone.txt, booking.txt obnovlen, _handle_booking_response obrabotka |
| 7 | Two Critical Bugs (duplicates + LLM errors) | HIGH | DONE | skip_db v avito_messenger, _is_llm_error + _mark_session_failed logika |
| 8 | Fix AI Claude Retry & Timeouts | HIGH | DONE | _is_retryable(), razdelnye timeout'y (15/120/30s), OpenAI retry 3x |
| 9 | Fix Connection Reuse | CRITICAL | DONE | _claude_client() vnutri retry loop (stroka 154 ai_claude.py) |
| 10 | Cascade Bugs (debounce, scheduled check, fork) | CRITICAL | DONE | incoming_processor rewrite s debounce, session check v message_scheduler, fork logic fix |
| 11 | Duplicate Messages in DB | MEDIUM | DONE | skip_db parametr v send_message, message_scheduler ispolzuet skip_db=True |
| 12 | Fix Emulator Duplicate Phone | LOW | PARTIAL | Ne nayden check existing applicant by phone v emulate endpoint -- BUG |
| 13 | Fix Test Emulator Vacancy Selection | MEDIUM | PARTIAL | Endpointy mogut byt' v admin.py, no polnyj dropdown ne podtverzhden |
| 14 | Time Display Moscow Timezone | LOW | NE PROVERENO | Trebuetsya proverka admin.html |
| 15 | Fix Vacancy Sync Deactivation | MEDIUM | DONE | Auto-deactivation otklyuchena s kommentariem (vacancy_sync.py:291-294) |
| 16 | Handover Delivery (Telegram + Admin Cards) | HIGH | DONE | telegram_notifier.py, handover endpoints v admin.py, topic_id, morning scheduler |
| 17 | Multi-Account Refactor | HIGH | DONE | account_id vo vsekh servisakh, per-account tokens, webhook routing |
| 18 | OpenAI Proxy | LOW | NE REALIZOVANO | _call_openai_fallback bez proxy (ai_claude.py:113) |
| 19 | Retry Fallback Mechanism | HIGH | DONE | 5x Claude + 3x OpenAI, _is_llm_error, event logging |
| 20 | Parse Work Address | HIGH | DONE | vacancy_parser.py: regexp + AI fallback |
| 21 | Redis Webhook Buffer | HIGH | DONE | redis_queue.py + webhook_consumer.py + enqueue v webhooks.py + main.py consumer |
| 22 | Remove Night Time Mentions | MEDIUM | PARTIAL | system.txt obnovlen, no transliteracija v telegram_notifier ostayotsya |
| 23 | Remove SOCKS5 Proxy | MEDIUM | PARTIAL | Proxy ubran iz _claude_client, NO ostayotsya v config.py i ne ubran iz OpenAI/Telegram |
| 24 | Premature Alternatives Close | MEDIUM | DONE | waiting_alternatives_criteria stage, migracija 009, _handle_alternatives_criteria handler |
| 25 | Telegram Fixes | HIGH | DONE | Night window filter, topic_id int cast, account_id parametr |
| 26 | Test Emulator | MEDIUM | DONE | emulate-application/message endpointy, test-chat- prefix interception |
| 27 | Webhook Bitrix CRM Toggle | MEDIUM | PARTIAL | Scripty obnovleny chastichno |

### Statistika

| Status | Kolichestvo |
|--------|-------------|
| DONE | 20 |
| PARTIAL | 5 |
| NE REALIZOVANO | 1 |
| NE PROVERENO | 1 |
| **Vsego** | **27** |

---

## 2. Detaljnyj razbor zadach

### TASK-01: Vacancy-Only Accounts (ai_enabled)
**Status: DONE**
- `models/db.py:56` -- `ai_enabled = Column(Boolean, default=False)` dobavleno v AvitoAccount
- `api/webhooks.py` -- proverka ai_enabled pered zapuskom AI
- `workers/incoming_processor.py` -- proverka ai_enabled
- Migracii `007_ai_enabled.sql` i `008_add_vacancy_accounts.sql` sozdany
- Vacancy sync rabotaet dlya vsekh akkauntov nezavisimo ot ai_enabled

### TASK-02: Web Admin Panel
**Status: DONE**
- `api/admin_web.py` -- login/logout, signed cookie sessions (HMAC, 24h expiry)
- `api/admin.py` -- 15+ REST endpointov (applicants, chats, handover, vacancies, stats, export)
- `templates/admin.html` -- SPA s razdelami
- `utils/event_logger.py` -- helper dlya logirovaniya
- `models/db.py:240-250` -- EventLog model'
- Migracija `004_event_log.sql` sozdana

### TASK-03: Vacancy List in Admin
**Status: DONE**
- Endpointy vacancies v admin.py s filtrami (account, active, search, pagination)

### TASK-04: Bitrix Agent Toggle
**Status: DONE**
- `services/bitrix_agent.py` (45 strok) -- toggle on/off
- Config: `bitrix_agent_toggle_url`, `bitrix_agent_secret`, `bitrix_agent_ids`

### TASK-05: Bitrix Line Deactivation
**Status: PARTIAL**
- Scripty obnovleny, no ne podtverzhdeno chto curl ispolzuetsya vmesto httpx

### TASK-06: Dialog Edge Cases
**Status: DONE**
- `prompts/no_phone.txt` i `prompts/asks_phone.txt` sozdany
- `prompts/booking.txt` obnovlen (mention "tomorrow")
- `prompts/system.txt` -- pravila pro telefon i ne davit'
- `ai_agent.py` -- booking_no_phone result, phone refusal detection

### TASK-07: Two Critical Bugs
**Status: DONE**
- Bug 1 (duplicates): `skip_db: bool = False` parametr v `avito_messenger.send_message()` (stroka 32)
- Bug 2 (LLM errors): `_is_llm_error()` (stroka 1279), `_mark_session_failed()` propuskaet LLM-oshibki (stroka 1285)

### TASK-08: Fix AI Claude Retry & Timeouts
**Status: DONE**
- `_is_retryable()` raspoznayet setevye oshibki (stroka 60-81)
- Razdelnye timeout'y: connect=15s, read=120s, write=30s (stroki 29-31)
- OpenAI fallback: 3 retry s delays [3, 10, 30] (stroka 230-231)

### TASK-09: Fix Connection Reuse
**Status: DONE**
- `async with _claude_client() as client:` vnutri retry loop (stroka 154)
- Kazhdaya popytka poluchaet svezheye soedineniye

### TASK-10: Cascade Bug Fixes
**Status: DONE**
- Debounce v incoming_processor.py (10s per chat, stroka 15)
- Session status check v message_scheduler.py (stroka 116)
- Fork handler logic v ai_agent.py

### TASK-11: Duplicate Messages in DB
**Status: DONE**
- `skip_db=True` v message_scheduler.process_scheduled() (stroka 132)
- Dublikaty ne sozdayutsya pri otpravke cherez scheduler

### TASK-12: Fix Emulator Duplicate Phone
**Status: PARTIAL**
- Trebuetsya proverka: check existing applicant by phone pered INSERT
- Vozmozhno ne realizovano -- BUG (sm. razdel 3)

### TASK-13: Fix Test Emulator Vacancy Selection
**Status: PARTIAL**
- Osnova rabotaet, no polnyj dropdown vacancy selection ne podtverzhden

### TASK-14: Time Display Moscow Timezone
**Status: NE PROVERENO**
- Trebuetsya proverka admin.html na nalichiye toMoscow() funkcii

### TASK-15: Fix Vacancy Sync Deactivation
**Status: DONE**
- Auto-deactivation otklyuchena: `vacancy_sync.py:291-294`
- Kommentarij: "API mozhet vozvrashchat' nepolnyj spisok"

### TASK-16: Handover Delivery
**Status: DONE**
- `services/telegram_notifier.py` -- morning summary per account per topic
- Handover endpointy v admin.py (GET /admin/api/handover, process)
- `telegram_topic_id` v AvitoAccount (stroka 54)
- Scheduler job v main.py

### TASK-17: Multi-Account Refactor
**Status: DONE**
- `account_id` vo vsekh servisakh (messenger, auth, applications, agent)
- Per-account token refresh v token_refresher
- Webhook routing po user_id v webhooks.py

### TASK-18: OpenAI Proxy
**Status: NE REALIZOVANO**
- `_call_openai_fallback()` (ai_claude.py:113) -- httpx.AsyncClient BEZ proxy
- Yesli router-level routing nastroyen, eto ne problema; inache OpenAI zaprosy budut napryamuyu

### TASK-19: Retry Fallback Mechanism
**Status: DONE**
- 5x Claude s RETRY_DELAYS = [2, 5, 15, 30, 60]
- 3x OpenAI s delays = [3, 10, 30]
- event_log dlya retry/fallback events
- _is_llm_error() -- sessiya ne padayet pri LLM-oshibkakh

### TASK-20: Parse Work Address
**Status: DONE**
- `services/vacancy_parser.py` -- parse_work_address_regexp() + parse_work_address_ai()
- Markery: "Adres ob'yekta:", "Adres mesta raboty:", etc.

### TASK-21: Redis Webhook Buffer
**Status: DONE**
- `services/redis_queue.py` -- Redis Streams (enqueue, consume, ack, recovery)
- `workers/webhook_consumer.py` -- async consumer
- `api/webhooks.py` -- enqueue + sync fallback
- `main.py` -- consumer start v lifespan
- `config.py` -- Redis settings

### TASK-22: Remove Night Time Mentions
**Status: PARTIAL**
- system.txt obnovlen
- `telegram_notifier.py` -- transliteraciya vmesto russkogo teksta (sm. bug #6)

### TASK-23: Remove SOCKS5 Proxy
**Status: PARTIAL**
- Proxy ubran iz `_claude_client()` (ai_claude.py:48-57) -- net proxy parametra
- Proxy NE ubran iz: config.py:21 (`claude_proxy`), vozmozhen v ai_rag.py
- Parametr `claude_proxy` ostayotsya v config -- mozhet vyzyvat' konfuziyu

### TASK-24: Premature Alternatives Close
**Status: DONE**
- `waiting_alternatives_criteria` stage v models/db.py:180
- Migracija 009_alternatives_criteria_stage.sql
- Handler `_handle_alternatives_criteria()` v ai_agent.py

### TASK-25: Telegram Fixes
**Status: DONE**
- Night window filter: 21:00-09:00 (telegram_notifier.py:79-83)
- `int(topic_id)` cast (stroka 28)
- `account_id` parametr (stroka 64)
- Proxy dlya Telegram NE dobavlen (sm. bug #7)

### TASK-26: Test Emulator
**Status: DONE**
- `avito_messenger.py:38` -- test-chat- prefix interception
- Emulate endpointy v admin.py

### TASK-27: Webhook Bitrix CRM Toggle
**Status: PARTIAL**
- Scripty obnovleny chastichno

---

## 3. Najdennye bagi

### BUG-01: CRITICAL -- Hardcoded credentials v config.py
**Fajl:** `config.py:11,19,24,36,45-48,65,73`

Vse sekrety (DB password, API keys Anthropic/OpenAI, Redis password, admin password, Telegram bot token, Bitrix secret) zahardkozheny v kode vmesto .env. Oni vidny v git history.

**Posledstviya:** Polnaya kompromyetaciya vsekh API i bazy dannyh pri utechke repozitoriya.

**Fix:** Udalit' vse default znacheniya dlya sekretov, trebovat' .env fajl. Rotatirovat' kompromyetirovannye klyuchi.

---

### BUG-02: HIGH -- Nevalidnaya model' openai_fallback_model
**Fajl:** `config.py:27`

`openai_fallback_model: str = "gpt-5.4"` -- takoy modeli ne sushchestvuyet. Odnovremenno v `ai_claude.py:17` ispol'zuyetsya konstanta `OPENAI_FALLBACK_MODEL = "gpt-4o"`.

**Posledstviya:** Config pole ignoriruyetsya (v pol'zu hardcoded "gpt-4o"), no eto nesootvetstviye mozhet privesti k oshibke pri refactoringe.

**Fix:** Ispravit' na `"gpt-4o"` v config.py. Ispol'zovat' `settings.openai_fallback_model` v ai_claude.py vmesto konstanty.

---

### BUG-03: HIGH -- NameError: candidate_speed v process_followup
**Fajl:** `services/ai_agent.py:397`

```python
delay = calc_message_delay(candidate_response_sec=candidate_speed)
```

Peremennaya `candidate_speed` nikogda ne opredelyayetsya vnutri funktsii `process_followup()` (stroka 353). Ona est' tol'ko v `process_incoming_message()` (stroka 328).

**Posledstviya:** `NameError` pri vyzove followup -- soobshcheniye ne budet otpravleno, sessiya padayet v failed.

**Fix:** Zamenit' na `calc_message_delay(candidate_response_sec=None)` ili dobavit' `candidate_speed = None` pered strochkoj 397.

---

### BUG-04: MEDIUM -- Duplicate _extract_json v 3 fajlakh
**Fajly:**
- `services/ai_agent.py:96`
- `services/segmentation.py:22`
- `services/vacancy_parser.py:71`

Funktsiya `_extract_json()` dublirovana v 3 mestakh s razlichayushcheysya logikoy parsinga.

**Fix:** Vyneye v `utils/json_helpers.py` i importirovat' vezde.

---

### BUG-05: MEDIUM -- Race condition v message_scheduler
**Fajl:** `services/message_scheduler.py:100-141`

Soobshcheniye poluchayetsya iz DB (stroka 88-98) BEZ blokirovki, zatem otpravlyayetsya (stroka 132), zatem obnovlyayetsya delivered_at (stroka 135-139). Pri parallel'nom vyzove -- dublikat.

**Fix:** Ispol'zovat' SELECT ... FOR UPDATE ili atomarnoye obnovleniye.

---

### BUG-06: MEDIUM -- Telegram notifier: transliteratsiya vmesto russkogo
**Fajl:** `services/telegram_notifier.py:1,14,39-61,121-127,140`

Ves' tekst v Telegram soobshcheniyakh na translite:
- "Utrennyaya svodka" vmesto "Utrennyaya svodka"
- "Zapisan na zvonok" vmesto normalnogo teksta
- "Gorod", "Vakansiya", "Rezultat", etc.

**Fix:** Zamenit' transliteraciyu na russkij tekst vo vsekh string'akh.

---

### BUG-07: LOW -- Telegram API bez proxy
**Fajl:** `services/telegram_notifier.py:30`

`httpx.AsyncClient(timeout=15.0)` -- bez proxy. Yesli Telegram zablokirovan.

**Fix:** Dobavit' proxy ili podtverdit' router-level routing.

---

### BUG-08: LOW -- Neispol'zuyemyj config parametr claude_proxy
**Fajl:** `config.py:21`

`claude_proxy: str = "socks5://127.0.0.1:1080"` ostayotsya v config, khotya proxy ubran iz _claude_client(). Konfuziya.

**Fix:** Udalit' iz config yesli proxy bol'she ne nuzhen, ili vernut' ispol'zovaniye.

---

### BUG-09: MEDIUM -- Type hints candidate_speed: float = 1.0
**Fajl:** `services/ai_agent.py:923,1028`

Dva handler'a imeyut `candidate_speed: float = 1.0` vmesto `candidate_speed: int | None = None`. Default 1.0 privodit k delay v 1 sekundu.

**Fix:** Ispravit' na `candidate_speed: int | None = None`.

---

### BUG-10: LOW -- DEPRECATED funktsiya ne udalena
**Fajl:** `utils/time_helpers.py:88-91`

`calc_channel_choice_delay()` pomechena DEPRECATED. Mertvyj kod.

**Fix:** Udalit'.

---

### BUG-11: MEDIUM -- phone unique constraint + emulator
**Fajl:** `models/db.py:68`

`phone = Column(String(20), unique=True)` -- emulator (TASK-12) ne proveryayet sushchestvuyushchego applicant'a po telefonu, mozhet padat' s IntegrityError.

**Fix:** V emulate endpoint dobavit' SELECT by phone pered INSERT.

---

## 4. Otkrytye TODO/FIXME/DEPRECATED

| # | Fajl | Stroka | Tip | Opisaniye |
|---|------|--------|-----|-----------|
| 1 | `utils/time_helpers.py` | 90 | DEPRECATED | `calc_channel_choice_delay()` -- channel_choice ubran iz flow |
| 2 | `services/vacancy_sync.py` | 291-294 | Implicit TODO | Auto-deactivation otklyuchena "dlya bezopasnosti" -- nuzhno resheniye |
| 3 | `config.py` | 27 | Implicit TODO | `openai_fallback_model: str = "gpt-5.4"` -- nevalidnaya model' |
| 4 | `config.py` | 48 | Implicit TODO | `admin_secret_key: str = "change-me-in-production"` -- placeholder |

---

## 5. Rekomendacii po prioritetam

### Neotlozhno (do sleduyushchego deploya)
1. **BUG-01**: Udalit' hardcoded sekrety iz config.py, rotatirovat' kompromyetirovannye klyuchi
2. **BUG-03**: Ispravit' `candidate_speed` v `process_followup()` -- NameError v runtime
3. **BUG-09**: Ispravit' type hints `candidate_speed: float = 1.0` -> `int | None = None`

### Vazhnye (v blizhajshiye nedeli)
4. **BUG-02**: Sinhronizirovat' `openai_fallback_model` mezhdu config i ai_claude.py
5. **BUG-05**: Ispravit' race condition v message_scheduler (SELECT FOR UPDATE)
6. **BUG-06**: Perevesti telegram_notifier na russkij yazyk
7. **BUG-04**: Vyneye _extract_json v utils/

### Nizhnij prioritet
8. **BUG-10**: Udalit' deprecated funkciyu
9. **TASK-18**: Dobavit' proxy v OpenAI fallback
10. **TASK-12**: Dobavit' proverku applicant by phone v emulator
11. **BUG-08**: Udalit' ili ispol'zovat' claude_proxy v config
