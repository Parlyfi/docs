Parly.fi Phase 2 Master Document
Waitlist, Shield Mining, TGE & OFT Architecture Canonical All-in-One PRD + UI/UX + TRD + Runbook Single Source of Truth

---
1. Canonical product law
Phase 2 is the marketing, waitlist, referral, leaderboard, TGE snapshot, and OFT launch layer for Parly.
It is not the protocol.
It sits on top of the current Parly V16.9.9 system, which already fixes the launch surface around:
- Shield on Tempo
- Shield from supported spokes into Tempo
- Private same-chain payouts on Tempo
- Private cross-chain stablecoin-only payouts
- Verify portal
- Off-chain removable shield-mining analytics
- Agentic SDK
- MCP server
This means the architectural hierarchy is:
Protocol events → Indexer → Campaign Worker → Waitlist Profile Merge → Leaderboard → Snapshot → TGE / OFT UX
That hierarchy is now law.

---
Part 1 — Product Requirements Document (PRD)
1.1 Objective
The goal of Phase 2 is to aggressively grow Parly by combining:
- social onboarding
- identity anchoring
- referral virality
- real capital allocation
- psychological rank progression
- TGE readiness
- omnichain token mobility after launch
The campaign should feel premium, competitive, and bullish, but it must remain technically honest.
1.2 Core product thesis
A user’s final campaign standing is determined by a single metric:
Total $Parly Points = Waitlist Points + Shield Points
But those two point systems come from different truth sources:
- Waitlist Points come from social / referral / profile actions
- Shield Points come from real finalized protocol deposit attribution
They are merged only at the profile layer.
1.3 Canonical point laws
Waitlist Points
- X sign-in / follow gate: 0 points
- Discord linked: +10
- Permanent Tempo bond wallet set: +10
- Successful verified referral: +1 per human
Shield Points
- Formula: Deposited stable amount ÷ 100
- Example: 5,000 USDC = 50 Shield Points
- Award type: one-time per finalized deposit event
- Not recurring
- Not TVL duration
- Not daily mining
- Not balance-time accrual
- Not reduced by later spends
1.4 Omnichain equality rule
A user earns the same Shield Points whether they:
- deposit directly on Tempo
- or deposit from Ethereum, Arbitrum, Base, or BSC and finalize into Tempo
What matters is finalized Tempo-side deposit attribution, not source-chain intent alone.
1.5 Permanent wallet law
Each user has exactly one primary Tempo wallet.
That wallet is:
- the permanent bond wallet
- the root identity anchor
- the only TGE receiver
- the allocation export address
- the waitlist profile root
A user may link additional wallets, but those are secondary wallets only. They can contribute Shield Points, but they never receive independent TGE allocation.
1.6 Ghost depositor law
If a wallet earns Shield Points before the owner joins the waitlist, that wallet appears as a Ghost Depositor.
Ghosts:
- can appear on the leaderboard
- can show Shield Points
- do not receive airdrop allocation yet
- must still join the waitlist and set a primary Tempo wallet to become eligible
1.7 Unfollow law
If a user unfollows @ParlyFi, the account is not deleted.
Instead:
- status becomes inactive
- profile is frozen
- referral generation is blocked
- leaderboard remains historically attributable but the profile is not active
- the user is forced back through the Verification Gate on next session or next detected check
- once follow is restored and verification passes, the profile can reactivate
This freeze model is much cleaner than deleting points or wiping profile state.
1.8 Tier law
Tiering is frontend-only and based on total points:
- Bronze: 0–99
- Silver: 100–999
- Gold: 1,000–9,999
- Platinum: 10,000+
No tier changes protocol rights.

---
Part 2 — UI / UX Master Walkthrough
This section is now written as a real product flow, not a loose feature list.
The placement, state flow, copy hierarchy, and population rules below are the source of truth.

---
2.1 Global structure
Surface map
There are four major campaign surfaces:
1. Waitlist Landing Page
2. Verification Gate
3. Waitlist Dashboard
4. Global Leaderboard
5. TGE / OFT Bridge Surface when live
Hosting rule
Canonical implementation lives inside the existing web app as:
- parly.fi/waitlist
- parly.fi/waitlist/leaderboard
- parly.fi/waitlist/bridge
A dedicated domain such as waitlist.parly.fi may point to the same routes through host mapping or reverse proxy, but the codebase remains one web app.
That avoids unnecessary duplication while still letting marketing present it as a distinct surface.

---
2.2 Waitlist Landing Page
Placement
This is the public acquisition page. It is separate from the privacy dapp home.
Layout
Top to bottom:
1. Sticky nav
2. Hero
3. Trust strip
4. Campaign explainer
5. Points explainer
6. Leaderboard preview
7. FAQ
8. Final CTA
Nav
Left:
- Parly wordmark
Right:
- Leaderboard
- Docs
- X
- Primary CTA: [ Sign in with X ]
Hero
Eyebrow
Agentic Privacy Campaign
Headline
Join the Agentic Economy.
Subheadline
Secure your place in the Parly network, bind your Tempo identity, allocate capital across supported chains, and mine $Parly before TGE.
Primary CTA
[ Sign in with X ]
Secondary CTA
[ View Leaderboard ]
Live social proof row
Directly below the hero CTA row, show three pills:
- Top Tier: Platinum
- Recent Activity: 5,000 USDC shielded on Base
- Leaderboard Snapshot: Rank #14 just crossed Gold
This keeps the page alive without turning it into spammy noise.
Why Parly section
Three premium cards:
Card 1 — Verified social identity
Join through X, complete your profile, and unlock campaign progression with a permanent Tempo wallet bond.
Card 2 — Real capital recognition
Shield Points are awarded from real finalized deposits across Tempo and supported spoke routes.
Card 3 — TGE-ready destination
Your bonded Tempo wallet becomes the only wallet eligible for future allocation and claim readiness.
Points explainer strip
A single premium strip with two panels:
Waitlist Points
Discord, referrals, and permanent Tempo bond
Shield Points
One-time rewards from real finalized shield deposits
Footer CTA
Headline
Secure your profile before the leaderboard hardens.
CTA
[ Sign in with X ]

