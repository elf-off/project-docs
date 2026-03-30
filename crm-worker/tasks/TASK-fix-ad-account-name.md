# TASK: Исправить отображение названия аккаунта в dropdown анализа объявлений

**Дата:** 2026-03-30
**Приоритет:** Низкий (hotfix, уже применён на проде)
**Проект:** CRM Worker

---

## Проблема

В dropdown выбора аккаунта на вкладке "Анализ объявлений" показывается `client_id` вместо человекочитаемого названия. Причина: API `/admin/accounts` возвращает поле `"name"`, а UI ищет `a.account_name`.

## Исправление

**Файл:** `templates/admin.html`

Найти строку (в функции `updateAdAccountFilter`):
```javascript
opt.textContent = a.account_name || a.client_id;
```

Заменить на:
```javascript
opt.textContent = a.account_name || a.name || a.client_id;
```

## Проверка

В браузере в dropdown должны отображаться: "Подработка 4 (Лавка)", "Avito 8k2", "Avito 6k2" — а не `vQ4J4O9k3l9ADKdt4I5S`.
