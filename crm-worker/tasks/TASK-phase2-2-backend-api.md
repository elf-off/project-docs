# TASK: Фаза 2.2 — Бэкенд API откликов (очередь, статусы, запись)

**Дата:** 2026-03-30
**Приоритет:** Высокий
**Проект:** `/var/www/new.kadry-24.ru/`
**Зависимости:** TASK-phase2-1-db-schema.md выполнен

---

## КОНТЕКСТ

Таблицы созданы (regions, vacancy_code_regions, operator_regions, region_work_schedule, response_assignments, system_settings). Нужен бэкенд: роутеры и сервисы для работы с откликами.

**Алгоритм "умный pull":**
1. Карточки из `handover_cards` (CRM Worker) попадают в очередь региона через `response_assignments` со статусом `queued`
2. Оператор открывает вкладку "Отклики" → GET запрос → система назначает ему порцию из очереди его регионов (FIFO, самые старые первыми)
3. Размер порции — из `system_settings` (ключ `responses.batch_size`, по умолчанию 5)
4. Оператор обрабатывает карточку: ставит статус или записывает кандидата
5. Когда все назначенные обработаны — получает следующую порцию

---

## 1. СТРУКТУРА ФАЙЛОВ

```
backend/app/
├── routers/
│   ├── responses.py          # НОВЫЙ — эндпоинты откликов
│   └── regions.py            # НОВЫЙ — CRUD регионов (админка)
├── services/
│   ├── response_service.py   # НОВЫЙ — логика очереди и назначений
│   └── region_service.py     # НОВЫЙ — логика регионов
├── schemas/
│   └── responses.py          # НОВЫЙ — Pydantic schemas
```

Зарегистрировать роутеры в `main.py`:
```python
from app.routers import responses, regions

app.include_router(responses.router, prefix="/api/operator/responses", tags=["responses"])
app.include_router(regions.router, prefix="/api/admin/regions", tags=["regions"])
```

---

## 2. PYDANTIC SCHEMAS (`backend/app/schemas/responses.py`)

```python
from pydantic import BaseModel
from typing import Optional, List
from datetime import datetime
from enum import Enum

class ResponseStatus(str, Enum):
    queued = "queued"
    assigned = "assigned"
    not_suitable = "not_suitable"
    refused = "refused"
    thinking = "thinking"
    recorded = "recorded"
    expired = "expired"

class ResponseCardOut(BaseModel):
    """Карточка отклика для оператора"""
    assignment_id: int          # response_assignments.id
    handover_card_id: int
    region_name: str
    status: ResponseStatus
    
    # Данные из handover_cards (cross-db)
    candidate_name: Optional[str] = None
    candidate_phone: Optional[str] = None
    candidate_age: Optional[int] = None
    candidate_city: Optional[str] = None
    candidate_metro: Optional[str] = None
    vacancy_title: Optional[str] = None
    vacancy_code: Optional[str] = None
    callback_slot: Optional[str] = None
    dialog_summary: Optional[str] = None
    result: Optional[str] = None       # booking, interested, not_interested...
    messages_count: Optional[int] = None
    
    assigned_at: Optional[datetime] = None
    created_at: datetime

class ResponseStatusUpdate(BaseModel):
    """Обновление статуса карточки"""
    status: ResponseStatus       # not_suitable, refused, thinking
    call_comment: Optional[str] = None

class ResponseRecordRequest(BaseModel):
    """Запись кандидата из отклика"""
    # Поля формы записи (как ApplicationForm на карте)
    object_id: int
    vacancy_id: int
    last_name: str
    first_name: str
    middle_name: Optional[str] = None
    phone: str
    exit_date: str               # YYYY-MM-DD
    shift: Optional[str] = None
    citizenship_id: Optional[int] = None
    medical_exam_id: Optional[int] = None
    notification: Optional[str] = None
    security_check: Optional[str] = None
    comment: Optional[str] = None

class ResponseListOut(BaseModel):
    """Список карточек с метаданными"""
    cards: List[ResponseCardOut]
    total_in_queue: int          # сколько ещё в очереди региона
    batch_size: int              # текущий размер порции

class DialogMessageOut(BaseModel):
    """Сообщение из диалога"""
    sender: str                  # ai / candidate
    text: str
    created_at: datetime

class RegionOut(BaseModel):
    id: int
    name: str
    timezone: str
    is_active: bool

class RegionWorkScheduleOut(BaseModel):
    day_of_week: int
    start_time: str
    end_time: str
    is_working: bool
```

---

## 3. СЕРВИС ОТКЛИКОВ (`backend/app/services/response_service.py`)

### 3.1 Синхронизация очереди

Функция `sync_handover_to_queue(engine)`:
- Читает `handover_cards` из `2_kadry_4_crm_avito` которых ещё нет в `response_assignments`
- Для каждой карточки определяет регион через vacancy_code → `vacancy_code_regions`
- Создаёт запись в `response_assignments` со статусом `queued`
- Карточки без маппинга (неизвестный vacancy_code) — логировать, не создавать assignment

