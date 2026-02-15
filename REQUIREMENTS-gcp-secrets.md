# Requirements: External Secrets Management for OpenClaw

**Issue:** [openclaw/openclaw#13610](https://github.com/openclaw/openclaw/issues/13610)
**Author:** Rye (AI) + Amichay Oren
**Date:** 2026-02-15
**Status:** Draft

---

## 1. Problem Statement

OpenClaw stores all credentials (API keys, tokens, secrets) in plaintext files on disk. This creates the following problems:

1. **Exposure** — Anyone with shell access can read all secrets
2. **No isolation between agents** — In a multi-agent setup, all agents share filesystem access to all credential files. There is no way to restrict Agent A from reading Agent B's secrets.
3. **No audit trail** — No visibility into when secrets were accessed or by whom
4. **Rotation friction** — Changing a secret requires manual file edits and service restarts
5. **Version control conflict** — Config files containing secrets can't be safely committed to git
6. **Compliance** — Enterprise deployments require centralized secrets management

## 2. Goals

1. Secrets must be stored in a centralized, encrypted, access-controlled secrets store — not in plaintext files
2. Agents must be able to retrieve secrets they are authorized to access, at runtime
3. Each agent's access to secrets must be independently controllable (agent-level isolation)
4. The system must be able to set itself up from scratch (create the secrets store, enable required APIs, configure access controls) if nothing exists yet
5. Existing plaintext secrets must be automatically migrated to the secrets store and purged from disk
6. The solution must not break existing OpenClaw installations that don't use a secrets store

## 3. Scope

### In Scope
- GCP Secret Manager as the first secrets provider
- Bootstrapping: automated setup of GCP Secret Manager (enable APIs, create resources, configure IAM) when it doesn't exist
- Secret references in OpenClaw config files, resolved at runtime
- Per-agent secret isolation via access controls
- Migration tool: automatically move existing plaintext secrets to the store and purge originals
- CLI commands for managing secrets
- Documentation

### Out of Scope (future work)
- Other providers (AWS Secrets Manager, Azure Key Vault, HashiCorp Vault) — should follow the same pattern established here
- Automatic secret rotation
- UI for managing secrets

## 4. Functional Requirements

### 4.1 Secret Storage & Retrieval

- Secrets must be stored in GCP Secret Manager, not on the local filesystem
- Agents must be able to reference secrets in configuration files without knowing the actual values
- Secrets must be fetched at runtime when needed
- Retrieved secrets must be cached in memory to avoid repeated network calls
- Cached secrets must never be written to disk
- Secret values must never appear in logs, error messages, or API responses

### 4.2 Per-Agent Isolation

- In a multi-agent setup, each agent must only be able to access secrets it is authorized for
- It must be possible to grant Agent A access to secret X without granting Agent B the same access
- The access control mechanism must use the secrets store's native access control (IAM), not application-level enforcement

### 4.3 Bootstrapping

When a user first enables the secrets provider:

- If the GCP Secret Manager API is not enabled, enable it
- If required IAM roles/service accounts don't exist, create them
- If per-agent service accounts are needed for isolation, create and configure them
- The bootstrapping must be idempotent (safe to run multiple times)
- The user must be informed of all changes being made and asked to confirm

### 4.4 Migration

For existing installations with plaintext secrets:

- Scan all known locations for plaintext secrets (config files, auth profiles, credential files)
- Upload each secret to the secrets store
- Replace plaintext values in config files with secret references
- Verify that all references resolve correctly
- Purge plaintext originals from disk
- Handle partial failures gracefully — never purge a secret that wasn't successfully stored
- Must be interactive (confirm before destructive actions) with an option to skip confirmation for automation

### 4.5 CLI

Users must be able to:

- Check the status of the secrets provider (is it set up? is it reachable?)
- Test that all secret references in the current config resolve successfully
- Manually store a new secret
- Run the migration from plaintext to secrets store
- Run the bootstrap setup

### 4.6 Error Handling

- If a secret cannot be retrieved, the error must clearly identify which secret failed and why (not found, permission denied, network error, provider not configured)
- A missing secret must not cause the entire system to crash — only the feature that depends on it should fail
- If the secrets provider is unreachable, previously cached values should be usable as a fallback

### 4.7 Backward Compatibility

- Existing OpenClaw installations that don't configure a secrets provider must continue to work exactly as they do today
- The secrets feature must be entirely opt-in
- No new required dependencies

## 5. User Stories

1. **As an OpenClaw admin**, I want to store API keys in an encrypted secrets store so they're not exposed in plaintext on my server
2. **As a multi-agent operator**, I want each agent to only access its own secrets, so a compromised or misbehaving agent can't read another agent's credentials
3. **As a new user**, I want the system to set up the secrets infrastructure for me, so I don't have to manually configure GCP Secret Manager, IAM roles, and service accounts
4. **As an existing user**, I want to migrate my current plaintext secrets to the store automatically, and have the old files cleaned up
5. **As a developer**, I want to commit my OpenClaw config to git without leaking secrets

## 6. Open Questions

1. Should secrets be passable to exec tool environments? (e.g., a script that needs an API key)
2. Should there be an option to prevent startup entirely if any required secret is unresolvable?
3. How should the system handle secret versioning? (always latest, or allow pinning?)

---

*This document describes WHAT the system must do. The HOW (architecture, interfaces, data flow, technology choices) will be covered in the Design document.*
