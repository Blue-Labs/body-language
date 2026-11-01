# Blue Labs — Body Language (`bodylanguage.blue-labs.org`)
**Architecture, Database Schema, and Implementation Master Plan**

---

## 1. Executive Summary & Core Objectives

**Body Language** is a self-hosted, privacy-first biometric and contextual telemetry platform developed by **Blue Labs**. It aggregates real-time health, body composition, nutrition, and environmental activity streams into a local PostgreSQL + TimescaleDB + PostGIS database, serving an interactive, event-driven "Futuristic Medicine" 3D/2D web interface over FastAPI.

### Primary Hardware & Sensor Ecosystem
* **Continuous Glucose Monitor (CGM):** Dexcom G7 (5-minute interstitial glucose, trend rate, arrows, and Clarity/Share reconciliation).
* **Wearable Vitals & Activity:** Samsung Galaxy Watch 9 paired with Samsung Galaxy S26 Ultra (heart rate, $SpO_2$, step counts, sleep, skin temperature, GPS/activity state via Android Health Connect).
* **Smart Body Fat Scale:** Ablegrid `FI2016LB` ("A-Scale X" / Fitdays protocol) captured locally via Bluetooth Low Energy (BLE) with transparent Python bioelectrical impedance ($\Omega$) mathematical equations and side-by-side Fitdays calculation verification.
* **Activity & Posture Context:** Workstation X11/Wayland keyboard/mouse idle daemon + PostGIS polygon geofences (e.g., sitting at keyboard vs. mowing the lawn) to correlate sedentary vs. active states with glycemic and cardiac responses.
* **AI-Assisted Meal & Nutrition Logging:** Two-pass hybrid vision pipeline utilizing a local 24 GB NVIDIA RTX 3090 (`Ollama`) for zero-cost classification and nutrition label OCR, escalating complex plated food photos to Cloud Vision APIs, paired with real-time consumed-portion sliders.

---

## 2. Target Technology Stack

| Layer | Technology / Specification |
| :--- | :--- |
| **OS & Kernel** | Linux Kernel 7.2+ (Arch Linux / Ubuntu) with native in-tree `btusb` BLE support |
| **Runtime & Packaging** | Python 3.14+ (forward-compatible with 3.15), managed via **Poetry** |
| **Web Framework** | **FastAPI** (100% async), Uvicorn ASGI with WebSockets (`uvloop`) |
| **Database Engine** | **PostgreSQL 16/17** + **TimescaleDB** (Community/Apache 2.0) + **PostGIS** + `pgcrypto` |
| **Frontend Architecture** | **Jinja2** HTML5 shell + **Vanilla ES6+ JavaScript** (`importmap`), **Three.js** (WebGL2/WebGPU), Google Maps JS API (Polygon Geofence Editor). **No HTMX, no React/SPA bloat, no full-DOM swaps.** |
| **DOM Update Strategy** | Async WebSocket Event Bus performing surgical updates on individual `[data-telemetry-id]` DOM nodes and Three.js shader uniforms. |
| **Authentication** | Google OAuth2 + FIDO2/WebAuthn Passkeys (`py_webauthn`) + TOTP (`pyotp`). Seamless 7-day rolling refresh sessions with quick Passkey/OTP re-verification. |
| **Reverse Proxy & TLS** | **Nginx** with Let's Encrypt TLS 1.3 (`bodylanguage.blue-labs.org`) and WebSocket upgrade headers. |
| **Local AI Node** | NVIDIA RTX 3090 (24 GB VRAM) running **Ollama** (`qwen2.5vl:7b`) for Pass-1 routing & label OCR. |

---

## 3. Repository Directory Structure

