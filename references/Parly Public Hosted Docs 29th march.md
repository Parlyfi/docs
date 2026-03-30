Production-Ready Public Documentation Set

---
Global Docs Rules
Canonical source
Parly has one private canonical monorepo. That repository is the internal source of truth for development, auditing, and release promotion.
Source distribution
The public website explains the product, the operator flow, and the integration model. GitHub provides the runnable public source surfaces.
Public GitHub distribution is split by role:
- public relayer repo for operators
- public SDK repo/package for developers
- public MCP repo for agent integrations
The public repos are distribution layers. They are not alternate sources of truth.
Operator model
A relayer operator should use the public relayer repo and its operator quickstart.
Operators do not need to clone the private canonical monorepo, and they do not need to boot the full app, indexer, MCP server, or campaign stack just to run a relayer.
Standard source CTAs
Across the docs, use these CTA labels consistently:
- Open App
- Open Verify
- Run a Relayer
- View Relayer Source
- View SDK Source
- View MCP Source
- Relayer Registration
- Operator Quickstart
Writing rules
Public docs should:
- explain the live product truthfully
- distinguish clearly between user, operator, and developer journeys
- never imply protocol permissioning where only frontend discovery is curated
- never promise features that are outside launch scope
- keep “submitted,” “pending,” “confirmed,” and “failed” distinct
- avoid telling public users to clone the private canonical monorepo

---
1. /docs
How Parly Works
Page Title
How Parly Works
Page Subtitle
Parly is a privacy protocol for stablecoin movement across supported networks. It is built for people, teams, and machine-driven systems that want private execution, deterministic recovery, and selective proof without turning the product into a trading terminal, a workflow scheduler, or a protocol-owned relay monopoly.
CTA Row
- Open App
- Architecture Overview
- Start Here
- Run a Relayer

---
Section 1 — What Parly does
Parly allows users to move supported stablecoins into a private balance on Tempo, spend privately on Tempo, and send privately from Tempo to supported destination networks. The protocol is intentionally built for immediate execution. It does not blur into delayed workflows, generalized swap logic, or a bloated “do everything” control panel.
At public launch, the live product surface includes direct shielding on Tempo, spoke-origin shielding into Tempo, private same-network payouts on Tempo, private cross-network stablecoin-only payouts from Tempo to supported spokes, third-party private relay execution over Waku, self-relay fallback, a public verification portal, a developer SDK, an MCP server, and an MPP-compatible adapter only at the SDK/MCP boundary.
Parly is meant to feel sharp, controlled, and legible. The product is narrower by design because narrow systems are easier to trust, easier to operate, and easier to recover.

---
Section 2 — What Parly does not do
Parly does not try to be everything at once. It does not launch with scheduling, delayed execution, intent vaults, REST intent APIs, swap routing, 1inch integration, volatile-token payout paths, passkey privacy mode, dual identity planes, on-chain session settlement, transient note chaining, or a protocol-owned exclusive relay network.
Those are not accidental omissions. They are deliberate exclusions that keep the launch surface stable and production-reasonable.
Parly is a privacy execution system. It is not a trading interface, not a scheduling engine, and not a protocol-owned relay monopoly.

---
Section 3 — Privacy and recovery
Parly is designed so recovery does not depend on browser kindness, local storage luck, or one specific device surviving forever. Private state is tied to a deterministic wallet-derived recovery identity plus the on-chain note and envelope data the protocol emits during shielding and execution.
Browser storage may improve convenience. It is not the recovery authority.
The protocol uses encrypted note envelopes so a user can recover live notes from chain-visible history on a future device. The envelope is created for the intended owner, and only the owner with the matching recovery private key can open it. Recovery therefore stays anchored to wallet control rather than device memory.
That model is what gives Parly its “serious infrastructure” feel. The user experience can be smooth, but the recovery truth stays cryptographic.

---
Section 4 — Verification and selective disclosure
Parly includes a public verification flow designed for selective disclosure. A user can disclose one payout lane without exposing the wider execution set. That is useful for counterparties, exchanges, auditors, finance teams, and reviewers who need proof of one payment without opening the rest of a private treasury graph.
Verification is intentionally narrow. It is not meant to become a batch-cost reconciliation console, a general-purpose explorer, or a backdoor treasury inspection surface.
The point is simple: prove what was deliberately disclosed, and nothing more.

