Parly Relayer Registry PRD/Design Doc
Lean Production Spec Single Source of Truth

---
1. Objective
Create a single ultra-lean page at:
/relayer
This page exists for one purpose only:
Allow a relayer operator to register a relayer for official frontend discovery.
This page is not:
- a relayer marketplace
- a public ranking system
- a moderation dashboard
- a support center
- a general education page
Operators who need education go to docs.
Operators who are ready to register submit directly from this page.

---
2. Product Law
2.1 Core law
Parly protocol participation remains open.
Official frontend discovery is curated.
That means:
- a relayer can exist without being listed
- the registry is for official app discovery + relayer public key distribution
- the registry is not protocol permissioning
This stays consistent with V16.9.9, which explicitly does not require a monopoly relayer for correctness, while the current frontend still reads relayers from a registry input.
2.2 Registry law
The official Parly registry supports a maximum of:
10 live relayers
If live relayers = 10:
- new registration is blocked
- register button is disabled
- page clearly shows registry full state
2.3 Registration law
Registration is automatic if validation passes.
There is:
- no submitted queue
- no manual moderation queue
- no internal review screen required for launch
The registry lifecycle is:
- verified
- inactive
- rejected
2.4 Inactive law
A verified relayer may be marked inactive and removed from official discovery if:
- it shows no successful relay activity for 30 days, measured from COALESCE(last_success_at, created_at)
- it is manually disabled
- the operator withdraws it
- it repeatedly fails health expectations
Inactive relayers:
- do not appear in official frontend discovery
- do not count toward the 10 live slots
- do not require deletion from the database
This keeps slot turnover automatic without forcing a manual review process.

---
3. UX Scope
3.1 Route
parly.fi/relayer
Single page only.
No multi-step flow.
No modal.
No hidden form.
The form is open on page load.
3.2 Audience
This page is for:
- relayer operators
- advanced users
- third-party infrastructure participants
It is not intended for ordinary end users.
3.3 Design style
Ultra lean.
Simpler than Verify.
One narrow content column.
Minimal visual noise.
Low explanation density.
High trust, operator-grade tone.

---
4. Page Layout
Top to bottom only:
1. Eyebrow
2. Title
3. Subtitle
4. Tiny status pills
5. Docs buttons
6. Open form
7. Inline success / error alerts
8. Registry rules block
9. Tiny footer docs note
Nothing else.

---
5. Exact Page Copy
5.1 Header
Eyebrow
Parly Relay Registry
Title
Register a Relayer
Subtitle
This page is for relayer operators only. Read the relayer docs before registering your execution address and public key.

---
5.2 Status Pills
Two tiny pills only.
Pill 1
Live Relayers: 4 / 10
Pill 2
Registry: Open
When full:
Pill 1
Live Relayers: 10 / 10
Pill 2
Registry: Full

---
5.3 Docs Actions
Three buttons maximum.
- [ Relayer Docs ]
- [ Operator Quickstart ]
- [ SDK / MCP Docs ]
Supporting microcopy:
Need setup help first? Read the docs before registering.

---
6. Registration Form
6.1 Section Heading
Title
Relayer Registration
Copy
Enter the relayer execution address and the Base64 public key generated from your relayer setup.

---
6.2 Fields
Field 1
Label
Relayer Execution Address
Placeholder
0x...
Field 2
Label
Relayer Public Key
Placeholder
BASE64_X25519_PUBLIC_KEY

---
6.3 Action
Primary button:
[ Connect Wallet & Sign Registration ]
Secondary button:
[ Clear ]
Button rule
If registry is full, the primary button must be:
- disabled
- visually muted
- non-clickable
Disabled text when full:
Registry Full

---
6.4 Required Statement
Single checkbox above the primary action:
I confirm this relayer is independently operated, the submitted details are accurate, and I understand registration does not create protocol exclusivity or guarantee user flow.
The primary button remains disabled until checked.

