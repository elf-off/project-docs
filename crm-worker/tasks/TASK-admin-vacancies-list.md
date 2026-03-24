# TASK: Список вакансий в админке

**Приоритет:** Средний
**Дата:** 2026-03-19

---

## 1. Контекст

В админке (`https://babito.kadry-24.ru/admin/`) нет раздела с вакансиями. После подключения новых аккаунтов (только синк вакансий) нужно видеть какие вакансии загрузились, с какого аккаунта, активны ли, проиндексированы ли в Qdrant.

---

## 2. Что нужно сделать

### 2.1. API-эндпоинт: `GET /admin/api/vacancies`

Файл: `api/admin.py`

**Параметры:**
- `account_id` (int, optional) — фильтр по аккаунту
- `is_active` (bool, optional) — фильтр по активности
- `search` (str, optional) — поиск по названию вакансии
- `limit` (int, default=100) — лимит
- `offset` (int, default=0) — сдвиг

**Логика:**

```python
@router.get("/api/vacancies", dependencies=[Depends(_verify_admin)])
async def list_vacancies(
    account_id: Optional[int] = None,
    is_active: Optional[bool] = None,
    search: Optional[str] = None,
    limit: int = 100,
    offset: int = 0,
):
    async with AsyncSessionFactory() as session:
        query = (
            select(Vacancy, AvitoAccount.account_name)
            .outerjoin(AvitoAccount, Vacancy.account_id == AvitoAccount.id)
            .order_by(Vacancy.updated_at.desc())
        )
        count_query = select(func.count(Vacancy.id))

        if account_id:
            query = query.where(Vacancy.account_id == account_id)
            count_query = count_query.where(Vacancy.account_id == account_id)
        if is_active is not None:
            query = query.where(Vacancy.is_active == is_active)
            count_query = count_query.where(Vacancy.is_active == is_active)
        if search:
            query = query.where(Vacancy.title.ilike(f"%{search}%"))
            count_query = count_query.where(Vacancy.title.ilike(f"%{search}%"))

        total_result = await session.execute(count_query)
        total = total_result.scalar() or 0

        query = query.limit(min(limit, 500)).offset(offset)
        result = await session.execute(query)
        rows = result.all()

    return {
        "total": total,
        "vacancies": [
            {
                "id": v.id,
                "avito_vacancy_id": v.avito_vacancy_id,
                "title": v.title,
                "city": v.city,
                "address": v.address,
                "business_area": v.business_area,
                "profession": v.profession,
                "schedule": v.schedule,
                "salary_raw": v.salary_raw,
                "salary_from": v.salary_from,
                "salary_to": v.salary_to,
                "is_active": v.is_active,
                "embedding_indexed": v.embedding_indexed,
                "account_id": v.account_id,
                "account_name": acct_name or "",
                "avito_url": v.avito_url,
                "last_synced_at": str(v.last_synced_at) if v.last_synced_at else None,
                "created_at": str(v.created_at) if v.created_at else None,
            }
            for v, acct_name in rows
        ],
    }
```

### 2.2. API-эндпоинт: `GET /admin/api/vacancies/stats`

Краткая статистика для шапки раздела:

```python
@router.get("/api/vacancies/stats", dependencies=[Depends(_verify_admin)])
async def vacancies_stats():
    async with AsyncSessionFactory() as session:
        # Общее количество
        total = await session.execute(select(func.count(Vacancy.id)))

        # Активные
        active = await session.execute(
            select(func.count(Vacancy.id)).where(Vacancy.is_active == True)
        )

        # Проиндексированные в Qdrant
        indexed = await session.execute(
            select(func.count(Vacancy.id)).where(Vacancy.embedding_indexed == True)
        )

        # По аккаунтам
        per_account = await session.execute(
            select(
                AvitoAccount.id,
                AvitoAccount.account_name,
                func.count(Vacancy.id).label("count"),
                func.sum(case((Vacancy.is_active == True, 1), else_=0)).label("active_count"),
            )
            .outerjoin(Vacancy, Vacancy.account_id == AvitoAccount.id)
            .where(AvitoAccount.is_active == True)
            .group_by(AvitoAccount.id, AvitoAccount.account_name)
        )
        accounts_stats = per_account.all()

    return {
        "total": total.scalar() or 0,
        "active": active.scalar() or 0,
        "indexed": indexed.scalar() or 0,
        "by_account": [
            {
                "account_id": row[0],
                "account_name": row[1] or "",
                "total": row[2] or 0,
                "active": row[3] or 0,
            }
            for row in accounts_stats
        ],
    }
```

**Импорт:** добавить `from sqlalchemy import case` если ещё не импортирован.

### 2.3. Обновить `templates/admin.html` — вкладка "Вакансии"

Добавить новую вкладку (tab) в админке — **"Вакансии"** — рядом с существующими (Аккаунты, Карточки, Лог событий).

**Содержимое вкладки:**

1. **Шапка со статистикой** — карточки:
   - "Всего вакансий: 150"
   - "Активных: 120"
   - "В Qdrant: 95"
   - По аккаунтам: "Лавка: 45 | Avito 8к2: 60 | Avito 6к2: 45"

2. **Фильтры** (одна строка):
   - Dropdown "Аккаунт" — все / конкретный
   - Dropdown "Статус" — все / активные / неактивные
   - Поле "Поиск по названию"

3. **Таблица вакансий:**

| Название | Город | Профессия | Зарплата | Аккаунт | Qdrant | Синк | Статус |
|----------|-------|-----------|----------|---------|--------|------|--------|
| Упаковщик... | Москва | Упаковщик | 3000/смена | Лавка | ✅ | 2 мин назад | Активна |
| Курьер... | Ростов | Курьер | 2500/смена | 8к2 | ❌ | 15 мин назад | Активна |

   - Колонка "Зарплата": показывать `salary_raw` или `salary_from-salary_to` если есть
   - Колонка "Qdrant": ✅ если `embedding_indexed=true`, ❌ если нет
   - Колонка "Синк": `last_synced_at` в формате "X мин назад" (Moscow time)
   - Колонка "Статус": бейдж "Активна" (зелёный) / "Неактивна" (серый)
   - При клике на строку — модальное окно с полной информацией о вакансии (описание, адрес, график, задачи и т.д.)

4. **Пагинация** — если вакансий много

**Стиль** — в том же стиле что остальная админка (уже есть таблицы для карточек, событий). Использовать `toMoscow()` для дат.

---

## 3. Файлы для изменения

| Файл | Действие |
|------|----------|
| `api/admin.py` | Добавить 2 эндпоинта (`/api/vacancies`, `/api/vacancies/stats`) |
| `templates/admin.html` | Новая вкладка "Вакансии" с таблицей, фильтрами и статистикой |

---

## 4. Тестирование

1. Открыть админку → вкладка "Вакансии"
2. Проверить что вакансии отображаются с правильными аккаунтами
3. Фильтр по аккаунту — показывает только вакансии выбранного
4. Поиск по названию работает
5. Статистика в шапке совпадает с реальными данными
6. Модальное окно при клике на вакансию показывает полную информацию