---
Section 5 — Supported networks and routing model
Parly uses Tempo as the privacy hub. Supported spoke networks route into Tempo for shielding and receive private stablecoin payouts from Tempo on outbound delivery. That topology is intentional. It keeps privacy state, recovery logic, and verification logic centered instead of spreading them across unrelated chains with inconsistent assumptions.
Not every network is treated as equivalent. The hub-and-spoke design is one of the reasons the system remains easier to reason about under pressure.

---
Section 6 — Machine payments and agent integrations
Parly supports developer and agent integrations through a public SDK and an MCP server. It also supports MPP compatibility, but only at the SDK and MCP boundary. That means Parly can fit into machine-payment workflows without rewriting the protocol core around session contracts or changing the settlement model into something it is not.
In practical terms, Parly remains the privacy settlement layer. Session-shaped workflows and machine-payment adapters live above it.

---
Section 7 — Reliability and execution clarity
Parly explicitly distinguishes between submitted, pending, confirmed, and failed.
Private relay execution stops at encrypted submission until execution is actually confirmed. Direct browser execution should only report success after a confirmed result on Tempo. When the outcome is still uncertain, Parly keeps that uncertainty visible instead of pretending the result is final.
That honesty matters. Privacy systems lose trust the moment they start collapsing “submitted” and “confirmed” into the same message.

---
Section 8 — Public launch scope
Parly currently supports:
- stablecoin-only private pools
- Tempo as the settlement hub
- shielding on Tempo
- spoke-origin shielding into Tempo
- private same-network payouts on Tempo
- private cross-network stablecoin-only payouts
- private relay over Waku
- self-relay fallback
- selective verification
- developer SDK
- MCP-based agent integration
- MPP-compatible integration at the SDK and MCP boundary only
- off-chain removable shield-mining analytics where enabled
Parly does not currently support:
- scheduling
- delayed execution
- intent vaults
- REST intent APIs
- swap routing
- volatile-token outputs
- passkey privacy mode
- protocol-owned exclusive relay operation
- on-chain session state for machine payments
- plaintext local note storage as a recovery dependency

---
Footer CTA
- Open App
- Open Verify
- Run a Relayer

---
2. /docs/quickstart
Start Here
Page Title
Start Here
Page Subtitle
Use this page to choose the right entry point into Parly, whether you want to use the product directly, operate relay infrastructure, build on the SDK, or connect Parly into an agent workflow.
CTA Row
- Open App
- Run a Relayer
- View SDK Source
- View MCP Source

---
Section 1 — I want to use Parly
Start with the App.
That is the user-facing control surface for wallet connection, authentication, shielding, private execution, activity review, selective receipts, and verification handoff. It is where product truth becomes usable.
From the App, a user can connect a wallet, authenticate private state, shield supported stablecoins, execute private payouts, review history, and prepare a disclosed payout lane for independent verification.
CTA
- Open App
- Using the App

---
Section 2 — I want to run a relayer
Start with the relayer docs and the public relayer source.
Parly ships a full relayer runtime, not just a vague relay concept. Operators do not need to invent their own private-execution daemon from zero before participating. The shipped relayer covers key generation, encrypted bundle handling, execution validation, submission, and pending-state reconciliation.
This is the right path if you want to operate private execution infrastructure.
CTA
- Run a Relayer
- View Relayer Source
- Operator Quickstart
- Relayer Registration

---
Section 3 — I want to integrate Parly in code
Start with the SDK.
The SDK is the developer integration surface for deterministic authentication, recovery identity derivation, note handling, execution request construction, exact approval discipline, and outcome handling. It mirrors the live protocol model instead of pretending the protocol is simpler than it really is.
This is the correct entry point for backend services, automation, service operators, and integrations that want protocol-accurate behavior.
CTA
- View SDK Source
- Parly SDK
- Architecture Overview

---
Section 4 — I want to build an agent workflow
Start with the MCP docs.
The MCP server exists for bounded agent workflows. It exposes agent-owned tools, keeps agent keys separate from relayer and treasury power, and supports MPP-compatible wrapping where needed without changing the core protocol law.
This is the right path for copilots, service agents, and headless automation.
CTA
- View MCP Source
- Parly MCP Server
- View SDK Source