```text
body-language/
├── PLAN.md                                   # This master architecture document
├── pyproject.toml                            # Poetry dependencies & Python 3.14+ config
├── .env.example                              # Template for master encryption key & bootstrap env
├── deploy/
│   ├── nginx/
│   │   └── bodylanguage.blue-labs.org.conf   # Nginx TLS 1.3, HTTP/2, WebSocket reverse proxy
│   └── systemd/
│       ├── body-language-web.service         # FastAPI Uvicorn systemd unit
│       ├── body-language-ble.service         # Ablegrid BLE scale listener daemon
│       └── body-language-seat.service        # Workstation keyboard/idle context daemon
├── migrations/
│   └── 001_initial_schema.sql                # TimescaleDB, PostGIS, RBAC, and Vault DDL
├── src/
│   └── body_language/
│       ├── __init__.py
│       ├── main.py                           # FastAPI entry point, lifespan tasks, routers
│       ├── core/
│       │   ├── config.py                     # Pydantic Settings loader
│       │   ├── crypto.py                     # Fernet / AES-GCM credential encryption & expiry check
│       │   └── websocket_bus.py              # Per-user async WebSocket pub/sub manager
│       ├── auth/
│       │   ├── oauth_google.py               # Google OAuth2 login & callback
│       │   ├── webauthn_passkey.py           # Passkey registration & 7-day fast re-auth
│       │   ├── totp.py                       # TOTP QR provisioning & verification
│       │   ├── session.py                    # 7-day rolling HTTPOnly token manager
│       │   └── rbac.py                       # Multi-tenant read/write & family read-only guards
│       ├── db/
│       │   ├── engine.py                     # SQLAlchemy 2.0 AsyncEngine (asyncpg)
│       │   └── models.py                     # Declarative ORM / SQL query helpers
│       ├── ingestion/
│       │   ├── scale_ble.py                  # Bleak scanner + transparent BIA math engine
│       │   ├── dexcom_poller.py              # Per-user Dexcom Share poller & Clarity CSV parser
│       │   ├── android_gateway.py            # Webhook for S26 Ultra (Health Connect + GPS)
│       │   └── workstation_daemon.py         # Linux input/idle tracker ("at keyboard" detector)
│       ├── services/
│       │   ├── meal_vision.py                # Pass 1 (RTX 3090 Ollama) -> Pass 2 (Cloud Vision)
│       │   ├── geofence_engine.py            # PostGIS ST_Contains point-in-polygon resolver
│       │   └── key_expiry_monitor.py         # Background scheduler for API/device key expiration
│       └── web/
│           ├── routes.py                     # View & JSON API endpoints
│           ├── templates/
│           │   ├── base.html.j2              # Master futuristic HUD chrome & avatar alert drawer
│           │   └── dashboard.html.j2         # Responsive desktop/mobile telemetry grid
│           └── static/
│               ├── css/
│               │   └── hud.css               # Futuristic medical styling, mobile breakpoints
│               └── js/
│                   ├── app.js                # Main ES module bootstrapper
│                   ├── bus.js                # Surgical WebSocket DOM & 3D uniform dispatcher
│                   ├── three_twin.js         # Three.js Holographic Digital Twin & Metabolic Ribbon
│                   ├── geofence_editor.js    # Interactive Google Maps polygon editor
│                   ├── meal_adjuster.js      # AI meal uploader, portion slider, item toggles
│                   └── vault_manager.js      # Encrypted key management & avatar badge handler
└── android_companion/                        # Optional S26 Ultra Kotlin Health Connect gateway spec
    └── README.md
```

---

## 4. Poetry Configuration (`pyproject.toml`)

```toml
[tool.poetry]
name = "body-language"
version = "0.1.0"
description = "Blue Labs Biometric, Metabolic & Contextual Telemetry Platform"
authors = ["Blue Labs <admin@blue-labs.org>"]
readme = "PLAN.md"
packages = [{include = "body_language", from = "src"}]

[tool.poetry.dependencies]
python = "^3.14"
fastapi = "^0.115.0"
uvicorn = {extras = ["standard"], version = "^0.32.0"}
sqlalchemy = {extras = ["asyncio"], version = "^2.0.36"}
asyncpg = "^0.30.0"
geoalchemy2 = "^0.16.0"
pydantic = "^2.10.0"
pydantic-settings = "^2.6.0"
jinja2 = "^3.1.4"
python-multipart = "^0.0.17"
httpx = "^0.28.0"
cryptography = "^44.0.0"
pyotp = "^2.9.0"
webauthn = "^2.2.0"
authlib = "^1.3.2"
itsdangerous = "^2.2.0"
bleak = "^0.22.3"
pydexcom = "^0.4.1"
pillow = "^11.0.0"

[tool.poetry.group.dev.dependencies]
pytest = "^8.3.0"
pytest-asyncio = "^0.24.0"
ruff = "^0.8.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

---

## 5. Database Schema (`migrations/001_initial_schema.sql`)

Designed for **PostgreSQL + TimescaleDB + PostGIS**. Supports multi-user family accounts, granular read-only sharing, encrypted API/device credentials with expiration warnings, transparent BIA scale equations, and spatial geofencing.

```sql
-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS timescaledb;
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- 1. Users & Biometric Baseline (Used for Scale BIA Equations)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT UNIQUE NOT NULL,
    display_name TEXT NOT NULL,
    avatar_url TEXT,
    is_app_owner BOOLEAN DEFAULT FALSE,
    birth_date DATE NOT NULL,
    biological_sex TEXT CHECK (biological_sex IN ('male', 'female')) NOT NULL,
    height_cm NUMERIC(5,2) NOT NULL,
    totp_secret_encrypted BYTEA,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 2. WebAuthn Passkey Credentials
