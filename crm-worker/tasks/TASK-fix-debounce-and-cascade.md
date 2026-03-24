# TASK: Исправление каскадных багов диалога (быстрые сообщения кандидата)

## Контекст проблемы

Кандидат Татьяна отправила 3 сообщения подряд за 30 секунд ("Москва", "метро тимирязевская", "44 года"). Бот обработал первое, переключил stage в `waiting_fork`, и два оставшихся сообщения с данными квалификации были интерпретированы как ответ на презентацию (ok/no → unclear → handover). Сессия закрылась, а запланированные сообщения (презентация и greeting) продолжили отправляться мёртвому диалогу.

**Чат:** `u2i-arltw~S~sUAfKCDbYlOBRA`  
**Время:** 2026-03-17 23:51–23:56 МСК  
**Аккаунт:** 2 (Лавка), `avito_user_id = 328697641`

### Хронология сбоя

```
23:51:18  Татьяна: "Доброй ночи ! я по поводу работы"
          → stage=waiting_qualification, clarify_count=1
          → бот спросил город/метро/возраст (scheduled)

23:52:42  Татьяна: "Москва"
          → stage=waiting_qualification → city=Москва
          → clarify_count>=1 → идём в presentation (scheduled)
          → stage=waiting_fork, clarify_count=0

23:52:59  Татьяна: "метро тимирязевская"
          → stage=waiting_fork → _handle_fork("метро тимирязевская")
          → intent="unclear" → _send_clarify
          → clarify_count=1, stage=clarify

23:53:06  Татьяна: "44 года"
          → stage=clarify → _handle_clarify_response
          → return_stage=waiting_fork → _handle_fork("44 года")
          → intent="unclear" → clarify_count>=1
          → result="unclear" → HANDOVER → СЕССИЯ ЗАКРЫТА

23:53:15  Scheduled presentation доставлена (сессия уже мертва!)
23:53:36  Татьяна: "да устраивает" → no_active_session → МОЛЧАНИЕ
23:53:48  Татьяна: "а ежедневных выплат нет?" → МОЛЧАНИЕ
23:54:52  Татьяна: "что нужно упаковывать?" → МОЛЧАНИЕ
23:55:25  Scheduled greeting доставлен → ПОВТОРНОЕ приветствие в мёртвый диалог
```

---

## Баг 1 (КРИТИЧЕСКИЙ): Нет дебаунса входящих сообщений

**Проблема:** Каждое сообщение кандидата обрабатывается мгновенно и последовательно. Если кандидат шлёт 3 сообщения за 30 сек, каждое меняет `dialog_stage`, и 2-е/3-е попадают в чужой обработчик стадии.

**Файл:** `workers/incoming_processor.py`

**Решение:** ПОЛНАЯ ЗАМЕНА файла. Добавить per-chat дебаунс — при получении сообщения ждать 10 секунд, и если за это время пришли ещё сообщения от того же кандидата, склеить их в одно.

