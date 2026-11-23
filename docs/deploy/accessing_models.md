# Accessing Models

Both auth modes expose the **same OpenAI-compatible API** on a single endpoint: the
`model` field in the request body selects which deployed model answers. What differs
is the hostname and the credential.

`auth_type` selects the mode — `jwt` (default) or `litellm_vkey`. Set it in your
environment's `env/<env>/config.inference.yaml`, which `init` copies from this repo's
`config.yaml`, then run `./toolkit_manager.sh install inference --env <env>` again.
Keycloak is deployed in both modes and people always sign in to the web UIs through it;
`auth_type` only selects the credential API clients send.

| | **Keycloak JWT** (`auth_type: jwt`) | **LiteLLM virtual key** (`auth_type: litellm_vkey`) |
|---|---|---|
| **Hostname** | `inference.<base_domain_name>` (path mode: `<base_domain_name>`) | `litellm.<base_domain_name>` (path mode: `<base_domain_name>`) |
| **Endpoint** | `https://inference.<base_domain_name>/v1/chat/completions` (path mode: `https://<base_domain_name>/inference/v1/...`) | `https://litellm.<base_domain_name>/v1/chat/completions` (path mode: `https://<base_domain_name>/litellm/v1/...`) |
| **Credential** | Keycloak JWT (`Authorization: Bearer <jwt>`) | LiteLLM master or virtual key (`Authorization: Bearer sk-...`) |
| **Validated by** | The edge gateway, before the request reaches a model | LiteLLM itself, before the request reaches a model |
| **Also deployed** | LiteLLM, Langfuse and Valkey only with `litellm_enabled: true` | LiteLLM, Langfuse, Valkey |
| **Model selection** | `"model"` field in the request body | `"model"` field in the request body |

Terms used here:

- **JWT** (JSON Web Token): a short-lived signed token that Keycloak issues.
- **Virtual key**: an API key that LiteLLM issues, which can be limited to some models,
  a budget and an expiry. The **master key** is LiteLLM's admin key.
- **Subdomain / path mode**: `routing_mode` in `global_config.yaml`. `subdomain` (the
  default) gives each service its own host name; `path` puts every service on
  `<base_domain_name>` under its own path.

> [!IMPORTANT]
> With `routing_mode: subdomain` the two modes serve on **different hostnames**: in
> `litellm_vkey` mode nothing is bound to `inference.<base_domain_name>` — requests there
> return `404`; use `litellm.<base_domain_name>`. With `routing_mode: path` both share
> `<base_domain_name>` and differ by path prefix: `/inference` (`jwt`) and `/litellm` (`litellm_vkey`).
> In `jwt` mode with `litellm_enabled: true`, LiteLLM also needs a Keycloak JWT.

Every hostname resolves to the same edge gateway LoadBalancer IP. Run the commands below
from the Toolkit Manager folder. Set `ENV` to the environment you installed (`--env`,
`local` by default), point `kubectl` at its cluster, and find the gateway IP (empty
output means the gateway is not ready yet). The LiteLLM address follows the
environment's `routing_mode`:

```bash
ENV=local
export KUBECONFIG=$(pwd)/env/$ENV/kubeconfig.yaml
CA=env/$ENV/logs/ai-solutions-ca.crt

GATEWAY_IP=$(kubectl get svc -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=eg-gateway \
  -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')
echo "$GATEWAY_IP"

DOMAIN=$(yq '.base_domain_name' env/$ENV/global_config.yaml)
if [ "$(yq '.routing_mode // "subdomain"' env/$ENV/global_config.yaml)" = "path" ]; then
  LITELLM_HOST=$DOMAIN;         LITELLM_URL=https://$DOMAIN/litellm
else
  LITELLM_HOST=litellm.$DOMAIN; LITELLM_URL=https://litellm.$DOMAIN
fi
```

The examples below use `curl --resolve`, which sends the request to the gateway IP under
the right `Host` and SNI without editing `/etc/hosts`, and `--cacert "$CA"`, which
trusts the installer's self-signed certificate authority
(`env/<env>/logs/ai-solutions-ca.crt`) instead of skipping the check with `-k`.

## Keycloak JWT mode (default)

Get a token. The helper script finds the gateway, reads the Keycloak client secret from
the cluster, and asks Keycloak for a JWT:

