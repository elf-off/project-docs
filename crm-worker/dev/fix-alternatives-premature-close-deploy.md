# Deploy: Исправление преждевременного закрытия сессии при запросе доп. вакансий

**Дата:** 2026-03-19

## Что сделано

Исправлена проблема: когда кандидат на этапе `alternatives` просил ещё варианты ("А есть ли ещё вакансии?"), бот задавал уточняющий вопрос, но при этом уже закрывал сессию (`status=completed, result=rejected`). Ответ кандидата после этого не обрабатывался.

### Корневая причина

В `_handle_alternatives_response()` intent парсился только как `ok`/`no`/`unclear`. Запрос "ещё варианты" распознавался как `unclear` или `no` и уходил в ветку возражений/закрытия.

### Решение

- Новый промпт `ALTERNATIVES_INTENT_PROMPT` с 4 вариантами: `selected`, `more`, `no`, `unclear`
- Intent `more` -- кандидат хочет ещё, бот спрашивает критерии, сессия остается active
- Новый stage `waiting_alternatives_criteria` -- ожидание критериев от кандидата
- Новый handler `_handle_alternatives_criteria` -- поиск в Qdrant по критериям, показ новых альтернатив

### Изменения

| Файл | Что изменено |
|------|-------------|
| `migrations/009_alternatives_criteria_stage.sql` | ALTER TABLE ai_sessions -- новое значение enum |
| `models/db.py` | `waiting_alternatives_criteria` в enum dialog_stage |
| `services/ai_agent.py` | Новый промпт, переписан `_handle_alternatives_response`, новый `_handle_alternatives_criteria`, регистрация stage в роутере |

### Новый flow

```
alternatives (кандидат отвечает)
  |-- intent=selected --> booking
  |-- intent=more --> спросить критерии --> waiting_alternatives_criteria
  |     |-- Qdrant нашел --> показать новые альтернативы --> alternatives
  |     |-- Qdrant пусто --> "вакансий больше нет" --> handover (no_match)
  |-- intent=no --> handover (rejected)
  |-- intent=unclear --> возражение (1 попытка) --> handover (rejected)
```

## Порядок деплоя

### 1. Применить миграцию

```bash
mysql -u crm_avito -p crm_avito < migrations/009_alternatives_criteria_stage.sql
```

### 2. Обновить код

```bash
cd /opt/crm-worker
git pull
```

### 3. Перезапустить сервис

```bash
sudo systemctl restart k24-crm-worker
```

## Проверка

### Сценарий 1: Кандидат просит ещё
1. Эмуляция --> ответить "45, Москва" --> на презентации "НЕТ"
2. Бот покажет альтернативы
3. Ответить "А есть ли ещё вакансии?"
4. Сессия должна остаться active, stage = `waiting_alternatives_criteria`
5. Ответить "Близость к дому"
6. Бот должен найти подходящие вакансии и предложить новые варианты

### Сценарий 2: Точный отказ
1. На альтернативах ответить "Нет, ничего не подходит"
2. Сессия закрывается, handover с result=rejected

### Сценарий 3: Вакансий больше нет
1. Кандидат просит ещё --> уточняет критерии
2. Qdrant не находит новых вакансий
3. "Подходящих вакансий больше нет, передам менеджеру" --> handover (no_match)