---
2.3 Verification Gate
This is the first protective state after sign-in.
Trigger
Any user who:
- signs in for the first time
- has not yet passed follower verification
- was previously marked inactive after unfollowing
- is in pending_audit and needs recheck
Layout
Centered card, strong block state.
State A — Passed
No special UI. Redirect to Dashboard.
State B — Not Following / Unfollowed
Title
Account Inactive
Body
To protect the integrity of the $Parly campaign and reduce sybil abuse, you must follow our official channel before your profile can remain active.
Primary CTA
[ Follow @ParlyFi on X ]
Secondary CTA
[ Verify Follow Status ]
Footnote
Your profile is frozen until verification succeeds again.
State C — Pending Audit
Use this when RapidAPI or the X provider is temporarily unavailable.
Title
Verification Delayed
Body
We could not confirm follow status right now. Your session can continue temporarily, but referral unlocks and campaign rewards remain subject to background re-verification.
Primary CTA
[ Continue to Dashboard ]
Secondary CTA
[ Try Verification Again ]
This is cleaner than pretending the user is fully verified.

---
2.4 Waitlist Dashboard — Pre-Bond State
This is the first real dashboard.
Placement
Top section:
- profile identity
- current status
- total points
- tier progress
Middle section:
- task stack
Bottom section:
- leaderboard CTA
- campaign explainer
Header
Left:
- avatar
- X handle
- status pill
Right:
- Tier badge
- Total $Parly points
Hero stats row
Three cards:
- Waitlist Points
- Shield Points
- Total $Parly
Pre-bond example:
- Waitlist Points: 0
- Shield Points: 0
- Total: 0
Progress bar
Label:
- Bronze Tier
Subcopy:
- Complete your setup to unlock your referral code and campaign visibility.
Task stack
Task 1 — X Connected
Status:
- Completed
Copy:
- Your social identity is connected.
Task 2 — Link Discord
Status:
- Pending or Complete
Primary line:
- Link Discord
Secondary line:
- Join the Parly Discord and claim +10 $Parly Points.
CTA:
- [ Link Discord ]
Task 3 — Set Permanent Tempo Bond
Status:
- Pending until completed
Title:
- Set Permanent Tempo Bond
Body:
- Bind your Tempo wallet to activate your profile, claim +10 points, and lock the only wallet eligible for your future TGE allocation.
CTA:
- [ Connect Tempo Wallet & Sign ]
Microcopy:
- This bond is permanent and should be the wallet you intend to use for claim and future ecosystem participation.

---
2.5 Permanent Tempo Bond Modal
Trigger
User clicks Connect Tempo Wallet & Sign
Modal structure
Top:
- title
- one short explanatory paragraph
- wallet preview
- bond warning block
- actions
Copy
Title
Set Permanent Tempo Bond
Body
Your bonded Tempo wallet becomes the permanent anchor for this profile. It unlocks referrals, activates your campaign profile, and becomes the only wallet eligible for TGE allocation.
Warning block title
Permanent Wallet Notice
Warning body
Do not continue unless this is the exact Tempo wallet you want bound to your Parly profile.
Actions
[ Cancel ]
[ Sign Permanent Bond ]
Success toast
Tempo bond set successfully
Success inline state
Task 3 becomes completed, and the dashboard upgrades into Active State.

---
2.6 Waitlist Dashboard — Active State
This is the main populated dashboard.
Header
Left:
- X avatar
- X handle
- active status pill
Right:
- Tier badge
- total points
Hero cards
1. Waitlist Points
2. Shield Points
3. Total $Parly
Example populated row
- Waitlist Points: 34
- Shield Points: 250
- Total: 284
- Tier: Silver
Card 1 — Referral Hub
Section Title
Referral Hub
Primary line
Your Agentic Network Code
Referral field
parly.fi/waitlist?ref=parly-user123
Substats
- Direct Referrals: 14
- Referral Points: 14
Actions
- [ Copy Link ]
- [ Share on X ]
Card 2 — Shield Mining Hub
Section Title
Shield Mining Hub
Primary line
Shield Points reflect real finalized capital allocated into Parly across Tempo and supported spokes.
Subline
Linked wallets automatically contribute to this profile once ownership is verified.
Stats row
- Linked Wallets: 3
- Last Sync: 4 min ago
- Latest Deposit Counted: 5,000 USDC on Base
Actions
- [ Link Secondary EVM Wallet ]
- [ View Leaderboard ]
Card 3 — Campaign Status
Section Title
Campaign Status
Show:
- Follow Status
- Discord Status
- Bond Status
- Snapshot Readiness
Example:
- X Follow: Verified
- Discord: Connected
- Tempo Bond: Locked
- Snapshot Readiness: Eligible if snapshot opened today

---
2.7 Unfollow Freeze Banner
This was missing in the previous rewrite and it matters.
Trigger
User returns after cron marked them inactive, or the login/session refresh detects is_following_x = false.
Placement
This appears as a full-width danger banner at the top of the dashboard.
Copy
Title
Profile Frozen
Body
Your profile is currently inactive because follow verification failed. Referral generation and campaign progression are paused until you follow @ParlyFi again and re-verify.
Actions
- [ Follow @ParlyFi ]
- [ Verify Follow Status ]
Dashboard behavior in this state
- referral code panel is blurred or locked
- new wallet linking is disabled
- point totals remain visible but styled as frozen
- leaderboard contribution remains historically visible but marked inactive
That captures the unfollow case properly.

---
2.8 Link Secondary Wallet Modal
Trigger
User clicks Link Secondary EVM Wallet
Placement
Drawer or modal from the right is ideal. This is not a full page.
Structure
1. explainer
2. connected-wallet state
3. chain hint selector
4. sign action
5. result state
Copy
Title
Link Secondary EVM Wallet
Body
Depositing from a different address on Ethereum, Arbitrum, Base, BSC, or another supported EVM wallet? Sign a zero-gas message below and merge its Shield Points into this master profile.
Constraint strip
A wallet can only be linked to one profile.
Fields
- Wallet Address
- Chain Hint
Actions
- [ Sign Link Message ]
- [ Cancel ]
Success state
Wallet linked successfully
Success subcopy
Future Shield Points from this wallet will merge into your master profile after indexed deposit sync.
Failure states
- Wallet already linked elsewhere
- Signature rejected
- Nonce expired
- Unsupported chain hint
- Same wallet as primary already bonded

