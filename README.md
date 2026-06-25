# OptimusAutomate_API-Data-Dashboard-Weather-Dashboard
"""
TASK 4: API Data Dashboard — Weather Dashboard
Optimus Automate — Python Programming Internship

Uses Open-Meteo API (free, no key required).
Optional: matplotlib chart if installed.
"""

import json
import time
import os
from datetime import datetime, timedelta
from urllib.request import urlopen
from urllib.parse import urlencode
from urllib.error import URLError

# ─── CONFIG ────────────────────────────────────────────────────────────────────

REFRESH_INTERVAL = 300  # seconds between auto-refresh (5 min)
CACHE_FILE       = ".weather_cache.json"

# Pre-defined city coordinates (no geocoding API needed)
CITIES = {
    "1": ("Rawalpindi, PK",  33.6007, 73.0679),
    "2": ("Karachi, PK",     24.8607, 67.0011),
    "3": ("Lahore, PK",      31.5497, 74.3436),
    "4": ("Islamabad, PK",   33.7294, 73.0931),
    "5": ("London, UK",      51.5074, -0.1278),
    "6": ("New York, US",    40.7128, -74.0060),
    "7": ("Dubai, UAE",      25.2048, 55.2708),
    "8": ("Tokyo, JP",       35.6762, 139.6503),
}

WMO_CODES = {
    0: "Clear sky ☀️",
    1: "Mainly clear 🌤️", 2: "Partly cloudy ⛅", 3: "Overcast ☁️",
    45: "Fog 🌫️", 48: "Depositing fog 🌫️",
    51: "Light drizzle 🌦️", 53: "Drizzle 🌦️", 55: "Dense drizzle 🌧️",
    61: "Slight rain 🌧️", 63: "Moderate rain 🌧️", 65: "Heavy rain 🌧️",
    71: "Slight snow 🌨️", 73: "Moderate snow 🌨️", 75: "Heavy snow ❄️",
    80: "Rain showers 🌦️", 81: "Moderate showers 🌧️", 82: "Violent showers ⛈️",
    95: "Thunderstorm ⛈️", 96: "Thunderstorm + hail ⛈️", 99: "Heavy thunderstorm ⛈️",
}

# ─── API FETCH ──────────────────────────────────────────────────────────────────

def fetch_weather(lat: float, lon: float) -> dict:
    params = {
        "latitude":  lat,
        "longitude": lon,
        "current":   "temperature_2m,relative_humidity_2m,wind_speed_10m,weathercode,apparent_temperature",
        "hourly":    "temperature_2m,precipitation_probability",
        "daily":     "temperature_2m_max,temperature_2m_min,precipitation_sum,weathercode",
        "timezone":  "auto",
        "forecast_days": 7,
    }
    url = "https://api.open-meteo.com/v1/forecast?" + urlencode(params)
    try:
        with urlopen(url, timeout=10) as resp:
            return json.loads(resp.read().decode())
    except URLError as e:
        return {"error": str(e)}


def save_cache(data: dict, city_name: str):
    cache = {"city": city_name, "timestamp": datetime.now().isoformat(), "data": data}
    with open(CACHE_FILE, "w") as f:
        json.dump(cache, f)


def load_cache() -> dict | None:
    if not os.path.exists(CACHE_FILE):
        return None
    with open(CACHE_FILE) as f:
        return json.load(f)


# ─── DISPLAY ───────────────────────────────────────────────────────────────────

def divider(char="─", width=52):
    print(char * width)

def header(text):
    divider("═")
    print(f"  {text}")
    divider("═")


def display_current(data: dict, city: str):
    c = data["current"]
    code = c.get("weathercode", 0)
    cond = WMO_CODES.get(code, "Unknown")

    header(f"🌍  Weather Dashboard — {city}")
    print(f"  Updated  : {datetime.now().strftime('%d %b %Y, %H:%M')}")
    divider()
    print(f"  Condition         : {cond}")
    print(f"  Temperature       : {c['temperature_2m']}°C  (Feels like {c['apparent_temperature']}°C)")
    print(f"  Humidity          : {c['relative_humidity_2m']}%")
    print(f"  Wind Speed        : {c['wind_speed_10m']} km/h")
    divider()


def display_7day_forecast(data: dict):
    d = data["daily"]
    print("\n  📅  7-Day Forecast")
    divider()
    print(f"  {'Date':<12} {'High':>6} {'Low':>6} {'Rain':>7}  Condition")
    divider()
    for i in range(len(d["time"])):
        date_str = d["time"][i]
        date_obj = datetime.strptime(date_str, "%Y-%m-%d")
        label = date_obj.strftime("%a %d %b")
        high  = d["temperature_2m_max"][i]
        low   = d["temperature_2m_min"][i]
        rain  = d["precipitation_sum"][i]
        code  = d["weathercode"][i]
        icon  = WMO_CODES.get(code, "—").split(" ")[-1]
        print(f"  {label:<12} {high:>5}° {low:>5}°  {rain:>5}mm  {icon}")
    divider()


