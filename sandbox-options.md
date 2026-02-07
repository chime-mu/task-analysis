# Sandbox Options for Claude Code with `--dangerously-skip-permissions`

## 1. Anthropic Devcontainer (Recommended for Docker)

Anthropic provides a production-ready devcontainer specifically for running Claude Code with `--dangerously-skip-permissions`.

- **Repo**: https://github.com/anthropics/claude-code/tree/main/.devcontainer
- Includes a firewall script (`init-firewall.sh`) that restricts outbound network to whitelisted domains only (npm registry, GitHub, Claude API, etc.)
- Default-deny network policy for all unwhitelisted domains
- Works with VS Code's "Reopen in Container" workflow
- Built on Node.js 20 with production-ready dependencies

The official guidance:

> Use `--dangerously-skip-permissions` in a **container without internet access**.

**Setup**:
1. Install VS Code and Remote - Containers extension
2. Clone the reference repository
3. Open in VS Code
4. Click "Reopen in Container" when prompted

**Best for**: Production use, client work, CI/CD pipelines.

## 2. Built-in OS-Level Sandboxing

Claude Code has built-in sandboxing that doesn't require Docker:

- **macOS**: Uses Apple's Seatbelt (`sandbox-exec`)
- **Linux**: Uses Bubblewrap (`bwrap`)
- Toggle with the `/sandbox` command
- Restricts what commands can do at the OS level

Note: Docker is incompatible with Claude Code's built-in sandboxing. If you need Docker inside the sandbox, add `docker` to `excludedCommands` to force it to run outside the sandbox.

**Best for**: Local development with fine-grained control, no Docker needed.

## 3. Claude Code on the Web (Managed VMs)

Each session runs in an isolated, Anthropic-managed virtual machine:

- **Isolated VM per session**: Spun up on demand, terminated after completion
- **Credential protection**: Git credentials and signing keys go through a secure proxy — they never enter the sandbox VM itself
- **Network controls**: Default limited access, all traffic routed through an HTTP/HTTPS proxy
- **Pre-installed toolchains**: Node.js, Python, Ruby, Go, Rust, Java, C++, PostgreSQL 16, Redis 7.0
- **Automatic cleanup**: VM is destroyed after the session ends
- **Branch restrictions**: Git push limited to the current working branch only
- **Audit logging**: All operations logged for compliance

The key security property: even if prompt injection tricks Claude into running malicious commands, the blast radius is contained to a disposable VM with no real credentials.

**Best for**: Background tasks, parallel work, zero local setup.

## 4. Open Source Sandbox Runtime

Anthropic publishes an open-source sandbox runtime that can sandbox any program:

- **Repo**: https://github.com/anthropic-experimental/sandbox-runtime
- **NPM**: `@anthropic-ai/sandbox-runtime`
- **Usage**: `npx @anthropic-ai/sandbox-runtime <command-to-sandbox>`

**Best for**: Custom requirements, sandboxing beyond Claude Code.

## Comparison

| Approach | Security | Setup | Internet Access | Best For |
|---|---|---|---|---|
| Anthropic Devcontainer | Very High | Moderate | Firewall-restricted | Production, CI/CD, client work |
| Built-in sandbox (Seatbelt/bwrap) | High | Easy (`/sandbox`) | Configurable | Local dev without Docker |
| Claude Code on Web (VM) | Very High | None | Configurable | Background tasks, no local setup |
| Open Source Sandbox Runtime | High | Low | Your choice | Custom requirements |

## Key Recommendations

1. **Always sandbox** when using `--dangerously-skip-permissions`
2. **Disable internet access** unless absolutely necessary
3. **Prefer the official devcontainer** for production and client work — it's maintained by Anthropic and includes firewall rules
4. **Use built-in sandboxing** (`/sandbox`) for local development when Docker isn't needed
5. **For the "proper VM" experience** like Claude Code on the web, use the open-source sandbox runtime or your own VM/container setup with network restrictions