---
2.9 Global Leaderboard
Placement
Standalone route.
Top strip
- title
- short explainer
- filters
- last sync timestamp
Copy
Title
Global Leaderboard
Subtitle
Ranked by total verified campaign strength across social progression and real shielded capital allocation.
Filter row
- [ All ]
- [ Platinum Only ]
- [ Ghost Depositors ]
Columns
- Rank
- Identity
- Tier
- Waitlist Points
- Shield Points
- Total $Parly
Identity rules
Verified profile:
- X handle first
- masked primary Tempo wallet in subline
Ghost profile:
- masked wallet only
- small “Ghost Depositor” pill
Row hover details
On hover or row expand:
- primary Tempo wallet if verified
- linked wallets count
- last counted deposit time
- profile status

---
2.10 TGE Readiness Card
Placement
Only shows once TGE phase is opened.
Copy
Title
TGE Readiness
Body
Your permanent Tempo bond wallet is the only wallet eligible to receive your campaign allocation at snapshot time.
Fields
- Primary Tempo Wallet
- Current Total Points
- Current Tier
- Snapshot Eligibility
- Snapshot Window
Actions
- [ Review Eligibility ]
- [ Learn About Claim ]

---
2.11 OFT Bridge Screen
Placement
Separate route, not mixed into the privacy dapp widget.
Structure
Single premium bridge card.
Copy
Headline
Omnichain Transfer
Subheadline
Move $PARLY between supported chains through the native Parly OFT transfer path.
Fields
- From
- To
- Amount
- Asset
Body copy
Parly uses native omnichain token transfer. Tokens are burned on the source chain and natively minted on the destination chain. One unified supply. No wrapped fragmentation.
CTA
[ Approve & Bridge ]
Supporting note
Estimated completion: 1–3 minutes depending on route and LayerZero finality.

---
Part 3 — Technical Requirements Document (TRD)
3.1 Canonical codebase layout
This is the full Phase 2 codebase map inside the existing V16.9.9 monorepo.
parly-fi-v16_9_9/
  apps/
    web/
      src/
        app/
          waitlist/
            page.tsx
            leaderboard/
              page.tsx
            bridge/
              page.tsx
          api/
            waitlist/
              me/route.ts
              verify-follow/route.ts
              bond-wallet/
                init/route.ts
                confirm/route.ts
              link-wallet/
                init/route.ts
                confirm/route.ts
              leaderboard/route.ts
              admin/
                export-snapshot/route.ts
        components/
          waitlist/
            WaitlistHero.tsx
            VerificationGate.tsx
            WaitlistDashboard.tsx
            ReferralHub.tsx
            ShieldMiningHub.tsx
            SecondaryWalletModal.tsx
            TgeReadinessCard.tsx
            LeaderboardTable.tsx
            GhostBadge.tsx
        server/
          auth/
            better-auth.ts
          db/
            pool.ts
            migrations/
              001_users.sql
              002_secondary_wallets.sql
              003_wallet_link_nonces.sql
              004_ghost_depositors.sql
              005_tge_snapshots.sql
              006_tge_snapshot_allocations.sql
              007_leaderboard_materialized_view.sql
          services/
            follow-verification.ts
            waitlist-profile.ts
            wallet-linking.ts
            shield-points-merge.ts
            leaderboard.ts
            snapshot-exporter.ts
            google-sheet-sync.ts
          types/
            waitlist.ts
    campaign-worker/
      src/
        schema.sql
        sync.ts
        merge-ghosts.ts
        cron.ts
    indexer/
      src/
        api/
          public-dashboard.ts
          provenance.ts
    mcp-server/
      src/
        ...
  packages/
    contracts/
      ...
    sdk/
      ...
3.2 Why this layout
This keeps the Phase 2 surface inside the same app and same repo, while respecting the current V16.9.9 service boundaries:
- apps/web remains the user-facing surface
- apps/campaign-worker remains analytics-only
- apps/indexer remains public chain-data layer
- the waitlist does not become protocol code
That preserves the architecture already laid out in V16.9.9.

---
3.3 Canonical database ownership
Railway PostgreSQL owns
- users
- secondary wallets
- wallet link nonces
- ghost depositors
- TGE snapshots
- snapshot allocations
- materialized leaderboard projection
- admin sync metadata
Campaign worker database rows own
- wallet-scoped shield mining analytics only
Indexer owns
- public chain facts only
Protocol contracts own
- actual deposits, leaves, notes, spends, payouts

---
3.4 Exact migrations
001_users.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE IF NOT EXISTS users (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),

  x_user_id VARCHAR(255) UNIQUE NOT NULL,
  x_handle VARCHAR(255) UNIQUE NOT NULL,
  x_account_age_days INT NOT NULL,
  is_following_x BOOLEAN NOT NULL DEFAULT FALSE,

  discord_id VARCHAR(255) UNIQUE,
  in_discord_server BOOLEAN NOT NULL DEFAULT FALSE,

  primary_tempo_wallet VARCHAR(42) UNIQUE,
  primary_tempo_wallet_bonded_at TIMESTAMP,

  referral_code VARCHAR(64) UNIQUE,
  referred_by_code VARCHAR(64),

  social_points INT NOT NULL DEFAULT 0,
  referral_points INT NOT NULL DEFAULT 0,

  status VARCHAR(32) NOT NULL DEFAULT 'pending',
  flagged_for_recheck BOOLEAN NOT NULL DEFAULT FALSE,
  follow_status_checked_at TIMESTAMP,

  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
