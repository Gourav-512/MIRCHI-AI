# CAREOON — Final Architecture & Implementation Plan

**Version:** 2.1 (Approved Base + 6 Strategic Corrections Applied)

---


## 🏛️ Careoon Engineering Law

> **Build the core once.**  
> Configure cities, services, providers, and business rules without rewriting the core.

**Expansion Path:** Sangli → Miraj → Kolhapur → Pune → Maharashtra → India

*The foundation does not get rewritten. Business rules get configured.*

---

## A. FINAL ARCHITECTURE

```
                           CAREOON
                              │
           ┌──────────────────┼──────────────────┐
           ▼                  ▼                  ▼
       Customer           Provider            Admin
    (Web + Flutter)   (Web + Flutter)     (Web Only)
           │                 │                 │
           └──────────────────┼─────────────────┘
                              ▼
                ┌─── APPLICATION LAYER ───┐
                │  Next.js API Routes     │
                │  Auth Guards · Zod      │
                │  Request IDs · Logging  │
                └────────────┬────────────┘
                             │
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
      MATCHING            BOOKING            PAYMENT
       ENGINE             ENGINE              ENGINE
    (Eligibility      (State Machine      (Razorpay
     + Ranking)       + Transitions)      Abstraction)
          │                 │                  │
          └──────────────────┼──────────────────┘
                             ▼
                ┌─── DOMAIN EVENTS ────────┐
                │  DomainEvent published   │
                │  → domain_events table   │
                │  → Outbox pattern (V1)   │
                │  → Async worker (Prod)   │
                └────────────┬─────────────┘
                             │
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
      Matching          Notification         Audit
      Worker             Worker              Worker
                             ▼
              ┌──── REPOSITORY LAYER ────────┐
              │  All Supabase calls here     │
              │  No SQL in UI / Services     │
              └────────────┬─────────────────┘
                           ▼
          ┌──── INFRASTRUCTURE ───────────┐
          │  Supabase (PG + Auth + RT)   │
          │  Cloudflare R2 (Storage)     │
          │  Razorpay (Payments)         │
          │  Firebase FCM (Push)         │
          │  Google Maps (Location)      │
          │  Resend (Email)              │
          └──────────────────────────────┘
```

---

## B. IMPLEMENTATION PLAN (6 CORRECTIONS APPLIED)

### CORRECTION 1 — Matching Engine: Full Deterministic Ranking Pipeline

**V1 is NOT "simple score." V1 is a complete deterministic ranking engine.**

ML signals are addable later without changing the engine interface.

#### Matching Pipeline (V1 — Deterministic)

```
Customer Request
      │
      ▼
┌──────────────────────────────────────┐
│  STEP 1: HARD ELIGIBILITY FILTER     │
│  (binary pass/fail — must clear all) │
│                                      │
│  ✓ provider.verification_status=ACTIVE│
│  ✓ has capability for service        │
│  ✓ has skills for requirement        │
│  ✓ covers customer's city/area/radius│
│  ✓ available on requested day        │
│  ✓ time window overlaps              │
│  ✓ no confirmed booking conflict     │
└─────────────┬──────────────────────┘
              │ eligible_providers[]
              ▼
┌──────────────────────────────────────┐
│  STEP 2: DISTANCE SCORING            │
│                                      │
│  distance_km = haversine(            │
│    customer.lat, customer.lng,       │
│    provider.base_lat, provider.base_lng│
│  )                                   │
│                                      │
│  distance_score = max(0,             │
│    100-(distance_km/max_radius)*100  │
│  )                                   │
└─────────────┬──────────────────────┘
              ▼
┌──────────────────────────────────────┐
│  STEP 3: RATING SCORE                │
│                                      │
│  rating_score = (avg_rating/5)*100   │
│  (0 if < MIN_REVIEWS_THRESHOLD)      │
│  (50 for new providers — neutral)    │
└─────────────┬──────────────────────┘
              ▼
┌──────────────────────────────────────┐
│  STEP 4: RELIABILITY SCORE           │
│                                      │
│  reliability_score =                 │
│    (acceptance_rate*W_acceptance) +  │
│    (response_rate*W_response) +      │
│    (completion_rate*W_completion)    │
└─────────────┬──────────────────────┘
              ▼
┌──────────────────────────────────────┐
│  STEP 5: EXPERIENCE SCORE            │
│                                      │
│  experience_score = min(100,         │
│    (experience_years/MAX_YEARS)*100) │
└─────────────┬──────────────────────┘
              ▼
┌──────────────────────────────────────┐
│  STEP 6: COMPOSITE WEIGHTED SCORE    │
│                                      │
│  final_score =                       │
│    distance_score   × W_distance  +  │
│    rating_score     × W_rating    +  │
│    reliability_score× W_reliability+ │
│    experience_score × W_experience   │
│                                      │
│  Weights (V1 defaults — configurable):│
│    W_distance    = 0.35              │
│    W_rating      = 0.30              │
│    W_reliability = 0.25              │
│    W_experience  = 0.10              │
└─────────────┬──────────────────────┘
              ▼
┌──────────────────────────────────────┐
│  STEP 7: RANKED RESULTS              │
│  sorted by final_score DESC          │
│  + factors[] returned for debugging  │
└──────────────────────────────────────┘
```

