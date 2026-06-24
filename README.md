# AYO — Amat Yakin Oke

**Sell More Subscription Platform (SMSP)**  
Agent network management · License engine · Commission calculation · Messaging

---

## What This Is

AYO is the commercial backend and multi-channel frontend system powering **Sell More** — a minimal Android POS application targeting 20 million Indonesian MSME merchants. The app itself runs entirely offline (local SQLite + Google Drive backup). AYO handles everything that requires a server: subscription licensing, agent network management, commission calculation, promotional pricing, and inter-agent communication.

**Core architectural principle:** Sell More's POS app has near-zero COGS because it runs locally. AYO is only contacted at install, activation, renewal, and backup. This makes any subscription price point profitable, enabling agent commission structures impossible for cloud-based competitors.

---

## Project Context

| Item | Detail |
|---|---|
| **Product** | Sell More — POS & Business Intelligence for Indonesian MSMEs |
| **Company** | Otter.id (backed by Accurate.id) |
| **Founder** | Darwin Tjoe |
| **PRD** | SMSP PRD v1.3 — see `/docs/PRD_v1.3.html` |
| **Target** | 25 million users, 3-level agent network |
| **Phase** | Phase 1 — Wave 1 launch (Java, 200–300 agents, up to 50K users) |

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     SELL MORE POS (Android)                  │
│              Local SQLite · Google Drive Backup              │
│         Contacts AYO only at: activate / renew / backup      │
└───────────────────┬─────────────────────────────────────────┘
                    │ REST API (lightweight, stateless)
┌───────────────────▼─────────────────────────────────────────┐
│                        AYO BACKEND                           │
│                                                              │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │ License     │  │ Agent Network│  │ Commission         │  │
│  │ Engine      │  │ Management   │  │ Calculator         │  │
│  └─────────────┘  └──────────────┘  └────────────────────┘  │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │ Promotions  │  │ Code/Voucher │  │ Messaging          │  │
│  │ Engine      │  │ Generator    │  │ Module             │  │
│  └─────────────┘  └──────────────┘  └────────────────────┘  │
└──────┬──────────────────┬──────────────────┬────────────────┘
       │                  │                  │
┌──────▼──────┐  ┌────────▼───────┐  ┌───────▼─────────────┐
│ Agent App   │  │ Coordinator &  │  │ Admin Console       │
│ (PWA)       │  │ Regional Portal│  │ (Web, Internal)     │
│ L1 agents   │  │ L2 / L3 web    │  │ Full system control │
└─────────────┘  └────────────────┘  └─────────────────────┘
```

---

## Repository Structure

```
ayo/
├── docs/
│   ├── PRD_v1.3.html              # Full product requirements document
│   ├── agent_schema_v2.html       # L2/L3 KPI schema with tenure clock
│   └── master_strategy.md         # Market strategy and positioning
│
├── backend/                       # AYO API server
│   ├── src/
│   │   ├── api/                   # REST endpoints
│   │   │   ├── activate.js        # POST /activate
│   │   │   ├── validate.js        # POST /validate
│   │   │   ├── renew.js           # POST /renew
│   │   │   ├── backup-auth.js     # POST /backup-auth
│   │   │   ├── report-event.js    # POST /report-event
│   │   │   ├── promos.js          # GET /promos
│   │   │   ├── verify-agent.js    # POST /verify-agent
│   │   │   └── generate-code.js   # POST /generate-code
│   │   ├── engine/
│   │   │   ├── commission/        # Commission calculation engine
│   │   │   ├── status/            # Agent status evaluation engine
│   │   │   ├── tenure/            # Tenure clock management
│   │   │   └── license/           # License payload & HMAC signing
│   │   ├── models/                # Database models
│   │   ├── jobs/                  # Scheduled jobs (status eval, payouts)
│   │   └── config/
│   │       └── parameters.js      # All admin-editable parameters (see below)
│   ├── migrations/                # DB schema migrations
│   └── tests/
│
├── agent-app/                     # L1 Agent PWA
│   ├── src/
│   │   ├── screens/
│   │   │   ├── Home/              # Dashboard: status, activations, income
│   │   │   ├── GenerateCode/      # 4-step code generation flow
│   │   │   ├── Merchants/         # My merchants list with status
│   │   │   └── Earnings/          # Commission breakdown & payout history
│   │   └── components/
│   └── public/
│
├── portal/                        # L2/L3 Coordinator & Regional Portal (web)
│   ├── src/
│   │   ├── views/
│   │   │   ├── TeamDashboard/     # Agent performance table, traffic lights
│   │   │   ├── TerritoryMap/      # OSM/Leaflet activation heatmap
│   │   │   └── Override/          # Override earnings breakdown
│   │   └── components/
│
├── admin/                         # Admin Console (web, internal)
│   ├── src/
│   │   ├── views/
│   │   │   ├── Overview/          # Real-time KPIs
│   │   │   ├── Pricing/           # Tier management
│   │   │   ├── Agents/            # Agent approval, status override
│   │   │   ├── Codes/             # Code/license management
│   │   │   ├── Parameters/        # All system parameters (see registry)
│   │   │   └── Reports/           # Revenue, commission, PPh 21
│   │   └── components/
│
└── shared/                        # Shared types, constants, utilities
    ├── types/
    └── constants/