CREATE TABLE user_passkeys (
    credential_id BYTEA PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    public_key BYTEA NOT NULL,
    sign_count BIGINT NOT NULL DEFAULT 0,
    device_name TEXT,
    transports TEXT[],
    created_at TIMESTAMPTZ DEFAULT NOW(),
    last_used_at TIMESTAMPTZ
);

-- 3. Multi-Tenant Family Profile Sharing (RBAC)
-- Users always have Read/Write on their own user_id.
-- Rows here grant Read-Only visibility of specific scopes to family members.
CREATE TABLE profile_shares (
    owner_user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    viewer_user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    scopes TEXT[] DEFAULT ARRAY['glucose', 'vitals', 'scale', 'meals', 'context'],
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (owner_user_id, viewer_user_id)
);

-- 4. Encrypted Credential & API Key Vault
-- Stores per-user device tokens (Dexcom, Fitdays, S26 Webhook) and App Owner keys (Claude, Gemini, Maps)
CREATE TABLE credential_vault (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    is_system_wide BOOLEAN DEFAULT FALSE, -- True for App Owner's shared Vision/Maps API keys
    provider TEXT NOT NULL,               -- 'dexcom', 'claude_api', 'gemini_api', 'google_maps', 's26_gateway'
    key_label TEXT NOT NULL,
    encrypted_payload BYTEA NOT NULL,     -- Encrypted with BLUELABS_MASTER_KEY via Fernet/AES-GCM
    issued_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ,               -- Nullable; drives UI avatar warning badge
    warn_days_before INTEGER DEFAULT 7,
    last_verified_at TIMESTAMPTZ,
    status TEXT DEFAULT 'active'
);

-- 5. Spatial Geofences (PostGIS Polygons)
CREATE TABLE geofences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    label TEXT NOT NULL,                  -- e.g., 'Main Desk', 'Front & Back Lawn', 'Gym'
    activity_category TEXT NOT NULL,      -- 'SEDENTARY_KEYBOARD', 'YARD_WORK', 'WORKOUT', 'HOME_GENERAL'
    color_hex TEXT DEFAULT '#00F0FF',
    boundary GEOMETRY(POLYGON, 4326) NOT NULL,
    extrusion_height_m REAL DEFAULT 3.5,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_geofences_boundary ON geofences USING GIST (boundary);

-- 6. Hypertable: Continuous Glucose Readings (Dexcom G7)
CREATE TABLE glucose_readings (
    time TIMESTAMPTZ NOT NULL,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    value_mg_dl SMALLINT NOT NULL,
    value_mmol_l NUMERIC(4,2) NOT NULL,
    trend_arrow TEXT,
    trend_rate NUMERIC(4,2),
    source TEXT DEFAULT 'dexcom_g7',
    UNIQUE (time, user_id)
);
SELECT create_hypertable('glucose_readings', 'time');

-- 7. Hypertable: Watch 9 & S26 Ultra Vital Telemetry
CREATE TABLE vital_telemetry (
    time TIMESTAMPTZ NOT NULL,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    metric_type TEXT NOT NULL,            -- 'heart_rate', 'spo2', 'steps_delta', 'skin_temp', 'sleep_stage'
    value NUMERIC(8,2) NOT NULL,
    unit TEXT NOT NULL,
    source_device TEXT DEFAULT 'galaxy_watch_9'
);
SELECT create_hypertable('vital_telemetry', 'time');
CREATE INDEX idx_vitals_user_metric ON vital_telemetry (user_id, metric_type, time DESC);