#### TypeScript Interface (Matching Engine)

```typescript
// /features/matching/engine.ts

interface MatchingCriteria {
  serviceId: string
  requirementIds: string[]
  cityId: string
  areaId?: string
  customerLat: number
  customerLng: number
  requestedStartTime: Date    // CORRECTION 3 — full time window
  requestedEndTime: Date
  durationMinutes: number
}

interface MatchScoreFactor {
  name: string        // 'distance' | 'rating' | 'reliability' | 'experience'
  rawValue: number
  score: number
  weight: number
  weightedScore: number
}

interface ProviderMatch {
  providerId: string
  providerName: string
  distanceKm: number
  finalScore: number
  factors: MatchScoreFactor[]   // full transparency for debugging
  estimatedArrivalMinutes?: number
}

interface MatchingEngine {
  match(criteria: MatchingCriteria): Promise<ProviderMatch[]>
}
```

#### Scoring Weights — Configurable (NOT hardcoded)

```typescript
// /config/matching.config.ts
// V1: static config file — future: DB table matching_configurations

export const MATCHING_WEIGHTS = {
  distance: 0.35,
  rating: 0.30,
  reliability: 0.25,
  experience: 0.10,
} as const

export const MATCHING_CONFIG = {
  maxRadiusKm: 15,
  maxExperienceYears: 10,
  minReviewsForRating: 3,
  newProviderRatingScore: 50,   // neutral score for unreviewed providers
  offerExpiryMinutes: 10,       // how long provider has to accept
} as const
```

**Future Enhancement:** ML signals (acceptance prediction, time-of-day demand, etc.) plug into Step 6 as additional weighted factors — engine interface unchanged.

---

### CORRECTION 2 — Geo/Location: Working in V1 (Not "Planned")

**Core Idea:** "My live location → nearby eligible nurses → ranking"

GPS polish can wait. Distance calculation cannot.

#### ✅ What IS in V1 (Working)

| Capability | V1 Status |
|---|---|
| Browser Geolocation API → lat/lng | ✅ Working |
| Manual address entry + city/area selection | ✅ Working |
| Haversine distance calculation (server-side) | ✅ Working |
| Distance-based ranking in Matching Engine | ✅ Working |
| Customer saved addresses (lat/lng stored) | ✅ Working |
| Provider base location (lat/lng stored) | ✅ Working |
| Provider service radius filter | ✅ Working |
| City-level service area filter | ✅ Working |

#### 🔜 What is PLANNED (clearly marked)

| Capability | Status |
|---|---|
| Google Maps autocomplete (address search) | [PLANNED] |
| Live GPS provider tracking during service | [PLANNED] |
| Real-time map showing provider location | [PLANNED] |
| Advanced PostGIS geofencing | [PLANNED] |
| Google Maps Directions API (ETA) | [PLANNED] |

