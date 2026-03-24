# ЗАДАЧА: Доставка карточек менеджерам (Telegram + Админка)

## Контекст

Проект: `/opt/openai/crm-worker/`
Таблица `handover_cards` уже заполняется AI-агентом после завершения диалога. Пример записи:

```
vacancy_title:   Нажиматель кнопок
candidate_name:  Ларина Юлия
candidate_phone: 79777031312
candidate_city:  Артём
candidate_metro: NULL
candidate_age:   24
result:          booking
callback_slot:   11-13
dialog_summary:  Юлия Ларина, 24 года, г. Артём. Ознакомлена с вакансией...
messages_count:  13
is_processed:    0
```

Нужно: 
1. Раздел "Карточки" в веб-админке
2. Telegram-бот для отправки сводки в 09:00
3. Оба канала — по необработанным карточкам (`is_processed = 0`)

---

## 1. АДМИНКА — раздел "Карточки"

### Меню

В `templates/admin.html` добавь третий пункт в левое меню: **"Карточки"** (между "Аккаунты" и "Логи").

### Интерфейс раздела

```
[Фильтр: ▼ Все аккаунты]  [Фильтр: ▼ Все результаты]  [Только необработанные: ✅]  [🔄]

┌─────────────────────────────────────────────────────────────────────┐
│ 📋 Ларина Юлия, 24 года                           10 мар, 16:00   │
│ 📞 79777031312  📍 Артём                                           │
│ 💼 Нажиматель кнопок                                               │
│ 📊 Результат: booking  🕐 Слот: 11:00–13:00                       │
│ 💬 13 сообщений                                                    │
│                                                                     │
│ Юлия Ларина, 24 года, г. Артём. Ознакомлена с вакансией            │
│ «Нажиматель кнопок» — условия приняла. Записана на звонок...       │
│                                                                     │
│ [✅ Обработано]  [📋 Показать диалог]                               │
└─────────────────────────────────────────────────────────────────────┘
```

### Карточка — поля для отображения

Из таблицы `handover_cards`:
- `candidate_name` + `candidate_age` — заголовок
- `candidate_phone` — с кнопкой копирования (click → clipboard)
- `candidate_city` + `candidate_metro`
- `vacancy_title`
- `result` — отображать человекочитаемо:
  - `booking` → "Записан на звонок" (зелёный)
  - `interested` → "Заинтересован" (жёлтый)
  - `not_interested` → "Не интересно" (серый)
  - `alternative` → "Нужны альтернативы" (синий)
  - другое → как есть
- `callback_slot` — слот для звонка
- `dialog_summary` — резюме диалога
- `messages_count`
- `created_at` — дата/время

### Кнопки на карточке

**[✅ Обработано]** — ставит `is_processed = 1`, `processed_at = NOW()`. Карточка уходит из списка (если фильтр "только необработанные"). Вызов: `POST /admin/api/handover/{id}/process`

**[📋 Показать диалог]** — открывает модальное окно с полной перепиской. Вызов: `GET /admin/api/handover/{id}/messages`

### Фильтры

- По аккаунту — dropdown со списком аккаунтов (берёт из `applications.account_id` → `avito_accounts`)
- По результату — dropdown: Все / booking / interested / not_interested / alternative
- Только необработанные — checkbox (по умолчанию включён)

### Сортировка

По умолчанию — новые сверху (`created_at DESC`).

---

## 2. API-ЭНДПОИНТЫ ДЛЯ КАРТОЧЕК

Добавить в `api/admin.py`:

### GET /admin/api/handover
Список карточек с фильтрами:
```
Параметры:
  - limit (int, default=50)
  - offset (int, default=0)
  - account_id (int, optional)
  - result (str, optional) — booking/interested/not_interested/alternative
  - unprocessed_only (bool, default=true)

Ответ:
{
  "total": 42,
  "cards": [
    {
      "id": 9,
      "candidate_name": "Ларина Юлия",
      "candidate_phone": "79777031312",
      "candidate_city": "Артём",
      "candidate_metro": null,
      "candidate_age": 24,
      "vacancy_title": "Нажиматель кнопок",
      "result": "booking",
      "callback_slot": "11-13",
      "dialog_summary": "...",
      "messages_count": 13,
      "is_processed": false,
      "created_at": "2026-03-10T16:00:02",
      "account_name": "Хостел Ростов"
    }
  ]
}
```

Для получения `account_name` — JOIN через `applications.account_id → avito_accounts.account_name`. Связка: `handover_cards.application_id → applications.id → applications.account_id`.

### GET /admin/api/handover/{id}/messages
Полная переписка по карточке:
```
Ответ:
{
  "card_id": 9,
  "candidate_name": "Ларина Юлия",
  "messages": [
    {"sender": "ai", "text": "Добрый вечер! Меня зовут Елена...", "sent_at": "..."},
    {"sender": "candidate", "text": "Здравствуйте, мне 24 года...", "sent_at": "..."},
    ...
  ]
}
```