---
Section 5 — I want to understand the system before using it
Start with How Parly Works and Architecture Overview.
Those pages explain the launch surface, recovery model, selective verification, relay model, SDK/MCP boundary, and the parts that are intentionally excluded.
CTA
- How Parly Works
- Architecture Overview

---
Section 6 — Public surfaces
App
The main execution surface for shielding, recovery, private execution, activity review, selective disclosure, and verification handoff.
Verify
The public verification surface for one disclosed payout lane.
Relayer
The operator runtime for private execution.
Registry
The official discovery surface for relayer listing in the public app.
SDK
The developer integration layer.
MCP Server
The agent tooling layer.

---
3. /docs/app
Using the App
Page Title
Using the App
Page Subtitle
The Parly App is the primary user-facing surface for shielding, recovery, private execution, activity review, and selective verification. This page explains the product flow from first connection through payout proof.
CTA Row
- Open App
- Open Verify
- How Parly Works

---
Section 1 — What the App is for
The App is the main control surface for Parly users. It is where a user connects a wallet, authenticates private state, shields supported stablecoins, executes private payouts, reviews activity, prepares selective-disclosure records, and hands a disclosed lane to the Verify portal when proof is needed.
Parly is intentionally designed so execution and review stay connected. Shielding, sending, history, receipts, and verification handoff belong to one coherent operator flow.

---
Section 2 — Connect and authenticate
Parly does not treat wallet connection and private-state access as the same thing.
Connecting a wallet gives the application an active address. Authenticating that wallet activates private-state recovery and execution by asking the wallet to sign a fixed authentication message. That message is used to derive the recovery identity used for private note recovery. It does not require private-key export.
In practical terms, the flow is:
1. Connect wallet
2. Authenticate
3. Recover private state
4. Use Shield, Execute, Ledger, and Verify
This separation matters because “connected” is not the same as “ready to recover private state.”

---
Section 3 — Shield
Shielding moves supported stablecoins into a private balance on Tempo.
There are two shielding paths:
- direct shielding on Tempo
- spoke-origin shielding into Tempo
Both paths are live. Spoke-origin shielding routes through the hub-and-spoke topology and later appears in history and provenance where that data is actually available.
The Shield surface should guide the user through origin network selection, supported asset selection, amount entry, token approval where required, transaction confirmation, and confirmed private-balance entry.
When shielding succeeds, the protocol emits the data required for future recovery and note reconstruction.

---
Section 4 — Execute
The Execute surface is where a user spends from private balance.
It supports:
- single send
- batch send
- private same-network payouts on Tempo
- private cross-network stablecoin-only payouts to supported spokes
- private relay by default
- self-relay fallback
The interface must keep submitted, pending, confirmed, and failed visually distinct. Parly is designed to keep uncertain post-broadcast states honest instead of flattening everything into “success.”
That distinction is not cosmetic. It is part of trust.

---
Section 5 — Ledger and Compliance
The App includes a Ledger and Compliance surface directly below the main execution console. This is where users review deposits, private executions, change-note activity, scoped payout records, and disclosure-preparation flows.
The compliance model is selective by design. A user can review one payout scope at a time, export a lane-scoped record, and hand that record to a counterparty, auditor, or exchange without exposing the wider private execution set.
This surface should be treated as:
- a review surface
- a receipt surface
- a proof-preparation surface
It is not a recovery console and not a general transaction explorer.

---
Section 6 — Scoped receipts
A scoped receipt is a record for one disclosed payout lane.
Depending on what is truthfully available, a scoped receipt may include confirmed settlement facts, the disclosed payout scope, local note relationship details where appropriate, and provenance details where the source and settlement path can be linked honestly.
A scoped receipt is intentionally narrower than the full private execution. It is built for disclosure of one payment, not exposure of a full treasury graph.

---
Section 7 — Verification handoff
When a scoped receipt is shared, the recipient can independently verify the disclosed payout lane using the Verify portal.
The typical handoff is:
- the confirmed Tempo transaction hash
- the disclosed child key
That verification flow is public, selective, and independent of the sender’s app session.

---
Section 8 — What the App does not do
The App is not a swap terminal, not a delayed execution console, and not a general-purpose chain-agnostic wallet. It is a private stablecoin execution console with deterministic recovery and selective verification.
That clarity is one of the reasons the product feels professional instead of chaotic.