```

---

## Core Data Model Constraints

These constraints are non-negotiable. Any schema deviation breaks the commission and ownership logic:

### Merchant Ownership
```sql
-- agent_id on a license record is IMMUTABLE after first activation.
-- Never update this field after INSERT.
ALTER TABLE licenses ADD COLUMN agent_id UUID;
-- No UPDATE trigger allowed on agent_id.
```

### Tenure Clock
```sql
-- Every L1-under-L2 relationship requires a start timestamp.
-- This drives the graduation mechanic.
CREATE TABLE agent_team_memberships (
  id            UUID PRIMARY KEY,
  l1_agent_id   UUID NOT NULL REFERENCES agents(id),
  l2_agent_id   UUID NOT NULL REFERENCES agents(id),
  enrolled_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  graduated_at  TIMESTAMPTZ,               -- NULL = active coaching
  graduation_type VARCHAR(20)              -- 'natural' | 'promoted' | 'exit'
);
-- Override on a given merchant is only valid if:
--   merchant.activated_by = l1_agent_id
--   AND merchant.activated_at < membership.graduated_at (or graduated_at IS NULL)
```

### Commission Calculation Rule
```
Override is NEVER applied to:
  - First-month commission on any tier
  - Closing commission on Annual, Lifetime Solo, Lifetime Duo
  - Finder fee payments

Override IS applied to:
  - Monthly residual (Y1 at 40%, Y2 at 30%, Y3 at 20%)
  - Annual renewal (Y2 at 20%)

