# Project Specification

## 1. Architecture Overview

* Infrastructure: GCP Project `<YOUR_PROJECT_ID>` (`asia-southeast1`), GKE Standard cluster with Gateway API and GCS FUSE CSI driver enabled, 2x L4 GPU Spot node pools (`g2-standard-8`), Cloud SQL PostgreSQL 16, and GCS bucket for model weights.
* Model Storage & Serving: Loads model weights from Hugging Face into GCS and mounts them to vLLM `0.29.0` pods via GCS FUSE (`gke-gcsfuse/volumes: "true"`), with PagedAttention V1 Prefix Caching enabled.
* Routing & Networking: Envoy AI Gateway ([Agentrouter](https://github.com/theagentrouter/agent-router) `1.1.0`) buffers JSON request bodies and routes by the `model` key. Internal routes use `InferencePool` (`inference.networking.k8s.io/v1`) with `llm-d-router` (`0.10.0`) scoring pods by prefix cache state. Gemini and Claude routes call Vertex AI using GCP credentials.
* Security & Multi-Tenancy: Unified authentication via Envoy Gateway `SecurityPolicy` supporting GCIP JWT for employees, Google SA ID tokens for internal services, and API keys for external partners with isolated model routing.
* Traffic Policy & Quotas: Redis-backed token rate limiting (`BackendTrafficPolicy`) and per-tenant token quota isolation (`QuotaPolicy`).
* Observability: [Arize Phoenix](https://github.com/Arize-ai/phoenix) connected to Cloud SQL via Auth Proxy sidecar for OTLP trace storage, plus `PodMonitoring` scraping vLLM `/metrics`.
* Validation: In-cluster benchmark job testing 3 routing configurations in randomized order with vLLM cache resets between runs.

***

## 2. Feature Inventory

| # | Feature | Description | Milestone |
| --- | --- | --- | --- |
| 1 | GCP Terraform VPC/GKE | Deploy GKE with Gateway API, Workload Identity, GCS FUSE | M1 (Completed) |
| 2 | GCP Terraform DB/Storage | Deploy Cloud SQL PG 16 and GCS Bucket | M1 (Completed) |
| 3 | GCP Terraform NodePool | Deploy 2x L4 Spot NodePool (`g2-standard-8`) | M1 (Completed) |
| 4 | GCP Terraform IAM | Service Accounts and KSA/GSA Workload Identity bindings | M1 (Completed) |
| 5 | Model Weight Loader | Job to pull `google/gemma-2-2b-it` from HF to GCS bucket | M2 (Completed) |
| 6 | vLLM Workload | Deploy vLLM `0.29.0` pods with GCS FUSE mount and Prefix Caching | M2 (Completed) |
| 7 | Phoenix Tracing Server | Deploy Arize Phoenix UI with Cloud SQL Auth Proxy sidecar | M3 (Completed) |
| 8 | K8s GIE & InferencePools | Define InferencePools for round-robin and EPP targets | M3 (Completed) |
| 9 | llm-d EPP Router | Deploy `llm-d-router` v0.10.0 with `prefix-cache` and `queue` scorers | M3 (Completed) |
| 10 | Agent Router Hybrid Mesh | Deploy Envoy AI Gateway (`1.1.0`) and `AIGatewayRoute` | M3 (Completed) |
| 11 | 3-Tier Multi-Authentication | GCIP JWT, Google SA ID Token, and Partner API Key auth with anti-spoofing | M3 (Completed) |
| 12 | Redis Traffic & Quota Policy | Rate limiting (`BackendTrafficPolicy`) and tenant quotas (`QuotaPolicy`) via Redis | M3 (Completed) |
| 13 | E2E Benchmark Suite | Benchmark job measuring TTFT and cache hits across 3 routing setups | E2E (Completed) |
| 14 | Teardown Automation | Makefile targets for automated provisioning and cleanup | M4 (Completed) |
| 15 | Bilingual Documentation | English and Korean documentation | Final (Completed) |

***

## 3. Milestones

| # | Name | Scope | Dependencies | Status |
| --- | --- | --- | --- | --- |
| M1 | Infrastructure (Terraform) | GCP resources (GKE, NodePools, Cloud SQL, GCS, IAM) | none | DONE |
| M2 | Models & Serving (vLLM) | HF weight loader, GCS FUSE mount, vLLM pods | M1 | DONE |
| M3 | Routing & Security | GIE, llm-d EPP, Agent Router, Phoenix, 3-tier auth, Redis quotas | M2 | DONE |
| M4 | Automation & Packaging | Numbered Kustomize manifests, Makefile automation, cleanup | M3 | DONE |
| E2E | Benchmark Verification | In-cluster 3-way load tests with cache reset and metrics collection | M3 | DONE |
| Final | Documentation | English and Korean documentation and credential checks | M4, E2E | DONE |

***

## 4. Interface Contracts

### 4.1 Terraform ↔ K8s

* KSA Name and GSA Email mapped for Workload Identity
* GCS Bucket Name exported for manifest mounts
* Cloud SQL Instance Connection Name exported for Auth Proxy

### 4.2 Agent Router ↔ Backend Routes

* Envoy handles HTTP requests to `/v1/chat/completions`.
* `model: "gemini-2.5-flash"`: Routes to Google Cloud Vertex AI Gemini
* `model: "claude-sonnet-5"`: Routes to Google Cloud Vertex AI Claude
* `model: "gemma-rr"`: Routes to standard K8s Service (L4 round-robin)
* `model: "gemma-epp"`: Routes to InferencePool via EPP with prefix scorer
* `model: "gemma-epp-noprefix"`: Routes to InferencePool via EPP queue scorer

### 4.3 Multi-Tenancy & Security Contracts

* Default Host (`*`): Verifies GCIP JWT or Google SA ID Token via `agent-router-jwt` and sets `x-tenant-id`.
* Partner Host (`partner.agent-router.internal`): Verifies API keys via `partner-apikey`, sets `x-tenant-id`, and strips `X-API-Key`.
* Redis Rate Limit / Quota: Tracks `llm_total_token` response metadata against tenant token buckets.