---
4. /docs/verify
Verify a Payout
Page Title
Verify a Payout
Page Subtitle
Parly Verify allows a counterparty, auditor, exchange, or reviewer to confirm one disclosed payout lane from a real Parly execution using a confirmed Tempo transaction hash and a child key.
CTA Row
- Open Verify
- Using the App
- How Parly Works

---
Section 1 — What Verify does
Verify is the public selective-disclosure surface for Parly.
It answers one narrow question extremely well:
Does this disclosed child key correspond to a real payout lane in the referenced Parly execution?
That makes Verify useful for counterparties, auditors, finance teams, exchanges, and compliance reviewers who need confidence in one disclosed payment without demanding access to the sender’s full private history.

---
Section 2 — What you need
To use Verify, you need:
- the confirmed Tempo transaction hash
- the disclosed child key
Those values are usually shared in a scoped receipt or directly by the payer.

---
Section 3 — What Verify checks
Verify checks whether the disclosed child key matches a real payout lane associated with the referenced Parly execution.
Depending on the available indexed and scoped data, the result may display:
- settlement confirmation
- payout recipient
- payout amount
- destination chain or route
- provenance details where truthfully available
The purpose is not to reveal everything. The purpose is to prove the disclosed lane cleanly, publicly, and independently.

---
Section 4 — What Verify does not do
Verify does not:
- reveal the entire private execution set
- expose unrelated payout lanes
- reconstruct full treasury behavior
- assign shared route economics to one disclosed lane
- replace a full internal audit system
It stays narrow because selective verification only works if the proof surface remains selective.

---
Section 5 — Who Verify is for
Verify is for anyone who needs confidence in one disclosed payout lane without opening the wider private set.
That includes:
- exchanges checking a claimed receipt
- counterparties verifying a payment
- internal finance reviewers
- auditors reviewing one specific lane
- compliance teams checking a disclosed record

---
Section 6 — Typical verification flow
1. Receive the confirmed Tempo transaction hash
2. Receive the disclosed child key
3. Open Parly Verify
4. Enter both values
5. Review the matched payout result
If the disclosed values match a real lane, Verify confirms it. If they do not, the result should fail clearly and cleanly.

---
Section 7 — Why selective verification matters
Selective verification gives users a practical compliance path without forcing them to open the rest of a private execution set.
Privacy without any proof surface is difficult to use in real operations.
Full exposure defeats the point.
Verify exists in the middle, where privacy remains useful.

---
5. /docs/relayers
Run a Parly Relayer
Page Title
Run a Parly Relayer
Page Subtitle
Parly ships a full relayer runtime as part of the standard product stack. This page explains what a relayer does, what an operator is responsible for, how to get the source, and how to run the runtime in production.
CTA Row
- Relayer Registration
- View Relayer Source
- Operator Quickstart
- Run a Relayer

---
Section 1 — What a relayer is
A Parly relayer is the runtime that receives encrypted private execution requests, decrypts only the requests addressed to it, validates whether they are safe and economically executable, submits valid executions, and keeps uncertain outcomes honest until they are resolved.
A relayer is not a treasury controller, not a settlement rewrite, not a privileged protocol owner, and not a replacement for the protocol itself.
It is an operator runtime for private execution.

---
Section 2 — What Parly already ships
Parly already ships a real relayer application. Independent operators do not need to invent relay architecture from scratch to participate. The shipped runtime includes the relayer service itself, key generation flow, bundle handling, execution path, and pending-state reconciliation model.
This is why the relayer is treated as a serious runtime surface rather than a toy script.

---
Section 3 — Source and distribution
The correct starting point for relayer operators is the public relayer repo.
Operators should:
1. open the public relayer source
2. follow the operator quickstart
3. configure the relayer environment
4. run the relayer runtime
5. run the pending reconciler beside it
6. register through the relayer registry if they want official public-app discovery
Operators do not need the private canonical monorepo to run a relayer. The canonical monorepo remains internal source of truth; the public relayer repo is the correct operator distribution surface.
CTA
- View Relayer Source
- Operator Quickstart

