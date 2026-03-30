# TASK: Фаза 2.3 — Фронтенд вкладки "Отклики" (Vue + PrimeVue)

**Дата:** 2026-03-30
**Приоритет:** Высокий
**Проект:** `/var/www/new.kadry-24.ru/`
**Зависимости:** TASK-phase2-2-backend-api.md выполнен

---

## КОНТЕКСТ

Бэкенд готов. Нужен Vue-фронтенд: третья вкладка "Отклики" в OperatorPage.vue + админка регионов.

**Существующая структура (`frontend/src/modules/operator/`):**
- `OperatorPage.vue` — главная страница с PrimeVue Tabs (Карта / Мои записи)
- `components/ApplicationForm.vue` — форма записи кандидата (переиспользуем)
- `composables/useOperatorApi.ts` — API-обёртки

---

## 1. НОВЫЕ КОМПОНЕНТЫ

```
frontend/src/modules/operator/
├── components/
│   ├── ResponsesList.vue           # НОВЫЙ — список карточек откликов
│   ├── ResponseCard.vue            # НОВЫЙ — одна карточка отклика
│   ├── ResponseDialog.vue          # НОВЫЙ — модалка с историей диалога AI
│   ├── ResponseRecordDrawer.vue    # НОВЫЙ — drawer записи кандидата из отклика
│   └── ResponseStats.vue           # НОВЫЙ — статистика очередей
├── composables/
│   └── useResponsesApi.ts          # НОВЫЙ — API откликов

frontend/src/modules/admin/
├── components/
│   ├── RegionsPage.vue             # НОВЫЙ — управление регионами
│   ├── RegionForm.vue              # НОВЫЙ — форма региона + расписание
│   ├── RegionOperators.vue         # НОВЫЙ — привязка операторов к регионам
│   └── VacancyCodeMapping.vue      # НОВЫЙ — маппинг vacancy_code → регион
```

---

## 2. ВКЛАДКА "ОТКЛИКИ" В OPERATORPAGE

### 2.1 OperatorPage.vue — добавить третий таб

Найти секцию PrimeVue Tabs. Сейчас два таба: "Карта" и "Мои записи".
Добавить третий:

```vue
<TabPanel header="Отклики" :disabled="!hasPermission('operator.responses.view')">
    <ResponsesList />
</TabPanel>
```

Проверить что permission `operator.responses.view` берётся из auth store.

### 2.2 ResponsesList.vue — основной компонент

**Верхняя панель:**
- Статистика очередей (компонент `ResponseStats`) — регион, в очереди, у меня, обработано сегодня
- Кнопка "Обновить" (повторный запрос GET /my-cards)

**Список карточек:**
- Компоненты `ResponseCard` в вертикальном списке
- Если нет карточек — сообщение "Все отклики обработаны" или "Нет откликов в ваших регионах"
- Если оператор не привязан к регионам — сообщение "Вы не привязаны к регионам. Обратитесь к администратору."

**Автообновление:**
- При открытии таба — загрузка карточек
- Polling каждые 30 секунд (проверка новых карточек) — **только пока таб активен**

### 2.3 ResponseCard.vue — карточка отклика

**Дизайн: PrimeVue Card.**

**Заголовок:** Имя кандидата + возраст (напр. "Ларина Юлия, 24 года")

**Тело карточки:**
- Телефон — с кнопкой копирования (click → clipboard, Toast "Скопировано")
- Город / Метро
- Вакансия + код вакансии
- Регион
- Слот для звонка (callback_slot)
- Краткое резюме диалога (dialog_summary) — 2-3 строки, с "показать полностью"
- Результат AI: цветной Badge
  - `booking` → зелёный "Записан на звонок"
  - `interested` → жёлтый "Заинтересован"
  - `not_interested` → серый "Не интересно"
  - `alternative` → синий "Нужны альтернативы"
  - `booking_no_phone` → оранжевый "Не дал телефон"

**Кнопки действий (внизу карточки):**
- Кнопки статусов: "Не подходит" (severity=secondary), "Отказался" (severity=warning), "Думает" (severity=info)
- Кнопка "Записать" (severity=success) — открывает ResponseRecordDrawer
- Кнопка "Диалог" (severity=help, outlined) — открывает ResponseDialog
- При клике на статус — Confirm Dialog "Вы уверены? Статус: Не подходит"
- После подтверждения — PUT /status, Toast, карточка исчезает из списка (кроме "Думает")

### 2.4 ResponseDialog.vue — модалка диалога

PrimeVue Dialog (modal, width 600px).

**Содержимое:**
- Заголовок: "Диалог с {имя кандидата}"
- Список сообщений в стиле чата:
  - AI (Елена) — слева, светлый фон
  - Кандидат — справа, цветной фон
  - Время под каждым сообщением
- Scroll to top при открытии (хронологический порядок)

### 2.5 ResponseRecordDrawer.vue — запись из отклика

PrimeVue Drawer (position right, width 500px).

**Переиспользует логику ApplicationForm.vue:**
- Поля: ФИО, телефон, дата выхода, смена, гражданство, мед.книжка, уведомление, СБ, комментарий
- **Дополнительно:** выбор проекта и вакансии (dropdown с поиском)
- **Автозаполнение:** ФИО, телефон, город — из данных карточки (handover_card)
- **Код вакансии:** подтягивается автоматом из карточки, read-only

**При сохранении:**
- POST /api/operator/responses/{assignment_id}/record
- Toast "Кандидат записан"
- Карточка исчезает из списка
- Запись появляется во вкладке "Мои записи"

