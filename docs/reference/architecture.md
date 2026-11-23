# Architecture

[← Docs index](../README.md)

This repository owns everything related to model inference — the Kubernetes serving
stack and the CLI that deploys and manages models at runtime.

## Components

| Component | Path | Purpose |
|-----------|------|---------|
| **KServe** | `roles/kserve/` | Model-serving platform — KServe controller, `LLMInferenceService` CRD, LWS (LeaderWorkerSet) operator |
| **Envoy AI Gateway** | `roles/envoy_ai_gateway/` | The internal `ai-gateway` — reads `"model"` from the request and routes it to that model via `AIGatewayRoute`; publishes `inference.<domain>` in `jwt` mode |
| **agentgateway** | `roles/agentgateway/` | Alternative AI gateway — native Gateway API `HTTPRoute` → `InferencePool`, plus MCP / A2A support |
| **LiteLLM** | `roles/litellm/`, `charts/litellm-helm/` | OpenAI-compatible proxy — virtual API keys, per-key budgets and rate limits, response cache (Valkey), content-safety guardrail, optional hybrid router |
| **Langfuse** | `roles/langfuse/` | LLM observability — request traces, token usage, prompt management |
| **LLM services** | `roles/llm_services/` | Provisions the inference namespace (`llm-inference`) and the model store, the Headlamp model-deploy jobs, and runs the model auto-deploy phase |
| **Keycloak config** | `roles/keycloak_config/` | Inference realm + client (a `KeycloakRealmImport`) for sign-in and JWT-based API auth (both auth modes) |
| **vLLM Semantic Router** | `roles/vllm_sr/`, `charts/vllm-sr/` | Opt-in classifier-based routing on a dedicated Envoy AI Gateway, plus its dashboard |
| **Model Manager** | `model_manager/` | CLI (`model-manager`, a Python package in `core/` behind a small Bash launcher) + catalog, runtime and manifest templates for the model lifecycle |

## Ansible roles

Each role is part of Intel® Inference Toolkit. They are registered in `components.yaml`
under `inference`, and Toolkit Manager runs them in dependency order.

| Role | Enabled when | Depends on |
|------|--------------|------------|
| `keycloak_config` | `keycloak_enabled` (default `true`; both auth modes) | — |
| `envoy_ai_gateway` | `kserve_enabled` and `ai_gateway_provider == envoy`, not on OpenShift | — |
| `agentgateway` | `ai_gateway_provider == agentgateway`, not on OpenShift | — |
| `kserve` | `kserve_enabled` (default `true`) | `envoy_ai_gateway`, `agentgateway` |
| `litellm` | `auth_type == litellm_vkey` or `litellm_enabled` | `postgresql` (platform) |
| `langfuse` | `litellm` is enabled and `langfuse_enabled` (default `true`) | `litellm`, `postgresql` |
| `llm_services` | `kserve_enabled` (default `true`) | `kserve` |
| `vllm_sr` | `vllm_sr_enabled` (default `false`) and `ai_gateway_provider == envoy` | `llm_services` |

> [!NOTE]
> `nri_cpu_balloons` is **not** here — it lives in the Toolkit Manager repo, because it
> configures kubelet-level settings on the nodes. See
> [CPU Pinning & NUMA](cpu-pinning.md).

Because the gateway roles are mutually exclusive on `ai_gateway_provider`, `kserve`
depends on both: whichever one is enabled runs first, and the other is skipped. On OpenShift
both are skipped: models are routed through the OpenShift AI inference Gateway.

## Two gateways, two concerns

| Gateway | Type | Purpose |
|---------|------|---------|
| `eg-gateway` | LoadBalancer (external IP) | TLS termination, hostname routing, JWT validation |
| `ai-gateway` | ClusterIP (internal only) | Model routing via header match, EPP scheduling or direct routing |

`eg-gateway` carries a deny-by-default Keycloak JWT policy; the LiteLLM API route in
`litellm_vkey` mode attaches its own no-auth `SecurityPolicy`, because LiteLLM checks the
virtual key itself. Web UIs such as Langfuse get a Keycloak sign-in policy instead. In **jwt** mode, `eg-gateway` validates the JWT and forwards
straight to `ai-gateway`; LiteLLM is not in the path. In **litellm_vkey** mode,
`eg-gateway` routes to LiteLLM, which validates the virtual key and then forwards to
`ai-gateway`.

