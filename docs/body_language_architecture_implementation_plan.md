# Blue Labs — Body Language (`bodylanguage.blue-labs.org`)

**Architecture, Database Schema, and Implementation Master Plan**

## 1. Executive Summary & Core Objectives

**Body Language** is a self-hosted, privacy-first biometric and contextual telemetry platform developed by **Blue Labs**. It aggregates real-time health, body composition, nutrition, and environmental activity streams into a local PostgreSQL + TimescaleDB + PostGIS database, serving an interactive, event-driven 3D/2D web interface over FastAPI.

### Primary Hardware & Sensor Ecosystem

* **Continuous Glucose Monitor (CGM):** Dexcom G7 (5-minute interstitial glucose, trend rate, arrows, and Clarity/Share reconciliation).
* **Wearable Vitals & Activity:** Samsung Galaxy Watch 9 paired with Samsung Galaxy S26 Ultra (heart rate, $SpO_2$, step counts, sleep, skin temperature, GPS/activity state via Android Health Connect).
* **Smart Body Fat Scale:** Ablegrid `FI2016LB` ("A-Scale X" / Fitdays protocol) captured locally via Bluetooth Low Energy (BLE) with transparent Python bioelectrical impedance ($\Omega$) mathematical equations and side-by-side Fitdays calculation verification.
* **Activity & Posture Context:** Workstation X11/Wayland keyboard/mouse idle daemon + PostGIS polygon geofences (e.g., sitting at keyboard vs. mowing the lawn) to correlate sedentary vs. active states with glycemic and cardiac responses.
* **AI-Assisted Meal & Nutrition Logging:** Two-pass hybrid vision pipeline utilizing a local 24 GB NVIDIA RTX 3090 (`Ollama`) for zero-cost classification and nutrition label OCR, escalating complex plated food photos to Cloud Vision APIs, paired with real-time consumed-portion sliders.

---

## 2. Target Technology Stack & Modern Semantics Mandate

| Layer | Technology / Specification |
| :--- | :--- |
| **OS & Kernel** | Linux Kernel 7.2+ (Arch Linux / Ubuntu) with native in-tree `btusb` BLE support |
| **Runtime & Packaging** | **Python 3.14+** (forward-compatible with 3.15), managed via **Poetry**. Uses modern PEP 695 `type` aliases, generic syntax, `asyncio.TaskGroup`, structural pattern matching (`match`/`case`), and `@dataclass(slots=True)`. |
| **Validation & Settings** | **Pydantic v2** strict mode validation across all API boundaries, WebSocket frames, and database `tunables`. Bootstrap `.env` holds only `DATABASE_URL` and `BLUELABS_MASTER_KEY`; all runtime tunables live in the PostgreSQL `tunables` meta table. |
| **Web Framework** | **FastAPI** (100% async), Uvicorn ASGI with WebSockets (`uvloop`) |
| **Database Engine** | **PostgreSQL 16/17** + **TimescaleDB** (Community/Apache 2.0) + **PostGIS** + `pgcrypto` |
| **HTML5 Semantics** | Strict structural consistency across all views: `<header>`, `<nav>`, `<main>`, `<aside>`, `<footer>`, native `<dialog>` with blurred `::backdrop` scrims, and Popover API (`popover="auto"`) triggered by vertical tri-dot kebab (`⋮`) buttons. |
| **CSS Architecture** | **Modern Nested & `@layer` CSS** (`@layer reset, tokens, themes, layout, components, utilities`). 100% relative color derivation via **`oklch(from ...)`**, `light-dark()`, CSS Grid/Subgrid, and container queries. Includes three core themes (`future-medicine`, `black-cat`, `forestry`) and three color modes (`light`, `dark`, `device`). |
| **JavaScript Semantics** | **Vanilla ES2025+ Modules** (`importmap`), **Three.js**, `requestAnimationFrame`-synchronized DOM/Canvas mutations, anticipatory `<link rel="modulepreload">` / `fetch` prefetching, and **Offline Write-Safety Queue** (`localStorage` + `IndexedDB`) for resilient mutations. |
| **Time & Unit Discipline** | **Strict UTC** (`TIMESTAMPTZ` / ISO-8601 `Z`) throughout all Python services, SQL storage, and WebSocket payloads. Converted to user timezone and preferred **Units of Measure** (`mg/dL` vs `mmol/L`, `lbs` vs `kg`, `°F` vs `°C`, `in` vs `cm`) strictly at the final presentation layer via `Intl.DateTimeFormat` and client unit-conversion formatters. |
| **Testing** | **Pytest** (`pytest-asyncio`) focused on thorough, pragmatic unit and integration coverage of BIA math, Pydantic models, `tunables` resolution, crypto vaulting, and offline sync idempotency without overengineered mocks. |

---

## 3. Repository Directory Structure

