# ЗАДАЧА: Веб-админка для CRM-воркера

## Контекст

Проект: `/opt/openai/crm-worker/`
Стек: Python 3.11, FastAPI, SQLAlchemy (async), MySQL.
Уже есть: `api/admin.py` с REST-эндпоинтами для управления аккаунтами (GET/POST /admin/accounts, register-webhooks, toggle).
Нужно: добавить веб-интерфейс (HTML-админку) для управления сервисом через браузер.

---

## АРХИТЕКТУРА

Всё в рамках существующего FastAPI-приложения. Без дополнительных фреймворков.

Структура новых файлов:
```
/opt/openai/crm-worker/
├── api/
│   ├── admin.py            # уже есть — REST API (НЕ ТРОГАТЬ, только дополнить если нужно)
│   └── admin_web.py        # NEW — роуты для веб-интерфейса
├── templates/
│   ├── login.html          # страница логина
│   └── admin.html          # основная админка (SPA — всё в одном файле)
└── main.py                 # подключить admin_web router + Jinja2
```

### Зависимости

Добавить в `requirements.txt`:
```
jinja2
python-multipart
```

В `main.py`:
```python
from fastapi.templating import Jinja2Templates
from fastapi.staticfiles import StaticFiles  # если понадобится
```

---

## 1. АВТОРИЗАЦИЯ — сессия через cookie

### Конфиг (.env)
```
ADMIN_LOGIN=admin
ADMIN_PASSWORD=надёжный-пароль-тут
ADMIN_SECRET_KEY=рандомная-строка-для-подписи-cookie
```

Добавь в `config.py`:
```python
admin_login: str = ""
admin_password: str = ""
admin_secret_key: str = "change-me-in-production"
```

### Логика

- `GET /admin/login` — отдаёт `login.html` с формой (логин + пароль)
- `POST /admin/login` — проверяет credentials, ставит signed cookie `admin_session` (itsdangerous или просто HMAC от timestamp+secret), редирект на `/admin/`
- `GET /admin/logout` — удаляет cookie, редирект на логин
- Все остальные `/admin/*` GET-роуты — проверяют cookie. Нет cookie или невалидный → редирект на `/admin/login`
- REST API (`/admin/accounts`, `/admin/accounts/{id}/...`) — тоже должны принимать и cookie, и `Authorization: Bearer ADMIN_TOKEN`. Оба способа валидны.

Реализация подписи cookie — простая: `value = f"{timestamp}:{hmac_sha256(timestamp, secret_key)}"`. Проверка: timestamp не старше 24 часов, HMAC совпадает.

НЕ использовать сторонние библиотеки для сессий. Просто signed cookie.

---

## 2. СТРАНИЦА АДМИНКИ — `templates/admin.html`

Одностраничное приложение. Весь HTML + CSS + JS в одном файле. Дизайн — чистый, минималистичный, без фреймворков (можно использовать только встроенный CSS).

### Визуальная структура

```
┌─────────────────────────────────────────────────────────┐
│  CRM Avito Admin                          [Выйти]      │
├────────────┬────────────────────────────────────────────┤
│            │                                            │
│ Аккаунты   │   (содержимое выбранного раздела)          │
│ Логи       │                                            │
│            │                                            │
├────────────┴────────────────────────────────────────────┤
│  footer: версия, время сервера                          │
└─────────────────────────────────────────────────────────┘
```

Левое меню — два раздела: "Аккаунты" и "Логи". Переключение без перезагрузки страницы (JS).

---

### 2.1. Раздел "Аккаунты"

**Таблица аккаунтов:**

| # | Название | client_id | avito_user_id | Токен | Вебхуки | Активен | Действия |
|---|----------|-----------|---------------|-------|---------|---------|----------|
| 1 | Хостел Ростов | xu-r3... | 197052425 | ✅ до 14:30 | ✅ | ✅ | [Ред.] [Вкл/Выкл] |
| 2 | Кадры 24 СПб | ab-c4... | 123456789 | ❌ нет | ❌ | ✅ | [Ред.] [Вкл/Выкл] |

Столбец "Токен":
- ✅ зелёный + "до HH:MM" — если `token_expires_at` в будущем
- ⚠️ жёлтый + "истекает" — если до истечения < 1 часа
- ❌ красный + "нет" — если токена нет или протух

Столбец "Вебхуки": ✅/❌ по полю `webhook_registered`

Столбец "Активен": toggle-переключатель (вызывает POST /admin/accounts/{id}/toggle)

**Кнопки:**
- [+ Добавить аккаунт] — открывает модальное окно / форму
- [Обновить вебхуки] рядом с каждым аккаунтом — POST /admin/accounts/{id}/register-webhooks

**Форма добавления/редактирования аккаунта (модальное окно):**

Поля:
- Название (text) — обязательное
- Client ID (text) — обязательное
- Client Secret (password) — обязательное при создании, при редактировании можно оставить пустым (не менять)
- Avito User ID (number) — обязательное

Кнопки: [Сохранить] [Отмена]