def display_hourly_bar_chart(data: dict):
    """Simple terminal bar chart for next 12 hours temperature."""
    hours = data["hourly"]["time"][:12]
    temps = data["hourly"]["temperature_2m"][:12]
    prec  = data["hourly"]["precipitation_probability"][:12]

    print("\n  🌡️  Next 12 Hours (Temperature & Rain %)")
    divider()
    min_t = min(temps)
    max_t = max(temps)
    span  = max_t - min_t if max_t != min_t else 1

    for i, (h, t, p) in enumerate(zip(hours, temps, prec)):
        hour_label = h[11:16]
        bar_len    = int(((t - min_t) / span) * 20)
        bar        = "█" * bar_len + "░" * (20 - bar_len)
        rain_icon  = "🌧" if p >= 60 else ("🌦" if p >= 30 else "  ")
        print(f"  {hour_label}  {bar}  {t:>5.1f}°C  {rain_icon} {p:>3}%")
    divider()


# ─── OPTIONAL MATPLOTLIB ────────────────────────────────────────────────────────

def try_matplotlib_chart(data: dict, city: str):
    try:
        import matplotlib.pyplot as plt
        import matplotlib.dates as mdates

        dates = [datetime.strptime(d, "%Y-%m-%d") for d in data["daily"]["time"]]
        highs = data["daily"]["temperature_2m_max"]
        lows  = data["daily"]["temperature_2m_min"]
        rain  = data["daily"]["precipitation_sum"]

        fig, (ax1, ax2) = plt.subplots(2, 1, figsize=(10, 6), sharex=True)
        fig.suptitle(f"7-Day Forecast — {city}", fontsize=14, fontweight="bold")

        ax1.plot(dates, highs, "o-", color="#e74c3c", label="High °C")
        ax1.plot(dates, lows,  "o-", color="#3498db", label="Low °C")
        ax1.fill_between(dates, lows, highs, alpha=0.1, color="#9b59b6")
        ax1.set_ylabel("Temperature (°C)")
        ax1.legend()
        ax1.grid(alpha=0.3)

        ax2.bar(dates, rain, color="#3498db", alpha=0.7, label="Rain (mm)")
        ax2.set_ylabel("Precipitation (mm)")
        ax2.xaxis.set_major_formatter(mdates.DateFormatter("%a %d"))
        ax2.legend()
        ax2.grid(alpha=0.3)

        plt.tight_layout()
        filename = f"weather_{city.replace(', ', '_').replace(' ', '_')}.png"
        plt.savefig(filename, dpi=100)
        print(f"\n  📊  Chart saved as '{filename}'")
        plt.show()

    except ImportError:
        print("\n  (Install matplotlib with: pip install matplotlib for charts)")


# ─── CITY SELECTOR ─────────────────────────────────────────────────────────────

def choose_city():
    print("\n  Select a city:")
    for key, (name, _, _) in CITIES.items():
        print(f"    {key}. {name}")
    print("    0. Custom coordinates")
    choice = input("\n  Choice: ").strip()

    if choice in CITIES:
        return CITIES[choice]
    elif choice == "0":
        try:
            name = input("  City name: ").strip() or "Custom"
            lat  = float(input("  Latitude : "))
            lon  = float(input("  Longitude: "))
            return name, lat, lon
        except ValueError:
            print("  ❌  Invalid coordinates.")
            return None
    else:
        print("  ❌  Invalid choice.")
        return None


# ─── MAIN ──────────────────────────────────────────────────────────────────────

def run_dashboard():
    print("\n╔══════════════════════════════════════╗")
    print("║   🌤️  Weather API Dashboard v1.0    ║")
    print("║   Powered by Open-Meteo (free API)  ║")
    print("╚══════════════════════════════════════╝")

    city_info = choose_city()
    if not city_info:
        return

    city_name, lat, lon = city_info
    auto_refresh = input("\n  Enable auto-refresh every 5 min? (y/n): ").strip().lower() == "y"
    show_chart   = input("  Show matplotlib chart? (y/n): ").strip().lower() == "y"

    while True:
        print(f"\n  Fetching weather data for {city_name}...")
        data = fetch_weather(lat, lon)

        if "error" in data:
            # Try cache
            cache = load_cache()
            if cache:
                print(f"  ⚠️  Network error. Showing cached data from {cache['timestamp'][:16]}")
                data = cache["data"]
                city_name = cache["city"]
            else:
                print(f"  ❌  Network error: {data['error']}")
                return
        else:
            save_cache(data, city_name)

        display_current(data, city_name)
        display_7day_forecast(data)
        display_hourly_bar_chart(data)

        if show_chart:
            try_matplotlib_chart(data, city_name)

        if not auto_refresh:
            print("\n  Press Enter to refresh or type 'q' to quit: ", end="")
            if input().strip().lower() == "q":
                break
        else:
            print(f"\n  ⏰  Auto-refresh in {REFRESH_INTERVAL // 60} min. Press Ctrl+C to stop.")
            try:
                time.sleep(REFRESH_INTERVAL)
            except KeyboardInterrupt:
                print("\n  Stopped. Goodbye! 👋")
                break


if __name__ == "__main__":
    run_dashboard()
