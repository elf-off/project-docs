# SPEC: Интеграция Bitrix24 Lead Creation

## Цель

Добавить в бота обработку личных сообщений с маркером `#Битрикс`. Бот должен парсить ФИО + телефон + город/регион из сообщения, создавать Lead в Битрикс24 через REST API и отправлять результат пользователю.

---

## Формат входящего сообщения

```
#Битрикс
ФИО (обязательно)
Телефон (обязательно)
Город/регион (опционально)
Любая доп. информация (опционально)
```

Примеры:

```
#Битрикс
Петров Иван Сергеевич
+79001234567
Москва
```

```
#Битрикс
Сидорова Мария
89161234567
Краснодар
Интересуется вакансией
```

```
#Битрикс
Зовут Алексей, фамилия Козлов, тел 8-900-123-45-67, из Самары
```

---

## JSON для Bitrix24 REST API

Endpoint: `POST {BITRIX_WEBHOOK_URL}/crm.lead.add.json`

```json
{
  "fields": {
    "TITLE": "Отклик из соц.сетей",
    "NAME": "Иван",
    "LAST_NAME": "Петров",
    "PHONE": [{"VALUE": "+79001234567", "VALUE_TYPE": "WORK"}],
    "SOURCE_ID": "WEB",
    "ASSIGNED_BY_ID": 565282,
    "COMMENTS": "Город: Москва. Интересуется вакансией"
  }
}
```

Поле `COMMENTS` собирается из города и всего текста, идущего после телефона.
Поле `SECOND_NAME` (отчество) — отправлять если есть.
`ASSIGNED_BY_ID` берётся из настроек (`.env`).

---

## Что нужно сделать

### 1. Новые настройки в `.env` и `app/config.py`

Добавить в `.env.example` и `.env`:
```
BITRIX_WEBHOOK_URL=https://ваш-домен.bitrix24.ru/rest/1/ваш-секрет
BITRIX_ASSIGNED_BY_ID=565282
```

Добавить в класс `Settings` в `app/config.py`:
```python
bitrix_webhook_url: str = ""
bitrix_assigned_by_id: int = 565282
```

### 2. SQL-миграция: `migrations/006_bitrix_leads.sql`

Создать таблицу для логирования создания лидов:

```sql
CREATE TABLE IF NOT EXISTS bitrix_leads (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tg_user_id BIGINT NOT NULL,
    first_name VARCHAR(255) NOT NULL,
    last_name VARCHAR(255) NOT NULL,
    middle_name VARCHAR(255) DEFAULT NULL,
    phone VARCHAR(20) NOT NULL,
    comments TEXT DEFAULT NULL,
    raw_text TEXT DEFAULT NULL,
    bitrix_lead_id INT DEFAULT NULL COMMENT 'ID лида в Битрикс24',
    success BOOLEAN DEFAULT FALSE,
    error_message TEXT DEFAULT NULL,
    correlation_id VARCHAR(36) DEFAULT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_tg_user_id (tg_user_id),
    INDEX idx_bitrix_lead_id (bitrix_lead_id),
    INDEX idx_success (success),
    INDEX idx_created_at (created_at),
    INDEX idx_correlation_id (correlation_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 3. SQLAlchemy модель: `app/models/bitrix_lead.py`

Создать модель `BitrixLead` для таблицы `bitrix_leads` по аналогии с другими моделями в `app/models/`.
Поля: `id`, `tg_user_id`, `first_name`, `last_name`, `middle_name`, `phone`, `comments`, `raw_text`, `bitrix_lead_id`, `success`, `error_message`, `correlation_id`, `created_at`.

Добавить импорт в `app/models/__init__.py`.

### 4. Сервис: `app/services/bitrix_service.py`

Создать `BitrixService` по аналогии с другими сервисами проекта.

**Конструктор**: принимает `db_session`, `correlation_id`. Использует `structlog` для логирования с `correlation_id` (как все остальные сервисы).

**Статический метод `is_bitrix_message(text: str) -> bool`**: проверяет, начинается ли первая строка текста с `#Битрикс` (регистронезависимо).

**Метод `process_message(message) -> dict`**: основной поток обработки:

1. Убрать маркер `#Битрикс` из текста
2. Попробовать regex-парсинг для стандартного формата
3. Если regex не справился → fallback на AI-парсинг через `AIClient` (Claude)
4. Валидация: обязательны `first_name`, `last_name`, `phone` (формат `+7XXXXXXXXXX`)
5. Отправка в Bitrix24 REST API через `httpx.AsyncClient` (async!)
6. Сохранение в БД (`bitrix_leads`)
7. Вернуть `{"success": bool, "message": str, "lead_id": int|None}`