```text
body-language/
├── PLAN.md                                   # This master architecture document
├── pyproject.toml                            # Poetry dependencies & Python 3.14+ config
├── .env.example                              # Bootstrap-only env (DB URL + Master Key)
├── deploy/
│   ├── nginx/
│   │   └── bodylanguage.blue-labs.org.conf   # Nginx TLS 1.3, HTTP/2, WebSocket reverse proxy
│   └── systemd/
│       ├── body-language-web.service         # FastAPI Uvicorn systemd unit
│       ├── body-language-ble.service         # Ablegrid BLE scale listener daemon
│       └── body-language-seat.service        # Workstation keyboard/idle context daemon
├── migrations/
│   └── 001_initial_schema.sql                # Tunables, Users, TimescaleDB, PostGIS, Vault DDL
├── src/
│   └── body_language/
│       ├── __init__.py
│       ├── main.py                           # FastAPI entry point, lifespan tasks, routers
│       ├── core/
│       │   ├── bootstrap.py                  # Minimal .env bootstrap (DB URI + Master Key only)
│       │   ├── tunables.py                   # Async DB `tunables` registry with LISTEN/NOTIFY cache
│       │   ├── crypto.py                     # Fernet / AES-GCM credential encryption & expiry check
│       │   ├── schemas.py                    # Pydantic v2 strict models (UTC enforced)
│       │   └── websocket_bus.py              # Per-user async WebSocket pub/sub + Presence tracker
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
│           │   ├── base.html.j2              # Master structural shell (<header>, <nav>, <main>, <aside>, <footer>)
│           │   ├── dashboard.html.j2         # Responsive desktop/mobile telemetry grid
│           │   └── partials/
│           │       ├── overlays.html.j2      # Blurred-scrim <dialog> containers (Meals, Vault, Formulas, Profile)
│           │       └── header_presence.html.j2 # Top-right Google Docs-style presence & Avatar kebab menu
│           └── static/
│               ├── css/
│               │   └── main.css              # @layer + nested CSS + oklch(from ...) 3-theme engine
│               └── js/
│                   ├── app.js                # Main ES module bootstrapper & anticipatory prefetcher
│                   ├── activity_indicator.js # rAF-driven undulating horizontal tri-dot controller
│                   ├── offline_queue.js      # localStorage/IndexedDB write-ahead queue & replay engine
│                   ├── units_time.js         # Final-layer UTC -> Local time & Unit of Measure converter
│                   ├── bus.js                # Surgical WebSocket DOM, Presence & 3D uniform dispatcher
│                   ├── three_twin.js         # Three.js Holographic Digital Twin & Metabolic Ribbon
│                   ├── geofence_editor.js    # Interactive Google Maps polygon editor
│                   ├── meal_adjuster.js      # AI meal uploader, portion slider, item toggles
│                   └── vault_manager.js      # Encrypted key management & avatar badge handler
└── tests/                                    # Thorough, pragmatic Pytest suite
    ├── conftest.py
    ├── test_bia_formulas.py
    ├── test_tunables_registry.py
    ├── test_crypto_vault.py
    ├── test_pydantic_utc_schemas.py
    └── test_offline_idempotency.py
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

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

---

## 5. Database Schema (`migrations/001_initial_schema.sql`)

All timestamps are strictly `TIMESTAMPTZ` stored in **UTC**. All non-bootstrap application configurations and runtime parameters reside in the `tunables` meta table with PostgreSQL `NOTIFY` invalidation so changes take effect immediately without restarting FastAPI.

```sql
-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS timescaledb;
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Enforce UTC at database level
ALTER DATABASE body_language SET timezone TO 'UTC';

-- 0. Meta Table: Runtime Configuration & Tunables (`tunables`)
-- Holds all non-bootstrap settings (polling intervals, AI model names, BLE thresholds, rate limits, etc.)
CREATE TABLE tunables (
    key TEXT PRIMARY KEY,
    value JSONB NOT NULL,
    value_type TEXT NOT NULL CHECK (value_type IN ('int', 'float', 'bool', 'str', 'json')),
    category TEXT NOT NULL DEFAULT 'general', -- 'ingestion', 'ai_vision', 'security', 'ui', 'ble'
    description TEXT NOT NULL,
    is_hot_reloadable BOOLEAN DEFAULT TRUE,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT (NOW() AT TIME ZONE 'UTC'),
    updated_by UUID
);

-- Trigger to broadcast live tunable changes to FastAPI workers via LISTEN/NOTIFY
CREATE OR REPLACE FUNCTION notify_tunable_change() RETURNS trigger AS $$
BEGIN
    PERFORM pg_notify('tunables_updated', row_to_json(NEW)::text);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_tunables_notify
AFTER INSERT OR UPDATE ON tunables
FOR EACH ROW EXECUTE FUNCTION notify_tunable_change();

