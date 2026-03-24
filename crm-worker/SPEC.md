# SPEC: AI-воркер ночного агента Avito

## Обзор

Python-сервис, работающий на Debian 12. Принимает вебхуки от Avito, в ночное окно (21:00–09:00) ведёт диалог с соискателями через Avito Messenger API от имени оператора, квалифицирует и сегментирует их, к 09:00 формирует карточки передачи операторам.

## Стек

- **Python 3.11+**
- **FastAPI** — HTTP-сервер (webhook endpoint + будущий API)
- **MariaDB** — основное хранилище (схема: `schema_v1.sql`)
- **Qdrant** — векторная БД для RAG-справочника по вакансиям (уже установлен на сервере)
- **Claude API (Anthropic)** — генерация ответов AI-агента
- **httpx** — async HTTP-клиент для Avito API и Claude API
- **SQLAlchemy 2.0** — async ORM (aiomysql драйвер)
- **APScheduler** — планировщик задач (отложенная отправка, follow-up, обновление токенов)
- **Uvicorn** — ASGI-сервер
- **systemd** — управление процессом

## Структура проекта

```
/opt/crm-worker/
├── main.py                     # Точка входа, FastAPI app, lifespan
├── config.py                   # Настройки из .env
├── .env                        # Секреты (не в git)
│
├── api/
│   ├── __init__.py
│   └── webhooks.py             # POST /webhooks/avito — приём вебхуков
│
├── services/
│   ├── __init__.py
│   ├── avito_auth.py           # Получение/обновление access_token
│   ├── avito_messenger.py      # Отправка/получение сообщений через Avito Messenger API
│   ├── avito_applications.py   # Получение данных отклика
│   ├── ai_agent.py             # Основная логика диалога AI
│   ├── ai_claude.py            # Обёртка над Claude API
│   ├── ai_rag.py               # RAG: поиск по Qdrant для подбора вакансий
│   ├── segmentation.py         # Логика блоков 1/2/3
│   ├── handover.py             # Формирование карточки передачи
│   └── message_scheduler.py    # Планировщик отложенной отправки с паузами
│
├── models/
│   ├── __init__.py
│   └── db.py                   # SQLAlchemy модели (все 10 таблиц)
│
├── workers/
│   ├── __init__.py
│   ├── incoming_processor.py   # Обработка входящего сообщения от соискателя
│   └── token_refresher.py      # Фоновая задача обновления токенов
│
├── prompts/
│   ├── system.txt              # Системный промпт AI-агента
│   ├── greeting.txt            # Шаблон приветствия
│   ├── qualification.txt       # Промпт для этапа квалификации
│   ├── presentation.txt        # Промпт для презентации вакансии
│   ├── objection.txt           # Промпт для работы с возражениями
│   └── followup.txt            # Промпт для follow-up
│
└── utils/
    ├── __init__.py
    ├── logger.py               # Логирование (structlog)
    └── time_helpers.py         # Проверка ночного окна, расчёт пауз
```

## Конфигурация (.env)

```env
# MariaDB
DB_HOST=localhost
DB_PORT=3306
DB_NAME=crm_avito
DB_USER=crm
DB_PASSWORD=...

# Qdrant
QDRANT_HOST=localhost
QDRANT_PORT=6333
QDRANT_COLLECTION=vacancies

# Anthropic Claude
ANTHROPIC_API_KEY=sk-ant-...
CLAUDE_MODEL=claude-sonnet-4-6
CLAUDE_PROXY=socks5://127.0.0.1:1080   # Локальный microsocks, запущенный от user openai → трафик уходит в VPN

# Avito (дефолтный аккаунт, остальные из БД)
# Запросы к Avito API идут напрямую, без прокси (от user crm)
AVITO_DEFAULT_ACCOUNT_ID=1

# Webhook
WEBHOOK_SECRET=...              # Секрет для верификации подписи от Avito
WEBHOOK_HOST=127.0.0.1          # Слушаем только localhost (nginx проксирует)
WEBHOOK_PORT=8800               # Внутренний порт, наружу — nginx на 443 (HTTPS)

# AI Agent
AI_WORK_START=21:00             # Начало ночного окна
AI_WORK_END=09:00               # Конец ночного окна
AI_PAUSE_MIN_SEC=120            # Мин. пауза между сообщениями (2 мин)
AI_PAUSE_MAX_SEC=360            # Макс. пауза (6 мин)
AI_MAX_FOLLOWUPS=2              # Макс. follow-up без ответа
AI_FOLLOWUP_DELAY_MIN=600       # Мин. задержка follow-up (10 мин)
AI_FOLLOWUP_DELAY_MAX=900       # Макс. задержка follow-up (15 мин)

# Timezone
TZ=Europe/Moscow
```

