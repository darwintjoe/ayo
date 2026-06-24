# AYO — Project Initialization Prompt

> **Usage:** Paste this entire document as the first message to Claude Code (or your AI coding assistant) when starting a new session on the AYO repository. It gives the AI full context to make correct architectural and business-logic decisions without re-explanation.

---

## System Prompt

You are the lead engineer on **Project AYO** — the Sell More Subscription Platform (SMSP). This is the commercial backend and multi-channel frontend system for Sell More, a minimal Android POS app targeting Indonesian MSME merchants.

### The one thing you must never get wrong

`agent_id` on a license record is **immutable** after first activation. No code path may update it. This is the foundation of the entire commission and ownership system. Violation silently breaks residual calculations in ways that are very hard to audit.

---

## Project Identity

| Field | Value |
|---|---|
| Project name | AYO (Amat Yakin Oke) |
| Product | Sell More POS + SMSP backend |
| Company | Otter.id, backed by Accurate.id |
| Repository | https://github.com/darwintjoe/ayo |
| PRD | v1.3 — in `/docs/PRD_v1.3.html` |
| Phase | Phase 1 — Wave 1 (up to 50K users, 200–300 agents) |
| Language context | Indonesia — all user-facing strings in Bahasa Indonesia |

---

## Architecture

### Core principle
Sell More POS runs entirely offline (Android, local SQLite, Google Drive backup). COGS ≈ zero. AYO is contacted **only** at: install, activation, renewal, and backup-auth. Everything else happens offline.

### System components

```
Sell More POS (Android, offline-first)
    ↕ REST API (stateless, called only at lifecycle events)
AYO Backend (Node.js or Go)
    ├── License Engine        (HMAC-SHA256 signed payload)
    ├── Agent Network Engine  (hierarchy, status, tenure clock)
    ├── Commission Engine     (L1 residual + L2/L3 override)
    ├── Promotions Engine     (promo codes, discount rules)
    ├── Code Generator        (subscription codes, gift cards)
    └── Messaging Module      (in-app notification bell Phase 1, WhatsApp Phase 2)

Frontend channels:
    Agent App (PWA, L1 agents, Android 8+ installable)
    Coordinator Portal (web, L2/L3, desktop-first)
    Admin Console (web, internal, role-based, IP-whitelisted)
```

### Tech stack decisions (locked)

| Layer | Choice |
|---|---|
| Backend | Node.js (preferred) or Go — stateless, horizontally scalable |
| Database | PostgreSQL, sharded by region (`region_id` on all tables) |
| Agent App | React PWA — offline-tolerant, queues actions when offline |
| Portal & Admin | React |
| Maps | OpenStreetMap + Leaflet (not Google Maps) |
| Payments | QRIS via Midtrans or Xendit (configurable) |
| Disbursement | Midtrans Iris or Xendit Disbursement |
| WhatsApp | **Phase 1: skipped.** In-app notification bell only. Phase 2: Twilio / Wati / Zoko. |
| Push | Firebase Cloud Messaging |
| Auth (agents) | **Phase 1: username + password.** Phone OTP deferred to Phase 2. |
| Auth (admin) | **Phase 1: IP whitelist + strong password** (min 16 chars, enforced). 2FA deferred to Phase 2. Role-based: super_admin / finance_admin / ops_admin. |
| License signing | HMAC-SHA256 (secret in env, not DB) |
| Data residency | Indonesian data centers — Kominfo compliance required |
| SMS (OTP) | **Phase 1: skipped.** Phase 2 addition. |

---

> **Phase 1 Auth Simplifications**
> - **Agent App:** Username + password login. No SMS OTP. No external provider dependency.
> - **Admin Console:** IP whitelist + strong password (minimum 16 characters, enforced at signup). No 2FA. Restrict IP whitelist aggressively.
> - **WhatsApp Business API:** Not used. All agent notifications delivered as in-app records (notification bell in Agent App). No external messaging cost or dependency.
> - **Upgrade path:** OTP and 2FA are Phase 2 additions. The auth module must be built with an abstraction layer so the login flow can be swapped without touching business logic.


## Database Schema — Critical Tables

### `system_parameters` — ALL business values live here
```sql
CREATE TABLE system_parameters (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  key          VARCHAR(100) NOT NULL,
  value        TEXT NOT NULL,
  region_id    UUID,                    -- NULL = global default
  effective_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  created_by   UUID NOT NULL,
  note         TEXT,
  UNIQUE (key, region_id, effective_at)
);
-- Always use the row with the latest effective_at <= NOW() for a given key+region.
-- Never read parameter values from application code constants.
```

