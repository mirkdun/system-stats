---
name: open-meteo-weather
description: Check current weather and forecasts through Open-Meteo — resolve location coordinates, get current conditions, and retrieve hourly or daily forecasts without an API key.
---

# Weather through Open-Meteo

Use this skill when the user asks about weather, temperature, precipitation, wind, or a forecast. Open-Meteo does not require an API key; shell/Python requests are enough, and a browser is unnecessary.

## 1. Resolve location coordinates

### If the user named a place
Use the Open-Meteo geocoding API:

```bash
python3 - <<'PY'
import json, urllib.parse, urllib.request
place = 'Moscow'  # replace with the user's place
url = 'https://geocoding-api.open-meteo.com/v1/search?' + urllib.parse.urlencode({
    'name': place,
    'count': 5,
    'language': 'en',
    'format': 'json',
})
data = json.load(urllib.request.urlopen(url, timeout=15))
for r in data.get('results', []):
    print(r.get('name'), r.get('admin1'), r.get('country'), r['latitude'], r['longitude'], r.get('timezone'))
PY
```

- Usually select the first result, but verify `admin1` and `country`.
- If several places share the same name and the right one is unclear, ask the user.
- If the user already provided coordinates, skip geocoding.

### If the user asks for weather “in your location”
This means the VM/Devin location, not the user's location. The VM has no exact street address; use approximate IP geolocation:

```bash
python3 - <<'PY'
import json, urllib.request
data = json.load(urllib.request.urlopen('https://ipapi.co/json/', timeout=15))
print(data.get('city'), data.get('region'), data.get('country_name'), data.get('latitude'), data.get('longitude'), data.get('timezone'))
PY
```

In the response, always say this is the VM's approximate IP-based location. Do not present it as an exact address or confuse it with the user's location. If IP geolocation is unavailable, ask for a city; the session timezone may be used only as a rough hint and must be explicitly labeled as an assumption.

## 2. Current weather

```bash
python3 - <<'PY'
import json, urllib.parse, urllib.request
lat, lon = 55.75204, 37.61781  # replace with the location coordinates
params = {
    'latitude': lat,
    'longitude': lon,
    'current': ','.join([
        'temperature_2m', 'relative_humidity_2m', 'apparent_temperature',
        'precipitation', 'weather_code', 'wind_speed_10m', 'wind_direction_10m'
    ]),
    'wind_speed_unit': 'ms',
    'timezone': 'auto',
}
url = 'https://api.open-meteo.com/v1/forecast?' + urllib.parse.urlencode(params)
data = json.load(urllib.request.urlopen(url, timeout=15))
print(json.dumps(data['current'], ensure_ascii=False, indent=2))
print(json.dumps(data['current_units'], ensure_ascii=False, indent=2))
PY
```

Use `timezone=auto` so timestamps are local to the coordinates. The coordinates returned by the API may differ slightly from the requested coordinates because the model snaps them to its grid; this is normal.

## 3. Forecast

### Hourly forecast
Parameters:

```text
hourly=temperature_2m,precipitation_probability,precipitation,weather_code,wind_speed_10m
wind_speed_unit=ms
timezone=auto
forecast_days=2
```

For “today”, “tomorrow”, “this evening”, and similar requests, filter hours by the local date returned by the API rather than by UTC.

### Daily forecast
Parameters:

```text
daily=weather_code,temperature_2m_max,temperature_2m_min,precipitation_sum,precipitation_probability_max,wind_speed_10m_max
wind_speed_unit=ms
timezone=auto
forecast_days=7
```

## 4. `weather_code` meanings (WMO)

| Code | Meaning |
|---:|---|
| 0 | clear sky |
| 1 | mainly clear |
| 2 | partly cloudy |
| 3 | overcast |
| 45, 48 | fog / depositing rime fog |
| 51, 53, 55 | drizzle: light / moderate / dense |
| 56, 57 | freezing drizzle |
| 61, 63, 65 | rain: slight / moderate / heavy |
| 66, 67 | freezing rain |
| 71, 73, 75 | snowfall: slight / moderate / heavy |
| 77 | snow grains |
| 80, 81, 82 | rain showers: slight / moderate / violent |
| 85, 86 | snow showers |
| 95 | thunderstorm |
| 96, 99 | thunderstorm with hail |

## 5. How to answer the user

- State the location and the local time of the observation or forecast first.
- For current weather, usually include: temperature, feels-like temperature, precipitation/sky condition, humidity, and wind.
- Request wind speed in m/s (`wind_speed_unit=ms`) unless the user asks for another unit.
- Do not paste raw JSON into the answer; use it only for parsing.
- Mention that the data is from Open-Meteo.
- Do not imply that IP geolocation is precise; when precision matters, ask the user for a city.