---
Section 4 — What the relayer does in practice
The relayer is responsible for:
- listening for private relay bundles
- decrypting only bundles addressed to its public key
- validating payload structure
- validating execution economics
- submitting valid private executions
- persisting uncertain post-broadcast outcomes
- reconciling pending outcomes later
This is why Parly treats the relayer as real infrastructure.

---
Section 5 — Operator requirements
A production relayer operator should have:
- a stable server or always-on deployment
- a Tempo RPC endpoint
- Waku connectivity
- a relayer execution wallet
- a relayer sealed-box keypair
- secure secret storage
- the pending reconciler running beside the main relayer process
This is an always-on infrastructure role. It is not a passive listing.

---
Section 6 — Quickstart
The relayer operator flow is straightforward:
Install dependencies, configure the environment, generate the relayer keypair, start the relayer runtime, start the pending reconciler, and then register the relayer for official frontend discovery if desired.
A typical operator journey looks like this:
- open the public relayer source
- copy .env.example to .env
- generate the sealed-box keypair
- configure execution address, RPC, Waku, and runtime thresholds
- start the relayer
- start the pending reconciler
- open /relayer and register for official discovery
The registry step improves visibility in the public app. It does not create protocol exclusivity.

---
Section 7 — Required configuration
At minimum, a relayer operator needs:
- the relayer execution private key
- the relayer sealed-box private key
- Waku connection values
- Tempo RPC
- profitability and retry settings
- pending reconciliation file paths
Parly deliberately keeps relayer execution keys, relayer sealed-box keys, treasury keys, and agent keys separate. That boundary should not be collapsed for convenience.

---
Section 8 — Official discovery and listing
The official relayer registry exists for two purposes:
- official frontend discovery
- public-key distribution for bundle encryption
It does not permission the protocol. A relayer can operate without being listed, but listing makes it visible to the public app and allows the official frontend to encrypt to it cleanly.

---
Section 9 — Reliability expectations
A relayer operator is expected to:
- keep the relayer online
- keep the pending reconciler online
- keep successful activity flowing back into registry state
- preserve honest pending state when results are uncertain
- avoid changing bundle semantics
- maintain secure key custody
A stale or inactive relayer may fall out of official discovery, but the protocol itself remains open.

---
Section 10 — Independent operation
If you want official frontend discovery, register through the relayer registry.
If you do not need official discovery, you can still run the relayer independently and integrate it through your own frontend or tooling. Official listing improves discoverability. It does not create exclusivity or monopoly rights.

---
6. /sdk
Parly SDK
Page Title
Parly SDK
Page Subtitle
The SDK is the developer integration layer for Parly. It mirrors the live recovery and execution model so services, automation systems, operators, and machine-payment environments can integrate the protocol without re-implementing its core client logic.
CTA Row
- View SDK Source
- View MCP Source
- How Parly Works
- Run a Relayer

---
Section 1 — What the SDK is
The SDK exists so developers can use Parly without rebuilding deterministic authentication, recovery identity derivation, note-envelope handling, note reconstruction, execution request construction, exact approval logic, and outcome handling from scratch.
It is not a fake abstraction layer. It preserves protocol reality while making integration practical.

---
Section 2 — What the SDK includes
The SDK includes support for:
- deterministic recovery key derivation from the fixed auth message
- note-envelope decryption
- reconstruction of private state from indexed history
- private execution flow construction
- exact-amount approval handling
- clear handling of uncertain or pending execution outcomes
- optional MPP-compatible wrappers above the same execution engine

---
Section 3 — Install
pnpm add @parly/sdk

---
Section 4 — Initialize the SDK
import { ParlySDK, ParlyMppAdapter } from "@parly/sdk"

const sdk = new ParlySDK({
  privateKeyHex: process.env.AGENT_PRIVATE_KEY as `0x${string}`,
  tempoChainId: Number(process.env.TEMPO_CHAIN_ID),
  tempoRpcUrl: process.env.TEMPO_RPC_URL!
})

const mpp = new ParlyMppAdapter(sdk, {
  serviceName: process.env.MPP_SERVICE_NAME,
  serviceVersion: process.env.MPP_SERVICE_VERSION
})
This is the intended initialization shape for service-side integrations and agent workflows. It matches the live SDK and MCP boundary model.