**SQL для получения новых карточек:**
```sql
SELECT hc.id, hc.vacancy_code
FROM 2_kadry_4_crm_avito.handover_cards hc
LEFT JOIN response_assignments ra ON ra.handover_card_id = hc.id
WHERE ra.id IS NULL
ORDER BY hc.created_at ASC
```

**SQL для определения региона:**
```sql
SELECT region_id FROM vacancy_code_regions WHERE vacancy_code = :code
```

**Вызов:** при каждом GET /my-cards (перед выдачей порции).

### 3.2 Получение порции карточек

Функция `get_operator_cards(engine, operator_user_id) -> ResponseListOut`:

**Алгоритм:**
1. Вызвать `sync_handover_to_queue()` — подтянуть новые карточки
2. Получить регионы оператора из `operator_regions`
3. Посчитать сколько у оператора уже assigned (не обработанных)
4. Если assigned > 0 — вернуть их (оператор ещё не закончил порцию)
5. Если assigned == 0 — назначить новую порцию:
   - Взять `batch_size` из `system_settings`
   - SELECT из `response_assignments` WHERE `status = 'queued'` AND `region_id IN (регионы оператора)` ORDER BY `created_at ASC` LIMIT batch_size
   - UPDATE `status = 'assigned'`, `operator_user_id`, `assigned_at = NOW()`
   - **⚠️ FOR UPDATE** — блокировка строк чтобы два оператора не получили одну карточку
6. Вернуть назначенные карточки с данными из `handover_cards` (JOIN cross-db)

**SQL для получения полных данных карточки:**
```sql
SELECT 
    ra.id as assignment_id,
    ra.handover_card_id,
    ra.status,
    ra.assigned_at,
    ra.created_at,
    r.name as region_name,
    hc.candidate_name, hc.candidate_phone, hc.candidate_age,
    hc.candidate_city, hc.candidate_metro,
    hc.vacancy_title, hc.vacancy_code, hc.callback_slot,
    hc.dialog_summary, hc.result, hc.messages_count
FROM response_assignments ra
JOIN regions r ON r.id = ra.region_id
JOIN 2_kadry_4_crm_avito.handover_cards hc ON hc.id = ra.handover_card_id
WHERE ra.operator_user_id = :user_id
  AND ra.status = 'assigned'
ORDER BY ra.created_at ASC
```

**⚠️ ВАЖНО:** Поля в `handover_cards` могут называться иначе. Перед реализацией проверить:
```bash
mysql -u root -p 2_kadry_4_crm_avito -e "DESCRIBE handover_cards;"
```
И адаптировать имена полей (candidate_name vs applicant_name, dialog_summary vs ai_summary и т.д.).

### 3.3 Обновление статуса

Функция `update_card_status(engine, assignment_id, operator_user_id, status, comment)`:
- Проверить что карточка принадлежит этому оператору
- Обновить `status`, `call_comment`, `processed_at = NOW()`
- Статусы `not_suitable`, `refused` — финальные
- Статус `thinking` — промежуточный, карточка остаётся у оператора

### 3.4 Запись кандидата из отклика

Функция `record_candidate(engine, assignment_id, operator_user_id, data: ResponseRecordRequest)`:
- Проверить что карточка принадлежит оператору
- Создать запись в `applications` (таблица в `2_kadry_4_maps`) — **переиспользовать `create_application()` из `application_service.py`**
- Обновить `response_assignments`: `status = 'recorded'`, `application_id = новый_id`, `processed_at = NOW()`
- Вернуть ID созданной заявки

### 3.5 Получение диалога

Функция `get_card_dialog(engine, assignment_id, operator_user_id)`:
- Проверить что карточка принадлежит оператору
- Получить `handover_card_id` → найти `application_id` в `2_kadry_4_crm_avito.handover_cards`
- Через `application_id` найти `chat_id` в `2_kadry_4_crm_avito.applications`
- Загрузить сообщения из `2_kadry_4_crm_avito.messages` WHERE `chat_id`
- Вернуть отсортированный список

**SQL:**
```sql
SELECT m.sender_type as sender, m.content as text, m.created_at
FROM 2_kadry_4_crm_avito.messages m
JOIN 2_kadry_4_crm_avito.applications app ON app.chat_id = m.chat_id
JOIN 2_kadry_4_crm_avito.handover_cards hc ON hc.application_id = app.id
WHERE hc.id = :handover_card_id
ORDER BY m.created_at ASC
```

### 3.6 Статистика очереди

Функция `get_queue_stats(engine, operator_user_id)`:
- Для каждого региона оператора: количество `queued`, `assigned`, обработанных за сегодня
- Для алерта: если `queued > 0` и нет активных операторов в регионе

---

## 4. РОУТЕР ОТКЛИКОВ (`backend/app/routers/responses.py`)

Все эндпоинты требуют авторизации через Redis-сессии. **⚠️ Использовать `user["user_id"]`, НЕ `user["id"]`!**

### 4.1 GET /api/operator/responses/my-cards

Получить текущую порцию карточек оператора (с автоназначением).
- Permission: `operator.responses.view`
- Response: `ResponseListOut`
- Логика: вызвать `get_operator_cards()`

### 4.2 PUT /api/operator/responses/{assignment_id}/status