002_secondary_wallets.sql
CREATE TABLE IF NOT EXISTS secondary_wallets (
  wallet_address VARCHAR(42) PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  chain_hint VARCHAR(32),
  linked_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  link_signature_digest VARCHAR(66) UNIQUE NOT NULL
);
003_wallet_link_nonces.sql
CREATE TABLE IF NOT EXISTS wallet_link_nonces (
  wallet_address VARCHAR(42) PRIMARY KEY,
  nonce VARCHAR(128) NOT NULL,
  expires_at TIMESTAMP NOT NULL,
  consumed_at TIMESTAMP
);
004_ghost_depositors.sql
CREATE TABLE IF NOT EXISTS ghost_depositors (
  wallet_address VARCHAR(42) PRIMARY KEY,
  shield_points_micro NUMERIC(78, 0) NOT NULL DEFAULT 0,
  total_deposited_base_units NUMERIC(78, 0) NOT NULL DEFAULT 0,
  asset_breakdown JSONB NOT NULL DEFAULT '{}'::jsonb,
  first_seen_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  last_seen_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
005_tge_snapshots.sql
CREATE TABLE IF NOT EXISTS tge_snapshots (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  snapshot_name VARCHAR(128) UNIQUE NOT NULL,
  snapshot_status VARCHAR(32) NOT NULL DEFAULT 'draft',
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  finalized_at TIMESTAMP
);
006_tge_snapshot_allocations.sql
CREATE TABLE IF NOT EXISTS tge_snapshot_allocations (
  snapshot_id UUID NOT NULL REFERENCES tge_snapshots(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  primary_tempo_wallet VARCHAR(42) NOT NULL,
  waitlist_points INT NOT NULL,
  shield_points_micro NUMERIC(78, 0) NOT NULL,
  total_points_micro NUMERIC(78, 0) NOT NULL,
  allocation_units NUMERIC(78, 0) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (snapshot_id, user_id)
);
007_leaderboard_materialized_view.sql
DROP MATERIALIZED VIEW IF EXISTS leaderboard_profiles;

CREATE MATERIALIZED VIEW leaderboard_profiles AS
SELECT
  u.id AS user_id,
  u.x_handle,
  u.primary_tempo_wallet,
  u.status,
  u.social_points,
  u.referral_points,
  (u.social_points + u.referral_points) AS waitlist_points,
  0::NUMERIC(78,0) AS shield_points_micro,
  0::NUMERIC(78,0) AS total_points_micro,
  CURRENT_TIMESTAMP AS refreshed_at
FROM users u;

CREATE UNIQUE INDEX IF NOT EXISTS leaderboard_profiles_user_id_idx
  ON leaderboard_profiles (user_id);

---
3.5 Existing wallet-level shield mining truth
V16.9.9 already defines shield_wallet_points as the wallet-level analytics table, with:
- wallet_address
- total_deposited_base_units
- shield_points_micro
- asset_breakdown
- updated_at
That is the only canonical wallet-level Shield Points store.
The waitlist must read from it. It must not reimplement a competing points engine.

---
3.6 Endpoint contracts
These are the exact API contracts.
POST /api/waitlist/verify-follow
Request:
{
  "userId": "uuid",
  "xHandle": "parly_user"
}
Response:
{
  "success": true,
  "status": "active",
  "followVerified": true,
  "checkedAt": "2026-03-26T10:00:00.000Z"
}
Fallback response:
{
  "success": true,
  "status": "pending_audit",
  "followVerified": false,
  "note": "Provider unavailable. Background recheck scheduled."
}
POST /api/waitlist/bond-wallet/init
Request:
{
  "walletAddress": "0x...",
  "chainId": 4217
}
Response:
{
  "nonce": "random_nonce",
  "expiresAt": "2026-03-26T10:10:00.000Z",
  "message": "Parly Waitlist Permanent Tempo Bond\nUser: ...\nWallet: ...\nNonce: ..."
}
POST /api/waitlist/bond-wallet/confirm
Request:
{
  "walletAddress": "0x...",
  "signature": "0x...",
  "nonce": "random_nonce"
}
Response:
{
  "success": true,
  "primaryTempoWallet": "0x...",
  "awardedSocialPoints": 10,
  "status": "active"
}
POST /api/waitlist/link-wallet/init
Request:
{
  "walletAddress": "0x...",
  "chainHint": "base"
}
Response:
{
  "nonce": "random_nonce",
  "expiresAt": "2026-03-26T10:10:00.000Z",
  "message": "Parly Link Secondary Wallet\nUser: ...\nWallet: ...\nNonce: ..."
}
POST /api/waitlist/link-wallet/confirm
Request:
{
  "walletAddress": "0x...",
  "chainHint": "base",
  "signature": "0x...",
  "nonce": "random_nonce"
}
Response:
{
  "success": true,
  "walletAddress": "0x...",
  "linkedAt": "2026-03-26T10:02:00.000Z"
}
GET /api/waitlist/me
Response:
{
  "profile": {
    "userId": "uuid",
    "xHandle": "parly_user",
    "status": "active",
    "primaryTempoWallet": "0x...",
    "linkedWallets": [
      "0x...",
      "0x..."
    ],
    "waitlistPoints": 34,
    "shieldPointsMicro": "250000000",
    "shieldPointsDisplay": "250.000000",
    "totalPointsDisplay": "284.000000",
    "tier": "Silver"
  }
}
GET /api/waitlist/leaderboard
Response:
{
  "rows": [
    {
      "rank": 1,
      "identityType": "verified",
      "xHandle": "parly_user",
      "wallet": "0x...",
      "tier": "Platinum",
      "waitlistPoints": 110,
      "shieldPointsDisplay": "1400.000000",
      "totalPointsDisplay": "1510.000000",
      "status": "active"
    }
  ],
  "ghostRows": [
    {
      "rank": 97,
      "identityType": "ghost",
      "wallet": "0xabc...123",
      "tier": "Gold",
      "waitlistPoints": 0,
      "shieldPointsDisplay": "350.000000",
      "totalPointsDisplay": "350.000000",
      "status": "ghost"
    }
  ],
  "lastSyncedAt": "2026-03-26T10:00:00.000Z"
}
POST /api/waitlist/admin/export-snapshot
Request:
{
  "snapshotName": "tge_phase_1"
}
Response:
{
  "success": true,
  "snapshotId": "uuid",
  "allocationCount": 12345
}

---
3.7 Exact cron responsibilities
There are four cron responsibilities.
Cron 1 — Campaign worker shield sync
Frequency:
- every 10 minutes
Responsibility:
- fetch indexed deposit events
- aggregate by depositor wallet
- update shield_wallet_points
- never mutate user profiles
Cron 2 — Follower recheck worker
Frequency:
- every 6 hours
Responsibility:
- sample active users
- reverify follow status
- mark unfollowed users inactive
- mark provider failures pending_audit
- never delete user points
Cron 3 — Ghost merge / leaderboard refresh
Frequency:
- every 5 minutes
Responsibility:
- read shield_wallet_points
- map wallet rows to primary or secondary wallets
- sum profile totals
- upsert ghost depositors
- refresh leaderboard_profiles
Cron 4 — Google Sheets admin sync
Frequency:
- every 15 minutes
Responsibility:
- read merged leaderboard output
- update admin sheet
- never act as source of truth

---
3.8 Exact merge algorithm
This is now canonical.
Inputs
- users
- secondary_wallets
- shield_wallet_points
- ghost_depositors
Algorithm
1. Load all users
2. Build wallet ownership map:
  - primary_tempo_wallet -> user_id
  - each secondary_wallet -> user_id
3. Load all shield wallet rows
4. For each shield wallet row:
  - if wallet is owned by a user, add its shield_points_micro to that user
  - if wallet is not owned, upsert ghost_depositors
5. For each user:
  - compute waitlist_points = social_points + referral_points
  - compute total_points_micro = waitlist_points_as_micro + shield_points_micro
  - assign tier
6. Refresh materialized leaderboard rows
7. Return API-ready sorted leaderboard
Hard rules
- one wallet cannot belong to two profiles
- unmatched wallets remain ghosts until explicitly linked
- primary Tempo wallet is the only exportable snapshot address

---
3.9 Exact snapshot exporter steps
1. Admin triggers snapshot creation
2. System creates a new row in tge_snapshots
3. System reads merged leaderboard profiles
4. System filters only eligible active users with a primary Tempo wallet
5. System computes allocation units from total points
6. System writes rows into tge_snapshot_allocations
7. System finalizes snapshot
8. Export utility produces:
  - JSON file for Merkle generation
  - CSV audit file
  - operator summary
Allocation rule
Only primary_tempo_wallet is written to snapshot export rows.

---
Part 4 — Plug-and-Play Runbook
This is the implementation section.

---
4.1 BetterAuth configuration contract
Because BetterAuth package surface can vary by pinned version, this section defines the required behavior contract and a reference implementation shape.
File
apps/web/src/server/auth/better-auth.ts
import { Pool } from "pg";
// Replace the import below with the exact BetterAuth import for the pinned package version.
import { betterAuth } from "better-auth";

const pool = new Pool({
  connectionString: process.env.RAILWAY_DATABASE_URL
});

export const auth = betterAuth({
  database: {
    provider: "postgres",
    pool
  },
  socialProviders: {
    twitter: {
      clientId: process.env.TWITTER_CLIENT_ID!,
      clientSecret: process.env.TWITTER_CLIENT_SECRET!,
      profile(profile: any) {
        const createdAt = new Date(profile.created_at).getTime();
        const ageInDays = Math.floor((Date.now() - createdAt) / (1000 * 60 * 60 * 24));

        return {
          id: profile.id_str || profile.id,
          username: profile.screen_name || profile.username,
          accountAgeDays: ageInDays
        };
      }
    },
    discord: {
      clientId: process.env.DISCORD_CLIENT_ID!,
      clientSecret: process.env.DISCORD_CLIENT_SECRET!,
      scope: ["identify", "guilds"]
    }
  },
  hooks: {
    async onUserCreated(ctx: any) {
      const { user, profile } = ctx;

      await pool.query(
        `
        INSERT INTO users (
          x_user_id,
          x_handle,
          x_account_age_days,
          status
        )
        VALUES ($1, $2, $3, 'pending')
        ON CONFLICT (x_user_id)
        DO NOTHING
        `,
        [
          user.id,
          user.username,
          profile.accountAgeDays ?? 0
        ]
      );
    }
  }
});
Required behavior
- create user row on first X login
- store x_user_id
- store x_handle
- store x_account_age_days
- default status = pending
- do not grant active status until follow verification passes

---
4.2 RapidAPI follower verification and state management
File
apps/web/src/app/api/waitlist/verify-follow/route.ts
import { NextResponse } from "next/server";
import { Pool } from "pg";

const pool = new Pool({
  connectionString: process.env.RAILWAY_DATABASE_URL
});

export async function POST(req: Request) {
  const { userId, xHandle } = await req.json();

  try {
    const res = await fetch(
      "https://twitter-x-api.p.rapidapi.com/user/followers?username=ParlyFi",
      {
        method: "GET",
        headers: {
          "X-RapidAPI-Key": process.env.RAPIDAPI_KEY!,
          "X-RapidAPI-Host": "twitter-x-api.p.rapidapi.com"
        }
      }
    );

    if (!res.ok) throw new Error(`RapidAPI failure: ${res.status}`);

    const data = await res.json();
    const followers = Array.isArray(data.followers) ? data.followers : [];
    const isFollowing = followers.some(
      (f: any) => String(f.screen_name || "").toLowerCase() === String(xHandle).toLowerCase()
    );

    await pool.query(
      `
      UPDATE users
      SET
        is_following_x = $2,
        status = $3,
        follow_status_checked_at = NOW(),
        flagged_for_recheck = FALSE,
        updated_at = NOW()
      WHERE id = $1
      `,
      [userId, isFollowing, isFollowing ? "active" : "inactive"]
    );

    return NextResponse.json({
      success: isFollowing,
      status: isFollowing ? "active" : "inactive",
      followVerified: isFollowing,
      checkedAt: new Date().toISOString()
    });
  } catch (error) {
    await pool.query(
      `
      UPDATE users
      SET
        status = 'pending_audit',
        flagged_for_recheck = TRUE,
        updated_at = NOW()
      WHERE id = $1
      `,
      [userId]
    );

    return NextResponse.json({
      success: true,
      status: "pending_audit",
      followVerified: false,
      note: "Provider unavailable. Background recheck scheduled."
    });
  }
}
State law
- active if following
- inactive if not following
- pending_audit if provider cannot confirm
This now clearly captures the unfollow/reactivation lifecycle.

---
4.3 Permanent bond wallet init / confirm
File
apps/web/src/app/api/waitlist/bond-wallet/init/route.ts
import { randomBytes } from "crypto";
import { NextResponse } from "next/server";
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.RAILWAY_DATABASE_URL });

