# Parly public docs

This repository is the Mintlify public docs surface for Parly.

## What lives here

- public docs pages in root `*.mdx`
- Mintlify site configuration in `docs.json`
- source material in `references/`

## Source of truth

Before editing public docs, read the reference files in `references/`. The runbook and core PRD govern launch-law accuracy. The hosted docs, UI/UX blueprint, relayer registry PRD, and Phase 2 document govern public framing, registry behavior, and removable analytics boundaries.

## Public source surfaces

- relayer operators: `https://github.com/Parlyfi/parly-relayer`
- developers: `https://github.com/Parlyfi/parly-sdk`
- agent builders: `https://github.com/Parlyfi/parly-mcp`

## Local preview

Install the Mintlify CLI if you need local preview:

```bash
npm i -g mint
```

Then run:

```bash
mint dev
```

For link validation, run:

```bash
mint broken-links
```
