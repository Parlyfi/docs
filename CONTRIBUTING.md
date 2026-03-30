# Contributing to Parly docs

Thank you for helping improve the Parly public docs.

## Before you edit

Read [AGENTS.md](./AGENTS.md) first. It is the canonical repo guidance file for this repository.

Use the files in `references/` as the source of truth for product law, public framing, registry behavior, and removable analytics boundaries.

## Contribution scope

Use this repo for:

- public docs page updates
- navigation updates in `docs.json`
- wording and structure cleanup for the Mintlify public docs surface

Do not use this repo to document killed launch features or unpublished internal surfaces unless explicitly asked.

## Local development

1. Install the Mintlify CLI: `npm i -g mint`
2. Run `mint dev` from the repo root
3. Preview the docs locally
4. Run `mint broken-links` before pushing when the CLI is available

## Writing rules

- use active voice and second person
- keep sentences concise
- distinguish between user, operator, developer, and agent-builder journeys
- keep `submitted`, `pending`, `confirmed`, and `failed` distinct where execution state matters
- do not imply that the relayer registry permissions protocol participation
- do not tell public users to clone the private canonical monorepo
