# ЗАДАЧА: Обработка edge-кейсов в диалоге (сегодня на смену + телефон)

## Контекст

Проект: `/opt/openai/crm-worker/`
Ветка: `main`

Бот (Елена) работает в **ночном окне** (`ai_work_start=18:00`, `ai_work_end=09:59` МСК).
Когда кандидат подтверждает интерес к вакансии → этап `booking` → спрашиваем удобное время для звонка менеджера.

**Три проблемы:**

1. Кандидат говорит «хочу выйти сегодня» / «можно сегодня на смену?» / «когда можно начать?» — но бот работает ночью, менеджер позвонит только **завтра в рабочее время**. Нужно мягко объяснить.

2. Кандидат отказывается давать номер телефона или говорит «не хочу звонок» / «не буду давать телефон» / «лучше тут в чате». Нужно обработать отказ, не давить, и всё равно довести до handover.

3. Кандидат сам просит номер телефона компании («дайте ваш номер», «какой у вас телефон?», «я сам позвоню»). Елена НЕ МОЖЕТ дать номер телефона — нужно мягко перенаправить на то что менеджер сам свяжется.

---

## Что нужно сделать

### 1. Обновить `prompts/system.txt`

Добавить в раздел «Правила» (после существующих правил):

```
- Ты работаешь в вечернее и ночное время. Менеджер свяжется с кандидатом ЗАВТРА в рабочее время (с 10:00 до 18:00). Если кандидат хочет выйти "сегодня" или "прямо сейчас" -- объясни что сейчас нерабочее время, менеджер свяжется завтра утром и всё оперативно организует.
- Если кандидат не хочет давать телефон или отказывается от звонка -- не дави. Скажи что передашь информацию менеджеру и он напишет прямо сюда в чат Авито. НЕ настаивай на телефоне.
- Если кандидат ПРОСИТ номер телефона компании ("дайте номер", "какой ваш телефон", "я сам позвоню") -- НЕЛЬЗЯ давать никакой номер. Скажи что у тебя нет возможности дать прямой номер, но менеджер сам свяжется с кандидатом завтра в удобное время. Предложи оставить удобное время для звонка.
```

### 2. Обновить `prompts/booking.txt`

Текущий промпт просит спросить удобное время для звонка. Нужно расширить:

**Заменить весь файл `prompts/booking.txt` на:**

```
Soiskatel' podtverdil interes k vakansii "{vacancy_title}".
Napishi podtverzhdeniye i predlozhi vybrat' udobnoye vremya dlya zvonka.

Obyazatel'nye elementy:
1. Poblagodarit' za "OK"
2. Podtverdit' chto eto khoroshiy vybor
3. Upomyanut' chto seychas nerabocheye vremya, menedzher svyazhetsya ZAVTRA v rabocheye vremya
4. Sprosit' v kakoye vremya zavtra udobno prinyat' zvonok menedzhera

Pravila:
- Toplyy pozitivnyy ton
- NE ispol'zuy emodži
- NE predlagay fiksirovannye sloty
- Yesli seychas noch' -- podcherkni chto menedzher pozvonil ZAVTRA
- 3-5 predlozheniy
```

### 3. Добавить новый промпт `prompts/no_phone.txt`

Создать файл:

```
Soiskatel' otkazalsya davat' nomer telefona ili ne khochet zvonok.

Yego soobshcheniye: "{message_text}"

Reagiruy spokoyno i uvazhi'tel'no:
1. Skazhi chto ponyala, ne problema
2. Predlozhi al'ternativu: menedzher mozhet napisat' pryamo syuda v chat Avito zavtra v rabocheye vremya
3. Utochni v kakoye vremya zavtra udobno poluchit' soobshcheniye

Pravila:
- NE davi i NE nastaivay na telefone
- NE ispol'zuy emodži
- Druzhlyubno, 2-3 predlozheniya
```

### 3b. Добавить новый промпт `prompts/asks_phone.txt`

Создать файл:

```
Soiskatel' prosit nomer telefona kompanii, khochet sam pozvonit'.

Yego soobshcheniye: "{message_text}"

Reagiruy vezhlyvo:
1. Skazhi chto, k sozhaleniyu, u tebya net vozmozhnosti dat' pryamoy nomer
2. Ob'yasni chto menedzher sam svyazhetsya s nim zavtra v rabocheye vremya -- eto bystree i nadyozhneye
3. Sprosit' v kakoye vremya zavtra udobno prinyat' zvonok

Pravila:
- NE davay nikakikh nomerov telefonov
- NE pridumyvay nomera
- Toplyy pozitivnyy ton
- NE ispol'zuy emodži
- 2-3 predlozheniya
```