-- Seed default runtime tunables
INSERT INTO tunables (key, value, value_type, category, description) VALUES
('dexcom.poll_interval_seconds', '300'::jsonb, 'int', 'ingestion', 'Polling interval for Dexcom G7 readings'),
('ble.scale_rssi_min_dbm', '-90'::jsonb, 'int', 'ble', 'Minimum BLE RSSI threshold for Ablegrid scale packets'),
('ble.scale_debounce_seconds', '30'::jsonb, 'int', 'ble', 'Cooldown window after locking in a stabilized weigh-in'),
('ai.pass1_ollama_url', '"http://127.0.0.1:11434"'::jsonb, 'str', 'ai_vision', 'Local RTX 3090 Ollama endpoint'),
('ai.pass1_model', '"qwen2.5vl:7b"'::jsonb, 'str', 'ai_vision', 'Local model for Pass-1 classification & Nutrition OCR'),
('ai.pass2_provider', '"claude"'::jsonb, 'str', 'ai_vision', 'Cloud Vision provider for complex food photos (claude | gemini)'),
('ai.pass2_model', '"claude-3-7-sonnet-latest"'::jsonb, 'str', 'ai_vision', 'Model identifier for Pass-2 volumetric food estimation'),
('auth.session_rolling_days', '7'::jsonb, 'int', 'security', 'Rolling lifetime for Passkey/OTP verified sessions'),
('vault.default_warn_days', '7'::jsonb, 'int', 'security', 'Days prior to key expiration to illuminate avatar badge'),
('seat.idle_threshold_seconds', '180'::jsonb, 'int', 'ingestion', 'Seconds without input before leaving AT_KEYBOARD state')
ON CONFLICT (key) DO NOTHING;

-- 1. Users, Biometric Baseline & Profile Preferences (Theme, Color Mode, Units of Measure)
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
    -- UI Theme & Appearance Options
    theme_name TEXT NOT NULL DEFAULT 'future-medicine'
        CHECK (theme_name IN ('future-medicine', 'black-cat', 'forestry')),
    color_mode TEXT NOT NULL DEFAULT 'device'
        CHECK (color_mode IN ('light', 'dark', 'device')),
    -- Units of Measure Preferences (Stored as canonical JSONB, validated by Pydantic)
    unit_preferences JSONB NOT NULL DEFAULT '{
        "glucose": "mg/dL",
        "mass": "lbs",
        "length": "in",
        "temperature": "F",
        "energy": "kcal"
    }'::jsonb,
    created_at TIMESTAMPTZ NOT NULL DEFAULT (NOW() AT TIME ZONE 'UTC')
);

-- 2. WebAuthn Passkey Credentials
CREATE TABLE user_passkeys (
    credential_id BYTEA PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    public_key BYTEA NOT NULL,
    sign_count BIGINT NOT NULL DEFAULT 0,
    device_name TEXT,
    transports TEXT[],
    created_at TIMESTAMPTZ NOT NULL DEFAULT (NOW() AT TIME ZONE 'UTC'),
    last_used_at TIMESTAMPTZ
);

-- 3. Multi-Tenant Family Profile Sharing (RBAC)
CREATE TABLE profile_shares (
    owner_user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    viewer_user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    scopes TEXT[] DEFAULT ARRAY['glucose', 'vitals', 'scale', 'meals', 'context'],
    created_at TIMESTAMPTZ NOT NULL DEFAULT (NOW() AT TIME ZONE 'UTC'),
    PRIMARY KEY (owner_user_id, viewer_user_id)
);

-- 4. Encrypted Credential & API Key Vault
CREATE TABLE credential_vault (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    is_system_wide BOOLEAN DEFAULT FALSE,
    provider TEXT NOT NULL,
    key_label TEXT NOT NULL,
    encrypted_payload BYTEA NOT NULL,
    issued_at TIMESTAMPTZ NOT NULL DEFAULT (NOW() AT TIME ZONE 'UTC'),
    expires_at TIMESTAMPTZ,
    warn_days_before INTEGER DEFAULT 7,
    last_verified_at TIMESTAMPTZ,
    status TEXT DEFAULT 'active'
);

-- 5. Spatial Geofences (PostGIS Polygons)
CREATE TABLE geofences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    label TEXT NOT NULL,
    activity_category TEXT NOT NULL,
    color_hex TEXT DEFAULT '#00F0FF',
    boundary GEOMETRY(POLYGON, 4326) NOT NULL,
    extrusion_height_m REAL DEFAULT 3.5,
    created_at TIMESTAMPTZ NOT NULL DEFAULT (NOW() AT TIME ZONE 'UTC')
);
CREATE INDEX idx_geofences_boundary ON geofences USING GIST (boundary);

-- 6. Hypertable: Continuous Glucose Readings (Dexcom G7 - Canonical mg/dL & mmol/L)
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
    metric_type TEXT NOT NULL,
    value NUMERIC(8,2) NOT NULL,
    unit TEXT NOT NULL,
    client_mutation_id UUID,              -- Idempotency token for offline queue replay
    source_device TEXT DEFAULT 'galaxy_watch_9',
    UNIQUE (time, user_id, metric_type)
);
SELECT create_hypertable('vital_telemetry', 'time');
CREATE INDEX idx_vitals_user_metric ON vital_telemetry (user_id, metric_type, time DESC);

-- 8. Hypertable: Ablegrid Scale Measurements & Formula Verification (Canonical kg + lbs)
CREATE TABLE scale_measurements (
    time TIMESTAMPTZ NOT NULL,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    weight_kg NUMERIC(5,2) NOT NULL,
    weight_lbs NUMERIC(5,2) NOT NULL,
    impedance_ohms INTEGER,
    calc_bmi NUMERIC(4,2),
    calc_body_fat_pct NUMERIC(4,2),
    calc_lbm_kg NUMERIC(5,2),
    calc_muscle_mass_kg NUMERIC(5,2),
    calc_water_pct NUMERIC(4,2),
    calc_visceral_fat NUMERIC(4,1),
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
    posture_state TEXT NOT NULL,
    kpm_rate INTEGER,
    heart_rate_snapshot SMALLINT,
    client_mutation_id UUID,
    source TEXT NOT NULL
);
SELECT create_hypertable('context_events', 'time');

