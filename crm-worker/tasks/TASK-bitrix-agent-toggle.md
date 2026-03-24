# TASK: Интеграция переключения агентов Битрикс24 в CRM-воркер

## Контекст

На сервере Битрикс24 (`b24.kadry-24.ru`) развёрнут PHP-скрипт для управления агентами:
```
https://b24.kadry-24.ru/local/source/avito/agent_toggle.php?secret=K24agentToggle2026&id={ID}&active={Y|N}
```

Агенты, которые нужно переключать при смене режима работы:
- ID=258088 — функция `avito_post34()`, обработка откликов на стороне Битрикса
- ID=258109 — (ещё один агент, может добавиться позже)

Их нужно **выключать** при переключении на наш AI-воркер и **включать** при возврате на Битрикс.

Скрипт протестирован и работает:
```bash
# Выключить
curl 'https://b24.kadry-24.ru/local/source/avito/agent_toggle.php?secret=K24agentToggle2026&id=258088&active=N'
# Ответ: {"ok":true,"id":258088,"active":"N"}

# Включить
curl 'https://b24.kadry-24.ru/local/source/avito/agent_toggle.php?secret=K24agentToggle2026&id=258088&active=Y'
# Ответ: {"ok":true,"id":258088,"active":"Y"}
```

## Что нужно сделать

### 1. Добавить настройки в `config.py`

Новые переменные окружения:
```
BITRIX_AGENT_TOGGLE_URL=https://b24.kadry-24.ru/local/source/avito/agent_toggle.php
BITRIX_AGENT_SECRET=K24agentToggle2026
BITRIX_AGENT_IDS=258088,258109
```

В классе `Settings`:
```python
bitrix_agent_toggle_url: str = "https://b24.kadry-24.ru/local/source/avito/agent_toggle.php"
bitrix_agent_secret: str = "K24agentToggle2026"
bitrix_agent_ids_raw: str = Field(default="258088", alias="bitrix_agent_ids")

@property
def bitrix_agent_ids(self) -> list[int]:
    return [int(x.strip()) for x in self.bitrix_agent_ids_raw.split(",") if x.strip()]
```

### 2. Создать утилиту `services/bitrix_agent.py`

```python
async def toggle_bitrix_agents(active: bool) -> list[dict]:
    """
    Включает/выключает ВСЕ агенты Битрикс24 из списка settings.bitrix_agent_ids.
    active=True -> Y (включить), active=False -> N (выключить)
    Возвращает список результатов по каждому агенту.
    """
    results = []
    for agent_id in settings.bitrix_agent_ids:
        result = await _toggle_one(agent_id, active)
        results.append(result)
    return results


async def _toggle_one(agent_id: int, active: bool) -> dict:
    """
    Переключает один агент. При ошибке — логирует warning, не падает.
    """
    params = {
        "secret": settings.bitrix_agent_secret,
        "id": agent_id,
        "active": "Y" if active else "N",
    }
    async with httpx.AsyncClient(timeout=15) as client:
        resp = await client.get(settings.bitrix_agent_toggle_url, params=params)
        return resp.json()
```

- При ошибке одного агента — логировать warning, продолжать остальные
- НЕ прерывать основной flow переключения вебхуков

### 3. Обновить `scripts/webhook_enable_ours.py`

При переключении на наш воркер — **выключить** все агенты Битрикса:
- После unsubscribe Bitrix вебхука и subscribe нашего
- Вызвать `await toggle_bitrix_agents(active=False)`
- Вывести результат по каждому агенту

### 4. Обновить `scripts/webhook_enable_bitrix.py`

При возврате на Битрикс — **включить** все агенты Битрикса:
- После unsubscribe нашего вебхука и subscribe Битрикса
- Вызвать `await toggle_bitrix_agents(active=True)`
- Вывести результат по каждому агенту

## Существующие файлы для изменения

- `config.py` — добавить 3 переменные (url, secret, ids через запятую)
- `scripts/webhook_enable_ours.py` — добавить вызов отключения агентов
- `scripts/webhook_enable_bitrix.py` — добавить вызов включения агентов

## Новые файлы

- `services/bitrix_agent.py` — утилита для переключения агентов

## Важно

- Список агентов хранить в `.env` через запятую: `BITRIX_AGENT_IDS=258088,258109`
- Легко добавить новый агент — просто дописать ID через запятую в `.env`
- Секрет `K24agentToggle2026` хранить в `.env`, не хардкодить
- Вызов agent_toggle — через httpx, как остальные HTTP-вызовы в проекте
- При ошибке вызова одного агента — логировать warning, продолжать остальные, НЕ прерывать основной flow
- ACCOUNT_ID для скриптов остаётся = 2 (аккаунт Лавка)