#### Haversine (Server-Side — No Google Maps Required)

```typescript
// /utils/geolocation.ts

export function haversineDistance(
  lat1: number, lng1: number,
  lat2: number, lng2: number
): number {
  const R = 6371 // Earth radius km
  const dLat = toRad(lat2 - lat1)
  const dLng = toRad(lng2 - lng1)
  const a =
    Math.sin(dLat / 2) ** 2 +
    Math.cos(toRad(lat1)) *
    Math.cos(toRad(lat2)) *
    Math.sin(dLng / 2) ** 2
  return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a))
}
```

**Note:** Google Maps API Key only required for address autocomplete (Planned). Distance matching works immediately without it.

---

### CORRECTION 3 — Availability Engine: Time-Window + Conflict Rules

**Real Problem:** "Nurse ला उद्या 10 वाजता यायला किती वेळ लागेल?" — This is the actual marketplace question.

#### Booking Time Model (Corrected)

```typescript
// bookings table — corrected fields:
interface BookingTimeWindow {
  requested_start_time: Date     // customer's requested start
  requested_end_time: Date       // calculated: start + duration
  duration_minutes: number       // service duration
  provider_travel_buffer_minutes: number  // estimated travel time
  provider_effective_start: Date // when provider needs to leave
  // = requested_start_time - travel_buffer
}
```

#### Availability Engine Rules (Explicit)

**Rule 1 — Weekly Schedule Filter:**
```
provider_availability.day_of_week must include booking day
provider_availability.start_time <= requested_start_time
provider_availability.end_time   >= requested_end_time
```

**Rule 2 — Date Override Filter:**
```
IF provider_availability_overrides EXISTS for booking date:
  AND is_available = false → HARD EXCLUDE
```

**Rule 3 — Booking Conflict Filter:**
```
No existing booking for provider where:
  status IN (ACCEPTED, ON_THE_WAY, ARRIVED, SERVICE_STARTED)
  AND booking time window overlaps requested window
  (with travel buffer on both sides)
```

**Rule 4 — Travel Buffer:**
```
provider_effective_start = requested_start_time - estimated_travel_minutes
Must not conflict with provider's prior booking end time
```

**Rule 5 — Offer Window:**
```
A provider in status PROVIDER_OFFERED for another booking
for the same time slot → EXCLUDE (prevent double-offers)
```

#### Database: Availability Conflict Check (SQL)

```sql
-- Check if provider has conflicting confirmed booking
SELECT COUNT(*) FROM bookings
WHERE provider_id = $provider_id
  AND status IN ('ACCEPTED', 'ON_THE_WAY', 'ARRIVED', 'SERVICE_STARTED')
  AND requested_start_time < $requested_end_time
  AND requested_end_time   > $requested_start_time
```

#### Updated bookings Table Fields (Delta)

```sql
-- Additional fields added to bookings table:
requested_start_time    TIMESTAMPTZ NOT NULL
requested_end_time      TIMESTAMPTZ NOT NULL   -- computed: start + duration
duration_minutes        INTEGER NOT NULL
travel_buffer_minutes   INTEGER DEFAULT 0
provider_effective_start TIMESTAMPTZ           -- when provider leaves
scheduled_at            TIMESTAMPTZ            -- kept as canonical field
```

---

### CORRECTION 4 — Multi-City: Top-Level Core Requirement

**Multi-city is NOT a feature. It IS a core architectural requirement.**

#### City Lifecycle (Admin-Controlled)

```
CONFIGURED
    ↓
SERVICES_ENABLED
    ↓
PROVIDERS_ONBOARDING
    ↓
SOFT_LAUNCH
    ↓
ACTIVE
    ↓
PAUSED
    ↓
INACTIVE
```

| Stage | Description |
|---|---|
| **CONFIGURED** | Admin adds city, areas, services |
| **SERVICES_ENABLED** | Pricing + commission configured |
| **PROVIDERS_ONBOARDING** | Providers being verified |
| **SOFT_LAUNCH** | Limited availability, internal testing |
| **ACTIVE** | Public — customers can book |
| **PAUSED** | Temporary — no new bookings |
| **INACTIVE** | Operations wound down |

