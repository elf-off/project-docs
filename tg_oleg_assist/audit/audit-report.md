# Audit Report -- tg_oleg_assist

**Data: 2026-03-24**
**Commit: 572dba9 (main, clean)**

---

## 1. Svodka po zadacham (Task Summary)

| Spec / Task | Status | Kritichnye zamechaniya |
|---|---|---|
| SPEC_planner_integration.md | DONE | Minornye otkloneniya v enum-akh i nazvaniyakh kolonok |
| SPEC_bitrix_integration.md | DONE | `is_bitrix_message` proverit exact match vmesto startswith |
| SPEC_growth_plan_fact_v3.md | DONE | Otchet ob'edinen (uluchshenie); dublirovanie `_get_weekly_exits_db` |
| TASK_fix_baseline_recalc.md | DONE | Bez zamechaniy |
| FIX_baseline_recalc.md | DONE | Dokumentaciya sootvetstvuet realizacii |

---

## 2. Detalnyi razbor zadach

### 2.1 SPEC_planner_integration.md (Planner GD v1.3)

**Status: DONE** -- Vse osnovnye funkcii realizovany: utrennee soobschenie (09:00), dva cikla obratnoy svyazi (14:00, 20:30), FSM, analiz Claude, auto-close (21:30).

**Otkloneniya ot specifikacii:**

1. **MeetingStatus enum** -- spec opredelyaet `held/postponed/not_held/no_response`, realizaciya ispolzuet `completed/partial/failed/no_response` (`planner_day_themes.py:23-35`). Eto sootvetstvuet bolee novym nazvaniyam knopok v1.3.

2. **DayOfWeek enum** -- spec opredelyaet string enum (`MONDAY = "Monday"`), realizaciya ispolzuet int enum (`MONDAY = 0`) v `planner_day_themes.py:14-20`. Rabotaet korrektno.

3. **Metriki exits_wa** -- spec ukazyvaet kolonki `date/vykhod/zayavka`, realizaciya zapraschivaet `exit_date` i `type_exit` s znacheniyami `plan/fact` (`planner_service.py:494-503`). Vozmozhnoe nesootvetstvie s vneshney BD.

4. **APScheduler vs TimerWorker** -- spec predlagaet APScheduler, realizaciya ispolzuet suschestvuyuschy TimerWorker (30-sek polling). Validnaya adaptaciya.

### 2.2 SPEC_bitrix_integration.md (Bitrix24)

**Status: DONE** -- Regex-parsing, AI-fallback, normalizaciya telefonov, Bitrix24 REST API, BD-logirovanie realizovany.

**Otkloneniya:**

1. **`is_bitrix_message`** -- spec govorit "pervaya stroka nachinaetsya s `#Bitrix`", realizaciya v `bitrix_service.py:34-35` proverit tochnoe ravenstvo `first_line.lower() == '#bitrix'`. Soobscheniya vida `#Bitrix Petrov` ne budut obrabotany.

### 2.3 SPEC_growth_plan_fact_v3.md (Plan-Fact)

**Status: DONE** -- Baseline pri otpravke plana, proverka fakta v ponedelnik, pereschet baseline, formatirovanie otcheta, logirovanie v growth_reports.

**Otkloneniya:**

1. **Ob'edinyonnyy otchet** -- spec opisyvaet otchet na kazhdogo menedzhera, realizaciya v `timer_worker.py:595-732` formiruet odin ob'edinyonnyy otchet. Eto uluchshenie.

2. **Deduplikaciya submissions** -- realizaciya v `timer_worker.py:634-655` obrabacyvaet dubli (posledniy submission na menedzhera). V speke etogo net.

3. **Dublirovanie koda** -- `_get_weekly_exits_db` v `timer_worker.py:869-909` -- kopiya logiki iz `planning_service.py`. Narushaet DRY.

### 2.4 TASK_fix_baseline_recalc.md

**Status: DONE** -- Fix prisutstvuet v `timer_worker.py:793-811`. Poryadok operaciy (fakt -> pereschet baseline -> vychislenie rosta -> status) sootvetstvuet specifikacii.

---

## 3. Naidennye bagi

### Kritichnye (CRITICAL)

Net kritichnykh bagov.

### Bagi (BUG)

