# Inference Request Flow

[← Docs index](../README.md)

End-to-end path for an inference request, from external client to model response.
`eg-gateway` is the platform's edge proxy (Envoy Gateway, with the external IP);
`ai-gateway` is the internal proxy (Envoy AI Gateway) that sends each request to its model.
The **EPP** (endpoint picker) is a KServe service that picks the best pod of a model for each request.

## Architecture

Two auth modes are supported. The flow differs based on `auth_type` in `env/<env>/config.inference.yaml`. Keycloak is deployed in both modes for sign-in; `auth_type` only selects the API client credential. With `routing_mode: path` the hosts below become `https://<domain>/inference/v1/...` and `https://<domain>/litellm/v1/...`. On OpenShift neither AI gateway is installed: models are published through the OpenShift AI inference Gateway (see [Deploy on OpenShift](../deployment/openshift.md)).

### `jwt` (default) — Unified Routing

```
                         EXTERNAL                          INTERNAL (cluster)

 Client ──HTTPS──► eg-gateway ──HTTP──► ai-gateway ──► Model (vLLM / OVMS)
                   (TLS term)           (EPP or direct)
                   (JWT auth via
                    Keycloak OIDC)
```

Single endpoint: `https://inference.<domain>/v1/chat/completions`
Model selected by `"model": "<name>"` in the JSON body (Envoy AI Gateway extracts it automatically).

### `litellm_vkey` mode

```
                         EXTERNAL                          INTERNAL (cluster)

 Client ──HTTPS──► eg-gateway ──HTTP──► LiteLLM ──HTTP──► ai-gateway ──► vLLM
                   (TLS term)           (auth,guardrail,   (EPP sched)   (model)
                                         cache)
                                            │
                                            ├──► ai-gateway ──► guard model (content-safety pre-call screen)
                                            ├──► Valkey (response cache)
                                            └──► Langfuse (traces)
```

## Two Gateways, Two Concerns

| Gateway | Type | Purpose | GatewayClass |
|---------|------|---------|--------------|
| `eg-gateway` | LoadBalancer (external IP) | TLS termination, hostname routing, JWT validation. `Gateway/eg-gateway` in `envoy-gateway-system` | `eg` (`enterprise-ai` if another controller already owns `eg`) |
| `ai-gateway` | ClusterIP (internal only) | Model routing via header match, EPP scheduling or direct routing. `Gateway/ai-gateway` in `llm-inference`; its Service `ai-gateway` (port 80) runs in `envoy-gateway-system`, beside the Envoy Gateway controller | `inference-pool-with-aigwroute` |

`eg-gateway` carries a deny-by-default Keycloak JWT policy (`SecurityPolicy/gateway-auth`). The LiteLLM API route in `litellm_vkey` mode attaches its own no-auth `SecurityPolicy` (`litellm-no-auth`), because LiteLLM checks the virtual key itself. Web UIs such as Langfuse get a Keycloak sign-in policy instead.

In `jwt` mode, `eg-gateway` validates the JWT and forwards directly to `ai-gateway`. No LiteLLM in the path. `inference.<domain>` is published only in this mode, so nothing can bypass LiteLLM in `litellm_vkey` mode.
In `litellm_vkey` mode, `eg-gateway` routes to LiteLLM (which validates the virtual key and, with the content-safety guardrail on, screens the request), and LiteLLM forwards to `ai-gateway`.

## Request Flow — `jwt` + Unified Routing (Default)

### Example: `qwen3-0-6b` (LLMInferenceService, EPP routing)

---

### Step 1: Client → eg-gateway (TLS + JWT Validation)

```
curl --cacert env/<env>/logs/ai-solutions-ca.crt \
  https://inference.inteltoolkits.ai/v1/chat/completions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"model": "qwen3-0-6b", "messages": [{"role":"user","content":"Hello"}]}'
```

- Client connects to `192.168.120.103:443` (eg-gateway LoadBalancer; the IP is an example)
- TLS terminated at the gateway, on the host's own listener: by default a ListenerSet `inference` with its own certificate (the wildcard `*.inteltoolkits.ai` listener only with `gateway_attach_mode: shared`)
- Hostname `inference.inteltoolkits.ai` matched by HTTPRoute (in path mode: `<domain>/inference`, prefix stripped)
- SecurityPolicy validates the JWT against Keycloak's signing keys
- Missing, invalid or expired token → HTTP 401 (request stops here)