-- 10. AI-Assisted Meal Logs with Offline Write Idempotency & Portion Adjustment
CREATE TABLE meal_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_mutation_id UUID UNIQUE,       -- Guarantees safe offline queue replay without duplicates
    time TIMESTAMPTZ NOT NULL DEFAULT (NOW() AT TIME ZONE 'UTC'),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    photo_url TEXT,
    image_classification TEXT CHECK (image_classification IN ('FOOD_PHOTO', 'NUTRITION_CHART', 'MANUAL')),
    pass1_model TEXT DEFAULT 'ollama/qwen2.5vl:7b',
    pass2_model TEXT,
    base_calories INTEGER NOT NULL,
    base_carbs_g NUMERIC(6,2) NOT NULL,
    base_protein_g NUMERIC(6,2) NOT NULL,
    base_fat_g NUMERIC(6,2) NOT NULL,
    base_fiber_g NUMERIC(6,2) DEFAULT 0,
    consumed_ratio NUMERIC(4,2) DEFAULT 1.00 NOT NULL,
    effective_calories INTEGER GENERATED ALWAYS AS (ROUND(base_calories * consumed_ratio)) STORED,
    effective_carbs_g NUMERIC(6,2) GENERATED ALWAYS AS (ROUND(base_carbs_g * consumed_ratio, 2)) STORED,
    effective_protein_g NUMERIC(6,2) GENERATED ALWAYS AS (ROUND(base_protein_g * consumed_ratio, 2)) STORED,
    effective_fat_g NUMERIC(6,2) GENERATED ALWAYS AS (ROUND(base_fat_g * consumed_ratio, 2)) STORED,
    items_json JSONB NOT NULL DEFAULT '[]'::jsonb,
    notes TEXT
);
```

---

## 6. Core Backend Specifications (Python 3.14+ & Pydantic v2)

### 6.1 Database `tunables` Registry (`src/body_language/core/tunables.py`)

Only `DATABASE_URL` and `BLUELABS_MASTER_KEY` are read from `.env` at process launch. Everything else is dynamically served from the `tunables` table and kept hot in memory via `LISTEN tunables_updated`.

```python
import asyncio
import json
from typing import Any
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncEngine

type TunableKey = str
type TunableValue = int | float | bool | str | dict[str, Any] | list[Any]


class TunablesRegistry:
    """In-memory async cache backed by PostgreSQL `tunables` table + LISTEN/NOTIFY."""

    def __init__(self, engine: AsyncEngine) -> None:
        self._engine = engine
        self._cache: dict[TunableKey, TunableValue] = {}
        self._lock = asyncio.Lock()

    async def initialize(self) -> None:
        """Loads all tunables from PostgreSQL on FastAPI startup."""
        async with self._engine.connect() as conn:
            result = await conn.execute(text("SELECT key, value FROM tunables"))
            async with self._lock:
                self._cache = {row.key: row.value for row in result}

    def get[T: TunableValue](self, key: TunableKey, default: T) -> T:
        """Type-safe generic retrieval of a tunable parameter."""
        val = self._cache.get(key, default)
        return val  # type: ignore[return-value]

    async def set(self, key: TunableKey, value: TunableValue, value_type: str, description: str = "") -> None:
        """Upserts a tunable in PostgreSQL; trigger automatically notifies workers."""
        async with self._engine.begin() as conn:
            await conn.execute(
                text("""
                    INSERT INTO tunables (key, value, value_type, description, updated_at)
                    VALUES (:k, CAST(:v AS jsonb), :vt, :desc, NOW() AT TIME ZONE 'UTC')
                    ON CONFLICT (key) DO UPDATE
                    SET value = EXCLUDED.value,
                        updated_at = EXCLUDED.updated_at
                """),
                {"k": key, "v": json.dumps(value), "vt": value_type, "desc": description},
            )
        async with self._lock:
            self._cache[key] = value
```

### 6.2 Strict UTC & Unit Preference Pydantic Schemas (`src/body_language/core/schemas.py`)

```python
from datetime import datetime, timezone
from typing import Literal
from uuid import UUID
from pydantic import BaseModel, ConfigDict, Field, field_validator

type ThemeName = Literal["future-medicine", "black-cat", "forestry"]
type ColorMode = Literal["light", "dark", "device"]
type GlucoseUnit = Literal["mg/dL", "mmol/L"]
type MassUnit = Literal["lbs", "kg"]
type LengthUnit = Literal["in", "cm"]
type TempUnit = Literal["F", "C"]


class UnitPreferences(BaseModel):
    model_config = ConfigDict(extra="forbid", frozen=True)

    glucose: GlucoseUnit = "mg/dL"
    mass: MassUnit = "lbs"
    length: LengthUnit = "in"
    temperature: TempUnit = "F"
    energy: Literal["kcal", "kJ"] = "kcal"


