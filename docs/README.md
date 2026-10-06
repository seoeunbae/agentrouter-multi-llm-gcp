# Agentrouter 기반 멀티 LLM 서빙 아키텍처

GKE 환경에서 [Agentrouter (formerly Envoy AI Gateway)](https://github.com/theagentrouter/agent-router)(`v1.1.0`), Kubernetes Gateway API Inference Extension(GIE `v1.6.0`), llm-d-router(`EPP v0.10.0`), vLLM, Google Cloud Model Armor, Cloud DLP, Vertex AI를 통합한 멀티 LLM 서빙 플랫폼입니다.

***

## 문서 바로가기

### 시작하기
* [빠른 시작 및 배포](getting-started-ko/quickstart.md): `make deploy` 명령으로 GKE, L4 GPU, Cloud SQL, GCS 인프라와 게이트웨이 스택을 배포합니다.
* [고객 워크숍 가이드](getting-started-ko/workshop-guide.md): IAM 권한, GPU 쿼터 점검, 다계층 인증, EPP 접두사 캐시, Model Armor, Phoenix 관측성 실습을 다룹니다.

### 아키텍처 & 설계
* [게이트웨이 및 리소스 계층 구조](architecture-ko/gateway-resources.md): Kubernetes Gateway API 리소스 계층과 라우팅·정책 연결 구조를 설명합니다.
* [엔드투엔드 요청 처리 흐름](architecture-ko/request-flow.md): 임직원, 마이크로서비스, 외부 파트너별 요청 처리 시퀀스를 정리합니다.
* [프로젝트 설계 명세](architecture-ko/project-spec.md): 기능 목록, 마일스톤 검증 기준, 컴포넌트 인터페이스 규격을 정의합니다.

### 검증 & 운영 가이드
* [수동 테스트 가이드](operations-ko/manual-test-guide.md): 시나리오별 수동 검증 절차와 명령어를 정리합니다.
* [Claude Code & Vertex AI 호환성 가이드](operations-ko/claude-code-compatibility.md): `advisor-tool-2026-03-01` 베타 헤더 오류 원인과 해결 방법을 설명합니다.

***

## 1. 아키텍처 개요

관리형 모델(Vertex AI Gemini, Anthropic Claude)과 자체 호스팅 모델(vLLM Gemma 2B)을 단일 진입점으로 제공합니다.

* 인프라: GKE Standard 클러스터, NVIDIA L4 GPU Spot 노드풀 2대(`g2-standard-8`), Cloud SQL PostgreSQL 16, Cloud Storage 버킷으로 구성됩니다.
* 모델 서빙: Cloud Storage FUSE로 모델 가중치를 마운트하고, vLLM `v0.29.0`에서 PagedAttention V1 접두사 캐싱(Prefix Caching)을 활성화합니다.
* 라우팅: [Agentrouter](https://github.com/theagentrouter/agent-router)가 요청 본문의 `model` 필드를 읽어 백엔드로 분기합니다. GIE `InferencePool`과 `llm-d-router`(EPP) 접두사 스코어러를 연동해 캐시 적중률이 높은 GPU 파드로 라우팅합니다.
* 보안 및 인가: 임직원 GCIP JWT, 내부 서비스 Google SA ID 토큰, 외부 파트너 API Key 인증을 게이트웨이에서 처리하며, Model Armor 및 Cloud DLP 가드레일(`08-model-armor`)을 적용합니다.
* 트래픽 제어: Redis 분산 카운터 기반으로 토큰 속도 제한(`BackendTrafficPolicy`)과 파트너별 쿼터 격리(`QuotaPolicy`)를 수행합니다.
* 관측성: [Arize Phoenix](https://github.com/Arize-ai/phoenix)와 Cloud SQL을 연동해 OTLP 트레이스를 저장하고, Cloud Monitoring으로 GPU 및 캐시 지표를 수집합니다.

### 1.1 레퍼런스 아키텍처

#### Reference Architecture: AgentRouter Multi-LLM on GCP

![Reference Architecture: AgentRouter Multi-LLM on GCP](assets/ref-arch-agentrouter-multi-llm.png)

#### Reference Architecture: Claude Apps Gateway on GCP

![Reference Architecture: Claude Apps Gateway on GCP](assets/ref-arch-claude-apps-gateway.png)

#### Reference Architecture: Claude Code with IdP (Okta)

![Reference Architecture: Claude Code with IdP (Okta)](assets/ref-arch-claude-code-idp.png)

#### Reference Architecture: AI Gateway with Apigee, Model Armor

![Reference Architecture: AI Gateway with Apigee, Model Armor](assets/architecture.png)

상세 아키텍처와 흐름은 아래 문서를 참고하세요.

* [게이트웨이 및 리소스 계층 구조](architecture-ko/gateway-resources.md)
* [엔드투엔드 요청 처리 흐름](architecture-ko/request-flow.md)
* [프로젝트 설계 명세](architecture-ko/project-spec.md)

***

## 2. 라우팅 경로 및 지원 모델

게이트웨이(`http://<GATEWAY_IP>:8080/v1/chat/completions`)로 유입된 요청은 `model` 값에 따라 각 백엔드로 분기됩니다.

| 모델 식별자 | 서빙 위치 | 백엔드 스키마 | 처리 방식 |
| --- | --- | --- | --- |
| `gemini-2.5-flash` | Google Cloud Vertex AI | `GCPVertexAI` | GCP 자격증명으로 Vertex AI 호출 |
| `claude-sonnet-5` | Google Cloud Vertex AI | `GCPAnthropic` | OpenAI 규격 및 Anthropic Messages API 지원 |
| `gemma-rr` | GKE GPU 클러스터 | `OpenAI` | K8s Service 기반 L4 라운드로빈 전달 |
| `gemma-epp` | GKE GPU 클러스터 | `OpenAI` | GIE EPP 접두사 캐시 스코어러 기반 라우팅 |
| `gemma-epp-noprefix` | GKE GPU 클러스터 | `OpenAI` | GIE EPP 대기열 스코어러 기반 라우팅 |

***

## 3. 다계층 인증 체계

클라이언트 환경에 따라 3가지 인증 방식을 지원합니다.

### 3.1 사내 임직원 및 Cloud Workstation (GCIP JWT)

* Identity Platform(GCIP)에서 발급한 JWT 서명을 Google 공개키(JWKS)로 검증합니다.
* 토큰의 `department` 클레임을 추출해 `x-tenant-id` 헤더로 주입합니다.
* 클라이언트가 보낸 기존 테넌트 헤더는 덮어써 위조를 방지합니다.
* Cloud Workstation의 Claude Code CLI와 연동됩니다.

### 3.2 내부 마이크로서비스 (Google SA ID 토큰)

* GKE Workload Identity로 발급받은 Google 서비스 계정 ID 토큰을 검증합니다.
* 토큰의 `email` 클레임을 `x-tenant-id` 헤더로 주입합니다.

### 3.3 외부 파트너 (API Key 및 쿼터 격리)

* 파트너 전용 도메인(`partner.agent-router.internal`)에 별도 `SecurityPolicy`를 적용합니다.
* 등록된 API Key를 검증한 뒤 `x-tenant-id` 헤더를 주입하고, 원본 API Key 헤더는 제거합니다.
* 허용된 `gemma-rr` 모델만 호출할 수 있으며, `claude-sonnet-5` 등 비인가 모델 호출 시 `HTTP 404`를 반환합니다.
* Key Issuer 서비스로 신규 API Key를 발급하고 K8s Secret에 동기화합니다.

헤더 위조 방지 동작은 `/authtest` 엔드포인트에서 확인할 수 있습니다.

***

## 4. 트래픽 제어 및 쿼터 정책

* 토큰 속도 제한 (`BackendTrafficPolicy`):
  * `claude-sonnet-5` 호출을 분당 500,000 토큰으로 제한합니다.
  * 응답의 `llm_total_token` 값을 읽어 Redis 카운터에 반영합니다.
* 파트너별 쿼터 격리 (`QuotaPolicy`):
  * 파트너별로 독립된 토큰 예산을 할당합니다 (`acme-corp` 분당 60토큰, `globex` 분당 500토큰).
  * `acme-corp`가 한도를 초과해 `HTTP 429`로 차단되어도 `globex` 요청은 정상 처리(`HTTP 200`)됩니다.

***

## 5. 라우팅 방식별 벤치마크 결과

GKE Standard 환경(NVIDIA L4 GPU 2대, vLLM `v0.29.0`, `google/gemma-2-2b-it`, 약 2,000 토큰 길이의 프롬프트 16개)에서 측정한 결과입니다. 각 측정 전 vLLM 파드를 재시작해 캐시를 초기화했습니다.

### 5.1 지연시간 및 캐시 적중 통계

| 라우팅 대상 | 라우팅 방식 | 1차 Cold TTFT P50 | 2차 Eval TTFT P50 | Eval P95 | Eval P99 | 캐시 적중 토큰 수 |
| --- | --- | --- | --- | --- | --- | --- |
| `gemma-epp` | InferencePool + EPP 접두사 스코어러 ON | 0.773s | 0.139s | 0.178s | 0.191s | 25,184 |
| `gemma-rr` | K8s Service L4 라운드로빈 | 1.351s | 0.613s | 1.829s | 1.877s | 10,560 |
| `gemma-epp-noprefix` | InferencePool + EPP 대기열 스코어러 | 0.807s | 0.251s | 0.590s | 0.644s | 15,520 |

### 5.2 요약

* 접두사 캐시 효과: `gemma-epp`는 Cold(`0.773s`) 대비 Eval(`0.139s`)에서 TTFT가 5.55배 단축되었습니다.
* 접두사 스코어러 이득: 대기열 기준 분산(`gemma-epp-noprefix`, `0.251s`) 대비 지연시간이 44.6%(`0.112s`) 줄었습니다.
* 캐시 균등 분산: 라운드로빈(`gemma-rr`)은 한쪽 파드에만 캐시가 쏠려 P99 지연시간(`1.877s`)이 증가한 반면, `gemma-epp`는 두 파드에 고르게 적중(`49.8%`)되어 총 25,184 토큰(라운드로빈 대비 2.38배)을 재사용했습니다.
* 인증 오버헤드: JWT 검증 필터 적용 후에도 Eval P50은 `0.139s`로, 적용 전(`0.148s`)과 차이가 없었습니다.

***

## 6. 관측성

* [Arize Phoenix](https://github.com/Arize-ai/phoenix): Cloud SQL Auth Proxy를 통해 PostgreSQL 16에 OTLP 트레이스를 저장하며, 웹 UI(`:6006`)에서 입출력 토큰과 구간별 지연시간을 조회할 수 있습니다.
* Google Cloud Monitoring: `PodMonitoring`으로 vLLM 메트릭(`:8000/metrics`)을 수집해 접두사 캐시 적중률과 TTFT 대시보드를 제공합니다.