L2 override = L1_residual_this_period × l2_override_rate × l2_status_multiplier
L3 override = L2_override_this_period × l3_override_rate × l3_status_multiplier
```

---

## Parameter Registry

All business logic values are stored in the `system_parameters` table and editable via Admin Console. **No business values are hardcoded.** See `backend/src/config/parameters.js` for the full list with defaults. Key parameters:

| Category | Key | Default |
|---|---|---|
| Tenure | `l1_l2_tenure_years` | 5 |
| Tenure | `l2_l3_tenure_years` | 5 |
| L1 Status | `l1_active_min_activations` | 3 /quarter |
| L1 Status | `l1_dormant_residual_pct` | 50% |
| L1 Status | `l1_inactive_residual_pct` | 25% |
| L1 Commission | `l1_monthly_commission_y1_pct` | 40% |
| L1 Commission | `l1_lifetime_solo_commission` | Rp 500,000 |
| L2 KPI | `l2_kpi_health_pct` | 60% of roster |
| L2 KPI | `l2_kpi_dev_min` | 1 event/quarter |
| L2 Override | `l2_override_rate` | 15% |
| L2 Multipliers | `l2_multiplier_active` / `coasting` / `developing` / `dormant` / `suspended` | 100/75/60/30/0% |
| L2 Roster | `l2_roster_soft_cap` | 15 agents |
| L2 Bonuses | `l2_bonus_area_threshold` / `amount` | 200 merchants / Rp 500K/mo |
| L2 Bonuses | `l2_promo_bonus_per_l1_promoted` | Rp 1,500,000 |
| L3 KPI | `l3_kpi_health_pct` | 60% of L2 roster |
| L3 KPI | `l3_kpi_period` | Semi-annual |
| L3 Override | `l3_override_rate` | 10% |
| L3 Multipliers | `l3_multiplier_active` / `coasting` / `developing` / `dormant` / `suspended` | 100/75/60/30/0% |
| L3 Bonuses | `l3_bonus_region_threshold` / `amount` | 2,000 merchants / Rp 2M/mo |
| L3 Bonuses | `l3_promo_bonus_per_l2_promoted` | Rp 3,000,000 |
| Payout | `payout_minimum_threshold` | Rp 50,000 |
| Payout | `pph21_threshold` | Rp 4,500,000/mo |

Regional overrides supported for all parameters. Changes are future-dated and audit-logged.

---

## Tech Stack

| Layer | Technology |
|---|---|
| POS App | Android (Java/Kotlin), SQLite, Google Drive API |
| Agent PWA | React / Vue, service worker, offline-first |
| Coordinator Portal | React, Leaflet/OSM for territory maps |
| Admin Console | React, role-based access |
| Backend API | Node.js / Go — stateless, horizontally scalable |
| Database | PostgreSQL, sharded by region |
| Payments | QRIS via Midtrans or Xendit |
| Disbursement | Midtrans Iris or Xendit Disbursement |
| WhatsApp | ~~Phase 1: skipped~~ — Phase 2: Twilio / Wati / Zoko |
| Maps | OpenStreetMap + Leaflet |
| Push | Firebase Cloud Messaging |
| Auth (agents) | Username + password (Phase 1). Phone OTP added in Phase 2. |
| Auth (admin) | IP whitelist + strong password (Phase 1). 2FA added in Phase 2. |
| License signing | HMAC-SHA256 |
| Data residency | Indonesian data centers (Kominfo compliance) |

---

## Phase 1 Scope (Wave 1)

Phase 1 builds the minimum viable system for launching with 200–300 agents and up to 50,000 users.

**In scope for Phase 1:**
- Agent App: code generation, merchant list, earnings, username/password auth
- Admin Console: agent approval, code management, basic pricing config
- API: `/activate`, `/validate`, `/renew`, `/backup-auth`, `/report-event`, `/promos`, `/verify-agent`, `/generate-code`
- Commission Engine: L1 only, weekly payout via e-wallet
- All 6 pricing tiers + basic promo code support
- System notifications: in-app notification bell only (Phase 1). WhatsApp Business API deferred to Phase 2.
- L1 status evaluation: Probation / Active / Dormant / Inactive / Suspended

**Deferred to Phase 2:**
- L2/L3 Coordinator Portal and override calculation
- In-app messaging (direct + team broadcast)
- Full Promotions Engine with agent-created promos
- Retail gift card batch generation

**Deferred to Phase 3:**
- L3 dashboard and semi-annual KPI evaluation
- Aggregated data reporting layer
- Geographic density data released to agents
- PPh 21 automation and cohort analysis

---

## Critical Business Rules

Read these before writing any code that touches commissions, ownership, or agent status.

1. **`agent_id` is immutable.** Once a merchant is activated, the activating agent owns that merchant's residual stream forever. Never reassign without explicit admin action and audit log entry.

2. **Override on residual only.** L2 and L3 overrides never apply to first-month or closing commissions. Only recurring residual payments are subject to override.

3. **Tenure clock determines override validity.** L2 earns override on a merchant only if that merchant was activated by the L1 *before* that L1's graduation date from L2's team.

4. **Status multipliers are applied at payout time.** Do not reduce the merchant's payment or the L1's residual. Reduce only the L2/L3 override disbursement.

5. **Lifetime conversions do not count toward KPI.** An L1 converting their existing monthly users to lifetime does not count toward their quarterly activation minimum (anti-exit-gaming rule).

6. **All parameters come from the database.** No commission rate, threshold, or period is hardcoded. Always read from `system_parameters` table (with per-region override support).

7. **Grace period for offline operation.** After license expiry, cashier login is blocked but admin (owner) login remains fully functional. Default grace: 3 days (`grace_period_days` in license payload).

---

## Getting Started

```bash
# Clone
git clone https://github.com/darwintjoe/ayo.git
cd ayo

# Backend
cd backend
cp .env.example .env          # fill in DB_URL, QRIS keys, HMAC_SECRET
npm install
npm run migrate               # runs all DB migrations including parameter seeds
npm run dev

# Agent PWA
cd ../agent-app
npm install
npm run dev

# Admin Console
cd ../admin
npm install
npm run dev
```

### Environment Variables (backend)

```
DATABASE_URL=postgresql://...
HMAC_SECRET=<32-byte secret for license signing>
QRIS_GATEWAY=midtrans|xendit
MIDTRANS_SERVER_KEY=...
XENDIT_SECRET_KEY=...
# WHATSAPP_PROVIDER=twilio|wati|zoko   # Phase 2
# WHATSAPP_API_KEY=...                  # Phase 2
GOOGLE_DRIVE_CLIENT_ID=...
GOOGLE_DRIVE_CLIENT_SECRET=...
FCM_SERVER_KEY=...
# OTP_PROVIDER=twilio|vonage           # Phase 2
# OTP_API_KEY=...                      # Phase 2
ADMIN_IP_WHITELIST=...
ADMIN_PASSWORD_MIN_LENGTH=16    # strong password enforced, 2FA deferred to Phase 2
```

---

## Documentation

| Document | Location |
|---|---|
| Product Requirements (PRD v1.3) | `docs/PRD_v1.3.html` |
| L2/L3 KPI Schema v2.0 | `docs/agent_schema_v2.html` |
| Master Strategy | `docs/master_strategy.md` |
| API Reference | `backend/docs/api.md` (generated) |
| Parameter Registry | `backend/src/config/parameters.js` |

---

## License

Confidential — Accurate.id / Otter.id / Sell More. All rights reserved.

---

*AYO — Amat Yakin Oke. Built to reach the 97.5% of Indonesian merchants that no POS player has ever served.*