class ProfileOptionsUpdate(BaseModel):
    model_config = ConfigDict(extra="forbid")

    theme_name: ThemeName
    color_mode: ColorMode
    unit_preferences: UnitPreferences


class TelemetryMutationPayload(BaseModel):
    """Base schema for queued client writes enforcing UTC timestamps & idempotency."""
    model_config = ConfigDict(extra="forbid")

    client_mutation_id: UUID
    recorded_at_utc: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))

    @field_validator("recorded_at_utc")
    @classmethod
    def ensure_utc(cls, v: datetime) -> datetime:
        if v.tzinfo is None:
            return v.replace(tzinfo=timezone.utc)
        return v.astimezone(timezone.utc)
```

---

## 7. Frontend Architecture: Structural Semantics, `oklch` Theme Engine & UX

### 7.1 Universal Page Structure (`src/body_language/web/templates/base.html.j2`)

Every page shares the exact same semantic skeleton. Non-full-page interactions open inside `<dialog class="scrim-overlay">` modals with blurred backdrops, and menus open via vertical tri-dot kebab (`⋮`) triggers.

```html
<!DOCTYPE html>
<html lang="en"
      data-theme="{{ user.theme_name | default('future-medicine') }}"
      data-color-mode="{{ user.color_mode | default('device') }}">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
  <title>Body Language — Blue Labs</title>

  <!-- Anticipatory Resource Preloading -->
  <link rel="preload" href="/static/css/main.css" as="style">
  <link rel="modulepreload" href="/static/js/app.js">
  <link rel="modulepreload" href="/static/js/bus.js">
  <link rel="modulepreload" href="/static/js/activity_indicator.js">
  <link rel="modulepreload" href="/static/js/offline_queue.js">
  <link rel="stylesheet" href="/static/css/main.css">

  <script type="importmap">
  {
    "imports": {
      "three": "https://cdn.jsdelivr.net/npm/three@0.170.0/build/three.module.js",
      "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.170.0/examples/jsm/"
    }
  }
  </script>
</head>
<body data-units='{{ user.unit_preferences | tojson }}'>

  <!-- 1. Universal Header -->
  <header class="app-header">
    <div class="brand-cluster">
      <span class="brand-glyph" aria-hidden="true"></span>
      <span class="brand-title">Body Language</span>
      <!-- Always-available Undulating Horizontal Tri-Dot Activity Indicator -->
      <output id="global-activity-indicator"
              class="tri-dot-loader"
              role="status"
              aria-live="polite"
              aria-label="Synchronizing telemetry"
              hidden>
        <span class="dot"></span>
        <span class="dot"></span>
        <span class="dot"></span>
      </output>
    </div>

    <!-- Primary Structural Navigation -->
    <nav class="app-nav" aria-label="Primary Views">
      <button type="button" data-view-target="twin" aria-current="page">Biometric Twin</button>
      <button type="button" data-view-target="ribbon">Metabolic Ribbon</button>
      <button type="button" data-view-target="geofence">Geofence &amp; Context</button>
    </nav>

    <!-- Top-Right Presence Cluster + User Profile Avatar + Kebab Menu -->
    <div class="presence-and-profile">
      <!-- Google Docs-style Live Presence Indicators (Who is viewing this profile) -->
      <div class="presence-strip" data-telemetry-id="presence-strip" aria-label="Active viewers"></div>

      <!-- Offline Queue Indicator Badge -->
      <span class="offline-queue-pill" data-telemetry-id="offline-queue-count" hidden>0 queued</span>

      <!-- User Profile Avatar with Key Renewal Notification Badge -->
      <div class="profile-cluster">
        <button type="button" class="avatar-button" data-overlay-open="dialog-profile-options">
          <img src="{{ user.avatar_url or '/static/img/default-avatar.svg' }}" alt="{{ user.display_name }}" class="avatar-img">
          <span class="avatar-alert-badge" data-telemetry-id="avatar-key-badge" hidden>0</span>
        </button>

        <!-- Vertical Tri-Dot Kebab Menu Trigger -->
        <button type="button"
                class="kebab-trigger"
                popovertarget="header-kebab-menu"
                aria-label="Open Actions Menu">
          <svg viewBox="0 0 24 24" width="20" height="20" aria-hidden="true">
            <circle cx="12" cy="5" r="2"/><circle cx="12" cy="12" r="2"/><circle cx="12" cy="19" r="2"/>
          </svg>
        </button>

        <!-- Kebab Popover Menu -->
        <menu id="header-kebab-menu" popover="auto" class="kebab-popover">
          <li><button type="button" data-overlay-open="dialog-meal-logger">Log Meal / Scan Label</button></li>
          <li><button type="button" data-overlay-open="dialog-bia-formulas">Scale BIA Equations</button></li>
          <li><button type="button" data-overlay-open="dialog-credential-vault">Encrypted Key Vault</button></li>
          <li><button type="button" data-overlay-open="dialog-tunables-editor">System Tunables</button></li>
          <li><button type="button" data-overlay-open="dialog-profile-options">Profile &amp; Theme Options</button></li>
        </menu>
      </div>
    </div>
  </header>

  <!-- 2. Universal Main Body -->
  <main class="app-main">
    {% block main_content %}{% endblock %}
  </main>

  <!-- 3. Universal Context Aside -->
  <aside class="app-aside" aria-label="Live Event Stream">
    {% block aside_content %}{% endblock %}
  </aside>

  <!-- 4. Universal Footer -->
  <footer class="app-footer">
    <span>Blue Labs &bull; Body Language Telemetry Engine</span>
    <span data-telemetry-id="connection-state">Connected (UTC Sync)</span>
  </footer>

  <!-- 5. Non-Full-Page Overlay Containers with Blurred Scrim (<dialog>) -->
  {% include "partials/overlays.html.j2" %}

  <script type="module" src="/static/js/app.js"></script>