---
7. UX States
7.1 Default State
On page load, the frontend must fetch:
GET /api/relayers
That backend response is the canonical source of truth for:
- liveCount
- maxCount
- registryOpen
The form is visible immediately.
The primary CTA is enabled only if:
- registryOpen = true
- execution address is valid
- public key is valid
- checkbox is checked
- request is not in flight
If registryOpen = false, the page must enter full-state immediately and the primary CTA must remain disabled.
7.2 Loading State
Primary button text:
Registering...
All fields disabled while request is in flight.
7.3 Success State
Inline success alert above form:
Relayer registered successfully
Your relayer is now active in the official Parly registry.
7.4 Registry Full State
Inline warning alert above form:
Official registry is currently full
New registrations reopen when an active slot becomes available.
Primary button text:
Registry Full
Disabled.
7.5 Validation Error States
Invalid address
Invalid relayer execution address
Enter a valid EVM address.
Invalid public key
Invalid relayer public key
Enter a valid Base64 relayer public key.
Duplicate address
Relayer already exists
This execution address is already registered.
Duplicate public key
Public key already exists
This public key is already registered.
Signature failed
Signature verification failed
Reconnect the correct execution wallet and try again.
Rate limited
Too many attempts
Please try again later.
Generic failure
Registration failed
We could not process this request right now.

---
8. Registry Rules Block
Place directly below the form.
Title
Registry Rules
Copy
- Official frontend discovery is capped at 10 active relayers.
- Registration is automatic if validation passes and a slot is available.
- Duplicate, invalid, or unavailable relayers may be rejected automatically.
- Inactive relayers are removed from live official discovery.
- Protocol participation remains open even if a relayer is not listed here.

---
9. Footer Note
Tiny single line.
Need setup instructions first? Read the relayer docs before registering.
Link text:
Relayer Docs

---
10. Technical Behavior
10.1 Current V16.9.9 baseline
Today, the latest runbook still expects:
- relayer operator runs pnpm keygen:box
- receives execution address and Base64 box public key
- app discovery comes from NEXT_PUBLIC_RELAYER_REGISTRY
- frontend parses entries using parseRelayerRegistry(...)
This spec replaces that env-only discovery path with a runtime-backed official registry.
10.2 Runtime registry design
Frontend must no longer depend only on:
- NEXT_PUBLIC_RELAYER_REGISTRY
Instead, the official app must fetch verified live relayers from backend at runtime.
Backend is the canonical official registry source.
That means:
- backend returns verified + active relayers only
- backend returns liveCount, maxCount, and registryOpen
- frontend uses returned execution address + public key for relay targeting and encryption
- frontend button state must come from backend registry state, not local guessing
Production rule
- runtime registry is canonical in production
- env registry may remain as optional local/dev fallback only

---
11. Backend Data Model
11.1 Table: relayers
Use one table only.
Fields
- id
- execution_address
- public_key_b64
- status
- created_at
- updated_at
- last_success_at
- last_seen_at
- disabled_reason
Status values
- verified
- inactive
- rejected
11.2 SQL schema
CREATE TABLE IF NOT EXISTS relayers (
  id BIGSERIAL PRIMARY KEY,
  execution_address VARCHAR(42) NOT NULL UNIQUE,
  public_key_b64 TEXT NOT NULL UNIQUE,
  status VARCHAR(16) NOT NULL CHECK (status IN ('verified', 'inactive', 'rejected')),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  last_success_at TIMESTAMP,
  last_seen_at TIMESTAMP,
  disabled_reason TEXT
);

CREATE INDEX IF NOT EXISTS relayers_status_idx ON relayers(status);
CREATE INDEX IF NOT EXISTS relayers_last_success_idx ON relayers(last_success_at);

---
12. Registration Validation Rules
These must all run server-side.
12.1 Registry capacity check
Registry capacity enforcement must happen in two places.
A. Page-load / pre-submit check
Frontend reads registryOpen from:
GET /api/relayers
If registryOpen = false:
- show full-state alert
- disable primary CTA
- block registration attempt in UI
B. Final server-side enforcement
Backend must perform a final capacity check inside the registration transaction before insert.
This must be atomic.
Implementation rule:
- acquire a transaction-scoped advisory lock before counting and inserting
- count current verified relayers
- if count >= 10, reject with REGISTRY_FULL
- else insert the new relayer as verified
Recommended PostgreSQL pattern:
SELECT pg_advisory_xact_lock(90010);
Then inside the same transaction:
SELECT COUNT(*) FROM relayers WHERE status = 'verified';
If count >= 10:
- rollback
- return REGISTRY_FULL
Else:
- insert row
- commit
12.2 Address validation
- required
- valid EVM address
- normalized to lowercase
12.3 Public key validation
- required
- valid Base64
- must decode successfully
- must match expected X25519 public key byte length
12.4 Duplicate address check
If execution address already exists in any relayer row:
- reject
12.5 Duplicate public key check
If public key already exists in any relayer row:
- reject
12.6 Signature ownership proof
This is required.
The registering wallet must prove control of the submitted execution address.
The flow is:
1. frontend requests nonce from backend
2. backend creates a nonce bound to:
  - execution address
  - domain
  - issuance time
  - expiry time
