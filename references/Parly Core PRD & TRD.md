Parly Core PRD & TRD
V16.9.9 Canonical Core Product & Technical Document
Status: Cross-team source of truth
Audience: Founders, product, design, frontend, backend, smart-contract engineers, circuit engineers, relayer operators, SDK engineers, MCP engineers, growth, compliance, and infrastructure teams

---
1. Document hierarchy, scope, and reading order
0.1 Purpose
This document is the cross-team anchor for Parly.
Its purpose is to give every team one shared understanding of what Parly is, what it is not, what is in launch scope, how the system works end to end, which trust boundaries are fixed, and which supporting documents govern deeper implementation detail. It is aligned to the latest V16.9.9 runbook, the canonical UI/UX blueprint, the Phase 2 master document, the relayer registry PRD, and the public hosted docs.
This document is written so that a founder, designer, frontend engineer, Solidity engineer, relayer operator, growth lead, compliance reviewer, or agent-integration engineer can all read the same text and leave with the same core picture.
0.2 How to use this document
This should be the first Parly document most people read.
Read this document first if you need to understand the product thesis, the live launch surface, the main user and operator flows, the architecture at system level, the roles of Tempo, Waku, Ponder, Railway, Verify, Ledger & Compliance, Admin Vault, SDK, MCP, relayers, and Phase 2 systems, and the line between protocol law and removable campaign infrastructure.
Then move to the supporting documents that are relevant to your role.
0.3 Supporting documents every team should know
The team should read these alongside this document:
1. Runbook V16.9.9 Final - Updated
Canonical engineering law for the current version. It governs implementation truth, architecture, contracts, relayer/runtime logic, repo distribution, validation, and acceptance gates.
2. Parly.fi Privacy Protocol UI/UX Master Blueprint
Canonical user-facing behavior and experience source. It governs routes, states, layout, nav, copy direction, authentication stages, Ledger & Compliance presentation, Verify behavior, and Admin Vault experience.
3. Parly.fi Phase 2 Master Document
Canonical growth-layer source. It governs waitlist, referrals, shield-mining analytics, ghost depositors, snapshots, TGE readiness, leaderboard logic, Railway ownership, Google Sheets sync, and the rule that Phase 2 sits above protocol truth rather than inside it.
4. Parly Relayer Registry PRD / Design Doc
Canonical registry source. It governs relayer registration, registry constraints, slot caps, validation, backend activity tracking, and the distinction between official frontend discovery and open protocol participation.
5. Parly Public Hosted Docs
Canonical public explanation source. It governs how users, operators, developers, and agent builders are directed to the correct surface without misrepresenting launch scope or repo distribution.
0.4 Conflict-resolution rule
If two documents appear to conflict, use this order of precedence:
1. Runbook V16.9.9 Final - Updated for engineering law and implementation truth.
2. Parly Core PRD & TRD for cross-team understanding and system alignment.
3. Parly.fi Privacy Protocol UI/UX Master Blueprint for user-facing behavior and experience structure.
4. Parly Relayer Registry PRD / Design Doc for registry-specific behavior.
5. Parly.fi Phase 2 Master Document for waitlist, leaderboard, snapshot, shield mining, and OFT/TGE growth systems.
6. Parly Public Hosted Docs for public-facing explanation and role-correct documentation language.
0.5 What this document does not do
This document does not replace the full runbook, the full UI/UX blueprint, the full relayer manual, or the full Phase 2 specification.
It connects them into one coherent picture.
0.6 Read-this-first summary
If someone reads only one page before joining work, they should leave with these truths:
Parly is a stablecoin-only privacy execution system. Tempo is the privacy hub. Waku is the private relay transport. Ponder is the public fact and indexing layer. Railway is the app-owned relational state layer where needed. Verify proves one disclosed payout lane rather than a whole treasury graph. Ledger & Compliance is a named product surface, not a side widget. The Admin Vault is a high-trust governance and revenue surface. SDK and MCP are real product surfaces, not decorative additions. Phase 2 growth systems are removable and must never be mistaken for protocol law.
0.7 Glossary of core terms
Shield
Moving supported stablecoins from public wallet state into Parly’s private balance model on Tempo.
Tempo
The privacy hub and settlement center of Parly’s hub-and-spoke design.
Spoke network
A supported external EVM network that can route stablecoins into Tempo for shielding and receive stablecoin payouts from Tempo.
Note
A private spendable value object in the protocol’s note-based accounting model.
Note envelope
An encrypted recovery payload emitted onchain so the rightful owner can later reconstruct and recover live notes.
Deterministic recovery identity
The wallet-derived cryptographic identity used to recover private state without relying on browser storage as the source of truth.
Relayer
An operator runtime that receives encrypted execution bundles, decrypts only bundles intended for it, validates them, submits valid executions, and tracks uncertain outcomes honestly.
Self-relay
An advanced fallback mode where the user submits directly instead of using third-party private relay as the default path.
Child key
The scoped identifier used for selective disclosure of one payout lane inside a wider private execution.
Verify
The public portal that checks whether a disclosed child key corresponds to a real payout lane in a referenced Parly execution.
Ledger & Compliance
The in-app review surface where deposits, executions, scoped receipts, and proof-preparation flows are shown to the user.
Ponder
The indexing and query layer that turns onchain events into structured product truth for history, verification, provenance, and analytics.
Railway
The managed application database and deployment layer used for service-owned relational state such as campaign, leaderboard, snapshot, or admin-sync data where needed.
SDK
The developer-facing integration layer that mirrors Parly’s live recovery and execution model.
MCP
The agent-facing tool surface for bounded AI and automation workflows.
MPP-compatible adapter
A wrapper at the SDK/MCP boundary that allows machine-payment-style workflows without changing protocol-core settlement law.

