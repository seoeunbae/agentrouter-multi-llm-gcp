# Overview

A multi-LLM serving platform on Google Kubernetes Engine (GKE) combining [Agentrouter (formerly Envoy AI Gateway)](https://github.com/theagentrouter/agent-router) (`v1.1.0`), Kubernetes Gateway API Inference Extension (GIE `v1.6.0`), llm-d-router (`EPP v0.10.0`), vLLM, Google Cloud Model Armor, Cloud DLP, and Vertex AI.

***

## Documentation Navigation

### Getting Started
* [Quickstart & Deployment](getting-started/quickstart.md): Deploy GKE, L4 GPUs, Cloud SQL, GCS, and the gateway stack with `make deploy`.
* [Customer Workshop Guide](getting-started/workshop-guide.md): Hands-on guide covering IAM, GPU quotas, 3-tier auth, EPP prefix cache, Model Armor, and Phoenix tracing.

### Architecture & Design
* [Gateway & Resource Hierarchy](architecture-and-design/gateway-resources.md): Kubernetes Gateway API resource hierarchy and policy attachments.
* [End-to-End Request Flow](architecture-and-design/request-flow.md): Sequence diagrams for employee, microservice, and partner requests.
* [Project Specification](architecture-and-design/project-spec.md): Feature inventory, milestone verification matrix, and component interfaces.

### Verification & Guides
* [Manual Testing Guide](verification-and-guides/manual-test-guide.md): Manual verification steps and test commands by scenario.
* [Claude Code & Vertex AI Compatibility](verification-and-guides/claude-code-compatibility.md): Cause and gateway fix for the `advisor-tool-2026-03-01` beta header error on Vertex AI.

***

## 1. Architecture Overview

Provides a single entry point for managed models (Vertex AI Gemini, Anthropic Claude) and self-hosted models (vLLM Gemma 2B).

* Infrastructure: GKE Standard cluster, 2x NVIDIA L4 GPU Spot node pools (`g2-standard-8`), Cloud SQL PostgreSQL 16, and Cloud Storage bucket.
* Model Serving: Mounts model weights via Cloud Storage FUSE and runs vLLM `v0.29.0` with PagedAttention V1 Prefix Caching.
* Routing: [Agentrouter](https://github.com/theagentrouter/agent-router) parses the request `model` field to route traffic, working with GIE `InferencePool` and `llm-d-router` (EPP) prefix scorers to select cache-warm GPU pods.
* Security & Auth: Handles employee GCIP JWT, internal Google SA ID tokens, and partner API keys at the gateway, with Model Armor and Cloud DLP guardrails (`08-model-armor`).
* Traffic Control: Enforces token rate limits (`BackendTrafficPolicy`) and per-partner quotas (`QuotaPolicy`) using Redis counters.
* Observability: Stores OTLP traces in Cloud SQL via [Arize Phoenix](https://github.com/Arize-ai/phoenix) and collects GPU and cache metrics in Google Cloud Monitoring.

### 1.1 Reference Architectures

#### Reference Architecture: AgentRouter Multi-LLM on GCP

![Reference Architecture: AgentRouter Multi-LLM on GCP](assets/ref-arch-agentrouter-multi-llm.png)

#### Reference Architecture: Claude Apps Gateway on GCP

![Reference Architecture: Claude Apps Gateway on GCP](assets/ref-arch-claude-apps-gateway.png)

#### Reference Architecture: Claude Code with IdP (Okta)

![Reference Architecture: Claude Code with IdP (Okta)](assets/ref-arch-claude-code-idp.png)

#### Reference Architecture: AI Gateway with Apigee, Model Armor

![Reference Architecture: AI Gateway with Apigee, Model Armor](assets/architecture.png)

See the following documents for details:

* [Gateway & Resource Hierarchy](architecture-and-design/gateway-resources.md)
* [End-to-End Request Flow](architecture-and-design/request-flow.md)
* [Project Specification](architecture-and-design/project-spec.md)

***

## 2. Routing Paths & Supported Models

Requests to `http://<GATEWAY_IP>:8080/v1/chat/completions` are routed by the `model` field:

| Model ID | Serving Backend | Schema | Description |
| --- | --- | --- | --- |
| `gemini-2.5-flash` | Google Cloud Vertex AI | `GCPVertexAI` | Calls Vertex AI using gateway GCP credentials |
| `claude-sonnet-5` | Google Cloud Vertex AI | `GCPAnthropic` | Supports OpenAI format and Anthropic Messages API |
| `gemma-rr` | GKE Self-Hosted GPU | `OpenAI` | L4 round-robin via standard K8s Service |
| `gemma-epp` | GKE Self-Hosted GPU | `OpenAI` | Routes to cached pod via GIE EPP prefix-cache scorer |
| `gemma-epp-noprefix` | GKE Self-Hosted GPU | `OpenAI` | Routes by load via GIE EPP queue scorer |

***

## 3. Multi-Tier Authentication

Supports three authentication methods based on client type:

### 3.1 Corporate Employees & Cloud Workstation (GCIP JWT)

* Validates JWT signatures against Google JWKS endpoints.
* Extracts the `department` claim into the `x-tenant-id` header.
* Overwrites client-supplied tenant headers to prevent spoofing.
* Works directly with Claude Code CLI on Cloud Workstations.

### 3.2 Internal Microservices (Google SA ID Token)

* Verifies Workload Identity tokens issued by the GKE metadata server.
* Sets the `email` claim as `x-tenant-id`.

### 3.3 External Partners (API Key & Quota Isolation)

* Applies a dedicated `SecurityPolicy` on `partner.agent-router.internal`.
* Maps API keys to client IDs, sets `x-tenant-id`, and strips `X-API-Key` before forwarding.
* Allows only `gemma-rr` and blocks unpermitted models (`claude-sonnet-5`) with `HTTP 404`.
* Uses the Key Issuer service to issue new partner keys and sync them to K8s Secrets.

Header injection and spoofing protection can be verified via `/authtest`.

***

## 4. Traffic Control & Quota Policies

* Token Rate Limiting (`BackendTrafficPolicy`):
  * Limits `claude-sonnet-5` to 500,000 tokens per minute.
  * Tracks `llm_total_token` response metadata in Redis.
* Tenant Quota Isolation (`QuotaPolicy`):
  * Assigns separate token budgets per partner (`acme-corp`: 60 tokens/min, `globex`: 500 tokens/min).
  * When `acme-corp` is rate-limited (`HTTP 429`), `globex` continues normally (`HTTP 200`).

***

## 5. Benchmark Results

Measured on GKE Standard with 2x NVIDIA L4 GPUs, vLLM `v0.29.0` (`google/gemma-2-2b-it`), and 16 persona prompts (~2,000 tokens each). vLLM pods were restarted before each run to clear caches.

### 5.1 Latency & Cache Hit Summary

| Routing Configuration | Routing Setup | Cold TTFT P50 | Eval TTFT P50 | Eval P95 | Eval P99 | Prefix Cache Hits (Tokens) |
| --- | --- | --- | --- | --- | --- | --- |
| `gemma-epp` | InferencePool + EPP Prefix Scorer ON | 0.773s | 0.139s | 0.178s | 0.191s | 25,184 |
| `gemma-rr` | K8s Service L4 Round-Robin | 1.351s | 0.613s | 1.829s | 1.877s | 10,560 |
| `gemma-epp-noprefix` | InferencePool + EPP Queue Scorer | 0.807s | 0.251s | 0.590s | 0.644s | 15,520 |

### 5.2 Summary

* Cache Speedup: `gemma-epp` reduced Eval TTFT P50 by 5.55x (`0.139s` vs Cold `0.773s`).
* Prefix Scorer Gain: Reduced latency by 44.6% (`0.112s`) compared to queue-only EPP (`gemma-epp-noprefix`).
* Cache Balance: Round-robin (`gemma-rr`) skewed cache hits to a single pod, raising P99 latency to `1.877s`. `gemma-epp` balanced hits across both pods (`49.8%` each) and reused 25,184 tokens (2.38x higher than round-robin).
* Auth Overhead: Gateway JWT validation added under 1ms overhead, keeping Eval P50 at `0.139s`.

***

## 6. Observability

* [Arize Phoenix](https://github.com/Arize-ai/phoenix): Connects to Cloud SQL PostgreSQL 16 via Auth Proxy and stores OpenInference traces viewable on `:6006`.
* Google Cloud Monitoring: Scrapes vLLM metrics (`:8000/metrics`) via `PodMonitoring` for prefix cache hit rate and TTFT dashboards.
