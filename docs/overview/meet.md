# Meet Intel® Inference Toolkit

[← Docs index](../README.md)

Intel® Inference Toolkit serves open models on your own Intel® Xeon® servers, with optional
Intel® Arc™ or NVIDIA GPUs, behind one OpenAI-compatible API. Any tool that talks to OpenAI
works against it, and no prompt or answer leaves your cluster.

## What you get

- **One endpoint for every model.** Clients name the model in the request; the Envoy AI
  Gateway routes it to that model's pods.
- **One command per model.** `./model-manager deploy <name>` sizes the pod, downloads the
  weights, starts the server and registers the model. A catalog (`models.yaml`) holds tested
  settings for each model; any Hugging Face model also works with `--hfid`.
- **Engines tuned for Intel®.** Intel® Inference Microservices (IIMs, the default) run vLLM
  with Intel-validated profiles. Upstream vLLM, OpenVINO™ Model Server, and vLLM on
  Intel® Arc™ Pro B-Series or NVIDIA GPUs are one flag away.
- **Access control.** Keycloak tokens (JWT, the default), or LiteLLM virtual keys with
  per-key model lists, budgets and spend tracking.
- **Optional extras.** Request tracing (Langfuse), a content-safety guardrail, routers that
  pick a model per prompt, and autoscaling with KEDA.

## How it fits

Intel® Inference Toolkit is layer 3 of
[Toolkit Manager](https://github.com/intel/toolkit-manager). Toolkit Manager builds the
cluster and the platform underneath (gateway, sign-in, storage, monitoring), clones this
repository, and installs it with `./toolkit_manager.sh install inference`. Intel® Agent
Toolkit and Intel® Chatbot Toolkit run on top of it and use its models.

```
Layer 4   Intel® Agent Toolkit · Intel® Chatbot Toolkit   (optional)
Layer 3   Intel® Inference Toolkit                        ◀── this repository
Layer 2   Platform: Envoy Gateway, Keycloak, storage, observability
Layer 1   Infrastructure: Kubernetes
```

## Where next

1. [Prerequisites](../getting-started/prerequisites.md): the machine, and what models add to it
2. [Quickstart](../getting-started/quickstart.md): install, deploy a model, send a request
3. [Architecture](../reference/architecture.md): the components and how a request travels
