# ЗАДАЧА: Добавить переключение CRM Битрикс в скрипты переключения вебхуков

## Проблема

При переключении вебхука мессенджера на бота, Битрикс продолжает создавать лиды через открытую линию (это отдельная интеграция, не связанная с вебхуком). Нужно при переключении на бота глушить CRM в открытой линии, а при возврате на Битрикс — включать обратно.

## Что менять

### Файл: `./scripts/webhook_enable_ours.py`

**Добавить после подписки нашего вебхука:**

HTTP POST на Битрикс REST API, который отключает CRM в открытой линии:

```
POST https://b24.kadry-24.ru/rest/8539/3j61afi1yqk890du/imopenlines.config.update
Content-Type: application/json

{"CONFIG_ID": 12, "PARAMS": {"CRM": "N", "CRM_CREATE": "none"}}
```

Вывести результат: `print(f"Bitrix CRM off: {r3.status_code} {r3.text}")`

Итоговый вывод скрипта: `"\nГотово: Битрикс отключён, наш включён, CRM в линии выключена."`

---

### Файл: `./scripts/webhook_enable_bitrix.py`

**Добавить после подписки Битрикс-вебхука:**

HTTP POST на Битрикс REST API, который включает CRM обратно:

```
POST https://b24.kadry-24.ru/rest/8539/3j61afi1yqk890du/imopenlines.config.update
Content-Type: application/json

{"CONFIG_ID": 12, "PARAMS": {"CRM": "Y"}}
```

Вывести результат: `print(f"Bitrix CRM on: {r3.status_code} {r3.text}")`

Итоговый вывод скрипта: `"\nГотово: наш отключён, Битрикс включён, CRM в линии включена."`

---

## Константы для добавления (в оба файла)

```python
BITRIX_REST = "https://b24.kadry-24.ru/rest/8539/3j61afi1yqk890du"
BITRIX_LINE_ID = 12  # Открытая линия Авито для аккаунта Лавка (id=2)
```

## Важно

- Запрос к Битрикс REST API идёт через обычный HTTP (не через Avito токен), авторизация уже в URL (входящий вебхук Битрикс)
- Не использовать `headers` с Avito-токеном для запросов к Битрикс — это отдельный клиент
- Если запрос к Битрикс упал — вывести ошибку, но не прерывать скрипт (вебхук уже переключён, это доп. шаг)

## Проверка

```bash
# Переключить на бота
python3 ./scripts/webhook_enable_ours.py

# Ожидаемый вывод:
# Unsubscribe Bitrix: 200 ...
# Subscribe ours: 200 ...
# Bitrix CRM off: 200 {"result":true,...}
# Готово: Битрикс отключён, наш включён, CRM в линии выключена.

# Переключить обратно
python3 ./scripts/webhook_enable_bitrix.py

# Ожидаемый вывод:
# Unsubscribe ours: 200 ...
# Subscribe Bitrix: 200 ...
# Bitrix CRM on: 200 {"result":true,...}
# Готово: наш отключён, Битрикс включён, CRM в линии включена.
```