| # | Fayl | Stroka | Opisanie |
|---|------|--------|----------|
| B1 | `app/services/timer_worker.py` | ~47 | `is_workday()` ispolzuet `datetime.now().weekday()` -- lokalnoe vremya servera vmesto moskovskogo. Esli server v UTC, rabochie dni opredelyayutsya nepravilno. |
| B2 | `app/services/timer_worker.py` | ~257 | Proverka prosrochennykh mentionov cherez `datetime.now()` -- naivnoe lokalnoe vremya. Mentiony mogut srabatyvat ranshe/pozshe na 3 chasa. |
| B3 | `app/services/notification.py` | ~391 | `restrict_chat_member` ispolzuet `datetime.now()` dlya `until_date`. Telegram ozhidaet UTC. Blokirovka mozhet dlitsya 4 chasa vmesto 1. |
| B4 | `app/services/planner/planner_service.py` | ~284 | `state.expires_at.replace(tzinfo=tz)` s pytz ne vypolnyaet konversiyu -- prosto naznachaet TZ bez korrektirovki vremeni. TTL proverka mozhet byt smeschena na 3 chasa. |
| B5 | `app/services/planner/planner_service.py` | ~176 | `end_of_day()` vsegda otpravlyaet soobschenie o zakrytii dnya, dazhe esli oby cikla uzhe zaversheny. CEO poluchaet lishnie uvedomleniya. |
| B6 | `app/services/message_router.py` | ~632-633 | `_handle_callback_query` obraschaetsya k vlozhennym dict bez zashchity: `callback_query["from"]["id"]`, `callback_query["message"]["chat"]["id"]`. Pri nestandartnoy strukture -- KeyError. |
| B7 | `app/services/rop_validator.py` | ~523-528 | `_build_ai_response` vyzyvaet `client.analyze()` kotoryy vozvrashchaet JSON, no prompt ozhidaet chistyy HTML-tekst. AI-generaciya otveta fakticheski nikogda ne rabotaet cherez `analyze()`, vsegda fallback. |

### Preduprezhdeniya (WARNING)

| # | Fayl | Stroka | Opisanie |
|---|------|--------|----------|
| W1 | `app/services/timer_worker.py` | ~846, 853 | SQL cherez f-string: `f"WHERE id IN ({ids_str})"`. Znacheniya iz BD (integer), no net parametrizovannykh zaprosov. |
| W2 | `app/services/timer_worker.py` | ~66-68 | `_report_checks_done` / `_planner_checks_done` / `_growth_checks_done` -- in-memory sety rastut neogranicheno. Utechka pamyati pri dolgoy rabote. |
| W3 | `app/services/timer_worker.py` | ~557-559 | Smeshivanie tz-aware i naive datetime v `_try_planner_job`. Vozmozhnye oshibki okolo polunochi. |
| W4 | `app/services/timer_worker.py` | ~770-771 | Dinamicheskie atributy na SQLAlchemy modelyakh (`entry._project_name`). Hrupkoe reshenie. |
| W5 | `app/services/message_router.py` | ~428 | `user_score_before = max(0, user_score_after - result.total_weight)` -- predpolagaet chto izmenyeniye = total_weight. Esli ScoringEngine primenyaet decay, mozhet narushat porogovuyu logiku (6/8/10). |
| W6 | `app/services/message_router.py` | ~97-101 | Bot sozdayotsya zanovo dlya kazhdogo Bitrix-soobscheniya vmesto ispolzovaniya obscheboy instancii. Lishnie HTTP-sessii. |
| W7 | `app/services/notification.py` | ~488-516 | Net proverki dliny soobscheniya pered otpravkoy. Telegram limit -- 4096 simvolov. Dlinnye soobscheniya vyzovut TelegramError. |
| W8 | `app/services/planning_service.py` | ~303-310 | `tz.localize()` mozhet byt neodnoznachnym vo vremya perehoda na zimnyee/letnyee vremya. |
| W9 | `app/services/planning_service.py` | ~482-512 | Net validacii AI-vozvraschennyh `project_id`/`group_id` protiv spiska proektov. Gallyucinirovannyy ID sozdast zapis s nesuschstvuyuschim proektom. |
| W10 | `app/services/planning_service.py` | ~625-629 | Udalenie `planning_entries` cherez raw SQL, zatem `PlanningSubmission` cherez ORM. Vozmozhen konflikt s cascade. |
| W11 | `app/services/planner/planner_service.py` | ~271-276 | `handle_text_message` vsegda beryot pervoye aktivnoye sostoyaniye (morning). Esli utrennyy cikl ne zavershyon, otvet uydet tuda vmesto vechernogo. |
| W12 | `app/services/rop_validator.py` | ~94, 211, 355 | Regex `[...warning_sign...]` -- soderzhit multi-codepoint simvoly. Variation selector mozhet srabatyvat otdelno ot osnovnogo simvola. |
| W13 | `app/services/rop_validator.py` | ~490-491 | `_deduplicate_violations` ispolzuet pervye 40 simvolov kak klyuch. Raznye narusheniya s odnim nachalom budut ob'edineny. |

