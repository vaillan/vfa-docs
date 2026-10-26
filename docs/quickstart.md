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

VFA is launched as `uvx vfa-mcp` over stdio, with `VFA_LICENSE_KEY` and your provider
credentials declared in the server definition's `"env"` object.

- **[MCP Client Wiring](clients.md)** — copy-paste configs for Zoo Code / Roo Code, Claude
  Desktop and Cursor, plus troubleshooting.
- **[Environment Variables](configuration.md#drop-in-presets)** — the `"env"` object to paste
  for each LLM provider.

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

- [MCP Client Wiring](clients.md) — client configs, presets and troubleshooting.
- [Environment Variables](configuration.md) — every knob, plus provider presets.
- [MCP Tools](tools.md) — parameter contracts and output schemas.
