# Intel® Inference Toolkit documentation

Start with the table. Each row is one job and one page. The sections under it are the same pages, grouped by folder, for when you already know the area.

> **Toolkit Manager** installs, upgrades and tears down Intel® Inference Toolkit. Clone [that repository](https://github.com/intel/toolkit-manager), run every command from its root, and `init inference` clones this repository under `ext/`. The three-step quick start is in the [project README](../README.md#quick-start).

## Find a page

| I want to… | Page |
|---|---|
| Deploy, update or remove a model with `model-manager` | [Deploy a model](getting-started/deploy-models.md) |
| Get an endpoint and credentials, and list what is serving | [Accessing models](getting-started/accessing-models.md) |
| Install and deploy models on OpenShift AI | [Deploy on OpenShift](deployment/openshift.md) |
| Add a model to the catalog, or deploy one at install time | [Model catalog](configuration/catalog.md) |
| Choose an engine, version or image | [Runtimes](configuration/runtimes.md) |
| See every option in `config.yaml` and the `MM_*` variables | [Configuration](configuration/configuration.md) |
| Screen each LiteLLM request with a guard model | [Guardrails](configuration/guardrails.md) |
| Send a prompt to one of several models | [Hybrid router](configuration/hybrid-router.md) and [vLLM semantic router](configuration/semantic-router.md) |
| See how the toolkit is built | [Architecture](reference/architecture.md) |
| Follow a request from client to model | [Inference request flow](reference/request-flow.md) |
| Understand CPU pinning and the capacity check | [CPU pinning and NUMA](reference/cpu-pinning.md) |
| Understand how memory and KV cache are sized | [Memory and KV-cache sizing](reference/memory-sizing.md) |
| See the LiteLLM and Langfuse database setup | [Database architecture](reference/database.md) |
| Fix a failed deploy, API call or pod | [Troubleshooting](faq-and-troubleshooting/troubleshooting.md) |
| Report a bug or change the toolkit | [Contributing](../CONTRIBUTING.md) |

## Folders

| Folder | When you open it |
|---|---|
| `getting-started/` | Deploying a first model and calling it |
| `deployment/` | Other platforms |
| `configuration/` | Models, engines, routing and the settings you edit after the install |
| `reference/` | How the toolkit is built and how a request travels |
| `faq-and-troubleshooting/` | Failures and their fixes |

## First model

| Guide | What it covers |
|---|---|
| [Deploy a model](getting-started/deploy-models.md) | Every `model-manager` command and flag: deploy, update, undeploy, gated models, ad-hoc Hugging Face deploys |
| [Accessing models](getting-started/accessing-models.md) | Endpoints and credentials for both auth modes, and how to list what's serving |

## Other ways to install

| Guide | What it covers |
|---|---|
| [Deploy on OpenShift](deployment/openshift.md) | `install inference` on OpenShift AI, deploying models with `model-manager`, CPU balloons, calling a model |

## After the install

Edit the YAML under `env/<name>/` and re-run `install inference`.

| Guide | What it covers |
|---|---|
| [Model catalog](configuration/catalog.md) | The `models.yaml` catalog: every field, and auto-deploy at install time |
| [Runtimes](configuration/runtimes.md) | vLLM (CPU, NVIDIA and Intel GPU) and OpenVINO: versions, images and per-workload defaults |
| [Configuration](configuration/configuration.md) | Every option in `config.yaml`, plus the `MM_*` environment variables |
| [Guardrails](configuration/guardrails.md) | Content-safety screening of LiteLLM requests with a guard model (Qwen3Guard) |
| [Hybrid router](configuration/hybrid-router.md) | Complexity-tier and bandit routing behind one virtual model in LiteLLM |
| [vLLM semantic router](configuration/semantic-router.md) | Opt-in classifier-based routing on a dedicated Envoy AI Gateway, and its dashboard |

## Reference

| Guide | What it covers |
|---|---|
| [Architecture](reference/architecture.md) | Components, Ansible roles, and how Toolkit Manager installs and operates this toolkit |
| [Inference request flow](reference/request-flow.md) | End-to-end request routing for both auth modes, with a verified network trace |
| [CPU pinning and NUMA](reference/cpu-pinning.md) | The three CPU policies, the pre-flight capacity check, and NUMA placement |
| [Memory and KV-cache sizing](reference/memory-sizing.md) | How a vLLM model's memory and KV cache are sized from its config, layer layouts, TP, and the memory pre-flight |
| [Database architecture](reference/database.md) | Shared PostgreSQL schema and provisioning for LiteLLM and Langfuse |

## When something fails

| Guide | What it covers |
|---|---|
| [Troubleshooting](faq-and-troubleshooting/troubleshooting.md) | Common symptoms and their fixes |

## Contribute

| Guide | What it covers |
|---|---|
| [Contributing](../CONTRIBUTING.md) | Reporting a problem or sending a change |
