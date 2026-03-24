# TASK: Добавить аккаунты Авито только для синка вакансий (без автоответа)

**Приоритет:** Высокий
**Дата:** 2026-03-19

---

## 1. Контекст

Подключаем два новых аккаунта Авито. На первом этапе — только синк вакансий. Автоответ AI НЕ включаем, вебхуки НЕ регистрируем. Вакансии должны подтянуться в БД и Qdrant для поиска альтернатив.

---

## 2. Новые аккаунты

### Аккаунт 3 — "Avito 8к2"
- **client_id:** `Np7wkUm_QJWnnklJhRN3`
- **client_secret:** `bLkLSZf1p-P__dMyE02MyeRY2yrPB5QJZkaec-vH`
- **avito_user_id:** `382795941`

### Аккаунт 4 — "Avito 6к2"
- **client_id:** `vqqYGCrM-evNERtPpUQz`
- **client_secret:** `GksHQXbEEpZY6cf_xltnDz6aZIaF-_XD06AJ84YT`
- **avito_user_id:** `376569486`

---

## 3. Что нужно сделать

### 3.1. Миграция: добавить поле `ai_enabled` в `avito_accounts`

Файл: `migrations/007_ai_enabled.sql`

```sql
-- Добавляем флаг ai_enabled — определяет, отвечает ли AI-бот на этом аккаунте
-- Для существующих аккаунтов (id=1, id=2) — TRUE (уже работают с ботом)
-- Для новых — FALSE (только вакансии)

ALTER TABLE avito_accounts ADD COLUMN ai_enabled BOOLEAN DEFAULT FALSE AFTER is_active;

-- Включаем AI для существующих аккаунтов
UPDATE avito_accounts SET ai_enabled = TRUE WHERE id IN (1, 2);
```

### 3.2. Обновить модель `AvitoAccount` в `models/db.py`

Добавить поле после `is_active`:

```python
ai_enabled          = Column(Boolean, default=False)
```

### 3.3. Обновить `api/webhooks.py` — проверка `ai_enabled`

В функции `_process_new_application`, перед созданием AI-сессии, проверять флаг:

```python
# Текущий код (примерно строка где is_night_window):
if is_night_window() and chat:
    application.status = "ai_active"
    ...

# Заменить на:
account = await session.get(AvitoAccount, account_id)
if is_night_window() and chat and account and account.ai_enabled:
    application.status = "ai_active"
    ...
```

Аналогично в конце функции, перед вызовом `process_new_application`:

```python
# Текущий код:
if is_night_window() and chat:
    from services.ai_agent import process_new_application
    await asyncio.sleep(2)
    await process_new_application(application.id)

# Заменить на:
if is_night_window() and chat and account and account.ai_enabled:
    from services.ai_agent import process_new_application
    await asyncio.sleep(2)
    await process_new_application(application.id)
```

**Важно:** переменная `account` уже может быть доступна через `_resolve_account_for_application` — нужно подтянуть объект аккаунта внутри сессии.

### 3.4. Обновить `workers/incoming_processor.py` — проверка `ai_enabled`

В `handle_incoming` (или где определяется, нужно ли обрабатывать входящее сообщение через AI), добавить проверку:

```python
# Перед вызовом ai_agent.process_incoming_message:
# Получить account через chat -> application -> account
account = await session.get(AvitoAccount, chat.account_id)
if not account or not account.ai_enabled:
    log.debug("ai_disabled_for_account", account_id=chat.account_id)
    return
```

### 3.5. Обновить `api/admin.py` — отображение и управление `ai_enabled`

В `list_accounts` добавить поле в ответ:

```python
"ai_enabled": a.ai_enabled,
```

В `CreateAccountRequest` добавить:

```python
ai_enabled: bool = False
```

В `create_account` — при создании аккаунта:

```python
account = AvitoAccount(
    ...
    ai_enabled=req.ai_enabled,
)
```

