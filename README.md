![Intel® Inference Toolkit](docs/assets/banner.jpg)

# Intel® Inference Toolkit

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://www.apache.org/licenses/LICENSE-2.0)
[![Installed and operated by: Toolkit Manager](https://img.shields.io/badge/Installed%20and%20operated%20by-Toolkit%20Manager-0068B5)](https://github.com/intel/toolkit-manager)
[![Platform: Intel® Xeon® and Intel® GPU](https://img.shields.io/badge/Platform-Intel%C2%AE%20Xeon%C2%AE%20%C2%B7%20Intel%C2%AE%20GPU-0068B5)](https://www.intel.com)
[![Serving: KServe](https://img.shields.io/badge/Serving-KServe-326CE5)](https://kserve.github.io/website/)
[![Runtimes: vLLM · OVMS](https://img.shields.io/badge/Runtimes-vLLM%20%C2%B7%20OVMS-purple)](https://vllm.ai)
[![API: OpenAI Compatible](https://img.shields.io/badge/API-OpenAI%20Compatible-green)](https://platform.openai.com/docs/api-reference)

**Serve any Hugging Face model on Intel® Xeon® CPUs or Intel® GPUs with one command. Toolkit Manager installs and operates this toolkit.**

> Provides the Ansible roles that stand up the model-serving stack — KServe, runtimes, AI gateway, LiteLLM, Langfuse — and **`model-manager`**, a CLI that takes a model from Hugging Face to a live OpenAI-compatible endpoint with NUMA-aware CPU pinning.

> [!IMPORTANT]
> **This repository is not used on its own.**
> [**Toolkit Manager**](https://github.com/intel/toolkit-manager)
> installs and operates it. Toolkit Manager clones this repository to
> `ext/enterprise.ai-inference/` and brings up the Kubernetes cluster and the services
> (cert-manager, Istio, MetalLB, Envoy Gateway, PostgreSQL, Keycloak, SeaweedFS, observability)
> this toolkit depends on.
> Every command below runs from the **Toolkit Manager repo root**, not from here.

---

## What is Intel® Inference Toolkit?

Serving a model on Kubernetes normally means hand-writing manifests, sizing CPU and memory, pinning cores to the right NUMA node, downloading weights into shared storage, and wiring a route through a gateway — for every model.

This repository replaces that with a catalog and a CLI. You describe a model once in `models.yaml`, or point `--hfid` at a Hugging Face repo, and `model-manager` resolves the configuration, checks the requested CPU fits the node, downloads the weights, creates the serving pod, and registers the model with the gateway.

The Ansible roles here deploy the serving infrastructure itself: KServe for the model CRDs, an in-cluster AI gateway for model-aware routing, the Keycloak realm and client used for sign-in, and — with `auth_type: litellm_vkey` or `litellm_enabled: true` — LiteLLM for virtual API keys and Langfuse for request tracing.

The result is a single OpenAI-compatible endpoint. The `model` field in the request body selects which model answers, so any OpenAI SDK client works unchanged.

> Want the full picture? See [Architecture](docs/reference/architecture.md) and [Inference Request Flow](docs/reference/request-flow.md).

## Architecture

An inference request enters through the **LLM gateway**, which authenticates and authorizes it, then routes it to a serving engine — **vLLM** (on Xeon CPUs, or on Intel GPUs with `vllm-xpu`) or **OpenVINO™ Model Server** — and returns an OpenAI-compatible response. The gateway also provides model endpoints, user and key management, token telemetry, and monitoring. It all runs on a Kubernetes-orchestrated, Helm-packaged stack over Intel® Xeon® and Intel® GPU infrastructure.

<p align="center">
  <img src="docs/assets/intel-inference-toolkit-architecture.png" alt="Intel® Inference Toolkit: run and scale generative AI inference on a simplified AI stack, on Intel® GPU D-series and Intel® Xeon®, with open-source orchestration. Packaged one-click software deployment. An inference request enters the stack and an inference response leaves it. Top to bottom: API applications and services (samples, API apps and functions, OpenAI-compatible API endpoints); LLM gateway (authentication and authorization, model endpoints, user and key management, token telemetry, monitoring); inferencing engines (vLLM, SGLang, OVMS); orchestration (Kubernetes orchestrator, Helm charts); infrastructure core components (Ubuntu 22.04, Xeon and GPU software: operators, firmware, and drivers)." />
</p>

> Under the hood there are two gateways: the edge gateway terminates TLS and authenticates, while the in-cluster AI gateway routes to the model the request body names. See the [Architecture deep-dive](docs/reference/architecture.md) and [Inference Request Flow](docs/reference/request-flow.md) for the full component list and execution flow.

---

## Quick Start

**Stands up model serving on a freshly prepared machine, then serves an LLM.** In three steps you'll install Intel® Inference Toolkit, deploy a model, and see it answer.

> [!NOTE]
> **Before you start:** Toolkit Manager installs and operates this toolkit. Its installer,
> `toolkit_manager.sh`, clones this repository into `ext/enterprise.ai-inference/`. Get
> Toolkit Manager, prepare the machine, and create an environment, then run every command
> below from the Toolkit Manager folder:
>
> ```bash
> git clone https://github.com/intel/toolkit-manager.git
> cd toolkit-manager
> ./toolkit_manager.sh configure        # one-time machine prep; answer y to the prompt
> ./toolkit_manager.sh init inference   # creates env/local/ and clones this repo into ext/
> ```
>
> Intel® Inference Toolkit needs the cluster, gateway, sign-in and storage that Toolkit
> Manager provides. The `install inference` step below brings those up with it. Hardware
> and software requirements:
> [Toolkit Manager prerequisites](https://github.com/intel/toolkit-manager/blob/main/docs/getting-started/prerequisites.md).

### Step 1 — Install the inference stack

Deploys KServe, the ServingRuntimes, the AI gateway, the Keycloak realm and client, and — with `auth_type: litellm_vkey` or `litellm_enabled: true` — LiteLLM and Langfuse. No models are deployed yet.

```bash
./toolkit_manager.sh install inference      # allow up to an hour on a first install

export KUBECONFIG=$(pwd)/env/local/kubeconfig.yaml
kubectl get pods -n kserve                  # the KServe controller shows Running
```

A successful run ends with an `INSTALLATION COMPLETE` banner.

This toolkit's settings live in `env/local/config.inference.yaml`. `init` copies it from
this repo's [`config.yaml`](config.yaml), and the install reads your copy. Edit it, then
run `install inference` again. See [Configuration](docs/configuration/configuration.md) for every
option, including the `envoy` / `agentgateway` choice and the cluster CPU-pinning policy.

### Step 2 — Deploy a model

`model-manager` resolves the configuration, verifies the requested CPU fits the node, downloads the weights, creates the serving pod, and registers the model with the gateway.

```bash
./model-manager deploy qwen3-0-6b --wait
```

Or skip the catalog and deploy any Hugging Face model directly:

```bash
./model-manager deploy --hfid Qwen/Qwen3-0.6B --cpu 8 --wait   # memory sized from the model
```

`--wait` blocks until the model is ready (up to 15 minutes by default). Run it again any
time: an unchanged model is left as it is.

> [!IMPORTANT]
> Gated models (Llama, Mistral, Gemma) require a free [Hugging Face token](https://huggingface.co/settings/tokens).
> Run `export HF_TOKEN=hf_...` before deploying.

When the command returns it prints the inference endpoint and a ready-to-run `curl` example. Full flag reference → [Deploy a Model](docs/getting-started/deploy-models.md).

### Step 3 — Send your first request

With the default `auth_type: jwt`, API clients send a Keycloak token (a JWT). The helper
script finds the gateway address and gets a token from the cluster for you:

```bash
source ./ext/enterprise.ai-inference/model_manager/scripts/get-keycloak-token.sh
# Exports: TOKEN, GATEWAY_IP, GATEWAY_DOMAIN, GATEWAY_URL
```

Call the model. The `model` field selects which one answers:

```bash
curl -s --noproxy '*' \
  --cacert env/local/logs/ai-solutions-ca.crt \
  --resolve "$GATEWAY_DOMAIN:443:$GATEWAY_IP" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen3-0-6b","messages":[{"role":"user","content":"Hello!"}],"max_tokens":64}' \
  "$GATEWAY_URL/v1/chat/completions"
```

A JSON reply with a `choices` list means the model is serving. `--resolve` sends the
request to the gateway IP without an `/etc/hosts` entry, and `--cacert` trusts the
installer's self-signed certificate authority. The token lasts 15 minutes; `source` the
script again for a new one.

See what is serving:

```bash
kubectl get llminferenceservices,inferenceservices -n llm-inference
```

> [!NOTE]
> **Using `auth_type: litellm_vkey` instead?** Set it in `env/<env>/config.inference.yaml`.
> Keycloak is still deployed for sign-in, but API clients authenticate with a LiteLLM master or
> virtual key on `litellm.<your-domain>` — not a JWT on `inference.<your-domain>`. Full walkthrough →
> [Accessing Models](docs/getting-started/accessing-models.md).

Everything speaks the OpenAI-compatible API, so any tool that works with OpenAI works against your stack.

---

## Advanced

The Quick Start deploys one catalog model with defaults. From here you can change the engine and its version, add models, tune CPU and memory, switch the AI gateway or the CPU-pinning policy, and control what deploys automatically at install time.

| Goal | Guide |
|---|---|
| Deploy, update, and remove models | [Deploy a Model](docs/getting-started/deploy-models.md) |
| Endpoints and authentication modes | [Accessing Models](docs/getting-started/accessing-models.md) |
| Add models — the `models.yaml` catalog | [Model Catalog](docs/configuration/catalog.md) |
| Engines, versions, and per-workload defaults | [Runtimes](docs/configuration/runtimes.md) |
| All configuration options | [Configuration](docs/configuration/configuration.md) |
| Components, roles, and how Toolkit Manager installs this toolkit | [Architecture](docs/reference/architecture.md) |
| How a request reaches a model | [Inference Request Flow](docs/reference/request-flow.md) |
| Content-safety screening of LiteLLM requests | [Guardrails](docs/configuration/guardrails.md) |
| Route each request to a model by prompt | [Hybrid Router](docs/configuration/hybrid-router.md), [vLLM Semantic Router](docs/configuration/semantic-router.md) |
| NUMA-aware CPU pinning | [CPU Pinning & NUMA](docs/reference/cpu-pinning.md) |
| Something isn't working | [Troubleshooting](docs/faq-and-troubleshooting/troubleshooting.md) |

---

## License

Licensed under the [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0).

## Links

- [Documentation Index](docs/README.md)
- [GitHub Repository](https://github.com/intel/inference-toolkit)
- [Toolkit Manager](https://github.com/intel/toolkit-manager) — installs and operates this toolkit
- [Architecture](docs/reference/architecture.md)
- [Inference Request Flow](docs/reference/request-flow.md)

---

*Intel® and Intel® Xeon® are registered trademarks of Intel Corporation or its subsidiaries. Licensed under the Apache License, Version 2.0.*
