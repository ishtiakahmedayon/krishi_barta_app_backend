# Bengali Geographic Search & Open-Meteo Weather Resolution Plan

This plan details the implementation of a comprehensive Bengali location resolver and Open-Meteo weather proxy in the Flask + PostgreSQL backend. It solves the issue where searching in English (`Tangail`) succeeds while searching in Bengali (`টাঙ্গাইল`) fails, enabling search across all 64 districts and 495 upazilas in Bangladesh.

### User Review & Critical Decisions

> [!IMPORTANT]
> **Why `Tangail` works but `টাঙ্গাইল` fails**:
> Global weather services (like OpenWeatherMap and Open-Meteo) index locations primarily using English names or numerical latitudes/longitudes. Sending raw Bengali characters (`টাঙ্গাইল`) causes external APIs to return a 404 or empty location result.
>
> **The Solution**:
> 1. The backend will store an indexed geographic table (`geo_locations`) mapping all 64 districts and 495 upazilas with their Bengali names, English transliterations, and exact GPS coordinates (`latitude`, `longitude`).
> 2. The backend will provide two unified endpoints:
>    - `GET /api/geo/search?q=টাঙ্গাইল` (autocomplete / area resolver)
>    - `GET /api/weather?q=টাঙ্গাইল` (coordinates resolution + live Open-Meteo forecast with localized Bengali weather status).

- **Confirmed Decision 1: Weather Provider**: Open-Meteo API (free, reliable, no API key required, zero billing friction, accurate hourly and daily forecasts for Bangladesh).
- **Confirmed Decision 2: Geographic Scope**: Complete coverage of all 64 Districts and all 495 Upazilas across Bangladesh's 8 divisions (Dhaka, Chittagong, Rajshahi, Khulna, Barisal, Sylhet, Rangpur, Mymensingh).
- **Confirmed Decision 3: Storage**: Dedicated PostgreSQL table `geo_locations` indexed on both `name_bn` and `name_en` for sub-millisecond lookup times.

---

### 1. Overview & Core Concept

- **What It Does**: Enables the KrishiApp mobile weather screen to understand Bengali place names (e.g., `টাঙ্গাইল`, `ঘাটাইল`, `মধুপুর`, `সখীপুর`, `মিরপুর`, `পটিয়া`) as well as English names, resolving them to precise coordinates and returning current weather conditions, humidity, wind, rainfall, and multi-day forecasts in Bengali.
- **Target Persona**: Bangladeshi farmers, extension workers, and rural users who use Bengali script to search for weather at the district or upazila/village level.
- **Key Value**: Eliminates language barriers in weather forecasting while providing localized agricultural climate insights.

---

### 2. User Experience & API Flow

1. **User Types in Flutter**: The user types `টা` or `টাঙ্গাইল` in the weather search bar.
2. **Lookup Resolution**: Flutter calls `GET /api/geo/search?q=টাঙ্গাইল` or `GET /api/weather?q=টাঙ্গাইল`.
3. **Database Match**: PostgreSQL matches `name_bn = 'টাঙ্গাইল'` or `name_bn ILIKE '%টাঙ্গাইল%'` and retrieves `lat: 24.2513, lon: 89.9167`.
4. **Open-Meteo Fetch**: Flask fetches the latest meteorological data from Open-Meteo using the resolved GPS coordinates.
5. **Bengali Weather Code Translation**: WMO weather codes are translated into natural Bengali terms:
   - `0` ➔ `পরিষ্কার আকাশ (Clear sky)`
   - `1, 2, 3` ➔ `আংশিক মেঘলা (Partly cloudy)`
   - `61, 63, 65` ➔ `বৃষ্টিপাত (Rainfall)`
   - `95, 96, 99` ➔ `বজ্রপাতসহ ঝড়বৃষ্টি (Thunderstorm)`
6. **Flutter Render**: The mobile UI renders temperature, weather icon, Bengali status, humidity, and forecast seamlessly.

---

### 3. Key Technical Decisions & Trade-Offs

- **Decision 1: Direct Backend Weather Proxy vs. Coordinates-Only Lookup**
  - *Chosen Approach*: Provide both `GET /api/geo/search` (coordinates only) and `GET /api/weather` (full weather).
  - *Why*: Allows Flutter developers the flexibility to either fetch weather directly through the backend in one call with Bengali descriptions pre-formatted, or fetch coordinates and query Open-Meteo client-side.
- **Decision 2: WMO Weather Interpretation Engine**
  - *Chosen Approach*: Built-in translation dictionary in `api/weather.py` mapping standard WMO codes to Bengali agricultural weather indicators and weather icons.
  - *Why*: Open-Meteo provides raw WMO code numbers (e.g. `61`); converting them on the backend ensures consistent Bengali labeling across mobile and web platforms.
