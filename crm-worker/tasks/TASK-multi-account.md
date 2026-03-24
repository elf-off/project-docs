# ЗАДАЧА: Переделать CRM-воркер на мультиаккаунт

## Контекст

Проект: CRM-система с AI-агентом для обработки откликов на вакансии Avito.
Путь: `/opt/openai/crm-worker/`
Стек: Python 3.11, FastAPI, SQLAlchemy (async), MySQL (MariaDB), Qdrant, Claude API.

Сейчас сервис работает с ОДНИМ аккаунтом Avito (данные захардкожены или берутся из единственной записи в `avito_accounts`). Нужно переделать на поддержку НЕСКОЛЬКИХ аккаунтов одновременно.

---

## ЧТО НУЖНО СДЕЛАТЬ

### 1. Таблица `avito_accounts` — добавить поля

Прочитай текущую модель в `models/db.py`. Убедись, что таблица `avito_accounts` содержит:

```
id                  INT PRIMARY KEY AUTO_INCREMENT
name                VARCHAR(100)       -- человекочитаемое название ("Кадры 24 Москва")
client_id           VARCHAR(100)       -- Avito OAuth client_id
client_secret       VARCHAR(255)       -- Avito OAuth client_secret
avito_user_id       BIGINT             -- ID основного аккаунта компании на Avito
access_token        TEXT               -- текущий токен
token_expires_at    DATETIME           -- когда протухнет
refresh_token       TEXT               -- если есть
is_active           BOOLEAN DEFAULT 1  -- включён/выключен
webhook_registered  BOOLEAN DEFAULT 0  -- зарегистрирован ли вебхук
created_at          DATETIME
updated_at          DATETIME
```

Если каких-то полей нет — добавь через ALTER TABLE (создай файл `migrations/003_multi_account.sql`). Модель в `db.py` тоже обнови.

### 2. Маршрутизация вебхуков по `avito_user_id`

Файл: `api/webhooks.py`

Сейчас при приёме вебхука аккаунт определяется жёстко. Нужно:

**Для messenger-вебхуков:**
- В payload есть `user_id` — это `avito_user_id` аккаунта-получателя
- По нему ищем `avito_accounts WHERE avito_user_id = {user_id} AND is_active = 1`
- Если не найден — логируем warning и отдаём 200 (чтобы Avito не ретраил)

**Для job-вебхуков (отклики):**
- В payload отклика нет явного `user_id`. Но есть `vacancy_id`
- Вариант 1: по `vacancy_id` найти вакансию → аккаунт (если есть таблица vacancies с account_id)
- Вариант 2: пробовать получить данные отклика по каждому активному аккаунту (fallback)
- Вариант 3: добавить поле `account_id` в `applications` и связать при создании

**Рекомендация:** Используй вариант 1 + 2 как fallback. Если в `vacancies` есть `account_id` — бери оттуда. Если нет — перебирай активные аккаунты.

Прокинь `account` (объект или id) дальше по цепочке: в `incoming_processor`, `ai_agent`, `avito_messenger`, `avito_applications`.

### 3. Авторизация — привязка к аккаунту

Файл: `services/avito_auth.py`

Сейчас, вероятно, `client_id` и `client_secret` берутся из `config.py` или из единственной записи. Переделай:

- Функция `get_token(account_id)` или `get_token(account: AvitoAccount)` — получает/обновляет токен для КОНКРЕТНОГО аккаунта
- Токен кэшируется в БД (поля `access_token`, `token_expires_at` в `avito_accounts`)
- Если токен протух — обновить через `client_credentials` grant с `client_id` и `client_secret` этого аккаунта

### 4. Token Refresher — для всех аккаунтов

Файл: `workers/token_refresher.py`

Переделай чтобы обновлял токены для ВСЕХ активных аккаунтов:

```python
accounts = await db.execute(select(AvitoAccount).where(AvitoAccount.is_active == True))
for account in accounts.scalars():
    await refresh_token_for_account(account)
```

### 5. Messenger — привязка к аккаунту

Файл: `services/avito_messenger.py`

Все функции отправки сообщений должны принимать `account` (или `account_id`) и использовать:
- `account.avito_user_id` в URL (`/messenger/v1/accounts/{user_id}/...`)
- `account.access_token` для авторизации

