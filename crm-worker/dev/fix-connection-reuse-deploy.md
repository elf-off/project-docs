# Deploy: Ispravlenie pereipolzovaniya soedinenij v ai_claude.py

## Izmenennyj fajl

- `services/ai_claude.py`

## Problema

Vse 5 popytok Claude padali s odnoj oshibkoj "Server disconnected without sending a response", potomu chto httpx.AsyncClient sozdavalsya ODIN RAZ na vse retry. Server zakryval TCP-soedinenie posle pervogo zaprosa, a httpx pytalsya pereipolzovat myortvoe soedinenie.

## Chto sdelano

Perenesli sozdanie klienta (`async with _claude_client() as client:`) VNUTR tsikla `for attempt in range(MAX_RETRIES)`. Teper kazhdaya popytka sozdayot novoe TCP-soedinenie.

**Bylo:**
```python
async with _claude_client() as client:      # odin klient
    for attempt in range(MAX_RETRIES):       # vse retry cherez nego
        resp = await client.post(...)
```

**Stalo:**
```python
for attempt in range(MAX_RETRIES):
    async with _claude_client() as client:   # novyj klient na kazhduyu popytku
        resp = await client.post(...)
```

OpenAI fallback uzhe sozdaval novyj klient pri kazhdom vyzove — ne menялos.

## Proverka

```bash
sudo systemctl restart k24-crm-worker
journalctl -u k24-crm-worker -f --no-pager | grep -E "claude_retry|claude_response|200 OK"
```

Ozhidanie: esli pervaya popytka upala — vtoraya dolzhna projti (svezhee soedinenie).

## Restart

```bash
sudo systemctl restart k24-crm-worker
```
