# Deploy: Udalenie SOCKS5-proksi iz HTTP-klientov

## Prichina

Marshrutizatsiya trafika k api.anthropic.com, api.openai.com i api.telegram.org teper nastroena na urovne MikroTik-routera (policy routing cherez VPN-tunnel). SOCKS5-proksi (microsocks na 127.0.0.1:1080) bolshe ne nuzhen.

## Izmenennye fajly

| Fajl | Chto udaleno |
|------|-------------|
| `services/ai_claude.py` | `proxy`, `limits`, `http1`, `http2` iz `_claude_client()` i `_call_openai_fallback()` |
| `services/ai_rag.py` | `proxy` iz `get_embedding()` |
| `services/telegram_notifier.py` | `proxy` iz `send_telegram_message()` |

## Chto sdelano

### 1. `services/ai_claude.py`

**`_claude_client()`** — ubran proxy, limits, http1/http2 (byli nuzhny dlya obkhoda problem s SOCKS5):

```python
# Bylo
httpx.AsyncClient(proxy=settings.claude_proxy, timeout=..., limits=..., http1=True, http2=False)

# Stalo
httpx.AsyncClient(timeout=httpx.Timeout(connect=15, read=120, write=30, pool=15))
```

**`_call_openai_fallback()`** — analogichno:

```python
# Bylo
httpx.AsyncClient(proxy=settings.claude_proxy, timeout=timeout, limits=..., http1=True, http2=False)

# Stalo
httpx.AsyncClient(timeout=timeout)
```

### 2. `services/ai_rag.py`

**`get_embedding()`** — ubran proxy dlya OpenAI Embeddings API:

```python
# Bylo
httpx.AsyncClient(proxy=settings.claude_proxy, timeout=30.0)

# Stalo
httpx.AsyncClient(timeout=30.0)
```

### 3. `services/telegram_notifier.py`

**`send_telegram_message()`** — ubran proxy dlya Telegram Bot API:

```python
# Bylo
httpx.AsyncClient(timeout=15.0, proxy=settings.claude_proxy)

# Stalo
httpx.AsyncClient(timeout=15.0)
```

## Chto NE menyalos

- Retry-logika (5 popytok Claude + 3 popytki OpenAI) — bez izmenenij
- `_is_retryable()` i `_RETRYABLE_EXCEPTIONS` — bez izmenenij
- Tajmauty (CONNECT_TIMEOUT, READ_TIMEOUT, WRITE_TIMEOUT) — bez izmenenij
- `_log_prompt()`, `call_claude()`, `ask_claude()` — bez izmenenij
- Raschet cost i usage — bez izmenenij
- Qdrant (ai_rag.py) — ispolzuet lokalnyj klient, ne httpx, ne tronuto

## Deploy

```bash
cd /opt/openai/crm-worker
git pull origin main
sudo systemctl restart k24-crm-worker
```

## Proverka

```bash
# Logi servisa — ubeditsya chto zaprosy prokhodyat bez oshibok
journalctl -u k24-crm-worker -f --no-pager | grep -E "claude_retry|claude_response|vacancy_parse|disconnected|openai_fallback|telegram_send"

# Proverit chto microsocks mozhno ostanovit (opcionalono, posle uspeshnogo testa)
# sudo systemctl stop microsocks
# sudo systemctl disable microsocks
```

## Ozhidaemoe povedenie

1. Zaprosy k Claude API — prokhodyat napriamuyu cherez router (VPN policy routing)
2. Zaprosy k OpenAI API (embeddings + fallback) — analogichno
3. Zaprosy k Telegram Bot API — napriamuyu
4. Oshibki "Server disconnected" dolzhny stat rezhe (ubran promezhutochnyj SOCKS5-hop)

## Otkat

Esli posle deploya zaprosy ne prokhodyat — vozmozhnye prichiny:

1. MikroTik policy routing ne nastroien — proverit `ip route` na routere
2. VPN-tunnel ne podnyat — proverit soedinenie

Dlya otkta: `git revert <commit>` i restart servisa.