- **Decision 3: Fuzzy and Prefix Bengali Substring Matching**
  - *Chosen Approach*: `ILIKE %query%` combined with exact match priority sorting.
  - *Why*: If a user searches `মধুপুর`, it matches `মধুপুর, টাঙ্গাইল` without requiring the user to type the full district specification.

---

### 4. Technical Architecture & Data Strategy

#### Architecture Diagram

```
┌────────────────────────────────────────────────────────┐
│              Flutter Mobile App (Weather View)         │
│         User searches: "টাঙ্গাইল" or "Tangail"         │
└───────────────────────────┬────────────────────────────┘
                            │ GET /api/weather?q=টাঙ্গাইল
                            ▼
┌────────────────────────────────────────────────────────┐
│             Flask Backend on Render                    │
│                                                        │
│   ┌────────────────────────────────────────────────┐   │
│   │ 1. api/weather.py                              │   │
│   │    • Normalize query: "টাঙ্গাইল"               │   │
│   │    • Query PostgreSQL geo_locations            │   │
│   └───────────────────────┬────────────────────────┘   │
│                           │
│   ┌───────────────────────▼────────────────────────┐   │
│   │ 2. PostgreSQL: geo_locations Table             │   │
│   │    Matched: Tangail (Lat: 24.2513, Lon: 89.9167)│   │
│   └───────────────────────┬────────────────────────┘   │
│                           │
│   ┌───────────────────────▼────────────────────────┐   │
│   │ 3. Fetch Open-Meteo API                        │   │
│   │    https://api.open-meteo.com/v1/forecast      │   │
│   │    ?latitude=24.2513&longitude=89.9167         │   │
│   └───────────────────────┬────────────────────────┘   │
│                           │
│   ┌───────────────────────▼────────────────────────┐   │
│   │ 4. WMO Code & Bengali Weather Formatter        │   │
│   │    WMO 61 ➔ "হালকা বৃষ্টিপাত", Icon: "rain"   │   │
│   └────────────────────────────────────────────────┘   │
└───────────────────────────┬────────────────────────────┘
                            │ HTTP 200 JSON Response
                            ▼
┌────────────────────────────────────────────────────────┐
│  {                                                     │
│    "location": { "name_bn": "টাঙ্গাইল", ... },        │
│    "current": { "temperature": 28.5, "condition": ... }│
│  }                                                     │
└────────────────────────────────────────────────────────┘
```

#### Proposed Database Schema (`geo_locations`)

```sql
CREATE TABLE IF NOT EXISTS geo_locations (
    id SERIAL PRIMARY KEY,
    name_bn TEXT NOT NULL,
    name_en TEXT NOT NULL,
    type TEXT NOT NULL DEFAULT 'upazila', -- 'district' or 'upazila'
    district_bn TEXT NOT NULL,
    district_en TEXT NOT NULL,
    division_bn TEXT NOT NULL,
    division_en TEXT NOT NULL,
    latitude DOUBLE PRECISION NOT NULL,
    longitude DOUBLE PRECISION NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_geo_locations_bn ON geo_locations(name_bn);
CREATE INDEX IF NOT EXISTS idx_geo_locations_en ON geo_locations(LOWER(name_en));
```

#### Endpoints Specification

1. **`GET /api/geo/search?q=<query>&limit=10`**:
   - Matches against `name_bn`, `name_en`, `district_bn`, `district_en`.
   - Returns a list of matching districts/upazilas with coordinates.
2. **`GET /api/weather?q=<name_or_query>`**:
   - Resolves location, queries Open-Meteo, returns current weather + 7-day forecast.
3. **`GET /api/weather?lat=<lat>&lon=<lon>`**:
   - Reverse-queries coordinates directly for GPS-based weather fetching.

---

### 5. Implementation Stages (Upon Approval)

1. **Step 1: Geographic Dataset & Migration (`artifacts/api-server/init_database.py`)**:
   Create `geo_locations` table and seed complete dataset of all 64 districts and all 495 upazilas with exact GPS coordinates.
2. **Step 2: Weather & Geo Blueprint (`artifacts/api-server/api/weather.py`)**:
   Build the search handler, Open-Meteo HTTP fetcher, and WMO code translator to Bengali.
3. **Step 3: App Factory Registration (`artifacts/api-server/api/__init__.py`)**:
   Register `weather_bp` in `create_app()`.
4. **Step 4: Frontend Operations & Flutter Testing Hub (`src/App.tsx`)**:
   Add an interactive Bengali Weather & Geo Search Simulator tab where you can type `টাঙ্গাইল`, `ঘাটাইল`, or any district/upazila and see the resolved coordinates, live Open-Meteo forecast, and copyable Flutter integration code.