-- 8. Hypertable: Ablegrid Scale Measurements & Formula Verification
CREATE TABLE scale_measurements (
    time TIMESTAMPTZ NOT NULL,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    weight_kg NUMERIC(5,2) NOT NULL,
    weight_lbs NUMERIC(5,2) NOT NULL,
    impedance_ohms INTEGER,
    -- Direct Python Formula Calculations
    calc_bmi NUMERIC(4,2),
    calc_body_fat_pct NUMERIC(4,2),
    calc_lbm_kg NUMERIC(5,2),
    calc_muscle_mass_kg NUMERIC(5,2),
    calc_water_pct NUMERIC(4,2),
    calc_visceral_fat NUMERIC(4,1),
    -- Fitdays / External Reported Values (for delta verification)
    external_body_fat_pct NUMERIC(4,2),
    external_muscle_mass_kg NUMERIC(5,2),
    external_water_pct NUMERIC(4,2),
    formula_version TEXT DEFAULT 'bluelabs_bia_v1',
    raw_ble_hex TEXT,
    UNIQUE (time, user_id)
);
SELECT create_hypertable('scale_measurements', 'time');

-- 9. Hypertable: Contextual Activity & Location Stream
CREATE TABLE context_events (
    time TIMESTAMPTZ NOT NULL,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    location GEOMETRY(POINT, 4326),
    matched_geofence_id UUID REFERENCES geofences(id) ON DELETE SET NULL,
    posture_state TEXT NOT NULL,          -- 'AT_KEYBOARD', 'STILL', 'WALKING', 'MOWING', 'IN_VEHICLE'
    kpm_rate INTEGER,                     -- Keystrokes/actions per minute from Linux desktop daemon
    heart_rate_snapshot SMALLINT,
    source TEXT NOT NULL                  -- 'workstation_daemon' | 's26_ultra_gateway'
);
SELECT create_hypertable('context_events', 'time');

