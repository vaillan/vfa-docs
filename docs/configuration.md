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

Copy the block for your provider into the `env` object of your MCP client configuration, or
export it in your shell.

!!! note "Placeholder credentials"
    Every key below is a documentation placeholder of the form `YOUR_*_API_KEY` — none of
    them is a real credential or a valid key format. Substitute your own values and keep them
    out of version control.

=== "DeepSeek"

    ```bash
    export VFA_LLM_PROVIDER="deepseek"
    export VFA_LLM_MODEL="deepseek-chat"
    export VFA_LLM_API_KEY="YOUR_DEEPSEEK_API_KEY"
    export VFA_LLM_BASE_URL="https://api.deepseek.com/v1"
    # DeepSeek is text-only: pair it with a vision model for grounding
    export VFA_VISION_PROVIDER="gemini"
    export VFA_VISION_MODEL="gemini-2.0-flash-exp"
    export VFA_VISION_API_KEY="YOUR_GOOGLE_AI_STUDIO_KEY"
    ```

    The lowest-cost configuration. Because `deepseek-chat` has no vision support, a vision
    override is strongly recommended.

=== "Claude 3.5 Sonnet"

    ```bash
    export VFA_LLM_PROVIDER="anthropic"
    export VFA_LLM_MODEL="claude-3-5-sonnet-latest"
    export VFA_LLM_API_KEY="YOUR_ANTHROPIC_API_KEY"
    # No vision override needed: the core model is multimodal
    ```

    The reference configuration — best grounding quality for complex flows. One model handles
    both reasoning and vision, so no override is necessary.

=== "Gemini 2.0 Flash"

    ```bash
    export VFA_LLM_PROVIDER="gemini"
    export VFA_LLM_MODEL="gemini-2.0-flash-exp"
    export VFA_LLM_API_KEY="YOUR_GOOGLE_AI_STUDIO_KEY"
    ```

    Fast, cheap and multimodal — a balanced default for high-volume audit runs.

=== "OpenRouter"

    ```bash
    export VFA_LLM_PROVIDER="openrouter"
    export VFA_LLM_MODEL="anthropic/claude-3.5-sonnet"
    export VFA_LLM_API_KEY="YOUR_OPENROUTER_API_KEY"
    export VFA_LLM_BASE_URL="https://openrouter.ai/api/v1"
    # Route screenshots to a cheaper multimodal model while keeping Sonnet for reasoning
    export VFA_VISION_PROVIDER="openrouter"
    export VFA_VISION_MODEL="google/gemini-2.0-flash-exp:free"
    ```

    Any model available on OpenRouter can be selected by changing `VFA_LLM_MODEL` — the
    provider prefix in the model string follows OpenRouter's own namespace.

=== "Local Ollama"

    ```bash
    export VFA_LLM_PROVIDER="ollama"
    export VFA_LLM_MODEL="llama3.1:70b"
    export VFA_LLM_BASE_URL="http://localhost:11434"
    # VFA_LLM_API_KEY is not required for a local daemon
    export VFA_VISION_PROVIDER="ollama"
    export VFA_VISION_MODEL="llava:latest"
    # Chrome does not need to reach a private range, but the model does:
    # keep the anti-SSRF guard on unless you are auditing local apps
    export VFA_ALLOW_LOCAL_HOST="false"
    ```

    Fully offline reasoning. `VFA_LICENSE_KEY` still requires one activation call to Creem;
    after the license is cached, subsequent runs work without internet access.

## Validation checklist

Before opening an issue, confirm the following:

1. `VFA_LICENSE_KEY` is present in the **client's** `env` block, not only in your shell.
2. `VFA_LLM_MODEL` exists for the chosen `VFA_LLM_PROVIDER` (a mismatch surfaces as a
   provider `404 model_not_found` error).
3. `playwright install chromium` has been executed for the same Python interpreter that runs
   VFA.
4. Any `VFA_VISION_MODEL` you set accepts image inputs.