export async function POST(req: Request) {
  const { walletAddress, chainId, userId } = await req.json();

  if (Number(chainId) !== 4217) {
    return NextResponse.json({ success: false, error: "Tempo chain required" }, { status: 400 });
  }

  const nonce = randomBytes(16).toString("hex");
  const expiresAt = new Date(Date.now() + 10 * 60 * 1000);

  await pool.query(
    `
    INSERT INTO wallet_link_nonces(wallet_address, nonce, expires_at)
    VALUES($1, $2, $3)
    ON CONFLICT(wallet_address)
    DO UPDATE SET nonce = EXCLUDED.nonce, expires_at = EXCLUDED.expires_at, consumed_at = NULL
    `,
    [walletAddress.toLowerCase(), nonce, expiresAt]
  );

  const message =
    `Parly Waitlist Permanent Tempo Bond\n` +
    `User: ${userId}\n` +
    `Wallet: ${walletAddress.toLowerCase()}\n` +
    `ChainId: ${chainId}\n` +
    `Nonce: ${nonce}\n` +
    `ExpiresAt: ${expiresAt.toISOString()}`;

  return NextResponse.json({
    nonce,
    expiresAt: expiresAt.toISOString(),
    message
  });
}
File
apps/web/src/app/api/waitlist/bond-wallet/confirm/route.ts
import { NextResponse } from "next/server";
import { Pool } from "pg";
// Replace with viem or your chosen EVM signature verification helper.
import { verifyMessage } from "viem";