```python
"""Обработка входящих сообщений от соискателей."""
import asyncio
from datetime import datetime
from sqlalchemy import select
from models.db import AsyncSessionFactory, Message, Chat, AISession
from services.ai_agent import process_incoming_message
from utils.logger import get_logger
from utils.time_helpers import is_night_window

log = get_logger(__name__)

# Per-chat дебаунс: chat_db_id → asyncio.Task
_debounce_tasks: dict[int, asyncio.Task] = {}
_debounce_messages: dict[int, list[int]] = {}  # chat_db_id → [message_ids]
DEBOUNCE_SEC = 10  # ждём 10 сек перед обработкой


async def handle_incoming(message_id: int) -> None:
    """
    Точка входа для обработки входящего сообщения.
    Использует дебаунс: ждёт 10 сек, собирая все сообщения от кандидата.
    """
    async with AsyncSessionFactory() as session:
        message = await session.get(Message, message_id)
        if not message:
            log.error("incoming_message_not_found", message_id=message_id)
            return

        chat = await session.get(Chat, message.chat_id)
        if not chat:
            log.error("chat_not_found_for_incoming", message_id=message_id)
            return

        # Найти активную ai_session
        result = await session.execute(
            select(AISession).where(
                AISession.chat_id == chat.id,
                AISession.status == "active",
            )
        )
        ai_session = result.scalar_one_or_none()
        if not ai_session:
            log.info("no_active_session", chat_id=chat.id, message_id=message_id)
            return

        # Обновить время последнего сообщения
        ai_session.last_applicant_msg_at = datetime.utcnow()
        chat.last_message_at = datetime.utcnow()
        await session.commit()

        chat_db_id = chat.id

    # Проверить ночное окно
    if not is_night_window():
        log.info("outside_night_window_incoming", message_id=message_id)
        return

    # --- Дебаунс ---
    # Добавить message_id в очередь для этого чата
    if chat_db_id not in _debounce_messages:
        _debounce_messages[chat_db_id] = []
    _debounce_messages[chat_db_id].append(message_id)

    # Отменить предыдущий таск дебаунса (если есть)
    if chat_db_id in _debounce_tasks:
        _debounce_tasks[chat_db_id].cancel()

    # Создать новый таск дебаунса
    _debounce_tasks[chat_db_id] = asyncio.create_task(
        _debounce_process(chat_db_id)
    )


async def _debounce_process(chat_db_id: int) -> None:
    """Ждёт DEBOUNCE_SEC и обрабатывает последнее сообщение (с учётом всех предыдущих)."""
    try:
        await asyncio.sleep(DEBOUNCE_SEC)

        message_ids = _debounce_messages.pop(chat_db_id, [])
        _debounce_tasks.pop(chat_db_id, None)

        if not message_ids:
            return

        # Берём ПОСЛЕДНИЙ message_id — он будет обработан
        last_message_id = message_ids[-1]

        if len(message_ids) > 1:
            # Склеить тексты всех сообщений в последнее
            await _merge_messages(message_ids)
            log.info(
                "debounce_merged",
                chat_db_id=chat_db_id,
                message_count=len(message_ids),
                processing_id=last_message_id,
            )
        else:
            log.info("processing_incoming_debounced", message_id=last_message_id)

        try:
            await process_incoming_message(last_message_id)
        except Exception as exc:
            log.error("incoming_processing_failed", message_id=last_message_id, error=str(exc))

    except asyncio.CancelledError:
        # Дебаунс сброшен — пришло ещё одно сообщение, ждём заново
        pass
    except Exception as exc:
        log.error("debounce_process_error", chat_db_id=chat_db_id, error=str(exc))


async def _merge_messages(message_ids: list[int]) -> None:
    """Склеивает тексты нескольких сообщений в последнее."""
    async with AsyncSessionFactory() as session:
        result = await session.execute(
            select(Message)
            .where(Message.id.in_(message_ids))
            .order_by(Message.created_at.asc())
        )
        messages = result.scalars().all()

        if len(messages) <= 1:
            return

        # Склеить тексты через перенос строки
        combined_text = "\n".join(m.content for m in messages if m.content)

        # Записать в последнее сообщение
        last_msg = messages[-1]
        last_msg.content = combined_text
        await session.commit()

        log.info(
            "messages_merged",
            original_count=len(messages),
            combined_length=len(combined_text),
            last_msg_id=last_msg.id,
        )
```

**ВАЖНО о дебаунсе:** 10 сек — компромисс. Кандидат не заметит задержки, т.к. бот и так отвечает с паузой 15-40 секунд. Но этого достаточно, чтобы собрать все быстрые сообщения.

---

## Баг 2 (КРИТИЧЕСКИЙ): Scheduled messages не проверяют актуальность сессии

**Проблема:** `message_scheduler.process_scheduled()` отправляет ВСЕ запланированные сообщения, даже если:
- Сессия закрыта (`status != "active"`)
- Stage ушёл далеко вперёд (greeting отправляется когда уже fork)

