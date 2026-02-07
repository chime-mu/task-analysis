# Enforced Sandbox Setup for Claude Code in Corporate Environments

## 1. Managed Settings (Enforce Policy Org-Wide)

Claude Code supports an IT-managed settings file that individual users **cannot override**:

- **macOS**: `/Library/Application Support/ClaudeCode/managed-settings.json`
- **Linux**: `/etc/claude-code/managed-settings.json`

Your IT team can deploy this via MDM (Jamf, Intune, etc.) to enforce sandbox behavior, restrict tools, and control permissions across the org.

## 2. Devcontainer + Secrets Proxy (The "Web VM" Pattern Locally)

This mirrors how Claude Code on the Web handles credentials — secrets never enter the sandbox:

```
┌─────────────────────────┐
│   Host Machine          │
│                         │
│  ┌── Secrets Proxy ──┐  │
│  │  Vault / 1Password│  │
│  │  Git credentials  │  │
│  │  API keys         │  │
│  └────────▲──────────┘  │
│           │              │
│  ┌────────┼──────────┐  │
│  │  Devcontainer     │  │
│  │  (sandboxed)      │  │
│  │                   │  │
│  │  Claude Code      │  │
│  │  --dangerously-   │  │
│  │  skip-permissions │  │
│  │                   │  │
│  │  No real creds    │  │
│  │  Firewall-locked  │  │
│  └───────────────────┘  │
└─────────────────────────┘
```

**How it works:**

- Start from Anthropic's official devcontainer (`https://github.com/anthropics/claude-code/tree/main/.devcontainer`)
- Git credentials flow through a **credential helper** on the host — the container calls out to the host's `git-credential-manager`, never storing tokens itself
- API keys come via a **secrets proxy** (HashiCorp Vault, AWS Secrets Manager, 1Password CLI) that issues short-lived tokens
- The firewall in `init-firewall.sh` is extended to only allow outbound traffic to your proxy + approved domains

## 3. Practical Corporate Starter

A realistic setup for your org:

1. **Standardized devcontainer** in your corporate repo template (fork Anthropic's)
2. **Extend the firewall** to whitelist only your internal domains, artifact registries, and the Claude API
3. **Mount a credential socket** (like Docker's `--mount type=bind` for SSH agent or git credential helper) so the container can authenticate without holding secrets
4. **Managed settings** pushed via MDM to enforce that devs can't disable the sandbox
5. **Audit logging** — the devcontainer can ship logs to your SIEM

## Recommended Rollout Order

1. **Deploy managed settings** via MDM to enforce baseline policies now (low effort, immediate coverage)
2. **Build a corporate devcontainer image** based on Anthropic's reference, adding your internal CA certs, registry mirrors, and secrets proxy integration
3. **Document the secrets proxy pattern** so developers know how to authenticate from inside the sandbox without copying credentials in
