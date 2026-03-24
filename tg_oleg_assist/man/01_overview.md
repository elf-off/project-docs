# Общее описание проекта

## Что это

**"Олег"** — Telegram-бот для мониторинга культуры корпоративной коммуникации. Он работает в нескольких рабочих группах и в личных чатах, выполняя разные задачи в зависимости от настроенного режима чата.

Основные функции:

- Анализ сообщений на нарушения этики (агрессия, манипуляции, обвинения) через Claude AI
- Автоматические предупреждения и временные блокировки нарушителей
- Отслеживание упоминаний сотрудников и дедлайнов ответа
- Контроль сдачи регулярных отчётов
- Дневной планёр для генерального директора
- Сбор и анализ еженедельных планов от руководителей
- Создание лидов в Bitrix24 CRM из личных сообщений

## Технологический стек

| Компонент | Технология |
|---|---|
| Web-фреймворк | FastAPI (webhook mode, async) |
| ORM / БД-клиент | SQLAlchemy async + aiomysql |
| База данных | MariaDB / MySQL |
| AI-модель | Anthropic Claude (claude-sonnet-4) |
| Telegram SDK | python-telegram-bot |
| HTTP-клиент | httpx (для Bitrix24) |
| Логирование | structlog с correlation_id |
| Конфигурация | pydantic-settings (.env) |
| Деплой | systemd-сервис (`k24-oleg-bot.service`) |

## Режимы работы чата

Каждый чат в БД имеет поле `mode`, которое определяет, что бот делает с сообщениями:

| Режим | Где | Что делает |
|---|---|---|
| `viewer` | Группа | Ничего (режим по умолчанию при добавлении бота) |
| `compliance` | Группа | Анализ этики + трекинг упоминаний + контроль отчётов |
| `planning` | Группа | Сбор еженедельных планов от менеджеров |
| `assist` | Личный чат | Дневной планёр для ГД |
| `operator` | Группа | Заглушка (Фаза 2, не реализована) |
| `evaluation` | Группа | Заглушка (Фаза 3, не реализована) |

## Высокоуровневая архитектура

```
Telegram
    |
    v POST /tg/webhook
[webhook.py]
    | проверка секрета, correlation_id, idempotency
    v
[MessageRouter]
    |
    +-- my_chat_member -----> авторегистрация / деактивация чата
    |
    +-- callback_query -----> PlannerService.handle_callback()
    |
    +-- private message
    |       +-- #Битрикс --> BitrixService
    |       +-- assist    --> PlannerService.handle_text_message()
    |
    +-- group message
            +-- viewer    --> игнор
            +-- planning  --> PlanningService
            +-- compliance
                    +-- ethics_analysis  --> EthicsAnalyzer
                    |                        --> ScoringEngine (FSM)
                    |                        --> ArbitrationService
                    |                        --> NotificationService
                    +-- mention_tracking --> MentionService
                    +-- report_reminders --> ReportService

[TimerWorker] (фон, каждые 30 сек)
    +-- mention_tracking  --> просроченные упоминания
    +-- report_reminders  --> напоминания об отчётах
    +-- planner           --> утреннее сообщение, обратная связь, закрытие дня
    +-- growth_report     --> план-факт по понедельникам
    +-- tasks             --> дедлайны задач (глобально)
```

## Структура директорий

```
app/
  api/
    webhook.py          # POST /tg/webhook
    admin.py            # Служебные эндпоинты
  services/
    message_router.py   # Центральный роутер
    chat_context.py     # ChatContext + ChatContextResolver
    idempotency.py      # Защита от повторной обработки
    ethics_analyzer.py  # Анализ этики (AI + fallback)
    scoring_engine.py   # FSM диалога, накопление баллов
    arbitration.py      # Выбор арбитра
    notification.py     # Отправка сообщений через Telegram Bot API
    mention_service.py  # Трекинг упоминаний
    report_service.py   # Трекинг отчётов + реформат РОП
    planning_service.py # Еженедельные планы менеджеров
    bitrix_service.py   # Интеграция с Bitrix24
    timer_worker.py     # Фоновый воркер
    rop_validator.py    # Валидация РОП-отчётов
    planner/
      planner_service.py      # Дневной планёр ГД
      planner_formatter.py    # Форматирование сообщений
      planner_prompts.py      # Промпты для Claude
      planner_day_themes.py   # Темы дней недели, FSM-шаги
  rules/
    rules.yaml          # Правила S1-S7 (нарушения этики)
    conflict_rules.yaml # Условия C1-C6 (смены состояния диалога)
    loader.py           # Загрузка правил (кэшируемый singleton)
  prompts/
    signal_detector.py  # Системный промпт для анализа этики
    instruction.py      # Другие промпты
  models/               # SQLAlchemy-модели
  config.py             # Настройки через pydantic-settings
  database.py           # Подключение к БД, get_db_session()
  main.py               # Точка входа FastAPI
migrations/             # SQL-миграции (001-007, не Alembic)
scripts/                # Вспомогательные скрипты
tests/                  # Тесты (plain Python, без pytest)
```
