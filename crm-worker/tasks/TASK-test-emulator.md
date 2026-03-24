# TASK: Эмулятор отклика для тестирования AI-бота

**Приоритет:** Средний
**Дата:** 2026-03-19

---

## 1. Контекст

Для тестирования AI-бота нужен механизм эмуляции отклика без реальных кандидатов с Авито. Сейчас для теста нужно ждать реальный отклик — это медленно и неудобно.

Нужен admin-эндпоинт, который создаёт все записи в БД напрямую (applicant, application, chat, ai_session) и запускает AI-flow. Сообщения AI будут сохраняться в БД, но НЕ отправляться в Авито (тестового чата в мессенджере нет).

---

## 2. Что нужно сделать

### 2.1. Новый эндпоинт: `POST /admin/api/test/emulate-application`

Файл: `api/admin.py`

**Входные параметры (JSON):**

```python
class EmulateApplicationRequest(BaseModel):
    account_id: int                      # ID аккаунта (1, 2, 3, 4)
    vacancy_id: Optional[int] = None     # avito_vacancy_id (если не указан — берём случайную активную вакансию аккаунта)
    candidate_name: str = "Тест Тестов"
    candidate_phone: str = "+79991234567"
    force_night: bool = True             # Принудительно считать что сейчас ночное окно (AI работает)
```

**Логика:**

1. Проверить авторизацию (`_verify_admin`)
2. Если `vacancy_id` не указан — взять случайную активную вакансию с `account_id`:
   ```python
   from sqlalchemy.sql.expression import func as sql_func
   result = await session.execute(
       select(Vacancy).where(
           Vacancy.account_id == account_id,
           Vacancy.is_active == True,
       ).order_by(sql_func.rand()).limit(1)
   )
   vacancy = result.scalar_one_or_none()
   ```
3. Сгенерировать уникальный `avito_application_id`: `f"test-{uuid4().hex[:12]}"`
4. Сгенерировать уникальный `avito_chat_id`: `f"test-chat-{uuid4().hex[:12]}"`
5. Создать записи в БД:
   - `Applicant` (name, phone)
   - `Application` (avito_application_id, applicant_id, account_id, avito_vacancy_id, status="ai_active")
   - `Chat` (avito_chat_id, application_id, account_id)
   - `AISession` (application_id, chat_id, dialog_stage="greeting", status="active")
6. Если `force_night=True` ИЛИ `is_night_window()` — запустить AI:
   ```python
   from services.ai_agent import process_new_application
   asyncio.create_task(process_new_application(application.id))
   ```
7. Вернуть:
   ```json
   {
     "status": "ok",
     "application_id": 123,
     "ai_session_id": 456,
     "chat_id": 789,
     "avito_chat_id": "test-chat-abc123",
     "vacancy_title": "Упаковщик на склад",
     "message": "Эмуляция запущена. AI-сессия создана, greeting будет сгенерирован."
   }
   ```

### 2.2. Новый эндпоинт: `POST /admin/api/test/emulate-message`

**Входные параметры (JSON):**

```python
class EmulateMessageRequest(BaseModel):
    chat_id: int          # Наш внутренний ID чата (из ответа emulate-application)
    text: str             # Текст "от кандидата"
```

**Логика:**

1. Проверить авторизацию (`_verify_admin`)
2. Найти `Chat` по `chat_id`
3. Создать `Message`:
   ```python
   msg = Message(
       chat_id=chat_id,
       avito_message_id=f"test-msg-{uuid4().hex[:12]}",
       direction="incoming",
       sender_type="applicant",
       content=text,
   )
   ```
4. Обновить `chat.last_message_at`
5. Запустить обработку:
   ```python
   from workers.incoming_processor import handle_incoming
   asyncio.create_task(handle_incoming(msg.id))
   ```
6. Вернуть:
   ```json
   {
     "status": "ok",
     "message_id": 789,
     "chat_id": 123,
     "text": "Москва, метро Тимирязевская, 30 лет"
   }
   ```

### 2.3. Перехват отправки для тестовых чатов

Файл: `services/avito_messenger.py`

В функции `send_message`, перед реальной отправкой в Авито, проверить:

