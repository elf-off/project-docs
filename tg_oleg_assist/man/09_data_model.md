# Модели данных

Схема находится в `app/models/`. Миграции — в `migrations/` (raw SQL, 001–007, не Alembic). БД: MariaDB/MySQL, драйвер `aiomysql`.

---

## Основные таблицы

### chats — чаты

| Поле | Тип | Описание |
|---|---|---|
| id | int PK | Внутренний ID |
| tg_chat_id | bigint UNIQUE | Telegram chat ID |
| title | varchar | Название чата |
| mode | varchar | Режим: viewer / compliance / planning / assist |
| is_active | bool | Бот активен в чате |
| features | JSON | Feature-флаги `{"ethics_analysis": true, ...}` |
| config | JSON | Параметры `{"mention_deadline_minutes": 15, ...}` |
| org_unit_id | int FK | Оргподразделение (для арбитража) |

### users — пользователи

| Поле | Тип | Описание |
|---|---|---|
| id | int PK | Внутренний ID |
| tg_id | bigint UNIQUE | Telegram user ID |
| username | varchar | @username (может быть NULL) |
| full_name | varchar | Полное имя |
| role | enum | manager / deputy_director / director / hr_compliance |
| is_active | bool | Активен ли пользователь |
| curator_id | int FK → users | Куратор (для менеджеров, используется в арбитраже) |
| org_unit_id | int FK | Оргподразделение |

Пользователи **не создаются автоматически** при сообщениях — они должны быть заведены администратором заранее. При получении сообщения бот ищет пользователя по `tg_id`; если не найден или `is_active=False` — сообщение игнорируется.

### messages — сообщения

| Поле | Тип | Описание |
|---|---|---|
| id | int PK | |
| tg_message_id | bigint | ID сообщения в Telegram |
| chat_id | int FK → chats | |
| user_id | int FK → users | |
| text | text | Текст сообщения |
| reply_to_message_id | bigint | На какое сообщение ответ (tg_message_id) |

### scoring_events — события нарушений

| Поле | Тип | Описание |
|---|---|---|
| id | int PK | |
| message_id | int FK → messages | |
| chat_id | int FK → chats | |
| user_id | int FK → users | |
| rule_id | varchar | S1–S7 |
| category | varchar | blaming / aggression / ... |
| weight | int | Вес нарушения |
| signal_type | varchar | explicit / implicit / non_textual |
| matched_signal | varchar | Фрагмент текста |
| ai_confidence | float | Уверенность модели |
| correlation_id | varchar | UUID запроса |
| created_at | datetime | Время события (используется для скользящего окна) |

### dialog_states — текущее состояние диалога

Одна запись на чат.

| Поле | Тип | Описание |
|---|---|---|
| id | int PK | |
| chat_id | int FK → chats UNIQUE | |
| state | enum | NORMAL / TENSION / CONFLICT / ESCALATION |
| previous_state | enum | Предыдущее состояние |
| current_score | int | Текущий накопленный балл |
| window_start | datetime | Начало текущего окна |
| triggered_by_condition | varchar | ID условия (C1–C6), вызвавшего переход |

### state_transitions — история переходов

| Поле | Тип | Описание |
|---|---|---|
| id | int PK | |
| chat_id | int FK → chats | |
| from_state | enum | |
| to_state | enum | |
| trigger_condition | varchar | C1–C6 или NULL |
| trigger_score | int | Балл на момент перехода |
| trigger_message_id | int FK → messages | |
| correlation_id | varchar | |
| created_at | datetime | |

### ethics_events — результаты анализа

| Поле | Тип | Описание |
|---|---|---|
| id | int PK | |
| message_id | int FK → messages | |
| risk_level | varchar | low / medium / high |
| total_score | int | Суммарный вес сигналов |
| ai_reasoning | text | Объяснение от Claude |
| triggered_rules | JSON | Список ID правил [S1, S3] |
| correlation_id | varchar | |

### mentions — упоминания

| Поле | Тип | Описание |
|---|---|---|
| id | int PK | |
| message_id | int FK → messages | |
| chat_id | int FK → chats | |
| author_id | int FK → users | Кто упомянул |
| target_user_id | int FK → users | Кого упомянули |
| mention_type | enum | username / reply |
| status | enum | WAITING / ANSWERED / OVERDUE |
| deadline_at | datetime | Дедлайн ответа |
| answered_at | datetime | Когда ответил |
| answered_message_id | int | Какое сообщение было ответом |
| notified_private | bool | Уже отправлено личное напоминание |
| notified_chat | bool | Уже отправлено публичное напоминание |
| correlation_id | varchar | |

### report_types — типы отчётов

| Поле | Тип | Описание |
|---|---|---|
| id | int PK | |
| code | varchar | Код типа (например, `rop`) |
| name | varchar | Название |
| keywords | JSON | Ключевые слова для детекции в первой строке |
| is_active | bool | |

### report_responsible — ответственные за отчёты

