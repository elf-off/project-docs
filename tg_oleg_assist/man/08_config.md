# Конфигурация

## Два уровня настроек

Конфигурация разделена на два уровня:

1. **Глобальные настройки** — в `.env` файле, применяются ко всему боту
2. **Per-chat настройки** — в БД в полях `chats.features` (JSON) и `chats.config` (JSON), применяются к конкретному чату

---

## Глобальные настройки (.env)

`app/config.py` — `pydantic-settings` класс, `extra="ignore"` (старые поля из .env не вызывают ошибку).

### Telegram

| Параметр | Описание |
|---|---|
| `TELEGRAM_BOT_TOKEN` | Токен бота от @BotFather |
| `WEBHOOK_URL` | Публичный URL для webhook |
| `WEBHOOK_SECRET` | Секрет для верификации запросов (`X-Telegram-Bot-Api-Secret-Token`) |

### База данных

| Параметр | По умолчанию | Описание |
|---|---|---|
| `DB_HOST` | localhost | Хост MariaDB/MySQL |
| `DB_PORT` | 3306 | Порт |
| `DB_NAME` | ethics_bot | Имя основной БД |
| `DB_USER` | — | Пользователь |
| `DB_PASSWORD` | — | Пароль |

Строка подключения: `mysql+aiomysql://user:password@host:port/dbname`

### Anthropic Claude

| Параметр | По умолчанию | Описание |
|---|---|---|
| `ANTHROPIC_API_KEY` | — | API-ключ |
| `ANTHROPIC_MODEL` | claude-sonnet-4-20250514 | Модель |
| `ANTHROPIC_TIMEOUT` | 30 | Таймаут (сек) |
| `ANTHROPIC_BASE_URL` | None | Переопределение base URL (прокси) |

### Bitrix24

| Параметр | По умолчанию | Описание |
|---|---|---|
| `BITRIX_WEBHOOK_URL` | "" | URL REST webhook |
| `BITRIX_ASSIGNED_BY_ID` | 565282 | ID ответственного в Bitrix24 |

### Scoring (глобальные)

| Параметр | По умолчанию | Описание |
|---|---|---|
| `AGGREGATION_WINDOW_MINUTES` | 30 | Окно накопления баллов |
| `LONG_TERM_WINDOW_DAYS` | 7 | Долгосрочное окно |
| `DEFAULT_TASK_DEADLINE_MINUTES` | 60 | Дедлайн задач по умолчанию |

### Growth Plan-Fact

| Параметр | По умолчанию | Описание |
|---|---|---|
| `GROWTH_REPORT_CHAT_ID` | 0 | `tg_chat_id` чата для отчётов (0 = выключено) |
| `GROWTH_CHECK_HOUR` | 10 | Час проверки (МСК) |
| `GROWTH_CHECK_MINUTE` | 0 | Минута проверки |

### Планёр ГД (глобальные дефолты)

Основная конфигурация планёра хранится в `chats.config`, но `.env` задаёт дефолты.

| Параметр | По умолчанию |
|---|---|
| `PLANNER_ENABLED` | false |
| `PLANNER_TIMEZONE` | Europe/Moscow |
| `PLANNER_MORNING_INFO_HOUR` | 9 |
| `PLANNER_MORNING_INFO_MINUTE` | 0 |
| `PLANNER_FEEDBACK_MORNING_HOUR` | 14 |
| `PLANNER_FEEDBACK_MORNING_MINUTE` | 0 |
| `PLANNER_FEEDBACK_AFTERNOON_HOUR` | 20 |
| `PLANNER_FEEDBACK_AFTERNOON_MINUTE` | 30 |
| `PLANNER_EOD_HOUR` | 21 |
| `PLANNER_EOD_MINUTE` | 30 |

---

## Per-chat настройки (chats.features и chats.config)

Хранятся в таблице `chats` как JSON-поля. Изменяются напрямую в БД или через admin API.

### chats.features — включение функций

```json
{
  "ethics_analysis": true,
  "mention_tracking": true,
  "report_reminders": true,
  "weekly_planning": true,
  "planner": true,
  "rop_reformat": true
}
```

| Флаг | Описание |
|---|---|
| `ethics_analysis` | Анализ этики сообщений через Claude |
| `mention_tracking` | Трекинг упоминаний и ответов |
| `report_reminders` | Контроль сдачи отчётов |
| `weekly_planning` | Сбор еженедельных планов менеджеров |
| `planner` | Дневной планёр ГД (только для mode=assist) |
| `rop_reformat` | Переформатирование РОП-отчётов по лидам |

### chats.config — параметры

```json
{
  "mention_deadline_minutes": 15,
  "mention_notify_private": true,
  "mention_notify_chat": true,
  "mention_detect_username": true,
  "mention_detect_reply": true,
  "workdays_only": true,
  
  "report_check_times": [
    {"hour": 10, "minute": 45},
    {"hour": 14, "minute": 15},
    {"hour": 19, "minute": 15}
  ],
  "report_notify_private": true,
  "report_notify_chat": true,
  
  "planning_deadline_day": 4,
  "planning_deadline_hour": 20,
  "planning_deadline_minute": 0,
  "planning_timezone": "Europe/Moscow",
  
  "planner_ceo_user_id": 123456789,
  "planner_timezone": "Europe/Moscow",
  "planner_morning_info_hour": 9,
  "planner_morning_info_minute": 0,
  "planner_feedback_morning_hour": 14,
  "planner_feedback_morning_minute": 0,
  "planner_feedback_afternoon_hour": 20,
  "planner_feedback_afternoon_minute": 30,
  "planner_eod_hour": 21,
  "planner_eod_minute": 30
}
```

| Параметр | Для чего |
|---|---|
| `mention_deadline_minutes` | Через сколько минут упоминание считается просроченным |
| `mention_notify_private` | Слать напоминание в личку упомянутому |
| `mention_notify_chat` | Слать напоминание в чат |
| `workdays_only` | Трекинг только в рабочие дни |
| `report_check_times` | Список времён проверки отчётов `[{hour, minute}]` |
| `planner_ceo_user_id` | `tg_id` гендиректора (только он может отвечать в планёре) |
| `planning_deadline_day` | День дедлайна планов (0=Пн, 4=Пт) |

---

## Получение настроек в коде

```python
# Глобальные
settings = get_settings()  # lru_cache, читается из .env один раз
bot_token = settings.telegram_bot_token

# Per-chat
ctx = await resolver.resolve(tg_chat_id)
ctx.has_feature('ethics_analysis')         # bool
ctx.get_config('mention_deadline_minutes', 15)  # значение или дефолт
ctx.get_report_check_times()               # [(h, m), ...]
```