При сохранении:
- POST /admin/accounts (создание) или PUT /admin/accounts/{id} (редактирование)
- После успеха — обновить таблицу
- При ошибке — показать сообщение

---

### 2.2. Раздел "Логи"

Показывает последние события из БД. Для этого нужен API-эндпоинт.

**Новый эндпоинт** (добавить в `api/admin.py`):

```
GET /admin/api/events?limit=50&account_id=&type=
```

Возвращает JSON со списком последних событий. Источники данных:

1. **Входящие вебхуки** — из таблицы `applications` (последние отклики):
   ```
   {time, type: "webhook", account: "Хостел Ростов", message: "Новый отклик от Иван И., vacancy_id=12345"}
   ```

2. **Отправленные сообщения** — из таблицы `messages` (последние AI-ответы):
   ```
   {time, type: "message_sent", account: "Хостел Ростов", message: "Приветствие отправлено в чат abc123"}
   ```

3. **Ошибки** — из таблицы `ai_sessions` где status=failed:
   ```
   {time, type: "error", account: "Кадры 24", message: "AI сессия failed: Claude API timeout"}
   ```

4. **Обновления токенов** — если есть лог в БД или можно добавить простую таблицу `event_log`:
   ```sql
   CREATE TABLE event_log (
       id INT PRIMARY KEY AUTO_INCREMENT,
       account_id INT,
       event_type VARCHAR(50),   -- 'token_refresh', 'webhook_register', 'error', 'info'
       message TEXT,
       created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
       FOREIGN KEY (account_id) REFERENCES avito_accounts(id)
   );
   ```

**Интерфейс логов:**

```
[Фильтр по аккаунту: ▼ Все]  [Фильтр по типу: ▼ Все]  [🔄 Обновить]  [Auto-refresh: ✅]

14:32:15  ✉️  Хостел Ростов    Приветствие отправлено в чат abc123
14:30:02  📥  Хостел Ростов    Новый отклик от Иван И., вакансия "Комплектовщик"  
14:28:55  🔑  Кадры 24 СПб     Токен обновлён, истекает в 15:28
14:25:11  ❌  Кадры 24 СПб     Claude API timeout, retry 1/3
```

- Иконки по типу события (emoji в тексте, не картинки)
- Auto-refresh — если включён, подгружать новые события каждые 10 секунд
- Фильтры работают через JS (фильтрация на клиенте из загруженных данных)
- Показывать последние 100 событий, кнопка "Загрузить ещё"

---

## 3. ДОБАВИТЬ ЛОГИРОВАНИЕ СОБЫТИЙ

Чтобы логи в админке были полезными, нужно писать ключевые события в таблицу `event_log`.

### SQL-миграция (добавить в `migrations/004_event_log.sql`):

```sql
CREATE TABLE IF NOT EXISTS event_log (
    id INT PRIMARY KEY AUTO_INCREMENT,
    account_id INT NULL,
    event_type VARCHAR(50) NOT NULL,
    message TEXT,
    details JSON NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES avito_accounts(id)
);

CREATE INDEX idx_event_log_created ON event_log(created_at DESC);
CREATE INDEX idx_event_log_account ON event_log(account_id, created_at DESC);
```

### Хелпер для записи:

Создай `utils/event_logger.py`:

```python
async def log_event(account_id: int | None, event_type: str, message: str, details: dict | None = None):
    """Записать событие в event_log для отображения в админке."""
    async with get_session() as session:
        event = EventLog(
            account_id=account_id,
            event_type=event_type,
            message=message,
            details=details
        )
        session.add(event)
        await session.commit()
```

### Где вызывать:

Добавь вызовы `log_event()` в ключевых местах:

| Место | event_type | Пример message |
|-------|-----------|----------------|
| `webhooks.py` — новый отклик | `webhook_application` | "Новый отклик {apply_id}, вакансия {vacancy_id}" |
| `webhooks.py` — входящее сообщение | `webhook_message` | "Входящее сообщение в чат {chat_id}" |
| `avito_messenger.py` — сообщение отправлено | `message_sent` | "Сообщение отправлено в чат {chat_id}" |
| `avito_auth.py` — токен обновлён | `token_refreshed` | "Токен обновлён, истекает {expires_at}" |
| `avito_auth.py` — ошибка обновления токена | `token_error` | "Ошибка обновления токена: {error}" |
| `ai_agent.py` — сессия завершена | `session_completed` | "Сессия {id} завершена, блок {block}" |
| `ai_agent.py` — ошибка | `session_error` | "Ошибка AI сессии {id}: {error}" |
| `admin.py` — аккаунт создан/изменён | `account_updated` | "Аккаунт '{name}' создан/обновлён" |
| `admin.py` — вебхуки зарегистрированы | `webhook_registered` | "Вебхуки зарегистрированы для '{name}'" |

**Важно:** Не ломай основной flow если event_log недоступен. Оберни в try/except — запись в лог не должна вызывать ошибку основного процесса.

### Автоочистка старых логов