</body>
</html>
```

---

### 7.2 Nested & Layered CSS with `oklch(from ...)` Relative Coloration (`src/body_language/web/static/css/main.css`)

All colors across the entire application derive mathematically from three root seed variables (`--theme-luminance`, `--theme-chroma`, `--theme-hue`) per theme, combined with light/dark/device polarity.

```css
@layer reset, tokens, themes, layout, components, overlays, utilities;

@layer themes {
  /* 1. Futuristic Sci-Fi Medicine (Default Cyan/Sapphire Medical HUD) */
  [data-theme='future-medicine'] {
    --theme-luminance: 0.50;
    --theme-chroma: 0.07;
    --theme-hue: 247;
    --accent-hue: 195;
  }

  /* 2. Black Cat (Obsidian Panther with Warm Amber/Gold Eye Accents) */
  [data-theme='black-cat'] {
    --theme-luminance: 0.36;
    --theme-chroma: 0.025;
    --theme-hue: 285;
    --accent-hue: 78;
  }

  /* 3. Forestry (Deep Pine Canopy, Moss & Warm Birch Bark) */
  [data-theme='forestry'] {
    --theme-luminance: 0.46;
    --theme-chroma: 0.065;
    --theme-hue: 148;
    --accent-hue: 110;
  }

  /* Color Mode Polarity: Light / Dark / Device-Driven */
  [data-color-mode='device'] { color-scheme: light dark; }
  [data-color-mode='light']  { color-scheme: light; }
  [data-color-mode='dark']   { color-scheme: dark; }
}

@layer tokens {
  :root {
    /* Seed Base Color */
    --seed-color: oklch(var(--theme-luminance) var(--theme-chroma) var(--theme-hue));

    /* Surface Hierarchy Derived Strictly via oklch(from ...) */
    --surface-canvas: light-dark(
      oklch(from var(--seed-color) calc(l + 0.44) calc(c * 0.35) h),
      oklch(from var(--seed-color) calc(l - 0.34) calc(c * 0.85) h)
    );
    --surface-card: oklch(from var(--surface-canvas) calc(l + 0.04) c h);
    --surface-elevated: oklch(from var(--surface-canvas) calc(l + 0.08) calc(c * 1.1) h);
    --surface-border: oklch(from var(--surface-canvas) calc(l + 0.16) c h / 0.45);

    /* Table Striping Derived from Table Base */
    --table-background-color: var(--surface-card);
    --table-row-background-color-odd: oklch(from var(--table-background-color) calc(l + 0.05) c h);
    --table-row-background-color-even: oklch(from var(--table-background-color) calc(l - 0.05) c h);

    /* Text & Interactive Accent Tokens */
    --text-primary: light-dark(
      oklch(from var(--seed-color) 0.18 calc(c * 0.5) h),
      oklch(from var(--seed-color) 0.94 calc(c * 0.4) h)
    );
    --text-muted: oklch(from var(--text-primary) calc(l - 0.25) calc(c * 0.7) h);
    --accent-primary: oklch(from var(--seed-color) 0.72 calc(c * 2.1) var(--accent-hue));
    --status-ok: oklch(from var(--seed-color) 0.74 0.16 152);
    --status-warn: oklch(from var(--seed-color) 0.78 0.17 75);
    --status-danger: oklch(from var(--seed-color) 0.65 0.22 25);
  }
}

@layer components {
  /* Top-Right Google Docs-Style Presence Strip & Avatar */
  .presence-and-profile {
    display: flex;
    align-items: center;
    gap: 0.75rem;

    & .presence-strip {
      display: flex;
      align-items: center;

      & .presence-pill {
        width: 2rem;
        height: 2rem;
        border-radius: 999px;
        border: 2px solid var(--accent-primary);
        background: var(--surface-elevated);
        margin-inline-start: -0.45rem;
        position: relative;
        transition: transform 160ms ease;

        &:hover {
          transform: translateY(-2px) scale(1.08);
          z-index: 5;
        }

        &::after {
          content: "";
          position: absolute;
          inset-block-end: 0;
          inset-inline-end: 0;
          width: 0.55rem;
          height: 0.55rem;
          border-radius: 50%;
          background: var(--status-ok);
        }
      }
    }

    & .kebab-trigger {
      display: inline-grid;
      place-items: center;
      padding: 0.4rem;
      border-radius: 0.5rem;
      background: transparent;
      color: var(--text-primary);
      border: 1px solid transparent;
      cursor: pointer;

      &:hover {
        background: oklch(from var(--surface-card) calc(l + 0.06) c h);
        border-color: var(--surface-border);
      }
    }
  }

  /* Undulating Horizontal Tri-Dot Loader (Driven by requestAnimationFrame) */
  .tri-dot-loader {
    display: inline-flex;
    align-items: center;
    gap: 0.35rem;
    padding-inline: 0.5rem;

    &[hidden] { display: none; }

    & .dot {
      width: 0.48rem;
      height: 0.48rem;
      border-radius: 50%;
      background: var(--accent-primary);
      transform: translateY(var(--dot-y, 0px)) scale(var(--dot-scale, 1));
      opacity: var(--dot-alpha, 0.85);
      will-change: transform, opacity;
    }
  }
}

