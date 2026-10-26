# MCP Client Wiring

Every MCP-capable client launches VFA the same way: `uvx vfa-mcp` over stdio, with credentials
injected through the server definition's `"env"` object. Only the location of the settings file
and the panel used to open it differ.

!!! note "Placeholder credentials"
    Values such as `sk-...` and `sk-ant-...` are documentation placeholders, not valid keys.
    Replace them with your own Creem license and provider credentials before saving the file.

## Base configuration

This block is the smallest working definition. Every client below accepts it verbatim — only
the surrounding file or panel changes.

```json
{
  "mcpServers": {
    "vfa": {
      "command": "uvx",
      "args": ["vfa-mcp"],
      "env": {
        "VFA_LICENSE_KEY": "YOUR_CREEM_LICENSE_KEY",
        "OPENAI_API_KEY": "sk-...",
        "HEADLESS": "true",
        "VFA_ALLOW_LOCAL_HOST": "true",
        "LITELLM_LOG": "ERROR"
      }
    }
  }
}
```

| Variable | Why it is here |
| --- | --- |
| `VFA_LICENSE_KEY` | Single-seat Creem key. Without it the server exits with `LICENSE_INVALID`. |
| `OPENAI_API_KEY` | Credential read by the provider selected in the preset. |
| `HEADLESS` | `true` keeps Chromium invisible inside a client-spawned process. |
| `VFA_ALLOW_LOCAL_HOST` | Lets the agent reach loopback targets — drop it (default `false`) when auditing only public URLs. |
| `LITELLM_LOG` | `ERROR` keeps JSON-RPC logs free of provider noise. |

## Client setup

### Zoo Code / Roo Code (`mcpSettings.json`)

Click the **MCP Servers** plug icon ⚙️ → **Edit MCP Settings**, then add:

```json
{
  "mcpServers": {
    "vfa": {
      "command": "uvx",
      "args": ["vfa-mcp"],
      "env": {
        "VFA_LICENSE_KEY": "YOUR_CREEM_LICENSE_KEY",
        "OPENAI_API_KEY": "sk-...",
        "HEADLESS": "true",
        "VFA_ALLOW_LOCAL_HOST": "true",
        "LITELLM_LOG": "ERROR"
      }
    }
  }
}
```

Keep `autoApprove` empty (or omit it) so every audit requires explicit confirmation before an
autonomous browser run starts.

### Claude Desktop (`claude_desktop_config.json`)

- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "vfa": {
      "command": "uvx",
      "args": ["vfa-mcp"],
      "env": {
        "VFA_LICENSE_KEY": "YOUR_CREEM_LICENSE_KEY",
        "OPENAI_API_KEY": "sk-...",
        "HEADLESS": "true",
        "VFA_ALLOW_LOCAL_HOST": "true",
        "LITELLM_LOG": "ERROR"
      }
    }
  }
}
```

Quit Claude Desktop completely (tray icon → **Quit**) and reopen it — tools are discovered only
at startup.

### Cursor (`.cursor/mcp.json` or `Settings → Features → MCP`)

```json
{
  "mcpServers": {
    "vfa": {
      "command": "uvx",
      "args": ["vfa-mcp"],
      "env": {
        "VFA_LICENSE_KEY": "YOUR_CREEM_LICENSE_KEY",
        "OPENAI_API_KEY": "sk-...",
        "HEADLESS": "true",
        "VFA_ALLOW_LOCAL_HOST": "true",
        "LITELLM_LOG": "ERROR"
      }
    }
  }
}
```

Use `.cursor/mcp.json` for a project-scoped server and `~/.cursor/mcp.json` for a global one,
then reload the window.

## Provider drop-in presets

Replace the `"env"` object in any configuration above with your preferred model. The full,
copy-paste-ready presets live in
[Environment Variables → Drop-in presets](configuration.md#drop-in-presets).

| Preset | Core model | Notes |
| --- | --- | --- |
| ⚡ DeepSeek | `deepseek-chat` | Cheapest reasoning; pairs with `deepseek-flash` for vision. |
| 🧠 Anthropic Claude 3.5 Sonnet | `claude-3-5-sonnet-20241022` | Reference setup, multimodal out of the box. |
| 🚀 Google Gemini 2.0 Flash | `gemini/gemini-2.0-flash` | Fast and multimodal, good volume default. |
| 🌐 OpenRouter | `openrouter/anthropic/claude-3.5-sonnet` | Universal gateway, any hosted model. |
| 🏠 Ollama | `ollama/llama3.2-vision` | 100% local, no provider key required. |
| ⚙️ OpenAI-compatible | `openai/deepseek-v4-pro` | vLLM, LiteLLM proxy, self-hosted clusters. |
| ⚙️ Anthropic-compatible | `claude-3-5-sonnet` | Messages-protocol proxies. |

!!! warning "Keep the license key"
    A preset replaces the whole `"env"` object, which drops `VFA_LICENSE_KEY`. Re-add it, or
    the server aborts on startup.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Server missing from the client list | `VFA_LICENSE_KEY` absent or expired — check the client's MCP log. |
| `404 model_not_found` | `VFA_LLM_MODEL` does not exist for the chosen `VFA_LLM_PROVIDER`. |
| Browser launch failure | `playwright install chromium` was never run for the interpreter that runs VFA. |
| Navigation blocked | Anti-SSRF guard rejected a loopback/private target; see `VFA_ALLOW_LOCAL_HOST`. |