-- 10. AI-Assisted Meal Logs with Live Portion Adjustment
CREATE TABLE meal_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    time TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    photo_url TEXT,
    image_classification TEXT CHECK (image_classification IN ('FOOD_PHOTO', 'NUTRITION_CHART', 'MANUAL')),
    pass1_model TEXT DEFAULT 'ollama/qwen2.5vl:7b',
    pass2_model TEXT,
    -- Base 100% Estimate from AI or Nutrition Label
    base_calories INTEGER NOT NULL,
    base_carbs_g NUMERIC(6,2) NOT NULL,
    base_protein_g NUMERIC(6,2) NOT NULL,
    base_fat_g NUMERIC(6,2) NOT NULL,
    base_fiber_g NUMERIC(6,2) DEFAULT 0,
    -- Adjustable Consumed Multiplier (0.0 = didn't eat, 0.5 = ate half, 1.5 = ate 150%)
    consumed_ratio NUMERIC(4,2) DEFAULT 1.00 NOT NULL,
    -- Computed Effective Totals (Generated Columns)
    effective_calories INTEGER GENERATED ALWAYS AS (ROUND(base_calories * consumed_ratio)) STORED,
    effective_carbs_g NUMERIC(6,2) GENERATED ALWAYS AS (ROUND(base_carbs_g * consumed_ratio, 2)) STORED,
    effective_protein_g NUMERIC(6,2) GENERATED ALWAYS AS (ROUND(base_protein_g * consumed_ratio, 2)) STORED,
    effective_fat_g NUMERIC(6,2) GENERATED ALWAYS AS (ROUND(base_fat_g * consumed_ratio, 2)) STORED,
    items_json JSONB NOT NULL DEFAULT '[]'::jsonb,
    notes TEXT
);
```

---

## 6. Core Backend Specifications

### 6.1 Authentication, Passkeys & 7-Day Rolling Sessions (`src/body_language/auth/`)
1. **Primary Identity:** Users authenticate initially via **Google OAuth2** (`authlib`).
2. **Second Factor & Fast Re-Auth:** Users enroll a **WebAuthn Passkey** (stored in `user_passkeys`—works with S26 Ultra fingerprint/screen lock and Chrome desktop hardware keys) and/or a **TOTP** authenticator app (`pyotp`).
3. **7-Day Rolling Session:**
   * Upon Passkey or TOTP verification, the server issues an `HttpOnly`, `Secure`, `SameSite=Lax` signed session cookie valid for **7 days**.
   * When the 7-day window expires (or when accessing the Encrypted Credential Vault), the UI triggers a non-disruptive modal overlay calling `navigator.credentials.get()` (Passkey biometric tap) or a 6-digit OTP prompt over WebSocket/Fetch without losing current page state.

### 6.2 Encrypted Credential Vault & Avatar Renewal Monitor (`src/body_language/core/crypto.py`)
* All user and system API keys are encrypted at rest using `cryptography.fernet.Fernet` initialized from `BLUELABS_MASTER_KEY`.
* **Expiration Tracking:** Every key in `credential_vault` supports an optional `expires_at` timestamp and `warn_days_before` threshold (default 7 days).
* **UI Notification Under User Avatar:**
  * On WebSocket connect (and every 6 hours via `key_expiry_monitor.py`), the server evaluates all keys belonging to the active user (plus system-wide keys if `is_app_owner = True`).
  * If any key is within its warning window or expired, a `key_expiry_alert` event is pushed to the top-right User Profile Avatar, illuminating a glowing amber/crimson badge and populating the quick-renewal dropdown drawer.

### 6.3 Transparent Ablegrid Scale BLE & BIA Math (`src/body_language/ingestion/scale_ble.py`)
Uses a high-gain USB Bluetooth adapter on Linux Kernel 7.2+ (`btusb`) with `bleak` to capture broadcasts from the Ablegrid `FI2016LB`.

#### Viewable Biometric Equations (Rendered in UI Formula Inspector)
Given Weight $W_{\text{kg}}$, Height $H_{\text{cm}}$ ($H_{\text{m}} = H_{\text{cm}}/100$), Impedance $Z$ ($\Omega$), Age $A$ (years), and Sex coefficient $S \in \{1\text{ (male)}, 0\text{ (female)}\}$:

1. **Body Mass Index (BMI):**
   $$\text{BMI} = \frac{W_{\text{kg}}}{H_{\text{m}}^2}$$
2. **Resistive Index ($RI$):**
   $$RI = \frac{H_{\text{cm}}^2}{Z}$$
3. **Lean Body Mass ($\text{LBM}_{\text{kg}}$):**
   $$\text{LBM}_{\text{kg}} = 0.485 \cdot RI + 0.338 \cdot W_{\text{kg}} + 5.32 + 3.82 \cdot S - 0.095 \cdot A$$
4. **Body Fat Percentage ($\text{BF}\%$):**
   $$\text{BF}\% = \left(\frac{W_{\text{kg}} - \text{LBM}_{\text{kg}}}{W_{\text{kg}}}\right) \times 100$$
5. **Total Body Water Percentage ($\text{TBW}\%$):**
   $$\text{TBW}\% = \left(\frac{0.732 \times \text{LBM}_{\text{kg}}}{W_{\text{kg}}}\right) \times 100$$
6. **Skeletal Muscle Mass ($\text{SMM}_{\text{kg}}$):**
   $$\text{SMM}_{\text{kg}} = \text{LBM}_{\text{kg}} \times (0.54 \cdot S + 0.48 \cdot (1 - S))$$
7. **Visceral Fat Rating ($\text{VFR}$):**
   $$\text{VFR} = \max\left(1.0,\; 0.32 \cdot \text{BMI} + 0.08 \cdot A + 1.8 \cdot S - 0.008 \cdot RI\right)$$

The UI displays these equations alongside a comparison table showing **Blue Labs Direct Calculation** vs. **Fitdays Reported Calculation** and the percentage delta.

### 6.4 Two-Pass Hybrid AI Meal Pipeline (`src/body_language/services/meal_vision.py`)
1. **Pass 1 — Local RTX 3090 (`Ollama` / `qwen2.5vl:7b`):**
   * Inspects the uploaded image at zero cloud cost.
   * Prompt returns strict JSON: `{"type": "NUTRITION_CHART" | "FOOD_PHOTO", "confidence": 0.98}`.
   * **If `NUTRITION_CHART`:** `qwen2.5vl:7b` immediately performs structured OCR on the nutrition facts label, extracting serving size, calories, carbs, fat, fiber, and protein locally on the 3090.
2. **Pass 2 — Cloud Escalation for `FOOD_PHOTO`:**
   * If Pass 1 identifies a complex plated meal (`FOOD_PHOTO`), the service decrypts the App Owner's configured Cloud Vision key (Claude / Gemini Pro Vision) from `credential_vault`.
   * Requests an itemized array of detected foods on the plate with estimated gram weights and macronutrients:
     ```json
     {
       "items": [
         {"name": "Grilled Ribeye Steak", "grams": 220, "calories": 640, "carbs_g": 0, "protein_g": 62, "fat_g": 44, "included": true},
         {"name": "Garlic Mashed Potatoes", "grams": 180, "calories": 210, "carbs_g": 32, "protein_g": 4, "fat_g": 7, "included": true}
       ]
     }
     ```
3. **Interactive Portion Adjustment:**
   * The frontend renders individual item checkboxes (in case you didn't eat the side dish) and a master **Consumed Percentage Slider (`0%` to `200%`)**.
   * Sliding the bar dispatches a lightweight WebSocket event (`adjust_meal_portion`) that recalculates `effective_calories` and `effective_carbs_g` in real time.

---

## 7. Frontend Architecture: "Futuristic Medicine" HUD & Three.js

### 7.1 Zero-Bloat Surgical Event Bus (`bus.js`)
* All dynamic values in the Jinja2 template carry a `data-telemetry-id` attribute (e.g., `<span data-telemetry-id="cgm-current-mgdl">104</span>`).
* Incoming WebSocket frames look like:
  ```json
  {
    "event_type": "telemetry_tick",
    "domain": "glucose",
    "target_id": "cgm-current-mgdl",
    "value": "108",
    "trend_arrow": "↗",
    "status_class": "in-range",
    "uniforms": {"uGlucoseMgDl": 108.0, "uHeartRate": 72.0}
  }
  ```
* `bus.js` updates `textContent` and `dataset.status` directly on that specific element and passes `uniforms` straight into the active Three.js shader material—zero HTML string parsing or DOM tree replacements.

### 7.2 Three.js Interactive 3D Modules (`three_twin.js`)
Loaded via native browser `<script type="importmap">` pointing to Three.js ES modules:
1. **Holographic Biometric Twin:**
   * Renders a stylized translucent medical wireframe body with custom GLSL shaders and `UnrealBloomPass`.
   * **Vascular Pulse Uniform (`uHeartRate`):** Modulates the rhythmic shader pulse frequency to match your live Watch 9 heart rate.
   * **Metabolic Core Aura (`uGlucoseMgDl`):** Shifts core luminescence smoothly from cyan/emerald ($70\text{–}140\text{ mg/dL}$) to amber ($140\text{–}180\text{ mg/dL}$) to pulsing crimson ($>180$ or $<70\text{ mg/dL}$).
   * **Body Composition Layers:** Clicking the torso or limbs isolates the Muscle Mass vs. Body Fat shader layers and opens the live BIA formula comparison overlay.
2. **3D Chrono-Metabolic Ribbon:**
   * Plots a 3D spline ribbon across the last 24–72 hours where **X = Time**, **Y = Glucose ($mg/dL$)**, and **Z = Heart Rate / Step Cadence**.
   * Meal events float as interactive 3D icosahedron nodes along the ribbon; clicking a node opens its meal photo and portion slider.
   * Sedentary "At Keyboard" intervals vs. active "Lawn Mowing" intervals render as color-coded ground plane zones beneath the ribbon.

### 7.3 Mobile & Desktop Responsiveness (`hud.css`)
* **Desktop (Chrome Widescreen):** 3-column Futuristic HUD layout—Left: Vitals, CGM & Scale Cards; Center: Interactive Three.js 3D Viewport / Geofence Map toggle; Right: AI Meal Logger, Sedentary/Activity Correlator & Formula Inspector.
* **Mobile (S26 Ultra Chrome):** CSS Grid collapses into a touch-friendly bottom-tabbed HUD dock with full-bleed 3D touch-orbit support and instant camera snap for meal logging.

---

## 8. S26 Ultra & Workstation Context Ingestion

### 8.1 S26 Ultra Gateway (`POST /api/v1/ingest/mobile`)
Authenticated via a per-user bearer key stored in `credential_vault` (`provider = 's26_gateway'`). Accepts batched or real-time JSON payloads from an Android Health Connect companion app (or Tasker / Health Connect webhook):
```json
{
  "timestamp": "2026-11-01T19:30:00Z",
  "location": {"lat": 37.7749, "lon": -122.4194, "accuracy_m": 4.5},
  "posture_state": "WALKING",
  "vitals": [
    {"metric_type": "heart_rate", "value": 114, "unit": "bpm"},
    {"metric_type": "spo2", "value": 98, "unit": "%"},
    {"metric_type": "steps_delta", "value": 85, "unit": "count"}
  ]
}
```
Upon receipt, `geofence_engine.py` runs a PostGIS query:
```sql
SELECT id, label, activity_category
FROM geofences
WHERE user_id = :uid
  AND ST_Contains(boundary, ST_SetSRID(ST_MakePoint(:lon, :lat), 4326))
