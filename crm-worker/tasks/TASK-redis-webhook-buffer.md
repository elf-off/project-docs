# TASK: Redis Streams буфер для входящих вебхуков

## Проблема

Сейчас вебхуки от Avito обрабатываются синхронно в `api/webhooks.py` — endpoint принимает запрос, выполняет всю бизнес-логику (БД, AI, отправка ответа), и только потом отвечает 200 OK. Avito требует ответ в пределах 2 секунд. При перезапуске сервиса входящие вебхуки теряются.

## Решение

Разделить приём и обработку:
1. **Endpoint** — принимает webhook, кладёт сырой payload в Redis Stream, мгновенно отвечает `{"ok": true}` (< 5ms)
2. **Worker** — отдельная asyncio-задача, читает из Redis через consumer group, выполняет текущую бизнес-логику

Два раздельных стрима:
- `webhooks:messenger` — сообщения из чатов (от `/messenger/v3/webhook`)
- `webhooks:applications` — отклики на вакансии (от `/job/v1/applications/webhook`)

## Структура проекта (что менять / создавать)

### НОВЫЕ ФАЙЛЫ

#### 1. `services/redis_queue.py`

Обёртка над Redis Streams. Содержит:

```
Константы:
  STREAM_MESSENGER = "webhooks:messenger"
  STREAM_APPLICATIONS = "webhooks:applications"
  CONSUMER_GROUP = "workers"

Функции:
  get_redis() -> redis.Redis
    - Ленивое создание пула (redis.asyncio)
    - Параметры из config: redis_host, redis_port, redis_db, redis_password
    - decode_responses=True, socket_connect_timeout=5, retry_on_timeout=True

  close_redis()
    - Закрытие пула, вызывается при shutdown

  ensure_consumer_groups()
    - xgroup_create для обоих стримов с mkstream=True
    - Если BUSYGROUP — молча пропускаем (группа уже есть)

  enqueue_webhook(stream: str, payload: dict, source_ip: str = "") -> str
    - XADD в нужный stream
    - Поля: data=json.dumps(payload), source_ip=source_ip, enqueued_at=time.time()
    - Возвращает message_id
    - При ошибке Redis — логировать и пробросить (пусть endpoint вернёт 500, Avito повторит)

  consume_webhooks(stream: str, consumer_name: str, count: int = 10, block_ms: int = 5000)
    - XREADGROUP groupname=CONSUMER_GROUP, consumername=consumer_name
    - Возвращает список (message_id, payload_dict) — payload уже распарсен из JSON
    - При пустом результате — возвращает пустой список

  ack_webhook(stream: str, message_id: str)
    - XACK stream CONSUMER_GROUP message_id

  recover_pending(stream: str, consumer_name: str, idle_ms: int = 300000)
    - XAUTOCLAIM / XPENDING + XCLAIM для сообщений зависших > idle_ms (5 мин)
    - Это для случаев когда воркер упал не сделав ACK
    - Возвращает список (message_id, payload_dict)

  get_stream_info(stream: str) -> dict
    - XINFO STREAM — для мониторинга в админке
    - Возвращает: length, first_entry_id, last_entry_id, groups (кол-во)
    - При ошибке — возвращает {"length": 0, "error": str(e)}
```

Зависимость: `redis[hiredis]>=5.0.0` — добавить в requirements.txt

#### 2. `workers/webhook_consumer.py`

Воркер, который читает из стримов и вызывает существующую бизнес-логику.

