# Deploy: Deaktivatsiya/aktivatsiya otkrytoj linii Bitriks cherez curl

## Problema

Vyzov `imopenlines.config.update` cherez httpx s `json=` ne rabotaet — Bitriks ne parsit vlozhennye `PARAMS` iz JSON. Cherez curl s `-H "Content-Type: application/json"` rabotaet korrektno (provereno vruchnuyu).

Dopolnitelno: bez parametra `ACTIVE: "N"` liniya Bitriks prodolzhaet sozdavat lidy v CRM dazhe kogda webhook pereklyuchyon na nashego bota.

## Izmenennye fajly

| Fajl | Chto sdelano |
|------|-------------|
| `scripts/webhook_enable_ours.py` | Zamena httpx na curl, ACTIVE=N + CRM=N + CRM_CREATE=none |
| `scripts/webhook_enable_bitrix.py` | Zamena httpx na curl, ACTIVE=Y + CRM=Y |

## Chto izmeneno

### 1. `scripts/webhook_enable_ours.py` (pereklyuchenie na nashego bota)

Dobavlen `import subprocess`.

Blok `try/except` s httpx-vyzovom `imopenlines.config.update` zamenyoen na:

```python
r3 = subprocess.run([
    "curl", "-s", "-X", "POST",
    f"{BITRIX_REST}/imopenlines.config.update",
    "-H", "Content-Type: application/json",
    "-d", '{"CONFIG_ID": 12, "PARAMS": {"ACTIVE": "N", "CRM": "N", "CRM_CREATE": "none"}}'
], capture_output=True, text=True)
print(f"Bitrix line OFF + CRM off: {r3.stdout}")
```

Effekt: pri pereklyuchenii na nashego bota liniya Bitriks deaktiviruetsya, CRM otklyuchaetsya.

### 2. `scripts/webhook_enable_bitrix.py` (vozvrat na Bitriks)

Dobavlen `import subprocess`.

Analogichnaya zamena:

```python
r3 = subprocess.run([
    "curl", "-s", "-X", "POST",
    f"{BITRIX_REST}/imopenlines.config.update",
    "-H", "Content-Type: application/json",
    "-d", '{"CONFIG_ID": 12, "PARAMS": {"ACTIVE": "Y", "CRM": "Y"}}'
], capture_output=True, text=True)
print(f"Bitrix line ON + CRM on: {r3.stdout}")
```

Effekt: pri vozvrate na Bitriks liniya aktiviruetsya, CRM vklyuchaetsya.

## Chto NE menyalos

- Avito webhook subscribe/unsubscribe (httpx) — rabotayut normalno, ne tronuto
- Konstanty `BITRIX_REST`, `BITRIX_LINE_ID` — bez izmenenij
- Logika polucheniya tokena i engine.dispose — bez izmenenij

## Deploy

```bash
cd /opt/openai/crm-worker
git pull origin main
```

Skripty zapuskayutsya vruchnuyu, restart servisa ne nuzhen.

## Proverka

### Pereklyuchenie na nashego bota:
```bash
cd /opt/openai/crm-worker
python webhook_enable_ours.py
```

Ozhidaemyj vyvod:
```
Unsubscribe Bitrix: 200 ...
Subscribe ours: 200 ...
Bitrix line OFF + CRM off: {"result":true}

Gotovo: Bitriks otklyuchyon, nash vklyuchyon, liniya deaktivirovana, CRM vyklyuchena.
```

### Vozvrat na Bitriks:
```bash
cd /opt/openai/crm-worker
python webhook_enable_bitrix.py
```

Ozhidaemyj vyvod:
```
Unsubscribe ours: 200 ...
Subscribe Bitrix: 200 ...
Bitrix line ON + CRM on: {"result":true}

Gotovo: nash otklyuchyon, Bitriks vklyuchyon, liniya aktivna, CRM vklyuchena.
```

### Proverka statusa linii v Bitriks:
```bash
curl -s "https://b24.kadry-24.ru/rest/8539/3j61afi1yqk890du/imopenlines.config.get?CONFIG_ID=12" | python3 -m json.tool | grep -E '"ACTIVE"|"CRM"'
```