LIMIT 1;
```
This automatically tags whether you are inside the **Yard/Lawn** polygon, **Home**, or **Gym**, correlating physical location with heart rate and glucose trends.

### 8.2 Workstation "At Keyboard" Daemon (`workstation_daemon.py`)
A lightweight background script on your Linux workstation that checks X11/Wayland idle time and keystroke cadence every 60 seconds. When active at the keyboard, it sends `posture_state = 'AT_KEYBOARD'` to the local FastAPI endpoint so you can quantify "sitting in the chair" hours vs. active periods.

---

## 9. Deployment & Bootstrap Checklist (`bodylanguage.blue-labs.org`)

### Step 1: Install System Dependencies (Arch / Ubuntu)
```bash
# Arch Linux
sudo pacman -S postgresql timescaledb postgis python poetry nginx certbot-nginx bluez bluez-utils ollama

# Ubuntu
sudo apt install postgresql timescaledb-2-postgresql-16 postgresql-16-postgis-3 python3-poetry nginx certbot python3-certbot-nginx bluez
```

### Step 2: Initialize PostgreSQL + TimescaleDB + PostGIS
```bash
sudo -u postgres psql -c "CREATE DATABASE body_language;"
sudo -u postgres psql -d body_language -f migrations/001_initial_schema.sql
```

### Step 3: Pull Local Vision Model on RTX 3090
```bash
ollama pull qwen2.5vl:7b
```

### Step 4: Generate Master Vault Key & Configure Environment
```bash
poetry install
poetry run python -c "from cryptography.fernet import Fernet; print('BLUELABS_MASTER_KEY=' + Fernet.generate_key().decode())" >> .env
```

### Step 5: Provision TLS Certificate & Start Services
```bash
sudo cp deploy/nginx/bodylanguage.blue-labs.org.conf /etc/nginx/sites-available/
sudo ln -s /etc/nginx/sites-available/bodylanguage.blue-labs.org.conf /etc/nginx/sites-enabled/
sudo certbot --nginx -d bodylanguage.blue-labs.org
sudo systemctl enable --now body-language-web.service body-language-ble.service
```

---

## 10. Phased Implementation Roadmap

* **Phase 1: Core Skeleton, DB & Auth/Vault**
  * Initialize Poetry project, SQLAlchemy 2.0 async engine, and run `001_initial_schema.sql`.
  * Implement Google OAuth2 + WebAuthn Passkey + TOTP + 7-day rolling cookie.
  * Build Encrypted Credential Vault UI and Avatar Expiration Notification drawer.
* **Phase 2: Telemetry Ingestion Daemons**
  * Deploy `dexcom_poller.py` (using encrypted user credentials from the Vault).
  * Calibrate `scale_ble.py` against the Ablegrid `FI2016LB` raw byte frame and build the BIA Formula Comparator UI.
  * Expose `/api/v1/ingest/mobile` and `/api/v1/ingest/workstation` for Watch 9, S26 Ultra GPS, and keyboard activity tracking.
* **Phase 3: Futuristic Medical HUD & Three.js Viewport**
  * Build `base.html.j2`, `hud.css`, and `bus.js` surgical WebSocket updater.
  * Implement `three_twin.js` (Holographic Body Avatar + 3D Chrono-Metabolic Ribbon).
  * Build `geofence_editor.js` with Google Maps polygon drawing and PostGIS persistence.
* **Phase 4: Two-Pass AI Meal Logger**
  * Wire `meal_vision.py` to local RTX 3090 Ollama (`qwen2.5vl:7b`) and Cloud Vision API fallback.
  * Build `meal_adjuster.js` with live consumed-percentage sliders and per-ingredient toggles.