Источник: таблица `messages` по `chat_id` из связанного `application`.

### POST /admin/api/handover/{id}/process
Пометить как обработанную:
```python
card.is_processed = True
card.processed_at = datetime.utcnow()
await session.commit()
```

---

## 3. TELEGRAM-БОТ — УТРЕННЯЯ СВОДКА

### Конфиг (config.py)

```python
# config.py
telegram_bot_token: str = ""           # токен от @BotFather
telegram_group_id: str = ""            # ID группы с топиками (например: -1003560902940)
telegram_morning_hour: int = 9         # час отправки (по МСК)
telegram_morning_minute: int = 0
```

### БД — поле в avito_accounts

Добавить в `avito_accounts`:
```sql
ALTER TABLE avito_accounts ADD COLUMN telegram_topic_id INT NULL;
```

Обновить модель `AvitoAccount` в `models/db.py`:
```python
telegram_topic_id = Column(Integer, nullable=True)
```

Добавить поле `telegram_topic_id` в форму добавления/редактирования аккаунта в админке (`templates/admin.html`) и в API (`api/admin.py` — POST/PUT /admin/accounts).

Каждый аккаунт шлёт уведомления в свой топик группы. Если `telegram_topic_id` не заполнен — не отправлять для этого аккаунта.

### Новый файл: `services/telegram_notifier.py`

```python
"""Отправка карточек менеджерам в Telegram (в топики группы)."""
import httpx
from config import settings
from utils.logger import get_logger

log = get_logger(__name__)


async def send_telegram_message(topic_id: int, text: str, parse_mode: str = "HTML") -> bool:
    """Отправить сообщение в топик Telegram-группы."""
    if not settings.telegram_bot_token or not settings.telegram_group_id:
        log.warning("telegram_not_configured")
        return False
    
    try:
        api_url = f"https://api.telegram.org/bot{settings.telegram_bot_token}"
        payload = {
            "chat_id": settings.telegram_group_id,
            "message_thread_id": topic_id,
            "text": text,
            "parse_mode": parse_mode,
        }
        async with httpx.AsyncClient(timeout=15.0) as client:
            resp = await client.post(f"{api_url}/sendMessage", json=payload)
            resp.raise_for_status()
            return True
    except Exception as e:
        log.error("telegram_send_error", topic_id=topic_id, error=str(e))
        return False


def format_card_for_telegram(card) -> str:
    """Форматирует карточку в HTML для Telegram."""
    
    result_emoji = {
        "booking": "✅",
        "interested": "🟡",
        "not_interested": "⚪",
        "alternative": "🔵",
    }
    emoji = result_emoji.get(card.result, "❓")
    
    slot = f"\n🕐 Слот: {card.callback_slot}" if card.callback_slot else ""
    metro = f" (м. {card.candidate_metro})" if card.candidate_metro else ""
    age = f", {card.candidate_age} лет" if card.candidate_age else ""
    
    return (
        f"{emoji} <b>{card.candidate_name}{age}</b>\n"
        f"📞 <code>{card.candidate_phone}</code>\n"
        f"📍 {card.candidate_city or '—'}{metro}\n"
        f"💼 {card.vacancy_title}{slot}\n"
        f"\n{card.dialog_summary or ''}"
    )


async def send_morning_summary() -> int:
    """
    Утренняя сводка: все необработанные карточки, сгруппированные по аккаунтам.
    Каждый аккаунт получает сводку в свой топик.
    Вызывается scheduler'ом в 09:00.
    """
    if not settings.telegram_bot_token or not settings.telegram_group_id:
        log.warning("telegram_morning_skip_not_configured")
        return 0
    
    from sqlalchemy import select
    from models.db import AsyncSessionFactory, HandoverCard, Application, AvitoAccount
    
    # Получить все активные аккаунты с telegram_topic_id
    async with AsyncSessionFactory() as session:
        result = await session.execute(
            select(AvitoAccount).where(
                AvitoAccount.is_active == True,
                AvitoAccount.telegram_topic_id != None,
            )
        )
        accounts = result.scalars().all()
    
    if not accounts:
        log.warning("telegram_morning_no_accounts_with_topics")
        return 0
    
    total_sent = 0
    
    for account in accounts:
        # Карточки этого аккаунта
        async with AsyncSessionFactory() as session:
            result = await session.execute(
                select(HandoverCard)
                .join(Application, HandoverCard.application_id == Application.id)
                .where(
                    Application.account_id == account.id,
                    HandoverCard.is_processed == False,
                )
                .order_by(HandoverCard.created_at.desc())
            )
            cards = result.scalars().all()
        
        if not cards:
            continue
        
        # Заголовок
        header = f"📋 <b>Утренняя сводка — {len(cards)} карточек</b>\n{'─' * 30}"
        await send_telegram_message(account.telegram_topic_id, header)
        
        # Каждая карточка — отдельным сообщением
        for card in cards:
            text = format_card_for_telegram(card)
            success = await send_telegram_message(account.telegram_topic_id, text)
            if success:
                total_sent += 1
            import asyncio
            await asyncio.sleep(0.3)
        
        # Итог
        bookings = sum(1 for c in cards if c.result == 'booking')
        footer = f"{'─' * 30}\n📊 Всего: {len(cards)} | Записи: {bookings}"
        await send_telegram_message(account.telegram_topic_id, footer)
    
    log.info("telegram_morning_sent", total=total_sent)
    
    try:
        from utils.event_logger import log_event
        await log_event(None, "telegram_summary", f"Утренняя сводка: {total_sent} карточек отправлено")
    except Exception:
        pass
    
    return total_sent
```

