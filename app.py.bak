import os
import json
import hmac
import hashlib
import base64
import httpx
from datetime import datetime, timedelta
from contextlib import asynccontextmanager
from pathlib import Path
from dotenv import load_dotenv
from fastapi import FastAPI
from fastapi.responses import HTMLResponse
from apscheduler.schedulers.asyncio import AsyncIOScheduler

load_dotenv()

# --- Configuration via .env ---
YEARLY_KWH = int(os.getenv("YEARLY_KWH", "4000"))
ADDRESS_LABEL = os.getenv("ADDRESS_LABEL", "Din adresse")
PRICE_AREA = os.getenv("PRICE_AREA", "DK1")  # DK1 (Vestdanmark) or DK2 (Østdanmark)

# Eloverblik (optional)
ELOVERBLIK_TOKEN = os.getenv("ELOVERBLIK_TOKEN", "")
METERING_POINTS = [
    mp.strip() for mp in os.getenv("METERING_POINTS", "").split(",") if mp.strip()
]
ELOVERBLIK_BASE = "https://api.eloverblik.dk/customerapi/api"

# Min Strøm API (optional — get access at https://docs.minstroem.app)
MINSTROEM_KEY = os.getenv("MINSTROEM_API_KEY", "")
MINSTROEM_SECRET = os.getenv("MINSTROEM_API_SECRET", "")
MINSTROEM_ADDRESS = os.getenv("MINSTROEM_ADDRESS_ID", "")
MINSTROEM_BASE = "https://api.minstroem.app/thirdParty"

# Energi Data Service (free, no auth needed)
ENERGI_DATA_URL = "https://api.energidataservice.dk/dataset/Elspotprices"

DATA_DIR = Path("data")
DATA_DIR.mkdir(exist_ok=True)


def minstroem_token() -> str:
    hashed = hmac.new(MINSTROEM_SECRET.encode(), MINSTROEM_KEY.encode(), hashlib.sha256).hexdigest()
    return base64.b64encode(f"{MINSTROEM_KEY}:{hashed}".encode()).decode()


# --- Electricity providers ---
# Customize this list with your own providers. Set "current": True on yours.
# tillaeg = spot markup in øre/kWh, abo = monthly subscription in kr
PROVIDERS = [
    {"name": "Enkel Energi",  "tillaeg": 0.0, "abo": 18,  "binding": "Ingen", "url": "https://enkelenergi.dk"},
    {"name": "Altid Energi",  "tillaeg": 0.0, "abo": 98,  "binding": "Ingen", "url": "https://altidenergi.dk", "note": "98 kr/md ved 12-20k kWh/år"},
    {"name": "Modstrøm",      "tillaeg": 2.0, "abo": 0,   "binding": "Ingen", "url": "https://modstrom.dk"},
    {"name": "Norlys",        "tillaeg": 4.0, "abo": 35,  "binding": "Ingen", "url": "https://norlys.dk"},
    {"name": "EWII Basis",    "tillaeg": 3.75,"abo": 29,  "binding": "Ingen", "url": "https://ewii.dk"},
    {"name": "EWII Plus",     "tillaeg": 5.5, "abo": 29,  "binding": "Ingen", "url": "https://ewii.dk", "note": "For forbrug >3700 kWh/år"},
    {"name": "EWII Fastpris", "tillaeg": None,"abo": 15,  "binding": "12 mdr", "url": "https://ewii.dk", "fast_pris_oere": 114.80},
    {"name": "NRGi",          "tillaeg": 9.0, "abo": 0,   "binding": "Ingen", "url": "https://nrgi.dk"},
    {"name": "OK El",         "tillaeg": 4.9, "abo": 29,  "binding": "Ingen", "url": "https://ok.dk"},
]

scheduler = AsyncIOScheduler()

# In-memory cache
cache = {
    "access_token": None,
    "spot_prices": [],
    "consumption": {},
    "providers": [],
    "last_updated": None,
    "errors": [],
    "history": [],
    "minstroem_prices": [],
}


def save_daily_snapshot():
    if not cache["spot_prices"]:
        return
    today = datetime.now().strftime("%Y-%m-%d")
    prices = cache["spot_prices"]
    avg = sum(p["price_oere"] for p in prices) / len(prices)
    mn = min(p["price_oere"] for p in prices)
    mx = max(p["price_oere"] for p in prices)
    snapshot = {"date": today, "avg_oere": round(avg, 2), "min_oere": round(mn, 2), "max_oere": round(mx, 2), "prices": prices}
    (DATA_DIR / f"spot_{today}.json").write_text(json.dumps(snapshot, ensure_ascii=False), encoding="utf-8")


def load_history() -> list[dict]:
    history = []
    for f in sorted(DATA_DIR.glob("spot_*.json")):
        try:
            data = json.loads(f.read_text(encoding="utf-8"))
            history.append({"date": data["date"], "avg": data["avg_oere"], "min": data["min_oere"], "max": data["max_oere"]})
        except Exception:
            continue
    return history


async def get_access_token() -> str | None:
    if not ELOVERBLIK_TOKEN:
        return None
    try:
        async with httpx.AsyncClient(timeout=30) as client:
            resp = await client.get(f"{ELOVERBLIK_BASE}/token", headers={"Authorization": f"Bearer {ELOVERBLIK_TOKEN}"})
            if resp.status_code == 200:
                return resp.json().get("result")
    except Exception:
        pass
    return None


