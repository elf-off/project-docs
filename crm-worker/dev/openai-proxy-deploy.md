# Deploy: Proksi dlya OpenAI fallback

## Izmenennyj fajl

- `services/ai_claude.py` (stroka 62)

## Chto sdelano

Dobavlen `proxy=settings.claude_proxy` v `httpx.AsyncClient` vnutri `_call_openai_fallback()`.

Ranee OpenAI fallback sozdaval klient bez proksi, chto privodilo k 403 Forbidden na servere gde OpenAI API zablokirovan. Teper ispolzuetsya tot zhe SOCKS5-proksi (socks5://127.0.0.1:1080), chto i dlya Claude API.

## Proverka

```bash
# Obe funkcii dolzhny ispolzovat proxy
grep -n "proxy" services/ai_claude.py
```

## Restart

```bash
sudo systemctl restart k24-crm-worker
```