```
class WebhookConsumer:
    def __init__(self, consumer_name: str = "worker-1"):
        self.consumer_name = consumer_name
        self.running = False

    async def start(self):
        """Запускает два параллельных цикла (messenger + applications)."""
        self.running = True
        await asyncio.gather(
            self._consume_loop(STREAM_MESSENGER, self._process_messenger),
            self._consume_loop(STREAM_APPLICATIONS, self._process_application),
            self._recovery_loop(),  # раз в 5 мин проверяет pending
        )

    async def stop(self):
        self.running = False

    async def _consume_loop(self, stream, handler):
        """
        Бесконечный цикл: consume_webhooks → для каждого сообщения вызвать handler → ack.
        При ошибке в handler — НЕ делать ack (сообщение уйдёт в pending).
        Логировать каждый шаг через structlog.
        Между итерациями — await asyncio.sleep(0.1) чтобы не жрать CPU.
        """

    async def _process_messenger(self, payload: dict):
        """
        Повторяет текущую логику из webhooks.py для messenger-вебхуков.
        
        Это ТА ЖЕ логика что сейчас в handle_avito_webhook() для типа "message":
        - Достать value.author_id, value.chat_id и т.д.
        - Фильтрация author_id == 0 и author_id в наших user_ids
        - Вызов incoming_processor.process_incoming_message(...)
        
        НЕ дублировать код — вынести общую логику в отдельную функцию 
        в webhooks.py или incoming_processor.py если нужно, и вызывать её.
        """

    async def _process_application(self, payload: dict):
        """
        Повторяет текущую логику для application-вебхуков.
        
        Это ТА ЖЕ логика что сейчас в handle_avito_webhook() для типа "application":
        - resolve account
        - get_application_details
        - create records
        - schedule greeting
        
        Аналогично — НЕ дублировать, а вынести и переиспользовать.
        """

    async def _recovery_loop(self):
        """Раз в 5 минут вызывает recover_pending для обоих стримов."""
```

### ИЗМЕНИТЬ СУЩЕСТВУЮЩИЕ ФАЙЛЫ

#### 3. `config.py`

Добавить настройки Redis:
```python
redis_host: str = os.getenv("REDIS_HOST", "127.0.0.1")
redis_port: int = int(os.getenv("REDIS_PORT", "6379"))
redis_db: int = int(os.getenv("REDIS_DB", "0"))
redis_password: str = os.getenv("REDIS_PASSWORD", "")
```

#### 4. `api/webhooks.py`

Изменить `handle_avito_webhook()`:

**БЫЛО:** Принимает payload → определяет тип → выполняет всю обработку → return 200

**СТАЛО:** Принимает payload → определяет тип (messenger или application) → enqueue_webhook в нужный стрим → return `{"ok": true}`

```
Логика определения типа (оставить в endpoint):
  - Если есть payload.type == "message" или payload.value.chat_id → STREAM_MESSENGER
  - Если есть payload.applyId или payload.type == "application" → STREAM_APPLICATIONS
  - Если не определено — залогировать warning, всё равно положить в STREAM_MESSENGER (fallback)

Endpoint должен быть максимально тонким:
  1. body = await request.json()
  2. определить stream
  3. await enqueue_webhook(stream, body, source_ip=request.client.host)
  4. return {"ok": true}

Обернуть в try/except:
  - Если Redis недоступен — FALLBACK: обработать синхронно как раньше (вызвать старую логику)
  - Залогировать error "redis_unavailable, falling_back_to_sync"
  - Это критично — нельзя терять вебхуки если Redis упал
```

**Важно:** Существующую логику обработки (фильтрация, resolve account, process_incoming_message и т.д.) — вынести в отдельные функции/методы, которые будет вызывать webhook_consumer.py. Не удалять — перенести.

#### 5. `main.py`

В startup event:
```
- await ensure_consumer_groups()
- Создать экземпляр WebhookConsumer
- Запустить consumer.start() как asyncio.Task (фоновая задача)
```

В shutdown event:
```
- await consumer.stop()
- await close_redis()
```

Пример:
```python
webhook_consumer = None

@app.on_event("startup")
async def startup():
    # ... существующий код ...
    from services.redis_queue import ensure_consumer_groups, close_redis
    from workers.webhook_consumer import WebhookConsumer
    
    await ensure_consumer_groups()
    
    global webhook_consumer
    webhook_consumer = WebhookConsumer(consumer_name="main-worker")
    asyncio.create_task(webhook_consumer.start())

@app.on_event("shutdown") 
async def shutdown():
    if webhook_consumer:
        await webhook_consumer.stop()
    await close_redis()
```

