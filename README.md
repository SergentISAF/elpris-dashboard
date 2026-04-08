# Elpris Dashboard

Et personligt dansk elpris-dashboard der sammenligner elselskaber, viser realtids-spotpriser og hjælper dig med at finde den billigste strøm.

![Dashboard](https://img.shields.io/badge/Made_in-Denmark-red)
![License](https://img.shields.io/badge/license-MIT-blue)

## Features

- **Realtids-spotpriser** fra Energi Data Service (gratis, ingen API-nøgle)
- **Fuldpriser inkl. tariffer** for din præcise adresse via [Min Strøm API](https://docs.minstroem.app) (valgfrit)
- **Udbyder-sammenligning** — sammenlign 10+ danske elselskaber baseret på dit forbrug
- **Forbrugssimulator** — juster dit årsforbrug og se hvem der er billigst
- **Prishistorik** — daglige snapshots bygger en trend-graf over tid
- **Docker-klar** — deploy nemt på NAS, server eller lokalt

## Hurtig start

### 1. Klon repo

```bash
git clone https://github.com/SergentISAF/elpris-dashboard.git
cd elpris-dashboard
```

### 2. Opret `.env`

```bash
cp .env.example .env
```

Rediger `.env` med dine oplysninger:

```env
YEARLY_KWH=4000
ADDRESS_LABEL=Din adresse, postnr by
PRICE_AREA=DK1
```

### 3. Kør med Docker

```bash
docker compose up -d --build
```

Dashboardet kører nu på **http://localhost:8585**

### Alternativt: Kør uden Docker

```bash
pip install -r requirements.txt
uvicorn app:app --host 0.0.0.0 --port 8585
```

## Valgfri integrationer

### Min Strøm API (anbefalet)

Giver fuldpriser inkl. transport og tariffer for din præcise adresse.

1. Anmod om API-adgang: send mail til `engberg@minstroem.app`
2. Tilføj til `.env`:

```env
MINSTROEM_API_KEY=din-api-key
MINSTROEM_API_SECRET=din-api-secret
MINSTROEM_ADDRESS_ID=dit-adresse-id
```

Find dit adresse-ID via API'en: `GET /prices/addresses/suggestions/{din adresse}`

### Eloverblik (forbrugsdata)

Viser dit faktiske elforbrug fra din elmåler.

1. Log ind på [eloverblik.dk](https://eloverblik.dk) med MitID
2. Gå til "Datadeling" og opret en token
3. Tilføj til `.env`:

```env
ELOVERBLIK_TOKEN=din-token
METERING_POINTS=571313113160184360,571313113162353856
```

## Tilpas udbydere

Rediger `PROVIDERS`-listen i `app.py` med dine egne elselskaber:

```python
PROVIDERS = [
    {"name": "Dit Selskab", "tillaeg": 2.0, "abo": 29, "binding": "Ingen", "url": "https://...", "current": True},
    # ...
]
```

- `tillaeg`: Spottillæg i øre/kWh (`None` for fastpris)
- `abo`: Månedligt abonnement i kr
- `current`: Sæt `True` på dit nuværende selskab
- `fast_pris_oere`: Kun for fastpris-produkter

## API endpoints

| Endpoint | Beskrivelse |
|----------|-------------|
| `GET /` | Dashboard (HTML) |
| `GET /api/data` | Alle data som JSON |
| `GET /api/refresh` | Tvungen opdatering |
| `GET /api/simulate/{kwh}` | Simuler med X kWh/år |

## Datakilder

- [Energi Data Service](https://www.energidataservice.dk/) — spotpriser (gratis)
- [Min Strøm API](https://docs.minstroem.app/) — fuldpriser inkl. tariffer (kræver adgang)
- [Eloverblik](https://eloverblik.dk/) — forbrugsdata (kræver MitID)

## Licens

MIT — brug det frit, del det videre.