Это привело к:
- Отправке презентации после handover
- Повторному greeting когда диалог уже на стадии fork

**Файл:** `services/message_scheduler.py`

**Решение:** Заменить метод `process_scheduled()` (строка ~85 и далее):

```python
async def process_scheduled() -> None:
    """
    Проверяет очередь отложенных сообщений и отправляет готовые.
    Вызывается каждые 5 секунд из APScheduler.
    Перед отправкой проверяет, что AI-сессия ещё активна.
    """
    now = datetime.utcnow()

    async with AsyncSessionFactory() as session:
        result = await session.execute(
            select(Message)
            .where(
                Message.direction == "outgoing",
                Message.scheduled_at <= now,
                Message.delivered_at == None,
            )
            .limit(50)
        )
        messages = result.scalars().all()

    for msg in messages:
        try:
            async with AsyncSessionFactory() as session:
                chat = await session.get(Chat, msg.chat_id)
                if not chat:
                    log.error("chat_not_found", msg_id=msg.id)
                    continue

                # --- НОВОЕ: проверить актуальность сессии ---
                from models.db import AISession
                result = await session.execute(
                    select(AISession).where(
                        AISession.chat_id == chat.id,
                    ).order_by(AISession.id.desc()).limit(1)
                )
                ai_session = result.scalar_one_or_none()

                if ai_session and ai_session.status != "active":
                    # Сессия закрыта — не отправлять, пометить как отменённое
                    db_msg = await session.get(Message, msg.id)
                    if db_msg:
                        db_msg.delivered_at = now
                        await session.commit()
                    log.info(
                        "scheduled_message_skipped_inactive",
                        msg_id=msg.id,
                        chat_id=msg.chat_id,
                        session_status=ai_session.status,
                    )
                    continue
                # --- КОНЕЦ НОВОГО ---

                avito_chat_id = chat.avito_chat_id
                account_id = chat.account_id

            await send_message(account_id, avito_chat_id, msg.content, skip_db=True)

            # Обновить delivered_at
            async with AsyncSessionFactory() as session:
                db_msg = await session.get(Message, msg.id)
                if db_msg:
                    db_msg.delivered_at = datetime.utcnow()
                    await session.commit()

            log.info("scheduled_message_delivered", msg_id=msg.id, chat_id=msg.chat_id)

        except Exception as exc:
            log.error("scheduled_message_failed", msg_id=msg.id, error=str(exc))
```

---

## Баг 3 (СРЕДНИЙ): `_handle_fork` не различает данные квалификации от ответа на презентацию

**Проблема:** Когда stage = `waiting_fork`, но presentation ещё не доставлена (в scheduled), кандидат продолжает слать данные квалификации. `_handle_fork` парсит "метро тимирязевская" через `INTENT_PARSE_PROMPT` и получает `unclear`.

**Файл:** `services/ai_agent.py`, функция `_handle_fork` (строка ~557)

**Решение:** Перед парсингом intent проверить — не содержит ли сообщение квалификационные данные. Если квалификация ещё не полная и сообщение содержит данные — дособрать вместо парсинга intent.

Заменить функцию `_handle_fork` (строки 557-588):