```python
async def send_message(account_id: int, chat_id: str, text: str, skip_db: bool = False) -> dict:
    # Тестовые чаты — не отправляем в Авито, только сохраняем в БД
    if chat_id.startswith("test-chat-"):
        log.info("test_message_intercepted", chat_id=chat_id, text_preview=text[:100])
        fake_result = {"id": f"test-sent-{__import__('uuid').uuid4().hex[:12]}"}
        if not skip_db:
            # ... сохранить в БД как обычно (тот же блок что уже есть)
        return fake_result

    # ... остальной код без изменений
```

Это позволит AI генерировать ответы, они будут сохраняться в БД как `delivered`, но не уйдут в реальный Авито.

### 2.4. Просмотр тестового диалога

Файл: `api/admin.py`

```python
@router.get("/api/test/dialog/{chat_id}", dependencies=[Depends(_verify_admin)])
async def get_test_dialog(chat_id: int):
    """Показать все сообщения тестового диалога."""
    async with AsyncSessionFactory() as session:
        chat = await session.get(Chat, chat_id)
        if not chat:
            raise HTTPException(status_code=404, detail="Chat not found")

        result = await session.execute(
            select(Message).where(Message.chat_id == chat_id)
            .order_by(Message.created_at.asc())
        )
        messages = result.scalars().all()

        # AI session info
        sess_result = await session.execute(
            select(AISession).where(AISession.chat_id == chat_id)
            .order_by(AISession.id.desc()).limit(1)
        )
        ai_session = sess_result.scalar_one_or_none()

    return {
        "chat_id": chat_id,
        "avito_chat_id": chat.avito_chat_id,
        "is_test": chat.avito_chat_id.startswith("test-chat-"),
        "ai_session": {
            "id": ai_session.id if ai_session else None,
            "stage": ai_session.dialog_stage if ai_session else None,
            "status": ai_session.status if ai_session else None,
            "result": ai_session.result if ai_session else None,
        } if ai_session else None,
        "messages": [
            {
                "id": m.id,
                "sender": m.sender_type,
                "direction": m.direction,
                "text": m.content,
                "scheduled_at": str(m.scheduled_at) if m.scheduled_at else None,
                "delivered_at": str(m.delivered_at) if m.delivered_at else None,
                "created_at": str(m.created_at),
            }
            for m in messages
        ],
    }
```

### 2.5. Кнопка в админке (опционально)

В `templates/admin.html` — секция "Тестирование" с формой:
- Выбор аккаунта (dropdown)
- Выбор вакансии (dropdown, подгружается по account_id) или "случайная"
- Имя кандидата, телефон
- Кнопка "Запустить эмуляцию"
- После запуска — показать лог диалога с auto-refresh (polling `/api/test/dialog/{chat_id}`)
- Поле ввода текста + кнопка "Отправить сообщение от кандидата"

---

## 3. Сценарий тестирования

```
1. POST /admin/api/test/emulate-application
   {"account_id": 2, "candidate_name": "Иван Тестов", "candidate_phone": "+79998887766"}

   → Создан application, chat, ai_session
   → AI генерирует greeting (через 5-15 сек)

2. GET /admin/api/test/dialog/{chat_id}
   → Видим greeting от Elena

3. POST /admin/api/test/emulate-message
   {"chat_id": 789, "text": "Москва, метро Бауманская, 25 лет"}

   → AI обрабатывает квалификацию → отправляет презентацию

4. POST /admin/api/test/emulate-message
   {"chat_id": 789, "text": "Да, интересно"}

   → AI предлагает запись на звонок

5. POST /admin/api/test/emulate-message
   {"chat_id": 789, "text": "Завтра в 10 утра"}

   → AI создаёт handover-карточку
```

---

## 4. Файлы для изменения

| Файл | Действие |
|------|----------|
| `api/admin.py` | Добавить 3 эндпоинта (emulate-application, emulate-message, test/dialog) |
| `services/avito_messenger.py` | Перехват отправки для `test-chat-*` |
| `templates/admin.html` | Секция "Тестирование" (опционально) |

---

## 5. Важные моменты

- Все тестовые ID начинаются с `test-` — легко отличить от реальных
- Тестовые сообщения НЕ уходят в Авито — перехватываются в `send_message`
- `force_night=True` по умолчанию — не нужно ждать ночного окна
- Тестовые данные останутся в БД — для очистки можно добавить отдельный эндпоинт позже
- Scheduler (`process_scheduled`) будет обрабатывать тестовые сообщения так же как реальные — но отправка перехватится в `send_message`
