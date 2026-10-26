# Environment Variables

VFA is configured exclusively through environment variables. In MCP clients they are
declared in the `env` block of the server definition; for standalone runs they are plain
shell exports.

!!! tip "Precedence"
    A variable defined in the MCP client's `env` block always wins over the parent shell
    environment, which in turn wins over VFA's built-in defaults.

## Core settings

| Variable | Required | Default | Description |
| --- | --- | --- | --- |
| `VFA_LICENSE_KEY` | **Yes** | — | Creem single-seat license key. Validated at startup and refreshed periodically; an invalid or over-activated key aborts the server with `LICENSE_INVALID`. |
| `VFA_LLM_PROVIDER` | **Yes** | — | LiteLLM provider prefix used for the reasoning model (`deepseek`, `anthropic`, `gemini`, `openrouter`, `ollama`, `azure`, …). |
| `VFA_LLM_MODEL` | **Yes** | — | Model identifier passed to the provider, e.g. `deepseek-chat`, `claude-3-5-sonnet-latest`, `gemini-2.0-flash-exp`. |
| `VFA_LLM_API_KEY` | **Yes**\* | — | API key for the reasoning provider. |
| `VFA_LLM_BASE_URL` | No | Provider default | Overrides the provider endpoint. Required for self-hosted gateways, proxies, Azure deployments and local Ollama. |

\* Not needed when the target provider requires no authentication (a local Ollama daemon,
for example).

## Vision overrides

The vision model performs screenshot understanding and visual grounding. When any
`VFA_VISION_*` variable is set, the vision sub-system stops inheriting the core LLM settings
and uses the dedicated model instead — useful for pairing a cheap text model with a strong
vision model.

| Variable | Required | Default | Description |
| --- | --- | --- | --- |
| `VFA_VISION_PROVIDER` | No | Value of `VFA_LLM_PROVIDER` | LiteLLM provider used for screenshot analysis and visual grounding. |
| `VFA_VISION_MODEL` | No | Value of `VFA_LLM_MODEL` | Vision-capable model identifier. Must accept image inputs. |
| `VFA_VISION_API_KEY` | No | Value of `VFA_LLM_API_KEY` | API key for the vision provider, when it differs from the text provider. |

!!! warning "Vision-capable models only"
    Setting `VFA_VISION_MODEL` to a text-only model (for example `deepseek-chat`) causes
    grounding to degrade to DOM heuristics and can fail audits that depend on visual
    confirmation. Use a multimodal model.

## Runtime & browser

| Variable | Required | Default | Description |
| --- | --- | --- | --- |
| `HEADLESS` | No | `true` | Runs Chromium without a visible window. Set to `false` to watch the audit live while debugging. |
| `VFA_ALLOW_LOCAL_HOST` | No | `false` | Disables the anti-SSRF firewall so VFA can reach `localhost` and private ranges. Enable only for trusted local testing. |
| `BROWSER_CDP_URL` | No | — | Connects to an existing Chromium over CDP instead of launching a managed browser, e.g. `http://127.0.0.1:9222`. Useful for reusing an authenticated session. |
| `VFA_SESSION_POOL_ENABLED` | No | `false` | Keeps warmed browser contexts alive between audits to cut cold-start latency. Contexts remain isolated per run; disable on shared or multi-tenant machines. |

!!! danger "Anti-SSRF bypass"
    `VFA_ALLOW_LOCAL_HOST=true` lets the agent navigate to cloud metadata endpoints
    (`169.254.169.254`) and internal services. Never enable it inside an environment that
    shares a network with production infrastructure.

## Drop-in presets

Each preset below is a ready-made `"env"` object. Swap it in for the `"env"` block of the
server definition described in [MCP Client Wiring](clients.md), or `export` the same names in
your shell for a standalone run. Every preset carries the runtime flags an MCP host normally
forgets: `HEADLESS`, `VFA_ALLOW_LOCAL_HOST` and `LITELLM_LOG`.

!!! note "Placeholder credentials"
    Every key below is a documentation placeholder of the form `sk-...` — none of them is a
    real credential or a valid key format. Substitute your own values and keep them out of
    version control.

!!! warning "`VFA_LICENSE_KEY` is not part of the presets"
    These snippets replace **only** the `"env"` object. Keep `"VFA_LICENSE_KEY"` alongside
    the provider keys — the server refuses to start without a valid Creem license.

=== "⚡ DeepSeek"

    ```json
    "env": {
      "VFA_LLM_PROVIDER": "deepseek",
      "VFA_LLM_MODEL": "deepseek-chat",
      "VFA_VISION_PROVIDER": "deepseek",
      "VFA_VISION_MODEL": "deepseek-flash",
      "DEEPSEEK_API_KEY": "sk-...",
      "HEADLESS": "true",
      "VFA_ALLOW_LOCAL_HOST": "true",
      "LITELLM_LOG": "ERROR"
    }
    ```

    Lowest cost per audit with an explicit reasoning + vision split. `deepseek-chat` handles
    planning while `deepseek-flash` keeps screenshot grounding visual. Set
    `LITELLM_LOG` to `ERROR` so client logs stay readable during long runs.