```bash
source ./ext/enterprise.ai-inference/model_manager/scripts/get-keycloak-token.sh
# Exports: TOKEN, GATEWAY_IP, GATEWAY_DOMAIN, GATEWAY_URL (includes /inference in path mode)
```

Call the model. A JSON reply with a `choices` list means it worked:

```bash
curl -s --noproxy '*' --cacert "$CA" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  --resolve "${GATEWAY_DOMAIN}:443:${GATEWAY_IP}" \
  -d '{"model":"qwen3-0-6b","messages":[{"role":"user","content":"Hello!"}],"max_tokens":64}' \
  "${GATEWAY_URL}/v1/chat/completions"
```

List the models the gateway will route to:

```bash
curl -s --noproxy '*' --cacert "$CA" \
  -H "Authorization: Bearer $TOKEN" \
  --resolve "${GATEWAY_DOMAIN}:443:${GATEWAY_IP}" \
  "${GATEWAY_URL}/v1/models"
```

> [!NOTE]
> Tokens expire after 15 minutes by default (`keycloak_token_lifespan: 900`). `source` the
> script again for a fresh one, or add `--lifespan 3600` for a 1-hour token (this changes
> the Keycloak client's setting, so every later token lasts that long too).

A request with a missing, malformed, or expired JWT is rejected with `401` at the
gateway, before it reaches a model.

## LiteLLM virtual key mode

LiteLLM is the auth boundary. The edge gateway terminates TLS and forwards to LiteLLM,
which checks the bearer key, that key's allowed models, budget and rate limit, then
routes on to the AI gateway. The content-safety guardrail (on by default in this mode)
screens each request with a guard model first: a blocked request returns HTTP `500`
"Blocked by the content-safety guardrail: …", and while the guard model is not Ready
every request is blocked — see [Guardrails](../customize/guardrails.md).

Get the master key (full admin rights; keep it secret):

```bash
MASTER_KEY=$(kubectl get secret litellm-master-key -n litellm \
  -o jsonpath='{.data.master_key}' | base64 -d)
```

Call the model:

```bash
curl -s --noproxy '*' --cacert "$CA" \
  -H "Authorization: Bearer $MASTER_KEY" \
  -H "Content-Type: application/json" \
  --resolve "${LITELLM_HOST}:443:${GATEWAY_IP}" \
  -d '{"model":"qwen3-0-6b","messages":[{"role":"user","content":"Hello!"}],"max_tokens":64}' \
  "${LITELLM_URL}/v1/chat/completions"
```

List the models registered with LiteLLM — every model that `model-manager deploy`
registered appears here:

```bash
curl -s --noproxy '*' --cacert "$CA" \
  -H "Authorization: Bearer $MASTER_KEY" \
  --resolve "${LITELLM_HOST}:443:${GATEWAY_IP}" \
  "${LITELLM_URL}/v1/models"
```

The public LiteLLM address answers only these paths: `/v1/chat/completions`,
`/v1/completions`, `/v1/embeddings`, `/v1/models` and `/health/liveliness`
(`litellm_public_paths`). Everything else, including the key API below, is reachable
only from inside the cluster.

### Virtual keys

The master key is a full-admin credential. For applications, mint a scoped virtual key
instead — with its own model allow-list, budget, and expiry. The key API is not on the
public address, so open a port-forward (a tunnel from your machine to the LiteLLM
service) and leave it running:

```bash
kubectl port-forward -n litellm svc/litellm 4000:4000 &
```

Create a key through the tunnel. The reply is `{"key": "sk-..."}`:

```bash
VIRTUAL_KEY=$(curl -s http://localhost:4000/key/generate \
  -H "Authorization: Bearer ${MASTER_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"models":["qwen3-0-6b"],"key_alias":"team-a","max_budget":5,"duration":"24h"}' \
  | jq -r '.key')
```

Use the virtual key on the public address exactly like the master key — swap
`$MASTER_KEY` for `$VIRTUAL_KEY`:

```bash
curl -s --noproxy '*' --cacert "$CA" \
  -H "Authorization: Bearer $VIRTUAL_KEY" \
  -H "Content-Type: application/json" \
  --resolve "${LITELLM_HOST}:443:${GATEWAY_IP}" \
  -d '{"model":"qwen3-0-6b","messages":[{"role":"user","content":"Hello!"}],"max_tokens":64}' \
  "${LITELLM_URL}/v1/chat/completions"
```

A request with an unknown, expired or revoked key is rejected with `401` by LiteLLM
before anything reaches a model. A key used for a model outside its allow-list is
rejected too.

#### Manage keys via API

Through the same tunnel (`kubectl port-forward -n litellm svc/litellm 4000:4000`):

```bash
# List (includes alias, allow-list, budget, and spend to date)
curl -s "http://localhost:4000/key/list?return_full_object=true" \
  -H "Authorization: Bearer ${MASTER_KEY}"

# Update (e.g. extend budget)
curl -s http://localhost:4000/key/update \
  -H "Authorization: Bearer ${MASTER_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"key":"sk-...","max_budget":20}'

# Revoke
curl -s http://localhost:4000/key/delete \
  -H "Authorization: Bearer ${MASTER_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"keys":["sk-..."]}'
```

#### Manage keys via UI

Open the UI (the trailing slash is required) → **Virtual Keys** tab. The UI lets you create,
view spend, and revoke keys without curl.

- `routing_mode: subdomain`: `https://litellm-ui.<base_domain_name>/ui/`
- `routing_mode: path`: `https://<base_domain_name>/litellm/ui/`

The UI signs you in through Keycloak; only `litellm_ui_groups` (default: the platform admin
group) are admitted, and password login is off. Without Keycloak the UI is not published on
the gateway. Reach it with `kubectl port-forward -n litellm svc/litellm 4000` at
`http://localhost:4000/ui/` and log in as `admin` with the password from the
`litellm-ui-auth` secret:
`kubectl get secret -n litellm litellm-ui-auth -o jsonpath='{.data.password}' | base64 -d`.

### Observability

Request traces, token usage, and cost land in Langfuse automatically:

| UI | URL | Login |
|---|---|---|
| Langfuse | `https://langfuse.<base_domain_name>` | Keycloak. The seeded admin `langfuse_admin_email` (default `admin@<base_domain_name>`) also keeps a password: `kubectl get secret -n langfuse langfuse-secrets -o jsonpath='{.data.admin_password}' \| base64 -d` |

## Streaming

Both modes support server-sent event streaming (the answer arrives as it is generated) —
add `"stream": true` to the body and `-N` to `curl` (Keycloak mode shown):

```bash
curl -s -N --noproxy '*' --cacert "$CA" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  --resolve "${GATEWAY_DOMAIN}:443:${GATEWAY_IP}" \
  -d '{"model":"qwen3-0-6b","messages":[{"role":"user","content":"Count to five."}],"stream":true}' \
  "${GATEWAY_URL}/v1/chat/completions"
```

## Other endpoints

Embedding and reranking models deployed with this toolkit serve on the same host:

| Workload | Endpoint |
|---|---|
| Chat / completion (`llm`, `vlm`) | `/v1/chat/completions`, `/v1/completions` |
| Embedding (`embed`) | `/v1/embeddings` |
| Reranking (`rerank`) | `/v1/score`, `/v1/rerank`, `/v2/rerank` — Keycloak mode only; not on the public LiteLLM address |

Reranking requests are routed by path, not by the `model` field. With more than one
reranker deployed, pick one with the header `x-ai-eg-model: <model name>`.

## From an SDK

Any OpenAI-compatible client works — only `base_url` and `api_key` change:

```python
import os
from openai import OpenAI

client = OpenAI(
    # auth_type jwt, after `source .../get-keycloak-token.sh`: GATEWAY_URL + "/v1"
    # auth_type litellm_vkey: "https://litellm.<base_domain_name>/v1" and a virtual key
    base_url=os.environ["GATEWAY_URL"] + "/v1",
    api_key=os.environ["TOKEN"],
)

print(client.chat.completions.create(
    model="qwen3-0-6b",
    messages=[{"role": "user", "content": "Hello!"}],
).choices[0].message.content)
```

Run it with `SSL_CERT_FILE=env/<env>/logs/ai-solutions-ca.crt` so Python trusts the
installer's certificate authority. Your machine must also resolve the host name to the
gateway IP (DNS or `/etc/hosts`); Python has no `--resolve`. For frameworks (LangChain,
LlamaIndex, and others) see the Toolkit Manager
[Integration Guide](https://github.com/intel/toolkit-manager/blob/main/docs/customize/integration.md).

For the full path a request takes through the gateways, see
[Inference Request Flow](../reference/request_flow.md).
