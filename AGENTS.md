# Docs repo instructions

This repository is the Mintlify public docs surface for Parly.

## Source of truth inside this repo
Use these files in /references as source material before editing public docs:
- references/Parly Public Hosted Docs 29th march.md
- references/Parly Core PRD & TRD.md
- references/Parly.fi Privacy Protocol UIUX Master doc 29th march.md
- references/Parly.fi Phase 2 Master Document 28 March 2026.md
- references/Parly Relayer Registry PRDDesign docs 28th march 2026.md
- references/Runbook V16.9.9 Final.md

## Output target
Create and update only the public Mintlify docs pages in the repo root plus `docs.json` unless explicitly asked otherwise.

## Product law
Do not reintroduce killed launch features such as:
- scheduling
- delayed execution
- intent vaults
- REST intent APIs
- swap routing
- 1inch
- volatile-token payout paths
- passkey privacy mode
- dual identity planes
- on-chain MPP session state
- protocol-owned relay monopoly

## Public repo links
Use these as the public source surfaces:
- https://github.com/Parlyfi/parly-relayer
- https://github.com/Parlyfi/parly-sdk
- https://github.com/Parlyfi/parly-mcp

## SDK install
Use:
- npm install @parly/sdk
- pnpm add @parly/sdk
