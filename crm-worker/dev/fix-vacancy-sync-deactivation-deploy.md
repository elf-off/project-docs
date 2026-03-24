# Deploy: Исправление ложной деактивации вакансий при синке

**Дата:** 2026-03-19

## Что сделано

Убрана автоматическая деактивация вакансий (`is_active=False`) после синка. API Avito может возвращать неполный список (пагинация/лимиты), из-за чего реально существующие вакансии ложно помечались как неактивные.

### Изменения

| Файл | Что изменено |
|------|-------------|
| `services/vacancy_sync.py` | Удален блок деактивации (строки ~286-295), заменен на лог-сообщение |

## Порядок деплоя

Миграции не требуются.

```bash
cd /opt/crm-worker
git pull
sudo systemctl restart k24-crm-worker
```

## После деплоя: восстановить деактивированные вакансии

```sql
UPDATE vacancies SET is_active = 1 WHERE account_id IN (3, 4) AND is_active = 0;
```

## Проверка

1. Подождать 30 мин (или дождаться синка)
2. Проверить: `SELECT is_active, COUNT(*) FROM vacancies WHERE account_id=3 GROUP BY is_active;`
3. Все вакансии должны остаться `is_active=1`
4. В логах должно быть `vacancy_deactivation_skipped reason=disabled_for_safety`