#### Cities Table — Updated

```sql
cities:
  id UUID PK
  name TEXT NOT NULL
  slug TEXT UNIQUE NOT NULL
  state TEXT NOT NULL
  country TEXT DEFAULT 'IN'
  center_latitude NUMERIC(10,7)
  center_longitude NUMERIC(10,7)
  status city_status ENUM (
    'CONFIGURED' | 'SERVICES_ENABLED' | 'PROVIDERS_ONBOARDING' |
    'SOFT_LAUNCH' | 'ACTIVE' | 'PAUSED' | 'INACTIVE'
  )
  launch_date DATE
  config JSONB   -- city-specific overrides (future use)
  created_at, updated_at
```

#### Admin City Activation Workflow

```
Admin: Add City (name, state, coordinates)
  → status: CONFIGURED
  
Admin: Add Areas (pincodes, coordinates)

Admin: Enable Services (link pricing_configurations)
  → status: SERVICES_ENABLED
  
Admin: Onboard Providers (set provider_service_areas for this city)
  → status: PROVIDERS_ONBOARDING
  
Admin: Activate City
  → status: ACTIVE
  → CITY_ACTIVATED domain event fired
  → customers in this city can now book
```

#### Hard Rules (Enforcement Required)

```typescript
// These checks must be enforced in MatchingService and BookingService:

// 1. Only book in ACTIVE cities
if (city.status !== 'ACTIVE') throw new AppError('CITY_NOT_ACTIVE')

// 2. Only match providers whose service area covers customer city
// → provider_service_areas.city_id = booking.city_id

// 3. Pricing resolved per city (most specific wins)
// pricing_configurations: service+city > service > default
```

---

### CORRECTION 5 — Payment Boundary: Clearly Defined Stages

**Revenue Model:** Customer Payment → Commission → Provider Settlement

This must work before real customer launch. Payment mock is ONLY for prototype.

#### Three Clear Stages

##### STAGE 1: PROTOTYPE (V1 2-day build)

```
Payment flow: [MOCK]
- No real Razorpay order created
- payment_status set to CAPTURED in DB directly
- Clearly marked with [MOCK] in code + UI banner
- Commission calculated (DB-driven) — real logic, mock money
- Settlement record created — real logic, mock settlement
- Booking completes fully through state machine
- Everything else is production-ready
```

##### STAGE 2: PILOT (Before first real customer)

```
Payment flow: [REAL — Razorpay Test Mode]
- Real Razorpay test orders
- Real webhook verification
- idempotency_key enforced
- Payment confirmed only after webhook
- Commission deducted — real calculations
- Settlement records accurate
- Provider earnings visible
- No actual money moved yet (test mode)
```

##### STAGE 3: PRODUCTION (Live operations)

```
Payment flow: [REAL — Razorpay Live Mode]
- All Stage 2 + real INR transactions
- Settlement payouts via Razorpay Route / bank transfer
- Refund flow complete
- Cancellation charge rules enforced
- Financial reconciliation audit log
- Fully auditable trail from booking → payment → settlement
```

#### Code Boundary Enforcement

```typescript
// /features/payments/payment.interface.ts
interface PaymentProvider {
  createOrder(params: CreateOrderParams): Promise<GatewayOrder>
  verifyPayment(params: VerifyParams): Promise<VerificationResult>
  initiateRefund(params: RefundParams): Promise<RefundResult>
  handleWebhook(payload: unknown, sig: string): Promise<WebhookEvent>
}

// /features/payments/providers/mock.provider.ts
// [MOCK] — clearly annotated
export class MockPaymentProvider implements PaymentProvider {
  // Simulates all states. Never touches real gateway.
  // UI shows: "⚠️ Test Mode — No real payment processed"
}

// /features/payments/providers/razorpay.provider.ts
// [REAL] — production implementation
export class RazorpayProvider implements PaymentProvider { ... }

// Factory — controlled by environment:
// APP_ENV=development → MockPaymentProvider
// APP_ENV=staging     → RazorpayProvider (test keys)
// APP_ENV=production  → RazorpayProvider (live keys)
```