### 4. Обновить `services/ai_agent.py` — обработка этапа `waiting_booking`

В функции `_handle_booking_response` (этап `waiting_booking`) сейчас парсится callback_slot. Нужно добавить обработку двух новых кейсов:

#### 4.1. Добавить новый промпт парсинга отказа от телефона

Рядом с существующим `BOOKING_PARSE_PROMPT` добавить:

```python
PHONE_REFUSAL_DETECT_PROMPT = """Opredeli, otkazyvaetsya li soiskatel' ot telefonnogo zvonka, ot predostavleniya nomera telefona, ILI prosit nomer telefona kompanii.
Soobshcheniye: "{message}"

Varianty:
- "refused" -- ne khochet zvonok, ne dayet telefon, khochet tol'ko chat ("ne khochu zvonok", "ne dam telefon", "luchshe v chate", "pishite syuda", "ne nado zvonit'", "bez zvonka")
- "asks_phone" -- prosit nomer telefona kompanii, khochet sam pozvonit' ("dayte nomer", "kakoy vash telefon", "ya sam pozvonyu", "skinte nomer", "telefon kompanii")
- "none" -- ni to ni drugoye, obychnyy otvet

Otvet' TOL'KO JSON: {{"phone_action": "refused|asks_phone|none"}}"""
```

#### 4.2. Изменить логику `_handle_booking_response`

**ВАЖНО:** Читай текущую реализацию `_handle_booking_response` в полном файле `services/ai_agent.py` (строки ~700-800). Не перезаписывай, а дополни.

Сейчас логика:
1. Парсим `callback_slot` из ответа кандидата
2. Если есть слот → сохраняем → handover(result="booking")
3. Если нет слота → clarify

**Добавить перед парсингом `callback_slot`:**

```python
# --- Проверяем отказ от телефона / запрос нашего номера ---
try:
    phone_check = await ask_claude(
        system="Ty -- parser.",
        messages=[{"role": "user", "content": PHONE_REFUSAL_DETECT_PROMPT.format(message=message.content)}],
        session_id=ai_session.id,
        application_id=application.id,
        dialog_stage="waiting_booking",
        temperature=0.0,
        max_tokens=50,
    )
    phone_data = _extract_json(phone_check)
    phone_action = phone_data.get("phone_action", "none")

    if phone_action == "refused":
        # Кандидат не хочет звонок -- мягкий ответ, предлагаем чат
        prompt_text = _load_prompt("no_phone").replace("{message_text}", message.content)
        system = await _build_system_prompt(ai_session, vacancy_data)
        history = await _get_dialog_history(chat.id)
        messages_for_claude = history + [{"role": "user", "content": prompt_text}]

        reply = await ask_claude(
            system=system,
            messages=messages_for_claude,
            session_id=ai_session.id,
            application_id=application.id,
            dialog_stage="waiting_booking",
        )
        delay = calc_message_delay(candidate_response_sec=candidate_speed)
        await schedule_message(chat.id, reply, delay)

        # Завершаем сессию — передаём менеджеру с пометкой
        async with AsyncSessionFactory() as session:
            sess = await session.get(AISession, ai_session.id)
            if sess:
                sess.callback_slot = "chat_only_no_phone"
                await session.commit()

        await create_handover_card(ai_session.id, result="booking_no_phone")
        return

    elif phone_action == "asks_phone":
        # Кандидат просит НАШ номер -- мягко отказываем, возвращаем к booking
        prompt_text = _load_prompt("asks_phone").replace("{message_text}", message.content)
        system = await _build_system_prompt(ai_session, vacancy_data)
        history = await _get_dialog_history(chat.id)
        messages_for_claude = history + [{"role": "user", "content": prompt_text}]

        reply = await ask_claude(
            system=system,
            messages=messages_for_claude,
            session_id=ai_session.id,
            application_id=application.id,
            dialog_stage="waiting_booking",
        )
        delay = calc_message_delay(candidate_response_sec=candidate_speed)
        await schedule_message(chat.id, reply, delay)

        # НЕ завершаем сессию -- остаёмся в waiting_booking,
        # ждём когда кандидат скажет удобное время
        log.info("candidate_asks_phone_redirected", ai_session_id=ai_session.id)
        return

except Exception as exc:
    log.warning("phone_action_check_failed", error=str(exc))
```