**Kubernetes resources:**
- `Gateway/eg-gateway` in `envoy-gateway-system`
- `HTTPRoute/inference-external` in `envoy-gateway-system` → backend `ai-gateway:80`
- `SecurityPolicy/gateway-auth` on `eg-gateway` (deny-by-default JWT validation against Keycloak)

---

### Step 2: eg-gateway → ai-gateway (Model Routing)

- Request forwarded to ai-gateway ClusterIP service (port 80)
- Envoy AI Gateway extracts `"model": "qwen3-0-6b"` from the JSON body
- Sets header `x-ai-eg-model: qwen3-0-6b` internally
- The model's route (created by model-manager at deploy time) matches the header and selects the backend

**Routing modes** (`--routing` on deploy):

| Mode | Backend | When |
|------|---------|------|
| `epp` | `AIGatewayRoute/aigw-<model>` → `InferencePool/<model>-inference-pool` → EPP scheduler picks optimal pod | LLMs (default) |
| `direct` | `HTTPRoute/direct-<model>` → `Service/<model>-kserve-workload-svc:8000` | Embedding and rerank models (default), or explicit `routing: direct` |
| `aigateway` | `AIGatewayRoute/aigw-<model>` → `AIServiceBackend/<model>` → the workload Service | Only when asked for |

With `ai_gateway_provider: agentgateway` the routes are plain `HTTPRoute`s with the same backends.

**Kubernetes resources:**
- `Gateway/ai-gateway` in `llm-inference` (GatewayClass `inference-pool-with-aigwroute`)
- `AIGatewayRoute/aigw-<model>` in the model's namespace (header-based match, no hostname)

---

### Step 3a: EPP Routing — ai-gateway → EPP Scheduler → vLLM

- Envoy's `ext_proc` filter invokes the model's EPP service
- EPP evaluates all available replicas: queue depth, KV cache utilization, load
- Returns the selected pod IP to Envoy
- Envoy forwards to `<pod-ip>:8000`

**Kubernetes resources** (KServe creates them from the `LLMInferenceService`):
- `InferencePool/<model>-inference-pool`, whose endpoint picker is the model's EPP service

### Step 3b: Direct Routing — ai-gateway → workload service

- `HTTPRoute/direct-<model>` matches the model header and forwards to `<model>-kserve-workload-svc:8000`
- No EPP involved — the default for embedding and rerank models (vLLM and OpenVINO LLMs both go through EPP)

---

### Step 4: Response Path

```
Model → ai-gateway → eg-gateway → Client
```

No caching or Langfuse tracing in this path. Add LiteLLM with `litellm_enabled: true` for them: LiteLLM is then published on `litellm.<domain>` behind the same gateway JWT check, and `inference.<domain>` stays the main endpoint.

---

## Request Flow — `litellm_vkey` Mode

### Example: `llama3-8b-awq` model, deployed via KServe LLMInferenceService

---

### Step 1: Client → eg-gateway (TLS Termination)

```
curl --cacert env/<env>/logs/ai-solutions-ca.crt \
  https://litellm.inteltoolkits.ai/v1/chat/completions \
  -H "Authorization: Bearer sk-<virtual-key>" \
  -d '{"model": "llama3-8b-awq", "messages": [...]}'
```

- Client connects to `192.168.120.103:443` (eg-gateway LoadBalancer)
- TLS terminated at the gateway, on the host's own listener: by default a ListenerSet `litellm` (namespace `litellm`) with its own certificate
- Hostname `litellm.inteltoolkits.ai` matched by HTTPRoute. The route publishes only these exact paths (`litellm_public_paths`): `/v1/chat/completions`, `/v1/completions`, `/v1/embeddings`, `/v1/models`, `/health/liveliness`. Any other path, such as the admin API, gets 404

