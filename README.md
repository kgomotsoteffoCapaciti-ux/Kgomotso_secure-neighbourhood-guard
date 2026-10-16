# Kgomotso_secure-neighbourhood-guard
Mobile app
# 🛡️ Secure Neighbourhood Guard (`save-my-community`)

> A robust, community-powered incident dispatch, tracking, and peer-to-peer safety coordination platform. Connects residents, vetted neighborhood patrollers, private security providers, and law enforcement (SAPS) in real time.

---

## 📑 Table of Contents
1. [Architecture Overview](#-architecture-overview)
2. [Tech Stack](#-tech-stack)
3. [Repository Directory Structure](#-repository-directory-structure)
4. [Core Modules & Component Breakdown](#-core-modules--component-breakdown)
5. [Data Flow & Fallback Architecture](#-data-flow--fallback-architecture)
6. [API & Webhook Endpoints](#-api--webhook-endpoints)
7. [Environment Variables](#-environment-variables)
8. [Local Development Setup](#-local-development-setup)

---

## 🏛️ Architecture Overview

The system operates across three primary layers:
1. **Client Tier (PWA/Mobile):** React + Vite client utilizing device APIs (Geolocation Watcher, Accelerometer for Shake-to-Alert, MediaRecorder for emergency audio).
2. **Realtime & Backend Tier (BaaS / Node):** Supabase (PostgreSQL + PostGIS for spatial queries, Row Level Security, Realtime subscriptions) and edge functions for multi-channel notification dispatch.
3. **Resilience & Fallback Layer:** Multi-protocol fallback to SMS/USSD gateways (e.g., Twilio / Africa's Talking) if web-socket connectivity fails or mobile data is exhausted.

---

## 🛠️ Tech Stack

| Domain | Technology | Purpose |
| :--- | :--- | :--- |
| **Frontend Framework** | React 18, TypeScript, Vite | Single-page responsive client |
| **Styling & UI Components** | Tailwind CSS, Radix UI, Lucide Icons | Accessible, high-contrast UI suited for emergency use |
| **State & Data Fetching** | TanStack Query (React Query), Zustand | Local cache management & offline sync queue |
| **Database & Auth** | Supabase (PostgreSQL + PostGIS) | Geospatial proximity queries, Auth, RLS |
| **Realtime Transport** | Supabase Realtime (WebSockets) | Live beacon broadcast & responder location streaming |
| **Mapping & GIS** | Mapbox GL JS / Leaflet | Visualizing heatmaps, responder positions, & safe routes |
| **Telemetry & Sensor APIs**| HTML5 DeviceMotionEvent, Geolocation API | Shake-to-alert triggers & background GPS breadcrumbs |
| **SMS/USSD Gateway** | Africa's Talking / Twilio | Offline panic triggers and outbound SMS broadcasts |

---

## 📂 Repository Directory Structure

```text
secure-neighbourhood-guard/
├── .github/
│   └── workflows/
│       ├── ci.yml                 # Linting, typecheck, unit tests
│       └── deploy.yml             # Automated deployment pipeline
├── public/
│   ├── audio/                     # Alarm sirens & alert chime assets
│   ├── icons/                     # PWA maskable icons
│   └── manifest.json              # Web app manifest for PWA installation
├── supabase/
│   ├── functions/                 # Supabase Edge Functions (Deno)
│   │   ├── dispatch-alert/        # Handles alert fanout (SMS, Push, In-App)
│   │   ├── handle-ussd/           # USSD session handling for offline panics
│   │   └── analyze-hotspots/      # Cron-driven risk scoring & incident aggregation
│   ├── migrations/                # Version-controlled SQL schemas & PostGIS functions
│   │   ├── 20260101_init.sql
│   │   └── 20260215_postgis_setup.sql
│   └── config.toml
├── src/
│   ├── assets/                    # Static vector graphics & brand assets
│   ├── components/                # Reusable UI primitives
│   │   ├── common/                # Buttons, Modal, Input, Badge, Toast
│   │   ├── layout/                # NavigationBar, Header, Sidebar, BottomNav
│   │   ├── maps/                  # MapContainer, HeatmapLayer, LivePinMarker
│   │   └── emergency/             # EmergencySlider, AudioRecorderIndicator
│   ├── features/                  # Domain-driven feature slices
│   │   ├── auth/
│   │   │   ├── components/        # LoginForm, RegisterForm, IDVerificationUpload
│   │   │   └── hooks/             # useAuth, useSession
│   │   ├── panic/
│   │   │   ├── components/        # PanicTriggerModal, ShakeDetectionOverlay
│   │   │   ├── hooks/             # useShakeDetector, useEmergencyAudioRecord
│   │   │   └── services/          # panicDispatcher.ts
│   │   ├── incident-reporting/
│   │   │   ├── components/        # IncidentForm, PhotoUploadGrid, StolenItemForm
│   │   │   └── hooks/             # useIncidentSubmit
│   │   ├── safe-walk/
│   │   │   ├── components/        # RouteTracker, CompanionPicker, StoppageAlert
│   │   │   └── hooks/             # useGeoWatch, useSafeWalkSession
│   │   ├── responders/
│   │   │   ├── components/        # RosterCard, VettingStatusBadge, PatrolToggle
│   │   │   └── services/          # responderLocationStream.ts
│   │   └── hotspots/
│   │       ├── components/        # HotspotFilterBar, IncidentTimelineView
│   │       └── utils/             # clusterRiskCalculator.ts
│   ├── hooks/                     # Shared low-level hooks (useNetworkStatus, useLocalStorage)
│   ├── lib/                       # External SDK clients (supabaseClient.ts, mapboxClient.ts)
│   ├── store/                     # Global stores (useAlertStore.ts, useUserLocationStore.ts)
│   ├── types/                     # Shared TypeScript interfaces & DB definitions
│   │   ├── database.types.ts
│   │   └── incident.types.ts
│   ├── utils/                     # Formatting, geo-distance (Haversine), cryptographic helpers
│   ├── App.tsx                    # Root routing & context provider wrappers
│   ├── main.tsx                   # Entry point
│   └── index.css                  # Tailwind directives & core layout rules
├── tests/
│   ├── unit/                      # Vitest component & hook tests
│   └── e2e/                       # Playwright tests (incident flow, panic trigger)
├── .env.example                   # Template for environment variables
├── package.json
├── tsconfig.json
├── tailwind.config.js
└── vite.config.ts
-- Find verified responders within 3km of an incident
SELECT id, full_name, phone_number,
       ST_Distance(current_location, ST_MakePoint(incident_lng, incident_lat)::geography) AS distance_meters
FROM responder_profiles
WHERE is_active = true 
  AND is_verified = true
  AND ST_DWithin(current_location, ST_MakePoint(incident_lng, incident_lat)::geography, 3000)
ORDER BY distance_meters ASC;
-- Find verified responders within 3km of an incident
SELECT id, full_name, phone_number,
       ST_Distance(current_location, ST_MakePoint(incident_lng, incident_lat)::geography) AS distance_meters
FROM responder_profiles
WHERE is_active = true 
  AND is_verified = true
  AND ST_DWithin(current_location, ST_MakePoint(incident_lng, incident_lat)::geography, 3000)
ORDER BY distance_meters ASC;
# App Configuration
VITE_APP_ENV=development
VITE_SITE_URL=http://localhost:5173

# Supabase Credentials
VITE_SUPABASE_URL=[https://your-project.supabase.co](https://your-project.supabase.co)
VITE_SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key

# Map Services
VITE_MAPBOX_ACCESS_TOKEN=pk.your_mapbox_token

# SMS / Offline Emergency Gateway
AFRICAS_TALKING_USERNAME=your_username
AFRICAS_TALKING_API_KEY=your_api_key
SMS_FALLBACK_SHORTCODE=31022