---
Section 5 — What the SDK handles
The SDK handles deterministic auth-message usage, recovery key derivation, note-envelope handling, note reconstruction, direct execution path construction, exact approval flow, and outcome handling.
That is why the SDK should be treated as a real integration surface and not just a sample package.

---
Section 6 — MPP-compatible integration
Parly supports MPP compatibility through an adapter that wraps the existing execution engine.
Create a session:
const session = await mpp.createSession({
  sessionId: "invoice-2026-03-27",
  counterparty: "0x1111111111111111111111111111111111111111",
  assetId: 1,
  spendLimit: "25.0",
  destinationEid: Number(process.env.TEMPO_LZ_EID),
  poolAddress: process.env.NEXT_PUBLIC_USDC_POOL as `0x${string}`
})
Settle from a session:
const outcome = await mpp.settleFromSession({
  sessionId: session.sessionId,
  destination: "0x1111111111111111111111111111111111111111",
  amount: "25.0"
}, session)
This adapter does not alter pool logic, circuit semantics, or contract-level settlement rules. It creates a developer-friendly session layer above the same privacy engine.

---
Section 7 — What the SDK does not do
The SDK does not:
- turn Parly into a different settlement protocol
- add on-chain session state
- bypass the recovery model
- hide exact approval handling
- blur the line between submitted, pending, and confirmed
Its job is to preserve protocol correctness while making integration easier.

---
7. /docs/mcp
Parly MCP Server
Page Title
Parly MCP Server
Page Subtitle
Parly includes an MCP server for agentic workflows. It exposes bounded tools over agent-owned execution only and keeps agent, relayer, and treasury responsibilities separate.
CTA Row
- View MCP Source
- View SDK Source
- How Parly Works
- Run a Relayer

---
Section 1 — Security model
The MCP server must run with an agent key only. It must not inherit relayer execution power, relayer sealed-box secrets, or treasury control.
This is one of the most important safety boundaries in the Parly stack.

---
Section 2 — What the MCP server exposes
The MCP server supports:
- recover_largest_note
- execute_shielded_payment
When MPP compatibility is enabled, it may also support:
- mpp_create_session
- mpp_settle_session_payment

---
Section 3 — recover_largest_note
This tool recovers the largest live note controlled by the configured agent key for a given asset. It is useful for agents that need to reason about current private spendable balance before attempting execution.

---
Section 4 — execute_shielded_payment
This tool executes a private Parly payment from notes controlled by the configured agent key. It is the direct immediate execution path for agent workflows.

---
Section 5 — mpp_create_session
This tool creates an MPP-compatible session descriptor around the current agent-owned execution path. It is useful when an agent or service wants a machine-readable spending lane without changing protocol law.

---
Section 6 — mpp_settle_session_payment
This tool settles a Parly payment from a previously created session descriptor. It reuses the same execution engine rather than introducing a separate settlement protocol.

---
Section 7 — Runtime behavior
When the MCP server starts, it should make the active key scope and adapter state explicit. That includes:
- whether it is running with the agent key only
- whether MPP compatibility is enabled
- which service name and version are active when the adapter is used
This visible boundary is good public runtime behavior because it makes the security model obvious instead of vague.

---
8. /docs/relayer-registry
Relayer Registry
Page Title
Relayer Registry
Page Subtitle
The relayer registry supports official frontend discovery and relayer public-key distribution. It does not permission the protocol and it does not create exclusive relay rights.
CTA Row
- Relayer Registration
- Run a Relayer
- View Relayer Source

---
Section 1 — What the registry is for
The official frontend needs to know which relayer execution addresses are officially discoverable and which relayer public keys it may encrypt bundles to. That is what the registry does.
It is a discovery layer for the official app. It is not a protocol-level allowlist.

---
Section 2 — Registry policy
The official Parly registry supports a maximum of 10 live relayers.
Registration is automatic if validation passes and a slot is available. Inactive relayers are removed from official live discovery so slots can reopen without turning the product into a moderation marketplace or review queue.
That keeps the registry lean, legible, and operator-grade.

---
Section 3 — Register a relayer
Go to:
parly.fi/relayer
You will need:
- the relayer execution address
- the Base64 relayer public key
- a wallet signature proving control of the submitted execution address
Registration succeeds automatically if:
- the registry is open
- the address is valid
- the public key is valid
- the signature is valid
- the relayer is not a duplicate entry