**Kubernetes resources:**
- `Gateway/eg-gateway` in `envoy-gateway-system` (listeners: `http`/80 redirect, `https-apex`, the wildcard `https` when enabled); the route attaches to the `litellm` ListenerSet
- `HTTPRoute/litellm` in `litellm` namespace → backend `litellm:4000`
- `SecurityPolicy/litellm-no-auth` (overrides the gateway JWT policy; LiteLLM checks the key)
- `litellm-rate-limit` (per client IP, `litellm_rate_limit_enabled`, default on)

---

### Step 2: eg-gateway → LiteLLM (Virtual Key Auth)

- Request forwarded to the `litellm` Service on port 4000
- LiteLLM validates the virtual key (`sk-<key>`) against its database
- Invalid key → HTTP 401 (request stops here)
- Valid key → spend tracking, rate limit check, proceed

**What LiteLLM does:**
1. Authenticates the virtual key
2. Screens the last user message with the content-safety guardrail (`guardrails_enabled`,
   on by default in this mode): a pre-call request through `ai-gateway` to the guard model
   (`qwen3guard`). Unsafe content returns HTTP 500 "Blocked by the content-safety
   guardrail: …" and the request stops here. The guardrail is fail-closed: while the guard
   model cannot be reached, every request is blocked. See
   [Guardrails](../configuration/guardrails.md).
3. Checks the Valkey cache (`litellm_cache_enabled`, default `true`; hit → return cached response, skip steps 4-6)
4. Resolves model name to backend `api_base` (from its database: `store_model_in_db: true`)
5. Forwards to the model endpoint
6. On response: writes to Langfuse (async), caches in Valkey

---

### Step 3: LiteLLM → ai-gateway (EPP Routing)

LiteLLM calls the registered `api_base`:
```
POST http://ai-gateway.envoy-gateway-system.svc.cluster.local
     /llm-inference/llama3-8b-awq/v1/chat/completions
```

- Request arrives at the ai-gateway Service (ClusterIP, port 80)
- Path `/llm-inference/llama3-8b-awq/v1/chat/completions` matches the HTTPRoute KServe created for the model
- Backend type is `InferencePool` (not a plain Service)

**Kubernetes resources:**
- `Gateway/ai-gateway` in `llm-inference` (GatewayClass: `inference-pool-with-aigwroute`)
- The KServe-generated `HTTPRoute` for `llama3-8b-awq` in `llm-inference`
- `InferencePool/llama3-8b-awq-inference-pool` in `llm-inference`

---

### Step 4: ai-gateway → EPP Scheduler (ext_proc)

- Envoy's `ext_proc` filter invokes the model's EPP service
- EPP scheduler evaluates all available model replicas
- Picks the best endpoint based on: queue depth, KV cache utilization, load
- Returns the selected pod IP to the Envoy proxy

**Kubernetes resources:**
- `InferencePool` spec: its endpoint picker is the model's EPP service
- EPP runs in the model's KServe router-scheduler pod

---

### Step 5: ai-gateway → vLLM Pod (Inference)

- Envoy rewrites path: `/llm-inference/llama3-8b-awq/v1/chat/completions` → `/v1/chat/completions`
- Routes to the EPP-selected pod on port 8000
- vLLM processes the request and returns the completion

**Kubernetes resources:**
- `Deployment/llama3-8b-awq-kserve` (vLLM pods with model loaded from PVC)
- `Service/llama3-8b-awq-kserve-workload-svc` (headless, for endpoint discovery)

---

### Step 6: Response Path (reverse)

```
vLLM → ai-gateway (EPP logs response metrics) → LiteLLM → eg-gateway → Client
```

LiteLLM on response:
- Caches in Valkey (`allkeys-lru`, TTL=600s) for identical future requests
- Sends trace to Langfuse (async callback, with `langfuse_enabled`, default `true`): input, output, latency, token usage, model

---

## Verified Network Trace

Request correlation from the logs of a test cluster (`litellm_vkey`; IP addresses are examples):