| Поле | Тип | Описание |
|---|---|---|
| id | int PK | |
| report_type_id | int FK → report_types | |
| user_id | int FK → users | |
| chat_id | int FK → chats | |
| is_active | bool | |

### daily_reports — журнал отчётов

| Поле | Тип | Описание |
|---|---|---|
| id | int PK | |
| report_type_id | int FK → report_types | |
| user_id | int FK → users | Кто сдал |
| chat_id | int FK → chats | |
| report_date | date | |
| check_time | time | К какому временному слоту относится |
| status | enum | SUBMITTED / REMINDED |
| submitted_at | datetime | |
| reminded_at | datetime | Когда напомнили |
| reminder_count | int | Сколько раз напоминали |
| message_id | int FK → messages | |

### planning_managers — менеджеры-плановики

| Поле | Тип | Описание |
|---|---|---|
| id | int PK | |
| chat_id | int FK → chats | |
| tg_user_id | bigint | |
| full_name | varchar | Обновляется при каждой отправке плана |
| username | varchar | |
| is_active | bool | |

В отличие от `users`, менеджеры-плановики **создаются автоматически** при первом сообщении `#Планирование`.

### planning_submissions — поданные планы

| Поле | Тип | Описание |
|---|---|---|
| id | int PK | |
| chat_id | int FK → chats | |
| manager_id | int FK → planning_managers | |
| tg_message_id | bigint | |
| plan_year | int | ISO год |
| plan_week | int | ISO номер недели |
| submitted_at | datetime | |
| is_accepted | bool | Подан до дедлайна |
| fact_checked_at | datetime | Когда проверили план-факт |
| fact_report_sent | bool | Включён ли в отчёт |

### planning_entries — строки плана

| Поле | Тип | Описание |
|---|---|---|
| id | int PK | |
| submission_id | int FK → planning_submissions | |
| project_id | int | ID из `2_kadry_4.ref_projects_wa` (NULL для группы) |
| group_id | int | ID из `2_kadry_4.ref_project_groups` (NULL для проекта) |
| change_percent | decimal | Целевое изменение в % |
| baseline_value | int | Факт прошлой недели (при подаче) |
| fact_value | int | Реальный факт (заполняется в понедельник) |
| fact_growth_percent | decimal | Фактический рост % |
| plan_status | varchar | achieved / partial / missed / no_data |

### planner_states — текущее FSM-состояние планёра ГД

| Поле | Тип | Описание |
|---|---|---|
| id | int PK | |
| chat_id | int FK → chats | |
| user_id | bigint | tg_id пользователя |
| half | varchar | morning / afternoon |
| current_step | varchar | AWAITING_STATUS / AWAITING_QUALITY / ... |
| date | date | За какой день |
| day_context | varchar | Название дня недели |
| meeting_status | varchar | completed / partial / failed |
| quality_score | int | Оценка 1–10 |
| what_worked | text | |
| what_to_add | text | |
| main_problem | text | |
| decision_taken | text | |
| expires_at | datetime | До 21:30 текущего дня |

UNIQUE KEY: `(user_id, half)`

### planner_feedbacks — завершённые сессии планёра

Аналог `planner_states`, но постоянное хранение. Создаётся по завершении FSM или при автозакрытии.

### planner_analysis_logs — логи вызовов Claude для планёра

| Поле | Тип | Описание |
|---|---|---|
| id | int PK | |
| feedback_id | int FK → planner_feedbacks | |
| api_request | JSON | Превью промпта |
| api_response | JSON | Полный ответ Claude |
| latency_ms | int | Задержка ответа |
| model | varchar | Название модели |

### bitrix_leads — журнал созданных лидов

| Поле | Тип | Описание |
|---|---|---|
| id | int PK | |
| tg_user_id | bigint | Кто создал |
| first_name / last_name / middle_name | varchar | |
| phone | varchar | Нормализованный номер |
| comments | text | Город + доп. инфо |
| raw_text | text | Исходный текст сообщения |
| bitrix_lead_id | int | ID лида в Bitrix24 (NULL при ошибке) |
| success | bool | |
| error_message | varchar | |
| correlation_id | varchar | |

### audit_log — журнал действий

Сквозная таблица для аудита значимых событий (анализ этики, переходы состояний и т.д.).

| Поле | Тип | Описание |
|---|---|---|
| id | int PK | |
| event_type | varchar | Например, `ethics_analyzed` |
| actor_id | int FK → users | Кто инициировал |
| target_type | varchar | Тип объекта (`message`, `chat`) |
| target_id | int | ID объекта |
| details | JSON | Дополнительные данные |
| correlation_id | varchar | |
| created_at | datetime | |

---

## Внешняя БД (2_kadry_4)

Бот обращается к таблицам в отдельной схеме на том же сервере:

| Таблица | Используется для |
|---|---|
| `ref_projects_wa` | Справочник проектов для планирования |
| `ref_project_groups` | Справочник групп проектов |
| `project_group_members` | Связь проектов с группами |
| `exits_wa` | Данные о выходах (план/факт) — основа для план-факта |
| `exits_wa_stajers` | Данные о стажёрах — для РОП-отчётов |
