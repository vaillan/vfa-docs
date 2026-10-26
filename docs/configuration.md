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

Each preset below is a **drop-in replacement for the `"env"` object** of your MCP client
configuration: copy the block, drop the previous `"env"` object, and paste the new one in its
place. For standalone runs, export the same keys in your shell instead.

!!! note "Placeholder credentials"
    `TU_LICENCIA_CREEM` and every `YOUR_*_API_KEY` value are documentation placeholders — none
    of them is a real credential or a valid key format. Substitute your own values and keep
    them out of version control.

=== "DeepSeek (Reasoning + Fast Vision)"

    ```json
    "env": {
      "VFA_LICENSE_KEY": "TU_LICENCIA_CREEM",
      "VFA_LLM_PROVIDER": "deepseek",
      "VFA_LLM_MODEL": "deepseek-chat",
      "DEEPSEEK_API_KEY": "YOUR_DEEPSEEK_API_KEY",
      "VFA_VISION_PROVIDER": "deepseek",
      "VFA_VISION_MODEL": "deepseek-flash",
      "HEADLESS": "true",
      "LITELLM_LOG": "ERROR"
    }
    ```

    The lowest-cost configuration. `deepseek-chat` handles the ReAct loop while the vision
    model consumes screenshots; if your account exposes no multimodal DeepSeek model, point
    `VFA_VISION_*` at Gemini 2.0 Flash or OpenRouter instead.

=== "Anthropic Claude 3.5 Sonnet"

    ```json
    "env": {
      "VFA_LICENSE_KEY": "TU_LICENCIA_CREEM",
      "VFA_LLM_PROVIDER": "anthropic",
      "VFA_LLM_MODEL": "claude-3-5-sonnet-latest",
      "ANTHROPIC_API_KEY": "YOUR_ANTHROPIC_API_KEY",
      "HEADLESS": "true",
      "LITELLM_LOG": "ERROR"
    }
    ```

    The reference configuration — best grounding quality on complex flows. One multimodal
    model covers both reasoning and vision, so no `VFA_VISION_*` override is needed.

=== "Google Gemini 3.8 Flash (Multimodal)"

    ```json
    "env": {
      "VFA_LICENSE_KEY": "TU_LICENCIA_CREEM",
      "VFA_LLM_PROVIDER": "gemini",
      "VFA_LLM_MODEL": "gemini-3.8-flash",
      "GEMINI_API_KEY": "YOUR_GOOGLE_AI_STUDIO_KEY",
      "HEADLESS": "true",
      "LITELLM_LOG": "ERROR"
    }
    ```

    Fast, cheap and natively multimodal — a balanced default for high-volume audit runs where
    latency matters more than deep reasoning.

=== "OpenRouter (Universal Gateway)"

    ```json
    "env": {
      "VFA_LICENSE_KEY": "TU_LICENCIA_CREEM",
      "VFA_LLM_PROVIDER": "openrouter",
      "VFA_LLM_MODEL": "anthropic/claude-3.5-sonnet",
      "OPENROUTER_API_KEY": "YOUR_OPENROUTER_API_KEY",
      "VFA_LLM_BASE_URL": "https://openrouter.ai/api/v1",
      "VFA_VISION_PROVIDER": "openrouter",
      "VFA_VISION_MODEL": "google/gemini-2.0-flash-exp:free",
      "HEADLESS": "true",
      "LITELLM_LOG": "ERROR"
    }
    ```

    One credential unlocks every hosted model. Any model on OpenRouter can be selected by
    changing `VFA_LLM_MODEL` with OpenRouter's own `vendor/model` namespace.

=== "Ollama (100% Local / Offline)"

    ```json
    "env": {
      "VFA_LICENSE_KEY": "TU_LICENCIA_CREEM",
      "VFA_LLM_PROVIDER": "ollama",
      "VFA_LLM_MODEL": "llama3.1:70b",
      "OLLAMA_API_BASE": "http://localhost:11434",
      "VFA_VISION_PROVIDER": "ollama",
      "VFA_VISION_MODEL": "llava:latest",
      "VFA_ALLOW_LOCAL_HOST": "false",
      "HEADLESS": "true",
      "LITELLM_LOG": "ERROR"
    }
    ```

    Fully offline reasoning — `YOUR_*_API_KEY` is not required for a local daemon. The first
    run still performs one Creem activation call; afterwards the cached license lets VFA run
    without internet access. Keep `VFA_ALLOW_LOCAL_HOST` at `false` unless you are auditing
    local apps.

=== "OpenAI-compatible gateway"

    ```json
    "env": {
      "VFA_LICENSE_KEY": "TU_LICENCIA_CREEM",
      "VFA_LLM_PROVIDER": "openai",
      "VFA_LLM_MODEL": "openai/deepseek-v4-pro",
      "VFA_VISION_PROVIDER": "openai",
      "VFA_VISION_MODEL": "openai/deepseek-flash",
      "DEEPSEEK_API_KEY": "YOUR_GATEWAY_API_KEY",
      "OPENAI_API_BASE": "https://api.deepseek.com",
      "HEADLESS": "true",
      "LITELLM_LOG": "ERROR"
    }
    ```

    Use this for vLLM, LiteLLM Proxy, LM Studio, Together or any endpoint that speaks the
    OpenAI Chat Completions API. Keep the `openai/` prefix on the model name and point
    `OPENAI_API_BASE` (or `VFA_LLM_BASE_URL`) at the gateway root.

