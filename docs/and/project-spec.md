# 프로젝트 설계 명세

## 1. 아키텍처 개요

* 인프라: GCP 프로젝트 `<YOUR_PROJECT_ID>`(`asia-southeast1`), Gateway API 및 Cloud Storage FUSE CSI 드라이버가 활성화된 GKE Standard 클러스터, NVIDIA L4 GPU Spot 노드풀 2대(`g2-standard-8`), Cloud SQL PostgreSQL 16, 모델 가중치 보관용 Cloud Storage 버킷.
* 모델 서빙: Hugging Face에서 내려받은 가중치를 Cloud Storage에 저장하고 Cloud Storage FUSE(`gke-gcsfuse/volumes: "true"`)로 vLLM `0.29.0` 파드에 마운트합니다. PagedAttention V1 접두사 캐싱(Prefix Caching)을 활성화합니다.
* 라우팅 및 네트워크: Envoy AI Gateway([Agentrouter](https://github.com/theagentrouter/agent-router) `1.1.0`)가 요청 본문의 `model` 키를 기준으로 L7 라우팅을 수행합니다. 내부 라우트는 `InferencePool`(`inference.networking.k8s.io/v1`)로 전달하며, `llm-d-router`(`0.10.0`)가 접두사 캐시 점수를 바탕으로 파드를 선택합니다. Gemini와 Claude 모델은 GCP 자격증명으로 Vertex AI를 호출합니다.
* 보안 및 멀티테넌시: Envoy Gateway `SecurityPolicy` 기반으로 임직원 GCIP JWT, 내부 마이크로서비스 Google SA ID 토큰, 외부 파트너 API Key 인증을 처리하고 모델 접근을 격리합니다.
* 트래픽 제어 및 쿼터: Redis 분산 카운터 기반으로 토큰 속도 제한(`BackendTrafficPolicy`)과 테넌트별 쿼터 격리(`QuotaPolicy`)를 수행합니다.
* 관측성: Cloud SQL Auth Proxy로 연결된 [Arize Phoenix](https://github.com/Arize-ai/phoenix)에 OTLP 트레이스를 저장하고, `PodMonitoring`으로 vLLM `/metrics`의 접두사 캐시 지표를 수집합니다.
* 검증: 클러스터 내부 벤치마크 파드가 3개 라우팅 대상을 무작위 순서로 부하 테스트하며, 각 측정 전 vLLM 캐시를 초기화합니다.

***

## 2. 기능 목록

| # | 기능 | 설명 | 마일스톤 |
| --- | --- | --- | --- |
| 1 | GCP Terraform VPC/GKE | Gateway API, Workload Identity, GCS FUSE가 포함된 GKE 배포 | M1 (완료) |
| 2 | GCP Terraform DB/Storage | Cloud SQL PostgreSQL 16 및 Cloud Storage 버킷 배포 | M1 (완료) |
| 3 | GCP Terraform NodePool | L4 Spot 노드풀 2대(`g2-standard-8`) 배포 | M1 (완료) |
| 4 | GCP Terraform IAM | 서비스 계정 생성 및 KSA/GSA Workload Identity 바인딩 | M1 (완료) |
| 5 | Model Weight Loader | Hugging Face에서 `google/gemma-2-2b-it`를 내려받아 GCS에 적재하는 Job | M2 (완료) |
| 6 | vLLM Workload | GCS FUSE 마운트를 사용하는 vLLM `0.29.0` 파드 배포 및 접두사 캐싱 활성화 | M2 (완료) |
| 7 | Phoenix Tracing Server | Cloud SQL Auth Proxy 사이드카가 포함된 Arize Phoenix UI 배포 | M3 (완료) |
| 8 | K8s GIE & InferencePools | 라운드로빈 및 EPP 대상을 정의하는 InferencePool 구성 | M3 (완료) |
| 9 | llm-d EPP Router | `prefix-cache` 및 `queue` 스코어러가 설정된 `llm-d-router` v0.10.0 배포 | M3 (완료) |
| 10 | Agent Router Hybrid Mesh | 요청 본문 버퍼링 및 백엔드 분기를 수행하는 Envoy AI Gateway 배포 | M3 (완료) |
| 11 | 3-Tier Multi-Authentication | GCIP JWT, Google SA ID 토큰, 파트너 API Key 인증 및 헤더 위조 방지 | M3 (완료) |
| 12 | Redis Traffic & Quota Policy | Redis 기반 속도 제한(`BackendTrafficPolicy`) 및 테넌트 쿼터 격리(`QuotaPolicy`) | M3 (완료) |
| 13 | E2E Benchmark Suite | 3대 라우팅 비교 부하 스크립트, TTFT/캐시 지표 수집, 캐시 초기화 | E2E (완료) |
| 14 | Teardown Automation | 전체 환경을 생성하고 정리하는 Makefile 자동화 | M4 (완료) |
| 15 | Bilingual Documentation | 영문 및 한국어 문서 구성 | Final (완료) |

***

## 3. 마일스톤

| # | 명칭 | 범위 | 의존성 | 상태 |
| --- | --- | --- | --- | --- |
| M1 | 인프라 프로비저닝 (Terraform) | GCP 기본 자원(GKE, NodePool, Cloud SQL, GCS, IAM) | 없음 | 완료 |
| M2 | 모델 및 서빙 엔진 (vLLM) | HF 가중치 적재, GCS FUSE 마운트, vLLM 파드 구동 | M1 | 완료 |
| M3 | 라우팅 및 보안 체계 | GIE, llm-d EPP, Agent Router, Phoenix, 3대 인증, Redis 쿼터 | M2 | 완료 |
| M4 | 자동화 및 패키징 | 번호별 Kustomize 매니페스트, Makefile 자동화, 삭제 절차 검증 | M3 | 완료 |
| E2E | 벤치마크 검증 | 클러스터 내부 3대 라우팅 비교 부하 테스트 및 지표 수집 | M3 | 완료 |
| Final | 문서화 및 배포 준비 | 영문 및 한국어 문서 정비, 민감정보 점검 | M4, E2E | 완료 |

***

## 4. 인터페이스 규격

### 4.1 Terraform ↔ K8s

* Workload Identity 연동용 KSA 이름 및 GSA 이메일 매핑
* 매니페스트 볼륨 마운트용 Cloud Storage 버킷 이름 전달
* Cloud SQL Auth Proxy 연결용 인스턴스 연결 이름 전달

### 4.2 Agent Router ↔ 백엔드 라우트

* Envoy가 `/v1/chat/completions` 요청을 수신합니다.
* `model: "gemini-2.5-flash"`: Google Cloud Vertex AI Gemini로 분기
* `model: "claude-sonnet-5"`: Google Cloud Vertex AI Claude로 분기
* `model: "gemma-rr"`: 일반 K8s Service(L4 라운드로빈)로 전달
* `model: "gemma-epp"`: 접두사 캐시 스코어러 기반 EPP InferencePool로 전달
* `model: "gemma-epp-noprefix"`: 대기열 스코어러 기반 EPP InferencePool로 전달

### 4.3 멀티테넌시 및 보안

* 기본 리스너(`*`): `agent-router-jwt` 정책으로 GCIP JWT 또는 Google SA ID 토큰을 검증하고 `x-tenant-id` 헤더를 주입합니다.
* 파트너 리스너(`partner.agent-router.internal`): `partner-apikey` 정책으로 API Key를 검증하고 `x-tenant-id` 헤더를 주입한 뒤 `X-API-Key` 헤더를 제거합니다.
* Redis 속도 제한 및 쿼터: 응답의 `llm_total_token` 값을 반영해 테넌트별 토큰 버킷을 차감합니다.
