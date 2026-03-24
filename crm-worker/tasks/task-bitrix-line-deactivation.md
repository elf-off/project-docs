# TASK: Добавить деактивацию/активацию открытой линии Битрикс через curl в скрипты переключения

## Файлы на сервере `babito.kadry-24.ru`

- `/opt/openai/crm-worker/webhook_enable_ours.py`
- `/opt/openai/crm-worker/webhook_enable_bitrix.py`

## Проблема

Вызов `imopenlines.config.update` через httpx с `json=` не работает — Битрикс не парсит вложенные `PARAMS` из JSON. Через curl с `-H "Content-Type: application/json"` работает (проверено вручную). Также нужно добавить управление параметром `ACTIVE` линии — без него линия Битрикс продолжает создавать лиды даже когда вебхук переключён на нашего бота.

## Что сделать

В обоих скриптах есть блок `try/except` с вызовом `imopenlines.config.update` через httpx. Заменить этот блок на вызов curl через `subprocess.run`.

### 1. `webhook_enable_ours.py`

Добавить `import subprocess` в начало файла.

Заменить блок `try/except` с httpx-вызовом `imopenlines.config.update` на:

```python
r3 = subprocess.run([
    "curl", "-s", "-X", "POST",
    f"{BITRIX_REST}/imopenlines.config.update",
    "-H", "Content-Type: application/json",
    "-d", '{"CONFIG_ID": ' + str(BITRIX_LINE_ID) + ', "PARAMS": {"ACTIVE": "N", "CRM": "N", "CRM_CREATE": "none"}}'
], capture_output=True, text=True)
print(f"Bitrix line OFF + CRM off: {r3.stdout}")
```

Обновить финальный print:

```
Готово: Битрикс отключён, наш включён, линия деактивирована, CRM выключена.
```

### 2. `webhook_enable_bitrix.py`

Добавить `import subprocess` в начало файла.

Заменить блок `try/except` с httpx-вызовом `imopenlines.config.update` на:

```python
r3 = subprocess.run([
    "curl", "-s", "-X", "POST",
    f"{BITRIX_REST}/imopenlines.config.update",
    "-H", "Content-Type: application/json",
    "-d", '{"CONFIG_ID": ' + str(BITRIX_LINE_ID) + ', "PARAMS": {"ACTIVE": "Y", "CRM": "Y"}}'
], capture_output=True, text=True)
print(f"Bitrix line ON + CRM on: {r3.stdout}")
```

Обновить финальный print:

```
Готово: наш отключён, Битрикс включён, линия активна, CRM включена.
```

## Константы (уже есть в скриптах, не менять)

```python
BITRIX_REST = "https://b24.kadry-24.ru/rest/8539/3j61afi1yqk890du"
BITRIX_LINE_ID = 12
```

## Примечания

- Блок `try/except` с httpx для Битрикс-вызова больше не нужен — curl не кидает исключений, ошибки видны в stdout.
- Остальные httpx-вызовы (Avito webhook subscribe/unsubscribe) оставить как есть — они работают нормально.
