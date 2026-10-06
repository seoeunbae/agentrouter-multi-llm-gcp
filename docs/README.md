# Overview

An enterprise multi-LLM serving platform combining [**Agentrouter (formerly Envoy AI Gateway)**](https://github.com/theagentrouter/agent-router) (`v1.1.0`), **Kubernetes Gateway API Inference Extension (GIE `v1.6.0`)**, **llm-d-router (`EPP v0.10.0`)**, **vLLM**, **Google Cloud Model Armor & Cloud DLP**, and **Vertex AI** on **Google Kubernetes Engine (GKE)**.

***

## Documentation Navigation

| Section                   | Guide                                                                                         | Summary                                                                                                                      |
| ------------------------- | --------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| **Getting Started**       | [Quickstart & Deployment](getting-started/quickstart.md)                                      | Provision GKE, L4 GPUs, Cloud SQL, GCS, and deploy the full AI gateway stack with `make deploy`.                             |
| **Getting Started**       | [Customer Workshop Guide](getting-started/workshop-guide.md)                                  | Step-by-step hands-on walkthrough covering IAM, GPU quotas, 3-tier auth, EPP prefix cache, Model Armor, and Phoenix tracing. |
| **Architecture & Design** | [Gateway & Resource Hierarchy](architecture-and-design/gateway-resources.md)                  | Explore Kubernetes Gateway API resource hierarchies and routing policy attachments.                                          |
| **Architecture & Design** | [End-to-End Request Flow](architecture-and-design/request-flow.md)                            | Sequence diagrams across employee, microservice, and external partner personas.                                              |
| **Architecture & Design** | [Project Specification](architecture-and-design/project-spec.md)                              | Full feature inventory, milestone verification matrix, and interface contracts.                                              |
| **Verification & Guides** | [Manual Testing Guide](verification-and-guides/manual-test-guide.md)                          | Scenario-by-scenario manual test procedures and verification commands.                                                       |
| **Verification & Guides** | [Claude Code & Vertex AI Compatibility](verification-and-guides/claude-code-compatibility.md) | Root-cause analysis and gateway solutions for `advisor-tool-2026-03-01` beta headers on Vertex AI.                           |

***

## 1. Architectural Overview

This platform provides a unified entry point bridging public cloud managed models (**Vertex AI Gemini**, **Anthropic Claude**) and self-hosted models (**vLLM Gemma 2B**).

* **Infrastructure Layer**: GKE Standard cluster, 2x NVIDIA L4 GPU Spot node pools (`g2-standard-8`), Cloud SQL PostgreSQL 16 instance, and Cloud Storage bucket.
* **Model Serving Layer**: Cloud Storage FUSE mounts model weights directly to the container. Powered by vLLM `v0.29.0` with PagedAttention V1 Prefix Caching enabled.
* **Intelligent Routing Layer**: [Agentrouter (formerly Envoy AI Gateway)](https://github.com/theagentrouter/agent-router) buffers the JSON request body and parses the `model` field to route requests. Integrated with Kubernetes GIE `InferencePool` and `llm-d-router` (EPP) prefix scorers to route traffic to cache-friendly GPU pods.
* **Multi-Tier Security & Auth**: Gateway-level unified authentication supporting employee GCIP JWT, internal service Google SA ID tokens, and external partner API keys, plus Google Cloud Model Armor & Sensitive Data Protection (Cloud DLP) guardrails (`08-model-armor`).
* **Traffic Policy Layer**: Dual rate limiting and quota management backed by distributed Redis counters (`BackendTrafficPolicy` and `QuotaPolicy`).
* **Observability Layer**: [Arize Phoenix](https://github.com/Arize-ai/phoenix) persisting OTLP traces into Cloud SQL PostgreSQL, complemented by Google Cloud Monitoring collecting GPU and prefix cache metrics.

### 1.1 Reference Architectures

#### Reference Architecture: AgentRouter Multi-LLM on GCP

![Reference Architecture: AgentRouter Multi-LLM on GCP](assets/ref-arch-agentrouter-multi-llm.png)

#### Reference Architecture: Claude Apps Gateway on GCP

![Reference Architecture: Claude Apps Gateway on GCP](assets/ref-arch-claude-apps-gateway.png)

#### Reference Architecture: Claude Code with IdP (Okta)

![Reference Architecture: Claude Code with IdP (Okta)](assets/ref-arch-claude-code-idp.png)

For detailed architecture diagrams and workflows:

* [Gateway & Resource Hierarchy Diagram](architecture-and-design/gateway-resources.md)
* [End-to-End Request Flow Sequence Diagram](architecture-and-design/request-flow.md)
* [Project Specification & Feature Inventory](architecture-and-design/project-spec.md)

***

## 2. Routing Paths & Supported Models

Incoming requests to `http://<GATEWAY_IP>:8080/v1/chat/completions` are routed according to the `model` parameter:

| Model ID             | Serving Backend        | Schema         | Description                                            |
| -------------------- | ---------------------- | -------------- | ------------------------------------------------------ |
| `gemini-2.5-flash`   | Google Cloud Vertex AI | `GCPVertexAI`  | Gateway delegates call via GCP credentials             |
| `claude-sonnet-5`    | Google Cloud Vertex AI | `GCPAnthropic` | Supports both OpenAI format and Anthropic Messages API |
| `gemma-rr`           | GKE Self-Hosted GPU    | `OpenAI`       | Pure L4 round-robin via standard K8s Service           |
| `gemma-epp`          | GKE Self-Hosted GPU    | `OpenAI`       | GIE EPP prefix-cache scorer routes to cached pod       |
| `gemma-epp-noprefix` | GKE Self-Hosted GPU    | `OpenAI`       | GIE EPP queue scorer distributes based on load         |

***

## 3. 3-Tier Multi-Authentication Architecture

To address varied client environments across enterprise boundaries, three authentication gates are provided:

### 3.1 Corporate Employees & Cloud Workstation (GCIP JWT)

* Validates JWT tokens against Google JWKS endpoints.
* Extracts the `department` claim and injects it into `x-tenant-id` header.
* Overwrites any client-supplied tenant headers to prevent identity spoofing.
* Seamlessly integrates with Claude Code CLI running on Cloud Workstations.

### 3.2 Internal Microservices (Google SA ID Token)

* Authenticates in-cluster workloads using Workload Identity tokens obtained from the GKE metadata server without static keys.
* Injects the `email` claim as `x-tenant-id`.

### 3.3 External Partners (API Key & Quota Isolation)

* Dedicated listener host `partner.agent-router.internal` applies a route-level `SecurityPolicy` overriding the gateway JWT default.
* Maps API keys to client IDs, injects `x-tenant-id`, and strips the `X-API-Key` header before forwarding.
* Restricts model access to `gemma-rr`, blocking expensive models like `claude-sonnet-5` with `HTTP 404 Route Not Found`.
* A built-in Key Issuer service dynamically provisions new partner keys and syncs them to K8s Secrets.

Anti-spoofing header injection can be tested via the `/authtest` echo endpoint.

***

## 4. Traffic Control & Quota Policies

Enforces tenant cost control and fair resource allocation in enterprise settings:

* **Token Rate Limiting (`BackendTrafficPolicy`)**:
  * Limits `claude-sonnet-5` calls to 500,000 tokens per minute.
  * Ingests `llm_total_token` metadata from response headers into Redis.
* **Tenant Quota Isolation (`QuotaPolicy`)**:
  * Allocates distinct token budgets for partner organizations.
  * `acme-corp` receives 60 tokens/min; `globex` receives 500 tokens/min.
  * When `acme-corp` is throttled with `HTTP 429`, `globex` requests continue to succeed with `HTTP 200`.

***

## 5. 3-Way Comparative Benchmark Results

Measured in GKE Standard on 2x NVIDIA L4 GPUs using vLLM `v0.29.0` (`google/gemma-2-2b-it`) and 16 independent persona prompts (\~2,000 tokens each). Caches were flushed before evaluating each routing configuration by restarting vLLM pods.

### 5.1 Latency & Prefix Cache Hit Statistics

| Routing Configuration    | Routing Setup                        | Cold TTFT P50 | Eval TTFT P50 | Eval P95   | Eval P99   | Prefix Cache Hits (Tokens) |
| ------------------------ | ------------------------------------ | ------------- | ------------- | ---------- | ---------- | -------------------------- |
| **`gemma-epp`**          | InferencePool + EPP Prefix Scorer ON | 0.773s        | **0.139s**    | **0.178s** | **0.191s** | **25,184**                 |
| **`gemma-rr`**           | K8s Service Pure L4 Round-Robin      | 1.351s        | 0.613s        | 1.829s     | 1.877s     | 10,560                     |
| **`gemma-epp-noprefix`** | InferencePool + EPP Queue Scorer     | 0.807s        | 0.251s        | 0.590s     | 0.644s     | 15,520                     |

### 5.2 Key Findings

* **Cache Acceleration**: `gemma-epp` achieved a **5.55x TTFT speedup** (Eval `0.139s` vs Cold `0.773s`).
* **Prefix Scorer Net Gain**: **44.6% latency reduction** (`+0.112s` faster) compared to queue-based EPP (`gemma-epp-noprefix`).
* **Balanced Cache Affinity**: Pure round-robin suffered from cache imbalance (one pod received all hits, the other 0), causing tail latency (`Eval P99: 1.877s`) to spike. `gemma-epp` distributed hits evenly (`49.8%` hit rate per pod) and delivered 25,184 cached tokens (**2.38x higher** than round-robin).
* **Zero Auth Overhead**: Gateway JWT authentication added under 1ms overhead, with Eval P50 holding steady at `0.139s`.

***

## 6. Enterprise Observability

* [**Arize Phoenix**](https://github.com/Arize-ai/phoenix): Connected to Cloud SQL PostgreSQL 16 via Auth Proxy, persisting OpenInference traces with input/output tokens and latency stages visible on `:6006`.
* **Google Cloud Monitoring**: Scrapes vLLM Prometheus metrics (`:8000/metrics`) via `PodMonitoring`, providing real-time dashboards for prefix cache hit rates and TTFT curves.