---

### CORRECTION 6 — Domain Events: Production Outbox/Async Path

**V1 synchronous is acceptable.** But production path must be designed from Day 1.

#### V1 (Synchronous + Outbox Table)

```
API Request
    ↓
Domain Service (e.g. BookingService.create())
    ↓
Database transaction:
  1. INSERT bookings row
  2. INSERT domain_events row (processed_at = NULL)
  3. Commit
    ↓
Same request — EventProcessor.processSync():
  - Reads unprocessed events from domain_events
  - Runs handlers (MatchingHandler, NotificationHandler, AuditHandler)
  - Marks event processed_at = NOW()
    ↓
API Response
```

#### Production Path (Async — Outbox → Worker)

```
API Request
    ↓
Domain Service
    ↓
Database transaction:
  1. INSERT bookings row
  2. INSERT domain_events row (processed_at = NULL)
  3. Commit
    ↓
API Response (FAST — no downstream processing in request)

[SEPARATE] — Outbox Worker (Vercel Cron / dedicated process):
  - Poll domain_events WHERE processed_at IS NULL
  - Process each event through handlers
  - On success: mark processed_at = NOW()
  - On failure: increment retry_count, set failed_at
  - After max retries: alert + dead-letter handling
```

#### Switching from Sync → Async: ONE Change

```typescript
// /lib/events/event-bus.interface.ts
interface EventBus {
  publish(event: DomainEvent): Promise<void>
}

// V1: SyncEventBus — processes in same request
// Production: OutboxWorkerEventBus — DB insert only, worker processes

// Domain services call EventBus.publish() — they never know which.
// Switching is purely infrastructure configuration.
```

#### domain_events Table — Outbox Pattern

```sql
domain_events:
  id               UUID PK
  event_type       TEXT NOT NULL          -- e.g. 'BOOKING_CREATED'
  event_version    TEXT DEFAULT '1.0'
  aggregate_type   TEXT NOT NULL          -- e.g. 'Booking'
  aggregate_id     UUID NOT NULL
  payload          JSONB NOT NULL
  correlation_id   UUID NOT NULL
  causation_id     UUID                   -- what triggered this
  actor_id         UUID FK → profiles
  
  -- Outbox processing fields:
  processed_at     TIMESTAMPTZ            -- NULL = pending
  failed_at        TIMESTAMPTZ
  failure_reason   TEXT
  retry_count      INTEGER DEFAULT 0
  max_retries      INTEGER DEFAULT 3
  
  created_at       TIMESTAMPTZ DEFAULT NOW()

-- Index for outbox polling:
CREATE INDEX idx_domain_events_unprocessed
  ON domain_events(created_at)
  WHERE processed_at IS NULL AND failed_at IS NULL;
```

---

## C. DEPENDENCY GRAPH (Updated)

### Phase 1: FOUNDATION
- Next.js + TypeScript + Tailwind + shadcn/ui
- Supabase clients (browser + server + admin)
- All 28 tables migrated
- Type system + constants + error handling
- Auth (register + login + middleware)
- Repository layer (all — stubbed)
- Domain service layer (all — stubbed)
- EventBus abstraction (SyncEventBus V1)
- StorageProvider abstraction (R2)
- PaymentProvider abstraction (Mock V1)
- Careoon Design System (all components)

**Required by:** ALL subsequent phases

### Phase 2: GEOGRAPHY + SERVICE CATALOGUE
- Cities + Areas + Addresses
- Service Categories + Services + Requirements + Skills
- Pricing Configurations
- Commission Configurations
- Provider profiles + skills + capabilities + availability
- Multi-city admin activation workflow

**Required by:** Phases 3, 4, 5

### Phase 3: MATCHING ENGINE
- Eligibility filter (all 5 hard rules)
- Availability Engine (time-window + conflict)
- Distance scoring (Haversine — no Google Maps needed)
- Deterministic weighted ranking
- Provider Assignments table + offer flow

