# ЗАДАЧА: Исправить преждевременное закрытие сессии при запросе дополнительных вакансий

## Контекст

Проект: `/opt/openai/crm-worker/`
Ветка: `main`

## Проблема

Когда кандидат на этапе `alternatives` просит ещё варианты ("А есть ли еще вакансии?"), бот:
1. ✅ Правильно спрашивает "Что для вас важно?"  
2. ❌ **Но при этом уже закрывает сессию** (`status=completed, result=rejected`)
3. Ответ кандидата ("Близость к дому") уже не обрабатывается — сессия мертва

**Скриншот подтверждает:** Stage: completed | Status: completed | Result: rejected — а бот только что спросил уточняющий вопрос, на который кандидат отвечает.

## Корень проблемы

В `services/ai_agent.py` → `_handle_alternatives_response()`:

Когда Claude отвечает на сообщение кандидата, и ответ содержит уточняющий вопрос ("что вам важно?") — бот уже вызвал `create_handover_card(result="rejected")` и пометил сессию как completed. Claude не знает что сессия закрывается — он просто генерирует ответ.

## Решение

### 1. В `_handle_alternatives_response()` — определять intent кандидата ПЕРЕД закрытием сессии

Когда кандидат отвечает на альтернативы, нужно парсить intent:

- **"selected"** — выбрал конкретную вакансию → показать презентацию (уже работает)
- **"more"** (НОВЫЙ) — просит ещё варианты / уточняет критерии → **НЕ закрывать сессию**, перейти на этап `clarify_alternatives`
- **"no" / "rejected"** — ничего не подходит, точно отказ → закрывать сессию, handover
- **"unclear"** — непонятно → уточнить

Добавить в промпт парсинга intent для alternatives:

```python
ALTERNATIVES_INTENT_PROMPT = """Определи намерение соискателя после предложенных альтернативных вакансий.
Сообщение: "{message}"

Варианты:
- "selected" — выбрал конкретную вакансию из предложенных
- "more" — просит ещё варианты, спрашивает что ещё есть, хочет другие предложения
- "no" — ничего не подходит, отказывается, уходит
- "unclear" — непонятно

Ответь ТОЛЬКО JSON: {{"intent": "selected|more|no|unclear"}}"""
```

### 2. Обработка intent "more"

Когда intent = "more":

```python
if intent == "more":
    # НЕ закрываем сессию!
    # Генерируем уточняющий вопрос через Claude
    clarify_text = await ask_claude(
        system=system,
        messages=history + [{"role": "user", "content": "Кандидат хочет ещё варианты. Спроси что для него важно: близость к дому, график, уровень оплаты или другое. Коротко, 1-2 предложения."}],
        ...
    )
    await schedule_message(chat.id, clarify_text, delay)
    
    # Переводим в новый этап ожидания критериев
    async with AsyncSessionFactory() as session:
        sess = await session.get(AISession, ai_session.id)
        sess.dialog_stage = "waiting_alternatives_criteria"
        await session.commit()
```

### 3. Новый stage handler: `waiting_alternatives_criteria`

Когда кандидат отвечает с критериями (например "Близость к дому"):

```python
async def _handle_alternatives_criteria(ai_session, application, chat, message, candidate_speed=None):
    """Кандидат уточнил критерии — ищем вакансии заново."""
    text = message.content
    
    # Поиск в Qdrant с учётом критериев кандидата
    # Формируем поисковый запрос: город + метро кандидата + его критерий
    search_query = f"{ai_session.collected_city} {ai_session.collected_metro} {text}"
    
    from services.ai_rag import search_vacancies
    results = await search_vacancies(
        query=search_query,
        city=ai_session.collected_city,
        limit=3,
        exclude_vacancy_id=application.avito_vacancy_id,  # исключить уже показанные
    )
    
    if results:
        # Показать новые альтернативы
        # ... (аналогично _send_alternatives, но с новыми результатами)
        # Stage → "alternatives" (ждём ответа на новые предложения)
    else:
        # Вакансий больше нет → handover
        reply = await ask_claude(...)  # "К сожалению, больше подходящих вакансий рядом нет. Передам ваши пожелания менеджеру."
        await schedule_message(...)
        await create_handover_card(ai_session.id, result="no_match")
```

### 4. Зарегистрировать новый stage в роутере

В `process_incoming_message()` добавить обработку нового этапа:

```python
elif stage == "waiting_alternatives_criteria":
    await _handle_alternatives_criteria(ai_session, application, chat, message, candidate_speed)
```

### 5. Добавить stage в ENUM

В `models/db.py` → `AISession.dialog_stage` добавить `'waiting_alternatives_criteria'` в список значений Enum.

**ВНИМАНИЕ:** Для MariaDB это потребует миграцию:

```sql
ALTER TABLE ai_sessions MODIFY COLUMN dialog_stage 
  ENUM('greeting','waiting_qualification','presentation','waiting_fork',
       'alternatives','booking','waiting_booking',
       'followup','clarify','handover','done','completed','failed',
       'channel_choice','qualification','segmentation',
       'waiting_alternatives_criteria')
  DEFAULT 'greeting';
```

## Порядок работы

1. Прочитай `services/ai_agent.py` — функцию `_handle_alternatives_response()` целиком
2. Прочитай `process_incoming_message()` — роутер стадий
3. Прочитай `models/db.py` — enum `dialog_stage`
4. Добавь новый intent "more" в парсинг альтернатив
5. Добавь handler `_handle_alternatives_criteria`
6. Зарегистрируй новый stage в роутере и enum
7. Создай SQL-миграцию

## Проверка

### Сценарий 1: Кандидат просит ещё
1. Запустить эмуляцию → ответить "45, Москва" → на презентации ответить "НЕТ"
2. Бот покажет альтернативы
3. Ответить "А есть ли ещё вакансии?"
4. **Ожидание:** Бот спросит "Что для вас важно?" И сессия **ОСТАЁТСЯ active**
5. Ответить "Близость к дому"
6. **Ожидание:** Бот найдёт подходящие вакансии в Qdrant и предложит новые варианты

### Сценарий 2: Кандидат отказывается
1. На альтернативах ответить "Нет, ничего не подходит"
2. **Ожидание:** Сессия закрывается, handover с result="rejected"

### Сценарий 3: Вакансий больше нет
1. Кандидат просит ещё → уточняет критерии
2. Qdrant не находит новых вакансий
3. **Ожидание:** "Подходящих вакансий больше нет, передам менеджеру" → handover

## Файлы для изменения

| Файл | Действие |
|------|----------|
| `services/ai_agent.py` | Новый intent "more", handler `_handle_alternatives_criteria`, регистрация в роутере |
| `models/db.py` | Добавить `waiting_alternatives_criteria` в Enum |
| `migrations/` | SQL для ALTER TABLE ai_sessions |

## SQL миграция (создать файл `migrations/007_alternatives_criteria.sql`)

```sql
ALTER TABLE ai_sessions MODIFY COLUMN dialog_stage 
  ENUM('greeting','waiting_qualification','presentation','waiting_fork',
       'alternatives','booking','waiting_booking',
       'followup','clarify','handover','done','completed','failed',
       'channel_choice','qualification','segmentation',
       'waiting_alternatives_criteria')
  DEFAULT 'greeting';
```
