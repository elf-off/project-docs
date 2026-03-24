# СПЕЦИФИКАЦИЯ: Интеграция модуля «Планёр ГД» в существующего CEO Agent

**Версия**: 1.3
**Дата**: 11.02.2026
**Назначение**: Передача в чат разработки для реализации

> ### ⚠️ КРИТИЧНЫЕ ОТЛИЧИЯ ОТ ИСХОДНЫХ ФАЙЛОВ
> 1. **Проект асинхронный** (async/await) — все методы, DB-операции, HTTP-клиенты должны быть async
> 2. **Конфигурация в БД** (`chats.config`), а НЕ в `.env` — не создавать переменных окружения для планёра
> 3. **Два цикла обратной связи в день**: 14:00 (блоки 10:00–14:00) и 20:30 (блоки 15:00–19:00)
> 4. **Утреннее сообщение (09:00) — информационное**, без кнопок. Кнопки отправляются в 14:00 и 20:30
> 5. **Автозакрытие в 21:30** (а не в 20:00). Напоминание в 17:30 убрано
> 6. **Webhook**: в `allowed_updates` обязательно включить `callback_query`
> 7. **Формулировки кнопок**: ✅ Выполнено / ⏳ Частично / ❌ Не удалось (вместо Проведено/Перенесено/Не проводилось)

---

## 1. КОНТЕКСТ

### 1.1 Что уже есть

На сервере (Debian 12) работает **CEO Agent** — Telegram-агент для мониторинга выходов персонала.

**Расположение**: `/opt/ai-agents/ceo_agent/`

**Текущая структура проекта**:
```
/opt/ai-agents/ceo_agent/
├── main.py                          # Точка входа, asyncio.run()
├── config/
│   └── settings.py                  # Конфигурация через env-переменные
├── src/
│   ├── orchestrator.py              # CEOAgent — основной класс, APScheduler
│   ├── telegram/
│   │   └── bot.py                   # TelegramBotManager (только отправка, без polling)
│   ├── storage/
│   │   ├── models.py                # SQLAlchemy-модели (Base)
│   │   └── database.py              # DatabaseManager, MariaDB-сессии
│   ├── connectors/
│   │   └── data_ingestion.py        # Загрузка данных из exits_wa
│   ├── analysis/
│   │   └── weekly_ranking.py        # Еженедельный рейтинг
│   └── reporting/
│       └── ceo_reports.py           # Генерация отчётов для CEO
├── .env                             # Секреты
└── test_agent.py
```

**Ключевые факты**:
- **БД**: MariaDB (база `2_kadry_4`)
- **ORM**: SQLAlchemy (⚠️ проект **асинхронный** — async/await)
- **Конфигурация**: хранится в БД (`chats.config`), а **НЕ** в `.env`
- **Telegram**: основной бот работает через **webhook** (другой процесс). CEO Agent использует только объект `Bot` для исходящих сообщений через `send_ceo_alert()`
- **Планировщик**: APScheduler (AsyncIOScheduler) внутри `CEOAgent.start()`
- **Логирование**: `structlog` (JSON-логи)
- **Сервис**: systemd unit `ceo-agent`
- **Существующие таблицы (read-only)**: `ref_projects_wa`, `ref_shifts`, `exits_wa`, `ref_project_chats`
- **Таблицы агента (read-write)**: `agent_snapshots`, `agent_snapshot_records`, `agent_incidents`, `agent_risk_scores`, `agent_audit_log`, и др.

### 1.2 Что нужно добавить

Модуль **«Планёр ГД»** — ежедневный управленческий цикл:
- Утреннее информационное сообщение с повесткой на весь день (09:00 Пн–Пт, БЕЗ кнопок)
- Запрос обратной связи по первой половине дня (14:00, С кнопками)
- Запрос обратной связи по второй половине дня (20:30, С кнопками)
- Сбор обратной связи по совещаниям (inline-кнопки + текстовые ответы)
- Анализ через **Claude API** (Anthropic)
- Рекомендации, прогноз рисков, задачи на завтра

### 1.3 Исходные файлы модуля

К этой спецификации прилагаются файлы-исходники, написанные под другой стек (PostgreSQL + asyncpg + Redis + отдельный polling-бот). Их нужно **адаптировать**, а не использовать напрямую.

**Что забрать из исходников** (ценная бизнес-логика):
- `config.py` → темы дней `DAY_THEMES`, чек-листы, эмодзи
- `models.py` → Pydantic-модели: `AnalysisPayload`, `AnalysisResponse`, `MeetingFeedback`, `Action`, `RiskForecast`
- `analysis_api.py` → system prompt для LLM (управленческие модели Адизеса, формат JSON-ответа)
- `message_formatter.py` → форматирование Telegram-сообщений