**Убрать автоматическую регистрацию вебхуков** при создании аккаунта, если `ai_enabled=False`:

```python
# Текущий код:
if token_ok:
    try:
        webhooks_ok = await _register_webhooks_for_account(account_id)
    ...

# Заменить на:
if token_ok and req.ai_enabled:
    try:
        webhooks_ok = await _register_webhooks_for_account(account_id)
    ...
```

### 3.6. Обновить админку `templates/admin.html`

В модальное окно создания/редактирования аккаунта добавить чекбокс "AI-бот включён". В списке аккаунтов — показать статус (бейдж "AI" рядом с именем или отдельная колонка).

### 3.7. Добавить два аккаунта в БД

Файл: `migrations/008_add_vacancy_accounts.sql`

```sql
-- Добавляем два аккаунта только для синка вакансий (ai_enabled = FALSE)

INSERT INTO avito_accounts (client_id, client_secret, avito_user_id, account_name, is_active, ai_enabled, webhook_registered)
VALUES
    ('Np7wkUm_QJWnnklJhRN3', 'bLkLSZf1p-P__dMyE02MyeRY2yrPB5QJZkaec-vH', '382795941', 'Avito 8к2', 1, 0, 0),
    ('vqqYGCrM-evNERtPpUQz', 'GksHQXbEEpZY6cf_xltnDz6aZIaF-_XD06AJ84YT', '376569486', 'Avito 6к2', 1, 0, 0);
```

### 3.8. Проверить что синк вакансий подхватит новые аккаунты

В `main.py` → `_refresh_vacancies_job` уже итерирует по всем `is_active=True`. Новые аккаунты `is_active=True` → вакансии подтянутся автоматически. **Проверить что `sync_all_vacancies` корректно получает токен для новых аккаунтов** (через `avito_auth.get_valid_token`).

---

## 4. Что НЕ нужно делать

- НЕ регистрировать вебхуки (messenger/job) для этих аккаунтов
- НЕ подключать Telegram-топики
- НЕ включать AI-автоответ
- НЕ менять скрипты переключения Битрикс ↔ наш (`webhook_enable_ours.py` / `webhook_enable_bitrix.py`) — они для аккаунта 2

---

## 5. Порядок применения

1. Миграция `007_ai_enabled.sql` — добавить поле
2. Миграция `008_add_vacancy_accounts.sql` — добавить аккаунты
3. Обновить `models/db.py` — поле `ai_enabled`
4. Обновить `api/webhooks.py` — проверка `ai_enabled`
5. Обновить `workers/incoming_processor.py` — проверка `ai_enabled`
6. Обновить `api/admin.py` — API + создание аккаунтов
7. Обновить `templates/admin.html` — чекбокс в UI
8. Перезапустить сервис: `systemctl restart k24-crm-worker`
9. Дождаться первого синка вакансий (до 30 мин) или вызвать вручную

---

## 6. Тестирование

1. **Проверить токены:** в логах должны появиться записи `token_refreshed` для новых аккаунтов
2. **Проверить синк вакансий:** в логах `vacancies_sync_account_done account_id=3` и `account_id=4`
3. **Проверить что AI НЕ отвечает:** отправить тестовое сообщение на аккаунт 3 — бот не должен реагировать
4. **Проверить админку:** новые аккаунты видны, у них нет бейджа "AI", вебхуки не зарегистрированы

---

## 7. Файлы для изменения

| Файл | Действие |
|------|----------|
| `migrations/007_ai_enabled.sql` | Создать (новый) |
| `migrations/008_add_vacancy_accounts.sql` | Создать (новый) |
| `models/db.py` | Добавить поле `ai_enabled` в `AvitoAccount` |
| `api/webhooks.py` | Проверка `ai_enabled` перед AI-сессией |
| `workers/incoming_processor.py` | Проверка `ai_enabled` перед обработкой AI |
| `api/admin.py` | Поле в API, условие на вебхуки |
| `templates/admin.html` | Чекбокс и отображение |