## Компоненты — детальное описание

### 1. Webhook endpoint (`api/webhooks.py`)

```
POST /webhooks/avito
```

Принимает JSON от Avito. Два типа событий:

**a) Новый отклик** (subscription на applications)
- Логировать сырой payload в `webhook_log`
- Извлечь: application_id, item_id, chat_id
- Через Avito API получить детали отклика (имя, телефон)
- Создать/обновить запись в `applicants` (дедупликация по телефону)
- Создать запись в `applications` (status=`new`)
- Создать запись в `chats`
- Проверить время: если внутри ночного окна → создать `ai_sessions` (stage=`greeting`), поставить отправку приветствия через паузу
- Если вне окна → оставить status=`new`, ничего не делать (операторы разберут)
- Вернуть HTTP 200 немедленно (обработка асинхронно)

**b) Новое входящее сообщение** (если Avito присылает уведомление о сообщении)
- Сохранить в `messages` (direction=`incoming`, sender_type=`applicant`)
- Найти активную `ai_sessions` для этого чата
- Если есть и ночное окно → передать в `incoming_processor`
- Обновить `chats.last_message_at`
- Вернуть HTTP 200

**Важно:** Вебхук должен отвечать за < 1 сек. Вся тяжёлая работа — через внутреннюю очередь (asyncio.Queue или просто фоновая asyncio-задача).

### 2. Avito Auth (`services/avito_auth.py`)

```python
async def get_valid_token(account_id: int) -> str
```

- Проверить `avito_accounts.token_expires_at`
- Если токен жив (с запасом 5 мин) → вернуть из БД
- Если протух → запросить новый: `POST https://api.avito.ru/token` с `grant_type=client_credentials`
- Обновить `access_token` и `token_expires_at` в БД
- Вернуть новый токен

Фоновая задача `token_refresher` — раз в 30 минут проверяет все активные аккаунты и превентивно обновляет токены.

### 3. Avito Messenger (`services/avito_messenger.py`)

```python
async def send_message(account_id: int, chat_id: str, text: str) -> dict
async def get_messages(account_id: int, chat_id: str) -> list[dict]
async def get_chat_info(account_id: int, chat_id: str) -> dict
```

API endpoints Avito Messenger:
- Отправка: `POST https://api.avito.ru/messenger/v1/accounts/{user_id}/chats/{chat_id}/messages`
- Получение: `GET https://api.avito.ru/messenger/v1/accounts/{user_id}/chats/{chat_id}/messages`

Body отправки:
```json
{
    "message": {
        "text": "Текст сообщения"
    },
    "type": "text"
}
```

Каждое отправленное сообщение сохранять в `messages` (direction=`outgoing`, sender_type=`ai`).

### 4. AI Agent — основная логика (`services/ai_agent.py`)

Центральный модуль. Управляет диалогом по этапам.

```python
async def process_new_application(application_id: int) -> None
async def process_incoming_message(message_id: int) -> None
async def process_followup(ai_session_id: int) -> None
```

#### Этапы диалога (`dialog_stage`):