Обновить статус карточки.
- Permission: `operator.responses.process`
- Body: `ResponseStatusUpdate`
- Проверки: карточка принадлежит оператору, статус валидный
- Переходы: `assigned → not_suitable|refused|thinking`, `thinking → not_suitable|refused`
- **Нельзя:** `queued → *` (не назначена), `recorded → *` (уже записан), `not_suitable/refused → *` (финальные)

### 4.3 POST /api/operator/responses/{assignment_id}/record

Записать кандидата из отклика.
- Permission: `operator.responses.process`
- Body: `ResponseRecordRequest`
- Логика: вызвать `record_candidate()`
- Response: `{ "application_id": 123, "message": "Кандидат записан" }`

### 4.4 GET /api/operator/responses/{assignment_id}/dialog

Получить историю диалога AI с кандидатом.
- Permission: `operator.responses.view`
- Response: `List[DialogMessageOut]`

### 4.5 GET /api/operator/responses/stats

Статистика по очередям регионов оператора.
- Permission: `operator.responses.view`
- Response: `{ "regions": [{ "name": "Ростов", "queued": 12, "my_assigned": 3, "processed_today": 7 }] }`

---

## 5. РОУТЕР РЕГИОНОВ (`backend/app/routers/regions.py`)

Админские эндпоинты для управления регионами.

### 5.1 GET /api/admin/regions — список регионов
### 5.2 POST /api/admin/regions — создать регион
### 5.3 PUT /api/admin/regions/{id} — обновить регион
### 5.4 GET /api/admin/regions/{id}/schedule — расписание региона
### 5.5 PUT /api/admin/regions/{id}/schedule — обновить расписание (массив из 7 дней)
### 5.6 GET /api/admin/regions/{id}/operators — операторы региона
### 5.7 POST /api/admin/regions/{id}/operators — добавить оператора в регион
### 5.8 DELETE /api/admin/regions/{id}/operators/{user_id} — убрать оператора из региона
### 5.9 GET /api/admin/regions/vacancy-codes — список маппингов vacancy_code → регион
### 5.10 POST /api/admin/regions/vacancy-codes — добавить/обновить маппинг
### 5.11 GET /api/admin/settings — системные настройки (batch_size и др.)
### 5.12 PUT /api/admin/settings/{key} — обновить настройку

Permissions: `admin.regions.view` для GET, `admin.regions.manage` для POST/PUT/DELETE, `admin.settings.manage` для настроек.

---

## 6. РЕГИСТРАЦИЯ В NGINX

Не нужна — роутеры зарегистрированы в main-api (порт 8100), nginx уже проксирует `/api/operator/` и `/api/admin/` на 8100.

**Проверить** что nginx конфиг пропускает `/api/admin/regions/`:
```bash
grep -A5 "location /api/" /etc/nginx/sites-enabled/new.kadry-24.ru*
```

Если `/api/admin/` не настроен отдельно — добавить:
```nginx
location /api/admin/ {
    proxy_pass http://127.0.0.1:8100;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

---

## 7. ПРОВЕРКА

```bash
# Перезапустить
sudo systemctl restart k24-main-api

# Health check
curl -s http://127.0.0.1:8100/api/health

# Проверить что роутеры подключены (должны быть в списке)
curl -s http://127.0.0.1:8100/docs | grep -o 'responses\|regions' | sort -u

# Проверить получение карточек (с валидной сессией оператора)
curl -s -b "session=VALID_SESSION" http://127.0.0.1:8100/api/operator/responses/my-cards

# Проверить список регионов (с сессией админа)
curl -s -b "session=ADMIN_SESSION" http://127.0.0.1:8100/api/admin/regions
```

### Чеклист:
- [ ] Schemas созданы
- [ ] response_service.py — sync, get_cards, update_status, record, dialog, stats
- [ ] region_service.py — CRUD регионов, расписание, операторы, маппинг, настройки
- [ ] responses.py роутер — 5 эндпоинтов, permissions проверяются
- [ ] regions.py роутер — 12 эндпоинтов, permissions проверяются
- [ ] Роутеры зарегистрированы в main.py
- [ ] FOR UPDATE в назначении порции (конкурентность)
- [ ] `user["user_id"]` везде (НЕ `user["id"]`)
- [ ] Cross-db запросы к `2_kadry_4_crm_avito` работают

---

## ⚠️ КРИТИЧЕСКИ ВАЖНО

1. **Redis session regression:** Все роутеры/сервисы ОБЯЗАНЫ использовать `user["user_id"]`, НЕ `user["id"]`. Проверить после реализации:
```bash
grep -rn 'user\["id"\]' backend/app/routers/responses.py backend/app/routers/regions.py backend/app/services/response_service.py backend/app/services/region_service.py
```
Результат должен быть **пустым**.

2. **Имена полей в handover_cards:** Перед реализацией выполнить:
```bash
mysql -u root -p 2_kadry_4_crm_avito -e "DESCRIBE handover_cards;"
```
И адаптировать имена в SQL и schemas.

3. **Переиспользовать create_application():** При записи кандидата из отклика не дублировать логику — вызывать существующую функцию из `application_service.py`.
