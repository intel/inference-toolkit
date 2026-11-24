# Frequently Asked Questions

[← Docs index](../README.md)

Short answers with links to the detail. Something failing? See
[Troubleshooting](troubleshooting.md).

## Getting started

### Do I install this repository myself?

No. Toolkit Manager clones it into `ext/enterprise.ai-inference/` with
`./toolkit_manager.sh init inference` and installs it with `install inference`. See
[Quickstart](../getting-started/quickstart.md).

### Do I need a GPU?

No. Models run on Intel® Xeon® CPUs with Intel® AMX. Intel® Arc™ Pro B-Series and NVIDIA
GPUs are optional. See [Prerequisites](../getting-started/prerequisites.md).

### Which models can I run?

The catalog in `env/<env>/models.yaml` lists tested models with their settings, and
`./model-manager list --catalog` prints them. Any other Hugging Face model deploys with
`./model-manager deploy --hfid <org/repo>`. See [Deploy a model](../getting-started/deploy-models.md).

### Which engine serves a model?

Intel® Inference Microservices (IIMs) by default. A catalog entry, or `--server vllm`,
`openvino`, `vllm-cuda` or `vllm-xpu` at deploy time, picks another. See
[Runtimes](../configuration/runtimes.md).

## Calling models

### Should I use `jwt` or `litellm_vkey`?

`jwt` (the default) for single sign-on and per-user access through Keycloak. `litellm_vkey`
for API keys with per-key model lists, budgets and spend tracking; it is also the simplest
first call. See [Accessing models](../getting-started/accessing-models.md).

### Can I serve several models at once?

Yes. Each deploy adds one; the `model` field in the request picks which one answers.

### Which clients work?

Anything that speaks the OpenAI API: `curl`, the OpenAI SDKs, LangChain and others. See
[From an SDK](../getting-started/accessing-models.md#from-an-sdk).

## Operating it

### How do I change a model's settings?

Edit its entry in `env/<env>/models.yaml` and run `./model-manager deploy <name>` again. A
one-off flag such as `--cpu` or `--memory` overrides the entry for that deploy.

### How do I scale a model?

`--replicas N` for a fixed count. With `keda_enabled: true`, `--min-replicas` and
`--max-replicas` autoscale on load. See
[Status, scaling, and logs](../getting-started/deploy-models.md#status-scaling-and-logs).

### How do I remove a model?

`./model-manager undeploy <name>`. The weights stay on the model store, so a later deploy
starts faster.

### How do I upgrade?

From the Toolkit Manager folder: `git pull`, `./toolkit_manager.sh init inference --upgrade`,
then `./toolkit_manager.sh install inference`. See [Upgrade a Layer (Toolkit Manager)](https://github.com/intel/toolkit-manager/blob/main/docs/deployment/upgrade.md).

### Does it run on OpenShift?

Yes, on OpenShift AI. See [Deploy on OpenShift](../deployment/openshift.md).