@layer overlays {
  /* Blurred Scrim Overlay for Non-Full-Page Containers */
  dialog.scrim-overlay {
    background: oklch(from var(--surface-elevated) l c h / 0.92);
    color: var(--text-primary);
    border: 1px solid var(--surface-border);
    border-radius: 1rem;
    padding: 1.5rem;
    max-width: min(92vw, 44rem);
    box-shadow: 0 24px 64px oklch(0 0 0 / 0.45);

    &::backdrop {
      background: oklch(from var(--surface-canvas) calc(l - 0.15) c h / 0.55);
      backdrop-filter: blur(12px) saturate(130%);
    }
  }
}
```

---

### 7.3 `requestAnimationFrame` Undulating Tri-Dot Activity Controller (`src/body_language/web/static/js/activity_indicator.js`)

Guarantees immediate visual feedback using a sinusoidal wave driven by `requestAnimationFrame` whenever any network fetch, WebSocket command,BLE query, or AI inference is in flight.

```javascript
export class ActivityIndicator {
  #activeTasks = new Set();
  #rafId = null;
  #container;
  #dots;

  constructor(selector = "#global-activity-indicator") {
    this.#container = document.querySelector(selector);
    this.#dots = this.#container ? Array.from(this.#container.querySelectorAll(".dot")) : [];
  }

  begin(taskId = crypto.randomUUID()) {
    this.#activeTasks.add(taskId);
    if (this.#container && this.#container.hidden) {
      this.#container.hidden = false;
      this.#rafId = requestAnimationFrame((ts) => this.#animate(ts));
    }
    return () => this.end(taskId);
  }

  end(taskId) {
    this.#activeTasks.delete(taskId);
    if (this.#activeTasks.size === 0 && this.#container) {
      if (this.#rafId) cancelAnimationFrame(this.#rafId);
      this.#rafId = null;
      this.#container.hidden = true;
    }
  }

  #animate(timestamp) {
    if (this.#activeTasks.size === 0) return;
    const t = timestamp * 0.0065;

    this.#dots.forEach((dot, idx) => {
      const phase = t - idx * 0.85;
      const wave = Math.sin(phase);
      const yOffset = (wave * 4.5).toFixed(2);
      const scale = (1 + wave * 0.22).toFixed(3);
      const alpha = (0.65 + (wave + 1) * 0.175).toFixed(2);

      dot.style.setProperty("--dot-y", `${yOffset}px`);
      dot.style.setProperty("--dot-scale", scale);
      dot.style.setProperty("--dot-alpha", alpha);
    });

    this.#rafId = requestAnimationFrame((ts) => this.#animate(ts));
  }
}

export const activity = new ActivityIndicator();
```

---

### 7.4 Offline Write-Safety Queue & Anticipatory Loader (`src/body_language/web/static/js/offline_queue.js`)

Enqueues all user mutations (`meal_logs`, `profile_options`, `geofence` edits, `tunables`) in `localStorage` before transmission, ensuring zero data loss during connectivity drops or server restarts.

```javascript
import { activity } from "./activity_indicator.js";

const STORAGE_KEY = "bluelabs_write_ahead_queue_v1";

export class ResilientWriteQueue {
  constructor() {
    window.addEventListener("online", () => this.flush());
  }

  #readQueue() {
    try {
      return JSON.parse(localStorage.getItem(STORAGE_KEY) || "[]");
    } catch {
      return [];
    }
  }

  #writeQueue(items) {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(items));
    const badge = document.querySelector('[data-telemetry-id="offline-queue-count"]');
    if (badge) {
      badge.hidden = items.length === 0;
      badge.textContent = `${items.length} queued`;
    }
  }

  async enqueueAndSend(endpoint, method, payload) {
    const mutation = {
      client_mutation_id: payload.client_mutation_id || crypto.randomUUID(),
      recorded_at_utc: new Date().toISOString(),
      endpoint,
      method,
      payload,
    };

    const queue = this.#readQueue();
    queue.push(mutation);
    this.#writeQueue(queue);

    return this.flush();
  }

  async flush() {
    if (!navigator.onLine) return;
    const queue = this.#readQueue();
    if (queue.length === 0) return;

    const stopActivity = activity.begin("flush-offline-queue");
    const remaining = [];

    for (const item of queue) {
      try {
        const res = await fetch(item.endpoint, {
          method: item.method,
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({
            ...item.payload,
            client_mutation_id: item.client_mutation_id,
            recorded_at_utc: item.recorded_at_utc,
          }),
        });
        if (!res.ok && res.status >= 500) {
          remaining.push(item);
        }
      } catch {
        remaining.push(item);
      }
    }

    this.#writeQueue(remaining);
    stopActivity();
  }
}
```

---

### 7.5 Final-Layer UTC-to-Local Time & Unit of Measure Formatter (`src/body_language/web/static/js/units_time.js`)

All backend and WebSocket payloads remain strictly in canonical UTC and metric/canonical units until rendered into DOM text nodes by this module.

```javascript
export class UserPresentationFormatter {
  constructor(unitPreferences) {
    this.units = unitPreferences;
    this.timeFormatter = new Intl.DateTimeFormat(undefined, {
      hour: "2-digit",
      minute: "2-digit",
      second: "2-digit",
      month: "short",
      day: "numeric",
    });
  }

