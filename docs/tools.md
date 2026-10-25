# MCP Tools

VFA exposes three tools over stdio. All of them return a single JSON object; failures return
a structured error object instead of raising, so an agent can reason about them.

| Tool | Purpose |
| --- | --- |
| [`qa_audit_url`](#qa_audit_url) | Autonomous black-box audit of a single page. |
| [`qa_execute_user_flow`](#qa_execute_user_flow) | Drives a multi-step natural-language user journey. |
| [`qa_get_runtime_errors`](#qa_get_runtime_errors) | Drains the buffered runtime error channel. |

Every invocation runs inside an isolated, ephemeral `BrowserContext` that is destroyed on
completion. See the [security overview](index.md#security-overview) for the guarantees this
provides.

## `qa_audit_url`

Opens a URL, waits for network idle, inspects the rendered page and reports functional,
visual and accessibility defects together with any console or network errors collected during
the visit.

### Parameters

| Name | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `url` | `string` | **Yes** | — | Absolute `http`/`https` URL to audit. Private and loopback targets are rejected unless `VFA_ALLOW_LOCAL_HOST=true`. |
| `viewport` | `object` | No | `{"width": 1440, "height": 900}` | Viewport used for rendering and screenshots. |
| `wait_until` | `string` | No | `"networkidle"` | Load strategy: `load`, `domcontentloaded` or `networkidle`. |
| `max_depth` | `integer` | No | `1` | Follow same-origin links up to this depth. `0` audits only the entry page. |
| `include_screenshots` | `boolean` | No | `true` | Attach base64 screenshots (viewport and full page) to the report. |
| `focus` | `string` | No | — | Free-text hint narrowing the audit, e.g. `"checkout funnel and form validation"`. |

### Behavior

1. Validates the target against the anti-SSRF firewall and the license gate.
2. Creates a fresh context, navigates and waits for the requested load state.
3. Captures a full-page screenshot and an annotated DOM snapshot for visual grounding.
4. Runs the reasoning pass: broken layouts, unreachable controls, missing labels, console
   errors and HTTP failures `>= 400` are all correlated into findings.
5. Returns the report and tears the context down.

### Output schema

```json
{
  "status": "completed",
  "url": "https://example.com/pricing",
  "duration_ms": 41230,
  "findings": [
    {
      "id": "FND-001",
      "severity": "high",
      "category": "functional",
      "title": "Primary CTA does not respond to click",
      "description": "Clicking 'Start free trial' produces no navigation and no state change.",
      "selector_hint": "button[data-testid='trial-cta']",
      "evidence": {
        "screenshot_index": 0,
        "console_errors": 1
      }
    }
  ],
  "metrics": {
    "pages_audited": 3,
    "console_errors": 2,
    "http_errors": 1,
    "duration_ms": 41230
  },
  "screenshots": [
    { "name": "viewport", "mime_type": "image/png", "data": "<base64>" }
  ]
}
```

`severity` is one of `critical`, `high`, `medium`, `low`, `info`. `category` is one of
`functional`, `visual`, `accessibility`, `performance`, `network`.

## `qa_execute_user_flow`

Executes a defined user journey — written in natural language — against a starting URL. This
is the functional counterpart to `qa_audit_url`: instead of inspecting a single page, VFA
attempts to *complete a task* the way a real user would, and reports where it got stuck.

### Parameters

| Name | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `url` | `string` | **Yes** | — | Entry point of the flow. Must be an absolute `http`/`https` URL. |
| `steps` | `array` | **Yes** | — | Ordered list of natural-language steps. Minimum length is `1`. |
| `expected_outcome` | `string` | No | — | Natural-language success condition, e.g. `"the order confirmation page is shown"`. |
| `timeout_ms` | `integer` | No | `120000` | Total budget for the whole flow. |
| `stop_on_failure` | `boolean` | No | `true` | Abort on the first failed step instead of continuing optimistically. |

### Step format

Each entry in `steps` is a plain string describing intent, not a selector:

```json
{
  "url": "https://shop.example.com",
  "steps": [
    "Open the product page for the first item in the catalog",
    "Select size M and add the item to the cart",
    "Go to the cart and apply the coupon code SAVE10",
    "Proceed to checkout without creating an account",
    "Fill the shipping form with a test address and submit"
  ],
  "expected_outcome": "The order confirmation page displays an order number",
  "timeout_ms": 180000
}
```

VFA resolves each step through active visual grounding: the screenshot and annotated DOM are
analyzed to locate the element a human would interact with, then the action is replayed in
the browser. Failed steps include the reason and the last observed page state.

### Output schema

```json
{
  "status": "failed",
  "url": "https://shop.example.com",
  "expected_outcome": "The order confirmation page displays an order number",
  "steps": [
    {
      "index": 0,
      "instruction": "Open the product page for the first item in the catalog",
      "status": "success",
      "action": "click",
      "target": "a.product-card__link",
      "duration_ms": 3120
    },
    {
      "index": 2,
      "instruction": "Go to the cart and apply the coupon code SAVE10",
      "status": "failed",
      "action": "fill",
      "target": "input[name='coupon']",
      "error": "Coupon field present but the apply button stayed disabled after input.",
      "duration_ms": 18400
    }
  ],
  "outcome_matched": false,
  "findings": [
    {
      "id": "FND-014",
      "severity": "critical",
      "category": "functional",
      "title": "Coupon apply button never enables",
      "description": "The apply button is disabled even with a syntactically valid coupon code."
    }
  ],
  "metrics": {
    "steps_total": 5,
    "steps_succeeded": 2,
    "steps_failed": 1,
    "steps_skipped": 2,
    "duration_ms": 60510
  },
  "screenshots": [
    { "name": "step_2_failure", "mime_type": "image/png", "data": "<base64>" }
  ]
}
```

`status` is `completed`, `failed` or `timeout`. A `steps[].status` of `skipped` appears when
`stop_on_failure` is `true` and an earlier step already failed.

## `qa_get_runtime_errors`

Drains VFA's runtime error buffer. The buffer collects everything the browser reported while
auditing, independent of what the reasoning model decided to report as findings — useful when
an agent wants raw evidence or a post-run regression signal.

### Parameters

| Name | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `clear` | `boolean` | No | `true` | Empties the buffer after reading. Set to `false` to peek without draining. |
| `source` | `string` | No | `"all"` | Filter by origin: `console`, `pageerror`, `http` or `all`. |
| `min_severity` | `string` | No | `"info"` | Drop entries below this level (`info`, `warning`, `error`). |

### Behavior

The buffer is populated continuously during every audit and flow run. It captures three
channels:

- `console` — `console.warn`, `console.error` and uncaught assertions emitted by the page.
- `pageerror` — uncaught JavaScript exceptions, including stack traces.
- `http` — every response with status `>= 400`, with method, URL and response time.

Calling the tool with `clear: true` returns the entries and empties the buffer, which keeps
successive agent turns from re-reading the same events.

### Output schema

```json
{
  "status": "completed",
  "count": 3,
  "entries": [
    {
      "id": "ERR-0001",
      "source": "console",
      "severity": "error",
      "level": "error",
      "message": "Uncaught TypeError: Cannot read properties of null (reading 'value')",
      "url": "https://shop.example.com/cart",
      "timestamp": "2025-01-14T10:22:41.512Z"
    },
    {
      "id": "ERR-0002",
      "source": "pageerror",
      "severity": "error",
      "message": "TypeError: cart.total is undefined",
      "stack": "at renderCart (cart.js:84:17)\n at HTMLDocument.dispatch (app.js:31:9)",
      "url": "https://shop.example.com/cart",
      "timestamp": "2025-01-14T10:22:41.517Z"
    },
    {
      "id": "ERR-0003",
      "source": "http",
      "severity": "warning",
      "message": "POST https://shop.example.com/api/coupon responded with 422",
      "status_code": 422,
      "duration_ms": 312,
      "url": "https://shop.example.com/cart",
      "timestamp": "2025-01-14T10:22:44.980Z"
    }
  ]
}
```

`count` reflects the number of entries returned in this call, not the historical total.

## Error responses

All three tools share a common failure shape:

```json
{
  "status": "error",
  "error": {
    "code": "SSRF_BLOCKED",
    "message": "Target resolves to a private address; navigation refused.",
    "hint": "Set VFA_ALLOW_LOCAL_HOST=true to audit local applications."
  }
}
```

| Code | Meaning |
| --- | --- |
| `LICENSE_INVALID` | `VFA_LICENSE_KEY` missing, expired or activated on too many seats. |
| `SSRF_BLOCKED` | Target resolved to loopback, link-local or private space. |
| `NAVIGATION_FAILED` | DNS failure, TLS error or the page never reached the requested load state. |
| `LLM_PROVIDER_ERROR` | Upstream provider rejected the request (bad key, unknown model, rate limit). |
| `BROWSER_UNAVAILABLE` | Chromium binary missing — run `playwright install chromium`. |
| `TIMEOUT` | The audit exceeded `timeout_ms`. |

See [Environment Variables](configuration.md) for the settings referenced above.
