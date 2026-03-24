# Deploy: Ispravlenie retry i tajmautov v ai_claude.py

## Izmenennyj fajl

- `services/ai_claude.py`

## Chto sdelano

### 1. _is_retryable() — raspoznayot setevye oshibki

Ranee funktsiya proverila tolko HTTP-statusy (429, 500...). "Server disconnected without sending a response" — eto `httpx.RemoteProtocolError`, ne HTTP-otvet, poetomu retry ne srabatyvalo (1 popytka vmesto 5).

Teper proverka vklyuchaet:
- Tipy isklyuchenij: `ConnectError`, `ConnectTimeout`, `ReadTimeout`, `WriteTimeout`, `RemoteProtocolError`, `CloseError`, `ConnectionError`, `OSError`
- Klyuchevye frazy: "disconnected", "connection reset", "timed out", "broken pipe" i dr.
- HTTP-statusy: 429, 500, 502, 503, 529

### 2. Razdelnye tajmauty

| Parametr | Bylo | Stalo |
|----------|------|-------|
| connect  | 60s (obshchij) | 15s |
| read     | 60s (obshchij) | 120s |
| write    | 60s (obshchij) | 30s |

Primeneno v `_claude_client()` i `_call_openai_fallback()`.

### 3. Retry dlya OpenAI fallback

Ranee: 1 popytka, pri oshibke — isklyuchenie.
Teper: 3 popytki s pauzami 3, 10, 30 sek.

## Ozhidaemoe povedenie

1. Pri "Server disconnected" — retry do 5 popytok Claude (pauzy 2, 5, 15, 30, 60 sek)
2. Esli vse 5 ne udalis — OpenAI cherez proksi (do 3 popytok, pauzy 3, 10, 30 sek)
3. Esli i OpenAI ne udalos — oshibka

## Proverka

```bash
sudo systemctl restart k24-crm-worker
journalctl -u k24-crm-worker -f --no-pager | grep -E "claude_retry|openai|fallback|disconnected"
```

## Restart

```bash
sudo systemctl restart k24-crm-worker
```
