# Requirements: External Secrets Management for OpenClaw

**Issue:** [openclaw/openclaw#13610](https://github.com/openclaw/openclaw/issues/13610)
**Author:** Rye (AI) + Amichay Oren
**Date:** 2026-02-15
**Status:** Draft

---

## 1. Problem Statement

OpenClaw stores all credentials (API keys, tokens, secrets) in plaintext files on disk:
- `openclaw.json` config file
- `auth-profiles.json` per agent
- User-created files (e.g., `~/.config/*/credentials.env`)

This creates several risks:
1. **Exposure** — Anyone with shell access can read all secrets
2. **No audit trail** — No visibility into when secrets were accessed
3. **Rotation friction** — Changing a secret requires editing files and restarting
4. **Version control conflict** — Config files can't be safely committed to git
5. **Multi-agent leakage** — Agents can read each other's credential files
6. **Compliance** — Enterprise deployments require centralized secrets management

## 2. Scope

### In Scope (this PR)
- GCP Secret Manager as the first external secrets provider
- Secret reference syntax in `openclaw.json` config
- Runtime resolution of secret references
- Caching with configurable TTL
- Graceful fallback and clear error messages
- Documentation

### Out of Scope (future work)
- AWS Secrets Manager, Azure Key Vault, HashiCorp Vault (follow same pattern)
- Automatic secret rotation
- UI for managing secrets
- Migration tool for existing plaintext secrets (nice-to-have, could be a follow-up)

## 3. Requirements

### 3.1 Secret Reference Syntax

Secrets in config files are referenced using a URI-like syntax:

```
${provider:secret-name}
${provider:secret-name#version}
```

Example in `openclaw.json`:
```json
{
  "tools": {
    "web": {
      "search": {
        "apiKey": "${gcp:openclaw-brave-api-key}"
      }
    }
  },
  "channels": {
    "slack": {
      "botToken": "${gcp:openclaw-slack-bot-token}"
    }
  }
}
```

### 3.2 Provider Configuration

A new top-level `secrets` section in `openclaw.json`:

```json
{
  "secrets": {
    "providers": {
      "gcp": {
        "project": "my-gcp-project",
        "cacheTtlSeconds": 300
      }
    },
    "defaults": {
      "provider": "gcp",
      "cacheTtlSeconds": 300
    }
  }
}
```

### 3.3 Resolution Behavior

1. **Parse time** — When config is loaded, scan all string values for `${...}` patterns
2. **Lazy resolution** — Fetch secrets only when the config value is first accessed (not at startup for unused secrets)
3. **Caching** — Cache resolved values in memory with TTL (default: 5 minutes)
4. **Cache invalidation** — On config reload / SIGUSR1 restart, clear the cache
5. **Fallback** — If a secret reference cannot be resolved:
   - Log a clear error identifying which secret failed and why
   - Do NOT fall back to treating the reference string as a literal value
   - The feature/channel that depends on the secret should fail gracefully

### 3.4 GCP Secret Manager Specifics

- Use `@google-cloud/secret-manager` npm package
- Authenticate via Application Default Credentials (ADC) — supports:
  - Service account key file (`GOOGLE_APPLICATION_CREDENTIALS`)
  - Compute Engine / GKE metadata server
  - `gcloud auth application-default login`
- Secret name format: `projects/{project}/secrets/{name}/versions/{version}`
  - Default version: `latest`
  - Users specify just the short name; provider constructs the full resource path
- Support accessing secrets across projects (if IAM allows):
  ```
  ${gcp:projects/other-project/secrets/my-secret}
  ```

### 3.5 Auth Profile Integration

`auth-profiles.json` should also support secret references:

```json
{
  "profiles": {
    "openai:default": {
      "type": "token",
      "provider": "openai",
      "token": "${gcp:openclaw-openai-key}"
    }
  }
}
```

### 3.6 Security Requirements

- Secret values MUST NOT be logged (not even at debug level)
- Secret values MUST NOT appear in error messages
- The `config.get` API (and CLI) MUST redact resolved secret values (show `${gcp:name}` or `[REDACTED]`)
- Cache is in-memory only — never written to disk
- Secret references in config should be clearly distinguishable from literal values

### 3.7 Error Handling

| Scenario | Behavior |
|---|---|
| Provider not configured | Error at resolution time: "Secrets provider 'gcp' not configured" |
| Secret not found | Error: "Secret 'name' not found in GCP project 'project'" |
| Permission denied | Error: "Permission denied accessing secret 'name'. Check IAM roles." |
| Network error | Error with retry (1 retry, 5s timeout). Then fail with clear message. |
| Invalid reference syntax | Error at config parse time: "Invalid secret reference: '...'" |

### 3.8 CLI Support

- `openclaw secrets list` — List configured providers and their status
- `openclaw secrets test` — Test connectivity to all configured providers
- `openclaw secrets set --provider gcp --name <name> --value <value>` — Store a secret (convenience command)

## 4. Non-Functional Requirements

- **Performance** — Secret resolution should add <100ms to startup (lazy loading helps)
- **Reliability** — If GCP is unreachable, cached values should be used if available (stale-while-revalidate)
- **Compatibility** — Config files without secret references should work exactly as before (zero breaking changes)
- **Testing** — Unit tests with mocked GCP client; integration test guide in docs

## 5. User Stories

1. **As an OpenClaw admin**, I want to store API keys in GCP Secret Manager so they're not in plaintext on my server
2. **As a multi-agent operator**, I want each agent's credentials isolated via IAM so Agent A can't access Agent B's secrets
3. **As a developer**, I want to commit my `openclaw.json` to git without leaking secrets
4. **As an enterprise user**, I want an audit trail of when my API keys were accessed

## 6. Migration Path

For existing users:
1. Install: Set up GCP Secret Manager, store secrets
2. Configure: Add `secrets.providers.gcp` to config
3. Replace: Change plaintext values to `${gcp:name}` references
4. Verify: `openclaw secrets test`
5. Clean up: Remove old plaintext files, rotate compromised keys

## 7. Open Questions

1. Should we support secret references in **environment variables** passed to exec tools? (e.g., `env: { "API_KEY": "${gcp:my-key}" }`)
2. Should there be a `secrets.required` flag that prevents startup if any secret is unresolvable?
3. Should the caching layer be shared across agents or per-agent?

---

## Appendix: Architecture Sketch

```
openclaw.json (with ${gcp:...} references)
    │
    ▼
Config Parser ──► Secret Reference Scanner
    │                    │
    │                    ▼
    │              Secret Resolver
    │                    │
    │         ┌──────────┼──────────┐
    │         ▼          ▼          ▼
    │      GCP SM    AWS SM     Vault
    │      Provider  Provider   Provider
    │         │
    │         ▼
    │    GCP Secret Manager API
    │         │
    │         ▼
    │    In-Memory Cache (TTL)
    │         │
    ▼         ▼
Resolved Config (secrets in memory only)
```