=== "Anthropic-compatible gateway"

    ```json
    "env": {
      "VFA_LICENSE_KEY": "TU_LICENCIA_CREEM",
      "VFA_LLM_PROVIDER": "anthropic",
      "VFA_LLM_MODEL": "claude-3-5-sonnet",
      "VFA_LLM_API_KEY": "YOUR_GATEWAY_API_KEY",
      "ANTHROPIC_BASE_URL": "https://anthropic-proxy.internal.example",
      "HEADLESS": "true",
      "LITELLM_LOG": "ERROR"
    }
    ```

    For Anthropic-compatible proxies or regional endpoints. `ANTHROPIC_BASE_URL` redirects the
    Messages API traffic; `VFA_LLM_BASE_URL` takes precedence when both are present.

## Complete reference

Both tables below cover every variable VFA reads. Values in the **Default** column are what
VFA uses when the variable is absent from the client's `env` block.

### Credentials & reasoning

| Variable | Default | Required when | Purpose |
| --- | --- | --- | --- |
| `VFA_LICENSE_KEY` | *(empty)* | **Always** | Commercial license validation from Creem |
| `VFA_LLM_PROVIDER` | `openai` | Using non-OpenAI | Protocol/routing (`openai`, `anthropic`, `gemini`, `deepseek`, `xai`, `openrouter`, `ollama`) |
| `VFA_LLM_MODEL` | `gpt-4o` | Using non-gpt-4o | Reasoning model (must support tool calling) |
| `VFA_LLM_API_KEY` | *(empty)* | Universal key | Generic credential overriding provider variable |
| `VFA_LLM_BASE_URL` | *(empty)* | Custom gateways | Base URL override (`OPENAI_BASE_URL` also honoured) |
| `VFA_VISION_PROVIDER` | inherits | Vision split | Routing for multimodal calls |
| `VFA_VISION_MODEL` | inherits | Vision split | Model for screenshots/visual grounding |
| `VFA_VISION_API_KEY` | inherits | Vision split | Credential for multimodal calls |
| `OLLAMA_API_BASE` | *(empty)* | Ollama local | Local endpoint, e.g. `http://localhost:11434` |
| `VFA_AGENT_MAX_ITERATIONS` | `12` | Long flows | ReAct loop iteration ceiling |
| `VFA_LLM_TEMPERATURE` | `0.0` | Never | Sampling temperature (deterministic QA) |
| `VFA_LLM_TIMEOUT` | `60` | Slow models | Per-call timeout in seconds |
| `VFA_LLM_REQUESTS_PER_SECOND` | `0` | Rate limits | Token-bucket rate limiter; 0 disables |
| `VFA_LLM_CHECKS_PER_SECOND` | `10.0` | Never | Rate limiter polling frequency |

### Browser & sandbox

| Variable | Default | Required when | Purpose |
| --- | --- | --- | --- |
| `HEADLESS` | `true` | UI debugging | Run Chromium without window |
| `VFA_ALLOW_LOCAL_HOST` | `true` | Production | Strict Anti-SSRF blocking when set to false |
| `BROWSER_CDP_URL` | *(empty)* | Existing browser | Optional remote CDP endpoint |
| `BROWSER_CDP_TIMEOUT_MS` | `2000` | Slow CDP | Timeout of CDP connection attempt |
| `VFA_ACTION_TIMEOUT_MS` | `10000` | Heavy SPAs | Individual Playwright action timeout |
| `VFA_SCREENSHOT_FULL_PAGE` | `true` | Viewport only | Full-page screenshots toggle |
| `VFA_SESSION_POOL_ENABLED` | `true` | Multi-tenant | Reuse browser sessions across calls |
| `VFA_SESSION_POOL_TTL` | `300` | Never | Idle seconds before closing session |
| `VFA_SESSION_POOL_MAX_SIZE` | `5` | High concurrency | Max concurrent pooled sessions (LRU) |
| `VFA_SESSION_STORAGE_STATE` | *(empty)* | Authenticated flows | Playwright storageState file path |
| `SSL_VERIFY` | `true` | Self-signed certs | TLS verification switch |

!!! tip "Single source of truth"
    `LITELLM_LOG` is passed straight through to LiteLLM. Set it to `ERROR` in client `env`
    blocks to keep MCP stderr clean; switch to `DEBUG` when diagnosing routing problems.

## Validation checklist

Before opening an issue, confirm the following:

1. `VFA_LICENSE_KEY` is present in the **client's** `env` block, not only in your shell.
2. `VFA_LLM_MODEL` exists for the chosen `VFA_LLM_PROVIDER` (a mismatch surfaces as a
   provider `404 model_not_found` error).
3. `playwright install chromium` has been executed for the same Python interpreter that runs
   VFA.
4. Any `VFA_VISION_MODEL` you set accepts image inputs.