**Разница в поведении:**
- `refused` (не хочет звонок) → завершаем сессию, handover с `booking_no_phone`, менеджер пишет в чат
- `asks_phone` (просит наш номер) → **НЕ завершаем**, Елена мягко объясняет что номера нет, просит выбрать время для звонка → ждём ответ в `waiting_booking`

#### 4.3. Обработка «хочу сегодня» — уже покрывается промптами

Кейс «хочу выйти сегодня» не требует отдельной логики в коде — достаточно обновлённых промптов `system.txt` и `booking.txt`. Claude сам ответит что менеджер свяжется завтра, а из ответа кандидата распарсит слот типа "zavtra utrom" / "s 10 do 12".

Если кандидат настаивает на «именно сегодня» — промпт в `system.txt` направит Елену мягко объяснить что сейчас ночь и завтра утром всё организуют.

#### 4.4. Запрос номера на ДРУГИХ этапах (не booking)

Если кандидат просит номер телефона на этапе qualification, presentation или другом — обработка через `system.txt` промпт. Специальная детекция в коде нужна ТОЛЬКО на этапе `waiting_booking`, потому что там ответ кандидата парсится автоматически и может быть неправильно интерпретирован как callback_slot.

### 5. Обновить `handover_cards` — новый результат

В `services/handover.py` и `services/telegram_notifier.py` добавить обработку нового результата `booking_no_phone`:

#### 5.1. `services/telegram_notifier.py` — в `format_card_for_telegram`:

Добавить в словарь `result_labels`:

```python
"booking_no_phone": "[Запись БЕЗ телефона -- писать в чат]",
```

#### 5.2. `templates/admin.html` — цвет для нового результата

В JS-секции где рисуются карточки, добавить обработку `booking_no_phone`:
- Цвет: оранжевый (предупреждение — менеджер должен писать в чат, а не звонить)
- Текст: «Запись без телефона — написать в чат Авито»

### 6. Обновить `models/db.py` — enum AISession.result

Поле `ai_sessions.result` — это `VARCHAR(50)`, так что новое значение `booking_no_phone` уже влезет. Если где-то есть валидация допустимых значений — добавить `booking_no_phone`.

---

## Порядок выполнения

1. Обновить `prompts/system.txt` — добавить 3 правила (ночное время, отказ от телефона, запрос нашего номера)
2. Заменить `prompts/booking.txt` — упоминание «завтра»
3. Создать `prompts/no_phone.txt`
4. Создать `prompts/asks_phone.txt`
5. Обновить `services/ai_agent.py` — добавить `PHONE_REFUSAL_DETECT_PROMPT` + логику в `_handle_booking_response`
6. Обновить `services/telegram_notifier.py` — новый label
7. Обновить `templates/admin.html` — цвет для `booking_no_phone`

---

## Проверка после выполнения

```bash
# Импорты
python -c "from services.ai_agent import PHONE_REFUSAL_DETECT_PROMPT; print('OK')"
python -c "from services.ai_agent import process_incoming_message; print('OK')"

# Промпты существуют
test -f prompts/no_phone.txt && echo "no_phone.txt OK" || echo "MISSING"
test -f prompts/asks_phone.txt && echo "asks_phone.txt OK" || echo "MISSING"
grep -q "ZAVTRA" prompts/booking.txt && echo "booking.txt updated" || echo "NOT UPDATED"
grep -q "ne davi" prompts/system.txt && echo "system.txt updated" || echo "NOT UPDATED"
grep -q "pryamoy nomer" prompts/system.txt && echo "system.txt phone rule OK" || echo "MISSING"

# Новый результат в telegram
grep -q "booking_no_phone" services/telegram_notifier.py && echo "telegram OK" || echo "MISSING"
```

---

## CHANGELOG

Создай `docs/dev/dialog-edge-cases-deploy.md`:

1. Таблица изменений (файл → что изменено → зачем)
2. Новые файлы
3. Тестовые сценарии:
   - Кандидат: «Хочу сегодня начать» → Елена: «Менеджер свяжется завтра утром»
   - Кандидат: «Не хочу звонок, пишите сюда» → Елена: «Хорошо, менеджер напишет в чат» → handover с result=booking_no_phone
   - Кандидат: «Дайте ваш номер, я сам позвоню» → Елена: «Номера нет, но менеджер сам свяжется. Когда удобно?» → остаёмся в waiting_booking
   - Кандидат: «Какой телефон?» → Елена: «Менеджер позвонит вам сам. В какое время завтра удобно?» → остаёмся в waiting_booking
   - Кандидат: «Завтра в 11 удобно» → обычный flow booking
