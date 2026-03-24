# Deploy: Интеграция переключения агентов Битрикс24

**Дата:** 2026-03-19

## Что сделано

При переключении вебхуков между нашим воркером и Битриксом теперь автоматически включаются/выключаются все агенты Битрикс24 из настраиваемого списка. Поддержка нескольких агентов через запятую в `.env`.

### Изменения

| Файл | Что изменено |
|------|-------------|
| `config.py` | 3 настройки: `bitrix_agent_toggle_url`, `bitrix_agent_secret`, `bitrix_agent_ids` (список через запятую) |
| `services/bitrix_agent.py` | Переписан: `toggle_bitrix_agents()` итерирует по списку агентов, `_toggle_one()` переключает каждый |
| `scripts/webhook_enable_ours.py` | Вызов `toggle_bitrix_agents(active=False)` -- выключает все агенты |
| `scripts/webhook_enable_bitrix.py` | Вызов `toggle_bitrix_agents(active=True)` -- включает все агенты |

### Добавление нового агента

Просто дописать ID через запятую в `.env`:
```
BITRIX_AGENT_IDS=258088,258109,999999
```

## Порядок деплоя

Миграции не требуются.

### 1. Добавить/обновить переменные в `.env`

```
BITRIX_AGENT_TOGGLE_URL=https://b24.kadry-24.ru/local/source/avito/agent_toggle.php
BITRIX_AGENT_SECRET=K24agentToggle2026
BITRIX_AGENT_IDS=258088,258109
```

### 2. Обновить код

```bash
cd /opt/crm-worker
git pull
```

### 3. Перезапустить сервис (если нужно)

```bash
sudo systemctl restart k24-crm-worker
```

## Проверка

```bash
# Переключить на наш воркер
python3 scripts/webhook_enable_ours.py
# Вывод по каждому агенту:
# Bitrix agent OFF: {'ok': True, 'id': 258088, 'active': 'N'}
# Bitrix agent OFF: {'ok': True, 'id': 258109, 'active': 'N'}

# Вернуть на Битрикс
python3 scripts/webhook_enable_bitrix.py
# Bitrix agent ON: {'ok': True, 'id': 258088, 'active': 'Y'}
# Bitrix agent ON: {'ok': True, 'id': 258109, 'active': 'Y'}
```