const pool = new Pool({ connectionString: process.env.RAILWAY_DATABASE_URL });

export async function POST(req: Request) {
  const { userId, walletAddress, signature, nonce, message } = await req.json();

  const { rows } = await pool.query(
    `SELECT nonce, expires_at, consumed_at FROM wallet_link_nonces WHERE wallet_address = $1`,
    [walletAddress.toLowerCase()]
  );

  if (!rows[0]) {
    return NextResponse.json({ success: false, error: "Nonce missing" }, { status: 400 });
  }

  if (rows[0].consumed_at) {
    return NextResponse.json({ success: false, error: "Nonce already consumed" }, { status: 400 });
  }

  if (new Date(rows[0].expires_at).getTime() < Date.now()) {
    return NextResponse.json({ success: false, error: "Nonce expired" }, { status: 400 });
  }

  const valid = await verifyMessage({
    address: walletAddress,
    message,
    signature
  });

  if (!valid) {
    return NextResponse.json({ success: false, error: "Invalid signature" }, { status: 400 });
  }

  await pool.query("BEGIN");

  try {
    await pool.query(
      `
      UPDATE users
      SET
        primary_tempo_wallet = $2,
        primary_tempo_wallet_bonded_at = NOW(),
        social_points = social_points + 10,
        updated_at = NOW()
      WHERE id = $1
      `,
      [userId, walletAddress.toLowerCase()]
    );

    await pool.query(
      `UPDATE wallet_link_nonces SET consumed_at = NOW() WHERE wallet_address = $1`,
      [walletAddress.toLowerCase()]
    );

    await pool.query("COMMIT");

    return NextResponse.json({
      success: true,
      primaryTempoWallet: walletAddress.toLowerCase(),
      awardedSocialPoints: 10,
      status: "active"
    });
  } catch (e) {
    await pool.query("ROLLBACK");
    throw e;
  }
}

---
4.4 Secondary wallet link init / confirm
File
apps/web/src/app/api/waitlist/link-wallet/init/route.ts
import { randomBytes } from "crypto";
import { NextResponse } from "next/server";
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.RAILWAY_DATABASE_URL });

export async function POST(req: Request) {
  const { walletAddress, chainHint, userId } = await req.json();

  const nonce = randomBytes(16).toString("hex");
  const expiresAt = new Date(Date.now() + 10 * 60 * 1000);

  await pool.query(
    `
    INSERT INTO wallet_link_nonces(wallet_address, nonce, expires_at)
    VALUES($1, $2, $3)
    ON CONFLICT(wallet_address)
    DO UPDATE SET nonce = EXCLUDED.nonce, expires_at = EXCLUDED.expires_at, consumed_at = NULL
    `,
    [walletAddress.toLowerCase(), nonce, expiresAt]
  );

  const message =
    `Parly Link Secondary Wallet\n` +
    `User: ${userId}\n` +
    `Wallet: ${walletAddress.toLowerCase()}\n` +
    `ChainHint: ${chainHint}\n` +
    `Nonce: ${nonce}\n` +
    `ExpiresAt: ${expiresAt.toISOString()}`;

  return NextResponse.json({
    nonce,
    expiresAt: expiresAt.toISOString(),
    message
  });
}
File
apps/web/src/app/api/waitlist/link-wallet/confirm/route.ts
import { createHash } from "crypto";
import { NextResponse } from "next/server";
import { Pool } from "pg";
import { verifyMessage } from "viem";

const pool = new Pool({ connectionString: process.env.RAILWAY_DATABASE_URL });

