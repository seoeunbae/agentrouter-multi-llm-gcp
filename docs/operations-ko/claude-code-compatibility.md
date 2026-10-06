# Claude Code & Vertex AI 호환성 가이드

Claude Code(v2.1+)에서 Envoy AI Gateway 또는 Google Cloud Vertex AI를 호출할 때 발생하는 `advisor-tool-2026-03-01` 베타 헤더 오류의 원인과 클라이언트·게이트웨이 해결 방법을 설명합니다.

***

## 1. 현상 및 에러 로그

Google Cloud Workstations 또는 로컬 환경에서 최신 Claude Code(예: `v2.1.272`)로 프롬프트를 입력하면 아래와 같은 `HTTP 400 Bad Request` 오류가 발생합니다.

```
 ▐▛███▛█   Claude Code v2.1.272
▝▜██████▀  Sonnet 5 with medium effort · API Usage Billing
  ▝▝ ▝▝    /tmp/test

❯ hi

● API Error: 400 Unexpected value(s) `advisor-tool-2026-03-01` for the `anthropic-beta` header. Please consult our documentation at platform.claude.com/docs or try again without the header.
```

이 오류는 Envoy AI Gateway 경유 시와 Vertex AI 직접 호출(`CLAUDE_CODE_USE_VERTEX=1`) 시 모두 동일하게 발생합니다.

***

## 2. Advisor Tool(`advisor-tool-2026-03-01`) 개요

`advisor-tool-2026-03-01`은 Anthropic의 Advisor Tool(조언자 도구) 베타 기능을 활성화하는 헤더 값입니다.

### 2.1 공식 문서