**greeting** → Приветственное сообщение.
Генерируется через Claude (НЕ фиксированный шаблон — иначе Avito детектит как автоответ и может заблокировать).
Claude получает контекст (имя агента, название вакансии, проект) и каждый раз формулирует приветствие немного по-разному: порядок слов, обороты, длина. Смысл один — форма разная.
Промпт в `prompts/greeting.txt`:
```
Напиши приветственное сообщение соискателю. Данные:
- Твоё имя: {имя_агента}
- Вакансия: {название_вакансии}
- Проект: {название_проекта}

Смысл: поблагодарить за отклик, представиться, упомянуть вакансию.
Каждый раз формулируй по-разному. 2-3 предложения. Без эмодзи.
```
После отправки → stage = `qualification`

**qualification** → Сбор данных.
Сообщение (фиксированное + лёгкая вариация через Claude):
```
С вашего позволения уточню несколько деталей, чтобы подобрать идеальное предложение.
Подскажите, пожалуйста: ваш возраст, гражданство и ближайшее метро?
```
Ждём ответ. Когда получаем — отправляем в Claude для парсинга:
- Системный промпт: «Извлеки из сообщения: возраст (число или null), гражданство (строка или null), метро/район (строка или null). Ответь JSON.»
- Сохранить в `ai_sessions.collected_*`
- Если чего-то не хватает — AI переспрашивает (через Claude)
- Когда всё собрано → stage = `presentation`

**presentation** → Информация о вакансии.
- Взять данные вакансии из `vacancies` + RAG из Qdrant (детали, условия)
- Claude формирует краткую презентацию вакансии
- Спросить: «Вам интересно? Могу рассказать подробнее.»
- Ждём ответ → stage = `segmentation`

**segmentation** → Определение блока.
На основе ответа соискателя + собранных данных:
- Claude анализирует и выдаёт решение (JSON: block, reason)
- Записать в `ai_sessions.assigned_block`
- Записать в `applications.block`
- Перейти к действиям по блоку → stage = `followup` или `handover`

**followup** → Действия по блоку (см. ниже).

**handover** → Финализация.
- Сформировать `handover_cards`
- Попросить Claude написать `ai_summary` (2-3 предложения)
- Обновить `applications.status = 'ai_done'`
- stage = `completed`

#### Действия по блокам:

**Блок 1 (приоритет):**
AI предлагает:
```
Отлично! Завтра после 9:00 мой коллега позвонит вам. 
Какой интервал удобнее: 10–12, 12–15 или 15–18?
```
Ждём ответ → парсим слот → записываем `callback_slot` → handover.

**Блок 2 (тёплый):**
AI задаёт уточняющий вопрос (через Claude):
- «Что для вас важнее: локация, график или оплата?»
- Если ответил → пробуем подобрать через RAG и сдвинуть в блок 1
- Если молчит → follow-up (макс 2 раза)
- handover с пометкой «clarify»

**Блок 3 (не подходит):**
AI:
```
Поняла вас. Могу предложить 1–2 вакансии ближе к вашему метро. 
Что для вас важнее: график, оплата или расстояние?
```
- Если согласен → RAG-поиск по метро/городу → предложить альтернативу
- Если отказ → «Могу поинтересоваться, что именно не понравилось?» (работа с возражением через Claude)
- handover с reason + отметка «альтернативы предложены / не предложены»

### 5. AI Claude — обёртка (`services/ai_claude.py`)

```python
async def ask_claude(
    system: str,
    messages: list[dict],   # [{"role": "user", "content": "..."}]
    session_id: int = None,
    application_id: int = None,
    temperature: float = 0.7,
    max_tokens: int = 500
) -> str
```

- Использовать `httpx.AsyncClient` с SOCKS5-прокси (`127.0.0.1:1080` → microsocks от user `openai` → VPN)
- Запросы к Avito API — отдельный `httpx.AsyncClient` БЕЗ прокси (напрямую)
- Endpoint: `POST https://api.anthropic.com/v1/messages`
- Модель: из конфига (`claude-sonnet-4-20250514`)
- Обязательно логировать в `ai_prompts_log`: tokens, cost, response_ms
- При ошибке API — retry 2 раза с backoff (2, 5 сек)
- При неудаче — логировать, НЕ отправлять сообщение, пометить сессию как `failed`

