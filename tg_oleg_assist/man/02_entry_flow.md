# Поток входящего сообщения

## Запуск приложения

`app/main.py` создаёт FastAPI-приложение. В lifespan-контексте запускается `TimerWorker` как фоновая `asyncio.Task`. Регистрируются два роутера: `/tg` (webhook) и `/admin`.

Эндпоинты самого приложения:

- `POST /tg/webhook` — приём обновлений от Telegram
- `GET /health` — healthcheck
- `GET /timer/status` — статус фонового воркера

## Шаг 1: Webhook

`POST /tg/webhook` (`app/api/webhook.py`):

1. Генерируется `correlation_id` (UUID) — уникальный идентификатор запроса, который проходит через все сервисы и логируется везде.
2. Проверяется заголовок `X-Telegram-Bot-Api-Secret-Token`. Если токен не совпадает — HTTP 403.
3. Парсится JSON-тело запроса, извлекается `update_id`.
4. **Idempotency-проверка**: `idempotency_guard.try_acquire(update_id)` — если `update_id` уже встречался, возвращается `{"ok": true}` без обработки. Telegram повторяет запросы при медленных ответах, и без этой защиты одно сообщение могло бы обработаться дважды.
5. Вызывается `MessageRouter(cid).process(payload)`.
6. В `finally`-блоке: `idempotency_guard.mark_done(update_id)` — отметить как обработанный.
7. Всегда возвращается `{"ok": true}` — даже при внутренних ошибках, чтобы Telegram не повторял нефатальные сбои.

## Шаг 2: IdempotencyGuard

`app/services/idempotency.py` — синглтон, живущий весь срок работы процесса.

Внутри — словарь `update_id -> (timestamp, status)` с TTL 10 минут. Защита thread-safe через `threading.Lock` (uvicorn может использовать несколько потоков). Очистка устаревших записей происходит лениво — только когда кэш достигает половины от `max_size` (10 000).

Два статуса записи:
- `processing` — обновление сейчас обрабатывается (параллельный retry будет отклонён)
- `done` — обработка завершена (поздний retry тоже будет отклонён, пока не истёк TTL)

## Шаг 3: MessageRouter

`app/services/message_router.py` — центральный диспетчер. Определяет тип апдейта и вызывает нужный обработчик.

### Авторегистрация чата (my_chat_member)

Когда бот добавляется в группу или удаляется из неё, Telegram присылает событие `my_chat_member`. MessageRouter обрабатывает его отдельно:

- статус `member` / `administrator` → `ChatContextResolver.register_chat(...)` создаёт запись в `chats` с `mode='viewer'`
- статус `left` / `kicked` → `ChatContextResolver.deactivate_chat(...)` ставит `is_active=False`

Приватные чаты и каналы игнорируются.

### Callback query (нажатие inline-кнопки)

Немедленно вызывается `notifier.answer_callback_query(callback_query_id)` — это убирает "часики" на кнопке в интерфейсе Telegram. Затем, если `callback_data` начинается с `planner_` и чат в режиме `assist`, вызывается `PlannerService.handle_callback()`.

### Личное сообщение (private chat)

Два варианта:
1. Текст начинается с `#Битрикс` (первая строка) → `BitrixService.process_message()` — создаёт лид в CRM.
2. Иначе → `_process_private_message()` → если чат в режиме `assist` → `PlannerService.handle_text_message()`.

### Групповое сообщение

Только текстовые сообщения из `group` / `supergroup`. Редактированные сообщения (`edited_message`) тоже обрабатываются.

```
1. Resolve ChatContext по tg_chat_id
2. Если чат не найден в БД — игнорировать
3. Если mode == 'viewer' — игнорировать
4. Если mode == 'planning' — PlanningService (не нужен user из users)
5. Найти user по tg_id в таблице users
   - если не найден или is_active=False — игнорировать
6. Если роль не manager / deputy_director / director — игнорировать
7. Сохранить сообщение в таблицу messages
8. Dispatch по mode:
   - 'compliance' → _process_compliance()
   - 'operator'   → заглушка
   - 'evaluation' → заглушка
```

## Шаг 4: Resolve ChatContext

`ChatContextResolver.resolve(tg_chat_id)` — ищет чат в in-memory кэше (словарь в рамках текущего запроса), при промахе делает запрос к БД. Возвращает объект `ChatContext`, инкапсулирующий:

- `mode` — режим работы
- `features` — JSON с feature-флагами (`{"ethics_analysis": true, "mention_tracking": true, ...}`)
- `config` — JSON с параметрами (`{"mention_deadline_minutes": 15, "report_check_times": [...]}`)
- `tg_chat_id`, `chat_id` (внутренний ID в БД), `title`, `org_unit_id`

Метод `ctx.has_feature('feature_name')` — проверяет feature-флаг. Метод `ctx.get_config('key', default)` — читает параметр с дефолтом.

> Кэш живёт только в рамках одного HTTP-запроса (объект `ChatContextResolver` создаётся заново в каждом `_process_message`). Это означает, что изменения в БД подхватываются немедленно, но и SQL-запрос к `chats` делается при каждом новом запросе.