```python
async def _handle_fork(
    ai_session: AISession,
    application: Application,
    chat: Chat,
    message: Message,
    candidate_speed: int | None = None,
) -> None:
    """Парсим намерение: ok → booking, no → alternatives, unclear → clarify.
    
    ВАЖНО: если квалификация ещё не полная, сначала пробуем
    извлечь квалификационные данные (город/метро/возраст).
    Это защита от ситуации когда кандидат шлёт данные быстрее
    чем бот доставляет presentation.
    """
    # --- НОВОЕ: проверить, не данные ли это квалификации ---
    needs_qualification = not ai_session.collected_age or not (
        ai_session.collected_city or ai_session.collected_metro
    )
    
    if needs_qualification:
        try:
            raw = await call_claude(
                system="Извлеки данные из сообщения. Ответь ТОЛЬКО JSON.",
                user_message=QUALIFICATION_PARSE_PROMPT.format(message=message.content),
                max_tokens=150,
                temperature=0.1,
            )
            qual_data = _extract_json(raw)
            
            has_qual_data = bool(
                qual_data.get("age") or qual_data.get("city") or qual_data.get("metro")
            )
            
            if has_qual_data:
                # Это данные квалификации — дособрать, не парсить intent
                log.info(
                    "fork_reclassified_as_qualification",
                    message=message.content[:100],
                    ai_session_id=ai_session.id,
                )
                
                async with AsyncSessionFactory() as session:
                    sess = await session.get(AISession, ai_session.id)
                    if sess:
                        if qual_data.get("age") and not sess.collected_age:
                            sess.collected_age = str(qual_data["age"])
                        if qual_data.get("city") and not sess.collected_city:
                            sess.collected_city = qual_data["city"]
                        if qual_data.get("metro") and not sess.collected_metro:
                            sess.collected_metro = qual_data["metro"]
                        await session.commit()
                
                # Не отправлять ничего — презентация уже в scheduled
                return
        except Exception as exc:
            log.warning("fork_qualification_check_failed", error=str(exc))
    # --- КОНЕЦ НОВОГО ---
    
    # Обычный парсинг intent
    try:
        raw = await call_claude(
            system="Определи намерение. Ответь ТОЛЬКО JSON.",
            user_message=INTENT_PARSE_PROMPT.format(message=message.content),
            max_tokens=50,
            temperature=0.1,
        )
        data = _extract_json(raw)
        intent = data.get("intent", "unclear")
    except Exception:
        intent = "unclear"

    log.info("fork_intent", intent=intent, ai_session_id=ai_session.id)

    if intent == "ok":
        await _send_booking(ai_session, application, chat, candidate_speed)
    elif intent == "no":
        await _send_alternatives(ai_session, application, chat, candidate_speed)
    else:
        await _send_clarify(ai_session, application, chat, message,
                            expected="ОК (согласен) или НЕТ (хочет другие варианты)",
                            return_stage="waiting_fork",
                            candidate_speed=candidate_speed)
```

---

## Порядок применения

1. **Баг 2** → `services/message_scheduler.py` — замена `process_scheduled()`. Минимальный риск, самое важное.
2. **Баг 1** → `workers/incoming_processor.py` — полная замена файла. Ключевое исправление.
3. **Баг 3** → `services/ai_agent.py` — замена функции `_handle_fork`. Дополнительная защита.

---

## Тестирование

После применения всех изменений:

1. Перезапустить сервис:
```bash
systemctl restart k24-crm-worker
```

2. Проверить запуск:
```bash
journalctl -u k24-crm-worker -f
```

3. **Тестовый сценарий** — отправить 3 сообщения подряд (интервал < 10 сек):
   - "Москва"
   - "метро Тимирязевская"  
   - "44 года"

4. **Ожидаемый результат в логах:**
   - `debounce_merged message_count=3` — сообщения склеены
   - `qualification_complete` — все данные собраны за раз
   - Презентация отправлена ОДИН раз
   - Greeting НЕ задублирован
   - Нет `result="unclear"` и handover

5. **Тест scheduled при закрытой сессии:**
   - Закрыть сессию вручную в БД (`UPDATE ai_sessions SET status='completed' WHERE id=X`)
   - Убедиться что запланированные сообщения не отправляются
   - В логах: `scheduled_message_skipped_inactive`

---

## Файлы для изменения

| Файл | Действие | Строки |
|------|----------|--------|
| `workers/incoming_processor.py` | Полная замена | весь файл |
| `services/message_scheduler.py` | Замена `process_scheduled()` | строки ~85-125 |
| `services/ai_agent.py` | Замена `_handle_fork()` | строки 557-588 |
