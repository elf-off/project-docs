# ЗАДАЧА: Исправить дубликат телефона в эмуляторе тестирования

## Контекст

Проект: `/opt/openai/crm-worker/`
Ветка: `main`

## Проблема

При повторном запуске эмуляции с тем же номером телефона — 500 ошибка:

```
sqlalchemy.exc.IntegrityError: (1062, "Duplicate entry '+79991234567' for key 'uq_phone'")
```

**Файл:** `api/admin.py`, строка ~705 (`emulate_application`)

Код создаёт нового `Applicant` без проверки, существует ли уже запись с таким phone.

## Решение

В эндпоинте `emulate_application` (файл `api/admin.py`) **перед** созданием Applicant — проверить, есть ли уже applicant с таким телефоном. Если есть — использовать существующего.

```python
# Ищем существующего applicant по телефону
applicant = None
if candidate_phone:
    existing = await session.execute(
        select(Applicant).where(Applicant.phone == candidate_phone)
    )
    applicant = existing.scalar_one_or_none()

# Если нет — создаём нового
if not applicant:
    applicant = Applicant(
        name=candidate_name or None,
        phone=candidate_phone or None,
    )
    session.add(applicant)
    await session.flush()
else:
    # Обновить имя если изменилось
    if candidate_name and applicant.name != candidate_name:
        applicant.name = candidate_name
```

Это точно такая же логика как в `_process_new_application` в `api/webhooks.py` (строки ~260-270) — там дублирование телефона уже обрабатывается. Нужно сделать аналогично в эмуляторе.

## Также

При повторной эмуляции — тестовые `Application`, `Chat`, `AISession` создаются заново (каждый раз новый диалог). Это правильное поведение — менять не нужно. Проблема только в Applicant.

## Проверка

1. Запустить эмуляцию с телефоном `+79991234567`
2. Дождаться завершения (или прервать)
3. Запустить эмуляцию **ещё раз** с тем же телефоном `+79991234567`
4. Не должно быть 500 ошибки — должен создаться новый чат с тем же applicant

## Файлы для изменения

| Файл | Действие |
|------|----------|
| `api/admin.py` | В `emulate_application` — добавить проверку существующего applicant по phone перед INSERT |
