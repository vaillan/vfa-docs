# Quickstart & License

!!! tip "Recommended: Zero-Install with `uvx`"
    You do **not** need to install `vfa-mcp` via `pip`. Using `uvx` allows your MCP client to fetch and run the latest version from PyPI in an isolated sandbox, avoiding global Python dependency conflicts and Windows `PATH` issues.

Get VFA running inside your MCP client in under five minutes.

## 1. Prerequisites

VFA requires **`uvx`** (to run in an isolated environment without installation) and **Chromium** (for local browser automation). Select your operating system below:

=== "Windows"

    **1. Install `uv` (includes `uvx`):**
    Open PowerShell and run:
    ```powershell
    powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
    ```

    *If PowerShell does not recognize `uvx` after installation, add it to your PATH:*
    ```powershell
    [Environment]::SetEnvironmentVariable("Path", [Environment]::GetEnvironmentVariable("Path", "User") + ";$env:USERPROFILE\.local\bin", "User")
    ```

    **2. Install Chromium (run once):**
    ```powershell
    uvx playwright install chromium
    ```

=== "macOS"

    **1. Install `uv`:**
    ```bash
    brew install uv
    # Or via curl:
    # curl -LsSf https://astral.sh/uv/install.sh | sh
    ```

    **2. Install Chromium (run once):**
    ```bash
    uvx playwright install chromium
    ```

=== "Linux (Ubuntu / Debian / Arch)"

    **1. Install `uv`:**
    ```bash
    curl -LsSf https://astral.sh/uv/install.sh | sh
    ```

    **2. Install Chromium with system dependencies (run once):**
    ```bash
    uvx playwright install --with-deps chromium
    ```

!!! tip "Zero-Docker Architecture"
    VFA drives Chromium natively on your host with an ephemeral memory sandbox and Anti-SSRF protection. You do **not** need Docker, containers, or remote browsers.

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
    `TU_LICENCIA_CREEM` and `sk-...` are documentation placeholders, not valid credentials.
    Replace them with your own Creem license and provider key before saving the file — the
    server exits with `LICENSE_INVALID` if the license key is still the placeholder.

!!! tip "Swapping providers"
    The block above is the **default OpenAI** configuration (`OPENAI_API_KEY` is honoured
    directly). To run DeepSeek, Claude, Gemini, OpenRouter or a local Ollama instance, replace
    only the `"env"` object with the preset of your choice — copy-paste presets for each
    provider live in [Environment Variables](configuration.md#drop-in-presets).

=== "Cursor"

    **Option A — file:** create or edit `.cursor/mcp.json` (project-scoped) or
    `~/.cursor/mcp.json` (global).

    **Option B — UI:** **Settings → Cursor Settings → Features → MCP → Add new MCP
    server**, then paste the same block.

    ```json
    {
      "mcpServers": {
        "vfa": {
          "command": "uvx",
          "args": ["vfa-mcp"],
          "env": {
            "VFA_LICENSE_KEY": "TU_LICENCIA_CREEM",
            "OPENAI_API_KEY": "sk-...",
            "HEADLESS": "true",
            "VFA_ALLOW_LOCAL_HOST": "true",
            "LITELLM_LOG": "ERROR"
          }
        }
      }
    }
    ```

    Reload the window (`Cmd/Ctrl + Shift + P` → **Developer: Reload Window**), then confirm
    `vfa` appears as a connected server under **Settings → Features → MCP** with its three
    tools listed.

=== "Claude Desktop"

    Edit `claude_desktop_config.json` (create it if it does not exist):

    - macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
    - Windows: `%APPDATA%\Claude\claude_desktop_config.json`

    ```json
    {
      "mcpServers": {
        "vfa": {
          "command": "uvx",
          "args": ["vfa-mcp"],
          "env": {
            "VFA_LICENSE_KEY": "TU_LICENCIA_CREEM",
            "OPENAI_API_KEY": "sk-...",
            "HEADLESS": "true",
            "VFA_ALLOW_LOCAL_HOST": "true",
            "LITELLM_LOG": "ERROR"
          }
        }
      }
    }
    ```

    Quit Claude Desktop completely (tray icon → **Quit**, not just closing the window) and
    reopen it. Tools are discovered on startup, so a full restart is mandatory.

=== "Zoo Code / Cline"

    Open **MCP Servers → Edit MCP Settings** to edit `mcpSettings.json`:

    - VS Code: `%APPDATA%\Code\User\globalStorage\saoudrizwan.claude-dev\settings\mcp_settings.json`
      (Windows) or `~/Library/Application Support/Code/User/globalStorage/saoudrizwan.claude-dev/settings/mcp_settings.json`
      (macOS)

    ```json
    {
      "mcpServers": {
        "vfa": {
          "command": "uvx",
          "args": ["vfa-mcp"],
          "disabled": false,
          "autoApprove": [],
          "env": {
            "VFA_LICENSE_KEY": "TU_LICENCIA_CREEM",
            "OPENAI_API_KEY": "sk-...",
            "HEADLESS": "true",
            "VFA_ALLOW_LOCAL_HOST": "true",
            "LITELLM_LOG": "ERROR"
          }
        }
      }
    }
    ```

    Roo Code and Cline both read the same settings format. Keep `autoApprove` empty so every
    audit requires explicit confirmation, and keep `"disabled": false` or the server will not
    start.

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

## Standalone CLI execution (Testing without an MCP client)

If you want to test VFA directly from your terminal or CI/CD without configuring an IDE:

```bash
# 1. Ensure Chromium is installed once
uvx playwright install chromium

# 2. Run directly from PyPI (no pip install or PATH setup required)
uvx vfa-mcp --help
```
!!! warning "Why avoid `pip install`?"
    Installing via `pip install vfa-mcp` places the executable in your Python `Scripts` directory. On Windows and macOS, if this folder is not in your system `$PATH`, your terminal will fail with `command not found: vfa-mcp`. Running via `uvx` eliminates this problem entirely.

The process speaks JSON-RPC over stdio, exactly like the client-managed launch — pipe a
`tools/call` request into it and read the response from stdout.

## Next steps

- [Environment Variables](configuration.md) — every knob, plus provider presets.
- [MCP Tools](tools.md) — parameter contracts and output schemas.