=== "🧠 Claude 3.5 Sonnet"

    ```json
    "env": {
      "VFA_LLM_PROVIDER": "anthropic",
      "VFA_LLM_MODEL": "claude-3-5-sonnet-20241022",
      "ANTHROPIC_API_KEY": "sk-ant-...",
      "HEADLESS": "true",
      "VFA_ALLOW_LOCAL_HOST": "true",
      "LITELLM_LOG": "ERROR"
    }
    ```

    The reference configuration. One multimodal model covers both reasoning and vision, so no
    `VFA_VISION_*` override is needed — best grounding quality on complex flows.

=== "🚀 Gemini 2.0 Flash"

    ```json
    "env": {
      "VFA_LLM_PROVIDER": "gemini",
      "VFA_LLM_MODEL": "gemini/gemini-2.0-flash",
      "GEMINI_API_KEY": "AIzaSy...",
      "HEADLESS": "true",
      "VFA_ALLOW_LOCAL_HOST": "true",
      "LITELLM_LOG": "ERROR"
    }
    ```

    Fast, cheap and multimodal — the balanced default for high-volume audit runs.

=== "🌐 OpenRouter"

    ```json
    "env": {
      "VFA_LLM_PROVIDER": "openrouter",
      "VFA_LLM_MODEL": "openrouter/anthropic/claude-3.5-sonnet",
      "VFA_VISION_MODEL": "openrouter/google/gemini-2.0-flash-001",
      "OPENROUTER_API_KEY": "sk-or-v1-...",
      "HEADLESS": "true",
      "VFA_ALLOW_LOCAL_HOST": "true",
      "LITELLM_LOG": "ERROR"
    }
    ```

    Universal gateway: any model in OpenRouter's namespace can be selected by changing
    `VFA_LLM_MODEL`. The preset keeps Sonnet for reasoning and routes screenshots to a cheaper
    Gemini Flash model.

=== "🏠 Ollama (local / offline)"

    ```json
    "env": {
      "VFA_LLM_PROVIDER": "ollama",
      "VFA_LLM_MODEL": "ollama/llama3.2-vision",
      "OLLAMA_API_BASE": "http://localhost:11434",
      "HEADLESS": "true",
      "VFA_ALLOW_LOCAL_HOST": "true",
      "LITELLM_LOG": "ERROR"
    }
    ```

    Fully offline reasoning: no provider key is required. `VFA_LICENSE_KEY` still validates
    once against Creem; after the license is cached, later runs work without internet access.
    Keep `VFA_ALLOW_LOCAL_HOST` aligned with what you audit — the model endpoint is local
    either way, but the guard governs *browser* targets.

=== "⚙️ OpenAI-compatible"

    ```json
    "env": {
      "VFA_LLM_PROVIDER": "openai",
      "VFA_LLM_MODEL": "openai/deepseek-v4-pro",
      "VFA_VISION_PROVIDER": "openai",
      "VFA_VISION_MODEL": "openai/deepseek-flash",
      "OPENAI_API_BASE": "https://api.deepseek.com",
      "OPENAI_API_KEY": "sk-...",
      "HEADLESS": "true",
      "VFA_ALLOW_LOCAL_HOST": "true",
      "LITELLM_LOG": "ERROR"
    }
    ```

    For any gateway that speaks the OpenAI wire format (vLLM, LiteLLM proxy, Together,
    Fireworks, a self-hosted cluster). Point `OPENAI_API_BASE` at the gateway root and keep the
    `openai/` prefix inside the model identifiers.

=== "⚙️ Anthropic-compatible"

    ```json
    "env": {
      "VFA_LLM_PROVIDER": "anthropic",
      "VFA_LLM_MODEL": "claude-3-5-sonnet",
      "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
      "ANTHROPIC_API_KEY": "sk-...",
      "HEADLESS": "true",
      "VFA_ALLOW_LOCAL_HOST": "true",
      "LITELLM_LOG": "ERROR"
    }
    ```

    For proxies exposing the Anthropic Messages API. `ANTHROPIC_BASE_URL` replaces the public
    endpoint while leaving the request shape untouched.

!!! tip "Escaping in JSON"
    Because `"env"` is a JSON object, never wrap values in single quotes and never add `export`
    — those are shell syntaxes, valid only in a terminal session.

## Validation checklist

## Validation checklist

Before opening an issue, confirm the following:

1. `VFA_LICENSE_KEY` is present in the **client's** `env` block, not only in your shell.
2. `VFA_LLM_MODEL` exists for the chosen `VFA_LLM_PROVIDER` (a mismatch surfaces as a
   provider `404 model_not_found` error).
3. `playwright install chromium` has been executed for the same Python interpreter that runs
   VFA.
4. Any `VFA_VISION_MODEL` you set accepts image inputs.