**Required by:** Phase 4

### Phase 4: BOOKING ENGINE + V1 VERTICAL SLICE
- Full 12-state booking state machine
- Provider offer → accept/reject/timeout
- Reassignment flow
- Booking status history (every transition logged)
- Audit trail for booking events

**Required by:** Phases 5, 6

### Phase 5: PAYMENTS + COMMISSION + SETTLEMENT
- PricingService (configurable breakdown)
- Mock payment flow (V1 prototype)
- Razorpay integration (Pilot)
- Webhook verification
- idempotency enforcement
- Commission calculation
- Settlement records

**Required by:** Phase 7

### Phase 6: NOTIFICATIONS
- In-app notifications (V1)
- Firebase FCM push (Pilot)
- Resend email (Pilot)
- NotificationService (channel abstraction)

**Parallel with:** Phase 5

### Phase 7: ADMIN / OPS DASHBOARD
- Provider verification workflow
- Booking operations + intervention
- City activation + configuration
- Service + pricing + commission management
- Settlements view
- Audit log viewer

**Requires:** Phases 2, 4, 5, 6

### Phase 8: FLUTTER MOBILE
- Customer App (same backend APIs as Phase 4)
- Provider App (same backend APIs as Phase 4)

**Requires:** Phases 2, 3, 4, 5, 6 stable

### Phase 9: SCALE (introduce only when justified)
- Async Outbox Worker (replace SyncEventBus)
- Redis caching (service catalogue, city data)
- Redis rate limiting
- PostGIS for advanced geofencing
- Read replicas
- Advanced ML matching signals

---

## D. DATABASE MODULES (28 Tables — Final)

### D.1 Identity & Profiles (3 tables)
`profiles` · `customers` · `providers`

### D.2 Documents & Storage (1 table)
`provider_documents`

### D.3 Geography / Multi-City (3 tables) ⭐ **Core**
`cities` · `areas` · `addresses`

### D.4 Service Catalogue (4 tables)
`service_categories` · `services` · `service_requirements` · `skills`

### D.5 Provider Capabilities (5 tables)
`provider_skills` · `provider_service_capabilities` · `provider_service_areas` · `provider_availability` · `provider_availability_overrides`

### D.6 Pricing & Commission (2 tables) ⭐ **Business-Critical**
`pricing_configurations` · `commission_configurations`

### D.7 Bookings (4 tables) ⭐ **Core**
`bookings` · `booking_requirements` · `booking_status_history` · `provider_assignments`

### D.8 Payments & Settlements (3 tables) ⭐ **Revenue-Critical**
`payments` · `refunds` · `settlements`

### D.9 Reviews, Notifications, Support (3 tables)
`reviews` · `notifications` · `support_tickets`

### D.10 Events & Audit (2 tables) ⭐ **Production-Critical**
`domain_events` · `audit_logs`

---

## E. CORE DOMAIN MODULES

| Module | File Location | V1 Status |
|---|---|---|
| AuthService | `/services/auth.service.ts` | ✅ V1 |
| MatchingEngine | `/features/matching/engine.ts` | ✅ V1 (deterministic) |
| AvailabilityEngine | `/features/matching/availability.ts` | ✅ V1 |
| BookingService | `/services/booking.service.ts` | ✅ V1 |
| BookingStateMachine | `/features/bookings/state-machine.ts` | ✅ V1 |
| PricingService | `/services/pricing.service.ts` | ✅ V1 |
| PaymentService | `/services/payment.service.ts` | ✅ V1 (Mock) |
| CommissionService | `/services/commission.service.ts` | ✅ V1 |
| SettlementService | `/services/settlement.service.ts` | ✅ V1 |
| NotificationService | `/services/notification.service.ts` | ✅ V1 (In-app) |
| AuditService | `/services/audit.service.ts` | ✅ V1 |
| EventBus | `/lib/events/event-bus.interface.ts` | ✅ V1 (Sync) |
| StorageProvider | `/lib/storage/storage.interface.ts` | ✅ V1 (R2) |
| GeoUtils | `/utils/geolocation.ts` | ✅ V1 (Haversine) |
| CityService | `/services/city.service.ts` | ✅ V1 |