```
Pod IPs:
  LiteLLM:     10.233.104.75
  ai-gateway:  10.233.104.102
  EPP router:  10.233.104.80
  vLLM:        10.233.104.79

ai-gateway access log:
  downstream_remote: 10.233.104.75 (LiteLLM)        ← confirms LiteLLM is the caller
  route: rule/1 (InferencePool-backed)               ← EPP route, not catch-all
  upstream_host: 10.233.104.79:8000 (vLLM)           ← EPP picked this pod
  path rewritten: /v1/chat/completions               ← prefix stripped
  x-request-id: b918d98a-...                         ← correlates across all logs

EPP scheduler log:
  "EPP received request" x-request-id=b918d98a-...   ← same request
  "EPP sent request body response(s) to proxy"       ← picked target pod
  "EPP sent response body back to proxy"             ← processed response

Langfuse trace:
  timestamp: 2026-06-10T13:58:52Z
  input: "What color is the sky? One word."
  output: "Blue."
  latency: 0.646s
```

---

## Cache Behavior

| Request | Path | Latency |
|---------|------|---------|
| First call | LiteLLM → ai-gateway → EPP → vLLM | ~2s (example) |
| Repeat (cached) | LiteLLM → Valkey (hit) | ~0.06s (example) |

TTL: `litellm_cache_ttl` (default 600 seconds). Eviction: `allkeys-lru`.

---

## Auto-Registration (Model Manager)

When a model is deployed via `model-manager deploy`:

1. model-manager applies the `LLMInferenceService` with `router.gateway.refs → ai-gateway` (namespace `llm-inference`)
2. model-manager creates the model's ai-gateway route (`aigw-<model>` or `direct-<model>`)
3. The KServe controller creates the workload, the InferencePool, the EPP and an HTTPRoute on ai-gateway
4. model-manager waits for the `Ready` condition: in the command with `--wait`, otherwise in a background process (log: `/tmp/model-manager-reconcile-<model>.log`)
5. On Kubernetes it marks the InferencePool `Accepted` (on OpenShift the gateway controller does it)
6. With `auth_type: litellm_vkey`, registers the model in LiteLLM (`POST /model/new`) with `api_base = http://ai-gateway.envoy-gateway-system.svc.cluster.local/<ns>/<model>/v1`

Run as the standalone `./model-manager` CLI (no `MM_LITELLM_KSERVE_URL`), it falls back to the direct workload service URL and warns when the `ai-gateway` Service does not exist; install-time and Headlamp deploys always set the URL.

---

## Key Configuration Files

| File | Purpose |
|------|---------|
| `roles/envoy_ai_gateway/tasks/install.yaml` | `Gateway/ai-gateway`, its Service, and `HTTPRoute/inference-external` |
| `roles/litellm/tasks/install.yaml` | Deploys LiteLLM + Valkey subchart via Helm; `HTTPRoute/litellm` and its policies |
| `roles/litellm/defaults/main.yaml` | Topology-aware sizing, `litellm_public_paths`, cache config, guardrail settings |
| `roles/litellm/tasks/register_guardrail.yaml` | Registers the content-safety guardrail with LiteLLM |
| `charts/litellm-helm/` | Vendored chart with bitnami/valkey subchart |
| `model_manager/core/routing.py` | Per-model routes, InferencePool status, auto-registration with LiteLLM API, printed endpoints |
| `model_manager/templates/llm-inference-service.yaml` | LLMInferenceService manifest (gateway ref) |

---

## Failure Modes

| Failure | Behavior |
|---------|----------|
| Missing or invalid JWT (`jwt` mode) | Rejected at eg-gateway (401), never reaches model |
| Invalid virtual key | Rejected at LiteLLM (401), never reaches model |
| Path not in `litellm_public_paths` | 404 from eg-gateway |
| Unsafe prompt (guardrail on) | Blocked at LiteLLM (500, "Blocked by the content-safety guardrail"), never reaches model |
| Guard model down or not Ready (guardrail on) | Every request blocked (500) — the guardrail is fail-closed |
| Model not registered in LiteLLM | LiteLLM returns 404 model not found |
| ai-gateway down | LiteLLM returns 503, no inference |
| EPP scheduler down | Depends on the InferencePool's failure mode, which KServe sets |
| vLLM pod crash | EPP stops selecting that pod (health-based routing) |
| Langfuse down | Inference still works, traces dropped silently |
| Valkey down | Inference still works, no caching (higher latency) |