3. backend stores the nonce server-side
4. frontend asks the wallet to sign the returned message
5. backend verifies:
  - nonce exists
  - nonce is unused
  - nonce is unexpired
  - signer address equals submitted execution address
6. nonce is consumed immediately after successful verification
Nonce rules:
- single-use only
- expires after 10 minutes
- bound to the submitted execution address
- invalid once consumed
Without this, anyone could register someone else’s relayer.
12.7 Checkbox confirmation
Must be checked or request is blocked.
12.8 Rate limit
Light rate limiting only.
Apply limits:
- by IP address
- by execution address
Recommended defaults:
- max 5 registration attempts per 24 hours per IP
- max 3 registration attempts per 24 hours per execution address
These limits apply to both init and confirm routes.

---
13. Registration Signature Contract
13.1 Init endpoint
POST /api/relayers/register/init
Request
{
  "executionAddress": "0x..."
}
Backend behavior
- validate execution address format
- check rate limit
- create nonce
- set expiry = now + 10 minutes
- store nonce server-side as single-use
- bind nonce to execution address
Response
{
  "nonce": "random_nonce",
  "expiresAt": "2026-03-27T12:00:00.000Z",
  "message": "Parly Relayer Registration\nExecution Address: 0x...\nNonce: abc123\nDomain: parly.fi\nExpires At: 2026-03-27T12:00:00.000Z"
}
13.2 Confirm endpoint
POST /api/relayers/register/confirm
Request
{
  "executionAddress": "0x...",
  "publicKeyB64": "BASE64_X25519_PUBLIC_KEY",
  "signature": "0x...",
  "nonce": "random_nonce"
}
Backend behavior
Server must perform, in this order:
1. rate limit check
2. execution address validation
3. public key validation
4. duplicate address check
5. duplicate public key check
6. nonce lookup and expiry check
7. signature verification
8. atomic capacity enforcement inside transaction
9. insert relayer as verified
10. consume nonce
11. return success
Success response
{
  "success": true,
  "status": "verified"
}
Failure responses
{
  "success": false,
  "error": "REGISTRY_FULL"
}
{
  "success": false,
  "error": "INVALID_SIGNATURE"
}
{
  "success": false,
  "error": "DUPLICATE_ADDRESS"
}
{
  "success": false,
  "error": "DUPLICATE_PUBLIC_KEY"
}
{
  "success": false,
  "error": "INVALID_PUBLIC_KEY"
}
{
  "success": false,
  "error": "RATE_LIMITED"
}

---
14. Frontend API Consumption
14.1 Public registry endpoint
GET /api/relayers
Returns verified live relayers only.
A live relayer is defined as:
- status = 'verified'
This endpoint is also the canonical source of truth for page capacity state.
Response
{
  "items": [
    {
      "executionAddress": "0x...",
      "publicKeyB64": "BASE64_X25519_PUBLIC_KEY"
    }
  ],
  "liveCount": 4,
  "maxCount": 10,
  "registryOpen": true
}
registryOpen must be computed server-side as:
- liveCount < maxCount
14.2 Frontend registry replacement
Current V16.9.9 uses parseRelayerRegistry(raw) from env.
Replace this with:
- fetchRelayerRegistry()
- server-backed runtime list
- same RelayerRegistryEntry shape in frontend memory
That keeps the rest of the relay selection flow simple.

---
15. Automatic Inactive Handling
15.1 Rule
A verified relayer becomes inactive if:
- COALESCE(last_success_at, created_at) is older than 30 days
or
- operator is manually disabled
or
- health criteria fail repeatedly
This ensures that a newly registered relayer that never successfully relays does not occupy a slot forever.
15.2 Why last_success_at
Do not use “listed but zero total tx ever” as the only logic.
Use a real activity timestamp tied to successful relay execution.
15.3 Inactive job
Run a daily cron:
UPDATE relayers
SET
  status = 'inactive',
  updated_at = CURRENT_TIMESTAMP,
  disabled_reason = 'AUTO_INACTIVE_30D'
