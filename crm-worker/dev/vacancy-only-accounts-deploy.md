# Deploy: Аккаунты Авито только для синка вакансий

**Дата:** 2026-03-19

## Что сделано

Добавлен флаг `ai_enabled` для аккаунтов Авито. Позволяет подключать аккаунты только для синка вакансий, без AI-автоответа и без регистрации вебхуков.

### Изменения

| Файл | Что изменено |
|------|-------------|
| `migrations/007_ai_enabled.sql` | Новое поле `ai_enabled` в `avito_accounts`, включено для аккаунтов 1 и 2 |
| `migrations/008_add_vacancy_accounts.sql` | Два новых аккаунта (Avito 8k2, Avito 6k2) с `ai_enabled=FALSE` |
| `models/db.py` | Поле `ai_enabled` в модели `AvitoAccount` |
| `api/webhooks.py` | Проверка `ai_enabled` перед созданием AI-сессии и запуском `process_new_application` |
| `workers/incoming_processor.py` | Проверка `ai_enabled` перед обработкой входящих сообщений через AI |
| `api/admin.py` | Поле `ai_enabled` в API (list/create/update), условие на регистрацию вебхуков |
| `templates/admin.html` | Чекбокс "AI-бот включён" в модалке, бейдж "AI" в таблице аккаунтов |

## Порядок деплоя

### 1. Применить миграции

```bash
mysql -u crm_avito -p crm_avito < migrations/007_ai_enabled.sql
mysql -u crm_avito -p crm_avito < migrations/008_add_vacancy_accounts.sql
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

### 4. Проверить

1. **Токены** -- в логах через 30 сек появятся записи `token_refreshed` для аккаунтов 3 и 4
2. **Синк вакансий** -- до 30 мин (или дождаться первого цикла), в логах `vacancies_sync_account_done account_id=3` и `account_id=4`
3. **AI НЕ отвечает** -- отправить тестовое сообщение на аккаунт 3 -- бот не должен реагировать
4. **Админка** -- открыть /admin, убедиться что новые аккаунты видны, у них нет бейджа "AI", вебхуки не зарегистрированы

## Откат

```sql
-- Удалить новые аккаунты
DELETE FROM avito_accounts WHERE client_id IN ('Np7wkUm_QJWnnklJhRN3', 'vqqYGCrM-evNERtPpUQz');

-- Удалить поле (при необходимости)
ALTER TABLE avito_accounts DROP COLUMN ai_enabled;
```

Перезапустить сервис после отката.