**Regex-парсинг** (быстрый путь):
- Строки до телефона = ФИО (Фамилия Имя [Отчество])
- Строка с телефоном = телефон (нормализовать: убрать пробелы/скобки/дефисы, `8...` → `+7...`)
- Строки после телефона = город + доп. информация → поле `COMMENTS`

**AI-парсинг** (fallback): отправить текст в `AIClient.analyze()` с system prompt который просит вернуть JSON:
```json
{"last_name": "...", "first_name": "...", "middle_name": "...", "phone": "+7...", "city": "...", "extra_info": "..."}
```
или `{"error": "..."}` если данных не хватает.

**Нормализация телефона**: `89001234567` → `+79001234567`, убрать все не-цифры.

**Отправка в Bitrix24**:
- URL: `{settings.bitrix_webhook_url}/crm.lead.add.json`
- Метод: POST JSON
- Timeout: 30 сек
- Обработать ответ: `result` → lead_id, `error` → ошибка
- Использовать `httpx.AsyncClient` (НЕ requests!)

**Ответы пользователю**:
- Успех: `✅ Лид создан в Битрикс24!\n👤 Фамилия Имя\n📞 Телефон\n🏙 Город\n🆔 Lead ID: XXX`
- Ошибка парсинга: `❌ Не удалось разобрать сообщение` + формат-подсказка
- Ошибка API: `❌ Ошибка при создании лида в Битрикс24`
- Пустое сообщение: `❌ Пустое сообщение` + формат-подсказка

### 5. Интеграция в `app/services/message_router.py`

Добавить обработку **ДО** dispatch по `ctx.mode`, потому что личные сообщения могут не иметь записи в таблице `chats`.

В методе `process()` (или как называется главный метод обработки update):

```python
# В начале обработки, до resolve ChatContext:
if message and message.chat.type == "private":
    text = message.text or ""
    if BitrixService.is_bitrix_message(text):
        bitrix_svc = BitrixService(db_session, correlation_id)
        result = await bitrix_svc.process_message(message)
        await bot.send_message(chat_id=message.chat.id, text=result["message"])
        return
```

Адаптировать под реальную структуру MessageRouter — посмотреть как там отправляются сообщения (через NotificationService или напрямую через Bot), как получается db_session, и использовать тот же паттерн.

**ВАЖНО**: `import BitrixService` добавить вверху файла.

---

## Ключевые требования

1. **Только async** — использовать `httpx.AsyncClient`, НЕ `requests`. Это критично для архитектуры проекта (см. паттерн Async Everything).
2. **Correlation ID** — передавать через весь стек, логировать в structlog, сохранять в БД.
3. **Логирование** — `structlog.get_logger()` с `.bind(correlation_id=..., service="bitrix")`.
4. **Только личные сообщения** — `message.chat.type == "private"`. В групповых чатах `#Битрикс` НЕ обрабатывать.
5. **Graceful errors** — при любой ошибке (AI, Bitrix API, парсинг) отправлять понятное сообщение пользователю, логировать ошибку, НЕ падать.
6. **Стиль кода** — следовать паттернам остальных сервисов: конструктор `(db_session, correlation_id)`, structlog, async/await.

---

## Файлы для создания/изменения

| Действие | Файл | Описание |
|----------|------|----------|
| CREATE | `migrations/006_bitrix_leads.sql` | SQL-миграция |
| CREATE | `app/models/bitrix_lead.py` | SQLAlchemy модель |
| CREATE | `app/services/bitrix_service.py` | Основной сервис |
| EDIT | `app/models/__init__.py` | Добавить `from .bitrix_lead import BitrixLead` |
| EDIT | `app/config.py` | Добавить `bitrix_webhook_url`, `bitrix_assigned_by_id` |
| EDIT | `app/services/message_router.py` | Добавить вызов BitrixService для DM с #Битрикс |
| EDIT | `.env.example` | Добавить `BITRIX_WEBHOOK_URL`, `BITRIX_ASSIGNED_BY_ID` |

---

## Проверка после внедрения

1. `python3 -m py_compile app/services/bitrix_service.py` — синтаксис
2. `python3 -m py_compile app/models/bitrix_lead.py` — синтаксис
3. `python3 -m py_compile app/services/message_router.py` — синтаксис после правок
4. Применить миграцию: `mysql -u root -p 2_kadry_4_ethics_bot < migrations/006_bitrix_leads.sql`
5. Перезапуск: `sudo systemctl restart ethics-bot`
6. Тест в Telegram: отправить боту в личку `#Битрикс\nТестов Тест\n+79001234567\nМосква`