В `main.py` добавь периодическую задачу (раз в сутки):
```python
DELETE FROM event_log WHERE created_at < NOW() - INTERVAL 30 DAY
```

---

## 4. ДОПОЛНИТЬ СУЩЕСТВУЮЩИЙ admin.py

Добавь в `api/admin.py` недостающие эндпоинты:

### PUT /admin/accounts/{id}
Редактирование аккаунта. Принимает те же поля что POST, но:
- `client_secret` опционален (если пустой — не менять)
- Возвращает обновлённый аккаунт

### GET /admin/api/events
Лог событий для админки:
```python
@router.get("/api/events")
async def get_events(limit: int = 100, account_id: int = None, event_type: str = None):
    query = select(EventLog).order_by(EventLog.created_at.desc()).limit(limit)
    if account_id:
        query = query.where(EventLog.account_id == account_id)
    if event_type:
        query = query.where(EventLog.event_type == event_type)
    ...
```

### GET /admin/api/stats
Краткая статистика для шапки или дашборда:
```json
{
    "total_accounts": 2,
    "active_accounts": 2,
    "applications_today": 15,
    "messages_today": 42,
    "errors_today": 1
}
```

### Авторизация

Все `/admin/*` эндпоинты (и API, и веб) должны проверять авторизацию. Сделай dependency:

```python
async def verify_admin(request: Request):
    # Проверяем cookie ИЛИ Bearer token
    cookie = request.cookies.get("admin_session")
    if cookie and validate_cookie(cookie):
        return True
    
    auth = request.headers.get("Authorization", "")
    if auth == f"Bearer {settings.admin_token}":
        return True
    
    raise HTTPException(401)
```

---

## 5. СТРАНИЦА ЛОГИНА — `templates/login.html`

Минимальная страница:
- Центрированная форма на белом фоне
- Логотип или заголовок "CRM Avito Admin"
- Поля: Логин, Пароль
- Кнопка "Войти"
- При ошибке — красное сообщение под формой
- После успеха — редирект на `/admin/`

---

## ДИЗАЙН (для обоих HTML)

**Цветовая схема:**
- Фон: #f5f5f5
- Боковое меню: #1a1a2e (тёмно-синий)
- Акценты: #0f3460
- Кнопки: #16a34a (зелёный — создать), #2563eb (синий — действие), #dc2626 (красный — удалить/ошибка)
- Карточки: белый фон, border-radius: 8px, лёгкая тень

**Типографика:**
- font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif
- Размер таблицы: 14px
- Заголовки: 600 weight

**Компоненты:**
- Таблицы: полная ширина, striped rows, hover подсветка
- Кнопки: 8px padding, border-radius: 6px, transition на hover
- Модальное окно: overlay тёмный, белая карточка по центру
- Уведомления (toast): правый верхний угол, автоскрытие через 3 сек
- Toggle: CSS-only переключатель для is_active

Всё БЕЗ внешних зависимостей — чистый HTML + CSS + vanilla JS.

---

## NGINX

Сейчас nginx проксирует только `/webhooks/`. Нужно добавить `/admin/`:

```nginx
location /admin/ {
    proxy_pass http://127.0.0.1:9800;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

Выведи эту инструкцию в CHANGELOG.

---

## ПОРЯДОК РАБОТЫ

1. Прочитай текущие `api/admin.py`, `models/db.py`, `config.py`, `main.py`
2. Создай миграцию `migrations/004_event_log.sql`
3. Добавь модель `EventLog` в `models/db.py`
4. Создай `utils/event_logger.py`
5. Добавь вызовы `log_event()` во все нужные места (webhooks, messenger, auth, agent)
6. Дополни `api/admin.py` (PUT, events, stats, авторизация cookie)
7. Создай `api/admin_web.py` (роуты для HTML-страниц, логин/логаут)
8. Создай `templates/login.html`
9. Создай `templates/admin.html`
10. Обнови `main.py` — подключи Jinja2, admin_web router
11. Обнови `config.py` — admin_login, admin_password, admin_secret_key
12. Проверь что всё импортируется без ошибок

---

## ПОСЛЕ ВЫПОЛНЕНИЯ — ОБЯЗАТЕЛЬНЫЙ ОТЧЁТ

Создай файл `CHANGELOG-admin-panel.md` в корне проекта. В нём:

1. **SQL-миграция** — полный текст `migrations/004_event_log.sql`

2. **Таблица изменений** — для КАЖДОГО изменённого файла:
   - Путь к файлу
   - Что изменено (конкретные функции)
   - Почему

3. **Новые файлы** — назначение каждого

4. **Инструкция запуска:**
   - Как накатить миграцию
   - Что добавить в `.env`
   - Что добавить в nginx
   - URL админки
   - Как войти

5. **Проверь импорты:**
```bash
cd /opt/openai/crm-worker
source venv/bin/activate
python -c "from api.admin_web import *; print('Admin web OK')"
python -c "from utils.event_logger import *; print('Event logger OK')"
python -c "from models.db import EventLog; print('EventLog model OK')"
```
