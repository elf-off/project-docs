# ЗАДАЧА: Исправить привязку вакансии в эмуляторе тестирования AI

## Контекст

Проект: `/opt/openai/crm-worker/`
Ветка: `main`

## Проблема

При тестировании AI-бота через веб-интерфейс эмулятора (вкладка "Тестирование" в админке), если выбрана вакансия "Случайная" — AI на этапе презентации (`waiting_fork`) пишет:

> "К сожалению, у меня сейчас нет подробной информации по этой вакансии — детали уточню у коллег и напишу вам."

Причина: при создании тестовой заявки (Application) поле `avito_vacancy_id` не заполняется или заполняется некорректно. Когда `ai_agent.py` → `_get_vacancy_data()` пытается найти вакансию через `ensure_vacancy()`, она не находит данных → `vacancy_data = None` → презентация пустая.

## Что нужно исправить

### 1. Эндпоинт эмуляции (api/admin.py или где он сейчас находится)

Найди эндпоинт, который обрабатывает запуск тестовой эмуляции (создаёт тестовую Application, Chat, AISession).

**При выборе "Случайная" вакансия:**
- Выбрать случайную АКТИВНУЮ вакансию из таблицы `vacancies` для выбранного `account_id`
- Записать её `avito_vacancy_id` в `Application.avito_vacancy_id`
- Также записать `avito_item_id` = str(avito_vacancy_id)

```python
from sqlalchemy import select, func
from models.db import Vacancy

# Случайная активная вакансия для аккаунта
result = await session.execute(
    select(Vacancy)
    .where(
        Vacancy.account_id == account_id,
        Vacancy.is_active == True,
    )
    .order_by(func.rand())  # MariaDB: RAND()
    .limit(1)
)
vacancy = result.scalar_one_or_none()

if vacancy:
    application.avito_vacancy_id = vacancy.avito_vacancy_id
    application.avito_item_id = str(vacancy.avito_vacancy_id)
```

**При выборе конкретной вакансии из dropdown:**
- Аналогично: записать `avito_vacancy_id` и `avito_item_id` из выбранной вакансии

### 2. Dropdown вакансий в HTML (templates/admin.html)

Если dropdown вакансий ещё не загружает реальные вакансии из БД — добавить:

**API-эндпоинт** (если нет): `GET /admin/api/vacancies/list?account_id=X` — возвращает список вакансий для аккаунта:

```python
@router.get("/api/vacancies/list", dependencies=[Depends(_verify_admin)])
async def list_vacancies_for_account(account_id: int):
    async with AsyncSessionFactory() as session:
        result = await session.execute(
            select(Vacancy)
            .where(
                Vacancy.account_id == account_id,
                Vacancy.is_active == True,
            )
            .order_by(Vacancy.title)
        )
        vacancies = result.scalars().all()
    
    return [
        {
            "id": v.id,
            "avito_vacancy_id": v.avito_vacancy_id,
            "title": v.title or f"Вакансия {v.avito_vacancy_id}",
            "city": v.city or "",
        }
        for v in vacancies
    ]
```

**В HTML** — при смене аккаунта загружать вакансии в dropdown:

```javascript
// При выборе аккаунта — обновить список вакансий
accountSelect.addEventListener('change', async () => {
    const accountId = accountSelect.value;
    const resp = await fetch(`/admin/api/vacancies/list?account_id=${accountId}`);
    const vacancies = await resp.json();
    
    vacancySelect.innerHTML = '<option value="random">Случайная</option>';
    vacancies.forEach(v => {
        vacancySelect.innerHTML += `<option value="${v.avito_vacancy_id}">${v.title} (${v.city})</option>`;
    });
});
```

### 3. Передача vacancy_id при запуске эмуляции

При отправке формы эмуляции — передавать `vacancy_id` в запрос:

```javascript
const body = {
    account_id: accountSelect.value,
    candidate_name: nameInput.value,
    candidate_phone: phoneInput.value,
    vacancy_id: vacancySelect.value,  // "random" или avito_vacancy_id
};
```

В эндпоинте:

```python
vacancy_id = req.vacancy_id  # "random" или число

if vacancy_id == "random" or not vacancy_id:
    # Случайная вакансия (код выше)
else:
    # Конкретная вакансия
    result = await session.execute(
        select(Vacancy).where(
            Vacancy.avito_vacancy_id == int(vacancy_id),
            Vacancy.is_active == True,
        )
    )
    vacancy = result.scalar_one_or_none()
    if vacancy:
        application.avito_vacancy_id = vacancy.avito_vacancy_id
        application.avito_item_id = str(vacancy.avito_vacancy_id)
```

## Порядок работы

1. Прочитай текущий код эндпоинта эмуляции в `api/admin.py` (или где он находится)
2. Прочитай шаблон `templates/admin.html` — секцию тестирования
3. Прочитай `services/ai_agent.py` → функцию `_get_vacancy_data()` — чтобы понять как именно vacancy загружается
4. Примени исправления
5. Проверь что dropdown вакансий загружается при смене аккаунта

## Проверка

1. Открыть админку → Тестирование
2. Выбрать аккаунт "Avito 8к2"
3. Выбрать "Случайная" вакансию
4. Запустить эмуляцию
5. Ответить "45, Москва"
6. **Ожидаемый результат**: AI должен написать РЕАЛЬНУЮ презентацию вакансии с деталями (адрес, график, зарплата и т.д.), а НЕ "К сожалению, у меня нет информации"
7. Повторить с конкретной вакансией из dropdown — тоже должна быть полная презентация

## Файлы для изменения

| Файл | Действие |
|------|----------|
| `api/admin.py` | Исправить эндпоинт эмуляции — привязка vacancy_id к Application |
| `api/admin.py` | Добавить `GET /admin/api/vacancies/list` если нет |
| `templates/admin.html` | Динамическая загрузка вакансий при смене аккаунта |
