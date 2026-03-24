# Deploy: 4 fiksa Telegram-uvedomlenij

## Izmenennyj fajl

- `services/telegram_notifier.py`

## Chto sdelano

### Fiks 1: Utrennyaya svodka — tolko kartochki za noch
- Dobavlen filtr po vremeni: `created_at` mezhdu 21:00 vchera i 09:00 segodnya (MSK -> UTC)
- Svodka teper grupiruetsya po akkauntam (kazhdyj akkaunt poluchaet svoi kartochki v svoj topik)

### Fiks 2: Otpravka v pravilnyj topik (message_thread_id)
- `send_telegram_message()` teper prinimaet `topic_id: int | None` vmesto `chat_id: str`
- `chat_id` vsegda beryotsya iz `settings.telegram_group_id` i konvertiruetsya v `int`
- `message_thread_id` dobavlyaetsya v payload tolko esli `topic_id` zadan i > 0
- Utrennyaya svodka peredayot `account.telegram_topic_id` dlya kazhdogo akkaunta

### Fiks 3: Proksi dlya Telegram API
- `httpx.AsyncClient` teper ispolzuet `proxy=settings.claude_proxy` (socks5://127.0.0.1:1080)

### Fiks 4: Bag test-summary endpoint
- `send_morning_summary()` teper prinimaet opcionalnyj `account_id: int | None = None`
- Esli `account_id` zadan — filtruet tolko po etomu akkauntu
- Eto ispravlyaet `TypeError` v `api/admin.py` endpoint `/api/telegram/test-summary`

## Proverka pered deploy

```bash
# Import check
python -c "from services.telegram_notifier import send_morning_summary, send_telegram_message; print('OK')"

# Proverka proksi
grep -n "proxy\|claude_proxy" services/telegram_notifier.py

# Proverka topic_id
grep -n "message_thread_id\|topic_id" services/telegram_notifier.py

# Proverka filtra po vremeni
grep -n "night_start\|night_end" services/telegram_notifier.py

# Proverka signatury
grep -n "def send_morning_summary" services/telegram_notifier.py
```

## Zavisimost

- `pytz` — dolzhen byt v `requirements.txt` (proverit: `pip show pytz`)

## Testirovanie

1. Vyzvat endpoint: `POST /api/telegram/test-summary` — proverit chto net TypeError
2. Vyzvat s parametrom: `POST /api/telegram/test-summary?account_id=1` — proverit filtraciya po akkauntu
3. Proverit chto soobshchenie prishlo v pravilnyj topik gruppy, a ne v obshchij chat
4. Proverit chto soobshchenie doshlo cherez proksi (esli Telegram zablokirovan)

## Restart

```bash
sudo systemctl restart k24-crm-worker
```
