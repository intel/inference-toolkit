# CPU Pinning & NUMA

[← Docs index](../README.md)

## Why it matters

On CPU clusters, inference throughput depends heavily on *which* cores a model runs on.
A model whose threads are spread across NUMA nodes pays a memory-latency penalty on
every token, and a model sharing cores with a noisy neighbour loses throughput
unpredictably.

A *NUMA node* is a group of cores with the memory attached closest to them. Intel® Inference
Toolkit pins model pods to physical cores that respect the node's NUMA topology, so each model
gets exclusive, locality-aware CPU.

## The three policies

One switch in [`config.yaml`](../configuration/configuration.md#cpu-policy) (in your environment:
`env/<env>/config.inference.yaml`) controls all CPU-pinning infrastructure. Every dependent
setting — NRI install, kubelet flags, model annotations — is derived from it.

```yaml
kubernetes_cpu_policy: "nri-balloons"   # nri-balloons | kubelet-static | best-effort
```

| Policy | Mechanism | Use when |
|---|---|---|
| `nri-balloons` (default) | NRI balloons resource policy — NUMA-aware pinning into named CPU pools | The default. Best throughput and the most control |
| `kubelet-static` | kubelet static CPU manager — Guaranteed QoS grants exclusive whole cores | You cannot run NRI, or you already standardized on the static CPU manager |
| `best-effort` | No pinning; the standard scheduler places threads | Development, oversubscribed nodes, or debugging a pinning problem |

> [!WARNING]
> This must match the value the cluster was **installed** with. It decides both how
> kubelet is configured (Toolkit Manager repo) and how model manifests are rendered (this
> repo). To change it on an existing cluster, follow
> [NRI CPU Balloons](https://github.com/intel/toolkit-manager/blob/main/docs/customize/nri_cpu_balloons.md)
> in the Toolkit Manager docs.

model-manager records the resolved policy on each model pod:

```yaml
annotations:
  cpu-policy.model-manager.io/type: nri-balloons
  cpu-policy.model-manager.io/balloon: model-balloon-tp1
```

Under `nri-balloons` a model needs a whole number of CPUs. If its `cpu` is not a whole
number, the pre-flight stops the deploy ("nri-balloons needs a whole number of CPUs").
With `MM_PREFLIGHT=warn`, `MM_SKIP_PREFLIGHT=1` or `MM_NRI_ENABLED=false` it deploys anyway:
model-manager annotates the pod `type: best-effort` with `reason: nri-resolution-failed`
rather than pinning it incorrectly, and the model serves without exclusive cores.

## NRI balloons

The NRI balloons policy groups CPUs into *balloons* (named sets of cores) and assigns pods to
them, keeping each pod's cores within one NUMA node where possible. model-manager picks the
balloon type from the model's tensor-parallel size (TP), rounded up to 1, 2, 4 or 8
(`model-balloon-tp1` ... `-tp8`); `MM_NRI_BALLOON_NAME=<type>` forces one type for a deploy.

Tuning lives in the Toolkit Manager repo, because it configures the nodes (see its
[NRI CPU Balloons](https://github.com/intel/toolkit-manager/blob/main/docs/customize/nri_cpu_balloons.md)
guide):

```yaml
# nri_reserved_cpu_list: ""            # Linux cpuset for system CPUs (auto-sized if empty)
# nri_deployment_profile: "siblings"   # siblings | minimal(8) | standard(12) | observability(16) | full(20)
```

The profile sizes the CPU reservation for platform services. The default, `siblings`,
reserves the second hyperthread of every physical core, so model balloons get physical cores
only. The others hold back the number of logical CPUs in parentheses. Pick a larger profile
when the cluster also runs observability or RAG workloads, so model pods don't contend with
them.

### Pre-flight capacity check

Before changing anything, `model-manager deploy` verifies the request actually fits:

- `cpu` × `replicas` against the available balloon pool, and
- for `--tp N`, that the shards can spread across NUMA nodes

If it would oversubscribe, the deploy is **blocked** (`NRI pre-flight failed: not enough
physical cores ...`) with actionable guidance rather than creating a pod that will never
schedule. The check runs before the weights are downloaded.

| Override | Effect |
|---|---|
| `MM_PREFLIGHT=warn` | Report the shortfall, then proceed anyway (`MM_NRI_PREFLIGHT=warn` also works) |
| `MM_SKIP_PREFLIGHT=1` | Skip the placement check entirely (`MM_NRI_SKIP_PREFLIGHT=1` skips only the NRI part) |
| `MM_NRI_ENABLED=false` | Legacy: deploy as `best-effort`, without NRI annotations |

Re-deploying an unchanged model is a **no-op**: model-manager diffs the rendered spec
server-side and skips both pre-flight and apply when nothing changed, so a running
model's own cores are never counted against it. Set `MM_FORCE_REDEPLOY=1` to force a full
re-apply.

### Sizing a model to a NUMA node

A catalog model with `cpu: auto` takes its CPU from the same balloon data: the free
physical cores of one NUMA node per TP rank, on the smallest schedulable node. A deployed
model keeps the CPU it has. See [Model Catalog](../configuration/catalog.md#cpu-auto).

### Inspecting what got pinned

`model-manager cpu-collisions-check` reports the actual CPU assignment for running model
pods — pod, balloon, node, NUMA node, CPUs, and hyperthread siblings — and flags cores used by
more than one model. It runs `model_manager/scripts/nri-cpu-report.py`, which needs the
`kubernetes` Python package (`pip install kubernetes`):

```bash
./model-manager cpu-collisions-check
```

## kubelet static CPU manager

Under `kubelet-static`, model pods are rendered as **Guaranteed QoS** — integer `cpu`
with requests equal to limits — so kubelet's static CPU manager grants them exclusive
whole cores.

With the kubelet option `full-pcpus-only` enabled (the Toolkit Manager default), a pod's `cpu` request must be a
multiple of the SMT width, or kubelet rejects the pod with `SMTAlignmentError`.
model-manager rounds `cpu` up to satisfy this automatically.

On an SMT-disabled cluster, tell it the width is 1:

```yaml
mm_kubelet_threads_per_core: 1     # exported as MM_KUBELET_THREADS_PER_CORE
```

## Node placement

Pinning decides *which cores*; node affinity decides *which node*. NFD, which the core platform
layer installs (`node_feature_discovery`), labels every node with its hardware capabilities:

```
feature.node.kubernetes.io/cpu-cpuid.AMXBF16=true
feature.node.kubernetes.io/cpu-cpuid.AVX512F=true
```

model-manager turns the AMX label into a required `nodeAffinity` rule: the catalog's
`require_amx: true` default keeps model pods off nodes without Intel® AMX
(`cpu-cpuid.AMXBF16`) — see [Model Catalog](../configuration/catalog.md#defaults). Pin a model to
one specific node with `--node <name>`: optional, but when given the pod runs on that node or
stays `Pending`.

## Overriding per deploy

```bash
# Force a policy for one model (must match how the cluster was installed)
./model-manager deploy qwen3-0-6b --cpu-policy best-effort --wait

# Give a model 16 cores, split over 2 NUMA nodes
./model-manager deploy llama3-8b-awq --cpu 16 --tp 2 --wait

# See what would be rendered, including annotations, without applying
./model-manager deploy qwen3-0-6b --dry-run
```

Related: [Configuration](../configuration/configuration.md#cpu-policy) ·
[Deploy a Model](../getting-started/deploy-models.md#deploy-options) ·
[Troubleshooting](../faq-and-troubleshooting/troubleshooting.md)