* 영문 문서: [Claude Platform Docs — Advisor tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/advisor-tool)
* 국문 문서: [Claude Platform Docs — Advisor 도구](https://platform.claude.com/docs/ko/agents-and-tools/tool-use/advisor-tool)
* 공식 설명:

  > _"Pair a faster executor model with a higher-intelligence advisor model that provides strategic guidance mid-generation."_\
  > _"The advisor tool is in beta. Include the beta header `advisor-tool-2026-03-01` in your requests."_

### 2.2 동작 원리 (Executor + Advisor 하이브리드 추론)

Advisor Tool은 비용과 추론 성능을 함께 확보하기 위한 서버 사이드 오케스트레이션 기능입니다.

1. Executor(실행자) 모델 수행:
   * 속도가 빠르고 비용이 낮은 실행자 모델(`claude-sonnet-5`, `claude-haiku-4-5`)이 파일 탐색이나 단순 코드 수정 등 기본 작업을 처리합니다.
2. 생성 도중(Mid-generation) 조언 요청:
   * 복잡한 설계나 추론이 필요한 시점에 실행자 모델이 `advisor` 서버 도구(`server_tool_use`)를 호출합니다.
3. 서버 내부 Advisor 호출 및 스트림 재개:
   * 클라이언트 연결을 유지한 채 Anthropic API 서버가 내부적으로 상위 모델인 Advisor 모델(`claude-opus-4-6`)에 대화 맥락을 전달합니다.
   * Advisor 모델이 400~700 토큰 분량의 가이드(`advisor_tool_result`)를 반환하면 실행자 모델이 이어서 생성을 완료합니다.

### 2.3 Claude Code의 자동 주입 동작

최신 Claude Code는 기본 모델을 Sonnet으로 설정했을 때 요청마다 아래 두 항목을 자동으로 추가합니다.

1. HTTP 요청 헤더:

   ```http
   anthropic-beta: advisor-tool-2026-03-01
   ```
2. JSON 요청 바디 (`tools` 배열):

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

## 3. 원인: Anthropic 1st-Party API와 Vertex AI의 구조 차이

이 오류는 Envoy AI Gateway의 문제가 아니라 업스트림 백엔드인 Google Cloud Vertex AI가 해당 베타 기능과 스키마를 지원하지 않아 거부하기 때문에 발생합니다.

### 3.1 Vertex AI에서 미지원하는 이유

* 단일 요청 내 다중 모델(Cross-Model) 라우팅 제약:
  * Anthropic 자체 API(`api.anthropic.com`)는 모든 Claude 모델을 단일 오케스트레이터에서 관리하므로 하나의 `/v1/messages` 요청 안에서 Sonnet과 Opus를 함께 호출하고 과금할 수 있습니다.
  * 반면 Vertex AI(`aiplatform.googleapis.com`)는 모델별로 엔드포인트(`/projects/.../models/claude-sonnet-5:streamRawPredict` 및 `.../models/claude-opus-4-6:streamRawPredict`)가 분리되어 있습니다.
  * GCP의 IAM 권한, 리전별 쿼터, 과금 파이프라인이 호출된 단일 엔드포인트 기준으로 동작하므로 Sonnet 엔드포인트 요청이 내부에서 Opus 엔드포인트를 호출할 수 없습니다.

### 3.2 Vertex AI의 2단계 검증 메커니즘

Vertex AI 서빙 엔진(`harpoon`)은 헤더와 바디 두 단계에 걸쳐 허용 목록(Allowlist)을 검증합니다.

#### 1차 차단: HTTP 헤더 검증

`anthropic-beta: advisor-tool-2026-03-01` 헤더를 포함해 호출한 경우:

```bash
curl -i -X POST http://<GATEWAY_IP>:8080/anthropic/v1/messages \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: advisor-tool-2026-03-01" \
  -d '{"model":"claude-sonnet-5","max_tokens":10,"messages":[{"role":"user","content":"hi"}]}'
```

* 응답 (`HTTP 400 Bad Request`):

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

#### 2차 차단: JSON 바디 `tools` 스키마 검증

헤더만 제거하고 바디의 `advisor_20260301` 도구 정의를 남겨둔 채 호출한 경우:

```bash
curl -i -X POST http://<GATEWAY_IP>:8080/anthropic/v1/messages \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -d '{"model":"claude-sonnet-5","max_tokens":10,"tools":[{"type":"advisor_20260301","name":"advisor","model":"claude-opus-4-6"}],"messages":[{"role":"user","content":"hi"}]}'
```

* 응답 (`HTTP 400 Bad Request`):

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

따라서 헤더만 제거하면 바디 검증 단계에서 다시 400 오류가 발생합니다.

***

## 4. 해결 방법: 클라이언트 베타 플래그 비활성화 (권장)

Claude Code에서 실험적 베타 헤더와 도구를 주입하지 않도록 환경 변수를 설정합니다.

### 4.1 환경 변수 설정 (`CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS=1`)

터미널 또는 `~/.bashrc`에 아래 환경 변수를 추가합니다.

```bash
export CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS=1
```

### 4.2 `~/.claude/settings.json` 설정

`~/.claude/settings.json`의 `env` 블록에 추가하면 모든 세션에 적용됩니다. [`scripts/prepare_ws_settings.py`](https://github.com/seoeunbae/agentrouter-multi-llm-gcp/blob/main/scripts/prepare_ws_settings.py) 스크립트는 이 설정을 자동으로 포함합니다.

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

이 설정을 적용하면 `anthropic-beta: advisor-tool-2026-03-01` 헤더와 `advisor_20260301` 도구 정의가 모두 생략되어 Vertex AI 직결 및 게이트웨이 경유 환경에서 정상 동작합니다.

***

## 5. 게이트웨이 수준 자동 정제 아키텍처

개별 개발자 워크스테이션 설정을 일괄 통제하기 어려운 경우, Envoy AI Gateway에서 미지원 헤더와 바디 필드를 제거해 Vertex AI로 전달할 수 있습니다.

### 5.1 방식 비교

| 항목 | 방안 A: EnvoyExtensionPolicy (Lua 필터) | 방안 B: HTTPRouteFilter (HeaderModifier) | 방안 C: Custom ExtProc (gRPC 서버) |
| --- | :---: | :---: | :---: |
| 헤더(`anthropic-beta`) 선별 제거 | 가능 (특정 플래그만 제거) | 불가능 (헤더 전체 삭제만 가능) | 가능 |
| 바디(`tools` 내 `advisor`) 제거 | 가능 | 불가능 (2차 400 오류 발생) | 가능 (JSON 파싱) |
| 추가 파드 배포 | 불필요 (CRD YAML 1개 추가) | 불필요 | 필요 (별도 Deployment 운영) |
| 추가 지연시간 | 없음 (< 0.1ms) | 없음 | 낮음 (1~3ms gRPC 홉) |

### 5.2 `EnvoyExtensionPolicy` (Lua 필터) 예시

Envoy Gateway의 `EnvoyExtensionPolicy` CRD를 사용하면 별도 컨테이너 없이 게이트웨이에서 헤더와 JSON 바디를 함께 정리할 수 있습니다.

(대상 `AIGatewayRoute` 또는 `HTTPRoute`에 `aigateway.envoyproxy.io/processing-body-mode: buffered` 어노테이션이 필요합니다.)

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

          -- 1. anthropic-beta 헤더에서 Vertex AI 미지원 플래그 제거
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

          -- 2. JSON 바디 내 advisor_20260301 도구 정의 제거
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

### 5.3 운영 권장 구성

1. 기본 설정: 워크스테이션 설정 스크립트([`scripts/prepare_ws_settings.py`](https://github.com/seoeunbae/agentrouter-multi-llm-gcp/blob/main/scripts/prepare_ws_settings.py))와 `~/.claude/settings.json`에 `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS: "1"`을 적용합니다.
2. 게이트웨이 보완: 클라이언트가 환경 변수를 누락한 경우에도 오류가 나지 않도록 필요 시 `EnvoyExtensionPolicy`를 함께 배포합니다.
