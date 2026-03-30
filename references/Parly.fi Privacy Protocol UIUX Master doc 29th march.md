Parly.fi Privacy Protocol UI/UX Master Blueprint (V16.9.9 Canonical Edition)
Date: 29th March 2026  
Architecture: Single Page Application (Next.js / Tailwind CSS / Wagmi V2 / Waku P2P)  
Core Theme: "Dark Pool Elegance" meets "Billion-Dollar Fintech SaaS."
- Primary Background: Deep onyx/slate (#020617) creating a sleek, premium terminal feel.
- Text: Stark white (#FFFFFF) for pristine, high-contrast readability.
- Accent: Electric neon green (#39FF14) reserved strictly for primary execution actions, Shield Mining point accruals, and live network indicators.
- Secondary Accent: Deep corporate blue (#2563EB) for informational tooltips and trusted compliance branding.

---
Part 1: Global UI Components, Connect States & Pre-Flight
Reusable components engineered to elegantly handle Web3 friction, omnichain delays, strict EVM authentication, and the L1 dual-layer compliance engine.
1. The Global Navigation Bar (Sticky Top)
- Left: Parly.fi Logo (Minimalist typography with a glowing neon green dot).
- Center Links: App | Agentic SDK (MCP) | Parly Verify | Docs
- Right (The Growth Engine & Status Hub):
  - TVL Ticker: Sleek, minimalist counter. Populated State: Total Shielded: $142,505,000 (Updates dynamically via Ponder Indexer).
  - Shield Mining Status (If Authenticated): A glowing gold/neon pill. Populated State: 🏆 14,050 $PARLY (Hovering reveals: "Earn $PARLY points instantly when you deposit." Clicking opens the Airdrop campaign modal).
  - Network Switcher Dropdown: Displays the current chain icon. Populated State: [ 🔵 Base ]. Clicking opens a sleek dropdown to switch RPC networks natively.
  - Connect Wallet Button: Dynamically reflects the 5 Authentication States (detailed below).
    
2. The 5-Phase Authentication Flow
V16.9.9 enforces strict deterministic identity via EIP-191 signatures. The UI elegantly guides the user through these 5 precise states:
- State 1: Disconnected
  - UI: Main button reads [ Connect EVM Wallet ]. Widget inputs are frosted/greyed out.
- State 2: Post-connect lock state
  - UI: Wallet is linked (*Sample:* 0x71C...9A23), but the enclave is locked under frosted glass.
  - Widget Button: [ Authenticate Wallet ]
- State 3: Authenticating
  - UI: User clicks Authenticate. Wallet prompts an EIP-191 signature.
  - Widget Button: [ Awaiting Signature... ]
- State 4: Recovering notes
  - UI: Signature succeeds. The frontend silently queries the Ponder Indexer to rebuild the user's UTXO tree.
  - Recovery Loading Copy: [ ⏳ Recovering shielded state from chain... ]
- State 5: Ready
  - UI: Shielded balance populates. Widget unlocks. User is fully authenticated and ready to transact.
    
3. Zero-Trust Compliance Pre-Flight (Frontend & On-Chain)
- Edge Geo-Fencing (Silent): Blocks OFAC-sanctioned IPs natively at the CDN level.
- The Enterprise Attestation Modal: Appears on the first successful wallet connection.
  - Headline: Protocol Compliance & Agentic Attestation
  - Copy: "Parly provides non-custodial, omnichain privacy infrastructure for global capital flows. To proceed, confirm your jurisdictional eligibility."
  - Checkbox: [ ] I accept the Parly.fi Terms of Service. I acknowledge this is a 100% non-custodial protocol. I authorize decentralized Relayers for private executions, and I attest under penalty of law that I am not operating from a sanctioned jurisdiction.
  - Action: [ Enter Parly Hub ] (Disabled until checked).
    
4. "Switch to Supported Network" State (The Omnichain Guard)
- Trigger: User connects via an unsupported chain (e.g., Optimism, Polygon) or attempts a Tempo-native execution while connected to a Spoke chain.
- The UI Overlay: The main widget blurs out, prioritizing an elegant, center-screen modal.
- Headline: Unsupported Origin Network
- Copy: "Parly.fi's canonical architecture routes exclusively through the Tempo Hub. Supported origin spokes include Ethereum, Arbitrum, Base, and BSC."
- Dynamic Action: [ Switch to Tempo L1 ] or [ Switch to Arbitrum ] *(Detects intended action and natively triggers wallet_switchEthereumChain to seamlessly bridge the user back to a supported environment).

---
Part 2: The Landing Page (The Billion-Dollar Institutional Narrative)
UI Copy is engineered for massive, bullish, enterprise-grade impact. It establishes Parly as the undisputed financial settlement layer for the 2026 Agentic Economy.
Section 1: Hero (Above the Fold)
- Status Badge (Top Center): 🟠 Live on Tempo Moderato (Chain ID: 4217)
- Headline: Private Stablecoin Omnichain Infrastructure for the Agentic Economy.
- Sub-headline: Shield your strategy. Empower autonomous agents. Settle capital flows globally with invisible, compliant execution.
- Body Text: "Stablecoins process trillions in agentic payments and enterprise settlements, yet public ledgers expose logic at machine speed. Parly acts as the ultimate omnichain privacy layer—utilizing Tempo as a central hub to route and shield stablecoin liquidity from every major EVM without fragmentation."
- CTAs (Bullish & Direct - Strict Ordering):
  - Primary (Vibrant Neon Green): [ Enter the Privacy Pool ]
  - Secondary (Outline/Ghost): [ Read the Whitepaper ]
    
Section 1.5: The Institutional Trust Bar (Subtle Carousel)
Tempo L1 (Native Settlement) | Stripe MPP (Machine Payments) | LayerZero V2 (Omnichain) | Anthropic MCP (Agent Tooling) | Waku P2P (MEV Protection)

Section 2: Core Market Advantages (6-Column Slider)
- 1. The Stablecoin Foundation: "Zero-knowledge routing strictly for USDC and USDT. Deep, permanent anonymity sets without the regulatory friction, slippage, or liquidity bleed of volatile altcoins."
- 2. MPP & Agentic Execution: "Built for the Machine Payments Protocol (MPP). Autonomous AI agents can execute shielded micro-transactions natively. Protect machine autonomy and keep strategic enterprise logic entirely off public chains."
- 3. Unified Omnichain Hub: "Fragmented liquidity creates metadata leaks. Parly funnels major EVMs (Ethereum, Arbitrum, Base, BSC) into one secure privacy pool on Tempo. One settlement layer, zero compromise."
- 4. Canonical Zero-Trust Recovery: "Your funds are yours. Parly utilizes deterministic EIP-191 signatures and asymmetric libsodium sealed-box cryptography. Recover your exact private state from on-chain data alone-"No local storage required."
- 5. Decentralized Waku Execution: "Total MEV protection. Encrypted ZK payloads are routed via the Waku P2P network directly to specific relayers, rendering your capital flows completely invisible to mempool snipers."
- 6. Deterministic Identity: "No vendor lock-in. No fragile dependencies. Your identity is mathematically secured by your existing EVM wallet, guaranteeing absolute fidelity and fund recovery across any device in the world."
  
Section 3: The 2026 Agentic Use Cases (7-Card Grid)
- Headline: Public ledgers expose strategy. Parly protects it.
- Card 1: Autonomous AI Supply Chains (MCP). "Deployed AI models autonomously negotiate and pay for compute, API access, and data feeds via Machine Payment Protocol standards without broadcasting their corporate sponsors, prompt volumes, or supply chains to competitors."
- Card 2: Autonomous Yield Repatriation. "As autonomous AI agents generate yield or service revenue, their wallets become targets for heuristic analysis. Parly allows deployed agents to natively sweep stablecoin profits back to the master corporate treasury without establishing a public metadata link between the machine and its creator."
- Card 3: B2B Omnichain Batch Payroll. "Paying global contractors shouldn't broadcast your corporate burn rate. Parly processes optimized batches of exactly 10 recipients per Zero-Knowledge proof, effortlessly distributing stablecoins across any supported network while permanently hiding salary bands."
- Card 4: Institutional Treasury Obfuscation. "Broadcasting massive stablecoin positions invites front-running and OTC mimicking. Shield your treasury movements to keep institutional accumulation completely invisible to whale watchers and tracking algorithms."
- Card 5: Enterprise Auditability (Child Keys). "Privacy does not mean evading audits. Instantly generate cryptographic Child Keys to prove a single, specific payment to authorities or vendors without exposing your broader treasury or batch execution data."
- Card 6: Algorithmic Dark-Pool Bidding. "In 2026, agents negotiate with agents. Parly provides the dark-pool liquidity necessary for autonomous procurement bots to settle B2B bids without exposing corporate pricing ceilings to the public ledger."
- Card 7: DAO Operational Stealth. "Protect decentralized organizations from hostile forks and predatory governance attacks by shielding treasury runway and operational capital flows from the public eye."
  
Section 4: The Airdrop & Shield Mining Campaign (Neon Highlight Section)
- Headline: The $PARLY Token. The Governance Standard for the Privacy Economy.
- Body: With a strictly hard-capped total supply of 1,000,000 $PARLY, scarcity is cryptographically guaranteed from day one. $PARLY empowers holders to govern fee parameters and direct omnichain liquidity across the agentic network. Secure your allocation before the TGE.
- The Hype Box (Bordered in Neon Green, Glowing Effect):
  - 🔴 Season 1 Growth Campaign: LIVE
  - "We reward the early adopters. Deposit stablecoins today to actively mine $PARLY points based on your shielded deposits. Refer to your network to grow your genesis allocation."
  - Primary Button (Massive & Neon): [ Join the Airdrop Campaign ]
  - Secondary Button (Subtle Outline): [ Start Shield Mining ] (Links directly to the Dapp deposit tab).
Section 5: Frequently Asked Questions
- Layout: Clean accordion layout. Clicking a question smoothly expands the answer box.
- Q1: What blockchains are currently supported?
  - A: We are currently Live on the Tempo Moderato network. Our canonical V16.9.9 architecture natively supports Omnichain routing from Arbitrum, Base, BSC, and Ethereum directly into the Tempo Hub.
- Q2: How does Parly interact with AI and the Machine Payments Protocol (MPP)?
  - A: Parly is natively integrated with Tempo/Stripe's MPP and Anthropic's MCP (Model Context Protocol). This allows autonomous AI agents to use the Parly SDK as a native "Tool" to execute private, zero-knowledge stablecoin settlements for APIs and compute, without exposing their underlying logical operations or corporate sponsors.
- Q3: Do I need native gas tokens (ETH/BNB) on my destination wallet to withdraw?
  - A: Absolutely not. Parly.fi utilizes a strictly "Gasless" architecture via decentralized relayers. The relayer pays the destination network gas for you, allowing you to withdraw to a completely empty, brand new wallet.
- Q4: How does Parly.fi maintain regulatory compliance?
  - A: Privacy does not mean hiding from audits. Through our "Parly Verify" portal, users can instantly generate a mathematical PDF receipt using a specific Child Key. This proves the exact origin of a single payment to an exchange or tax authority without exposing the rest of your treasury.
- Q5: Are my private keys or viewing keys stored on your servers?
  - A: No. Parly never stores, sees, or transmits your private keys or cryptographic viewing keys. Your existing Web3 wallet signs a deterministic EIP-191 message, and our frontend uses that signature to silently generate your Zero-Knowledge secrets locally in your browser's RAM. We operate a 100% zero-trust architecture.
- Q6: How do I earn $PARLY points?
  - A: Participate in our Shield Mining campaign! You earn $PARLY points instantly when you deposit USDC or USDT into the privacy pool. You can also refer enterprise clients using your unique link to multiply your allocation ahead of the Token Generation Event.
    
Section 6: Global Footer
- Layout: Minimalist footer at the absolute bottom.
- Links: X (Twitter), Discord, GitHub, Agentic SDK Docs, Whitepaper, Terms of Service.
- General Disclaimer: "Parly.fi is a decentralized smart contract protocol. Interacting with smart contracts carries inherent risks. Parly.fi does not custody your funds. Use it at your own risk. Not financial advice."

---
Part 3: The Dapp Widget (The Canonical Omnichain Execution Engine)
This widget sits permanently on the /app route. It mathematically enforces V16.9.9's canonical rules (stablecoin-only payouts, rigid 10-slot batch padding, deterministic EIP-191 signing, Waku payloads).
State 1: Locked (Post-Connect)
- UI State: If connected but not authenticated, a frosted-glass overlay covers inputs.
- Headline: Synchronize Cryptographic State
- Button: [ Authenticate Wallet ]
Tab 1: Shield (The Deposit Flow & Shield Mining Engine)
- Omnichain Dropdowns: Origin Network (Tempo, Ethereum, Arbitrum, Base, BSC). Asset (USDC | USDT).
- Input Section:
  - Top Right: Wallet Balance: 15,250.00 USDC (Clicking MAX auto-fills).
  - Input Field: 5000.00 (Huge typography. Regex enforced to max 6 decimals; negative numbers strictly blocked).
- The Quick-Fill Row (High-Conversion Element):
  - Sleek, clickable pills below the input: [ 100 ] | [ 1,000 ] | [ 10,000 ] | [ 50,000 ]. Clicking one instantly populates the input field.
- Mining Microcopy (Pulsing Gold/Neon below the Quick-Fills):  ✨ Shielding 5,000 USDC earns you 50 $PARLY Points! (This number dynamically updates as the user types).
- UX Nudge: 💡 Tip: Use standard round figures (e.g., 1,000 or 5,000) for maximum anonymity against chain-analysis heuristics.
- Widget Footer (Info Row): Shielded Balance: 150,000.00 USDC
  - 🏆 1,500 $PARLY Points Earned -> [ View Leaderboard ] (Clickable text routing to the waitlist page).
- Dynamic Button States (Strict V16.9.9 Sequencing):
  - Empty: Enter Amount (Disabled/Grey).
  - Needs Approval: Approve USDC (Active/Blue) -> Waits for exact on-chain receipt.
  - Approved: Shield Funds (Active/Neon Green).
  - Processing: Shielding...(Disabled).
  - Deferred Ingress State: Use this when the source shield transaction landed, but Tempo settlement has not finalized yet.
    - Shield Requires Recovery
      - Your shield request reached the source chain, but final Tempo ingress has not completed yet.
    - Retry Shielding (Primary)
    - Claim Refund (Secondary)
  - Final Failure State: Use this when shielding failed cleanly and no deferred recovery path exists.
    - Shield Failed
      - Our shield request did not complete successfully.
    - Retry Shielding (Primary)
- Success State: Input clears.
  - Toast 1: ✅ 5,000 USDC Shielded Successfully.
  - Toast 2: 🏆 You've earned 50 $PARLY!
  - Dev Note: UI dynamically increments the internal nonce, allowing continuous multiple deposits without re-signing.
Tab 2: Execute (The Private Send & Batch Flow)
This tab handles Single Sends and Batch Sends. Outputs are purely stablecoins.
- Mode Selector (Pill Toggle): [ Single Send ] | [ Batch Execution ]
- Relay Mode : Default: Private Relay Active
  - Advanced settings 
    - User toggles between [ Private Relay ] and [ Self-Relay ].
  - (i) Tooltip: Default State Copy: Parly routes execution through the private relay path by default for stronger privacy and smoother settlement.
   (i)Tooltip: Manual Fallback State Copy: This execution will be sent directly from your wallet on Tempo. This path is lower privacy and intended for manual recovery or fallback.
  Warning Card
  - Title: Manual Self-Relay Enabled
  - Body: This execution will be submitted directly from your wallet on Tempo
  - Requirement Chips:
    - Switch to Tempo Network
    - Use the same authenticated wallet
    - Ensure the wallet is ready for Tempo-side fee-token approval before submitting.
- Mode A: Single Send UI
  - Recipient: Recipient EVM Address (Placeholder: 0x...)
  - Amount: 0.00
  - Destination Network: Dropdown (Tempo L1, Ethereum, Arbitrum, Base, BSC).
  - Validation Error: If user pastes 0x0000000000000000000000000000000000000000, block input with red text: ⚠️ Zero address recipient not allowed.
    
- Mode B: Batch Execution UI (The ZK Payroll Engine)
  - Input Method Toggle: [ Upload CSV ] | [ Manual Entry ]
  - Option 1: Upload CSV:
    - Drag-and-drop .csv file interface.
    - Format Helper: "Required columns: address, amount, destEid."
    - Success State: "✅ Loaded 8 recipients" (Transitions automatically to Populated Rows View).
  - Option 2: Manual Entry:
    - User clicks a sleek [ + Add Recipient ] button to dynamically append new transfer rows to the UI, up to the 10-output cryptographic limit.
  - Populated Rows View (Pre-Flight Inline Editing):
    - The UI renders a clean, scrollable list of the recipients. Users can edit inputs directly before executing.
    - Sample Populated Row 1: Address: 0xAbC...123 | Amount: 1500.00 | Dest: Base | [ Remove ]
    - Sample Populated Row 2: Address: 0xDeF...456 | Amount: 3200.00 | Dest: Arbitrum | [ Remove ]
  - Validation Constraint: "Parly utilizes highly-optimized Zero-Knowledge cryptographic batching. One proof supports a maximum of 10 outputs."
    
- The Note Selector (UTXO Management):
  - A dropdown mapping unspent deposits. Strictly readOnly to preserve cryptographic join-split integrity.
  - Sample: [ 10,000 USDC | deposit | 0xabc123... ]
Execution Summary
Card Header
- Left: Execution Summary
- Right:
  - Same-chain state: Tempo
  - Cross-chain state: Cross-Chain
- Main Rows
  - Recipient Count: 3
  - Total Payout: 4,676.50 USDC
  - Total Fee: 24.00 USDC
  - Total Shielded Deduction: 4,700.50 USDC 
    Primary Link Row
    - [ View Fee Breakdown ]
Fee Breakdown Drawer
    Drawer Title: Execution Fee Breakdown
State A — Same-Chain + Private Relay
      - Protocol Fee: 23.50 USDC
      - Relay Execution Fee: 0.50 USDC
      - Total Fee: 24.00 USDC
      - Total Shielded Deduction: 4,700.50 USDC
State B — Cross-Chain + Private Relay
      - Protocol Fee: 23.50 USDC
      - Relay Execution Fee: 0.50 USDC (i)
      - Total Fee: 24.00 USDC
      - Total Shielded Deduction: 4,700.50 USDC
      Info Tooltip
      - Includes cross-chain delivery cost for this route in this version
State C — Same-Chain + Self-Relay
      - Protocol Fee: 23.50 USDC
      - Relay Execution Fee: 0.00 USDC
      - Total Fee: 23.50 USDC
      - Total Shielded Deduction: 4,700.00 USDC
State D — Cross-Chain + Self-Relay
      - Protocol Fee: 23.50 USDC
      - Cross-Chain Delivery Fee: 0.25 USDC
      - Relay Execution Fee: 0.00 USDC
      - Total Fee: 23.75 USDC
      - Total Shielded Deduction: 4,700.00 USDC
OpSec Strip
      - Icon: Warning
      - Copy: Use a fresh wallet or an authorized MPP agent for stronger privacy
      - OpSec Warning: ⚠️ OpSec Warning: To maintain absolute anonymity, only withdraw to a brand new wallet with 0 transaction history, or an authorized MPP Agent.
      
    Drawer Visibility Rules:
      Same-Chain + Private Relay
        Show Protocol Fee
        Show Relay Execution Fee
        Do not show Cross-Chain Delivery Fee
       Cross-Chain + Private Relay:
        Show Protocol Fee
        Show Relay Execution Fee with info icon
        Do not show Cross-Chain Delivery Fee
      Same-Chain + Self-Relay:
        Show Protocol Fee
        Show Relay Execution Fee as 0
        Do not show Cross-Chain Delivery Fee
      Cross-Chain + Self-Relay:
        Show Protocol Fee
        Show Cross-Chain Delivery Fee
        Show Relay Execution Fee as 0
- The ZK Loading & Exact Execution Copy (V16.9.9):
  - Action: User clicks [ Execute Privacy Proof ].
  - Proving: Generating proof...
  - Broadcasting: Broadcasting...
  - Outcome A (Relayer Submitted): Relayer submitted: -> Encrypted bundle sent to the selected relayer. Settlement completes when the relayer lands the transaction.
  - Outcome B (Pending confirmation): Pending confirmation: -> Transaction was broadcast, but finality could not yet be confirmed. Check explorer before retrying.
    - Action Row: Pending Confirmation (Disabled) Refresh Status
  - Outcome C (Success): Success: -> On-chain execution confirmed.
  - Outcome D (Final failure): Execution Failed: -> This execution did not complete successfully.
    - Action Row: Retry Execution

---

Part 3.5: Parly Ledger & Compliance
Architectural Outline: To ensure the best UX without crowding the widget, the Parly Ledger & Compliance section sits directly below the main Dapp widget. It prevents extra cluttered sub-pages and awkward tab overloads. Every activity item appears first as one clean expandable ledger card. The user can expand for more detail. Only Private Execution cards expose a separate action to open the payout scope selector drawer. Verify stays entirely standalone for external parties.

Section Header & Filter Bar
- Title: Parly Ledger & Compliance
- Subtitle: Track shielded capital, review private executions, and generate audit-ready payout proofs from one premium control surface.
- Primary Action: [ Export Visible CSV ]
- Supporting Microcopy: Every record is truth-labeled by source. Expand a card for deeper detail. Compliance exports stay scoped to one disclosed payout lane at a time.
  
Filter Bar:
- Search Placeholder: Search tx hash, commitment, nullifier, recipient, or GUID
- Filter 1: All Assets | USDC | USDT
- Filter 2: All Activity | Deposits | Private Executions | Change Notes
- Filter 3: All Origins | Tempo Native | Supported Spokes
- Truth Legend: [ Chain-Indexed ] [ Calldata-Decoded ] [ Local Scope ] [ Provenance ]
  
The Ledger Card System
UX Design Rule: Each activity appears as one premium ledger card in collapsed mode first. Clicking the chevron expands the card into a clean detail panel. Only Private Execution cards include a dedicated “Review Payout Scopes” action.

1. Tempo-Native Deposit Card
Collapsed State:
- Card Type: Shielded Deposit
- Primary Amount: + 5,000.00 USDC
- Primary Meta: Oct 22, 2026, 14:00 UTC
- Secondary Meta: Tempo tx 0x4c91ab...8f20
- Origin Pill: Tempo Native
- Truth Badges: [ Chain-Indexed ] [ Local Scope ]
- Right Action: View Details ▾
  
Expanded State:
- Section Title: Deposit Details
- Summary Copy: Capital entered the Parly privacy pool directly on Tempo and was converted into shielded balance.
- Fields:
  - Asset: USDC
  - Shielded Amount: 5,000.00 USDC
  - Confirmed At: Oct 22, 2026, 14:00 UTC
  - Tempo Transaction Hash: 0x4c91ab...8f20
  - Commitment: 0x91fe28...b731
  - Origin: Tempo Native
- Supporting Note: This deposit was created natively on the Parly settlement layer. No cross-chain origin record applies to this card.
  
2. Supported Spoke Deposit Card
Collapsed State:
- Card Type: Shielded Deposit
- Primary Amount: + 10,000.00 USDC
- Primary Meta: Oct 24, 2026, 10:15 UTC
- Secondary Meta: Tempo tx 0xdef456...92ab
- Origin Pill: Supported Spoke
- Truth Badges: [ Chain-Indexed ] [ Local Scope ] [ Provenance ]
- Right Action: View Details ▾
  
Expanded State:
- Section Title: Deposit Details
- Summary Copy: Capital arrived from a supported spoke network and settled into Parly’s Tempo privacy pool with provenance attached.
- Fields:
  - Asset: USDC
  - Shielded Amount: 10,000.00 USDC
  - Confirmed At: Oct 24, 2026, 10:15 UTC
  - Tempo Transaction Hash: 0xdef456...92ab
  - Commitment: 0x88ac14...f91d
  - Origin: Supported Spoke
- Subsection Title: Origin Provenance
- Provenance Fields:
  - LayerZero GUID: 0x8a9b77...ab41
  - Source Chain: Base
  - Source Transaction Hash: 0x123abc...ee90
  - Source Sender: 0x71C7...9A23
  - Tempo Settlement Anchor: 0xdef456...92ab
- Supporting Note: This record gives finance and audit teams a clearer cross-chain trail without weakening the privacy posture of the broader shielded balance.
  
3. Private Execution Card
Collapsed State:
- Card Type: Private Execution
- Primary Amount: - 4,700.00 USDC
- Primary Meta: Oct 25, 2026, 09:30 UTC
- Secondary Meta: Tempo tx 0x789ghi...17cd
- Fee Meta: Protocol Fee 23.50 USDC • Executor Fee 0.50 USDC
- Execution Mode: Private Relay or Self-Relay - (state aware)
- Route: Tempo or Cross-Chain (State aware)
- Cross-chain delivery Fee Settled: 0.25 USDC 
  (Note: This is a conditional Route Cost Pill show only when Self-Relay and Cross-Chain.)
- Truth Badges: [ Chain-Indexed ] [ Local Scope ]
- Primary Action: [ Review Payout Scopes ]
- Secondary Action: View Details ▾
  
Expanded State:
- Section Title: Execution Details
- Summary Copy: This private execution moved shielded capital through Parly’s settlement engine. Review payout scopes to open one selective-disclosure lane for audit or counterparty verification.
- Fields:
  - Asset: USDC
  - Executed Amount: 4,700.00 USDC
  - Confirmed At: Oct 25, 2026, 09:30 UTC
  - Tempo Transaction Hash: 0x789ghi...17cd
  - Nullifier Hash: 0x54bc9e...1f20
  - Protocol Fee: 23.50 USDC
  - Executor Fee: 0.50 USDC
  - Source Note Commitment: 0x88ac14...f91d
  - Route: Tempo or Cross-Chain (State aware)
  - Cross-chain delivery Fee Settled: 0.25 USDC 
    (Note: This is a conditional Route Cost Pill show only when Self-Relay and Cross-Chain.)
- Callout Box: Selective disclosure is available at the payout-scope level, not the whole batch level.
- Button: [ Review Payout Scopes ]
  
4. Change Note Card
Collapsed State:
- Card Type: Change Note Created
- Primary Amount: + 5,276.00 USDC
- Primary Meta: Oct 25, 2026, 09:30 UTC
- Secondary Meta: Commitment 0x44de21...cc08
- Truth Badges: [ Local Scope ]
- Right Action: View Details ▾
  
Expanded State:
- Section Title: Change Note Details
- Summary Copy: This record reflects shielded value returned to the remaining private balance after execution.
- Fields:
  - Asset: USDC
  - Remaining Shielded Value: 5,276.00 USDC
  - Created At: Oct 25, 2026, 09:30 UTC
  - Commitment: 0x44de21...cc08
  - Linked Execution: 0x789ghi...17cd
- Supporting Note: Change stays private inside the user’s shielded state and is not presented as a public payout event.
  
Payout-Scope Selector Flow
UX Flow: User clicks Review Payout Scopes ➔ Dedicated drawer opens showing separate payout scope cards ➔ User chooses one scope ➔ Opens the scoped receipt preview ➔ Exports PDF.

Drawer Shell:
- Drawer Title: Select Payout Scope
- Drawer Subtitle: This private execution contains multiple payout lanes. Choose the single payout scope you want to review, export, or verify. The rest of the batch remains private.
- Top Label: Available Payout Scopes
  
Scope Card 01:
- Scope Label: Payout Scope 01
- Truth Badges: [ Chain-Indexed ] [ Calldata-Decoded ] [ Local Scope ]
- Fields: * Recipient: 0xRecipient...91A2 
  - Amount: 1,500.00 USDC 
  - Destination Chain: Base 
  - Destination EID: 30184 
  - Route Type: Cross-Chain 
  - Child Key: vk_parly_child_1_7f3a...b91e
- Primary Action: [ Open Scoped Receipt ]
  
Scope Card 02:
- Scope Label: Payout Scope 02
- Truth Badges: [ Chain-Indexed ] [ Calldata-Decoded ] [ Local Scope ]
- Fields: * Recipient: 0xRecipient...44BF 
  - Amount: 2,000.00 USDC 
  - Destination Chain: Arbitrum 
  - Destination EID: 30110 
  - Route Type: Cross-Chain 
  - Child Key: vk_parly_child_2_18dd...c772
- Primary Action: [ Open Scoped Receipt ]
  
Scope Card 03:
- Scope Label: Payout Scope 03
- Truth Badges: [ Chain-Indexed ] [ Calldata-Decoded ] [ Local Scope ]
- Fields: * Recipient: 0xRecipient...63D9 
  - Amount: 1,200.00 USDC 
  - Destination Chain: Tempo 
  - Destination EID: 4217 
  - Route Type: Same-Chain 
  - Child Key: vk_parly_child_3_5c91...1a40
- Primary Action: [ Open Scoped Receipt ]
  
Drawer Footer:
- Footer Note: Private by default. You are opening one disclosed payout lane only.
- Secondary Note: This gives auditors and business counterparties the exact lane they need without exposing the wider execution set.
  
Scoped Compliance Receipt Preview Modal
Modal Shell:
- Eyebrow: Compliance Preview
- Modal Title: Parly Scoped Compliance Receipt
- Modal Subtitle: Export a premium audit-ready record for one verified payout lane. Designed for finance teams, exchanges, auditors, and counterparties.
- Top Right Actions: [ Download PDF ] [ Close ]
Receipt Body Layout:
- Section: Verified Settlement Facts
  - Protocol: Parly 
  - Network: Tempo 
  - Action: Private Execution 
  - Asset: USDC 
  - Confirmed Tempo Transaction Hash: 0x789ghi...17cd
  - Finalized Timestamp: Oct 25, 2026, 09:30 UTC 
  - Protocol Fee: 23.50 USDC 
  - Executor Fee: 0.50 USDC 
  - Selected Child Key: vk_parly_child_1_7f3a...b91e
- Section: Verified Payout Scope
  - Recipient Address: 0xRecipient...91A2 
  - Payout Amount: 1,500.00 USDC 
  - Destination Chain: Base 
  - Destination EID: 30184 
  - Route Type: Cross-Chain 
  - Output Index: 0
  - Supporting Copy: This section defines the exact payout lane disclosed for business verification. It does not expose the rest of the execution batch.
- Section: Private Local Scope
  - Source Note Commitment: 0x88ac14...f91d 
  - Related Commitment: 0x44de21...cc08
- Section: Origin Provenance
  - LayerZero GUID: 0x8a9b77...ab41 
  - Source Chain: Base 
  - Source Transaction Hash: 0x123abc...ee90 
  - Source Sender: 0x71C7...9A23 
  - Tempo Settlement Anchor: 0xdef456...92ab
- Truth Labels & Disclaimer:
  - Truth Badges: [ Chain-Indexed ] [ Calldata-Decoded ] [ Local Scope ] [ Provenance ]
  - Disclaimer Title: Scope & Source Notice
  - Disclaimer Copy: This receipt presents confirmed chain-indexed facts and, where labeled, scoped local or provenance-linked context. It is intentionally limited to one disclosed payout lane and does not claim facts beyond that scope.
- Verify Instruction Footer:
  - Footer Title: Verify Independently
  - Footer Copy: To independently validate this payout scope, open Parly Verify and enter: 1. the confirmed Tempo transaction hash 2. the selected child key
  - Secondary Copy: This document is selective disclosure by design. It proves one payout lane without opening the rest of the private execution.
    
Exported PDF Document Copy
Header:
- Brand: Parly
- Title: Scoped Compliance Receipt
- Subtitle: Private settlement. Selective disclosure. Audit-ready proof.
Sections:
- Verified Settlement Facts: Protocol: Parly 
  - Network: Tempo 
  - Action: Private Execution 
  - Asset: USDC 
  - Confirmed Tempo Transaction Hash: 0x789ghi...17cd 
  - Finalized Timestamp: Oct 25, 2026, 09:30 UTC 
  - Protocol Fee: 23.50 USDC 
  - Executor Fee: 0.50 USDC 
  - Selected Child Key: vk_parly_child_1_7f3a...b91e
- Verified Payout Scope: 
  - Recipient Address: 0xRecipient...91A2 
  - Payout Amount: 1,500.00 USDC 
  - Destination Chain: Base 
  - Destination EID: 30184 
  - Route Type: Cross-Chain 
  - Output Index: 0
- Private Local Scope: 
  - Source Note Commitment: 0x88ac14...f91d 
  - Related Commitment: 0x44de21...cc08
- Origin Provenance: 
  - LayerZero GUID: 0x8a9b77...ab41 
  - Source Chain: Base | Source Transaction Hash: 0x123abc...ee90 
  - Source Sender: 0x71C7...9A23
  - Tempo Settlement Anchor: 0xdef456...92ab
  
Footer:
  - Title: Independent Verification
  - Copy: Open Parly Verify and enter the confirmed Tempo transaction hash together with the selected child key to confirm this payout scope independently.
  - Secondary Copy: This document discloses one payout lane only. The remainder of the private execution remains undisclosed by design.

---

Part 4: The Dedicated Verification Portal (parly.fi/verify)
A completely standalone, public-facing portal for auditors and enterprises. No Web3 wallet connection is required.
Header
Title: Parly Verify 
Subtitle: Selective disclosure for private capital flows. Built for auditors, exchanges, treasury teams, compliance reviewers, and business counterparties.
Input State
- Page Title: Verify a Parly Payout Scope
- Utility Link: Preview Sample Report

Verify Form:
- Field 1 Label: Confirmed Tempo Transaction Hash
  - Helper: Required. This anchors the verification of a finalized Parly execution on Tempo.
  - Example Value: 0x789ghi...17cd
- Field 2 Label: Child Key
  - Helper: Required. This narrows the verification to one disclosed payout lane only.
  - Example Value: vk_parly_child_1_7f3a...b91e
- Primary Button: [ Verify Payout Scope ]
Loading State
- Querying indexed settlement data...
- Matching child-key scope...
- Resolving disclosed payout lane...
Success State
- Status Badge: Verified
- Result Title: Parly Payout Scope Confirmed
- Result Subtitle: This child key matches a real indexed Tempo execution and resolves to one disclosed payout lane.
- Truth Badges: [ Chain-Indexed ] [ Calldata-Decoded ] [ Local Scope ] [ Provenance ]
Execution Summary:
  - Protocol: Parly
  - Asset: USDC 
  - Confirmed Tempo Transaction Hash: 0x789ghi...17cd 
  - Verified Date: Oct 25, 2026 
  - Protocol Fee: 23.50 USDC 
  - Executor Fee: 0.50 USDC
Matched Payout Scope:
  - Recipient Address: 0xRecipient...91A2 
  - Payout Amount: 1,500.00 USDC
  - Destination Chain: Base
  - Destination EID: 30184
  - Route Type: Cross-Chain
  - Child Key Scope: Single Disclosed Payout Lane
Additional Context:
  - Source Note Commitment: 0x88ac14...f91d
  - LayerZero GUID: 0x8a9b77...ab41
  - Source Chain: Base
  - Source Transaction Hash: 0x123abc...ee90
  - Source Sender: 0x71C7...9A23
  - Tempo Settlement Anchor: 0xdef456...92ab

Closing Statement:
- Closing Copy: Parly has verified that this child key resolves to a real payout lane anchored to the referenced Tempo execution.
- Secondary Copy: This verification is selective by design. It confirms the disclosed lane without exposing the remainder of the private execution.

---
Part 5: The Admin Vault (Hidden Route: parly.fi/vault-admin)
A stark, high-security dashboard accessed only by the founding team's multisig wallets. Built strictly on the V16.9.9 ProtocolTreasury.sol.

Screen 1: Access Control (The Unauthorized Block)
- UI: A completely blank, dark screen with a single [ Connect Wallet ] button.
- Validation: If the connected address is not in the owners array of the ProtocolTreasury.sol contract, the screen flashes red/amber.
- Error State: A massive full-screen block appears: Error 403: Unauthorized. Wallet 0x71C...9A23 is not a registered ProtocolTreasury signer.
  
Screen 2: Vault Dashboard
- Validation: Connected address is verified in the owners array.
- Top Metric Row (Sample Populated State):
  - USDC Protocol Revenue: 14,500 USDC
  - USDT Protocol Revenue: 2,100 USDC
  - Global Standard Fee: 50 BPS
  - Pending Confirmations: 1
  - Signer Quorum: 2 / 3 Required.
  - Active Owners: 0xOwner1..., 0xOwner2..., 0xOwner3...
- Pending Actions Table: Displays the 50 most recent multisig proposals fetched directly from the RPC.
  
Screen 3: Create Proposal (The 4 Immutable Actions)
Developer Note: Solidity does not support decimals. The UI must enforce Basis Points (BPS) inputs for fees. The frontend translates this visually.
Accessed via a + New Proposal button. Opens a modal with a dropdown to select the Action Type.
- Action Type 1: Withdraw Profit (Sweep Revenue).
  - Inputs: Select Token (USDC/USDT), Destination Address, Amount.
  - Helper Text: "Moves accumulated protocol profit from the Treasury to an external wallet."
  - Button: [ Submit Withdrawal Proposal ]
- Action Type 2: Change Global Fee.
  - UI State: Reads on-chain contract globalFeeBps (e.g., 50 BPS).
  - Input Field Label: Enter New Fee (in BPS): (Validation: Max 100 BPS / 1%).
  - Dynamic UI Feedback: As admin types 75, text span displays: (Equals 0.75%).
  - Button: [ Submit Fee Change Proposal ]
- Action Type 3: Manage Signers (Add/Remove/Replace).
  - Inputs: Action Toggle (Add Signer, Remove Signer, Replace Signer), Wallet Address(es).
  - Validation Error State: ⚠️ Cannot remove signer. Roster would fall below the required 2-signer threshold. This strict validation prevents permanently bricking the vault.
  - Button: [ Submit Roster Proposal ]
- Action Type 4: Cancel Transaction.
  - Inputs: Tx Index ID.
  - Helper Text: "Cancels a pending multi-sig proposal before execution."
  - Button: [ Submit Cancellation Proposal ]
    
Screen 4: Sign, Execute, & Revoke (The Multisig Flow)
When Founder B logs in, they see Founder A's proposal in the "Pending Actions" table.
- List View: #4 | Action Target: 0xTreasury... | Value: 0 | Confirmations: 1/2
- Interaction: Founder B clicks the row. A side-drawer opens showing the exact raw hex data (Calldata: 0x...) so the founder can verify the transaction payload before signing.
- Button 1: [ Confirm (Sign) ] -> Founder B signs via MetaMask.
- Button 2 (Revoke Confirmation): Rendered in Red only if the connected user has already confirmed. Triggers revokeConfirmation(txIndex).
- State Change: Upon 2nd signature, status updates to 2 / 2 Ready to Execute.
- Button 3: [ Execute ] (Neon Green).
- UI Warning (DevEx Polish): ⚠️ Note: Executing a proposal pays the network gas fee. Ensure your admin wallet holds native Tempo gas tokens.