#### Формирование контекста для Claude:

В `messages` передавать ПОСЛЕДНИЕ 10 сообщений из `messages` таблицы для данного чата (чтобы Claude видел контекст диалога). Формат:
```python
[
    {"role": "user", "content": "текст от соискателя"},
    {"role": "assistant", "content": "текст AI-ответа"},
    {"role": "user", "content": "следующее сообщение соискателя"},
    ...
]
```

Системный промпт загружать из файлов `prompts/*.txt` в зависимости от `dialog_stage`.

### 6. RAG — поиск по вакансиям (`services/ai_rag.py`)

```python
async def search_vacancies(query: str, city: str = None, metro: str = None, top_k: int = 3) -> list[dict]
async def index_vacancy(vacancy_id: int, text: str, metadata: dict) -> None
async def sync_vacancies_from_avito(account_id: int) -> int  # Возвращает кол-во обновлённых
```

- Qdrant коллекция: `vacancies`
- **Источник данных: Avito API** — наиболее полная и свежая информация. Другие источники не успевают обновляться.
- Синхронизация: `GET https://api.avito.ru/core/v1/accounts/{user_id}/items` — получить все активные объявления, вытащить описание, условия, требования, адрес.
- При `sync_vacancies_from_avito`: забрать все активные объявления → для каждого создать/обновить запись в `vacancies` + отправить текст в Qdrant
- Embedding: использовать Claude API (или переключить на отдельную embedding-модель позже)
- При поиске: фильтры по city/metro + семантический поиск
- Возвращать: название, проект, условия, требования
- **Расписание синхронизации:** раз в 2 часа через APScheduler (вакансии могут меняться)

### 7. Message Scheduler (`services/message_scheduler.py`)

```python
async def schedule_message(chat_id: int, text: str, delay_sec: int, sender_type: str = 'ai') -> int
async def process_scheduled() -> None    # Вызывается каждые 5 сек
```

Логика пауз (имитация человека):
- Базовая пауза: случайная от `AI_PAUSE_MIN_SEC` до `AI_PAUSE_MAX_SEC` (2-6 мин)
- Если соискатель прислал длинное сообщение (> 100 символов) → добавить +30-60 сек (как будто читаем)
- Если серия сообщений от соискателя (3+ за минуту) → подождать пока прекратит, потом ответить с обычной паузой
- Никогда не отвечать быстрее 60 секунд

Реализация:
- Записать в `messages` с `scheduled_at = NOW() + delay`
- Фоновая задача `process_scheduled` каждые 5 сек проверяет: `SELECT * FROM messages WHERE scheduled_at <= NOW() AND direction='outgoing' AND delivered_at IS NULL`
- Отправить через Avito Messenger API
- Обновить `delivered_at`

### 8. Incoming Processor (`workers/incoming_processor.py`)

```python
async def handle_incoming(message_id: int) -> None
```

Вызывается при получении входящего сообщения от соискателя в ночное окно:

1. Загрузить сообщение, чат, application, ai_session
2. Проверить есть ли запланированные но ещё не отправленные сообщения — если да, не сбрасывать (AI «уже печатает»)
3. Определить текущий `dialog_stage`
4. Вызвать соответствующий метод `ai_agent`
5. Получить ответ от Claude
6. Запланировать отправку через `message_scheduler` с паузой

### 9. Handover — формирование карточки (`services/handover.py`)

```python
async def create_handover_card(ai_session_id: int) -> int
```

- Собрать все данные из `ai_sessions`
- Попросить Claude сформировать `ai_summary` на основе всей переписки (краткая выжимка: кто, что хочет, что договорились)
- Определить `next_action`:
  - block_1 + callback_slot → `call`
  - block_2 → `clarify`
  - block_3 + alt offered → `offer_alternatives`
  - block_3 + final reject → `close`
