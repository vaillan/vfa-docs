# Quickstart & License

Get VFA running inside your MCP client in under five minutes.

## 1. Prerequisites

| Requirement | Notes |
| --- | --- |
| Python 3.10+ | Only needed for the `pip` install path. `uvx` manages its own runtime. |
| `uv` / `uvx` | Recommended launcher. Install with `curl -LsSf https://astral.sh/uv/install.sh \| sh`. |
| Playwright Chromium | VFA drives a **local** browser. Install the binary once with the command below. |
| LLM API key | Any provider supported by LiteLLM (DeepSeek, Anthropic, Google, OpenRouter, Ollama…). |
| Creem license | A single-seat `VFA_LICENSE_KEY` — required, the server refuses to start without it. |

```bash
# Install the Chromium build Playwright needs (run once per machine)
playwright install chromium
```

If `playwright` is not on your `PATH`, use `python -m playwright install chromium`, or let
`uvx` do it: `uvx --from playwright playwright install chromium`.

## 2. Activate your license

VFA is sold and licensed through **Creem**. After purchase, your Creem receipt contains a
license key in the `VFA-` prefixed format; copy it from the customer portal. The key is
validated at server startup and refreshed periodically during the session.

```bash
export VFA_LICENSE_KEY="YOUR_CREEM_LICENSE_KEY"
```

!!! danger "Single-seat license"
    A license is bound to one seat. Sharing a key across machines or running concurrent
    instances under the same key will trigger Creem's activation limit and the server will
    exit with a `LICENSE_INVALID` error. Deactivate old machines from your Creem customer
    portal before moving to a new one.

!!! tip "Never hardcode the key in a commit"
    Store `VFA_LICENSE_KEY` in your shell profile, a secret manager, or the `env` block of
    your MCP client configuration — never in a tracked file.

## 3. Configure your LLM provider

At minimum, set a provider, a model and an API key:

```bash
export VFA_LLM_PROVIDER="deepseek"
export VFA_LLM_MODEL="deepseek-chat"
export VFA_LLM_API_KEY="YOUR_PROVIDER_API_KEY"
```

Check [Environment Variables](configuration.md) for the complete reference and copy-paste
presets per provider.

## 4. Wire VFA into your MCP client

Every snippet below injects `VFA_LICENSE_KEY` (plus your LLM credentials) into the server's
`env` block.

!!! note "Placeholder credentials"
    Values such as `YOUR_CREEM_LICENSE_KEY` and `YOUR_PROVIDER_API_KEY` are documentation
    placeholders, not valid credentials. Replace them with your own Creem license and provider
    key before saving the file.

=== "Cursor"

    Create or edit `.cursor/mcp.json` (project-scoped) or `~/.cursor/mcp.json` (global):

    ```json
    {
      "mcpServers": {
        "vfa": {
          "command": "uvx",
          "args": ["vfa-mcp"],
          "env": {
            "VFA_LICENSE_KEY": "YOUR_CREEM_LICENSE_KEY",
            "VFA_LLM_PROVIDER": "deepseek",
            "VFA_LLM_MODEL": "deepseek-chat",
            "VFA_LLM_API_KEY": "YOUR_PROVIDER_API_KEY"
          }
        }
      }
    }
    ```

    Reload the window, then confirm `vfa` appears as a connected server under
    **Settings → MCP**.

=== "Claude Desktop"

    Edit `claude_desktop_config.json`:

    - macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
    - Windows: `%APPDATA%\Claude\claude_desktop_config.json`

    ```json
    {
      "mcpServers": {
        "vfa": {
          "command": "uvx",
          "args": ["vfa-mcp"],
          "env": {
            "VFA_LICENSE_KEY": "YOUR_CREEM_LICENSE_KEY",
            "VFA_LLM_PROVIDER": "anthropic",
            "VFA_LLM_MODEL": "claude-3-5-sonnet-latest",
            "VFA_LLM_API_KEY": "YOUR_ANTHROPIC_API_KEY"
          }
        }
      }
    }
    ```

    Quit Claude Desktop completely (tray icon → **Quit**) and reopen it. Tools are discovered
    on startup.

=== "Roo Code / Cline"

    Edit the extension's `mcpSettings.json` (**MCP Servers → Edit MCP Settings**):

    ```json
    {
      "mcpServers": {
        "vfa": {
          "command": "uvx",
          "args": ["vfa-mcp"],
          "disabled": false,
          "autoApprove": [],
          "env": {
            "VFA_LICENSE_KEY": "YOUR_CREEM_LICENSE_KEY",
            "VFA_LLM_PROVIDER": "openrouter",
            "VFA_LLM_MODEL": "anthropic/claude-3.5-sonnet",
            "VFA_LLM_API_KEY": "YOUR_OPENROUTER_API_KEY"
          }
        }
      }
    }
    ```

    Roo Code and Cline both read the same settings file format; keep `autoApprove` empty so
    every audit requires explicit confirmation.

!!! note "Windows paths"
    On Windows, quote the command if it contains spaces, e.g.
    `"command": "C:\\Users\\you\\.local\\bin\\uvx.exe"`. Run `where uvx` to find the exact
    path.

## 5. Verify the installation

Ask your client to audit a public URL:

> Use `qa_audit_url` to audit `https://example.com` and summarize the findings.

A healthy response returns a JSON report containing `status`, `findings`, `screenshots` and
metrics. If the server fails to appear, check the client's MCP log output — the usual causes
are a missing `VFA_LICENSE_KEY`, an expired key, or Chromium not being installed.

## Standalone CLI execution

VFA is also runnable without any client integration, which is handy for CI pipelines or
debugging:

```bash
pip install vfa-mcp
playwright install chromium
export VFA_LICENSE_KEY="YOUR_CREEM_LICENSE_KEY"

vfa-mcp --help
```

The process speaks JSON-RPC over stdio, exactly like the client-managed launch — pipe a
`tools/call` request into it and read the response from stdout.

## Next steps

- [Environment Variables](configuration.md) — every knob, plus provider presets.
- [MCP Tools](tools.md) — parameter contracts and output schemas.
