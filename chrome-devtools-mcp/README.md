# chrome-devtools-mcp

A **provision-only mixin** that makes [`chrome-devtools-mcp`](https://github.com/ChromeDevTools/chrome-devtools-mcp) work inside an sbx sandbox.

## What it does

- Installs a cross-architecture headless Chromium (via Playwright — works on Apple Silicon arm64 where Google Chrome has no Linux build)
- Installs `libnss3-tools` and imports the sandbox proxy CA into Chrome's NSS certificate store so HTTPS works without bypassing certificate verification
- Configures Chrome to route through the sandbox network proxy, so browsing is subject to the same egress policy as the rest of the sandbox
- Drops a `chrome-devtools-mcp` wrapper on `$PATH` that bakes in all container-appropriate flags

The kit **does not** automatically register the MCP server with your agent — that is a one-line step you do after composing the kit (see [Registering](#registering-the-mcp-server)).

## Usage

```bash
sbx run claude --kit ./chrome-devtools-mcp/ /path/to/project
```

Or with any other agent:

```bash
sbx run codex --kit ./chrome-devtools-mcp/ .
sbx run gemini --kit ./chrome-devtools-mcp/ .
```

## Registering the MCP server

Once inside the sandbox, register `chrome-devtools-mcp` with your agent. Do this once per sandbox (or once per re-create if you use `--scope local`).

### Claude

```bash
claude mcp add chrome-devtools --scope local -- chrome-devtools-mcp
```

Verify: `claude mcp list` should show both `mcp-gateway` (the sbx gateway) and `chrome-devtools`.

### Codex

Append to `~/.codex/config.toml`:

```toml
[mcp_servers.chrome-devtools]
type = "stdio"
command = "chrome-devtools-mcp"
args = []
```

### Gemini

```bash
jq -s '.[0] * .[1]' ~/.gemini/settings.json \
  '{"mcpServers":{"chrome-devtools":{"command":"chrome-devtools-mcp","args":[]}}}' \
  > /tmp/gs.json && mv /tmp/gs.json ~/.gemini/settings.json
```

### opencode

Merge into `~/.config/opencode/config.json`:

```json
{
  "mcp": {
    "chrome-devtools": {
      "type": "local",
      "command": "chrome-devtools-mcp",
      "args": []
    }
  }
}
```

## Credentials and login

When Chrome visits a site requiring login, session cookies and tokens are stored in Chrome's profile. The wrapper uses `--isolated`, so this is a temporary directory that chrome-devtools-mcp cleans up on exit — the session is gone when the MCP server stops.

**The agent has full Chrome DevTools Protocol access**, which means it can read cookies, execute JavaScript in the page context, and intercept network requests. A prompt-injected agent could use these capabilities to exfiltrate session credentials — but only to domains permitted by the sandbox network policy. That firewall is the key protection.

### Why the host-side alternative is worse

Running chrome-devtools-mcp on the host (via `sbx mcp add`) rather than inside the sandbox has the same profile behaviour when `--isolated` is used — sessions are still ephemeral. The meaningful difference is the **egress firewall**: host-side Chrome can reach anything your host can, regardless of the sandbox network policy. A prompt-injected agent exfiltrating session credentials has no policy constraint on where it sends them.

### Sites that require login

For sites where you need to authenticate, prefer attach mode (`CDMCP_BROWSER_URL`): start Chrome separately, log in manually outside the agent session, then point the MCP server at it. The agent still has CDP access to the live session, but it did not drive the authentication flow.

```bash
google-chrome --headless --no-sandbox \
  --remote-debugging-port=9222 --remote-debugging-address=127.0.0.1 &

export CDMCP_BROWSER_URL=http://127.0.0.1:9222
claude mcp add chrome-devtools --scope local -- chrome-devtools-mcp
```

## Network allowlist

The browser's egress goes through the **sandbox network proxy** and is subject to the sandbox's network policy. Domains your agent navigates to must be explicitly allowed.

```bash
# On the host, allow a domain for this sandbox:
sbx policy allow network example.com

# Or allow all (disables egress filtering):
sbx policy allow network "**"
```

The kit itself needs these domains at install time (already declared in `spec.yaml`):

| Domain | Purpose |
|---|---|
| `archive.ubuntu.com`, `security.ubuntu.com`, `ports.ubuntu.com` | apt (Chrome system libs) |
| `registry.npmjs.org` | npm / npx (Playwright, chrome-devtools-mcp) |
| `cdn.playwright.dev`, `playwright.download.prss.microsoft.com` | Chromium binary download |

## Configuration

These environment variables control the wrapper and can be overridden in the sandbox:

| Variable | Default | Description |
|---|---|---|
| `CDMCP_CHROME_PATH` | *(resolved at install)* | Path to the Chromium binary |
| `CDMCP_PROXY` | `http://gateway.docker.internal:3128` | Proxy to route Chrome through |
| `CDMCP_BROWSER_URL` | *(empty)* | Set to attach to an existing Chrome (`http://127.0.0.1:9222`) instead of launching one |

## Attaching to an existing Chrome (advanced)

If you start Chrome separately — for example to preserve state across MCP server restarts — set `CDMCP_BROWSER_URL` before registration:

```bash
# Start Chrome with remote debugging inside the sandbox:
google-chrome --headless --no-sandbox --remote-debugging-port=9222 --remote-debugging-address=127.0.0.1 &

# Then register with attach mode:
export CDMCP_BROWSER_URL=http://127.0.0.1:9222
claude mcp add chrome-devtools --scope local -- chrome-devtools-mcp
```

> **Note:** A long-lived Chrome background process can become a zombie if it crashes, because PID 1 in the sandbox is `sleep` and does not reap children. Prefer launch mode (the default) for most use cases.

## Why `--no-sandbox`

Chromium's internal renderer sandbox uses Linux user namespaces (`clone(CLONE_NEWUSER)`), which require `CAP_SYS_ADMIN` or kernel-level unprivileged user namespace support. Neither is available in a standard Docker container.

`--no-sandbox` is the standard practice for headless Chrome in CI and container environments (used by Puppeteer, Playwright, and virtually every CI system that runs Chrome). The **sbx container itself** is the security boundary: Chrome's egress is firewalled, a renderer compromise is contained to the disposable sandbox, and the container is thrown away after the session.

The alternative — `security.privileged: true` in the kit spec — grants near-root-on-host access, which weakens the outer boundary far more than `--no-sandbox` weakens the inner one.

## How it works internally

1. **`commands.install`** (at sandbox create, root):
   - Ensures Node.js ≥ 20.19
   - Installs `libnss3-tools` (provides `certutil`)
   - Runs `npx playwright install --with-deps chromium` to download Chromium and its system libs, and persists `CDMCP_CHROME_PATH` into `/etc/sandbox-persistent.sh`
   - Pre-creates the empty NSS certificate database at `~/.pki/nssdb`

2. **`commands.startup`** (each sandbox boot, agent user):
   - Imports the sandbox proxy CA (`/usr/local/share/ca-certificates/proxy-ca.crt`) into Chrome's NSS store, idempotently
   - This is necessary because sandboxd injects the proxy CA at startup (not build time), and Chrome does not use the system `ca-certificates` bundle that Node/curl trust

3. **`files/home/.local/bin/chrome-devtools-mcp`** (mode 755):
   - A thin shell wrapper that calls `npx -y chrome-devtools-mcp@latest` with all the container-appropriate flags pre-set
   - Supports `CDMCP_BROWSER_URL` for attach mode

## Background: why this kit exists

Running a headless browser in an sbx sandbox has non-obvious failure modes that this kit encapsulates:

- **No arm64 Google Chrome** — arm64 requires Playwright's Chromium instead
- **Chromium from apt requires snapd** — which isn't available (no systemd), so apt `chromium-browser` silently installs nothing useful
- **Chrome needs `--no-sandbox`** — without it, Chrome crashes immediately in a container
- **Chrome's NSS store vs. system CA bundle** — Node/curl trust the proxy CA via `NODE_EXTRA_CA_CERTS`; Chrome does not. Without the NSS import, HTTPS navigation fails with TLS errors even though the rest of the sandbox works fine. This is the hardest-to-discover failure and the main motivation for this kit
- **Zombie Chrome processes** — PID 1 is `sleep`; don't keep Chrome running as a background process if you can avoid it

These pain points were independently discovered by at least two Docker Sandboxes users in early 2026 before this kit existed.