---
Section 4 — Registry states
Verified
Live in official frontend discovery.
Inactive
Previously live, now removed from official discovery because of inactivity, operator withdrawal, manual disable action, or failed health expectations.
Rejected
Failed automatic validation.

---
Section 5 — Important note
Official listing improves public frontend discovery only. It does not control whether the protocol is open, and it does not stop advanced users, SDK users, or custom frontend operators from targeting independent relayers directly.

---
9. /docs/architecture
Architecture Overview
Page Title
Architecture Overview
Page Subtitle
Parly is designed as a focused privacy execution stack rather than a single frontend with hidden assumptions. This page explains the core system components, how they relate to one another, and why the product is structured this way.
CTA Row
- Open App
- View Relayer Source  → https://github.com/Parlyfi/parly-relayer
- View SDK Source → https://github.com/Parlyfi/parly-sdk
- View MCP Source → https://github.com/Parlyfi/parly-mcp

---
Section 1 — High-level system design
Parly uses a hub-and-spoke model with Tempo as the privacy hub.
At a high level, the stack consists of:
- the web application
- on-chain contracts
- zero-knowledge circuits
- the relayer runtime
- the indexer
- the SDK
- the MCP server
- optional analytics infrastructure
Each component has a specific role. The architecture is designed so that recovery logic, public indexing, relay execution, agent tooling, and analytics do not collapse into one trust domain.

---
Section 2 — Web application
The web application is the user-facing product surface.
It handles:
- wallet connection
- authentication
- private-state recovery
- shielding
- execution
- activity review
- scoped receipts
- verification handoff
The web app is where protocol truth becomes usable. It is not itself the protocol’s ultimate source of truth.

---
Section 3 — Contracts and circuits
The contract layer holds the settlement logic for shielding, private execution, and hub/spoke routing. The circuit layer defines the proof system used for private execution and keeps the execution model bounded and predictable.
Parly uses a fixed padded execution model at launch. That stability matters because frontend, relayer, SDK, and contract boundaries all need the same execution shape.

---
Section 4 — Relayer
The relayer is the private execution daemon.
It:
- receives encrypted relay requests
- decrypts only the requests addressed to it
- validates and submits execution
- tracks uncertain post-broadcast states
- reconciles pending outcomes later
Parly ships the relayer implementation directly. Independent operators run it through the public relayer distribution surface.

---
Section 5 — Indexer
The indexer is the public data layer.
It serves:
- activity history
- verification queries
- provenance joins where available
- analytics surfaces
The indexer matters because selective disclosure needs a public fact layer that does not depend on loose inference.

---
Section 6 — SDK
The SDK is the developer integration layer.
It mirrors:
- deterministic authentication
- recovery identity derivation
- note handling
- execution flow construction
- approval discipline
- outcome handling
This makes it suitable for operators, service integrations, and automation.

---
Section 7 — MCP server
The MCP server is the bounded agent tooling layer.
It allows:
- note recovery
- immediate private payment execution
- session-shaped machine-payment flows through the MPP-compatible adapter when enabled
It is intentionally separated from relayer and treasury power.

---
Section 8 — MPP compatibility
Parly supports MPP compatibility only at the SDK and MCP boundary.
That means:
- Parly can fit into machine-payment workflows
- the adapter can expose session-shaped developer and agent flows
- the protocol core does not become an on-chain session-settlement system
That boundary is deliberate. It is not an omission.

---
Section 9 — Repository and deployment posture
Parly maintains one private canonical monorepo as the internal source of truth. Public code distribution is split by role into:
- a public relayer repo for operators
- a public SDK repo/package for developers
- a public MCP repo for agent integrations
A realistic production deployment is not frontend-only. Depending on the surface being operated, the broader stack may include the web app, relayer runtime, indexer, MCP server, and optional analytics services.
That is why runtime separation, secret separation, and role-specific source distribution matter from day one.

---
Final Recommended Public Docs Navigation
Use this public documentation structure:
- /docs
- /docs/quickstart
- /docs/app
- /docs/verify
- /docs/relayers
- /sdk
- /docs/mcp
- /docs/relayer-registry
- /docs/architecture
That is the fully updated hosted docs set for the current Parly launch surface.