**Что НЕ использовать** (нужно переписать):
- `bot.py` — целиком; нельзя запускать отдельный polling
- `redis_store.py` — FSM хранить в MariaDB
- `database.py`, `init_db.py` — другая БД

---

## 2. АРХИТЕКТУРНЫЕ РЕШЕНИЯ

### 2.1 Где живёт модуль

Новые файлы добавляются в существующую структуру CEO Agent:

```
src/
├── planner/                          # НОВЫЙ МОДУЛЬ
│   ├── __init__.py
│   ├── planner_service.py            # Бизнес-логика планёра (FSM, оркестрация)
│   ├── claude_client.py              # Клиент Claude API (Anthropic)
│   ├── prompts.py                    # System prompt и формирование user message
│   ├── formatter.py                  # Форматирование Telegram-сообщений
│   └── day_themes.py                 # Темы дней, чек-листы (из config.py исходников)
├── storage/
│   └── models.py                     # + НОВЫЕ модели (PlannerFeedback, PlannerAnalysisLog, PlannerState)
├── orchestrator.py                   # + НОВЫЕ scheduled jobs для планёра
└── telegram/
    └── bot.py                        # + НОВЫЕ методы отправки для планёра
```

### 2.2 Взаимодействие с Telegram

**Проблема**: CEO Agent только отправляет сообщения. Он не принимает входящие (это делает основной webhook-бот).

**Решение**: В основном боте (который работает через webhook) нужно добавить **роль `assist`** — обработчик inline-кнопок и текстовых сообщений для планёра. Эта роль:
1. Принимает callback_query от inline-кнопок планёра
2. Принимает текстовые ответы на вопросы FSM
3. Вызывает методы `PlannerService` для обработки
4. Или: CEO Agent сам пишет в БД, а webhook-бот читает из БД

**Альтернатива (проще для MVP)**: CEO Agent отправляет сообщения без inline-кнопок. ГД отвечает текстом в определённом формате (например, цифрой для оценки). Основной webhook-бот маршрутизирует сообщения от ГД (по chat_id) в PlannerService.

**Решение по маршрутизации нужно принять в чате разработки совместно с теми, кто делает основного бота.**

### 2.3 Хранение состояния FSM

Вместо Redis — таблица `planner_state` в MariaDB. У планёра один пользователь (ГД), нагрузка минимальная, TTL реализуется через поле `expires_at`.

### 2.4 API анализа — Claude (Anthropic)

Формат запроса к Claude API отличается от OpenAI:

```python
# OpenAI (так в исходниках — НЕ ИСПОЛЬЗОВАТЬ)
headers = {"Authorization": "Bearer sk-..."}
body = {"model": "gpt-4", "messages": [...]}
response["choices"][0]["message"]["content"]

# Claude API (НУЖНО РЕАЛИЗОВАТЬ)
headers = {
    "x-api-key": "sk-ant-...",
    "anthropic-version": "2023-06-01",
    "content-type": "application/json"
}
body = {
    "model": "claude-sonnet-4-5-20250929",
    "max_tokens": 4096,
    "system": system_prompt,        # system prompt — отдельное поле, НЕ в messages
    "messages": [
        {"role": "user", "content": user_message}
    ],
    "temperature": 0.3
}
# Ответ:
response["content"][0]["text"]      # НЕ choices[0].message.content
```

**URL**: `https://api.anthropic.com/v1/messages`

**Альтернатива — Python SDK** (проще, чем raw HTTP):
```python
# pip install anthropic
from anthropic import AsyncAnthropic

client = AsyncAnthropic(api_key="sk-ant-...")
message = await client.messages.create(
    model="claude-sonnet-4-5-20250929",
    max_tokens=4096,
    system=system_prompt,
    messages=[{"role": "user", "content": user_message}],
    temperature=0.3
)
result_text = message.content[0].text
```

---

## 3. НОВЫЕ ТАБЛИЦЫ (MariaDB)

Добавить в `src/storage/models.py` и создать через `Base.metadata.create_all()`.

> ⚠️ Проект асинхронный. SQLAlchemy-модели определяются стандартно (declarative), но обращения к БД — через `async_session`. Адаптировать под существующий паттерн работы с БД в проекте.

### 3.1 `planner_feedback`

Обратная связь ГД по блокам дня. **Два цикла в день** — утро (morning) и вечер (afternoon).

