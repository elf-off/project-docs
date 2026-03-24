# Deploy: Pereklyuchenie CRM Bitriks v skriptakh vebhukov

## Izmenennye fajly

- `scripts/webhook_enable_ours.py`
- `scripts/webhook_enable_bitrix.py`

## Chto sdelano

V oba skripta dobavlen shag pereklyucheniya CRM v otkrytoj linii Bitriks cherez REST API (`imopenlines.config.update`).

- **webhook_enable_ours.py** — posle podpiski nashego vebhuka otklyuchaet CRM v otkrytoj linii (`CRM: N`, `CRM_CREATE: none`), chtoby Bitriks ne sozdaval lidy
- **webhook_enable_bitrix.py** — posle podpiski Bitriks-vebhuka vklyuchaet CRM obratno (`CRM: Y`)

Zapros k Bitriks idet cherez otdelnyj httpx-klient (bez Avito-tokenov). Esli zapros upal — vyvoditsya oshibka, no skript ne preryvaetsya (vebhuk uzhe pereklyuchen).

## Konstanty

```
BITRIX_REST = https://b24.kadry-24.ru/rest/8539/3j61afi1yqk890du
BITRIX_LINE_ID = 12  (otkrytaya liniya Avito dlya akkaunta Lavka, id=2)
```

## Proverka

```bash
# Pereklyuchit na bota
python3 ./scripts/webhook_enable_ours.py
# Ozhidaemyj vyvod:
# Unsubscribe Bitrix: 200 ...
# Subscribe ours: 200 ...
# Bitrix CRM off: 200 {"result":true,...}
# Gotovo: Bitriks otklyuchen, nash vklyuchen, CRM v linii vyklyuchena.

# Pereklyuchit obratno
python3 ./scripts/webhook_enable_bitrix.py
# Ozhidaemyj vyvod:
# Unsubscribe ours: 200 ...
# Subscribe Bitrix: 200 ...
# Bitrix CRM on: 200 {"result":true,...}
# Gotovo: nash otklyuchen, Bitriks vklyuchen, CRM v linii vklyuchena.
```