---
Part I — Product Requirements Document (PRD)
1. Positioning
Parly is private stablecoin omnichain infrastructure for the agentic economy.
It is built for people, teams, services, and machine-driven systems that need to move stablecoins privately across supported networks without turning the product into a trading terminal, a workflow scheduler, or a protocol-owned relay monopoly. The UI/UX source fixes this positioning directly in the hero and trust narrative, framing Parly as a premium institutional system built on Next.js, Tailwind CSS, Wagmi V2, and Waku P2P.
Parly is intentionally narrow at launch. That narrowness is a strength. The product is meant to feel serious, legible, and production-grade rather than feature-bloated.
2. Executive summary
Parly is a privacy execution stack for stablecoins.
At launch, it supports shielding on Tempo, shielding from supported spoke networks into Tempo, private same-chain payouts on Tempo, private cross-chain stablecoin-only payouts from Tempo to supported spokes, third-party private relay execution over Waku, self-relay fallback, a public Verify portal, off-chain removable shield-mining analytics, a developer SDK, an MCP server, and an MPP-compatible adapter only at the SDK/MCP boundary. The runbook calls this the exact supported launch surface.
It does not launch with scheduling, delayed execution, intent vaults, REST intent APIs, swap routing, 1inch, volatile-token payout paths, passkey privacy mode, dual identity planes, transient note chaining, multi-proof chaining from one active note, protocol-operated monopoly relay, plaintext local note storage as a recovery dependency, custom spoke withdrawal receivers, or contract-level MPP session settlement.
That means Parly should be understood as a privacy-native stablecoin settlement layer, not a general-purpose DeFi dashboard.
3. Product vision
Parly exists because public stablecoin rails reveal too much.
Public ledgers expose treasury relationships, recipient graphs, timing behavior, operational logic, machine-payment flows, and strategic activity by default. Parly’s answer is not fantasy invisibility. Its answer is disciplined private execution, deterministic recovery, selective proof, open relay participation, and real developer and agent integration surfaces that make the protocol usable in actual business and machine contexts. This framing is consistent across the runbook, hosted docs, and UI/UX materials.
The long-term vision is simple: a human treasury operator, a startup backend, an autonomous agent, and an enterprise machine-payment environment should all be able to use the same privacy-native settlement layer.
4. Core product promise
Parly lets a user:
1. shield supported stablecoins into a private pool,
2. execute private payouts from that shielded balance,
3. send privately on Tempo or from Tempo to supported spokes,
4. preserve privacy by default through third-party relay execution,
5. recover state deterministically from wallet-derived identity plus chain data,
6. selectively disclose one payout lane when needed,
7. and verify that disclosed lane through a dedicated public Verify surface.
That is what makes Parly usable in real operations. It is not just “private transfer.” It is private transfer with deterministic recovery and scoped proof.
5. Canonical launch law
The current canonical launch surface includes:
- shield on Tempo,
- shield from supported spokes into Tempo using OFT compose,
- private same-chain payouts on Tempo,
- private cross-chain stablecoin-only payouts from Tempo to supported spokes,
- single send and batch send up to 10 outputs per proof,
- third-party private relay execution over Waku,
- self-relay fallback,
- Verify,
- removable off-chain shield-mining analytics,
- Agentic SDK,
- MCP server,
- and an MPP-compatible adapter at the SDK/MCP boundary only.
The current canonical exclusions are:
- scheduling,
- delayed execution,
- intent vaults,
- REST intent APIs,
- swap routers,
- 1inch,
- volatile-token payout paths,
- passkey privacy mode,
- dual identity planes,
- MPP-native contract settlement primitives,
- on-chain MPP session state,
- transient note chaining,
- multi-proof chaining from one active note,
- protocol-operated monopoly relayer,
- plaintext local note storage as a recovery dependency,
- and custom spoke withdrawal receivers.
That is protocol law for V16.9.9.
6. MPP boundary law
MPP is present, but it must be described correctly.
MPP compatibility exists only as an adapter around the existing Parly execution engine. It does not change pool behavior, change circuit semantics, add contract-level session state, replace the current Agentic SDK execution path, or weaken AGENT-key versus RELAYER-key separation. The only valid place for MPP in V16.9.9 is the SDK/MCP boundary. Both the runbook and public docs say this directly.
That means Parly can participate in machine-payment environments without becoming a session-contract protocol.
7. Product thesis
Parly wins by doing a small number of important things exceptionally well.
It is stablecoin-only, which keeps accounting, compliance language, pool semantics, and user understanding tighter. It uses Tempo as the privacy hub, which keeps privacy state, recovery logic, and verification logic centered instead of fragmented. It treats relay privacy, deterministic recovery, and selective verification as first-class concerns. It gives developers and agents real integration surfaces. It keeps growth systems removable rather than confusing them with protocol truth.
That is what makes Parly feel like infrastructure rather than a campaign page with contracts behind it.
8. User and customer profiles
Parly is built for five major groups.
The first is the direct user: an individual, startup, operator, or team that wants to move stablecoins privately across supported routes.
The second is the relayer operator: a participant who wants to run the private execution runtime and optionally be visible to the official frontend through the registry.
The third is the developer: a backend or application engineer integrating Parly through the SDK into a service or workflow.
The fourth is the agent builder: a team that wants AI services or automated systems to use Parly through bounded MCP tools and the SDK boundary.
The fifth is the verifier or reviewer: an exchange, counterparty, auditor, finance reviewer, or compliance stakeholder who needs confidence in one disclosed payout lane without seeing the sender’s full treasury graph.
9. Supported assets and network topology
Supported assets in protocol core are:
- USDC
- USDT
Each asset is isolated into its own pool. No mixed-asset pool behavior is allowed. The runbook’s canonical trust topology fixes TempoShieldedPool for USDC and TempoShieldedPool for USDT on the hub side.
The topology is:
- Hub: Tempo
- Supported spokes: Ethereum, Arbitrum, Base, BSC
The protocol is omnichain in routing reach, but not chain-symmetric in trust model. It is explicitly hub-and-spoke with Tempo at the center.
10. Main product surfaces
10.1 App
The App is the main user-facing control surface. It handles wallet connection, authentication, recovery, shielding, execution, Parly Ledger & Compliance, scoped receipt preparation, and verification handoff. The hosted docs describe it as the primary user surface, and the UI/UX blueprint places it at the center of the navigation and operating flow.
10.2 Spoke Shield
This is the dedicated spoke-origin shielding surface for supported spoke-to-Tempo deposit flow. The launch law keeps spoke shielding as a first-class protocol feature.
10.3 Parly Ledger & Compliance
This is a named product surface. It is where users review deposits, private executions, change-note activity, and scoped payout records, and prepare one disclosed lane for verification or export. The UI/UX blueprint places it directly below the main dapp widget and defines it as the control surface for audit-ready payout proofs.
10.4 Verify
Verify is the standalone public selective-disclosure portal for one payout lane. A reviewer provides a confirmed Tempo transaction hash and child key, and Verify answers one narrow question well. The hosted docs and UI/UX blueprint are aligned on this.
10.5 Relayer Registry
The relayer registry exists for official frontend discovery and relayer public-key distribution. It is not protocol permissioning. The relayer registry PRD is explicit that protocol participation remains open even if official frontend discovery is curated.
10.6 Agentic SDK
The SDK is the developer-facing integration surface.
10.7 MCP Server
The MCP server is the bounded agent tooling surface for agent-owned execution.
10.8 Admin Vault
The Admin Vault is the hidden governance and treasury-management surface used by the founding team or authorized signers. It manages protocol revenue, signer governance, fee control, and proposal execution. The UI/UX blueprint defines it as a high-security dashboard tied strictly to the ProtocolTreasury contract.
10.9 Phase 2 growth layer
Waitlist, referrals, shield-mining analytics, leaderboard, snapshots, and OFT/TGE launch layers belong to Phase 2. They sit above protocol truth. They are not protocol law. The Phase 2 document states this hierarchy directly.
11. Core user journeys
11.1 Tempo-native shield
The user connects an EVM wallet, authenticates with deterministic signing, selects asset and amount, approves where needed, shields funds on Tempo, and receives private balance plus emitted recovery data. The UI/UX blueprint defines the five auth states and the shield flow; the launch law keeps shielding on Tempo in scope.
11.2 Spoke-to-Tempo shield
The user starts on a supported spoke, selects a supported stablecoin and amount, submits shielding from the spoke, and the finalized deposit settles into Tempo-side shielded balance. Provenance may then be shown later through indexed source-dispatch and Tempo settlement correlation. The runbook is explicit that provenance must be deterministic and GUID-based where available, not guessed through timestamps or amounts.
11.3 Same-chain private payout on Tempo
The user opens Execute, chooses recipient and amount, generates the proof, submits through private relay by default, and later has child-key-scoped compliance available.
11.4 Cross-chain private payout
The user opens Execute, selects recipient, amount, and supported destination spoke, generates the proof, the relay path handles source-side execution on Tempo, and stablecoin delivery completes toward the spoke. The runbook fixes hub cross-chain payouts to stablecoin-only and alt-fee-token-only semantics on the Tempo side.
11.5 Selective disclosure
The user opens Ledger & Compliance, finds one execution card, opens one payout scope only, previews a scoped receipt, exports the record, and gives the recipient the confirmed Tempo transaction hash plus child key so they can verify through the public Verify portal.
12. UX and brand direction
Parly should always feel premium, institutional, privacy-native, and exact.
The design language is closer to a premium financial operating system than to a meme DeFi product. The UI/UX blueprint explicitly frames the theme as “Dark Pool Elegance” meets “Billion-Dollar Fintech SaaS”, fixes the SPA architecture as Next.js, Tailwind CSS, Wagmi V2, and Waku P2P, and gives the landing page a strong institutional narrative centered on Tempo, private stablecoin infrastructure, agents, and global capital flows.
The tone should feel sharp, controlled, and operator-grade.
13. Authentication law
The UI source defines a five-phase auth sequence:
1. disconnected,
2. post-connect lock state,
3. authenticating,
4. recovering notes,
5. ready.
This staged flow matters because wallet connection and private-state access are not the same thing. A wallet being connected does not mean the enclave is unlocked. Inputs remain locked until deterministic authentication succeeds.
14. Privacy model
Parly is a privacy execution system, not an invisibility fantasy.
Its privacy model combines stablecoin-only private pools, Tempo as the hub, note-based private execution, deterministic recovery identity, encrypted note envelopes, Waku relay transport, and selective disclosure when the user intentionally reveals one payout lane. The hosted docs say plainly that recovery does not depend on browser kindness, that private state is tied to a deterministic wallet-derived recovery identity plus on-chain emitted data, and that note envelopes allow future-device recovery.
15. Recovery model
Recovery correctness must not depend on local storage.
The runbook names the authoritative recovery surface:
- wallet-derived deterministic recovery identity,
- on-chain note envelopes,
- on-chain leaves,
- on-chain nullifiers,
with local cache remaining optional, encrypted, disposable, and non-authoritative. It also preserves deterministic asymmetric sealed-box recovery for note envelopes.
In practical terms, recovery should be possible on a future device from wallet control plus indexed chain data.
16. Compliance and Verify law
Verify exists to answer one narrow question well:
Does this disclosed child key correspond to a real payout lane in the referenced Parly execution?
The Verify surface must not become a whole-batch accounting engine, a route-cost apportionment engine, or a treasury-inspection backdoor. It is a selective proof surface. The hosted docs and UI/UX blueprint both reinforce this.
17. Fee UX law
The product must separate user-facing fee explanation from protocol complexity.
The fee domains are:
- protocol fee,
- relay execution fee,
- cross-chain delivery cost,
- total fee,
- total shielded deduction.
The UX law is:
- private relay remains the default,
- same-chain routes show no cross-chain delivery cost,
- private-relay cross-chain routes must not double-count route delivery cost,
- self-relay cross-chain flows may show more raw route-cost detail at execute time,
- Ledger, receipts, and Verify must stay cleaner and more scoped than raw execution economics.
18. Failure and recovery UX law
Shield recovery belongs in the Shield surface, not in Ledger.
Execute recovery belongs in the Execute surface, not in Ledger.
Ledger is a record surface, not a recovery console.
The interface must keep submitted, pending, confirmed, and failed visibly distinct. The hosted docs state this directly, and the UI/UX blueprint mirrors the same honesty rule.
19. Phase 2 boundary
Phase 2 is the growth and token-launch layer. It is not the protocol.
The Phase 2 document defines it as the marketing, waitlist, referral, leaderboard, snapshot, and OFT launch layer, and fixes the architectural hierarchy as:
Protocol → Indexer → Campaign Worker → Waitlist Profile Merge → Leaderboard → Snapshot → TGE / OFT UX
That means Phase 2 can be powerful, but it must remain removable.
20. Success metrics
The product should optimize for:
- shield completion rate,
- private execution completion rate,
- relay success rate,
- recovery success rate,
- Verify completion rate,
- average time to finality,
- batch execution usability,
- scoped receipt export rate,
- relayer readiness,
- SDK adoption,
- MCP and agent-workflow adoption.
21. Non-goals
Parly is not:
- a general-purpose swap terminal,
- a scheduler,
- a protocol-owned relay monopoly,
- a volatile-asset privacy router,
- a passkey-privacy product,
- a whole-batch accounting engine,
- or a growth campaign disguised as protocol law.