**⚠️ DatePicker, Select, Dropdown** — обязательно `appendTo="self"` (иначе закроют drawer, баг из Фазы 1).

### 2.6 ResponseStats.vue — статистика

Простая панель над списком карточек.

Для каждого региона оператора:
```
| Регион   | В очереди | У меня | Обработано сегодня |
| Ростов   | 12        | 3      | 7                  |
| Москва   | 5         | 0      | 2                  |
```

PrimeVue DataTable (compact, no pagination).

---

## 3. COMPOSABLE `useResponsesApi.ts`

```typescript
// GET /api/operator/responses/my-cards
export function getMyCards(): Promise<ResponseListOut>

// PUT /api/operator/responses/{id}/status
export function updateCardStatus(assignmentId: number, status: string, comment?: string): Promise<void>

// POST /api/operator/responses/{id}/record
export function recordCandidate(assignmentId: number, data: ResponseRecordRequest): Promise<{ application_id: number }>

// GET /api/operator/responses/{id}/dialog
export function getCardDialog(assignmentId: number): Promise<DialogMessage[]>

// GET /api/operator/responses/stats
export function getResponseStats(): Promise<RegionStats[]>
```

Базовый URL: `/api/operator/responses/`. Авторизация через cookie (как в `useOperatorApi.ts`).

---

## 4. АДМИНКА РЕГИОНОВ

### 4.1 Маршрут

В `frontend/src/modules/admin/routes.ts` добавить:
```typescript
{
    path: '/admin/regions',
    name: 'admin-regions',
    component: () => import('./components/RegionsPage.vue'),
    meta: { permission: 'admin.regions.view' }
}
```

В sidebar (`AppSidebar.vue`) добавить пункт "Регионы" в секцию "Администрирование".

### 4.2 RegionsPage.vue

**Список регионов** — DataTable: название, часовой пояс, активен, действия (редактировать, расписание, операторы).

**Создание/редактирование** — Dialog с формой (название, часовой пояс, is_active).

**Табы внутри региона:**
- "Расписание" — таблица 7 дней, для каждого: рабочий (checkbox), время начала, время окончания
- "Операторы" — список привязанных операторов + кнопка "Добавить" (dropdown пользователей с ролью operator)
- "Коды вакансий" — список vacancy_code привязанных к этому региону + добавление

### 4.3 VacancyCodeMapping.vue

**Общая таблица маппинга** (не привязанная к конкретному региону):
- DataTable: vacancy_code, регион, дата добавления
- Добавление: ввод кода + выбор региона из dropdown
- Bulk import (будущее): загрузка из Google Docs / CSV

---

## 5. ТИПЫ TypeScript

Создать `frontend/src/modules/operator/types/responses.ts`:

```typescript
export interface ResponseCard {
    assignment_id: number
    handover_card_id: number
    region_name: string
    status: 'queued' | 'assigned' | 'not_suitable' | 'refused' | 'thinking' | 'recorded' | 'expired'
    candidate_name: string | null
    candidate_phone: string | null
    candidate_age: number | null
    candidate_city: string | null
    candidate_metro: string | null
    vacancy_title: string | null
    vacancy_code: string | null
    callback_slot: string | null
    dialog_summary: string | null
    result: string | null
    messages_count: number | null
    assigned_at: string | null
    created_at: string
}

export interface ResponseListResponse {
    cards: ResponseCard[]
    total_in_queue: number
    batch_size: number
}

export interface DialogMessage {
    sender: 'ai' | 'candidate'
    text: string
    created_at: string
}

export interface RegionStats {
    name: string
    queued: number
    my_assigned: number
    processed_today: number
}
```

---

## 6. СБОРКА И ДЕПЛОЙ

```bash
cd /var/www/new.kadry-24.ru/frontend
npm run build

# Перезапустить бэкенд
sudo systemctl restart k24-main-api
```

---

## 7. ПРОВЕРКА

### Чеклист:
- [ ] Вкладка "Отклики" отображается в OperatorPage (только с permission)
- [ ] При открытии — загружаются карточки из очереди регионов оператора
- [ ] Карточка показывает все данные кандидата
- [ ] Кнопка копирования телефона работает
- [ ] Кнопки статусов работают (Confirm → PUT → Toast → карточка исчезает)
- [ ] "Думает" — карточка остаётся в списке
- [ ] Кнопка "Диалог" — модалка с перепиской
- [ ] Кнопка "Записать" — drawer с формой, автозаполнение из карточки
- [ ] После записи — карточка исчезает, запись в "Мои записи"
- [ ] Статистика очередей отображается
- [ ] Polling каждые 30 сек (только на активном табе)
- [ ] Админка регионов — CRUD, расписание, операторы, коды вакансий
- [ ] `appendTo="self"` на DatePicker/Select/Dropdown в drawer
- [ ] Нет ошибок в консоли браузера

---

## ⚠️ ВАЖНЫЕ МОМЕНТЫ

1. **appendTo="self"** на все popup-компоненты PrimeVue внутри drawers/dialogs
2. **Polling** — useIntervalFn из VueUse или setInterval с onUnmounted cleanup
3. **Конкурентность** — если два оператора одновременно запрашивают карточки из одного региона, бэкенд решает через FOR UPDATE. Фронтенду ничего специального делать не нужно
4. **Пустое состояние** — корректные сообщения для: нет регионов, нет карточек, не привязан к регионам
5. **Responsive** — карточки должны нормально выглядеть на планшете (операторы могут использовать)
