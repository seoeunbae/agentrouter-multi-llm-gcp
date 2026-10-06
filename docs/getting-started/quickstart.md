# Quickstart & Deployment

Covers prerequisites, automated deployment via `Makefile`, and manual Kubernetes manifest deployment for the Agentrouter Multi-LLM GCP platform.

***

## 1. Prerequisites

* Google Cloud SDK (`gcloud`) authenticated with a GCP project
* Terraform 1.5+
* Kubernetes CLI (`kubectl`)
* `jq`, `curl`
* Hugging Face Access Token with `Read` permission and accepted license terms for `google/gemma-2-2b-it`

***

## 2. One-Command Deployment

```bash
# 1. Export GCP Project ID and Hugging Face token
export GCP_PROJECT="<YOUR_PROJECT_ID>"
export GCP_REGION="asia-southeast1"
export HF_TOKEN="hf_your_token_here"

# 2. Enable required GCP APIs (for new projects)
gcloud services enable compute.googleapis.com container.googleapis.com \
  sqladmin.googleapis.com storage.googleapis.com iam.googleapis.com \
  iamcredentials.googleapis.com aiplatform.googleapis.com \
  identitytoolkit.googleapis.com firebase.googleapis.com \
  modelarmor.googleapis.com dlp.googleapis.com --project=$GCP_PROJECT

# 3. Provision infrastructure and deploy manifests
make deploy

# 4. Run comparative benchmark
make benchmark

# 5. Clean up resources (deletes K8s LoadBalancer before Terraform destroy)
make clean
```

***

## 3. Step-by-Step Manifest Deployment

Manifests are organized in numbered directories (`00-setup` through `08-model-armor`) by dependency order:

```bash
# 0. Substitute placeholders and create backend credentials secret
make update-manifests
make setup-secrets

# 1. Install CRDs and core controllers
kubectl apply --server-side -f manifests/00-setup/envoy-gateway-crds.yaml
kubectl apply --server-side -f manifests/00-setup/agent-router-crds.yaml
kubectl apply -f manifests/00-setup/envoy-gateway-controller.yaml
kubectl apply -f manifests/00-setup/gie-install.yaml
kubectl apply -f manifests/00-setup/agent-router.yaml
kubectl apply -f manifests/00-setup/envoy-gateway-config.yaml
kubectl apply -f manifests/00-setup/envoy-ai-gateway-ratelimit-svc.yaml
kubectl rollout status deployment/envoy-gateway -n envoy-gateway-system --timeout=180s
kubectl rollout status deployment/ai-gateway-controller -n default --timeout=180s

# 2. Apply gateway and workload manifests sequentially
kubectl apply -k manifests/01-gateway
kubectl apply -k manifests/02-security
kubectl apply -k manifests/03-vllm
kubectl apply -k manifests/04-inference-pool
kubectl apply -k manifests/05-routing
kubectl apply -k manifests/06-traffic-policy
kubectl apply -k manifests/07-observability
kubectl apply -k manifests/08-model-armor
```

***

## 4. Makefile Targets

| Target | Description |
| --- | --- |
| `make deploy` | Runs `terraform apply`, fetches GKE credentials, substitutes placeholders, and deploys all manifests (`00`–`08`) |
| `make deploy-model-armor` | Provisions Model Armor and Cloud DLP templates via Terraform and applies `manifests/08-model-armor` |
| `make update-manifests` | Substitutes Terraform outputs (`PROJECT_ID`, `GCS_BUCKET_NAME`, `GSA_EMAIL`, `SQL_CONNECTION_NAME`, `MODEL_ARMOR_TEMPLATE_ID`) into manifests |
| `make setup-secrets` | Creates the `vertex-ai-sa-key` Secret in the `routing` namespace |
| `make apply-manifests` | Applies all Kustomize directories (`00-setup` through `08-model-armor`) and creates the Cloud Monitoring dashboard |
| `make benchmark` | Runs `e2e-benchmark-job` in-cluster and prints latency and cache hit results |
| `make placeholders` | Restores placeholder tokens across manifests before `git commit` |
| `make clean` / `make destroy` | Deletes Kubernetes LoadBalancer services and workloads before running `terraform destroy` |

***

## 5. Next Steps

* Workshop Walkthrough: See the [Customer Workshop Guide](workshop-guide.md) for IAM checks, GPU quota checks, and hands-on testing.
* Scenario Verification: See the [Manual Testing Guide](../verification-and-guides/manual-test-guide.md) to verify JWT auth, SA tokens, Partner API keys, Redis quotas, and Prefix Caching.
* Claude Code Integration: See the [Claude Code & Vertex AI Compatibility Guide](../verification-and-guides/claude-code-compatibility.md) for `advisor-tool-2026-03-01` header handling.