### 6. Applications API — привязка к аккаунту

Файл: `services/avito_applications.py`

Аналогично — при запросе данных отклика использовать токен конкретного аккаунта.

### 7. Фильтрация своих сообщений

Файл: `api/webhooks.py`

Сейчас фильтр: `author_id == avito_user_id` (одного аккаунта). Переделай:
- Получить список всех `avito_user_id` из активных аккаунтов
- Пропускать сообщения где `author_id` совпадает с ЛЮБЫМ из наших аккаунтов

### 8. AI Agent — передача контекста аккаунта

Файл: `services/ai_agent.py`

При вызове AI-агента нужно знать, от какого аккаунта отвечать. Прокинь `account_id` через:
- `ai_sessions` (добавь поле `account_id` если нет)
- Или определяй по `application.account_id` → `account`

### 9. Таблица `applications` — привязка к аккаунту

Добавь поле `account_id INT` (FK на `avito_accounts.id`) если его нет. При создании application — записывай, от какого аккаунта пришёл отклик.

### 10. Утилита регистрации вебхуков

Создай CLI-команду или API-эндпоинт для регистрации вебхуков на новый аккаунт:

```python
# POST /admin/accounts/{account_id}/register-webhooks
# Или CLI: python -m utils.register_webhooks --account-id 2

async def register_webhooks(account: AvitoAccount):
    token = await get_token(account)
    
    # Messenger webhook
    await httpx.post(
        "https://api.avito.ru/messenger/v3/webhook",
        headers={"Authorization": f"Bearer {token}"},
        json={"url": "https://babito.kadry-24.ru/webhooks/avito"}
    )
    
    # Job applications webhook
    await httpx.put(
        "https://api.avito.ru/job/v1/applications/webhook",
        headers={"Authorization": f"Bearer {token}"},
        json={"url": "https://babito.kadry-24.ru/webhooks/avito"}
    )
    
    account.webhook_registered = True
    await db.commit()
```

### 11. Эндпоинт добавления нового аккаунта

Создай `POST /admin/accounts` для добавления нового аккаунта:

```python
# Принимает: name, client_id, client_secret, avito_user_id
# Действия:
#   1. Сохраняет в avito_accounts
#   2. Пробует получить токен (проверка credentials)
#   3. Регистрирует вебхуки
#   4. Возвращает результат
```

Защити admin-эндпоинты простым токеном из `.env` (`ADMIN_TOKEN`).

---

## ВАЖНЫЕ ПРАВИЛА

1. **НЕ ломай текущую работу.** Старый аккаунт (id=1) должен продолжать работать. Все изменения — обратно совместимы.

2. **Миграции — отдельный SQL-файл.** Создай `migrations/003_multi_account.sql` со всеми ALTER TABLE. Не делай автомиграции.

3. **Логирование.** Во все логи добавь `account_id` или `account_name`, чтобы в логах было видно, от какого аккаунта идёт обработка.

4. **config.py** — убери `client_id`, `client_secret`, `avito_user_id` из конфига, если они там захардкожены. Всё должно браться из БД.

5. **Не трогай:** промпты (`prompts/`), AI-логику диалога (flow), Claude API обёртку. Меняй только "обвязку" вокруг аккаунтов.

---

## ПОРЯДОК РАБОТЫ

1. Сначала прочитай ВСЕ файлы проекта, пойми текущую архитектуру
2. Создай SQL-миграцию
3. Обнови `models/db.py`
4. Обнови `services/avito_auth.py` — мультиаккаунт
5. Обнови `workers/token_refresher.py` — цикл по аккаунтам
6. Обнови `api/webhooks.py` — маршрутизация по user_id
7. Обнови `services/avito_messenger.py` — account в параметрах
8. Обнови `services/avito_applications.py` — account в параметрах
9. Обнови `services/ai_agent.py` — прокидывание account
10. Создай admin-эндпоинты (добавление аккаунта, регистрация вебхуков)
11. Протестируй что старый аккаунт не сломался

---

## ПОСЛЕ ВЫПОЛНЕНИЯ

Покажи:
- SQL-миграцию (полный текст)
- Список изменённых файлов с кратким описанием что изменилось
- Команды для добавления нового аккаунта