### Запросы к Telegram API — через SOCKS5-прокси или напрямую?

Telegram API может быть заблокирован в РФ. Проверь с сервера:
```bash
curl -s https://api.telegram.org/bot<TOKEN>/getMe
```

Если не работает — нужно через прокси. В этом случае замени `httpx.AsyncClient(timeout=15.0)` на:
```python
httpx.AsyncClient(
    timeout=15.0,
    proxy=f"socks5://127.0.0.1:1080"  # тот же microsocks что для Claude API
)
```

### Scheduler в main.py

Добавь задачу:

```python
from services.telegram_notifier import send_morning_summary

scheduler.add_job(
    send_morning_summary,
    "cron",
    hour=settings.telegram_morning_hour,
    minute=settings.telegram_morning_minute,
    timezone=settings.tz,
    id="telegram_morning_summary",
    max_instances=1,
)
```

**ВАЖНО:** Таймзона должна быть московская (Europe/Moscow или UTC+3). Проверь что `settings.tz` — правильная. Если нет — используй `timezone="Europe/Moscow"` явно.

---

## 4. МОДАЛЬНОЕ ОКНО ДИАЛОГА (в admin.html)

При клике на "Показать диалог" — модалка с перепиской:

```
┌─────────────────────────────────────────┐
│  Диалог с Ларина Юлия            [✕]   │
├─────────────────────────────────────────┤
│                                         │
│  🤖 Елена (AI)           16:01         │
│  ┌──────────────────────────────┐      │
│  │ Добрый вечер! Меня зовут    │      │
│  │ Елена, я специалист...      │      │
│  └──────────────────────────────┘      │
│                                         │
│                    👤 Юлия     16:03    │
│        ┌──────────────────────────┐    │
│        │ Здравствуйте, мне 24    │    │
│        │ года, живу в Артёме     │    │
│        └──────────────────────────┘    │
│                                         │
│  🤖 Елена (AI)           16:04         │
│  ┌──────────────────────────────┐      │
│  │ Спасибо! Расскажу коротко   │      │
│  │ по вакансии...              │      │
│  └──────────────────────────────┘      │
│                                         │
└─────────────────────────────────────────┘
```

- AI-сообщения — слева, серый фон
- Сообщения кандидата — справа, синий/голубой фон
- Время — мелким шрифтом
- Прокрутка внутри модалки если много сообщений

---

## 5. ТЕСТОВЫЙ ЭНДПОИНТ ДЛЯ TELEGRAM

Для проверки без ожидания 09:00:

```
POST /admin/api/telegram/test-summary
POST /admin/api/telegram/test-summary?account_id=2  — только для конкретного аккаунта
```

Без `account_id` — вызывает `send_morning_summary()` для всех аккаунтов.
С `account_id` — отправляет только карточки этого аккаунта в его топик.

Защищён admin-авторизацией.

---

## ПОРЯДОК РАБОТЫ

1. Прочитай текущие `api/admin.py`, `templates/admin.html`, `models/db.py`, `services/handover.py`
2. Добавь API-эндпоинты для карточек в `api/admin.py`
3. Создай `services/telegram_notifier.py`
4. Обнови `templates/admin.html` — раздел "Карточки" + модалка диалога
5. Обнови `main.py` — scheduler для утренней сводки
6. Обнови `config.py` — telegram-настройки
7. Проверь импорты

---

## ПОСЛЕ ВЫПОЛНЕНИЯ — ОБЯЗАТЕЛЬНЫЙ ОТЧЁТ

Создай `CHANGELOG-handover-delivery.md` в корне проекта:

1. **Таблица изменений** — каждый файл, что изменено, зачем
2. **Новые файлы** — назначение
3. **Инструкция:**
   - SQL: `ALTER TABLE avito_accounts ADD COLUMN telegram_topic_id INT NULL;`
   - Что добавить в config.py (`telegram_bot_token`, `telegram_group_id`)
   - Как заполнить `telegram_topic_id` для аккаунтов (через админку или SQL)
   - Как проверить Telegram (тестовый эндпоинт)
   - URL раздела карточек в админке
4. **Проверка импортов:**
```bash
python -c "from services.telegram_notifier import send_morning_summary; print('Telegram OK')"
python -c "from api.admin import router; print('Admin API OK')"
```