---

## 4. Narusheniya code style

### Fayly prevyshayuschie 300 strok (pravilo: max ~300 strok na fayl)

| Fayl | Strok | Kommentariy |
|------|-------|-------------|
| `app/services/timer_worker.py` | 1046 | Nuzhdaetsya v razdelenii: mentions, reports, planner, growth |
| `app/services/planning_service.py` | 750 | Mozhno vydelit validaciyu i AI-parsing |
| `app/services/message_router.py` | 703 | Mozhno vydelit obrabotchiki po rezhimam |
| `app/services/rop_validator.py` | 631 | Mozhno vydelit AI-validaciyu |
| `app/services/planner/planner_service.py` | 631 | Mozhno vydelit analitiku |
| `app/services/notification.py` | 618 | Mozhno vydelit formatirovanie |
| `app/services/planner/planner_day_themes.py` | 539 | Danyye raspisaniya, dopustimo |
| `app/services/report_service.py` | 536 | Mozhno vydelit ROP-logiku |
| `app/services/bitrix_service.py` | 459 | Mozhno vydelit parsing |
| `app/services/old_rop_validator.py` | 371 | DEPRECATED -- udalit |

### Emoji v ishodnom kode

Emoji shiroko ispolzuyutsya v user-facing soobscheniyakh (notification.py, rop_validator.py, timer_worker.py, planning_service.py, planner_service.py, monthly_exits_by_le.py). Formalno narushaet pravilo "no emoji in source code", no neobhodimo dlya Telegram-soobscheniy.

### Ustarevshy kod

| Fayl | Prichina |
|------|----------|
| `app/services/old_ai_client.py` (134 str) | Zamenyonen na `ai_client.py` |
| `app/services/old_rop_validator.py` (371 str) | Zamenyonen na `rop_validator.py` |
| `app/config.py` -- polya `openai_api_key`, `openai_model` | Ne ispolzuyutsya, migrirovali na Anthropic |

### DEPRECATED metody

| Fayl | Stroka | Metod |
|------|--------|-------|
| `app/services/planner/planner_formatter.py` | 79 | `format_morning_message()` -- DEPRECATED |
| `app/services/planner/planner_formatter.py` | 108 | `format_partial_message_old()` -- DEPRECATED |
| `app/services/planner/planner_formatter.py` | 112 | `format_failed_message_old()` -- DEPRECATED |

---

## 5. Otkrytye TODO/FIXME

**V ishodnom kode app/ i tests/ ne naideno ni odnogo yavnogo TODO ili FIXME.**

Vse zadachi iz docs/tasks/ imeyut status DONE.

---

## 6. Prochie zamechaniya

### Testirovanie

- 12 testovykh faylov (plain assertion scripts, ne pytest)
- Net testov dlya: `bitrix_service.py`, `report_service.py`, `monthly_exits_by_le.py`
- `test_manual_morning.py:23` -- hardcoded put `/mnt/d/server_emul/...`, ne budet rabotat na drugikh mashinakh
- Net testov konkurentnoy obrabotki (race conditions v parallel webhook processing)

### Zavisimosti

- `requirements.txt` soderzhit `openai==1.12.0` i `tenacity==8.2.3` -- ispolzuyutsya tolko v `old_ai_client.py` (deprecated). Mozhno udalit.
- `alembic==1.13.1` prisutstvuet no ne ispolzuetsya (migracii ruchnye SQL)

### Bezopasnost

- SQL cherez f-string v `timer_worker.py:846,853` -- nizkiy risk (znacheniya iz BD), no narushaet best practices
- `001_multichat_architecture.sql:65` -- hardcoded chat ID v migracii

### Arkhitektura

- IdempotencyGuard -- in-memory, ne perezhivaet restart servera
- TimerWorker dedup sety -- ne ochischayutsya, rastut beskonechno
- `resolve_all_with_feature` zagruzhaet VSE chaty -- ne masshtabiruetsya
- Sinhronnyy Anthropic SDK v async kode (blokiruet event loop) -- ukazano v docs/man/10_nuances.md