export async function POST(req: Request) {
  const { userId, walletAddress, chainHint, signature, nonce, message } = await req.json();

  const normalizedWallet = String(walletAddress).toLowerCase();

  const nonceRow = await pool.query(
    `SELECT nonce, expires_at, consumed_at FROM wallet_link_nonces WHERE wallet_address = $1`,
    [normalizedWallet]
  );

  if (!nonceRow.rows[0]) {
    return NextResponse.json({ success: false, error: "Nonce missing" }, { status: 400 });
  }

  if (nonceRow.rows[0].consumed_at) {
    return NextResponse.json({ success: false, error: "Nonce already consumed" }, { status: 400 });
  }

  if (new Date(nonceRow.rows[0].expires_at).getTime() < Date.now()) {
    return NextResponse.json({ success: false, error: "Nonce expired" }, { status: 400 });
  }

  const sigValid = await verifyMessage({
    address: normalizedWallet as `0x${string}`,
    message,
    signature
  });

  if (!sigValid) {
    return NextResponse.json({ success: false, error: "Invalid signature" }, { status: 400 });
  }

  const existingUser = await pool.query(
    `
    SELECT id FROM users
    WHERE primary_tempo_wallet = $1
    `,
    [normalizedWallet]
  );

  if (existingUser.rowCount) {
    return NextResponse.json(
      { success: false, error: "Wallet already bound as a primary wallet" },
      { status: 409 }
    );
  }

  const digest = "0x" + createHash("sha256").update(message + signature).digest("hex");

  await pool.query("BEGIN");

  try {
    await pool.query(
      `
      INSERT INTO secondary_wallets(wallet_address, user_id, chain_hint, link_signature_digest)
      VALUES($1, $2, $3, $4)
      `,
      [normalizedWallet, userId, chainHint, digest]
    );

    await pool.query(
      `UPDATE wallet_link_nonces SET consumed_at = NOW() WHERE wallet_address = $1`,
      [normalizedWallet]
    );

    await pool.query("COMMIT");

    return NextResponse.json({
      success: true,
      walletAddress: normalizedWallet,
      linkedAt: new Date().toISOString()
    });
  } catch (e: any) {
    await pool.query("ROLLBACK");
    if (String(e.message || "").includes("duplicate")) {
      return NextResponse.json(
        { success: false, error: "Wallet already linked to another profile" },
        { status: 409 }
      );
    }
    throw e;
  }
}

---
4.5 Shield-points merge service
File
apps/web/src/server/services/shield-points-merge.ts
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.RAILWAY_DATABASE_URL });

const MICRO = 1_000_000n;

function toBigIntSafe(value: unknown): bigint {
  return BigInt(String(value ?? "0"));
}

export async function rebuildLeaderboardProjection() {
  const [usersRes, secondaryRes, walletPointsRes] = await Promise.all([
    pool.query(`SELECT * FROM users`),
    pool.query(`SELECT * FROM secondary_wallets`),
    pool.query(`SELECT * FROM shield_wallet_points`)
  ]);

  const users = usersRes.rows;
  const secondary = secondaryRes.rows;
  const walletRows = walletPointsRes.rows;

  const ownerByWallet = new Map<string, string>();

  for (const user of users) {
    if (user.primary_tempo_wallet) {
      ownerByWallet.set(String(user.primary_tempo_wallet).toLowerCase(), String(user.id));
    }
  }

  for (const row of secondary) {
    ownerByWallet.set(String(row.wallet_address).toLowerCase(), String(row.user_id));
  }

  const shieldByUser = new Map<string, bigint>();

  for (const row of walletRows) {
    const wallet = String(row.wallet_address).toLowerCase();
    const points = toBigIntSafe(row.shield_points_micro);
    const owner = ownerByWallet.get(wallet);

    if (!owner) {
      await pool.query(
        `
        INSERT INTO ghost_depositors(wallet_address, shield_points_micro, total_deposited_base_units, asset_breakdown, last_seen_at)
        VALUES($1, $2, $3, $4, NOW())
        ON CONFLICT(wallet_address)
        DO UPDATE SET
          shield_points_micro = EXCLUDED.shield_points_micro,
          total_deposited_base_units = EXCLUDED.total_deposited_base_units,
          asset_breakdown = EXCLUDED.asset_breakdown,
          last_seen_at = NOW()
        `,
        [
          wallet,
          String(row.shield_points_micro),
          String(row.total_deposited_base_units),
          row.asset_breakdown
        ]
      );
      continue;
    }

    shieldByUser.set(owner, (shieldByUser.get(owner) || 0n) + points);
  }

  await pool.query(`REFRESH MATERIALIZED VIEW leaderboard_profiles`);

  for (const user of users) {
    const waitlistPoints = BigInt((user.social_points || 0) + (user.referral_points || 0)) * MICRO;
    const shieldPoints = shieldByUser.get(String(user.id)) || 0n;
    const total = waitlistPoints + shieldPoints;

    await pool.query(
      `
      UPDATE leaderboard_profiles
      SET
        shield_points_micro = $2,
        total_points_micro = $3,
        refreshed_at = NOW()
      WHERE user_id = $1
      `,
      [user.id, String(shieldPoints), String(total)]
    );
  }
}

---
4.6 Campaign worker
This should keep using the V16.9.9 model:
- public deposit history only
- integer accounting only
- analytics rows only
You do not want a second shield-points engine in the waitlist app.
The only Phase 2 addition I recommend in the worker is a tiny cron wrapper.
File
apps/campaign-worker/src/cron.ts
import { exec } from "node:child_process";

const intervalMs = Number(process.env.CAMPAIGN_WORKER_INTERVAL_MS || "600000");

async function tick() {
  exec("pnpm start", { cwd: process.cwd() }, (err, stdout, stderr) => {
    if (err) {
      console.error("Campaign worker tick failed:", err.message);
      return;
    }
    if (stdout) console.log(stdout);
    if (stderr) console.error(stderr);
  });
}

tick();
setInterval(tick, intervalMs);

---
4.7 Google Sheets admin sync worker
File
apps/web/src/server/services/google-sheet-sync.ts
import { GoogleSpreadsheet } from "google-spreadsheet";
import { JWT } from "google-auth-library";
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.RAILWAY_DATABASE_URL });

