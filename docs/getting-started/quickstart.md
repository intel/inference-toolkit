# Quickstart

[← Docs index](../README.md)

From a prepared machine to a model answering a request. Check the
[Prerequisites](prerequisites.md) first.

## 1. Get Toolkit Manager

Toolkit Manager installs this toolkit. Run every command below from its folder.

```bash
git clone https://github.com/intel/toolkit-manager.git
cd toolkit-manager
./toolkit_manager.sh configure        # once per machine: Python, yq, kubectl, helm
./toolkit_manager.sh init inference   # creates env/local/ and clones this repo into ext/
```

**Which branch:** Toolkit Manager `main` installs this repository's `main`, which these docs
describe. For the latest release, clone with `--branch v1.2.0`: it installs Intel® Inference
Toolkit v1.1.0, documented at its
[v1.1.0 tag](https://github.com/intel/inference-toolkit/tree/v1.1.0).

## 2. Choose how clients sign in

In `env/local/config.inference.yaml`, `auth_type` picks the credential API clients send:

| `auth_type` | Clients send | Endpoint | Pick it for |
|---|---|---|---|
| `jwt` (default) | A Keycloak token, valid 15 minutes | `https://inference.<domain>/v1` | Single sign-on and per-user access |
| `litellm_vkey` | A LiteLLM key | `https://litellm.<domain>/v1` | The simplest first call; per-key budgets, spend and traces |

People sign in to the web UIs through Keycloak either way. See
[Accessing models](accessing-models.md).

## 3. Install

```bash
./toolkit_manager.sh install inference      # brings up the cluster and platform too; allow up to an hour
./toolkit_manager.sh validate inference     # health checks; stops at the first failure with the reason
export KUBECONFIG=$(pwd)/env/local/kubeconfig.yaml
```

## 4. Deploy a model

```bash
./model-manager deploy qwen3-0-6b --wait    # first start downloads the weights
./model-manager list --deployed
```

To deploy models during the install instead, see
[Auto-deploy on install](../configuration/catalog.md#auto-deploy-on-install).

## 5. Send a request

With `auth_type: jwt`:

```bash
source ./ext/enterprise.ai-inference/model_manager/scripts/get-keycloak-token.sh   # sets TOKEN, GATEWAY_IP, GATEWAY_DOMAIN, GATEWAY_URL
curl -s --noproxy '*' --cacert env/local/logs/toolkit-manager-ca.crt \
  --resolve "$GATEWAY_DOMAIN:443:$GATEWAY_IP" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"model":"qwen3-0-6b","messages":[{"role":"user","content":"Hello!"}],"max_tokens":64}' \
  "$GATEWAY_URL/v1/chat/completions"
```

With `auth_type: litellm_vkey`:

```bash
GATEWAY_IP=$(kubectl get svc -n envoy-gateway-system -l gateway.envoyproxy.io/owning-gateway-name=eg-gateway \
  -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')
DOMAIN=$(yq '.base_domain_name' env/local/global_config.yaml)
MK=$(kubectl get secret litellm-master-key -n litellm -o jsonpath='{.data.master_key}' | base64 -d)

curl -s --noproxy '*' --cacert env/local/logs/toolkit-manager-ca.crt --resolve "litellm.$DOMAIN:443:$GATEWAY_IP" \
  -H "Authorization: Bearer $MK" -H "Content-Type: application/json" \
  -d '{"model":"qwen3-0-6b","messages":[{"role":"user","content":"Hello!"}],"max_tokens":64}' \
  "https://litellm.$DOMAIN/v1/chat/completions"
```

A JSON reply with `choices` means the model is serving. The admin key has full rights: give
each application its own [virtual key](accessing-models.md#virtual-keys).

| Symptom | Fix |
|---|---|
| `curl` exits with code 60 (certificate problem) | Pass `--cacert env/local/logs/toolkit-manager-ca.crt` |
| `401` "Jwt is expired" | `source` the token script again |
| `401` "Authentication Error" | Wrong, expired or revoked LiteLLM key |
| `404` "No matching route found", or `400` "Invalid model name" | The `model` field must be a deployed name: `./model-manager list --deployed` |

More → [Troubleshooting](../faq-and-troubleshooting/troubleshooting.md).

## Next steps

- [Deploy a model](deploy-models.md): every `model-manager` flag, gated models, GPUs
- [Model catalog](../configuration/catalog.md): add your own models
- [Integration guide (Toolkit Manager)](https://github.com/intel/toolkit-manager/blob/main/docs/reference/integration.md): LangChain, the OpenAI SDK and other clients
