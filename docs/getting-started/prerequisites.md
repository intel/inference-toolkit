# Prerequisites

[← Docs index](../README.md)

Toolkit Manager sets up the machine and the cluster, so start with its
[prerequisites](https://github.com/intel/toolkit-manager/blob/main/docs/getting-started/prerequisites.md):
the operating system, `sudo`, internet access, and the minimum size (32 CPU cores, 64 GB of
memory and 300 GB of disk for the platform plus a small model). This page lists what the
inference layer adds.

## CPU nodes

- **Intel® AMX.** Catalog models run only on nodes with Intel® AMX, 4th Gen Intel® Xeon®
  Scalable processors or newer (`require_amx: true` in `models.yaml`). On older nodes, set
  `require_amx: false` under `defaults:` in `env/<env>/models.yaml`. See
  [Model catalog](../configuration/catalog.md).
- **CPU and memory per model.** Each catalog entry sets them: `qwen3-0-6b` needs 8 cores and
  24 GiB, larger models more. `model-manager` checks that a model fits before it deploys it.
  See [Memory and KV-cache sizing](../reference/memory-sizing.md).

## Storage

- **Model store.** One shared volume, `200Gi` by default, holds the weights of every model.
  For more, set `llm_services_model_store_size` in `env/<env>/config.inference.yaml` and
  re-run `install inference`; the volume grows but never shrinks.
- **Several nodes.** Models on more than one node need storage every node can mount
  (`ReadWriteMany`, for example NFS or Ceph), set in Toolkit Manager's `storage_backend`.

## Network

- **Downloads.** The nodes pull images from `docker.io` and `ghcr.io`, and weights from
  `huggingface.co`. Behind a proxy, set it in Toolkit Manager's `global_config.yaml`; models
  the install deploys inherit it. For `./model-manager` runs, set the `network:` section of
  `env/<env>/models.yaml`, or export `HTTPS_PROXY`. See
  [Troubleshooting](../faq-and-troubleshooting/troubleshooting.md#deploy-and-storage).
- **Gated models** (Llama, Mistral, Gemma) need a free
  [Hugging Face token](https://huggingface.co/settings/tokens): `export HF_TOKEN=hf_...`
  before you deploy them.

## GPU nodes (optional)

- **NVIDIA GPUs** (`vllm-cuda`): the node advertises `nvidia.com/gpu`, from the NVIDIA GPU
  operator or device plugin.
- **Intel® Arc™ Pro B-Series GPUs** (`vllm-xpu`): the node advertises `gpu.intel.com/xe`.
  Toolkit Manager installs the Intel GPU device plugin on Intel GPU nodes.

`model-manager` refuses a GPU deploy when no Ready node offers the resource. See
[Deploy on GPU nodes](deploy-models.md#deploy-on-gpu-nodes).

Next: [Quickstart](quickstart.md).
