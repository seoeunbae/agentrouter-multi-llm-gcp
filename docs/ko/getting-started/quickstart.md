# 빠른 시작 및 배포 (Quickstart & Deployment)

이 문서는 Agentrouter Multi-LLM GCP 플랫폼의 사전 요구사항, `Makefile` 기반 원커맨드 자동 배포, 그리고 번호별 Kubernetes 매니페스트 수동 배포 절차를 안내합니다.

---

## 1. 사전 요구사항

- Google Cloud SDK (`gcloud`): 대상 GCP 프로젝트 인증 완료
- Terraform 1.5 이상
- Kubernetes CLI (`kubectl`)
- `jq`, `curl`
- Hugging Face Access Token: `google/gemma-2-2b-it` 모델 라이선스 동의가 완료된 `Read` 권한 토큰 필수

---

## 2. 한 번에 배포하기 (One-Command Deployment)

```bash
# 1. 프로젝트 ID 및 Hugging Face 토큰 환경변수 등록
export GCP_PROJECT="<YOUR_PROJECT_ID>"
export GCP_REGION="asia-southeast1"
export HF_TOKEN="hf_your_token_here"

# 2. 필수 GCP API 일괄 활성화 (신규 프로젝트인 경우)
gcloud services enable compute.googleapis.com container.googleapis.com \
  sqladmin.googleapis.com storage.googleapis.com iam.googleapis.com \
  iamcredentials.googleapis.com aiplatform.googleapis.com \
  identitytoolkit.googleapis.com firebase.googleapis.com \
  modelarmor.googleapis.com dlp.googleapis.com --project=$GCP_PROJECT

# 3. 인프라 프로비저닝 및 매니페스트 원스톱 배포
make deploy

# 4. 3대 라우팅 비교 벤치마크 실행
make benchmark

# 5. 인프라 정리 및 비용 차단 (LoadBalancer 선삭제 후 Terraform Destroy)
make clean
```

---

## 3. 단계별 매니페스트 수동 배포

매니페스트는 의존성 순서에 따라 번호별 폴더(`00-setup` ~ `08-model-armor`)로 정렬되어 있습니다.

```bash
# 0. 환경변수 치환 및 백엔드 인증 시크릿 생성
make update-manifests
make setup-secrets

# 1. 기본 CRD 및 컨트롤러 설치
kubectl apply --server-side -f manifests/00-setup/envoy-gateway-crds.yaml
kubectl apply --server-side -f manifests/00-setup/agent-router-crds.yaml
kubectl apply -f manifests/00-setup/envoy-gateway-controller.yaml
kubectl apply -f manifests/00-setup/gie-install.yaml
kubectl apply -f manifests/00-setup/agent-router.yaml
kubectl apply -f manifests/00-setup/envoy-gateway-config.yaml
kubectl apply -f manifests/00-setup/envoy-ai-gateway-ratelimit-svc.yaml
kubectl rollout status deployment/envoy-gateway -n envoy-gateway-system --timeout=180s
kubectl rollout status deployment/ai-gateway-controller -n default --timeout=180s

# 2. 게이트웨이 및 워크로드 순차 배포
kubectl apply -k manifests/01-gateway
kubectl apply -k manifests/02-security
kubectl apply -k manifests/03-vllm
kubectl apply -k manifests/04-inference-pool
kubectl apply -k manifests/05-routing
kubectl apply -k manifests/06-traffic-policy
kubectl apply -k manifests/07-observability
kubectl apply -k manifests/08-model-armor
```

---

## 4. Makefile 타겟 레퍼런스

| 명령어 | 설명 |
|---|---|
| `make deploy` | `terraform apply` 실행 후 GKE 자격증명 획득, 매니페스트 플레이스홀더 치환, 전체 K8s 매니페스트(`00`~`08`) 배포 |
| `make deploy-model-armor` | Google Cloud Model Armor 및 Cloud DLP 템플릿만 타겟 프로비저닝 후 `manifests/08-model-armor` 배포 |
| `make update-manifests` | Terraform 출력값(`PROJECT_ID`, `GCS_BUCKET_NAME`, `GSA_EMAIL`, `SQL_CONNECTION_NAME`, `MODEL_ARMOR_TEMPLATE_ID`)을 YAML 매니페스트에 치환 |
| `make setup-secrets` | `routing` 네임스페이스에 `vertex-ai-sa-key` Kubernetes Secret 생성 |
| `make apply-manifests` | 번호별 Kustomize 디렉토리(`00-setup` ~ `08-model-armor`) 순차 배포 및 Cloud Monitoring 대시보드 생성 |
| `make benchmark` | 클러스터 내부 `e2e-benchmark-job` 실행 및 3대 라우팅 지연시간/캐시 적중 결과 출력 |
| `make placeholders` | Git 커밋 전 매니페스트 내 환경변수 값을 플레이스홀더로 복원 |
| `make clean` / `make destroy` | Kubernetes LoadBalancer 및 워크로드 선삭제 후 `terraform destroy` 수행 |

---

## 5. 다음 단계

- 고객 워크숍 실습: GCP IAM 권한, L4 GPU 쿼터 확인 및 전체 시나리오 실습은 [고객 워크숍 가이드](workshop-guide.md)를 참고하십시오.
- 시나리오별 수동 검증: 상세한 검증 명령어는 [수동 테스트 가이드](../operations/manual-test-guide.md)를 참고하십시오.
- Claude Code 연동: 실험적 베타 헤더(`advisor-tool-2026-03-01`)와 Vertex AI 간 호환성 해결 방안은 [Claude Code & Vertex AI 호환성 가이드](../operations/claude-code-compatibility.md)를 참고하십시오.
