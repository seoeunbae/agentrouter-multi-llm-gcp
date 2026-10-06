# Quickstart & Deployment

This guide covers prerequisites, one-command automated deployment via `Makefile`, and step-by-step Kubernetes manifest application for the Agentrouter Multi-LLM GCP platform.

---

## 1. Prerequisites

- Google Cloud SDK (`gcloud`) authenticated with an active GCP project
- Terraform 1.5+
- Kubernetes CLI (`kubectl`)
- `jq`, `curl`
- Hugging Face Access Token: Must have accepted the model license terms for `google/gemma-2-2b-it` with `Read` permission

---

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

# 3. Provision infrastructure and deploy manifests end-to-end
make deploy

# 4. Execute 3-way comparative benchmark suite
make benchmark

# 5. Clean teardown and stop billing (detaches K8s LoadBalancer before Terraform destroy)
make clean
```

---

## 3. Step-by-Step Manifest Application

Manifests are organized into numbered directories (`00-setup` through `08-model-armor`) following execution dependencies:

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

---

## 4. Makefile Target Reference

| Target | Description |
|---|---|
| `make deploy` | Runs `terraform apply`, fetches GKE credentials, substitutes manifest placeholders, and deploys all Kubernetes manifests (`00`–`08`) |
| `make deploy-model-armor` | Provisions only Google Cloud Model Armor & Cloud DLP templates via Terraform and applies `manifests/08-model-armor` |
| `make update-manifests` | Substitutes Terraform outputs (`PROJECT_ID`, `GCS_BUCKET_NAME`, `GSA_EMAIL`, `SQL_CONNECTION_NAME`, `MODEL_ARMOR_TEMPLATE_ID`) into YAML manifests |
| `make setup-secrets` | Creates the `vertex-ai-sa-key` Kubernetes Secret in the `routing` namespace |
| `make apply-manifests` | Applies all numbered Kustomize directories (`00-setup` through `08-model-armor`) and creates the Cloud Monitoring dashboard |
| `make benchmark` | Launches the in-cluster `e2e-benchmark-job` and streams comparative latency & cache hit results |
| `make placeholders` | Restores placeholder tokens across all manifests prior to `git commit` |
| `make clean` / `make destroy` | Deletes Kubernetes LoadBalancer services and workloads before running `terraform destroy` |

---

## 5. Next Steps

- Hands-on Workshop Walkthrough: Follow the [Customer Workshop Guide](workshop-guide.md) for IAM checks, GPU quota verification, and end-to-end scenario testing.
- Scenario Verification: Run through the [Manual Testing Guide](../operations/manual-test-guide.md) to validate JWT auth, SA tokens, Partner API keys, Redis quotas, and Prefix Caching.
- Claude Code Integration: See the [Claude Code & Vertex AI Compatibility Guide](../operations/claude-code-compatibility.md) for `advisor-tool-2026-03-01` header handling.