---
Part II — Technical Requirements Document (TRD)
22. Technical objective
The technical system must ship Parly as a real privacy-execution infrastructure stack rather than as a frontend demo.
The runbook is explicit that V16.9.9 is a stronger static production candidate, not a live deployment proof by itself. Real deployability still depends on compiling the actual repo, generating the exact verifier from the exact final zkey, wiring the real OFTs, endpoints, libraries, and DVNs, funding and booting the real services, and passing the acceptance matrix.
23. High-level architecture
Parly is a multi-layer privacy execution stack rather than a single frontend with hidden assumptions.
At a high level, the system includes:
- a web application,
- Solidity contracts,
- zero-knowledge circuits,
- a relayer runtime,
- an indexer,
- an optional campaign worker,
- an SDK,
- an MCP server,
- and supporting infrastructure such as PostgreSQL and deployment/runtime services.
Each layer has a distinct responsibility. Recovery logic, indexing, relay execution, agent tooling, treasury control, and analytics must not collapse into one trust domain. The hosted docs and runbook are aligned on this.
24. Repository and distribution model
The latest runbook defines one private canonical monorepo and three public distribution surfaces:
- a public relayer repo for operators,
- a public SDK repo or package for developers,
- and a public MCP repo for agent integrations.
Public repos are distribution layers. They are not alternate sources of truth. Operators should not need to clone the private canonical monorepo once the public relayer surface exists. The hosted docs and the repo-export workstream are intended to harmonize with this model.
This matters because Parly is infrastructure, not just a frontend.
25. Canonical monorepo shape
The canonical runbook repo layout includes:
- apps/web
- apps/relayer
- apps/indexer
- apps/campaign-worker
- apps/mcp-server
- packages/env-utils
- packages/protocol-abis
- packages/shared-types
- packages/crypto-utils
- packages/registry-client
- packages/contracts
- packages/circuits
- packages/sdk
- tools/repo-export
- release/versions.json
- and generated public export trees for relayer, SDK, and MCP.
26. Recommended concrete tech stack
26.1 Frontend / App
- Next.js
- Tailwind CSS
- Wagmi v2
- viem
- injected EVM-wallet support
- browser-safe WASM handling for proving artifacts
The UI/UX blueprint explicitly fixes the frontend as a single-page application on Next.js, Tailwind CSS, Wagmi V2, and Waku P2P.
26.2 Private relay transport
- Waku P2P
Waku is the encrypted immediate relay delivery layer. It is not durable job storage and not a scheduling system. That matches both the runbook and the hosted docs.
26.3 Smart contracts
- Solidity
- Foundry for build, test, and deployment scripting
26.4 Circuits / proving
- Circom
- snarkjs
- Groth16
- Poseidon-based hashing support
- fixed Merkle tree tooling
26.5 Indexer
- Ponder
- viem
The runbook defines a corrected Ponder indexer that materializes deposits, leaves, envelopes, batch metadata, source-chain shield dispatches, and Tempo-side ingress settlement anchors, and states that nothing private is inferred.
26.6 Relayer runtime
- TypeScript
- viem
- @waku/sdk
- libsodium-wrappers-sumo
26.7 SDK
- TypeScript package
26.8 MCP server
- Model Context Protocol server stack
- @parly/sdk
26.9 Application database
- PostgreSQL
- Railway for managed deployment of app-owned relational state
The Phase 2 architecture explicitly assigns users, secondary wallets, wallet-link nonces, ghost depositors, TGE snapshots, snapshot allocations, leaderboard materialization, and admin sync metadata to Railway PostgreSQL.
26.10 Admin sync surface
- Google Sheets API as an operational sync destination
The Phase 2 document makes clear that Google Sheets is fed from the merged leaderboard projection and is not a source of truth.
26.11 RPC provider layer
- reliable RPC endpoints for Tempo and all supported spokes
- environment-driven configuration
- dRPC may be standardized operationally if the team wants one provider layer, but provider choice should remain an infrastructure decision rather than protocol law
27. Responsibilities by codebase
27.1 apps/web
Owns:
- app UI,
- wallet connection,
- deterministic auth flow,
- note recovery orchestration,
- Shield UI,
- Execute UI,
- Ledger & Compliance UI,
- Verify UI,
- relayer registry UI surfaces,
- Admin Vault UI,
- CSV import/export,
- proof orchestration,
- Waku payload preparation,
- receipt rendering and export,
- supported-network enforcement,
- user-visible execution-state honesty.
27.2 apps/relayer
Owns:
- Waku message intake,
- encrypted bundle handling,
- sealed-box decryption for addressed bundles only,
- profitability checks,
- bounded chain submission,
- pending-state persistence,
- pending reconciliation,
- relayer health and activity logging.
27.3 apps/indexer
Owns:
- deposit indexing,
- execution or batch-withdrawal indexing,
- nullifier and commitment indexing,
- provenance joins,
- public query surfaces for history, Verify, dashboard, and analytics.
27.4 apps/campaign-worker
Owns:
- shield-mining analytics only,
- leaderboard sync support,
- ghost depositor merge support,
- snapshot assistance,
- cron-based analytics sync.
It must remain safe to delete without breaking protocol correctness. The Phase 2 document is explicit that the worker should use public deposit history only, integer accounting only, and analytics rows only.
27.5 apps/mcp-server
Owns:
- bounded agent tools,
- SDK wrapping for agent execution,
- AGENT-only key boundary,
- optional MPP adapter exposure when enabled.
27.6 packages/contracts
Owns:
- pool contracts,
- hub composer,
- spoke gateways,
- verifier integration,
- treasury and multisig contracts,
- peer and endpoint wiring scripts,
- asset isolation and payout rules.
27.7 packages/circuits
Owns:
- circuit source,
- witness harness,
- proving artifacts,
- proof-related scripts,
- deterministic public-signal shape.
27.8 packages/sdk
Owns:
- typed protocol helpers,
- note recovery helpers,
- envelope handling,
- execution helpers,
- relay bundle creation helpers,
- MPP-compatible adapter layer.
27.9 Shared packages
The runbook also formalizes reusable shared boundaries:
- @parly/env-utils
- @parly/protocol-abis
- @parly/shared-types
- @parly/crypto-utils
- @parly/registry-client
28. Canonical product rules the code must enforce
The code must enforce all of the following:
1. Tempo is the hub.
2. USDC and USDT are isolated pools.
3. Supported spoke shielding into Tempo exists.
4. Same-chain private payout on Tempo exists.
5. Cross-chain private payout exists only for supported stablecoin routes.
6. One proof supports at most 10 outputs.
7. Recovery comes from deterministic wallet-derived identity plus note envelopes and chain history.
8. Waku is for immediate private relay transport, not durable job storage.
9. Campaign analytics are removable.
10. Self-relay is fallback, not default product posture.
11. MPP exists only at the SDK/MCP boundary.
29. Onchain system design
29.1 Pool architecture
Parly uses isolated stablecoin pools. USDC and USDT are not mixed.
Each pool must enforce:
- asset isolation,
- deposit correctness,
- ingress deposit correctness,
- note insertion,
- nullifier protection,
- change-note insertion,
- batch withdraw execution,
- fee movement,
- and event emission for history, recovery, and compliance surfaces.
29.2 Hub-side architecture
Tempo hosts the privacy hub. Hub-side core components include the shielded pools and the hub composer.
29.3 Spoke-side architecture
Each supported spoke has per-asset gateway logic for routing shield-origin actions into Tempo.
29.4 Treasury and governance
The treasury layer supports signer management, fee updates, proposal execution, proposal cancel or revoke behavior, and revenue withdrawal.
30. Canonical trust topology
The runbook fixes one valid trust topology.
Hub side
- TempoShieldedPool for USDC
- TempoShieldedPool for USDT
- ParlyHubComposer
- one ERC20-adapter-compatible LayerZero endpoint per asset on Tempo
Spoke side
- one ERC20-adapter-compatible LayerZero endpoint per asset per spoke chain
- one ParlySpokeGateway per asset per chain for spoke-to-Tempo shielding
- no custom spoke withdrawal receiver
Compose trust anchor
- the hub composer trusts the compose sender surfaced by LayerZero compose wrapping
- in this topology, the contract calling send(...) is the spoke gateway, so the trusted source is the spoke gateway, not the spoke OFT.
31. Fee-mode law
The runbook fixes one valid hub payout fee rule:
Tempo hub cross-chain payouts are alt-fee-token only.
That means:
- quote path uses payInLzToken = true,
- execution uses alt-fee approval flow only,
- hub send path always uses msg.value == 0,
- the hub payout path does not mix native-fee and alt-fee semantics.
Spoke-to-hub shielding may support either native-fee or alt-fee mode on spokes depending on environment, but quote and send must use the same flag.
32. Circuit and proof model
Parly uses a bounded proof model with a maximum of 10 outputs per proof. The launch surface keeps single send and batch send up to 10 outputs per proof.
The canonical relay payload freezes these cardinalities:
- 50 public signals,
- 10 recipients,
- 10 amounts,
- 10 destination EIDs,
- 10 LayerZero options entries.
That matters because frontend, relayer, SDK, and contracts must all agree on payload shape. Private execution fails if those layers drift.
33. Recovery and note-encryption model
Parly uses deterministic wallet-derived recovery identity plus asymmetric sealed-box note envelopes.
In plain language, anyone can create an envelope for the intended owner, but only the holder of the matching recovery private key can open it later. That allows future-device recovery without making local browser storage the authority. The runbook preserves deterministic asymmetric sealed-box recovery, and the public docs reinforce that browser storage is convenience only, not the recovery authority.
This recovery model is one of the protocol’s core technical differentiators.
34. Data model and truth boundaries
Parly uses multiple truth layers.
34.1 Onchain truth
Onchain truth includes:
- deposits,
- note commitments,
- nullifiers,
- note envelopes,
- payout execution anchors,
- fee state,
- and routing facts actually emitted by contracts.
34.2 Indexed public truth
Ponder-derived indexed truth includes:
- deposit rows,
- execution rows,
- provenance joins where available,
- Verify queries,
- public history,
- and analytics surfaces.
34.3 App-owned relational truth
Railway/PostgreSQL holds:
- campaign profiles,
- linked wallets,
- ghost depositor mappings,
- leaderboard projections,
- snapshots,
- and service-owned admin metadata.
34.4 Local reconstructed state
The browser or SDK may locally reconstruct:
- decrypted notes,
- live spendable notes,
- local receipt-preview context,
- and optional encrypted cache data.
That local layer is never the authoritative recovery source.
35. Ponder explained in plain English
Ponder is Parly’s public fact engine.
It turns raw blockchain events into structured product truth for:
- history,
- verification,
- deterministic provenance enrichment,
- analytics,
- TVL and surface metrics,
- and parts of private-state reconstruction.
Without Ponder, Parly would be much weaker at rendering history, rebuilding recoverable state, powering Verify, driving dashboard metrics, and joining real spoke-origin provenance. The indexer section of the runbook makes this explicit by keeping the schema minimal, public, deterministic, and GUID-anchored where provenance exists.
36. Railway explained in plain English
Railway is where Parly keeps service-owned relational state that should not live onchain and does not belong in the public indexer.
That includes user profiles, secondary-wallet linkage, ghost depositor records, snapshots, leaderboard materialization, and admin-sync metadata for Phase 2 systems. The Phase 2 master document makes that split explicit.
The simplest explanation is:
Contracts move money. Ponder tracks public chain facts. Railway holds app-owned profile and campaign data.
37. Google Sheets boundary
Google Sheets is not a source of truth. It is an operational sync destination.
The Phase 2 architecture explicitly says Sheets syncs from the merged leaderboard projection and exists to fix drift rather than create another source of truth.
38. Indexing and provenance model
The canonical provenance rule is strict: provenance should be shown only when real source and Tempo-side data can be joined deterministically. The runbook says source provenance exists only when a real source dispatch event and a real Tempo settlement event share the same LayerZero GUID. It explicitly rejects fuzzy inference.
This keeps Ledger, Verify, and compliance surfaces honest.
39. Relayer model
39.1 What the relayer does
A relayer receives encrypted bundles, decrypts only bundles addressed to its public key, validates payload structure and economics, submits valid executions, and reconciles uncertain outcomes later. The hosted docs describe the relayer as a real runtime service, not a toy script.
39.2 What the relayer is not
A relayer is not a treasury controller, not a protocol owner, and not the protocol itself.
39.3 Registry relationship
The registry is for official frontend discovery and key distribution. It does not permission protocol participation.
39.4 Registry operating rule
The official relayer registry supports a maximum of 10 live relayers at once. Registration is automatic if validation passes and a slot is available, and inactive relayers fall out of discovery without requiring a moderation queue.
39.5 Registry runtime rule
Official frontend discovery must be backend-backed in production. The registry PRD states that runtime registry is canonical in production, frontend should fetch verified live relayers from backend, and env registry may remain only as local or dev fallback.
39.6 Activity-tracking rule
Successful relay activity must update last_success_at and last_seen_at, either through direct DB update or an authenticated heartbeat endpoint. The registry PRD explicitly requires this for inactive-slot turnover to work correctly.
40. Waku requirements
Waku is used for:
- encrypted immediate relay delivery only.
It is not used for:
- scheduling,
- durable guaranteed storage,
- future retrieval assumptions,
- or protocol job persistence.
This boundary matters because Parly explicitly killed scheduling, intent vaults, and REST intent APIs from launch scope.
41. SDK and MCP model
41.1 SDK requirements
The SDK must support:
- deterministic authentication-message handling,
- recovery identity derivation,
- note-envelope handling,
- note reconstruction,
- private execution construction,
- exact approval handling,
- outcome handling,
- and the MPP-compatible adapter boundary.
The runbook’s SDK materials state that the SDK must mirror protocol reality rather than hiding it behind fake abstraction. It should preserve the deterministic auth message, asymmetric recovery-key derivation, note-envelope format, proving path, direct pool execution path, exact approval flow, and MPP-compatible adapter only at the SDK boundary.
41.2 MCP requirements
The MCP server must:
- expose bounded tools only,
- operate under AGENT-only key scope,
- never inherit relayer execution authority,
- never inherit treasury authority,
- and route its real work through the SDK.
41.3 MPP boundary rule
MPP compatibility is allowed only as a wrapper around the existing execution engine at the SDK/MCP boundary. It does not create new contract law.
42. Web app requirements
42.1 Authentication flow
The app must preserve a staged flow:
1. connect wallet,
2. authenticate,
3. recover private state,
4. ready state.
This mirrors the five-phase UI auth model.
42.2 Shield surface
The Shield surface must support origin-network choice, asset choice, amount entry, approval handling, shield submission, success state, failure state, and honest recovery or finality states.
42.3 Execute surface
The Execute surface must support:
- single send,
- batch send,
- CSV import,
- private relay as default,
- self-relay fallback,
- fee display,
- explicit pending handling,
- and clean retry rules.
42.4 Ledger & Compliance surface
This surface sits below the main execution widget and must support:
- deposits,
- executions,
- change-note-related history,
- payout-scope review,
- export flows,
- and truth-labeled data display. The UI/UX blueprint defines truth badges such as Chain-Indexed, Calldata-Decoded, Local Scope, and Provenance.
42.5 Verify handoff
The app must support clean handoff from scoped-receipt preparation to public Verify.
42.6 Compliance pre-flight
The UI/UX blueprint also requires:
- silent edge geo-fencing for sanctioned IPs,
- an enterprise attestation modal on first successful wallet connection,
- supported-network enforcement,
- and a switch-to-supported-network guard state.
42.7 Shield-mining indicator where enabled
Where Phase 2 growth systems are enabled, the app may show a shield-mining indicator based on public deposit units, but the underlying analytics must remain removable and must never become protocol correctness. The UI/UX blueprint includes a shield-mining pill, while the Phase 2 worker keeps shield mining analytics-only.
43. Admin and treasury requirements
The Admin Vault must support:
- signer visibility,
- signer management,
- fee visibility,
- revenue monitoring,
- proposal creation,
- proposal confirmation,
- proposal execution,
- and revoke or cancel flows where applicable.
The UI/UX blueprint defines four immutable admin action types:
- Withdraw Profit,
- Change Global Fee,
- Manage Signers,
- Cancel Transaction.
It also requires strict signer-based access control and a full-screen unauthorized block for non-signers.
The Admin Vault must remain a high-trust operational surface rather than a public product surface.
44. Phase 2 technical boundary
Phase 2 must be described correctly across all teams.
It is the marketing, waitlist, referral, leaderboard, snapshot, and OFT/TGE layer. It is not the protocol. The Phase 2 master document fixes the hierarchy as Protocol → Indexer → Campaign Worker → Waitlist Profile Merge → Leaderboard → Snapshot → TGE / OFT UX, assigns Railway to relational campaign state, keeps campaign worker analytics-only, and places Google Sheets as an admin output only.
45. Deployment model and environment requirements
A real Parly deployment is not frontend-only.
Depending on the operated surfaces, the full deployment can include:
- web app,
- contracts,
- relayer runtime,
- pending reconciler,
- indexer,
- Verify route,
- relayer registry routes,
- Admin Vault,
- MCP server,
- Railway PostgreSQL,
- optional Google Sheets admin sync,
- and optional campaign-worker services.
The hosted docs say plainly that Parly is not a frontend-only product and that a realistic deployment includes web app, relayer, indexer, MCP server, and optional analytics services. The runbook also gives an explicit zero-trust launch order that includes building circuits, deploying contracts, wiring peers, starting indexer, starting frontend, generating relayer keys, registering relayers, starting the pending reconciler, starting campaign worker, and then starting the MCP server before final acceptance rehearsals.
46. Environment configuration requirements
At minimum, environment-driven configuration must cover:
- Tempo RPC,
- supported spoke RPCs,
- Tempo chain ID,
- LayerZero EIDs,
- token addresses,
- pool addresses,
- gateway addresses,
- hub composer address,
- endpoint, peer, and DVN configuration,
- Waku bootstrap values,
- relayer execution key,
- relayer sealed-box private key,
- AGENT key,
- treasury signer keys,
- campaign-worker database config,
- Railway database URL,
- optional Google Sheets admin-sync credentials,
- and MPP adapter flags and metadata where enabled.
The runbook’s environment model is explicitly fail-fast for critical runtime values such as chain IDs and EIDs.
47. Security requirements
The system must preserve the following laws:
No passkey privacy mode in protocol core.
No scheduling in launch scope.
No intent vaults.
No swap routing.
No volatile-token payout routing.
No protocol-owned relayer monopoly.
No browser-local-only recovery assumptions.
No shared-key collapse between AGENT, RELAYER, and treasury roles.
No whole-batch disclosure through Verify.
No unbounded output cardinality per proof.
No fake provenance fabricated through heuristics.
No growth-layer dependency for protocol correctness.
48. Observability and operational requirements
48.1 Web app
Monitor:
- auth success and failure,
- proof-generation errors,
- shield attempts,
- execute attempts,
- Verify requests,
- receipt exports,
- recovery-reconstruction failures,
- and network mismatch events.
48.2 Relayer
Monitor:
- bundles received,
- bundles decrypted,
- profitability pass or fail,
- tx submitted,
- tx pending,
- tx failed,
- and tx finalized.
48.3 Indexer
Monitor:
- sync lag,
- event-ingestion health,
- provenance-join health,
- and query correctness.
48.4 Campaign worker
Monitor:
- sync lag,
- leaderboard materialization,
- snapshot generation,
- ghost resolution,
- and admin-export health.
49. Acceptance criteria
A Parly core implementation is acceptable only if all of the following are true:
1. Users can shield USDC and USDT on Tempo.
2. Users can shield from supported spokes into Tempo.
3. Users can execute private same-chain payouts on Tempo.
4. Users can execute private cross-chain stablecoin-only payouts to supported spokes.
5. One proof never exceeds 10 outputs.
6. Recovery works from deterministic wallet-derived identity plus chain-visible recovery data.
7. Verify can confirm one disclosed payout lane.
8. Ledger & Compliance can render deposits, executions, and scoped proof flows cleanly.
9. Private relay works as the default path.
10. Self-relay remains a real fallback.
11. Growth analytics can be removed without breaking protocol correctness.
12. SDK can execute bounded private settlement flows correctly.
13. MCP can expose bounded tools without privilege escalation.
14. Public distribution surfaces for relayer, SDK, and MCP remain role-correct and do not require cloning the private canonical monorepo.
15. Repo-root typecheck and build pass on the assembled implementation.
16. Acceptance smoke and adversarial checks pass before launch sign-off.
50. The single most important operational sentence
Parly is infrastructure, not just a frontend.
A real deployment may include contracts, web app, relayer, pending reconciler, indexer, Verify surface, registry routes, Admin Vault, MCP server, Railway PostgreSQL, campaign worker where enabled, and Google Sheets sync where enabled. If any team forgets that, the product will be documented incorrectly and built with the wrong assumptions.

---
51. Final cross-team summary
Parly is private stablecoin omnichain infrastructure for the agentic economy.
It lets users, teams, services, and machine-driven systems shield stablecoins into private balance on Tempo, execute private same-chain and cross-chain payouts, recover state deterministically from wallet-derived cryptographic identity plus chain-visible recovery data, and prove one disclosed payout lane when verification is required.
Under the hood, Parly combines stablecoin-only privacy pools, Tempo-centered hub-and-spoke routing, Waku-powered private relay transport, Ponder-indexed public fact surfaces, selective verification, a real relayer runtime, Railway-backed app state where needed, a public SDK, and an agent-bound MCP server into one focused settlement stack.
It is not a scheduler. It is not a swap terminal. It is not a protocol-owned relay monopoly. It is not a growth campaign pretending to be protocol law.
It is a serious privacy execution system with real operator, developer, and agent surfaces.