  formatTimestamp(utcIsoString) {
    const date = new Date(utcIsoString);
    return this.timeFormatter.format(date);
  }

  formatGlucose(mgDl) {
    if (this.units.glucose === "mmol/L") {
      return `${(mgDl / 18.0182).toFixed(1)} mmol/L`;
    }
    return `${Math.round(mgDl)} mg/dL`;
  }

  formatMass(kg) {
    if (this.units.mass === "lbs") {
      return `${(kg * 2.20462262).toFixed(1)} lbs`;
    }
    return `${Number(kg).toFixed(2)} kg`;
  }

  formatTemperature(celsius) {
    if (this.units.temperature === "F") {
      return `${((celsius * 9) / 5 + 32).toFixed(1)} °F`;
    }
    return `${Number(celsius).toFixed(1)} °C`;
  }
}
```

---

## 8. Pragmatic Pytest Suite Specification (`tests/`)

Focused, high-value unit tests verifying mathematical accuracy, UTC timestamp enforcement, `tunables` fallback behavior, encryption round-trips, and offline queue idempotency:

1. **`tests/test_bia_formulas.py`:** Tests `calculate_bia_composition()` against known male/female heights, ages, weights, and impedances ($\Omega$), verifying exact BMI, Lean Body Mass, Body Fat %, and Visceral Fat boundaries.
2. **`tests/test_pydantic_utc_schemas.py`:** Verifies that naive datetimes or non-UTC timezone offsets submitted by clients are automatically normalized to `timezone.utc`, and invalid `ThemeName` or `UnitPreferences` values raise `ValidationError`.
3. **`tests/test_tunables_registry.py`:** Verifies type-safe generic retrieval `tunables.get("dexcom.poll_interval_seconds", 300)` and hot-reload cache updates.
4. **`tests/test_crypto_vault.py`:** Verifies `CredentialVaultCrypto` Fernet round-trips and `evaluate_key_health()` state transitions (`valid` $\rightarrow$ `expiring_soon` $\rightarrow$ `expired`).
5. **`tests/test_offline_idempotency.py`:** Verifies that replaying a `TelemetryMutationPayload` with the same `client_mutation_id` is safely deduplicated without raising unhandled integrity errors.

---

## 9. Phased Implementation Roadmap

* **Phase 1: Core Skeleton, `tunables` Meta Table, Auth & Vault**
  * Initialize Poetry project, SQLAlchemy 2.0 async engine, and run `001_initial_schema.sql`.
  * Implement `TunablesRegistry` with PostgreSQL `LISTEN/NOTIFY` hot-reloading.
  * Implement Google OAuth2 + WebAuthn Passkey + TOTP + 7-day rolling cookie.
  * Build Encrypted Credential Vault UI and Avatar Expiration Notification badge.
* **Phase 2: Structural Shell, `oklch` Theme Engine & Resilient Event Bus**
  * Build `base.html.j2` (`<header>`, `<nav>`, `<main>`, `<aside>`, `<footer>`, `<dialog class="scrim-overlay">`, and kebab `⋮` popovers).
  * Implement `main.css` with `@layer`, CSS nesting, and `oklch(from ...)` relative color derivations for `future-medicine`, `black-cat`, and `forestry` across `light`, `dark`, and `device` modes.
  * Wire `activity_indicator.js` (`requestAnimationFrame` tri-dot loader), `offline_queue.js`, `units_time.js`, and Google Docs-style live viewer presence pills.
* **Phase 3: Telemetry Ingestion Daemons & Three.js Viewport**
  * Deploy `dexcom_poller.py` and `scale_ble.py` (with transparent BIA Formula Inspector overlay).
  * Expose `/api/v1/ingest/mobile` and `/api/v1/ingest/workstation` for Watch 9, S26 Ultra GPS, and keyboard activity tracking.
  * Implement `three_twin.js` (Holographic Body Avatar + 3D Chrono-Metabolic Ribbon) and `geofence_editor.js`.
* **Phase 4: Two-Pass AI Meal Logger & Pytest Verification**
  * Wire `meal_vision.py` to local RTX 3090 Ollama (`qwen2.5vl:7b`) and Cloud Vision API fallback with live consumed-percentage sliders.
  * Run the `pytest` verification suite across all core modules.