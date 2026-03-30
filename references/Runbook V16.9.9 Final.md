# PARLY.FI MASTER TECHNICAL BLUEPRINT

## Final Specification V16.9.9 - Canonical Zero-Trust Engineering Runbook

### Part 1 - Launch Law, Trust Topology, Monorepo, Environment, Web Baseline, Circuit, Interfaces, Treasury, Corrected Hub Pool

---

## 0. Reality-first preface

V16.9.9 is the corrected successor to the V16.9.7, V16.9.4, V16.9.3, V16.9.2, V16.9.1, V16.9.0, V16.8, and V16.8.1 runbooks.

Its job is not to add novelty for its own sake. Its job is to keep the launch surface stable, preserve the useful architecture from the stronger branches, and remove the defects that would otherwise block a real deployment or mislead a junior engineer.

This revision deliberately does all of the following:

1. keeps the launch scope unchanged,
2. fixes the audited compile blockers,
3. fixes the relayer profit-unit mismatch,
4. fixes secret-handling footguns in relayer key generation,
5. hardens relay bundle validation and per-bundle fault isolation,
6. fixes unsafe eager post-broadcast approval revocation in relayer, SDK, and browser direct-execution paths,
7. fixes zero-alt-fee quote/send mismatch on spoke gateways,
8. adds bounded pre-submit retry and persistent pending-state logging for uncertain relay submissions,
9. fixes browser direct deposit and direct execution finality handling so terminal failures clean approvals while uncertain post-broadcast states stay honest,
10. hardens numeric runtime-config parsing for browser finality and relayer payload-size guards,
11. fixes remaining metadata drift between the document and shipped code snippets,
12. keeps every launch-critical file explicit,
13. preserves the operational reasoning behind zero-trust choices,
14. keeps the acceptance matrix and smoke flow aligned with the code that appears later in the document,
15. rejects zero-address payout recipients across pool, browser, SDK, and relayer flows,
16. makes shield-mining divisor configuration fail fast instead of dividing by zero at runtime,
17. tightens the production web CSP by removing blanket unsafe-eval,
18. fixes witness-harness `path` shadowing so test-witness generation no longer crashes at runtime,
19. replaces the SDK's Node-only `Buffer` hex path with a portable byte-to-hex conversion,
20. removes the unstable `Uint8Array` dependency smell from `useRecoveredNotes` by keying the effect off a deterministic fingerprint,
21. adds deterministic spoke-deposit provenance support using LayerZero message GUID correlation across source dispatch and Tempo settlement surfaces,
22. updates the public indexer and fetcher layer so provenance is queryable without timestamp or amount inference,
23. and removes the temporary prompt-pack appendix in favor of actual shipped code changes.

This runbook uses three note types.

### 0.1 Founder Notes

Founder Notes call out the parts of the system that require founder or protocol-operator participation because they are trust-boundary actions, not routine coding chores.

They focus on:

* MPC ceremony participation and recordkeeping,
* treasury signer custody,
* protocol-admin transfer discipline,
* endpoint and peer verification,
* launch freeze records,
* and separation of human, treasury, relayer, and agent powers.

### 0.2 Dev Notes

Dev Notes are the section-local implementation checklist for the engineer following the runbook.

They emphasize:

* exact file names,
* exact environment keys,
* safe step order,
* anti-footgun validation,
* and the practical "do this next" guidance a junior engineer needs during build, wire, test, and operate phases.

### 0.3 Architectural Reasoning

Architectural Reasoning explains why the system is built this way so a future engineer does not accidentally weaken it while trying to "simplify" it.

Its job is to preserve invariants and prevent regressions such as:

* theft paths,
* fee-path contradictions,
* recovery breakage,
* ownership drift,
* or trust collapse.

### 0.4 Honest boundary

This document is a **stronger static production candidate**, not a cryptographic, operational, or deployment proof by itself.

Real deployability still depends on:

* compiling the exact repo generated from this runbook,
* generating the exact verifier from the exact final .zkey,
* wiring the real OFTs, endpoints, libraries, and DVNs,
* funding and booting the actual services,
* and passing the acceptance matrix on target environments.

### 0.5 Provenance boundary

V16.9.9 now supports deterministic spoke-deposit provenance only where the data is explicitly captured by real source-chain and Tempo-side events.

That means:

* Tempo settlement tx hash comes from Tempo deposit / ingress settlement events,
* the final deposit commitment comes from the actual Tempo pool settlement path,
* source chain and source EID come from the indexed source gateway contract and configured network identity,
* source tx hash comes from the real source-chain gateway event transaction,
* and source sender is exposed only because that sender is already public on the source-chain gateway event.

What V16.9.9 still does **not** do is infer provenance from loose heuristics such as:

* amount matching,
* timestamp matching,
* wallet-guessing from Tempo settlement rows,
* or unindexed "probably related" source transactions.

---

# 1. Canonical launch law

## 1.1 Exact supported launch surface

V16.9.9 supports exactly this launch surface:

1. Shield on Tempo.
2. Shield from a supported spoke into Tempo using OFT compose.
3. Private same-chain payouts on Tempo.
4. Private cross-chain **stablecoin-only** payouts from Tempo to supported spokes.
5. Single send and batch send up to 10 outputs per proof.
6. Third-party private relay execution over Waku.
7. Self-relay fallback.
8. Verify portal.
9. Off-chain removable shield-mining analytics.
10. Agentic SDK.
11. MCP server.
12. MPP-compatible adapter at the SDK/MCP boundary.

## 1.2 Explicitly killed launch scope

The following are **not** in V16.9.9 launch scope:

* scheduling,
* delayed execution,
* intent vaults,
* REST intent APIs,
* swap routers,
* 1inch,
* volatile-token payout paths,
* passkey privacy mode,
* dual identity planes,
* MPP-native contract settlement primitives,
* on-chain MPP session state,
* transient note chaining,
* multi-proof chaining from one active note,
* protocol-operated monopoly relayer,
* plaintext local note storage,
* custom spoke withdrawal receiver.

## 1.2.1 MPP boundary rule

MPP compatibility exists only as an adapter around the existing Parly execution engine.

It does **not**:

* change pool behavior,
* change circuit semantics,
* add contract-level session state,
* replace the current Agentic SDK execution path,
* or weaken AGENT key vs RELAYER key separation.

The only valid place for MPP in V16.9.9 is the SDK/MCP boundary.

## 1.3 Canonical trust topology

There is one valid topology.

### Hub side

* `TempoShieldedPool` for USDC
* `TempoShieldedPool` for USDT
* `ParlyHubComposer`
* one ERC20-adapter-compatible LayerZero endpoint per asset on Tempo

### Spoke side

* one ERC20-adapter-compatible LayerZero endpoint per asset per spoke chain
* one `ParlySpokeGateway` per asset per chain for spoke-to-Tempo shielding
* no custom spoke withdrawal receiver

### Compose trust anchor

The hub composer trusts the **compose sender surfaced by LayerZero compose wrapping**.

In this topology, the contract calling `send(...)` is the **spoke gateway**, so the trusted source is the **spoke gateway**, not the spoke OFT.

### Recovery model

Correctness must not depend on local storage.

Recovery source of truth is:

* wallet-derived deterministic recovery identity,
* on-chain note envelopes,
* on-chain leaves,
* on-chain nullifiers.

Local cache is optional, encrypted, disposable, and non-authoritative.

### Hub payout fee mode

V16.9.9 keeps a single hub payout rule:

* **Tempo hub cross-chain payouts are Alt-fee-token only.**

That means:

* quote path uses `payInLzToken = true`,
* execution uses alt-fee approval flow only,
* hub send path always uses `msg.value == 0`,
* the hub payout path does not mix native-fee and alt-fee semantics.

### Spoke shield fee mode

Spoke-to-hub shielding supports both:

* **native-fee mode** when the spoke has no alt fee token,
* **alt-fee mode** when the spoke uses an alt fee token.

Quote and send must use the same fee-mode flag.

### Agentic recovery model

V16.9.9 keeps deterministic asymmetric sealed-box recovery for note envelopes.

That gives the intended launch model:

* anybody may create an envelope for the intended owner,
* only the owner with the matching recovery private key can open it,
* protocol correctness does not depend on local device state.

---

# 2. Repository layout

## 2.1 Canonical monorepo structure

```text
parly-fi-v16_9_9/
  .github/
    workflows/
  apps/
    web/
      public/
        circuits/
      src/
        app/
          spoke-shield/
          verify/
        components/
        lib/
    relayer/
      src/
    indexer/
      src/
        api/
    campaign-worker/
      src/
    mcp-server/
      src/
  packages/
    env-utils/
      src/
    protocol-abis/
      src/
    shared-types/
      src/
    crypto-utils/
      src/
    registry-client/
      src/
    contracts/
      src/
        generated/
        interfaces/
      script/
      test/
      lib/
    circuits/
      scripts/
    sdk/
      src/
      assets/
      examples/
  tools/
    repo-export/
      templates/
        relayer/
        sdk/
        mcp/
  release/
    versions.json
  dist/
    public-relayer/
    public-sdk/
    public-mcp/
```

## 2.2 Architectural reasoning

This shape is deliberate.

* `apps/web` is the user-facing product surface.
* `apps/relayer` is the private execution daemon.
* `apps/indexer` is the public indexed data layer.
* `apps/campaign-worker` is analytics-only infrastructure.
* `apps/mcp-server` is privileged agent tooling.
* `packages/env-utils` is the fail-fast runtime config layer shared across public and private surfaces.
* `packages/protocol-abis` is the stable ABI boundary for relayer, SDK, MCP, and web consumers.
* `packages/shared-types` is the stable launch-type surface for encrypted bundles, registry records, and repo-export tooling.
* `packages/crypto-utils` holds small protocol-adjacent cryptographic helpers that are safe to share outside the web app.
* `packages/registry-client` is the explicit HTTP/runtime client for relayer discovery and operator registration hooks.
* `packages/contracts` contains on-chain logic, scripts, and tests.
* `packages/circuits` contains circuit, ceremony outputs, and witness harness.
* `packages/sdk` mirrors protocol reality for agents and service-side automation.
* `tools/repo-export` is the canonical export/mirror layer that generates public distribution repos from this private monorepo.

### Distribution model

This monorepo remains the internal source of truth.

Public repos are distribution layers derived from it:

* `dist/public-relayer` for operators,
* `dist/public-sdk` for developers,
* `dist/public-mcp` for agent integrators.

Operators and third-party developers must not need the full canonical monorepo to consume those surfaces.

---

# 3. Bootstrap scaffold

## 3.1 Dev Note

Run this once from a POSIX shell environment, then fill the root `.env` before touching deployment scripts.

If the workstation is Windows, use WSL or Git Bash for the shell snippets in this runbook.

## 3.2 File: `init-parly-v16_9_9.sh`

```bash
#!/bin/bash
set -euo pipefail

echo "Starting Parly.fi V16.9.9 monorepo initialization..."

ROOT="parly-fi-v16_9_9"
mkdir -p "$ROOT"
cd "$ROOT"

git init

cat << 'EOF' > .gitignore
.env
.env.local
node_modules/
dist/
.next/
coverage/
out/
cache/
artifacts/
broadcast/
typechain-types/
ponder-env.d.ts
.secrets/
*.ptau
*.zkey
*.r1cs
*.sym
*.wasm
.DS_Store
EOF

cat << 'EOF' > package.json
{
  "name": "parly-monorepo-v16_9_9",
  "private": true,
  "packageManager": "pnpm@10.6.0",
  "scripts": {
    "dev": "pnpm -r --parallel --filter './apps/**' --if-present run dev",
    "build:packages": "pnpm -r --parallel --filter './packages/**' --if-present run build",
    "build:apps": "pnpm -r --parallel --filter './apps/**' --if-present run build",
    "build": "pnpm run build:packages && pnpm run build:apps",
    "test": "pnpm -r --parallel --filter './packages/**' --if-present run test",
    "lint": "pnpm -r --parallel --filter './apps/**' --if-present run lint && pnpm -r --parallel --filter './packages/**' --if-present run lint",
    "typecheck": "pnpm --filter @parly/env-utils run typecheck && pnpm --filter @parly/protocol-abis run typecheck && pnpm --filter @parly/shared-types run typecheck && pnpm --filter @parly/crypto-utils run typecheck && pnpm --filter @parly/registry-client run typecheck && pnpm --filter @parly/web exec tsc --noEmit && pnpm --filter @parly/relayer run typecheck && pnpm --filter @parly/indexer exec tsc --noEmit && pnpm --filter @parly/campaign-worker exec tsc --noEmit && pnpm --filter @parly/mcp-server run typecheck && pnpm --filter @parly/sdk run typecheck",
    "install:all": "pnpm install",
    "repo:validate:public-exports": "node ./tools/repo-export/validate-public-exports.mjs",
    "repo:export:relayer": "node ./tools/repo-export/export-relayer.mjs",
    "repo:export:sdk": "node ./tools/repo-export/export-sdk.mjs",
    "repo:export:mcp": "node ./tools/repo-export/export-mcp.mjs",
    "repo:export:all": "pnpm repo:export:relayer && pnpm repo:export:sdk && pnpm repo:export:mcp",
    "repo:smoke:public-exports": "node ./tools/repo-export/smoke-public-exports.mjs"
  },
  "workspaces": [
    "apps/*",
    "packages/*"
  ]
}
EOF

cat << 'EOF' > pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"
EOF

cat << 'EOF' > tsconfig.base.json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["DOM", "DOM.Iterable", "ES2022"],
    "strict": true,
    "skipLibCheck": true,
    "noEmit": true,
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "preserve",
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "baseUrl": ".",
    "paths": {
      "@parly/env-utils": ["packages/env-utils/src/index.ts"],
      "@parly/env-utils/*": ["packages/env-utils/src/*"],
      "@parly/protocol-abis": ["packages/protocol-abis/src/index.ts"],
      "@parly/protocol-abis/*": ["packages/protocol-abis/src/*"],
      "@parly/shared-types": ["packages/shared-types/src/index.ts"],
      "@parly/shared-types/*": ["packages/shared-types/src/*"],
      "@parly/crypto-utils": ["packages/crypto-utils/src/index.ts"],
      "@parly/crypto-utils/*": ["packages/crypto-utils/src/*"],
      "@parly/registry-client": ["packages/registry-client/src/index.ts"],
      "@parly/registry-client/*": ["packages/registry-client/src/*"],
      "@parly/sdk": ["packages/sdk/src/index.ts"],
      "@parly/sdk/*": ["packages/sdk/src/*"]
    }
  }
}
EOF

mkdir -p .github/workflows
mkdir -p apps/web/src/{app,components,lib,db,app/app,app/verify,app/spoke-shield,app/history,app/docs,app/docs/relayers,app/docs/quickstart,app/sdk,app/vault-admin,app/relayer}
mkdir -p apps/web/src/app/api/relayers/register/init
mkdir -p apps/web/src/app/api/relayers/register/confirm
mkdir -p apps/web/src/app/api/relayers/heartbeat
mkdir -p apps/web/src/app/api/cron/relayers/inactive
mkdir -p apps/web/public/circuits
mkdir -p apps/relayer/src
mkdir -p apps/indexer/src/api
mkdir -p apps/campaign-worker/src
mkdir -p apps/mcp-server/src
mkdir -p packages/env-utils/src
mkdir -p packages/protocol-abis/src
mkdir -p packages/shared-types/src
mkdir -p packages/crypto-utils/src
mkdir -p packages/registry-client/src
mkdir -p packages/contracts/{src,script,test,lib}
mkdir -p packages/contracts/src/{interfaces,generated}
mkdir -p packages/circuits/scripts
mkdir -p packages/sdk/{src,assets,scripts,examples}
mkdir -p tools/repo-export/templates/{relayer,sdk,mcp}
mkdir -p release
mkdir -p dist/public-{relayer,sdk,mcp}

echo "OK Parly.fi V16.9.9 repo scaffold created."
```

---

# 4. Root TypeScript configs

## 4.0 File: `packages/env-utils/package.json`

```json
{
  "name": "@parly/env-utils",
  "version": "16.9.9",
  "private": true,
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    },
    "./package.json": "./package.json"
  },
  "files": ["dist", "package.json"],
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "typecheck": "tsc -p tsconfig.json --noEmit"
  },
  "devDependencies": {
    "typescript": "5.8.2"
  }
}
```

## 4.0 File: `packages/env-utils/tsconfig.json`

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "ESNext",
    "target": "ES2022",
    "declaration": true,
    "outDir": "dist",
    "noEmit": false
  },
  "include": ["src/**/*.ts"]
}
```

## 4.0 File: `packages/env-utils/src/index.ts`

```ts
export type EnvLike = Record<string, string | undefined>

export function requireEnv(name: string, env: EnvLike = process.env): string {
  const value = env[name]
  if (!value) {
    throw new Error(`${name} missing`)
  }
  return value
}

function requirePositiveNumber(raw: string | undefined, name: string, label: string): number {
  const value = Number(raw ?? "")
  if (!Number.isFinite(value) || !Number.isInteger(value) || value <= 0) {
    throw new Error(`${name} must be a ${label}.`)
  }
  return value
}

export function requirePositiveInt(name: string, env: EnvLike = process.env): number {
  return requirePositiveNumber(env[name], name, "positive integer")
}

export function requirePositiveChainId(name: string, env: EnvLike = process.env): number {
  return requirePositiveNumber(env[name], name, "positive integer chain ID")
}

export function requirePositiveEid(name: string, env: EnvLike = process.env): number {
  return requirePositiveNumber(env[name], name, "positive LayerZero EID")
}

export function requirePositiveIntValue(raw: string | undefined, name: string): number {
  return requirePositiveNumber(raw, name, "positive integer")
}

export function requirePositiveChainIdValue(raw: string | undefined, name: string): number {
  return requirePositiveNumber(raw, name, "positive integer chain ID")
}

export function requirePositiveEidValue(raw: string | undefined, name: string): number {
  return requirePositiveNumber(raw, name, "positive LayerZero EID")
}
```

## 4.0a File: `packages/protocol-abis/package.json`

```json
{
  "name": "@parly/protocol-abis",
  "version": "16.9.9",
  "private": true,
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    },
    "./package.json": "./package.json"
  },
  "files": ["dist", "package.json"],
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "typecheck": "tsc -p tsconfig.json --noEmit"
  },
  "dependencies": {
    "viem": "2.43.5"
  },
  "devDependencies": {
    "typescript": "5.8.2"
  }
}
```

## 4.0a File: `packages/protocol-abis/tsconfig.json`

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "ESNext",
    "target": "ES2022",
    "declaration": true,
    "outDir": "dist",
    "noEmit": false
  },
  "include": ["src/**/*.ts"]
}
```

## 4.0a File: `packages/protocol-abis/src/index.ts`

```ts
import { parseAbi } from "viem"

export const POOL_ABI = parseAbi([
  "function deposit(uint256,uint256,uint256,bytes) external",
  "function batchWithdraw(uint256[2],uint256[2][2],uint256[2],uint256[50],address[],uint256[],uint32[],bytes32,bytes,bytes[]) external",
  "function quoteCrossChainFee(uint32,address,uint256,bytes) view returns (uint256)",
  "function globalFeeBps() view returns (uint256)",
  "function setGlobalFeeBps(uint256) external",
  "function lzFeeToken() view returns (address)",
  "function nullifierHashes(bytes32) view returns (bool)",
  "function currentRoot() view returns (bytes32)"
])

export const SPOKE_GATEWAY_ABI = parseAbi([
  "function quoteShieldFee(uint256,uint256,address,bytes) view returns (uint256,bool)",
  "function shieldCrossChain(uint256,uint256,bytes) payable",
  "function underlyingToken() view returns (address)",
  "function lzFeeToken() view returns (address)",
  "function previewExecutionOptions() view returns (bytes)",
  "event ShieldedCrossChain(address indexed sender, uint16 indexed assetId, bytes32 indexed guid, uint256 amount, uint256 innerCommitment)"
])

export const ERC20_ABI = parseAbi([
  "function approve(address,uint256) external returns (bool)",
  "function allowance(address,address) external view returns (uint256)"
])

export const TREASURY_ABI = parseAbi([
  "function getOwners() view returns (address[])",
  "function threshold() view returns (uint256)",
  "function getTransactionCount() view returns (uint256)",
  "function pendingTransactionCount() view returns (uint256)",
  "function transactions(uint256) view returns (address target, uint256 value, bytes data, bool executed, bool cancelled, uint256 numConfirmations)",
  "function isConfirmed(uint256,address) view returns (bool)",
  "function submitTransaction(address,uint256,bytes) external",
  "function confirmTransaction(uint256) external",
  "function revokeConfirmation(uint256) external",
  "function executeTransaction(uint256) external",
  "function cancelTransaction(uint256) external",
  "function addSigner(address) external",
  "function removeSigner(address) external",
  "function replaceSigner(address,address) external",
  "function sweepRevenue(address,address,uint256) external"
])
```

## 4.0b File: `packages/shared-types/package.json`

```json
{
  "name": "@parly/shared-types",
  "version": "16.9.9",
  "private": true,
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    },
    "./package.json": "./package.json"
  },
  "files": ["dist", "package.json"],
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "typecheck": "tsc -p tsconfig.json --noEmit"
  },
  "devDependencies": {
    "typescript": "5.8.2"
  }
}
```

## 4.0b File: `packages/shared-types/tsconfig.json`

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "ESNext",
    "target": "ES2022",
    "declaration": true,
    "outDir": "dist",
    "noEmit": false
  },
  "include": ["src/**/*.ts"]
}
```

## 4.0b File: `packages/shared-types/src/index.ts`

```ts
export type CanonicalRelayExecutionPayload = {
  pool: `0x${string}`
  pA: [`0x${string}`, `0x${string}`]
  pB: [[`0x${string}`, `0x${string}`], [`0x${string}`, `0x${string}`]]
  pC: [`0x${string}`, `0x${string}`]
  pubSignals: string[]
  recipients: `0x${string}`[]
  amounts: string[]
  destEids: number[]
  newChangeCommitment: `0x${string}`
  newChangeEnvelope: `0x${string}`
  lzOptions: `0x${string}`[]
}

export type RelayCipherBundle = {
  version: 1
  relayerAddress: `0x${string}`
  ciphertext: `0x${string}`
}

export type RelayerRegistryEntry = {
  executionAddress: `0x${string}`
  publicKeyB64: string
}

export type RelayerRegistryResponse = {
  items: RelayerRegistryEntry[]
  liveCount: number
  maxCount: number
  registryOpen: boolean
}
```

## 4.0c File: `packages/crypto-utils/package.json`

```json
{
  "name": "@parly/crypto-utils",
  "version": "16.9.9",
  "private": true,
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    },
    "./package.json": "./package.json"
  },
  "files": ["dist", "package.json"],
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "typecheck": "tsc -p tsconfig.json --noEmit"
  },
  "devDependencies": {
    "typescript": "5.8.2"
  }
}
```

## 4.0c File: `packages/crypto-utils/tsconfig.json`

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "ESNext",
    "target": "ES2022",
    "declaration": true,
    "outDir": "dist",
    "noEmit": false
  },
  "include": ["src/**/*.ts"]
}
```

## 4.0c File: `packages/crypto-utils/src/index.ts`

```ts
export function looksLikeBase64(value: string) {
  return /^[A-Za-z0-9+/=]+$/.test(value)
}

export function buildRelayerHeartbeatMessage(
  executionAddress: `0x${string}`,
  event: "SUCCESS",
  timestamp: number
) {
  return [
    "Parly Relayer Heartbeat",
    `Execution Address: ${executionAddress}`,
    `Event: ${event}`,
    `Timestamp: ${timestamp}`,
    "Domain: parly.fi"
  ].join("\n")
}
```

## 4.0d File: `packages/registry-client/package.json`

```json
{
  "name": "@parly/registry-client",
  "version": "16.9.9",
  "private": true,
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    },
    "./package.json": "./package.json"
  },
  "files": ["dist", "package.json"],
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "typecheck": "tsc -p tsconfig.json --noEmit"
  },
  "dependencies": {
    "@parly/crypto-utils": "workspace:*",
    "@parly/shared-types": "workspace:*",
    "viem": "2.43.5"
  },
  "devDependencies": {
    "typescript": "5.8.2"
  }
}
```

## 4.0d File: `packages/registry-client/tsconfig.json`

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "ESNext",
    "target": "ES2022",
    "declaration": true,
    "outDir": "dist",
    "noEmit": false
  },
  "include": ["src/**/*.ts"]
}
```

## 4.0d File: `packages/registry-client/src/index.ts`

```ts
import { isAddress } from "viem"
import { looksLikeBase64 } from "@parly/crypto-utils"
import type { RelayerRegistryEntry, RelayerRegistryResponse } from "@parly/shared-types"

export type FetchRelayerRegistryOptions = {
  fetchImpl?: typeof fetch
  endpoint?: string
  allowLegacyFallback?: boolean
  legacyRegistryRaw?: string
}

export function parseLegacyRelayerRegistry(raw = ""): RelayerRegistryEntry[] {
  return raw
    .split(",")
    .map((entry) => entry.trim())
    .filter(Boolean)
    .map((entry) => {
      const [executionAddress, publicKeyB64] = entry.split(":")
      return {
        executionAddress: String(executionAddress || "").trim().toLowerCase() as `0x${string}`,
        publicKeyB64: String(publicKeyB64 || "").trim()
      }
    })
    .filter((x) => isAddress(x.executionAddress) && looksLikeBase64(x.publicKeyB64))
}

export function isLikelyRelayerPublicKey(value: string) {
  return looksLikeBase64(value.trim()) && value.trim().length >= 40
}

export async function fetchOfficialRelayerRegistry(
  options: FetchRelayerRegistryOptions = {}
): Promise<RelayerRegistryResponse> {
  const fetchImpl = options.fetchImpl || fetch
  const endpoint = options.endpoint || "/api/relayers"

  try {
    const res = await fetchImpl(endpoint, { cache: "no-store" as RequestCache })
    if (!res.ok) {
      throw new Error(`Registry request failed with HTTP ${res.status}`)
    }

    const json = await res.json()
    if (!Array.isArray(json.items)) {
      throw new Error("Registry response malformed")
    }

    const items = json.items
      .map((row: any) => ({
        executionAddress: String(row.executionAddress || "").trim().toLowerCase() as `0x${string}`,
        publicKeyB64: String(row.publicKeyB64 || "").trim()
      }))
      .filter((row: RelayerRegistryEntry) => isAddress(row.executionAddress) && looksLikeBase64(row.publicKeyB64))

    return {
      items,
      liveCount: Number(json.liveCount || items.length),
      maxCount: Number(json.maxCount || 10),
      registryOpen: Boolean(json.registryOpen ?? items.length < Number(json.maxCount || 10))
    }
  } catch (error: any) {
    if (options.allowLegacyFallback) {
      const items = parseLegacyRelayerRegistry(options.legacyRegistryRaw || "")
      return {
        items,
        liveCount: items.length,
        maxCount: 10,
        registryOpen: items.length < 10
      }
    }

    throw new Error(error?.message || "Official relayer registry unavailable")
  }
}
```

## 4.1 File: `apps/web/tsconfig.json`

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "plugins": [{ "name": "next" }]
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

## 4.2 File: `apps/relayer/tsconfig.json`

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "target": "ES2022",
    "lib": ["ES2022"],
    "types": ["node"],
    "outDir": "dist",
    "noEmit": false
  },
  "include": ["src/**/*.ts"]
}
```

## 4.3 File: `apps/indexer/tsconfig.json`

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "target": "ES2022",
    "lib": ["ES2022"],
    "types": ["node"]
  },
  "include": ["src/**/*.ts", "ponder.config.ts", "ponder.schema.ts"]
}
```

## 4.4 File: `apps/campaign-worker/tsconfig.json`

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "target": "ES2022",
    "lib": ["ES2022"],
    "types": ["node"],
    "outDir": "dist",
    "noEmit": false
  },
  "include": ["src/**/*.ts"]
}
```

## 4.5 File: `apps/mcp-server/tsconfig.json`

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "target": "ES2022",
    "lib": ["ES2022"],
    "types": ["node"]
  },
  "include": ["src/**/*.ts"]
}
```

## 4.6 File: `packages/sdk/tsconfig.json`

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "target": "ES2022",
    "declaration": true,
    "outDir": "dist",
    "noEmit": false,
    "types": ["node"]
  },
  "include": ["src/**/*.ts"]
}
```

---

# 5. Canonical environment model

## 5.1 Founder Note

Environment layout is a trust boundary.

V16.9.9 keeps these separated:

* deployer key,
* treasury signers,
* protocol admin,
* relayer execution key,
* relayer sealed-box private key,
* agent key,
* public frontend config,
* private service-side config.

## 5.2 Dev Note

Do not improvise key names.

The scripts assume these exact names.

## 5.3 File: `.env.example`

```dotenv
# ------------------------------------------------------------------------------
# CORE
# ------------------------------------------------------------------------------
DEPLOYER_PRIVATE_KEY=0x0000000000000000000000000000000000000000000000000000000000000000
PROTOCOL_ADMIN_ADDRESS=0x0000000000000000000000000000000000000000
PROTOCOL_TREASURY_ADDRESS=0x0000000000000000000000000000000000000000
NEXT_PUBLIC_PROTOCOL_TREASURY_ADDRESS=0x0000000000000000000000000000000000000000

MULTISIG_OWNER_1=0x0000000000000000000000000000000000000000
MULTISIG_OWNER_2=0x0000000000000000000000000000000000000000
MULTISIG_OWNER_3=0x0000000000000000000000000000000000000000

# ------------------------------------------------------------------------------
# TEMPO
# ------------------------------------------------------------------------------
# Use exact official target-network values here. Tempo mainnet chain ID is 4217
# per Tempo docs; the LayerZero EID must be copied from the official LayerZero
# deployed-endpoint set for the same Tempo target before launch.
TEMPO_RPC_URL=https://rpc.example.tempo.xyz
TEMPO_CHAIN_ID=4217
# Must be set to the real Tempo LayerZero EID before boot. Placeholder values are invalid.
TEMPO_LZ_EID=REPLACE_WITH_REAL_TEMPO_LZ_EID

NEXT_PUBLIC_TEMPO_RPC_URL=https://rpc.example.tempo.xyz
NEXT_PUBLIC_TEMPO_CHAIN_ID=4217
NEXT_PUBLIC_TEMPO_LZ_EID=REPLACE_WITH_REAL_TEMPO_LZ_EID

TEMPO_USDC_ADDRESS=0x0000000000000000000000000000000000000000
TEMPO_USDT_ADDRESS=0x0000000000000000000000000000000000000000

NEXT_PUBLIC_USDC_ADDRESS=0x0000000000000000000000000000000000000000
NEXT_PUBLIC_USDT_ADDRESS=0x0000000000000000000000000000000000000000

TEMPO_USDC_KYT_POLICY_ID=2
TEMPO_USDT_KYT_POLICY_ID=3

# Hub cross-chain payout mode is Alt-fee-token only.
TEMPO_LZ_FEE_TOKEN=0x0000000000000000000000000000000000000000
NEXT_PUBLIC_TEMPO_LZ_FEE_TOKEN=0x0000000000000000000000000000000000000000

# Relayer profitability converts LZ fee-token units into USD(6) using these values.
# If the fee token is a 1 USD stable with 6 decimals, keep 6 and 1000000.
TEMPO_LZ_FEE_TOKEN_DECIMALS=6
TEMPO_LZ_FEE_TOKEN_USD_6DEC=1000000
NEXT_PUBLIC_TEMPO_LZ_FEE_TOKEN_DECIMALS=6
NEXT_PUBLIC_TEMPO_LZ_FEE_TOKEN_USD_6DEC=1000000

POSEIDON_HASHER_ADDRESS=0x0000000000000000000000000000000000000000
GROTH16_VERIFIER_ADDRESS=0x0000000000000000000000000000000000000000

# ------------------------------------------------------------------------------
# HUB OUTPUTS
# ------------------------------------------------------------------------------
NEXT_PUBLIC_USDC_POOL=0x0000000000000000000000000000000000000000
NEXT_PUBLIC_USDT_POOL=0x0000000000000000000000000000000000000000
NEXT_PUBLIC_HUB_COMPOSER=0x0000000000000000000000000000000000000000

HUB_USDC_OFT=0x0000000000000000000000000000000000000000
HUB_USDT_OFT=0x0000000000000000000000000000000000000000

# ------------------------------------------------------------------------------
# SPOKE RPCS
# ------------------------------------------------------------------------------
ETHEREUM_RPC_URL=https://eth.llamarpc.com
ARBITRUM_RPC_URL=https://arb1.arbitrum.io/rpc
BASE_RPC_URL=https://mainnet.base.org
BSC_RPC_URL=https://bsc-dataseed.binance.org

NEXT_PUBLIC_ETHEREUM_RPC_URL=https://eth.llamarpc.com
NEXT_PUBLIC_ARBITRUM_RPC_URL=https://arb1.arbitrum.io/rpc
NEXT_PUBLIC_BASE_RPC_URL=https://mainnet.base.org
NEXT_PUBLIC_BSC_RPC_URL=https://bsc-dataseed.binance.org

# ------------------------------------------------------------------------------
# CHAIN IDS
# ------------------------------------------------------------------------------
ETHEREUM_CHAIN_ID=1
ARBITRUM_CHAIN_ID=42161
BASE_CHAIN_ID=8453
BSC_CHAIN_ID=56

NEXT_PUBLIC_ETHEREUM_CHAIN_ID=1
NEXT_PUBLIC_ARBITRUM_CHAIN_ID=42161
NEXT_PUBLIC_BASE_CHAIN_ID=8453
NEXT_PUBLIC_BSC_CHAIN_ID=56

# ------------------------------------------------------------------------------
# LAYERZERO EIDS
# ------------------------------------------------------------------------------
LZ_EID_ETHEREUM=30101
LZ_EID_ARBITRUM=30110
LZ_EID_BASE=30184
LZ_EID_BNB=30102

NEXT_PUBLIC_LZ_EID_ETHEREUM=30101
NEXT_PUBLIC_LZ_EID_ARBITRUM=30110
NEXT_PUBLIC_LZ_EID_BASE=30184
NEXT_PUBLIC_LZ_EID_BNB=30102

# ------------------------------------------------------------------------------
# SPOKE -> TEMPO EXECUTION GAS PROFILE
# These values fund the hub OFT credit + composer execution path for spoke shielding.
# Profile them against the exact deployed OFTs/composer before launch, then keep them
# consistent across all spoke gateways.
# ------------------------------------------------------------------------------
SPOKE_TO_TEMPO_LZRECEIVE_GAS=65000
SPOKE_TO_TEMPO_LZCOMPOSE_GAS=500000
SPOKE_TO_TEMPO_LZCOMPOSE_VALUE=0

# ------------------------------------------------------------------------------
# LAYERZERO ENDPOINTS / LIBRARIES / DVNS
# ------------------------------------------------------------------------------
LAYERZERO_ENDPOINT_TEMPO=0x0000000000000000000000000000000000000000
LAYERZERO_SEND_LIB_TEMPO=0x0000000000000000000000000000000000000000
LAYERZERO_RECEIVE_LIB_TEMPO=0x0000000000000000000000000000000000000000

LAYERZERO_ENDPOINT_ETHEREUM=0x0000000000000000000000000000000000000000
LAYERZERO_SEND_LIB_ETHEREUM=0x0000000000000000000000000000000000000000
LAYERZERO_RECEIVE_LIB_ETHEREUM=0x0000000000000000000000000000000000000000

LAYERZERO_ENDPOINT_ARBITRUM=0x0000000000000000000000000000000000000000
LAYERZERO_SEND_LIB_ARBITRUM=0x0000000000000000000000000000000000000000
LAYERZERO_RECEIVE_LIB_ARBITRUM=0x0000000000000000000000000000000000000000

LAYERZERO_ENDPOINT_BASE=0x0000000000000000000000000000000000000000
LAYERZERO_SEND_LIB_BASE=0x0000000000000000000000000000000000000000
LAYERZERO_RECEIVE_LIB_BASE=0x0000000000000000000000000000000000000000

LAYERZERO_ENDPOINT_BNB=0x0000000000000000000000000000000000000000
LAYERZERO_SEND_LIB_BNB=0x0000000000000000000000000000000000000000
LAYERZERO_RECEIVE_LIB_BNB=0x0000000000000000000000000000000000000000

DVN_GOOGLE_CLOUD=0x0000000000000000000000000000000000000000
DVN_LAYERZERO_LABS=0x0000000000000000000000000000000000000000

# ------------------------------------------------------------------------------
# SPOKE TOKENS
# ------------------------------------------------------------------------------
SPOKE_USDC_TOKEN_ETHEREUM=0x0000000000000000000000000000000000000000
SPOKE_USDC_TOKEN_ARBITRUM=0x0000000000000000000000000000000000000000
SPOKE_USDC_TOKEN_BASE=0x0000000000000000000000000000000000000000
SPOKE_USDC_TOKEN_BNB=0x0000000000000000000000000000000000000000

SPOKE_USDT_TOKEN_ETHEREUM=0x0000000000000000000000000000000000000000
SPOKE_USDT_TOKEN_ARBITRUM=0x0000000000000000000000000000000000000000
SPOKE_USDT_TOKEN_BASE=0x0000000000000000000000000000000000000000
SPOKE_USDT_TOKEN_BNB=0x0000000000000000000000000000000000000000

# ------------------------------------------------------------------------------
# SPOKE FEE TOKENS
# zero-address means that spoke uses native fee mode.
# ------------------------------------------------------------------------------
SPOKE_LZ_FEE_TOKEN_ETHEREUM=0x0000000000000000000000000000000000000000
SPOKE_LZ_FEE_TOKEN_ARBITRUM=0x0000000000000000000000000000000000000000
SPOKE_LZ_FEE_TOKEN_BASE=0x0000000000000000000000000000000000000000
SPOKE_LZ_FEE_TOKEN_BNB=0x0000000000000000000000000000000000000000

# ------------------------------------------------------------------------------
# SPOKE OFTs
# ------------------------------------------------------------------------------
SPOKE_USDC_OFT_ETHEREUM=0x0000000000000000000000000000000000000000
SPOKE_USDC_OFT_ARBITRUM=0x0000000000000000000000000000000000000000
SPOKE_USDC_OFT_BASE=0x0000000000000000000000000000000000000000
SPOKE_USDC_OFT_BNB=0x0000000000000000000000000000000000000000

SPOKE_USDT_OFT_ETHEREUM=0x0000000000000000000000000000000000000000
SPOKE_USDT_OFT_ARBITRUM=0x0000000000000000000000000000000000000000
SPOKE_USDT_OFT_BASE=0x0000000000000000000000000000000000000000
SPOKE_USDT_OFT_BNB=0x0000000000000000000000000000000000000000

# ------------------------------------------------------------------------------
# SPOKE GATEWAYS
# ------------------------------------------------------------------------------
NEXT_PUBLIC_SPOKE_USDC_GATEWAY_ETHEREUM=0x0000000000000000000000000000000000000000
NEXT_PUBLIC_SPOKE_USDC_GATEWAY_ARBITRUM=0x0000000000000000000000000000000000000000
NEXT_PUBLIC_SPOKE_USDC_GATEWAY_BASE=0x0000000000000000000000000000000000000000
NEXT_PUBLIC_SPOKE_USDC_GATEWAY_BNB=0x0000000000000000000000000000000000000000

NEXT_PUBLIC_SPOKE_USDT_GATEWAY_ETHEREUM=0x0000000000000000000000000000000000000000
NEXT_PUBLIC_SPOKE_USDT_GATEWAY_ARBITRUM=0x0000000000000000000000000000000000000000
NEXT_PUBLIC_SPOKE_USDT_GATEWAY_BASE=0x0000000000000000000000000000000000000000
NEXT_PUBLIC_SPOKE_USDT_GATEWAY_BNB=0x0000000000000000000000000000000000000000

# ------------------------------------------------------------------------------
# WALLETCONNECT / WAKU
# ------------------------------------------------------------------------------
NEXT_PUBLIC_WC_PROJECT_ID=your_walletconnect_project_id

WAKU_CLUSTER_ID=1
WAKU_BOOTSTRAP=true
WAKU_CONTENT_TOPIC=/parly/16/9/9/app/proto

NEXT_PUBLIC_WAKU_CLUSTER_ID=1
NEXT_PUBLIC_WAKU_BOOTSTRAP=true
NEXT_PUBLIC_WAKU_CONTENT_TOPIC=/parly/16/9/9/app/proto

# comma-separated format: 0xRelayerExecAddress:BASE64_X25519_PUBLIC_KEY
# Legacy fallback only. Official frontend discovery uses GET /api/relayers at runtime.
NEXT_PUBLIC_RELAYER_REGISTRY=0xRelayer1:BASE64PUBKEY1

# ------------------------------------------------------------------------------
# KEY SEGREGATION
# ------------------------------------------------------------------------------
AGENT_PRIVATE_KEY=0x0000000000000000000000000000000000000000000000000000000000000000
# Feature-flagged MPP compatibility at the SDK/MCP boundary only.
# Leave true to expose the adapter surface without changing protocol-core behavior.
ENABLE_MPP_ADAPTER=true
NEXT_PUBLIC_ENABLE_MPP_ADAPTER=true
MPP_SERVICE_NAME=parly-mpp-adapter
MPP_SERVICE_VERSION=16.9.9

RELAYER_PRIVATE_KEY=0x0000000000000000000000000000000000000000000000000000000000000000
RELAYER_BOX_PRIVATE_KEY_B64=
RELAYER_ADDRESS=0x0000000000000000000000000000000000000000
# Exact-amount fee approvals are consumed to zero on successful execution.
# Do not eagerly revoke a fee approval after broadcast unless terminal failure is known.
# Uncertain post-broadcast states are appended to RELAYER_PENDING_EXECUTIONS_FILE.
RELAYER_MIN_PROFIT_USD=0.02
RELAYER_MAX_PAYLOAD_BYTES=32768
RELAYER_PRE_SUBMIT_RETRY_LIMIT=2
RELAYER_PRE_SUBMIT_RETRY_BACKOFF_MS=1500
RELAYER_RECEIPT_TIMEOUT_MS=300000
RELAYER_KEY_OUTPUT_FILE=.secrets/relayer-box.env
# Optional: set only when opting into official registry discovery and heartbeat updates
RELAYER_REGISTRY_API_BASE_URL=http://localhost:3000
RELAYER_REGISTRY_HEARTBEAT_TTL_SECONDS=300
RELAYER_PENDING_EXECUTIONS_FILE=.secrets/relayer-pending-executions.ndjson
RELAYER_PENDING_EXECUTIONS_ARCHIVE_FILE=.secrets/relayer-pending-executions.resolved.ndjson
RELAYER_PENDING_RECONCILE_INTERVAL_MS=30000
RELAYER_PENDING_RECONCILE_LOCK_FILE=.secrets/relayer-pending-executions.lock
SDK_PENDING_EXECUTIONS_FILE=.secrets/sdk-pending-executions.ndjson
SDK_RECOVERY_CACHE_FILE=.cache/sdk-recovery.json
SDK_RECEIPT_TIMEOUT_MS=300000
# Browser direct Tempo execution uses this timeout before surfacing an uncertain-finality state.
NEXT_PUBLIC_BROWSER_RECEIPT_TIMEOUT_MS=300000
# Browser spoke-to-Tempo UX uses this timeout while waiting for the new Tempo leaf to appear.
NEXT_PUBLIC_SPOKE_INGRESS_TIMEOUT_MS=180000

# ------------------------------------------------------------------------------
# INDEXER / API
# ------------------------------------------------------------------------------
PONDER_RPC_URL_TEMPO=https://rpc.example.tempo.xyz
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/parly

NEXT_PUBLIC_PONDER_GRAPHQL_URL=http://localhost:42069/graphql
INDEXER_ALLOWED_ORIGINS=http://localhost:3000,https://app.parly.fi
PONDER_GRAPHQL_URL=http://localhost:42069/graphql
NEXT_PUBLIC_INDEXER_PAGE_SIZE=500
INDEXER_PAGE_SIZE=500
NEXT_PUBLIC_SHIELD_MINING_POINTS_DIVISOR=100
RELAYER_REGISTRY_NONCE_SECRET=change-me
RELAYER_REGISTRY_CRON_SECRET=change-me
RELAYER_REGISTRY_IP_ATTEMPTS_PER_24H=5
RELAYER_REGISTRY_ADDRESS_ATTEMPTS_PER_24H=3
RELAYER_REGISTRY_NONCE_TTL_SECONDS=900

START_BLOCK_USDC=0
START_BLOCK_USDT=0
START_BLOCK_HUB_COMPOSER=0
START_BLOCK_SPOKE_USDC_ETHEREUM=0
START_BLOCK_SPOKE_USDC_ARBITRUM=0
START_BLOCK_SPOKE_USDC_BASE=0
START_BLOCK_SPOKE_USDC_BNB=0
START_BLOCK_SPOKE_USDT_ETHEREUM=0
START_BLOCK_SPOKE_USDT_ARBITRUM=0
START_BLOCK_SPOKE_USDT_BASE=0
START_BLOCK_SPOKE_USDT_BNB=0

# ------------------------------------------------------------------------------
# OPTIONAL CLIENT CACHE
# ------------------------------------------------------------------------------
NEXT_PUBLIC_ENABLE_OPTIONAL_NOTE_CACHE=0

# ---------------------------------------------------------------------------
# OPERATOR MAINTENANCE
# Use a caller key that is actually authorized for the action being taken.
# For post-handover owner actions, this may be omitted in favor of multisig
# calldata generation instead of direct script broadcast.
# ---------------------------------------------------------------------------
PARLY_CALLER_PRIVATE_KEY=

# ------------------------------------------------------------------------------
# SHIELD MINING
# ------------------------------------------------------------------------------
RAILWAY_DATABASE_URL=postgresql://postgres:postgres@localhost:5432/parly
# Must be a positive integer greater than zero.
SHIELD_MINING_POINTS_DIVISOR=100

# ------------------------------------------------------------------------------
# VERIFIER IMPORT / SDK ASSETS
# ------------------------------------------------------------------------------
GENERATED_VERIFIER_SOURCE=../../circuits/ZKVerifier.sol
PARLY_SDK_ASSETS_PATH=./assets

# ------------------------------------------------------------------------------
# DEPLOY SWITCH
# ------------------------------------------------------------------------------
PARLY_ASSET=USDC
```

---

# 6. Web baseline and build hardening

## 6.1 Architectural reasoning

The web app keeps explicit webpack handling for browser-safe WASM and client-side fallbacks.

That is important because the privacy stack includes:

* in-browser proof generation,
* WASM artifacts,
* client-only crypto helpers.

Production CSP should allow the WASM-specific escape hatch that proof tooling needs without also allowing blanket script eval.

## 6.2 File: `apps/web/package.json`

```json
{
  "name": "@parly/web",
  "private": true,
  "scripts": {
    "dev": "next dev --webpack",
    "build": "next build --webpack",
    "start": "next start",
    "lint": "next lint",
    "init:relayers": "psql $DATABASE_URL -f src/db/relayers.sql"
  },
  "dependencies": {
    "@layerzerolabs/lz-v2-utilities": "2.3.0",
    "@tanstack/react-query": "^5.66.0",
    "@waku/sdk": "^0.0.28",
    "circomlibjs": "0.1.7",
    "fixed-merkle-tree": "0.7.3",
    "libsodium-wrappers-sumo": "^0.7.13",
    "next": "15.2.3",
    "papaparse": "5.4.1",
    "pdf-lib": "^1.17.1",
    "pg": "^8.13.1",
    "react": "19.0.0",
    "react-dom": "19.0.0",
    "snarkjs": "0.7.4",
    "viem": "2.43.5",
    "wagmi": "2.14.16"
  },
  "devDependencies": {
    "@types/node": "22.13.10",
    "@types/papaparse": "5.3.14",
    "@types/pg": "^8.11.11",
    "@types/react": "19.0.10",
    "@types/react-dom": "19.0.4",
    "autoprefixer": "10.4.20",
    "postcss": "8.4.49",
    "tailwindcss": "3.4.17",
    "typescript": "5.8.2"
  }
}
```

## 6.3 File: `apps/web/next.config.js`

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  webpack: (config, { isServer }) => {
    if (!isServer) {
      config.resolve.fallback = {
        ...config.resolve.fallback,
        fs: false,
        path: false,
        crypto: false,
        os: false,
        readline: false
      }
    }

    config.experiments = {
      ...config.experiments,
      asyncWebAssembly: true
    }

    return config
  }
}

module.exports = nextConfig
```

## 6.4 File: `apps/web/middleware.ts`

```ts
import { NextResponse } from "next/server"
import type { NextRequest } from "next/server"

export function middleware(_request: NextRequest) {
  const res = NextResponse.next()
  const scriptSrc =
    process.env.NODE_ENV === "production"
      ? "script-src 'self' 'wasm-unsafe-eval'"
      : "script-src 'self' 'unsafe-eval' 'wasm-unsafe-eval'"

  res.headers.set("X-Frame-Options", "DENY")
  res.headers.set("X-Content-Type-Options", "nosniff")
  res.headers.set("Referrer-Policy", "strict-origin-when-cross-origin")
  res.headers.set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
  res.headers.set(
    "Content-Security-Policy",
    [
      "default-src 'self'",
      scriptSrc,
      "connect-src 'self' https: wss:",
      "img-src 'self' data: blob:",
      "style-src 'self' 'unsafe-inline'",
      "font-src 'self' https: data:",
      "frame-ancestors 'none'",
      "base-uri 'self'",
      "form-action 'self'"
    ].join("; ")
  )

  return res
}

export const config = {
  matcher: ["/((?!api|_next/static|_next/image|favicon.ico).*)"]
}
```

## 6.5 File: `apps/web/postcss.config.js`

```js
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {}
  }
}
```

## 6.6 File: `apps/web/tailwind.config.js`

```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    "./src/app/**/*.{js,ts,jsx,tsx,mdx}",
    "./src/components/**/*.{js,ts,jsx,tsx,mdx}",
    "./src/lib/**/*.{js,ts,jsx,tsx,mdx}"
  ],
  theme: {
    extend: {}
  },
  plugins: []
}
```

## 6.7 File: `apps/web/src/app/globals.css`

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

:root {
  --parly-bg: #04111f;
  --parly-panel: rgba(7, 20, 38, 0.72);
  --parly-panel-strong: rgba(10, 24, 44, 0.94);
  --parly-line: rgba(255, 255, 255, 0.1);
  --parly-copy: #eef6ff;
  --parly-muted: #9fb2c8;
  --parly-cyan: #6ee7f9;
  --parly-green: #6ee7b7;
}

* {
  box-sizing: border-box;
}

html,
body {
  min-height: 100%;
  background:
    radial-gradient(circle at top left, rgba(41, 182, 246, 0.18), transparent 28%),
    radial-gradient(circle at top right, rgba(16, 185, 129, 0.12), transparent 24%),
    linear-gradient(180deg, #020617 0%, #04111f 48%, #07101d 100%);
  color: var(--parly-copy);
  font-family: var(--font-space-grotesk), sans-serif;
}

body {
  margin: 0;
  background-attachment: fixed;
}

a {
  color: inherit;
  text-decoration: none;
}

::selection {
  background: rgba(110, 231, 249, 0.28);
  color: white;
}
```

## 6.8 File: `apps/web/src/wagmi.config.ts`

```ts
import { createConfig, http } from "wagmi"
import { arbitrum, base, bsc, mainnet, defineChain } from "viem/chains"
import { injected, metaMask, walletConnect } from "wagmi/connectors"
import { requireEnv, requirePositiveChainId } from "@parly/env-utils"

const tempoChainId = requirePositiveChainId("NEXT_PUBLIC_TEMPO_CHAIN_ID")
const tempoRpcUrl = requireEnv("NEXT_PUBLIC_TEMPO_RPC_URL")

export const tempoChain = defineChain({
  id: tempoChainId,
  name: "Tempo L1",
  nativeCurrency: { name: "TMP", symbol: "TMP", decimals: 18 },
  rpcUrls: {
    default: {
      http: [tempoRpcUrl]
    }
  },
  blockExplorers: {
    default: {
      name: "Tempo Explorer",
      url: "https://contracts.tempo.xyz"
    }
  },
  testnet: false
})

export const wagmiConfig = createConfig({
  connectors: [
    metaMask(),
    walletConnect({
      projectId: process.env.NEXT_PUBLIC_WC_PROJECT_ID || ""
    }),
    injected()
  ],
  chains: [tempoChain, mainnet, arbitrum, base, bsc],
  multiInjectedProviderDiscovery: false,
  transports: {
    [tempoChain.id]: http(tempoRpcUrl),
    [mainnet.id]: http(process.env.NEXT_PUBLIC_ETHEREUM_RPC_URL || "https://eth.llamarpc.com"),
    [arbitrum.id]: http(process.env.NEXT_PUBLIC_ARBITRUM_RPC_URL || "https://arb1.arbitrum.io/rpc"),
    [base.id]: http(process.env.NEXT_PUBLIC_BASE_RPC_URL || "https://mainnet.base.org"),
    [bsc.id]: http(process.env.NEXT_PUBLIC_BSC_RPC_URL || "https://bsc-dataseed.binance.org")
  }
})
```

39.3 File: apps/web/src/lib/parly-session.tsx

```tsx
"use client"

import type { ReactNode } from "react"
import {
  createContext,
  startTransition,
  useCallback,
  useContext,
  useEffect,
  useMemo,
  useState
} from "react"
import { useAccount, useSignMessage } from "wagmi"
import { assertDeterministicSigner } from "./wallet-compat"
import {
  MASTER_AUTH_MESSAGE,
  deriveOptionalCacheKey,
  deriveRecoveryKeypair
} from "./recovery-keys"

export type ParlyAuthPhase =
  | "disconnected"
  | "connected_unauthed"
  | "authenticating"
  | "recovering_notes"
  | "ready"

type ParlyAuthResult = {
  address: `0x${string}`
  recoveryPublicKeyB64: string
  recoveryPrivateKeyB64: string
  optionalCacheKey: Uint8Array
}

type ParlySessionValue = {
  address?: `0x${string}`
  authenticatedAddress: `0x${string}` | null
  signature: `0x${string}` | null
  recoveryPublicKeyB64: string | null
  recoveryPrivateKeyB64: string | null
  optionalCacheKey: Uint8Array | null
  recoveryEpoch: number
  phase: ParlyAuthPhase
  error: string | null
  authenticate: () => Promise<ParlyAuthResult | null>
  logout: (message?: string | null) => void
  refreshRecovery: () => void
  setSessionError: (value: string | null) => void
  setRecoveryActivity: (scope: string, active: boolean) => void
}

const ParlySessionContext = createContext<ParlySessionValue | null>(null)

export function ParlySessionProvider({ children }: { children: ReactNode }) {
  const { address } = useAccount()
  const { signMessageAsync } = useSignMessage()

  const [signature, setSignature] = useState<`0x${string}` | null>(null)
  const [authenticatedAddress, setAuthenticatedAddress] = useState<`0x${string}` | null>(null)
  const [recoveryPublicKeyB64, setRecoveryPublicKeyB64] = useState<string | null>(null)
  const [recoveryPrivateKeyB64, setRecoveryPrivateKeyB64] = useState<string | null>(null)
  const [optionalCacheKey, setOptionalCacheKey] = useState<Uint8Array | null>(null)
  const [recoveryEpoch, setRecoveryEpoch] = useState(0)
  const [authenticating, setAuthenticating] = useState(false)
  const [recoveryScopes, setRecoveryScopes] = useState<Record<string, boolean>>({})
  const [error, setError] = useState<string | null>(null)
  const recoveringNotes = Object.keys(recoveryScopes).length > 0

  const setRecoveryActivity = useCallback((scope: string, active: boolean) => {
    if (!scope) return

    setRecoveryScopes((current) => {
      const next = { ...current }

      if (active) {
        next[scope] = true
      } else {
        delete next[scope]
      }

      return next
    })
  }, [])

  useEffect(() => {
    if (!authenticatedAddress) return

    if (!address || address.toLowerCase() !== authenticatedAddress.toLowerCase()) {
      setSignature(null)
      setAuthenticatedAddress(null)
      setRecoveryPublicKeyB64(null)
      setRecoveryPrivateKeyB64(null)
      setOptionalCacheKey(null)
      setRecoveryEpoch(0)
      setRecoveryScopes({})
      setError(address ? "Connected wallet changed. Authenticate again for the active wallet." : null)
    }
  }, [address, authenticatedAddress])

  async function authenticate() {
    if (!address) {
      setError("Connect a wallet first.")
      return null
    }

    setAuthenticating(true)
    setError(null)

    try {
      const stableSignature = await assertDeterministicSigner(signMessageAsync, MASTER_AUTH_MESSAGE)
      const keypair = await deriveRecoveryKeypair(stableSignature)
      const cacheKey = await deriveOptionalCacheKey(stableSignature)

      setSignature(stableSignature)
      setAuthenticatedAddress(address)
      setRecoveryPublicKeyB64(keypair.publicKeyB64)
      setRecoveryPrivateKeyB64(keypair.privateKeyB64)
      setOptionalCacheKey(cacheKey)
      startTransition(() => setRecoveryEpoch((current) => current + 1))
      return {
        address,
        recoveryPublicKeyB64: keypair.publicKeyB64,
        recoveryPrivateKeyB64: keypair.privateKeyB64,
        optionalCacheKey: cacheKey
      }
    } catch (e: any) {
      setError(e?.message || "Authentication failed")
      return null
    } finally {
      setAuthenticating(false)
    }
  }

  function logout(message: string | null = null) {
    setSignature(null)
    setAuthenticatedAddress(null)
    setRecoveryPublicKeyB64(null)
    setRecoveryPrivateKeyB64(null)
    setOptionalCacheKey(null)
    setRecoveryEpoch(0)
    setRecoveryScopes({})
    setError(message)
  }

  function refreshRecovery() {
    startTransition(() => setRecoveryEpoch((current) => current + 1))
  }

  const phase: ParlyAuthPhase = !address
    ? "disconnected"
    : authenticating
    ? "authenticating"
    : recoveringNotes
    ? "recovering_notes"
    : signature && authenticatedAddress && address.toLowerCase() === authenticatedAddress.toLowerCase()
    ? "ready"
    : "connected_unauthed"

  const value = useMemo<ParlySessionValue>(
    () => ({
      address,
      authenticatedAddress,
      signature,
      recoveryPublicKeyB64,
      recoveryPrivateKeyB64,
      optionalCacheKey,
      recoveryEpoch,
      phase,
      error,
      authenticate,
      logout,
      refreshRecovery,
      setSessionError: setError,
      setRecoveryActivity
    }),
    [
      address,
      authenticatedAddress,
      signature,
      recoveryPublicKeyB64,
      recoveryPrivateKeyB64,
      optionalCacheKey,
      recoveryEpoch,
      phase,
      error,
      setRecoveryActivity
    ]
  )

  return <ParlySessionContext.Provider value={value}>{children}</ParlySessionContext.Provider>
}

export function useParlySession() {
  const ctx = useContext(ParlySessionContext)
  if (!ctx) {
    throw new Error("useParlySession must be used inside ParlySessionProvider")
  }
  return ctx
}
```

39.4 File: apps/web/src/lib/dashboard.ts

```ts
import { fetchApproxPublicDepositVolume, fetchShieldMiningDepositedTotal } from "./indexer"

function parsePositiveDivisor(raw: string | undefined): bigint | null {
  try {
    const parsed = BigInt(raw || "0")
    return parsed > 0n ? parsed : null
  } catch {
    return null
  }
}

export async function fetchPublicDashboardStats() {
  return fetchApproxPublicDepositVolume()
}

export async function fetchShieldMiningEstimate(address: string) {
  const divisor = parsePositiveDivisor(process.env.NEXT_PUBLIC_SHIELD_MINING_POINTS_DIVISOR)
  if (!divisor) return null

  const totalDeposited = await fetchShieldMiningDepositedTotal(address)

  return {
    totalDeposited,
    divisor,
    estimatedPoints: totalDeposited / divisor
  }
}
```

39.5 File: apps/web/src/lib/history.ts

```ts
import type { DepositHistoryRow, ScopedBatchOutputRow, WithdrawalRow } from "./indexer"
import type { LiveNote } from "./note-crypto"

export type TruthKind =
  | "chain_indexed"
  | "onchain_calldata_decoded"
  | "locally_reconstructed"
  | "provenance_record"
export type HistoryAction = "deposit" | "spend" | "change"
export type HistoryAsset = "USDC" | "USDT"

export type HistoryRow = {
  id: string
  action: HistoryAction
  asset: HistoryAsset
  assetId: 1 | 2
  timestamp: number
  headline: string
  amount?: string
  txHash?: string
  commitment?: string
  nullifierHash?: string
  protocolFee?: string
  executorFee?: string
  childKeys?: string[]
  origin?: "tempo" | "spoke"
  truth: TruthKind[]
  provenance?: {
    guid: string
    settlementChain: string
    settlementEid: number
    settlementTxHash: string
    sourceChain?: string
    sourceEid: number
    sourceTxHash?: string
    sourceSender?: string
    composeFrom: string
  }
  localScope?: {
    sourceNoteCommitment?: string
  }
}

function assetLabel(assetId: 1 | 2): HistoryAsset {
  return assetId === 1 ? "USDC" : "USDT"
}

function parseChildKeys(value: string[] | string | undefined): string[] {
  if (!value) return []
  const keys = Array.isArray(value) ? value : JSON.parse(value)
  return keys.filter(
    (key: string) =>
      typeof key === "string" &&
      key !== "0x0000000000000000000000000000000000000000000000000000000000000000"
  )
}

export function buildHistoryRowsForAsset(args: {
  assetId: 1 | 2
  notes: LiveNote[]
  deposits: DepositHistoryRow[]
  withdrawals: WithdrawalRow[]
}): HistoryRow[] {
  const asset = assetLabel(args.assetId)
  const depositByCommitment = new Map(
    args.deposits.map((row) => [String(row.commitment).toLowerCase(), row] as const)
  )
  const withdrawalByNullifier = new Map(
    args.withdrawals.map((row) => [String(row.nullifierHash).toLowerCase(), row] as const)
  )

  const rows: HistoryRow[] = []

  for (const note of args.notes) {
    const commitmentKey = note.commitment.toLowerCase()
    const depositRow = depositByCommitment.get(commitmentKey)

    if (note.kind === "deposit" && depositRow) {
      rows.push({
        id: `deposit-${note.commitment}`,
        action: "deposit",
        asset,
        assetId: args.assetId,
        timestamp: Number(depositRow.timestamp),
        headline: "Shielded Deposit",
        amount: note.amount,
        txHash: depositRow.txHash,
        commitment: note.commitment,
        origin: depositRow.origin,
        truth: depositRow.provenance
          ? ["chain_indexed", "provenance_record", "locally_reconstructed"]
          : ["chain_indexed", "locally_reconstructed"],
        provenance: depositRow.provenance
      })
    }

    if (note.kind === "change") {
      rows.push({
        id: `change-${note.commitment}`,
        action: "change",
        asset,
        assetId: args.assetId,
        timestamp: note.createdAt,
        headline: "Change Note Created",
        amount: note.amount,
        commitment: note.commitment,
        truth: ["locally_reconstructed"]
      })
    }

    if (note.status === "spent") {
      const withdrawal = withdrawalByNullifier.get(note.nullifierHash.toLowerCase())
      if (!withdrawal) continue

      rows.push({
        id: `spend-${note.nullifierHash}`,
        action: "spend",
        asset,
        assetId: args.assetId,
        timestamp: Number(withdrawal.timestamp),
        headline: "Private Execution",
        amount: note.amount,
        txHash: withdrawal.txHash,
        nullifierHash: note.nullifierHash,
        protocolFee: withdrawal.protocolFee,
        executorFee: withdrawal.executorFee,
        childKeys: parseChildKeys(withdrawal.childKeys),
        origin: depositRow?.origin,
        truth: depositRow?.provenance
          ? ["chain_indexed", "locally_reconstructed", "provenance_record"]
          : ["chain_indexed", "locally_reconstructed"],
        provenance: depositRow?.provenance,
        localScope: {
          sourceNoteCommitment: note.commitment
        }
      })
    }
  }

  return rows.sort((a, b) => b.timestamp - a.timestamp)
}

export function flattenHistoryRowsForCsv(rows: HistoryRow[]) {
  return rows.map((row) => ({
    action: row.action,
    asset: row.asset,
    timestamp: new Date(row.timestamp * 1000).toISOString(),
    headline: row.headline,
    amount_base_units: row.amount || "",
    tx_hash: row.txHash || "",
    commitment: row.commitment || "",
    nullifier_hash: row.nullifierHash || "",
    protocol_fee_base_units: row.protocolFee || "",
    executor_fee_base_units: row.executorFee || "",
    truth_labels: row.truth.join(","),
    origin: row.origin || "",
    source_note_commitment: row.localScope?.sourceNoteCommitment || "",
    provenance_guid: row.provenance?.guid || "",
    provenance_settlement_chain: row.provenance?.settlementChain || "",
    provenance_settlement_eid: row.provenance?.settlementEid || "",
    provenance_source_chain: row.provenance?.sourceChain || "",
    provenance_source_eid: row.provenance?.sourceEid || "",
    provenance_source_tx_hash: row.provenance?.sourceTxHash || "",
    provenance_source_sender: row.provenance?.sourceSender || "",
    provenance_settlement_tx_hash: row.provenance?.settlementTxHash || ""
  }))
}
```

39.6 File: apps/web/src/lib/receipt-pdf.ts

```ts
import { PDFDocument, StandardFonts, rgb } from "pdf-lib"
import { formatUnits } from "viem"
import type { HistoryRow } from "./history"
import type { ScopedBatchOutputRow } from "./indexer"

export type ReceiptSection = {
  title: string
  rows: Array<{ label: string; value: string }>
}

function formatAssetAmount(value: string | undefined, asset: string) {
  if (!value) return undefined
  return `${formatUnits(BigInt(value), 6)} ${asset}`
}

function pushIfPresent(
  rows: Array<{ label: string; value: string }>,
  label: string,
  value: string | number | undefined | null
) {
  if (value === undefined || value === null || value === "") return
  rows.push({ label, value: String(value) })
}

export function buildReceiptSections(
  row: HistoryRow,
  selectedChildKey?: string,
  selectedOutput?: ScopedBatchOutputRow | null
): ReceiptSection[] {
  const settlementFacts: ReceiptSection = {
    title: "Verified Settlement Facts",
    rows: []
  }
  pushIfPresent(settlementFacts.rows, "Protocol", "Parly")
  pushIfPresent(settlementFacts.rows, "Network", "Tempo")
  pushIfPresent(settlementFacts.rows, "Action", "Private Execution")
  pushIfPresent(settlementFacts.rows, "Asset", row.asset)
  pushIfPresent(settlementFacts.rows, "Confirmed Tempo Transaction Hash", row.txHash)
  pushIfPresent(
    settlementFacts.rows,
    "Finalized Timestamp",
    new Date(row.timestamp * 1000).toISOString()
  )
  pushIfPresent(settlementFacts.rows, "Protocol Fee", formatAssetAmount(row.protocolFee, row.asset))
  pushIfPresent(settlementFacts.rows, "Executor Fee", formatAssetAmount(row.executorFee, row.asset))
  pushIfPresent(settlementFacts.rows, "Selected Child Key", selectedChildKey)

  const payoutScope: ReceiptSection = {
    title: "Verified Payout Scope",
    rows: []
  }
  pushIfPresent(payoutScope.rows, "Recipient Address", selectedOutput?.recipient)
  pushIfPresent(
    payoutScope.rows,
    "Payout Amount",
    formatAssetAmount(selectedOutput?.amount, row.asset)
  )
  pushIfPresent(payoutScope.rows, "Destination Chain", selectedOutput?.destinationChain)
  pushIfPresent(payoutScope.rows, "Destination EID", selectedOutput?.destEid)
  pushIfPresent(
    payoutScope.rows,
    "Route Type",
    selectedOutput
      ? selectedOutput.routeType === "same_chain"
        ? "Same-Chain"
        : "Cross-Chain"
      : undefined
  )
  pushIfPresent(payoutScope.rows, "Output Index", selectedOutput?.outputIndex)

  const localScope: ReceiptSection = {
    title: "Private Local Scope",
    rows: []
  }
  pushIfPresent(localScope.rows, "Source Note Commitment", row.localScope?.sourceNoteCommitment)

  const provenance: ReceiptSection = {
    title: "Origin Provenance",
    rows: []
  }
  pushIfPresent(provenance.rows, "LayerZero GUID", row.provenance?.guid)
  pushIfPresent(provenance.rows, "Source Chain", row.provenance?.sourceChain)
  pushIfPresent(provenance.rows, "Source Transaction Hash", row.provenance?.sourceTxHash)
  pushIfPresent(provenance.rows, "Source Sender", row.provenance?.sourceSender)
  pushIfPresent(provenance.rows, "Tempo Settlement Anchor", row.provenance?.settlementTxHash)

  return [settlementFacts, payoutScope, localScope, provenance].filter(
    (section) => section.rows.length > 0
  )
}

export async function exportReceiptPdf(
  row: HistoryRow,
  selectedChildKey?: string,
  selectedOutput?: ScopedBatchOutputRow | null
) {
  const doc = await PDFDocument.create()
  const titleFont = await doc.embedFont(StandardFonts.HelveticaBold)
  const bodyFont = await doc.embedFont(StandardFonts.Helvetica)
  const sections = buildReceiptSections(row, selectedChildKey, selectedOutput)

  let page = doc.addPage([612, 792])
  let y = 744

  const addPageIfNeeded = () => {
    if (y >= 90) return
    page = doc.addPage([612, 792])
    y = 744
  }

  page.drawText("Parly", {
    x: 40,
    y,
    size: 20,
    font: titleFont,
    color: rgb(0.91, 0.98, 1)
  })
  y -= 28
  page.drawText("Scoped Compliance Receipt", {
    x: 40,
    y,
    size: 16,
    font: titleFont,
    color: rgb(0.3, 0.88, 0.9)
  })
  y -= 22
  page.drawText(
    "Private settlement. Selective disclosure. Audit-ready proof.",
    {
      x: 40,
      y,
      size: 10,
      font: bodyFont,
      color: rgb(0.72, 0.8, 0.89),
      maxWidth: 520
    }
  )
  y -= 34

  for (const section of sections) {
    addPageIfNeeded()
    page.drawText(section.title, {
      x: 40,
      y,
      size: 12,
      font: titleFont,
      color: rgb(0.3, 0.88, 0.9)
    })
    y -= 18

    for (const fact of section.rows) {
      addPageIfNeeded()

      page.drawText(`${fact.label}: ${fact.value}`, {
        x: 48,
        y,
        size: 10,
        font: bodyFont,
        color: rgb(0.94, 0.97, 1),
        maxWidth: 510
      })
      y -= 16
    }

    y -= 10
  }

  addPageIfNeeded()
  page.drawText("Independent Verification", {
    x: 40,
    y,
    size: 12,
    font: titleFont,
    color: rgb(0.3, 0.88, 0.9)
  })
  y -= 18
  page.drawText(
    "Open Parly Verify and enter the confirmed Tempo transaction hash together with the selected child key to confirm this payout scope independently.",
    {
      x: 48,
      y,
      size: 10,
      font: bodyFont,
      color: rgb(0.94, 0.97, 1),
      maxWidth: 510
    }
  )
  y -= 28
  page.drawText(
    "This document discloses one payout lane only. The remainder of the private execution remains undisclosed by design.",
    {
      x: 48,
      y,
      size: 9,
      font: bodyFont,
      color: rgb(0.72, 0.8, 0.89),
      maxWidth: 510
    }
  )

  const bytes = await doc.save()
  const blob = new Blob([bytes], { type: "application/pdf" })
  const url = URL.createObjectURL(blob)
  const link = document.createElement("a")
  link.href = url
  link.download = `parly-scoped-compliance-receipt-${row.asset.toLowerCase()}-${row.timestamp}.pdf`
  link.click()
  setTimeout(() => URL.revokeObjectURL(url), 500)
}
```

39.7 File: apps/web/src/components/TruthBadge.tsx

```tsx
const truthCopy = {
  chain_indexed: { label: "Chain-Indexed", className: "border-cyan-400/40 bg-cyan-500/10 text-cyan-100" },
  onchain_calldata_decoded: { label: "Calldata-Decoded", className: "border-fuchsia-400/40 bg-fuchsia-500/10 text-fuchsia-100" },
  locally_reconstructed: { label: "Local Scope", className: "border-amber-400/40 bg-amber-500/10 text-amber-100" },
  provenance_record: { label: "Provenance", className: "border-emerald-400/40 bg-emerald-500/10 text-emerald-100" }
} as const

export default function TruthBadge({
  truth
}: {
  truth: keyof typeof truthCopy
}) {
  const item = truthCopy[truth]
  return (
    <span className={`inline-flex items-center rounded-full border px-2.5 py-1 text-[11px] font-semibold uppercase tracking-[0.18em] ${item.className}`}>
      {item.label}
    </span>
  )
}
```

39.8 File: apps/web/src/components/TvlTicker.tsx

```tsx
"use client"

import { useEffect, useState } from "react"
import { formatUnits } from "viem"
import { fetchPublicDashboardStats } from "../lib/dashboard"

export default function TvlTicker() {
  const [text, setText] = useState("Public deposit volume loading...")

  useEffect(() => {
    let cancelled = false

    fetchPublicDashboardStats()
      .then((stats) => {
        if (cancelled) return
        setText(
          `Public deposit volume approx. ${formatUnits(stats.total, 6)} stablecoin units`
        )
      })
      .catch(() => {
        if (!cancelled) {
          setText("Public deposit volume unavailable")
        }
      })

    return () => {
      cancelled = true
    }
  }, [])

  return <div className="text-xs uppercase tracking-[0.22em] text-slate-400">{text}</div>
}
```

39.9 File: apps/web/src/components/ShieldMiningPill.tsx

```tsx
"use client"

import { useEffect, useState } from "react"
import { formatUnits } from "viem"
import { fetchShieldMiningEstimate } from "../lib/dashboard"
import { useParlySession } from "../lib/parly-session"

export default function ShieldMiningPill() {
  const { address, phase } = useParlySession()
  const [text, setText] = useState("Shield mining")

  useEffect(() => {
    let cancelled = false

    if (!address || phase !== "ready") {
      setText("Authenticate for shield mining")
      return
    }

    fetchShieldMiningEstimate(address)
      .then((estimate) => {
        if (cancelled) return
        if (!estimate) {
          setText("Shield mining unavailable")
          return
        }
        setText(
          `${estimate.estimatedPoints.toString()} pts from ${formatUnits(
            estimate.totalDeposited,
            6
          )} public deposit units`
        )
      })
      .catch(() => {
        if (!cancelled) setText("Shield mining unavailable")
      })

    return () => {
      cancelled = true
    }
  }, [address, phase])

  return (
    <div className="rounded-full border border-emerald-400/30 bg-emerald-500/10 px-3 py-1 text-[11px] font-semibold uppercase tracking-[0.18em] text-emerald-100">
      {text}
    </div>
  )
}
```

39.10 File: apps/web/src/components/SiteChrome.tsx

```tsx
"use client"

import type { ReactNode } from "react"
import Link from "next/link"
import { usePathname } from "next/navigation"
import { useAccount } from "wagmi"
import TvlTicker from "./TvlTicker"
import ShieldMiningPill from "./ShieldMiningPill"
import { useParlySession } from "../lib/parly-session"

const navItems = [
  { href: "/", label: "Home" },
  { href: "/app", label: "App" },
  { href: "/history", label: "Parly Ledger & Compliance" },
  { href: "/spoke-shield", label: "Spoke Shield" },
  { href: "/relayer", label: "Relayer" },
  { href: "/verify", label: "Verify a Parly Payout Scope" },
  { href: "/docs", label: "Docs" },
  { href: "/sdk", label: "Agentic SDK" }
]

export default function SiteChrome({ children }: { children: ReactNode }) {
  const pathname = usePathname()
  const { address } = useAccount()
  const { phase } = useParlySession()

  return (
    <>
      <header className="sticky top-0 z-40 border-b border-white/10 bg-slate-950/85 backdrop-blur-xl">
        <div className="mx-auto flex w-full max-w-7xl flex-col gap-4 px-4 py-4 lg:flex-row lg:items-center lg:justify-between">
          <div className="flex flex-col gap-3 lg:flex-row lg:items-center">
            <Link href="/" className="text-lg font-black tracking-[0.22em] text-white">
              PARLY.FI
            </Link>
            <TvlTicker />
          </div>

          <nav className="flex flex-wrap items-center gap-2">
            {navItems.map((item) => {
              const active = pathname === item.href
              return (
                <Link
                  key={item.href}
                  href={item.href}
                  className={`rounded-full px-3 py-2 text-sm transition ${
                    active
                      ? "bg-white text-slate-950"
                      : "text-slate-300 hover:bg-white/10 hover:text-white"
                  }`}
                >
                  {item.label}
                </Link>
              )
            })}
          </nav>

          <div className="flex flex-wrap items-center gap-3">
            <ShieldMiningPill />
            <div className="rounded-full border border-white/10 bg-white/5 px-3 py-2 text-[11px] font-semibold uppercase tracking-[0.18em] text-slate-300">
              {phase === "ready"
                ? address
                  ? `${address.slice(0, 6)}...${address.slice(-4)}`
                  : "Ready"
                : phase.replaceAll("_", " ")}
            </div>
          </div>
        </div>
      </header>

      <main className="mx-auto w-full max-w-7xl px-4 py-8">{children}</main>
    </>
  )
}
```
39.11 File: apps/web/src/components/ReceiptPreviewModal.tsx

```tsx
"use client"

import { formatUnits } from "viem"
import type { HistoryRow } from "../lib/history"
import type { ScopedBatchOutputRow } from "../lib/indexer"
import { buildReceiptSections, exportReceiptPdf } from "../lib/receipt-pdf"
import TruthBadge from "./TruthBadge"

export default function ReceiptPreviewModal({
  row,
  availableOutputs,
  selectedChildKey,
  selectedOutput,
  outputLoading,
  outputError,
  onSelectChildKey,
  onClose
}: {
  row: HistoryRow
  availableOutputs: ScopedBatchOutputRow[]
  selectedChildKey?: string
  selectedOutput?: ScopedBatchOutputRow | null
  outputLoading: boolean
  outputError: string | null
  onSelectChildKey: (value: string) => void
  onClose: () => void
}) {
  const sections = buildReceiptSections(row, selectedChildKey, selectedOutput)
  const truth = selectedOutput
    ? Array.from(new Set([...row.truth, "onchain_calldata_decoded" as const]))
    : row.truth
  const selectingScope = !selectedChildKey

  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center bg-slate-950/80 p-4 backdrop-blur-sm">
      <div className="max-h-[90vh] w-full max-w-3xl overflow-y-auto rounded-3xl border border-white/10 bg-slate-950 p-6 shadow-2xl shadow-cyan-950/30">
        <div className="flex items-start justify-between gap-6">
          <div>
            {selectingScope ? (
              <>
                <p className="text-xs uppercase tracking-[0.25em] text-cyan-300">Review Payout Scopes</p>
                <h3 className="mt-2 text-2xl font-black text-white">Select Payout Scope</h3>
                <p className="mt-3 max-w-2xl text-sm text-slate-400">
                  This private execution contains multiple payout lanes. Choose the single payout
                  scope you want to review, export, or verify. The rest of the batch remains private.
                </p>
              </>
            ) : (
              <>
                <p className="text-xs uppercase tracking-[0.25em] text-cyan-300">Compliance Preview</p>
                <h3 className="mt-2 text-2xl font-black text-white">Parly Scoped Compliance Receipt</h3>
                <p className="mt-3 max-w-2xl text-sm text-slate-400">
                  Export a premium audit-ready record for one verified payout lane. Designed for
                  finance teams, exchanges, auditors, and counterparties.
                </p>
              </>
            )}
          </div>

          <div className="flex flex-wrap gap-3">
            {!selectingScope && (
              <button
                onClick={() => exportReceiptPdf(row, selectedChildKey, selectedOutput)}
                disabled={!selectedOutput}
                className="rounded-full bg-cyan-400 px-4 py-2 text-sm font-semibold text-slate-950 transition hover:bg-cyan-300 disabled:cursor-not-allowed disabled:opacity-50"
              >
                Download PDF
              </button>
            )}
            <button
              onClick={onClose}
              className="rounded-full border border-white/10 px-3 py-2 text-sm text-slate-300 transition hover:border-white/30 hover:text-white"
            >
              Close
            </button>
          </div>
        </div>

        <div className="mt-5 flex flex-wrap gap-2">
          {truth.map((truth) => (
            <TruthBadge key={truth} truth={truth} />
          ))}
        </div>

        {outputLoading && (
          <div className="mt-6 rounded-2xl border border-fuchsia-400/20 bg-fuchsia-500/10 p-4 text-sm text-fuchsia-100">
            Resolving disclosed payout lane from confirmed Tempo calldata...
          </div>
        )}

        {outputError && (
          <div className="mt-6 rounded-2xl border border-red-500/30 bg-red-500/10 p-4 text-sm text-red-100">
            {outputError}
          </div>
        )}

        {selectingScope ? (
          <div className="mt-6 space-y-4">
            <div className="text-xs font-semibold uppercase tracking-[0.18em] text-slate-500">
              Available Payout Scopes
            </div>

            {availableOutputs.length === 0 && !outputLoading && (
              <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-5 text-sm text-slate-400">
                No decoded payout scopes are available for this execution yet.
              </div>
            )}

            {availableOutputs.map((output) => {
              const scopeTruth = row.provenance
                ? ["chain_indexed", "onchain_calldata_decoded", "locally_reconstructed", "provenance_record"] as const
                : ["chain_indexed", "onchain_calldata_decoded", "locally_reconstructed"] as const

              return (
                <div
                  key={output.childKey}
                  className="rounded-2xl border border-white/10 bg-white/[0.03] p-5"
                >
                  <div className="flex flex-col gap-4 xl:flex-row xl:items-start xl:justify-between">
                    <div className="space-y-3">
                      <div className="text-xs font-semibold uppercase tracking-[0.18em] text-slate-500">
                        Payout Scope {output.outputIndex + 1}
                      </div>
                      <div className="flex flex-wrap gap-2">
                        {scopeTruth.map((item) => (
                          <TruthBadge key={`${output.childKey}-${item}`} truth={item} />
                        ))}
                      </div>
                      <div className="grid gap-3 text-sm text-slate-300 md:grid-cols-2">
                        <div>
                          <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">
                            Recipient
                          </div>
                          <div className="mt-1 break-all">{output.recipient}</div>
                        </div>
                        <div>
                          <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">
                            Amount
                          </div>
                          <div className="mt-1">
                            {formatUnits(BigInt(output.amount), 6)} {row.asset}
                          </div>
                        </div>
                        <div>
                          <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">
                            Destination Chain
                          </div>
                          <div className="mt-1">{output.destinationChain}</div>
                        </div>
                        <div>
                          <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">
                            Destination EID
                          </div>
                          <div className="mt-1">{output.destEid}</div>
                        </div>
                        <div>
                          <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">
                            Route Type
                          </div>
                          <div className="mt-1">
                            {output.routeType === "same_chain" ? "Same-Chain" : "Cross-Chain"}
                          </div>
                        </div>
                        <div>
                          <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">
                            Child Key
                          </div>
                          <div className="mt-1 break-all">{output.childKey}</div>
                        </div>
                      </div>
                    </div>

                    <button
                      onClick={() => onSelectChildKey(output.childKey)}
                      className="rounded-full bg-cyan-400 px-4 py-3 text-sm font-semibold text-slate-950 transition hover:bg-cyan-300"
                    >
                      Open Scoped Receipt
                    </button>
                  </div>
                </div>
              )
            })}

            <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-4 text-sm text-slate-400">
              <div className="font-semibold text-slate-200">Private by default.</div>
              <p className="mt-2">
                You are opening one disclosed payout lane only. This gives auditors and business
                counterparties the exact lane they need without exposing the wider execution set.
              </p>
            </div>
          </div>
        ) : (
          <>
            <button
              onClick={() => onSelectChildKey("")}
              className="mt-6 rounded-full border border-white/10 px-3 py-2 text-sm text-slate-300 transition hover:border-white/30 hover:text-white"
            >
              Back to scopes
            </button>

            <div className="mt-6 rounded-2xl border border-white/10 bg-white/[0.03] p-4 text-sm text-slate-400">
              <div className="font-semibold text-slate-200">Scope & Source Notice</div>
              <p className="mt-2">
                This receipt presents confirmed chain-indexed facts and, where labeled, scoped local
                or provenance-linked context. It is intentionally limited to one disclosed payout lane
                and does not claim facts beyond that scope.
              </p>
              <p className="mt-2">
                Shared route delivery cost remains execution-scoped and is intentionally omitted from this payout-scoped disclosure.
              </p>
            </div>

            <div className="mt-6 space-y-4">
              {sections.map((section) => (
                <div key={section.title} className="rounded-2xl border border-white/10 bg-white/[0.03] p-4">
                  <div className="text-xs font-semibold uppercase tracking-[0.2em] text-cyan-300">
                    {section.title}
                  </div>
                  {section.title === "Verified Payout Scope" && (
                    <p className="mt-3 text-sm text-slate-400">
                      This section defines the exact payout lane disclosed for business verification. It
                      does not expose the rest of the execution batch.
                    </p>
                  )}
                  <div className="mt-4 space-y-3">
                    {section.rows.map((fact) => (
                      <div key={`${section.title}-${fact.label}`} className="rounded-xl border border-white/5 bg-slate-950/70 p-3">
                        <div className="text-[11px] font-semibold uppercase tracking-[0.18em] text-slate-500">
                          {fact.label}
                        </div>
                        <div className="mt-2 break-all text-sm text-slate-100">{fact.value}</div>
                      </div>
                    ))}
                  </div>
                </div>
              ))}
            </div>

            <div className="mt-6 rounded-2xl border border-cyan-400/20 bg-cyan-500/10 p-4 text-sm text-cyan-100">
              <div className="font-semibold">Verify Independently</div>
              <p className="mt-2">
                To independently validate this payout scope, open Parly Verify and enter:
              </p>
              <ol className="mt-3 list-decimal space-y-1 pl-5 text-cyan-50">
                <li>the confirmed Tempo transaction hash</li>
                <li>the selected child key</li>
              </ol>
              <p className="mt-3">
                This document is selective disclosure by design. It proves one payout lane without
                opening the rest of the private execution.
              </p>
            </div>
          </>
        )}
      </div>
    </div>
  )
}
```

39.12 File: apps/web/src/components/HistoryCompliancePanel.tsx

```tsx
"use client"

import { useDeferredValue, useEffect, useMemo, useState } from "react"
import Papa from "papaparse"
import { formatUnits } from "viem"
import {
  fetchDepositHistory,
  fetchScopedBatchOutputs,
  fetchWithdrawals,
  type ScopedBatchOutputRow
} from "../lib/indexer"
import {
  buildHistoryRowsForAsset,
  flattenHistoryRowsForCsv,
  type HistoryAction,
  type HistoryRow
} from "../lib/history"
import { useParlySession } from "../lib/parly-session"
import { useRecoveredNotes } from "../lib/useRecoveredNotes"
import ReceiptPreviewModal from "./ReceiptPreviewModal"
import TruthBadge from "./TruthBadge"

type AssetFilter = "ALL" | "USDC" | "USDT"
type ActionFilter = "ALL" | HistoryAction
type OriginFilter = "ALL" | "tempo" | "spoke"

function formatAssetAmount(value: string | undefined, asset: string) {
  if (!value) return `0 ${asset}`
  return `${formatUnits(BigInt(value), 6)} ${asset}`
}

function formatTimestamp(value: number) {
  return new Date(value * 1000).toLocaleString()
}

export default function HistoryCompliancePanel() {
  const session = useParlySession()
  const [assetFilter, setAssetFilter] = useState<AssetFilter>("ALL")
  const [actionFilter, setActionFilter] = useState<ActionFilter>("ALL")
  const [originFilter, setOriginFilter] = useState<OriginFilter>("ALL")
  const [query, setQuery] = useState("")
  const [expandedRowId, setExpandedRowId] = useState<string | null>(null)
  const [selectedRow, setSelectedRow] = useState<HistoryRow | null>(null)
  const [selectedChildKey, setSelectedChildKey] = useState("")
  const [loadingData, setLoadingData] = useState(false)
  const [dataError, setDataError] = useState<string | null>(null)
  const [usdcDeposits, setUsdcDeposits] = useState<any[]>([])
  const [usdtDeposits, setUsdtDeposits] = useState<any[]>([])
  const [usdcWithdrawals, setUsdcWithdrawals] = useState<any[]>([])
  const [usdtWithdrawals, setUsdtWithdrawals] = useState<any[]>([])
  const [scopedOutputsByTxHash, setScopedOutputsByTxHash] = useState<Record<string, ScopedBatchOutputRow[]>>({})
  const [scopedOutputLoading, setScopedOutputLoading] = useState<Record<string, boolean>>({})
  const [scopedOutputErrors, setScopedOutputErrors] = useState<Record<string, string>>({})
  const deferredQuery = useDeferredValue(query.trim().toLowerCase())

  const usdcRecovery = useRecoveredNotes({
    address: session.address,
    assetId: 1,
    recoveryPublicKeyB64: session.recoveryPublicKeyB64,
    recoveryPrivateKeyB64: session.recoveryPrivateKeyB64,
    optionalCacheKey: session.optionalCacheKey,
    refreshNonce: session.recoveryEpoch
  })

  const usdtRecovery = useRecoveredNotes({
    address: session.address,
    assetId: 2,
    recoveryPublicKeyB64: session.recoveryPublicKeyB64,
    recoveryPrivateKeyB64: session.recoveryPrivateKeyB64,
    optionalCacheKey: session.optionalCacheKey,
    refreshNonce: session.recoveryEpoch
  })

  useEffect(() => {
    session.setRecoveryActivity("history", usdcRecovery.loading || usdtRecovery.loading)
    return () => {
      session.setRecoveryActivity("history", false)
    }
  }, [session, usdcRecovery.loading, usdtRecovery.loading])

  useEffect(() => {
    if (session.phase !== "ready") return

    let cancelled = false
    setLoadingData(true)
    setDataError(null)

    Promise.all([
      fetchDepositHistory(1),
      fetchDepositHistory(2),
      fetchWithdrawals(1),
      fetchWithdrawals(2)
    ])
      .then(([nextUsdcDeposits, nextUsdtDeposits, nextUsdcWithdrawals, nextUsdtWithdrawals]) => {
        if (cancelled) return
        setUsdcDeposits(nextUsdcDeposits)
        setUsdtDeposits(nextUsdtDeposits)
        setUsdcWithdrawals(nextUsdcWithdrawals)
        setUsdtWithdrawals(nextUsdtWithdrawals)
      })
      .catch((e: any) => {
        if (!cancelled) {
          setDataError(e?.message || "History fetch failed")
        }
      })
      .finally(() => {
        if (!cancelled) {
          setLoadingData(false)
        }
      })

    return () => {
      cancelled = true
    }
  }, [session.phase, session.recoveryEpoch])

  const allRows = useMemo(() => {
    const usdcRows = buildHistoryRowsForAsset({
      assetId: 1,
      notes: usdcRecovery.notes,
      deposits: usdcDeposits,
      withdrawals: usdcWithdrawals
    })
    const usdtRows = buildHistoryRowsForAsset({
      assetId: 2,
      notes: usdtRecovery.notes,
      deposits: usdtDeposits,
      withdrawals: usdtWithdrawals
    })

    return [...usdcRows, ...usdtRows].sort((a, b) => b.timestamp - a.timestamp)
  }, [
    usdcRecovery.notes,
    usdtRecovery.notes,
    usdcDeposits,
    usdtDeposits,
    usdcWithdrawals,
    usdtWithdrawals
  ])

  const baseFilteredRows = useMemo(() => {
    return allRows.filter((row) => {
      if (assetFilter !== "ALL" && row.asset !== assetFilter) return false
      if (actionFilter !== "ALL" && row.action !== actionFilter) return false
      if (originFilter !== "ALL" && row.origin !== originFilter) return false
      return true
    })
  }, [allRows, assetFilter, actionFilter, originFilter])

  useEffect(() => {
    const targets = new Set<string>()

    if (selectedRow?.txHash) {
      targets.add(selectedRow.txHash.toLowerCase())
    }

    const expandedRow = allRows.find((row) => row.id === expandedRowId)
    if (expandedRow?.action === "spend" && expandedRow.txHash) {
      targets.add(expandedRow.txHash.toLowerCase())
    }

    if (deferredQuery) {
      for (const row of baseFilteredRows) {
        if (row.action === "spend" && row.txHash) {
          targets.add(row.txHash.toLowerCase())
        }
      }
    }

    const spendTxHashes = Array.from(targets)

    const missing = spendTxHashes.filter(
      (txHash) =>
        scopedOutputsByTxHash[txHash] === undefined &&
        !scopedOutputLoading[txHash] &&
        !scopedOutputErrors[txHash]
    )

    if (missing.length === 0) return

    let cancelled = false

    setScopedOutputLoading((current) => {
      const next = { ...current }
      for (const txHash of missing) next[txHash] = true
      return next
    })

    Promise.all(
      missing.map(async (txHash) => {
        try {
          const outputs = await fetchScopedBatchOutputs(txHash as `0x${string}`)
          return { txHash, outputs }
        } catch (error: any) {
          return { txHash, error: error?.message || "Failed to decode payout scopes." }
        }
      })
    ).then((results) => {
      if (cancelled) return

      setScopedOutputsByTxHash((current) => {
        const next = { ...current }
        for (const result of results) {
          if ("outputs" in result) {
            next[result.txHash] = result.outputs
          }
        }
        return next
      })

      setScopedOutputErrors((current) => {
        const next = { ...current }
        for (const result of results) {
          if ("error" in result) {
            next[result.txHash] = result.error
          }
        }
        return next
      })

      setScopedOutputLoading((current) => {
        const next = { ...current }
        for (const txHash of missing) next[txHash] = false
        return next
      })
    })

    return () => {
      cancelled = true
    }
  }, [
    allRows,
    baseFilteredRows,
    deferredQuery,
    expandedRowId,
    scopedOutputErrors,
    scopedOutputLoading,
    scopedOutputsByTxHash,
    selectedRow
  ])

  const filteredRows = useMemo(() => {
    return baseFilteredRows.filter((row) => {
      if (!deferredQuery) return true

      const scopedOutputs = row.txHash ? scopedOutputsByTxHash[row.txHash.toLowerCase()] || [] : []

      const haystack = [
        row.headline,
        row.txHash,
        row.commitment,
        row.nullifierHash,
        row.localScope?.sourceNoteCommitment,
        row.provenance?.guid,
        row.provenance?.sourceTxHash,
        row.provenance?.settlementTxHash,
        ...scopedOutputs.flatMap((output) => [
          output.childKey,
          output.recipient,
          output.destinationChain,
          String(output.destEid)
        ])
      ]
        .filter(Boolean)
        .join(" ")
        .toLowerCase()

      return haystack.includes(deferredQuery)
    })
  }, [baseFilteredRows, deferredQuery, scopedOutputsByTxHash])

  const selectedOutputs = useMemo(() => {
    if (!selectedRow?.txHash) return []
    return scopedOutputsByTxHash[selectedRow.txHash.toLowerCase()] || []
  }, [scopedOutputsByTxHash, selectedRow])

  const selectedOutput = useMemo(() => {
    if (!selectedChildKey) return null
    return selectedOutputs.find((output) => output.childKey === selectedChildKey) || null
  }, [selectedChildKey, selectedOutputs])

  const selectedOutputLoading =
    !!selectedRow?.txHash && !!scopedOutputLoading[selectedRow.txHash.toLowerCase()]
  const selectedOutputError =
    selectedRow?.txHash ? scopedOutputErrors[selectedRow.txHash.toLowerCase()] || null : null

  function downloadCsv() {
    const csv = Papa.unparse(flattenHistoryRowsForCsv(filteredRows))
    const blob = new Blob([csv], { type: "text/csv;charset=utf-8;" })
    const url = URL.createObjectURL(blob)
    const link = document.createElement("a")
    link.href = url
    link.download = "parly-ledger-compliance.csv"
    link.click()
    setTimeout(() => URL.revokeObjectURL(url), 500)
  }

  if (session.phase === "disconnected") {
    return (
      <div className="rounded-3xl border border-white/10 bg-slate-950/70 p-8">
        <p className="text-xs uppercase tracking-[0.25em] text-cyan-300">Parly Ledger & Compliance</p>
        <h2 className="mt-3 text-3xl font-black text-white">Connect a wallet to begin</h2>
        <p className="mt-3 max-w-2xl text-sm text-slate-400">
          This surface never invents provenance. It only becomes useful once the wallet and
          deterministic recovery identity are available.
        </p>
      </div>
    )
  }

  if (session.phase === "connected_unauthed") {
    return (
      <div className="rounded-3xl border border-white/10 bg-slate-950/70 p-8">
        <p className="text-xs uppercase tracking-[0.25em] text-cyan-300">Parly Ledger & Compliance</p>
        <h2 className="mt-3 text-3xl font-black text-white">Authenticate to recover history</h2>
        <p className="mt-3 max-w-2xl text-sm text-slate-400">
          The app needs the deterministic recovery signature before it can decrypt note envelopes
          and reconstruct your private activity surface.
        </p>
        <button
          onClick={session.authenticate}
          className="mt-6 rounded-full bg-cyan-400 px-5 py-3 text-sm font-semibold text-slate-950 transition hover:bg-cyan-300"
        >
          Authenticate now
        </button>
      </div>
    )
  }

  return (
    <>
      <div className="rounded-3xl border border-white/10 bg-slate-950/70 p-6 shadow-2xl shadow-cyan-950/10">
        <div className="flex flex-col gap-5 xl:flex-row xl:items-end xl:justify-between">
          <div>
            <p className="text-xs uppercase tracking-[0.25em] text-cyan-300">Parly Ledger & Compliance</p>
            <h2 className="mt-3 text-3xl font-black text-white">Parly Ledger & Compliance</h2>
            <p className="mt-3 max-w-3xl text-sm text-slate-400">
              Track shielded capital, review private executions, and generate audit-ready payout
              proofs from one premium control surface.
            </p>
            <p className="mt-3 max-w-3xl text-sm text-slate-500">
              Every record is truth-labeled by source. Expand a card for deeper detail. Compliance
              exports expose only surfaced card fields here, while payout-scope disclosure stays one lane at a time.
            </p>
          </div>

          <div className="flex flex-wrap gap-3">
            <button
              onClick={session.refreshRecovery}
              className="rounded-full border border-white/10 px-4 py-3 text-sm font-semibold text-slate-200 transition hover:border-white/30 hover:text-white"
            >
              Refresh recovery
            </button>
            <button
              onClick={downloadCsv}
              disabled={filteredRows.length === 0}
              className="rounded-full bg-white px-4 py-3 text-sm font-semibold text-slate-950 transition hover:bg-slate-200 disabled:cursor-not-allowed disabled:opacity-50"
            >
              Export Visible CSV
            </button>
          </div>
        </div>

        <div className="mt-5 flex flex-wrap gap-2">
          <TruthBadge truth="chain_indexed" />
          <TruthBadge truth="onchain_calldata_decoded" />
          <TruthBadge truth="locally_reconstructed" />
          <TruthBadge truth="provenance_record" />
        </div>

        <div className="mt-6 grid gap-4 xl:grid-cols-[1.3fr_0.85fr_0.85fr_0.85fr]">
          <input
            value={query}
            onChange={(event) => setQuery(event.target.value)}
            placeholder="Search tx hash, commitment, nullifier, recipient, or GUID"
            className="rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white"
          />

          <select
            value={assetFilter}
            onChange={(event) => setAssetFilter(event.target.value as AssetFilter)}
            className="rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white"
          >
            <option value="ALL">All Assets</option>
            <option value="USDC">USDC</option>
            <option value="USDT">USDT</option>
          </select>

          <select
            value={actionFilter}
            onChange={(event) => setActionFilter(event.target.value as ActionFilter)}
            className="rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white"
          >
            <option value="ALL">All Activity</option>
            <option value="deposit">Deposits</option>
            <option value="spend">Private Executions</option>
            <option value="change">Change Notes</option>
          </select>

          <select
            value={originFilter}
            onChange={(event) => setOriginFilter(event.target.value as OriginFilter)}
            className="rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white"
          >
            <option value="ALL">All Origins</option>
            <option value="tempo">Tempo Native</option>
            <option value="spoke">Supported Spokes</option>
          </select>
        </div>

        {(loadingData || usdcRecovery.loading || usdtRecovery.loading) && (
          <div className="mt-6 rounded-2xl border border-cyan-400/20 bg-cyan-500/10 p-4 text-sm text-cyan-100">
            Rebuilding deterministic history from indexed leaves, envelopes, and nullifiers...
          </div>
        )}

        {(dataError || usdcRecovery.error || usdtRecovery.error || session.error) && (
          <div className="mt-6 rounded-2xl border border-red-500/30 bg-red-500/10 p-4 text-sm text-red-100">
            {dataError || usdcRecovery.error || usdtRecovery.error || session.error}
          </div>
        )}

        {!loadingData && filteredRows.length === 0 && !dataError && (
          <div className="mt-6 rounded-2xl border border-white/10 bg-white/[0.03] p-6 text-sm text-slate-400">
            No recoverable rows match the current filters yet.
          </div>
        )}

        <div className="mt-6 space-y-4">
          {filteredRows.map((row) => {
            const scopedOutputs = row.txHash ? scopedOutputsByTxHash[row.txHash.toLowerCase()] || [] : []
            const totalPayout =
              row.action === "spend" && scopedOutputs.length > 0
                ? scopedOutputs.reduce((sum, output) => sum + BigInt(output.amount), 0n).toString()
                : row.action === "spend"
                ? null
                : row.amount || "0"
            const isExpanded = expandedRowId === row.id
            const cardType =
              row.action === "deposit"
                ? "Shielded Deposit"
                : row.action === "spend"
                ? "Private Execution"
                : "Change Note Created"
            const primaryAmountDisplay =
              row.action === "spend"
                ? totalPayout
                  ? `- ${formatAssetAmount(totalPayout, row.asset)}`
                  : "Resolving payout total..."
                : `+ ${formatAssetAmount(totalPayout || "0", row.asset)}`

            return (
              <div key={row.id} className="rounded-3xl border border-white/10 bg-white/[0.03] p-5">
                <div className="flex flex-col gap-4 xl:flex-row xl:items-start xl:justify-between">
                  <div className="space-y-4">
                    <div className="flex flex-wrap items-center gap-2">
                      {row.truth.map((truth) => (
                        <TruthBadge key={`${row.id}-${truth}`} truth={truth} />
                      ))}
                    </div>

                    <div className="space-y-2">
                      <div className="text-xs uppercase tracking-[0.2em] text-slate-500">{cardType}</div>
                      <div className="text-3xl font-black text-white">{primaryAmountDisplay}</div>
                      <div className="text-sm text-slate-400">{formatTimestamp(row.timestamp)}</div>
                      {row.txHash ? (
                        <div className="text-sm text-slate-300">
                          Tempo tx <span className="break-all">{row.txHash}</span>
                        </div>
                      ) : row.commitment ? (
                        <div className="text-sm text-slate-300">
                          Commitment <span className="break-all">{row.commitment}</span>
                        </div>
                      ) : null}
                      {row.action === "spend" && (
                        <div className="text-sm text-slate-300">
                          Protocol Fee {formatAssetAmount(row.protocolFee, row.asset)} • Executor Fee{" "}
                          {formatAssetAmount(row.executorFee, row.asset)}
                        </div>
                      )}
                    </div>

                    {row.action === "deposit" && row.origin && (
                      <div className="inline-flex rounded-full border border-white/10 bg-slate-950/60 px-3 py-1 text-[11px] font-semibold uppercase tracking-[0.18em] text-slate-300">
                        {row.origin === "spoke" ? "Supported Spoke" : "Tempo Native"}
                      </div>
                    )}
                  </div>

                  <div className="flex w-full max-w-sm flex-col gap-3">
                    {row.action === "spend" && (
                      <button
                        onClick={() => {
                          setSelectedRow(row)
                          setSelectedChildKey("")
                        }}
                        className="rounded-full bg-cyan-400 px-4 py-3 text-sm font-semibold text-slate-950 transition hover:bg-cyan-300"
                      >
                        Review Payout Scopes
                      </button>
                    )}
                    <button
                      onClick={() => setExpandedRowId(isExpanded ? null : row.id)}
                      className="rounded-full border border-white/10 px-4 py-3 text-sm font-semibold text-slate-200 transition hover:border-white/30 hover:text-white"
                    >
                      {isExpanded ? "Hide Details ▴" : "View Details ▾"}
                    </button>
                  </div>
                </div>

                {isExpanded && (
                  <div className="mt-6 rounded-3xl border border-white/10 bg-slate-950/40 p-5">
                    {row.action === "deposit" && (
                      <div className="space-y-5">
                        <div>
                          <div className="text-xs uppercase tracking-[0.18em] text-cyan-300">
                            Deposit Details
                          </div>
                          <p className="mt-3 max-w-3xl text-sm text-slate-400">
                            {row.origin === "spoke"
                              ? "Capital arrived from a supported spoke network and settled into Parly’s Tempo privacy pool with provenance attached."
                              : "Capital entered the Parly privacy pool directly on Tempo and was converted into shielded balance."}
                          </p>
                        </div>

                        <div className="grid gap-4 md:grid-cols-2 xl:grid-cols-3">
                          <div>
                            <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">Asset</div>
                            <div className="mt-1 text-sm text-slate-100">{row.asset}</div>
                          </div>
                          <div>
                            <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">Shielded Amount</div>
                            <div className="mt-1 text-sm text-slate-100">{formatAssetAmount(row.amount, row.asset)}</div>
                          </div>
                          <div>
                            <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">Confirmed At</div>
                            <div className="mt-1 text-sm text-slate-100">{formatTimestamp(row.timestamp)}</div>
                          </div>
                          {row.txHash && (
                            <div>
                              <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">Tempo Transaction Hash</div>
                              <div className="mt-1 break-all text-sm text-slate-100">{row.txHash}</div>
                            </div>
                          )}
                          {row.commitment && (
                            <div>
                              <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">Commitment</div>
                              <div className="mt-1 break-all text-sm text-slate-100">{row.commitment}</div>
                            </div>
                          )}
                          <div>
                            <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">Origin</div>
                            <div className="mt-1 text-sm text-slate-100">
                              {row.origin === "spoke" ? "Supported Spoke" : "Tempo Native"}
                            </div>
                          </div>
                        </div>

                        {row.provenance ? (
                          <div className="rounded-2xl border border-emerald-400/20 bg-emerald-500/10 p-4">
                            <div className="text-xs uppercase tracking-[0.18em] text-emerald-200">
                              Origin Provenance
                            </div>
                            <div className="mt-4 grid gap-4 md:grid-cols-2">
                              <div>
                                <div className="text-[11px] uppercase tracking-[0.18em] text-emerald-100/70">LayerZero GUID</div>
                                <div className="mt-1 break-all text-sm text-emerald-50">{row.provenance.guid}</div>
                              </div>
                              {row.provenance.sourceChain && (
                                <div>
                                  <div className="text-[11px] uppercase tracking-[0.18em] text-emerald-100/70">Source Chain</div>
                                  <div className="mt-1 text-sm text-emerald-50">{row.provenance.sourceChain}</div>
                                </div>
                              )}
                              {row.provenance.sourceTxHash && (
                                <div>
                                  <div className="text-[11px] uppercase tracking-[0.18em] text-emerald-100/70">Source Transaction Hash</div>
                                  <div className="mt-1 break-all text-sm text-emerald-50">{row.provenance.sourceTxHash}</div>
                                </div>
                              )}
                              {row.provenance.sourceSender && (
                                <div>
                                  <div className="text-[11px] uppercase tracking-[0.18em] text-emerald-100/70">Source Sender</div>
                                  <div className="mt-1 break-all text-sm text-emerald-50">{row.provenance.sourceSender}</div>
                                </div>
                              )}
                              <div>
                                <div className="text-[11px] uppercase tracking-[0.18em] text-emerald-100/70">Tempo Settlement Anchor</div>
                                <div className="mt-1 break-all text-sm text-emerald-50">{row.provenance.settlementTxHash}</div>
                              </div>
                            </div>
                            <p className="mt-4 text-sm text-emerald-50/80">
                              This record gives finance and audit teams a clearer cross-chain trail
                              without weakening the privacy posture of the broader shielded balance.
                            </p>
                          </div>
                        ) : (
                          <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-4 text-sm text-slate-400">
                            This deposit was created natively on the Parly settlement layer. No
                            cross-chain origin record applies to this card.
                          </div>
                        )}
                      </div>
                    )}

                    {row.action === "spend" && (
                      <div className="space-y-5">
                        <div>
                          <div className="text-xs uppercase tracking-[0.18em] text-cyan-300">
                            Execution Details
                          </div>
                          <p className="mt-3 max-w-3xl text-sm text-slate-400">
                            This private execution moved shielded capital through Parly’s settlement
                            engine. Review payout scopes to open one selective-disclosure lane for
                            audit or counterparty verification.
                          </p>
                        </div>

                        <div className="grid gap-4 md:grid-cols-2 xl:grid-cols-3">
                          <div>
                            <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">Asset</div>
                            <div className="mt-1 text-sm text-slate-100">{row.asset}</div>
                          </div>
                          <div>
                            <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">Executed Amount</div>
                            <div className="mt-1 text-sm text-slate-100">
                              {totalPayout
                                ? formatAssetAmount(totalPayout, row.asset)
                                : "Resolving payout total from confirmed Tempo calldata..."}
                            </div>
                          </div>
                          <div>
                            <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">Confirmed At</div>
                            <div className="mt-1 text-sm text-slate-100">{formatTimestamp(row.timestamp)}</div>
                          </div>
                          {row.txHash && (
                            <div>
                              <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">Tempo Transaction Hash</div>
                              <div className="mt-1 break-all text-sm text-slate-100">{row.txHash}</div>
                            </div>
                          )}
                          {row.nullifierHash && (
                            <div>
                              <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">Nullifier Hash</div>
                              <div className="mt-1 break-all text-sm text-slate-100">{row.nullifierHash}</div>
                            </div>
                          )}
                          {row.protocolFee && (
                            <div>
                              <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">Protocol Fee</div>
                              <div className="mt-1 text-sm text-slate-100">{formatAssetAmount(row.protocolFee, row.asset)}</div>
                            </div>
                          )}
                          {row.executorFee && (
                            <div>
                              <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">Executor Fee</div>
                              <div className="mt-1 text-sm text-slate-100">{formatAssetAmount(row.executorFee, row.asset)}</div>
                            </div>
                          )}
                          {row.localScope?.sourceNoteCommitment && (
                            <div>
                              <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">Source Note Commitment</div>
                              <div className="mt-1 break-all text-sm text-slate-100">{row.localScope.sourceNoteCommitment}</div>
                            </div>
                          )}
                        </div>

                        <div className="rounded-2xl border border-amber-400/20 bg-amber-500/10 p-4 text-sm text-amber-50">
                          <div className="font-semibold">
                            Selective disclosure is available at the payout-scope level, not the whole batch level.
                          </div>
                          <p className="mt-2 text-amber-50/80">
                            Shared route delivery cost stays on the Execute surface and is not attributed to one disclosed payout lane here.
                          </p>
                        </div>

                        <button
                          onClick={() => {
                            setSelectedRow(row)
                            setSelectedChildKey("")
                          }}
                          className="rounded-full bg-cyan-400 px-4 py-3 text-sm font-semibold text-slate-950 transition hover:bg-cyan-300"
                        >
                          Review Payout Scopes
                        </button>
                      </div>
                    )}

                    {row.action === "change" && (
                      <div className="space-y-5">
                        <div>
                          <div className="text-xs uppercase tracking-[0.18em] text-cyan-300">
                            Change Note Details
                          </div>
                          <p className="mt-3 max-w-3xl text-sm text-slate-400">
                            This record reflects shielded value returned to the remaining private
                            balance after execution.
                          </p>
                        </div>

                        <div className="grid gap-4 md:grid-cols-2 xl:grid-cols-3">
                          <div>
                            <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">Asset</div>
                            <div className="mt-1 text-sm text-slate-100">{row.asset}</div>
                          </div>
                          <div>
                            <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">Remaining Shielded Value</div>
                            <div className="mt-1 text-sm text-slate-100">{formatAssetAmount(row.amount, row.asset)}</div>
                          </div>
                          <div>
                            <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">Created At</div>
                            <div className="mt-1 text-sm text-slate-100">{formatTimestamp(row.timestamp)}</div>
                          </div>
                          {row.commitment && (
                            <div>
                              <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">Commitment</div>
                              <div className="mt-1 break-all text-sm text-slate-100">{row.commitment}</div>
                            </div>
                          )}
                        </div>

                        <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-4 text-sm text-slate-400">
                          Change stays private inside the user’s shielded state and is not presented
                          as a public payout event.
                        </div>
                      </div>
                    )}
                  </div>
                )}
              </div>
            )
          })}
        </div>
      </div>

      {selectedRow && (
        <ReceiptPreviewModal
          row={selectedRow}
          availableOutputs={selectedOutputs}
          selectedChildKey={selectedChildKey}
          selectedOutput={selectedOutput}
          outputLoading={selectedOutputLoading}
          outputError={selectedOutputError}
          onSelectChildKey={setSelectedChildKey}
          onClose={() => {
            setSelectedRow(null)
            setSelectedChildKey("")
          }}
        />
      )}
    </>
  )
}
```

39.13 File: apps/web/src/components/AdminVault.tsx

```tsx
"use client"

import { useEffect, useMemo, useState } from "react"
import {
  encodeFunctionData,
  formatUnits,
  isAddress,
  parseEther
} from "viem"
import { useAccount, useWriteContract } from "wagmi"
import { POOL_ABI, TREASURY_ABI } from "../lib/abis"
import { tempoPublicClient } from "../lib/public-client"

type TreasuryTx = {
  txIndex: number
  target: `0x${string}`
  value: bigint
  data: `0x${string}`
  executed: boolean
  cancelled: boolean
  numConfirmations: bigint
  confirmedByActiveOwner: boolean
}

function parseBps(raw: string) {
  const value = Number(raw)
  if (!Number.isInteger(value) || value < 0 || value > 100) {
    throw new Error("Fee bps must be an integer between 0 and 100.")
  }
  return BigInt(value)
}

function parseBaseUnits(raw: string) {
  const trimmed = raw.trim()
  if (!/^(0|[1-9][0-9]*)$/.test(trimmed)) {
    throw new Error("Sweep amount must be entered as integer base units.")
  }
  return BigInt(trimmed)
}

export default function AdminVault() {
  const treasury = process.env.NEXT_PUBLIC_PROTOCOL_TREASURY_ADDRESS as `0x${string}` | undefined
  const { address } = useAccount()
  const { writeContractAsync } = useWriteContract()

  const [owners, setOwners] = useState<string[]>([])
  const [threshold, setThreshold] = useState<bigint>(0n)
  const [transactions, setTransactions] = useState<TreasuryTx[]>([])
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)
  const [accessChecked, setAccessChecked] = useState(false)

  const [customTarget, setCustomTarget] = useState("")
  const [customValue, setCustomValue] = useState("0")
  const [customCalldata, setCustomCalldata] = useState("0x")
  const [selectedPool, setSelectedPool] = useState<"USDC" | "USDT">("USDC")
  const [feeBps, setFeeBps] = useState("25")
  const [newSigner, setNewSigner] = useState("")
  const [removeSignerAddress, setRemoveSignerAddress] = useState("")
  const [replaceOldSigner, setReplaceOldSigner] = useState("")
  const [replaceNewSigner, setReplaceNewSigner] = useState("")
  const [sweepToken, setSweepToken] = useState("")
  const [sweepTo, setSweepTo] = useState("")
  const [sweepAmount, setSweepAmount] = useState("")
  const [cancelProposalIndex, setCancelProposalIndex] = useState("")

  const activeOwner = useMemo(() => {
    if (!address) return false
    return owners.some((owner) => owner.toLowerCase() === address.toLowerCase())
  }, [address, owners])

  const selectedPoolAddress =
    selectedPool === "USDC"
      ? process.env.NEXT_PUBLIC_USDC_POOL
      : process.env.NEXT_PUBLIC_USDT_POOL

  async function refresh() {
    if (!treasury || !isAddress(treasury)) {
      setError("NEXT_PUBLIC_PROTOCOL_TREASURY_ADDRESS is missing or invalid.")
      setAccessChecked(true)
      setLoading(false)
      setOwners([])
      setThreshold(0n)
      setTransactions([])
      return
    }

    if (!address) {
      setError(null)
      setAccessChecked(true)
      setLoading(false)
      setOwners([])
      setThreshold(0n)
      setTransactions([])
      return
    }

    setLoading(true)
    setError(null)

    try {
      const nextOwners = await tempoPublicClient.readContract({
        address: treasury,
        abi: TREASURY_ABI,
        functionName: "getOwners"
      }) as string[]

      setOwners(nextOwners)

      const isActiveOwner = nextOwners.some((owner) => owner.toLowerCase() === address.toLowerCase())
      setAccessChecked(true)

      if (!isActiveOwner) {
        setThreshold(0n)
        setTransactions([])
        return
      }

      const nextThreshold = await tempoPublicClient.readContract({
        address: treasury,
        abi: TREASURY_ABI,
        functionName: "threshold"
      }) as bigint

      const txCount = await tempoPublicClient.readContract({
        address: treasury,
        abi: TREASURY_ABI,
        functionName: "getTransactionCount"
      }) as bigint

      const total = Number(txCount)
      const start = Math.max(0, total - 20)
      const nextTransactions: TreasuryTx[] = []

      for (let index = total - 1; index >= start; index--) {
        const row = await tempoPublicClient.readContract({
          address: treasury,
          abi: TREASURY_ABI,
          functionName: "transactions",
          args: [BigInt(index)]
        }) as [
          `0x${string}`,
          bigint,
          `0x${string}`,
          boolean,
          boolean,
          bigint
        ]

        let confirmedByActiveOwner = false
        if (address) {
          confirmedByActiveOwner = await tempoPublicClient.readContract({
            address: treasury,
            abi: TREASURY_ABI,
            functionName: "isConfirmed",
            args: [BigInt(index), address]
          }) as boolean
        }

        nextTransactions.push({
          txIndex: index,
          target: row[0],
          value: row[1],
          data: row[2],
          executed: row[3],
          cancelled: row[4],
          numConfirmations: row[5],
          confirmedByActiveOwner
        })
      }

      setThreshold(nextThreshold)
      setTransactions(nextTransactions)
    } catch (e: any) {
      setAccessChecked(true)
      setError(e?.message || "Failed to read treasury state")
    } finally {
      setLoading(false)
    }
  }

  useEffect(() => {
    refresh()
  }, [address, treasury])

  async function submitProposal(target: `0x${string}`, value: bigint, data: `0x${string}`) {
    if (!treasury || !isAddress(treasury)) {
      throw new Error("Treasury address missing.")
    }
    if (!activeOwner) {
      throw new Error("Connected wallet is not an active treasury owner.")
    }

    const hash = await writeContractAsync({
      address: treasury,
      abi: TREASURY_ABI,
      functionName: "submitTransaction",
      args: [target, value, data]
    })

    await tempoPublicClient.waitForTransactionReceipt({ hash })
    await refresh()
  }

  async function callTreasury(
    functionName: "confirmTransaction" | "revokeConfirmation" | "executeTransaction",
    txIndex: number
  ) {
    if (!treasury || !isAddress(treasury)) {
      throw new Error("Treasury address missing.")
    }
    if (!activeOwner) {
      throw new Error("Connected wallet is not an active treasury owner.")
    }

    const hash = await writeContractAsync({
      address: treasury,
      abi: TREASURY_ABI,
      functionName,
      args: [BigInt(txIndex)]
    })

    await tempoPublicClient.waitForTransactionReceipt({ hash })
    await refresh()
  }

  async function submitCustomTransaction() {
    if (!isAddress(customTarget as `0x${string}`)) {
      throw new Error("Custom target must be a valid address.")
    }
    if (!customCalldata.startsWith("0x")) {
      throw new Error("Custom calldata must be hex-prefixed.")
    }

    await submitProposal(
      customTarget as `0x${string}`,
      parseEther(customValue || "0"),
      customCalldata as `0x${string}`
    )
  }

  async function submitFeeProposal() {
    if (!selectedPoolAddress || !isAddress(selectedPoolAddress as `0x${string}`)) {
      throw new Error("Pool address missing in frontend config.")
    }

    const data = encodeFunctionData({
      abi: POOL_ABI,
      functionName: "setGlobalFeeBps",
      args: [parseBps(feeBps)]
    })

    await submitProposal(selectedPoolAddress as `0x${string}`, 0n, data)
  }

  async function submitAddSignerProposal() {
    if (!treasury || !isAddress(treasury)) throw new Error("Treasury address missing.")
    if (!isAddress(newSigner as `0x${string}`)) throw new Error("Signer must be a valid address.")

    const data = encodeFunctionData({
      abi: TREASURY_ABI,
      functionName: "addSigner",
      args: [newSigner as `0x${string}`]
    })

    await submitProposal(treasury, 0n, data)
  }

  async function submitRemoveSignerProposal() {
    if (!treasury || !isAddress(treasury)) throw new Error("Treasury address missing.")
    if (!isAddress(removeSignerAddress as `0x${string}`)) {
      throw new Error("Signer to remove must be a valid address.")
    }

    const data = encodeFunctionData({
      abi: TREASURY_ABI,
      functionName: "removeSigner",
      args: [removeSignerAddress as `0x${string}`]
    })

    await submitProposal(treasury, 0n, data)
  }

  async function submitReplaceSignerProposal() {
    if (!treasury || !isAddress(treasury)) throw new Error("Treasury address missing.")
    if (!isAddress(replaceOldSigner as `0x${string}`) || !isAddress(replaceNewSigner as `0x${string}`)) {
      throw new Error("Old and new signer addresses must be valid.")
    }

    const data = encodeFunctionData({
      abi: TREASURY_ABI,
      functionName: "replaceSigner",
      args: [replaceOldSigner as `0x${string}`, replaceNewSigner as `0x${string}`]
    })

    await submitProposal(treasury, 0n, data)
  }

  async function submitSweepProposal() {
    if (!treasury || !isAddress(treasury)) throw new Error("Treasury address missing.")
    if (!isAddress(sweepToken as `0x${string}`) || !isAddress(sweepTo as `0x${string}`)) {
      throw new Error("Sweep token and recipient must be valid addresses.")
    }

    const data = encodeFunctionData({
      abi: TREASURY_ABI,
      functionName: "sweepRevenue",
      args: [sweepToken as `0x${string}`, sweepTo as `0x${string}`, parseBaseUnits(sweepAmount || "0")]
    })

    await submitProposal(treasury, 0n, data)
  }

  async function submitCancelProposal() {
    if (!treasury || !isAddress(treasury)) throw new Error("Treasury address missing.")

    const data = encodeFunctionData({
      abi: TREASURY_ABI,
      functionName: "cancelTransaction",
      args: [BigInt(cancelProposalIndex)]
    })

    await submitProposal(treasury, 0n, data)
  }

  return (
    <div className="rounded-3xl border border-white/10 bg-slate-950/80 p-6 shadow-2xl shadow-cyan-950/10">
      <div className="flex flex-col gap-3 xl:flex-row xl:items-end xl:justify-between">
        <div>
          <p className="text-xs uppercase tracking-[0.25em] text-cyan-300">Hidden admin vault</p>
          <h2 className="mt-3 text-3xl font-black text-white">Protocol treasury multisig surface</h2>
          <p className="mt-3 max-w-3xl text-sm text-slate-400">
            This route exposes only real treasury powers. Wallet-only functions are routed through
            treasury proposals instead of being faked as direct signer actions.
          </p>
        </div>

        <button
          onClick={refresh}
          className="rounded-full border border-white/10 px-4 py-3 text-sm font-semibold text-slate-200 transition hover:border-white/30 hover:text-white"
        >
          Refresh vault state
        </button>
      </div>

      {(error || loading) && (
        <div className={`mt-6 rounded-2xl p-4 text-sm ${error ? "border border-red-500/30 bg-red-500/10 text-red-100" : "border border-cyan-400/20 bg-cyan-500/10 text-cyan-100"}`}>
          {error || "Loading treasury state..."}
        </div>
      )}

      {accessChecked && !loading && !error && !activeOwner && (
        <div className="mt-6 rounded-2xl border border-amber-400/20 bg-amber-500/10 p-4 text-sm text-amber-50">
          This hidden signer-only route is available only to active treasury owners.
          Connect an authorized signer wallet to continue. Treasury state and proposal data are
          not exposed to non-signers.
        </div>
      )}

      {activeOwner && (
      <>
      <div className="mt-6 grid gap-4 xl:grid-cols-3">
        <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-4">
          <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">Owners</div>
          <div className="mt-3 space-y-2 text-sm text-slate-200">
            {owners.map((owner) => (
              <div key={owner} className="break-all">{owner}</div>
            ))}
          </div>
        </div>

        <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-4">
          <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">Threshold</div>
          <div className="mt-3 text-3xl font-black text-white">{threshold.toString()}</div>
        </div>

        <div className="rounded-2xl border border-amber-400/20 bg-amber-500/10 p-4 text-sm text-amber-50">
          <div className="text-[11px] font-semibold uppercase tracking-[0.18em] text-amber-200">
            Operator warning
          </div>
          <div className="mt-3 space-y-2">
            <div>Signer removals and replacements are wallet-only treasury calls routed via proposals.</div>
            <div>Removing signers while proposals are pending will revert on chain.</div>
            <div>Threshold and owner safety must be reviewed before execution to avoid vault bricking.</div>
          </div>
        </div>
      </div>

      <div className="mt-6 grid gap-4 xl:grid-cols-2">
        <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-4">
          <div className="text-sm font-semibold text-white">Submit arbitrary treasury transaction</div>
          <div className="mt-4 space-y-3">
            <input value={customTarget} onChange={(e) => setCustomTarget(e.target.value)} placeholder="Target address" className="w-full rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white" />
            <input value={customValue} onChange={(e) => setCustomValue(e.target.value)} placeholder="Value in native token" className="w-full rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white" />
            <textarea value={customCalldata} onChange={(e) => setCustomCalldata(e.target.value)} placeholder="0x..." rows={4} className="w-full rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white" />
            <button onClick={() => submitCustomTransaction().catch((e) => setError(e.message))} className="rounded-full bg-white px-4 py-3 text-sm font-semibold text-slate-950">
              Submit custom proposal
            </button>
          </div>
        </div>

        <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-4">
          <div className="text-sm font-semibold text-white">Fee update proposal</div>
          <div className="mt-4 grid gap-3 md:grid-cols-2">
            <select value={selectedPool} onChange={(e) => setSelectedPool(e.target.value as "USDC" | "USDT")} className="rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white">
              <option value="USDC">USDC pool</option>
              <option value="USDT">USDT pool</option>
            </select>
            <input value={feeBps} onChange={(e) => setFeeBps(e.target.value)} placeholder="Fee bps" className="rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white" />
          </div>
          <button onClick={() => submitFeeProposal().catch((e) => setError(e.message))} className="mt-4 rounded-full bg-cyan-400 px-4 py-3 text-sm font-semibold text-slate-950">
            Queue fee update
          </button>
        </div>

        <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-4">
          <div className="text-sm font-semibold text-white">Signer-management proposals</div>
          <div className="mt-4 space-y-3">
            <input value={newSigner} onChange={(e) => setNewSigner(e.target.value)} placeholder="Add signer address" className="w-full rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white" />
            <button onClick={() => submitAddSignerProposal().catch((e) => setError(e.message))} className="rounded-full border border-white/10 px-4 py-3 text-sm font-semibold text-slate-200">
              Queue add signer
            </button>

            <input value={removeSignerAddress} onChange={(e) => setRemoveSignerAddress(e.target.value)} placeholder="Remove signer address" className="w-full rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white" />
            <button onClick={() => submitRemoveSignerProposal().catch((e) => setError(e.message))} className="rounded-full border border-white/10 px-4 py-3 text-sm font-semibold text-slate-200">
              Queue remove signer
            </button>

            <div className="grid gap-3 md:grid-cols-2">
              <input value={replaceOldSigner} onChange={(e) => setReplaceOldSigner(e.target.value)} placeholder="Old signer" className="rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white" />
              <input value={replaceNewSigner} onChange={(e) => setReplaceNewSigner(e.target.value)} placeholder="New signer" className="rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white" />
            </div>
            <button onClick={() => submitReplaceSignerProposal().catch((e) => setError(e.message))} className="rounded-full border border-white/10 px-4 py-3 text-sm font-semibold text-slate-200">
              Queue replace signer
            </button>
          </div>
        </div>

        <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-4">
          <div className="text-sm font-semibold text-white">Revenue sweep and cancel proposals</div>
          <div className="mt-4 space-y-3">
            <input value={sweepToken} onChange={(e) => setSweepToken(e.target.value)} placeholder="ERC20 token address" className="w-full rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white" />
            <input value={sweepTo} onChange={(e) => setSweepTo(e.target.value)} placeholder="Sweep recipient" className="w-full rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white" />
            <input value={sweepAmount} onChange={(e) => setSweepAmount(e.target.value)} placeholder="Amount in integer token base units" className="w-full rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white" />
            <button onClick={() => submitSweepProposal().catch((e) => setError(e.message))} className="rounded-full border border-white/10 px-4 py-3 text-sm font-semibold text-slate-200">
              Queue revenue sweep
            </button>

            <input value={cancelProposalIndex} onChange={(e) => setCancelProposalIndex(e.target.value)} placeholder="Proposal index to cancel via wallet-only path" className="w-full rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white" />
            <button onClick={() => submitCancelProposal().catch((e) => setError(e.message))} className="rounded-full border border-white/10 px-4 py-3 text-sm font-semibold text-slate-200">
              Queue cancel proposal
            </button>
          </div>
        </div>
      </div>

      <div className="mt-8 rounded-2xl border border-white/10 bg-white/[0.03] p-4">
        <div>
          <div className="text-sm font-semibold text-white">Recent proposals</div>
          <div className="mt-1 text-sm text-slate-400">
            Active owner: {activeOwner ? "yes" : "no"} {address ? `(${address})` : ""}
          </div>
        </div>

        <div className="mt-4 space-y-3">
          {transactions.map((tx) => (
            <div key={tx.txIndex} className="rounded-2xl border border-white/10 bg-slate-950/70 p-4">
              <div className="flex flex-col gap-3 xl:flex-row xl:items-start xl:justify-between">
                <div className="space-y-2 text-sm text-slate-300">
                  <div className="text-[11px] uppercase tracking-[0.18em] text-slate-500">
                    Proposal #{tx.txIndex}
                  </div>
                  <div className="break-all">Target: {tx.target}</div>
                  <div>Value: {formatUnits(tx.value, 18)} native</div>
                  <div className="break-all">Calldata: {tx.data}</div>
                  <div>
                    Status: {tx.executed ? "executed" : tx.cancelled ? "cancelled" : "pending"} with{" "}
                    {tx.numConfirmations.toString()} / {threshold.toString()} confirmations
                  </div>
                </div>

                <div className="flex flex-wrap gap-2">
                  <button
                    disabled={!activeOwner || tx.executed || tx.cancelled || tx.confirmedByActiveOwner}
                    onClick={() => callTreasury("confirmTransaction", tx.txIndex).catch((e) => setError(e.message))}
                    className="rounded-full border border-white/10 px-4 py-2 text-sm font-semibold text-slate-200 disabled:opacity-40"
                  >
                    Confirm
                  </button>
                  <button
                    disabled={!activeOwner || tx.executed || tx.cancelled || !tx.confirmedByActiveOwner}
                    onClick={() => callTreasury("revokeConfirmation", tx.txIndex).catch((e) => setError(e.message))}
                    className="rounded-full border border-white/10 px-4 py-2 text-sm font-semibold text-slate-200 disabled:opacity-40"
                  >
                    Revoke
                  </button>
                  <button
                    disabled={!activeOwner || tx.executed || tx.cancelled || tx.numConfirmations < threshold}
                    onClick={() => callTreasury("executeTransaction", tx.txIndex).catch((e) => setError(e.message))}
                    className="rounded-full bg-cyan-400 px-4 py-2 text-sm font-semibold text-slate-950 disabled:opacity-40"
                  >
                    Execute
                  </button>
                </div>
              </div>
            </div>
          ))}

          {!transactions.length && !loading && (
            <div className="rounded-2xl border border-white/10 bg-slate-950/70 p-4 text-sm text-slate-400">
              No proposals indexed from the treasury yet.
            </div>
          )}
        </div>
      </div>
      </>
      )}
    </div>
  )
}
```

## 6.9 File: `apps/web/src/app/providers.tsx`

```tsx
"use client"

import type { ReactNode } from "react"
import { useState } from "react"
import { QueryClient, QueryClientProvider } from "@tanstack/react-query"
import { WagmiProvider } from "wagmi"
import { ParlySessionProvider } from "../lib/parly-session"
import { wagmiConfig } from "../wagmi.config"

export default function Providers({ children }: { children: ReactNode }) {
  const [queryClient] = useState(() => new QueryClient())

  return (
    <WagmiProvider config={wagmiConfig}>
      <QueryClientProvider client={queryClient}>
        <ParlySessionProvider>{children}</ParlySessionProvider>
      </QueryClientProvider>
    </WagmiProvider>
  )
}
```

## 6.10 File: `apps/web/src/app/layout.tsx`

```tsx
import type { Metadata } from "next"
import type { ReactNode } from "react"
import { Space_Grotesk } from "next/font/google"
import SiteChrome from "../components/SiteChrome"
import Providers from "./providers"
import "./globals.css"

const spaceGrotesk = Space_Grotesk({
  subsets: ["latin"],
  variable: "--font-space-grotesk"
})

export const metadata: Metadata = {
  title: "Parly.fi | V16.9.9 Canonical Privacy Core",
  description: "Canonical private stablecoin settlement on Tempo."
}

export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <html lang="en" className={spaceGrotesk.variable}>
      <body className="min-h-screen bg-slate-950 text-white antialiased">
        <Providers>
          <SiteChrome>{children}</SiteChrome>
        </Providers>
      </body>
    </html>
  )
}
```

---

# 7. Circuit package and canonical proof model

## 7.1 Architectural reasoning

The circuit must solve four things cleanly:

1. exact public signal ordering,
2. fixed 10-slot output alignment,
3. relayer binding without breaking nullifier semantics,
4. deterministic change-note creation.

V16.9.9 keeps:

* `nullifierHash = Poseidon(secret, nullifier)`
* `relayer` as a separate public input
* `child_keys_out[10]` as public outputs
* `out_commitment` as the original note commitment output

### Canonical public signal map

Verifier-facing public signal order is:

* `[0..9]` = `child_keys_out[0..9]`
* `[10]` = `out_commitment`
* `[11]` = `root`
* `[12]` = `nullifierHash`
* `[13]` = `protocol_fee`
* `[14]` = `executor_fee`
* `[15]` = `total_input_amount`
* `[16]` = `asset_id`
* `[17]` = `local_eid`
* `[18..27]` = `recipients[0..9]`
* `[28..37]` = `amounts[0..9]`
* `[38..47]` = `dest_eids[0..9]`
* `[48]` = `new_change_commitment`
* `[49]` = `relayer`

## 7.2 Founder Note

The MPC ceremony is not paperwork.

Record at minimum:

* Phase 1 file hash,
* final `.zkey` hash,
* generated verifier hash,
* exact circuit commit,
* public signal count,
* export command transcript.

## 7.3 File: `packages/circuits/package.json`

```json
{
  "name": "@parly/circuits",
  "private": true,
  "scripts": {
    "build": "circom joinsplit.circom --r1cs --wasm --sym --inspect -o ."
  },
  "dependencies": {
    "circomlib": "^2.0.5",
    "circomlibjs": "0.1.7",
    "snarkjs": "0.7.4"
  }
}
```

## 7.4 File: `packages/circuits/joinsplit.circom`

```circom
pragma circom 2.1.6;

include "node_modules/circomlib/circuits/poseidon.circom";
include "node_modules/circomlib/circuits/bitify.circom";
include "node_modules/circomlib/circuits/comparators.circom";

template JoinSplit(levels, maxOutputs) {
    signal input root;
    signal input nullifierHash;
    signal input protocol_fee;
    signal input executor_fee;
    signal input total_input_amount;
    signal input asset_id;
    signal input local_eid;
    signal input recipients[maxOutputs];
    signal input amounts[maxOutputs];
    signal input dest_eids[maxOutputs];
    signal input new_change_commitment;
    signal input relayer;

    signal input secret;
    signal input nullifier;
    signal input depositor;
    signal input pathElements[levels];
    signal input pathIndices[levels];
    signal input new_change_secret;
    signal input new_change_nullifier;

    signal output child_keys_out[maxOutputs];
    signal output out_commitment;

    component relayerBits = Num2Bits(160);
    relayerBits.in <== relayer;

    component depositorBits = Num2Bits(160);
    depositorBits.in <== depositor;

    component assetBits = Num2Bits(32);
    assetBits.in <== asset_id;

    component localEidBits = Num2Bits(32);
    localEidBits.in <== local_eid;

    component inputInner = Poseidon(4);
    inputInner.inputs[0] <== secret;
    inputInner.inputs[1] <== nullifier;
    inputInner.inputs[2] <== depositor;
    inputInner.inputs[3] <== asset_id;

    component inputCommitmentHasher = Poseidon(2);
    inputCommitmentHasher.inputs[0] <== inputInner.out;
    inputCommitmentHasher.inputs[1] <== total_input_amount;

    signal commitment;
    commitment <== inputCommitmentHasher.out;
    out_commitment <== commitment;

    signal currentHash[levels + 1];
    component treeHashers[levels];

    currentHash[0] <== commitment;

    for (var i = 0; i < levels; i++) {
        pathIndices[i] * (1 - pathIndices[i]) === 0;

        treeHashers[i] = Poseidon(2);

        signal left_i;
        signal right_i;

        left_i <== currentHash[i] - pathIndices[i] * (currentHash[i] - pathElements[i]);
        right_i <== pathElements[i] - pathIndices[i] * (pathElements[i] - currentHash[i]);

        treeHashers[i].inputs[0] <== left_i;
        treeHashers[i].inputs[1] <== right_i;

        currentHash[i + 1] <== treeHashers[i].out;
    }

    root === currentHash[levels];

    component nullifierHasher = Poseidon(2);
    nullifierHasher.inputs[0] <== secret;
    nullifierHasher.inputs[1] <== nullifier;
    nullifierHash === nullifierHasher.out;

    signal sumOutputs[maxOutputs + 1];
    sumOutputs[0] <== 0;

    for (var j = 0; j < maxOutputs; j++) {
        component recipientBits = Num2Bits(160);
        recipientBits.in <== recipients[j];

        component amountBits = Num2Bits(128);
        amountBits.in <== amounts[j];

        component eidBits = Num2Bits(32);
        eidBits.in <== dest_eids[j];

        sumOutputs[j + 1] <== sumOutputs[j] + amounts[j];
    }

    signal total_spend;
    total_spend <== sumOutputs[maxOutputs] + protocol_fee + executor_fee;

    component protocolFeeBits = Num2Bits(128);
    protocolFeeBits.in <== protocol_fee;

    component executorFeeBits = Num2Bits(128);
    executorFeeBits.in <== executor_fee;

    component totalInputBits = Num2Bits(128);
    totalInputBits.in <== total_input_amount;

    component balanceCheck = LessEqThan(128);
    balanceCheck.in[0] <== total_spend;
    balanceCheck.in[1] <== total_input_amount;
    balanceCheck.out === 1;

    signal change_amount;
    change_amount <== total_input_amount - total_spend;

    component changeBits = Num2Bits(128);
    changeBits.in <== change_amount;

    component innerChange = Poseidon(4);
    innerChange.inputs[0] <== new_change_secret;
    innerChange.inputs[1] <== new_change_nullifier;
    innerChange.inputs[2] <== depositor;
    innerChange.inputs[3] <== asset_id;

    component changeHasher = Poseidon(2);
    changeHasher.inputs[0] <== innerChange.out;
    changeHasher.inputs[1] <== change_amount;

    component isZeroChange = IsZero();
    isZeroChange.in <== change_amount;

    (new_change_commitment - changeHasher.out) * (1 - isZeroChange.out) === 0;
    new_change_commitment * isZeroChange.out === 0;

    for (var k = 0; k < maxOutputs; k++) {
        component childKey = Poseidon(4);
        childKey.inputs[0] <== secret;
        childKey.inputs[1] <== recipients[k];
        childKey.inputs[2] <== amounts[k];
        childKey.inputs[3] <== dest_eids[k];
        child_keys_out[k] <== childKey.out;
    }
}

component main {
    public [
        root,
        nullifierHash,
        protocol_fee,
        executor_fee,
        total_input_amount,
        asset_id,
        local_eid,
        recipients,
        amounts,
        dest_eids,
        new_change_commitment,
        relayer
    ]
} = JoinSplit(32, 10);
```

## 7.5 Dev Note

Do not use a fake handwritten witness blob for launch-quality proof rehearsal.

Witness inputs must come from:

* actual note material,
* actual Merkle paths,
* actual relayer binding,
* actual output arrays.

---

# 8. Contracts workspace tooling

## 8.1 File: `packages/contracts/package.json`

```json
{
  "name": "@parly/contracts",
  "private": true,
  "scripts": {
    "build": "forge build",
    "test": "forge test -vvv",
    "fmt": "forge fmt"
  },
  "devDependencies": {
    "@layerzerolabs/lz-definitions": "^3.0.118",
    "@layerzerolabs/metadata-tools": "^0.1.0",
    "@layerzerolabs/toolbox-hardhat": "^0.3.0",
    "circomlibjs": "0.1.7",
    "dotenv": "16.4.7",
    "hardhat": "^2.26.3",
    "ts-node": "10.9.2",
    "typescript": "5.8.2",
    "viem": "2.43.5"
  }
}
```

## 8.2 File: `packages/contracts/foundry.toml`

```toml
[profile.default]
src = "src"
test = "test"
out = "out"
libs = ["lib"]
solc_version = "0.8.24"
evm_version = "cancun"
optimizer = true
optimizer_runs = 200
ffi = true
fs_permissions = [{ access = "read-write", path = "./" }]
remappings = [
  "@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/",
  "@layerzerolabs/lz-evm-protocol-v2/=lib/LayerZero-v2/packages/layerzero-v2/evm/protocol/",
  "@layerzerolabs/oapp-evm/=lib/devtools/packages/oapp-evm/",
  "@layerzerolabs/oft-evm/=lib/devtools/packages/oft-evm/"
]

[rpc_endpoints]
tempo = "${TEMPO_RPC_URL}"
ethereum = "${ETHEREUM_RPC_URL}"
arbitrum = "${ARBITRUM_RPC_URL}"
base = "${BASE_RPC_URL}"
bnb = "${BSC_RPC_URL}"
```

---

# 9. Core interfaces

## 9.1 File: `packages/contracts/src/interfaces/IParlyOFT.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface IParlyOFT {
    struct SendParam {
        uint32 dstEid;
        bytes32 to;
        uint256 amountLD;
        uint256 minAmountLD;
        bytes extraOptions;
        bytes composeMsg;
        bytes oftCmd;
    }

    struct MessagingFee {
        uint256 nativeFee;
        uint256 lzTokenFee;
    }

    struct MessagingReceipt {
        bytes32 guid;
        uint64 nonce;
        MessagingFee fee;
    }

    struct OFTReceipt {
        uint256 amountSentLD;
        uint256 amountReceivedLD;
    }

    function quoteSend(
        SendParam calldata sendParam,
        bool payInLzToken
    ) external view returns (MessagingFee memory);

    function send(
        SendParam calldata sendParam,
        MessagingFee calldata fee,
        address refundAddress
    ) external payable returns (MessagingReceipt memory, OFTReceipt memory);

    function token() external view returns (address);
}
```

## 9.2 File: `packages/contracts/src/interfaces/IZKVerifier.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface IZKVerifier {
    function verifyProof(
        uint[2] memory pA,
        uint[2][2] memory pB,
        uint[2] memory pC,
        uint[50] memory pubSignals
    ) external view returns (bool);
}
```

## 9.3 File: `packages/contracts/src/interfaces/IPoseidon.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface IPoseidon {
    function poseidon(uint256[2] memory inputs) external view returns (uint256);
}
```

## 9.4 File: `packages/contracts/src/interfaces/ITIP403Registry.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface ITIP403Registry {
    function isAuthorized(uint64 policyId, address account) external view returns (bool);
}
```

---

# 10. Protocol treasury

## 10.1 Architectural reasoning

Treasury must remain:

* simple,
* auditable,
* deterministic,
* independent of runtime execution.

No hidden operator shortcuts.
No relayer-managed treasury mutation.
No sticky deployer admin.

## 10.2 Founder Note

The deployer may deploy treasury infrastructure.
The deployer must not remain steady-state treasury authority.

## 10.3 File: `packages/contracts/src/ProtocolTreasury.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract ProtocolTreasury {
    using SafeERC20 for IERC20;

    address[] public owners;
    mapping(address => bool) public isOwner;
    uint256 public threshold;

    struct Transaction {
        address target;
        uint256 value;
        bytes data;
        bool executed;
        bool cancelled;
        uint256 numConfirmations;
    }

    Transaction[] public transactions;
    mapping(uint256 => mapping(address => bool)) public isConfirmed;

    event TransactionSubmitted(uint256 indexed txIndex, address indexed submitter, address indexed target);
    event TransactionConfirmed(uint256 indexed txIndex, address indexed owner);
    event TransactionRevoked(uint256 indexed txIndex, address indexed owner);
    event TransactionCancelled(uint256 indexed txIndex);
    event TransactionExecuted(uint256 indexed txIndex, address indexed executor);
    event SignerAdded(address indexed signer);
    event SignerRemoved(address indexed signer);
    event SignerReplaced(address indexed oldSigner, address indexed newSigner);

    modifier onlyOwner() {
        require(isOwner[msg.sender], "not owner");
        _;
    }

    modifier onlyWallet() {
        require(msg.sender == address(this), "wallet only");
        _;
    }

    constructor(address[] memory _owners, uint256 _threshold) {
        require(_owners.length >= _threshold, "bad threshold");
        require(_threshold > 1, "threshold too low");

        for (uint256 i = 0; i < _owners.length; i++) {
            require(_owners[i] != address(0), "zero owner");
            require(!isOwner[_owners[i]], "duplicate owner");
            isOwner[_owners[i]] = true;
            owners.push(_owners[i]);
        }

        threshold = _threshold;
    }

    function getOwners() external view returns (address[] memory) {
        return owners;
    }

    function getTransactionCount() external view returns (uint256) {
        return transactions.length;
    }

    function pendingTransactionCount() public view returns (uint256 count) {
        for (uint256 i = 0; i < transactions.length; i++) {
            if (!transactions[i].executed && !transactions[i].cancelled) {
                count += 1;
            }
        }
    }

    function submitTransaction(address target, uint256 value, bytes memory data) external onlyOwner {
        transactions.push(
            Transaction({
                target: target,
                value: value,
                data: data,
                executed: false,
                cancelled: false,
                numConfirmations: 0
            })
        );

        emit TransactionSubmitted(transactions.length - 1, msg.sender, target);
    }

    function confirmTransaction(uint256 txIndex) external onlyOwner {
        require(txIndex < transactions.length, "bad index");
        Transaction storage txn = transactions[txIndex];
        require(!txn.executed && !txn.cancelled, "inactive");
        require(!isConfirmed[txIndex][msg.sender], "already confirmed");

        isConfirmed[txIndex][msg.sender] = true;
        txn.numConfirmations += 1;

        emit TransactionConfirmed(txIndex, msg.sender);
    }

    function revokeConfirmation(uint256 txIndex) external onlyOwner {
        require(txIndex < transactions.length, "bad index");
        Transaction storage txn = transactions[txIndex];
        require(!txn.executed && !txn.cancelled, "inactive");
        require(isConfirmed[txIndex][msg.sender], "not confirmed");

        isConfirmed[txIndex][msg.sender] = false;
        txn.numConfirmations -= 1;

        emit TransactionRevoked(txIndex, msg.sender);
    }

    function cancelTransaction(uint256 txIndex) external onlyWallet {
        require(txIndex < transactions.length, "bad index");
        Transaction storage txn = transactions[txIndex];
        require(!txn.executed && !txn.cancelled, "inactive");
        txn.cancelled = true;
        emit TransactionCancelled(txIndex);
    }

    function executeTransaction(uint256 txIndex) external onlyOwner {
        require(txIndex < transactions.length, "bad index");
        Transaction storage txn = transactions[txIndex];
        require(!txn.executed && !txn.cancelled, "inactive");
        require(txn.numConfirmations >= threshold, "threshold not met");

        txn.executed = true;

        (bool success, bytes memory returndata) = txn.target.call{value: txn.value}(txn.data);
        if (!success) {
            txn.executed = false;
            assembly {
                revert(add(returndata, 32), mload(returndata))
            }
        }

        emit TransactionExecuted(txIndex, msg.sender);
    }

    function addSigner(address signer) external onlyWallet {
        require(pendingTransactionCount() == 0, "pending txs");
        require(signer != address(0), "zero signer");
        require(!isOwner[signer], "exists");

        isOwner[signer] = true;
        owners.push(signer);

        emit SignerAdded(signer);
    }

    function removeSigner(address signer) external onlyWallet {
        require(pendingTransactionCount() == 0, "pending txs");
        require(isOwner[signer], "missing");
        require(owners.length > 2, "min owners");
        require(owners.length - 1 >= threshold, "below threshold");

        isOwner[signer] = false;

        for (uint256 i = 0; i < owners.length; i++) {
            if (owners[i] == signer) {
                owners[i] = owners[owners.length - 1];
                owners.pop();
                break;
            }
        }

        emit SignerRemoved(signer);
    }

    function replaceSigner(address oldSigner, address newSigner) external onlyWallet {
        require(pendingTransactionCount() == 0, "pending txs");
        require(isOwner[oldSigner], "old missing");
        require(newSigner != address(0), "zero new");
        require(!isOwner[newSigner], "new exists");

        for (uint256 i = 0; i < owners.length; i++) {
            if (owners[i] == oldSigner) {
                owners[i] = newSigner;
                break;
            }
        }

        isOwner[oldSigner] = false;
        isOwner[newSigner] = true;

        emit SignerReplaced(oldSigner, newSigner);
    }

    function sweepRevenue(address token, address to, uint256 amount) external onlyWallet {
        IERC20(token).safeTransfer(to, amount);
    }

    receive() external payable {}
}
```

---

# 11. Core hub pool

## 11.1 Architectural reasoning

This is the core launch contract.

It must do all of these correctly:

* validate proof against exactly 50 public signals,
* bind execution to the selected relayer,
* reject unknown roots,
* reject spent nullifiers,
* reject calldata/proof desync,
* enforce padded-slot discipline,
* insert change notes correctly,
* support same-chain payouts,
* support cross-chain stablecoin payouts,
* keep hub payout fee mode coherent,
* keep the Merkle capacity guard safe.

### Critical V16.9.9 corrections

V16.9.9 fixes the remaining core pool defects from V16.8:

* removes the fatal undeclared `childKeys` fragment,
* upgrades `nextLeafIndex` to `uint64`,
* fixes the tree-capacity guard,
* keeps hub fee mode Alt-fee-only,
* keeps event shape aligned with indexer expectations.

## 11.2 Dev Note

Do not reintroduce dual native/token fee logic into the hub payout path.
Do not revert `nextLeafIndex` back to `uint32`.
Do not simplify padded-slot checks.

## 11.3 Founder Note

The pool is where:

* private-state correctness,
* execution binding,
* policy enforcement,
* and treasury fee collection

meet each other.

That is why this contract is intentionally strict.

## 11.4 File: `packages/contracts/src/TempoShieldedPool.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

import "./interfaces/IZKVerifier.sol";
import "./interfaces/IPoseidon.sol";
import "./interfaces/ITIP403Registry.sol";
import "./interfaces/IParlyOFT.sol";

contract TempoShieldedPool is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    IERC20 public immutable underlyingToken;
    IERC20 public immutable lzFeeToken;
    IZKVerifier public immutable verifier;
    IPoseidon public immutable poseidonHasher;

    uint256 public immutable assetId;
    uint32 public immutable localEid;
    uint64 public immutable kytPolicyId;

    address public immutable protocolTreasury;
    address public ingressComposer;
    address public hubOft;

    address public constant TEMPO_KYT_PRECOMPILE = 0x403c000000000000000000000000000000000000;

    uint256 public globalFeeBps = 50;
    uint256 public constant MAX_FEE_BPS = 100;

    uint32 public constant TREE_DEPTH = 32;
    uint64 public nextLeafIndex = 0;

    bytes32[32] public filledSubtrees;
    bytes32[32] public zeros;
    bytes32 public currentRoot;

    mapping(bytes32 => bool) public knownRoots;
    mapping(bytes32 => bool) public nullifierHashes;
    mapping(bytes32 => bool) public knownCommitments;
    mapping(uint32 => bool) public supportedDstEids;

    event Deposit(address indexed depositor, bytes32 indexed commitment, uint256 amount, uint32 leafIndex);
    event LeafInserted(bytes32 indexed commitment, uint32 indexed leafIndex, uint256 indexed assetId, uint8 kind);
    event NoteEnvelope(bytes32 indexed commitment, uint8 indexed kind, uint256 indexed assetId, bytes envelope);
    event BatchWithdrawal(
        bytes32 indexed nullifierHash,
        address indexed executor,
        uint256 indexed assetId,
        uint256 protocolFee,
        uint256 executorFee,
        bytes32[] childKeys
    );
    event HubOftSet(address indexed hubOft);
    event IngressComposerSet(address indexed composer);
    event SupportedDstEidSet(uint32 indexed eid, bool supported);
    event GlobalFeeUpdated(uint256 oldFeeBps, uint256 newFeeBps);

    modifier onlyIngressComposer() {
        require(msg.sender == ingressComposer, "not ingress");
        _;
    }

    constructor(
        address _underlyingToken,
        address _lzFeeToken,
        address _verifier,
        address _poseidon,
        uint32 _localEid,
        uint256 _assetId,
        uint64 _kytPolicyId,
        address _treasury,
        address _owner
    ) Ownable(_owner) {
        require(_owner != address(0), "bad owner");
        require(_underlyingToken != address(0), "bad token");
        require(_lzFeeToken != address(0), "bad fee token");
        require(_verifier != address(0), "bad verifier");
        require(_poseidon != address(0), "bad poseidon");
        require(_treasury != address(0), "bad treasury");

        underlyingToken = IERC20(_underlyingToken);
        lzFeeToken = IERC20(_lzFeeToken);
        verifier = IZKVerifier(_verifier);
        poseidonHasher = IPoseidon(_poseidon);

        localEid = _localEid;
        assetId = _assetId;
        kytPolicyId = _kytPolicyId;
        protocolTreasury = _treasury;

        _initializeZerosAndRoot();
    }

    function setIngressComposer(address composer) external onlyOwner {
        require(composer != address(0), "bad composer");
        ingressComposer = composer;
        emit IngressComposerSet(composer);
    }

    function setHubOft(address oft) external onlyOwner {
        require(oft != address(0), "bad oft");
        hubOft = oft;
        emit HubOftSet(oft);
    }

    function setSupportedDstEid(uint32 eid, bool supported) external onlyOwner {
        supportedDstEids[eid] = supported;
        emit SupportedDstEidSet(eid, supported);
    }

    function setGlobalFeeBps(uint256 newFeeBps) external onlyOwner {
        require(newFeeBps <= MAX_FEE_BPS, "fee too high");
        uint256 old = globalFeeBps;
        globalFeeBps = newFeeBps;
        emit GlobalFeeUpdated(old, newFeeBps);
    }

    function deposit(
        uint256 innerCommitment,
        uint256 principalAmount,
        uint256 depositorField,
        bytes calldata envelope
    ) external nonReentrant {
        require(principalAmount > 0, "zero amount");
        require(envelope.length > 0 && envelope.length <= 4096, "bad envelope");

        address actualDepositor = address(uint160(depositorField));
        require(uint256(uint160(msg.sender)) == depositorField, "depositor mismatch");
        require(
            ITIP403Registry(TEMPO_KYT_PRECOMPILE).isAuthorized(kytPolicyId, actualDepositor),
            "TIP403 blocked"
        );

        bytes32 finalCommitment = _finalCommitment(innerCommitment, principalAmount);
        require(!knownCommitments[finalCommitment], "commitment exists");

        knownCommitments[finalCommitment] = true;
        underlyingToken.safeTransferFrom(msg.sender, address(this), principalAmount);

        uint32 idx = _insert(finalCommitment, 0);
        emit Deposit(actualDepositor, finalCommitment, principalAmount, idx);
        emit NoteEnvelope(finalCommitment, 0, assetId, envelope);
    }

    function depositFromIngress(
        uint256 innerCommitment,
        uint256 principalAmount,
        uint256 depositorField,
        bytes calldata envelope
    ) external onlyIngressComposer nonReentrant returns (bytes32 finalCommitment, uint32 leafIndex) {
        require(principalAmount > 0, "zero amount");
        require(envelope.length > 0 && envelope.length <= 4096, "bad envelope");

        address actualDepositor = address(uint160(depositorField));
        require(
            ITIP403Registry(TEMPO_KYT_PRECOMPILE).isAuthorized(kytPolicyId, actualDepositor),
            "TIP403 blocked"
        );

        finalCommitment = _finalCommitment(innerCommitment, principalAmount);
        require(!knownCommitments[finalCommitment], "commitment exists");

        knownCommitments[finalCommitment] = true;
        underlyingToken.safeTransferFrom(msg.sender, address(this), principalAmount);

        leafIndex = _insert(finalCommitment, 0);
        emit Deposit(actualDepositor, finalCommitment, principalAmount, leafIndex);
        emit NoteEnvelope(finalCommitment, 0, assetId, envelope);
    }

    function quoteCrossChainFee(
        uint32 dstEid,
        address recipient,
        uint256 amount,
        bytes calldata options
    ) external view returns (uint256) {
        require(hubOft != address(0), "oft missing");
        require(dstEid != localEid, "local path");
        require(supportedDstEids[dstEid], "unsupported");
        require(recipient != address(0), "bad recipient");

        IParlyOFT.SendParam memory p = IParlyOFT.SendParam({
            dstEid: dstEid,
            to: bytes32(uint256(uint160(recipient))),
            amountLD: amount,
            minAmountLD: amount,
            extraOptions: options,
            composeMsg: bytes(""),
            oftCmd: bytes("")
        });

        IParlyOFT.MessagingFee memory fee = IParlyOFT(hubOft).quoteSend(p, true);
        require(fee.lzTokenFee > 0, "alt fee unavailable");
        return fee.lzTokenFee;
    }

    function batchWithdraw(
        uint[2] calldata pA,
        uint[2][2] calldata pB,
        uint[2] calldata pC,
        uint[50] calldata pubSignals,
        address[] calldata recipients,
        uint256[] calldata amounts,
        uint32[] calldata destEids,
        bytes32 newChangeCommitment,
        bytes calldata newChangeEnvelope,
        bytes[] calldata lzOptions
    ) external nonReentrant {
        require(recipients.length == 10, "bad recipients");
        require(amounts.length == 10, "bad amounts");
        require(destEids.length == 10, "bad eids");
        require(lzOptions.length == 10, "bad options");

        require(pubSignals[16] == assetId, "asset mismatch");
        require(pubSignals[17] == localEid, "local eid mismatch");

        address proofRelayer = address(uint160(pubSignals[49]));
        require(msg.sender == proofRelayer, "relayer mismatch");

        bytes32 activeNullifier = bytes32(pubSignals[12]);
        require(!nullifierHashes[activeNullifier], "spent");
        require(knownRoots[bytes32(pubSignals[11])], "unknown root");

        uint256 expectedProtocolFee = 0;
        uint256 totalLzFeeNeeded = 0;
        IParlyOFT.MessagingFee[10] memory fees;

        for (uint256 i = 0; i < 10; i++) {
            require(uint256(uint160(recipients[i])) == pubSignals[18 + i], "recipient mismatch");
            require(amounts[i] == pubSignals[28 + i], "amount mismatch");
            require(uint256(destEids[i]) == pubSignals[38 + i], "eid mismatch");

            if (amounts[i] == 0) {
                require(recipients[i] == address(0), "nonzero padded recipient");
                require(destEids[i] == localEid, "bad padded eid");
                require(lzOptions[i].length == 0, "bad padded options");
                continue;
            }

            require(recipients[i] != address(0), "zero recipient");

            expectedProtocolFee += (amounts[i] * globalFeeBps) / 10_000;

            if (destEids[i] != localEid) {
                require(supportedDstEids[destEids[i]], "unsupported dst");
                require(hubOft != address(0), "oft missing");

                IParlyOFT.SendParam memory q = IParlyOFT.SendParam({
                    dstEid: destEids[i],
                    to: bytes32(uint256(uint160(recipients[i]))),
                    amountLD: amounts[i],
                    minAmountLD: amounts[i],
                    extraOptions: lzOptions[i],
                    composeMsg: bytes(""),
                    oftCmd: bytes("")
                });

                fees[i] = IParlyOFT(hubOft).quoteSend(q, true);
                require(fees[i].lzTokenFee > 0, "zero alt fee quote");
                totalLzFeeNeeded += fees[i].lzTokenFee;
            }
        }

        require(pubSignals[13] == expectedProtocolFee, "protocol fee mismatch");
        require(verifier.verifyProof(pA, pB, pC, pubSignals), "invalid proof");

        nullifierHashes[activeNullifier] = true;

        if (totalLzFeeNeeded > 0) {
            if (address(lzFeeToken) == address(underlyingToken)) {
                underlyingToken.safeTransferFrom(msg.sender, address(this), totalLzFeeNeeded);
            } else {
                lzFeeToken.safeTransferFrom(msg.sender, address(this), totalLzFeeNeeded);
            }
        }

        if (uint256(newChangeCommitment) != 0) {
            require(uint256(newChangeCommitment) == pubSignals[48], "change mismatch");
            require(!knownCommitments[newChangeCommitment], "change exists");
            require(newChangeEnvelope.length > 0 && newChangeEnvelope.length <= 4096, "bad change envelope");

            knownCommitments[newChangeCommitment] = true;
            _insert(newChangeCommitment, 1);
            emit NoteEnvelope(newChangeCommitment, 1, assetId, newChangeEnvelope);
        } else {
            require(pubSignals[48] == 0, "zero change mismatch");
            require(newChangeEnvelope.length == 0, "unexpected zero-change envelope");
        }

        bytes32[] memory childKeys = new bytes32[](10);
        for (uint256 i = 0; i < 10; i++) {
            childKeys[i] = bytes32(pubSignals[i]);
        }

        for (uint256 i = 0; i < 10; i++) {
            if (amounts[i] == 0) continue;

            if (destEids[i] == localEid) {
                underlyingToken.safeTransfer(recipients[i], amounts[i]);
            } else {
                uint256 feeAmount = fees[i].lzTokenFee;

                IParlyOFT.SendParam memory p = IParlyOFT.SendParam({
                    dstEid: destEids[i],
                    to: bytes32(uint256(uint160(recipients[i]))),
                    amountLD: amounts[i],
                    minAmountLD: amounts[i],
                    extraOptions: lzOptions[i],
                    composeMsg: bytes(""),
                    oftCmd: bytes("")
                });

                if (address(lzFeeToken) == address(underlyingToken)) {
                    underlyingToken.forceApprove(hubOft, amounts[i] + feeAmount);
                    IParlyOFT(hubOft).send(p, fees[i], msg.sender);
                    underlyingToken.forceApprove(hubOft, 0);
                } else {
                    underlyingToken.forceApprove(hubOft, amounts[i]);
                    lzFeeToken.forceApprove(hubOft, feeAmount);
                    IParlyOFT(hubOft).send(p, fees[i], msg.sender);
                    underlyingToken.forceApprove(hubOft, 0);
                    lzFeeToken.forceApprove(hubOft, 0);
                }
            }
        }

        uint256 protocolFee = pubSignals[13];
        uint256 executorFee = pubSignals[14];

        if (protocolFee > 0) {
            underlyingToken.safeTransfer(protocolTreasury, protocolFee);
        }

        if (executorFee > 0) {
            underlyingToken.safeTransfer(msg.sender, executorFee);
        }

        emit BatchWithdrawal(activeNullifier, msg.sender, assetId, protocolFee, executorFee, childKeys);
    }

    function _initializeZerosAndRoot() internal {
        zeros[0] = _hashPair(bytes32(0), bytes32(0));
        filledSubtrees[0] = zeros[0];

        for (uint256 i = 1; i < TREE_DEPTH; i++) {
            zeros[i] = _hashPair(zeros[i - 1], zeros[i - 1]);
            filledSubtrees[i] = zeros[i];
        }

        currentRoot = zeros[TREE_DEPTH - 1];
        knownRoots[currentRoot] = true;
    }

    function _finalCommitment(
        uint256 innerCommitment,
        uint256 principalAmount
    ) internal view returns (bytes32) {
        uint256[2] memory inputs;
        inputs[0] = innerCommitment;
        inputs[1] = principalAmount;
        return bytes32(poseidonHasher.poseidon(inputs));
    }

    function _hashPair(bytes32 left, bytes32 right) internal view returns (bytes32) {
        uint256[2] memory inputs;
        inputs[0] = uint256(left);
        inputs[1] = uint256(right);
        return bytes32(poseidonHasher.poseidon(inputs));
    }

    function _insert(bytes32 leaf, uint8 kind) internal returns (uint32 insertedIndex) {
        require(nextLeafIndex < (uint64(1) << TREE_DEPTH), "tree full");

        insertedIndex = uint32(nextLeafIndex);
        bytes32 currentLevelHash = leaf;
        uint64 currentIndex = nextLeafIndex;

        for (uint32 i = 0; i < TREE_DEPTH; i++) {
            bytes32 left;
            bytes32 right;

            if (currentIndex % 2 == 0) {
                left = currentLevelHash;
                right = zeros[i];
                filledSubtrees[i] = currentLevelHash;
            } else {
                left = filledSubtrees[i];
                right = currentLevelHash;
            }

            currentLevelHash = _hashPair(left, right);
            currentIndex /= 2;
        }

        currentRoot = currentLevelHash;
        knownRoots[currentRoot] = true;
        nextLeafIndex += 1;

        emit LeafInserted(leaf, insertedIndex, assetId, kind);
    }
}
```

---

# 12. End of Part 1

Part 2 continues immediately below.

---

# PARLY.FI MASTER TECHNICAL BLUEPRINT

## Final Specification V16.9.9 - Canonical Zero-Trust Engineering Runbook

### Part 2 - Hub Composer, Spoke Gateway, Verifier Import Path, Deployment Scripts, Wiring, Ownership Transfer, Corrected Contract Tests, Canonical CLI Order

---

## 13. Superseding corrections from V16.8

## 13.1 Architectural reasoning

V16.9.9 corrects the remaining contract-layer and deployment-layer issues that still existed in V16.8:

* fixes malformed Solidity snippets,
* fixes the unsafe tree-capacity guard,
* fixes the treasury deployer compile blocker,
* fixes the pool compile blocker,
* restores event/indexer ABI alignment,
* keeps deployment order coherent,
* keeps ownership transfer explicit,
* keeps tests compile-clean.

## 13.2 Dev Note

Treat this part as authoritative over older deployment, test, and wiring sections from earlier branches.

---

## 14. Hub composer

## 14.1 Architectural reasoning

The hub composer has one narrow responsibility:

* accept trusted OFT compose delivery,
* verify source provenance,
* decode app payload,
* push the credited amount into the correct pool,
* and emit the deterministic Tempo settlement anchor that lets the indexer join source dispatch and hub settlement without guesswork.

It does **not** handle:

* proof validation,
* nullifier logic,
* payout logic,
* protocol fee accounting.

## 14.2 Dev Note

Whitelist the **spoke gateway** as the trusted compose sender per source EID.

Do not whitelist the spoke OFT in this topology.

The canonical spoke-deposit correlation key is the LayerZero message `guid`, not a timestamp or amount match.

## 14.3 File: `packages/contracts/src/ParlyHubComposer.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import { IOAppComposer } from "@layerzerolabs/oapp-evm/contracts/oapp/interfaces/IOAppComposer.sol";
import { OFTComposeMsgCodec } from "@layerzerolabs/oft-evm/contracts/libs/OFTComposeMsgCodec.sol";
import "./interfaces/IParlyOFT.sol";

interface IShieldedIngressPool {
    function depositFromIngress(
        uint256 innerCommitment,
        uint256 principalAmount,
        uint256 depositorField,
        bytes calldata envelope
    ) external returns (bytes32 finalCommitment, uint32 leafIndex);

    function underlyingToken() external view returns (address);
}

contract ParlyHubComposer is IOAppComposer, Ownable {
    using SafeERC20 for IERC20;

    enum DeferredIngressReason {
        NONE,
        BAD_ASSET,
        BAD_DEPOSITOR,
        ZERO_AMOUNT,
        WRONG_LOCAL_OFT,
        UNAPPROVED_GATEWAY,
        POOL_MISSING,
        TOKEN_MISSING,
        DEPOSIT_REVERTED
    }

    struct DeferredIngress {
        bool exists;
        uint16 assetId;
        address token;
        address depositor;
        uint256 amountLD;
        uint256 innerCommitment;
        bytes envelope;
        uint32 srcEid;
        bytes32 composeFrom;
        address localOft;
        DeferredIngressReason reason;
        bytes errorData;
    }

    address public immutable endpoint;

    mapping(uint16 => address) public localOfts;
    mapping(uint16 => address) public pools;
    mapping(uint16 => address) public underlyingTokens;
    mapping(uint32 => mapping(bytes32 => bool)) public approvedGateways;
    mapping(bytes32 => bool) public ingressSettled;
    mapping(bytes32 => bool) public ingressDeposited;
    mapping(bytes32 => DeferredIngress) internal deferredIngresses;

    event LocalOftSet(uint16 indexed assetId, address indexed oft);
    event PoolSet(uint16 indexed assetId, address indexed pool);
    event UnderlyingTokenSet(uint16 indexed assetId, address indexed token);
    event ApprovedGatewaySet(uint32 indexed srcEid, bytes32 indexed gateway, bool approved);
    event IngressCompleted(
        bytes32 indexed guid,
        bytes32 indexed commitment,
        uint32 indexed srcEid,
        uint16 assetId,
        address depositor,
        uint256 amountLD,
        uint256 innerCommitment,
        uint32 settlementLeafIndex,
        bytes32 composeFrom
    );
    event IngressDeferred(
        bytes32 indexed guid,
        uint16 indexed assetId,
        address indexed depositor,
        address token,
        uint256 amountLD,
        DeferredIngressReason reason
    );
    event DeferredIngressRetried(bytes32 indexed guid, bool success, DeferredIngressReason reason);
    event DeferredIngressClaimed(bytes32 indexed guid, address indexed depositor, uint256 amountLD);

    constructor(address _endpoint, address _owner) Ownable(_owner) {
        require(_endpoint != address(0), "bad endpoint");
        require(_owner != address(0), "bad owner");
        endpoint = _endpoint;
    }

    function setLocalOft(uint16 assetId, address oft) external onlyOwner {
        require(assetId == 1 || assetId == 2, "bad asset");
        require(oft != address(0), "bad oft");
        localOfts[assetId] = oft;
        emit LocalOftSet(assetId, oft);
    }

    function setPool(uint16 assetId, address pool) external onlyOwner {
        require(assetId == 1 || assetId == 2, "bad asset");
        require(pool != address(0), "bad pool");
        pools[assetId] = pool;
        address token = IShieldedIngressPool(pool).underlyingToken();
        require(token != address(0), "bad token");
        underlyingTokens[assetId] = token;
        emit PoolSet(assetId, pool);
        emit UnderlyingTokenSet(assetId, token);
    }

    function setApprovedGateway(uint32 srcEid, bytes32 gateway, bool approved) external onlyOwner {
        require(gateway != bytes32(0), "bad gateway");
        approvedGateways[srcEid][gateway] = approved;
        emit ApprovedGatewaySet(srcEid, gateway, approved);
    }

    function revokeGateway(uint32 srcEid, bytes32 gateway) external onlyOwner {
        approvedGateways[srcEid][gateway] = false;
        emit ApprovedGatewaySet(srcEid, gateway, false);
    }

    function getDeferredIngress(bytes32 guid) external view returns (DeferredIngress memory) {
        return deferredIngresses[guid];
    }

    function retryDeferredIngress(bytes32 guid) external returns (bool) {
        DeferredIngress storage record = deferredIngresses[guid];
        require(record.exists, "missing ingress");
        require(msg.sender == owner() || msg.sender == record.depositor, "not allowed");
        require(!ingressSettled[guid], "ingress settled");

        (
            bool ok,
            DeferredIngressReason reason,
            bytes memory errorData
        ) = _attemptIngress(
                guid,
                record.assetId,
                record.localOft,
                record.srcEid,
                record.composeFrom,
                record.depositor,
                record.amountLD,
                record.innerCommitment,
                record.envelope
            );

        if (!ok) {
            _recordDeferredIngress(
                guid,
                record.assetId,
                record.depositor,
                record.amountLD,
                record.innerCommitment,
                record.envelope,
                record.srcEid,
                record.composeFrom,
                record.localOft,
                reason,
                errorData
            );
            emit DeferredIngressRetried(guid, false, reason);
            return false;
        }

        emit DeferredIngressRetried(guid, true, DeferredIngressReason.NONE);
        return true;
    }

    function claimDeferredIngress(bytes32 guid) external {
        DeferredIngress storage record = deferredIngresses[guid];
        require(record.exists, "missing ingress");
        require(!ingressSettled[guid], "ingress settled");
        require(record.token != address(0), "token missing");
        require(msg.sender == owner() || msg.sender == record.depositor, "not allowed");

        // Claim is the principal-only refund path. It returns the already credited token back to
        // the original depositor and intentionally does not re-run TIP-403 because it is not a
        // new pool ingress.
        ingressSettled[guid] = true;
        delete deferredIngresses[guid];

        IERC20(record.token).safeTransfer(record.depositor, record.amountLD);
        emit DeferredIngressClaimed(guid, record.depositor, record.amountLD);
    }

    function lzCompose(
        address _oApp,
        bytes32 guid,
        bytes calldata _message,
        address,
        bytes calldata
    ) external payable override {
        require(msg.sender == endpoint, "not endpoint");
        if (ingressSettled[guid]) {
            return;
        }

        uint32 srcEid = OFTComposeMsgCodec.srcEid(_message);
        uint256 amountLD = OFTComposeMsgCodec.amountLD(_message);
        bytes32 composeFrom = OFTComposeMsgCodec.composeFrom(_message);
        bytes memory composeMsg = OFTComposeMsgCodec.composeMsg(_message);

        (uint16 assetId, address originalSender, uint256 innerCommitment, bytes memory envelope) =
            abi.decode(composeMsg, (uint16, address, uint256, bytes));

        (
            bool ok,
            DeferredIngressReason reason,
            bytes memory errorData
        ) = _attemptIngress(
                guid,
                assetId,
                _oApp,
                srcEid,
                composeFrom,
                originalSender,
                amountLD,
                innerCommitment,
                envelope
            );

        if (!ok) {
            _recordDeferredIngress(
                guid,
                assetId,
                originalSender,
                amountLD,
                innerCommitment,
                envelope,
                srcEid,
                composeFrom,
                _oApp,
                reason,
                errorData
            );
        }
    }

    function _attemptIngress(
        bytes32 guid,
        uint16 assetId,
        address localOft,
        uint32 srcEid,
        bytes32 composeFrom,
        address depositor,
        uint256 amountLD,
        uint256 innerCommitment,
        bytes memory envelope
    ) internal returns (bool ok, DeferredIngressReason reason, bytes memory errorData) {
        if (assetId != 1 && assetId != 2) {
            return (false, DeferredIngressReason.BAD_ASSET, bytes("bad asset"));
        }
        if (depositor == address(0)) {
            return (false, DeferredIngressReason.BAD_DEPOSITOR, bytes("bad depositor"));
        }
        if (amountLD == 0) {
            return (false, DeferredIngressReason.ZERO_AMOUNT, bytes("zero amount"));
        }
        if (localOft != localOfts[assetId]) {
            return (false, DeferredIngressReason.WRONG_LOCAL_OFT, bytes("wrong local oft"));
        }
        if (!approvedGateways[srcEid][composeFrom]) {
            return (false, DeferredIngressReason.UNAPPROVED_GATEWAY, bytes("unapproved gateway"));
        }

        address pool = pools[assetId];
        if (pool == address(0)) {
            return (false, DeferredIngressReason.POOL_MISSING, bytes("pool missing"));
        }

        address tokenAddress = _resolveCreditedToken(assetId, localOft);
        if (tokenAddress == address(0)) {
            return (false, DeferredIngressReason.TOKEN_MISSING, bytes("token missing"));
        }

        IERC20 token = IERC20(tokenAddress);
        token.forceApprove(pool, amountLD);

        bytes32 finalCommitment;
        uint32 settlementLeafIndex;

        try IShieldedIngressPool(pool).depositFromIngress(
            innerCommitment,
            amountLD,
            uint256(uint160(depositor)),
            envelope
        ) returns (bytes32 commitment, uint32 leafIndex) {
            ok = true;
            finalCommitment = commitment;
            settlementLeafIndex = leafIndex;
        } catch (bytes memory caught) {
            ok = false;
            errorData = caught;
        }

        token.forceApprove(pool, 0);

        if (!ok) {
            return (false, DeferredIngressReason.DEPOSIT_REVERTED, errorData);
        }

        ingressSettled[guid] = true;
        ingressDeposited[guid] = true;
        delete deferredIngresses[guid];

        emit IngressCompleted(
            guid,
            finalCommitment,
            srcEid,
            assetId,
            depositor,
            amountLD,
            innerCommitment,
            settlementLeafIndex,
            composeFrom
        );
        return (true, DeferredIngressReason.NONE, bytes(""));
    }

    function _recordDeferredIngress(
        bytes32 guid,
        uint16 assetId,
        address depositor,
        uint256 amountLD,
        uint256 innerCommitment,
        bytes memory envelope,
        uint32 srcEid,
        bytes32 composeFrom,
        address localOft,
        DeferredIngressReason reason,
        bytes memory errorData
    ) internal {
        // Refundability must track the actually credited Tempo-side token, even when the compose
        // payload is malformed. Prefer configured asset wiring when available, then fall back to
        // the credited local OFT adapter itself.
        address token = _resolveCreditedToken(assetId, localOft);

        deferredIngresses[guid] = DeferredIngress({
            exists: true,
            assetId: assetId,
            token: token,
            depositor: depositor,
            amountLD: amountLD,
            innerCommitment: innerCommitment,
            envelope: envelope,
            srcEid: srcEid,
            composeFrom: composeFrom,
            localOft: localOft,
            reason: reason,
            errorData: errorData
        });

        emit IngressDeferred(guid, assetId, depositor, token, amountLD, reason);
    }

    function _resolveCreditedToken(uint16 assetId, address localOft) internal view returns (address) {
        address configured = underlyingTokens[assetId];
        if (configured != address(0)) {
            return configured;
        }
        if (localOft == address(0)) {
            return address(0);
        }

        try IParlyOFT(localOft).token() returns (address token) {
            return token;
        } catch {
            return address(0);
        }
    }
}
```

---

## 15. Spoke gateway

## 15.1 Architectural reasoning

The spoke gateway stays intentionally small.

Its job is only to:

* collect the stablecoin,
* construct the compose payload,
* quote messaging fee coherently,
* route the shield request to Tempo,
* and emit the real source-chain dispatch event that the indexer can join to Tempo settlement by `guid`.

It is not a general spoke protocol surface.

## 15.2 Dev Note

V16.9.9 still keeps two spoke shield modes:

* native-fee mode when `lzFeeToken == address(0)`,
* alt-fee mode when `lzFeeToken != address(0)`.

Quote and send must use the same `payInLzToken` decision.

## 15.3 File: `packages/contracts/src/ParlySpokeGateway.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import { OptionsBuilder } from "@layerzerolabs/oapp-evm/contracts/oapp/libs/OptionsBuilder.sol";

import "./interfaces/IParlyOFT.sol";

contract ParlySpokeGateway is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;
    using OptionsBuilder for bytes;

    IERC20 public immutable underlyingToken;
    address public immutable lzFeeToken;
    address public immutable oft;

    uint16 public immutable assetId;
    uint32 public immutable tempoHubEid;
    bytes32 public immutable hubComposerBytes32;

    uint128 public lzReceiveGas;
    uint128 public lzComposeGas;
    uint128 public lzComposeValue;

    event ShieldedCrossChain(
        address indexed sender,
        uint16 indexed assetId,
        bytes32 indexed guid,
        uint256 amount,
        uint256 innerCommitment
    );
    event ExecutionOptionsSet(uint128 lzReceiveGas, uint128 lzComposeGas, uint128 lzComposeValue);

    constructor(
        address _underlyingToken,
        address _lzFeeToken,
        address _oft,
        uint16 _assetId,
        uint32 _tempoHubEid,
        bytes32 _hubComposerBytes32,
        uint128 _lzReceiveGas,
        uint128 _lzComposeGas,
        uint128 _lzComposeValue,
        address _owner
    ) Ownable(_owner) {
        require(_underlyingToken != address(0), "bad token");
        require(_oft != address(0), "bad oft");
        require(_hubComposerBytes32 != bytes32(0), "bad composer");
        require(_owner != address(0), "bad owner");
        require(_assetId == 1 || _assetId == 2, "bad asset");
        require(_lzReceiveGas > 0, "bad receive gas");
        require(_lzComposeGas > 0, "bad compose gas");

        underlyingToken = IERC20(_underlyingToken);
        lzFeeToken = _lzFeeToken;
        oft = _oft;
        assetId = _assetId;
        tempoHubEid = _tempoHubEid;
        hubComposerBytes32 = _hubComposerBytes32;
        lzReceiveGas = _lzReceiveGas;
        lzComposeGas = _lzComposeGas;
        lzComposeValue = _lzComposeValue;
    }

    function setExecutionOptions(
        uint128 _lzReceiveGas,
        uint128 _lzComposeGas,
        uint128 _lzComposeValue
    ) external onlyOwner {
        require(_lzReceiveGas > 0, "bad receive gas");
        require(_lzComposeGas > 0, "bad compose gas");
        lzReceiveGas = _lzReceiveGas;
        lzComposeGas = _lzComposeGas;
        lzComposeValue = _lzComposeValue;
        emit ExecutionOptionsSet(_lzReceiveGas, _lzComposeGas, _lzComposeValue);
    }

    function previewExecutionOptions() external view returns (bytes memory) {
        return _buildOptions();
    }

    function quoteShieldFee(
        uint256 innerCommitment,
        uint256 amount,
        address originalSender,
        bytes calldata envelope
    ) external view returns (uint256 feeAmount, bool usesAltFeeToken) {
        require(innerCommitment != 0, "bad commitment");
        require(amount > 0, "zero amount");
        require(originalSender != address(0), "bad sender");
        require(envelope.length > 0 && envelope.length <= 4096, "bad envelope");

        bool payInLzToken = lzFeeToken != address(0);
        bytes memory composeMsg = abi.encode(assetId, originalSender, innerCommitment, envelope);
        bytes memory options = _buildOptions();

        IParlyOFT.SendParam memory p = IParlyOFT.SendParam({
            dstEid: tempoHubEid,
            to: hubComposerBytes32,
            amountLD: amount,
            minAmountLD: amount,
            extraOptions: options,
            composeMsg: composeMsg,
            oftCmd: bytes("")
        });

        IParlyOFT.MessagingFee memory fee = IParlyOFT(oft).quoteSend(p, payInLzToken);

        if (payInLzToken) {
            require(fee.lzTokenFee > 0, "zero alt fee");
            feeAmount = fee.lzTokenFee;
        } else {
            feeAmount = fee.nativeFee;
        }

        usesAltFeeToken = payInLzToken;
    }

    function shieldCrossChain(
        uint256 innerCommitment,
        uint256 amount,
        bytes calldata envelope
    ) external payable nonReentrant {
        require(innerCommitment != 0, "bad commitment");
        require(amount > 0, "zero amount");
        require(envelope.length > 0 && envelope.length <= 4096, "bad envelope");

        bool payInLzToken = lzFeeToken != address(0);
        bytes memory composeMsg = abi.encode(assetId, msg.sender, innerCommitment, envelope);
        bytes memory options = _buildOptions();
        IParlyOFT.MessagingReceipt memory receipt;
        IParlyOFT.OFTReceipt memory oftReceipt;

        IParlyOFT.SendParam memory p = IParlyOFT.SendParam({
            dstEid: tempoHubEid,
            to: hubComposerBytes32,
            amountLD: amount,
            minAmountLD: amount,
            extraOptions: options,
            composeMsg: composeMsg,
            oftCmd: bytes("")
        });

        IParlyOFT.MessagingFee memory fee = IParlyOFT(oft).quoteSend(p, payInLzToken);

        if (!payInLzToken) {
            require(msg.value == fee.nativeFee, "bad native fee");

            underlyingToken.safeTransferFrom(msg.sender, address(this), amount);
            underlyingToken.forceApprove(oft, amount);

            (receipt, oftReceipt) = IParlyOFT(oft).send{ value: fee.nativeFee }(p, fee, msg.sender);

            underlyingToken.forceApprove(oft, 0);
        } else if (lzFeeToken == address(underlyingToken)) {
            require(msg.value == 0, "msg.value forbidden");
            require(fee.lzTokenFee > 0, "zero alt fee");

            uint256 totalPull = amount + fee.lzTokenFee;
            underlyingToken.safeTransferFrom(msg.sender, address(this), totalPull);
            underlyingToken.forceApprove(oft, totalPull);

            (receipt, oftReceipt) = IParlyOFT(oft).send(p, fee, msg.sender);

            underlyingToken.forceApprove(oft, 0);
        } else {
            require(msg.value == 0, "msg.value forbidden");
            require(fee.lzTokenFee > 0, "zero alt fee");

            underlyingToken.safeTransferFrom(msg.sender, address(this), amount);
            IERC20(lzFeeToken).safeTransferFrom(msg.sender, address(this), fee.lzTokenFee);

            underlyingToken.forceApprove(oft, amount);
            IERC20(lzFeeToken).forceApprove(oft, fee.lzTokenFee);

            (receipt, oftReceipt) = IParlyOFT(oft).send(p, fee, msg.sender);

            underlyingToken.forceApprove(oft, 0);
            IERC20(lzFeeToken).forceApprove(oft, 0);
        }

        // Silence unused-variable warnings while keeping both OFT return values explicit for junior readers.
        oftReceipt;
        emit ShieldedCrossChain(msg.sender, assetId, receipt.guid, amount, innerCommitment);
    }

    function _buildOptions() internal view returns (bytes memory options) {
        options = OptionsBuilder.newOptions();
        options = options.addExecutorLzReceiveOption(lzReceiveGas, 0);
        options = options.addExecutorLzComposeOption(0, lzComposeGas, lzComposeValue);
    }
}
```

---

## 16. Generated verifier import path

## 16.1 Architectural reasoning

The verifier must come from the actual final `.zkey`.

That means the engineering order is:

* export verifier from final ceremony artifact,
* copy that generated verifier into contracts workspace,
* deploy that exact generated verifier.

## 16.2 Dev Note

Do not hand-edit the generated verifier.

## 16.3 File: `packages/contracts/script/CopyGeneratedVerifier.js`

```js
const fs = require("fs");
const path = require("path");
require("dotenv").config({ path: path.resolve(__dirname, "../../../.env") });

function main() {
  const src = process.env.GENERATED_VERIFIER_SOURCE || "../../circuits/ZKVerifier.sol";
  const from = path.resolve(__dirname, src);
  const to = path.resolve(__dirname, "../src/generated/ZKVerifier.sol");

  if (!fs.existsSync(from)) {
    throw new Error(`Generated verifier not found: ${from}`);
  }

  fs.mkdirSync(path.dirname(to), { recursive: true });
  fs.copyFileSync(from, to);

  console.log(`Copied verifier: ${from} -> ${to}`);
}

try {
  main();
} catch (e) {
  console.error(e);
  process.exit(1);
}
```

## 16.4 File: `packages/contracts/script/DeployVerifier.s.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import { Script, console2 } from "forge-std/Script.sol";
import "../src/generated/ZKVerifier.sol";

contract DeployVerifier is Script {
    function run() external {
        uint256 pk = vm.envUint("DEPLOYER_PRIVATE_KEY");

        vm.startBroadcast(pk);
        Groth16Verifier verifier = new Groth16Verifier();
        vm.stopBroadcast();

        console2.log("GROTH16_VERIFIER_ADDRESS=", address(verifier));
    }
}
```

---

## 17. Poseidon deployer

## 17.1 Dev Note

Deploy Poseidon from the actual target chain environment.

## 17.2 File: `packages/contracts/script/DeployPoseidon.js`

```js
require("dotenv").config({ path: require("path").resolve(__dirname, "../../../.env") });

const { createWalletClient, http, publicActions } = require("viem");
const { privateKeyToAccount } = require("viem/accounts");
const { defineChain } = require("viem/chains");
const { poseidonContract } = require("circomlibjs");

function requirePositiveChainId(name) {
  const value = Number(process.env[name] ?? "");
  if (!Number.isFinite(value) || !Number.isInteger(value) || value <= 0) {
    throw new Error(`${name} invalid`);
  }
  return value;
}

async function main() {
  const rpcUrl = process.env.TEMPO_RPC_URL;
  const chainId = requirePositiveChainId("TEMPO_CHAIN_ID");

  if (!rpcUrl) throw new Error("TEMPO_RPC_URL missing");
  if (!process.env.DEPLOYER_PRIVATE_KEY) throw new Error("DEPLOYER_PRIVATE_KEY missing");

  const account = privateKeyToAccount(process.env.DEPLOYER_PRIVATE_KEY);

  const chain = defineChain({
    id: chainId,
    name: "Tempo L1",
    nativeCurrency: { name: "TMP", symbol: "TMP", decimals: 18 },
    rpcUrls: { default: { http: [rpcUrl] } }
  });

  const client = createWalletClient({
    account,
    chain,
    transport: http(rpcUrl)
  }).extend(publicActions);

  const abi = poseidonContract.generateABI(2);
  const bytecode = `0x${poseidonContract.createCode(2)}`;

  const hash = await client.deployContract({
    abi,
    bytecode
  });

  const receipt = await client.waitForTransactionReceipt({ hash });
  console.log("POSEIDON_HASHER_ADDRESS=", receipt.contractAddress);
}

main().catch((e) => {
  console.error(e);
  process.exit(1);
});
```

---

## 18. Treasury deployer

## 18.1 Founder Note

Treasury deployment is signer-custody setup, not a casual dev step.

## 18.2 File: `packages/contracts/script/DeployTreasury.s.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import { Script, console2 } from "forge-std/Script.sol";
import "../src/ProtocolTreasury.sol";

contract DeployTreasury is Script {
    function run() external {
        uint256 pk = vm.envUint("DEPLOYER_PRIVATE_KEY");

        address[] memory owners = new address[](3);
        owners[0] = vm.envAddress("MULTISIG_OWNER_1");
        owners[1] = vm.envAddress("MULTISIG_OWNER_2");
        owners[2] = vm.envAddress("MULTISIG_OWNER_3");

        vm.startBroadcast(pk);
        ProtocolTreasury treasury = new ProtocolTreasury(owners, 2);
        vm.stopBroadcast();

        console2.log("PROTOCOL_TREASURY_ADDRESS=", address(treasury));
    }
}
```

---

## 19. Hub core deployer

## 19.1 Architectural reasoning

The hub deployer should only:

* deploy the two pools,
* deploy the composer,
* wire the composer into the pools.

It should not guess OFT addresses or hidden config.

## 19.2 File: `packages/contracts/script/DeployHubCore.s.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import { Script, console2 } from "forge-std/Script.sol";
import "../src/TempoShieldedPool.sol";
import "../src/ParlyHubComposer.sol";

contract DeployHubCore is Script {
    function run() external {
        uint256 pk = vm.envUint("DEPLOYER_PRIVATE_KEY");
        address owner = vm.addr(pk);
        address treasury = vm.envAddress("PROTOCOL_TREASURY_ADDRESS");
        uint32 localEid = uint32(vm.envUint("TEMPO_LZ_EID"));

        vm.startBroadcast(pk);

        TempoShieldedPool usdcPool = new TempoShieldedPool(
            vm.envAddress("TEMPO_USDC_ADDRESS"),
            vm.envAddress("TEMPO_LZ_FEE_TOKEN"),
            vm.envAddress("GROTH16_VERIFIER_ADDRESS"),
            vm.envAddress("POSEIDON_HASHER_ADDRESS"),
            localEid,
            1,
            uint64(vm.envUint("TEMPO_USDC_KYT_POLICY_ID")),
            treasury,
            owner
        );

        TempoShieldedPool usdtPool = new TempoShieldedPool(
            vm.envAddress("TEMPO_USDT_ADDRESS"),
            vm.envAddress("TEMPO_LZ_FEE_TOKEN"),
            vm.envAddress("GROTH16_VERIFIER_ADDRESS"),
            vm.envAddress("POSEIDON_HASHER_ADDRESS"),
            localEid,
            2,
            uint64(vm.envUint("TEMPO_USDT_KYT_POLICY_ID")),
            treasury,
            owner
        );

        ParlyHubComposer composer = new ParlyHubComposer(
            vm.envAddress("LAYERZERO_ENDPOINT_TEMPO"),
            owner
        );

        composer.setPool(1, address(usdcPool));
        composer.setPool(2, address(usdtPool));

        usdcPool.setIngressComposer(address(composer));
        usdtPool.setIngressComposer(address(composer));

        vm.stopBroadcast();

        console2.log("NEXT_PUBLIC_USDC_POOL=", address(usdcPool));
        console2.log("NEXT_PUBLIC_USDT_POOL=", address(usdtPool));
        console2.log("NEXT_PUBLIC_HUB_COMPOSER=", address(composer));
    }
}
```

---

## 20. Spoke gateway deployer

## 20.1 Dev Note

Run this on the target spoke chain.

It selects the correct token, OFT, and fee token based on:

* active chain ID,
* `PARLY_ASSET`.

## 20.2 File: `packages/contracts/script/DeploySpokeGateway.s.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import { Script, console2 } from "forge-std/Script.sol";
import "../src/ParlySpokeGateway.sol";

contract DeploySpokeGateway is Script {
    function run() external {
        uint256 pk = vm.envUint("DEPLOYER_PRIVATE_KEY");
        string memory asset = vm.envString("PARLY_ASSET");
        bool isUsdc = keccak256(bytes(asset)) == keccak256(bytes("USDC"));

        address token;
        address oft;
        address feeToken;
        string memory outEnvKey;

        if (block.chainid == vm.envUint("ETHEREUM_CHAIN_ID")) {
            token = vm.envAddress(isUsdc ? "SPOKE_USDC_TOKEN_ETHEREUM" : "SPOKE_USDT_TOKEN_ETHEREUM");
            oft = vm.envAddress(isUsdc ? "SPOKE_USDC_OFT_ETHEREUM" : "SPOKE_USDT_OFT_ETHEREUM");
            feeToken = vm.envAddress("SPOKE_LZ_FEE_TOKEN_ETHEREUM");
            outEnvKey = isUsdc ? "NEXT_PUBLIC_SPOKE_USDC_GATEWAY_ETHEREUM" : "NEXT_PUBLIC_SPOKE_USDT_GATEWAY_ETHEREUM";
        } else if (block.chainid == vm.envUint("ARBITRUM_CHAIN_ID")) {
            token = vm.envAddress(isUsdc ? "SPOKE_USDC_TOKEN_ARBITRUM" : "SPOKE_USDT_TOKEN_ARBITRUM");
            oft = vm.envAddress(isUsdc ? "SPOKE_USDC_OFT_ARBITRUM" : "SPOKE_USDT_OFT_ARBITRUM");
            feeToken = vm.envAddress("SPOKE_LZ_FEE_TOKEN_ARBITRUM");
            outEnvKey = isUsdc ? "NEXT_PUBLIC_SPOKE_USDC_GATEWAY_ARBITRUM" : "NEXT_PUBLIC_SPOKE_USDT_GATEWAY_ARBITRUM";
        } else if (block.chainid == vm.envUint("BASE_CHAIN_ID")) {
            token = vm.envAddress(isUsdc ? "SPOKE_USDC_TOKEN_BASE" : "SPOKE_USDT_TOKEN_BASE");
            oft = vm.envAddress(isUsdc ? "SPOKE_USDC_OFT_BASE" : "SPOKE_USDT_OFT_BASE");
            feeToken = vm.envAddress("SPOKE_LZ_FEE_TOKEN_BASE");
            outEnvKey = isUsdc ? "NEXT_PUBLIC_SPOKE_USDC_GATEWAY_BASE" : "NEXT_PUBLIC_SPOKE_USDT_GATEWAY_BASE";
        } else if (block.chainid == vm.envUint("BSC_CHAIN_ID")) {
            token = vm.envAddress(isUsdc ? "SPOKE_USDC_TOKEN_BNB" : "SPOKE_USDT_TOKEN_BNB");
            oft = vm.envAddress(isUsdc ? "SPOKE_USDC_OFT_BNB" : "SPOKE_USDT_OFT_BNB");
            feeToken = vm.envAddress("SPOKE_LZ_FEE_TOKEN_BNB");
            outEnvKey = isUsdc ? "NEXT_PUBLIC_SPOKE_USDC_GATEWAY_BNB" : "NEXT_PUBLIC_SPOKE_USDT_GATEWAY_BNB";
        } else {
            revert("unsupported chain");
        }

        require(token != address(0), "token missing");
        require(oft != address(0), "oft missing");

        vm.startBroadcast(pk);

        ParlySpokeGateway gateway = new ParlySpokeGateway(
            token,
            feeToken,
            oft,
            isUsdc ? 1 : 2,
            uint32(vm.envUint("TEMPO_LZ_EID")),
            bytes32(uint256(uint160(vm.envAddress("NEXT_PUBLIC_HUB_COMPOSER")))),
            uint128(vm.envUint("SPOKE_TO_TEMPO_LZRECEIVE_GAS")),
            uint128(vm.envUint("SPOKE_TO_TEMPO_LZCOMPOSE_GAS")),
            uint128(vm.envUint("SPOKE_TO_TEMPO_LZCOMPOSE_VALUE")),
            vm.addr(pk)
        );

        vm.stopBroadcast();

        console2.log(outEnvKey);
        console2.log("=", address(gateway));
    }
}
```

---

## 21. Hub configuration script

## 21.1 Architectural reasoning

This is where the deployed hub becomes coherent:

* pools learn their hub OFTs,
* composer learns local OFTs,
* pools learn supported destination EIDs,
* composer learns approved source gateways.

## 21.2 File: `packages/contracts/script/ConfigureHub.s.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import { Script } from "forge-std/Script.sol";
import "../src/TempoShieldedPool.sol";
import "../src/ParlyHubComposer.sol";

contract ConfigureHub is Script {
    function run() external {
        uint256 pk = vm.envUint("DEPLOYER_PRIVATE_KEY");

        TempoShieldedPool usdcPool = TempoShieldedPool(vm.envAddress("NEXT_PUBLIC_USDC_POOL"));
        TempoShieldedPool usdtPool = TempoShieldedPool(vm.envAddress("NEXT_PUBLIC_USDT_POOL"));
        ParlyHubComposer composer = ParlyHubComposer(vm.envAddress("NEXT_PUBLIC_HUB_COMPOSER"));

        vm.startBroadcast(pk);

        composer.setLocalOft(1, vm.envAddress("HUB_USDC_OFT"));
        composer.setLocalOft(2, vm.envAddress("HUB_USDT_OFT"));

        usdcPool.setHubOft(vm.envAddress("HUB_USDC_OFT"));
        usdtPool.setHubOft(vm.envAddress("HUB_USDT_OFT"));

        uint32[4] memory eids = [
            uint32(vm.envUint("LZ_EID_ETHEREUM")),
            uint32(vm.envUint("LZ_EID_ARBITRUM")),
            uint32(vm.envUint("LZ_EID_BASE")),
            uint32(vm.envUint("LZ_EID_BNB"))
        ];

        bytes32[4] memory usdcGateways = [
            bytes32(uint256(uint160(vm.envAddress("NEXT_PUBLIC_SPOKE_USDC_GATEWAY_ETHEREUM")))),
            bytes32(uint256(uint160(vm.envAddress("NEXT_PUBLIC_SPOKE_USDC_GATEWAY_ARBITRUM")))),
            bytes32(uint256(uint160(vm.envAddress("NEXT_PUBLIC_SPOKE_USDC_GATEWAY_BASE")))),
            bytes32(uint256(uint160(vm.envAddress("NEXT_PUBLIC_SPOKE_USDC_GATEWAY_BNB"))))
        ];

        bytes32[4] memory usdtGateways = [
            bytes32(uint256(uint160(vm.envAddress("NEXT_PUBLIC_SPOKE_USDT_GATEWAY_ETHEREUM")))),
            bytes32(uint256(uint160(vm.envAddress("NEXT_PUBLIC_SPOKE_USDT_GATEWAY_ARBITRUM")))),
            bytes32(uint256(uint160(vm.envAddress("NEXT_PUBLIC_SPOKE_USDT_GATEWAY_BASE")))),
            bytes32(uint256(uint160(vm.envAddress("NEXT_PUBLIC_SPOKE_USDT_GATEWAY_BNB"))))
        ];

        for (uint256 i = 0; i < 4; i++) {
            usdcPool.setSupportedDstEid(eids[i], true);
            usdtPool.setSupportedDstEid(eids[i], true);

            composer.setApprovedGateway(eids[i], usdcGateways[i], true);
            composer.setApprovedGateway(eids[i], usdtGateways[i], true);
        }

        vm.stopBroadcast();
    }
}
```

---

## 22. Ownership transfer script

## 22.1 Founder Note

Do not leave deployer EOAs as production owners.

## 22.2 Dev Note

Run this only after:

* hub configuration,
* OFT wiring,
* smoke success under deployer ownership.

## 22.3 File: `packages/contracts/script/TransferTempoOwnership.s.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import { Script } from "forge-std/Script.sol";
import "../src/TempoShieldedPool.sol";
import "../src/ParlyHubComposer.sol";

contract TransferTempoOwnership is Script {
    function run() external {
        uint256 pk = vm.envUint("DEPLOYER_PRIVATE_KEY");
        address newOwner = vm.envAddress("PROTOCOL_ADMIN_ADDRESS");

        vm.startBroadcast(pk);

        TempoShieldedPool(vm.envAddress("NEXT_PUBLIC_USDC_POOL")).transferOwnership(newOwner);
        TempoShieldedPool(vm.envAddress("NEXT_PUBLIC_USDT_POOL")).transferOwnership(newOwner);
        ParlyHubComposer(vm.envAddress("NEXT_PUBLIC_HUB_COMPOSER")).transferOwnership(newOwner);

        vm.stopBroadcast();
    }
}
```

---

## 22.4 File: `packages/contracts/script/TransferSpokeGatewayOwnership.s.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import { Script } from "forge-std/Script.sol";
import "../src/ParlySpokeGateway.sol";

contract TransferSpokeGatewayOwnership is Script {
    function run() external {
        uint256 pk = vm.envUint("DEPLOYER_PRIVATE_KEY");
        address newOwner = vm.envAddress("PROTOCOL_ADMIN_ADDRESS");
        string memory asset = vm.envString("PARLY_ASSET");
        bool isUsdc = keccak256(bytes(asset)) == keccak256(bytes("USDC"));

        string memory key;

        if (block.chainid == vm.envUint("ETHEREUM_CHAIN_ID")) {
            key = isUsdc ? "NEXT_PUBLIC_SPOKE_USDC_GATEWAY_ETHEREUM" : "NEXT_PUBLIC_SPOKE_USDT_GATEWAY_ETHEREUM";
        } else if (block.chainid == vm.envUint("ARBITRUM_CHAIN_ID")) {
            key = isUsdc ? "NEXT_PUBLIC_SPOKE_USDC_GATEWAY_ARBITRUM" : "NEXT_PUBLIC_SPOKE_USDT_GATEWAY_ARBITRUM";
        } else if (block.chainid == vm.envUint("BASE_CHAIN_ID")) {
            key = isUsdc ? "NEXT_PUBLIC_SPOKE_USDC_GATEWAY_BASE" : "NEXT_PUBLIC_SPOKE_USDT_GATEWAY_BASE";
        } else if (block.chainid == vm.envUint("BSC_CHAIN_ID")) {
            key = isUsdc ? "NEXT_PUBLIC_SPOKE_USDC_GATEWAY_BNB" : "NEXT_PUBLIC_SPOKE_USDT_GATEWAY_BNB";
        } else {
            revert("unsupported chain");
        }

        address gateway = vm.envAddress(key);
        require(gateway != address(0), "gateway missing");

        vm.startBroadcast(pk);
        ParlySpokeGateway(gateway).transferOwnership(newOwner);
        vm.stopBroadcast();
    }
}
```

---

## 22.5 File: `packages/contracts/script/ResolveDeferredIngress.s.sol`

Founder Note:
If hub ownership has already moved to a multisig, do not pretend a deployer EOA script is still the canonical owner path. Use multisig calldata for owner-side retries, or let the original depositor claim directly when the desired outcome is refund rather than retry.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import { Script } from "forge-std/Script.sol";
import "../src/ParlyHubComposer.sol";

contract ResolveDeferredIngress is Script {
    function run() external {
        uint256 pk = vm.envUint("PARLY_CALLER_PRIVATE_KEY");
        address composerAddress = vm.envAddress("NEXT_PUBLIC_HUB_COMPOSER");
        bytes32 guid = vm.envBytes32("PARLY_DEFERRED_GUID");
        string memory action = vm.envString("PARLY_DEFERRED_ACTION");

        vm.startBroadcast(pk);

        if (keccak256(bytes(action)) == keccak256(bytes("retry"))) {
            ParlyHubComposer(composerAddress).retryDeferredIngress(guid);
        } else if (keccak256(bytes(action)) == keccak256(bytes("claim"))) {
            ParlyHubComposer(composerAddress).claimDeferredIngress(guid);
        } else {
            revert("bad deferred action");
        }

        vm.stopBroadcast();
    }
}
```

---

## 23. Hardhat config and LayerZero wiring config

## 23.1 Architectural reasoning

Foundry deploys contracts.
LayerZero tooling wires peers and pathways.

Do not merge those jobs.

## 23.2 File: `packages/contracts/hardhat.config.ts`

```ts
import * as dotenv from "dotenv"
import { HardhatUserConfig } from "hardhat/config"
import "@layerzerolabs/toolbox-hardhat"
import { requireEnv, requirePositiveChainId } from "@parly/env-utils"

dotenv.config({ path: "../../.env" })

const accounts =
  process.env.DEPLOYER_PRIVATE_KEY && process.env.DEPLOYER_PRIVATE_KEY.length > 2
    ? [process.env.DEPLOYER_PRIVATE_KEY]
    : []

const tempoRpcUrl = requireEnv("TEMPO_RPC_URL")
const ethereumRpcUrl = requireEnv("ETHEREUM_RPC_URL")
const arbitrumRpcUrl = requireEnv("ARBITRUM_RPC_URL")
const baseRpcUrl = requireEnv("BASE_RPC_URL")
const bscRpcUrl = requireEnv("BSC_RPC_URL")

const tempoChainId = requirePositiveChainId("TEMPO_CHAIN_ID")
const ethereumChainId = requirePositiveChainId("ETHEREUM_CHAIN_ID")
const arbitrumChainId = requirePositiveChainId("ARBITRUM_CHAIN_ID")
const baseChainId = requirePositiveChainId("BASE_CHAIN_ID")
const bscChainId = requirePositiveChainId("BSC_CHAIN_ID")

const config: HardhatUserConfig = {
  solidity: {
    version: "0.8.24",
    settings: {
      optimizer: { enabled: true, runs: 200 },
      evmVersion: "cancun"
    }
  },
  networks: {
    tempo: {
      url: tempoRpcUrl,
      chainId: tempoChainId,
      accounts
    },
    ethereum: {
      url: ethereumRpcUrl,
      chainId: ethereumChainId,
      accounts
    },
    arbitrum: {
      url: arbitrumRpcUrl,
      chainId: arbitrumChainId,
      accounts
    },
    base: {
      url: baseRpcUrl,
      chainId: baseChainId,
      accounts
    },
    bnb: {
      url: bscRpcUrl,
      chainId: bscChainId,
      accounts
    }
  }
}

export default config
```

## 23.3 File: `packages/contracts/layerzero.config.ts`

```ts
import * as dotenv from "dotenv"
import { requireEnv, requirePositiveEid } from "@parly/env-utils"

dotenv.config({ path: "../../.env" })

type OAppNode = {
  contractName: string
  eid: number
  address: string
}

type Connection = {
  from: OAppNode
  to: OAppNode
  config: {
    sendLibrary: string
    receiveLibrary: string
    requiredDVNs: string[]
    confirmations: number
  }
}

const dvns = [requireEnv("DVN_GOOGLE_CLOUD"), requireEnv("DVN_LAYERZERO_LABS")]

const tempoUsdc: OAppNode = {
  contractName: "HUB_USDC_OFT",
  eid: requirePositiveEid("TEMPO_LZ_EID"),
  address: requireEnv("HUB_USDC_OFT")
}

const tempoUsdt: OAppNode = {
  contractName: "HUB_USDT_OFT",
  eid: requirePositiveEid("TEMPO_LZ_EID"),
  address: requireEnv("HUB_USDT_OFT")
}

const ethUsdc: OAppNode = {
  contractName: "SPOKE_USDC_OFT_ETHEREUM",
  eid: requirePositiveEid("LZ_EID_ETHEREUM"),
  address: requireEnv("SPOKE_USDC_OFT_ETHEREUM")
}

const arbUsdc: OAppNode = {
  contractName: "SPOKE_USDC_OFT_ARBITRUM",
  eid: requirePositiveEid("LZ_EID_ARBITRUM"),
  address: requireEnv("SPOKE_USDC_OFT_ARBITRUM")
}

const baseUsdc: OAppNode = {
  contractName: "SPOKE_USDC_OFT_BASE",
  eid: requirePositiveEid("LZ_EID_BASE"),
  address: requireEnv("SPOKE_USDC_OFT_BASE")
}

const bnbUsdc: OAppNode = {
  contractName: "SPOKE_USDC_OFT_BNB",
  eid: requirePositiveEid("LZ_EID_BNB"),
  address: requireEnv("SPOKE_USDC_OFT_BNB")
}

const ethUsdt: OAppNode = {
  contractName: "SPOKE_USDT_OFT_ETHEREUM",
  eid: requirePositiveEid("LZ_EID_ETHEREUM"),
  address: requireEnv("SPOKE_USDT_OFT_ETHEREUM")
}

const arbUsdt: OAppNode = {
  contractName: "SPOKE_USDT_OFT_ARBITRUM",
  eid: requirePositiveEid("LZ_EID_ARBITRUM"),
  address: requireEnv("SPOKE_USDT_OFT_ARBITRUM")
}

const baseUsdt: OAppNode = {
  contractName: "SPOKE_USDT_OFT_BASE",
  eid: requirePositiveEid("LZ_EID_BASE"),
  address: requireEnv("SPOKE_USDT_OFT_BASE")
}

const bnbUsdt: OAppNode = {
  contractName: "SPOKE_USDT_OFT_BNB",
  eid: requirePositiveEid("LZ_EID_BNB"),
  address: requireEnv("SPOKE_USDT_OFT_BNB")
}

function mkConnection(
  from: OAppNode,
  to: OAppNode,
  sendLibrary: string,
  receiveLibrary: string
): Connection {
  return {
    from,
    to,
    config: {
      sendLibrary,
      receiveLibrary,
      requiredDVNs: dvns,
      confirmations: 15
    }
  }
}

const connections: Connection[] = [
  mkConnection(tempoUsdc, ethUsdc, requireEnv("LAYERZERO_SEND_LIB_TEMPO"), requireEnv("LAYERZERO_RECEIVE_LIB_ETHEREUM")),
  mkConnection(ethUsdc, tempoUsdc, requireEnv("LAYERZERO_SEND_LIB_ETHEREUM"), requireEnv("LAYERZERO_RECEIVE_LIB_TEMPO")),
  mkConnection(tempoUsdc, arbUsdc, requireEnv("LAYERZERO_SEND_LIB_TEMPO"), requireEnv("LAYERZERO_RECEIVE_LIB_ARBITRUM")),
  mkConnection(arbUsdc, tempoUsdc, requireEnv("LAYERZERO_SEND_LIB_ARBITRUM"), requireEnv("LAYERZERO_RECEIVE_LIB_TEMPO")),
  mkConnection(tempoUsdc, baseUsdc, requireEnv("LAYERZERO_SEND_LIB_TEMPO"), requireEnv("LAYERZERO_RECEIVE_LIB_BASE")),
  mkConnection(baseUsdc, tempoUsdc, requireEnv("LAYERZERO_SEND_LIB_BASE"), requireEnv("LAYERZERO_RECEIVE_LIB_TEMPO")),
  mkConnection(tempoUsdc, bnbUsdc, requireEnv("LAYERZERO_SEND_LIB_TEMPO"), requireEnv("LAYERZERO_RECEIVE_LIB_BNB")),
  mkConnection(bnbUsdc, tempoUsdc, requireEnv("LAYERZERO_SEND_LIB_BNB"), requireEnv("LAYERZERO_RECEIVE_LIB_TEMPO")),

  mkConnection(tempoUsdt, ethUsdt, requireEnv("LAYERZERO_SEND_LIB_TEMPO"), requireEnv("LAYERZERO_RECEIVE_LIB_ETHEREUM")),
  mkConnection(ethUsdt, tempoUsdt, requireEnv("LAYERZERO_SEND_LIB_ETHEREUM"), requireEnv("LAYERZERO_RECEIVE_LIB_TEMPO")),
  mkConnection(tempoUsdt, arbUsdt, requireEnv("LAYERZERO_SEND_LIB_TEMPO"), requireEnv("LAYERZERO_RECEIVE_LIB_ARBITRUM")),
  mkConnection(arbUsdt, tempoUsdt, requireEnv("LAYERZERO_SEND_LIB_ARBITRUM"), requireEnv("LAYERZERO_RECEIVE_LIB_TEMPO")),
  mkConnection(tempoUsdt, baseUsdt, requireEnv("LAYERZERO_SEND_LIB_TEMPO"), requireEnv("LAYERZERO_RECEIVE_LIB_BASE")),
  mkConnection(baseUsdt, tempoUsdt, requireEnv("LAYERZERO_SEND_LIB_BASE"), requireEnv("LAYERZERO_RECEIVE_LIB_TEMPO")),
  mkConnection(tempoUsdt, bnbUsdt, requireEnv("LAYERZERO_SEND_LIB_TEMPO"), requireEnv("LAYERZERO_RECEIVE_LIB_BNB")),
  mkConnection(bnbUsdt, tempoUsdt, requireEnv("LAYERZERO_SEND_LIB_BNB"), requireEnv("LAYERZERO_RECEIVE_LIB_TEMPO"))
]

export default {
  contracts: [
    tempoUsdc,
    tempoUsdt,
    ethUsdc,
    arbUsdc,
    baseUsdc,
    bnbUsdc,
    ethUsdt,
    arbUsdt,
    baseUsdt,
    bnbUsdt
  ],
  connections
}
```

---

## 24. Contract tests

## 24.1 Architectural reasoning

Minimum acceptable unit coverage must prove:

* deposit envelope bounds,
* relayer binding,
* padded-slot discipline,
* supported-destination behavior,
* fee-mode coherence on gateway quote paths,
* allowance cleanup at rest,
* deferred-ingress retry and claim behavior on the hub composer,
* the corrected tree index type does not break core behavior.

## 24.2 File: `packages/contracts/test/TempoShieldedPool.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "../src/TempoShieldedPool.sol";
import "../src/interfaces/IParlyOFT.sol";
import "../src/interfaces/IZKVerifier.sol";
import "../src/interfaces/IPoseidon.sol";
import "../src/interfaces/ITIP403Registry.sol";

contract MockERC20 is IERC20 {
    string public name = "Mock";
    string public symbol = "MOCK";
    uint8 public decimals = 6;
    uint256 public override totalSupply;

    mapping(address => uint256) public override balanceOf;
    mapping(address => mapping(address => uint256)) public override allowance;

    function transfer(address to, uint256 amount) external override returns (bool) {
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }

    function approve(address spender, uint256 amount) external override returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external override returns (bool) {
        uint256 allowed = allowance[from][msg.sender];
        if (allowed != type(uint256).max) {
            allowance[from][msg.sender] = allowed - amount;
        }
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        emit Transfer(from, to, amount);
        return true;
    }

    function mint(address to, uint256 amount) external {
        totalSupply += amount;
        balanceOf[to] += amount;
        emit Transfer(address(0), to, amount);
    }
}

contract MockVerifier is IZKVerifier {
    bool public result = true;

    function setResult(bool v) external {
        result = v;
    }

    function verifyProof(
        uint[2] memory,
        uint[2][2] memory,
        uint[2] memory,
        uint[50] memory
    ) external view returns (bool) {
        return result;
    }
}

contract MockPoseidon is IPoseidon {
    function poseidon(uint256[2] memory inputs) external pure returns (uint256) {
        return uint256(keccak256(abi.encode(inputs[0], inputs[1])));
    }
}

contract MockHubOFT is IParlyOFT {
    uint256 public feeAmount = 7;

    function quoteSend(
        SendParam calldata,
        bool payInLzToken
    ) external view returns (MessagingFee memory) {
        require(payInLzToken, "expected alt fee");
        return MessagingFee({ nativeFee: 0, lzTokenFee: feeAmount });
    }

    function send(
        SendParam calldata,
        MessagingFee calldata,
        address
    ) external payable returns (MessagingReceipt memory, OFTReceipt memory) {
        return (
            MessagingReceipt({
                guid: bytes32(uint256(1)),
                nonce: 1,
                fee: MessagingFee({ nativeFee: 0, lzTokenFee: 0 })
            }),
            OFTReceipt({ amountSentLD: 0, amountReceivedLD: 0 })
        );
    }
}

contract TempoShieldedPoolTest is Test {
    MockERC20 token;
    MockVerifier verifier;
    MockPoseidon poseidon;
    MockHubOFT hubOft;
    TempoShieldedPool pool;

    address user = address(0x1111);
    address relayer = address(0x2222);
    address treasury = address(0x3333);

    function setUp() public {
        token = new MockERC20();
        verifier = new MockVerifier();
        poseidon = new MockPoseidon();
        hubOft = new MockHubOFT();

        pool = new TempoShieldedPool(
            address(token),
            address(token),
            address(verifier),
            address(poseidon),
            40439,
            1,
            2,
            treasury,
            address(this)
        );

        pool.setHubOft(address(hubOft));
        pool.setSupportedDstEid(30101, true);

        vm.mockCall(
            0x403c000000000000000000000000000000000000,
            abi.encodeWithSelector(ITIP403Registry.isAuthorized.selector, uint64(2), user),
            abi.encode(true)
        );

        token.mint(user, 1_000_000_000);

        vm.prank(user);
        token.approve(address(pool), type(uint256).max);
    }

    function _blankArrays()
        internal
        view
        returns (
            address[] memory recipients,
            uint256[] memory amounts,
            uint32[] memory destEids,
            bytes[] memory opts
        )
    {
        recipients = new address[](10);
        amounts = new uint256[](10);
        destEids = new uint32[](10);
        opts = new bytes[](10);

        for (uint256 i = 0; i < 10; i++) {
            recipients[i] = address(0);
            amounts[i] = 0;
            destEids[i] = 40439;
            opts[i] = bytes("");
        }
    }

    function testDepositRejectsOversizedEnvelope() public {
        bytes memory envelope = new bytes(4097);

        vm.prank(user);
        vm.expectRevert(bytes("bad envelope"));
        pool.deposit(1, 100, uint256(uint160(user)), envelope);
    }

    function testDepositRejectsUnauthorizedUser() public {
        vm.mockCall(
            0x403c000000000000000000000000000000000000,
            abi.encodeWithSelector(ITIP403Registry.isAuthorized.selector, uint64(2), user),
            abi.encode(false)
        );

        vm.prank(user);
        vm.expectRevert(bytes("TIP403 blocked"));
        pool.deposit(1, 100, uint256(uint160(user)), hex"1234");
    }

    function testBatchWithdrawRejectsWrongRelayer() public {
        uint[50] memory pubSignals;
        pubSignals[11] = uint256(pool.currentRoot());
        pubSignals[12] = uint256(bytes32("nf"));
        pubSignals[16] = 1;
        pubSignals[17] = 40439;
        pubSignals[49] = uint256(uint160(relayer));

        (address[] memory recipients, uint256[] memory amounts, uint32[] memory destEids, bytes[] memory opts) = _blankArrays();

        for (uint256 i = 0; i < 10; i++) {
            pubSignals[18 + i] = 0;
            pubSignals[28 + i] = 0;
            pubSignals[38 + i] = 40439;
        }

        vm.expectRevert(bytes("relayer mismatch"));
        pool.batchWithdraw(
            [uint256(0), uint256(0)],
            [[uint256(0), uint256(0)], [uint256(0), uint256(0)]],
            [uint256(0), uint256(0)],
            pubSignals,
            recipients,
            amounts,
            destEids,
            bytes32(0),
            bytes(""),
            opts
        );
    }

    function testBatchWithdrawRejectsBadPaddedRecipient() public {
        uint[50] memory pubSignals;
        pubSignals[11] = uint256(pool.currentRoot());
        pubSignals[12] = uint256(bytes32("nf"));
        pubSignals[16] = 1;
        pubSignals[17] = 40439;
        pubSignals[49] = uint256(uint160(address(this)));

        (address[] memory recipients, uint256[] memory amounts, uint32[] memory destEids, bytes[] memory opts) = _blankArrays();

        for (uint256 i = 0; i < 10; i++) {
            pubSignals[18 + i] = 0;
            pubSignals[28 + i] = 0;
            pubSignals[38 + i] = 40439;
        }

        recipients[3] = address(0x9999);

        vm.expectRevert(bytes("recipient mismatch"));
        pool.batchWithdraw(
            [uint256(0), uint256(0)],
            [[uint256(0), uint256(0)], [uint256(0), uint256(0)]],
            [uint256(0), uint256(0)],
            pubSignals,
            recipients,
            amounts,
            destEids,
            bytes32(0),
            bytes(""),
            opts
        );
    }

    function testBatchWithdrawRejectsZeroRecipientForNonZeroAmount() public {
        uint[50] memory pubSignals;
        pubSignals[11] = uint256(pool.currentRoot());
        pubSignals[12] = uint256(bytes32("nf"));
        pubSignals[16] = 1;
        pubSignals[17] = 40439;
        pubSignals[49] = uint256(uint160(address(this)));

        (address[] memory recipients, uint256[] memory amounts, uint32[] memory destEids, bytes[] memory opts) = _blankArrays();

        for (uint256 i = 0; i < 10; i++) {
            pubSignals[18 + i] = 0;
            pubSignals[28 + i] = 0;
            pubSignals[38 + i] = 40439;
        }

        amounts[0] = 100;
        pubSignals[28] = 100;

        vm.expectRevert(bytes("zero recipient"));
        pool.batchWithdraw(
            [uint256(0), uint256(0)],
            [[uint256(0), uint256(0)], [uint256(0), uint256(0)]],
            [uint256(0), uint256(0)],
            pubSignals,
            recipients,
            amounts,
            destEids,
            bytes32(0),
            bytes(""),
            opts
        );
    }

    function testQuoteCrossChainFeeUsesAltFeeQuote() public {
        uint256 fee = pool.quoteCrossChainFee(
            30101,
            address(0xBEEF),
            1000,
            hex"0102"
        );

        assertEq(fee, 7);
    }

    function testQuoteCrossChainFeeRejectsZeroRecipient() public {
        vm.expectRevert(bytes("bad recipient"));
        pool.quoteCrossChainFee(30101, address(0), 1000, hex"0102");
    }

    function testNextLeafIndexStartsAtZero() public view {
        assertEq(pool.nextLeafIndex(), 0);
    }
}
```

## 24.3 File: `packages/contracts/test/ParlySpokeGateway.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "../src/ParlySpokeGateway.sol";
import "../src/interfaces/IParlyOFT.sol";

contract GatewayMockToken is IERC20 {
    string public name = "Mock";
    string public symbol = "MOCK";
    uint8 public decimals = 6;
    uint256 public override totalSupply;

    mapping(address => uint256) public override balanceOf;
    mapping(address => mapping(address => uint256)) public override allowance;

    function transfer(address to, uint256 amount) external override returns (bool) {
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }

    function approve(address spender, uint256 amount) external override returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external override returns (bool) {
        uint256 allowed = allowance[from][msg.sender];
        if (allowed != type(uint256).max) {
            allowance[from][msg.sender] = allowed - amount;
        }
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        emit Transfer(from, to, amount);
        return true;
    }

    function mint(address to, uint256 amount) external {
        totalSupply += amount;
        balanceOf[to] += amount;
        emit Transfer(address(0), to, amount);
    }
}

contract MockOFTAltFee is IParlyOFT {
    uint256 public feeAmount = 5;
    bytes public lastOptions;

    function setFee(uint256 v) external {
        feeAmount = v;
    }

    function quoteSend(
        SendParam calldata p,
        bool payInLzToken
    ) external view returns (MessagingFee memory) {
        require(payInLzToken, "must use alt fee");
        require(p.extraOptions.length > 0, "options missing");
        return MessagingFee({ nativeFee: 0, lzTokenFee: feeAmount });
    }

    function send(
        SendParam calldata p,
        MessagingFee calldata,
        address
    ) external payable returns (MessagingReceipt memory, OFTReceipt memory) {
        lastOptions = p.extraOptions;
        return (
            MessagingReceipt({
                guid: bytes32(uint256(1)),
                nonce: 1,
                fee: MessagingFee({ nativeFee: 0, lzTokenFee: 0 })
            }),
            OFTReceipt({ amountSentLD: 0, amountReceivedLD: 0 })
        );
    }
}

contract MockOFTNativeFee is IParlyOFT {
    uint256 public feeAmount = 9;
    bytes public lastOptions;

    function quoteSend(
        SendParam calldata p,
        bool payInLzToken
    ) external view returns (MessagingFee memory) {
        require(!payInLzToken, "must use native fee");
        require(p.extraOptions.length > 0, "options missing");
        return MessagingFee({ nativeFee: feeAmount, lzTokenFee: 0 });
    }

    function send(
        SendParam calldata p,
        MessagingFee calldata,
        address
    ) external payable returns (MessagingReceipt memory, OFTReceipt memory) {
        lastOptions = p.extraOptions;
        require(msg.value == feeAmount, "bad msg.value");
        return (
            MessagingReceipt({
                guid: bytes32(uint256(2)),
                nonce: 2,
                fee: MessagingFee({ nativeFee: 0, lzTokenFee: 0 })
            }),
            OFTReceipt({ amountSentLD: 0, amountReceivedLD: 0 })
        );
    }
}

contract ParlySpokeGatewayTest is Test {
    GatewayMockToken token;
    GatewayMockToken feeToken;
    MockOFTAltFee oftAlt;
    MockOFTNativeFee oftNative;
    ParlySpokeGateway altGateway;
    ParlySpokeGateway nativeGateway;

    address user = address(0x1111);

    function setUp() public {
        token = new GatewayMockToken();
        feeToken = new GatewayMockToken();
        oftAlt = new MockOFTAltFee();
        oftNative = new MockOFTNativeFee();

        altGateway = new ParlySpokeGateway(
            address(token),
            address(feeToken),
            address(oftAlt),
            1,
            40210,
            bytes32(uint256(uint160(address(0xBEEF)))),
            65_000,
            500_000,
            0,
            address(this)
        );

        nativeGateway = new ParlySpokeGateway(
            address(token),
            address(0),
            address(oftNative),
            1,
            40210,
            bytes32(uint256(uint160(address(0xBEEF)))),
            65_000,
            500_000,
            0,
            address(this)
        );

        token.mint(user, 1_000_000);
        feeToken.mint(user, 1_000_000);

        vm.startPrank(user);
        token.approve(address(altGateway), type(uint256).max);
        token.approve(address(nativeGateway), type(uint256).max);
        feeToken.approve(address(altGateway), type(uint256).max);
        vm.stopPrank();
    }

    function testQuoteShieldFeeUsesAltFeeMode() public {
        (uint256 feeAmount, bool usesAlt) = altGateway.quoteShieldFee(
            123,
            100,
            user,
            hex"1234"
        );

        assertEq(feeAmount, 5);
        assertTrue(usesAlt);
    }

    function testQuoteShieldFeeRejectsZeroAltFeeQuote() public {
        oftAlt.setFee(0);

        vm.expectRevert(bytes("zero alt fee"));
        altGateway.quoteShieldFee(123, 100, user, hex"1234");
    }

    function testQuoteShieldFeeUsesNativeMode() public {
        (uint256 feeAmount, bool usesAlt) = nativeGateway.quoteShieldFee(
            123,
            100,
            user,
            hex"1234"
        );

        assertEq(feeAmount, 9);
        assertFalse(usesAlt);
    }

    function testShieldRejectsOversizedEnvelope() public {
        vm.prank(user);
        vm.expectRevert(bytes("bad envelope"));
        altGateway.shieldCrossChain(123, 100, new bytes(4097));
    }

    function testAltFeeShieldLeavesZeroAllowanceAtRest() public {
        vm.prank(user);
        altGateway.shieldCrossChain(123, 100, hex"1234");

        assertEq(token.allowance(address(altGateway), address(oftAlt)), 0);
        assertEq(feeToken.allowance(address(altGateway), address(oftAlt)), 0);
        assertGt(oftAlt.lastOptions().length, 0);
    }

    function testNativeFeeShieldLeavesZeroAllowanceAtRest() public {
        vm.prank(user);
        nativeGateway.shieldCrossChain{ value: 9 }(123, 100, hex"1234");

        assertEq(token.allowance(address(nativeGateway), address(oftNative)), 0);
        assertGt(oftNative.lastOptions().length, 0);
    }

    function testPreviewExecutionOptionsIsNonEmpty() public view {
        assertGt(altGateway.previewExecutionOptions().length, 0);
    }
}
```

---

## 24.4 File: `packages/contracts/test/ParlyHubComposer.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "../src/ParlyHubComposer.sol";

contract ComposerMockToken is IERC20 {
    string public name = "Mock";
    string public symbol = "MOCK";
    uint8 public decimals = 6;
    uint256 public override totalSupply;

    mapping(address => uint256) public override balanceOf;
    mapping(address => mapping(address => uint256)) public override allowance;

    function transfer(address to, uint256 amount) external override returns (bool) {
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }

    function approve(address spender, uint256 amount) external override returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external override returns (bool) {
        uint256 allowed = allowance[from][msg.sender];
        if (allowed != type(uint256).max) {
            allowance[from][msg.sender] = allowed - amount;
        }
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        emit Transfer(from, to, amount);
        return true;
    }

    function mint(address to, uint256 amount) external {
        totalSupply += amount;
        balanceOf[to] += amount;
        emit Transfer(address(0), to, amount);
    }
}

contract ComposerMockOFTAdapter {
    address public immutable token;

    constructor(address token_) {
        token = token_;
    }
}

contract ComposerMockPool {
    address public immutable underlyingToken;
    bool public revertDeposit;
    bool public ingressAuthorized = true;
    uint256 public depositCalls;
    uint256 public lastInnerCommitment;
    uint256 public lastAmount;
    uint256 public lastDepositorField;
    bytes32 public lastCommitment;
    uint32 public lastLeafIndex;
    bytes public lastEnvelope;

    constructor(address token) {
        underlyingToken = token;
    }

    function setRevertDeposit(bool v) external {
        revertDeposit = v;
    }

    function setIngressAuthorized(bool v) external {
        ingressAuthorized = v;
    }

    function depositFromIngress(
        uint256 innerCommitment,
        uint256 principalAmount,
        uint256 depositorField,
        bytes calldata envelope
    ) external returns (bytes32 finalCommitment, uint32 leafIndex) {
        depositCalls += 1;
        if (!ingressAuthorized) revert("TIP403 blocked");
        if (revertDeposit) revert("mock deposit failure");

        lastInnerCommitment = innerCommitment;
        lastAmount = principalAmount;
        lastDepositorField = depositorField;
        lastEnvelope = envelope;
        lastCommitment = keccak256(abi.encode(innerCommitment, principalAmount));
        lastLeafIndex = 7;

        IERC20(underlyingToken).transferFrom(msg.sender, address(this), principalAmount);
        return (lastCommitment, lastLeafIndex);
    }
}

contract ParlyHubComposerHarness is ParlyHubComposer {
    constructor(address endpoint, address owner_) ParlyHubComposer(endpoint, owner_) {}

    function simulateComposeIngress(
        bytes32 guid,
        uint16 assetId,
        address localOft_,
        uint32 srcEid,
        bytes32 composeFrom,
        address depositor,
        uint256 amountLD,
        uint256 innerCommitment,
        bytes calldata envelope
    ) external returns (bool) {
        (bool ok, DeferredIngressReason reason, bytes memory errorData) = _attemptIngress(
            guid,
            assetId,
            localOft_,
            srcEid,
            composeFrom,
            depositor,
            amountLD,
            innerCommitment,
            envelope
        );

        if (!ok) {
            _recordDeferredIngress(
                guid,
                assetId,
                depositor,
                amountLD,
                innerCommitment,
                envelope,
                srcEid,
                composeFrom,
                localOft_,
                reason,
                errorData
            );
        }

        return ok;
    }

    function seedDeferredIngress(
        bytes32 guid,
        uint16 assetId,
        address token,
        address depositor,
        uint256 amountLD,
        uint256 innerCommitment,
        bytes calldata envelope,
        uint32 srcEid,
        bytes32 composeFrom,
        address localOft,
        DeferredIngressReason reason,
        bytes calldata errorData
    ) external {
        deferredIngresses[guid] = DeferredIngress({
            exists: true,
            assetId: assetId,
            token: token,
            depositor: depositor,
            amountLD: amountLD,
            innerCommitment: innerCommitment,
            envelope: envelope,
            srcEid: srcEid,
            composeFrom: composeFrom,
            localOft: localOft,
            reason: reason,
            errorData: errorData
        });
    }
}

contract ParlyHubComposerTest is Test {
    ComposerMockToken token;
    ComposerMockOFTAdapter localAdapter;
    ComposerMockPool pool;
    ParlyHubComposerHarness composer;

    address endpoint = address(0x1111);
    address depositor = address(0x2222);
    address gateway = address(0x3333);
    address localOft;

    function setUp() public {
        token = new ComposerMockToken();
        localAdapter = new ComposerMockOFTAdapter(address(token));
        pool = new ComposerMockPool(address(token));
        composer = new ParlyHubComposerHarness(endpoint, address(this));
        localOft = address(localAdapter);

        composer.setLocalOft(1, localOft);
        composer.setPool(1, address(pool));
    }

    function testComposeFailureCreatesDeferredIngressRecord() public {
        bytes32 guid = bytes32(uint256(11));
        token.mint(address(composer), 88);
        composer.setApprovedGateway(30101, bytes32(uint256(uint160(gateway))), true);
        pool.setRevertDeposit(true);

        bool ok = composer.simulateComposeIngress(
            guid,
            1,
            localOft,
            30101,
            bytes32(uint256(uint160(gateway))),
            depositor,
            88,
            123,
            hex"1234"
        );

        assertFalse(ok);
        ParlyHubComposer.DeferredIngress memory record = composer.getDeferredIngress(guid);
        assertTrue(record.exists);
        assertEq(record.token, address(token));
        assertEq(record.depositor, depositor);
        assertEq(record.amountLD, 88);
        assertEq(uint256(record.reason), uint256(ParlyHubComposer.DeferredIngressReason.DEPOSIT_REVERTED));
    }

    function testRetryDeferredIngressSucceedsAfterGatewayApproval() public {
        bytes32 guid = bytes32(uint256(1));
        token.mint(address(composer), 100);

        composer.seedDeferredIngress(
            guid,
            1,
            address(token),
            depositor,
            100,
            123,
            hex"1234",
            30101,
            bytes32(uint256(uint160(gateway))),
            localOft,
            ParlyHubComposer.DeferredIngressReason.UNAPPROVED_GATEWAY,
            bytes("unapproved gateway")
        );

        composer.setApprovedGateway(30101, bytes32(uint256(uint160(gateway))), true);

        bool ok = composer.retryDeferredIngress(guid);

        assertTrue(ok);
        assertTrue(composer.ingressSettled(guid));
        assertTrue(composer.ingressDeposited(guid));
        assertEq(token.balanceOf(address(pool)), 100);
        assertEq(pool.lastInnerCommitment(), 123);
        assertEq(pool.lastAmount(), 100);
        assertEq(pool.lastDepositorField(), uint256(uint160(depositor)));
        assertEq(uint256(pool.lastLeafIndex()), 7);
        ParlyHubComposer.DeferredIngress memory record = composer.getDeferredIngress(guid);
        assertFalse(record.exists);
    }

    function testRetryDeferredIngressRerunsPoolComplianceCheck() public {
        bytes32 guid = bytes32(uint256(12));
        token.mint(address(composer), 40);
        composer.setApprovedGateway(30101, bytes32(uint256(uint160(gateway))), true);
        pool.setIngressAuthorized(false);

        bool first = composer.simulateComposeIngress(
            guid,
            1,
            localOft,
            30101,
            bytes32(uint256(uint160(gateway))),
            depositor,
            40,
            777,
            hex"beef"
        );

        assertFalse(first);
        assertEq(pool.depositCalls(), 1);

        pool.setIngressAuthorized(true);
        bool second = composer.retryDeferredIngress(guid);

        assertTrue(second);
        assertEq(pool.depositCalls(), 2);
        assertEq(token.balanceOf(address(pool)), 40);
    }

    function testClaimDeferredIngressReturnsPrincipalOnlyToDepositor() public {
        bytes32 guid = bytes32(uint256(2));
        token.mint(address(composer), 55);

        composer.seedDeferredIngress(
            guid,
            1,
            address(token),
            depositor,
            55,
            456,
            hex"cafe",
            30110,
            bytes32(uint256(uint160(gateway))),
            localOft,
            ParlyHubComposer.DeferredIngressReason.DEPOSIT_REVERTED,
            bytes("deposit reverted")
        );

        uint256 beforeBalance = token.balanceOf(depositor);

        vm.prank(depositor);
        composer.claimDeferredIngress(guid);

        assertTrue(composer.ingressSettled(guid));
        assertEq(token.balanceOf(depositor), beforeBalance + 55);
        assertEq(token.balanceOf(address(composer)), 0);
        ParlyHubComposer.DeferredIngress memory record = composer.getDeferredIngress(guid);
        assertFalse(record.exists);
    }

    function testClaimDeferredIngressWorksWhenPoolIsMissingButAdapterTokenIsKnown() public {
        ParlyHubComposerHarness poolMissingComposer = new ParlyHubComposerHarness(endpoint, address(this));
        poolMissingComposer.setLocalOft(1, localOft);
        poolMissingComposer.setApprovedGateway(30101, bytes32(uint256(uint160(gateway))), true);
        token.mint(address(poolMissingComposer), 61);

        bool ok = poolMissingComposer.simulateComposeIngress(
            bytes32(uint256(77)),
            1,
            localOft,
            30101,
            bytes32(uint256(uint160(gateway))),
            depositor,
            61,
            999,
            hex"abba"
        );

        assertFalse(ok);
        ParlyHubComposer.DeferredIngress memory record = poolMissingComposer.getDeferredIngress(bytes32(uint256(77)));
        assertEq(record.token, address(token));

        vm.prank(depositor);
        poolMissingComposer.claimDeferredIngress(bytes32(uint256(77)));

        assertEq(token.balanceOf(depositor), 61);
    }

    function testClaimDeferredIngressWorksForBadAssetWhenCreditedAdapterTokenIsKnown() public {
        ParlyHubComposerHarness badAssetComposer = new ParlyHubComposerHarness(endpoint, address(this));
        token.mint(address(badAssetComposer), 29);

        bool ok = badAssetComposer.simulateComposeIngress(
            bytes32(uint256(88)),
            9,
            localOft,
            30101,
            bytes32(uint256(uint160(gateway))),
            depositor,
            29,
            1234,
            hex"feed"
        );

        assertFalse(ok);
        ParlyHubComposer.DeferredIngress memory record = badAssetComposer.getDeferredIngress(bytes32(uint256(88)));
        assertEq(record.token, address(token));

        vm.prank(depositor);
        badAssetComposer.claimDeferredIngress(bytes32(uint256(88)));

        assertEq(token.balanceOf(depositor), 29);
    }

    function testClaimDeferredIngressDoesNotInvokeSecondComplianceGate() public {
        bytes32 guid = bytes32(uint256(19));
        token.mint(address(composer), 33);

        composer.seedDeferredIngress(
            guid,
            1,
            address(token),
            depositor,
            33,
            555,
            hex"c0de",
            30101,
            bytes32(uint256(uint160(gateway))),
            localOft,
            ParlyHubComposer.DeferredIngressReason.DEPOSIT_REVERTED,
            bytes("TIP403 blocked")
        );

        pool.setIngressAuthorized(false);
        uint256 beforeCalls = pool.depositCalls();

        vm.prank(depositor);
        composer.claimDeferredIngress(guid);

        assertEq(pool.depositCalls(), beforeCalls);
        assertEq(token.balanceOf(depositor), 33);
    }
}
```

---

## 25. Canonical deployment CLI order

## 25.1 Dev Note

Run from repo root unless the step explicitly changes directory.

These CLI snippets assume a POSIX shell. On Windows, execute them inside WSL or Git Bash.

## 25.2 Corrected CLI sequence

```bash
# ==============================================================================
# STEP 1 - TOOLCHAIN
# ==============================================================================
foundryup
cd parly-fi-v16_9_9
pnpm install

# contracts deps
cd packages/contracts
forge install OpenZeppelin/openzeppelin-contracts@v5.0.2 --no-commit
forge install LayerZero-Labs/LayerZero-v2 --no-commit
forge install layerzero-labs/devtools --no-commit
forge install foundry-rs/forge-std --no-commit
cd ../../

# ==============================================================================
# STEP 2 - ENVIRONMENT
# ==============================================================================
cp .env.example .env
# Fill all required values before running scripts.

# ==============================================================================
# STEP 3 - BUILD CIRCUIT
# ==============================================================================
cd packages/circuits
pnpm install
pnpm build

wget https://storage.googleapis.com/zkevm/ptau/powersOfTau28_hez_final_14.ptau
b2sum powersOfTau28_hez_final_14.ptau
# expected BLAKE2b:
# eeefbcf7c3803b523c94112023c7ff89558f9b8e0cf5d6cdcba3ade60f168af4a181c9c21774b94fbae6c90411995f7d854d02ebd93fb66043dbb06f17a831c1

npx snarkjs groth16 setup joinsplit.r1cs powersOfTau28_hez_final_14.ptau joinsplit_0000.zkey
npx snarkjs zkey contribute joinsplit_0000.zkey joinsplit_final.zkey --name="PARLY_v16_9_9_ENTROPY" -v
npx snarkjs zkey export solidityverifier joinsplit_final.zkey ZKVerifier.sol

mkdir -p ../../apps/web/public/circuits
mkdir -p ../../packages/sdk/assets

cp joinsplit.wasm ../../apps/web/public/circuits/
cp joinsplit_final.zkey ../../apps/web/public/circuits/
cp joinsplit.wasm ../../packages/sdk/assets/
cp joinsplit_final.zkey ../../packages/sdk/assets/

cd ../contracts

# ==============================================================================
# STEP 4 - COPY GENERATED VERIFIER
# ==============================================================================
node script/CopyGeneratedVerifier.js

# ==============================================================================
# STEP 5 - REPO-WIDE TYPECHECK, BUILD, AND CONTRACT TEST GATES
# ==============================================================================
# Re-run here after circuit artifact copy so generated verifier and workspace assembly drift
# are caught before any deployment broadcast.
cd ../..
pnpm typecheck
pnpm build
cd packages/contracts

forge build
forge test -vvv

# ==============================================================================
# STEP 6 - DEPLOY POSEIDON
# ==============================================================================
node script/DeployPoseidon.js
# Copy POSEIDON_HASHER_ADDRESS into .env

# ==============================================================================
# STEP 7 - DEPLOY GROTH16 VERIFIER
# ==============================================================================
forge script script/DeployVerifier.s.sol --rpc-url $TEMPO_RPC_URL --broadcast -vvvv
# Copy GROTH16_VERIFIER_ADDRESS into .env

# ==============================================================================
# STEP 8 - DEPLOY TREASURY
# ==============================================================================
forge script script/DeployTreasury.s.sol --rpc-url $TEMPO_RPC_URL --broadcast -vvvv
# Copy PROTOCOL_TREASURY_ADDRESS into .env

# ==============================================================================
# STEP 9 - DEPLOY HUB CORE
# ==============================================================================
forge script script/DeployHubCore.s.sol --rpc-url $TEMPO_RPC_URL --broadcast -vvvv
# Copy:
# NEXT_PUBLIC_USDC_POOL
# NEXT_PUBLIC_USDT_POOL
# NEXT_PUBLIC_HUB_COMPOSER

# ==============================================================================
# STEP 10 - DEPLOY HUB OFTs / ADAPTERS
# ==============================================================================
# Use the approved LayerZero ERC20-adapter-compatible flow for Tempo.
# Populate HUB_USDC_OFT and HUB_USDT_OFT.

# ==============================================================================
# STEP 11 - DEPLOY SPOKE OFTs / ADAPTERS
# ==============================================================================
# Use the approved LayerZero ERC20-adapter-compatible flow on each spoke.
# Populate all SPOKE_USDC_OFT_* and SPOKE_USDT_OFT_* values.

# ==============================================================================
# STEP 12 - DEPLOY SPOKE GATEWAYS
# ==============================================================================
PARLY_ASSET=USDC forge script script/DeploySpokeGateway.s.sol --rpc-url $ETHEREUM_RPC_URL --broadcast -vvvv
PARLY_ASSET=USDC forge script script/DeploySpokeGateway.s.sol --rpc-url $ARBITRUM_RPC_URL --broadcast -vvvv
PARLY_ASSET=USDC forge script script/DeploySpokeGateway.s.sol --rpc-url $BASE_RPC_URL --broadcast -vvvv
PARLY_ASSET=USDC forge script script/DeploySpokeGateway.s.sol --rpc-url $BSC_RPC_URL --broadcast -vvvv

PARLY_ASSET=USDT forge script script/DeploySpokeGateway.s.sol --rpc-url $ETHEREUM_RPC_URL --broadcast -vvvv
PARLY_ASSET=USDT forge script script/DeploySpokeGateway.s.sol --rpc-url $ARBITRUM_RPC_URL --broadcast -vvvv
PARLY_ASSET=USDT forge script script/DeploySpokeGateway.s.sol --rpc-url $BASE_RPC_URL --broadcast -vvvv
PARLY_ASSET=USDT forge script script/DeploySpokeGateway.s.sol --rpc-url $BSC_RPC_URL --broadcast -vvvv

# ==============================================================================
# STEP 13 - UPDATE GATEWAY ADDRESSES IN .ENV
# ==============================================================================
# Fill NEXT_PUBLIC_SPOKE_USDC_GATEWAY_* and NEXT_PUBLIC_SPOKE_USDT_GATEWAY_*
# Also fill START_BLOCK_HUB_COMPOSER and every START_BLOCK_SPOKE_* value so
# source dispatch and Tempo settlement provenance are indexed from the correct blocks.

# ==============================================================================
# STEP 14 - CONFIGURE HUB
# ==============================================================================
forge script script/ConfigureHub.s.sol --rpc-url $TEMPO_RPC_URL --broadcast -vvvv

# ==============================================================================
# STEP 15 - WIRE LAYERZERO PATHWAYS
# ==============================================================================
npx hardhat lz:oapp:wire --oapp-config layerzero.config.ts

# ==============================================================================
# STEP 16 - KEEP DEPLOYER OWNERSHIP FOR SMOKE
# ==============================================================================
# Stop here for deployment. Do not transfer ownership from this condensed build
# sequence. The final canonical flow later in the runbook performs smoke testing
# first and only then hands control to PROTOCOL_ADMIN_ADDRESS.

cd ../../
```

---

## 26. End of Part 2

---

PARLY.FI MASTER TECHNICAL BLUEPRINT

Final Specification V16.9.9 - Canonical Zero-Trust Engineering Runbook

Part 3 - Web Client, Deterministic Recovery, Note Cryptography, Optional Cache, Corrected Ponder Indexer, Waku Relay, Relayer, SDK

---

27. Superseding client and off-chain corrections from V16.8

27.1 Architectural reasoning

V16.9.9 corrects the remaining off-chain and client defects that still mattered after V16.8:
	*	batch-withdrawal event ABI now matches the contract,
	*	batch-withdrawal indexer rows now use on-chain assetId instead of hardcoded values,
	*	relayer no longer marks nullifiers as seen before transaction confirmation,
	*	approval helpers are hardened for zero-reset ERC20 behavior,
	*	relay UX no longer shows the same success state as direct on-chain completion,
	*	relayer key generation is now explicit and junior-readable,
	*	SDK build script is explicit,
	*	the off-chain stack stays aligned with the on-chain pool and event model.

27.2 Dev Note

Treat this Part 3 as authoritative over any earlier client, relayer, SDK, and indexer sections.

---

28. Shared browser helpers

28.1 Architectural reasoning

Parly client logic must stay predictable across:
	*	browser,
	*	relayer,
	*	SDK,
	*	and future service tools.

So V16.9.9 centralizes the most error-prone transformations:
	*	hex/bytes conversion,
	*	field/address conversions,
	*	bigint-safe JSON preparation,
	*	registry parsing,
	*	exact-length enforcement.

28.2 File: apps/web/src/lib/encoding.ts

export function hexToBytes(hex: string): Uint8Array {
  const clean = hex.startsWith("0x") ? hex.slice(2) : hex
  if (clean.length % 2 !== 0) {
    throw new Error("Invalid hex length")
  }
  if (!/^[0-9a-fA-F]*$/.test(clean)) {
    throw new Error("Invalid hex value")
  }

  const out = new Uint8Array(clean.length / 2)
  for (let i = 0; i < out.length; i++) {
    const byte = Number.parseInt(clean.slice(i * 2, i * 2 + 2), 16)
    if (Number.isNaN(byte)) {
      throw new Error("Invalid hex byte")
    }
    out[i] = byte
  }
  return out
}

export function bytesToHex(bytes: Uint8Array): `0x${string}` {
  return `0x${Array.from(bytes)
    .map((b) => b.toString(16).padStart(2, "0"))
    .join("")}` as `0x${string}`
}

export function bigintReplacer(_key: string, value: unknown) {
  return typeof value === "bigint" ? value.toString() : value
}

export function isExactLengthArray<T>(value: unknown, length: number): value is T[] {
  return Array.isArray(value) && value.length === length
}

28.3 File: apps/web/src/lib/field.ts

import { isAddress } from "viem"

const MAX_EVM_ADDRESS_FIELD = (1n << 160n) - 1n

export function addressToField(address: string): bigint {
  if (!isAddress(address as `0x${string}`)) {
    throw new Error(`Invalid address: ${address}`)
  }
  return BigInt(address)
}

export function fieldToAddress(field: bigint): `0x${string}` {
  if (field < 0n || field > MAX_EVM_ADDRESS_FIELD) {
    throw new Error("Field does not fit into 160-bit EVM address space")
  }
  return `0x${field.toString(16).padStart(40, "0")}` as `0x${string}`
}

28.4 File: apps/web/src/lib/relayer-registry.ts

export type {
  RelayerRegistryEntry,
  RelayerRegistryResponse
} from "@parly/shared-types"

export {
  fetchOfficialRelayerRegistry,
  isLikelyRelayerPublicKey,
  parseLegacyRelayerRegistry
} from "@parly/registry-client"

28.5 File: apps/web/src/lib/relayer-registry-server.ts

import { createHmac, randomBytes, timingSafeEqual } from "node:crypto"
import { Pool } from "pg"
import { getAddress, isAddress, recoverMessageAddress } from "viem"
import { buildRelayerHeartbeatMessage } from "@parly/crypto-utils"
import type { NextRequest } from "next/server"
import { NextResponse } from "next/server"
import { requireEnv, requirePositiveInt } from "@parly/env-utils"
import type { RelayerRegistryResponse } from "@parly/shared-types"

export type RelayerStatus = "verified" | "inactive" | "rejected"

export class RelayerRegistryError extends Error {
  constructor(
    public code:
      | "REGISTRY_FULL"
      | "INVALID_ADDRESS"
      | "INVALID_PUBLIC_KEY"
      | "INVALID_HEARTBEAT"
      | "DUPLICATE_ADDRESS"
      | "DUPLICATE_PUBLIC_KEY"
      | "INVALID_SIGNATURE"
      | "RATE_LIMITED"
      | "UNKNOWN_ERROR",
    message: string,
    public httpStatus = 400
  ) {
    super(message)
  }
}

declare global {
  var __parlyRelayerRegistryPool: Pool | undefined
}

const registryPool =
  globalThis.__parlyRelayerRegistryPool ||
  new Pool({
    connectionString: requireEnv("DATABASE_URL")
  })

if (!globalThis.__parlyRelayerRegistryPool) {
  globalThis.__parlyRelayerRegistryPool = registryPool
}

const REGISTRY_MAX_LIVE = 10

function nonceTtlSeconds() {
  return requirePositiveInt("RELAYER_REGISTRY_NONCE_TTL_SECONDS")
}

function heartbeatTtlSeconds() {
  return requirePositiveInt("RELAYER_REGISTRY_HEARTBEAT_TTL_SECONDS")
}

function ipAttemptLimit() {
  return requirePositiveInt("RELAYER_REGISTRY_IP_ATTEMPTS_PER_24H")
}

function addressAttemptLimit() {
  return requirePositiveInt("RELAYER_REGISTRY_ADDRESS_ATTEMPTS_PER_24H")
}

function normalizeExecutionAddress(value: unknown): `0x${string}` {
  if (typeof value !== "string" || !isAddress(value as `0x${string}`)) {
    throw new RelayerRegistryError("INVALID_ADDRESS", "Invalid relayer execution address", 400)
  }
  return getAddress(value).toLowerCase() as `0x${string}`
}

function normalizePublicKey(value: unknown): string {
  const raw = String(value || "").trim()
  if (!/^[A-Za-z0-9+/=]+$/.test(raw)) {
    throw new RelayerRegistryError("INVALID_PUBLIC_KEY", "Invalid relayer public key", 400)
  }

  let decoded: Buffer
  try {
    decoded = Buffer.from(raw, "base64")
  } catch {
    throw new RelayerRegistryError("INVALID_PUBLIC_KEY", "Invalid relayer public key", 400)
  }

  if (decoded.length !== 32) {
    throw new RelayerRegistryError("INVALID_PUBLIC_KEY", "Invalid relayer public key", 400)
  }

  return raw
}

export function buildRelayerRegistrationMessage(
  executionAddress: `0x${string}`,
  nonce: string
) {
  return [
    "Parly Relayer Registration",
    `Execution Address: ${executionAddress}`,
    `Nonce: ${nonce}`,
    "Domain: parly.fi"
  ].join("\n")
}

export function createRelayerRegistrationNonce(executionAddress: `0x${string}`) {
  const issuedAt = Math.floor(Date.now() / 1000)
  const entropy = randomBytes(18).toString("hex")
  const payload = `${executionAddress}:${issuedAt}:${entropy}`
  const mac = createHmac("sha256", requireEnv("RELAYER_REGISTRY_NONCE_SECRET"))
    .update(payload)
    .digest("hex")

  return `${issuedAt}.${entropy}.${mac}`
}

export function verifyRelayerRegistrationNonce(
  executionAddress: `0x${string}`,
  nonce: string
) {
  const [issuedAtRaw, entropy, mac] = String(nonce || "").split(".")
  const issuedAt = Number(issuedAtRaw || "0")

  if (!Number.isFinite(issuedAt) || !Number.isInteger(issuedAt) || issuedAt <= 0 || !entropy || !mac) {
    throw new RelayerRegistryError("INVALID_SIGNATURE", "Signature verification failed", 401)
  }

  const now = Math.floor(Date.now() / 1000)
  if (issuedAt + nonceTtlSeconds() < now) {
    throw new RelayerRegistryError("INVALID_SIGNATURE", "Signature verification failed", 401)
  }

  const payload = `${executionAddress}:${issuedAt}:${entropy}`
  const expected = createHmac("sha256", requireEnv("RELAYER_REGISTRY_NONCE_SECRET"))
    .update(payload)
    .digest("hex")

  const left = Buffer.from(mac, "hex")
  const right = Buffer.from(expected, "hex")

  if (left.length !== right.length || !timingSafeEqual(left, right)) {
    throw new RelayerRegistryError("INVALID_SIGNATURE", "Signature verification failed", 401)
  }
}

async function countPersistentAttempts(keyType: "ip" | "address", keyValue: string) {
  const result = await registryPool.query<{ count: string }>(
    `
      SELECT COUNT(*)::text AS count
      FROM relayer_registration_attempts
      WHERE key_type = $1
        AND key_value = $2
        AND created_at >= NOW() - INTERVAL '24 hours'
    `,
    [keyType, keyValue]
  )

  return Number(result.rows[0]?.count || "0")
}

async function recordPersistentAttempt(keyType: "ip" | "address", keyValue: string) {
  await registryPool.query(
    `
      INSERT INTO relayer_registration_attempts (key_type, key_value, created_at)
      VALUES ($1, $2, CURRENT_TIMESTAMP)
    `,
    [keyType, keyValue]
  )
}

export function getRequestIp(req: NextRequest) {
  const candidates = [
    req.headers.get("cf-connecting-ip"),
    req.headers.get("x-real-ip"),
    req.headers.get("x-vercel-forwarded-for"),
    req.headers.get("x-forwarded-for")
  ]

  for (const raw of candidates) {
    if (!raw) continue
    const value = raw.split(",")[0].trim()
    if (/^[A-Fa-f0-9:.]+$/.test(value)) {
      return value
    }
  }

  return null
}

export async function enforceRegistrationRateLimits(
  ip: string | null,
  executionAddress: `0x${string}`
) {
  const nextAddressCount = await countPersistentAttempts("address", executionAddress)
  if (nextAddressCount >= addressAttemptLimit()) {
    throw new RelayerRegistryError("RATE_LIMITED", "Too many attempts", 429)
  }

  if (ip) {
    const nextIpCount = await countPersistentAttempts("ip", ip)
    if (nextIpCount >= ipAttemptLimit()) {
      throw new RelayerRegistryError("RATE_LIMITED", "Too many attempts", 429)
    }
  }

  await recordPersistentAttempt("address", executionAddress)
  if (ip) {
    await recordPersistentAttempt("ip", ip)
  }
}

export async function pruneInactiveRelayers() {
  const result = await registryPool.query(
    `
      UPDATE relayers
      SET
        status = 'inactive',
        updated_at = CURRENT_TIMESTAMP,
        disabled_reason = 'AUTO_INACTIVE_30D'
      WHERE
        status = 'verified'
        AND COALESCE(last_success_at, created_at) < NOW() - INTERVAL '30 days'
    `
  )

  return result.rowCount || 0
}

export async function getLiveRelayerRegistry(): Promise<RelayerRegistryResponse> {
  await pruneInactiveRelayers()

  const maxCount = REGISTRY_MAX_LIVE
  const rows = await registryPool.query<{
    execution_address: string
    public_key_b64: string
  }>(
    `
      SELECT execution_address, public_key_b64
      FROM relayers
      WHERE status = 'verified'
      ORDER BY COALESCE(last_success_at, created_at) DESC, created_at ASC
      LIMIT $1
    `,
    [maxCount]
  )

  const countResult = await registryPool.query<{ count: string }>(
    `SELECT COUNT(*)::text AS count FROM relayers WHERE status = 'verified'`
  )

  const liveCount = Number(countResult.rows[0]?.count || "0")

  return {
    items: rows.rows.map((row) => ({
      executionAddress: row.execution_address as `0x${string}`,
      publicKeyB64: row.public_key_b64
    })),
    liveCount,
    maxCount,
    registryOpen: liveCount < maxCount
  }
}

async function assertRegistryHasCapacity() {
  const snapshot = await getLiveRelayerRegistry()
  if (!snapshot.registryOpen) {
    throw new RelayerRegistryError("REGISTRY_FULL", "Registry is currently full", 409)
  }
  return snapshot
}

export async function createRegistrationChallenge(executionAddressInput: unknown) {
  const executionAddress = normalizeExecutionAddress(executionAddressInput)
  await assertRegistryHasCapacity()

  const nonce = createRelayerRegistrationNonce(executionAddress)
  const message = buildRelayerRegistrationMessage(executionAddress, nonce)

  return { executionAddress, nonce, message }
}

export async function confirmRelayerRegistration(args: {
  executionAddress: unknown
  publicKeyB64: unknown
  signature: unknown
  nonce: unknown
  ip: string | null
}) {
  const executionAddress = normalizeExecutionAddress(args.executionAddress)
  const publicKeyB64 = normalizePublicKey(args.publicKeyB64)
  const signature = String(args.signature || "") as `0x${string}`
  const nonce = String(args.nonce || "")

  await enforceRegistrationRateLimits(args.ip, executionAddress)
  verifyRelayerRegistrationNonce(executionAddress, nonce)

  const message = buildRelayerRegistrationMessage(executionAddress, nonce)
  const recovered = await recoverMessageAddress({
    message,
    signature
  })

  if (recovered.toLowerCase() !== executionAddress) {
    throw new RelayerRegistryError("INVALID_SIGNATURE", "Signature verification failed", 401)
  }

  const client = await registryPool.connect()

  try {
    await client.query("BEGIN")
    await client.query("LOCK TABLE relayers IN SHARE ROW EXCLUSIVE MODE")

    const liveCountResult = await client.query<{ count: string }>(
      `SELECT COUNT(*)::text AS count FROM relayers WHERE status = 'verified'`
    )
    const liveCount = Number(liveCountResult.rows[0]?.count || "0")
    if (liveCount >= REGISTRY_MAX_LIVE) {
      throw new RelayerRegistryError("REGISTRY_FULL", "Registry is currently full", 409)
    }

    const existingByAddress = await client.query<{
      id: string
      status: RelayerStatus
      public_key_b64: string
    }>(
      `SELECT id::text, status, public_key_b64 FROM relayers WHERE execution_address = $1 LIMIT 1`,
      [executionAddress]
    )

    const addressRow = existingByAddress.rows[0]
    if (addressRow?.status === "verified" || addressRow?.status === "rejected") {
      throw new RelayerRegistryError("DUPLICATE_ADDRESS", "Relayer already exists", 409)
    }

    const existingByPublicKey = await client.query<{
      id: string
      status: RelayerStatus
      execution_address: string
    }>(
      `SELECT id::text, status, execution_address FROM relayers WHERE public_key_b64 = $1 LIMIT 1`,
      [publicKeyB64]
    )

    const publicKeyRow = existingByPublicKey.rows[0]
    if (publicKeyRow && publicKeyRow.execution_address !== executionAddress) {
      throw new RelayerRegistryError("DUPLICATE_PUBLIC_KEY", "Public key already exists", 409)
    }

    if (addressRow?.status === "inactive") {
      await client.query(
        `
          UPDATE relayers
          SET
            public_key_b64 = $2,
            status = 'verified',
            updated_at = CURRENT_TIMESTAMP,
            last_seen_at = CURRENT_TIMESTAMP,
            disabled_reason = NULL
          WHERE id = $1::bigint
        `,
        [addressRow.id, publicKeyB64]
      )
      await client.query("COMMIT")
      return { success: true, status: "verified" as const, reactivated: true }
    }

    await client.query(
      `
        INSERT INTO relayers (
          execution_address,
          public_key_b64,
          status,
          created_at,
          updated_at,
          last_seen_at
        )
        VALUES ($1, $2, 'verified', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP)
      `,
      [executionAddress, publicKeyB64]
    )

    await client.query("COMMIT")
  } catch (error) {
    await client.query("ROLLBACK")
    throw error
  } finally {
    client.release()
  }

  return {
    success: true as const,
    status: "verified" as const
  }
}

export async function recordRelayerHeartbeat(args: {
  executionAddress: unknown
  event: unknown
  timestamp: unknown
  signature: unknown
}) {
  const executionAddress = normalizeExecutionAddress(args.executionAddress)
  const event = String(args.event || "")
  const timestamp = Number(args.timestamp ?? 0)
  const signature = String(args.signature || "") as `0x${string}`

  if (event !== "SUCCESS") {
    throw new RelayerRegistryError("INVALID_HEARTBEAT", "Heartbeat verification failed", 400)
  }

  if (!Number.isFinite(timestamp) || !Number.isInteger(timestamp) || timestamp <= 0) {
    throw new RelayerRegistryError("INVALID_HEARTBEAT", "Heartbeat verification failed", 401)
  }

  const now = Math.floor(Date.now() / 1000)
  if (Math.abs(now - timestamp) > heartbeatTtlSeconds()) {
    throw new RelayerRegistryError("INVALID_HEARTBEAT", "Heartbeat verification failed", 401)
  }

  const recovered = await recoverMessageAddress({
    message: buildRelayerHeartbeatMessage(executionAddress, "SUCCESS", timestamp),
    signature
  })

  if (recovered.toLowerCase() !== executionAddress) {
    throw new RelayerRegistryError("INVALID_HEARTBEAT", "Heartbeat verification failed", 401)
  }

  const update = await registryPool.query(
    `
      UPDATE relayers
      SET
        updated_at = CURRENT_TIMESTAMP,
        last_success_at = CURRENT_TIMESTAMP,
        last_seen_at = CURRENT_TIMESTAMP
      WHERE execution_address = $1
        AND status = 'verified'
      RETURNING id
    `,
    [executionAddress]
  )

  if ((update.rowCount || 0) !== 1) {
    throw new RelayerRegistryError(
      "RELAYER_NOT_ACTIVE",
      "Heartbeat update did not match an active verified relayer row",
      409
    )
  }

  return {
    success: true as const,
    status: "verified" as const,
    updated: true as const
  }
}

export function registryRouteError(error: unknown) {
  if (error instanceof RelayerRegistryError) {
    return NextResponse.json(
      {
        error: error.code,
        message: error.message
      },
      { status: error.httpStatus }
    )
  }

  return NextResponse.json(
    {
      error: "UNKNOWN_ERROR",
      message: "Registration failed"
    },
    { status: 500 }
  )
}

28.6 File: apps/web/src/db/relayers.sql

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

CREATE TABLE IF NOT EXISTS relayer_registration_attempts (
  id BIGSERIAL PRIMARY KEY,
  key_type VARCHAR(16) NOT NULL CHECK (key_type IN ('ip', 'address')),
  key_value TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS relayer_registration_attempts_key_idx
  ON relayer_registration_attempts(key_type, key_value, created_at);


---

29. Deterministic wallet compatibility and recovery key derivation

29.1 Architectural reasoning

The recovery model is chain-first, not device-first.

Source of truth is:
	1.	wallet,
	2.	deterministic signature over a fixed domain-separated message,
	3.	on-chain envelopes,
	4.	on-chain leaves,
	5.	on-chain nullifiers.

That is why V16.9.9 keeps asymmetric sealed-box envelopes.

29.2 Founder Note

This does not require raw private-key export.

The browser only asks for a deterministic signature over a fixed authentication string.

29.3 Dev Note

Reject non-deterministic signers in privacy mode.

29.4 File: apps/web/src/lib/wallet-compat.ts

export async function assertDeterministicSigner(
  signMessageAsync: (args: { message: string }) => Promise<`0x${string}`>,
  message: string
): Promise<`0x${string}`> {
  const first = await signMessageAsync({ message })
  const second = await signMessageAsync({ message })

  if (first !== second) {
    throw new Error("Unsupported signer for privacy mode. Use a deterministic software EVM wallet.")
  }

  return first
}

29.5 File: apps/web/src/lib/recovery-keys.ts

import sodium from "libsodium-wrappers-sumo"
import { keccak256, encodePacked } from "viem"
import { hexToBytes } from "./encoding"

export const MASTER_AUTH_MESSAGE =
  "Unlock Parly.fi Privacy.\n" +
  "Domain: app.parly.fi\n" +
  "Purpose: asymmetric note recovery and private execution.\n" +
  "Only sign on the official Parly domain."

export type RecoveryKeypair = {
  publicKeyB64: string
  privateKeyB64: string
}

export async function deriveRecoveryKeypair(
  signature: `0x${string}`
): Promise<RecoveryKeypair> {
  await sodium.ready

  const seedHex = keccak256(
    encodePacked(["bytes", "string"], [signature, "PARLY_v16_9_9_RECOVERY_SEED"])
  )

  const seed = hexToBytes(seedHex).slice(0, sodium.crypto_box_SEEDBYTES)
  const kp = sodium.crypto_box_seed_keypair(seed)

  return {
    publicKeyB64: sodium.to_base64(kp.publicKey, sodium.base64_variants.ORIGINAL),
    privateKeyB64: sodium.to_base64(kp.privateKey, sodium.base64_variants.ORIGINAL)
  }
}

export async function deriveOptionalCacheKey(
  signature: `0x${string}`
): Promise<Uint8Array> {
  await sodium.ready

  const seedHex = keccak256(
    encodePacked(["bytes", "string"], [signature, "PARLY_v16_9_9_OPTIONAL_CACHE_KEY"])
  )

  return hexToBytes(seedHex).slice(0, sodium.crypto_secretbox_KEYBYTES)
}


---

30. Note cryptography and note model

30.1 Architectural reasoning

A note is cryptographic state, not UI state.

So V16.9.9 keeps the launch model:
	*	each note has random secret,
	*	each note has random nullifier,
	*	the envelope carries those values,
	*	the envelope is sealed to the intended owner's recovery public key,
	*	the user reconstructs the note from chain data alone.

30.2 Dev Note

The contract does not inspect envelope contents.
Envelopes exist for recovery only.

30.3 File: apps/web/src/lib/note-crypto.ts

import sodium from "libsodium-wrappers-sumo"
import { buildPoseidon } from "circomlibjs"
import { bytesToHex, hexToBytes } from "./encoding"

export type NoteEnvelopePayload = {
  version: 1
  kind: "deposit" | "change"
  assetId: 1 | 2
  amount: string
  secret: string
  nullifier: string
  depositor: string
  ownerRecoveryPubKeyB64: string
  createdAt: number
}

export type LiveNote = NoteEnvelopePayload & {
  commitment: `0x${string}`
  nullifierHash: `0x${string}`
  status: "live" | "spent"
}

function randomFieldElementFromBytes(bytes: Uint8Array): string {
  const hex = Array.from(bytes)
    .map((b) => b.toString(16).padStart(2, "0"))
    .join("")
  return BigInt(`0x${hex}`).toString()
}

export async function randomFieldElement(): Promise<string> {
  await sodium.ready
  const bytes = sodium.randombytes_buf(31)
  return randomFieldElementFromBytes(bytes)
}

export async function calcInnerCommitment(
  secret: string,
  nullifier: string,
  depositor: string,
  assetId: 1 | 2
): Promise<string> {
  const poseidon = await buildPoseidon()
  const out = poseidon([
    BigInt(secret),
    BigInt(nullifier),
    BigInt(depositor),
    BigInt(assetId)
  ])
  return BigInt(poseidon.F.toString(out)).toString()
}

export async function calcFinalCommitment(
  innerCommitment: string,
  amount: string
): Promise<`0x${string}`> {
  const poseidon = await buildPoseidon()
  const out = poseidon([BigInt(innerCommitment), BigInt(amount)])
  return `0x${BigInt(poseidon.F.toString(out)).toString(16).padStart(64, "0")}` as `0x${string}`
}

export async function calcNullifierHash(
  secret: string,
  nullifier: string
): Promise<`0x${string}`> {
  const poseidon = await buildPoseidon()
  const out = poseidon([BigInt(secret), BigInt(nullifier)])
  return `0x${BigInt(poseidon.F.toString(out)).toString(16).padStart(64, "0")}` as `0x${string}`
}

export async function calcChildKey(
  secret: string,
  recipient: string,
  amount: string,
  destEid: number
): Promise<`0x${string}`> {
  const poseidon = await buildPoseidon()
  const out = poseidon([
    BigInt(secret),
    BigInt(recipient),
    BigInt(amount),
    BigInt(destEid)
  ])
  return `0x${BigInt(poseidon.F.toString(out)).toString(16).padStart(64, "0")}` as `0x${string}`
}

export async function createSealedNoteEnvelope(
  recipientRecoveryPubKeyB64: string,
  payload: NoteEnvelopePayload
): Promise<`0x${string}`> {
  await sodium.ready

  const recipientPub = sodium.from_base64(
    recipientRecoveryPubKeyB64,
    sodium.base64_variants.ORIGINAL
  )

  const msg = sodium.from_string(JSON.stringify(payload))
  const sealed = sodium.crypto_box_seal(msg, recipientPub)

  return bytesToHex(sealed)
}

export async function openSealedNoteEnvelope(
  recipientPublicKeyB64: string,
  recipientPrivateKeyB64: string,
  envelope: `0x${string}`
): Promise<NoteEnvelopePayload> {
  await sodium.ready

  const pk = sodium.from_base64(recipientPublicKeyB64, sodium.base64_variants.ORIGINAL)
  const sk = sodium.from_base64(recipientPrivateKeyB64, sodium.base64_variants.ORIGINAL)
  const ciphertext = hexToBytes(envelope)

  const opened = sodium.crypto_box_seal_open(ciphertext, pk, sk)
  if (!opened) {
    throw new Error("Envelope decryption failed")
  }

  return JSON.parse(sodium.to_string(opened)) as NoteEnvelopePayload
}

export async function createFreshNote(
  recipientRecoveryPubKeyB64: string,
  depositor: string,
  assetId: 1 | 2,
  amount: string,
  kind: "deposit" | "change"
) {
  const secret = await randomFieldElement()
  const nullifier = await randomFieldElement()
  const innerCommitment = await calcInnerCommitment(secret, nullifier, depositor, assetId)
  const commitment = await calcFinalCommitment(innerCommitment, amount)
  const nullifierHash = await calcNullifierHash(secret, nullifier)

  const payload: NoteEnvelopePayload = {
    version: 1,
    kind,
    assetId,
    amount,
    secret,
    nullifier,
    depositor,
    ownerRecoveryPubKeyB64: recipientRecoveryPubKeyB64,
    createdAt: Math.floor(Date.now() / 1000)
  }

  const envelope = await createSealedNoteEnvelope(recipientRecoveryPubKeyB64, payload)

  return {
    secret,
    nullifier,
    innerCommitment,
    commitment,
    nullifierHash,
    envelope,
    payload
  }
}

export async function reconstructEnvelopeNote(
  payload: NoteEnvelopePayload
): Promise<{
  commitment: `0x${string}`
  nullifierHash: `0x${string}`
}> {
  const inner = await calcInnerCommitment(
    payload.secret,
    payload.nullifier,
    payload.depositor,
    payload.assetId
  )

  const commitment = await calcFinalCommitment(inner, payload.amount)
  const nullifierHash = await calcNullifierHash(payload.secret, payload.nullifier)

  return { commitment, nullifierHash }
}


---

31. Optional encrypted local cache

31.1 Architectural reasoning

Local storage must not be required for correctness.

So V16.9.9 keeps it:
	*	optional,
	*	encrypted,
	*	disposable,
	*	non-authoritative.

31.2 File: apps/web/src/lib/optional-note-cache.ts

import sodium from "libsodium-wrappers-sumo"
import type { LiveNote } from "./note-crypto"

export type OptionalRecoveryHeads = {
  leafCursor: string | null
  envelopeCursor: string | null
  nullifierCursor: string | null
}

export type OptionalNoteCachePayload = {
  version: 1
  notes: LiveNote[]
  heads: OptionalRecoveryHeads | null
}

function storageKey(address: string, assetId: 1 | 2) {
  return `parly_v16_9_9_optional_cache_${address.toLowerCase()}_${assetId}`
}

function bytesToBase64(bytes: Uint8Array): string {
  let binary = ""
  for (let i = 0; i < bytes.length; i++) binary += String.fromCharCode(bytes[i])
  return btoa(binary)
}

function base64ToBytes(base64: string): Uint8Array {
  const binary = atob(base64)
  const out = new Uint8Array(binary.length)
  for (let i = 0; i < binary.length; i++) out[i] = binary.charCodeAt(i)
  return out
}

export async function saveOptionalEncryptedCache(
  address: string,
  assetId: 1 | 2,
  key: Uint8Array,
  payload: OptionalNoteCachePayload
) {
  if (typeof window === "undefined") return
  if (process.env.NEXT_PUBLIC_ENABLE_OPTIONAL_NOTE_CACHE !== "1") return

  await sodium.ready

  const nonce = sodium.randombytes_buf(sodium.crypto_secretbox_NONCEBYTES)
  const msg = new TextEncoder().encode(JSON.stringify(payload))
  const cipher = sodium.crypto_secretbox_easy(msg, nonce, key)

  const packed = new Uint8Array(nonce.length + cipher.length)
  packed.set(nonce)
  packed.set(cipher, nonce.length)

  localStorage.setItem(storageKey(address, assetId), bytesToBase64(packed))
}

export async function loadOptionalEncryptedCache(
  address: string,
  assetId: 1 | 2,
  key: Uint8Array
): Promise<OptionalNoteCachePayload | null> {
  if (typeof window === "undefined") return null
  if (process.env.NEXT_PUBLIC_ENABLE_OPTIONAL_NOTE_CACHE !== "1") return null

  await sodium.ready

  const raw = localStorage.getItem(storageKey(address, assetId))
  if (!raw) return null

  try {
    const packed = base64ToBytes(raw)
    const nonce = packed.slice(0, sodium.crypto_secretbox_NONCEBYTES)
    const cipher = packed.slice(sodium.crypto_secretbox_NONCEBYTES)
    const msg = sodium.crypto_secretbox_open_easy(cipher, nonce, key)
    if (!msg) {
      localStorage.removeItem(storageKey(address, assetId))
      return null
    }
    const parsed = JSON.parse(new TextDecoder().decode(msg)) as
      | OptionalNoteCachePayload
      | LiveNote[]

    if (Array.isArray(parsed)) {
      return {
        version: 1,
        notes: parsed,
        heads: null
      }
    }

    return {
      version: 1,
      notes: Array.isArray(parsed.notes) ? parsed.notes : [],
      heads: parsed.heads || null
    }
  } catch {
    localStorage.removeItem(storageKey(address, assetId))
    return null
  }
}


---

32. Merkle helpers

32.1 Architectural reasoning

Client-side tree reconstruction must match on-chain zero initialization:
	*	zero[0] = Poseidon(0,0)
	*	zero[i+1] = Poseidon(zero[i], zero[i])

32.2 File: apps/web/src/lib/merkle.ts

import { MerkleTree } from "fixed-merkle-tree"
import { buildPoseidon } from "circomlibjs"

export async function buildParlyTree(leaves: string[]) {
  const poseidon = await buildPoseidon()

  const zero = BigInt(poseidon.F.toString(poseidon([0n, 0n])))

  return new MerkleTree(
    32,
    leaves.map((x) => BigInt(x).toString()),
    {
      hashFunction: (l: string | bigint, r: string | bigint) =>
        poseidon.F.toString(poseidon([BigInt(l), BigInt(r)])),
      zeroElement: zero.toString()
    }
  )
}


---

33. Public clients and shared ABIs

33.1 Architectural reasoning

The frontend should not duplicate ABI fragments across many files.

33.2 File: apps/web/src/lib/abis.ts

export {
  POOL_ABI,
  SPOKE_GATEWAY_ABI,
  ERC20_ABI,
  TREASURY_ABI
} from "@parly/protocol-abis"

33.3 File: apps/web/src/lib/public-client.ts

import { createPublicClient, http, defineChain } from "viem"
import { arbitrum, base, bsc, mainnet } from "viem/chains"
import { requireEnv, requirePositiveChainId } from "@parly/env-utils"

const tempoChainId = requirePositiveChainId("NEXT_PUBLIC_TEMPO_CHAIN_ID")
const tempoRpcUrl = requireEnv("NEXT_PUBLIC_TEMPO_RPC_URL")

export const tempoChain = defineChain({
  id: tempoChainId,
  name: "Tempo L1",
  nativeCurrency: { name: "TMP", symbol: "TMP", decimals: 18 },
  rpcUrls: {
    default: {
      http: [tempoRpcUrl]
    }
  },
  blockExplorers: {
    default: {
      name: "Tempo Explorer",
      url: "https://contracts.tempo.xyz"
    }
  },
  testnet: false
})

export const tempoPublicClient = createPublicClient({
  chain: tempoChain,
  transport: http(tempoRpcUrl)
})

export const ethereumPublicClient = createPublicClient({
  chain: mainnet,
  transport: http(process.env.NEXT_PUBLIC_ETHEREUM_RPC_URL || "https://eth.llamarpc.com")
})

export const arbitrumPublicClient = createPublicClient({
  chain: arbitrum,
  transport: http(process.env.NEXT_PUBLIC_ARBITRUM_RPC_URL || "https://arb1.arbitrum.io/rpc")
})

export const basePublicClient = createPublicClient({
  chain: base,
  transport: http(process.env.NEXT_PUBLIC_BASE_RPC_URL || "https://mainnet.base.org")
})

export const bscPublicClient = createPublicClient({
  chain: bsc,
  transport: http(process.env.NEXT_PUBLIC_BSC_RPC_URL || "https://bsc-dataseed.binance.org")
})


---

33.4 File: apps/web/src/lib/spoke-config.ts

import {
  ethereumPublicClient,
  arbitrumPublicClient,
  basePublicClient,
  bscPublicClient
} from "./public-client"

export type SupportedSpoke = "ETHEREUM" | "ARBITRUM" | "BASE" | "BNB"

export type BrowserSpokeConfig = {
  key: SupportedSpoke
  label: string
  chainId: number
  gateway: `0x${string}`
  publicClient:
    | typeof ethereumPublicClient
    | typeof arbitrumPublicClient
    | typeof basePublicClient
    | typeof bscPublicClient
}

const ZERO_ADDRESS = "0x0000000000000000000000000000000000000000"

export function getBrowserSpokes(asset: "USDC" | "USDT"): BrowserSpokeConfig[] {
  const ethereumGateway =
    asset === "USDC"
      ? process.env.NEXT_PUBLIC_SPOKE_USDC_GATEWAY_ETHEREUM
      : process.env.NEXT_PUBLIC_SPOKE_USDT_GATEWAY_ETHEREUM
  const arbitrumGateway =
    asset === "USDC"
      ? process.env.NEXT_PUBLIC_SPOKE_USDC_GATEWAY_ARBITRUM
      : process.env.NEXT_PUBLIC_SPOKE_USDT_GATEWAY_ARBITRUM
  const baseGateway =
    asset === "USDC"
      ? process.env.NEXT_PUBLIC_SPOKE_USDC_GATEWAY_BASE
      : process.env.NEXT_PUBLIC_SPOKE_USDT_GATEWAY_BASE
  const bnbGateway =
    asset === "USDC"
      ? process.env.NEXT_PUBLIC_SPOKE_USDC_GATEWAY_BNB
      : process.env.NEXT_PUBLIC_SPOKE_USDT_GATEWAY_BNB

  const out: BrowserSpokeConfig[] = [
    {
      key: "ETHEREUM",
      label: "Ethereum",
      chainId: Number(process.env.NEXT_PUBLIC_ETHEREUM_CHAIN_ID || 1),
      gateway: (ethereumGateway || ZERO_ADDRESS) as `0x${string}`,
      publicClient: ethereumPublicClient
    },
    {
      key: "ARBITRUM",
      label: "Arbitrum",
      chainId: Number(process.env.NEXT_PUBLIC_ARBITRUM_CHAIN_ID || 42161),
      gateway: (arbitrumGateway || ZERO_ADDRESS) as `0x${string}`,
      publicClient: arbitrumPublicClient
    },
    {
      key: "BASE",
      label: "Base",
      chainId: Number(process.env.NEXT_PUBLIC_BASE_CHAIN_ID || 8453),
      gateway: (baseGateway || ZERO_ADDRESS) as `0x${string}`,
      publicClient: basePublicClient
    },
    {
      key: "BNB",
      label: "BNB Chain",
      chainId: Number(process.env.NEXT_PUBLIC_BSC_CHAIN_ID || 56),
      gateway: (bnbGateway || ZERO_ADDRESS) as `0x${string}`,
      publicClient: bscPublicClient
    }
  ]

  return out.filter((x) => x.gateway.toLowerCase() !== ZERO_ADDRESS)
}


---

34. Hardened ERC20 approval helper for browser flows

34.1 Architectural reasoning

Because launch scope includes USDT-style tokens, raw one-shot approve(spender, amount) is not strong enough.

For off-chain approval paths, V16.9.9 hardens approval flow to:
	1.	read current allowance,
	2.	if current allowance is nonzero and target amount is nonzero, reset to zero first,
	3.	wait for receipt,
	4.	set desired amount,
	5.	wait for receipt.

34.2 File: apps/web/src/lib/erc20-approval.ts

import type { Abi, PublicClient } from "viem"
import { ERC20_ABI } from "./abis"
import { tempoPublicClient } from "./public-client"

export async function safeApproveExactBrowser(args: {
  token: `0x${string}`
  spender: `0x${string}`
  amount: bigint
  writeAndWait: (args: {
    address: `0x${string}`
    abi: Abi
    functionName: string
    args?: readonly unknown[]
  }) => Promise<`0x${string}`>
  owner: `0x${string}`
  publicClient?: PublicClient
}) {
  const readClient = args.publicClient || tempoPublicClient

  const currentAllowance = await readClient.readContract({
    address: args.token,
    abi: ERC20_ABI,
    functionName: "allowance",
    args: [args.owner, args.spender]
  }) as bigint

  if (currentAllowance === args.amount) return

  if (currentAllowance !== 0n && args.amount !== 0n) {
    await args.writeAndWait({
      address: args.token,
      abi: ERC20_ABI,
      functionName: "approve",
      args: [args.spender, 0n]
    })
  }

  await args.writeAndWait({
    address: args.token,
    abi: ERC20_ABI,
    functionName: "approve",
    args: [args.spender, args.amount]
  })
}


---

35. Corrected indexer fetchers

35.1 Architectural reasoning

The client only needs public indexed chain data:
	*	deposits,
	*	leaves,
	*	envelopes,
	*	nullifiers,
	*	batch metadata,
	*	and deterministic source + settlement provenance for spoke-origin deposits.

That is enough for recovery without trusting local state.

35.2 File: apps/web/src/lib/indexer.ts

import { decodeFunctionData } from "viem"
import { requirePositiveEid } from "@parly/env-utils"
import { POOL_ABI } from "./abis"
import { tempoPublicClient } from "./public-client"

const PONDER_URL = process.env.NEXT_PUBLIC_PONDER_GRAPHQL_URL!

function indexerApiBaseUrl() {
  return PONDER_URL.replace(/\/graphql\/?$/i, "")
}

function parsePositiveInt(raw: string | undefined, fallback: number): number {
  const value = Number(raw ?? String(fallback))
  if (!Number.isFinite(value) || !Number.isInteger(value) || value <= 0) {
    return fallback
  }
  return value
}

const INDEXER_PAGE_SIZE = parsePositiveInt(process.env.NEXT_PUBLIC_INDEXER_PAGE_SIZE, 500)
const TEMPO_LOCAL_EID = requirePositiveEid("NEXT_PUBLIC_TEMPO_LZ_EID")
const ETHEREUM_LZ_EID = requirePositiveEid("NEXT_PUBLIC_LZ_EID_ETHEREUM")
const ARBITRUM_LZ_EID = requirePositiveEid("NEXT_PUBLIC_LZ_EID_ARBITRUM")
const BASE_LZ_EID = requirePositiveEid("NEXT_PUBLIC_LZ_EID_BASE")
const BNB_LZ_EID = requirePositiveEid("NEXT_PUBLIC_LZ_EID_BNB")

export type DepositRow = {
  depositor: string
  commitment: string
  amount: string
  leafIndex: number
  assetId: number
  txHash: string
  timestamp: string
}

export type IngressSettlementRow = {
  guid: string
  commitment: string
  assetId: number
  depositor: string
  amount: string
  innerCommitment: string
  settlementLeafIndex: number
  settlementChain: string
  settlementEid: number
  sourceEid: number
  composeFrom: string
  txHash: string
  timestamp: string
}

export type SpokeShieldDispatchRow = {
  guid: string
  assetId: number
  sender: string
  amount: string
  innerCommitment: string
  sourceChain: string
  sourceEid: number
  gateway: string
  txHash: string
  timestamp: string
}

export type WithdrawalRow = {
  nullifierHash: string
  executor: string
  protocolFee: string
  executorFee: string
  childKeys: string[] | string
  assetId: number
  txHash: string
  timestamp: string
}

export type DepositHistoryRow = DepositRow & {
  origin: "tempo" | "spoke"
  provenance?: {
    guid: string
    settlementChain: string
    settlementEid: number
    settlementTxHash: string
    sourceChain?: string
    sourceEid: number
    sourceTxHash?: string
    sourceSender?: string
    composeFrom: string
  }
}

export type RecoveryHeads = {
  leafCursor: string | null
  envelopeCursor: string | null
  nullifierCursor: string | null
}

export type ScopedBatchOutputRow = {
  childKey: string
  outputIndex: number
  recipient: `0x${string}`
  amount: string
  destEid: number
  destinationChain: string
  routeType: "same_chain" | "cross_chain"
  txHash: string
  assetId: 1 | 2
  timestamp: number
  protocolFee: string
  executorFee: string
  nullifierHash: string
}

function parseChildKeySlots(value: string[] | string): string[] {
  if (Array.isArray(value)) return value
  return JSON.parse(value)
}

function destinationChainLabel(destEid: number): string {
  if (destEid === TEMPO_LOCAL_EID) return "Tempo"
  if (destEid === ETHEREUM_LZ_EID) return "Ethereum"
  if (destEid === ARBITRUM_LZ_EID) return "Arbitrum"
  if (destEid === BASE_LZ_EID) return "Base"
  if (destEid === BNB_LZ_EID) return "BNB Chain"
  return `EID ${destEid}`
}

async function gql(query: string, variables: Record<string, unknown> = {}) {
  const res = await fetch(PONDER_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ query, variables }),
    cache: "no-store"
  })

  if (!res.ok) {
    throw new Error(`Indexer request failed with HTTP ${res.status}`)
  }

  const json = await res.json()
  if (json.errors?.length) {
    throw new Error(json.errors[0].message)
  }

  return json.data
}

async function fetchIndexerJson(pathname: string) {
  const res = await fetch(`${indexerApiBaseUrl()}${pathname}`, {
    cache: "no-store"
  })

  if (!res.ok) {
    throw new Error(`Indexer stats request failed with HTTP ${res.status}`)
  }

  return res.json()
}

async function fetchAllPages<T>(
  query: string,
  rootKey: string,
  variables: Record<string, unknown> = {}
): Promise<T[]> {
  let after: string | null = null
  const out: T[] = []

  while (true) {
    const data = await gql(query, {
      ...variables,
      limit: INDEXER_PAGE_SIZE,
      after
    })

    const page = data?.[rootKey]
    const items = page?.items || []
    out.push(...items)

    if (!page?.pageInfo?.hasNextPage || !page?.pageInfo?.endCursor) {
      break
    }

    after = page.pageInfo.endCursor
  }

  return out
}

export async function fetchDeposits(assetId: number): Promise<DepositRow[]> {
  const q = `
    query($assetId:Int!, $limit:Int!, $after:String) {
      depositEvents(
        where:{assetId:$assetId},
        orderBy:"timestamp",
        orderDirection:"asc",
        limit:$limit,
        after:$after
      ) {
        items { depositor commitment amount leafIndex assetId txHash timestamp }
        pageInfo { hasNextPage endCursor }
      }
    }
  `

  return fetchAllPages(q, "depositEvents", { assetId })
}

export async function fetchIngressSettlements(assetId: number): Promise<IngressSettlementRow[]> {
  const q = `
    query($assetId:Int!, $limit:Int!, $after:String) {
      ingressSettlementEvents(
        where:{assetId:$assetId},
        orderBy:"timestamp",
        orderDirection:"asc",
        limit:$limit,
        after:$after
      ) {
        items {
          guid
          commitment
          assetId
          depositor
          amount
          innerCommitment
          settlementLeafIndex
          settlementChain
          settlementEid
          sourceEid
          composeFrom
          txHash
          timestamp
        }
        pageInfo { hasNextPage endCursor }
      }
    }
  `

  return fetchAllPages(q, "ingressSettlementEvents", { assetId })
}

export async function fetchSpokeShieldDispatches(
  assetId: number
): Promise<SpokeShieldDispatchRow[]> {
  const q = `
    query($assetId:Int!, $limit:Int!, $after:String) {
      spokeShieldDispatchEvents(
        where:{assetId:$assetId},
        orderBy:"timestamp",
        orderDirection:"asc",
        limit:$limit,
        after:$after
      ) {
        items {
          guid
          assetId
          sender
          amount
          innerCommitment
          sourceChain
          sourceEid
          gateway
          txHash
          timestamp
        }
        pageInfo { hasNextPage endCursor }
      }
    }
  `

  return fetchAllPages(q, "spokeShieldDispatchEvents", { assetId })
}

export async function fetchDepositHistory(assetId: number): Promise<DepositHistoryRow[]> {
  const [deposits, settlements, dispatches] = await Promise.all([
    fetchDeposits(assetId),
    fetchIngressSettlements(assetId),
    fetchSpokeShieldDispatches(assetId)
  ])

  const settlementByCommitment = new Map(
    settlements.map((row) => [String(row.commitment).toLowerCase(), row] as const)
  )
  const dispatchByGuid = new Map(
    dispatches.map((row) => [String(row.guid).toLowerCase(), row] as const)
  )

  return deposits.map((row) => {
    const settlement = settlementByCommitment.get(String(row.commitment).toLowerCase())
    if (!settlement) {
      return { ...row, origin: "tempo" as const }
    }

    const dispatch = dispatchByGuid.get(String(settlement.guid).toLowerCase())
    if (!dispatch) {
      return { ...row, origin: "tempo" as const }
    }

    return {
      ...row,
      origin: "spoke" as const,
      provenance: {
        guid: settlement.guid,
        settlementChain: settlement.settlementChain,
        settlementEid: settlement.settlementEid,
        settlementTxHash: settlement.txHash,
        sourceChain: dispatch.sourceChain,
        sourceEid: settlement.sourceEid,
        sourceTxHash: dispatch.txHash,
        sourceSender: dispatch.sender,
        composeFrom: settlement.composeFrom
      }
    }
  })
}

export async function fetchRecoveryHeads(assetId: number): Promise<RecoveryHeads> {
  const q = `
    query($assetId:Int!) {
      leafInsertedEvents(where:{assetId:$assetId}, orderBy:"leafIndex", orderDirection:"desc", limit:1) {
        items { commitment leafIndex txHash }
      }
      noteEnvelopeEvents(where:{assetId:$assetId}, orderBy:"timestamp", orderDirection:"desc", limit:1) {
        items { commitment txHash timestamp }
      }
      batchWithdrawalEvents(where:{assetId:$assetId}, orderBy:"timestamp", orderDirection:"desc", limit:1) {
        items { nullifierHash txHash timestamp }
      }
    }
  `

  const data = await gql(q, { assetId })
  const leaf = data?.leafInsertedEvents?.items?.[0]
  const envelope = data?.noteEnvelopeEvents?.items?.[0]
  const nullifier = data?.batchWithdrawalEvents?.items?.[0]

  return {
    leafCursor: leaf ? `${leaf.leafIndex}:${leaf.commitment}:${leaf.txHash}` : null,
    envelopeCursor: envelope ? `${envelope.commitment}:${envelope.txHash}:${envelope.timestamp}` : null,
    nullifierCursor: nullifier ? `${nullifier.nullifierHash}:${nullifier.txHash}:${nullifier.timestamp}` : null
  }
}

export async function fetchLeaves(assetId: number) {
  const q = `
    query($assetId:Int!, $limit:Int!, $after:String) {
      leafInsertedEvents(
        where:{assetId:$assetId},
        orderBy:"leafIndex",
        orderDirection:"asc",
        limit:$limit,
        after:$after
      ) {
        items { commitment leafIndex kind txHash timestamp }
        pageInfo { hasNextPage endCursor }
      }
    }
  `

  return fetchAllPages(q, "leafInsertedEvents", { assetId })
}

export async function fetchWithdrawals(assetId: number): Promise<WithdrawalRow[]> {
  const q = `
    query($assetId:Int!, $limit:Int!, $after:String) {
      batchWithdrawalEvents(
        where:{assetId:$assetId},
        orderBy:"timestamp",
        orderDirection:"asc",
        limit:$limit,
        after:$after
      ) {
        items { nullifierHash executor protocolFee executorFee childKeys assetId txHash timestamp }
        pageInfo { hasNextPage endCursor }
      }
    }
  `

  return fetchAllPages(q, "batchWithdrawalEvents", { assetId })
}

export async function fetchDepositsByDepositor(depositor: string): Promise<DepositRow[]> {
  const q = `
    query($depositor:String!, $limit:Int!, $after:String) {
      depositEvents(
        where:{depositor:$depositor},
        orderBy:"timestamp",
        orderDirection:"asc",
        limit:$limit,
        after:$after
      ) {
        items { depositor commitment amount leafIndex assetId txHash timestamp }
        pageInfo { hasNextPage endCursor }
      }
    }
  `

  return fetchAllPages(q, "depositEvents", { depositor })
}

export async function fetchApproxPublicDepositVolume() {
  const data = await fetchIndexerJson("/stats/public-deposit-volume")
  const usdcTotal = BigInt(String(data.usdc || "0"))
  const usdtTotal = BigInt(String(data.usdt || "0"))

  return {
    usdc: usdcTotal,
    usdt: usdtTotal,
    total: usdcTotal + usdtTotal
  }
}

export async function fetchShieldMiningDepositedTotal(depositor: string): Promise<bigint> {
  const data = await fetchIndexerJson(`/stats/shield-mining/${encodeURIComponent(depositor)}`)
  return BigInt(String(data.totalDeposited || "0"))
}

export async function fetchDepositByCommitment(
  assetId: number,
  commitment: string
): Promise<DepositRow | null> {
  const q = `
    query($assetId:Int!, $commitment:String!) {
      depositEvents(where:{assetId:$assetId, commitment:$commitment}, limit:1) {
        items { depositor commitment amount leafIndex assetId txHash timestamp }
      }
    }
  `

  const data = await gql(q, { assetId, commitment })
  return data?.depositEvents?.items?.[0] || null
}

export async function fetchEnvelopes(assetId: number) {
  const q = `
    query($assetId:Int!, $limit:Int!, $after:String) {
      noteEnvelopeEvents(
        where:{assetId:$assetId},
        orderBy:"timestamp",
        orderDirection:"asc",
        limit:$limit,
        after:$after
      ) {
        items { commitment kind envelope txHash timestamp }
        pageInfo { hasNextPage endCursor }
      }
    }
  `

  return fetchAllPages(q, "noteEnvelopeEvents", { assetId })
}

export async function fetchNullifiers(assetId: number) {
  const q = `
    query($assetId:Int!, $limit:Int!, $after:String) {
      batchWithdrawalEvents(
        where:{assetId:$assetId},
        orderBy:"timestamp",
        orderDirection:"asc",
        limit:$limit,
        after:$after
      ) {
        items { nullifierHash txHash timestamp }
        pageInfo { hasNextPage endCursor }
      }
    }
  `

  return fetchAllPages(q, "batchWithdrawalEvents", { assetId })
}

export async function fetchTxBatches(txHash: string) {
  const q = `
    query($txHash:String!) {
      batchWithdrawalEvents(where:{txHash:$txHash}) {
        items { childKeys protocolFee executorFee txHash timestamp assetId nullifierHash }
      }
    }
  `

  const data = await gql(q, { txHash })
  return data?.batchWithdrawalEvents?.items || []
}

export async function fetchScopedBatchOutputs(txHash: `0x${string}`): Promise<ScopedBatchOutputRow[]> {
  const transaction = await tempoPublicClient.getTransaction({ hash: txHash })
  const decoded = decodeFunctionData({
    abi: POOL_ABI,
    data: transaction.input
  })

  if (decoded.functionName !== "batchWithdraw") {
    throw new Error("Transaction is not a Tempo batchWithdraw call.")
  }

  const args = decoded.args as unknown[]
  const recipients = args[4] as `0x${string}`[]
  const amounts = args[5] as bigint[]
  const destEids = args[6] as number[]
  const localEid = TEMPO_LOCAL_EID
  const rows = await fetchTxBatches(txHash)
  const out: ScopedBatchOutputRow[] = []

  if (recipients.length !== 10 || amounts.length !== 10 || destEids.length !== 10) {
    throw new Error("Decoded batchWithdraw calldata does not match the 10-output protocol model.")
  }

  for (const row of rows) {
    const childKeys = parseChildKeySlots(row.childKeys)

    for (let index = 0; index < childKeys.length; index++) {
      const childKey = String(childKeys[index]).toLowerCase()
      if (
        childKey === "0x0000000000000000000000000000000000000000000000000000000000000000"
      ) {
        continue
      }

      if (index >= recipients.length || index >= amounts.length || index >= destEids.length) {
        throw new Error("BatchWithdrawal child key slots did not align with decoded calldata arrays.")
      }

      out.push({
        childKey: childKeys[index],
        outputIndex: index,
        recipient: recipients[index],
        amount: amounts[index].toString(),
        destEid: Number(destEids[index]),
        destinationChain: destinationChainLabel(Number(destEids[index])),
        routeType: Number(destEids[index]) === localEid ? "same_chain" : "cross_chain",
        txHash: row.txHash,
        assetId: Number(row.assetId) as 1 | 2,
        timestamp: Number(row.timestamp),
        protocolFee: String(row.protocolFee),
        executorFee: String(row.executorFee),
        nullifierHash: String(row.nullifierHash)
      })
    }
  }

  return out
}

export async function fetchScopedBatchOutput(
  txHash: `0x${string}`,
  childKey: string
): Promise<ScopedBatchOutputRow | null> {
  const normalized = childKey.toLowerCase()
  const outputs = await fetchScopedBatchOutputs(txHash)
  return outputs.find((row) => row.childKey.toLowerCase() === normalized) || null
}


---

36. Recovery orchestration

36.1 Architectural reasoning

Recovery must be explicit:
	1.	decrypt envelope,
	2.	reconstruct commitment,
	3.	confirm commitment exists in leaf history,
	4.	derive nullifier hash,
	5.	mark note live or spent from public nullifier history.

36.2 File: apps/web/src/lib/recover-notes.ts

import type { LiveNote } from "./note-crypto"
import { openSealedNoteEnvelope, reconstructEnvelopeNote } from "./note-crypto"

export async function recoverNotesFromChain(args: {
  assetId: 1 | 2
  recoveryPublicKeyB64: string
  recoveryPrivateKeyB64: string
  envelopes: Array<{ commitment: string; envelope: `0x${string}` }>
  leaves: Array<{ commitment: string }>
  nullifiers: Array<{ nullifierHash: string }>
}): Promise<LiveNote[]> {
  const leafSet = new Set(args.leaves.map((x) => String(x.commitment).toLowerCase()))
  const nullifierSet = new Set(
    args.nullifiers.map((x) => String(x.nullifierHash).toLowerCase())
  )

  const recovered: LiveNote[] = []

  for (const env of args.envelopes) {
    try {
      const payload = await openSealedNoteEnvelope(
        args.recoveryPublicKeyB64,
        args.recoveryPrivateKeyB64,
        env.envelope
      )

      if (payload.assetId !== args.assetId) continue

      const { commitment, nullifierHash } = await reconstructEnvelopeNote(payload)

      if (String(commitment).toLowerCase() !== String(env.commitment).toLowerCase()) continue
      if (!leafSet.has(String(commitment).toLowerCase())) continue

      const spent = nullifierSet.has(String(nullifierHash).toLowerCase())

      recovered.push({
        ...payload,
        commitment,
        nullifierHash,
        status: spent ? "spent" : "live"
      })
    } catch {
      continue
    }
  }

  recovered.sort((a, b) => {
    const aa = BigInt(a.amount)
    const bb = BigInt(b.amount)
    if (aa === bb) return b.createdAt - a.createdAt
    return aa > bb ? -1 : 1
  })

  return recovered
}

36.3 File: apps/web/src/lib/useRecoveredNotes.ts

"use client"

import { useEffect, useState } from "react"
import {
  fetchEnvelopes,
  fetchLeaves,
  fetchNullifiers,
  fetchRecoveryHeads,
  type RecoveryHeads
} from "./indexer"
import { recoverNotesFromChain } from "./recover-notes"
import { loadOptionalEncryptedCache, saveOptionalEncryptedCache } from "./optional-note-cache"
import type { LiveNote } from "./note-crypto"

function headsEqual(a: RecoveryHeads | null, b: RecoveryHeads | null) {
  if (!a || !b) return false
  return (
    a.leafCursor === b.leafCursor &&
    a.envelopeCursor === b.envelopeCursor &&
    a.nullifierCursor === b.nullifierCursor
  )
}

export function useRecoveredNotes(args: {
  address?: string
  assetId: 1 | 2
  recoveryPublicKeyB64: string | null
  recoveryPrivateKeyB64: string | null
  optionalCacheKey: Uint8Array | null
  refreshNonce?: number
}) {
  const [notes, setNotes] = useState<LiveNote[]>([])
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)
  const optionalCacheKeyFingerprint = args.optionalCacheKey
    ? Array.from(args.optionalCacheKey).map((b) => b.toString(16).padStart(2, "0")).join("")
    : null

  useEffect(() => {
    let cancelled = false

    async function run() {
      if (!args.address || !args.recoveryPublicKeyB64 || !args.recoveryPrivateKeyB64) {
        setNotes([])
        return
      }

      try {
        setLoading(true)
        setError(null)
        setNotes([])

        let cachedHeads: RecoveryHeads | null = null

        if (args.optionalCacheKey) {
          const cached = await loadOptionalEncryptedCache(
            args.address,
            args.assetId,
            args.optionalCacheKey
          )
          cachedHeads = cached?.heads || null

          if (!cancelled) {
            setNotes(cached?.notes || [])
          }
        }

        const latestHeads = await fetchRecoveryHeads(args.assetId)

        if (args.optionalCacheKey && headsEqual(cachedHeads, latestHeads)) {
          return
        }

        const [envelopes, leaves, nullifiers] = await Promise.all([
          fetchEnvelopes(args.assetId),
          fetchLeaves(args.assetId),
          fetchNullifiers(args.assetId)
        ])

        const recovered = await recoverNotesFromChain({
          assetId: args.assetId,
          recoveryPublicKeyB64: args.recoveryPublicKeyB64,
          recoveryPrivateKeyB64: args.recoveryPrivateKeyB64,
          envelopes,
          leaves,
          nullifiers
        })

        if (!cancelled) {
          setNotes(recovered)
        }

        if (!cancelled && args.optionalCacheKey) {
          await saveOptionalEncryptedCache(
            args.address,
            args.assetId,
            args.optionalCacheKey,
            {
              version: 1,
              notes: recovered,
              heads: latestHeads
            }
          )
        }
      } catch (e: any) {
        if (!cancelled) setError(e.message || "Recovery failed")
      } finally {
        if (!cancelled) setLoading(false)
      }
    }

    run()

    return () => {
      cancelled = true
    }
  }, [
    args.address,
    args.assetId,
    args.recoveryPublicKeyB64,
    args.recoveryPrivateKeyB64,
    optionalCacheKeyFingerprint,
    args.refreshNonce || 0
  ])

  return { notes, loading, error, setNotes }
}


---

37. Relay payload normalization and relay crypto

37.1 Architectural reasoning

The browser and relayer must agree on one exact payload shape.

Because JSON does not natively serialize bigint cleanly, proof-related numerics are normalized to strings.

Fixed cardinalities remain:
	*	10 recipients,
	*	10 amounts,
	*	10 destination EIDs,
	*	10 LayerZero option blobs,
	*	50 public signals.

37.2 File: apps/web/src/lib/relay-types.ts

export type {
  CanonicalRelayExecutionPayload,
  RelayCipherBundle
} from "@parly/shared-types"

37.3 File: apps/web/src/lib/relay-crypto.ts

import sodium from "libsodium-wrappers-sumo"
import { bytesToHex } from "./encoding"
import type { CanonicalRelayExecutionPayload, RelayCipherBundle } from "./relay-types"

export async function sealForRelayer(
  relayerAddress: `0x${string}`,
  relayerPubKeyB64: string,
  payload: CanonicalRelayExecutionPayload
): Promise<RelayCipherBundle> {
  await sodium.ready

  const pub = sodium.from_base64(relayerPubKeyB64, sodium.base64_variants.ORIGINAL)
  const plaintext = sodium.from_string(JSON.stringify(payload))
  const sealed = sodium.crypto_box_seal(plaintext, pub)

  return {
    version: 1,
    relayerAddress,
    ciphertext: bytesToHex(sealed)
  }
}


---

38. Waku client

38.1 Architectural reasoning

The browser publishes one Waku message containing one relayer-specific encrypted bundle.

Each relayer opens only the bundle addressed to it.

38.2 File: apps/web/src/lib/waku.ts

import { createLightNode, createEncoder, Protocols } from "@waku/sdk"

export async function publishToWaku(payloadBytes: Uint8Array) {
  const contentTopic = process.env.NEXT_PUBLIC_WAKU_CONTENT_TOPIC || "/parly/16/9/9/app/proto"

  const node = await createLightNode({
    defaultBootstrap: process.env.NEXT_PUBLIC_WAKU_BOOTSTRAP === "true",
    networkConfig: {
      clusterId: Number(process.env.NEXT_PUBLIC_WAKU_CLUSTER_ID || 1),
      contentTopics: [contentTopic]
    }
  })

  await node.start()
  await node.waitForPeers([Protocols.LightPush])

  const encoder = createEncoder({ contentTopic, ephemeral: true })
  await node.lightPush.send(encoder, { payload: payloadBytes })

  await node.stop()
}


---

39. Prover worker

39.1 Dev Note

This assumes:
	*	joinsplit.wasm
	*	joinsplit_final.zkey

were copied into apps/web/public/circuits/.

39.2 File: apps/web/src/lib/prover.worker.ts

import * as snarkjs from "snarkjs"

self.addEventListener("message", async (event) => {
  try {
    const { circuitInputs } = event.data

    const { proof, publicSignals } = await snarkjs.groth16.fullProve(
      circuitInputs,
      "/circuits/joinsplit.wasm",
      "/circuits/joinsplit_final.zkey"
    )

    const pA = [
      `0x${BigInt(proof.pi_a[0]).toString(16)}`,
      `0x${BigInt(proof.pi_a[1]).toString(16)}`
    ] as const

    const pB = [
      [
        `0x${BigInt(proof.pi_b[0][1]).toString(16)}`,
        `0x${BigInt(proof.pi_b[0][0]).toString(16)}`
      ],
      [
        `0x${BigInt(proof.pi_b[1][1]).toString(16)}`,
        `0x${BigInt(proof.pi_b[1][0]).toString(16)}`
      ]
    ] as const

    const pC = [
      `0x${BigInt(proof.pi_c[0]).toString(16)}`,
      `0x${BigInt(proof.pi_c[1]).toString(16)}`
    ] as const

    self.postMessage({
      success: true,
      pA,
      pB,
      pC,
      publicSignals: publicSignals.map((x: string) => BigInt(x).toString())
    })
  } catch (e: any) {
    self.postMessage({
      success: false,
      error: e?.message || "WASM proving failed"
    })
  }
})


---

40. Main widget

40.1 Architectural reasoning

The launch widget must do all of the following coherently:
	1.	authenticate deterministically,
	2.	derive asymmetric recovery keys,
	3.	recover notes from chain,
	4.	shield deposits,
	5.	support single and batch spends,
	6.	pad to 10 outputs,
	7.	create change notes automatically,
	8.	use one selected private relayer honestly,
	9.	wait for approval receipts before dependent execution,
	10.	use zero-reset-safe approval paths for off-chain approval flows,
	11.	clean exact approvals only when direct browser submission never happens or when failure is terminal,
	12.	and surface uncertain post-broadcast browser states honestly instead of pretending they failed cleanly.

40.2 Dev Note

Relay mode must not show the same final state as confirmed on-chain direct execution.

Execution mode and route type are different axes in the widget:
	*	execution mode = Private Relay or Self-Relay
	*	route type = Same-Chain Tempo, Cross-Chain Route, or Mixed Route

The widget keeps Private Relay as the default launch path.

Self-Relay stays behind `Advanced Settings` only, because it is the lower-privacy fallback path rather than the default surface state.

Cross-chain delivery cost stays execution-scoped:
	*	same-chain routes never show a cross-chain delivery fee,
	*	private relay cross-chain keeps that cost inside relay execution pricing for this version,
	*	self-relay cross-chain shows the delivery fee only inside the Execute fee breakdown,
	*	self-relay cross-chain presents that route-delivery row as a USD estimate instead of mislabeling it as the payout asset,
	*	Ledger & Compliance, Scoped Compliance Receipt, PDF, and Verify never attribute that shared route cost to one disclosed payout lane.

Execute labels are intentional too:
	*	`Recipient Total` is banned because it conflates recipient count with payout value,
	*	the main surface uses `Recipient count` and `Total payout` as separate rows,
	*	`Total shielded deduction` stays explicit because it is the balance-impact number,
	*	and the relay-execution info icon appears only for private-relay cross-chain routes because only that mode absorbs shared route delivery cost into relay pricing.

So V16.9.9 uses:
	*	SUBMITTED_TO_RELAYER after Waku publish,
	*	PENDING_CONFIRMATION when a direct Tempo transaction was broadcast but finality could not yet be confirmed,
	*	SUCCESS only for direct/self execution after confirmed receipt.

40.3 File: apps/web/src/components/Widget.tsx

"use client"

import Link from "next/link"
import { useEffect, useMemo, useState } from "react"
import Papa from "papaparse"
import { formatUnits, isAddress, parseUnits, type Abi } from "viem"
import { useWriteContract } from "wagmi"
import { requireEnv, requirePositiveEid } from "@parly/env-utils"
import { buildCrossChainPayoutOptions } from "@parly/sdk/lz-options"

import { POOL_ABI } from "../lib/abis"
import { tempoPublicClient } from "../lib/public-client"
import { safeApproveExactBrowser } from "../lib/erc20-approval"
import { createFreshNote } from "../lib/note-crypto"
import { useParlySession } from "../lib/parly-session"
import { useRecoveredNotes } from "../lib/useRecoveredNotes"
import { buildParlyTree } from "../lib/merkle"
import { fetchLeaves } from "../lib/indexer"
import {
  fetchOfficialRelayerRegistry,
  type RelayerRegistryEntry,
  type RelayerRegistryResponse
} from "../lib/relayer-registry"
import { addressToField } from "../lib/field"
import { sealForRelayer } from "../lib/relay-crypto"
import { publishToWaku } from "../lib/waku"
import type { CanonicalRelayExecutionPayload } from "../lib/relay-types"

type BatchRow = {
  address: `0x${string}`
  amount: string
  destEid: number
}

type UiStatus =
  | "IDLE"
  | "PROVING"
  | "BROADCAST"
  | "PENDING_CONFIRMATION"
  | "SUBMITTED_TO_RELAYER"
  | "SUCCESS"

type AppTab = "SHIELD" | "EXECUTE"

type DirectTempoOutcome =
  | { kind: "success"; hash: `0x${string}` }
  | { kind: "terminal_failure"; hash: `0x${string}`; message: string }
  | { kind: "uncertain"; hash: `0x${string}`; message: string }

type PreparedExecutionRow = {
  address: `0x${string}`
  payoutAmountRaw: bigint
  protocolFeeRaw: bigint
  enteredAmountRaw: bigint
  destEid: number
  lzOptions: `0x${string}`
}

function sanitizeAmount(value: string): string {
  return value.match(/^\d+(?:\.\d{0,6})?/)?.[0] || "0"
}

function zeroAddress(): `0x${string}` {
  return "0x0000000000000000000000000000000000000000"
}

function parseBrowserNonNegativeInt(raw: string | undefined, fallback: number, name: string): number {
  const value = Number(raw ?? String(fallback))
  if (!Number.isFinite(value) || !Number.isInteger(value) || value < 0) {
    console.warn(`${name} invalid, using fallback ${fallback}`)
    return fallback
  }
  return value
}

function parsePositiveBigInt(raw: string | undefined, fallback: bigint): bigint {
  try {
    const value = BigInt(raw || fallback.toString())
    return value > 0n ? value : fallback
  } catch {
    return fallback
  }
}

function maxBigInt(a: bigint, b: bigint) {
  return a > b ? a : b
}

function protocolFeeFromPayout(payoutAmountRaw: bigint, feeBps: bigint) {
  return (payoutAmountRaw * feeBps) / 10_000n
}

function feeTokenRawToUsd6(raw: bigint, decimals: bigint, usd6PerToken: bigint) {
  if (raw === 0n || usd6PerToken === 0n) return 0n
  return (raw * usd6PerToken) / (10n ** decimals)
}

function recommendedExecutorFeeUsd6(args: {
  sameChainOutputCount: number
  crossChainOutputCount: number
  totalQuotedCrossChainFeeUsd6: bigint
}) {
  const sameCount = BigInt(args.sameChainOutputCount)
  const crossCount = BigInt(args.crossChainOutputCount)

  if (args.crossChainOutputCount === 0) {
    return 100_000n + (sameCount > 0n ? (sameCount - 1n) * 20_000n : 0n)
  }

  const requiredMarginUsd6 = maxBigInt(
    350_000n,
    args.totalQuotedCrossChainFeeUsd6 / 5n + crossCount * 50_000n + sameCount * 20_000n
  )
  return args.totalQuotedCrossChainFeeUsd6 + requiredMarginUsd6
}

function routeChipLabel(args: {
  activeOutputCount: number
  sameChainOutputCount: number
  crossChainOutputCount: number
}) {
  if (args.activeOutputCount === 0 || args.crossChainOutputCount === 0) {
    return "Same-Chain Tempo"
  }
  if (args.sameChainOutputCount === 0) {
    return "Cross-Chain Route"
  }
  return "Mixed Route"
}

function formatAssetText(value: bigint, asset: "USDC" | "USDT") {
  return `${formatUnits(value, 6)} ${asset}`
}

function formatUsdEstimateText(value: bigint) {
  return `${formatUnits(value, 6)} USD est.`
}

const TEMPO_LOCAL_EID = requirePositiveEid("NEXT_PUBLIC_TEMPO_LZ_EID")
const ETHEREUM_PAYOUT_EID = requirePositiveEid("NEXT_PUBLIC_LZ_EID_ETHEREUM")
const ARBITRUM_PAYOUT_EID = requirePositiveEid("NEXT_PUBLIC_LZ_EID_ARBITRUM")
const BASE_PAYOUT_EID = requirePositiveEid("NEXT_PUBLIC_LZ_EID_BASE")
const BNB_PAYOUT_EID = requirePositiveEid("NEXT_PUBLIC_LZ_EID_BNB")
const LZ_FEE_TOKEN_DECIMALS = BigInt(requireEnv("NEXT_PUBLIC_TEMPO_LZ_FEE_TOKEN_DECIMALS"))
const LZ_FEE_TOKEN_USD_6DEC = BigInt(requireEnv("NEXT_PUBLIC_TEMPO_LZ_FEE_TOKEN_USD_6DEC"))
const USDC_POOL_ADDRESS = requireEnv("NEXT_PUBLIC_USDC_POOL") as `0x${string}`
const USDT_POOL_ADDRESS = requireEnv("NEXT_PUBLIC_USDT_POOL") as `0x${string}`
const USDC_TOKEN_ADDRESS = requireEnv("NEXT_PUBLIC_USDC_ADDRESS") as `0x${string}`
const USDT_TOKEN_ADDRESS = requireEnv("NEXT_PUBLIC_USDT_ADDRESS") as `0x${string}`

export default function Widget() {
  const session = useParlySession()
  const { writeContractAsync } = useWriteContract()

  const [tab, setTab] = useState<AppTab>("SHIELD")
  const [relayerDirectory, setRelayerDirectory] = useState<RelayerRegistryResponse | null>(null)
  const [relayerDirectoryError, setRelayerDirectoryError] = useState<string | null>(null)

  const [asset, setAsset] = useState<"USDC" | "USDT">("USDC")
  const [depositAmount, setDepositAmount] = useState("")
  const [recipient, setRecipient] = useState("")
  const [sendAmount, setSendAmount] = useState("")
  const [destEid, setDestEid] = useState<number>(TEMPO_LOCAL_EID)
  const [privateRelay, setPrivateRelay] = useState(true)
  const [mode, setMode] = useState<"SINGLE" | "BATCH">("SINGLE")
  const [csvRows, setCsvRows] = useState<BatchRow[]>([])
  const [draftRows, setDraftRows] = useState<BatchRow[]>([
    {
      address: "" as `0x${string}`,
      amount: "",
      destEid: TEMPO_LOCAL_EID
    }
  ])
  const [selectedCommitment, setSelectedCommitment] = useState("")
  const [status, setStatus] = useState<UiStatus>("IDLE")
  const [error, setError] = useState<string | null>(null)
  const [pendingHash, setPendingHash] = useState<`0x${string}` | null>(null)
  const [lastAction, setLastAction] = useState<"shield" | "execute" | null>(null)
  const [showAdvancedSettings, setShowAdvancedSettings] = useState(false)
  const [showFeeBreakdown, setShowFeeBreakdown] = useState(false)
  const [globalFeeBps, setGlobalFeeBps] = useState<bigint>(50n)
  const [executeQuoteLoading, setExecuteQuoteLoading] = useState(false)
  const [quotedCrossChainFeeUsd6, setQuotedCrossChainFeeUsd6] = useState<bigint>(0n)
  const [executeQuoteError, setExecuteQuoteError] = useState<string | null>(null)

  const assetId: 1 | 2 = asset === "USDC" ? 1 : 2
  const pool =
    asset === "USDC"
      ? USDC_POOL_ADDRESS
      : USDT_POOL_ADDRESS
  const token =
    asset === "USDC"
      ? USDC_TOKEN_ADDRESS
      : USDT_TOKEN_ADDRESS
  const localEid = TEMPO_LOCAL_EID
  const browserReceiptTimeoutMs = parseBrowserNonNegativeInt(
    process.env.NEXT_PUBLIC_BROWSER_RECEIPT_TIMEOUT_MS,
    300000,
    "NEXT_PUBLIC_BROWSER_RECEIPT_TIMEOUT_MS"
  )
  const shieldMiningDivisor = parsePositiveBigInt(
    process.env.NEXT_PUBLIC_SHIELD_MINING_POINTS_DIVISOR,
    100n
  )
  const destinationOptions = [
    { label: "Tempo settlement", eid: localEid },
    { label: "Ethereum payout", eid: ETHEREUM_PAYOUT_EID },
    { label: "Arbitrum payout", eid: ARBITRUM_PAYOUT_EID },
    { label: "Base payout", eid: BASE_PAYOUT_EID },
    { label: "BNB payout", eid: BNB_PAYOUT_EID }
  ]

  const { notes, loading: recovering } = useRecoveredNotes({
    address: session.address,
    assetId,
    recoveryPublicKeyB64: session.recoveryPublicKeyB64,
    recoveryPrivateKeyB64: session.recoveryPrivateKeyB64,
    optionalCacheKey: session.optionalCacheKey,
    refreshNonce: session.recoveryEpoch
  })

  const liveNotes = useMemo(() => notes.filter((n) => n.status === "live"), [notes])

  const activeNote = useMemo(() => {
    return (
      liveNotes.find((n) => n.commitment.toLowerCase() === selectedCommitment.toLowerCase()) ||
      liveNotes[0] ||
      null
    )
  }, [liveNotes, selectedCommitment])

  const shieldedBalance = useMemo(() => {
    return liveNotes.reduce((sum, n) => sum + BigInt(n.amount), 0n)
  }, [liveNotes])
  const parsedDepositUnits = useMemo(() => {
    try {
      return parseUnits(sanitizeAmount(depositAmount), 6)
    } catch {
      return 0n
    }
  }, [depositAmount])
  const estimatedShieldPoints = useMemo(() => {
    if (parsedDepositUnits <= 0n) return 0n
    return parsedDepositUnits / shieldMiningDivisor
  }, [parsedDepositUnits, shieldMiningDivisor])

  const relayers = useMemo(() => relayerDirectory?.items || [], [relayerDirectory])
  const activeRelayerEntry = useMemo<RelayerRegistryEntry | null>(() => relayers[0] || null, [relayers])
  const busy =
    status === "PROVING" ||
    status === "BROADCAST" ||
    status === "PENDING_CONFIRMATION" ||
    status === "SUBMITTED_TO_RELAYER"

  useEffect(() => {
    let cancelled = false

    fetchOfficialRelayerRegistry()
      .then((directory) => {
        if (cancelled) return
        setRelayerDirectory(directory)
        setRelayerDirectoryError(null)
      })
      .catch((nextError: any) => {
        if (cancelled) return
        setRelayerDirectory({
          items: [],
          liveCount: 0,
          maxCount: 10,
          registryOpen: true
        })
        setRelayerDirectoryError(nextError?.message || "Official relayer registry unavailable")
      })

    return () => {
      cancelled = true
    }
  }, [])

  useEffect(() => {
    session.setRecoveryActivity("widget", recovering)
    return () => {
      session.setRecoveryActivity("widget", false)
    }
  }, [recovering, session])

  useEffect(() => {
    setSelectedCommitment("")
    setCsvRows([])
    setDraftRows([{ address: "" as `0x${string}`, amount: "", destEid: localEid }])
    setError(null)
    setStatus("IDLE")
    setPendingHash(null)
    setShowFeeBreakdown(false)
    setShowAdvancedSettings(false)
    setQuotedCrossChainFeeUsd6(0n)
    setExecuteQuoteError(null)
  }, [asset, localEid])

  useEffect(() => {
    if (session.phase === "ready") return

    setSelectedCommitment("")
    setPendingHash(null)
    setError(null)
    setStatus("IDLE")
    setLastAction(null)
    setShowFeeBreakdown(false)
  }, [session.phase, session.address, session.authenticatedAddress])

  useEffect(() => {
    if (!privateRelay || !relayerDirectoryError) return
    setError((current) => current || relayerDirectoryError)
  }, [privateRelay, relayerDirectoryError])

  useEffect(() => {
    let cancelled = false

    tempoPublicClient
      .readContract({
        address: pool as `0x${string}`,
        abi: POOL_ABI,
        functionName: "globalFeeBps"
      })
      .then((value) => {
        if (!cancelled) setGlobalFeeBps(value as bigint)
      })
      .catch((nextError: any) => {
        if (!cancelled) {
          console.error(`Failed to read globalFeeBps: ${nextError?.message || nextError}`)
        }
      })

    return () => {
      cancelled = true
    }
  }, [pool])

  const previewInputRows = useMemo(() => {
    const candidateRows =
      mode === "SINGLE"
        ? [{ address: recipient as `0x${string}`, amount: sendAmount, destEid }]
        : draftRows.filter((row) => row.address || row.amount)

    return candidateRows
      .map((row) => {
        try {
          const payoutAmountRaw = parseUnits(sanitizeAmount(row.amount), 6)
          if (!isAddress(row.address) || row.address.toLowerCase() === zeroAddress() || payoutAmountRaw <= 0n) {
            return null
          }

          const protocolFeeRaw = protocolFeeFromPayout(payoutAmountRaw, globalFeeBps)
          const nextDestEid = row.destEid || localEid

          return {
            address: row.address,
            payoutAmountRaw,
            protocolFeeRaw,
            enteredAmountRaw: payoutAmountRaw + protocolFeeRaw,
            destEid: nextDestEid,
            lzOptions: nextDestEid === localEid ? ("0x" as `0x${string}`) : buildCrossChainPayoutOptions(250000, 0)
          } satisfies PreparedExecutionRow
        } catch {
          return null
        }
      })
      .filter(Boolean) as PreparedExecutionRow[]
  }, [destEid, draftRows, globalFeeBps, localEid, mode, recipient, sendAmount])

  const sameChainRows = useMemo(
    () => previewInputRows.filter((row) => row.destEid === localEid),
    [localEid, previewInputRows]
  )
  const crossChainRows = useMemo(
    () => previewInputRows.filter((row) => row.destEid !== localEid),
    [localEid, previewInputRows]
  )
  const activeOutputCount = previewInputRows.length
  const sameChainOutputCount = sameChainRows.length
  const crossChainOutputCount = crossChainRows.length
  const hasCrossChain = crossChainOutputCount > 0
  const routeChip = routeChipLabel({
    activeOutputCount,
    sameChainOutputCount,
    crossChainOutputCount
  })
  const totalPayoutRaw = previewInputRows.reduce((sum, row) => sum + row.payoutAmountRaw, 0n)
  const totalProtocolFeeRaw = previewInputRows.reduce((sum, row) => sum + row.protocolFeeRaw, 0n)
  const totalEnteredAmountRaw = previewInputRows.reduce((sum, row) => sum + row.enteredAmountRaw, 0n)
  const recommendedExecutorFeeRaw = useMemo(
    () =>
      privateRelay
        ? recommendedExecutorFeeUsd6({
            sameChainOutputCount,
            crossChainOutputCount,
            totalQuotedCrossChainFeeUsd6: quotedCrossChainFeeUsd6
          })
        : 0n,
    [crossChainOutputCount, privateRelay, quotedCrossChainFeeUsd6, sameChainOutputCount]
  )
  const displayedRelayExecutionFeeRaw = privateRelay ? recommendedExecutorFeeRaw : 0n
  const displayedCrossChainDeliveryFeeUsd6 = hasCrossChain ? quotedCrossChainFeeUsd6 : 0n
  const displayedTotalFeeRaw = privateRelay
    ? totalProtocolFeeRaw + displayedRelayExecutionFeeRaw
    : totalProtocolFeeRaw + displayedCrossChainDeliveryFeeUsd6
  const displayedTotalShieldedDeduction = privateRelay
    ? totalEnteredAmountRaw + displayedRelayExecutionFeeRaw
    : totalEnteredAmountRaw
  const displayedProtocolFeeText = formatAssetText(totalProtocolFeeRaw, asset)
  const displayedRelayExecutionFeeText = privateRelay
    ? formatAssetText(displayedRelayExecutionFeeRaw, asset)
    : formatAssetText(0n, asset)
  const displayedCrossChainDeliveryFeeText = formatUsdEstimateText(
    displayedCrossChainDeliveryFeeUsd6
  )
  const displayedTotalFeeText =
    !privateRelay && hasCrossChain
      ? `${displayedProtocolFeeText} + ${displayedCrossChainDeliveryFeeText}`
      : formatAssetText(displayedTotalFeeRaw, asset)
  const displayedTotalShieldedDeductionText = formatAssetText(
    displayedTotalShieldedDeduction,
    asset
  )

  useEffect(() => {
    if (tab !== "EXECUTE") return
    if (!hasCrossChain) {
      setQuotedCrossChainFeeUsd6(0n)
      setExecuteQuoteError(null)
      setExecuteQuoteLoading(false)
      return
    }

    let cancelled = false
    setExecuteQuoteLoading(true)
    setExecuteQuoteError(null)

    Promise.all(
      crossChainRows.map((row) =>
        tempoPublicClient.readContract({
          address: pool as `0x${string}`,
          abi: POOL_ABI,
          functionName: "quoteCrossChainFee",
          args: [row.destEid, row.address, row.payoutAmountRaw, row.lzOptions]
        })
      )
    )
      .then((fees) => {
        if (cancelled) return
        const totalRaw = (fees as bigint[]).reduce((sum, fee) => sum + fee, 0n)
        setQuotedCrossChainFeeUsd6(
          feeTokenRawToUsd6(totalRaw, LZ_FEE_TOKEN_DECIMALS, LZ_FEE_TOKEN_USD_6DEC)
        )
      })
      .catch((nextError: any) => {
        if (cancelled) return
        setQuotedCrossChainFeeUsd6(0n)
        setExecuteQuoteError(nextError?.message || "Failed to quote cross-chain delivery cost.")
      })
      .finally(() => {
        if (!cancelled) setExecuteQuoteLoading(false)
      })

    return () => {
      cancelled = true
    }
  }, [crossChainRows, hasCrossChain, pool, tab])

  async function writeAndWait(args: {
    address: `0x${string}`
    abi: Abi
    functionName: string
    args?: readonly unknown[]
  }) {
    const hash = await writeContractAsync(args as any)
    await tempoPublicClient.waitForTransactionReceipt({ hash })
    return hash
  }

  async function waitForDirectTempoFinality(args: {
    address: `0x${string}`
    abi: Abi
    functionName: string
    args?: readonly unknown[]
  }): Promise<DirectTempoOutcome> {
    let submittedHash: `0x${string}` | null = null
    let finalHash: `0x${string}` | null = null
    let replacementReason: "replaced" | "repriced" | "cancelled" | null = null

    try {
      submittedHash = await writeContractAsync(args as any)
      finalHash = submittedHash

      const receipt = await tempoPublicClient.waitForTransactionReceipt({
        hash: submittedHash,
        timeout: browserReceiptTimeoutMs,
        onReplaced: (replacement) => {
          replacementReason = replacement.reason
          finalHash = replacement.transactionReceipt.transactionHash as `0x${string}`
        }
      })

      if (replacementReason === "cancelled") {
        return {
          kind: "terminal_failure",
          hash: finalHash!,
          message: "Transaction cancelled before inclusion."
        }
      }

      if (replacementReason === "replaced") {
        return {
          kind: "terminal_failure",
          hash: finalHash!,
          message: "Transaction replaced before inclusion."
        }
      }

      if (receipt.status !== "success") {
        return {
          kind: "terminal_failure",
          hash: finalHash!,
          message: "Transaction reverted on chain."
        }
      }

      return { kind: "success", hash: finalHash! }
    } catch (e: any) {
      if (!submittedHash) {
        throw e
      }

      return {
        kind: "uncertain",
        hash: finalHash || submittedHash,
        message: e?.message || "Finality could not be confirmed."
      }
    }
  }

  async function cleanupBrowserApproval(
    approvalToken: `0x${string}`,
    approvalSpender: `0x${string}`
  ) {
    if (!session.address) return

    await safeApproveExactBrowser({
      token: approvalToken,
      spender: approvalSpender,
      amount: 0n,
      owner: session.address,
      writeAndWait
    })
  }

  async function runTempoActionWithOptionalApproval(args: {
    approvalToken?: `0x${string}` | null
    approvalSpender?: `0x${string}` | null
    approvalAmount?: bigint
    action: () => Promise<DirectTempoOutcome>
  }) {
    let shouldCleanupApproval = false
    let pendingError: Error | null = null

    try {
      if (
        session.address &&
        args.approvalToken &&
        args.approvalSpender &&
        (args.approvalAmount || 0n) > 0n
      ) {
        await safeApproveExactBrowser({
          token: args.approvalToken,
          spender: args.approvalSpender,
          amount: args.approvalAmount!,
          owner: session.address,
          writeAndWait
        })
      }

      const outcome = await args.action()

      if (outcome.kind === "success") {
        setPendingHash(null)
        return outcome.hash
      }

      if (outcome.kind === "uncertain") {
        setPendingHash(outcome.hash)
        setStatus("PENDING_CONFIRMATION")
        setError(
          `Transaction ${outcome.hash} was broadcast, but finality could not be confirmed yet. Check Tempo explorer before retrying or manually revoking approvals.`
        )
        return null
      }

      shouldCleanupApproval = true
      throw new Error(`${outcome.message} ${outcome.hash}`)
    } catch (e: any) {
      shouldCleanupApproval = true
      pendingError = e instanceof Error ? e : new Error(String(e))
    }

    if (shouldCleanupApproval && args.approvalToken && args.approvalSpender) {
      try {
        await cleanupBrowserApproval(args.approvalToken, args.approvalSpender)
      } catch (cleanupError: any) {
        const cleanupMessage =
          `Approval cleanup also failed. Manually revoke allowance. ${cleanupError.message || cleanupError}`
        console.error(`Browser approval cleanup error: ${cleanupMessage}`)
        pendingError = pendingError
          ? new Error(`${pendingError.message} ${cleanupMessage}`)
          : new Error(cleanupMessage)
      }
    }

    if (pendingError) {
      throw pendingError
    }
  }

  async function shieldOnTempo() {
    try {
      setError(null)
      setPendingHash(null)
      setLastAction("shield")

      if (!session.address || !session.recoveryPublicKeyB64) throw new Error("Authenticate first.")

      const amount = parseUnits(sanitizeAmount(depositAmount), 6)
      if (amount <= 0n) throw new Error("Invalid deposit amount.")

      const fresh = await createFreshNote(
        session.recoveryPublicKeyB64,
        session.address,
        assetId,
        amount.toString(),
        "deposit"
      )

      setStatus("BROADCAST")

      const finalHash = await runTempoActionWithOptionalApproval({
        approvalToken: token as `0x${string}`,
        approvalSpender: pool as `0x${string}`,
        approvalAmount: amount,
        action: () =>
          waitForDirectTempoFinality({
            address: pool as `0x${string}`,
            abi: POOL_ABI,
            functionName: "deposit",
            args: [BigInt(fresh.innerCommitment), amount, addressToField(session.address), fresh.envelope]
          })
      })

      if (!finalHash) return

      setDepositAmount("")
      session.refreshRecovery()
      setStatus("SUCCESS")
    } catch (e: any) {
      setError(e.message || "Deposit failed")
      setStatus("IDLE")
    }
  }

  function parseCsv(file: File) {
    Papa.parse(file, {
      header: true,
      skipEmptyLines: true,
      complete: (results) => {
        const rows = (results.data as any[])
          .map((row) => ({
            address: String(row.address || "").trim() as `0x${string}`,
            amount: String(row.amount || "").trim(),
            destEid: Number(row.destEid || localEid)
          }))
          .filter((row) => row.address && row.amount)

        setCsvRows(rows)
        setDraftRows(rows.slice(0, 10))
        setMode("BATCH")
      }
    })
  }

  function updateDraftRow(index: number, patch: Partial<BatchRow>) {
    setDraftRows((current) =>
      current.map((row, rowIndex) => (rowIndex === index ? { ...row, ...patch } : row))
    )
  }

  function addDraftRow() {
    setDraftRows((current) => {
      if (current.length >= 10) return current
      return [...current, { address: "" as `0x${string}`, amount: "", destEid: localEid }]
    })
  }

  function removeDraftRow(index: number) {
    setDraftRows((current) => {
      const next = current.filter((_, rowIndex) => rowIndex !== index)
      return next.length ? next : [{ address: "" as `0x${string}`, amount: "", destEid: localEid }]
    })
  }

  function validateRows(rows: BatchRow[]) {
    if (rows.length === 0) throw new Error("No outputs provided.")
    if (rows.length > 10) throw new Error("One proof supports max 10 outputs.")

    for (const row of rows) {
      if (!isAddress(row.address)) {
        throw new Error(`Invalid address: ${row.address}`)
      }
      if (row.address.toLowerCase() === zeroAddress()) {
        throw new Error("Zero address recipient not allowed.")
      }
      const amt = parseUnits(sanitizeAmount(row.amount), 6)
      if (amt <= 0n) {
        throw new Error(`Invalid amount for ${row.address}`)
      }
    }
  }

  async function buildRows(): Promise<BatchRow[]> {
    if (mode === "SINGLE") {
      return [
        {
          address: recipient as `0x${string}`,
          amount: sendAmount,
          destEid
        }
      ]
    }
    return draftRows.filter((row) => row.address || row.amount)
  }

  async function execute() {
    try {
      setError(null)
      setPendingHash(null)
      setLastAction("execute")

      if (!session.address || !session.recoveryPublicKeyB64 || !activeNote) {
        throw new Error("No active note.")
      }

      if (privateRelay && !activeRelayerEntry) {
        throw new Error("No relayer configured.")
      }

      const rows = await buildRows()
      validateRows(rows)

      const feeBps = await tempoPublicClient.readContract({
        address: pool as `0x${string}`,
        abi: POOL_ABI,
        functionName: "globalFeeBps"
      }) as bigint

      const preparedRows = rows.map((row) => {
        const payoutAmountRaw = parseUnits(sanitizeAmount(row.amount), 6)
        const protocolFeeRaw = protocolFeeFromPayout(payoutAmountRaw, feeBps)
        const nextDestEid = row.destEid || localEid

        return {
          address: row.address,
          payoutAmountRaw,
          protocolFeeRaw,
          enteredAmountRaw: payoutAmountRaw + protocolFeeRaw,
          destEid: nextDestEid,
          lzOptions: nextDestEid === localEid ? ("0x" as `0x${string}`) : buildCrossChainPayoutOptions(250000, 0)
        } satisfies PreparedExecutionRow
      })

      const sameChainOutputCount = preparedRows.filter((row) => row.destEid === localEid).length
      const crossChainRows = preparedRows.filter((row) => row.destEid !== localEid)
      const crossChainOutputCount = crossChainRows.length

      let totalCrossChainFeeRaw = 0n
      for (const row of crossChainRows) {
        const fee = await tempoPublicClient.readContract({
          address: pool as `0x${string}`,
          abi: POOL_ABI,
          functionName: "quoteCrossChainFee",
          args: [row.destEid, row.address, row.payoutAmountRaw, row.lzOptions]
        }) as bigint
        totalCrossChainFeeRaw += fee
      }

      const totalCrossChainFeeUsd6 = feeTokenRawToUsd6(
        totalCrossChainFeeRaw,
        LZ_FEE_TOKEN_DECIMALS,
        LZ_FEE_TOKEN_USD_6DEC
      )
      const executorFee = privateRelay
        ? recommendedExecutorFeeUsd6({
            sameChainOutputCount,
            crossChainOutputCount,
            totalQuotedCrossChainFeeUsd6: totalCrossChainFeeUsd6
          })
        : 0n

      const leaves = await fetchLeaves(assetId)
      const tree = await buildParlyTree(leaves.map((x: any) => String(x.commitment)))

      const leafIndex = leaves.findIndex(
        (x: any) => String(x.commitment).toLowerCase() === activeNote.commitment.toLowerCase()
      )

      if (leafIndex === -1) {
        throw new Error("Selected note not found in Merkle tree.")
      }

      const { pathElements, pathIndices } = tree.path(leafIndex)

      const paddedRecipients = Array<`0x${string}`>(10).fill(zeroAddress())
      const paddedAmounts = Array<bigint>(10).fill(0n)
      const paddedDestEids = Array<number>(10).fill(localEid)
      const paddedOptions = Array<`0x${string}`>(10).fill("0x")

      for (let i = 0; i < preparedRows.length; i++) {
        paddedRecipients[i] = preparedRows[i].address
        paddedAmounts[i] = preparedRows[i].payoutAmountRaw
        paddedDestEids[i] = preparedRows[i].destEid
        paddedOptions[i] = preparedRows[i].lzOptions
      }

      const protocolFee = preparedRows.reduce((sum, row) => sum + row.protocolFeeRaw, 0n)
      const totalOut = paddedAmounts.reduce((a, b) => a + b, 0n)
      const changeAmount = BigInt(activeNote.amount) - totalOut - protocolFee - executorFee

      if (changeAmount < 0n) {
        throw new Error("Insufficient balance.")
      }

      let newChangeCommitment =
        "0x0000000000000000000000000000000000000000000000000000000000000000" as `0x${string}`
      let newChangeEnvelope = "0x" as `0x${string}`
      let newChangeSecret = "0"
      let newChangeNullifier = "0"

      if (changeAmount > 0n) {
        const freshChange = await createFreshNote(
          session.recoveryPublicKeyB64,
          activeNote.depositor,
          assetId,
          changeAmount.toString(),
          "change"
        )

        newChangeCommitment = freshChange.commitment
        newChangeEnvelope = freshChange.envelope
        newChangeSecret = freshChange.secret
        newChangeNullifier = freshChange.nullifier
      }

      const activeRelayer = privateRelay
        ? activeRelayerEntry!.executionAddress
        : session.address

      const circuitInputs = {
        root: BigInt(tree.root).toString(),
        nullifierHash: BigInt(activeNote.nullifierHash).toString(),
        protocol_fee: protocolFee.toString(),
        executor_fee: executorFee.toString(),
        total_input_amount: activeNote.amount.toString(),
        asset_id: assetId.toString(),
        local_eid: String(localEid),
        recipients: paddedRecipients.map((a) => addressToField(a).toString()),
        amounts: paddedAmounts.map((a) => a.toString()),
        dest_eids: paddedDestEids.map(String),
        new_change_commitment: BigInt(newChangeCommitment).toString(),
        relayer: addressToField(activeRelayer).toString(),
        secret: activeNote.secret,
        nullifier: activeNote.nullifier,
        depositor: addressToField(activeNote.depositor).toString(),
        pathElements: pathElements.map((x: string | bigint) => BigInt(x).toString()),
        pathIndices,
        new_change_secret: newChangeSecret,
        new_change_nullifier: newChangeNullifier
      }

      setStatus("PROVING")

      const worker = new Worker(new URL("../lib/prover.worker.ts", import.meta.url))
      const proofBundle = await new Promise<any>((resolve, reject) => {
        worker.postMessage({ circuitInputs })

        worker.onmessage = (e) => {
          worker.terminate()
          if (e.data.success) resolve(e.data)
          else reject(new Error(e.data.error))
        }

        worker.onerror = (err) => {
          worker.terminate()
          reject(err)
        }
      })

      if (!Array.isArray(proofBundle.publicSignals) || proofBundle.publicSignals.length !== 50) {
        throw new Error("Proof public signals length mismatch.")
      }

      setStatus("BROADCAST")

      if (privateRelay) {
        const canonicalPayload: CanonicalRelayExecutionPayload = {
          pool: pool as `0x${string}`,
          pA: proofBundle.pA,
          pB: proofBundle.pB,
          pC: proofBundle.pC,
          pubSignals: proofBundle.publicSignals,
          recipients: paddedRecipients,
          amounts: paddedAmounts.map((x) => x.toString()),
          destEids: paddedDestEids,
          newChangeCommitment,
          newChangeEnvelope,
          lzOptions: paddedOptions
        }

        const bundle = await sealForRelayer(
          activeRelayerEntry!.executionAddress,
          activeRelayerEntry!.publicKeyB64,
          canonicalPayload
        )

        await publishToWaku(new TextEncoder().encode(JSON.stringify(bundle)))
        setStatus("SUBMITTED_TO_RELAYER")
        setPendingHash(null)
        return
      }

      let approvalToken: `0x${string}` | null = null

      if (totalCrossChainFeeRaw > 0n) {
        approvalToken = await tempoPublicClient.readContract({
          address: pool as `0x${string}`,
          abi: POOL_ABI,
          functionName: "lzFeeToken"
        }) as `0x${string}`
      }

      const finalHash = await runTempoActionWithOptionalApproval({
        approvalToken,
        approvalSpender: pool as `0x${string}`,
        approvalAmount: totalCrossChainFeeRaw,
        action: () =>
          waitForDirectTempoFinality({
            address: pool as `0x${string}`,
            abi: POOL_ABI,
            functionName: "batchWithdraw",
            args: [
              proofBundle.pA,
              proofBundle.pB,
              proofBundle.pC,
              proofBundle.publicSignals.map((x: string) => BigInt(x)),
              paddedRecipients,
              paddedAmounts,
              paddedDestEids,
              newChangeCommitment,
              newChangeEnvelope,
              paddedOptions
            ]
          })
      })

      if (!finalHash) return

      setSelectedCommitment("")
      session.refreshRecovery()
      setStatus("SUCCESS")
    } catch (e: any) {
      setError(e.message || "Execution failed")
      setStatus("IDLE")
    }
  }

  return (
    <div className="mx-auto max-w-5xl rounded-3xl border border-white/10 bg-slate-950/75 p-6 text-white shadow-2xl shadow-cyan-950/10">
      <div className="flex flex-col gap-4 xl:flex-row xl:items-start xl:justify-between">
        <div>
          <p className="text-xs uppercase tracking-[0.25em] text-cyan-300">Tempo app</p>
          <h1 className="mt-3 text-4xl font-black">Private settlement without protocol drift</h1>
          <p className="mt-3 max-w-3xl text-sm text-slate-400">
            Shield on Tempo here, or route to the supported spoke ingress flow when your origin
            network is Ethereum, Arbitrum, Base, or BNB.
          </p>
        </div>

        <div className="flex flex-wrap gap-3">
          {session.phase === "ready" ? (
            <button
              onClick={() => session.logout("Recovery session cleared.")}
              className="rounded-full border border-white/10 px-4 py-3 text-sm font-semibold text-slate-200 transition hover:border-white/30 hover:text-white"
            >
              Clear session
            </button>
          ) : (
            <button
              className="rounded-full bg-cyan-400 px-4 py-3 text-sm font-semibold text-slate-950 transition hover:bg-cyan-300"
              onClick={session.authenticate}
            >
              {session.phase === "authenticating" ? "Authenticating..." : "Authenticate wallet"}
            </button>
          )}
          <Link
            href="/spoke-shield"
            className="rounded-full border border-white/10 px-4 py-3 text-sm font-semibold text-slate-200 transition hover:border-white/30 hover:text-white"
          >
            Supported spoke deposit
          </Link>
        </div>
      </div>

      {session.error && (
        <div className="mt-6 rounded-2xl border border-amber-500/40 bg-amber-500/10 p-4 text-sm text-amber-100">
          {session.error}
        </div>
      )}

      {session.phase !== "ready" ? (
        <div className="mt-6 grid gap-4 xl:grid-cols-[1.45fr_0.95fr]">
          <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-5">
            <p className="text-xs uppercase tracking-[0.22em] text-slate-500">Session state</p>
            <h2 className="mt-3 text-2xl font-black text-white">
              {session.phase === "disconnected" && "Connect a wallet to start"}
              {session.phase === "connected_unauthed" && "Authenticate this wallet for deterministic recovery"}
              {session.phase === "authenticating" && "Waiting for the deterministic Parly signature"}
              {session.phase === "recovering_notes" && "Recovering notes from indexed chain history"}
            </h2>
            <p className="mt-3 max-w-2xl text-sm text-slate-400">
              The browser derives one recovery identity from a fixed domain-separated wallet signature and then reconstructs notes from public chain data plus sealed envelopes.
            </p>
          </div>

          <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-5">
            <p className="text-xs uppercase tracking-[0.22em] text-slate-500">Ready after auth</p>
            <ul className="mt-4 space-y-3 text-sm text-slate-300">
              <li>Shield on Tempo with deterministic recovery.</li>
              <li>Build same-chain or cross-chain private payout batches.</li>
              <li>Use a Waku relayer or self-broadcast directly.</li>
              <li>Open history, verification, and compliance tooling honestly.</li>
            </ul>
          </div>
        </div>
      ) : (
        <div className="mt-6 space-y-6">
          <div className="grid gap-4 xl:grid-cols-[1.2fr_1fr_1fr]">
            <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-4">
              <p className="text-xs uppercase tracking-[0.18em] text-slate-500">Recovery public key</p>
              <p className="mt-2 break-all text-xs text-slate-200">{session.recoveryPublicKeyB64}</p>
            </div>
            <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-4">
              <p className="text-xs uppercase tracking-[0.18em] text-slate-500">Shielded balance</p>
              <p className="mt-2 text-3xl font-black">
                {formatUnits(shieldedBalance, 6)} {asset}
              </p>
              <p className="mt-2 text-sm text-slate-400">
                {liveNotes.length} live note{liveNotes.length === 1 ? "" : "s"} recovered for this asset
              </p>
              {recovering && <p className="mt-2 text-xs text-cyan-300">Recovering notes from chain...</p>}
            </div>
            <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-4">
              <p className="text-xs uppercase tracking-[0.18em] text-slate-500">Selected note</p>
              {activeNote ? (
                <>
                  <p className="mt-2 text-2xl font-black">
                    {formatUnits(BigInt(activeNote.amount), 6)} {asset}
                  </p>
                  <p className="mt-2 text-xs text-slate-400">{activeNote.kind} note</p>
                  <p className="mt-2 break-all text-[11px] text-slate-500">{activeNote.commitment}</p>
                </>
              ) : (
                <p className="mt-2 text-sm text-slate-400">No live note recovered for this asset yet.</p>
              )}
            </div>
          </div>

          <div className="flex flex-wrap items-center gap-3">
            <div className="inline-flex rounded-full border border-white/10 bg-white/[0.03] p-1">
              {(["USDC", "USDT"] as const).map((nextAsset) => (
                <button
                  key={nextAsset}
                  onClick={() => setAsset(nextAsset)}
                  className={`rounded-full px-4 py-2 text-sm font-semibold transition ${
                    asset === nextAsset
                      ? nextAsset === "USDC"
                        ? "bg-blue-500 text-white"
                        : "bg-emerald-600 text-white"
                      : "text-slate-300 hover:bg-white/5 hover:text-white"
                  }`}
                >
                  {nextAsset}
                </button>
              ))}
            </div>

            <div className="inline-flex rounded-full border border-white/10 bg-white/[0.03] p-1">
              {(["SHIELD", "EXECUTE"] as const).map((nextTab) => (
                <button
                  key={nextTab}
                  onClick={() => setTab(nextTab)}
                  className={`rounded-full px-4 py-2 text-sm font-semibold transition ${
                    tab === nextTab
                      ? "bg-white text-slate-950"
                      : "text-slate-300 hover:bg-white/5 hover:text-white"
                  }`}
                >
                  {nextTab === "SHIELD" ? "Shield" : "Execute"}
                </button>
              ))}
            </div>

            <Link
              href="/history"
              className="rounded-full border border-white/10 px-4 py-2 text-sm font-semibold text-slate-200 transition hover:border-white/30 hover:text-white"
            >
              Parly Ledger & Compliance
            </Link>
            <Link
              href="/verify"
              className="rounded-full border border-white/10 px-4 py-2 text-sm font-semibold text-slate-200 transition hover:border-white/30 hover:text-white"
            >
              Verify a Parly Payout Scope
            </Link>
          </div>

          {liveNotes.length > 0 && (
            <select
              value={activeNote?.commitment || ""}
              onChange={(e) => setSelectedCommitment(e.target.value)}
              className="w-full rounded-2xl border border-white/10 bg-slate-950/70 p-4 text-sm outline-none transition focus:border-cyan-400/60"
            >
              {liveNotes.map((n) => (
                <option key={n.commitment} value={n.commitment}>
                  {formatUnits(BigInt(n.amount), 6)} {asset} | {n.kind} | {n.commitment.slice(0, 12)}...
                </option>
              ))}
            </select>
          )}

          {tab === "SHIELD" ? (
            <div className="grid gap-4 xl:grid-cols-[1.2fr_0.8fr]">
              <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-5">
                <div className="flex items-center justify-between gap-3">
                  <div>
                    <p className="text-xs uppercase tracking-[0.22em] text-slate-500">Shield on Tempo</p>
                    <h3 className="mt-2 text-2xl font-black">Deposit and recover deterministically</h3>
                  </div>
                  <div className="rounded-full border border-cyan-400/30 bg-cyan-500/10 px-3 py-2 text-[11px] font-semibold uppercase tracking-[0.18em] text-cyan-100">
                    Stablecoin only
                  </div>
                </div>

                <div className="mt-5">
                  <label className="mb-2 block text-xs font-semibold uppercase tracking-[0.18em] text-slate-500">
                    Deposit amount
                  </label>
                  <input
                    className="w-full rounded-2xl border border-white/10 bg-slate-950/70 p-4 text-base outline-none transition focus:border-cyan-400/60"
                    placeholder="0.00"
                    value={depositAmount}
                    onChange={(e) => setDepositAmount(sanitizeAmount(e.target.value))}
                  />
                </div>

                <div className="mt-4 flex flex-wrap gap-2">
                  {["100", "250", "500", "1000"].map((preset) => (
                    <button
                      key={preset}
                      onClick={() => setDepositAmount(preset)}
                      className="rounded-full border border-white/10 px-3 py-2 text-xs font-semibold text-slate-300 transition hover:border-white/30 hover:text-white"
                    >
                      {preset} {asset}
                    </button>
                  ))}
                </div>

                <button
                  className="mt-5 w-full rounded-2xl bg-cyan-400 py-4 font-bold text-slate-950 transition hover:bg-cyan-300 disabled:opacity-50"
                  onClick={shieldOnTempo}
                  disabled={busy || parsedDepositUnits <= 0n}
                >
                  {status === "BROADCAST" && lastAction === "shield"
                    ? "Broadcasting Tempo deposit..."
                    : "Shield on Tempo"}
                </button>
              </div>

              <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-5">
                <p className="text-xs uppercase tracking-[0.22em] text-slate-500">Deposit preview</p>
                <div className="mt-4 space-y-4">
                  <div>
                    <div className="text-sm text-slate-400">Selected asset</div>
                    <div className="mt-1 text-xl font-bold text-white">{asset}</div>
                  </div>
                  <div>
                    <div className="text-sm text-slate-400">Estimated shield mining points</div>
                    <div className="mt-1 text-2xl font-black text-emerald-300">
                      {estimatedShieldPoints.toString()}
                    </div>
                    <div className="mt-1 text-xs text-slate-500">
                      Derived from the configured points divisor and the current deposit input.
                    </div>
                  </div>
                  <div>
                    <div className="text-sm text-slate-400">Recovery model</div>
                    <div className="mt-1 text-sm text-slate-300">
                      The app rebuilds note state from indexed leaves plus encrypted envelopes addressed to this recovery key.
                    </div>
                  </div>
                </div>
              </div>
            </div>
          ) : (
            <div className="space-y-5">
              <div className="grid gap-4 xl:grid-cols-[1.25fr_0.75fr]">
                <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-5">
                  <div className="flex flex-wrap items-center justify-between gap-3">
                    <div>
                      <p className="text-xs uppercase tracking-[0.22em] text-slate-500">Execution builder</p>
                      <h3 className="mt-2 text-2xl font-black">Private payout batch builder</h3>
                    </div>
                    <div className="rounded-full border border-white/10 bg-white/[0.04] px-3 py-2 text-[11px] font-semibold uppercase tracking-[0.18em] text-slate-300">
                      Max 10 outputs
                    </div>
                  </div>

                  <div className="mt-5 inline-flex rounded-full border border-white/10 bg-white/[0.03] p-1">
                    <button
                      onClick={() => setMode("SINGLE")}
                      className={`rounded-full px-4 py-2 text-sm font-semibold transition ${
                        mode === "SINGLE"
                          ? "bg-white text-slate-950"
                          : "text-slate-300 hover:bg-white/5 hover:text-white"
                      }`}
                    >
                      Single send
                    </button>
                    <button
                      onClick={() => setMode("BATCH")}
                      className={`rounded-full px-4 py-2 text-sm font-semibold transition ${
                        mode === "BATCH"
                          ? "bg-white text-slate-950"
                          : "text-slate-300 hover:bg-white/5 hover:text-white"
                      }`}
                    >
                      Batch builder
                    </button>
                  </div>

                  {mode === "SINGLE" ? (
                    <div className="mt-5 grid gap-4">
                      <input
                        className="rounded-2xl border border-white/10 bg-slate-950/70 p-4 text-sm outline-none transition focus:border-cyan-400/60"
                        placeholder="Recipient address"
                        value={recipient}
                        onChange={(e) => setRecipient(e.target.value)}
                      />
                      <input
                        className="rounded-2xl border border-white/10 bg-slate-950/70 p-4 text-sm outline-none transition focus:border-cyan-400/60"
                        placeholder="Amount"
                        value={sendAmount}
                        onChange={(e) => setSendAmount(sanitizeAmount(e.target.value))}
                      />
                      <select
                        className="rounded-2xl border border-white/10 bg-slate-950/70 p-4 text-sm outline-none transition focus:border-cyan-400/60"
                        value={destEid}
                        onChange={(e) => setDestEid(Number(e.target.value))}
                      >
                        {destinationOptions.map((option) => (
                          <option key={option.eid} value={option.eid}>
                            {option.label}
                          </option>
                        ))}
                      </select>
                    </div>
                  ) : (
                    <div className="mt-5 space-y-4">
                      <div className="grid gap-3">
                        {draftRows.map((row, index) => (
                          <div
                            key={`${index}-${row.address}-${row.amount}-${row.destEid}`}
                            className="grid gap-3 rounded-2xl border border-white/10 bg-slate-950/50 p-4 xl:grid-cols-[1.2fr_0.7fr_0.8fr_auto]"
                          >
                            <input
                              className="rounded-xl border border-white/10 bg-slate-950/80 p-3 text-sm outline-none transition focus:border-cyan-400/60"
                              placeholder="Recipient address"
                              value={row.address}
                              onChange={(e) =>
                                updateDraftRow(index, { address: e.target.value as `0x${string}` })
                              }
                            />
                            <input
                              className="rounded-xl border border-white/10 bg-slate-950/80 p-3 text-sm outline-none transition focus:border-cyan-400/60"
                              placeholder="Amount"
                              value={row.amount}
                              onChange={(e) =>
                                updateDraftRow(index, { amount: sanitizeAmount(e.target.value) })
                              }
                            />
                            <select
                              className="rounded-xl border border-white/10 bg-slate-950/80 p-3 text-sm outline-none transition focus:border-cyan-400/60"
                              value={row.destEid}
                              onChange={(e) =>
                                updateDraftRow(index, { destEid: Number(e.target.value) })
                              }
                            >
                              {destinationOptions.map((option) => (
                                <option key={option.eid} value={option.eid}>
                                  {option.label}
                                </option>
                              ))}
                            </select>
                            <button
                              onClick={() => removeDraftRow(index)}
                              className="rounded-xl border border-red-500/30 px-3 py-2 text-sm font-semibold text-red-200 transition hover:bg-red-500/10"
                            >
                              Remove
                            </button>
                          </div>
                        ))}
                      </div>

                      <div className="flex flex-wrap gap-3">
                        <button
                          onClick={addDraftRow}
                          disabled={draftRows.length >= 10}
                          className="rounded-full border border-white/10 px-4 py-2 text-sm font-semibold text-slate-200 transition hover:border-white/30 hover:text-white disabled:opacity-50"
                        >
                          Add output row
                        </button>
                        <label className="rounded-full border border-white/10 px-4 py-2 text-sm font-semibold text-slate-200 transition hover:border-white/30 hover:text-white">
                          Import CSV
                          <input
                            type="file"
                            accept=".csv"
                            className="hidden"
                            onChange={(e) => e.target.files?.[0] && parseCsv(e.target.files[0])}
                          />
                        </label>
                        {csvRows.length > 0 && (
                          <div className="rounded-full border border-emerald-400/30 bg-emerald-500/10 px-4 py-2 text-sm font-semibold text-emerald-100">
                            {csvRows.length} CSV row(s) loaded
                          </div>
                        )}
                      </div>
                    </div>
                  )}
                </div>

                <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-5">
                  <div className="flex flex-wrap items-center justify-between gap-3">
                    <div>
                      <p className="text-xs uppercase tracking-[0.22em] text-slate-500">Execution summary</p>
                      <h3 className="mt-2 text-2xl font-black text-white">Route-aware settlement preview</h3>
                    </div>
                    <div className="rounded-full border border-cyan-400/30 bg-cyan-500/10 px-3 py-2 text-[11px] font-semibold uppercase tracking-[0.18em] text-cyan-100">
                      {privateRelay ? "Private Relay Active" : "Self-Relay Active"}
                    </div>
                  </div>

                  <div className="mt-4 flex flex-wrap gap-2">
                    <div className="rounded-full border border-white/10 bg-slate-950/60 px-3 py-2 text-[11px] font-semibold uppercase tracking-[0.18em] text-slate-300">
                      {routeChip}
                    </div>
                    <div className="rounded-full border border-white/10 bg-slate-950/60 px-3 py-2 text-[11px] font-semibold uppercase tracking-[0.18em] text-slate-300">
                      {status === "IDLE" ? "READY" : status.replaceAll("_", " ")}
                    </div>
                  </div>

                  <div className="mt-5 space-y-3 rounded-2xl border border-white/10 bg-slate-950/50 p-4">
                    <div className="flex items-center justify-between gap-4 text-sm">
                      <span className="text-slate-400">Recipient count</span>
                      <span className="font-semibold text-white">{activeOutputCount}</span>
                    </div>
                    <div className="flex items-center justify-between gap-4 text-sm">
                      <span className="text-slate-400">Total payout</span>
                      <span className="font-semibold text-white">{formatUnits(totalPayoutRaw, 6)} {asset}</span>
                    </div>
                    <div className="flex items-center justify-between gap-4 text-sm">
                      <span className="text-slate-400">Total fee</span>
                      <span className="font-semibold text-white">{displayedTotalFeeText}</span>
                    </div>
                    <div className="flex items-center justify-between gap-4 text-sm">
                      <span className="text-slate-400">Total shielded deduction</span>
                      <span className="font-semibold text-white">{displayedTotalShieldedDeductionText}</span>
                    </div>
                    <button
                      onClick={() => setShowFeeBreakdown((current) => !current)}
                      className="w-full rounded-full border border-white/10 px-4 py-3 text-sm font-semibold text-slate-200 transition hover:border-white/30 hover:text-white"
                    >
                      View fee breakdown
                    </button>
                  </div>

                  {showFeeBreakdown && (
                    <div className="mt-4 space-y-3 rounded-2xl border border-white/10 bg-white/[0.03] p-4">
                      <div className="flex items-center justify-between gap-4 text-sm">
                        <span className="text-slate-400">Protocol fee</span>
                        <span className="font-semibold text-white">{displayedProtocolFeeText}</span>
                      </div>

                      {!privateRelay && hasCrossChain && (
                        <div className="flex items-center justify-between gap-4 text-sm">
                          <span className="text-slate-400">Cross-chain delivery fee</span>
                          <span className="font-semibold text-white">{displayedCrossChainDeliveryFeeText}</span>
                        </div>
                      )}

                      <div className="flex items-center justify-between gap-4 text-sm">
                        <div className="flex items-center gap-2 text-slate-400">
                          <span>Relay execution fee</span>
                          {privateRelay && hasCrossChain && (
                            <span
                              className="inline-flex h-5 w-5 items-center justify-center rounded-full border border-white/10 text-[11px] text-slate-200"
                              title="Includes cross-chain delivery cost for this route in this version."
                            >
                              i
                            </span>
                          )}
                        </div>
                        <span className="font-semibold text-white">{displayedRelayExecutionFeeText}</span>
                      </div>

                      <div className="flex items-center justify-between gap-4 border-t border-white/10 pt-3 text-sm">
                        <span className="text-slate-400">Total fee</span>
                        <span className="font-semibold text-white">{displayedTotalFeeText}</span>
                      </div>
                      <div className="flex items-center justify-between gap-4 text-sm">
                        <span className="text-slate-400">Total shielded deduction</span>
                        <span className="font-semibold text-white">{displayedTotalShieldedDeductionText}</span>
                      </div>
                    </div>
                  )}

                  <div className="mt-4 space-y-2 text-sm text-slate-400">
                    {hasCrossChain ? (
                      <>
                        <p>Cross-chain delivery fees apply only to outputs leaving Tempo.</p>
                        <p>
                          {privateRelay
                            ? "Private Relay pricing includes route delivery cost for this version."
                            : "Self-Relay shows cross-chain delivery cost separately."}
                        </p>
                      </>
                    ) : (
                      <p>Same-chain Tempo payouts do not include a cross-chain delivery fee.</p>
                    )}
                    {executeQuoteLoading && <p>Quoting route delivery cost for the current output set...</p>}
                    {executeQuoteError && <p className="text-amber-300">{executeQuoteError}</p>}
                  </div>

                  <div className="mt-5 rounded-2xl border border-white/10 bg-slate-950/50 p-4">
                    <button
                      onClick={() => setShowAdvancedSettings((current) => !current)}
                      className="flex w-full items-center justify-between gap-3 text-left"
                    >
                      <span className="text-sm font-semibold text-white">Advanced Settings</span>
                      <span className="text-xs uppercase tracking-[0.18em] text-slate-400">
                        {showAdvancedSettings ? "Hide" : "Show"}
                      </span>
                    </button>

                    {showAdvancedSettings && (
                      <div className="mt-4 space-y-4">
                        <label className="flex items-start gap-3 rounded-2xl border border-white/10 bg-white/[0.03] p-4 text-sm text-slate-300">
                          <input
                            type="checkbox"
                            checked={!privateRelay}
                            onChange={(e) => setPrivateRelay(!e.target.checked)}
                            className="mt-1"
                          />
                          <span>
                            <span className="block font-semibold text-white">Use Manual Self-Relay</span>
                            <span className="mt-1 block text-slate-400">
                              Keep Private Relay as default unless you need a direct Tempo fallback path.
                            </span>
                          </span>
                        </label>

                        <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-4 text-sm text-slate-300">
                          <div className="font-semibold text-white">
                            {privateRelay ? "Private Relay is active" : "Self-Relay is active"}
                          </div>
                          <p className="mt-2 text-slate-400">
                            {privateRelay
                              ? "Parly will route this execution through the private relay path by default. This is the recommended path for privacy and smoother execution."
                              : "This execution will be sent directly from your wallet on Tempo. This is lower privacy and should only be used for manual execution, recovery, or fallback."}
                          </p>
                        </div>

                        <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-4 text-sm text-slate-300">
                          <div className="font-semibold text-white">
                            {privateRelay ? "What Parly will do" : "Before you continue"}
                          </div>
                          <p className="mt-2 text-slate-400">
                            {privateRelay
                              ? "Parly prepares the proof, evaluates the route, and forwards the execution through the relay path. If the route includes a cross-chain payout, that route delivery cost is handled inside relay execution pricing for this version."
                              : "Use the network switcher to move to Tempo. Keep the same wallet address that authenticated your Parly session. Make sure the wallet is ready for Tempo-side fee-token approval before submitting. This path is manual and lower privacy."}
                          </p>
                        </div>
                      </div>
                    )}
                  </div>

                  <button
                    onClick={execute}
                    className="mt-5 w-full rounded-2xl bg-emerald-400 py-4 font-bold text-slate-950 transition hover:bg-emerald-300 disabled:opacity-50"
                    disabled={busy || !activeNote || activeOutputCount === 0 || executeQuoteLoading}
                  >
                    {status === "PROVING"
                      ? "Generating proof..."
                      : status === "BROADCAST"
                      ? "Broadcasting..."
                      : status === "PENDING_CONFIRMATION"
                      ? "Pending confirmation"
                      : status === "SUBMITTED_TO_RELAYER"
                      ? "Submitted to relayer"
                      : "Execute private payment"}
                  </button>
                </div>
              </div>
            </div>
          )}

          {error && (
            <div className="rounded-2xl border border-red-600/50 bg-red-950/40 p-4 text-sm text-red-200">
              {error}
            </div>
          )}

          {status === "SUCCESS" && (
            <div className="rounded-2xl border border-green-600/50 bg-green-950/40 p-4 text-sm text-green-200">
              {lastAction === "shield"
                ? "Tempo deposit confirmed. Recovery will detect the new note from indexed leaves and envelopes."
                : "Direct execution confirmed on Tempo."}
            </div>
          )}

          {status === "SUBMITTED_TO_RELAYER" && (
            <div className="rounded-2xl border border-yellow-600/50 bg-yellow-950/30 p-4 text-sm text-yellow-200">
              Encrypted bundle sent to the selected relayer. Settlement completes when the relayer lands the transaction.
            </div>
          )}

          {status === "PENDING_CONFIRMATION" && pendingHash && (
            <div className="rounded-2xl border border-amber-500/50 bg-amber-950/30 p-4 text-sm text-amber-100">
              Tempo transaction {pendingHash} was broadcast, but finality could not be confirmed yet. Check the explorer before retrying or manually revoking approvals.
            </div>
          )}
        </div>
      )}
    </div>
  )
}


---

40.4 File: apps/web/src/components/SpokeShieldWidget.tsx

"use client"

import Link from "next/link"
import { useEffect, useMemo, useState } from "react"
import { decodeEventLog, formatUnits, parseUnits, type Abi } from "viem"
import { useAccount, useChainId, useSwitchChain, useWriteContract } from "wagmi"

import { SPOKE_GATEWAY_ABI } from "../lib/abis"
import { safeApproveExactBrowser } from "../lib/erc20-approval"
import { fetchDepositByCommitment } from "../lib/indexer"
import { createFreshNote } from "../lib/note-crypto"
import { useParlySession } from "../lib/parly-session"
import { getBrowserSpokes, type BrowserSpokeConfig, type SupportedSpoke } from "../lib/spoke-config"

type ShieldStatus =
  | "IDLE"
  | "QUOTING"
  | "APPROVING"
  | "BROADCAST"
  | "WAITING_FOR_INGRESS"
  | "PENDING_INGRESS"
  | "SUCCESS"

type QuotePreview = {
  amount: bigint
  feeAmount: bigint
  usesAltFeeToken: boolean
  underlyingToken: `0x${string}`
  lzFeeToken: `0x${string}`
  commitment: `0x${string}`
}

type SourceOutcome =
  | { kind: "success"; hash: `0x${string}`; guid: `0x${string}` | null }
  | { kind: "terminal_failure"; hash: `0x${string}`; message: string }
  | { kind: "uncertain"; hash: `0x${string}`; message: string }

function sanitizeAmount(value: string): string {
  return value.match(/^\d+(?:\.\d{0,6})?/)?.[0] || "0"
}

function zeroAddress(): `0x${string}` {
  return "0x0000000000000000000000000000000000000000"
}

function parseBrowserNonNegativeInt(raw: string | undefined, fallback: number, name: string): number {
  const value = Number(raw ?? String(fallback))
  if (!Number.isFinite(value) || !Number.isInteger(value) || value < 0) {
    console.warn(`${name} invalid, using fallback ${fallback}`)
    return fallback
  }
  return value
}

function sleep(ms: number) {
  return new Promise((resolve) => setTimeout(resolve, ms))
}

export default function SpokeShieldWidget() {
  const session = useParlySession()
  const { address } = useAccount()
  const connectedChainId = useChainId()
  const { switchChainAsync } = useSwitchChain()
  const { writeContractAsync } = useWriteContract()

  const [asset, setAsset] = useState<"USDC" | "USDT">("USDC")
  const [selectedSpokeKey, setSelectedSpokeKey] = useState<SupportedSpoke | "">("")
  const [depositAmount, setDepositAmount] = useState("")
  const [status, setStatus] = useState<ShieldStatus>("IDLE")
  const [error, setError] = useState<string | null>(null)
  const [pendingHash, setPendingHash] = useState<`0x${string}` | null>(null)
  const [pendingGuid, setPendingGuid] = useState<`0x${string}` | null>(null)
  const [quotePreview, setQuotePreview] = useState<QuotePreview | null>(null)

  const assetId: 1 | 2 = asset === "USDC" ? 1 : 2
  const spokes = useMemo(() => getBrowserSpokes(asset), [asset])
  const selectedSpoke = useMemo(() => {
    return spokes.find((spoke) => spoke.key === selectedSpokeKey) || spokes[0] || null
  }, [spokes, selectedSpokeKey])

  const browserReceiptTimeoutMs = parseBrowserNonNegativeInt(
    process.env.NEXT_PUBLIC_BROWSER_RECEIPT_TIMEOUT_MS,
    300000,
    "NEXT_PUBLIC_BROWSER_RECEIPT_TIMEOUT_MS"
  )
  const ingressTimeoutMs = parseBrowserNonNegativeInt(
    process.env.NEXT_PUBLIC_SPOKE_INGRESS_TIMEOUT_MS,
    180000,
    "NEXT_PUBLIC_SPOKE_INGRESS_TIMEOUT_MS"
  )
  const busy =
    status === "QUOTING" ||
    status === "APPROVING" ||
    status === "BROADCAST" ||
    status === "WAITING_FOR_INGRESS" ||
    status === "PENDING_INGRESS"

  useEffect(() => {
    if (!spokes.length) {
      setSelectedSpokeKey("")
      return
    }

    if (!selectedSpokeKey || !spokes.some((spoke) => spoke.key === selectedSpokeKey)) {
      setSelectedSpokeKey(spokes[0].key)
    }
  }, [spokes, selectedSpokeKey])

  useEffect(() => {
    if (status === "WAITING_FOR_INGRESS" || status === "PENDING_INGRESS") {
      return
    }

    setQuotePreview(null)
    setPendingHash(null)
    setPendingGuid(null)
    setStatus("IDLE")
  }, [asset, selectedSpokeKey, depositAmount])

  useEffect(() => {
    if (session.phase === "ready") return

    setPendingHash(null)
    setPendingGuid(null)
    setQuotePreview(null)
    setError(null)
    setStatus("IDLE")
  }, [session.phase, address])

  async function ensureAuthenticated() {
    if (!address) throw new Error("Connect wallet first.")
    if (session.phase !== "ready" || !session.recoveryPublicKeyB64) {
      const auth = await session.authenticate()
      if (auth?.recoveryPublicKeyB64) {
        return { publicKeyB64: auth.recoveryPublicKeyB64 }
      }
    }

    if (!session.recoveryPublicKeyB64) {
      throw new Error("Authenticate this wallet before shielding from a spoke.")
    }

    return { publicKeyB64: session.recoveryPublicKeyB64 }
  }

  async function loadGatewayConfig(spoke: BrowserSpokeConfig) {
    const underlyingToken = await spoke.publicClient.readContract({
      address: spoke.gateway,
      abi: SPOKE_GATEWAY_ABI,
      functionName: "underlyingToken"
    }) as `0x${string}`

    const lzFeeToken = await spoke.publicClient.readContract({
      address: spoke.gateway,
      abi: SPOKE_GATEWAY_ABI,
      functionName: "lzFeeToken"
    }) as `0x${string}`

    return {
      underlyingToken,
      lzFeeToken
    }
  }

  async function waitForSpokeFinality(args: {
    spoke: BrowserSpokeConfig
    address: `0x${string}`
    abi: Abi
    functionName: string
    args?: readonly unknown[]
    value?: bigint
  }): Promise<SourceOutcome> {
    let submittedHash: `0x${string}` | null = null
    let finalHash: `0x${string}` | null = null
    let replacementReason: "replaced" | "repriced" | "cancelled" | null = null

    try {
      const { spoke, ...txArgs } = args
      submittedHash = await writeContractAsync({
        ...(txArgs as any),
        chainId: spoke.chainId
      })
      finalHash = submittedHash

      const receipt = await spoke.publicClient.waitForTransactionReceipt({
        hash: submittedHash,
        timeout: browserReceiptTimeoutMs,
        onReplaced: (replacement) => {
          replacementReason = replacement.reason
          finalHash = replacement.transactionReceipt.transactionHash as `0x${string}`
        }
      })

      if (replacementReason === "cancelled") {
        return {
          kind: "terminal_failure",
          hash: finalHash!,
          message: "Source transaction was cancelled before inclusion."
        }
      }

      if (replacementReason === "replaced") {
        return {
          kind: "terminal_failure",
          hash: finalHash!,
          message: "Source transaction was replaced before inclusion."
        }
      }

      if (receipt.status !== "success") {
        return {
          kind: "terminal_failure",
          hash: finalHash!,
          message: "Source transaction reverted on chain."
        }
      }

      let guid: `0x${string}` | null = null

      for (const log of receipt.logs) {
        try {
          const decoded = decodeEventLog({
            abi: SPOKE_GATEWAY_ABI,
            data: log.data,
            topics: log.topics
          })

          if (decoded.eventName === "ShieldedCrossChain") {
            guid = decoded.args.guid as `0x${string}`
            break
          }
        } catch {
          continue
        }
      }

      return { kind: "success", hash: finalHash!, guid }
    } catch (e: any) {
      if (!submittedHash) {
        throw e
      }

      return {
        kind: "uncertain",
        hash: finalHash || submittedHash,
        message: e?.message || "Source-chain finality could not be confirmed yet."
      }
    }
  }

  async function cleanupApproval(
    spoke: BrowserSpokeConfig,
    approvalToken: `0x${string}`,
    approvalSpender: `0x${string}`
  ) {
    if (!address) return

    await safeApproveExactBrowser({
      token: approvalToken,
      spender: approvalSpender,
      amount: 0n,
      owner: address,
      publicClient: spoke.publicClient,
      writeAndWait: async (args) => {
        const hash = await writeContractAsync({
          ...(args as any),
          chainId: spoke.chainId
        })
        await spoke.publicClient.waitForTransactionReceipt({
          hash,
          timeout: browserReceiptTimeoutMs
        })
        return hash
      }
    })
  }

  async function waitForTempoIngress(commitment: `0x${string}`) {
    const startedAt = Date.now()

    while (Date.now() - startedAt < ingressTimeoutMs) {
      const deposit = await fetchDepositByCommitment(assetId, commitment)
      if (deposit) {
        return true
      }
      await sleep(5000)
    }

    return false
  }

  async function quoteShield() {
    try {
      setError(null)
      setPendingHash(null)
      setPendingGuid(null)

      if (!address) throw new Error("Connect wallet first.")
      if (!selectedSpoke) throw new Error("No configured spoke gateway for this asset.")

      const { publicKeyB64 } = await ensureAuthenticated()
      const amount = parseUnits(sanitizeAmount(depositAmount), 6)
      if (amount <= 0n) throw new Error("Invalid shield amount.")

      setStatus("QUOTING")

      const fresh = await createFreshNote(publicKeyB64, address, assetId, amount.toString(), "deposit")
      const gatewayConfig = await loadGatewayConfig(selectedSpoke)
      const [feeAmount, usesAltFeeToken] = await selectedSpoke.publicClient.readContract({
        address: selectedSpoke.gateway,
        abi: SPOKE_GATEWAY_ABI,
        functionName: "quoteShieldFee",
        args: [BigInt(fresh.innerCommitment), amount, address, fresh.envelope]
      }) as [bigint, boolean]

      setQuotePreview({
        amount,
        feeAmount,
        usesAltFeeToken,
        underlyingToken: gatewayConfig.underlyingToken,
        lzFeeToken: gatewayConfig.lzFeeToken,
        commitment: fresh.commitment
      })
      setStatus("IDLE")
    } catch (e: any) {
      setStatus("IDLE")
      setError(e.message || "Quote failed")
    }
  }

  async function shieldFromSpoke() {
    let approvalsToCleanup: Array<{
      token: `0x${string}`
      spender: `0x${string}`
    }> = []
    let shouldCleanupApproval = false
    let confirmedSourceHash: `0x${string}` | null = null

    try {
      setError(null)
      setPendingHash(null)
      setPendingGuid(null)

      if (!address) throw new Error("Connect wallet first.")
      if (!selectedSpoke) throw new Error("No configured spoke gateway for this asset.")

      if (connectedChainId !== selectedSpoke.chainId) {
        await switchChainAsync({ chainId: selectedSpoke.chainId })
      }

      const { publicKeyB64 } = await ensureAuthenticated()
      const amount = parseUnits(sanitizeAmount(depositAmount), 6)
      if (amount <= 0n) throw new Error("Invalid shield amount.")

      const fresh = await createFreshNote(publicKeyB64, address, assetId, amount.toString(), "deposit")
      const gatewayConfig = await loadGatewayConfig(selectedSpoke)
      const [feeAmount, usesAltFeeToken] = await selectedSpoke.publicClient.readContract({
        address: selectedSpoke.gateway,
        abi: SPOKE_GATEWAY_ABI,
        functionName: "quoteShieldFee",
        args: [BigInt(fresh.innerCommitment), amount, address, fresh.envelope]
      }) as [bigint, boolean]

      const writeAndWait = async (args: {
        address: `0x${string}`
        abi: Abi
        functionName: string
        args?: readonly unknown[]
      }) => {
        const hash = await writeContractAsync({
          ...(args as any),
          chainId: selectedSpoke.chainId
        })
        await selectedSpoke.publicClient.waitForTransactionReceipt({
          hash,
          timeout: browserReceiptTimeoutMs
        })
        return hash
      }

      const zero = zeroAddress()
      const lzFeeToken = gatewayConfig.lzFeeToken
      const underlyingToken = gatewayConfig.underlyingToken

      if (usesAltFeeToken && lzFeeToken.toLowerCase() === underlyingToken.toLowerCase()) {
        approvalsToCleanup = [{ token: underlyingToken, spender: selectedSpoke.gateway }]
      } else {
        approvalsToCleanup = [{ token: underlyingToken, spender: selectedSpoke.gateway }]
        if (usesAltFeeToken && lzFeeToken.toLowerCase() !== zero.toLowerCase()) {
          approvalsToCleanup.push({ token: lzFeeToken, spender: selectedSpoke.gateway })
        }
      }

      setStatus("APPROVING")

      if (usesAltFeeToken && lzFeeToken.toLowerCase() === underlyingToken.toLowerCase()) {
        await safeApproveExactBrowser({
          token: underlyingToken,
          spender: selectedSpoke.gateway,
          amount: amount + feeAmount,
          owner: address,
          publicClient: selectedSpoke.publicClient,
          writeAndWait
        })
      } else {
        await safeApproveExactBrowser({
          token: underlyingToken,
          spender: selectedSpoke.gateway,
          amount,
          owner: address,
          publicClient: selectedSpoke.publicClient,
          writeAndWait
        })

        if (usesAltFeeToken && lzFeeToken.toLowerCase() !== zero.toLowerCase()) {
          await safeApproveExactBrowser({
            token: lzFeeToken,
            spender: selectedSpoke.gateway,
            amount: feeAmount,
            owner: address,
            publicClient: selectedSpoke.publicClient,
            writeAndWait
          })
        }
      }

      setStatus("BROADCAST")

      const outcome = await waitForSpokeFinality({
        spoke: selectedSpoke,
        address: selectedSpoke.gateway,
        abi: SPOKE_GATEWAY_ABI,
        functionName: "shieldCrossChain",
        args: [BigInt(fresh.innerCommitment), amount, fresh.envelope],
        value: usesAltFeeToken ? 0n : feeAmount
      })

      if (outcome.kind === "uncertain") {
        setPendingHash(outcome.hash)
        setStatus("PENDING_INGRESS")
        setError(
          `Source transaction ${outcome.hash} was broadcast, but source-chain finality could not yet be confirmed. Do not retry or revoke approvals until chain state is known.`
        )
        return
      }

      if (outcome.kind === "terminal_failure") {
        shouldCleanupApproval = true
        throw new Error(`${outcome.message} ${outcome.hash}`)
      }

      setPendingHash(outcome.hash)
      setPendingGuid(outcome.guid)
      confirmedSourceHash = outcome.hash
      setQuotePreview({
        amount,
        feeAmount,
        usesAltFeeToken,
        underlyingToken,
        lzFeeToken,
        commitment: fresh.commitment
      })
      setStatus("WAITING_FOR_INGRESS")

      const visible = await waitForTempoIngress(fresh.commitment)
      if (!visible) {
        setStatus("PENDING_INGRESS")
        setError(
          `Source transaction ${outcome.hash} confirmed, but the Tempo leaf did not appear before timeout. Inspect hub composer deferred ingress state and use retry/claim tooling if needed.`
        )
        return
      }

      session.refreshRecovery()
      setStatus("SUCCESS")
    } catch (e: any) {
      if (confirmedSourceHash) {
        setPendingHash(confirmedSourceHash)
        setStatus("PENDING_INGRESS")
        setError(
          `Source transaction ${confirmedSourceHash} confirmed, but Tempo ingress could not be re-checked cleanly. Inspect the indexer and hub composer deferred-ingress state before retrying anything.`
        )
        return
      }

      shouldCleanupApproval = true
      setStatus("IDLE")
      setError(e.message || "Spoke shielding failed")
    }

    if (shouldCleanupApproval && selectedSpoke) {
      for (const approval of approvalsToCleanup) {
        try {
          await cleanupApproval(selectedSpoke, approval.token, approval.spender)
        } catch (cleanupError: any) {
          console.error(`Spoke approval cleanup failed: ${cleanupError.message || cleanupError}`)
          setError((current) =>
            current
              ? `${current} Approval cleanup also failed. Manually revoke allowance on the source chain.`
              : "Approval cleanup failed. Manually revoke allowance on the source chain."
          )
        }
      }
    }
  }

  return (
    <div className="mx-auto w-full max-w-3xl rounded-2xl border border-slate-800 bg-slate-900/80 p-6 text-white shadow-2xl">
      <div className="flex flex-col gap-3 md:flex-row md:items-end md:justify-between">
        <div>
          <p className="text-xs uppercase tracking-[0.3em] text-cyan-300">Spoke ingress</p>
          <h2 className="mt-2 text-3xl font-black tracking-tight">Shield from a supported spoke</h2>
          <p className="mt-3 max-w-2xl text-sm text-slate-400">
            This flow approves the spoke gateway, sends the OFT compose message toward Tempo, and
            then waits for the new Tempo leaf. If the source send confirms but the Tempo leaf does
            not appear yet, the UI stays honest and points you to deferred-ingress recovery instead
            of pretending the shield finished.
          </p>
        </div>
        {selectedSpoke && (
          <div className="rounded-lg border border-slate-700 bg-slate-950/70 px-4 py-3 text-sm">
            <div className="text-slate-400">Selected source</div>
            <div className="mt-1 font-semibold text-slate-100">{selectedSpoke.label}</div>
          </div>
        )}
      </div>

      {!spokes.length ? (
        <div className="mt-6 rounded border border-amber-500 bg-amber-950/30 p-4 text-sm text-amber-200">
          No spoke gateways are configured for this asset yet. Fill the
          `NEXT_PUBLIC_SPOKE_*_GATEWAY_*` env values before using the spoke shield flow.
        </div>
        ) : (
        <div className="mt-6 space-y-5">
          {session.error && (
            <div className="rounded-2xl border border-amber-500/40 bg-amber-500/10 p-4 text-sm text-amber-100">
              {session.error}
            </div>
          )}

          <div className="grid gap-4 md:grid-cols-[1.1fr_0.9fr]">
            <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-5">
              <div className="grid gap-4 md:grid-cols-2">
                <div>
                  <label className="mb-2 block text-xs font-semibold uppercase tracking-[0.2em] text-slate-400">
                    Asset
                  </label>
                  <div className="flex gap-2">
                    <button
                      onClick={() => setAsset("USDC")}
                      className={`flex-1 rounded-xl py-3 font-semibold ${asset === "USDC" ? "bg-blue-600" : "bg-slate-800"}`}
                    >
                      USDC
                    </button>
                    <button
                      onClick={() => setAsset("USDT")}
                      className={`flex-1 rounded-xl py-3 font-semibold ${asset === "USDT" ? "bg-emerald-700" : "bg-slate-800"}`}
                    >
                      USDT
                    </button>
                  </div>
                </div>

                <div>
                  <label className="mb-2 block text-xs font-semibold uppercase tracking-[0.2em] text-slate-400">
                    Source chain
                  </label>
                  <select
                    value={selectedSpoke?.key || ""}
                    onChange={(e) => setSelectedSpokeKey(e.target.value as SupportedSpoke)}
                    className="w-full rounded-xl border border-slate-700 bg-slate-950 p-3 text-sm"
                  >
                    {spokes.map((spoke) => (
                      <option key={spoke.key} value={spoke.key}>
                        {spoke.label}
                      </option>
                    ))}
                  </select>
                </div>
              </div>

              <div className="mt-4">
                <label className="mb-2 block text-xs font-semibold uppercase tracking-[0.2em] text-slate-400">
                  Shield amount
                </label>
                <input
                  className="w-full rounded-2xl border border-slate-700 bg-slate-950 p-4 text-sm"
                  placeholder="0.00"
                  value={depositAmount}
                  onChange={(e) => setDepositAmount(sanitizeAmount(e.target.value))}
                />
              </div>

              <div className="mt-4 flex flex-wrap gap-2">
                {["100", "250", "500", "1000"].map((preset) => (
                  <button
                    key={preset}
                    onClick={() => setDepositAmount(preset)}
                    className="rounded-full border border-white/10 px-3 py-2 text-xs font-semibold text-slate-300 transition hover:border-white/30 hover:text-white"
                  >
                    {preset} {asset}
                  </button>
                ))}
              </div>

              {session.phase !== "ready" ? (
                <button
                  className="mt-5 w-full rounded-2xl bg-cyan-500 py-4 font-bold text-black disabled:opacity-50"
                  onClick={session.authenticate}
                  disabled={busy}
                >
                  {session.phase === "authenticating" ? "Authenticating..." : "Authenticate wallet"}
                </button>
              ) : (
                <div className="mt-5 rounded-2xl border border-white/10 bg-slate-950/70 p-4">
                  <div className="text-xs uppercase tracking-[0.2em] text-slate-400">Recovery public key</div>
                  <div className="mt-2 break-all text-xs text-slate-200">{session.recoveryPublicKeyB64}</div>
                </div>
              )}

              <div className="mt-4 grid gap-3 md:grid-cols-2">
                <button
                  className="rounded-2xl bg-slate-700 py-3 font-semibold disabled:opacity-50"
                  onClick={quoteShield}
                  disabled={busy || session.phase === "authenticating"}
                >
                  {status === "QUOTING" ? "Quoting..." : "Quote shield"}
                </button>
                <button
                  className="rounded-2xl bg-cyan-500 py-3 font-bold text-black disabled:opacity-50"
                  onClick={shieldFromSpoke}
                  disabled={busy || !selectedSpoke || session.phase !== "ready"}
                >
                  {status === "APPROVING"
                    ? "Approving..."
                    : status === "BROADCAST"
                    ? "Broadcasting..."
                    : status === "WAITING_FOR_INGRESS"
                    ? "Waiting for Tempo ingress..."
                    : "Shield to Tempo"}
                </button>
              </div>
            </div>

            <div className="rounded-2xl border border-white/10 bg-white/[0.03] p-5">
              <p className="text-xs uppercase tracking-[0.22em] text-slate-500">Flow model</p>
              <div className="mt-4 space-y-4 text-sm text-slate-300">
                <div>
                  <div className="font-semibold text-white">Origin</div>
                  <div className="mt-1">{selectedSpoke ? selectedSpoke.label : "No configured spoke"}</div>
                </div>
                <div>
                  <div className="font-semibold text-white">Destination</div>
                  <div className="mt-1">Tempo settlement and Tempo-side note recovery</div>
                </div>
                <div>
                  <div className="font-semibold text-white">Provenance model</div>
                  <div className="mt-1">
                    Once the source dispatch and Tempo settlement share a LayerZero guid, the indexer can expose deterministic spoke-origin provenance without guessing.
                  </div>
                </div>
              </div>
              <div className="mt-5 flex flex-wrap gap-2">
                <Link
                  href="/history"
                  className="rounded-full border border-white/10 px-3 py-2 text-xs font-semibold text-slate-200 transition hover:border-white/30 hover:text-white"
                >
                  Open history
                </Link>
                <Link
                  href="/app"
                  className="rounded-full border border-white/10 px-3 py-2 text-xs font-semibold text-slate-200 transition hover:border-white/30 hover:text-white"
                >
                  Back to Tempo app
                </Link>
              </div>
            </div>
          </div>

          {selectedSpoke && connectedChainId !== selectedSpoke.chainId && (
            <div className="rounded-2xl border border-amber-500 bg-amber-950/30 p-4 text-sm text-amber-200">
              Connected wallet is on chain ID {connectedChainId}. The shield flow will request a switch to {selectedSpoke.label} before broadcasting.
            </div>
          )}

          {quotePreview && (
            <div className="rounded-2xl border border-slate-700 bg-slate-950/70 p-4 text-sm">
              <div className="font-semibold text-slate-100">Quote preview</div>
              <div className="mt-3 grid gap-2 text-slate-300 md:grid-cols-2">
                <div>Amount: {formatUnits(quotePreview.amount, 6)} {asset}</div>
                <div>Fee mode: {quotePreview.usesAltFeeToken ? "Alt fee token" : "Native gas token"}</div>
                <div>Fee amount (raw units): {quotePreview.feeAmount.toString()}</div>
                <div className="break-all">Pending commitment: {quotePreview.commitment}</div>
              </div>
            </div>
          )}

          {pendingHash && (
            <div className="rounded-2xl border border-slate-700 bg-slate-950/70 p-4 text-sm text-slate-300">
              <div>
                Source tx hash: <span className="break-all">{pendingHash}</span>
              </div>
              {pendingGuid && (
                <div className="mt-2">
                  LayerZero guid: <span className="break-all">{pendingGuid}</span>
                </div>
              )}
            </div>
          )}

          {error && (
            <div className="rounded-2xl border border-red-600 bg-red-950/40 p-4 text-sm text-red-300">
              {error}
            </div>
          )}

          {status === "SUCCESS" && (
            <div className="rounded-2xl border border-green-600 bg-green-950/40 p-4 text-sm text-green-300">
              Source transaction confirmed and the new Tempo leaf is now visible in the indexer and ready for Tempo-side recovery.
            </div>
          )}

          {status === "PENDING_INGRESS" && !error && (
            <div className="rounded-2xl border border-amber-500 bg-amber-950/30 p-4 text-sm text-amber-200">
              <div className="space-y-2">
                <p>
                  The source transfer completed, but Tempo ingress is still deferred or pending. Do not treat this as a successful private deposit yet.
                </p>
                <p>
                  Use exactly one of the ordinary recovery paths for this deferred guid:
                  retry deposit after fixing the root cause, or claim refund back to the original depositor if you want principal returned instead of ingress.
                </p>
                <p>
                  Refund returns principal only. It does not reimburse source-chain gas or LayerZero messaging fees already spent.
                </p>
              </div>
            </div>
          )}
        </div>
      )}
    </div>
  )
}


---

41. App routes and verification portal

41.1 File: apps/web/src/app/page.tsx

import Link from "next/link"

const launchCards = [
  {
    href: "/app",
    eyebrow: "Tempo",
    title: "Shield and execute on Tempo",
    copy: "Recover notes, deposit on Tempo, and execute private same-chain or cross-chain payouts."
  },
  {
    href: "/spoke-shield",
    eyebrow: "Spoke ingress",
    title: "Shield from a supported spoke",
    copy: "Use Ethereum, Arbitrum, Base, or BNB as the origin network and settle into the Tempo note model."
  },
  {
    href: "/history",
    eyebrow: "Compliance",
    title: "Parly Ledger & Compliance",
    copy: "Track shielded capital, review private executions, and open one payout scope at a time for audit-ready disclosure."
  },
  {
    href: "/verify",
    eyebrow: "Verification",
    title: "Verify a Parly Payout Scope",
    copy: "Confirm who was paid, how much, and where from one real Tempo transaction hash plus one child key."
  },
  {
    href: "/docs",
    eyebrow: "Docs",
    title: "Protocol truth boundaries",
    copy: "Read the product invariants, recovery model, provenance rules, and operator-safe deployment notes."
  },
  {
    href: "/sdk",
    eyebrow: "Agentic SDK",
    title: "Automate the launch surface",
    copy: "Use the SDK for deterministic recovery, private execution, and the optional MPP-compatible adapter without changing protocol semantics."
  }
]

export default function HomePage() {
  return (
    <div className="mx-auto flex min-h-[80vh] w-full max-w-7xl flex-col justify-center py-10">
      <section className="rounded-[2rem] border border-white/10 bg-[radial-gradient(circle_at_top_left,rgba(34,211,238,0.16),transparent_32%),radial-gradient(circle_at_bottom_right,rgba(16,185,129,0.14),transparent_28%),rgba(15,23,42,0.82)] px-6 py-10 md:px-10">
        <p className="text-xs uppercase tracking-[0.35em] text-cyan-300">Parly.fi V16.9.9</p>
        <h1 className="mt-4 max-w-5xl text-4xl font-black tracking-tight text-white md:text-6xl">
          Stablecoin-only private settlement on Tempo with deterministic recovery and honest provenance boundaries
        </h1>
        <p className="mt-5 max-w-3xl text-base text-slate-300 md:text-lg">
          Shield on Tempo, shield from supported spokes, execute private same-chain or cross-chain payouts, and surface only the metadata the protocol can actually support.
        </p>

        <div className="mt-8 flex flex-wrap gap-3">
          <Link
            href="/app"
            className="rounded-full bg-white px-5 py-3 text-sm font-semibold text-slate-950 transition hover:bg-cyan-200"
          >
            Open app
          </Link>
          <Link
            href="/spoke-shield"
            className="rounded-full border border-white/15 px-5 py-3 text-sm font-semibold text-slate-100 transition hover:border-white/35 hover:text-white"
          >
            Spoke shield flow
          </Link>
          <Link
            href="/history"
            className="rounded-full border border-white/15 px-5 py-3 text-sm font-semibold text-slate-100 transition hover:border-white/35 hover:text-white"
          >
            Parly Ledger & Compliance
          </Link>
        </div>
      </section>

      <section className="mt-8 grid gap-4 md:grid-cols-2 xl:grid-cols-3">
        {launchCards.map((card) => (
          <Link
            key={card.href}
            href={card.href}
            className="rounded-3xl border border-white/10 bg-white/[0.03] p-6 transition hover:border-cyan-400/40 hover:bg-white/[0.05]"
          >
            <div className="text-xs uppercase tracking-[0.25em] text-slate-400">{card.eyebrow}</div>
            <div className="mt-3 text-2xl font-bold text-white">{card.title}</div>
            <p className="mt-3 text-sm leading-6 text-slate-400">{card.copy}</p>
          </Link>
        ))}
      </section>

      <section className="mt-8 grid gap-4 xl:grid-cols-3">
        <div className="rounded-3xl border border-white/10 bg-white/[0.03] p-6">
          <div className="text-xs uppercase tracking-[0.22em] text-slate-500">Invariant</div>
          <div className="mt-3 text-xl font-bold text-white">No fake provenance</div>
          <p className="mt-3 text-sm leading-6 text-slate-400">
            Source chain, source tx hash, and origin sender only appear when explicit events and indexed joins make them deterministic.
          </p>
        </div>
        <div className="rounded-3xl border border-white/10 bg-white/[0.03] p-6">
          <div className="text-xs uppercase tracking-[0.22em] text-slate-500">Invariant</div>
          <div className="mt-3 text-xl font-bold text-white">Deterministic recovery</div>
          <p className="mt-3 text-sm leading-6 text-slate-400">
            The wallet signs one fixed domain-separated message, derives the recovery keypair locally, and reconstructs notes from indexed chain data.
          </p>
        </div>
        <div className="rounded-3xl border border-white/10 bg-white/[0.03] p-6">
          <div className="text-xs uppercase tracking-[0.22em] text-slate-500">Invariant</div>
          <div className="mt-3 text-xl font-bold text-white">Stablecoin-only launch scope</div>
          <p className="mt-3 text-sm leading-6 text-slate-400">
            V16.9.9 keeps stablecoin-only behavior, private Waku relay, self-relay fallback, and the fixed 10-output padded batch model intact.
          </p>
        </div>
      </section>
    </div>
  )
}

41.2 File: apps/web/src/app/app/page.tsx

"use client"

import { useState } from "react"
import HistoryCompliancePanel from "../../components/HistoryCompliancePanel"
import Widget from "../../components/Widget"

export default function AppPage() {
  const [showLedger, setShowLedger] = useState(false)

  return (
    <div className="space-y-6">
      <div>
        <p className="text-xs uppercase tracking-[0.3em] text-cyan-300">Tempo application</p>
        <h1 className="mt-3 text-4xl font-black tracking-tight text-white">Shield, recover, and execute</h1>
        <p className="mt-3 max-w-3xl text-sm text-slate-400">
          This route is the primary Tempo console. It keeps the live launch behavior intact while
          placing Parly Ledger & Compliance directly below the execution surface so operational
          review, scoped receipts, and verification flows stay connected without front-loading the
          heavier ledger decode path before the operator opens it.
        </p>
      </div>

      <Widget />

      <div className="rounded-3xl border border-white/10 bg-slate-950/70 p-6 shadow-2xl shadow-cyan-950/10">
        <div className="flex flex-col gap-4 xl:flex-row xl:items-end xl:justify-between">
          <div>
            <p className="text-xs uppercase tracking-[0.25em] text-cyan-300">Parly Ledger & Compliance</p>
            <h2 className="mt-3 text-3xl font-black text-white">Parly Ledger & Compliance</h2>
            <p className="mt-3 max-w-3xl text-sm text-slate-400">
              Load the full compliance surface when you are ready to review history, open one payout
              scope at a time, or export lane-scoped records.
            </p>
          </div>

          <button
            type="button"
            onClick={() => setShowLedger((current) => !current)}
            className="rounded-full bg-white px-4 py-3 text-sm font-semibold text-slate-950 transition hover:bg-slate-200"
          >
            {showLedger ? "Hide Parly Ledger & Compliance" : "Open Parly Ledger & Compliance"}
          </button>
        </div>
      </div>

      {showLedger && <HistoryCompliancePanel />}
    </div>
  )
}

41.3 File: apps/web/src/app/history/page.tsx

import HistoryCompliancePanel from "../../components/HistoryCompliancePanel"

export default function HistoryPage() {
  return (
    <div className="space-y-6">
      <div>
        <p className="text-xs uppercase tracking-[0.3em] text-cyan-300">Parly Ledger & Compliance</p>
        <h1 className="mt-3 text-4xl font-black tracking-tight text-white">Parly Ledger & Compliance</h1>
        <p className="mt-3 max-w-3xl text-sm text-slate-400">
          The standalone route mirrors the in-app ledger surface for teams that want a direct path
          into scoped compliance review without re-opening the primary widget first.
        </p>
      </div>

      <HistoryCompliancePanel />
    </div>
  )
}

41.4 File: apps/web/src/app/spoke-shield/page.tsx

import SpokeShieldWidget from "../../components/SpokeShieldWidget"

export default function SpokeShieldPage() {
  return (
    <div className="space-y-6">
      <div>
        <p className="text-xs uppercase tracking-[0.35em] text-cyan-300">Cross-chain ingress</p>
        <h1 className="mt-3 text-4xl font-black tracking-tight text-white">
          Shield from a supported spoke into Tempo
        </h1>
        <p className="mt-3 max-w-3xl text-sm text-slate-400">
          This route exists because spoke shielding is a real launch feature, not just a contract primitive. It quotes on the selected source chain, switches the wallet if needed, submits the gateway transaction, and then checks Tempo ingress honestly.
        </p>
      </div>

      <SpokeShieldWidget />
    </div>
  )
}

41.5 File: apps/web/src/app/verify/page.tsx

"use client"

import { useEffect, useMemo, useState } from "react"
import { formatUnits } from "viem"
import TruthBadge from "../../components/TruthBadge"
import {
  fetchDepositHistory,
  fetchTxBatches,
  fetchScopedBatchOutput
} from "../../lib/indexer"
import { useParlySession } from "../../lib/parly-session"
import { useRecoveredNotes } from "../../lib/useRecoveredNotes"

type TruthLabel =
  | "chain_indexed"
  | "onchain_calldata_decoded"
  | "locally_reconstructed"
  | "provenance_record"

type VerifyResult = {
  childKey: string
  txHash: string
  timestamp: number
  assetId: number
  protocolFee: string
  executorFee: string
  nullifierHash: string
  truth: TruthLabel[]
  output: {
    outputIndex: number
    recipient: `0x${string}`
    amount: string
    destEid: number
    destinationChain: string
    routeType: "same_chain" | "cross_chain"
  }
  local?: {
    sourceNoteCommitment: string
    sourceNoteAmount: string
  }
  provenance?: {
    settlementTxHash: string
    settlementChain: string
    settlementEid: number
    sourceChain?: string
    sourceEid: number
    sourceTxHash?: string
    sourceSender?: string
    guid: string
  }
}

function normalizeHex(value: string): `0x${string}` {
  const clean = value.trim().toLowerCase()
  return (clean.startsWith("0x") ? clean : `0x${clean}`) as `0x${string}`
}

function assetLabel(assetId: number) {
  return assetId === 1 ? "USDC" : assetId === 2 ? "USDT" : `Asset ${assetId}`
}

function parseChildKeys(raw: string[] | string): string[] {
  if (Array.isArray(raw)) return raw
  try {
    return JSON.parse(raw)
  } catch {
    return []
  }
}

export default function VerifyPage() {
  const session = useParlySession()
  const [txHash, setTxHash] = useState("")
  const [childKey, setChildKey] = useState("")
  const [showSampleReport, setShowSampleReport] = useState(false)
  const [result, setResult] = useState<VerifyResult | null>(null)
  const [error, setError] = useState("")
  const [loading, setLoading] = useState(false)

  const usdcRecovery = useRecoveredNotes({
    address: session.address,
    assetId: 1,
    recoveryPublicKeyB64: session.recoveryPublicKeyB64,
    recoveryPrivateKeyB64: session.recoveryPrivateKeyB64,
    optionalCacheKey: session.optionalCacheKey,
    refreshNonce: session.recoveryEpoch
  })
  const usdtRecovery = useRecoveredNotes({
    address: session.address,
    assetId: 2,
    recoveryPublicKeyB64: session.recoveryPublicKeyB64,
    recoveryPrivateKeyB64: session.recoveryPrivateKeyB64,
    optionalCacheKey: session.optionalCacheKey,
    refreshNonce: session.recoveryEpoch
  })

  useEffect(() => {
    session.setRecoveryActivity("verify", usdcRecovery.loading || usdtRecovery.loading)
    return () => {
      session.setRecoveryActivity("verify", false)
    }
  }, [session, usdcRecovery.loading, usdtRecovery.loading])

  const spentNotes = useMemo(() => {
    return [...usdcRecovery.notes, ...usdtRecovery.notes].filter((note) => note.status === "spent")
  }, [usdcRecovery.notes, usdtRecovery.notes])

  async function verify() {
    setError("")
    setResult(null)
    setLoading(true)

    try {
      if (!txHash.trim()) {
        throw new Error("Strict audit mode requires the confirmed Tempo tx hash.")
      }
      if (!childKey.trim()) {
        throw new Error("Provide a child key to verify.")
      }

      const normalizedTxHash = normalizeHex(txHash)
      const normalizedChildKey = normalizeHex(childKey)
      const rows = await fetchTxBatches(normalizedTxHash)
      const batch =
        rows.find((row: any) =>
          parseChildKeys(row.childKeys).some(
            (candidate) => String(candidate).toLowerCase() === normalizedChildKey.toLowerCase()
          )
        ) || null

      if (!batch) {
        throw new Error("That child key was not found in the indexed BatchWithdrawal event for the supplied Tempo tx hash.")
      }

      const scopedOutput = await fetchScopedBatchOutput(normalizedTxHash, normalizedChildKey)
      if (!scopedOutput) {
        throw new Error("The selected child key did not map to a decoded payout output in the confirmed Tempo calldata.")
      }

      const truth: TruthLabel[] = ["chain_indexed", "onchain_calldata_decoded"]
      const nextResult: VerifyResult = {
        childKey: normalizedChildKey,
        txHash: batch.txHash,
        timestamp: Number(batch.timestamp),
        assetId: Number(batch.assetId),
        protocolFee: String(batch.protocolFee),
        executorFee: String(batch.executorFee),
        nullifierHash: String(batch.nullifierHash),
        truth,
        output: {
          outputIndex: scopedOutput.outputIndex,
          recipient: scopedOutput.recipient,
          amount: scopedOutput.amount,
          destEid: scopedOutput.destEid,
          destinationChain: scopedOutput.destinationChain,
          routeType: scopedOutput.routeType
        }
      }

      const localNote = spentNotes.find(
        (note) =>
          note.assetId === Number(batch.assetId) &&
          String(note.nullifierHash).toLowerCase() === String(batch.nullifierHash).toLowerCase()
      )

      if (localNote) {
        nextResult.local = {
          sourceNoteCommitment: localNote.commitment,
          sourceNoteAmount: localNote.amount
        }
        truth.push("locally_reconstructed")

        const depositHistory = await fetchDepositHistory(Number(batch.assetId))
        const depositRow = depositHistory.find(
          (row) => String(row.commitment).toLowerCase() === String(localNote.commitment).toLowerCase()
        )

        if (depositRow?.provenance) {
          nextResult.provenance = {
            settlementTxHash: depositRow.provenance.settlementTxHash,
            settlementChain: depositRow.provenance.settlementChain,
            settlementEid: depositRow.provenance.settlementEid,
            sourceChain: depositRow.provenance.sourceChain,
            sourceEid: depositRow.provenance.sourceEid,
            sourceTxHash: depositRow.provenance.sourceTxHash,
            sourceSender: depositRow.provenance.sourceSender,
            guid: depositRow.provenance.guid
          }
          truth.push("provenance_record")
        }
      }

      setResult(nextResult)
    } catch (e: any) {
      setError(e.message || "Verification failed")
    } finally {
      setLoading(false)
    }
  }

  return (
    <div className="mx-auto w-full max-w-5xl space-y-6">
      <div>
        <p className="text-xs uppercase tracking-[0.35em] text-cyan-300">Verify a Parly Payout Scope</p>
        <h1 className="mt-3 text-4xl font-black tracking-tight text-white">Verify a Parly Payout Scope</h1>
        <p className="mt-3 max-w-3xl text-sm text-slate-400">
          Confirm one disclosed payout lane from a real Parly execution using the finalized Tempo
          transaction hash and child key.
        </p>
        <p className="mt-3 max-w-3xl text-sm text-slate-500">
          Built for auditors, exchanges, treasury teams, compliance reviewers, and business
          counterparties.
        </p>
      </div>

      <div className="rounded-3xl border border-white/10 bg-white/[0.03] p-6">
        <button
          type="button"
          onClick={() => setShowSampleReport((current) => !current)}
          className="mb-4 text-sm font-semibold text-cyan-300 transition hover:text-cyan-200"
        >
          {showSampleReport ? "Hide Sample Report" : "Preview Sample Report"}
        </button>
        <div className="grid gap-4 md:grid-cols-2">
          <div className="space-y-2">
            <label className="text-xs font-semibold uppercase tracking-[0.18em] text-slate-500">
              Confirmed Tempo Transaction Hash
            </label>
            <input
              className="rounded-2xl border border-slate-700 bg-slate-950 p-4"
              value={txHash}
              onChange={(e) => setTxHash(e.target.value)}
              placeholder="0x..."
            />
            <p className="text-sm text-slate-500">
              Required. This anchors the verification to a finalized Parly execution on Tempo.
            </p>
          </div>
          <div className="space-y-2">
            <label className="text-xs font-semibold uppercase tracking-[0.18em] text-slate-500">
              Child Key
            </label>
            <input
              className="rounded-2xl border border-slate-700 bg-slate-950 p-4"
              value={childKey}
              onChange={(e) => setChildKey(e.target.value)}
              placeholder="0x..."
            />
            <p className="text-sm text-slate-500">
              Required. This narrows the verification to one disclosed payout lane only.
            </p>
          </div>
        </div>

        <button
          className="mt-4 w-full rounded-2xl bg-cyan-500 px-4 py-4 font-bold text-slate-950"
          onClick={verify}
        >
          {loading ? "Verifying payout scope..." : "Verify Payout Scope"}
        </button>

        <div className="mt-4 flex flex-wrap gap-2">
          <TruthBadge truth="chain_indexed" />
          <TruthBadge truth="onchain_calldata_decoded" />
          <TruthBadge truth="locally_reconstructed" />
          <TruthBadge truth="provenance_record" />
        </div>

        {showSampleReport && (
          <div className="mt-4 rounded-2xl border border-white/10 bg-slate-950/50 p-4 text-sm text-slate-300">
            <div className="font-semibold text-white">Sample report structure</div>
            <p className="mt-2 text-slate-400">
              This preview shows the exact verification sections and truth labels without inventing
              sample settlement facts.
            </p>
            <ul className="mt-3 list-disc space-y-1 pl-5 text-slate-300">
              <li>Verified Settlement Facts</li>
              <li>Matched Payout Scope</li>
              <li>Additional Context</li>
            </ul>
          </div>
        )}

        {loading && (
          <div className="mt-4 rounded-2xl border border-cyan-400/20 bg-cyan-500/10 p-4 text-sm text-cyan-100">
            <p>Querying indexed settlement data...</p>
            <p className="mt-2">Matching child-key scope...</p>
            <p className="mt-2">Resolving disclosed payout lane...</p>
          </div>
        )}
      </div>

      {error && (
        <div className="rounded-2xl border border-red-600/50 bg-red-950/40 p-4 text-sm text-red-200">
          {error}
        </div>
      )}

      {result && (
        <div className="space-y-4">
          <div className="rounded-3xl border border-emerald-500/30 bg-emerald-500/10 p-6">
            <div className="inline-flex rounded-full border border-emerald-400/30 bg-emerald-500/10 px-3 py-1 text-[11px] font-semibold uppercase tracking-[0.18em] text-emerald-100">
              Verified
            </div>
            <div className="flex flex-wrap gap-2">
              {result.truth.map((truth) => (
                <TruthBadge key={truth} truth={truth} />
              ))}
            </div>
            <h2 className="mt-4 text-2xl font-black text-white">Parly Payout Scope Confirmed</h2>
            <p className="mt-3 text-sm text-emerald-50/90">
              This child key matches a real indexed Tempo execution and resolves to one disclosed payout lane.
            </p>
            <div className="mt-4 grid gap-3 lg:grid-cols-3">
              <div className="rounded-2xl border border-white/10 bg-slate-950/60 p-4">
                <div className="text-xs uppercase tracking-[0.18em] text-slate-500">Verified Settlement Facts</div>
                <div className="mt-3 space-y-2 text-sm text-slate-200">
                  <div>Protocol: Parly</div>
                  <div>Asset: {assetLabel(result.assetId)}</div>
                  <div className="break-all">Confirmed Tempo Transaction Hash: {result.txHash}</div>
                  <div>Verified Date: {new Date(result.timestamp * 1000).toLocaleString()}</div>
                  <div>Protocol Fee: {formatUnits(BigInt(result.protocolFee), 6)} {assetLabel(result.assetId)}</div>
                  <div>Executor Fee: {formatUnits(BigInt(result.executorFee), 6)} {assetLabel(result.assetId)}</div>
                </div>
              </div>

              <div className="rounded-2xl border border-white/10 bg-slate-950/60 p-4">
                <div className="text-xs uppercase tracking-[0.18em] text-slate-500">Matched Payout Scope</div>
                <div className="mt-3 space-y-2 text-sm text-slate-200">
                  <div className="break-all">Recipient Address: {result.output.recipient}</div>
                  <div>Payout Amount: {formatUnits(BigInt(result.output.amount), 6)} {assetLabel(result.assetId)}</div>
                  <div>Destination Chain: {result.output.destinationChain}</div>
                  <div>Destination EID: {result.output.destEid}</div>
                  <div>Route Type: {result.output.routeType === "same_chain" ? "Same-Chain" : "Cross-Chain"}</div>
                  <div className="break-all">Child Key Scope: Single Disclosed Payout Lane ({result.childKey})</div>
                </div>
              </div>

              <div className="rounded-2xl border border-white/10 bg-slate-950/60 p-4">
                <div className="text-xs uppercase tracking-[0.18em] text-slate-500">Additional Context</div>
                <div className="mt-3 space-y-3 text-sm text-slate-300">
                  {result.local ? (
                    <div className="break-all">Source Note Commitment: {result.local.sourceNoteCommitment}</div>
                  ) : (
                    <div>No selective-disclosure local scope was available for this browser session.</div>
                  )}

                  {result.provenance ? (
                    <>
                      <div className="break-all">LayerZero GUID: {result.provenance.guid}</div>
                      {result.provenance.sourceChain && <div>Source Chain: {result.provenance.sourceChain}</div>}
                      {result.provenance.sourceTxHash && (
                        <div className="break-all">Source Transaction Hash: {result.provenance.sourceTxHash}</div>
                      )}
                      {result.provenance.sourceSender && (
                        <div className="break-all">Source Sender: {result.provenance.sourceSender}</div>
                      )}
                      <div className="break-all">Tempo Settlement Anchor: {result.provenance.settlementTxHash}</div>
                    </>
                  ) : (
                    <div>No origin provenance record was available for this verified child key.</div>
                  )}
                </div>
              </div>
            </div>

            <div className="mt-4 rounded-2xl border border-white/10 bg-slate-950/50 p-4 text-sm text-slate-300">
              <div className="font-semibold text-white">
                Parly has verified that this child key resolves to a real payout lane anchored to the referenced Tempo execution.
              </div>
              <p className="mt-2 text-slate-400">
                This verification is selective by design. It confirms the disclosed lane without
                exposing the remainder of the private execution.
              </p>
              <p className="mt-2 text-slate-500">
                Shared route delivery cost remains execution-scoped and is not attributed to one disclosed payout lane in this verification surface.
              </p>
            </div>
          </div>
        </div>
      )}
    </div>
  )
}

41.6 File: apps/web/src/app/docs/page.tsx

const docBlocks = [
  {
    title: "Privacy and note model",
    copy: "Parly keeps deterministic recovery, sealed-box envelopes, and the 10-output padded execution model intact. Browser storage remains optional and non-authoritative."
  },
  {
    title: "MPP boundary",
    copy: "MPP compatibility lives only at the SDK and MCP boundary. It wraps immediate Parly execution, adds no on-chain session state, and does not alter pool or circuit semantics."
  },
  {
    title: "Truth boundaries",
    copy: "Direct Tempo deposits always have Tempo settlement data. Spoke-origin provenance appears only when indexed source and settlement events share the same guid."
  },
  {
    title: "Operational honesty",
    copy: "Relay mode stops at submission to Waku. Direct browser mode only shows success after confirmed Tempo receipt. Uncertain states stay explicit."
  },
  {
    title: "Launch scope",
    copy: "V16.9.9 stays stablecoin-only, keeps Waku private relay plus self-relay fallback, restores an MPP-compatible adapter only at the SDK/MCP boundary, and does not reintroduce scheduling, swaps, passkeys, or dual identity systems."
  }
]

export default function DocsPage() {
  return (
    <div className="space-y-6">
      <div>
        <p className="text-xs uppercase tracking-[0.3em] text-cyan-300">Docs</p>
        <h1 className="mt-3 text-4xl font-black tracking-tight text-white">Protocol truth boundaries and launch notes</h1>
        <p className="mt-3 max-w-3xl text-sm text-slate-400">
          This route is a quick product-facing reference for the live launch surface. It complements the deeper runbook without changing protocol behavior.
        </p>
      </div>

      <div className="grid gap-4 xl:grid-cols-2">
        {docBlocks.map((block) => (
          <div key={block.title} className="rounded-3xl border border-white/10 bg-white/[0.03] p-6">
            <div className="text-xl font-bold text-white">{block.title}</div>
            <p className="mt-3 text-sm leading-6 text-slate-400">{block.copy}</p>
          </div>
        ))}
      </div>
    </div>
  )
}

41.7 File: apps/web/src/app/sdk/page.tsx

const mppEnabled = (process.env.NEXT_PUBLIC_ENABLE_MPP_ADAPTER ?? "true").toLowerCase() !== "false"

const sample = `import { ParlySDK, ParlyMppAdapter } from "@parly/sdk"

const sdk = new ParlySDK({
  privateKeyHex: process.env.AGENT_PRIVATE_KEY as \`0x\${string}\`,
  tempoChainId: Number(process.env.TEMPO_CHAIN_ID),
  tempoRpcUrl: process.env.TEMPO_RPC_URL!
})

const mpp = new ParlyMppAdapter(sdk, {
  serviceName: process.env.MPP_SERVICE_NAME,
  serviceVersion: process.env.MPP_SERVICE_VERSION
})

const session = await mpp.createSession({
  sessionId: "invoice-2026-03-27",
  counterparty: "0x1111111111111111111111111111111111111111",
  assetId: 1,
  spendLimit: "25.0",
  destinationEid: Number(process.env.TEMPO_LZ_EID),
  poolAddress: process.env.NEXT_PUBLIC_USDC_POOL as \`0x\${string}\`
})

const outcome = await mpp.settleFromSession({
  sessionId: session.sessionId,
  destination: "0x1111111111111111111111111111111111111111",
  amount: "25.0"
}, session)`

export default function SdkPage() {
  return (
    <div className="space-y-6">
      <div>
        <p className="text-xs uppercase tracking-[0.3em] text-cyan-300">Agentic SDK</p>
        <h1 className="mt-3 text-4xl font-black tracking-tight text-white">Deterministic recovery for operators and agents</h1>
        <p className="mt-3 max-w-3xl text-sm text-slate-400">
          The SDK mirrors protocol reality: deterministic recovery, real note reconstruction, exact approvals, honest pending-confirmation handling, and an optional MPP-compatible adapter that stays outside protocol-core settlement.
        </p>
      </div>

      <div className="grid gap-4 xl:grid-cols-[0.95fr_1.05fr]">
        <div className="rounded-3xl border border-white/10 bg-white/[0.03] p-6">
          <div className="text-xl font-bold text-white">Shipped surface</div>
          <ul className="mt-4 space-y-3 text-sm text-slate-300">
            <li>Deterministic recovery key derivation from the fixed auth message.</li>
            <li>Envelope decryption and Merkle reconstruction from indexed history.</li>
            <li>Private batch withdrawal execution with exact-amount fee approvals.</li>
            <li>Pending-confirmation honesty for uncertain post-broadcast outcomes.</li>
            <li>Optional MPP-compatible session descriptors that wrap the existing SDK path instead of replacing it.</li>
          </ul>
          <p className="mt-4 text-xs uppercase tracking-[0.24em] text-slate-500">
            MPP adapter {mppEnabled ? "enabled in this build" : "disabled in this build"}
          </p>
        </div>

        <div className="rounded-3xl border border-white/10 bg-slate-950/80 p-6">
          <div className="text-xs uppercase tracking-[0.22em] text-slate-500">Example</div>
          <pre className="mt-4 overflow-x-auto whitespace-pre-wrap text-sm text-slate-200">
            <code>{sample}</code>
          </pre>
        </div>
      </div>
    </div>
  )
}

41.8 File: apps/web/src/app/vault-admin/page.tsx

import AdminVault from "../../components/AdminVault"

export default function VaultAdminPage() {
  return (
    <div className="space-y-6">
      <div>
        <p className="text-xs uppercase tracking-[0.3em] text-cyan-300">Vault admin</p>
        <h1 className="mt-3 text-4xl font-black tracking-tight text-white">Protocol treasury control surface</h1>
        <p className="mt-3 max-w-3xl text-sm text-slate-400">
          This hidden signer-only route wraps the existing ProtocolTreasury semantics in an
          operational UI without inventing new admin powers or weakening multisig constraints.
          Non-signers receive a hard unauthorized state and no readable vault data.
        </p>
      </div>

      <AdminVault />
    </div>
  )
}

41.9 File: apps/web/src/app/relayer/page.tsx

"use client"

import Link from "next/link"
import { useEffect, useMemo, useState } from "react"
import { getAddress, isAddress } from "viem"
import { useAccount, useConnect, useSignMessage } from "wagmi"
import {
  fetchOfficialRelayerRegistry,
  isLikelyRelayerPublicKey,
  type RelayerRegistryResponse
} from "../../lib/relayer-registry"

type AlertState =
  | { kind: "success"; title: string; body: string }
  | { kind: "error"; title: string; body: string }
  | { kind: "warning"; title: string; body: string }
  | null

function mapRegistryError(code: string | undefined) {
  switch (code) {
    case "INVALID_ADDRESS":
      return "Invalid relayer execution address"
    case "INVALID_PUBLIC_KEY":
      return "Invalid relayer public key"
    case "DUPLICATE_ADDRESS":
      return "Relayer already exists"
    case "DUPLICATE_PUBLIC_KEY":
      return "Public key already exists"
    case "INVALID_SIGNATURE":
      return "Signature verification failed"
    case "RATE_LIMITED":
      return "Too many attempts"
    case "REGISTRY_FULL":
      return "Registry is currently full"
    default:
      return "Registration failed"
  }
}

export default function RelayerPage() {
  const { address } = useAccount()
  const { connectors, connectAsync } = useConnect()
  const { signMessageAsync } = useSignMessage()

  const [executionAddress, setExecutionAddress] = useState("")
  const [publicKeyB64, setPublicKeyB64] = useState("")
  const [confirmed, setConfirmed] = useState(false)
  const [submitting, setSubmitting] = useState(false)
  const [alert, setAlert] = useState<AlertState>(null)
  const [registry, setRegistry] = useState<RelayerRegistryResponse | null>(null)

  const normalizedExecutionAddress = useMemo(() => {
    if (!isAddress(executionAddress as `0x${string}`)) return null
    return getAddress(executionAddress).toLowerCase() as `0x${string}`
  }, [executionAddress])

  const addressLooksValid = !!normalizedExecutionAddress
  const publicKeyLooksValid = isLikelyRelayerPublicKey(publicKeyB64)
  const registryOpen = registry?.registryOpen ?? true
  const liveCount = registry?.liveCount ?? 0
  const maxCount = registry?.maxCount ?? 10

  useEffect(() => {
    let cancelled = false

    fetchOfficialRelayerRegistry()
      .then((nextRegistry) => {
        if (!cancelled) {
          setRegistry(nextRegistry)
        }
      })
      .catch((error: any) => {
        if (!cancelled) {
          setAlert({
            kind: "error",
            title: "Registration failed",
            body: error?.message || "Registration failed"
          })
        }
      })

    return () => {
      cancelled = true
    }
  }, [])

  async function submit() {
    if (!normalizedExecutionAddress || !publicKeyLooksValid || !confirmed || submitting || !registryOpen) {
      return
    }

    setSubmitting(true)
    setAlert(null)

    try {
      let connectedAddress = address

      if (!connectedAddress) {
        const connector = connectors[0]
        if (!connector) {
          throw new Error("Signature verification failed")
        }
        const connection = await connectAsync({ connector })
        connectedAddress = connection.accounts[0] as `0x${string}`
      }

      if (!connectedAddress || connectedAddress.toLowerCase() !== normalizedExecutionAddress) {
        throw new Error("Signature verification failed")
      }

      const initRes = await fetch("/api/relayers/register/init", {
        method: "POST",
        headers: { "content-type": "application/json" },
        body: JSON.stringify({
          executionAddress: normalizedExecutionAddress
        })
      })
      const initJson = await initRes.json()
      if (!initRes.ok) {
        throw new Error(mapRegistryError(initJson.error))
      }

      const signature = await signMessageAsync({
        message: initJson.message
      })

      const confirmRes = await fetch("/api/relayers/register/confirm", {
        method: "POST",
        headers: { "content-type": "application/json" },
        body: JSON.stringify({
          executionAddress: normalizedExecutionAddress,
          publicKeyB64: publicKeyB64.trim(),
          signature,
          nonce: initJson.nonce
        })
      })
      const confirmJson = await confirmRes.json()
      if (!confirmRes.ok) {
        throw new Error(mapRegistryError(confirmJson.error))
      }

      const nextRegistry = await fetchOfficialRelayerRegistry()
      setRegistry(nextRegistry)
      setAlert({
        kind: "success",
        title: "Relayer registered successfully",
        body: "Your relayer is now active in the official Parly registry."
      })
    } catch (error: any) {
      const body = mapRegistryError(error?.message && error.message.toUpperCase().includes("_") ? error.message : undefined)
      setAlert({
        kind: error?.message === "Registry is currently full" ? "warning" : "error",
        title: error?.message === "Registry is currently full" ? "Official registry is currently full" : "Registration failed",
        body:
          error?.message === "Registry is currently full"
            ? "New registrations reopen when an active slot becomes available."
            : body === "Registration failed"
            ? error?.message || "Registration failed"
            : body
      })
    } finally {
      setSubmitting(false)
    }
  }

  function clearForm() {
    setExecutionAddress("")
    setPublicKeyB64("")
    setConfirmed(false)
    setAlert(null)
  }

  const primaryDisabled =
    !registryOpen || !confirmed || !addressLooksValid || !publicKeyLooksValid || submitting

  return (
    <div className="mx-auto flex w-full max-w-3xl flex-col gap-6 py-10">
      <div>
        <p className="text-xs uppercase tracking-[0.3em] text-cyan-300">Parly Relay Registry</p>
        <h1 className="mt-3 text-4xl font-black tracking-tight text-white">Register a Relayer</h1>
        <p className="mt-3 max-w-2xl text-sm text-slate-400">
          This page is for relayer operators only. Read the relayer docs before registering your execution address and public key.
        </p>
      </div>

      <div className="flex flex-wrap items-center gap-3">
        <div className="rounded-full border border-white/10 bg-white/[0.03] px-4 py-2 text-xs font-semibold uppercase tracking-[0.18em] text-slate-200">
          Live Relayers: {liveCount} / {maxCount}
        </div>
        <div className="rounded-full border border-white/10 bg-white/[0.03] px-4 py-2 text-xs font-semibold uppercase tracking-[0.18em] text-slate-200">
          Registry: {registryOpen ? "Open" : "Full"}
        </div>
      </div>

      <div className="flex flex-wrap gap-3">
        <Link href="/docs/relayers" className="rounded-full border border-white/10 px-4 py-3 text-sm font-semibold text-slate-200 transition hover:border-white/30 hover:text-white">
          Relayer Docs
        </Link>
        <Link href="/docs/quickstart" className="rounded-full border border-white/10 px-4 py-3 text-sm font-semibold text-slate-200 transition hover:border-white/30 hover:text-white">
          Operator Quickstart
        </Link>
        <Link href="/sdk" className="rounded-full border border-white/10 px-4 py-3 text-sm font-semibold text-slate-200 transition hover:border-white/30 hover:text-white">
          SDK / MCP Docs
        </Link>
      </div>

      <p className="text-sm text-slate-500">Need setup help first? Read the docs before registering.</p>

      {!registryOpen && (
        <div className="rounded-2xl border border-amber-400/30 bg-amber-500/10 p-4 text-sm text-amber-100">
          <div className="font-semibold text-amber-50">Official registry is currently full</div>
          <p className="mt-2">New registrations reopen when an active slot becomes available.</p>
        </div>
      )}

      {alert && (
        <div
          className={`rounded-2xl p-4 text-sm ${
            alert.kind === "success"
              ? "border border-emerald-400/30 bg-emerald-500/10 text-emerald-100"
              : alert.kind === "warning"
              ? "border border-amber-400/30 bg-amber-500/10 text-amber-100"
              : "border border-red-500/30 bg-red-500/10 text-red-100"
          }`}
        >
          <div className="font-semibold text-white">{alert.title}</div>
          <p className="mt-2">{alert.body}</p>
        </div>
      )}

      <div className="rounded-3xl border border-white/10 bg-white/[0.03] p-6">
        <div className="text-xs uppercase tracking-[0.22em] text-cyan-300">Relayer Registration</div>
        <p className="mt-3 text-sm text-slate-400">
          Enter the relayer execution address and the Base64 public key generated from your relayer setup.
        </p>

        <div className="mt-6 space-y-4">
          <label className="block space-y-2">
            <span className="text-sm font-semibold text-white">Relayer Execution Address</span>
            <input
              value={executionAddress}
              onChange={(event) => setExecutionAddress(event.target.value)}
              placeholder="0x..."
              className="w-full rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white"
            />
          </label>

          <label className="block space-y-2">
            <span className="text-sm font-semibold text-white">Relayer Public Key</span>
            <input
              value={publicKeyB64}
              onChange={(event) => setPublicKeyB64(event.target.value)}
              placeholder="BASE64_X25519_PUBLIC_KEY"
              className="w-full rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white"
            />
          </label>

          <label className="flex items-start gap-3 rounded-2xl border border-white/10 bg-slate-950/70 p-4 text-sm text-slate-300">
            <input
              type="checkbox"
              checked={confirmed}
              onChange={(event) => setConfirmed(event.target.checked)}
              className="mt-1 h-4 w-4 rounded border-white/20 bg-slate-950"
            />
            <span>
              I confirm this relayer is independently operated, the submitted details are accurate, and I understand registration does not create protocol exclusivity or guarantee user flow.
            </span>
          </label>
        </div>

        <div className="mt-6 flex flex-wrap gap-3">
          <button
            disabled={primaryDisabled}
            onClick={submit}
            className={`rounded-full px-5 py-3 text-sm font-semibold transition ${
              primaryDisabled
                ? "cursor-not-allowed bg-slate-800 text-slate-500"
                : "bg-cyan-400 text-slate-950 hover:bg-cyan-300"
            }`}
          >
            {registryOpen ? "Connect Wallet & Sign Registration" : "Registry Full"}
          </button>
          <button
            type="button"
            onClick={clearForm}
            className="rounded-full border border-white/10 px-5 py-3 text-sm font-semibold text-slate-200 transition hover:border-white/30 hover:text-white"
          >
            Clear
          </button>
        </div>
      </div>

      <div className="rounded-3xl border border-white/10 bg-slate-950/60 p-6">
        <div className="text-xs uppercase tracking-[0.22em] text-cyan-300">Registry Rules</div>
        <div className="mt-4 space-y-3 text-sm text-slate-300">
          <p>- Official frontend discovery is capped at 10 active relayers.</p>
          <p>- Registration is automatic if validation passes and a slot is available.</p>
          <p>- Duplicate, invalid, or unavailable relayers may be rejected automatically.</p>
          <p>- Inactive relayers are removed from live official discovery.</p>
          <p>- Inactive relayers must re-register to regain official discovery.</p>
          <p>- Protocol participation remains open even if a relayer is not listed here.</p>
        </div>
      </div>

      <p className="text-sm text-slate-500">
        Need setup instructions first? Read the relayer docs before registering.
      </p>
    </div>
  )
}

41.10 File: apps/web/src/app/docs/relayers/page.tsx

import Link from "next/link"

export default function RelayerDocsPage() {
  return (
    <div className="mx-auto flex w-full max-w-3xl flex-col gap-6 py-10">
      <div>
        <p className="text-xs uppercase tracking-[0.3em] text-cyan-300">Relayer Docs</p>
        <h1 className="mt-3 text-4xl font-black tracking-tight text-white">Official relayer discovery and operator flow</h1>
        <p className="mt-3 text-sm text-slate-400">
          Use <code>/relayer</code> to register your execution address and public key into the official runtime registry when you want official frontend discovery. Protocol participation remains open even when a relayer is not listed there.
        </p>
      </div>

      <div className="rounded-3xl border border-white/10 bg-white/[0.03] p-6 text-sm text-slate-300">
        <p>1. Run <code>pnpm keygen:box</code> to generate your sealed-box keypair.</p>
        <p className="mt-3">2. Start the daemon with <code>pnpm start</code> and keep <code>pnpm reconcile:pending:loop</code> live.</p>
        <p className="mt-3">3. If you want official discovery, set <code>RELAYER_REGISTRY_API_BASE_URL</code> and register the execution address plus public key at <code>/relayer</code>.</p>
      </div>

      <div className="flex flex-wrap gap-3">
        <Link href="/relayer" className="rounded-full bg-cyan-400 px-4 py-3 text-sm font-semibold text-slate-950 transition hover:bg-cyan-300">
          Open Relayer Registration
        </Link>
        <Link href="/docs/quickstart" className="rounded-full border border-white/10 px-4 py-3 text-sm font-semibold text-slate-200 transition hover:border-white/30 hover:text-white">
          Operator Quickstart
        </Link>
      </div>
    </div>
  )
}

41.11 File: apps/web/src/app/docs/quickstart/page.tsx

import Link from "next/link"

export default function QuickstartPage() {
  return (
    <div className="mx-auto flex w-full max-w-3xl flex-col gap-6 py-10">
      <div>
        <p className="text-xs uppercase tracking-[0.3em] text-cyan-300">Operator Quickstart</p>
        <h1 className="mt-3 text-4xl font-black tracking-tight text-white">Lean relayer launch checklist</h1>
        <p className="mt-3 text-sm text-slate-400">
          This route is a lightweight operator stub: keygen, start, reconcile, and register. It does not replace the deeper runbook.
        </p>
      </div>

      <div className="rounded-3xl border border-white/10 bg-white/[0.03] p-6 text-sm text-slate-300">
        <p>- Generate your Base64 X25519 public key with <code>pnpm keygen:box</code>.</p>
        <p className="mt-3">- Keep the relayer daemon and pending reconciler running.</p>
        <p className="mt-3">- Register at <code>/relayer</code> to join official frontend discovery when a slot is available.</p>
      </div>

      <div className="flex flex-wrap gap-3">
        <Link href="/relayer" className="rounded-full bg-cyan-400 px-4 py-3 text-sm font-semibold text-slate-950 transition hover:bg-cyan-300">
          Register a Relayer
        </Link>
        <Link href="/sdk" className="rounded-full border border-white/10 px-4 py-3 text-sm font-semibold text-slate-200 transition hover:border-white/30 hover:text-white">
          SDK / MCP Docs
        </Link>
      </div>
    </div>
  )
}

41.12 File: apps/web/src/app/api/relayers/route.ts

import { NextResponse } from "next/server"
import {
  getLiveRelayerRegistry,
  registryRouteError
} from "../../../lib/relayer-registry-server"

export const runtime = "nodejs"
export const dynamic = "force-dynamic"

export async function GET() {
  try {
    const snapshot = await getLiveRelayerRegistry()
    return NextResponse.json(snapshot)
  } catch (error) {
    return registryRouteError(error)
  }
}

41.13 File: apps/web/src/app/api/relayers/register/init/route.ts

import { NextRequest, NextResponse } from "next/server"
import {
  createRegistrationChallenge,
  registryRouteError
} from "../../../../../lib/relayer-registry-server"

export const runtime = "nodejs"
export const dynamic = "force-dynamic"

export async function POST(req: NextRequest) {
  try {
    const body = await req.json()
    const challenge = await createRegistrationChallenge(body.executionAddress)
    return NextResponse.json({
      nonce: challenge.nonce,
      message: challenge.message
    })
  } catch (error) {
    return registryRouteError(error)
  }
}

41.14 File: apps/web/src/app/api/relayers/register/confirm/route.ts

import { NextRequest, NextResponse } from "next/server"
import {
  confirmRelayerRegistration,
  getRequestIp,
  registryRouteError
} from "../../../../../lib/relayer-registry-server"

export const runtime = "nodejs"
export const dynamic = "force-dynamic"

export async function POST(req: NextRequest) {
  try {
    const body = await req.json()
    const result = await confirmRelayerRegistration({
      executionAddress: body.executionAddress,
      publicKeyB64: body.publicKeyB64,
      signature: body.signature,
      nonce: body.nonce,
      ip: getRequestIp(req)
    })

    return NextResponse.json(result)
  } catch (error) {
    return registryRouteError(error)
  }
}

41.15 File: apps/web/src/app/api/relayers/heartbeat/route.ts

import { NextRequest, NextResponse } from "next/server"
import {
  recordRelayerHeartbeat,
  registryRouteError
} from "../../../../lib/relayer-registry-server"

export const runtime = "nodejs"
export const dynamic = "force-dynamic"

export async function POST(req: NextRequest) {
  try {
    const body = await req.json()
    const result = await recordRelayerHeartbeat({
      executionAddress: body.executionAddress,
      event: body.event,
      timestamp: body.timestamp,
      signature: body.signature
    })
    return NextResponse.json(result)
  } catch (error) {
    return registryRouteError(error)
  }
}

41.16 File: apps/web/src/app/api/cron/relayers/inactive/route.ts

import { NextRequest, NextResponse } from "next/server"
import { requireEnv } from "@parly/env-utils"
import { pruneInactiveRelayers } from "../../../../../lib/relayer-registry-server"

export const runtime = "nodejs"
export const dynamic = "force-dynamic"

export async function POST(req: NextRequest) {
  const auth = req.headers.get("x-parly-cron-secret") || ""
  if (auth !== requireEnv("RELAYER_REGISTRY_CRON_SECRET")) {
    return NextResponse.json({ error: "UNKNOWN_ERROR", message: "Registration failed" }, { status: 401 })
  }

  const pruned = await pruneInactiveRelayers()
  return NextResponse.json({ success: true, pruned })
}


---

42. Corrected Ponder indexer

42.1 Architectural reasoning

The indexer schema stays minimal, public, and deterministic.

It materializes:
	*	deposits,
	*	leaves,
	*	envelopes,
	*	batch metadata,
	*	source-chain spoke shield dispatches,
	*	and Tempo-side ingress settlement anchors.

Nothing private is inferred.
Nothing local is trusted.
Source provenance exists only when a real source dispatch event and a real Tempo settlement event share the same LayerZero `guid`.

42.2 File: apps/indexer/package.json

{
  "name": "@parly/indexer",
  "private": true,
  "scripts": {
    "dev": "ponder dev",
    "start": "ponder start",
    "build": "ponder build",
    "codegen": "ponder codegen"
  },
  "dependencies": {
    "dotenv": "16.4.7",
    "hono": "^4.7.2",
    "ponder": "^0.13.9",
    "viem": "2.43.5"
  },
  "devDependencies": {
    "typescript": "5.8.2"
  }
}

42.3 File: apps/indexer/ponder.config.ts

import "dotenv/config"
import { createConfig } from "ponder"
import { http, parseAbiItem } from "viem"
import { requireEnv, requirePositiveInt } from "@parly/env-utils"

const ponderTempoRpcUrl = requireEnv("PONDER_RPC_URL_TEMPO")
const ponderTempoChainId = requirePositiveInt("NEXT_PUBLIC_TEMPO_CHAIN_ID")
const ethereumChainId = requirePositiveInt("ETHEREUM_CHAIN_ID")
const arbitrumChainId = requirePositiveInt("ARBITRUM_CHAIN_ID")
const baseChainId = requirePositiveInt("BASE_CHAIN_ID")
const bscChainId = requirePositiveInt("BSC_CHAIN_ID")
const ethereumRpcUrl = requireEnv("ETHEREUM_RPC_URL")
const arbitrumRpcUrl = requireEnv("ARBITRUM_RPC_URL")
const baseRpcUrl = requireEnv("BASE_RPC_URL")
const bscRpcUrl = requireEnv("BSC_RPC_URL")

const poolAbi = [
  parseAbiItem("event Deposit(address indexed depositor, bytes32 indexed commitment, uint256 amount, uint32 leafIndex)"),
  parseAbiItem("event LeafInserted(bytes32 indexed commitment, uint32 indexed leafIndex, uint256 indexed assetId, uint8 kind)"),
  parseAbiItem("event NoteEnvelope(bytes32 indexed commitment, uint8 indexed kind, uint256 indexed assetId, bytes envelope)"),
  parseAbiItem("event BatchWithdrawal(bytes32 indexed nullifierHash, address indexed executor, uint256 indexed assetId, uint256 protocolFee, uint256 executorFee, bytes32[] childKeys)")
] as const

const composerAbi = [
  parseAbiItem(
    "event IngressCompleted(bytes32 indexed guid, bytes32 indexed commitment, uint32 indexed srcEid, uint16 assetId, address depositor, uint256 amountLD, uint256 innerCommitment, uint32 settlementLeafIndex, bytes32 composeFrom)"
  )
] as const

const gatewayAbi = [
  parseAbiItem(
    "event ShieldedCrossChain(address indexed sender, uint16 indexed assetId, bytes32 indexed guid, uint256 amount, uint256 innerCommitment)"
  )
] as const

export default createConfig({
  database: {
    kind: "postgres",
    connectionString: process.env.DATABASE_URL || ""
  },
  networks: {
    tempo: {
      chainId: ponderTempoChainId,
      transport: http(ponderTempoRpcUrl)
    },
    ethereum: {
      chainId: ethereumChainId,
      transport: http(ethereumRpcUrl)
    },
    arbitrum: {
      chainId: arbitrumChainId,
      transport: http(arbitrumRpcUrl)
    },
    base: {
      chainId: baseChainId,
      transport: http(baseRpcUrl)
    },
    bnb: {
      chainId: bscChainId,
      transport: http(bscRpcUrl)
    }
  },
  contracts: {
    TempoShieldedPoolUSDC: {
      network: "tempo",
      abi: poolAbi,
      address: process.env.NEXT_PUBLIC_USDC_POOL as `0x${string}`,
      startBlock: Number(process.env.START_BLOCK_USDC || 0)
    },
    TempoShieldedPoolUSDT: {
      network: "tempo",
      abi: poolAbi,
      address: process.env.NEXT_PUBLIC_USDT_POOL as `0x${string}`,
      startBlock: Number(process.env.START_BLOCK_USDT || 0)
    },
    ParlyHubComposer: {
      network: "tempo",
      abi: composerAbi,
      address: process.env.NEXT_PUBLIC_HUB_COMPOSER as `0x${string}`,
      startBlock: Number(process.env.START_BLOCK_HUB_COMPOSER || 0)
    },
    SpokeGatewayUSDCEthereum: {
      network: "ethereum",
      abi: gatewayAbi,
      address: process.env.NEXT_PUBLIC_SPOKE_USDC_GATEWAY_ETHEREUM as `0x${string}`,
      startBlock: Number(process.env.START_BLOCK_SPOKE_USDC_ETHEREUM || 0)
    },
    SpokeGatewayUSDCArbitrum: {
      network: "arbitrum",
      abi: gatewayAbi,
      address: process.env.NEXT_PUBLIC_SPOKE_USDC_GATEWAY_ARBITRUM as `0x${string}`,
      startBlock: Number(process.env.START_BLOCK_SPOKE_USDC_ARBITRUM || 0)
    },
    SpokeGatewayUSDCBase: {
      network: "base",
      abi: gatewayAbi,
      address: process.env.NEXT_PUBLIC_SPOKE_USDC_GATEWAY_BASE as `0x${string}`,
      startBlock: Number(process.env.START_BLOCK_SPOKE_USDC_BASE || 0)
    },
    SpokeGatewayUSDCBNB: {
      network: "bnb",
      abi: gatewayAbi,
      address: process.env.NEXT_PUBLIC_SPOKE_USDC_GATEWAY_BNB as `0x${string}`,
      startBlock: Number(process.env.START_BLOCK_SPOKE_USDC_BNB || 0)
    },
    SpokeGatewayUSDTEthereum: {
      network: "ethereum",
      abi: gatewayAbi,
      address: process.env.NEXT_PUBLIC_SPOKE_USDT_GATEWAY_ETHEREUM as `0x${string}`,
      startBlock: Number(process.env.START_BLOCK_SPOKE_USDT_ETHEREUM || 0)
    },
    SpokeGatewayUSDTArbitrum: {
      network: "arbitrum",
      abi: gatewayAbi,
      address: process.env.NEXT_PUBLIC_SPOKE_USDT_GATEWAY_ARBITRUM as `0x${string}`,
      startBlock: Number(process.env.START_BLOCK_SPOKE_USDT_ARBITRUM || 0)
    },
    SpokeGatewayUSDTBase: {
      network: "base",
      abi: gatewayAbi,
      address: process.env.NEXT_PUBLIC_SPOKE_USDT_GATEWAY_BASE as `0x${string}`,
      startBlock: Number(process.env.START_BLOCK_SPOKE_USDT_BASE || 0)
    },
    SpokeGatewayUSDTBNB: {
      network: "bnb",
      abi: gatewayAbi,
      address: process.env.NEXT_PUBLIC_SPOKE_USDT_GATEWAY_BNB as `0x${string}`,
      startBlock: Number(process.env.START_BLOCK_SPOKE_USDT_BNB || 0)
    }
  }
})

42.4 File: apps/indexer/ponder.schema.ts

import { onchainTable } from "ponder"

export const depositEvent = onchainTable("deposit_event", (t) => ({
  id: t.text().primaryKey(),
  depositor: t.text().notNull(),
  commitment: t.text().notNull(),
  amount: t.bigint().notNull(),
  leafIndex: t.integer().notNull(),
  assetId: t.integer().notNull(),
  txHash: t.text().notNull(),
  timestamp: t.bigint().notNull()
}))

export const ingressSettlementEvent = onchainTable("ingress_settlement_event", (t) => ({
  guid: t.text().primaryKey(),
  commitment: t.text().notNull(),
  assetId: t.integer().notNull(),
  depositor: t.text().notNull(),
  amount: t.bigint().notNull(),
  innerCommitment: t.bigint().notNull(),
  settlementLeafIndex: t.integer().notNull(),
  settlementChain: t.text().notNull(),
  settlementEid: t.integer().notNull(),
  sourceEid: t.integer().notNull(),
  composeFrom: t.text().notNull(),
  txHash: t.text().notNull(),
  timestamp: t.bigint().notNull()
}))

export const spokeShieldDispatchEvent = onchainTable("spoke_shield_dispatch_event", (t) => ({
  guid: t.text().primaryKey(),
  assetId: t.integer().notNull(),
  sender: t.text().notNull(),
  amount: t.bigint().notNull(),
  innerCommitment: t.bigint().notNull(),
  sourceChain: t.text().notNull(),
  sourceEid: t.integer().notNull(),
  gateway: t.text().notNull(),
  txHash: t.text().notNull(),
  timestamp: t.bigint().notNull()
}))

export const leafInsertedEvent = onchainTable("leaf_inserted_event", (t) => ({
  id: t.text().primaryKey(),
  commitment: t.text().notNull(),
  leafIndex: t.integer().notNull(),
  assetId: t.integer().notNull(),
  kind: t.integer().notNull(),
  txHash: t.text().notNull(),
  timestamp: t.bigint().notNull()
}))

export const noteEnvelopeEvent = onchainTable("note_envelope_event", (t) => ({
  id: t.text().primaryKey(),
  commitment: t.text().notNull(),
  kind: t.integer().notNull(),
  assetId: t.integer().notNull(),
  envelope: t.text().notNull(),
  txHash: t.text().notNull(),
  timestamp: t.bigint().notNull()
}))

export const batchWithdrawalEvent = onchainTable("batch_withdrawal_event", (t) => ({
  id: t.text().primaryKey(),
  nullifierHash: t.text().notNull(),
  executor: t.text().notNull(),
  protocolFee: t.bigint().notNull(),
  executorFee: t.bigint().notNull(),
  childKeys: t.json().notNull(),
  assetId: t.integer().notNull(),
  txHash: t.text().notNull(),
  timestamp: t.bigint().notNull()
}))

42.5 File: apps/indexer/src/index.ts

import { ponder } from "ponder:registry"
import { requirePositiveEidValue } from "@parly/env-utils"
import {
  depositEvent,
  ingressSettlementEvent,
  spokeShieldDispatchEvent,
  leafInsertedEvent,
  noteEnvelopeEvent,
  batchWithdrawalEvent
} from "../ponder.schema"

const tempoSettlementEid = requirePositiveEidValue(
  process.env.NEXT_PUBLIC_TEMPO_LZ_EID || process.env.TEMPO_LZ_EID,
  "TEMPO_LZ_EID"
)

const sourceEids = {
  ethereum: requirePositiveEidValue(
    process.env.NEXT_PUBLIC_LZ_EID_ETHEREUM || process.env.LZ_EID_ETHEREUM,
    "LZ_EID_ETHEREUM"
  ),
  arbitrum: requirePositiveEidValue(
    process.env.NEXT_PUBLIC_LZ_EID_ARBITRUM || process.env.LZ_EID_ARBITRUM,
    "LZ_EID_ARBITRUM"
  ),
  base: requirePositiveEidValue(
    process.env.NEXT_PUBLIC_LZ_EID_BASE || process.env.LZ_EID_BASE,
    "LZ_EID_BASE"
  ),
  bnb: requirePositiveEidValue(
    process.env.NEXT_PUBLIC_LZ_EID_BNB || process.env.LZ_EID_BNB,
    "LZ_EID_BNB"
  )
}

async function indexDeposit(
  assetId: 1 | 2,
  event: any,
  context: any
) {
  await context.db.insert(depositEvent).values({
    id: event.log.id,
    depositor: event.args.depositor,
    commitment: event.args.commitment,
    amount: event.args.amount,
    leafIndex: Number(event.args.leafIndex),
    assetId,
    txHash: event.transaction.hash,
    timestamp: event.block.timestamp
  })
}

async function indexIngressSettlement(event: any, context: any) {
  await context.db.insert(ingressSettlementEvent).values({
    guid: event.args.guid,
    commitment: event.args.commitment,
    assetId: Number(event.args.assetId),
    depositor: event.args.depositor,
    amount: event.args.amountLD,
    innerCommitment: event.args.innerCommitment,
    settlementLeafIndex: Number(event.args.settlementLeafIndex),
    settlementChain: "tempo",
    settlementEid: tempoSettlementEid,
    sourceEid: Number(event.args.srcEid),
    composeFrom: event.args.composeFrom,
    txHash: event.transaction.hash,
    timestamp: event.block.timestamp
  })
}

async function indexSpokeShieldDispatch(
  sourceChain: "ethereum" | "arbitrum" | "base" | "bnb",
  event: any,
  context: any
) {
  await context.db.insert(spokeShieldDispatchEvent).values({
    guid: event.args.guid,
    assetId: Number(event.args.assetId),
    sender: event.args.sender,
    amount: event.args.amount,
    innerCommitment: event.args.innerCommitment,
    sourceChain,
    sourceEid: sourceEids[sourceChain],
    gateway: event.log.address,
    txHash: event.transaction.hash,
    timestamp: event.block.timestamp
  })
}

ponder.on("TempoShieldedPoolUSDC:Deposit", async ({ event, context }) => {
  await indexDeposit(1, event, context)
})

ponder.on("TempoShieldedPoolUSDT:Deposit", async ({ event, context }) => {
  await indexDeposit(2, event, context)
})

ponder.on("ParlyHubComposer:IngressCompleted", async ({ event, context }) => {
  await indexIngressSettlement(event, context)
})

ponder.on("TempoShieldedPoolUSDC:LeafInserted", async ({ event, context }) => {
  await context.db.insert(leafInsertedEvent).values({
    id: event.log.id,
    commitment: event.args.commitment,
    leafIndex: Number(event.args.leafIndex),
    assetId: 1,
    kind: Number(event.args.kind),
    txHash: event.transaction.hash,
    timestamp: event.block.timestamp
  })
})

ponder.on("TempoShieldedPoolUSDT:LeafInserted", async ({ event, context }) => {
  await context.db.insert(leafInsertedEvent).values({
    id: event.log.id,
    commitment: event.args.commitment,
    leafIndex: Number(event.args.leafIndex),
    assetId: 2,
    kind: Number(event.args.kind),
    txHash: event.transaction.hash,
    timestamp: event.block.timestamp
  })
})

ponder.on("TempoShieldedPoolUSDC:NoteEnvelope", async ({ event, context }) => {
  await context.db.insert(noteEnvelopeEvent).values({
    id: event.log.id,
    commitment: event.args.commitment,
    kind: Number(event.args.kind),
    assetId: 1,
    envelope: event.args.envelope,
    txHash: event.transaction.hash,
    timestamp: event.block.timestamp
  })
})

ponder.on("TempoShieldedPoolUSDT:NoteEnvelope", async ({ event, context }) => {
  await context.db.insert(noteEnvelopeEvent).values({
    id: event.log.id,
    commitment: event.args.commitment,
    kind: Number(event.args.kind),
    assetId: 2,
    envelope: event.args.envelope,
    txHash: event.transaction.hash,
    timestamp: event.block.timestamp
  })
})

ponder.on("TempoShieldedPoolUSDC:BatchWithdrawal", async ({ event, context }) => {
  await context.db.insert(batchWithdrawalEvent).values({
    id: event.log.id,
    nullifierHash: event.args.nullifierHash,
    executor: event.args.executor,
    protocolFee: event.args.protocolFee,
    executorFee: event.args.executorFee,
    childKeys: JSON.stringify(event.args.childKeys),
    assetId: Number(event.args.assetId),
    txHash: event.transaction.hash,
    timestamp: event.block.timestamp
  })
})

ponder.on("TempoShieldedPoolUSDT:BatchWithdrawal", async ({ event, context }) => {
  await context.db.insert(batchWithdrawalEvent).values({
    id: event.log.id,
    nullifierHash: event.args.nullifierHash,
    executor: event.args.executor,
    protocolFee: event.args.protocolFee,
    executorFee: event.args.executorFee,
    childKeys: JSON.stringify(event.args.childKeys),
    assetId: Number(event.args.assetId),
    txHash: event.transaction.hash,
    timestamp: event.block.timestamp
  })
})

ponder.on("SpokeGatewayUSDCEthereum:ShieldedCrossChain", async ({ event, context }) => {
  await indexSpokeShieldDispatch("ethereum", event, context)
})

ponder.on("SpokeGatewayUSDCArbitrum:ShieldedCrossChain", async ({ event, context }) => {
  await indexSpokeShieldDispatch("arbitrum", event, context)
})

ponder.on("SpokeGatewayUSDCBase:ShieldedCrossChain", async ({ event, context }) => {
  await indexSpokeShieldDispatch("base", event, context)
})

ponder.on("SpokeGatewayUSDCBNB:ShieldedCrossChain", async ({ event, context }) => {
  await indexSpokeShieldDispatch("bnb", event, context)
})

ponder.on("SpokeGatewayUSDTEthereum:ShieldedCrossChain", async ({ event, context }) => {
  await indexSpokeShieldDispatch("ethereum", event, context)
})

ponder.on("SpokeGatewayUSDTArbitrum:ShieldedCrossChain", async ({ event, context }) => {
  await indexSpokeShieldDispatch("arbitrum", event, context)
})

ponder.on("SpokeGatewayUSDTBase:ShieldedCrossChain", async ({ event, context }) => {
  await indexSpokeShieldDispatch("base", event, context)
})

ponder.on("SpokeGatewayUSDTBNB:ShieldedCrossChain", async ({ event, context }) => {
  await indexSpokeShieldDispatch("bnb", event, context)
})

42.6 File: apps/indexer/src/api/index.ts

import { db } from "ponder:api"
import schema from "ponder:schema"
import { graphql } from "ponder"
import { sql } from "drizzle-orm"
import { Hono } from "hono"
import { cors } from "hono/cors"

const app = new Hono()
const rawAllowedOrigins = process.env.INDEXER_ALLOWED_ORIGINS || "*"
const allowedOrigins = rawAllowedOrigins
  .split(",")
  .map((origin) => origin.trim())
  .filter(Boolean)

app.use(
  "*",
  cors({
    origin:
      allowedOrigins.length === 0 || (allowedOrigins.length === 1 && allowedOrigins[0] === "*")
        ? "*"
        : allowedOrigins,
    allowMethods: ["GET", "POST", "OPTIONS"]
  })
)

app.get("/stats/public-deposit-volume", async (c) => {
  const rows = await db
    .select({
      assetId: schema.depositEvent.assetId,
      total: sql<string>`coalesce(sum(${schema.depositEvent.amount}), 0)`
    })
    .from(schema.depositEvent)
    .groupBy(schema.depositEvent.assetId)

  let usdc = 0n
  let usdt = 0n

  for (const row of rows) {
    const total = BigInt(String(row.total || "0"))
    if (Number(row.assetId) === 1) usdc = total
    if (Number(row.assetId) === 2) usdt = total
  }

  return c.json({
    usdc: usdc.toString(),
    usdt: usdt.toString(),
    total: (usdc + usdt).toString()
  })
})

app.get("/stats/shield-mining/:depositor", async (c) => {
  const depositor = String(c.req.param("depositor") || "").toLowerCase()
  if (!/^0x[0-9a-f]{40}$/.test(depositor)) {
    return c.json({ error: "invalid depositor" }, 400)
  }

  const rows = await db
    .select({
      amount: schema.depositEvent.amount
    })
    .from(schema.depositEvent)
    .where(sql`lower(${schema.depositEvent.depositor}) = ${depositor}`)

  const totalDeposited = rows.reduce(
    (sum, row) => sum + BigInt(String(row.amount || "0")),
    0n
  )

  return c.json({
    depositor,
    totalDeposited: totalDeposited.toString()
  })
})

app.use("/", graphql({ db, schema }))
app.use("/graphql", graphql({ db, schema }))

export default app


---

43. Relayer daemon

43.1 Architectural reasoning

The relayer must:
	1.	decrypt only bundles addressed to it,
	2.	reject malformed relay bundle envelopes before field access,
	3.	verify proof relayer binding,
	4.	verify exact payload cardinalities and numeric encodings,
	5.	check on-chain nullifier state,
	6.	re-quote cross-chain fees,
	7.	compute a route-aware execution pricing floor,
	8.	use exact-amount hardened approval flow,
	9.	never eagerly revoke a fee approval after broadcast unless terminal failure is known,
	10.	retry pre-submit transient failures with bounded backoff,
	11.	persist uncertain post-broadcast states for reconciliation instead of pretending they are final,
	12.	isolate one bad bundle from later valid bundles in the same payload,
	13.	simulate,
	14.	submit,
	15.	mark nullifier as seen only after confirmed transaction inclusion.

43.2 File: apps/relayer/package.json

{
  "name": "@parly/relayer",
  "version": "16.9.9-operator.0",
  "private": true,
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "typecheck": "tsc -p tsconfig.json --noEmit",
    "start": "ts-node src/bot.ts",
    "keygen:box": "ts-node src/generate-relayer-box-key.ts",
    "reconcile:pending": "ts-node src/reconcile-pending.ts",
    "reconcile:pending:loop": "ts-node src/reconcile-pending-loop.ts",
    "smoke:env": "node dist/smoke-config.js",
    "smoke:keygen": "node dist/generate-relayer-box-key.js"
  },
  "dependencies": {
    "@parly/crypto-utils": "workspace:*",
    "@parly/env-utils": "workspace:*",
    "@parly/protocol-abis": "workspace:*",
    "@parly/registry-client": "workspace:*",
    "@parly/shared-types": "workspace:*",
    "@waku/sdk": "^0.0.28",
    "dotenv": "16.4.7",
    "libsodium-wrappers-sumo": "^0.7.13",
    "viem": "2.43.5"
  },
  "devDependencies": {
    "ts-node": "10.9.2",
    "typescript": "5.8.2"
  }
}

43.3 File: apps/relayer/src/waku.ts

import { createLightNode, createDecoder, Protocols } from "@waku/sdk"
import { requireEnv, requirePositiveInt } from "@parly/env-utils"

export async function startWakuRelay(onPayload: (payload: Uint8Array) => Promise<void>) {
  const contentTopic = requireEnv("WAKU_CONTENT_TOPIC")
  const clusterId = requirePositiveInt("WAKU_CLUSTER_ID")
  const bootstrap = requireEnv("WAKU_BOOTSTRAP")
  if (bootstrap !== "true" && bootstrap !== "false") {
    throw new Error("WAKU_BOOTSTRAP must be either true or false.")
  }

  const node = await createLightNode({
    defaultBootstrap: bootstrap === "true",
    networkConfig: {
      clusterId,
      contentTopics: [contentTopic]
    }
  })

  await node.start()
  await node.waitForPeers([Protocols.Filter])

  const decoder = createDecoder(contentTopic)
  const { error, subscription } = await node.filter.createSubscription({
    contentTopics: [contentTopic]
  })

  if (error) throw new Error(error)

  await subscription.subscribe([decoder], async (msg: any) => {
    if (!msg.payload) return
    await onPayload(msg.payload)
  })

  return node
}

43.4 File: apps/relayer/src/generate-relayer-box-key.ts

import "dotenv/config"
import fs from "node:fs"
import path from "node:path"
import sodium from "libsodium-wrappers-sumo"
import { privateKeyToAccount } from "viem/accounts"

async function main() {
  await sodium.ready

  if (!process.env.RELAYER_PRIVATE_KEY) {
    throw new Error("RELAYER_PRIVATE_KEY missing")
  }

  const account = privateKeyToAccount(process.env.RELAYER_PRIVATE_KEY as `0x${string}`)
  const kp = sodium.crypto_box_keypair()
  const outputFile = path.resolve(process.cwd(), process.env.RELAYER_KEY_OUTPUT_FILE || ".secrets/relayer-box.env")

  const pubB64 = sodium.to_base64(kp.publicKey, sodium.base64_variants.ORIGINAL)
  const privB64 = sodium.to_base64(kp.privateKey, sodium.base64_variants.ORIGINAL)

  fs.mkdirSync(path.dirname(outputFile), { recursive: true })
  fs.writeFileSync(
    outputFile,
    [
      `RELAYER_ADDRESS=${account.address}`,
      `RELAYER_BOX_PRIVATE_KEY_B64=${privB64}`,
      `RELAYER_BOX_PUBLIC_KEY_B64=${pubB64}`,
      `RELAYER_REGISTRATION_EXECUTION_ADDRESS=${account.address}`,
      `RELAYER_REGISTRATION_PUBLIC_KEY_B64=${pubB64}`,
      `# Legacy fallback only: runtime discovery now prefers /api/relayers`,
      `NEXT_PUBLIC_RELAYER_REGISTRY=${account.address}:${pubB64}`,
    ].join("\n") + "\n",
    { encoding: "utf8", mode: 0o600 }
  )

  console.log("RELAYER_EXECUTION_ADDRESS=", account.address)
  console.log("RELAYER_BOX_PUBLIC_KEY_B64=", pubB64)
  console.log("RELAYER_REGISTRATION_PAGE=", "/relayer")
  console.log("RELAYER_REGISTRATION_EXECUTION_ADDRESS=", account.address)
  console.log("RELAYER_REGISTRATION_PUBLIC_KEY_B64=", pubB64)
  console.log("LEGACY_NEXT_PUBLIC_RELAYER_REGISTRY=", `${account.address}:${pubB64}`)
  console.log("WROTE_SECRET_FILE=", outputFile)
}

main().catch((e) => {
  console.error(e)
  process.exit(1)
})

43.4a File: apps/relayer/src/registry-heartbeat.ts

import type { PrivateKeyAccount } from "viem/accounts"
import { buildRelayerHeartbeatMessage } from "@parly/crypto-utils"

export async function postRelayerHeartbeat(account: PrivateKeyAccount) {
  const baseUrl = String(process.env.RELAYER_REGISTRY_API_BASE_URL || "").trim()
  if (!baseUrl) {
    return { sent: false as const, reason: "registry_not_configured" as const }
  }

  const endpoint = new URL("/api/relayers/heartbeat", baseUrl).toString()
  const timestamp = Math.floor(Date.now() / 1000)
  const signature = await account.signMessage({
    message: buildRelayerHeartbeatMessage(account.address, "SUCCESS", timestamp)
  })
  const res = await fetch(endpoint, {
    method: "POST",
    headers: {
      "content-type": "application/json"
    },
    body: JSON.stringify({
      executionAddress: account.address,
      event: "SUCCESS",
      timestamp,
      signature
    })
  })

  if (!res.ok) {
    throw new Error(`Relayer heartbeat failed with HTTP ${res.status}`)
  }

  return { sent: true as const }
}

43.4b File: apps/relayer/src/smoke-config.ts

import "dotenv/config"
import { requireEnv, requirePositiveChainId, requirePositiveEid, requirePositiveInt } from "@parly/env-utils"

requireEnv("RELAYER_PRIVATE_KEY")
requireEnv("RELAYER_BOX_PRIVATE_KEY_B64")
requireEnv("TEMPO_RPC_URL")
requirePositiveChainId("TEMPO_CHAIN_ID")
requirePositiveEid("TEMPO_LZ_EID")
requireEnv("WAKU_CONTENT_TOPIC")
const bootstrap = requireEnv("WAKU_BOOTSTRAP")
if (bootstrap !== "true" && bootstrap !== "false") {
  throw new Error("WAKU_BOOTSTRAP must be either true or false.")
}
requirePositiveInt("WAKU_CLUSTER_ID")
const registryBaseUrl = String(process.env.RELAYER_REGISTRY_API_BASE_URL || "").trim()
if (registryBaseUrl) {
  new URL(registryBaseUrl)
}

console.log("Relayer standalone config smoke check passed.")

43.5 File: apps/relayer/src/erc20.ts

import type { Abi, PublicClient, WalletClient } from "viem"
import { parseAbi } from "viem"

const ERC20_ABI = parseAbi([
  "function approve(address,uint256) external returns (bool)",
  "function allowance(address,address) external view returns (uint256)"
])

export async function safeApproveExactRelayer(args: {
  token: `0x${string}`
  spender: `0x${string}`
  amount: bigint
  owner: `0x${string}`
  publicClient: PublicClient
  walletClient: WalletClient
}) {
  const currentAllowance = await args.publicClient.readContract({
    address: args.token,
    abi: ERC20_ABI,
    functionName: "allowance",
    args: [args.owner, args.spender]
  }) as bigint

  if (currentAllowance === args.amount) return

  if (currentAllowance !== 0n && args.amount !== 0n) {
    const hash0 = await args.walletClient.writeContract({
      address: args.token,
      abi: ERC20_ABI as Abi,
      functionName: "approve",
      args: [args.spender, 0n]
    })
    await args.publicClient.waitForTransactionReceipt({ hash: hash0 })
  }

  const hash1 = await args.walletClient.writeContract({
    address: args.token,
    abi: ERC20_ABI as Abi,
    functionName: "approve",
    args: [args.spender, args.amount]
  })
  await args.publicClient.waitForTransactionReceipt({ hash: hash1 })
}

43.6 File: apps/relayer/src/bot.ts

import "dotenv/config"
import fs from "node:fs"
import path from "node:path"
import sodium from "libsodium-wrappers-sumo"
import { requireEnv, requirePositiveInt } from "@parly/env-utils"
import { POOL_ABI } from "@parly/protocol-abis"
import type {
  CanonicalRelayExecutionPayload,
  RelayCipherBundle
} from "@parly/shared-types"
import {
  createPublicClient,
  createWalletClient,
  defineChain,
  http,
  formatUnits,
  isAddress
} from "viem"
import { privateKeyToAccount } from "viem/accounts"
import { startWakuRelay } from "./waku"
import { safeApproveExactRelayer } from "./erc20"
import { postRelayerHeartbeat } from "./registry-heartbeat"

type PendingRelayerRecord = {
  version: 1
  recordedAt: number
  phase: "pre_submit_exhausted" | "submitted_receipt_unknown"
  attempt: number
  relayerAddress: `0x${string}`
  pool: `0x${string}` | null
  nullifierHash: `0x${string}` | null
  submittedHash: `0x${string}` | null
  approvalToken: `0x${string}` | null
  approvalSpender: `0x${string}` | null
  reason: string
  rawBundle: unknown
}

const ZERO_ADDRESS = "0x0000000000000000000000000000000000000000" as const

function hexToBytes(hex: string): Uint8Array {
  const clean = hex.startsWith("0x") ? hex.slice(2) : hex
  if (clean.length % 2 !== 0) throw new Error("Invalid hex length")
  if (!/^[0-9a-fA-F]*$/.test(clean)) throw new Error("Invalid hex value")
  const out = new Uint8Array(clean.length / 2)
  for (let i = 0; i < out.length; i++) {
    const byte = Number.parseInt(clean.slice(i * 2, i * 2 + 2), 16)
    if (Number.isNaN(byte)) throw new Error("Invalid hex byte")
    out[i] = byte
  }
  return out
}

function isHexString(value: unknown): value is `0x${string}` {
  return typeof value === "string" && /^0x[0-9a-fA-F]*$/.test(value)
}

function isDecimalString(value: unknown): value is string {
  return typeof value === "string" && /^(0|[1-9][0-9]*)$/.test(value)
}

function isUint32Number(value: unknown): value is number {
  return typeof value === "number" && Number.isInteger(value) && value >= 0 && value <= 0xffffffff
}

function isHexTuple2(value: unknown): value is [`0x${string}`, `0x${string}`] {
  return Array.isArray(value) && value.length === 2 && value.every(isHexString)
}

function isHexTuple2x2(
  value: unknown
): value is [[`0x${string}`, `0x${string}`], [`0x${string}`, `0x${string}`]] {
  return Array.isArray(value) && value.length === 2 && value.every(isHexTuple2)
}

function isRelayCipherBundle(x: unknown): x is RelayCipherBundle {
  return (
    !!x &&
    typeof x === "object" &&
    (x as any).version === 1 &&
    typeof (x as any).relayerAddress === "string" &&
    isAddress((x as any).relayerAddress as `0x${string}`) &&
    isHexString((x as any).ciphertext)
  )
}

function isCanonicalPayload(x: any): x is CanonicalRelayExecutionPayload {
  return (
    x &&
    typeof x.pool === "string" &&
    isAddress(x.pool) &&
    isHexTuple2(x.pA) &&
    isHexTuple2x2(x.pB) &&
    isHexTuple2(x.pC) &&
    Array.isArray(x.pubSignals) &&
    x.pubSignals.length === 50 &&
    x.pubSignals.every(isDecimalString) &&
    Array.isArray(x.recipients) &&
    x.recipients.length === 10 &&
    x.recipients.every((a: unknown) => typeof a === "string" && isAddress(a as `0x${string}`)) &&
    Array.isArray(x.amounts) &&
    x.amounts.length === 10 &&
    x.amounts.every(isDecimalString) &&
    Array.isArray(x.destEids) &&
    x.destEids.length === 10 &&
    x.destEids.every(isUint32Number) &&
    Array.isArray(x.lzOptions) &&
    x.lzOptions.length === 10 &&
    x.lzOptions.every(isHexString) &&
    isHexString(x.newChangeCommitment) &&
    isHexString(x.newChangeEnvelope)
  )
}

function parseNonNegativeInt(raw: string | undefined, fallback: number, name: string): number {
  const value = Number(raw ?? String(fallback))
  if (!Number.isFinite(value) || !Number.isInteger(value) || value < 0) {
    throw new Error(`${name} invalid`)
  }
  return value
}

function isRetryablePreSubmitError(error: unknown): boolean {
  const message = String((error as any)?.message || error).toLowerCase()
  return (
    message.includes("timeout") ||
    message.includes("fetch failed") ||
    message.includes("network") ||
    message.includes("socket") ||
    message.includes("temporar") ||
    message.includes("503") ||
    message.includes("429") ||
    message.includes("gateway timeout") ||
    message.includes("connection")
  )
}

const tempoRpcUrl = requireEnv("TEMPO_RPC_URL")
const tempoChainId = requirePositiveInt("TEMPO_CHAIN_ID")
const relayerPrivateKey = requireEnv("RELAYER_PRIVATE_KEY") as `0x${string}`

const tempo = defineChain({
  id: tempoChainId,
  name: "Tempo",
  nativeCurrency: { name: "TMP", symbol: "TMP", decimals: 18 },
  rpcUrls: {
    default: { http: [tempoRpcUrl] }
  }
})

const account = privateKeyToAccount(relayerPrivateKey)
const relayerAddressEnv = String(process.env.RELAYER_ADDRESS || "").toLowerCase()
const preSubmitRetryLimit = parseNonNegativeInt(
  process.env.RELAYER_PRE_SUBMIT_RETRY_LIMIT,
  2,
  "RELAYER_PRE_SUBMIT_RETRY_LIMIT"
)
const preSubmitRetryBackoffMs = parseNonNegativeInt(
  process.env.RELAYER_PRE_SUBMIT_RETRY_BACKOFF_MS,
  1500,
  "RELAYER_PRE_SUBMIT_RETRY_BACKOFF_MS"
)
const receiptTimeoutMs = parseNonNegativeInt(
  process.env.RELAYER_RECEIPT_TIMEOUT_MS,
  300000,
  "RELAYER_RECEIPT_TIMEOUT_MS"
)
const maxPayloadBytes = parseNonNegativeInt(
  process.env.RELAYER_MAX_PAYLOAD_BYTES,
  32768,
  "RELAYER_MAX_PAYLOAD_BYTES"
)
const pendingExecutionsFile = path.resolve(
  process.cwd(),
  process.env.RELAYER_PENDING_EXECUTIONS_FILE || ".secrets/relayer-pending-executions.ndjson"
)

const publicClient = createPublicClient({
  chain: tempo,
  transport: http(tempoRpcUrl)
})

const walletClient = createWalletClient({
  account,
  chain: tempo,
  transport: http(process.env.TEMPO_RPC_URL)
})

const seenNullifiers = new Set<string>()

async function openBundle(bundle: unknown) {
  await sodium.ready

  if (!isRelayCipherBundle(bundle)) {
    return null
  }

  if (bundle.relayerAddress.toLowerCase() !== relayerAddressEnv) {
    return null
  }

  if (!process.env.RELAYER_BOX_PRIVATE_KEY_B64) {
    throw new Error("RELAYER_BOX_PRIVATE_KEY_B64 missing")
  }

  const sk = sodium.from_base64(
    process.env.RELAYER_BOX_PRIVATE_KEY_B64!,
    sodium.base64_variants.ORIGINAL
  )

  const pk = sodium.crypto_scalarmult_base(sk)
  const sealed = hexToBytes(bundle.ciphertext)
  const opened = sodium.crypto_box_seal_open(sealed, pk, sk)

  if (!opened) return null
  return JSON.parse(sodium.to_string(opened))
}

function persistPendingExecution(record: PendingRelayerRecord) {
  fs.mkdirSync(path.dirname(pendingExecutionsFile), { recursive: true })
  fs.appendFileSync(pendingExecutionsFile, JSON.stringify(record) + "\n", {
    encoding: "utf8",
    mode: 0o600
  })
}

async function processBundle(raw: unknown, attempt = 0) {
  let decoded: CanonicalRelayExecutionPayload | null = null
  let approvalToken: `0x${string}` | null = null
  let approvalSpender: `0x${string}` | null = null
  let nullifierHash: `0x${string}` | null = null
  let submittedHash: `0x${string}` | null = null
  let finalHash: `0x${string}` | null = null
  let shouldCleanupApproval = false
  let replacementReason: "replaced" | "repriced" | "cancelled" | null = null

  try {
    decoded = await openBundle(raw)
    if (!decoded || !isCanonicalPayload(decoded)) return

    nullifierHash =
      `0x${BigInt(decoded.pubSignals[12]).toString(16).padStart(64, "0")}` as `0x${string}`

    if (seenNullifiers.has(nullifierHash.toLowerCase())) {
      console.log(`Ignored cached replay ${nullifierHash}`)
      return
    }

    const proofRelayerAddress =
      `0x${BigInt(decoded.pubSignals[49]).toString(16).padStart(40, "0")}`.toLowerCase()

    if (proofRelayerAddress !== account.address.toLowerCase()) {
      console.error(`Dropping bundle bound to ${proofRelayerAddress}`)
      return
    }

    const spent = await publicClient.readContract({
      address: decoded.pool,
      abi: POOL_ABI,
      functionName: "nullifierHashes",
      args: [nullifierHash]
    }) as boolean

    if (spent) {
      console.log(`On-chain spent, dropping ${nullifierHash}`)
      seenNullifiers.add(nullifierHash.toLowerCase())
      return
    }

    const localEid = Number(decoded.pubSignals[17])
    let totalCrossChainFee = 0n
    let sameChainOutputCount = 0
    let crossChainOutputCount = 0

    for (let i = 0; i < decoded.amounts.length; i++) {
      const amt = BigInt(decoded.amounts[i])
      const dst = decoded.destEids[i]

      if (amt === 0n) continue
      if (decoded.recipients[i].toLowerCase() === ZERO_ADDRESS) {
        console.error("Dropped bundle: zero recipient")
        return
      }
      if (dst === localEid) {
        sameChainOutputCount += 1
        continue
      }

      crossChainOutputCount += 1

      const fee = await publicClient.readContract({
        address: decoded.pool,
        abi: POOL_ABI,
        functionName: "quoteCrossChainFee",
        args: [dst, decoded.recipients[i], amt, decoded.lzOptions[i]]
      }) as bigint

      totalCrossChainFee += fee
    }

    const executorFee = BigInt(decoded.pubSignals[14])
    const feeTokenDecimals = BigInt(process.env.TEMPO_LZ_FEE_TOKEN_DECIMALS || "6")
    const feeTokenUsd6 = BigInt(process.env.TEMPO_LZ_FEE_TOKEN_USD_6DEC || "1000000")
    const totalCrossChainFeeUsd6 =
      feeTokenUsd6 === 0n ? 0n : (totalCrossChainFee * feeTokenUsd6) / (10n ** feeTokenDecimals)
    const requiredMarginUsd6 =
      crossChainOutputCount === 0
        ? 100_000n + (sameChainOutputCount > 0 ? BigInt(sameChainOutputCount - 1) * 20_000n : 0n)
        : (() => {
            const dynamicMargin =
              totalCrossChainFeeUsd6 / 5n +
              BigInt(crossChainOutputCount) * 50_000n +
              BigInt(sameChainOutputCount) * 20_000n
            return dynamicMargin > 350_000n ? dynamicMargin : 350_000n
          })()
    const recommendedExecutorFee =
      crossChainOutputCount === 0
        ? requiredMarginUsd6
        : totalCrossChainFeeUsd6 + requiredMarginUsd6

    if (executorFee < recommendedExecutorFee) {
      console.log(
        `Ignored bundle: executor fee ${formatUnits(executorFee, 6)} below recommended floor ${formatUnits(recommendedExecutorFee, 6)}`
      )
      return
    }

    if (totalCrossChainFee > 0n) {
      approvalToken = await publicClient.readContract({
        address: decoded.pool,
        abi: POOL_ABI,
        functionName: "lzFeeToken"
      }) as `0x${string}`
      approvalSpender = decoded.pool

      await safeApproveExactRelayer({
        token: approvalToken,
        spender: approvalSpender,
        amount: totalCrossChainFee,
        owner: account.address,
        publicClient,
        walletClient
      })
    }

    const { request } = await publicClient.simulateContract({
      account,
      address: decoded.pool,
      abi: POOL_ABI,
      functionName: "batchWithdraw",
      args: [
        decoded.pA,
        decoded.pB,
        decoded.pC,
        decoded.pubSignals.map((x) => BigInt(x)) as any,
        decoded.recipients,
        decoded.amounts.map((x) => BigInt(x)),
        decoded.destEids,
        decoded.newChangeCommitment,
        decoded.newChangeEnvelope,
        decoded.lzOptions
      ]
    })

    submittedHash = await walletClient.writeContract(request)
    finalHash = submittedHash

    const receipt = await publicClient.waitForTransactionReceipt({
      hash: submittedHash,
      timeout: receiptTimeoutMs,
      onReplaced: (replacement) => {
        replacementReason = replacement.reason
        finalHash = replacement.transactionReceipt.transactionHash as `0x${string}`
      }
    })

    if (replacementReason === "cancelled") {
      shouldCleanupApproval = true
      console.error(`Execution cancelled before inclusion for ${finalHash}`)
      return
    }

    if (replacementReason === "replaced") {
      shouldCleanupApproval = true
      console.error(`Execution replaced before inclusion for ${finalHash}`)
      return
    }

    if (receipt.status !== "success") {
      shouldCleanupApproval = true
      console.error(`Execution reverted for ${finalHash}`)
      return
    }

    seenNullifiers.add(nullifierHash.toLowerCase())
    try {
      const heartbeat = await postRelayerHeartbeat(account)
      if (!heartbeat.sent) {
        console.log("Registry heartbeat skipped: RELAYER_REGISTRY_API_BASE_URL not configured.")
      }
    } catch (heartbeatError: any) {
      console.error(`Relayer heartbeat error: ${heartbeatError.message || heartbeatError}`)
    }
    console.log(`Relayed ${finalHash}`)
  } catch (e: any) {
    const message = String(e?.message || e)

    if (!submittedHash && isRetryablePreSubmitError(e) && attempt < preSubmitRetryLimit) {
      shouldCleanupApproval = true
      const delayMs = preSubmitRetryBackoffMs * (2 ** attempt)
      console.error(
        `Retrying bundle in ${delayMs}ms after transient pre-submit error (${attempt + 1}/${preSubmitRetryLimit}): ${message}`
      )
      setTimeout(() => {
        void processBundle(raw, attempt + 1)
      }, delayMs)
      return
    }

    if (!submittedHash) {
      shouldCleanupApproval = true

      if (isRetryablePreSubmitError(e)) {
        persistPendingExecution({
          version: 1,
          recordedAt: Math.floor(Date.now() / 1000),
          phase: "pre_submit_exhausted",
          attempt,
          relayerAddress: account.address,
          pool: decoded ? decoded.pool : null,
          nullifierHash,
          submittedHash: null,
          approvalToken,
          approvalSpender,
          reason: message,
          rawBundle: raw
        })
        console.error(`Persisted exhausted pre-submit bundle: ${message}`)
      } else {
        console.error(`Dropped bundle: ${message}`)
      }
      return
    }

    persistPendingExecution({
      version: 1,
      recordedAt: Math.floor(Date.now() / 1000),
      phase: "submitted_receipt_unknown",
      attempt,
      relayerAddress: account.address,
      pool: decoded ? decoded.pool : null,
      nullifierHash,
      submittedHash: finalHash || submittedHash,
      approvalToken,
      approvalSpender,
      reason: message,
      rawBundle: raw
    })
    console.error(`Persisted pending execution ${finalHash || submittedHash}: ${message}`)
  } finally {
    if (approvalToken && approvalSpender && (shouldCleanupApproval || !submittedHash)) {
      try {
        await safeApproveExactRelayer({
          token: approvalToken,
          spender: approvalSpender,
          amount: 0n,
          owner: account.address,
          publicClient,
          walletClient
        })
      } catch (cleanupError: any) {
        console.error(`Relayer cleanup error: ${cleanupError.message || cleanupError}`)
      }
    }
  }
}

async function main() {
  if (!relayerAddressEnv) {
    throw new Error("RELAYER_ADDRESS missing")
  }
  if (relayerAddressEnv !== account.address.toLowerCase()) {
    throw new Error("RELAYER_ADDRESS must equal the address derived from RELAYER_PRIVATE_KEY")
  }

  console.log(`Relayer listening as ${account.address}`)

  await startWakuRelay(async (payloadBytes) => {
    try {
      if (payloadBytes.byteLength > maxPayloadBytes) {
        console.error("Dropped oversized payload")
        return
      }

      const rawPayload = JSON.parse(new TextDecoder().decode(payloadBytes))
      const bundles = Array.isArray(rawPayload) ? rawPayload : [rawPayload]

      for (const raw of bundles) {
        await processBundle(raw, 0)
      }
    } catch (e: any) {
      console.error(`Relayer error: ${e.message || e}`)
    }
  })
}

main().catch(console.error)


---

43.7 File: apps/relayer/src/reconcile-pending.ts

import "dotenv/config"
import fs from "node:fs"
import path from "node:path"
import { requireEnv, requirePositiveInt } from "@parly/env-utils"
import {
  createPublicClient,
  createWalletClient,
  defineChain,
  http,
  isAddress,
  zeroAddress
} from "viem"
import { privateKeyToAccount } from "viem/accounts"
import { safeApproveExactRelayer } from "./erc20"

type PendingRelayerRecord = {
  version: 1
  recordedAt: number
  phase: "pre_submit_exhausted" | "submitted_receipt_unknown"
  attempt: number
  relayerAddress: `0x${string}`
  pool: `0x${string}` | null
  nullifierHash: `0x${string}` | null
  submittedHash: `0x${string}` | null
  approvalToken: `0x${string}` | null
  approvalSpender: `0x${string}` | null
  reason: string
  rawBundle: unknown
}

type ResolvedRelayerRecord = PendingRelayerRecord & {
  resolvedAt: number
  resolution:
    | "confirmed_success"
    | "confirmed_failure"
    | "external_success_without_receipt"
    | "pre_submit_terminal_failure"
  cleanupAttempted: boolean
  cleanupSucceeded: boolean
}

const NULLIFIER_ABI = [
  {
    type: "function",
    name: "nullifierHashes",
    stateMutability: "view",
    inputs: [{ name: "", type: "bytes32" }],
    outputs: [{ name: "", type: "bool" }]
  }
] as const

const tempoRpcUrl = requireEnv("TEMPO_RPC_URL")
const tempoChainId = requirePositiveInt("TEMPO_CHAIN_ID")
const relayerPrivateKey = requireEnv("RELAYER_PRIVATE_KEY") as `0x${string}`

const tempo = defineChain({
  id: tempoChainId,
  name: "Tempo",
  nativeCurrency: { name: "TMP", symbol: "TMP", decimals: 18 },
  rpcUrls: {
    default: { http: [tempoRpcUrl] }
  }
})

const account = privateKeyToAccount(relayerPrivateKey)
const publicClient = createPublicClient({
  chain: tempo,
  transport: http(tempoRpcUrl)
})
const walletClient = createWalletClient({
  account,
  chain: tempo,
  transport: http(tempoRpcUrl)
})

const pendingFile = path.resolve(
  process.cwd(),
  process.env.RELAYER_PENDING_EXECUTIONS_FILE || ".secrets/relayer-pending-executions.ndjson"
)
const archiveFile = path.resolve(
  process.cwd(),
  process.env.RELAYER_PENDING_EXECUTIONS_ARCHIVE_FILE || ".secrets/relayer-pending-executions.resolved.ndjson"
)
const lockFile = path.resolve(
  process.cwd(),
  process.env.RELAYER_PENDING_RECONCILE_LOCK_FILE || ".secrets/relayer-pending-executions.lock"
)

function readPending(): PendingRelayerRecord[] {
  if (!fs.existsSync(pendingFile)) return []

  return fs
    .readFileSync(pendingFile, "utf8")
    .split(/\r?\n/)
    .map((line) => line.trim())
    .filter(Boolean)
    .map((line) => JSON.parse(line) as PendingRelayerRecord)
}

function writePending(records: PendingRelayerRecord[]) {
  fs.mkdirSync(path.dirname(pendingFile), { recursive: true })
  const body = records.length ? `${records.map((r) => JSON.stringify(r)).join("\n")}\n` : ""
  const tmpFile = `${pendingFile}.tmp`
  fs.writeFileSync(tmpFile, body)
  fs.renameSync(tmpFile, pendingFile)
}

function appendResolved(records: ResolvedRelayerRecord[]) {
  if (!records.length) return
  fs.mkdirSync(path.dirname(archiveFile), { recursive: true })
  const body = `${records.map((r) => JSON.stringify(r)).join("\n")}\n`
  fs.appendFileSync(archiveFile, body)
}

function acquireLock(): (() => void) | null {
  fs.mkdirSync(path.dirname(lockFile), { recursive: true })

  try {
    const fd = fs.openSync(lockFile, "wx")
    fs.writeFileSync(fd, String(process.pid))

    return () => {
      try {
        fs.closeSync(fd)
      } catch {}
      try {
        fs.unlinkSync(lockFile)
      } catch {}
    }
  } catch (e: any) {
    if (e?.code === "EEXIST") {
      console.log(`Pending reconcile lock already held at ${lockFile}, skipping this pass.`)
      return null
    }
    throw e
  }
}

async function tryCleanup(record: PendingRelayerRecord) {
  if (
    !record.approvalToken ||
    !record.approvalSpender ||
    !isAddress(record.approvalToken) ||
    !isAddress(record.approvalSpender) ||
    record.approvalToken.toLowerCase() === zeroAddress
  ) {
    return { attempted: false, succeeded: false }
  }

  try {
    await safeApproveExactRelayer({
      token: record.approvalToken,
      spender: record.approvalSpender,
      amount: 0n,
      owner: account.address,
      publicClient,
      walletClient
    })
    return { attempted: true, succeeded: true }
  } catch (e: any) {
    console.error(`Cleanup failed for ${record.submittedHash || record.nullifierHash || "unknown"}: ${e.message || e}`)
    return { attempted: true, succeeded: false }
  }
}

async function isNullifierSpent(record: PendingRelayerRecord): Promise<boolean> {
  if (!record.pool || !record.nullifierHash || !isAddress(record.pool)) return false

  return await publicClient.readContract({
    address: record.pool,
    abi: NULLIFIER_ABI,
    functionName: "nullifierHashes",
    args: [record.nullifierHash]
  }) as boolean
}

async function resolveRecord(record: PendingRelayerRecord): Promise<{
  keep: boolean
  resolved?: ResolvedRelayerRecord
}> {
  const resolvedAt = Math.floor(Date.now() / 1000)

  if (!record.submittedHash) {
    const spent = await isNullifierSpent(record)
    if (spent) {
      return {
        keep: false,
        resolved: {
          ...record,
          resolvedAt,
          resolution: "external_success_without_receipt",
          cleanupAttempted: false,
          cleanupSucceeded: false
        }
      }
    }

    const cleanup = await tryCleanup(record)
    return {
      keep: false,
      resolved: {
        ...record,
        resolvedAt,
        resolution: "pre_submit_terminal_failure",
        cleanupAttempted: cleanup.attempted,
        cleanupSucceeded: cleanup.succeeded
      }
    }
  }

  try {
    const receipt = await publicClient.getTransactionReceipt({ hash: record.submittedHash })
    if (receipt.status === "success") {
      return {
        keep: false,
        resolved: {
          ...record,
          resolvedAt,
          resolution: "confirmed_success",
          cleanupAttempted: false,
          cleanupSucceeded: false
        }
      }
    }

    const cleanup = await tryCleanup(record)
    return {
      keep: false,
      resolved: {
        ...record,
        resolvedAt,
        resolution: "confirmed_failure",
        cleanupAttempted: cleanup.attempted,
        cleanupSucceeded: cleanup.succeeded
      }
    }
  } catch {
    const spent = await isNullifierSpent(record)
    if (spent) {
      return {
        keep: false,
        resolved: {
          ...record,
          resolvedAt,
          resolution: "external_success_without_receipt",
          cleanupAttempted: false,
          cleanupSucceeded: false
        }
      }
    }

    return { keep: true }
  }
}

export async function reconcilePendingOnce() {
  const releaseLock = acquireLock()
  if (!releaseLock) {
    return
  }

  try {
    const pending = readPending()
    if (!pending.length) {
      console.log("No pending relay executions to reconcile.")
      return
    }

    const keep: PendingRelayerRecord[] = []
    const resolved: ResolvedRelayerRecord[] = []

    for (const record of pending) {
      const outcome = await resolveRecord(record)
      if (outcome.keep) {
        keep.push(record)
      } else if (outcome.resolved) {
        resolved.push(outcome.resolved)
      }
    }

    // Append to the durable archive before rewriting pending state so a crash
    // cannot silently discard a resolved record.
    appendResolved(resolved)
    writePending(keep)

    console.log(
      `Reconciled relay executions: resolved=${resolved.length} still_pending=${keep.length} archive=${archiveFile}`
    )
  } finally {
    releaseLock()
  }
}

if (require.main === module) {
  reconcilePendingOnce().catch((e) => {
    console.error(e)
    process.exit(1)
  })
}


---

43.8 File: apps/relayer/src/reconcile-pending-loop.ts

import "dotenv/config"
import { reconcilePendingOnce } from "./reconcile-pending"

function parseInterval(raw: string | undefined, fallback: number): number {
  const value = Number(raw ?? String(fallback))
  if (!Number.isFinite(value) || !Number.isInteger(value) || value <= 0) {
    return fallback
  }
  return value
}

function sleep(ms: number) {
  return new Promise((resolve) => setTimeout(resolve, ms))
}

async function main() {
  const intervalMs = parseInterval(process.env.RELAYER_PENDING_RECONCILE_INTERVAL_MS, 30000)

  while (true) {
    try {
      await reconcilePendingOnce()
    } catch (e: any) {
      console.error(`Pending reconcile loop error: ${e.message || e}`)
    }

    await sleep(intervalMs)
  }
}

main().catch((e) => {
  console.error(e)
  process.exit(1)
})


---

44. SDK

44.1 Architectural reasoning

The SDK must mirror protocol reality.

So V16.9.9 keeps the same:
	*	deterministic auth message,
	*	asymmetric recovery key derivation,
	*	note envelope format,
	*	recovery reconstruction logic,
	*	proving path,
	*	direct pool execution path,
	*	an MPP-compatible wrapper only at the SDK boundary,
	*	real change-note creation,
	*	hardened approval flow for zero-reset tokens,
	*	receipt waiting after approvals,
	*	exact-amount fee approvals that naturally clear on successful execution,
	*	cleanup only when execution never broadcasts or when failure is terminal,
	*	no eager post-broadcast cleanup on uncertain receipt outcomes,
	*	and no contract-core dependency on MPP session state.

44.2 File: packages/sdk/package.json

{
  "name": "@parly/sdk",
  "version": "16.9.9",
  "private": true,
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    },
    "./lz-options": {
      "types": "./dist/lz-options.d.ts",
      "default": "./dist/lz-options.js"
    },
    "./package.json": "./package.json"
  },
  "files": [
    "dist",
    "README.md",
    "CHANGELOG.md",
    "COMPATIBILITY.md",
    "examples"
  ],
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "typecheck": "tsc -p tsconfig.json --noEmit",
    "test:lz-options": "pnpm run build && node ./scripts/verify-lz-options.mjs",
    "pack:dry-run": "pnpm publish --dry-run --no-git-checks"
  },
  "dependencies": {
    "@parly/crypto-utils": "workspace:*",
    "@parly/env-utils": "workspace:*",
    "@parly/protocol-abis": "workspace:*",
    "@parly/shared-types": "workspace:*",
    "@layerzerolabs/lz-v2-utilities": "2.3.0",
    "circomlibjs": "0.1.7",
    "fixed-merkle-tree": "0.7.3",
    "libsodium-wrappers-sumo": "^0.7.13",
    "snarkjs": "0.7.4",
    "viem": "2.43.5"
  },
  "devDependencies": {
    "@types/node": "22.13.10",
    "typescript": "5.8.2"
  }
}

44.2a File: packages/sdk/README.md

```md
# @parly/sdk

TypeScript SDK for Parly V16.9.9 private execution flows.

This canonical workspace package is the internal source package.

The public npm/package surface is produced through the public SDK export pipeline, which rewrites internal workspace dependencies into a self-contained public `@parly/sdk` package.

## Install

```bash
npm install @parly/sdk
```

## Initialization

Create the SDK with:

- `privateKeyHex`
- `tempoRpcUrl`
- `tempoChainId`

## Examples

- `examples/basic-payment.ts`
- `examples/agent-integration.ts`
- `examples/error-handling.ts`

## Compatibility

See [COMPATIBILITY.md](./COMPATIBILITY.md).
```

44.2b File: packages/sdk/CHANGELOG.md

```md
# Changelog

## 16.9.9

- publish-ready package metadata
- deterministic payout-scope verification support
- MPP-compatible adapter restored at the SDK boundary only
```

44.2c File: packages/sdk/COMPATIBILITY.md

```md
# Compatibility

- `@parly/sdk@16.9.9` targets Parly core `16.9.9`
- compatible with MCP release `16.9.9-mcp.0`
- requires the canonical V16.9.9 proof model and Tempo settlement assumptions
```

44.2d File: packages/sdk/examples/basic-payment.ts

```ts
import { ParlySDK } from "@parly/sdk"

function requireEnv(name: string) {
  const value = process.env[name]
  if (!value) throw new Error(`${name} missing`)
  return value
}

function requirePositiveInt(name: string) {
  const value = Number(requireEnv(name))
  if (!Number.isFinite(value) || !Number.isInteger(value) || value <= 0) {
    throw new Error(`${name} must be a positive integer`)
  }
  return value
}

const sdk = new ParlySDK({
  privateKeyHex: requireEnv("AGENT_PRIVATE_KEY") as `0x${string}`,
  tempoRpcUrl: requireEnv("TEMPO_RPC_URL"),
  tempoChainId: requirePositiveInt("TEMPO_CHAIN_ID")
})

const outcome = await sdk.executeAgenticPayment({
  destination: "0x1111111111111111111111111111111111111111",
  amount: "10",
  assetId: 1,
  destinationEid: requirePositiveInt("TEMPO_LZ_EID"),
  poolAddress: requireEnv("PARLY_USDC_POOL") as `0x${string}`
})

console.log(outcome)
```

44.2e File: packages/sdk/examples/agent-integration.ts

```ts
import { ParlySDK, ParlyMppAdapter } from "@parly/sdk"

function requireEnv(name: string) {
  const value = process.env[name]
  if (!value) throw new Error(`${name} missing`)
  return value
}

function requirePositiveInt(name: string) {
  const value = Number(requireEnv(name))
  if (!Number.isFinite(value) || !Number.isInteger(value) || value <= 0) {
    throw new Error(`${name} must be a positive integer`)
  }
  return value
}

const sdk = new ParlySDK({
  privateKeyHex: requireEnv("AGENT_PRIVATE_KEY") as `0x${string}`,
  tempoRpcUrl: requireEnv("TEMPO_RPC_URL"),
  tempoChainId: requirePositiveInt("TEMPO_CHAIN_ID")
})

const mpp = new ParlyMppAdapter(sdk, {
  serviceName: process.env.MPP_SERVICE_NAME,
  serviceVersion: process.env.MPP_SERVICE_VERSION
})

const session = await mpp.createSession({
  sessionId: "invoice-001",
  counterparty: "0x2222222222222222222222222222222222222222",
  assetId: 1,
  spendLimit: "250",
  destinationEid: requirePositiveInt("TEMPO_LZ_EID"),
  poolAddress: requireEnv("PARLY_USDC_POOL") as `0x${string}`
})

console.log(session)
```

44.2f File: packages/sdk/examples/error-handling.ts

```ts
import { ParlySDK } from "@parly/sdk"

function requireEnv(name: string) {
  const value = process.env[name]
  if (!value) throw new Error(`${name} missing`)
  return value
}

function requirePositiveInt(name: string) {
  const value = Number(requireEnv(name))
  if (!Number.isFinite(value) || !Number.isInteger(value) || value <= 0) {
    throw new Error(`${name} must be a positive integer`)
  }
  return value
}

const sdk = new ParlySDK({
  privateKeyHex: requireEnv("AGENT_PRIVATE_KEY") as `0x${string}`,
  tempoRpcUrl: requireEnv("TEMPO_RPC_URL"),
  tempoChainId: requirePositiveInt("TEMPO_CHAIN_ID")
})

try {
  const outcome = await sdk.executeAgenticPayment({
    destination: "0x3333333333333333333333333333333333333333",
    amount: "5",
    assetId: 2,
    destinationEid: requirePositiveInt("TEMPO_LZ_EID"),
    poolAddress: requireEnv("PARLY_USDT_POOL") as `0x${string}`
  })

  if (outcome.kind === "pending_confirmation") {
    console.warn("Pending finality:", outcome)
  } else {
    console.log(outcome)
  }
} catch (error) {
  console.error("SDK execution failed:", error)
}
```

44.3 File: packages/sdk/src/recovery-keys.ts

import sodium from "libsodium-wrappers-sumo"
import { keccak256, encodePacked } from "viem"

export const MASTER_AUTH_MESSAGE =
  "Unlock Parly.fi Privacy.\n" +
  "Domain: app.parly.fi\n" +
  "Purpose: asymmetric note recovery and private execution.\n" +
  "Only sign on the official Parly domain."

function hexToBytes(hex: string): Uint8Array {
  const clean = hex.startsWith("0x") ? hex.slice(2) : hex
  if (clean.length % 2 !== 0) throw new Error("Invalid hex length")
  if (!/^[0-9a-fA-F]*$/.test(clean)) throw new Error("Invalid hex value")
  const out = new Uint8Array(clean.length / 2)
  for (let i = 0; i < out.length; i++) {
    const byte = Number.parseInt(clean.slice(i * 2, i * 2 + 2), 16)
    if (Number.isNaN(byte)) throw new Error("Invalid hex byte")
    out[i] = byte
  }
  return out
}

export async function deriveRecoveryKeypair(signature: `0x${string}`) {
  await sodium.ready

  const seedHex = keccak256(
    encodePacked(["bytes", "string"], [signature, "PARLY_v16_9_9_RECOVERY_SEED"])
  )

  const seed = hexToBytes(seedHex).slice(0, sodium.crypto_box_SEEDBYTES)
  const kp = sodium.crypto_box_seed_keypair(seed)

  return {
    publicKeyB64: sodium.to_base64(kp.publicKey, sodium.base64_variants.ORIGINAL),
    privateKeyB64: sodium.to_base64(kp.privateKey, sodium.base64_variants.ORIGINAL)
  }
}

44.4 File: packages/sdk/src/note-crypto.ts

import sodium from "libsodium-wrappers-sumo"
import { buildPoseidon } from "circomlibjs"

export type NoteEnvelopePayload = {
  version: 1
  kind: "deposit" | "change"
  assetId: 1 | 2
  amount: string
  secret: string
  nullifier: string
  depositor: string
  ownerRecoveryPubKeyB64: string
  createdAt: number
}

function hexToBytes(hex: `0x${string}`): Uint8Array {
  const clean = hex.startsWith("0x") ? hex.slice(2) : hex
  if (clean.length % 2 !== 0) throw new Error("Invalid hex length")
  if (!/^[0-9a-fA-F]*$/.test(clean)) throw new Error("Invalid hex value")
  const out = new Uint8Array(clean.length / 2)
  for (let i = 0; i < out.length; i++) {
    const byte = Number.parseInt(clean.slice(i * 2, i * 2 + 2), 16)
    if (Number.isNaN(byte)) throw new Error("Invalid hex byte")
    out[i] = byte
  }
  return out
}

function bytesToHex(bytes: Uint8Array): `0x${string}` {
  return `0x${Array.from(bytes).map((b) => b.toString(16).padStart(2, "0")).join("")}` as `0x${string}`
}

async function randomFieldElement(): Promise<string> {
  await sodium.ready
  const bytes = sodium.randombytes_buf(31)
  const hex = Array.from(bytes).map((b) => b.toString(16).padStart(2, "0")).join("")
  return BigInt(`0x${hex}`).toString()
}

export async function calcInnerCommitment(
  secret: string,
  nullifier: string,
  depositor: string,
  assetId: 1 | 2
): Promise<string> {
  const poseidon = await buildPoseidon()
  const out = poseidon([
    BigInt(secret),
    BigInt(nullifier),
    BigInt(depositor),
    BigInt(assetId)
  ])
  return BigInt(poseidon.F.toString(out)).toString()
}

export async function calcFinalCommitment(
  innerCommitment: string,
  amount: string
): Promise<`0x${string}`> {
  const poseidon = await buildPoseidon()
  const out = poseidon([BigInt(innerCommitment), BigInt(amount)])
  return `0x${BigInt(poseidon.F.toString(out)).toString(16).padStart(64, "0")}` as `0x${string}`
}

export async function calcNullifierHash(secret: string, nullifier: string): Promise<`0x${string}`> {
  const poseidon = await buildPoseidon()
  const out = poseidon([BigInt(secret), BigInt(nullifier)])
  return `0x${BigInt(poseidon.F.toString(out)).toString(16).padStart(64, "0")}` as `0x${string}`
}

export async function createSealedNoteEnvelope(
  recipientRecoveryPubKeyB64: string,
  payload: NoteEnvelopePayload
): Promise<`0x${string}`> {
  await sodium.ready
  const recipientPub = sodium.from_base64(
    recipientRecoveryPubKeyB64,
    sodium.base64_variants.ORIGINAL
  )
  const msg = sodium.from_string(JSON.stringify(payload))
  const sealed = sodium.crypto_box_seal(msg, recipientPub)
  return bytesToHex(sealed)
}

export async function openSealedNoteEnvelope(
  recipientPublicKeyB64: string,
  recipientPrivateKeyB64: string,
  envelope: `0x${string}`
): Promise<NoteEnvelopePayload> {
  await sodium.ready
  const pk = sodium.from_base64(recipientPublicKeyB64, sodium.base64_variants.ORIGINAL)
  const sk = sodium.from_base64(recipientPrivateKeyB64, sodium.base64_variants.ORIGINAL)
  const ciphertext = hexToBytes(envelope)
  const opened = sodium.crypto_box_seal_open(ciphertext, pk, sk)
  if (!opened) throw new Error("Envelope decryption failed")
  return JSON.parse(sodium.to_string(opened)) as NoteEnvelopePayload
}

export async function reconstructEnvelopeNote(payload: NoteEnvelopePayload) {
  const inner = await calcInnerCommitment(
    payload.secret,
    payload.nullifier,
    payload.depositor,
    payload.assetId
  )
  const commitment = await calcFinalCommitment(inner, payload.amount)
  const nullifierHash = await calcNullifierHash(payload.secret, payload.nullifier)
  return { commitment, nullifierHash }
}

export async function createFreshNote(
  recipientRecoveryPubKeyB64: string,
  depositor: string,
  assetId: 1 | 2,
  amount: string,
  kind: "deposit" | "change"
) {
  const secret = await randomFieldElement()
  const nullifier = await randomFieldElement()
  const innerCommitment = await calcInnerCommitment(secret, nullifier, depositor, assetId)
  const commitment = await calcFinalCommitment(innerCommitment, amount)
  const nullifierHash = await calcNullifierHash(secret, nullifier)

  const payload: NoteEnvelopePayload = {
    version: 1,
    kind,
    assetId,
    amount,
    secret,
    nullifier,
    depositor,
    ownerRecoveryPubKeyB64: recipientRecoveryPubKeyB64,
    createdAt: Math.floor(Date.now() / 1000)
  }

  const envelope = await createSealedNoteEnvelope(recipientRecoveryPubKeyB64, payload)

  return {
    secret,
    nullifier,
    innerCommitment,
    commitment,
    nullifierHash,
    envelope,
    payload
  }
}

44.5 File: packages/sdk/src/indexer.ts

const PONDER_URL =
  process.env.PONDER_GRAPHQL_URL || process.env.NEXT_PUBLIC_PONDER_GRAPHQL_URL!

export type DepositRow = {
  depositor: string
  commitment: string
  amount: string
  leafIndex: number
  assetId: number
  txHash: string
  timestamp: string
}

export type IngressSettlementRow = {
  guid: string
  commitment: string
  assetId: number
  depositor: string
  amount: string
  innerCommitment: string
  settlementLeafIndex: number
  settlementChain: string
  settlementEid: number
  sourceEid: number
  composeFrom: string
  txHash: string
  timestamp: string
}

export type SpokeShieldDispatchRow = {
  guid: string
  assetId: number
  sender: string
  amount: string
  innerCommitment: string
  sourceChain: string
  sourceEid: number
  gateway: string
  txHash: string
  timestamp: string
}

export type WithdrawalRow = {
  nullifierHash: string
  executor: string
  protocolFee: string
  executorFee: string
  childKeys: string[] | string
  assetId: number
  txHash: string
  timestamp: string
}

export type DepositHistoryRow = DepositRow & {
  origin: "tempo" | "spoke"
  provenance?: {
    guid: string
    settlementChain: string
    settlementEid: number
    settlementTxHash: string
    sourceChain?: string
    sourceEid: number
    sourceTxHash?: string
    sourceSender?: string
    composeFrom: string
  }
}

function parsePositiveInt(raw: string | undefined, fallback: number): number {
  const value = Number(raw ?? String(fallback))
  if (!Number.isFinite(value) || !Number.isInteger(value) || value <= 0) {
    return fallback
  }
  return value
}

const INDEXER_PAGE_SIZE = parsePositiveInt(
  process.env.INDEXER_PAGE_SIZE || process.env.NEXT_PUBLIC_INDEXER_PAGE_SIZE,
  500
)

async function gql(query: string, variables: Record<string, unknown> = {}) {
  const res = await fetch(PONDER_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ query, variables }),
    cache: "no-store"
  })
  if (!res.ok) {
    throw new Error(`Indexer request failed with HTTP ${res.status}`)
  }

  const json = await res.json()
  if (json.errors?.length) throw new Error(json.errors[0].message)
  return json.data
}

async function fetchAllPages<T>(
  query: string,
  rootKey: string,
  variables: Record<string, unknown> = {}
): Promise<T[]> {
  let after: string | null = null
  const out: T[] = []

  while (true) {
    const data = await gql(query, {
      ...variables,
      limit: INDEXER_PAGE_SIZE,
      after
    })

    const page = data?.[rootKey]
    const items = page?.items || []
    out.push(...items)

    if (!page?.pageInfo?.hasNextPage || !page?.pageInfo?.endCursor) {
      break
    }

    after = page.pageInfo.endCursor
  }

  return out
}

export async function fetchLeaves(assetId: number) {
  const q = `
    query($assetId:Int!, $limit:Int!, $after:String) {
      leafInsertedEvents(
        where:{assetId:$assetId},
        orderBy:"leafIndex",
        orderDirection:"asc",
        limit:$limit,
        after:$after
      ) {
        items { commitment leafIndex kind txHash timestamp }
        pageInfo { hasNextPage endCursor }
      }
    }
  `
  return fetchAllPages(q, "leafInsertedEvents", { assetId })
}

export async function fetchDeposits(assetId: number): Promise<DepositRow[]> {
  const q = `
    query($assetId:Int!, $limit:Int!, $after:String) {
      depositEvents(
        where:{assetId:$assetId},
        orderBy:"timestamp",
        orderDirection:"asc",
        limit:$limit,
        after:$after
      ) {
        items { depositor commitment amount leafIndex assetId txHash timestamp }
        pageInfo { hasNextPage endCursor }
      }
    }
  `
  return fetchAllPages(q, "depositEvents", { assetId })
}

export async function fetchIngressSettlements(assetId: number): Promise<IngressSettlementRow[]> {
  const q = `
    query($assetId:Int!, $limit:Int!, $after:String) {
      ingressSettlementEvents(
        where:{assetId:$assetId},
        orderBy:"timestamp",
        orderDirection:"asc",
        limit:$limit,
        after:$after
      ) {
        items {
          guid
          commitment
          assetId
          depositor
          amount
          innerCommitment
          settlementLeafIndex
          settlementChain
          settlementEid
          sourceEid
          composeFrom
          txHash
          timestamp
        }
        pageInfo { hasNextPage endCursor }
      }
    }
  `
  return fetchAllPages(q, "ingressSettlementEvents", { assetId })
}

export async function fetchSpokeShieldDispatches(
  assetId: number
): Promise<SpokeShieldDispatchRow[]> {
  const q = `
    query($assetId:Int!, $limit:Int!, $after:String) {
      spokeShieldDispatchEvents(
        where:{assetId:$assetId},
        orderBy:"timestamp",
        orderDirection:"asc",
        limit:$limit,
        after:$after
      ) {
        items {
          guid
          assetId
          sender
          amount
          innerCommitment
          sourceChain
          sourceEid
          gateway
          txHash
          timestamp
        }
        pageInfo { hasNextPage endCursor }
      }
    }
  `
  return fetchAllPages(q, "spokeShieldDispatchEvents", { assetId })
}

export async function fetchDepositHistory(assetId: number): Promise<DepositHistoryRow[]> {
  const [deposits, settlements, dispatches] = await Promise.all([
    fetchDeposits(assetId),
    fetchIngressSettlements(assetId),
    fetchSpokeShieldDispatches(assetId)
  ])

  const settlementByCommitment = new Map(
    settlements.map((row) => [String(row.commitment).toLowerCase(), row] as const)
  )
  const dispatchByGuid = new Map(
    dispatches.map((row) => [String(row.guid).toLowerCase(), row] as const)
  )

  return deposits.map((row) => {
    const settlement = settlementByCommitment.get(String(row.commitment).toLowerCase())
    if (!settlement) {
      return { ...row, origin: "tempo" as const }
    }

    const dispatch = dispatchByGuid.get(String(settlement.guid).toLowerCase())
    if (!dispatch) {
      return { ...row, origin: "tempo" as const }
    }

    return {
      ...row,
      origin: "spoke" as const,
      provenance: {
        guid: settlement.guid,
        settlementChain: settlement.settlementChain,
        settlementEid: settlement.settlementEid,
        settlementTxHash: settlement.txHash,
        sourceChain: dispatch.sourceChain,
        sourceEid: settlement.sourceEid,
        sourceTxHash: dispatch.txHash,
        sourceSender: dispatch.sender,
        composeFrom: settlement.composeFrom
      }
    }
  })
}

export async function fetchRecoveryHeads(assetId: number): Promise<RecoveryHeads> {
  const q = `
    query($assetId:Int!) {
      leafInsertedEvents(where:{assetId:$assetId}, orderBy:"leafIndex", orderDirection:"desc", limit:1) {
        items { commitment leafIndex txHash }
      }
      noteEnvelopeEvents(where:{assetId:$assetId}, orderBy:"timestamp", orderDirection:"desc", limit:1) {
        items { commitment txHash timestamp }
      }
      batchWithdrawalEvents(where:{assetId:$assetId}, orderBy:"timestamp", orderDirection:"desc", limit:1) {
        items { nullifierHash txHash timestamp }
      }
    }
  `

  const data = await gql(q, { assetId })
  const leaf = data?.leafInsertedEvents?.items?.[0]
  const envelope = data?.noteEnvelopeEvents?.items?.[0]
  const nullifier = data?.batchWithdrawalEvents?.items?.[0]

  return {
    leafCursor: leaf ? `${leaf.leafIndex}:${leaf.commitment}:${leaf.txHash}` : null,
    envelopeCursor: envelope ? `${envelope.commitment}:${envelope.txHash}:${envelope.timestamp}` : null,
    nullifierCursor: nullifier ? `${nullifier.nullifierHash}:${nullifier.txHash}:${nullifier.timestamp}` : null
  }
}

export async function fetchEnvelopes(assetId: number) {
  const q = `
    query($assetId:Int!, $limit:Int!, $after:String) {
      noteEnvelopeEvents(
        where:{assetId:$assetId},
        orderBy:"timestamp",
        orderDirection:"asc",
        limit:$limit,
        after:$after
      ) {
        items { commitment kind envelope txHash timestamp }
        pageInfo { hasNextPage endCursor }
      }
    }
  `
  return fetchAllPages(q, "noteEnvelopeEvents", { assetId })
}

export async function fetchWithdrawals(assetId: number): Promise<WithdrawalRow[]> {
  const q = `
    query($assetId:Int!, $limit:Int!, $after:String) {
      batchWithdrawalEvents(
        where:{assetId:$assetId},
        orderBy:"timestamp",
        orderDirection:"asc",
        limit:$limit,
        after:$after
      ) {
        items { nullifierHash executor protocolFee executorFee childKeys assetId txHash timestamp }
        pageInfo { hasNextPage endCursor }
      }
    }
  `
  return fetchAllPages(q, "batchWithdrawalEvents", { assetId })
}

export async function fetchNullifiers(assetId: number) {
  const q = `
    query($assetId:Int!, $limit:Int!, $after:String) {
      batchWithdrawalEvents(
        where:{assetId:$assetId},
        orderBy:"timestamp",
        orderDirection:"asc",
        limit:$limit,
        after:$after
      ) {
        items { nullifierHash txHash timestamp }
        pageInfo { hasNextPage endCursor }
      }
    }
  `
  return fetchAllPages(q, "batchWithdrawalEvents", { assetId })
}

export async function fetchDepositsByDepositor(depositor: string): Promise<DepositRow[]> {
  const normalized = depositor.toLowerCase()
  const [usdc, usdt] = await Promise.all([fetchDeposits(1), fetchDeposits(2)])
  return [...usdc, ...usdt].filter(
    (row) => String(row.depositor).toLowerCase() === normalized
  )
}

export async function fetchApproxPublicDepositVolume() {
  const [usdc, usdt] = await Promise.all([fetchDeposits(1), fetchDeposits(2)])
  const usdcTotal = usdc.reduce((sum, row) => sum + BigInt(row.amount), 0n)
  const usdtTotal = usdt.reduce((sum, row) => sum + BigInt(row.amount), 0n)
  return {
    usdc: usdcTotal,
    usdt: usdtTotal,
    total: usdcTotal + usdtTotal
  }
}

export async function fetchTxBatches(txHash: string) {
  const q = `
    query($txHash:String!) {
      batchWithdrawalEvents(where:{txHash:$txHash}) {
        items { childKeys protocolFee executorFee txHash timestamp assetId nullifierHash }
      }
    }
  `

  const data = await gql(q, { txHash })
  return data?.batchWithdrawalEvents?.items || []
}

44.6 File: packages/sdk/src/erc20.ts

import { parseAbi, type Abi, type PublicClient, type WalletClient } from "viem"

const ERC20_ABI = parseAbi([
  "function approve(address,uint256) external returns (bool)",
  "function allowance(address,address) external view returns (uint256)"
])

export async function safeApproveExactSdk(args: {
  token: `0x${string}`
  spender: `0x${string}`
  amount: bigint
  owner: `0x${string}`
  publicClient: PublicClient
  walletClient: WalletClient
}) {
  const currentAllowance = await args.publicClient.readContract({
    address: args.token,
    abi: ERC20_ABI,
    functionName: "allowance",
    args: [args.owner, args.spender]
  }) as bigint

  if (currentAllowance === args.amount) return

  if (currentAllowance !== 0n && args.amount !== 0n) {
    const hash0 = await args.walletClient.writeContract({
      address: args.token,
      abi: ERC20_ABI as Abi,
      functionName: "approve",
      args: [args.spender, 0n]
    })
    await args.publicClient.waitForTransactionReceipt({ hash: hash0 })
  }

  const hash1 = await args.walletClient.writeContract({
    address: args.token,
    abi: ERC20_ABI as Abi,
    functionName: "approve",
    args: [args.spender, args.amount]
  })
  await args.publicClient.waitForTransactionReceipt({ hash: hash1 })
}

44.7 File: packages/sdk/src/core.ts

import * as snarkjs from "snarkjs"
import fs from "node:fs"
import path from "node:path"
import { requirePositiveEidValue } from "@parly/env-utils"
import { POOL_ABI } from "@parly/protocol-abis"
import {
  isAddress,
  parseUnits,
  createPublicClient,
  createWalletClient,
  defineChain,
  http
} from "viem"
import { privateKeyToAccount } from "viem/accounts"

import { MASTER_AUTH_MESSAGE, deriveRecoveryKeypair } from "./recovery-keys.js"
import { buildCrossChainPayoutOptions } from "./lz-options.js"
import {
  openSealedNoteEnvelope,
  reconstructEnvelopeNote,
  createFreshNote
} from "./note-crypto.js"
import {
  fetchLeaves,
  fetchEnvelopes,
  fetchNullifiers,
  fetchRecoveryHeads,
  type RecoveryHeads
} from "./indexer.js"
import { safeApproveExactSdk } from "./erc20.js"
import { MerkleTree } from "fixed-merkle-tree"
import { buildPoseidon } from "circomlibjs"

const SDK_LOCAL_EID = requirePositiveEidValue(
  process.env.TEMPO_LZ_EID ?? process.env.NEXT_PUBLIC_TEMPO_LZ_EID,
  "TEMPO_LZ_EID/NEXT_PUBLIC_TEMPO_LZ_EID"
)

export type ExecutePaymentParams = {
  destination: `0x${string}`
  amount: string
  assetId: 1 | 2
  destinationEid: number
  poolAddress: `0x${string}`
  proofAssetsBasePath?: string
}

export type ExecutePaymentOutcome =
  | { kind: "success"; hash: `0x${string}` }
  | { kind: "pending_confirmation"; hash: `0x${string}`; message: string; persistedTo: string | null }
  | { kind: "terminal_failure"; hash?: `0x${string}`; message: string }

type PendingSdkExecutionRecord = {
  version: 1
  recordedAt: number
  submittedHash: `0x${string}`
  nullifierHash: `0x${string}`
  pool: `0x${string}`
  destination: `0x${string}`
  destinationEid: number
  assetId: 1 | 2
  approvalToken: `0x${string}` | null
  approvalSpender: `0x${string}` | null
  reason: string
}

const ZERO_ADDRESS = "0x0000000000000000000000000000000000000000" as const

type CachedSdkBestNote = {
  payload: any
  commitment: `0x${string}`
  nullifierHash: `0x${string}`
  amount: string
}

type SdkRecoveryCacheEntry = {
  version: 1
  assetId: 1 | 2
  heads: RecoveryHeads
  leaves: Array<{
    commitment: string
    leafIndex: number
    kind?: number
    txHash?: string
    timestamp?: string
  }>
  best: CachedSdkBestNote | null
}

type SdkRecoveryCacheFile = {
  version: 1
  assets: Record<string, SdkRecoveryCacheEntry>
}

function recoveryHeadsEqual(a: RecoveryHeads | null, b: RecoveryHeads | null) {
  if (!a || !b) return false
  return (
    a.leafCursor === b.leafCursor &&
    a.envelopeCursor === b.envelopeCursor &&
    a.nullifierCursor === b.nullifierCursor
  )
}

function loadSdkRecoveryCache(assetId: 1 | 2): SdkRecoveryCacheEntry | null {
  const raw = process.env.SDK_RECOVERY_CACHE_FILE
  if (!raw) return null

  const cacheFile = path.resolve(process.cwd(), raw)
  if (!fs.existsSync(cacheFile)) return null

  try {
    const parsed = JSON.parse(fs.readFileSync(cacheFile, "utf8")) as SdkRecoveryCacheFile
    const entry = parsed?.assets?.[String(assetId)]
    return entry && entry.assetId === assetId ? entry : null
  } catch {
    return null
  }
}

function saveSdkRecoveryCache(entry: SdkRecoveryCacheEntry) {
  const raw = process.env.SDK_RECOVERY_CACHE_FILE
  if (!raw) return

  const cacheFile = path.resolve(process.cwd(), raw)
  fs.mkdirSync(path.dirname(cacheFile), { recursive: true })

  let file: SdkRecoveryCacheFile = {
    version: 1,
    assets: {}
  }

  try {
    if (fs.existsSync(cacheFile)) {
      file = JSON.parse(fs.readFileSync(cacheFile, "utf8")) as SdkRecoveryCacheFile
    }
  } catch {
    file = {
      version: 1,
      assets: {}
    }
  }

  file.version = 1
  file.assets[String(entry.assetId)] = entry
  fs.writeFileSync(cacheFile, JSON.stringify(file, null, 2) + "\n", "utf8")
}

function persistSdkPendingExecution(record: PendingSdkExecutionRecord): string | null {
  const raw = process.env.SDK_PENDING_EXECUTIONS_FILE
  if (!raw) return null

  const outFile = path.resolve(process.cwd(), raw)
  fs.mkdirSync(path.dirname(outFile), { recursive: true })
  fs.appendFileSync(outFile, `${JSON.stringify(record)}\n`)
  return outFile
}

export class ParlySDK {
  private account
  private publicClient
  private walletClient

  constructor(args: { privateKeyHex: `0x${string}`; tempoChainId: number; tempoRpcUrl: string }) {
    const chain = defineChain({
      id: args.tempoChainId,
      name: "Tempo",
      nativeCurrency: { name: "TMP", symbol: "TMP", decimals: 18 },
      rpcUrls: { default: { http: [args.tempoRpcUrl] } }
    })

    this.account = privateKeyToAccount(args.privateKeyHex)
    this.publicClient = createPublicClient({ chain, transport: http(args.tempoRpcUrl) })
    this.walletClient = createWalletClient({
      account: this.account,
      chain,
      transport: http(args.tempoRpcUrl)
    })
  }

  async authenticate() {
    return this.walletClient.signMessage({
      account: this.account,
      message: MASTER_AUTH_MESSAGE
    }) as Promise<`0x${string}`>
  }

  async getRecoveryKeypair() {
    const sig = await this.authenticate()
    return deriveRecoveryKeypair(sig)
  }

  async recoverLargestNote(assetId: 1 | 2) {
    const kp = await this.getRecoveryKeypair()
    const latestHeads = await fetchRecoveryHeads(assetId)
    const cached = loadSdkRecoveryCache(assetId)

    if (cached && recoveryHeadsEqual(cached.heads, latestHeads)) {
      return {
        best: cached.best
          ? {
              ...cached.best,
              amount: BigInt(cached.best.amount)
            }
          : null,
        leaves: cached.leaves,
        kp
      }
    }

    const [envelopes, leaves, nullifiers] = await Promise.all([
      fetchEnvelopes(assetId),
      fetchLeaves(assetId),
      fetchNullifiers(assetId)
    ])

    const leafSet = new Set(leaves.map((x: any) => String(x.commitment).toLowerCase()))
    const spentSet = new Set(nullifiers.map((x: any) => String(x.nullifierHash).toLowerCase()))

    let best: any = null

    for (const env of envelopes) {
      try {
        const payload = await openSealedNoteEnvelope(
          kp.publicKeyB64,
          kp.privateKeyB64,
          env.envelope as `0x${string}`
        )

        const reconstructed = await reconstructEnvelopeNote(payload)
        if (String(reconstructed.commitment).toLowerCase() !== String(env.commitment).toLowerCase()) continue
        if (!leafSet.has(String(reconstructed.commitment).toLowerCase())) continue
        if (spentSet.has(String(reconstructed.nullifierHash).toLowerCase())) continue

        const row = {
          payload,
          commitment: reconstructed.commitment,
          nullifierHash: reconstructed.nullifierHash,
          amount: BigInt(payload.amount)
        }

        if (!best || row.amount > best.amount) best = row
      } catch {
        continue
      }
    }

    saveSdkRecoveryCache({
      version: 1,
      assetId,
      heads: latestHeads,
      leaves,
      best: best
        ? {
            payload: best.payload,
            commitment: best.commitment,
            nullifierHash: best.nullifierHash,
            amount: best.amount.toString()
          }
        : null
    })

    return { best, leaves, kp }
  }

  async executeAgenticPayment(params: ExecutePaymentParams): Promise<ExecutePaymentOutcome> {
    let approvalToken: `0x${string}` | null = null
    let submittedHash: `0x${string}` | null = null
    let shouldCleanupApproval = false
    let finalHash: `0x${string}` | null = null
    let replacementReason: "replaced" | "repriced" | "cancelled" | null = null
    let activeNullifierHash = "0x0000000000000000000000000000000000000000000000000000000000000000" as `0x${string}`

    try {
      if (!isAddress(params.destination) || params.destination.toLowerCase() === ZERO_ADDRESS) {
        throw new Error("Destination must be a non-zero address.")
      }
      if (!isAddress(params.poolAddress) || params.poolAddress.toLowerCase() === ZERO_ADDRESS) {
        throw new Error("Pool address must be a non-zero address.")
      }

      const { best, leaves, kp } = await this.recoverLargestNote(params.assetId)
      if (!best) throw new Error("No live note found.")
      activeNullifierHash = best.nullifierHash as `0x${string}`

      const poseidon = await buildPoseidon()
      const zeroElement = BigInt(poseidon.F.toString(poseidon([0n, 0n]))).toString()

      const tree = new MerkleTree(
        32,
        leaves.map((x: any) => String(x.commitment)),
        {
          hashFunction: (l: string | bigint, r: string | bigint) =>
            poseidon.F.toString(poseidon([BigInt(l), BigInt(r)])),
          zeroElement
        }
      )

      const idx = leaves.findIndex(
        (x: any) => String(x.commitment).toLowerCase() === String(best.commitment).toLowerCase()
      )
      if (idx === -1) throw new Error("Note not found in Merkle tree.")

      const { pathElements, pathIndices } = tree.path(idx)

      const localEid = SDK_LOCAL_EID
      const amount = parseUnits(params.amount, 6)

      const feeBps = await this.publicClient.readContract({
        address: params.poolAddress,
        abi: POOL_ABI,
        functionName: "globalFeeBps"
      }) as bigint

      const protocolFee = (amount * feeBps) / 10_000n
      const executorFee = 0n
      const changeAmount = best.amount - amount - protocolFee - executorFee
      if (changeAmount < 0n) throw new Error("Insufficient balance.")

      let option = "0x" as `0x${string}`
      let quotedCrossChainFee = 0n

      if (params.destinationEid !== localEid) {
        option = buildCrossChainPayoutOptions(250000, 0)

        quotedCrossChainFee = await this.publicClient.readContract({
          address: params.poolAddress,
          abi: POOL_ABI,
          functionName: "quoteCrossChainFee",
          args: [params.destinationEid, params.destination, amount, option]
        }) as bigint
      }

      let newChangeCommitment =
        "0x0000000000000000000000000000000000000000000000000000000000000000" as `0x${string}`
      let newChangeEnvelope = "0x" as `0x${string}`
      let newChangeSecret = "0"
      let newChangeNullifier = "0"

      if (changeAmount > 0n) {
        const freshChange = await createFreshNote(
          kp.publicKeyB64,
          best.payload.depositor,
          params.assetId,
          changeAmount.toString(),
          "change"
        )

        newChangeCommitment = freshChange.commitment
        newChangeEnvelope = freshChange.envelope
        newChangeSecret = freshChange.secret
        newChangeNullifier = freshChange.nullifier
      }

      const circuitInputs = {
        root: BigInt(tree.root).toString(),
        nullifierHash: BigInt(best.nullifierHash).toString(),
        protocol_fee: protocolFee.toString(),
        executor_fee: executorFee.toString(),
        total_input_amount: best.payload.amount,
        asset_id: params.assetId.toString(),
        local_eid: String(localEid),
        recipients: [BigInt(params.destination).toString(), ...Array(9).fill("0")],
        amounts: [amount.toString(), ...Array(9).fill("0")],
        dest_eids: [String(params.destinationEid), ...Array(9).fill(String(localEid))],
        new_change_commitment: BigInt(newChangeCommitment).toString(),
        relayer: BigInt(this.account.address).toString(),
        secret: best.payload.secret,
        nullifier: best.payload.nullifier,
        depositor: BigInt(best.payload.depositor).toString(),
        pathElements: pathElements.map((x: string | bigint) => BigInt(x).toString()),
        pathIndices,
        new_change_secret: newChangeSecret,
        new_change_nullifier: newChangeNullifier
      }

      const base = params.proofAssetsBasePath || process.env.PARLY_SDK_ASSETS_PATH || "./assets"

      const { proof, publicSignals } = await snarkjs.groth16.fullProve(
        circuitInputs,
        `${base}/joinsplit.wasm`,
        `${base}/joinsplit_final.zkey`
      )

      const pA = [proof.pi_a[0], proof.pi_a[1]].map(
        (v: string) => `0x${BigInt(v).toString(16)}`
      ) as [`0x${string}`, `0x${string}`]

      const pB = [
        [proof.pi_b[0][1], proof.pi_b[0][0]].map((v: string) => `0x${BigInt(v).toString(16)}`),
        [proof.pi_b[1][1], proof.pi_b[1][0]].map((v: string) => `0x${BigInt(v).toString(16)}`)
      ] as [[`0x${string}`, `0x${string}`], [`0x${string}`, `0x${string}`]]

      const pC = [proof.pi_c[0], proof.pi_c[1]].map(
        (v: string) => `0x${BigInt(v).toString(16)}`
      ) as [`0x${string}`, `0x${string}`]

      const mappedSignals = publicSignals.map((s: string) => BigInt(s))
      const receiptTimeoutMs = Number(process.env.SDK_RECEIPT_TIMEOUT_MS || "300000")
      if (!Number.isFinite(receiptTimeoutMs) || !Number.isInteger(receiptTimeoutMs) || receiptTimeoutMs < 0) {
        throw new Error("SDK_RECEIPT_TIMEOUT_MS invalid")
      }

      if (quotedCrossChainFee > 0n) {
        approvalToken = await this.publicClient.readContract({
          address: params.poolAddress,
          abi: POOL_ABI,
          functionName: "lzFeeToken"
        }) as `0x${string}`

        await safeApproveExactSdk({
          token: approvalToken,
          spender: params.poolAddress,
          amount: quotedCrossChainFee,
          owner: this.account.address,
          publicClient: this.publicClient,
          walletClient: this.walletClient
        })
      }

      submittedHash = await this.walletClient.writeContract({
        address: params.poolAddress,
        abi: POOL_ABI,
        functionName: "batchWithdraw",
        args: [
          pA,
          pB,
          pC,
          mappedSignals as any,
          [params.destination, ...Array(9).fill("0x0000000000000000000000000000000000000000")],
          [amount, ...Array(9).fill(0n)],
          [params.destinationEid, ...Array(9).fill(localEid)],
          newChangeCommitment,
          newChangeEnvelope,
          [option, ...Array(9).fill("0x")]
        ]
      })

      finalHash = submittedHash

      const receipt = await this.publicClient.waitForTransactionReceipt({
        hash: submittedHash,
        timeout: receiptTimeoutMs,
        onReplaced: (replacement) => {
          replacementReason = replacement.reason
          finalHash = replacement.transactionReceipt.transactionHash as `0x${string}`
        }
      })

      if (replacementReason === "cancelled") {
        shouldCleanupApproval = true
        return {
          kind: "terminal_failure",
          hash: finalHash!,
          message: "Payment transaction cancelled before inclusion."
        }
      }

      if (replacementReason === "replaced") {
        shouldCleanupApproval = true
        return {
          kind: "terminal_failure",
          hash: finalHash!,
          message: "Payment transaction replaced before inclusion."
        }
      }

      if (receipt.status !== "success") {
        shouldCleanupApproval = true
        return {
          kind: "terminal_failure",
          hash: finalHash!,
          message: `Payment transaction reverted: ${finalHash}`
        }
      }

      return { kind: "success", hash: finalHash! }
    } catch (e: any) {
      const message = e?.message || String(e)

      if (submittedHash) {
        const persistedTo = persistSdkPendingExecution({
          version: 1,
          recordedAt: Math.floor(Date.now() / 1000),
          submittedHash: finalHash || submittedHash,
          nullifierHash: activeNullifierHash,
          pool: params.poolAddress,
          destination: params.destination,
          destinationEid: params.destinationEid,
          assetId: params.assetId,
          approvalToken,
          approvalSpender: params.poolAddress,
          reason: message
        })

        return {
          kind: "pending_confirmation",
          hash: finalHash || submittedHash,
          message,
          persistedTo
        }
      }

      shouldCleanupApproval = true
      return { kind: "terminal_failure", message }
    } finally {
      if (approvalToken && (shouldCleanupApproval || !submittedHash)) {
        try {
          await safeApproveExactSdk({
            token: approvalToken,
            spender: params.poolAddress,
            amount: 0n,
            owner: this.account.address,
            publicClient: this.publicClient,
            walletClient: this.walletClient
          })
        } catch (cleanupError: any) {
          console.error(`SDK cleanup error: ${cleanupError.message || cleanupError}`)
        }
      }
    }
  }
}

44.8 File: packages/sdk/src/lz-options.ts

import { OptionsBuilder } from "@layerzerolabs/lz-v2-utilities"

// LayerZero option encoding is a pinned-dependency interface. App and SDK code must
// import this local wrapper instead of constructing option bytes ad hoc.
function assertOptionHex(value: unknown): `0x${string}` {
  if (typeof value !== "string" || !value.startsWith("0x") || value.length <= 2) {
    throw new Error("LayerZero options helper returned invalid bytes")
  }
  return value as `0x${string}`
}

export function buildCrossChainPayoutOptions(
  lzReceiveGas: number,
  msgValue: number
): `0x${string}` {
  return assertOptionHex(
    OptionsBuilder.newOptions().addExecutorLzReceiveOption(lzReceiveGas, msgValue).toHex()
  )
}

44.9 File: packages/sdk/scripts/verify-lz-options.mjs

import { buildCrossChainPayoutOptions } from "../dist/lz-options.js"

const option = buildCrossChainPayoutOptions(250000, 0)

if (typeof option !== "string" || !option.startsWith("0x") || option.length <= 2) {
  throw new Error("LayerZero payout options smoke test failed")
}

console.log("LayerZero options wrapper smoke check passed.")

44.10 File: packages/sdk/src/mpp.ts

import { parseUnits } from "viem"
import {
  ParlySDK,
  type ExecutePaymentOutcome
} from "./core.js"

export type MppSessionSpec = {
  sessionId: string
  counterparty: `0x${string}`
  assetId: 1 | 2
  spendLimit: string
  destinationEid: number
  poolAddress: `0x${string}`
}

export type MppSessionRecord = {
  version: 1
  protocol: "parly"
  settlementMode: "parly-private-immediate"
  serviceName: string
  serviceVersion: string
  sessionId: string
  counterparty: `0x${string}`
  assetId: 1 | 2
  spendLimit: string
  destinationEid: number
  poolAddress: `0x${string}`
  status: "created"
}

export type MppSettlementRequest = {
  sessionId: string
  destination: `0x${string}`
  amount: string
  proofAssetsBasePath?: string
}

type MppAdapterMetadata = {
  serviceName?: string
  serviceVersion?: string
}

function resolveServiceName(meta: MppAdapterMetadata): string {
  return meta.serviceName || process.env.MPP_SERVICE_NAME || "parly-mpp-adapter"
}

function resolveServiceVersion(meta: MppAdapterMetadata): string {
  return meta.serviceVersion || process.env.MPP_SERVICE_VERSION || "16.9.9"
}

function assertWithinSpendLimit(amount: string, spendLimit: string) {
  const requested = parseUnits(amount, 6)
  const cap = parseUnits(spendLimit, 6)
  if (requested > cap) {
    throw new Error("MPP session spend limit exceeded.")
  }
}

export class ParlyMppAdapter {
  constructor(
    private sdk: ParlySDK,
    private metadata: MppAdapterMetadata = {}
  ) {}

  async createSession(spec: MppSessionSpec): Promise<MppSessionRecord> {
    return {
      version: 1,
      protocol: "parly",
      settlementMode: "parly-private-immediate",
      serviceName: resolveServiceName(this.metadata),
      serviceVersion: resolveServiceVersion(this.metadata),
      sessionId: spec.sessionId,
      counterparty: spec.counterparty,
      assetId: spec.assetId,
      spendLimit: spec.spendLimit,
      destinationEid: spec.destinationEid,
      poolAddress: spec.poolAddress,
      status: "created"
    }
  }

  async settleFromSession(
    req: MppSettlementRequest,
    session: MppSessionRecord
  ): Promise<ExecutePaymentOutcome> {
    if (req.sessionId !== session.sessionId) {
      throw new Error("MPP session ID mismatch.")
    }

    assertWithinSpendLimit(req.amount, session.spendLimit)

    return this.sdk.executeAgenticPayment({
      destination: req.destination,
      amount: req.amount,
      assetId: session.assetId,
      destinationEid: session.destinationEid,
      poolAddress: session.poolAddress,
      proofAssetsBasePath: req.proofAssetsBasePath
    })
  }
}

44.11 File: packages/sdk/src/index.ts

export * from "./core.js"
export * from "./recovery-keys.js"
export * from "./note-crypto.js"
export * from "./indexer.js"
export * from "./erc20.js"
export * from "./lz-options.js"
export * from "./mpp.js"


---

45. End of Part 3

Part 4 continues immediately below.

---

PARLY.FI MASTER TECHNICAL BLUEPRINT

Final Specification V16.9.9 - Canonical Zero-Trust Engineering Runbook

Part 4 - Witness Harness, Campaign Worker, MCP Server, Acceptance Matrix, Smoke Tests, Final Launch Order, Final CLI, Closure

---

46. Canonical witness-generation harness

46.1 Architectural reasoning

The runbook must not rely on a fake handwritten witness blob.

So V16.9.9 keeps a deterministic witness-generation harness that:
	*	computes a real note commitment,
	*	computes a real nullifier hash,
	*	builds a real 32-level Merkle path,
	*	creates a real output,
	*	creates a real change note,
	*	emits coherent circuit input JSON for snarkjs fullprove.

This harness is for proof rehearsal and verifier sanity testing.

`TEMPO_LZ_EID` is mandatory for this harness.

It must be the real Tempo LayerZero deployment EID for the active environment.

It must never be inferred from a chain ID, silently replaced with a fake local constant, or substituted from a placeholder.

46.2 Amend file: packages/circuits/package.json

{
  "name": "@parly/circuits",
  "private": true,
  "scripts": {
    "build": "circom joinsplit.circom --r1cs --wasm --sym --inspect -o .",
    "gen:witness": "node ./scripts/generate-test-witness.mjs"
  },
  "dependencies": {
    "circomlib": "^2.0.5",
    "circomlibjs": "0.1.7",
    "fixed-merkle-tree": "0.7.3",
    "snarkjs": "0.7.4"
  }
}

46.3 File: packages/circuits/scripts/generate-test-witness.mjs

import "dotenv/config"
import fs from "fs"
import path from "path"
import { fileURLToPath } from "url"
import { buildPoseidon } from "circomlibjs"
import { MerkleTree } from "fixed-merkle-tree"

const __filename = fileURLToPath(import.meta.url)
const __dirname = path.dirname(__filename)

function requireTempoLzEid() {
  const raw = String(
    process.env.TEMPO_LZ_EID ?? process.env.NEXT_PUBLIC_TEMPO_LZ_EID ?? ""
  ).trim()

  if (!raw || raw === "0" || raw.includes("REPLACE_WITH_REAL")) {
    throw new Error(
      "TEMPO_LZ_EID missing or invalid. Provide the real Tempo LayerZero deployment EID."
    )
  }

  let value
  try {
    value = BigInt(raw)
  } catch {
    throw new Error(
      "TEMPO_LZ_EID missing or invalid. Provide the real Tempo LayerZero deployment EID."
    )
  }

  if (value <= 0n) {
    throw new Error(
      "TEMPO_LZ_EID missing or invalid. Provide the real Tempo LayerZero deployment EID."
    )
  }

  return value
}

async function main() {
  const poseidon = await buildPoseidon()

  const poseidon2 = (a, b) => BigInt(poseidon.F.toString(poseidon([BigInt(a), BigInt(b)]))).toString()
  const poseidon4 = (a, b, c, d) =>
    BigInt(poseidon.F.toString(poseidon([BigInt(a), BigInt(b), BigInt(c), BigInt(d)]))).toString()

  const assetId = 1n
  // local_eid is the LayerZero endpoint ID for Tempo, not the Tempo chain ID.
  const localEid = requireTempoLzEid()

  const depositor = BigInt("0x1111111111111111111111111111111111111111")
  const relayer = BigInt("0x2222222222222222222222222222222222222222")
  const recipient = BigInt("0x3333333333333333333333333333333333333333")

  const totalInputAmount = 1_000_000n
  const sendAmount = 400_000n
  const protocolFee = 1_000n
  const executorFee = 0n
  const changeAmount = totalInputAmount - sendAmount - protocolFee - executorFee

  if (changeAmount < 0n) {
    throw new Error("Invalid witness setup: negative change.")
  }

  const secret = 1234567890123456789012345678901234567890n
  const nullifier = 987654321098765432109876543210987654321n

  const newChangeSecret = 555555555555555555555555555555555555555n
  const newChangeNullifier = 777777777777777777777777777777777777777n

  const innerCommitment = poseidon4(secret, nullifier, depositor, assetId)
  const commitment = poseidon2(innerCommitment, totalInputAmount)
  const nullifierHash = poseidon2(secret, nullifier)

  const newChangeInner = poseidon4(newChangeSecret, newChangeNullifier, depositor, assetId)
  const newChangeCommitment = poseidon2(newChangeInner, changeAmount)

  const zero0 = poseidon2(0n, 0n)

  const tree = new MerkleTree(
    32,
    [commitment],
    {
      hashFunction: (l, r) => poseidon2(l, r),
      zeroElement: zero0
    }
  )

  const merkleProof = tree.path(0)

  const recipients = Array(10).fill("0")
  const amounts = Array(10).fill("0")
  const dest_eids = Array(10).fill(localEid.toString())

  recipients[0] = recipient.toString()
  amounts[0] = sendAmount.toString()
  dest_eids[0] = localEid.toString()

  const witness = {
    root: BigInt(tree.root).toString(),
    nullifierHash: BigInt(nullifierHash).toString(),
    protocol_fee: protocolFee.toString(),
    executor_fee: executorFee.toString(),
    total_input_amount: totalInputAmount.toString(),
    asset_id: assetId.toString(),
    local_eid: localEid.toString(),
    recipients,
    amounts,
    dest_eids,
    new_change_commitment: BigInt(newChangeCommitment).toString(),
    relayer: relayer.toString(),
    secret: secret.toString(),
    nullifier: nullifier.toString(),
    depositor: depositor.toString(),
    pathElements: merkleProof.pathElements.map((x) => BigInt(x).toString()),
    pathIndices: merkleProof.pathIndices.map((x) => Number(x)),
    new_change_secret: newChangeSecret.toString(),
    new_change_nullifier: newChangeNullifier.toString()
  }

  const outFile = path.resolve(__dirname, "../witness-fixture.json")
  fs.writeFileSync(outFile, JSON.stringify(witness, null, 2))

  console.log(`Witness written to ${outFile}`)
  console.log(`Root: ${witness.root}`)
  console.log(`Commitment: ${commitment}`)
  console.log(`NullifierHash: ${nullifierHash}`)
  console.log(`NewChangeCommitment: ${newChangeCommitment}`)
}

main().catch((err) => {
  console.error(err)
  process.exit(1)
})


---

47. Campaign worker

47.1 Architectural reasoning

Shield-mining analytics must never become protocol truth.

So the worker is intentionally constrained:
	*	consumes only public deposit history,
	*	performs only integer accounting,
	*	writes only analytics rows,
	*	does not infer private balances,
	*	does not decide spendability.

47.2 File: apps/campaign-worker/package.json

{
  "name": "@parly/campaign-worker",
  "private": true,
  "scripts": {
    "start": "ts-node src/sync.ts",
    "init-db": "psql $RAILWAY_DATABASE_URL -f src/schema.sql"
  },
  "dependencies": {
    "dotenv": "16.4.7",
    "pg": "^8.13.1"
  },
  "devDependencies": {
    "ts-node": "10.9.2",
    "typescript": "5.8.2"
  }
}

47.3 File: apps/campaign-worker/src/schema.sql

CREATE TABLE IF NOT EXISTS shield_wallet_points (
  wallet_address VARCHAR(42) PRIMARY KEY,
  total_deposited_base_units NUMERIC(78, 0) NOT NULL DEFAULT 0,
  shield_points_micro NUMERIC(78, 0) NOT NULL DEFAULT 0,
  asset_breakdown JSONB NOT NULL DEFAULT '{}'::jsonb,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS worker_meta (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

47.4 File: apps/campaign-worker/src/sync.ts

import "dotenv/config"
import { Pool } from "pg"

const db = new Pool({
  connectionString: process.env.RAILWAY_DATABASE_URL
})

const PONDER_URL =
  process.env.PONDER_GRAPHQL_URL || process.env.NEXT_PUBLIC_PONDER_GRAPHQL_URL!

type DepositRow = {
  depositor: string
  amount: string
  assetId: number
}

function parsePositiveInt(raw: string | undefined, fallback: number): number {
  const value = Number(raw ?? String(fallback))
  if (!Number.isFinite(value) || !Number.isInteger(value) || value <= 0) {
    return fallback
  }
  return value
}

const WORKER_PAGE_SIZE = parsePositiveInt(process.env.CAMPAIGN_WORKER_PAGE_SIZE, 500)

async function gql(query: string, variables: Record<string, unknown> = {}) {
  const res = await fetch(PONDER_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ query, variables }),
    cache: "no-store"
  })
  if (!res.ok) {
    throw new Error(`Indexer request failed with HTTP ${res.status}`)
  }

  const json = await res.json()
  if (json.errors?.length) throw new Error(json.errors[0].message)
  return json.data
}

function computeShieldPointsMicro(
  totalBaseUnits: bigint,
  divisor: bigint,
  tokenDecimals = 6n
): bigint {
  const scale = 10n ** tokenDecimals
  return (totalBaseUnits * 1_000_000n) / (scale * divisor)
}

function parsePositiveDivisor(raw: string | undefined): bigint {
  try {
    const value = BigInt(raw || "100")
    if (value <= 0n) {
      throw new Error("SHIELD_MINING_POINTS_DIVISOR must be greater than zero")
    }
    return value
  } catch (e: any) {
    throw new Error(`Invalid SHIELD_MINING_POINTS_DIVISOR: ${e.message || e}`)
  }
}

async function fetchAllDeposits(): Promise<DepositRow[]> {
  const query = `
    query($limit:Int!, $after:String) {
      depositEvents(orderBy:"timestamp", orderDirection:"asc", limit:$limit, after:$after) {
        items { depositor amount assetId }
        pageInfo { hasNextPage endCursor }
      }
    }
  `

  let after: string | null = null
  const out: DepositRow[] = []

  while (true) {
    const data = await gql(query, { limit: WORKER_PAGE_SIZE, after })
    const page = data?.depositEvents
    const items = page?.items || []
    out.push(...items)

    if (!page?.pageInfo?.hasNextPage || !page?.pageInfo?.endCursor) {
      break
    }

    after = page.pageInfo.endCursor
  }

  return out
}

async function saveWorkerHeartbeat(processedWallets: number) {
  await db.query(
    `
    INSERT INTO worker_meta(key, value, updated_at)
    VALUES ($1, $2, NOW())
    ON CONFLICT(key)
    DO UPDATE SET value = EXCLUDED.value, updated_at = NOW()
    `,
    ["last_full_sync_wallet_count", String(processedWallets)]
  )
}

async function main() {
  const divisor = parsePositiveDivisor(process.env.SHIELD_MINING_POINTS_DIVISOR)
  const rows = await fetchAllDeposits()

  const aggregate = new Map<
    string,
    {
      total: bigint
      assets: Record<string, string>
    }
  >()

  for (const row of rows) {
    const wallet = String(row.depositor).toLowerCase()
    const amount = BigInt(row.amount)
    const current = aggregate.get(wallet) || { total: 0n, assets: {} }

    current.total += amount

    const assetKey = String(row.assetId)
    current.assets[assetKey] = (
      BigInt(current.assets[assetKey] || "0") + amount
    ).toString()

    aggregate.set(wallet, current)
  }

  await db.query("BEGIN")

  try {
    for (const [wallet, value] of aggregate.entries()) {
      const pointsMicro = computeShieldPointsMicro(value.total, divisor)

      await db.query(
        `
        INSERT INTO shield_wallet_points(
          wallet_address,
          total_deposited_base_units,
          shield_points_micro,
          asset_breakdown,
          updated_at
        )
        VALUES($1,$2,$3,$4,NOW())
        ON CONFLICT(wallet_address)
        DO UPDATE SET
          total_deposited_base_units = EXCLUDED.total_deposited_base_units,
          shield_points_micro = EXCLUDED.shield_points_micro,
          asset_breakdown = EXCLUDED.asset_breakdown,
          updated_at = NOW()
        `,
        [
          wallet,
          value.total.toString(),
          pointsMicro.toString(),
          JSON.stringify(value.assets)
        ]
      )
    }

    await saveWorkerHeartbeat(aggregate.size)
    await db.query("COMMIT")
  } catch (error) {
    await db.query("ROLLBACK")
    throw error
  }

  console.log(`Synced ${aggregate.size} wallet point rows.`)
}

main()
  .then(async () => {
    await db.end()
  })
  .catch(async (e) => {
    console.error("Campaign Sync Error:", e.message || e)
    await db.end()
    process.exit(1)
  })


---

48. MCP server

48.1 Architectural reasoning

The MCP server is a privileged agent-only automation surface.

Its scope is intentionally narrow:
	*	recover notes owned by the agent key,
	*	execute shielded payments from notes owned by the agent key,
	*	expose a small tool surface,
	*	and, if enabled, expose an MPP-compatible adapter that wraps the same AGENT-owned execution path.

It must not:
	*	use relayer execution key,
	*	use treasury key,
	*	impersonate a human wallet,
	*	spend a third party note,
	*	or introduce contract-level MPP session state.

48.2 File: apps/mcp-server/package.json

{
  "name": "@parly/mcp-server",
  "version": "16.9.9-mcp.0",
  "private": true,
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "typecheck": "tsc -p tsconfig.json --noEmit",
    "start": "pnpm --filter @parly/sdk run build && node --loader ts-node/esm src/index.ts",
    "smoke:config": "node dist/smoke-config.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "1.8.0",
    "@parly/env-utils": "workspace:*",
    "@parly/sdk": "workspace:*",
    "dotenv": "16.4.7",
    "ts-node": "10.9.2",
    "typescript": "5.8.2"
  }
}

48.3 File: apps/mcp-server/src/index.ts

import "dotenv/config"
import { Server } from "@modelcontextprotocol/sdk/server/index.js"
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js"
import {
  CallToolRequestSchema,
  ListToolsRequestSchema
} from "@modelcontextprotocol/sdk/types.js"
import { requireEnv, requirePositiveChainIdValue } from "@parly/env-utils"
import {
  ParlySDK,
  ParlyMppAdapter,
  type ExecutePaymentOutcome,
  type MppSessionRecord
} from "@parly/sdk"

const agentPrivateKey = requireEnv("AGENT_PRIVATE_KEY") as `0x${string}`
const tempoRpcUrl = requireEnv("TEMPO_RPC_URL")
const tempoChainId = requirePositiveChainIdValue(
  process.env.TEMPO_CHAIN_ID ?? process.env.NEXT_PUBLIC_TEMPO_CHAIN_ID,
  "TEMPO_CHAIN_ID/NEXT_PUBLIC_TEMPO_CHAIN_ID"
)
const mppEnabled =
  (process.env.ENABLE_MPP_ADAPTER ?? process.env.NEXT_PUBLIC_ENABLE_MPP_ADAPTER ?? "true").toLowerCase() !==
  "false"
const mppServiceName = process.env.MPP_SERVICE_NAME ?? "parly-mpp-adapter"
const mppServiceVersion = process.env.MPP_SERVICE_VERSION ?? "16.9.9"

const sdk = new ParlySDK({
  privateKeyHex: agentPrivateKey,
  tempoChainId,
  tempoRpcUrl
})
const mpp = new ParlyMppAdapter(sdk, {
  serviceName: mppServiceName,
  serviceVersion: mppServiceVersion
})

const server = new Server(
  { name: "parly-agentic-hub", version: "16.9.9" },
  { capabilities: { tools: {} } }
)

function formatExecutionOutcome(outcome: ExecutePaymentOutcome) {
  if (outcome.kind === "success") {
    return {
      content: [{ type: "text", text: `SUCCESS: ${outcome.hash}` }]
    }
  }

  if (outcome.kind === "pending_confirmation") {
    return {
      content: [
        {
          type: "text",
          text: JSON.stringify({
            status: outcome.kind,
            hash: outcome.hash,
            message: outcome.message,
            persistedTo: outcome.persistedTo
          })
        }
      ]
    }
  }

  return {
    content: [
      {
        type: "text",
        text: JSON.stringify({
          status: outcome.kind,
          hash: outcome.hash || null,
          message: outcome.message
        })
      }
    ],
    isError: true
  }
}

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "recover_largest_note",
      description: "Recover the largest live note controlled by the configured AGENT_PRIVATE_KEY for a given asset.",
      inputSchema: {
        type: "object",
        properties: {
          assetId: {
            type: "number",
            description: "1 for USDC, 2 for USDT"
          }
        },
        required: ["assetId"]
      }
    },
    {
      name: "execute_shielded_payment",
      description: "Execute a private Parly payment immediately from notes controlled by the configured AGENT_PRIVATE_KEY.",
      inputSchema: {
        type: "object",
        properties: {
          destination: { type: "string" },
          amount: { type: "string" },
          assetId: {
            type: "number",
            description: "1 for USDC, 2 for USDT"
          },
          destinationEid: { type: "number" },
          poolAddress: { type: "string" }
        },
        required: ["destination", "amount", "assetId", "destinationEid", "poolAddress"]
      }
    },
    ...(mppEnabled
      ? [
          {
            name: "mpp_create_session",
            description:
              "Create an MPP-compatible Parly session descriptor that wraps the current AGENT-owned execution path.",
            inputSchema: {
              type: "object",
              properties: {
                sessionId: { type: "string" },
                counterparty: { type: "string" },
                assetId: {
                  type: "number",
                  description: "1 for USDC, 2 for USDT"
                },
                spendLimit: { type: "string" },
                destinationEid: { type: "number" },
                poolAddress: { type: "string" }
              },
              required: ["sessionId", "counterparty", "assetId", "spendLimit", "destinationEid", "poolAddress"]
            }
          },
          {
            name: "mpp_settle_session_payment",
            description:
              "Settle a Parly payment from a previously created MPP-compatible session descriptor.",
            inputSchema: {
              type: "object",
              properties: {
                session: { type: "object", description: "The session returned by mpp_create_session." },
                sessionId: { type: "string" },
                destination: { type: "string" },
                amount: { type: "string" }
              },
              required: ["session", "sessionId", "destination", "amount"]
            }
          }
        ]
      : [])
  ]
}))

server.setRequestHandler(CallToolRequestSchema, async (req) => {
  try {
    const args = req.params.arguments as any

    if (req.params.name === "recover_largest_note") {
      const result = await sdk.recoverLargestNote(Number(args.assetId) as 1 | 2)

      return {
        content: [
          {
            type: "text",
            text: JSON.stringify({
              hasNote: !!result.best,
              amount: result.best?.amount?.toString() || null,
              commitment: result.best?.commitment || null
            })
          }
        ]
      }
    }

    if (req.params.name === "execute_shielded_payment") {
      const outcome = await sdk.executeAgenticPayment({
        destination: args.destination,
        amount: args.amount,
        assetId: Number(args.assetId) as 1 | 2,
        destinationEid: Number(args.destinationEid),
        poolAddress: args.poolAddress
      })

      return formatExecutionOutcome(outcome)
    }

    if (req.params.name === "mpp_create_session") {
      if (!mppEnabled) {
        throw new Error("MPP adapter disabled.")
      }

      const session = await mpp.createSession({
        sessionId: args.sessionId,
        counterparty: args.counterparty as `0x${string}`,
        assetId: Number(args.assetId) as 1 | 2,
        spendLimit: args.spendLimit,
        destinationEid: Number(args.destinationEid),
        poolAddress: args.poolAddress as `0x${string}`
      })

      return {
        content: [{ type: "text", text: JSON.stringify(session) }]
      }
    }

    if (req.params.name === "mpp_settle_session_payment") {
      if (!mppEnabled) {
        throw new Error("MPP adapter disabled.")
      }

      const outcome = await mpp.settleFromSession(
        {
          sessionId: args.sessionId,
          destination: args.destination as `0x${string}`,
          amount: args.amount
        },
        args.session as MppSessionRecord
      )

      return formatExecutionOutcome(outcome)
    }

    throw new Error("Tool not found")
  } catch (error: any) {
    return {
      content: [{ type: "text", text: `PROTOCOL ERROR: ${error.message}` }],
      isError: true
    }
  }
})

await server.connect(new StdioServerTransport())
console.error(
  `Parly MCP Server V16.9.9 running with AGENT_PRIVATE_KEY only. MPP adapter ${mppEnabled ? `enabled (${mppServiceName}@${mppServiceVersion})` : "disabled"}.`
)

48.3a File: apps/mcp-server/src/smoke-config.ts

import "dotenv/config"
import { requireEnv, requirePositiveChainIdValue } from "@parly/env-utils"
import { ParlySDK, ParlyMppAdapter } from "@parly/sdk"

const agentPrivateKey = requireEnv("AGENT_PRIVATE_KEY") as `0x${string}`
const tempoRpcUrl = requireEnv("TEMPO_RPC_URL")
const tempoChainId = requirePositiveChainIdValue(
  process.env.TEMPO_CHAIN_ID ?? process.env.NEXT_PUBLIC_TEMPO_CHAIN_ID,
  "TEMPO_CHAIN_ID/NEXT_PUBLIC_TEMPO_CHAIN_ID"
)

const sdk = new ParlySDK({
  privateKeyHex: agentPrivateKey,
  tempoChainId,
  tempoRpcUrl
})

new ParlyMppAdapter(sdk, {
  serviceName: process.env.MPP_SERVICE_NAME ?? "parly-mpp-adapter",
  serviceVersion: process.env.MPP_SERVICE_VERSION ?? "16.9.9"
})

console.log("MCP standalone config smoke check passed.")


---

49. Canonical encrypted relay payload reference

49.1 Architectural reasoning

Private execution breaks if frontend and relayer disagree about payload shape.

V16.9.9 freezes one canonical launch payload and exact cardinalities:
	*	pubSignals.length === 50
	*	recipients.length === 10
	*	amounts.length === 10
	*	destEids.length === 10
	*	lzOptions.length === 10

49.2 Canonical payload type

export type CanonicalRelayExecutionPayload = {
  pool: `0x${string}`
  pA: [`0x${string}`, `0x${string}`]
  pB: [[`0x${string}`, `0x${string}`], [`0x${string}`, `0x${string}`]]
  pC: [`0x${string}`, `0x${string}`]
  pubSignals: string[]
  recipients: `0x${string}`[]
  amounts: string[]
  destEids: number[]
  newChangeCommitment: `0x${string}`
  newChangeEnvelope: `0x${string}`
  lzOptions: `0x${string}`[]
}


---

50. Final acceptance criteria

50.1 Architectural reasoning

A launch is not accepted because the repo is long.

It is accepted only if the exact launch surface behaves correctly under adversarial and operationally realistic conditions.

50.2 Acceptance matrix

Precondition for release sign-off: repo-root `pnpm typecheck` and repo-root `pnpm build` must both pass after the exact file tree is assembled. They are hard release gates, not optional hygiene.

50.2.1 Cryptographic and proof integrity
	1.	The generated verifier accepts the intended 50-signal ordering and rejects reordered arrays.
	2.	A proof bound to relayer A reverts when submitted by relayer B.
	3.	Calldata/proof desync reverts across all 10 slots, including padded slots.
	4.	Zero-change spends require zero external change fields.
	5.	Unknown notes cannot produce valid spends.
	6.	Asset mismatch reverts.
	7.	Local EID mismatch reverts.
	8.	Reused nullifiers are rejected.
	9.	Malformed envelopes cannot reconstruct valid on-chain commitments.
	10.	Browser and SDK reconstruct identical commitments from the same envelope.

50.2.2 Witness harness and proof rehearsal
	11.	pnpm gen:witness emits coherent witness JSON.
	12.	snarkjs groth16 fullprove succeeds using the generated witness.
	13.	The harness exercises the nonzero change-note path.
	14.	The exported verifier corresponds to the same final .zkey.

50.2.3 Pool behavior
	15.	deposit() transfers funds, inserts a leaf, emits Deposit, LeafInserted, NoteEnvelope.
	16.	depositFromIngress() succeeds only for configured composer.
	17.	TIP-403 blocks unauthorized direct depositors.
	18.	TIP-403 blocks unauthorized ingress depositors.
	19.	Protocol fee transfers exactly once after successful verification.
	20.	Executor fee transfers exactly once to the actual executor.
	21.	Unsupported destination EIDs revert.
	22.	Padded zero slots require zero recipient, zero amount, local EID, empty LZ options.
	23.	Duplicate commitments are rejected.
	24.	Duplicate change commitments are rejected.
	25.	Tree capacity guard remains safe under the uint64 nextLeafIndex layout.

50.2.4 Omnichain behavior
	26.	A supported spoke gateway can shield into the Tempo composer and create a valid Tempo leaf.
	27.	Unsupported source gateways are rejected.
	28.	Same-chain Tempo payout succeeds only with a nonzero recipient.
	29.	Supported cross-chain payout succeeds through the hub OFT only with a nonzero recipient.
	30.	Unsupported cross-chain destinations and zero-recipient outputs revert.
	31.	Alt-fee token mode succeeds with explicit approval and msg.value == 0.
	32.	Native-fee spoke mode succeeds with exact msg.value.
	33.	Hub composer only accepts the configured endpoint.
	34.	Hub composer only accepts the correct local OFT for the selected asset.
	35.	Hub composer trusts spoke gateway as compose sender, not the spoke OFT.

50.2.5 Recovery behavior
	36.	A fresh browser profile can authenticate and recover notes from chain envelopes alone.
	37.	Local cache can be fully disabled without breaking recovery.
	38.	Optional local cache stores ciphertext only.
	39.	Recovery recomputes commitment and matches on-chain leaf.
	40.	Recovery recomputes nullifier hash and correctly marks live vs spent.
	41.	Corrupt cache data is deleted and chain recovery still succeeds.
	42.	Recovery does not depend on previously cached plaintext notes, and the hook does not depend directly on unstable Uint8Array reference identity.

50.2.6 Partial-spend behavior
	43.	Spending less than note amount creates a valid change note.
	44.	Full-spend produces no change note.
	45.	The consumed note becomes spent after payout.
	46.	Unrelated notes remain recoverable.
	47.	Same-chain partial spend computes exact change after fees.
	48.	Cross-chain partial spend computes exact change after fees and fee-token requirements.

50.2.7 Approval sequencing and token quirks
	49.	Browser deposit approval waits for receipt before deposit call, and terminal direct-deposit failures clean the approval instead of leaving it at rest.
	50.	Browser direct-execution fee-token approval waits for receipt before batchWithdraw, and uncertain post-broadcast states do not pretend to be clean failures.
	51.	Relayer fee-token approval waits for receipt before simulate and execute.
	52.	SDK fee-token approval waits for receipt before batchWithdraw.
	53.	Browser approval helpers handle zero-reset ERC20 behavior.
	54.	Relayer approval helpers use exact-amount fee approvals, bounded pre-submit retry, and cleanup only when submission never happens or terminal failure is known.
	55.	SDK approval helpers use exact-amount fee approvals and cleanup only when submission never happens or terminal failure is known.

50.2.8 Relayer behavior
	56.	Relayer decrypts only bundles addressed to it.
	57.	Relayer rejects malformed bundle envelopes and enforces exact payload lengths and numeric encodings before quote or simulate paths.
	58.	Relayer checks on-chain nullifier state before simulation.
	59.	Relayer re-quotes every cross-chain output before execution.
	60.	Relayer converts quoted LZ fee-token cost into normalized USD(6) and computes a recommended executor-fee floor from route shape and delivery cost.
	61.	Relayer skips execution when executorFee is below that recommended floor.
	62.	Re-broadcasting executed ciphertext does not result in second payout.
	63.	A copied encrypted bundle cannot be stolen by a different relayer account.
	64.	Bundles bound to another relayer are dropped.
	65.	Oversized Waku payloads are rejected, one malformed bundle does not abort later valid bundles in the same payload, and exhausted pre-submit transient failures are persisted.
	66.	Relayer marks nullifier seen only after confirmed execution receipt, and uncertain post-broadcast states are written to RELAYER_PENDING_EXECUTIONS_FILE instead of triggering eager approval revocation.

50.2.9 Relay topology honesty
	67.	One proof is bound to one relayer.
	68.	Browser encrypts one bundle to one selected relayer.
	69.	UI does not falsely imply on-chain finality after Waku publish or after an uncertain direct-browser receipt wait.
	70.	Relay-mode status and direct-browser PENDING_CONFIRMATION are distinct from confirmed direct execution.

50.2.10 Indexer behavior
	71.	Ponder boots successfully with current config layout.
	72.	ponder:registry import resolves in src/index.ts.
	73.	GraphQL is served from /graphql.
	74.	BatchWithdrawal ABI includes indexed assetId.
	75.	BatchWithdrawal rows use event.args.assetId rather than hardcoded asset IDs.
	76.	Leaf, envelope, deposit, and batch rows appear in GraphQL after real transactions.
	77.	Browser and SDK history fetchers walk indexer history in bounded pages instead of one unbounded request.

50.2.11 Spoke-ingress and recovery operations
	78.	The browser spoke shield page can quote on the selected source chain using the configured spoke gateway.
	79.	Successful spoke shielding waits for the exact new Tempo leaf instead of treating source-chain broadcast as final ingress.
	80.	If source-chain shield succeeds but Tempo ingress is delayed or deferred, the UI surfaces a pending/deferred state instead of a fake success state.
	81.	Deferred ingress has exactly two ordinary user recovery paths: retry ingress or claim refund.
	82.	Retry re-runs the real depositFromIngress path and therefore naturally re-runs the pool-side TIP-403 gate.
	83.	Claim refund returns principal only to the original recorded depositor, is not a new deposit, and does not re-run TIP-403.
	84.	Refund does not reimburse already-spent gas or LayerZero messaging fees.
	85.	No ordinary deferred refund depends on discretionary admin review.
	86.	Changing the connected wallet clears browser recovery keys, signature state, selected commitment, and pending hashes.

50.2.12 SDK and MCP behavior
	84.	SDK recovers the same largest note the browser would recover.
	85.	SDK can execute same-chain payments from its own recovered note.
	86.	SDK can execute cross-chain payments with hardened exact-approval flow without unsafe eager post-broadcast cleanup.
	87.	SDK creates a real change note when change is nonzero, using byte-to-hex conversion that does not require Node-only Buffer globals.
	88.	MCP server can recover agent-owned notes.
	89.	MCP server can execute payments from agent-owned notes.
	90.	MCP server cannot spend notes it does not control.

50.2.12a MPP adapter behavior
	*	If the MPP adapter is enabled, the SDK re-exports `ParlyMppAdapter` from the public package surface.
	*	`createSession()` returns a deterministic session descriptor and does not write on-chain session state.
	*	`settleFromSession()` wraps `executeAgenticPayment()` and preserves the same `success` / `pending_confirmation` / `terminal_failure` outcome model.
	*	MCP exposes `mpp_create_session` and `mpp_settle_session_payment` only when the adapter is enabled.
	*	MPP tooling uses `AGENT_PRIVATE_KEY` only and never instantiates relayer authority.
	*	MPP compatibility stays at the SDK/MCP boundary and does not alter pool, relayer, or circuit semantics.

50.2.12b Official relayer registry behavior
	*	`/relayer` loads as a single-page open form without any review queue or moderation dashboard.
	*	Valid address + valid Base64 public key + valid signature + available slot results in immediate `verified` registration.
	*	Duplicate execution addresses are rejected.
	*	Duplicate public keys are rejected.
	*	The primary registration action is disabled when the live registry count reaches the 10-slot cap.
	*	`GET /api/relayers` returns verified live relayers only, shaped as `executionAddress` plus `publicKeyB64`.
	*	Runtime frontend relayer discovery prefers the backend registry and uses the legacy env string only as fallback.
	*	Successful relayer activity updates `last_success_at` and `last_seen_at`.
	*	Inactive relayers stop counting toward the 10-slot cap.
	*	No manual review flow, moderation console, or registry ranking table exists in launch scope.

50.2.13 Campaign worker behavior
	91.	A 0.5 USDC deposit with divisor 100 yields exactly 5000 micro-points.
	92.	No float math is used for stored accounting, and invalid or zero divisors fail fast.
	93.	Re-running the worker on same history is idempotent.
	94.	The worker consumes only public deposit events.

50.2.14 Scope integrity
	95.	No scheduling queue exists in launch scope.
	96.	No delayed-execution runtime exists in launch scope.
	97.	No swap router exists in launch scope.
	98.	Launch payouts are stablecoin-only.
	99.	No monopoly relayer is required for correctness.
	100.	No plaintext local note storage is required for correctness.
	101.	Test-witness generation succeeds without variable shadowing in the helper harness.

---

51. Zero-trust launch order

51.1 Architectural reasoning

Correct components still fail if launched in the wrong order.

51.2 Final deployment order
	1.	Initialize monorepo and install dependencies.
	2.	Fill root .env.
	3.	Build the circuit.
	4.	Run Groth16 setup.
	5.	Contribute entropy.
	6.	Record final .zkey hash.
	7.	Export verifier contract.
	8.	Generate witness harness.
	9.	Rehearse fullprove.
	10.	Copy proving artifacts to web and SDK assets.
	11.	Copy generated verifier into contracts workspace.
	12.	Build contracts.
	13.	Run contract tests.
	14.	Deploy Poseidon.
	15.	Deploy verifier.
	16.	Deploy treasury.
	17.	Deploy hub core.
	18.	Deploy hub OFTs or OFT adapters.
	19.	Deploy spoke OFTs or OFT adapters.
	20.	Deploy spoke gateways.
	21.	Configure hub pools and composer.
	22.	Wire LayerZero peers.
	23.	Start indexer.
	24.	Apply the relayer registry migration and start frontend.
	25.	Generate relayer box keypair.
	26.	Start relayer.
	27.	Register relayer at `/relayer`.
	28.	Start relayer pending reconciler.
	29.	Start campaign worker.
	30.	Start MCP server.
	31.	Execute full acceptance matrix under deployer ownership.
	32.	Rehearse fresh-device recovery.
	33.	Rehearse relayer replay rejection.
	34.	Rehearse same-chain payout.
	35.	Rehearse spoke-to-Tempo shielding.
	36.	Rehearse cross-chain payout.
	37.	Transfer ownership to protocol admin after smoke success.
	38.	Freeze verified deployment record.

---

52. Final canonical CLI runbook

52.1 Toolchain and repo

# ------------------------------------------------------------------------------
# STEP 1 - INSTALL FOUNDRY
# ------------------------------------------------------------------------------
foundryup

# ------------------------------------------------------------------------------
# STEP 2 - INITIALIZE REPO
# ------------------------------------------------------------------------------
chmod +x init-parly-v16_9_9.sh
./init-parly-v16_9_9.sh
cd parly-fi-v16_9_9

cp .env.example .env
# Fill every required key, address, RPC, OFT, endpoint, signer, relayer, and chain value

52.2 Circuit build, ceremony, witness rehearsal

# ------------------------------------------------------------------------------
# STEP 3 - BUILD CIRCUIT
# ------------------------------------------------------------------------------
cd packages/circuits
pnpm install
pnpm build

wget https://storage.googleapis.com/zkevm/ptau/powersOfTau28_hez_final_14.ptau
b2sum powersOfTau28_hez_final_14.ptau
# expected BLAKE2b:
# eeefbcf7c3803b523c94112023c7ff89558f9b8e0cf5d6cdcba3ade60f168af4a181c9c21774b94fbae6c90411995f7d854d02ebd93fb66043dbb06f17a831c1

npx snarkjs groth16 setup joinsplit.r1cs powersOfTau28_hez_final_14.ptau joinsplit_0000.zkey
npx snarkjs zkey contribute joinsplit_0000.zkey joinsplit_final.zkey --name="PARLY_v16_9_9_ENTROPY" -v
npx snarkjs zkey export solidityverifier joinsplit_final.zkey ZKVerifier.sol

# ------------------------------------------------------------------------------
# STEP 4 - GENERATE WITNESS HARNESS
# ------------------------------------------------------------------------------
pnpm gen:witness

# ------------------------------------------------------------------------------
# STEP 5 - REHEARSE FULLPROVE
# ------------------------------------------------------------------------------
npx snarkjs groth16 fullprove witness-fixture.json joinsplit.wasm joinsplit_final.zkey

mkdir -p ../../apps/web/public/circuits
mkdir -p ../../packages/sdk/assets

cp joinsplit.wasm ../../apps/web/public/circuits/
cp joinsplit_final.zkey ../../apps/web/public/circuits/
cp joinsplit.wasm ../../packages/sdk/assets/
cp joinsplit_final.zkey ../../packages/sdk/assets/

cd ../contracts

52.3 Contracts build and deployment

# ------------------------------------------------------------------------------
# STEP 6 - COPY GENERATED VERIFIER
# ------------------------------------------------------------------------------
node script/CopyGeneratedVerifier.js

# ------------------------------------------------------------------------------
# STEP 7 - REPO-WIDE TYPECHECK, BUILD, AND CONTRACT TEST GATES
# ------------------------------------------------------------------------------
# Re-run here in the canonical launch flow so env-stub, generated-artifact, and workspace
# assembly drift are caught again immediately before deployment broadcast.
cd ../..
pnpm typecheck
pnpm build
cd packages/contracts

forge build
forge test -vvv

# ------------------------------------------------------------------------------
# STEP 8 - DEPLOY POSEIDON
# ------------------------------------------------------------------------------
node script/DeployPoseidon.js
# Copy POSEIDON_HASHER_ADDRESS into .env

# ------------------------------------------------------------------------------
# STEP 9 - DEPLOY VERIFIER
# ------------------------------------------------------------------------------
forge script script/DeployVerifier.s.sol --rpc-url $TEMPO_RPC_URL --broadcast -vvvv
# Copy GROTH16_VERIFIER_ADDRESS into .env

# ------------------------------------------------------------------------------
# STEP 10 - DEPLOY TREASURY
# ------------------------------------------------------------------------------
forge script script/DeployTreasury.s.sol --rpc-url $TEMPO_RPC_URL --broadcast -vvvv
# Copy PROTOCOL_TREASURY_ADDRESS into .env

# ------------------------------------------------------------------------------
# STEP 11 - DEPLOY HUB CORE
# ------------------------------------------------------------------------------
forge script script/DeployHubCore.s.sol --rpc-url $TEMPO_RPC_URL --broadcast -vvvv
# Copy NEXT_PUBLIC_USDC_POOL, NEXT_PUBLIC_USDT_POOL, NEXT_PUBLIC_HUB_COMPOSER into .env

52.4 OFTs, gateways, and hub configuration

# ------------------------------------------------------------------------------
# STEP 12 - DEPLOY HUB OFTs / ADAPTERS
# ------------------------------------------------------------------------------
# Use the approved LayerZero ERC20-adapter-compatible deployment flow for Tempo.
# Populate HUB_USDC_OFT and HUB_USDT_OFT.

# ------------------------------------------------------------------------------
# STEP 13 - DEPLOY SPOKE OFTs / ADAPTERS
# ------------------------------------------------------------------------------
# Use the approved LayerZero ERC20-adapter-compatible flow on each spoke.
# Populate all SPOKE_USDC_OFT_* and SPOKE_USDT_OFT_* values.

# ------------------------------------------------------------------------------
# STEP 14 - DEPLOY SPOKE GATEWAYS
# ------------------------------------------------------------------------------
PARLY_ASSET=USDC forge script script/DeploySpokeGateway.s.sol --rpc-url $ETHEREUM_RPC_URL --broadcast -vvvv
PARLY_ASSET=USDC forge script script/DeploySpokeGateway.s.sol --rpc-url $ARBITRUM_RPC_URL --broadcast -vvvv
PARLY_ASSET=USDC forge script script/DeploySpokeGateway.s.sol --rpc-url $BASE_RPC_URL --broadcast -vvvv
PARLY_ASSET=USDC forge script script/DeploySpokeGateway.s.sol --rpc-url $BSC_RPC_URL --broadcast -vvvv

PARLY_ASSET=USDT forge script script/DeploySpokeGateway.s.sol --rpc-url $ETHEREUM_RPC_URL --broadcast -vvvv
PARLY_ASSET=USDT forge script script/DeploySpokeGateway.s.sol --rpc-url $ARBITRUM_RPC_URL --broadcast -vvvv
PARLY_ASSET=USDT forge script script/DeploySpokeGateway.s.sol --rpc-url $BASE_RPC_URL --broadcast -vvvv
PARLY_ASSET=USDT forge script script/DeploySpokeGateway.s.sol --rpc-url $BSC_RPC_URL --broadcast -vvvv

# ------------------------------------------------------------------------------
# STEP 15 - UPDATE GATEWAY ADDRESSES IN .ENV
# ------------------------------------------------------------------------------
# Fill NEXT_PUBLIC_SPOKE_USDC_GATEWAY_* and NEXT_PUBLIC_SPOKE_USDT_GATEWAY_*
# Also fill START_BLOCK_HUB_COMPOSER and every START_BLOCK_SPOKE_* value so
# source dispatch and Tempo settlement provenance are indexed from the correct blocks.

# ------------------------------------------------------------------------------
# STEP 16 - CONFIGURE HUB
# ------------------------------------------------------------------------------
forge script script/ConfigureHub.s.sol --rpc-url $TEMPO_RPC_URL --broadcast -vvvv

# ------------------------------------------------------------------------------
# STEP 17 - WIRE LAYERZERO PATHWAYS
# ------------------------------------------------------------------------------
npx hardhat lz:oapp:wire --oapp-config layerzero.config.ts

# ------------------------------------------------------------------------------
# STEP 18 - HOLD OWNERSHIP UNDER DEPLOYER FOR SMOKE
# ------------------------------------------------------------------------------
# Do not transfer ownership yet. The next steps intentionally boot and smoke-test
# the system while deployer ownership still exists, because recovery and final
# admin handoff should only happen after end-to-end proof.

52.5 Service boot and smoke

# ------------------------------------------------------------------------------
# STEP 19 - START INDEXER
# ------------------------------------------------------------------------------
cd ../../apps/indexer
pnpm install
pnpm codegen
pnpm dev

# ------------------------------------------------------------------------------
# STEP 20 - START WEB
# ------------------------------------------------------------------------------
cd ../web
pnpm install
pnpm run init:relayers
pnpm dev
# Configure a daily POST to /api/cron/relayers/inactive with x-parly-cron-secret
# so verified relayers that go quiet for 30 days are marked inactive automatically.

# ------------------------------------------------------------------------------
# STEP 21 - GENERATE RELAYER BOX KEYPAIR
# ------------------------------------------------------------------------------
cd ../relayer
pnpm install
pnpm keygen:box
# Copy RELAYER_ADDRESS and RELAYER_BOX_PRIVATE_KEY_B64 from the generated
# secret file path printed by the command into the root .env.
# Then register the printed execution address and Base64 public key at /relayer.
# NEXT_PUBLIC_RELAYER_REGISTRY remains a legacy fallback only.

# ------------------------------------------------------------------------------
# STEP 22 - START RELAYER
# ------------------------------------------------------------------------------
pnpm start

# ------------------------------------------------------------------------------
# STEP 22A - REGISTER RELAYER IN THE OFFICIAL RUNTIME REGISTRY
# ------------------------------------------------------------------------------
# Open /relayer in the frontend, submit the execution address and Base64 public key,
# connect the same execution wallet, and sign the registration message.

# ------------------------------------------------------------------------------
# STEP 22B - START RELAYER PENDING RECONCILER
# ------------------------------------------------------------------------------
pnpm reconcile:pending:loop
# Leave this running beside the relayer. Use `pnpm reconcile:pending` for a one-off pass.

# ------------------------------------------------------------------------------
# STEP 23 - START CAMPAIGN WORKER
# ------------------------------------------------------------------------------
cd ../campaign-worker
pnpm install
pnpm run init-db
pnpm start

# ------------------------------------------------------------------------------
# STEP 24 - START MCP SERVER
# ------------------------------------------------------------------------------
cd ../mcp-server
pnpm install
pnpm start
# If ENABLE_MPP_ADAPTER=true, confirm the MCP tool list includes:
# - mpp_create_session
# - mpp_settle_session_payment

# ------------------------------------------------------------------------------
# STEP 25 - RUN THE FULL SMOKE / ACCEPTANCE MATRIX
# ------------------------------------------------------------------------------
# Run the smoke checklist and acceptance matrix below while contracts are still
# under deployer ownership. Only continue once the deployer-owned system passes.

# ------------------------------------------------------------------------------
# STEP 26 - TRANSFER OWNERSHIP AFTER SMOKE SUCCESS
# ------------------------------------------------------------------------------
cd ../../packages/contracts
forge script script/TransferTempoOwnership.s.sol --rpc-url $TEMPO_RPC_URL --broadcast -vvvv

PARLY_ASSET=USDC forge script script/TransferSpokeGatewayOwnership.s.sol --rpc-url $ETHEREUM_RPC_URL --broadcast -vvvv
PARLY_ASSET=USDC forge script script/TransferSpokeGatewayOwnership.s.sol --rpc-url $ARBITRUM_RPC_URL --broadcast -vvvv
PARLY_ASSET=USDC forge script script/TransferSpokeGatewayOwnership.s.sol --rpc-url $BASE_RPC_URL --broadcast -vvvv
PARLY_ASSET=USDC forge script script/TransferSpokeGatewayOwnership.s.sol --rpc-url $BSC_RPC_URL --broadcast -vvvv

PARLY_ASSET=USDT forge script script/TransferSpokeGatewayOwnership.s.sol --rpc-url $ETHEREUM_RPC_URL --broadcast -vvvv
PARLY_ASSET=USDT forge script script/TransferSpokeGatewayOwnership.s.sol --rpc-url $ARBITRUM_RPC_URL --broadcast -vvvv
PARLY_ASSET=USDT forge script script/TransferSpokeGatewayOwnership.s.sol --rpc-url $BASE_RPC_URL --broadcast -vvvv
PARLY_ASSET=USDT forge script script/TransferSpokeGatewayOwnership.s.sol --rpc-url $BSC_RPC_URL --broadcast -vvvv

52.6 Pending execution and deferred-ingress recovery

If a relayer, SDK, or browser flow lands in a pending or deferred state, use the following rules:
	1.	Do not revoke source-chain or Tempo fee approvals just because a receipt wait timed out after broadcast.
	2.	Run `pnpm reconcile:pending` or keep `pnpm reconcile:pending:loop` live before doing any manual relayer cleanup.
	3.	If a spoke shield source transaction succeeded but the Tempo leaf did not appear, inspect the hub composer for the deferred `guid`.
	4.	Deferred ingress has exactly two ordinary user recovery actions: retry ingress or claim refund.
	5.	Retry is the technical recovery path. After fixing the root cause, such as gateway approval or pool wiring, use `PARLY_CALLER_PRIVATE_KEY=<authorized_eoa_key> PARLY_DEFERRED_GUID=<guid> PARLY_DEFERRED_ACTION=retry forge script script/ResolveDeferredIngress.s.sol --rpc-url $TEMPO_RPC_URL --broadcast -vvvv`.
	6.	Claim is the principal-return path. It sends principal only to the original recorded depositor and intentionally does not re-run TIP-403 because it is not a new pool ingress. Use `PARLY_CALLER_PRIVATE_KEY=<depositor_key> PARLY_DEFERRED_GUID=<guid> PARLY_DEFERRED_ACTION=claim forge script script/ResolveDeferredIngress.s.sol --rpc-url $TEMPO_RPC_URL --broadcast -vvvv`.
	7.	Refund does not reimburse already-spent gas or LayerZero messaging fees.
	8.	If the owner is a multisig, build calldata and submit it through the multisig instead of pretending a deployer-key script is still canonical:
cast calldata "retryDeferredIngress(bytes32)" $PARLY_DEFERRED_GUID
cast calldata "claimDeferredIngress(bytes32)" $PARLY_DEFERRED_GUID
	9.	For SDK pending outcomes, inspect the hash first, then reconcile the on-chain state before attempting any replacement or allowance revocation.


---

53. Smoke-test checklist

53.1 Direct deposit smoke test

Approve 1 USDC to the Tempo pool and call deposit.

Confirm:
	*	token moved,
	*	one leaf inserted,
	*	Deposit emitted,
	*	NoteEnvelope emitted.

53.2 Recovery smoke test

Open a fresh browser profile.

Authenticate with the same wallet.

Confirm:
	*	recovery succeeds from chain envelopes alone,
	*	disabled local cache does not break recovery,
	*	recovered note amount matches deposit.

53.3 Same-chain payout smoke test

Spend one recovered note to a Tempo recipient.

Confirm:
	*	proof passes,
	*	selected note becomes spent,
	*	recipient receives funds,
	*	child key appears in BatchWithdrawal,
	*	the Execute fee breakdown shows no cross-chain delivery fee row in either Private Relay or Self-Relay mode,
	*	and Ledger & Compliance, the scoped receipt preview, the PDF, and Verify show no cross-chain delivery fee field for that same-chain payout lane.

53.4 Partial-spend smoke test

Select a note larger than the send amount.

Spend only part of it.

Confirm:
	*	payout succeeds,
	*	original note becomes spent,
	*	one new change note is emitted,
	*	recovered change amount matches exact remainder.

53.5 Spoke-shield smoke test

Open the spoke shield page and submit one small USDC or USDT shield from a configured spoke.

Confirm:
	*	source-chain quote succeeds,
	*	the wallet switches to the selected source chain when needed,
	*	the source transaction lands,
	*	the Tempo leaf appears for the exact quoted commitment,
	*	the source gateway emits a `guid` that also appears on the Tempo ingress settlement event,
	*	the indexer can resolve source tx hash and Tempo settlement tx hash from that shared `guid`,
	*	and if ingress is deliberately blocked, the UI surfaces a pending/deferred state instead of faking success.

53.6 Cross-chain payout smoke test

Spend one note to a supported spoke destination.

Confirm:
	*	quote path works,
	*	hardened fee-token approval succeeds,
	*	destination recipient receives bridged stablecoin,
	*	proof note is consumed once,
	*	Private Relay mode shows no separate cross-chain delivery fee row and keeps that shared route cost inside relay execution pricing for this version,
	*	Self-Relay mode shows the cross-chain delivery fee only inside the Execute fee breakdown,
	*	and Ledger & Compliance, the scoped receipt preview, the PDF, and Verify still suppress any separate cross-chain delivery fee field for the disclosed payout lane.

53.7 Relayer theft smoke test

Copy a Waku payload intended for relayer A.

Attempt execution from relayer B.

Confirm:
	*	relayer B cannot decrypt or execute it,
	*	relayer mismatch blocks theft.

53.8 Replay smoke test

Re-broadcast an already executed ciphertext.

Confirm:
	*	relayer checks nullifier state,
	*	relayer waits for the original receipt,
	*	second execution is skipped.

53.8.1 Official relayer registry smoke test

Open `/relayer`, load live registry state from `GET /api/relayers`, and complete one signature-backed registration with a new execution address plus public key.

Confirm:
	*	the page loads as a single open form,
	*	the docs buttons route correctly,
	*	duplicate address and duplicate public key paths reject with the correct error family,
	*	the registry disables the primary action when the 10-slot cap is reached,
	*	and the runtime registry response remains shaped as `executionAddress` plus `publicKeyB64`.

53.9 SDK smoke test

Use the SDK with an agent-owned note.

Confirm:
	*	SDK recovers the largest live note,
	*	SDK executes a same-chain payment,
	*	SDK creates a real change note,
	*	SDK uses hardened exact-approval flow before cross-chain execution.

53.9.1 MPP adapter smoke test

If `ENABLE_MPP_ADAPTER=true`, create one session through the SDK, settle one payment from that session, and query the MCP tool list.

Confirm:
	*	the SDK exports `ParlyMppAdapter`,
	*	the session descriptor is deterministic and contains only adapter-layer metadata,
	*	the session settlement path returns the same structured outcome kind as direct SDK execution,
	*	MCP lists `mpp_create_session` and `mpp_settle_session_payment`,
	*	and no MPP flow requires RELAYER credentials or any pool contract changes.

53.10 Indexer smoke test

Trigger one deposit and one spend.

Confirm:
	*	depositEvents query returns deposit row,
	*	ingressSettlementEvents query returns a Tempo settlement row for spoke-origin deposits,
	*	spokeShieldDispatchEvents query returns the source dispatch row for the same `guid`,
	*	leafInsertedEvents query returns leaf row,
	*	noteEnvelopeEvents query returns envelope row,
	*	batchWithdrawalEvents query returns spend row with correct assetId,
	*	/graphql responds successfully,
	*	and paged history fetch still returns the full asset history without a single monolithic query.

53.11 Pending-state smoke test

Force a relayer receipt-wait failure after broadcast, a browser direct Tempo receipt-wait failure after broadcast, or manually replace a submitted SDK transaction with a cancellation.

Confirm:
	*	exact fee approval is not eagerly revoked immediately after broadcast,
	*	browser direct execution surfaces PENDING_CONFIRMATION instead of resetting to a misleading clean idle state,
	*	a relayer uncertain state is appended to RELAYER_PENDING_EXECUTIONS_FILE,
	*	the pending reconciler later archives resolved relay states safely,
	*	a definitively cancelled or reverted transaction does trigger safe approval cleanup,
	*	and successful execution still consumes the exact approved fee amount to zero naturally.

53.12 History, receipt, and verify smoke test

Authenticate in the browser, recover at least one spent note, open Parly Ledger & Compliance, export one CSV, review one payout scope, export one Scoped Compliance Receipt PDF, and then verify the same lane on the Verify route using the confirmed Tempo tx hash plus child key.

Confirm:
	*	the embedded Ledger & Compliance surface on `/app` does not front-load its heavier decode path before the operator opens it,
	*	ledger cards label facts as chain-indexed, calldata-decoded, local, or provenance-backed instead of overclaiming,
	*	the CSV export contains only surfaced row fields,
	*	payout-scope review stays lane-scoped and does not open the full execution batch by default,
	*	the PDF suppresses unavailable fields instead of printing authoritative-looking blanks,
	*	the Verify route confirms the disclosed payout lane from a real Tempo batch,
	*	no separate cross-chain delivery fee field appears in Ledger & Compliance, the scoped receipt preview, the PDF, or the Verify route,
	*	and local or provenance sections appear only when those facts are actually available.

53.13 Admin vault smoke test

If ProtocolTreasury is deployed, open the hidden admin route and exercise one harmless multisig proposal path, such as a submit + confirm + revoke cycle on a non-executed test transaction.

Confirm:
	*	current owners and threshold load,
	*	pending proposals render,
	*	confirmation state changes reflect on chain,
	*	and the UI does not expose powers that do not exist in ProtocolTreasury itself.

---

54. Final issue-closure summary for V16.9.9

54.1 Closed issue classes

V16.9.9 closes these grounded issue classes from the V16.9.7, V16.9.4, V16.9.3, V16.9.2, V16.9.1, V16.9.0, V16.8, and V16.8.1 audits:
	1.	Fatal pool compile blocker
The invalid undeclared childKeys fragment is replaced with a real `bytes32[] memory childKeys = new bytes32[](10)` allocation.
	2.	Fatal treasury deployer compile blocker
The invalid owners declaration is replaced with a real fixed owner array allocation.
	3.	Unsafe tree-capacity guard
nextLeafIndex is upgraded to uint64 and checked against (uint64(1) << TREE_DEPTH).
	4.	Indexer ABI drift
BatchWithdrawal ABI now includes indexed assetId.
	5.	Indexer asset drift
Batch withdrawal rows now use Number(event.args.assetId) instead of hardcoded asset IDs.
	6.	Relayer replay-state timing bug
Nullifier is marked seen only after confirmed transaction receipt.
	7.	Missing circuit dependency in contracts workspace
circomlibjs is re-added to contracts package for Poseidon deploy tooling.
	8.	Bootstrap omission
packages/circuits/scripts is created by scaffold.
	9.	Workspace script footguns
Root scripts use --if-present.
	10.	USDT-style approval risk
Browser, relayer, and SDK approval helpers now implement zero-reset-safe approval flow.
	11.	Relay UX dishonesty
Waku publish now maps to SUBMITTED_TO_RELAYER, not the same success state as confirmed self-execution.
	12.	Relayer keygen omission
The relayer X25519 helper now writes the private key to a local secret file instead of dumping it to stdout.
	13.	Relayer profit-unit mismatch
The relayer now converts LZ fee-token units into normalized USD(6) and enforces a route-aware executor-fee floor instead of the older naive fee-minus-threshold compare.
	14.	Indexer API dev-topology gap
The indexer API now includes explicit CORS handling for the documented frontend-to-indexer topology.
	15.	Relay bundle pre-validation gap
Relay bundles are now structurally validated before relayerAddress or ciphertext fields are dereferenced.
	16.	Multi-bundle relay fault-coupling
One malformed bundle now drops in isolation instead of aborting later valid bundles in the same Waku payload.
	17.	Unsafe eager approval cleanup
Relayer and SDK now keep exact-amount approvals live after broadcast and only clear them when submission never happened or terminal failure is proven.
	18.	Spoke quote/send zero-fee mismatch
Spoke quoteShieldFee now rejects zero alt-fee quotes the same way shieldCrossChain does.
	19.	MCP metadata drift
The MCP server version string and startup banner now match the V16.9.9 runbook revision.
	20.	Relay transient-drop brittleness
Relayer now retries pre-submit transient failures with bounded backoff and persists exhausted attempts instead of only dropping them in memory.
	21.	Post-broadcast uncertainty handling gap
Relayer now persists uncertain submitted states to RELAYER_PENDING_EXECUTIONS_FILE instead of revoking approvals before terminal state is known.
	22.	Browser direct finality and cleanup gap
Browser direct deposits and self-executed spends now distinguish terminal failure from uncertain post-broadcast states, and only clean approvals when submission never happened or terminal failure is proven.
	23.	Zero-recipient payout gap
Pool, browser, SDK, and relayer now reject zero-address recipients for nonzero outputs instead of letting them reach revert-or-burn paths.
	24.	Campaign-worker divisor validation gap
Shield-mining analytics now fail fast on zero or invalid SHIELD_MINING_POINTS_DIVISOR values instead of dividing by zero at runtime.
	25.	Production CSP overexposure
The web middleware now limits unsafe-eval to development-only while preserving wasm-unsafe-eval for production proof tooling.
	26.	Witness-harness path shadowing
The test-witness helper now avoids shadowing the imported path module, so path.resolve remains callable at runtime.
	27.	SDK Node-only random byte conversion
The SDK note-crypto helper now converts random bytes to hex without relying on Buffer globals.
	28.	Recovery-hook unstable dependency smell
useRecoveredNotes now keys the effect off a deterministic cache-key fingerprint instead of a raw Uint8Array dependency.
	29.	Missing spoke-to-Tempo product flow
The frontend now ships a dedicated spoke shield page and widget instead of leaving spoke ingress as a contracts-only feature.
	30.	Source-chain browser approval-chain mismatch
Browser approval helpers can now read allowances and wait receipts on the selected spoke client instead of assuming Tempo for every approval flow.
	31.	Unbounded recovery history pulls
Browser and SDK history fetchers now walk indexer history in bounded pages instead of one monolithic request.
	32.	Deferred-ingress operator gap
The runbook now includes deferred-ingress retry and claim tooling plus hub-composer tests for that recovery path.
	33.	Relayer pending-state reconciliation gap
The relayer now ships both a one-off reconciler and a loop worker for pending relay executions.
	34.	Wallet-switch recovery-state leak
Changing the connected browser wallet now clears recovery authority, selected note state, and pending hashes.
	35.	Tempo network-config drift footgun
The runbook now uses Tempo mainnet chain ID 4217 by default and leaves Tempo LayerZero EID as an explicit operator-filled value instead of silently embedding an unverified number.
	36.	Ownership-handoff order contradiction
The canonical deployment flow now keeps deployer ownership through smoke and only transfers to protocol admin after the acceptance matrix succeeds.
	37.	Post-handover deferred-ingress recovery gap
Deferred-ingress recovery now uses an authorized caller key for single-signer actions and gives explicit multisig calldata commands for post-handover owner operations.
	38.	New route scaffold omission
The scaffold now creates the `apps/web/src/app/spoke-shield` route that the runbook ships later.
	39.	Reconciler multi-process and crash-safety gap
The pending reconciler now uses a lock file plus atomic pending-file rewrite, and archives resolved rows before removing them from pending state.
	40.	Spoke-deposit provenance ambiguity
Spoke-origin deposit provenance is now joined deterministically by LayerZero `guid` across source gateway dispatch rows and Tempo ingress settlement rows instead of relying on timestamp or amount heuristics.

54.2 Honest status after these fixes

After these corrections, V16.9.9 is:
	*	materially stronger than V16.9.7, V16.9.4, V16.9.3, V16.9.2, V16.9.1, V16.9.0, and V16.8,
	*	more compile-coherent,
	*	more operationally honest,
	*	closer to a real deployment candidate,
	*	substantially better aligned across pool, relayer, frontend, SDK, and indexer,
	*	and materially stronger on deterministic spoke-deposit provenance.

It still depends on real operator input and real execution:
	*	compile this exact generated repo,
	*	run the actual tests,
	*	generate the final verifier from the final ceremony output,
	*	fill the exact Tempo LayerZero EID and endpoint values from official deployment sources,
	*	wire the real OFTs / adapters / peers / libraries / DVNs,
	*	boot the real services,
	*	and pass the acceptance matrix on the actual target environment.

---

55. Final verdict

V16.9.9 is the corrected canonical launch runbook for the defined Parly launch surface.

It is stronger than V16.9.7, V16.9.4, V16.9.3, V16.9.2, V16.9.1, V16.9.0, and V16.8 because it combines:
	*	strict scope discipline,
	*	correct hub and spoke trust anchoring,
	*	asymmetric chain-based recovery,
	*	no local-storage dependence for correctness,
	*	canonical partial-spend behavior,
	*	coherent fee-path handling,
	*	hardened approval sequencing,
	*	per-bundle relay fault isolation,
	*	safer post-broadcast approval lifecycle handling,
	*	bounded transient retry with pending-state persistence,
	*	honest single-relayer private execution,
	*	corrected indexer event alignment,
	*	deterministic `guid`-based spoke-deposit provenance,
	*	explicit relayer key generation,
	*	corrected replay handling,
	*	and explicit operational role separation.

It is the strongest static production-candidate version in this lineage from the materials provided here.

The remaining zero-trust step is execution reality:
	*	compile this exact repo,
	*	generate the exact verifier,
	*	fill the exact Tempo LayerZero EID and endpoint values from official deployment sources,
	*	wire the exact OFTs and endpoints,
	*	boot the exact services,
	*	and pass the acceptance matrix end to end.

---

### Part 5 - Deterministic Spoke-Deposit Provenance Supplement

56. Deterministic spoke-deposit provenance in V16.9.9

56.1 Architectural reasoning

V16.9.9 now treats spoke-deposit provenance as a two-sided public record:

* the source gateway emits the dispatch event and source tx hash,
* the Tempo hub composer emits the ingress settlement anchor and settlement tx hash,
* and the indexer joins those two public records by the LayerZero message `guid`.

That is the canonical correlation model.

It deliberately avoids:

* timestamp matching,
* amount matching,
* wallet guessing from Tempo rows,
* or any claim that a source tx exists if there is no indexed source dispatch row.

56.2 Guaranteed provenance fields after this revision

For spoke-origin deposits created after this revision and indexed from the correct start blocks, V16.9.9 can now deterministically expose:

* Tempo settlement tx hash,
* final deposit commitment on Tempo,
* Tempo settlement chain and settlement EID,
* original source chain,
* original source EID,
* original source tx hash,
* source gateway address,
* source sender address,
* and the shared LayerZero `guid`.

56.3 Fields still unavailable unless later protocol work adds them

V16.9.9 still does **not** prove or expose:

* source-wallet ownership beyond the already-public source gateway sender field,
* any hidden metadata not emitted by contracts,
* retroactive source provenance for old rows that were never indexed from the needed source events,
* or provenance for direct Tempo deposits that never traversed a spoke gateway.

Direct Tempo deposits remain valid, but they are correctly shown as Tempo-settled only.

56.4 Migration note

This provenance expansion is forward-safe but not retroactively magical.

Operationally that means:

* existing historical direct Tempo deposits remain Tempo-only rows,
* historical spoke-origin deposits can only gain full provenance if the indexer is re-run from source gateway and hub composer start blocks that cover the original transactions,
* if either the source dispatch row or the Tempo ingress settlement row is missing, the frontend and SDK must degrade to Tempo-settled only instead of fabricating missing provenance,
* and future history / compliance / verify surfaces should treat provenance as optional enrichment, not a universal guarantee.

---

### Part 6 - Repository Distribution, Public Export, and Release Discipline

57. Production repository distribution model

57.1 Architectural reasoning

V16.9.9 keeps one private canonical monorepo but now formalizes three clean public distribution surfaces:

* a public relayer repo for operators,
* a public SDK repo/package for developers,
* and a public MCP repo for agent integrations.

This is a packaging and release-boundary change, not a protocol-law change.

The protocol core remains anchored to the same contracts, circuits, recovery model, relayer payload shape, and launch scope defined earlier in this runbook.

Support packages under `packages/*` are internal shared build/export packages inside the private canonical monorepo.

They are not separate promised public products unless Parly explicitly releases them later.

The only public distribution surfaces promised by V16.9.9 are:

* the public relayer repo,
* the public SDK repo/package,
* and the public MCP repo.

57.2 Founder Note

Public repos are distribution layers.

They are not alternate sources of truth.

All development, auditing, and release promotion still begin in the private canonical monorepo.

57.3 Dev Note

Do not make public repos depend on `apps/web` internals or private-only scripts.

If relayer, SDK, or MCP code needs shared logic, move that logic into `packages/*` first, then export from there.

57.4 File: `README.md`

```md
# Parly Canonical Monorepo

This repository is the private canonical source of truth for Parly V16.9.9.

Public GitHub distribution surfaces are generated from this monorepo:

- `dist/public-relayer`
- `dist/public-sdk`
- `dist/public-mcp`

## What lives here

- `apps/web` private product surface
- `apps/relayer` private source for the operator relayer distribution
- `apps/indexer` indexed data and API layer
- `apps/campaign-worker` analytics-only worker
- `apps/mcp-server` private source for the public MCP distribution
- `packages/*` internal shared build/export packages used to keep public exports clean and maintainable

## Export commands

```bash
pnpm repo:validate:public-exports
pnpm repo:export:relayer
pnpm repo:export:sdk
pnpm repo:export:mcp
```

## Release rule

Do not tell operators, SDK consumers, or MCP users to clone this canonical monorepo directly once the public distribution repos exist.

- Relayer operators should be directed to the public relayer repo.
- SDK consumers should be directed to the public SDK repo/package.
- MCP users should be directed to the public MCP repo.
```

57.5 File: `release/versions.json`

```json
{
  "coreRelease": "16.9.9",
  "registryApiVersion": "relayer-registry/v1",
  "publicSurfaces": {
    "relayerRepo": "16.9.9-operator.0",
    "sdkRepo": "16.9.9-sdk.0",
    "sdkPackage": "16.9.9",
    "mcpRepo": "16.9.9-mcp.0",
    "internalBuildPackages": {
      "envUtils": "16.9.9",
      "protocolAbis": "16.9.9",
      "sharedTypes": "16.9.9",
      "cryptoUtils": "16.9.9",
      "registryClient": "16.9.9"
    }
  },
  "compatibility": {
    "relayerRepo": {
      "coreRelease": "16.9.9",
      "registryApiVersion": "relayer-registry/v1"
    },
    "sdkPackage": {
      "coreRelease": "16.9.9",
      "proofModel": "joinsplit-10-output-v16.9.9"
    },
    "mcpRepo": {
      "sdkPackage": "16.9.9",
      "agentKeyModel": "AGENT_PRIVATE_KEY only"
    }
  }
}
```

57.6 File: `.github/workflows/public-exports.yml`

```yaml
name: public-exports

on:
  pull_request:
  push:
    branches: [main]

jobs:
  validate-and-export:
    runs-on: ubuntu-latest
    env:
      TEMPO_LZ_EID: ${{ vars.TEMPO_LZ_EID }}
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: 10.6.0
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck
      - run: pnpm build
      - run: pnpm repo:validate:public-exports
      - run: pnpm repo:export:all
      - run: pnpm repo:smoke:public-exports
```

58. Canonical export tooling

58.1 File: `tools/repo-export/shared.mjs`

```js
import fs from "node:fs"
import path from "node:path"
import { spawnSync } from "node:child_process"

export const REPO_ROOT = process.cwd()
export const DIST_ROOT = path.join(REPO_ROOT, "dist")

const COPY_IGNORE_NAMES = new Set([
  ".git",
  "node_modules",
  "dist",
  ".next",
  ".secrets",
  ".env",
  ".env.local"
])

const SCAN_IGNORE_NAMES = new Set([
  ".git",
  "node_modules",
  "dist",
  ".next",
  ".secrets"
])

const DEP_FIELDS = ["dependencies", "devDependencies", "peerDependencies", "optionalDependencies"]
const BANNED_PUBLIC_WORDING = [
  "Clone Canonical Repo",
  "clone the canonical repo",
  "clone the full canonical monorepo",
  "clone the full repository",
  "operators should clone the full"
]
const INTERNAL_SUPPORT_PACKAGE_NAMES = new Set([
  "@parly/env-utils",
  "@parly/protocol-abis",
  "@parly/shared-types",
  "@parly/crypto-utils",
  "@parly/registry-client"
])

export function ensureDir(dir) {
  fs.mkdirSync(dir, { recursive: true })
}

export function emptyDir(dir) {
  fs.rmSync(dir, { recursive: true, force: true })
  ensureDir(dir)
}

export function copyFileRelative(sourceRel, outRoot, destinationRel = sourceRel) {
  const source = path.join(REPO_ROOT, sourceRel)
  const destination = path.join(outRoot, destinationRel)
  ensureDir(path.dirname(destination))
  fs.copyFileSync(source, destination)
}

export function copyDirRelative(sourceRel, outRoot, destinationRel = sourceRel) {
  const source = path.join(REPO_ROOT, sourceRel)
  const destination = path.join(outRoot, destinationRel)
  ensureDir(destination)

  for (const entry of fs.readdirSync(source, { withFileTypes: true })) {
    if (COPY_IGNORE_NAMES.has(entry.name)) continue

    const nextSource = path.join(sourceRel, entry.name)
    const nextDestination = path.join(destinationRel, entry.name)

    if (entry.isDirectory()) {
      copyDirRelative(nextSource, outRoot, nextDestination)
    } else {
      copyFileRelative(nextSource, outRoot, nextDestination)
    }
  }
}

export function readText(relativePath) {
  return fs.readFileSync(path.join(REPO_ROOT, relativePath), "utf8")
}

export function readJsonAbsolute(absolutePath) {
  return JSON.parse(fs.readFileSync(absolutePath, "utf8"))
}

export function writeJsonAbsolute(absolutePath, value) {
  fs.writeFileSync(absolutePath, JSON.stringify(value, null, 2) + "\n", "utf8")
}

export function collectFiles(rootDir, predicate) {
  const out = []
  const stack = [rootDir]

  while (stack.length > 0) {
    const current = stack.pop()
    for (const entry of fs.readdirSync(current, { withFileTypes: true })) {
      if (SCAN_IGNORE_NAMES.has(entry.name)) continue
      const absolute = path.join(current, entry.name)
      if (entry.isDirectory()) {
        stack.push(absolute)
      } else if (predicate(absolute, entry.name)) {
        out.push(absolute)
      }
    }
  }

  return out
}

export function collectPackageJsonFiles(rootDir) {
  return collectFiles(rootDir, (_absolute, name) => name === "package.json")
}

export function assertNoSecretsInTree(rootDir) {
  const stack = [rootDir]

  while (stack.length > 0) {
    const current = stack.pop()
    for (const entry of fs.readdirSync(current, { withFileTypes: true })) {
      if (SCAN_IGNORE_NAMES.has(entry.name)) continue
      const absolute = path.join(current, entry.name)
      if (entry.isDirectory()) {
        stack.push(absolute)
        continue
      }
      if (entry.name === ".env" || entry.name.startsWith(".env.")) {
        throw new Error(`Forbidden env file in public export tree: ${absolute}`)
      }
      if (/\.pem$|\.key$|\.ptau$|\.zkey$/i.test(entry.name)) {
        throw new Error(`Forbidden secret-like artifact in public export tree: ${absolute}`)
      }
    }
  }
}

export function assertRequiredFiles(rootDir, relativePaths) {
  for (const relativePath of relativePaths) {
    const absolute = path.join(rootDir, relativePath)
    if (!fs.existsSync(absolute)) {
      throw new Error(`Missing required public-export file: ${absolute}`)
    }
  }
}

export function assertNoForbiddenImports(rootDir, forbiddenSnippets) {
  const stack = [rootDir]

  while (stack.length > 0) {
    const current = stack.pop()
    for (const entry of fs.readdirSync(current, { withFileTypes: true })) {
      if (SCAN_IGNORE_NAMES.has(entry.name)) continue
      const absolute = path.join(current, entry.name)
      if (entry.isDirectory()) {
        stack.push(absolute)
        continue
      }
      if (!/\.(ts|tsx|js|mjs|json|md)$/i.test(entry.name)) continue
      const content = fs.readFileSync(absolute, "utf8")
      for (const needle of forbiddenSnippets) {
        if (content.includes(needle)) {
          throw new Error(`Forbidden reference "${needle}" found in ${absolute}`)
        }
      }
    }
  }
}

export function assertNoBannedPublicWording(rootDir) {
  for (const absolute of collectFiles(rootDir, (_absolute, name) => /\.(md|txt|json)$/i.test(name))) {
    const content = fs.readFileSync(absolute, "utf8")
    for (const needle of BANNED_PUBLIC_WORDING) {
      if (content.includes(needle)) {
        throw new Error(`Banned public wording "${needle}" found in ${absolute}`)
      }
    }
  }
}

export function assertWorkspaceDepsResolvable(rootDir) {
  const packageJsonPaths = collectPackageJsonFiles(rootDir)
  const packageNames = new Set(
    packageJsonPaths
      .map((absolutePath) => readJsonAbsolute(absolutePath).name)
      .filter(Boolean)
  )

  for (const absolutePath of packageJsonPaths) {
    const manifest = readJsonAbsolute(absolutePath)
    for (const field of DEP_FIELDS) {
      const deps = manifest[field] || {}
      for (const [depName, depVersion] of Object.entries(deps)) {
        if (typeof depVersion === "string" && depVersion.startsWith("workspace:") && !packageNames.has(depName)) {
          throw new Error(`Unresolved workspace dependency ${depName} in ${absolutePath}`)
        }
      }
    }
  }
}

export function assertPackageManifestShape(rootDir) {
  const publishablePackages = []

  for (const absolutePath of collectPackageJsonFiles(rootDir)) {
    const manifest = readJsonAbsolute(absolutePath)
    const relativePath = path.relative(rootDir, absolutePath)

    if (!manifest.name) {
      throw new Error(`Package manifest missing name: ${absolutePath}`)
    }

    if (relativePath === "package.json") {
      if (!manifest.packageManager) {
        throw new Error(`Root public manifest missing packageManager: ${absolutePath}`)
      }
      if (!manifest.scripts || typeof manifest.scripts !== "object") {
        throw new Error(`Root public manifest missing scripts: ${absolutePath}`)
      }
      continue
    }

    if (!manifest.version) {
      throw new Error(`Workspace package missing version: ${absolutePath}`)
    }

    if (manifest.private !== true) {
      publishablePackages.push(manifest.name)
    }

    if (INTERNAL_SUPPORT_PACKAGE_NAMES.has(manifest.name) && manifest.private !== true) {
      throw new Error(`Internal support package must stay private in public exports: ${absolutePath}`)
    }

    if (relativePath === path.join("packages", "sdk", "package.json")) {
      for (const field of DEP_FIELDS) {
        const deps = manifest[field] || {}
        for (const [depName, depVersion] of Object.entries(deps)) {
          if (typeof depVersion === "string" && depVersion.startsWith("workspace:")) {
            throw new Error(`Public SDK package must not retain workspace dependencies in ${absolutePath}`)
          }
        }
      }
    }
  }

  const surfaceName = path.basename(rootDir)

  if (surfaceName === "public-sdk") {
    const unique = [...new Set(publishablePackages)].sort()
    if (unique.length !== 1 || unique[0] !== "@parly/sdk") {
      throw new Error(`Public SDK export must expose only @parly/sdk as a publishable package: ${rootDir}`)
    }

    for (const absolutePath of collectPackageJsonFiles(rootDir)) {
      const manifest = readJsonAbsolute(absolutePath)
      if (INTERNAL_SUPPORT_PACKAGE_NAMES.has(manifest.name)) {
        throw new Error(`Public SDK export must vendor internal support packages into @parly/sdk instead of exporting them as workspaces: ${absolutePath}`)
      }
    }
  }

  if (surfaceName === "public-relayer" || surfaceName === "public-mcp") {
    const unexpected = [...new Set(publishablePackages)]
    if (unexpected.length > 0) {
      throw new Error(`Public ${surfaceName.replace("public-", "")} export must not expose publishable workspace packages: ${unexpected.join(", ")}`)
    }
  }
}

export function runCommand(command, args, cwd, extraEnv = {}) {
  const result = spawnSync(command, args, {
    cwd,
    env: {
      ...process.env,
      CI: "true",
      ...extraEnv
    },
    stdio: "inherit",
    shell: process.platform === "win32"
  })

  if (result.status !== 0) {
    throw new Error(`Command failed in ${cwd}: ${command} ${args.join(" ")}`)
  }
}
```

58.2 File: `tools/repo-export/validate-public-exports.mjs`

```js
import fs from "node:fs"
import path from "node:path"
import {
  REPO_ROOT,
  DIST_ROOT,
  assertNoBannedPublicWording,
  assertNoForbiddenImports,
  assertNoSecretsInTree,
  assertPackageManifestShape,
  assertRequiredFiles,
  assertWorkspaceDepsResolvable
} from "./shared.mjs"

export const SURFACES = {
  relayer: {
    roots: [
      "apps/relayer",
      "packages/env-utils",
      "packages/protocol-abis",
      "packages/shared-types",
      "packages/crypto-utils",
      "packages/registry-client"
    ],
    templates: [
      "tools/repo-export/templates/relayer/README.md",
      "tools/repo-export/templates/relayer/.env.example",
      "tools/repo-export/templates/relayer/package.json",
      "tools/repo-export/templates/relayer/Dockerfile",
      "tools/repo-export/templates/relayer/docker-compose.yml",
      "tools/repo-export/templates/relayer/CHANGELOG.md",
      "tools/repo-export/templates/relayer/COMPATIBILITY.md"
    ],
    outputRequired: [
      "README.md",
      ".env.example",
      "package.json",
      "pnpm-workspace.yaml",
      "tsconfig.base.json",
      "CHANGELOG.md",
      "COMPATIBILITY.md",
      "Dockerfile",
      "docker-compose.yml",
      ".github/workflows/ci.yml",
      "scripts/healthcheck.mjs",
      "apps/relayer/package.json",
      "apps/relayer/src/bot.ts",
      "apps/relayer/src/smoke-config.ts",
      "packages/env-utils/package.json",
      "packages/registry-client/package.json"
    ],
    forbidden: ["apps/web/", "@parly/web", "campaign-worker", "vault-admin", "TGE", "referral"]
  },
  sdk: {
    roots: [
      "packages/sdk",
      "packages/env-utils",
      "packages/protocol-abis",
      "packages/shared-types",
      "packages/crypto-utils"
    ],
    templates: [
      "tools/repo-export/templates/sdk/README.md",
      "tools/repo-export/templates/sdk/package.json",
      "tools/repo-export/templates/sdk/CHANGELOG.md",
      "tools/repo-export/templates/sdk/COMPATIBILITY.md"
    ],
    outputRequired: [
      "README.md",
      "package.json",
      "pnpm-workspace.yaml",
      "tsconfig.base.json",
      "CHANGELOG.md",
      "COMPATIBILITY.md",
      ".github/workflows/ci.yml",
      "packages/sdk/package.json",
      "packages/sdk/README.md",
      "packages/sdk/examples/basic-payment.ts",
      "packages/sdk/src/internal/env-utils/index.ts",
      "packages/sdk/src/internal/protocol-abis/index.ts",
      "packages/sdk/src/internal/shared-types/index.ts",
      "packages/sdk/src/internal/crypto-utils/index.ts"
    ],
    forbidden: ["apps/web/", "apps/relayer/", "@parly/web", "campaign-worker", "vault-admin"]
  },
  mcp: {
    roots: [
      "apps/mcp-server",
      "packages/sdk",
      "packages/env-utils",
      "packages/protocol-abis",
      "packages/shared-types",
      "packages/crypto-utils"
    ],
    templates: [
      "tools/repo-export/templates/mcp/README.md",
      "tools/repo-export/templates/mcp/.env.example",
      "tools/repo-export/templates/mcp/package.json",
      "tools/repo-export/templates/mcp/CHANGELOG.md",
      "tools/repo-export/templates/mcp/COMPATIBILITY.md"
    ],
    outputRequired: [
      "README.md",
      ".env.example",
      "package.json",
      "pnpm-workspace.yaml",
      "tsconfig.json",
      "tsconfig.base.json",
      "CHANGELOG.md",
      "COMPATIBILITY.md",
      "Dockerfile",
      ".github/workflows/ci.yml",
      "apps/mcp-server/package.json",
      "apps/mcp-server/src/index.ts",
      "apps/mcp-server/src/smoke-config.ts",
      "packages/sdk/package.json",
      "packages/env-utils/package.json"
    ],
    forbidden: ["apps/web/", "apps/relayer/", "campaign-worker", "vault-admin"]
  }
}

export function validateSurface(name) {
  const spec = SURFACES[name]
  if (!spec) {
    throw new Error(`Unknown public surface: ${name}`)
  }

  for (const relativePath of [...spec.roots, ...spec.templates]) {
    const absolute = path.join(REPO_ROOT, relativePath)
    if (!fs.existsSync(absolute)) {
      throw new Error(`Missing required file or directory for ${name}: ${relativePath}`)
    }
  }

  for (const root of spec.roots) {
    assertNoForbiddenImports(path.join(REPO_ROOT, root), spec.forbidden)
  }
}

export function validateExportedSurface(name, rootDir = path.join(DIST_ROOT, `public-${name}`)) {
  const spec = SURFACES[name]
  if (!spec) {
    throw new Error(`Unknown public surface: ${name}`)
  }

  assertRequiredFiles(rootDir, spec.outputRequired)
  assertNoForbiddenImports(rootDir, spec.forbidden)
  assertNoSecretsInTree(rootDir)
  assertPackageManifestShape(rootDir)
  assertWorkspaceDepsResolvable(rootDir)
  assertNoBannedPublicWording(rootDir)
}

const requested = process.argv
  .find((arg) => arg.startsWith("--surface="))
  ?.split("=")[1]

if (requested) {
  validateSurface(requested)
  const outputRoot = path.join(DIST_ROOT, `public-${requested}`)
  if (fs.existsSync(outputRoot)) {
    validateExportedSurface(requested, outputRoot)
  }
  console.log(`Validated public export surface: ${requested}`)
} else {
  for (const name of Object.keys(SURFACES)) {
    validateSurface(name)
    const outputRoot = path.join(DIST_ROOT, `public-${name}`)
    if (fs.existsSync(outputRoot)) {
      validateExportedSurface(name, outputRoot)
    }
    console.log(`Validated public export surface: ${name}`)
  }
}
```

58.3 File: `tools/repo-export/export-relayer.mjs`

```js
import path from "node:path"
import { DIST_ROOT, emptyDir, copyDirRelative, copyFileRelative, assertNoSecretsInTree } from "./shared.mjs"
import { validateSurface, validateExportedSurface } from "./validate-public-exports.mjs"

const outRoot = path.join(DIST_ROOT, "public-relayer")

validateSurface("relayer")
emptyDir(outRoot)

copyDirRelative("apps/relayer", outRoot, "apps/relayer")
copyDirRelative("packages/env-utils", outRoot, "packages/env-utils")
copyDirRelative("packages/protocol-abis", outRoot, "packages/protocol-abis")
copyDirRelative("packages/shared-types", outRoot, "packages/shared-types")
copyDirRelative("packages/crypto-utils", outRoot, "packages/crypto-utils")
copyDirRelative("packages/registry-client", outRoot, "packages/registry-client")
copyDirRelative("tools/repo-export/templates/relayer", outRoot, ".")
copyFileRelative("release/versions.json", outRoot, "versions.json")

assertNoSecretsInTree(outRoot)
validateExportedSurface("relayer", outRoot)
console.log(`Exported public relayer repo to ${outRoot}`)
```

58.3a Expected exported tree: `dist/public-relayer`

```text
dist/public-relayer/
├─ README.md
├─ .env.example
├─ package.json
├─ pnpm-workspace.yaml
├─ tsconfig.base.json
├─ eslint.config.mjs
├─ Dockerfile
├─ docker-compose.yml
├─ CHANGELOG.md
├─ COMPATIBILITY.md
├─ versions.json
├─ ecosystem.config.cjs
├─ parly-relayer.service
├─ .github/workflows/ci.yml
├─ scripts/
│  ├─ healthcheck.mjs
│  ├─ start-relayer.sh
│  └─ start-reconciler.sh
├─ apps/
│  └─ relayer/
└─ packages/
   ├─ env-utils/
   ├─ protocol-abis/
   ├─ shared-types/
   ├─ crypto-utils/
   └─ registry-client/
```

58.4 File: `tools/repo-export/export-sdk.mjs`

```js
import fs from "node:fs"
import path from "node:path"
import {
  DIST_ROOT,
  emptyDir,
  copyDirRelative,
  copyFileRelative,
  assertNoSecretsInTree,
  collectFiles,
  readJsonAbsolute,
  writeJsonAbsolute
} from "./shared.mjs"
import { validateSurface, validateExportedSurface } from "./validate-public-exports.mjs"

const outRoot = path.join(DIST_ROOT, "public-sdk")
const vendorMap = [
  ["@parly/env-utils", "packages/env-utils/src", "packages/sdk/src/internal/env-utils"],
  ["@parly/protocol-abis", "packages/protocol-abis/src", "packages/sdk/src/internal/protocol-abis"],
  ["@parly/shared-types", "packages/shared-types/src", "packages/sdk/src/internal/shared-types"],
  ["@parly/crypto-utils", "packages/crypto-utils/src", "packages/sdk/src/internal/crypto-utils"]
]

validateSurface("sdk")
emptyDir(outRoot)

copyDirRelative("packages/sdk", outRoot, "packages/sdk")
copyDirRelative("tools/repo-export/templates/sdk", outRoot, ".")
copyFileRelative("release/versions.json", outRoot, "versions.json")

for (const [, sourceRel, destinationRel] of vendorMap) {
  copyDirRelative(sourceRel, outRoot, destinationRel)
}

for (const absolutePath of collectFiles(path.join(outRoot, "packages/sdk/src"), (absolute) => /\.(ts|tsx|mts|cts)$/i.test(absolute))) {
  const fromDir = path.dirname(absolutePath)
  let content = fs.readFileSync(absolutePath, "utf8")

  for (const [specifier, , destinationRel] of vendorMap) {
    const target = path.join(outRoot, destinationRel, "index.ts")
    let replacement = path.relative(fromDir, target).replace(/\\/g, "/")
    if (!replacement.startsWith(".")) replacement = `./${replacement}`
    replacement = replacement.replace(/\.ts$/, ".js")
    content = content.replaceAll(`"${specifier}"`, `"${replacement}"`)
    content = content.replaceAll(`'${specifier}'`, `'${replacement}'`)
  }

  fs.writeFileSync(absolutePath, content, "utf8")
}

const sdkPackagePath = path.join(outRoot, "packages/sdk/package.json")
const sdkManifest = readJsonAbsolute(sdkPackagePath)
sdkManifest.private = false
sdkManifest.publishConfig = { access: "public" }
for (const field of ["dependencies", "devDependencies", "peerDependencies", "optionalDependencies"]) {
  const deps = sdkManifest[field] || {}
  for (const [specifier] of vendorMap) {
    delete deps[specifier]
  }
  sdkManifest[field] = deps
}
writeJsonAbsolute(sdkPackagePath, sdkManifest)

assertNoSecretsInTree(outRoot)
validateExportedSurface("sdk", outRoot)
console.log(`Exported public SDK repo to ${outRoot}`)
```

58.4a Expected exported tree: `dist/public-sdk`

```text
dist/public-sdk/
├─ README.md
├─ package.json
├─ pnpm-workspace.yaml
├─ tsconfig.base.json
├─ eslint.config.mjs
├─ CHANGELOG.md
├─ COMPATIBILITY.md
├─ versions.json
├─ .github/workflows/ci.yml
└─ packages/
   ├─ sdk/
   │  ├─ README.md
   │  ├─ CHANGELOG.md
   │  ├─ COMPATIBILITY.md
   │  ├─ examples/
   │  └─ src/
   │     └─ internal/
   │        ├─ env-utils/
   │        ├─ protocol-abis/
   │        ├─ shared-types/
   │        └─ crypto-utils/
```

58.5 File: `tools/repo-export/export-mcp.mjs`

```js
import path from "node:path"
import { DIST_ROOT, emptyDir, copyDirRelative, copyFileRelative, assertNoSecretsInTree } from "./shared.mjs"
import { validateSurface, validateExportedSurface } from "./validate-public-exports.mjs"

const outRoot = path.join(DIST_ROOT, "public-mcp")

validateSurface("mcp")
emptyDir(outRoot)

copyDirRelative("apps/mcp-server", outRoot, "apps/mcp-server")
copyDirRelative("packages/sdk", outRoot, "packages/sdk")
copyDirRelative("packages/env-utils", outRoot, "packages/env-utils")
copyDirRelative("packages/protocol-abis", outRoot, "packages/protocol-abis")
copyDirRelative("packages/shared-types", outRoot, "packages/shared-types")
copyDirRelative("packages/crypto-utils", outRoot, "packages/crypto-utils")
copyDirRelative("tools/repo-export/templates/mcp", outRoot, ".")
copyFileRelative("release/versions.json", outRoot, "versions.json")

assertNoSecretsInTree(outRoot)
validateExportedSurface("mcp", outRoot)
console.log(`Exported public MCP repo to ${outRoot}`)
```

58.6 File: `tools/repo-export/smoke-public-exports.mjs`

```js
import path from "node:path"
import { DIST_ROOT, runCommand } from "./shared.mjs"
import { validateExportedSurface } from "./validate-public-exports.mjs"

const DUMMY_PRIVATE_KEY = "0x1111111111111111111111111111111111111111111111111111111111111111"

function requireConfiguredTempoLzEid() {
  const raw = String(process.env.TEMPO_LZ_EID ?? process.env.NEXT_PUBLIC_TEMPO_LZ_EID ?? "").trim()

  if (!/^[1-9][0-9]*$/.test(raw) || raw.includes("REPLACE_WITH_REAL")) {
    throw new Error("TEMPO_LZ_EID must be supplied to repo:smoke:public-exports as the real Tempo LayerZero deployment EID")
  }

  return raw
}

const TEMPO_LZ_EID = requireConfiguredTempoLzEid()

const SURFACES = {
  relayer: {
    root: path.join(DIST_ROOT, "public-relayer"),
    env: {
      RELAYER_PRIVATE_KEY: DUMMY_PRIVATE_KEY,
      RELAYER_BOX_PRIVATE_KEY_B64: "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=",
      RELAYER_KEY_OUTPUT_FILE: ".tmp/relayer-box.env",
      RELAYER_REGISTRY_API_BASE_URL: "https://registry.example.parly.fi",
      TEMPO_RPC_URL: "https://rpc.example.tempo.xyz",
      TEMPO_CHAIN_ID: "4217",
      TEMPO_LZ_EID,
      WAKU_CLUSTER_ID: "1",
      WAKU_BOOTSTRAP: "false",
      WAKU_CONTENT_TOPIC: "/parly/16/9/9/app/proto"
    },
    commands: [
      ["pnpm", ["install", "--no-frozen-lockfile"]],
      ["pnpm", ["typecheck"]],
      ["pnpm", ["build"]],
      ["pnpm", ["smoke:env"]],
      ["pnpm", ["smoke:keygen"]]
    ]
  },
  sdk: {
    root: path.join(DIST_ROOT, "public-sdk"),
    commands: [
      ["pnpm", ["install", "--no-frozen-lockfile"]],
      ["pnpm", ["typecheck"]],
      ["pnpm", ["build"]],
      ["pnpm", ["test"]],
      ["pnpm", ["publish:dry-run"]]
    ]
  },
  mcp: {
    root: path.join(DIST_ROOT, "public-mcp"),
    env: {
      AGENT_PRIVATE_KEY: DUMMY_PRIVATE_KEY,
      TEMPO_RPC_URL: "https://rpc.example.tempo.xyz",
      TEMPO_CHAIN_ID: "4217",
      ENABLE_MPP_ADAPTER: "true",
      MPP_SERVICE_NAME: "parly-mpp-adapter",
      MPP_SERVICE_VERSION: "16.9.9"
    },
    commands: [
      ["pnpm", ["install", "--no-frozen-lockfile"]],
      ["pnpm", ["typecheck"]],
      ["pnpm", ["build"]],
      ["pnpm", ["smoke:config"]]
    ]
  }
}

for (const [name, spec] of Object.entries(SURFACES)) {
  validateExportedSurface(name, spec.root)
  for (const [command, args] of spec.commands) {
    runCommand(command, args, spec.root, spec.env || {})
  }
  console.log(`Standalone smoke validation passed: ${name}`)
}
```

58.5a Expected exported tree: `dist/public-mcp`

```text
dist/public-mcp/
├─ README.md
├─ .env.example
├─ package.json
├─ pnpm-workspace.yaml
├─ tsconfig.json
├─ tsconfig.base.json
├─ eslint.config.mjs
├─ Dockerfile
├─ CHANGELOG.md
├─ COMPATIBILITY.md
├─ versions.json
├─ .github/workflows/ci.yml
├─ apps/
│  └─ mcp-server/
└─ packages/
   ├─ sdk/
   ├─ env-utils/
   ├─ protocol-abis/
   ├─ shared-types/
   └─ crypto-utils/
```

59. Public relayer repo template

59.1 File: `tools/repo-export/templates/relayer/package.json`

```json
{
  "name": "parly-relayer-repo",
  "private": true,
  "packageManager": "pnpm@10.6.0",
  "scripts": {
    "lint": "eslint apps packages scripts --ext .ts,.js,.mjs",
    "build": "pnpm -r --if-present run build",
    "typecheck": "pnpm -r --if-present run typecheck",
    "start": "pnpm --filter @parly/relayer run start",
    "keygen:box": "pnpm --filter @parly/relayer run keygen:box",
    "reconcile:pending": "pnpm --filter @parly/relayer run reconcile:pending",
    "reconcile:pending:loop": "pnpm --filter @parly/relayer run reconcile:pending:loop",
    "smoke:env": "pnpm --filter @parly/relayer run smoke:env",
    "smoke:keygen": "pnpm --filter @parly/relayer run smoke:keygen",
    "smoke:standalone": "pnpm lint && pnpm typecheck && pnpm build && pnpm smoke:env && pnpm smoke:keygen"
  },
  "engines": {
    "node": ">=22"
  },
  "devDependencies": {
    "@eslint/js": "^9.22.0",
    "eslint": "^9.22.0",
    "typescript-eslint": "^8.26.1"
  },
  "workspaces": [
    "apps/*",
    "packages/*"
  ]
}
```

59.2 File: `tools/repo-export/templates/relayer/pnpm-workspace.yaml`

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

59.3 File: `tools/repo-export/templates/relayer/tsconfig.base.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "strict": true,
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "baseUrl": ".",
    "paths": {
      "@parly/env-utils": ["packages/env-utils/src/index.ts"],
      "@parly/protocol-abis": ["packages/protocol-abis/src/index.ts"],
      "@parly/shared-types": ["packages/shared-types/src/index.ts"],
      "@parly/crypto-utils": ["packages/crypto-utils/src/index.ts"],
      "@parly/registry-client": ["packages/registry-client/src/index.ts"]
    }
  }
}
```

59.3a File: `tools/repo-export/templates/relayer/eslint.config.mjs`

```js
import js from "@eslint/js"
import tseslint from "typescript-eslint"

export default tseslint.config(
  js.configs.recommended,
  ...tseslint.configs.recommended,
  {
    files: ["**/*.ts", "**/*.tsx", "**/*.js", "**/*.mjs"],
    ignores: ["dist/**", "node_modules/**"]
  }
)
```

59.4 File: `tools/repo-export/templates/relayer/README.md`

```md
# Parly Relayer

Production relayer runtime for Parly private execution.

## What this repo is

This repo is the public operator-facing Parly relayer distribution. It contains only the relayer runtime, reconciler, registry client support, validation helpers, and the minimal shared packages needed to run those services cleanly.

## Who this repo is for

This repo is for third-party relayer operators.

You do not need the full canonical protocol monorepo to run a relayer.

## Prerequisites

- Node 22+
- pnpm 10+
- a funded Tempo execution wallet
- Postgres only if you also run your own registry-compatible backend services

## Install

```bash
cp .env.example .env
pnpm install
pnpm keygen:box
pnpm start
pnpm reconcile:pending:loop
```

## Repository layout

```text
apps/relayer
packages/env-utils
packages/protocol-abis
packages/shared-types
packages/crypto-utils
packages/registry-client
scripts/healthcheck.mjs
```

## Environment

Key variables:

- `RELAYER_PRIVATE_KEY`
- `RELAYER_BOX_PRIVATE_KEY_B64`
- `TEMPO_RPC_URL`
- `TEMPO_CHAIN_ID`
- `TEMPO_LZ_EID`
- `WAKU_CONTENT_TOPIC`

Optional official-registry variable:

- `RELAYER_REGISTRY_API_BASE_URL`

Fail fast on missing config. Do not rely on soft defaults for production.

`TEMPO_LZ_EID` must be the real Tempo LayerZero deployment EID for the environment and must never be inferred from the chain ID or replaced with a local fake default.

## Key generation flow

Run `pnpm keygen:box` before first startup. The command generates the sealed-box keypair used for relay payload encryption and writes the operator-facing output path configured by `RELAYER_KEY_OUTPUT_FILE`.

## Runtime flow

1. Generate the sealed-box keypair with `pnpm keygen:box`
2. Start the relayer daemon with `pnpm start`
3. Start the pending reconciler with `pnpm reconcile:pending:loop`
4. If you want official frontend discovery, set `RELAYER_REGISTRY_API_BASE_URL`
5. Open the Parly frontend `/relayer` page
6. Register the execution address and public key
7. Keep the relayer running so successful execution heartbeats keep the registry entry active while the row remains verified

## Pending reconciler flow

The reconciler watches `RELAYER_PENDING_EXECUTIONS_FILE`, re-checks uncertain submissions, archives resolved records, and should run continuously in production beside the main relayer daemon.

## Registry registration flow

Official registry listing is optional.

After the relayer is healthy, operators who want official frontend discovery should open the public Parly `/relayer` registration page, submit the execution address plus Base64 public key, sign the registration message, and wait for immediate verification if a live slot is available.

Operators who are pruned to `inactive` must re-register through `/relayer` to regain a verified slot. Heartbeats update only already-verified rows.

## Heartbeat behavior

Successful relays emit a signed heartbeat to the registry backend after confirmed inclusion only when `RELAYER_REGISTRY_API_BASE_URL` is configured.

Heartbeat success is strict.

The registry only returns success when the signed execution address already matches an active verified relayer row and that row is actually updated. A missing, inactive, rejected, or otherwise unmatched row fails the heartbeat request.

## Health and logging

- structured JSON logs are recommended in production
- use `scripts/healthcheck.mjs` for a lightweight startup/config probe
- expose relayer and reconciler processes separately in your process manager

## Deployment snippets

Use the included deployment helpers depending on your environment:

- `docker-compose.yml` for simple containerized deployment
- `ecosystem.config.cjs` for pm2-managed processes
- `parly-relayer.service` as a systemd starting point

## Standalone verification

Run `pnpm smoke:standalone` after cloning to verify install, typecheck, build, environment validation, and key generation in one pass.

## Common failure modes

- invalid relayer box key
- Tempo RPC misconfiguration
- incomplete registry configuration
- stale approval state requiring reconciler cleanup
- Waku peer availability issues

## Upgrade notes

Check [COMPATIBILITY.md](./COMPATIBILITY.md) before upgrading.

## Version compatibility matrix

See [versions.json](./versions.json) and [COMPATIBILITY.md](./COMPATIBILITY.md).

## Security notes

- never commit `.env`
- never publish relayer private keys
- keep AGENT and RELAYER keys separate
- registration in the official registry does not create protocol exclusivity

## Reporting issues

Report relayer runtime issues through the Parly operator support path documented by your deployment owner.
```

59.5 File: `tools/repo-export/templates/relayer/.env.example`

```dotenv
RELAYER_PRIVATE_KEY=0x0000000000000000000000000000000000000000000000000000000000000000
RELAYER_BOX_PRIVATE_KEY_B64=
RELAYER_KEY_OUTPUT_FILE=.secrets/relayer-box.env
# Optional: set only if you want official registry listing and heartbeat updates
RELAYER_REGISTRY_API_BASE_URL=https://app.parly.fi
RELAYER_PENDING_EXECUTIONS_FILE=.secrets/relayer-pending-executions.ndjson
RELAYER_PENDING_EXECUTIONS_ARCHIVE_FILE=.secrets/relayer-pending-executions.resolved.ndjson
RELAYER_PENDING_RECONCILE_LOCK_FILE=.secrets/relayer-pending-executions.lock
RELAYER_PENDING_RECONCILE_INTERVAL_MS=30000
TEMPO_RPC_URL=https://rpc.example.tempo.xyz
TEMPO_CHAIN_ID=4217
TEMPO_LZ_EID=REPLACE_WITH_REAL_TEMPO_LZ_EID
WAKU_CLUSTER_ID=1
WAKU_BOOTSTRAP=true
WAKU_CONTENT_TOPIC=/parly/16/9/9/app/proto
```

59.6 File: `tools/repo-export/templates/relayer/Dockerfile`

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY . .
RUN corepack enable && pnpm install --no-frozen-lockfile
RUN pnpm build
CMD ["pnpm", "start"]
```

59.7 File: `tools/repo-export/templates/relayer/docker-compose.yml`

```yaml
services:
  relayer:
    build: .
    env_file: .env
    command: pnpm start
    restart: unless-stopped
  reconciler:
    build: .
    env_file: .env
    command: pnpm reconcile:pending:loop
    restart: unless-stopped
```

59.8 File: `tools/repo-export/templates/relayer/CHANGELOG.md`

```md
# Changelog

## 16.9.9-operator.0

- initial public relayer distribution extracted from the canonical V16.9.9 monorepo
- includes runtime registry heartbeat support
- includes pending reconciler and key generation utility
```

59.9 File: `tools/repo-export/templates/relayer/COMPATIBILITY.md`

```md
# Compatibility

- Relayer repo `16.9.9-operator.0` is compatible with canonical core `16.9.9`
- Registry API compatibility: `relayer-registry/v1`
- Waku payload shape compatibility: canonical V16.9.9 encrypted bundle schema

Upgrade only when the registry API and canonical relay payload version remain compatible.
```

59.10 File: `tools/repo-export/templates/relayer/scripts/healthcheck.mjs`

```js
import process from "node:process"

const required = [
  "RELAYER_PRIVATE_KEY",
  "RELAYER_BOX_PRIVATE_KEY_B64",
  "TEMPO_RPC_URL",
  "TEMPO_CHAIN_ID",
  "TEMPO_LZ_EID",
  "WAKU_CLUSTER_ID",
  "WAKU_BOOTSTRAP",
  "WAKU_CONTENT_TOPIC"
]

for (const key of required) {
  if (!process.env[key]) {
    throw new Error(`${key} missing`)
  }
}

if (!/^[1-9][0-9]*$/.test(String(process.env.TEMPO_CHAIN_ID))) {
  throw new Error("TEMPO_CHAIN_ID must be a positive integer chain ID")
}

if (!/^[1-9][0-9]*$/.test(String(process.env.TEMPO_LZ_EID)) || String(process.env.TEMPO_LZ_EID).includes("REPLACE_WITH_REAL")) {
  throw new Error("TEMPO_LZ_EID must be the real Tempo LayerZero deployment EID")
}

if (!/^[1-9][0-9]*$/.test(String(process.env.WAKU_CLUSTER_ID))) {
  throw new Error("WAKU_CLUSTER_ID must be a positive integer")
}

if (process.env.WAKU_BOOTSTRAP !== "true" && process.env.WAKU_BOOTSTRAP !== "false") {
  throw new Error("WAKU_BOOTSTRAP must be either true or false")
}

if (process.env.RELAYER_REGISTRY_API_BASE_URL) {
  new URL(String(process.env.RELAYER_REGISTRY_API_BASE_URL))
}

console.log("Relayer healthcheck configuration OK")
```

59.11 File: `tools/repo-export/templates/relayer/scripts/start-relayer.sh`

```bash
#!/bin/sh
set -eu
pnpm start
```

59.12 File: `tools/repo-export/templates/relayer/scripts/start-reconciler.sh`

```bash
#!/bin/sh
set -eu
pnpm reconcile:pending:loop
```

59.13 File: `tools/repo-export/templates/relayer/ecosystem.config.cjs`

```js
module.exports = {
  apps: [
    {
      name: "parly-relayer",
      script: "pnpm",
      args: "start"
    },
    {
      name: "parly-relayer-reconciler",
      script: "pnpm",
      args: "reconcile:pending:loop"
    }
  ]
}
```

59.14 File: `tools/repo-export/templates/relayer/parly-relayer.service`

```ini
[Unit]
Description=Parly Relayer
After=network.target

[Service]
WorkingDirectory=/opt/parly-relayer
ExecStart=/usr/bin/pnpm start
Restart=always
EnvironmentFile=/opt/parly-relayer/.env

[Install]
WantedBy=multi-user.target
```

59.15 File: `tools/repo-export/templates/relayer/.github/workflows/ci.yml`

```yaml
name: relayer-ci

on:
  push:
  pull_request:

jobs:
  relayer:
    runs-on: ubuntu-latest
    env:
      RELAYER_PRIVATE_KEY: 0x1111111111111111111111111111111111111111111111111111111111111111
      RELAYER_BOX_PRIVATE_KEY_B64: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=
      RELAYER_KEY_OUTPUT_FILE: .tmp/relayer-box.env
      RELAYER_REGISTRY_API_BASE_URL: https://registry.example.parly.fi
      TEMPO_RPC_URL: https://rpc.example.tempo.xyz
      TEMPO_CHAIN_ID: "4217"
      TEMPO_LZ_EID: ${{ vars.TEMPO_LZ_EID }}
      WAKU_CLUSTER_ID: "1"
      WAKU_BOOTSTRAP: "false"
      WAKU_CONTENT_TOPIC: /parly/16/9/9/app/proto
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: 10.6.0
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --no-frozen-lockfile
      - run: pnpm smoke:standalone
      - run: node ./scripts/healthcheck.mjs
```

60. Public SDK repo template

60.1 File: `tools/repo-export/templates/sdk/package.json`

```json
{
  "name": "parly-sdk-repo",
  "private": true,
  "packageManager": "pnpm@10.6.0",
  "scripts": {
    "lint": "eslint packages/sdk --ext .ts,.js,.mjs",
    "build": "pnpm -r --if-present run build",
    "typecheck": "pnpm -r --if-present run typecheck",
    "test": "pnpm --filter @parly/sdk run test:lz-options",
    "pack:dry-run": "pnpm --filter @parly/sdk run pack:dry-run",
    "publish:dry-run": "pnpm --filter @parly/sdk publish --dry-run --no-git-checks",
    "smoke:standalone": "pnpm lint && pnpm typecheck && pnpm build && pnpm test && pnpm publish:dry-run"
  },
  "engines": {
    "node": ">=22"
  },
  "devDependencies": {
    "@eslint/js": "^9.22.0",
    "eslint": "^9.22.0",
    "typescript-eslint": "^8.26.1"
  },
  "workspaces": [
    "packages/sdk"
  ]
}
```

60.2 File: `tools/repo-export/templates/sdk/pnpm-workspace.yaml`

```yaml
packages:
  - "packages/sdk"
```

60.3 File: `tools/repo-export/templates/sdk/tsconfig.base.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "strict": true,
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "baseUrl": ".",
    "paths": {
      "@parly/sdk": ["packages/sdk/src/index.ts"]
    }
  }
}
```

60.3a File: `tools/repo-export/templates/sdk/eslint.config.mjs`

```js
import js from "@eslint/js"
import tseslint from "typescript-eslint"

export default tseslint.config(
  js.configs.recommended,
  ...tseslint.configs.recommended,
  {
    files: ["**/*.ts", "**/*.tsx", "**/*.js", "**/*.mjs"],
    ignores: ["dist/**", "node_modules/**"]
  }
)
```

60.4 File: `tools/repo-export/templates/sdk/README.md`

```md
# Parly SDK

Production SDK for Parly private execution flows.

## What this repo is

This repo contains the public Parly SDK package surface together with the vendored internal helper modules needed to build and validate that package from source.

The primary published developer package is `@parly/sdk`.

The exported SDK repo does not promise separate public support packages. Internal helper code is vendored into the SDK source during export so the public `@parly/sdk` package remains self-contained.

## Who this repo is for

This repo is for developers integrating Parly payment execution into apps, services, or agent workflows without needing the full internal monorepo.

You do not need the private canonical protocol monorepo to use this public SDK repo or the published `@parly/sdk` package.

## Use from npm

Install the main package directly:

```bash
npm install @parly/sdk
```

## Work from this public repo

Use the public repo itself when you need the full examples or release-validation flow:

```bash
pnpm install
pnpm smoke:standalone
```

## Initialization

Create the SDK with an AGENT-owned execution key, Tempo RPC URL, and Tempo chain ID.

## Repository layout

```text
packages/sdk
packages/sdk/src/internal
```

## Example flows

- basic shielded payment
- agent/app integration
- error handling
- MPP-compatible adapter at the SDK boundary only

See the `packages/sdk/examples` directory for runnable example scaffolds included in the public export.

## Environment requirements

- funded Tempo execution wallet
- valid Tempo RPC
- compatible proof assets

## Network assumptions

Parly V16.9.9 remains Tempo-settled and stablecoin-only.

## Security notes

- keep AGENT keys separate from RELAYER keys
- do not run the SDK in a browser with server-only secrets
- MPP compatibility does not create contract-level session state

## Limitations

- no scheduling
- no swaps
- no passkeys in privacy mode

## Release and compatibility

See [COMPATIBILITY.md](./COMPATIBILITY.md) and [versions.json](./versions.json).

## Standalone verification

Run `pnpm smoke:standalone` to verify install, typecheck, build, tests, and publish dry-run behavior for the exported SDK workspace.
```

60.5 File: `tools/repo-export/templates/sdk/CHANGELOG.md`

```md
# Changelog

## 16.9.9-sdk.0

- initial public SDK repo extracted from canonical V16.9.9
- includes MPP-compatible adapter surface at the SDK boundary
- includes publish-ready package metadata for `@parly/sdk`
```

60.6 File: `tools/repo-export/templates/sdk/COMPATIBILITY.md`

```md
# Compatibility

- `@parly/sdk@16.9.9` is compatible with canonical core `16.9.9`
- compatible with Parly MCP `16.9.9-mcp.0`
- proof model: joinsplit 10-output padded batch
```

60.7 File: `tools/repo-export/templates/sdk/.github/workflows/ci.yml`

```yaml
name: sdk-ci

on:
  push:
  pull_request:

jobs:
  sdk:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: 10.6.0
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --no-frozen-lockfile
      - run: pnpm smoke:standalone
```

61. Public MCP repo template

61.1 File: `tools/repo-export/templates/mcp/package.json`

```json
{
  "name": "parly-mcp-repo",
  "private": true,
  "packageManager": "pnpm@10.6.0",
  "scripts": {
    "lint": "eslint apps packages --ext .ts,.js,.mjs",
    "build": "pnpm -r --if-present run build",
    "typecheck": "pnpm -r --if-present run typecheck",
    "start": "pnpm --filter @parly/mcp-server run start",
    "smoke:config": "pnpm --filter @parly/mcp-server run smoke:config",
    "smoke:standalone": "pnpm lint && pnpm typecheck && pnpm build && pnpm smoke:config"
  },
  "engines": {
    "node": ">=22"
  },
  "devDependencies": {
    "@eslint/js": "^9.22.0",
    "eslint": "^9.22.0",
    "typescript-eslint": "^8.26.1"
  },
  "workspaces": [
    "apps/*",
    "packages/*"
  ]
}
```

61.2 File: `tools/repo-export/templates/mcp/tsconfig.json`

```json
{
  "extends": "./tsconfig.base.json",
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "types": ["node"]
  },
  "include": ["apps/mcp-server/src/**/*.ts", "packages/**/*.ts"]
}
```

61.2a File: `tools/repo-export/templates/mcp/pnpm-workspace.yaml`

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

61.2b File: `tools/repo-export/templates/mcp/tsconfig.base.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "strict": true,
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "baseUrl": ".",
    "paths": {
      "@parly/env-utils": ["packages/env-utils/src/index.ts"],
      "@parly/protocol-abis": ["packages/protocol-abis/src/index.ts"],
      "@parly/shared-types": ["packages/shared-types/src/index.ts"],
      "@parly/crypto-utils": ["packages/crypto-utils/src/index.ts"],
      "@parly/sdk": ["packages/sdk/src/index.ts"]
    }
  }
}
```

61.2c File: `tools/repo-export/templates/mcp/eslint.config.mjs`

```js
import js from "@eslint/js"
import tseslint from "typescript-eslint"

export default tseslint.config(
  js.configs.recommended,
  ...tseslint.configs.recommended,
  {
    files: ["**/*.ts", "**/*.tsx", "**/*.js", "**/*.mjs"],
    ignores: ["dist/**", "node_modules/**"]
  }
)
```

61.3 File: `tools/repo-export/templates/mcp/README.md`

```md
# Parly MCP Server

Production MCP server for Parly agent integrations.

## What this repo is

This repo is the public MCP distribution for Parly. It packages the MCP server with only the minimal shared packages needed to expose the supported Parly tool surface to agent stacks.

## Who this repo is for

This repo is for teams integrating Parly into agent or AI workflows.

## Install

```bash
cp .env.example .env
pnpm install
pnpm start
```

## Repository layout

```text
apps/mcp-server
packages/sdk
packages/env-utils
packages/protocol-abis
packages/shared-types
packages/crypto-utils
```

## Configuration

- `AGENT_PRIVATE_KEY`
- `TEMPO_RPC_URL`
- `TEMPO_CHAIN_ID`
- optional MPP adapter flags

Fail fast on missing required configuration. Do not configure relayer keys in this repo.

## Run locally

```bash
cp .env.example .env
pnpm install
pnpm start
```

## Integrate with an agent stack

Point your MCP host at the Parly MCP server entrypoint, provide AGENT credentials plus Tempo connectivity, and expose only the tool surface your deployment actually needs.

## Supported tools

- `recover_largest_note`
- `execute_shielded_payment`
- `mpp_create_session`
- `mpp_settle_session_payment`

## Known limitations

- AGENT-key execution only
- no relayer runtime inside this repo
- no contract-core MPP session state
- no scheduling or swap extensions

## Security notes

- MCP uses AGENT key only
- do not configure RELAYER keys here
- MPP compatibility stays at the SDK/MCP boundary only

## Compatibility

See [COMPATIBILITY.md](./COMPATIBILITY.md) and [versions.json](./versions.json).

## Standalone verification

Run `pnpm smoke:standalone` to verify install, typecheck, build, and minimal startup/config validation for the exported MCP workspace.
```

61.4 File: `tools/repo-export/templates/mcp/.env.example`

```dotenv
AGENT_PRIVATE_KEY=0x0000000000000000000000000000000000000000000000000000000000000000
TEMPO_RPC_URL=https://rpc.example.tempo.xyz
TEMPO_CHAIN_ID=4217
ENABLE_MPP_ADAPTER=true
MPP_SERVICE_NAME=parly-mpp-adapter
MPP_SERVICE_VERSION=16.9.9
```

61.5 File: `tools/repo-export/templates/mcp/Dockerfile`

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY . .
RUN corepack enable && pnpm install --no-frozen-lockfile
CMD ["pnpm", "start"]
```

61.6 File: `tools/repo-export/templates/mcp/CHANGELOG.md`

```md
# Changelog

## 16.9.9-mcp.0

- initial public MCP repo extracted from canonical V16.9.9
- depends on the public `@parly/sdk` package surface
- keeps AGENT-only execution semantics
```

61.7 File: `tools/repo-export/templates/mcp/COMPATIBILITY.md`

```md
# Compatibility

- MCP repo `16.9.9-mcp.0` expects `@parly/sdk@16.9.9`
- compatible with canonical core `16.9.9`
- AGENT key model only
```

61.8 File: `tools/repo-export/templates/mcp/.github/workflows/ci.yml`

```yaml
name: mcp-ci

on:
  push:
  pull_request:

jobs:
  mcp:
    runs-on: ubuntu-latest
    env:
      AGENT_PRIVATE_KEY: 0x1111111111111111111111111111111111111111111111111111111111111111
      TEMPO_RPC_URL: https://rpc.example.tempo.xyz
      TEMPO_CHAIN_ID: "4217"
      ENABLE_MPP_ADAPTER: "true"
      MPP_SERVICE_NAME: parly-mpp-adapter
      MPP_SERVICE_VERSION: 16.9.9
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: 10.6.0
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --no-frozen-lockfile
      - run: pnpm smoke:standalone
```

62. Repo-internal wording rule

62.1 Architectural reasoning

Once public distribution repos exist, repo-internal wording must stop telling role-specific users to clone the full canonical monorepo.

That means:

* relayer operators are directed to the public relayer repo,
* SDK consumers are directed to the public SDK repo/package,
* MCP consumers are directed to the public MCP repo,
* and the canonical monorepo README stays maintainer-facing.

62.2 Dev Note

If any generated public text still says "clone the canonical repo", treat that as export-template drift and fail the public export validation pass until it is corrected.



















