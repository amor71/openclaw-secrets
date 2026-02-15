# OpenClaw External Secrets Management

A contribution to [OpenClaw](https://github.com/openclaw/openclaw) adding native GCP Secret Manager integration — so credentials are stored encrypted in a centralized secrets store, not in plaintext files on disk.

**Related issue:** [openclaw/openclaw#13610](https://github.com/openclaw/openclaw/issues/13610)

## Documents

| Document | Description |
|---|---|
| [REQUIREMENTS](./REQUIREMENTS-gcp-secrets.md) | What the feature must do (the asks) |
| [DESIGN](./DESIGN-gcp-secrets.md) | How it will be implemented (architecture, interfaces, data flow) |

## Status

- [x] Requirements — approved ✅
- [x] Design — approved ✅
- [x] Tests — 96 passing (74 unit + 22 CLI) ✅
- [x] Implementation — complete ✅
- [x] Documentation — `docs/concepts/secrets.md` ✅
- [ ] Local testing with real GCP Secret Manager — in progress
- [ ] PR to openclaw/openclaw — not started

## Why

OpenClaw currently stores all API keys, tokens, and secrets in plaintext files. This is a problem because:

- Anyone with shell access can read all secrets
- Multi-agent setups share filesystem access to all credential files
- No audit trail of secret access
- Config files can't be safely committed to git
- No standard way for agents to securely pass credentials to sub-agents

## What

- **GCP Secret Manager** as the first external secrets provider
- **`${gcp:secret-name}`** reference syntax in config files
- **Bootstrapping** — automated setup of GCP Secret Manager, APIs, and IAM
- **Migration** — automatically move existing plaintext secrets to the store and purge originals
- **Per-agent isolation** — each agent only accesses secrets it's authorized for, enforced via IAM
- **CLI commands** — `openclaw secrets setup|migrate|test|list|set`

## Authors

- **Amichay Oren** ([@amor71](https://github.com/amor71)) — Requirements & review
- **Rye** 🥃 — Design & implementation (AI agent)

## License

MIT — see [LICENSE](./LICENSE)