WHERE
  status = 'verified'
  AND COALESCE(last_success_at, created_at) < NOW() - INTERVAL '30 days';
15.4 Live slot reopening
Once inactive:
- relayer disappears from GET /api/relayers
- relayer no longer counts toward the 10 live slots
- registration opens again automatically if below 10

---
16. How last_success_at is updated
This must be technically connected end to end.
When a relayer successfully lands a confirmed private execution:
- relayer service updates its execution result
- relayer registry activity must also be updated
Two valid implementation paths are allowed.
Path A — Direct DB update from relayer service
After confirmed inclusion:
- set last_success_at = NOW()
- set last_seen_at = NOW()
Path B — Authenticated heartbeat endpoint
Relayer posts a success heartbeat to backend.
Recommended internal endpoint:
POST /api/relayers/heartbeat
Request
{
  "executionAddress": "0x...",
  "event": "SUCCESS",
  "timestamp": "2026-03-27T12:05:00.000Z",
  "signature": "0x..."
}
Signature message
Heartbeat must be signed by the relayer execution wallet.
Example message:
Parly Relayer Heartbeat
Execution Address: 0x...
Event: SUCCESS
Timestamp: 2026-03-27T12:05:00.000Z
Domain: parly.fi
Backend behavior
- verify signature
- confirm signer == submitted execution address
- reject stale timestamps older than 5 minutes
- if valid and event = SUCCESS:
  - set last_success_at = NOW()
  - set last_seen_at = NOW()
This endpoint must not accept unauthenticated success updates.

---
17. Production Integration Plan
17.1 Web app
Add:
- app/relayer/page.tsx
- app/api/relayers/route.ts
- app/api/relayers/register/init/route.ts
- app/api/relayers/register/confirm/route.ts
- optional app/api/relayers/heartbeat/route.ts
17.2 Frontend registry path
Replace env-only parser in relay discovery path with backend fetch.
Current logic uses env parse.
New logic:
- fetch backend registry
- return same shape:
  - executionAddress
  - publicKeyB64
17.3 Relayer service
Relayer remains otherwise unchanged:
- pnpm keygen:box
- pnpm start
- pnpm reconcile:pending:loop
These remain operator setup commands.
Required addition:
- on confirmed successful relay execution, the relayer must update official registry activity using either:
  - direct DB update, or
  - authenticated heartbeat endpoint
No registry activity update means inactive-slot turnover will not work correctly.

---
18. UI Logic
18.1 Button enable logic
Primary button is enabled only if:
- backend registryOpen = true
- valid execution address
- valid public key
- checkbox checked
- not currently submitting
Primary button is disabled if:
- backend registryOpen = false
- invalid execution address
- invalid public key
- checkbox unchecked
- request is in flight
When registryOpen = false:
- button text = Registry Full
- button is disabled
- warning alert is shown immediately from backend state
18.2 No extra screens
No modal.
No operator dashboard.
No moderation queue UI.
No internal review screen required.

---
19. Docs Requirements
This page depends on docs being available from day one.
Buttons must point to:
- /docs/relayers
- /docs/quickstart
- /sdk
The page itself should not try to teach relayer setup.

---
20. Release Policy
20.1 Official registry
Official app consumes only:
- verified
- active
- max 10
20.2 Protocol openness
Even if a relayer is not listed:
- protocol participation remains open
- SDK / custom frontend users can still target independent relayers directly
That stays aligned with the latest runbook’s “no monopoly relayer required for correctness” law.

---
21. Acceptance Criteria
This feature is only complete if all of these are true:
1. /relayer loads as a single-page open form.
2. Docs buttons are visible and work.
3. Valid address + valid key + valid signature = direct registration.
4. Duplicate address is rejected.
5. Duplicate public key is rejected.
6. Registry full state disables registration from backend-provided state.
7. GET /api/relayers returns verified live entries only.
8. Official frontend relay discovery can consume backend registry instead of env-only input.
9. Relayer activity updates last_success_at.
10. Inactive relayers stop counting toward the 10-slot cap.
11. Concurrent registrations cannot exceed 10 verified live relayers.
12. No manual review UI is required for launch.

---
22. Final Decision
This is the lean production model:
- single page
- open form on load
- no review queue
- no modal
- max 10 live relayers
- automatic validation
- automatic registration
- automatic inactive turnover
- backend-backed registry
- frontend discovery fed from backend
- docs-first operator UX
That is the cleanest professional version and it is technically grounded end to end.