# Parly public docs

This repository is the Mintlify public docs surface for Parly.

## What lives here

- public docs pages in root `*.mdx`
- Mintlify site configuration in `docs.json`
- source material in `references/`

## Source of truth

Keep public docs aligned with the live product, public repositories, and current Terms of Service. Do not publish internal runbook language, launch-only labels, or unsupported features as user-facing docs.

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