---

## F. EVENT MODULES

### Domain Events → Handlers

| Event | Sync Handlers (V1) | Async Handlers (Production) |
|---|---|---|
| BOOKING_CREATED | AuditLog | Matching Worker → offer provider |
| PROVIDER_OFFERED | AuditLog | Timeout Worker (set offer expiry) |
| PROVIDER_ACCEPTED | BookingState → ACCEPTED, AuditLog | NotificationWorker (customer) |
| PROVIDER_REJECTED | AuditLog | ReassignmentWorker |
| PROVIDER_OFFER_EXPIRED | ReassignmentLogic, AuditLog | NotificationWorker |
| SERVICE_COMPLETED | BookingState, AuditLog | PaymentWorker → trigger payment |
| PAYMENT_CAPTURED | SettlementCreate, AuditLog | NotificationWorker |
| PAYMENT_FAILED | AuditLog | NotificationWorker |
| REFUND_CREATED | SettlementUpdate, AuditLog | NotificationWorker |
| PROVIDER_STATUS_CHANGED | AuditLog | NotificationWorker |
| CITY_ACTIVATED | AuditLog | NotificationWorker (ops team) |
| REVIEW_SUBMITTED | ProviderRatingUpdate, AuditLog | — |

---

## G. INFRASTRUCTURE MODULES

### G.1 Storage — Cloudflare R2 (via StorageProvider abstraction)

```typescript
interface StorageProvider {
  upload(params: UploadParams): Promise<UploadResult>
  getSignedUrl(key: string, expiresInSeconds: number): Promise<string>
  delete(key: string): Promise<void>
}

// CloudflareR2Provider — initial
// Swap: S3Provider / SupabaseStorageProvider — zero domain changes
```

### G.2 Workers (V1 via Vercel Cron — Production via dedicated process)

| Worker | V1 Mechanism | Production Mechanism |
|---|---|---|
| Offer Timeout | Vercel Cron `/api/webhooks/cron/offer-timeout` | Dedicated Node worker + Redis |
| Outbox Processor | Vercel Cron `/api/webhooks/cron/events` | Dedicated async worker |
| Settlement | Vercel Cron `/api/webhooks/cron/settlements` | Dedicated worker |
| Provider Rating | Supabase DB Trigger (auto on review) | Same |

### G.3 Rate Limiting

| Endpoint | V1 | Production |
|---|---|---|
| `/api/auth/*` | Vercel Edge middleware (IP) | Redis token bucket |
| `/api/matching` | — | Redis per-user rate limit |
| `/api/bookings` | — | Redis per-user rate limit |
| `/api/webhooks/*` | Signature verification | Same + IP allowlist |

---

## H. SECURITY MODULES

### H.1 Auth + RBAC

| Role | Permissions |
|---|---|
| **CUSTOMER** | own profile, bookings, payments, reviews, addresses |
| **PROVIDER** | own profile, capabilities, assigned bookings, earnings |
| **ADMIN** | provider mgmt, booking ops, service config, city config |
| **OPERATIONS** | booking ops, assignment intervention |
| **SUPPORT** | ticket management, booking view (read-only) |
| **SUPER_ADMIN** | full platform access, role management |

### H.2 RLS (All tables — enforced by PostgreSQL)

| Sensitivity | Tables | Policy |
|---|---|---|
| **Private — own only** | profiles, customers, bookings, payments, addresses, notifications | `auth.uid() = owner_id` |
| **Private — role-scoped** | providers, provider_documents, settlements | provider own + admin |
| **City-gated read** | cities, areas | `status = 'ACTIVE'` for public + admin all |
| **Service-gated read** | services, service_categories | `is_active = true` for public + admin all |
| **Admin only** | commission_configurations, pricing_configurations, audit_logs, domain_events | — |

---

**End of Implementation Plan**
