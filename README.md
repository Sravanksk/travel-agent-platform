# Travel Agency Platform — Engineering Design & Architecture Document

**Document Version:** 1.0  
**Classification:** Internal — Engineering & Product  
**Status:** Draft

---

## Table of Contents

1. [Executive Summary & Product Vision](#1-executive-summary--product-vision)
2. [System Architecture Overview](#2-system-architecture-overview)
3. [Core Modules & Feature Design](#3-core-modules--feature-design)
4. [Data Sources, APIs & Cost Analysis](#4-data-sources-apis--cost-analysis)
5. [AI Agent Design & LLM Integration](#5-ai-agent-design--llm-integration)
6. [PDF & Export Engine Design](#6-pdf--export-engine-design)
7. [Infrastructure, Scalability & Cost Estimates](#7-infrastructure-scalability--cost-estimates)
8. [Security, Compliance & Data Privacy](#8-security-compliance--data-privacy)

---

## 1. Executive Summary & Product Vision

### 1.1 What the Platform Is

This platform is an AI-native travel agency solution designed to eliminate the operational and administrative burden from travel planning workflows. It supports a travel agent across the complete lifecycle of a client inquiry — from the moment a call is received, through proposal generation and iterative edits, all the way to payment collection.

The platform is built around a single core principle: **let the agent focus on crafting experiences, not filling forms**.

### 1.2 Target Users

- **Travel agents** at boutique and mid-size travel agencies
- **Agency admins** managing team members, subscriptions, and brand settings
- **DMC (Destination Management Company) partners** whose documents serve as the primary pricing and itinerary source

### 1.3 Platform Lifecycle Coverage

| Lifecycle Stage | What the Platform Does |
|---|---|
| Client call received | Transcribe conversation, extract trip details via AI |
| Initial planning | Generate skeleton itinerary from voice/text input or DMC documents |
| Proposal creation | AI-generated day-by-day itinerary with hotels, flights, transfers, pricing |
| Iteration & revision | Natural-language edits to structure, days, pricing, transport |
| Export | Branded PDF and interactive web link with cinematic layouts |
| Payment & reconciliation | Client receivables, vendor payables, payment schedules, due-date tracking |

### 1.4 Design Philosophy

- **AI-first, not AI-added** — AI is the primary interaction layer, not a bolt-on feature
- **Document-aware** — DMC documents in any format are normalized and used as structured knowledge
- **Conversational editing** — every change is expressed in natural language; the system handles complexity
- **Synchronized state** — changes in any module (flights, hotels, pricing) propagate automatically
- **Agency-branded output** — every export carries the agency's identity consistently

---

## 2. System Architecture Overview

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                                 │
│           Web App (React/Next.js)  ·  Mobile-Responsive UI          │
└────────────────────────────┬────────────────────────────────────────┘
                             │ HTTPS / WebSocket
┌────────────────────────────▼────────────────────────────────────────┐
│                        API GATEWAY                                  │
│          Auth · Rate Limiting · Request Routing · Versioning         │
└───┬───────────────┬───────────────────┬──────────────────────┬──────┘
    │               │                   │                      │
┌───▼────┐   ┌──────▼──────┐   ┌────────▼───────┐   ┌────────▼──────┐
│ Auth   │   │  Trip &     │   │  AI Orchestr-  │   │  Export       │
│ Service│   │  Inquiry    │   │  ation Service │   │  Service      │
│        │   │  Service    │   │  (Agent Core)  │   │               │
└───┬────┘   └──────┬──────┘   └────────┬───────┘   └────────┬──────┘
    │               │                   │                     │
┌───▼───────────────▼───────────────────▼─────────────────────▼──────┐
│                        INTERNAL SERVICE BUS (Event-Driven)          │
│                        (Redis Pub/Sub or Kafka)                     │
└───┬───────────────┬───────────────────┬─────────────────────┬──────┘
    │               │                   │                     │
┌───▼────┐   ┌──────▼──────┐   ┌────────▼───────┐   ┌────────▼──────┐
│  User  │   │  Document   │   │  Pricing &     │   │  Reconcili-   │
│  &     │   │  Hub &      │   │  Inventory     │   │  ation        │
│ Agency │   │  Parser     │   │  Service       │   │  Service      │
│  DB    │   │  Service    │   │                │   │               │
└───┬────┘   └──────┬──────┘   └────────┬───────┘   └────────┬──────┘
    │               │                   │                     │
┌───▼───────────────▼───────────────────▼─────────────────────▼──────┐
│                        DATA LAYER                                   │
│   PostgreSQL · Redis · S3/Object Storage · Vector DB (Pinecone/PG)  │
└─────────────────────────────────────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────────┐
│                     EXTERNAL INTEGRATIONS                           │
│  LLM APIs · Flight APIs · Hotel APIs · Maps/Transit · Booking APIs  │
│  Image APIs · Payment Gateway · WhatsApp API                        │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 Architectural Pattern

The platform follows a **modular microservices** architecture with event-driven communication between services. Each service owns its domain data and exposes well-defined APIs. The AI Orchestration Service acts as the central brain, coordinating all services during planning, editing, and export operations.

**Key architectural choices:**

| Decision | Choice | Rationale |
|---|---|---|
| Frontend | Next.js (React) | SSR for fast initial load, SEO, file-based routing |
| Backend services | Node.js / Python (FastAPI) | Node for real-time/web, Python for AI/ML workloads |
| Primary DB | PostgreSQL (via Supabase or RDS) | Relational structure for trips, pricing, users |
| Cache layer | Redis | Session cache, pub/sub, API response caching |
| Object storage | AWS S3 / Cloudflare R2 | DMC documents, images, generated PDFs |
| Vector store | pgvector (on Postgres) or Pinecone | Document embeddings for semantic retrieval |
| Message bus | Redis Pub/Sub (MVP) → Kafka (scale) | Async event propagation between services |
| AI backbone | OpenAI GPT-4o + Function Calling | Primary LLM for planning, editing, extraction |
| Search/Retrieval | Hybrid: BM25 + vector similarity | Document chunk retrieval for RAG pipeline |

### 2.3 Deployment Architecture

```
┌──────────────────────────────────────────┐
│           AWS / GCP Cloud                │
│                                          │
│  ┌──────────┐    ┌─────────────────────┐ │
│  │ CDN      │    │  Load Balancer      │ │
│  │ (CF/CF)  │    │  (ALB / Nginx)      │ │
│  └────┬─────┘    └──────────┬──────────┘ │
│       │                     │            │
│  ┌────▼─────────────────────▼──────────┐ │
│  │        ECS / Kubernetes Cluster     │ │
│  │  ┌──────┐ ┌──────┐ ┌──────┐        │ │
│  │  │ API  │ │ Trip │ │ AI   │  ...   │ │
│  │  │ GW   │ │ Svc  │ │ Orch │        │ │
│  │  └──────┘ └──────┘ └──────┘        │ │
│  └─────────────────────────────────────┘ │
│                                          │
│  ┌───────────┐  ┌──────────┐  ┌───────┐ │
│  │ RDS       │  │ Redis    │  │ S3    │ │
│  │ Postgres  │  │ ElastiC. │  │ R2    │ │
│  └───────────┘  └──────────┘  └───────┘ │
└──────────────────────────────────────────┘
```

---

## 3. Core Modules & Feature Design

### 3.1 Module Map

```
Travel Agency Platform
├── 3.2  Auth & Agency Management
├── 3.3  Inquiry / Trip Lifecycle Management
├── 3.4  Voice & Text Input Parser (AI Extraction)
├── 3.5  Document Hub & DMC Document Parser
├── 3.6  Skeleton Trip Planner
├── 3.7  Full Trip Generation Engine
├── 3.8  Trip Editing Engine (Flights · Hotels · Days · Structure)
├── 3.9  Pricing & Packaging Engine
├── 3.10 Reconciliation & Payment Tracking
└── 3.11 Export Engine (PDF · Web Link · Dynamic Pages)
```

---

### 3.2 Auth & Agency Management

**Functional Requirements:**
- Sign up via Google OAuth or email/password
- Agency creation (new) or joining (via Agency ID)
- Role-based access control: `Admin` and `Agent`
- Admin approval workflow for new team members
- Agency profile: logo upload, brand colors, contact details
- Profile fields propagated to all client-facing exports

**Data Models:**

```
User
  id, name, email, password_hash, google_id,
  role (admin | agent), agency_id,
  mobile, whatsapp, title,
  created_at, updated_at

Agency
  id, name, logo_url, brand_colors (JSON),
  subscription_status, trips_remaining,
  created_at, approved_at, approved_by
```

**Auth Flow:**
- JWT-based authentication with refresh tokens
- Google OAuth via NextAuth.js or Auth0
- Agency join requests stored as pending; admin receives notification (email + in-app)
- Roles enforced at API gateway middleware level

---

### 3.3 Inquiry / Trip Lifecycle Management

**Trip Stages (State Machine):**

```
Inbound → Planning → Under Revision → Awaiting Approval
       → Awaiting Payment → Booked → Archived
```

**Trip Data Model (Core):**

```
Trip
  id, agency_id, agent_id, client_id,
  name, stage, travel_dates (start, end),
  destination_countries[], pax (adults, children, infants),
  currency, dietary_prefs[], special_requirements,
  created_at, updated_at, archived_at

TripVersion
  id, trip_id, version_number, structure (JSON),
  created_at, change_summary

Client
  id, agency_id, name, email, phone, whatsapp,
  notes, created_at
```

**Key Behaviors:**
- Every structural edit creates a new `TripVersion` — non-destructive versioning
- Trip stage transitions are tracked with timestamps and agent ID
- Trips do not consume quota at skeleton stage; quota deducted at full generation
- Archive/delete operations are soft-deletes

---

### 3.4 Voice & Text Input Parser

**Purpose:** Convert unstructured voice or typed natural language descriptions into structured trip parameters without any manual form-filling.

**Input:** "I just got a call from my client. They want 10 days in Vietnam in August — family of 3, 2 adults and a 1-year-old, flying from Mumbai."

**Output (structured JSON):**
```json
{
  "destination": ["Vietnam"],
  "duration_days": 10,
  "travel_month": "August",
  "pax": { "adults": 2, "children": 0, "infants": 1 },
  "origin_city": "Mumbai",
  "client_type": "family"
}
```

**Implementation:**

```
User Input (voice/text)
        │
        ▼
   [Web Speech API / Whisper]   ← voice transcription
        │
        ▼
   [LLM Extraction Prompt]
   System: "Extract structured trip parameters from this input."
   Output: validated JSON schema
        │
        ▼
   [Schema Validator]           ← zod / pydantic validation
        │
        ▼
   [Pre-fill Trip Form]         ← auto-populate UI fields
```

- Voice input: browser-native Web Speech API (free) or OpenAI Whisper API ($0.006/min)
- Extraction: GPT-4o with structured output / JSON mode
- Schema validation: Zod (frontend) + Pydantic (backend)
- Confidence scoring: fields with low confidence are highlighted for agent review

---

### 3.5 Document Hub & DMC Document Parser

**Purpose:** Accept DMC documents in any format (PDF, Word, Excel, HTML) and normalize them into a structured internal representation that can drive itinerary generation and pricing.

**Supported Input Formats:** PDF, DOCX, XLSX, HTML, plain text

**Processing Pipeline:**

```
Raw Document Upload (S3)
        │
        ▼
   [Format Detector]
        │
   ┌────┴────────────────┐
   ▼                     ▼
PDF/DOCX Parser       Excel Parser
(pypdf / docx2txt)    (openpyxl / pandas)
        │
        ▼
   [Text Chunker]         ← chunk by semantic section
        │
        ▼
   [Embedding Model]      ← text-embedding-3-small (OpenAI)
        │
        ▼
   [Vector Store]         ← pgvector / Pinecone
        │
        ▼
   [LLM Structured Extraction]
     - destinations, nights
     - hotels per category (3-star, 4-star)
     - inclusions / exclusions
     - pricing tables
     - activity lists
        │
        ▼
   [Normalized DMC Object]  ← stored in Postgres JSONB
```

**Normalized DMC Schema:**
```json
{
  "document_id": "uuid",
  "destination": "Vietnam",
  "duration_nights": 9,
  "price_tiers": ["3-star", "4-star"],
  "segments": [
    {
      "city": "Hanoi",
      "nights": 2,
      "hotels": {
        "3-star": { "name": "...", "price_per_night": 80 },
        "4-star": { "name": "...", "price_per_night": 130 }
      },
      "activities": [...]
    }
  ],
  "inclusions": [...],
  "exclusions": [...],
  "pricing_notes": "..."
}
```

**Document Hub:**
- Persistent library per agency — upload once, reuse across any trip
- Searchable by destination, duration, price tier
- Version history per document
- Access scoped to agency (not shared across agencies)

---

### 3.6 Skeleton Trip Planner

**Purpose:** Produce an initial lightweight structure — destinations, nights per city, rough route — before full generation. This is fast, free (no quota deduction), and serves as the agent's confirmation checkpoint.

**Input sources (3 modes):**

| Mode | Source | How It Works |
|---|---|---|
| Document-based | Uploaded or Hub DMC doc | Parse doc → extract structure → present as skeleton |
| Platform package DB | Built-in destination packages | AI matches user input to nearest package (~50+ countries) |
| Freeform AI | No document | LLM generates multiple route options creatively |

**Consistency Check:**
- If client requested 10 days but document covers 9 — flagged as inconsistent
- Agent can proceed (with pricing caveat) or adjust structure

**Replicate Mode:**
- Agent can modify an existing trip's skeleton as a starting point for a new one
- Smart diff shows what changed from source trip

**Skeleton Data Model:**
```json
{
  "trip_id": "uuid",
  "version": 1,
  "segments": [
    { "city": "Hanoi", "nights": 2 },
    { "city": "Halong Bay", "nights": 1 },
    { "city": "Da Nang", "nights": 3 },
    { "city": "Ho Chi Minh City", "nights": 2 }
  ],
  "total_nights": 8,
  "consistency_status": "consistent | inconsistent",
  "reference_document_id": "uuid | null"
}
```

---

### 3.7 Full Trip Generation Engine

**Purpose:** Transform the skeleton into a complete, client-ready itinerary — with real flights, hotels, transfers, day-by-day plans, and pricing — in one AI-powered generation step.

**Generation Pipeline:**

```
Skeleton + Agent Preferences
        │
        ▼
   [Key Variable Extraction]
   (e.g. hotel tier, arrival flexibility)
        │
        ▼
   [Parallel Data Fetch]
   ┌────────┬────────────┬────────────┬──────────┐
   ▼        ▼            ▼            ▼          ▼
Flights  Hotels      Transfers   Activities  Weather
(Amadeus/ (DMC doc +  (Google    (GetYourGu- /Crowd
Duffel)  Booking.com  Maps API)   ide API)   Data
        │
        ▼
   [LLM Planning Engine]
   - Merge all fetched data
   - Apply day planning preferences
   - Generate day-by-day narrative
   - Apply custom instructions
        │
        ▼
   [Complete Trip Object]
   - Flights (with live pricing)
   - Hotels (with images, descriptions)
   - Intercity transfers (with timing)
   - Day plans (structured activities)
   - Pricing breakdown
```

**Day Planning Configuration (Agent-Controlled):**
- Leisure days: keep free or fill with activities
- Arrival/departure days: plan conditionally (e.g., only if arriving before 4 PM)
- Meal preferences: include lunch/dinner slots
- Max activities per day
- Time format: Morning/Afternoon/Evening or exact slots
- Preferred start/end times per day

---

### 3.8 Trip Editing Engine

All components of a generated trip are editable. Editing is handled via natural language or direct UI manipulation.

#### Hotels
| Edit Method | How |
|---|---|
| AI suggestions | Curated alternatives surfaced per city |
| Name search | Fuzzy match → auto-fetch details |
| Booking confirmation upload | Image/text → OCR/parse → populate |
| Manual entry | Free-form hotel details |

#### Flights
| Edit Method | How |
|---|---|
| Ticket/screenshot upload | OCR → extract flight details |
| Flight number search | Live fetch from flight data API |
| Manual entry | Free-form flight details + pricing |

#### Day Plans
| Edit Method | How |
|---|---|
| Freeform text | "Add a night walk every evening in Hanoi" |
| AI-assisted (Plan a Day) | Suggest ideas → select → apply |
| Drag & drop | Reorder activities within a day |
| Copy-paste verbatim | Paste plan from external source → AI structures it |
| Bookable activities | Browse GetYourGuide → add with availability |

#### Trip Structure (Top-Level)
- Modify dates, durations, city order, transport modes
- Full natural-language support: "Postpone the trip by 5 days and add 1 more night in Da Nang"
- Full regeneration (not patching) — all dependencies recalculated

#### Sync Behavior
Any edit to flights/hotels/transport triggers:
1. Day plan stale flag → agent prompted to refresh
2. Pricing section auto-recalculation
3. Reconciliation section sync

---

### 3.9 Pricing & Packaging Engine

**Pricing Sources:**
- Hotels: from DMC document pricing table
- Flights: live inventory price
- Activities: bookable price or manual entry
- Transfers: manual or estimated

**Pricing Controls:**
- Base price per component
- Markup (percentage or flat) per component or globally
- Optional add-ons (e.g., visa fees, travel insurance)
- Tax component breakdown
- Package grouping: bundle multiple components under one price label

**Pricing Data Model:**
```json
{
  "trip_id": "uuid",
  "packages": [
    {
      "name": "Main Land Package",
      "items": [
        { "type": "hotel", "ref_id": "...", "base": 800, "markup_pct": 10, "final": 880 },
        { "type": "transfer", "label": "Airport Transfer", "base": 50, "markup_pct": 0, "final": 50 }
      ],
      "subtotal": 930
    }
  ],
  "flights": { "base": 620, "markup_pct": 5, "final": 651 },
  "optional_addons": [
    { "label": "Visa Fee", "amount": 250 }
  ],
  "taxes": [
    { "label": "GST 5%", "amount": 78 }
  ],
  "grand_total": 1909
}
```

---

### 3.10 Reconciliation & Payment Tracking

**Purpose:** Replace spreadsheets and WhatsApp notes with a built-in financial tracking system tied directly to the trip.

**Core Entities:**
- **Receivable:** money owed by client to agency
- **Payable:** money owed by agency to vendor/DMC

**Features:**
- Split receivables and payables into logical buckets (flights, land package, activities)
- Multi-stage payment schedules (e.g., 50% advance, 30% mid, 20% final)
- Due dates per split with visual status:
  - Red: overdue
  - Amber: due within 3 days
  - Green: paid
- Record payment with optional proof attachment (screenshot, receipt)
- Auto-sync with pricing changes — no manual reconciliation needed

**Reconciliation Data Model:**
```json
{
  "trip_id": "uuid",
  "receivables": [
    {
      "label": "Main Package",
      "total": 1200,
      "splits": [
        { "label": "Advance", "amount": 600, "due_date": "2025-07-01", "status": "paid" },
        { "label": "Final Balance", "amount": 600, "due_date": "2025-08-01", "status": "pending" }
      ]
    }
  ],
  "payables": [
    {
      "label": "DMC Vietnam Payment",
      "total": 880,
      "splits": [...]
    }
  ]
}
```

---

## 4. Data Sources, APIs & Cost Analysis

This section covers every external data source the platform depends on, with selection rationale, reliability assessment, pricing, and recommended choices.

---

### 4.1 Flight Data

#### Requirements
- Search flights by origin, destination, date, and passenger count
- Fetch flight details by flight number
- Live pricing (not cached/estimated)
- Support for multi-city routing

#### Options Compared

| Provider | Type | Strengths | Weaknesses | Pricing |
|---|---|---|---|---|
| **Amadeus for Developers** | GDS aggregator | Industry standard, 400+ airlines, live fares | Complex API, rate limits on free tier | Free tier: 2000 calls/month. Paid: ~$0.006–$0.01/search |
| **Duffel** | Modern flights API | Clean REST API, booking support, NDC | Smaller airline coverage than Amadeus | Revenue-share model; no per-search fee if booking through them |
| **Skyscanner API** | Aggregator | Consumer brand trust, broad coverage | Primarily B2C, limited B2B API access now | Contact for commercial terms |
| **Kiwi.com / Tequila API** | Aggregator | Strong multi-city, low-cost carriers | Less structured for B2B | Free for search; booking fee applies |
| **FlightAware / AviationStack** | Flight status & schedules | Accurate schedules, flight number lookup | Not for fare search | Free: 100 calls/month; Pro: $9.99–$249/mo |

#### Recommendation

**Primary:** Amadeus for Developers (flight search + live pricing)  
**Secondary:** AviationStack or FlightAware (flight number lookup, schedule data)  
**Booking path:** Route to Duffel if live booking integration is required in a later phase

**Cost estimate (per trip generation):**
- 1–3 Amadeus flight search calls per trip = ~$0.02–$0.03/trip at scale

---

### 4.2 Hotel Data

#### Requirements
- Hotel search by city and dates
- Hotel details: name, description, images, star rating
- Pricing per night by category
- Support for hotel name fuzzy search

#### Options Compared

| Provider | Type | Strengths | Weaknesses | Pricing |
|---|---|---|---|---|
| **Amadeus Hotel Search API** | GDS | 150K+ hotels globally, live rates | Requires booking integration for live rates | Free tier + per-call beyond |
| **Booking.com Affiliate API** | OTA | Massive inventory, images, reviews | Must be affiliate partner; designed for booking | Revenue share model |
| **Expedia Rapid API** | OTA | 700K+ properties, rich content | Complex onboarding, designed for booking sites | Per-booking commission |
| **Google Places API** | General POI | Reliable details, images, reviews | Not travel-specific; no pricing | $17/1000 requests (Details API) |
| **Hotelbeds API** | B2B wholesale | True B2B pricing, no markup required | Requires registration/contract | Wholesale net rates |
| **RateHawk / Ostrovok B2B** | B2B wholesale | Good Asian coverage, agent-focused | Smaller global footprint | Net B2B rates |

#### Recommendation

| Use Case | Recommended Provider |
|---|---|
| Hotel details + images (general) | Google Places API + Unsplash |
| Live hotel pricing (document-first) | DMC document as primary pricing source |
| Hotel search suggestions | Amadeus Hotel Search API |
| Booking confirmation parsing | In-house OCR (GPT-4o Vision) |
| B2B wholesale pricing (future) | Hotelbeds or RateHawk |

**Key insight:** For the MVP, DMC documents are the authoritative pricing source. Hotel APIs serve primarily to enrich content (images, descriptions, star ratings) — not as the pricing source. This significantly reduces dependency on expensive OTA APIs in early phases.

---

### 4.3 Activities & Experiences

#### Requirements
- Browse activities by destination
- Check availability for specific dates
- Retrieve pricing, descriptions, images
- Support adding as main activity or optional add-on

#### Options Compared

| Provider | Strengths | Weaknesses | Pricing |
|---|---|---|---|
| **GetYourGuide Activities API** | 80,000+ experiences, trusted brand, availability check | Affiliate partnership required | Commission-based (20–30%) |
| **Viator (TripAdvisor) API** | Largest inventory globally (300K+ products) | Requires affiliate approval | Commission-based |
| **Musement API** | Strong European coverage | Limited global reach | Commission-based |
| **Klook API** | Excellent Asia-Pacific coverage | Less relevant for non-APAC trips | Commission-based |

#### Recommendation

**Primary:** GetYourGuide API (broad coverage, strong developer support)  
**Secondary:** Klook API for APAC-heavy destinations (Vietnam, Thailand, Japan, etc.)

Both are commission-based — no upfront API cost, but the platform earns or shares a commission on bookings. This is a potential revenue stream.

**For non-bookable activities** (agent-curated): free-form entry with Google Places enrichment.

---

### 4.4 Maps, Transit & Transfer Data

#### Requirements
- Intercity transfer time estimation
- Route calculation between cities
- Transport mode options (car, train, flight)
- Real travel duration data

#### Options Compared

| Provider | Strengths | Weaknesses | Pricing |
|---|---|---|---|
| **Google Maps Distance Matrix API** | Most accurate, covers 200+ countries | Cost at scale | $5/1000 elements |
| **Google Maps Directions API** | Route details, multi-modal | Cost at scale | $10/1000 requests |
| **HERE Routing API** | Competitive pricing, strong coverage | Less consumer trust than Google | Free: 250K transactions/mo; then $1–2/1000 |
| **Mapbox API** | Good for custom map rendering | Less strong for routing accuracy | Free: 100K requests/mo; $0.50/1000 after |
| **Rome2rio API** | Multi-modal travel options (plane, train, bus) | Data sometimes less precise | Free tier limited; contact for B2B |

#### Recommendation

**Primary:** Google Maps Distance Matrix + Directions API  
**Alternative at scale:** HERE Routing to reduce cost

**Cost estimate:** ~2–4 API calls per trip for intercity transfers = $0.01–$0.05/trip

---

### 4.5 Destination Content (Images)

#### Requirements
- High-quality destination images for export
- City-level, hotel-level, activity-level imagery
- Ability to search and select images

#### Options Compared

| Provider | Strengths | Weaknesses | Pricing |
|---|---|---|---|
| **Unsplash API** | Free, high quality, no attribution required | General imagery, not travel-specific | Free |
| **Pexels API** | Free, large library | Similar to Unsplash | Free |
| **Google Places Photos** | Specific to location | Resolution limits; per-request cost | $7/1000 photo requests |
| **Pixabay API** | Free, large library | Mixed quality | Free |

#### Recommendation

**Primary:** Unsplash API (free, high quality, clean license)  
**Fallback:** Pexels API  
**For hotel-specific images:** Booking.com affiliate content or agent-uploaded

---

### 4.6 Weather & Crowd Data (Trip Timing Intelligence)

#### Requirements
- Weather conditions at destination during scheduled trip dates
- Crowd levels (peak season warnings, event-based surges)
- Contextual advisories for trip planning

#### Options Compared

| Provider | Data Type | Strengths | Pricing |
|---|---|---|---|
| **OpenWeatherMap API** | Current + forecast + historical | 16-day forecast, 1000+ calls/day free | Free: 1000 calls/day; Paid from $40/mo |
| **WeatherAPI.com** | Forecast + historical | 14-day forecast, history API | Free: 1M calls/mo; from $4/mo |
| **Visual Crossing** | Historical + forecast | Strong historical data for planning | Free: 1000/day; from $35/mo |
| **Google Popular Times** | Crowd levels (via Places API) | Real crowd data | Bundled with Places API cost |
| **Teleport API** | City quality scores, cost of living | Good for destination overviews | Free |
| **Numbeo** | Cost of living, safety index | Useful context data | Free (scrape) / API on request |

#### Recommendation

| Use Case | Provider |
|---|---|
| Weather forecast for trip dates | WeatherAPI.com (best free tier for forecast range) |
| Historical weather for seasonal advisory | Visual Crossing |
| Crowd/popularity heuristics | OpenWeatherMap + curated internal dataset |
| Destination insights (cost, safety) | Teleport API |

---

### 4.7 Document Parsing

#### Requirements
- Parse PDF, DOCX, XLSX documents (DMC packages)
- Extract structured pricing, itinerary, hotels
- Handle inconsistent formats

#### Recommended Stack

| Tool | Purpose | Cost |
|---|---|---|
| **pypdf / pdfplumber** | PDF text extraction | Free (open source) |
| **python-docx** | DOCX extraction | Free (open source) |
| **openpyxl / pandas** | Excel parsing | Free (open source) |
| **GPT-4o (with vision)** | Parse complex/scanned PDFs, tables | ~$0.005–$0.015/document page |
| **AWS Textract** | OCR for scanned documents | $1.50/1000 pages (text); $15/1000 pages (tables) |
| **LlamaParse** | Specialized document parsing for LLMs | Free: 1000 pages/day; $0.003/page after |

#### Recommendation

- **Standard PDFs:** pypdf + pdfplumber (free, fast)
- **Complex tables / scanned docs:** LlamaParse (best accuracy for LLM ingestion)
- **Fallback:** GPT-4o Vision for unstructured or image-heavy documents

---

### 4.8 Voice Transcription

| Provider | Accuracy | Languages | Pricing |
|---|---|---|---|
| **OpenAI Whisper API** | Excellent | 50+ | $0.006/min |
| **Web Speech API (Browser)** | Good | ~30 | Free (browser-native) |
| **Deepgram** | Excellent | 30+ | $0.0043/min (Nova model) |
| **Google Cloud Speech-to-Text** | Excellent | 125+ | Free: 60 min/mo; $0.009/min after |

#### Recommendation

**MVP:** Web Speech API (zero cost, no backend needed)  
**Production:** OpenAI Whisper or Deepgram for accuracy and multi-language support

---

### 4.9 API Cost Summary Table

| Data Domain | Recommended Provider | Est. Cost/Trip | Monthly (500 trips) |
|---|---|---|---|
| Flight search | Amadeus API | ~$0.03 | ~$15 |
| Flight number lookup | AviationStack | ~$0.01 | ~$5 |
| Hotel details/images | Google Places + Unsplash | ~$0.04 | ~$20 |
| Activities | GetYourGuide API | $0 (commission) | $0 |
| Maps / transit | Google Distance Matrix | ~$0.02 | ~$10 |
| Destination images | Unsplash + Pexels | $0 | $0 |
| Weather | WeatherAPI.com | ~$0.005 | ~$2.50 |
| Document parsing | LlamaParse + pypdf | ~$0.05 | ~$25 |
| Voice input | Web Speech API (MVP) | $0 | $0 |
| LLM (OpenAI GPT-4o) | OpenAI | ~$0.15–$0.30 | ~$75–$150 |
| **Total per trip (approx.)** | | **~$0.30–$0.45** | **~$150–$225** |

> Note: These are conservative estimates for an MVP. Costs decrease significantly with caching, batching, and model optimization at scale. LLM cost is the dominant factor and can be reduced by using GPT-4o-mini for extraction tasks and reserving GPT-4o for planning/generation.

---

## 5. AI Agent Design & LLM Integration

### 5.1 The AI Agent Model

The platform's AI is not a single monolithic prompt — it is an **orchestrated multi-step agent** that routes user intent to the appropriate tool or sub-pipeline. The agent can:

- Extract structured data from natural language
- Retrieve relevant document chunks via RAG
- Call external APIs and synthesize results
- Generate human-quality travel narrative
- Apply natural-language edits to structured trip data

### 5.2 Agent Architecture

```
User Input (text / voice)
        │
        ▼
┌──────────────────────────┐
│  Intent Classifier       │  ← "What does the user want to do?"
│  (LLM / rules hybrid)    │
└──────────┬───────────────┘
           │
     ┌─────┴───────────────────────────────────────────┐
     │                  INTENT ROUTER                  │
     └──┬──────────┬──────────┬──────────┬─────────────┘
        │          │          │          │
        ▼          ▼          ▼          ▼
  [Extract     [Plan      [Edit      [Generate
   Trip Info]   Trip]      Trip]      Export]
        │          │          │          │
        ▼          ▼          ▼          ▼
  [Extraction  [Planning  [Edit      [Export
   Pipeline]   Pipeline]  Pipeline]  Pipeline]
```

### 5.3 Core AI Pipelines

#### Pipeline 1: Trip Extraction
```
Input: Freeform text/voice
→ GPT-4o (structured output mode)
→ Pydantic validation
→ Populate trip form
```

#### Pipeline 2: Document RAG (Planning from DMC doc)
```
Document chunks (vectorized)
→ User query + skeleton structure
→ Semantic retrieval (top-k chunks)
→ GPT-4o with retrieved context
→ Normalized trip structure
```

#### Pipeline 3: Full Trip Generation
```
Skeleton + preferences + fetched data (flights, hotels, weather)
→ System prompt: "You are a senior travel planner..."
→ GPT-4o with function calling
→ Structured day-by-day plan with timing
→ Pricing computation
```

#### Pipeline 4: Natural Language Trip Editing
```
User edit request: "Add one more night in Halong Bay and change flight to train"
→ Intent: structural_edit
→ Current trip JSON as context
→ GPT-4o: "Return the updated trip JSON with this change applied"
→ Diff validator → apply to DB
→ Trigger sync to pricing + day plans
```

#### Pipeline 5: Dynamic Page Generation (Export)
```
User describes page: "A sustainability-themed page about Vietnam"
→ GPT-4o: generate headlines, body text, layout suggestions
→ Unsplash API: fetch matching images
→ Render into template
```

### 5.4 LLM Selection Strategy

| Task | Recommended Model | Reason |
|---|---|---|
| Trip info extraction | GPT-4o-mini | Fast, cheap, structured output |
| Document parsing | GPT-4o | Complex table/PDF understanding |
| Full trip generation | GPT-4o | Highest quality narrative |
| Day plan editing | GPT-4o-mini | Simple structured edits |
| Trip structure editing | GPT-4o | Multi-dependency reasoning |
| Dynamic page content | GPT-4o | Creative, styled copy |
| Export text editing | GPT-4o-mini | Fast inline edits |

### 5.5 Prompt Engineering Principles

- **System prompts are role-specific:** "You are a senior luxury travel planner with 15 years of experience in Southeast Asia..."
- **Output format is always specified:** JSON schema, markdown, or structured prose — never ambiguous
- **Context window management:** For large trips, use chunked context + RAG rather than stuffing full trip JSON
- **Temperature settings:** Low (0.2–0.4) for extraction/editing; Medium (0.6–0.8) for generation/creativity
- **Fallback handling:** If LLM output fails schema validation, retry once with stricter prompt before surfacing error to agent

### 5.6 RAG Design (Document-Based Planning)

```
Indexing Time:
  Document → Chunks (512 tokens, 64 overlap)
           → Embeddings (text-embedding-3-small)
           → pgvector index

Query Time:
  User query + destination context
           → Query embedding
           → Cosine similarity search (top 5 chunks)
           → Rerank (if needed, via Cohere Rerank)
           → Inject into LLM context
```

- Embedding model: `text-embedding-3-small` ($0.02/1M tokens — effectively near-free)
- Chunk size: 512 tokens with 64 token overlap
- Top-k retrieval: 5–8 chunks per query
- Reranking: optional (Cohere Rerank API at $1/1000 requests for higher accuracy)

---

## 6. PDF & Export Engine Design

### 6.1 Export Types

| Type | Description | Format |
|---|---|---|
| Interactive Web Link | Hosted URL with animations, dynamic sections | HTML/CSS/JS (React SSR or static) |
| PDF Export | Downloadable formatted document | PDF (headless browser rendered) |

### 6.2 Export Architecture

```
Trip Data (structured)
        │
        ▼
┌──────────────────────┐
│  Export Configurator │  ← agent sets layout, colors, sections
└──────────┬───────────┘
           │
     ┌─────┴──────────────────────────────┐
     ▼                                    ▼
[Web Link Renderer]               [PDF Renderer]
   React + Tailwind                 Puppeteer / WeasyPrint
   Next.js SSG/SSR                  Headless Chrome
   Hosted on CDN                    S3 storage
           │                                    │
           ▼                                    ▼
   Shareable URL                     Downloadable PDF
```

### 6.3 Section System (Modular)

Each export is composed of independently renderable sections:

| Section | Content | Configurable |
|---|---|---|
| Cover Page | Agency logo, client name, trip name, destination imagery | Layout style (5+ options) |
| Trip Summary | Destinations, dates, pax, highlights | Show/hide |
| Hotels | Per-city hotel with images, description, star rating | Layout style, images |
| Day-by-Day Itinerary | Full plan per day with activities, transfers | Layout style |
| Pricing | Package breakdown, taxes, add-ons, optional upgrades | Show/hide line items |
| Inclusions / Exclusions | Bulleted list | Editable text |
| Dynamic Pages | AI-generated custom thematic pages | Full customization |
| Contact / Footer | Agent details, WhatsApp CTA | Auto-populated from profile |

### 6.4 Branding System

- Agency logo embedded in all exports
- Brand color palette (primary, secondary, accent) applied globally
- AI-suggested brand colors derived from logo (via color extraction library, e.g., `colorthief`)
- Typography: curated travel-appropriate font pairings (Google Fonts)
- All brand settings stored per agency, applied at render time

### 6.5 Cover Page Styles

| Style | Description |
|---|---|
| Full-bleed | Single large image spanning the full cover |
| Cinematic | Dark overlay with centered typographic layout |
| Editorial | Split-panel with image and text side by side |
| Watermark | Logo/brand as large watermark over destination photo |
| Mosaic | Grid of destination images with title |

### 6.6 Curtain Raiser Feature

- Enabled per export
- Client receives a URL that shows only the cover page initially
- A prominent "Enter Your Trip" CTA reveals the full itinerary
- Creates a higher-engagement, premium client experience
- Implemented as a single-page app with a gated reveal animation (CSS + Framer Motion)

### 6.7 Dynamic Pages

- Agent describes the page in natural language
- GPT-4o generates: headlines, body copy, layout blocks
- Unsplash API fetches matching images
- Pages rendered using the same template system as standard sections
- Supports: Bento grid, editorial, mosaic, multi-block summary layouts

### 6.8 PDF Generation

**Recommended stack:** Puppeteer (headless Chrome)

**Process:**
1. Render export as HTML page (same as web link)
2. Puppeteer navigates to the rendered page
3. `page.pdf()` with `format: A4`, `printBackground: true`
4. PDF stored in S3 with signed URL returned to agent
5. PDF cache invalidated when trip or export config changes

**Alternative:** WeasyPrint (Python) for simpler documents without complex CSS animations

---

## 7. Infrastructure, Scalability & Cost Estimates

### 7.1 Tech Stack Summary

| Layer | Technology | Notes |
|---|---|---|
| Frontend | Next.js 14 + Tailwind CSS | App Router, React Server Components |
| Backend API | Node.js (Express / Fastify) | REST + WebSocket |
| AI/ML Service | Python (FastAPI) | LLM calls, document parsing, embeddings |
| Database | PostgreSQL (via Supabase or RDS) | Primary relational store |
| Vector DB | pgvector extension on Postgres | Avoid separate vector DB at MVP |
| Cache | Redis (Upstash or ElastiCache) | Sessions, rate limiting, API cache |
| Object Storage | AWS S3 or Cloudflare R2 | Documents, images, PDFs |
| CDN | Cloudflare | Static assets + export links |
| Auth | Supabase Auth or NextAuth.js | Google + email/password |
| Email | Resend or AWS SES | Notifications, invitations |
| Payments | Stripe | Subscription billing for agencies |
| Monitoring | Sentry + Datadog / Grafana | Errors + infra metrics |
| CI/CD | GitHub Actions + Docker | Automated build, test, deploy |

### 7.2 Infrastructure Architecture (MVP — Lean)

**Recommended MVP stack (low cost, high velocity):**

| Service | Provider | Monthly Cost (est.) |
|---|---|---|
| App hosting (2 services) | Railway / Render / Fly.io | $20–$50 |
| PostgreSQL + pgvector | Supabase (Pro) | $25 |
| Redis | Upstash | $0–$10 |
| Object Storage | Cloudflare R2 | $0–$5 (first 10GB free) |
| CDN | Cloudflare Free | $0 |
| Auth | Supabase Auth | Included |
| Email | Resend (3000 emails/mo free) | $0 |
| LLM (OpenAI) | OpenAI API | $75–$150 (500 trips/mo) |
| External APIs | Amadeus + Google + WeatherAPI | ~$50 |
| **Total MVP infra/API (500 trips/mo)** | | **~$175–$290/mo** |

### 7.3 Scale-Up Architecture (Post-PMF)

Once product-market fit is established (2,000+ trips/month):

| Upgrade | Reason |
|---|---|
| Migrate to AWS ECS / GKE | Finer control, autoscaling |
| RDS Multi-AZ Postgres | HA for production |
| ElastiCache Redis | Managed, clustered |
| Kafka (MSK) | Replace Redis Pub/Sub for reliable event streaming |
| Worker queue (BullMQ / SQS) | Async document processing, PDF generation |
| CDN-hosted export links | Sub-second load anywhere globally |
| LLM cost optimization | GPT-4o-mini for simple tasks; cache common prompts |

**Estimated monthly cost at 2,000 trips/month:** ~$800–$1,500 (infra + APIs + LLM)  
**Cost per trip at scale:** ~$0.40–$0.75

### 7.4 Caching Strategy

| Cache Target | TTL | Impact |
|---|---|---|
| Flight search results | 15 min | Avoid redundant Amadeus calls |
| Hotel details + images | 24 hours | Google Places cost reduction |
| Weather data | 6 hours | WeatherAPI call reduction |
| Generated day plans | Until trip edit | Avoid regeneration on page reload |
| PDF exports | Until trip/export config changes | S3 cached PDF served directly |
| Document embeddings | Permanent (until doc update) | No re-embedding cost |

---

## 8. Security, Compliance & Data Privacy

### 8.1 Authentication & Authorization

- All API endpoints protected by JWT bearer token validation at API gateway
- Role-based access control (RBAC): Admin vs. Agent permissions enforced server-side
- Agency data isolation: all queries scoped by `agency_id` — no cross-agency data leakage
- Google OAuth via PKCE flow
- Refresh token rotation on every use
- Short-lived access tokens (15 min) + long-lived refresh tokens (30 days)

### 8.2 Data Encryption

| Layer | Encryption |
|---|---|
| Data at rest | AES-256 (S3 + RDS default encryption) |
| Data in transit | TLS 1.3 (all HTTP connections) |
| Sensitive fields | Encrypted at application layer (e.g., payment details) |
| Environment secrets | AWS Secrets Manager / Doppler |

### 8.3 Client Data Handling

- Client PII (name, email, phone) stored only within agency scope
- No client data shared with third-party analytics
- GDPR-compliant data deletion: full trip + client data purge on agency account termination
- DMC documents treated as agency-confidential; never shared across agencies or used for training

### 8.4 API Key Management

- All third-party API keys (OpenAI, Amadeus, Google, etc.) stored in secrets manager
- Keys never exposed to frontend
- Per-agency API key isolation for future white-label support
- Rate limiting per agency to prevent abuse

### 8.5 Export Link Security

- Interactive export links use signed, time-limited URLs
- Optional: password-protect specific export links
- Link access logs maintained (view count, last accessed)
- Agency can revoke/expire any link at any time

### 8.6 Compliance Considerations

| Area | Approach |
|---|---|
| GDPR (EU clients) | Data processing agreement, right to erasure, data portability |
| PCI-DSS | No raw card data stored; delegate entirely to Stripe |
| SOC 2 (future) | Audit logging, access controls, incident response plan |
| Travel regulations | No ticket booking = lower regulatory burden at MVP |

---

## Appendix A: Recommended MVP Build Order

| Phase | Features | Timeline (est.) |
|---|---|---|
| **Phase 1** | Auth, Agency setup, Trip CRUD, Voice/text extraction | Weeks 1–4 |
| **Phase 2** | Document Hub, DMC parser, Skeleton planner | Weeks 4–8 |
| **Phase 3** | Full trip generation (flights + hotels + days + pricing) | Weeks 8–14 |
| **Phase 4** | Editing engine (all components), Versioning | Weeks 12–18 |
| **Phase 5** | Export engine (PDF + web link + branding) | Weeks 16–22 |
| **Phase 6** | Reconciliation, payment tracking | Weeks 20–24 |
| **Phase 7** | Dynamic pages, Curtain Raiser, advanced export styles | Weeks 22–28 |

---

## Appendix B: Key Design Decisions & Trade-offs

| Decision | Choice | Alternative Considered | Reason |
|---|---|---|---|
| DMC as pricing source (MVP) | Yes | Live hotel APIs | Faster to build; agents already work with DMC docs |
| Vector DB | pgvector | Pinecone | Avoid separate service cost at MVP |
| LLM Provider | OpenAI GPT-4o | Anthropic Claude / Gemini | Best function calling, JSON mode, vision |
| PDF rendering | Puppeteer | WeasyPrint / PDFKit | Best fidelity for CSS-heavy layouts |
| Frontend | Next.js | React SPA | SSR needed for export link rendering |
| Auth | Supabase Auth | Auth0 | Cost + Postgres integration |
| Booking | No live booking (MVP) | Amadeus booking API | Complexity + regulatory; defer to Phase 2+ |

---

## Appendix C: Glossary

| Term | Definition |
|---|---|
| DMC | Destination Management Company — a local operator that provides ground services and packages |
| Skeleton Proposal | Lightweight trip structure (cities + nights) before full generation |
| Full Generation | Complete AI-generated trip with flights, hotels, day plans, and pricing |
| Trip Version | An immutable snapshot of a trip state, enabling rollback |
| RAG | Retrieval-Augmented Generation — LLM generation grounded in retrieved document chunks |
| Reconciliation | Financial tracking of receivables (from client) and payables (to vendors) |
| Dynamic Page | AI-generated custom export section with layout, copy, and images |
| Curtain Raiser | Cover-page-first reveal experience for interactive export links |

---

*Document prepared for engineering and product planning purposes. All cost estimates are indicative and subject to actual usage patterns and provider pricing changes.*