#### 6. `api/admin.py` (опционально, но желательно)

Добавить endpoint для мониторинга очередей:
```
GET /admin/api/queues
  - Вызывает get_stream_info для обоих стримов
  - Возвращает: { "messenger": {...}, "applications": {...} }
```

Это нужно чтобы в админке видеть, не копятся ли необработанные сообщения.

#### 7. `requirements.txt`

Добавить строку:
```
redis[hiredis]>=5.0.0
```

### НОВЫЙ ФАЙЛ

#### 8. `migrations/005_redis_setup.md`

Это не SQL-миграция, а инструкция по установке Redis:
```markdown
# Redis setup

## Установка
sudo apt update && sudo apt install redis-server -y

## Настройка
sudo systemctl enable redis-server
sudo systemctl start redis-server

## Проверка
redis-cli ping
# Ответ: PONG

## Если нужен пароль (опционально)
# В /etc/redis/redis.conf раскомментировать:
# requirepass your_password
# И добавить REDIS_PASSWORD=your_password в .env

## Проверка стримов (после запуска сервиса)
redis-cli XINFO STREAM webhooks:messenger
redis-cli XINFO STREAM webhooks:applications
```

---

## Логирование

Все операции с Redis логировать через structlog:
- `redis.enqueue` — stream, message_id, payload_type (info)
- `redis.consume` — stream, message_id, consumer_name (debug)  
- `redis.ack` — stream, message_id (debug)
- `redis.recover` — stream, count_recovered (warning)
- `redis.error` — stream, error (error)
- `redis.fallback_sync` — при недоступности Redis (error)

Использовать event_logger.log_event() для ключевых моментов:
- `webhook_enqueued` — когда вебхук попал в очередь
- `webhook_processed` — когда воркер обработал
- `webhook_failed` — когда обработка упала (с traceback в details)

---

## Поведение при ошибках

| Ситуация | Поведение |
|----------|-----------|
| Redis недоступен при enqueue | Fallback на синхронную обработку (как было) |
| Redis недоступен при consume | Worker спит 10 сек, пробует снова |
| Ошибка в handler (бизнес-логика) | НЕ делать ack, сообщение остаётся в pending |
| Сообщение в pending > 5 мин | recovery_loop перехватывает и обрабатывает повторно |
| Сообщение в pending > 1 час (3 retry) | Логировать error, ack чтобы не блокировать очередь, записать в event_log как failed |

Для подсчёта retry — использовать delivery count из XPENDING (Redis 7+) или хранить счётчик в самом сообщении (поле `retry_count`).

---

## Что НЕ менять

- Flow диалога (ai_agent.py) — не трогать
- Промпты — не трогать
- Модели БД — не трогать (новых таблиц нет)
- Scheduler (message_scheduler, token_refresher, vacancy_sync) — не трогать
- Логику фильтрации своих/системных сообщений — перенести как есть
- Отправку сообщений (avito_messenger) — не трогать

---

## Порядок проверки

1. `redis-cli ping` → PONG
2. Перезапустить сервис: `systemctl restart crm-worker`
3. Проверить логи: должны быть `redis.group_created` для обоих стримов
4. Отправить тестовый вебхук: `curl -X POST http://localhost:9800/webhooks/avito -H "Content-Type: application/json" -d '{"test": true}'`
5. Проверить что вебхук попал в стрим: `redis-cli XLEN webhooks:messenger`
6. Проверить что воркер обработал: `redis-cli XINFO GROUPS webhooks:messenger` — pending должен быть 0
7. Остановить Redis (`systemctl stop redis-server`), отправить вебхук — должен обработаться синхронно (fallback), в логах `redis.fallback_sync`