- Сохранить в `handover_cards`
- Обновить `applications.status = 'ai_done'`

## Системный промпт AI-агента (`prompts/system.txt`)

```
Ты — HR-менеджер по имени {имя}. Ты работаешь в кадровом агентстве и общаешься с соискателями в чате Avito.

Правила:
- Пиши дружелюбно, но профессионально. Без канцелярита.
- Используй простые предложения. Пиши как в мессенджере, не как в деловом письме.
- НЕ используй эмодзи (Avito может резать).
- НЕ упоминай WhatsApp, Telegram или другие мессенджеры (санкции Avito).
- НЕ говори что ты бот или AI. Ты — живой человек.
- Не отправляй ссылки на внешние ресурсы.
- Если соискатель грубит — вежливо завершай диалог.
- Если соискатель задаёт вопрос, на который не знаешь ответ — скажи что уточнишь у коллег и передашь информацию.

Контекст:
Вакансия: {vacancy_title}
Проект: {project_name}
Город: {city}
{vacancy_details из RAG}

Текущий этап: {dialog_stage}
Собранные данные: {collected_data}
```

## Обработка ошибок

| Ситуация | Действие |
|---|---|
| Avito API 401 | Обновить токен, повторить запрос |
| Avito API 429 | Backoff 30 сек, повторить (макс 3 раза) |
| Avito API 5xx | Retry через 10 сек, логировать, после 3 попыток — пометить сессию `failed` |
| Claude API ошибка | Retry 2 раза (2, 5 сек), при неудаче — НЕ отправлять сообщение, сессия `failed` |
| Webhook дубликат | Проверять `avito_application_id` на уникальность, пропускать дубли |
| Соискатель молчит > 15 мин | Follow-up (макс 2), потом handover как блок 2 |
| Сообщение от Avito вне ночного окна | Сохранить в messages, НЕ запускать AI |

## Запуск и деплой

**systemd unit** (`/etc/systemd/system/crm-worker.service`):

```ini
[Unit]
Description=CRM Avito AI Worker
After=network.target mariadb.service

[Service]
Type=exec
User=crm
WorkingDirectory=/opt/crm-worker
ExecStart=/opt/crm-worker/venv/bin/uvicorn main:app --host 127.0.0.1 --port 8800
Restart=always
RestartSec=5
EnvironmentFile=/opt/crm-worker/.env

[Install]
WantedBy=multi-user.target
```

**Инициализация:**
```bash
cd /opt/crm-worker
python -m venv venv
source venv/bin/activate
pip install fastapi uvicorn "httpx[socks]" sqlalchemy aiomysql apscheduler anthropic qdrant-client structlog
```

## Инфраструктура: nginx + microsocks

### nginx — HTTPS-прокси для вебхуков Avito

Avito отправляет вебхуки на HTTPS с валидным сертификатом. Настройка в два этапа.

**Этап A — HTTP-конфиг для получения сертификата** (`/etc/nginx/sites-available/crm-webhook`):

```nginx
server {
    listen 80;
    server_name webhook.yourdomain.ru;

    # Для certbot
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 444;
    }
}
```

```bash
mkdir -p /var/www/certbot
ln -s /etc/nginx/sites-available/crm-webhook /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx

# Получаем сертификат
certbot certonly --webroot -w /var/www/certbot -d webhook.yourdomain.ru
```

**Этап B — Переключаем на HTTPS** (заменяем содержимое `/etc/nginx/sites-available/crm-webhook`):

```nginx
server {
    listen 80;
    server_name webhook.yourdomain.ru;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name webhook.yourdomain.ru;

    ssl_certificate     /etc/letsencrypt/live/webhook.yourdomain.ru/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/webhook.yourdomain.ru/privkey.pem;

    location /webhooks/ {
        proxy_pass http://127.0.0.1:8800;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Для автообновления сертификата
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
}
```