```sql
CREATE TABLE planner_feedback (
    id INT AUTO_INCREMENT PRIMARY KEY,
    date DATE NOT NULL,
    user_id BIGINT NOT NULL,
    half VARCHAR(10) NOT NULL COMMENT 'morning (10:00-14:00) или afternoon (15:00-19:00)',
    day_context VARCHAR(20) NOT NULL COMMENT 'Monday, Tuesday, ...',
    meeting_status VARCHAR(20) NOT NULL COMMENT 'held, postponed, not_held, no_response',
    quality_score TINYINT NULL CHECK (quality_score >= 1 AND quality_score <= 10),
    what_worked TEXT NULL,
    what_to_add TEXT NULL,
    main_problem TEXT NULL,
    decision_taken TEXT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_pf_date_user (date, user_id),
    UNIQUE KEY uk_pf_date_user_half (date, user_id, half)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 3.2 `planner_analysis_logs`

Логи вызовов Claude API.

```sql
CREATE TABLE planner_analysis_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    feedback_id INT NOT NULL,
    api_request JSON NOT NULL,
    api_response JSON NOT NULL,
    latency_ms INT NOT NULL,
    model VARCHAR(100) NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_pal_feedback (feedback_id),
    CONSTRAINT fk_pal_feedback FOREIGN KEY (feedback_id) REFERENCES planner_feedback(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 3.3 `planner_state`

FSM-состояние (вместо Redis). Поле `half` определяет какой цикл обратной связи сейчас активен.

```sql
CREATE TABLE planner_state (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    half VARCHAR(10) NOT NULL COMMENT 'morning или afternoon',
    current_step VARCHAR(50) NOT NULL COMMENT 'awaiting_status, awaiting_quality, ...',
    date DATE NOT NULL,
    day_context VARCHAR(20) NOT NULL,
    meeting_status VARCHAR(20) NULL,
    quality_score TINYINT NULL,
    what_worked TEXT NULL,
    what_to_add TEXT NULL,
    main_problem TEXT NULL,
    decision_taken TEXT NULL,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    expires_at DATETIME NOT NULL COMMENT 'TTL — удалять записи старше этого времени',
    UNIQUE KEY uk_ps_user_half (user_id, half)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 4. КОНФИГУРАЦИЯ ПЛАНЁРА

### ⚠️ Конфигурация хранится в БД, а НЕ в .env

Проект использует таблицу `chats.config` для хранения конфигурации. Настройки планёра нужно добавить туда же, по аналогии с существующими конфигами.

**Настройки планёра для `chats.config`:**

| Ключ | Значение | Описание |
|------|----------|----------|
| `planner.enabled` | `true` | Включение/выключение модуля |
| `planner.ceo_user_id` | `123456789` | Telegram user_id ГД (для FSM) |
| `planner.morning_info_time` | `09:00` | Время повестки дня (информационное, без кнопок) |
| `planner.feedback_morning_time` | `14:00` | Время запроса обратной связи по 1-й половине |
| `planner.feedback_afternoon_time` | `20:30` | Время запроса обратной связи по 2-й половине |
| `planner.eod_time` | `21:30` | Время автозакрытия незавершённых циклов |
| `planner.anthropic_api_key` | `sk-ant-...` | API-ключ Claude (Anthropic) |
| `planner.anthropic_model` | `claude-sonnet-4-5-20250929` | Модель Claude |
| `planner.anthropic_timeout` | `60` | Таймаут запроса к API (сек) |
| `planner.anthropic_max_retries` | `3` | Макс. количество ретраев |

**Важно**: `TELEGRAM_BOT_TOKEN` и `TELEGRAM_CEO_CHAT_ID` уже существуют в конфиге. Планёр использует те же значения. Конкретную структуру ключей адаптировать под существующий формат `chats.config`.

---

## 5. РЕАЛИЗАЦИЯ ПО ФАЙЛАМ

### 5.1 `src/planner/day_themes.py`

**Переписать полностью** на основе листа «Недельный график CEO» из приложенного Excel (`CEO_Weekly_Schedule-4.xlsx`). Исходный `config.py` НЕ использовать — там упрощённые темы, не соответствующие реальному расписанию.

Структура данных:

```python
from enum import Enum
from dataclasses import dataclass

class DayOfWeek(Enum):
    MONDAY = "Monday"
    TUESDAY = "Tuesday"
    WEDNESDAY = "Wednesday"
    THURSDAY = "Thursday"
    FRIDAY = "Friday"

class MeetingStatus(Enum):
    HELD = "held"
    POSTPONED = "postponed"
    NOT_HELD = "not_held"
    NO_RESPONSE = "no_response"

@dataclass
class TimeBlock:
    """Блок расписания"""
    time: str           # "10:00–10:30"
    title: str          # Краткое название
    details: list[str]  # Пункты чек-листа
    participants: str   # "CEO solo", "CEO + COO", и т.д.

@dataclass
class DaySchedule:
    """Расписание одного дня"""
    day: DayOfWeek
    theme: str              # "🧠 AI-аналитика и Board Meeting"
    blocks: list[TimeBlock]
    evening_ritual: str     # "Итог дня: 3 ключевых вывода..."
```

**Данные расписания** (из Excel, лист «Недельный график CEO»):

#### ПОНЕДЕЛЬНИК — 🧠 AI-аналитика и Board Meeting
| Время | Блок | Участники |
|-------|------|-----------|
| 10:00–10:30 | Утренний ритуал CEO: AI-дашборд (выход персонала, % невыходов, заполненность смен, алерты) | CEO solo + AI |
| 10:30–12:00 | Глубокий анализ рынка на основе AI (тренды аутсорсинга, спрос e-commerce/HoReCa, ценообразование конкурентов, прогнозные модели, новые ниши) | CEO solo + AI |
| 12:15–13:30 | Анализ операционной эффективности (юнит-экономика по объектам, COGS на 1 сотрудника, маржинальность направлений, узкие места) | CEO + COO |
| 13:30–14:00 | Подготовка к Board Meeting с COO: сверка данных, повестка, ключевые точки | CEO + COO |
| 15:00–17:00 | BOARD MEETING: Ч1 — итоги недели по отделам, синергия. Ч2 — итоги направлений, успехи/провалы. Stakeholder Theory + устойчивое развитие | COO, Реклама, Подбор, Развитие, СММ, СБ, Рук. направлений |
| 17:15–18:30 | AI-саморазвитие: разбор Zoom-записей совещаний с РН, выявление слабых мест, корректировки стиля менеджмента | CEO solo + AI |
| 18:30–19:00 | Итог дня: 3 ключевых вывода, задачи на завтра, анализ ответов менеджеров на вопросы AI | CEO solo |

#### ВТОРНИК — ⚙️ Операции и команда
| Время | Блок | Участники |
|-------|------|-----------|
| 10:00–10:30 | AI-контроль присутствия: автопроверка чек-инов менеджеров в чатах, разбор отклонений с COO | COO + AI |
| 10:30–12:00 | Операционный блок с COO (KPI бригад, жалобы, замены, проблемные объекты, метрики СБ: логистика/проживание) | CEO + COO |
| 12:15–13:30 | Проверка качества подбора (воронка, скорость закрытия, NPS внутренний) + 1-on-1 с Директором по развитию (статус лидов, конверсия) | HR-дир. + Дир. развития |
| 13:30–14:00 | Блок CEO-решений: согласования, подписи, срочные эскалации | CEO solo |
| 15:00–17:00 | Внешние коммуникации и нетворкинг (отраслевые мероприятия, вебинары, партнёрства, обход объектов) | Внешние |
| 17:15–18:30 | AI-саморазвитие: разбор Zoom-записей совещаний с РН, анализ стиля управления, обучение | CEO solo + AI |
| 18:30–19:00 | Итог дня: 3 ключевых вывода, задачи на завтра, анализ ответов менеджеров на вопросы AI | CEO solo |

#### СРЕДА — 🚀 Стратегия роста и клиенты
| Время | Блок | Участники |
|-------|------|-----------|
| 10:00–10:30 | Утренний ритуал CEO: обзор воронки продаж, статус переговоров, пайплайн клиентов | CEO solo |
| 10:30–12:00 | DEEP WORK: стратегические проекты (масштабирование, пилоты, автоматизация подбора, цифровизация, фин.моделирование) | CEO solo |
| 12:15–13:30 | Встречи с ключевыми клиентами (сервис-ревью, расширение сотрудничества, КП для тендеров) | Клиенты + Развитие |
| 13:30–14:00 | Стратегическое совещание со Службой безопасности (единственное в неделю): метрики, логистика, проживание, риски | Нач. СБ |
| 15:00–17:00 | Переговоры/презентации для потенциальных клиентов (e-commerce, ритейл, HoReCa, CRM-аналитика, ценообразование) | Отдел Развития |
| 17:15–18:30 | Работа над стратегическими проектами (продолжение Deep Work) | CEO solo |
| 18:30–19:00 | Итог дня: 3 ключевых вывода, задачи на завтра, анализ ответов менеджеров на вопросы AI | CEO solo |

#### ЧЕТВЕРГ — 👥 Люди, маркетинг, процессы
| Время | Блок | Участники |
|-------|------|-----------|
| 10:00–10:30 | Утренний ритуал CEO: обзор HR-метрик (текучесть, воронка найма, укомплектованность) | CEO solo |
| 10:30–12:00 | Совещание HR/Подбор + Реклама/СММ (план найма, воронка, качество кандидатов, эффективность рекламных каналов, бюджет) | HR-дир., Рук. рекламы, СММ |
| 12:15–13:30 | DEEP WORK: стратегические проекты (продолжение проектов из среды, моделирование, подготовка решений) | CEO solo |
| 13:30–14:00 | 1-on-1 с COO: оптимизация процессов, SLA, обратная связь по РН, стандарты качества | COO |
| 15:00–17:00 | Стратегическая сессия по маркетингу (контент-план, бренд, реферальная программа) + Блок CEO-решений (согласования, эскалации) | Маркетинг + Развитие |
| 17:15–18:30 | AI-саморазвитие: разбор Zoom-записей совещаний с РН, итоговый анализ за неделю по РН | CEO solo + AI |
| 18:30–19:00 | Итог дня: 3 ключевых вывода, задачи на завтра, анализ ответов менеджеров на вопросы AI | CEO solo |

#### ПЯТНИЦА — 💰 Финансы, IT и рефлексия
| Время | Блок | Участники |
|-------|------|-----------|
| 10:00–10:30 | Утренний ритуал CEO: обзор финансовых показателей за неделю, подготовка к рефлексии | CEO solo |
| 10:30–12:00 | Финансовый блок (P&L, дебиторка, денежный поток, задолженности клиентов, налоги, compliance, аудит зарплатных проектов, сверки) | Главбух |
| 12:15–13:30 | 1-on-1 с IT-отделом (статус CRM, автоматизация процессов, безопасность данных, AI-инфраструктура) | IT-директор |
| 13:30–14:00 | AI-рефлексия недели: разбор Zoom-записей совещаний с РН через AI, выявление слабых мест | CEO solo + AI |
| 15:00–17:00 | Еженедельная рефлексия + планирование (анализ достигнутого, корректировка приоритетов, блокировка календаря, делегирование, 3 ключевые цели на следующую неделю) | CEO solo |
| 17:15–18:30 | Завершение текущих дел (ответы на отложенные сообщения, делегирование задач на выходные, подготовка к понедельнику) | CEO solo |
| 18:30–19:00 | Итог недели: запись главных достижений и зон роста в журнал CEO | CEO solo |

**Разделение блоков на две половины дня**:
- `morning_blocks` (10:00–14:00) — используются для обратной связи в 14:00
- `afternoon_blocks` (15:00–19:00) — используются для обратной связи в 20:30
- Утреннее информационное сообщение (09:00) показывает ВСЕ блоки дня целиком

В dataclass `DaySchedule` добавить свойства:
```python
@property
def morning_blocks(self) -> list[TimeBlock]:
    """Блоки до обеда (10:00–14:00)"""
    return [b for b in self.blocks if b.time < "14:00"]

@property
def afternoon_blocks(self) -> list[TimeBlock]:
    """Блоки после обеда (15:00–19:00)"""
    return [b for b in self.blocks if b.time >= "15:00"]
```

### 5.2 `src/planner/prompts.py`

Забрать из исходного `analysis_api.py`:
- Метод `_build_system_prompt()` — system prompt с моделями Адизеса, уровнями зрелости, форматом JSON-ответа. **Без изменений.**
- Метод `_build_user_message()` — формирование user message из payload. **Без изменений.**

### 5.3 `src/planner/claude_client.py`

**Написать заново.** Клиент для Anthropic Messages API.

Ключевые отличия от исходного `analysis_api.py`:
- URL: `https://api.anthropic.com/v1/messages`
- Заголовок: `x-api-key` (не `Authorization: Bearer`)
- Поле `system` — отдельно от `messages`
- Парсинг ответа: `response["content"][0]["text"]` (не `choices[0].message.content`)
- Retry с экспоненциальной задержкой (логику можно забрать из исходника)
- HTTP-клиент: `httpx.AsyncClient` или `anthropic.AsyncAnthropic` (Python SDK) — проект асинхронный

**Внимание**: проект **асинхронный** (async/await). Claude-клиент тоже должен быть асинхронным — использовать `httpx.AsyncClient` или `anthropic.AsyncAnthropic`.

### 5.4 `src/planner/formatter.py`

Забрать из исходного `message_formatter.py` полностью:
- `format_morning_message()` — утреннее сообщение с чек-листом
- `format_morning_info()` — повестка на весь день (БЕЗ кнопок)
- `format_feedback_request(half)` — запрос обратной связи по половине дня (С кнопками)
- `format_quality_question()`, `format_worked_question()`, и т.д. — вопросы FSM
- `format_analysis_message()` — финальное сообщение с анализом
- `format_end_of_day()` — сообщение об автозакрытии
- Все остальные методы

**Изменение**: Telegram Markdown может ломаться на спецсимволах. Рекомендуется перейти на `parse_mode='HTML'` (как в остальном CEO Agent) и заменить `*жирный*` на `<b>жирный</b>`.

### 5.5 `src/planner/planner_service.py`

**Написать заново.** Центральный класс модуля.

```python
class PlannerService:
    """Бизнес-логика планёра ГД — два цикла обратной связи в день"""

    async def send_morning_info(self):
        """Вызывается из APScheduler в 09:00 Пн–Пт
        Информационное сообщение с повесткой на ВЕСЬ день. БЕЗ кнопок."""
        # 1. Определить текущий день недели (timezone-aware! Europe/Moscow)
        # 2. Если выходной — выход
        # 3. Сформировать сообщение со ВСЕМИ блоками дня через formatter
        # 4. Отправить через telegram_bot (БЕЗ inline-кнопок)

    async def send_feedback_request(self, half: str):
        """Вызывается из APScheduler:
           half='morning' → 14:00 (блоки 10:00–14:00)
           half='afternoon' → 20:30 (блоки 15:00–19:00)
        Отправляет запрос обратной связи С кнопками."""
        # 1. Определить текущий день недели
        # 2. Если выходной — выход
        # 3. Сформировать сообщение с чек-листом блоков нужной половины
        # 4. Отправить с inline-кнопками (✅ Выполнено / ⏳ Частично / ❌ Не удалось)
        # 5. Создать запись в planner_state (step='awaiting_status', half=half)

    async def handle_status_callback(self, user_id: int, status: str):
        """Вызывается webhook-ботом при нажатии inline-кнопки"""
        # 1. Найти active state для данного user_id (по half из state)
        # 2. Обновить planner_state (meeting_status = status)
        # 3. Если held → step='awaiting_quality', отправить вопрос
        # 4. Если postponed/not_held → сохранить в planner_feedback (с half), удалить state

    async def handle_text_message(self, user_id: int, text: str):
        """Вызывается webhook-ботом при получении текста от ГД"""
        # 1. Прочитать planner_state (может быть morning или afternoon)
        # 2. В зависимости от current_step — обработать ответ
        # 3. Перейти к следующему шагу или завершить сбор
        # 4. При завершении — вызвать run_analysis()

    async def run_analysis(self, feedback_id: int):
        """Вызов Claude API и отправка результата"""
        # 1. Прочитать feedback из БД (включая half)
        # 2. Получить метрики из существующих таблиц (exits_wa, и т.д.)
        # 3. Сформировать payload (prompts.py) — с указанием какая половина дня
        # 4. Вызвать Claude API (claude_client.py)
        # 5. Сохранить лог в planner_analysis_logs
        # 6. Сформировать сообщение (formatter.py)
        # 7. Отправить через telegram_bot

    async def end_of_day(self):
        """Вызывается из APScheduler в 21:30"""
        # Для ВСЕХ незавершённых state (morning и afternoon):
        # → сохранить feedback с status='no_response', удалить state
```

### 5.6 Изменения в существующих файлах

**`src/orchestrator.py`** — добавить 4 scheduled jobs:
```python
# В методе start() после существующих jobs:
if settings.PLANNER_ENABLED:
    self.scheduler.add_job(
        planner_service.send_morning_info,
        CronTrigger(day_of_week='mon-fri', hour=9, minute=0, timezone=tz),
        id='planner_morning_info'
    )
    self.scheduler.add_job(
        lambda: planner_service.send_feedback_request('morning'),
        CronTrigger(day_of_week='mon-fri', hour=14, minute=0, timezone=tz),
        id='planner_feedback_morning'
    )
    self.scheduler.add_job(
        lambda: planner_service.send_feedback_request('afternoon'),
        CronTrigger(day_of_week='mon-fri', hour=20, minute=30, timezone=tz),
        id='planner_feedback_afternoon'
    )
    self.scheduler.add_job(
        planner_service.end_of_day,
        CronTrigger(day_of_week='mon-fri', hour=21, minute=30, timezone=tz),
        id='planner_eod'
    )
```

**`src/telegram/bot.py`** — добавить метод:
```python
async def send_planner_message(self, text: str, reply_markup=None, parse_mode='HTML'):
    """Отправка сообщения планёра в чат ГД"""
    await self.bot.send_message(
        chat_id=settings.TELEGRAM_CEO_CHAT_ID,
        text=text,
        parse_mode=parse_mode,
        reply_markup=reply_markup
    )
```

**`src/storage/models.py`** — добавить SQLAlchemy-модели:
- `PlannerFeedback`
- `PlannerAnalysisLog`
- `PlannerState`

(см. SQL-схему в разделе 3)

---

## 6. БАГИ В ИСХОДНИКАХ (НЕ ПЕРЕНОСИТЬ)

При адаптации **не копировать** следующие проблемы из исходных файлов:

| Баг | Файл | Что делать |
|-----|-------|-----------|
| `import asyncio` отсутствует в теле модуля, но используется `asyncio.sleep()` в retry | `analysis_api.py` | В новом клиенте импортировать корректно |
| `date.today()` без timezone — даёт серверное время, не московское | `bot.py` | Использовать `datetime.now(pytz.timezone('Europe/Moscow')).date()` |
| `/skip` не обрабатывается из-за фильтра `~filters.COMMAND` | `bot.py` | Обрабатывать слово «пропустить» как текст, или кнопку «Пропустить» |
| `datetime.utcnow()` deprecated в Python 3.12+ | `models.py` | Использовать `datetime.now(timezone.utc)` |
| При ошибке БД возвращаются фиктивные метрики без предупреждения | `database.py` | Бросать исключение или уведомлять ГД |
| `load_dotenv()` нигде не вызывается | — | В CEO Agent уже есть загрузка env через settings |
| Нет валидации пользовательского ввода перед отправкой в LLM | `bot.py` | Санитизировать текст (убирать потенциальные prompt injections) |

---

## 7. МЕТРИКИ ДЛЯ PAYLOAD

System prompt ожидает метрики: выходы, эффективность, воронка подбора, лиды Авито. Часть этих данных уже есть в БД (`exits_wa`), часть — нет.

**Что есть в MariaDB**:
- `exits_wa` → план/факт выходов по проектам и сменам (это `weekly_outputs` в payload)
- Из `exits_wa` можно рассчитать `operational_efficiency.avg_output_per_person`

**Чего нет в MariaDB** (нужно решить):
- `recruiting_funnel` (вакансии, заявки, собеседования, офферы) — либо добавить таблицу, либо ГД вводит вручную, либо убрать из промпта
- `demand_metrics` (лиды Авито, конверсия, прямые обращения) — аналогично

**Рекомендация для MVP**: убрать из промпта те метрики, которых нет в БД. Не подставлять фиктивные данные. Промпт адаптировать: «Анализируй только те метрики, которые предоставлены. Если данных нет — укажи это.»

---

## 8. ЦИКЛ ВЗАИМОДЕЙСТВИЯ (ДИАГРАММА)

```
09:00  APScheduler → PlannerService.send_morning_info()
              │
              ▼
       Telegram: повестка на ВЕСЬ день (БЕЗ кнопок, информационное)


14:00  APScheduler → PlannerService.send_feedback_request('morning')
              │
              ▼
       Telegram: «Как прошла первая половина дня?» + чек-лист блоков 10:00–14:00
              │     + 3 кнопки: ✅ Выполнено / ⏳ Частично / ❌ Не удалось
              │
              ▼ (ГД нажимает кнопку)
       Webhook → PlannerService.handle_status_callback()
              │
              ├─ «Частично» / «Не удалось» → save feedback(half='morning') → КОНЕЦ
              │
              └─ «Выполнено» → FSM: 5 вопросов
                      │
                      ▼ (ГД отвечает текстом, ×5)
               Webhook → PlannerService.handle_text_message()
                      │
                      ▼
               PlannerService.run_analysis()
                      │
                      └─ Claude API → диагностика + рекомендации → Telegram


20:30  APScheduler → PlannerService.send_feedback_request('afternoon')
              │
              ▼
       Telegram: «Как прошла вторая половина дня?» + чек-лист блоков 15:00–19:00
              │     + 3 кнопки
              │
              ▼ (аналогичный цикл FSM → Claude → результат)


21:30  APScheduler → PlannerService.end_of_day()
              │
              └─ Для ВСЕХ незавершённых state (morning/afternoon):
                 → save feedback(status='no_response') → delete state
```

---

## 9. ОТКРЫТЫЕ ВОПРОСЫ (решить в чате разработки)

1. ~~**Маршрутизация входящих сообщений**~~ → **РЕШЕНО:** `MessageRouter._process_private_message()` проверяет `ctx.mode == 'assist'` и роутит в PlannerService. Текст от ГД определяется по наличию активного `planner_state`.

2. ~~**Inline-кнопки**~~ → **РЕШЕНО:** `MessageRouter._handle_callback_query()` обрабатывает `callback_data.startswith("planner_")`. Webhook уже настроен с `allowed_updates` включающим `callback_query`.

3. ~~**Синхронный vs асинхронный код**~~ → **РЕШЕНО: проект асинхронный.** Использовать `async/await` везде, включая Claude API клиент и DB-операции.

4. **Метрики для промпта**: какие данные реально доступны в MariaDB помимо exits_wa? Есть ли таблицы по подбору, Авито, качеству?

5. **Пропуск вопросов**: вместо `/skip` (команда, которую бот может не увидеть) — использовать слово «пропустить» или inline-кнопку «Пропустить ➡️»?

6. **Одновременные FSM**: может ли ГД иметь два активных state (morning + afternoon)? Например, если не ответил на morning до 20:30. Рекомендация: при `send_feedback_request('afternoon')` — автозакрыть незавершённый morning state как `no_response`.

---

## 10. ПОРЯДОК РЕАЛИЗАЦИИ

### Этап 1: Фундамент (1–2 дня)
- [ ] Создать/обновить таблицы в MariaDB (раздел 3) — добавить поле `half` в `planner_feedback` и `planner_state`
- [ ] Добавить SQLAlchemy-модели в `models.py`
- [ ] Добавить конфиг в `chats.config` (раздел 4)
- [ ] Создать `src/planner/day_themes.py` с `morning_blocks` / `afternoon_blocks`
- [ ] Создать `src/planner/prompts.py` (копия из исходников)

### Этап 2: Claude API клиент (1 день)
- [ ] Создать `src/planner/claude_client.py` (AsyncAnthropic)
- [ ] Протестировать вызов с тестовым payload

### Этап 3: Бизнес-логика (2–3 дня)
- [ ] Создать `src/planner/planner_service.py` — 4 метода: `send_morning_info()`, `send_feedback_request(half)`, `handle_status_callback()`, `handle_text_message()`, `run_analysis()`, `end_of_day()`
- [ ] Создать `src/planner/formatter.py` — `format_morning_info()`, `format_feedback_request(half)`, и т.д.
- [ ] Добавить 4 scheduled jobs в orchestrator (09:00, 14:00, 20:30, 21:30)
- [ ] Добавить метод отправки в `bot.py`

### Этап 4: Интеграция с webhook-ботом (1–2 дня)
- [ ] Маршрутизация callback_query (уже реализована в `message_router.py`)
- [ ] Маршрутизация текстовых сообщений (уже реализована)
- [ ] Убедиться что webhook включает `callback_query` в `allowed_updates`
- [ ] Тестирование полного цикла

### Этап 5: Тестирование (1 день)
- [ ] Полный цикл: 09:00 повестка → 14:00 кнопки → FSM → анализ
- [ ] Второй цикл: 20:30 кнопки → FSM → анализ
- [ ] Сценарий «Частично» / «Не удалось»
- [ ] Автозакрытие 21:30 (morning + afternoon no_response)
- [ ] Ошибка Claude API (retry)

---

## ПРИЛОЖЕНИЕ: Файлы для передачи

Вместе с этой спецификацией передать следующие исходные файлы:

| Файл | Что из него забрать |
|------|-------------------|
| `config.py` | DAY_THEMES, EMOJI, DayOfWeek, MeetingStatus |
| `models.py` | Pydantic-модели: AnalysisPayload, AnalysisResponse, MeetingFeedback, Action, RiskForecast |
| `analysis_api.py` | System prompt (_build_system_prompt), user message (_build_user_message) |
| `message_formatter.py` | Все методы форматирования сообщений |
| `tz_corrected.md` | Полное ТЗ для понимания бизнес-контекста |

**НЕ передавать** (бесполезны): `bot.py`, `redis_store.py`, `init_db.py`, `database.py`, `requirements.txt`, `.env.example`, `.gitignore`