export async function syncLeaderboardToSheet() {
  const serviceAccountAuth = new JWT({
    email: process.env.GOOGLE_SERVICE_ACCOUNT_EMAIL,
    key: process.env.GOOGLE_PRIVATE_KEY?.replace(/\\n/g, "\n"),
    scopes: ["https://www.googleapis.com/auth/spreadsheets"]
  });

  const doc = new GoogleSpreadsheet(process.env.GOOGLE_SHEET_ID!, serviceAccountAuth);
  await doc.loadInfo();

  const sheet = doc.sheetsByIndex[0];

  const { rows } = await pool.query(`
    SELECT
      lp.user_id,
      lp.x_handle,
      lp.primary_tempo_wallet,
      lp.status,
      lp.shield_points_micro,
      lp.total_points_micro,
      (lp.total_points_micro / 1000000) AS total_points_display,
      (lp.shield_points_micro / 1000000) AS shield_points_display
    FROM leaderboard_profiles lp
    ORDER BY lp.total_points_micro DESC
  `);

  await sheet.clearRows();
  await sheet.setHeaderRow([
    "User ID",
    "X Handle",
    "Primary Tempo Wallet",
    "Status",
    "Shield Points",
    "Total Points"
  ]);

  await sheet.addRows(
    rows.map((r) => ({
      "User ID": r.user_id,
      "X Handle": r.x_handle,
      "Primary Tempo Wallet": r.primary_tempo_wallet || "Not bonded",
      "Status": r.status,
      "Shield Points": r.shield_points_display,
      "Total Points": r.total_points_display
    }))
  );
}
Important rule
Sheets syncs from the merged leaderboard projection, not raw users only.
That fixes the drift problem.

---
4.8 Snapshot exporter
File
apps/web/src/server/services/snapshot-exporter.ts
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.RAILWAY_DATABASE_URL });

export async function exportSnapshot(snapshotName: string) {
  await pool.query("BEGIN");

  try {
    const snapshotRes = await pool.query(
      `
      INSERT INTO tge_snapshots(snapshot_name, snapshot_status)
      VALUES($1, 'draft')
      RETURNING id
      `,
      [snapshotName]
    );

    const snapshotId = snapshotRes.rows[0].id;

    const { rows } = await pool.query(`
      SELECT
        lp.user_id,
        lp.primary_tempo_wallet,
        lp.total_points_micro,
        lp.shield_points_micro,
        (lp.total_points_micro / 1000000) AS total_points_display,
        ((SELECT social_points + referral_points FROM users WHERE id = lp.user_id)) AS waitlist_points
      FROM leaderboard_profiles lp
      WHERE lp.status = 'active'
        AND lp.primary_tempo_wallet IS NOT NULL
    `);

    for (const row of rows) {
      await pool.query(
        `
        INSERT INTO tge_snapshot_allocations(
          snapshot_id,
          user_id,
          primary_tempo_wallet,
          waitlist_points,
          shield_points_micro,
          total_points_micro,
          allocation_units
        )
        VALUES($1,$2,$3,$4,$5,$6,$7)
        `,
        [
          snapshotId,
          row.user_id,
          row.primary_tempo_wallet,
          row.waitlist_points,
          row.shield_points_micro,
          row.total_points_micro,
          row.total_points_micro
        ]
      );
    }

    await pool.query(
      `
      UPDATE tge_snapshots
      SET snapshot_status = 'finalized', finalized_at = NOW()
      WHERE id = $1
      `,
      [snapshotId]
    );

    await pool.query("COMMIT");

    return {
      snapshotId,
      allocationCount: rows.length
    };
  } catch (e) {
    await pool.query("ROLLBACK");
    throw e;
  }
}

---
Part 5 — Exact cron responsibilities
This is the operations truth.
5.1 Campaign worker shield sync
- Frequency: every 10 minutes
- Input: indexer deposit events
- Output: shield_wallet_points
- Owner: apps/campaign-worker
5.2 Follower recheck
- Frequency: every 6 hours
- Input: active users
- Output: users.status, is_following_x, flagged_for_recheck
- Owner: apps/web service layer
5.3 Leaderboard rebuild
- Frequency: every 5 minutes
- Input: users, linked wallets, shield wallet rows
- Output: refreshed materialized leaderboard + ghost depositors
- Owner: apps/web service layer
5.4 Google Sheets sync
- Frequency: every 15 minutes
- Input: merged leaderboard projection
- Output: admin sheet
- Owner: apps/web service layer
5.5 Snapshot export
- Frequency: manual / admin-triggered only
- Input: finalized active leaderboard projection
- Output: snapshot tables + export files
- Owner: admin route / operator workflow

---
Part 6 — Unfollow handling, fully captured
This was one of your main concerns, so here is the exact law.
6.1 If the user unfollows @ParlyFi
The account is not deleted.
The system must:
1. set users.status = 'inactive'
2. set users.is_following_x = false
3. preserve points
4. freeze campaign progression
5. require re-verification at the gate
6.2 UI consequences
- dashboard shows Profile Frozen banner
- referral code is locked
- linking new secondary wallets is disabled
- TGE readiness card shows inactive state
- points remain visible but frozen
6.3 Recovery
Once follow is restored and verification succeeds:
- set status = 'active'
- remove banner
- unlock referral / progression again
That is now fully captured.

---
Part 7 — Acceptance criteria
A Phase 2 implementation is only acceptable if all of these are true.
1. A finalized deposit updates wallet-level Shield Points.
2. A profile can aggregate points from its primary and secondary wallets.
3. An unmatched wallet appears as a ghost.
4. A ghost can later convert into a real profile.
5. A wallet linked to one profile cannot link to another.
6. Unfollowed users are frozen, not deleted.
7. Re-following can reactivate the same profile.
8. TGE snapshot includes only primary Tempo wallets.
9. Google Sheets reflects merged leaderboard truth.
10. Shield Points are awarded once per finalized deposit event only.

---
Part 8 — Founder verdict
This is now the clean Phase 2 architecture:
Protocol → Indexer → Campaign Worker → Waitlist Profile Merge → Leaderboard → Snapshot → TGE / OFT UX
It fixes the earlier weak areas:
- no competing shield-point engines
- no fuzzy unfollow logic
- no ghost depositor ambiguity
- no secondary-wallet TGE confusion
- no Sheets drift from real campaign totals
- no shallow UI/UX flow
This document should now be treated as the single source of truth for Phase 2.