```
                    EXTERNAL                            INTERNAL (cluster)

  jwt mode (default)
  Client ──HTTPS──► eg-gateway ─────────HTTP────────► ai-gateway ──► vLLM / OVMS
                    (TLS + JWT)                        (model routing)   (model)

  litellm_vkey mode
  Client ──HTTPS──► eg-gateway ──► LiteLLM ──HTTP──► ai-gateway ──► vLLM / OVMS
                    (TLS term)     (virtual keys,     (model routing)   (model)
                                    budgets, cache)
                                        │
                                        ├──► Valkey    (response cache)
                                        └──► Langfuse  (traces, token usage)
```

Step-by-step packet detail, including a verified network trace, is in
[Inference Request Flow](request-flow.md).

## Integration with Toolkit Manager

Toolkit Manager clones this repo to `ext/enterprise.ai-inference/` when you run
`./toolkit_manager.sh init inference` — the repo and branch are set in the Toolkit Manager
repo's `configs/repos/repos.inference.yaml` — and installs and operates it:

```
toolkit-manager/
├── toolkit_manager.sh            # Toolkit Manager entry point (es_auto_installer.sh: former name, still works)
├── model-manager                 # thin wrapper → ext/.../model_manager/model-manager
├── configs/repos/repos.inference.yaml  # declares this repo, and which branch to clone
├── env/<env>/global_config.yaml     # Toolkit Manager settings (base_domain_name, …)
├── env/<env>/config.inference.yaml  # inference settings (auth_type, …), seeded from config.yaml
├── env/<env>/models.yaml            # per-environment model catalog (seeded by `init`)
└── ext/
    └── enterprise.ai-inference/  # ← this repo
        ├── components.yaml       # component registry, merged into the core registry
        ├── config.yaml           # seed for env/<env>/config.inference.yaml
        ├── roles/                # discovered via roles_path
        ├── charts/               # vendored Helm charts (LiteLLM, vLLM Semantic Router)
        └── model_manager/        # the model-manager CLI + templates
```

1. `init inference` clones this repo, copies `config.yaml` to `env/<env>/config.inference.yaml`
   and `model_manager/models.yaml` to `env/<env>/models.yaml` (existing files are kept).
   At preflight, `components.yaml` is auto-discovered and merged into the core component
   registry. The Jinja in each `enabled:` expression resolves before resolution.
2. `install inference` runs the enabled roles here in dependency order.
3. `env/<env>/config.inference.yaml` is loaded as extra vars, overriding role defaults.
4. After install, `./model-manager` operates standalone against the live cluster, reading
   the environment's `models.yaml` catalog.

## Model objects

`model-manager` creates standard Kubernetes CRDs, so `kubectl` manages them after
deploy:

| Runtime | Object | Routing |
|---|---|---|
| vLLM (`vllm`) | `LLMInferenceService` | `epp` — EPP endpoint picker; `direct` for embedding and rerank models |
| OpenVINO (`openvino`) | `LLMInferenceService` | `epp` — EPP endpoint picker; `direct` for embedding and rerank models |
| vLLM on NVIDIA GPUs (`vllm-cuda`) | `LLMInferenceService` | `epp` — EPP endpoint picker; `direct` for embedding and rerank models |
| vLLM on Intel Arc Pro B-Series (`vllm-xpu`) | `LLMInferenceService` | `epp` — EPP endpoint picker |

All land in `llm_services_namespace` (default `llm-inference`). `--routing epp | direct |
aigateway` overrides the routing for one deploy.

## Related

- [Inference Request Flow](request-flow.md) — end-to-end routing for both auth modes
- [CPU Pinning & NUMA](cpu-pinning.md) — the three CPU policies and the capacity check
- [Database Architecture](database.md) — shared PostgreSQL for LiteLLM and Langfuse
- [Architecture & Design (Toolkit Manager)](https://github.com/intel/toolkit-manager/blob/main/docs/reference/architecture.md) — layers, the installer and its execution flow
- [Configuration](../configuration/configuration.md) — every setting in `config.yaml`