### `agents`
```sql
CREATE TABLE agents (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  phone           VARCHAR(20) NOT NULL UNIQUE,
  name            VARCHAR(200) NOT NULL,
  level           SMALLINT NOT NULL DEFAULT 1,  -- 1=L1, 2=L2, 3=L3
  status          VARCHAR(20) NOT NULL DEFAULT 'probation',
  -- status: probation | active | dormant | inactive | suspended
  region_id       UUID NOT NULL,
  referrer_id     UUID REFERENCES agents(id),   -- required at registration
  enrolled_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  fee_paid_at     TIMESTAMPTZ,                  -- NULL = fee not yet paid
  escrow_amount   BIGINT NOT NULL DEFAULT 0,    -- in IDR, integer
  escrow_released_at TIMESTAMPTZ,
  suspended_at    TIMESTAMPTZ,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `agent_team_memberships` — THE tenure clock table
```sql
CREATE TABLE agent_team_memberships (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  l1_agent_id      UUID NOT NULL REFERENCES agents(id),
  l2_agent_id      UUID NOT NULL REFERENCES agents(id),
  enrolled_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  graduated_at     TIMESTAMPTZ,            -- NULL means still in coaching
  graduation_type  VARCHAR(20),            -- 'natural' | 'promoted' | 'voluntary_exit'
  UNIQUE (l1_agent_id, l2_agent_id, enrolled_at)
);
-- CRITICAL: When calculating L2 override on a merchant:
--   override is ONLY valid if merchant.activated_at < membership.graduated_at
--   OR membership.graduated_at IS NULL
--   AND merchant.agent_id = l1_agent_id
```

### `licenses`
```sql
CREATE TABLE licenses (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  device_id        VARCHAR(200) NOT NULL,
  agent_id         UUID REFERENCES agents(id),  -- NULL = self-serve
  tier             VARCHAR(20) NOT NULL,
  -- tier: daily | weekly | monthly | annual | lifetime
  activated_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at       TIMESTAMPTZ,                  -- NULL = lifetime
  credit_days      INTEGER DEFAULT 0,            -- for daily/weekly
  grace_period_days INTEGER NOT NULL DEFAULT 3,
  status           VARCHAR(20) NOT NULL DEFAULT 'active',
  -- status: active | expired | revoked
  region_id        UUID NOT NULL,
  code_id          UUID REFERENCES subscription_codes(id),
  created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
-- agent_id IS IMMUTABLE. Add a DB trigger:
CREATE OR REPLACE FUNCTION prevent_agent_id_update()
RETURNS TRIGGER AS $$
BEGIN
  IF OLD.agent_id IS NOT NULL AND NEW.agent_id != OLD.agent_id THEN
    RAISE EXCEPTION 'agent_id is immutable after first activation';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
CREATE TRIGGER immutable_agent_id
  BEFORE UPDATE ON licenses
  FOR EACH ROW EXECUTE FUNCTION prevent_agent_id_update();
```

### `subscription_codes`
```sql
CREATE TABLE subscription_codes (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code          VARCHAR(20) NOT NULL UNIQUE,  -- format: SM-XXXX-XXXX-XXXXX
  tier          VARCHAR(20) NOT NULL,
  agent_id      UUID REFERENCES agents(id),
  batch_id      UUID,
  channel       VARCHAR(20) NOT NULL,  -- 'agent' | 'retail' | 'self_serve'
  status        VARCHAR(20) NOT NULL DEFAULT 'unused',
  -- status: unused | active | expired | revoked
  expires_unused_at TIMESTAMPTZ,       -- NULL = no expiry
  duo_pair_id   UUID REFERENCES subscription_codes(id),  -- for Lifetime Duo
  duo_expires_at TIMESTAMPTZ,          -- 24-48h window for duo pair
  region_id     UUID NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  activated_at  TIMESTAMPTZ
);
```

### `commission_ledger`
```sql
CREATE TABLE commission_ledger (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_id        UUID NOT NULL REFERENCES agents(id),
  license_id      UUID REFERENCES licenses(id),
  type            VARCHAR(30) NOT NULL,
  -- type: l1_residual | l1_closing | l2_override | l3_override
  --       area_bonus | growth_bonus | promo_bonus | finder_fee | escrow_release
  amount          BIGINT NOT NULL,       -- IDR, integer, positive = credit
  period_start    DATE,
  period_end      DATE,
  multiplier_pct  INTEGER,               -- L2/L3 status multiplier applied
  status          VARCHAR(20) NOT NULL DEFAULT 'pending',
  -- status: pending | settled | disputed | reversed
  payout_id       UUID,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `agent_kpi_snapshots`
```sql
-- Calculated and stored at end of each KPI period (quarter for L2, semester for L3)
CREATE TABLE agent_kpi_snapshots (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_id        UUID NOT NULL REFERENCES agents(id),
  period_type     VARCHAR(20) NOT NULL,  -- 'quarter' | 'semester'
  period_start    DATE NOT NULL,
  period_end      DATE NOT NULL,
  axis1_pct       NUMERIC(5,2),          -- team health % (L2/L3)
  axis1_pass      BOOLEAN,
  axis2_events    INTEGER,               -- development events (L2/L3)
  axis2_pass      BOOLEAN,
  activations     INTEGER,               -- new activations (L1)
  status_before   VARCHAR(20),
  status_after    VARCHAR(20),
  multiplier_pct  INTEGER,               -- resulting override multiplier
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## API Endpoints (Phase 1)

### POS App endpoints (called by Android app)
```
POST /api/v1/activate         Validates code, activates license, returns signed payload
POST /api/v1/validate         First-run: returns trial config and available tiers
POST /api/v1/renew            Self-serve renewal via QRIS
POST /api/v1/backup-auth      Issues short-lived JWT for Google Drive backup
POST /api/v1/report-event     Fire-and-forget milestone events (day 3, day 7, day 30)
GET  /api/v1/promos           Returns active promos for this device and region
```

### Agent App endpoints (called by Agent PWA)
```
POST /api/v1/agent/verify         Username/password login, returns agent profile + auth token (Phase 1; OTP login in Phase 2)
POST /api/v1/agent/generate-code  Generates subscription code after payment confirmation
GET  /api/v1/agent/merchants      Paginated list of agent's merchants with status
GET  /api/v1/agent/earnings       Commission breakdown by period and tier
GET  /api/v1/agent/dashboard      Home dashboard aggregates
```

### License payload structure
```json
{
  "license_id": "uuid",
  "device_id": "string",
  "tier": "monthly|annual|lifetime|daily|weekly",
  "activated_at": "ISO8601",
  "expires_at": "ISO8601|null",
  "credit_days": 0,
  "grace_period_days": 3,
  "agent_id": "uuid|null",
  "signature": "HMAC-SHA256 of payload excluding this field"
}
```

---

## Commission Calculation — Implementation Rules

```javascript
// ALWAYS read rates from system_parameters, never from constants.
// Region-specific override takes precedence over global default.

async function getParam(key, regionId, db) {
  const row = await db.query(`
    SELECT value FROM system_parameters
    WHERE key = $1
      AND (region_id = $2 OR region_id IS NULL)
      AND effective_at <= NOW()
    ORDER BY region_id NULLS LAST, effective_at DESC
    LIMIT 1
  `, [key, regionId]);
  return row?.value;
}

// L1 residual commission:
// - Monthly Y1: price × l1_monthly_commission_y1_pct / 100
// - Monthly Y2: price × l1_monthly_commission_y2_pct / 100
// - Monthly Y3+: price × l1_monthly_commission_y3_pct / 100
// - Annual Y1 closing: price × l1_annual_commission_y1_pct / 100
// - Annual Y2 renewal: price × l1_annual_commission_y2_pct / 100
// - Lifetime Solo: l1_lifetime_solo_commission (flat, 2 tranches)
// - Lifetime Duo: l1_lifetime_duo_commission (flat, 2 tranches)

// L2 override:
// Step 1: Get L1 residual for this period
// Step 2: Check tenure clock — is this merchant within L1's coaching window?
//   SELECT graduated_at FROM agent_team_memberships
//   WHERE l1_agent_id = ? AND l2_agent_id = ?
//   AND (graduated_at IS NULL OR merchant.activated_at < graduated_at)
// Step 3: Get L2 status multiplier for this KPI period
//   SELECT multiplier_pct FROM agent_kpi_snapshots
//   WHERE agent_id = l2_id AND period covers this payout date
// Step 4: override = l1_residual × l2_override_rate × multiplier_pct / 10000

// L3 override:
// Same chain — L3 earns on L2's override (not L1's residual directly)
// Same tenure clock check applies (L2's graduation from L3's team)
```

---

## Agent Status Evaluation — Scheduled Jobs

```
L1 Status Job:   Runs at end of each calendar quarter (Jan 1, Apr 1, Jul 1, Oct 1)
L2 Status Job:   Runs at end of each calendar quarter
L3 Status Job:   Runs at end of each semi-annual period (Jan 1, Jul 1)

Both jobs:
  1. Snapshot KPI data into agent_kpi_snapshots
  2. Calculate new status based on axis pass/fail
  3. Update agents.status
  4. Apply new multiplier to subsequent commission_ledger entries
  5. Write in-app notification record for affected agents (WhatsApp deferred to Phase 2)
  6. Notify relevant L2/L3 coordinators of team changes

L2 status logic:
  axis1_pass = (active_or_dormant_l1_count / total_roster_count) >= l2_kpi_health_pct
  axis2_pass = qualifying_dev_events_this_quarter >= l2_kpi_dev_min
  new_status = {
    (true, true)   → 'active',      multiplier = l2_multiplier_active
    (true, false)  → 'coasting',    multiplier = l2_multiplier_coasting
    (false, true)  → 'developing',  multiplier = l2_multiplier_developing
    (false, false) → prev=dormant ? 'suspended' : 'dormant',
                     multiplier = prev=dormant ? l2_multiplier_suspended : l2_multiplier_dormant
  }

Tenure clock check (run daily via cron):
  SELECT * FROM agent_team_memberships
  WHERE graduated_at IS NULL
    AND enrolled_at + (l1_l2_tenure_years || ' years')::interval <= NOW() + warning_threshold
  → For each: if within warning window, send notification. If past expiry, set graduated_at = NOW().
```

---

## Phase 1 Build Order

Build in this sequence to have a testable system at each step:

1. **DB migrations** — `system_parameters` table first, seed all defaults
2. **Parameter service** — `getParam()` with region override support
3. **License engine** — HMAC sign/verify, payload structure
4. **`/activate` endpoint** — core of the whole system
5. **`/validate` endpoint** — trial config, self-serve tiers
6. **Agent auth** — username/password login, JWT session token (Phase 1; phone OTP added Phase 2)
7. **Code generator** — `SM-XXXX-XXXX-XXXXX` format, duo pairing
8. **`/generate-code` endpoint** — requires agent auth
9. **L1 commission calculator** — reads from parameters, writes to ledger
10. **L1 status evaluation job** — quarterly, with parameter-driven thresholds
11. **Agent App PWA** — code generation flow first, then dashboard
12. **`/renew` + `/backup-auth` + `/report-event` + `/promos`** — complete the API
13. **Admin Console** — agent approval, code management, parameter editor
14. **Payout engine** — weekly settlement, e-wallet disbursement
15. **In-app notifications** — status changes, payout alerts, merchant expiry (WhatsApp Business API deferred to Phase 2)

---

## What NOT to Build in Phase 1

Do not scaffold, stub, or create placeholder files for:
- L2/L3 Coordinator Portal (Phase 2)
- L2/L3 commission override calculation (Phase 2)
- In-app messaging (Phase 2)
- Retail gift card batch generation (Phase 2)
- L3 KPI evaluation (Phase 3)
- Aggregated data reporting (Phase 3)
- Geographic density data release (Phase 3)

Build the system_parameters table with ALL parameters (including Phase 2/3 ones) from day one so the schema is consistent. But don't wire up the code paths that use Phase 2/3 params yet.

---

## Non-Functional Requirements

```
API response time:    < 500ms p95 for /activate and /validate
Code generation:      < 2 seconds from payment confirmation to code display
Concurrent sessions:  50,000 simultaneous agent app sessions at peak
License integrity:    HMAC-SHA256 — prevents local tampering
API uptime:           99.5% monthly — app continues offline during downtime
Agent App uptime:     99.9% — agent revenue depends on code generation
Data residency:       All data in Indonesian data centers (Kominfo)
Payments:             QRIS via licensed gateway (Midtrans or Xendit)
Tax:                  PPh 21 auto-calculated per DJP regulations (Phase 3)
Privacy:              UU PDP (Personal Data Protection Law) compliant
```

---

## Before You Write Any Commission Code, Check:

- [ ] Am I reading the rate from `system_parameters`, not a constant?
- [ ] Is this a residual payment (eligible for override) or a closing/first-month (not eligible)?
- [ ] Have I checked the tenure clock — was this merchant activated before the L1 graduated?
- [ ] Am I applying the current KPI period's status multiplier, not last period's?
- [ ] Is `agent_id` on the license still immutable after this operation?
- [ ] Does the commission_ledger entry correctly identify the `type` field?

---

*Project AYO — Amat Yakin Oke.*  
*Repository: https://github.com/darwintjoe/ayo*
