# Claude Code & Vertex AI Compatibility

Explains the cause of the `advisor-tool-2026-03-01` beta header error (`HTTP 400 Bad Request`) when calling Google Cloud Vertex AI or Envoy AI Gateway from Claude Code (v2.1+), along with client and gateway solutions.

***

## 1. Symptom & Error Log

When running recent versions of Claude Code (e.g., `v2.1.272`) on Google Cloud Workstations or local environments, submitting a prompt returns an `HTTP 400 Bad Request` error:

```
 ▐▛███▛█   Claude Code v2.1.272
▝▜██████▀  Sonnet 5 with medium effort · API Usage Billing
  ▝▝ ▝▝    /tmp/test

❯ hi

● API Error: 400 Unexpected value(s) `advisor-tool-2026-03-01` for the `anthropic-beta` header. Please consult our documentation at platform.claude.com/docs or try again without the header.
```

This occurs both when routing through Envoy AI Gateway and when calling Vertex AI directly (`CLAUDE_CODE_USE_VERTEX=1`).

***

## 2. What is the Advisor Tool (`advisor-tool-2026-03-01`)?

`advisor-tool-2026-03-01` enables Anthropic's Advisor Tool beta feature.

### 2.1 Official Documentation

* English Docs: [Claude Platform Docs — Advisor tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/advisor-tool)
* Korean Docs: [Claude Platform Docs — Advisor 도구](https://platform.claude.com/docs/ko/agents-and-tools/tool-use/advisor-tool)
* Specification Excerpt:

  > _"Pair a faster executor model with a higher-intelligence advisor model that provides strategic guidance mid-generation."_\
  > _"The advisor tool is in beta. Include the beta header `advisor-tool-2026-03-01` in your requests."_

### 2.2 How It Works (Executor + Advisor Hybrid Reasoning)

The Advisor Tool is a server-side orchestration feature that pairs a fast model with a higher-capability model:

1. Executor Model:
   * A faster model (`claude-sonnet-5` or `claude-haiku-4-5`) handles routine tasks such as file exploration and code edits.
2. Mid-Generation Consultation:
   * When complex reasoning is needed, the executor emits a `server_tool_use` call to invoke the `advisor` tool mid-generation.
3. Server-Side Advisor Call:
   * Keeping the client stream open, Anthropic's API server forwards the conversation context to an Advisor model (`claude-opus-4-6`).
   * The advisor returns a short plan (`advisor_tool_result`, 400–700 tokens), and the executor resumes generation.

### 2.3 Automatic Injection by Claude Code

When Sonnet is selected, recent versions of Claude Code automatically add two fields to `/v1/messages` requests:

1. HTTP Request Header:

   ```http
   anthropic-beta: advisor-tool-2026-03-01
   ```
2. JSON Request Body (`tools` Array):

   ```json
   {
     "model": "claude-sonnet-5",
     "tools": [
       {
         "type": "advisor_20260301",
         "name": "advisor",
         "model": "claude-opus-4-6",
         "max_uses": 3
       }
     ]
   }
   ```

***

## 3. Root Cause: Anthropic API vs. Vertex AI Architecture

This error is returned by Google Cloud Vertex AI, which does not support this beta feature and rejects unrecognized headers and tool types.

### 3.1 Why Vertex AI Rejects It

* Cross-Model Routing vs. Endpoint Isolation:
  * Anthropic's 1st-party API (`api.anthropic.com`) manages all Claude models under a single orchestrator, allowing a single `/v1/messages` request to invoke both Sonnet and Opus.
  * Google Cloud Vertex AI (`aiplatform.googleapis.com`) exposes each model as a separate endpoint (`/projects/.../models/claude-sonnet-5:streamRawPredict` vs. `.../models/claude-opus-4-6:streamRawPredict`).
  * Because IAM, regional quotas, and billing in GCP are scoped per endpoint, a request to the Sonnet endpoint cannot invoke the Opus endpoint internally.

### 3.2 Two-Layer Validation on Vertex AI

Vertex AI (`harpoon`) validates requests against an allowlist at two layers:

#### Layer 1: HTTP Header Validation (`anthropic-beta`)

Calling with `-H "anthropic-beta: advisor-tool-2026-03-01"`:

```bash
curl -i -X POST http://<GATEWAY_IP>:8080/anthropic/v1/messages \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: advisor-tool-2026-03-01" \
  -d '{"model":"claude-sonnet-5","max_tokens":10,"messages":[{"role":"user","content":"hi"}]}'
```

* Response (`HTTP 400 Bad Request`):

  ```http
  HTTP/1.1 400 Bad Request
  x-vertex-ai-internal-prediction-backend: harpoon
  request-id: req_vrtx_011Cf69ffRgRk2w8QxkdR5ZA

  {
    "type": "error",
    "error": {
      "type": "invalid_request_error",
      "message": "Unexpected value(s) `advisor-tool-2026-03-01` for the `anthropic-beta` header. Please consult our documentation at platform.claude.com/docs or try again without the header."
    }
  }
  ```

#### Layer 2: JSON Body Schema Validation (`tools` Array)

Stripping the `anthropic-beta` header while leaving `advisor_20260301` in the JSON body:

```bash
curl -i -X POST http://<GATEWAY_IP>:8080/anthropic/v1/messages \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -d '{"model":"claude-sonnet-5","max_tokens":10,"tools":[{"type":"advisor_20260301","name":"advisor","model":"claude-opus-4-6"}],"messages":[{"role":"user","content":"hi"}]}'
```

* Response (`HTTP 400 Bad Request`):

  ```http
  HTTP/1.1 400 Bad Request
  x-vertex-ai-internal-prediction-backend: harpoon
  request-id: req_vrtx_011Cf69mJL5zmnNWDZji9DuH

  {
    "type": "error",
    "error": {
      "type": "invalid_request_error",
      "message": "tools.0: Input tag 'advisor_20260301' found using 'type' does not match any of the expected tags: 'bash_20250124', 'browser_toolset_20260801', 'computer_toolset_20260801', 'custom', 'memory_20250818', 'text_editor_20250124', 'text_editor_20250429', 'text_editor_20250728', 'tool_search_tool_bm25', 'tool_search_tool_bm25_20251119', 'tool_search_tool_regex', 'tool_search_tool_regex_20251119', 'web_search_20250305'"
    }
  }
  ```

Removing only the HTTP header is not sufficient because the request still fails body validation at Layer 2.

***

## 4. Recommended Fix: Disable Client Experimental Betas

Configure Claude Code to omit experimental beta headers and tool definitions.

### 4.1 Environment Variable (`CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS=1`)

Set the variable in your shell or `~/.bashrc`:

```bash
export CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS=1
```

### 4.2 Persistent Setting in `~/.claude/settings.json`

Add the flag to the `env` block in `~/.claude/settings.json`. [`scripts/prepare_ws_settings.py`](https://github.com/seoeunbae/agentrouter-multi-llm-gcp/blob/main/scripts/prepare_ws_settings.py) includes this setting by default:

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://<GATEWAY_IP>:8080/anthropic",
    "ANTHROPIC_MODEL": "claude-sonnet-5",
    "CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS": "1"
  },
  "model": "claude-sonnet-5"
}
```

With this enabled, Claude Code omits both the `anthropic-beta: advisor-tool-2026-03-01` header and the `advisor_20260301` tool entry.

***

## 5. Gateway-Level Sanitization Architecture

Where client environment variables cannot be enforced across all workstations, Envoy AI Gateway can strip unsupported headers and body fields before forwarding requests to Vertex AI.

### 5.1 Option Comparison

| Criteria | Option A: EnvoyExtensionPolicy (Lua Filter) | Option B: HTTPRouteFilter (HeaderModifier) | Option C: Custom ExtProc (gRPC Service) |
| --- | :---: | :---: | :---: |
| Selective Header Filtering (`anthropic-beta`) | Supported | Unsupported (drops entire header) | Supported |
| Body Tool Removal (`advisor_20260301`) | Supported | Unsupported (fails with Layer 2 HTTP 400) | Supported (JSON AST parsing) |
| Extra Pod Deployment | None (single CRD YAML) | None | Required (separate Deployment) |
| Added Latency | < 0.1ms | None | 1–3ms gRPC hop |

### 5.2 `EnvoyExtensionPolicy` (Lua Filter) Example

Envoy Gateway's `EnvoyExtensionPolicy` CRD can sanitize both the header and JSON body in-flight without extra containers.

(The target `AIGatewayRoute` or `HTTPRoute` must have `aigateway.envoyproxy.io/processing-body-mode: buffered` enabled.)

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyExtensionPolicy
metadata:
  name: anthropic-vertex-compat-sanitizer
  namespace: routing
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: envoy-ai-gateway
  lua:
    - type: Inline
      inline: |
        function envoy_on_request(request_handle)
          local path = request_handle:headers():get(":path")
          if not path or not string.find(path, "^/anthropic") then
            return
          end

          -- 1. Strip unsupported flags from anthropic-beta header
          local beta_hdr = request_handle:headers():get("anthropic-beta")
          if beta_hdr then
            local valid_betas = {}
            for token in string.gmatch(beta_hdr, "([^,]+)") do
              local trimmed = token:match("^%s*(.-)%s*$")
              if not string.find(trimmed, "^advisor%-tool") and
                 not string.find(trimmed, "^prompt%-caching") then
                table.insert(valid_betas, trimmed)
              end
            end
            if #valid_betas == 0 then
              request_handle:headers():remove("anthropic-beta")
            else
              request_handle:headers():replace("anthropic-beta", table.concat(valid_betas, ","))
            end
          end

          -- 2. Remove advisor_20260301 tool definition from JSON body
          local body = request_handle:body()
          if body then
            local raw = body:getBytes(0, body:length())
            if raw and string.find(raw, "advisor_20260301", 1, true) then
              local sanitized = raw
              sanitized = string.gsub(sanitized, '%s*{[^{}]*"type"%s*:%s*"advisor_20260301"[^{}]*}%s*,?', '')
              sanitized = string.gsub(sanitized, ',%s*%]', ']')
              sanitized = string.gsub(sanitized, '"tools"%s*:%s*%[%s*%]%s*,?', '')
              sanitized = string.gsub(sanitized, ',%s*}', '}')

              body:setBytes(sanitized)
              request_handle:headers():replace("content-length", tostring(#sanitized))
            end
          end
        end
```

### 5.3 Recommended Setup

1. Client Default: Set `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS: "1"` in workstation setup scripts ([`scripts/prepare_ws_settings.py`](https://github.com/seoeunbae/agentrouter-multi-llm-gcp/blob/main/scripts/prepare_ws_settings.py)) and `~/.claude/settings.json`.
2. Gateway Fallback: Optionally apply the `EnvoyExtensionPolicy` above so unconfigured clients continue working without errors.