async def fetch_minstroem_prices():
    if not MINSTROEM_KEY or not MINSTROEM_ADDRESS:
        return
    headers = {"Authorization": f"Bearer {minstroem_token()}"}
    try:
        async with httpx.AsyncClient(timeout=15) as client:
            resp = await client.get(f"{MINSTROEM_BASE}/prices/addresses/{MINSTROEM_ADDRESS}", headers=headers)
            if resp.status_code == 200:
                cache["minstroem_prices"] = [
                    {
                        "hour": p["date"],
                        "spot_kr": round(p["price"] - p["charges"], 4),
                        "charges_kr": round(p["charges"], 4),
                        "total_kr": round(p["price"], 4),
                        "total_oere": round(p["price"] * 100, 2),
                        "color": p.get("color", ""),
                    }
                    for p in resp.json()
                ]
    except Exception as e:
        cache["errors"].append(f"Min Strøm fejl: {e}")


async def fetch_spot_prices():
    params = {"filter": f'{{"PriceArea":"{PRICE_AREA}"}}', "sort": "HourUTC desc", "limit": 48}
    try:
        async with httpx.AsyncClient(timeout=30) as client:
            resp = await client.get(ENERGI_DATA_URL, params=params)
            if resp.status_code == 200:
                prices = []
                for r in resp.json().get("records", []):
                    hour = r.get("HourDK", r.get("HourUTC", ""))
                    price_dkk = r.get("SpotPriceDKK")
                    if price_dkk is not None:
                        prices.append({"hour": hour, "price_oere": round(price_dkk / 10, 2)})
                prices.reverse()
                cache["spot_prices"] = prices
    except Exception as e:
        cache["errors"].append(f"Spotpris fejl: {e}")


async def fetch_consumption():
    token = await get_access_token()
    if not token or not METERING_POINTS:
        return
    cache["access_token"] = token
    now = datetime.now()
    headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
    payload = {"meteringPoints": {"meteringPoint": METERING_POINTS}}
    try:
        async with httpx.AsyncClient(timeout=60) as client:
            date_from = (now - timedelta(days=365)).strftime("%Y-%m-%d")
            date_to = now.strftime("%Y-%m-%d")
            resp = await client.post(f"{ELOVERBLIK_BASE}/meterdata/gettimeseries/{date_from}/{date_to}/Month", headers=headers, json=payload)
            if resp.status_code == 200:
                cache["consumption"] = resp.json()
    except Exception:
        pass


def calculate_provider_costs(yearly_kwh: float) -> list[dict]:
    results = []
    avg_spot = 0.0
    if cache["spot_prices"]:
        avg_spot = sum(p["price_oere"] for p in cache["spot_prices"]) / len(cache["spot_prices"])

    for p in PROVIDERS:
        if p.get("tillaeg") is not None:
            elpris_oere = avg_spot + p["tillaeg"]
            yearly_cost = (elpris_oere / 100) * yearly_kwh + p["abo"] * 12
        else:
            elpris_oere = p.get("fast_pris_oere", 0)
            yearly_cost = (elpris_oere / 100) * yearly_kwh + p["abo"] * 12

        results.append({
            "name": p["name"], "tillaeg": p.get("tillaeg"), "abo": p["abo"],
            "binding": p["binding"], "elpris_oere": round(elpris_oere, 2),
            "yearly_cost": round(yearly_cost), "monthly_cost": round(yearly_cost / 12),
            "url": p["url"], "note": p.get("note", ""), "current": p.get("current", False),
        })
    results.sort(key=lambda x: x["yearly_cost"])
    return results


async def refresh_data():
    cache["errors"] = []
    await fetch_spot_prices()
    await fetch_minstroem_prices()
    cache["providers"] = calculate_provider_costs(YEARLY_KWH)
    cache["last_updated"] = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    cache["history"] = load_history()


async def daily_save():
    await fetch_spot_prices()
    save_daily_snapshot()
    cache["history"] = load_history()


@asynccontextmanager
async def lifespan(app: FastAPI):
    await refresh_data()
    save_daily_snapshot()
    cache["history"] = load_history()
    scheduler.add_job(refresh_data, "interval", hours=1)
    scheduler.add_job(daily_save, "cron", hour=23, minute=55)
    scheduler.start()
    yield
    scheduler.shutdown()


app = FastAPI(title="Elpris Dashboard", lifespan=lifespan)


@app.get("/api/data")
async def get_data():
    spot = cache["spot_prices"]
    avg_spot = round(sum(p["price_oere"] for p in spot) / len(spot), 2) if spot else 0
    current_spot = spot[-1]["price_oere"] if spot else 0
    min_spot = min((p["price_oere"] for p in spot), default=0)
    max_spot = max((p["price_oere"] for p in spot), default=0)

    return {
        "spot": {"current": current_spot, "average": avg_spot, "min": min_spot, "max": max_spot, "prices": spot[-48:]},
        "providers": cache["providers"],
        "metering_points": METERING_POINTS,
        "yearly_kwh": YEARLY_KWH,
        "address": ADDRESS_LABEL,
        "price_area": PRICE_AREA,
        "last_updated": cache["last_updated"],
        "token_ok": cache["access_token"] is not None,
        "errors": cache["errors"],
        "history": cache["history"],
        "minstroem": cache["minstroem_prices"],
    }


@app.get("/api/refresh")
async def manual_refresh():
    await refresh_data()
    save_daily_snapshot()
    return {"status": "ok", "last_updated": cache["last_updated"]}


@app.get("/api/simulate/{kwh}")
async def simulate(kwh: int):
    return {"yearly_kwh": kwh, "providers": calculate_provider_costs(kwh)}


@app.get("/", response_class=HTMLResponse)
async def index():
    return open("static/index.html", encoding="utf-8").read()