```bash
nginx -t && systemctl reload nginx
```

**Автообновление сертификата** — certbot ставит cron/timer автоматически. Проверить: `systemctl list-timers | grep certbot`.

URL для регистрации вебхука в Avito: `https://webhook.yourdomain.ru/webhooks/avito`

### microsocks — SOCKS5-прокси для Claude API через VPN

На сервере трафик от Linux-пользователя `openai` маршрутизируется в VPN. Чтобы воркер (user `crm`) мог отправлять запросы к Claude через VPN, нужен локальный SOCKS5-прокси, запущенный от user `openai`.

**Установка:**
```bash
apt install microsocks
```

**systemd unit** (`/etc/systemd/system/microsocks-vpn.service`):
```ini
[Unit]
Description=MicroSOCKS proxy (VPN via openai user)
After=network.target

[Service]
Type=simple
User=openai
ExecStart=/usr/bin/microsocks -i 127.0.0.1 -p 1080
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

```bash
systemctl enable microsocks-vpn
systemctl start microsocks-vpn
```

Проверка: `curl --socks5 127.0.0.1:1080 https://api.anthropic.com/v1/models` — должен пройти через VPN.

## Порядок разработки

0. **Инфраструктура**: nginx (SSL + reverse proxy), microsocks (SOCKS5 от user openai), БД `crm_avito` в MariaDB (schema_v1.sql)
1. `config.py` + `models/db.py` + подключение к MariaDB
2. `services/avito_auth.py` + `workers/token_refresher.py` — получение/обновление токенов
3. `api/webhooks.py` — приём вебхуков, запись в `webhook_log` и `applications`
4. `services/avito_messenger.py` — отправка/получение сообщений
5. `services/avito_applications.py` — получение данных отклика + список объявлений
6. `services/ai_rag.py` — синхронизация вакансий из Avito API → Qdrant
7. `services/message_scheduler.py` — очередь с паузами
8. `services/ai_claude.py` — обёртка Claude API + логирование + прокси через VPN
9. `services/ai_agent.py` — логика диалога по этапам (greeting через Claude + qualification)
10. `workers/incoming_processor.py` — обработка входящих
11. `services/segmentation.py` — блоки 1/2/3
12. `services/handover.py` — карточки передачи
13. Тесты, отладка, деплой

## Важные заметки

- **Маршрутизация трафика (VPN)**: На сервере настроен VPN для Linux-пользователя `openai` (через iptables/ip rule по UID — весь трафик от этого пользователя уходит в VPN). Воркер запускается от user `crm`. Для маршрутизации Claude API через VPN используется `microsocks`, запущенный от user `openai` на `127.0.0.1:1080`. Воркер отправляет запросы к Claude через этот SOCKS5-прокси → трафик идёт от UID `openai` → уходит в VPN. Запросы к Avito API идут напрямую (без прокси, от user `crm`).
- **Два HTTP-клиента**: в приложении два `httpx.AsyncClient` — один с SOCKS5-прокси (`127.0.0.1:1080`) для Claude API, второй без прокси для Avito API. Клиенты создаются при старте приложения в lifespan и переиспользуются.
- **Qdrant RAG** наполняется автоматически из Avito API (объявления аккаунта). Синхронизация раз в 2 часа. Это основной и самый свежий источник данных по вакансиям.
- **Follow-up логика**: если соискатель не отвечает 15 минут — первый follow-up. Ещё 15 минут — второй. После этого — handover как блок 2 с пометкой «молчит».
- **Параллельные диалоги**: воркер должен обрабатывать множество чатов одновременно. Никаких глобальных блокировок. Каждый чат — независимая цепочка.
- **Идемпотентность**: один и тот же webhook может прийти дважды. Проверять по `avito_application_id` / `avito_message_id`.
- **Graceful shutdown**: при остановке сервиса — дождаться отправки запланированных сообщений или сохранить состояние для переподхвата.
