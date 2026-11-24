# Database Architecture

[← Docs index](../README.md)

## Overview

The Intel Inference Toolkit uses a **shared CloudNativePG (CNPG) PostgreSQL cluster**, which Toolkit Manager's platform layer installs (the `postgresql` role). Each consumer (LiteLLM, Langfuse, Keycloak) gets its own isolated database and user within the same cluster. The `litellm` and `langfuse` databases exist only while those services are enabled (`auth_type: litellm_vkey` or `litellm_enabled: true`; Langfuse also needs `langfuse_enabled`, default `true`).

Langfuse keeps its traces, observations and scores in **ClickHouse** (installed by the `langfuse` role through the ClickHouse operator), its queues in a **Valkey** from its Helm chart, and uploads in the platform **S3 object store** (SeaweedFS by default). PostgreSQL holds its projects, users, API keys and settings. LiteLLM's response cache is its own Valkey, deployed with the LiteLLM chart.

```
┌─────────────────────────────────────────────────────────────────┐
│  PostgreSQL Cluster (CNPG)                                      │
│  Pod: postgresql-1                                              │
│  Service: postgresql-rw.postgresql.svc.cluster.local:5432       │
│                                                                 │
│  ┌──────────────┬──────────────┬──────────────┐                │
│  │ DB: keycloak │ DB: litellm  │ DB: langfuse │                │
│  │ User:keycloak│ User:litellm │ User:langfuse│                │
│  └──────────────┴──────────────┴──────────────┘                │
└─────────────────────────────────────────────────────────────────┘
```

## Components

| Component | Role | Description |
|-----------|------|-------------|
| CNPG Operator | `cnpg-system` namespace | Manages PostgreSQL cluster lifecycle, failover, backups |
| PostgreSQL Cluster | `postgresql` namespace | Single shared PostgreSQL 17 instance (`postgresql_instances`, default `1`) |
| Database CRDs | `postgresql` namespace | Declarative database creation via CNPG `Database` custom resource |
| Consumer Secrets | Per-consumer namespace | Credentials copied from postgresql namespace to each consumer |

## Configuration

Databases are declared in `postgresql_databases`, a default of the Toolkit Manager `postgresql` role (`roles/postgresql/defaults/main.yaml` there):

```yaml
postgresql_databases:
  - name: keycloak            # first entry = bootstrap DB (created at cluster init)
    user: keycloak
  - name: litellm
    user: litellm
    enabled: "{{ auth_type | default('jwt') == 'litellm_vkey' or litellm_enabled | default(false) | bool }}"
  - name: langfuse
    user: langfuse
    enabled: "{{ (auth_type | default('jwt') == 'litellm_vkey' or litellm_enabled | default(false) | bool) and langfuse_enabled | default(true) | bool }}"
```

`enabled` (default `true`) decides whether a database is created. `target_namespace` (default: the `name`) is the namespace that receives `<name>-db-secret`.

## Provisioning Flow

### Phase 1: PostgreSQL role (runs first in playbook)

```
Step 1 ─ Install CNPG Operator (Helm)
  └─→ Deploys operator in cnpg-system namespace
       (on OpenShift: uses the marketplace operator when one is installed)

Step 2 ─ Create credential Secrets (check-then-create, never overwritten)
  └─→ postgresql-keycloak-secret   (namespace: postgresql)
  └─→ postgresql-litellm-secret    (namespace: postgresql)
  └─→ postgresql-langfuse-secret   (namespace: postgresql)
       Each contains: {username, password (auto-generated)}

Step 3 ─ Create SQL init Secrets (create or update each user)
  └─→ postgresql-init-litellm      (namespace: postgresql)
  └─→ postgresql-init-langfuse     (namespace: postgresql)
       Each contains SQL: CREATE USER if missing, else ALTER USER ... WITH PASSWORD (resets it to the stored one)

Step 4 ─ Deploy CNPG Cluster CR
  └─→ bootstrap.initdb creates first DB (keycloak)
  └─→ postInitSQLRefs runs SQL from init Secrets (creates users)
  └─→ On a cluster that already runs: creates missing users directly
  └─→ Result: postgresql-1 Pod running PostgreSQL 17

Step 5 ─ Create additional databases via CNPG Database CRD
  └─→ Database CR "litellm"  → operator creates DB, owner=litellm
  └─→ Database CR "langfuse" → operator creates DB, owner=langfuse
  └─→ Waits until each reports status.applied = true

Step 6 ─ Copy secrets to consumer namespaces
  └─→ keycloak-db-secret  (namespace: keycloak)
  └─→ litellm-db-secret   (namespace: litellm)
  └─→ langfuse-db-secret  (namespace: langfuse)
       Each contains:
         username: <user>
         password: <password>
         database: <db_name>
         host: postgresql-rw.postgresql.svc.cluster.local
         port: 5432
```

### Phase 2: Consumer roles (run later in playbook)

Each consumer role follows the same pattern:

```
Step 1 ─ Read "<name>-db-secret" in its namespace
  └─→ Missing (e.g. after teardown litellm --only): runs the postgresql
      role's consumer step, which creates just this database, user and
      secret; stops with "install postgresql first" if PostgreSQL is absent

Step 2 ─ Wait for Database CRD status.applied = true (up to 5 minutes)
  └─→ Confirms the database actually exists in PostgreSQL

Step 3 ─ Extract connection details from secret
  └─→ host, port, database, username, password

Step 4 ─ Pass credentials to Helm chart
  └─→ LiteLLM:  db.useExisting=true, db.endpoint=<host>,
               db.secret.name=litellm-db-secret
  └─→ Langfuse: postgresql.deploy=false, postgresql.host=<host>,
               postgresql.auth.*
```

## Example: LiteLLM connecting to PostgreSQL

```
LiteLLM Pod
  │
  │ reads litellm-db-secret from namespace "litellm"
  │   username: litellm
  │   password: <generated>
  │   host: postgresql-rw.postgresql.svc.cluster.local
  │   port: 5432
  │   database: litellm
  │
  ▼
postgresql-rw.postgresql.svc.cluster.local:5432
  │
  ▼
postgresql-1 Pod (CNPG managed)
  └─→ database "litellm", owned by user "litellm"
      └─→ Tables: virtual keys, spend logs, team/org data (managed by Prisma)
```

## Example: Langfuse connecting to PostgreSQL

```
Langfuse Web/Worker Pods
  │
  │ Helm values inject credentials directly:
  │   postgresql.host: postgresql-rw.postgresql.svc.cluster.local
  │   postgresql.auth.username: langfuse
  │   postgresql.auth.password: <from langfuse-db-secret>
  │   postgresql.auth.database: langfuse
  │
  ▼
postgresql-rw.postgresql.svc.cluster.local:5432
  │
  ▼
postgresql-1 Pod (CNPG managed)
  └─→ database "langfuse", owned by user "langfuse"
      └─→ Tables: projects, users, API keys, settings (managed by Prisma)

Traces, observations and scores go to ClickHouse, not PostgreSQL. After its first
start the langfuse role checks this database for failed Prisma migrations and clears
them, so the next start retries them.
```

## Design Decisions

### Why shared PostgreSQL (not per-service)?

| Concern | Shared CNPG | Per-service Bitnami standalone |
|---------|-------------|-------------------------------|
| High availability | CNPG can replicate and fail over (raise `postgresql_instances`; the default `1` has no failover) | Single-node, no HA |
| Backups | CNPG supports WAL archiving and PITR (not configured by default) | Manual, no built-in |
| Resource efficiency | 1 pod serves all services | N pods (one per service) |
| Operational complexity | Single cluster to manage | Multiple independent instances |
| Isolation | Separate databases + users (can't access each other) | Full process isolation |

### Why passwords are stable across re-runs

The postgresql role uses a **check-then-create** pattern:
1. Check if `postgresql-<name>-secret` exists
2. Only create it if it doesn't exist
3. Never overwrite existing secrets

This ensures re-running the playbook won't rotate passwords and break running services.

### Why secrets are copied to consumer namespaces

- The postgresql role creates the credentials in the `postgresql` namespace
- Pods in other namespaces (litellm, langfuse) can't read cross-namespace secrets
- The postgresql role copies each consumer's credentials into their target namespace
- Consumer roles just wait for their local secret to appear

## Secrets Reference

| Secret name | Namespace | Created by | Contents |
|-------------|-----------|------------|----------|
| `postgresql-<name>-secret` | postgresql | postgresql role | username, password |
| `postgresql-init-<name>` | postgresql | postgresql role | SQL that creates the user, or resets its password |
| `keycloak-db-secret` | keycloak | postgresql role (copied) | username, password, database, host, port |
| `litellm-db-secret` | litellm | postgresql role (copied) | username, password, database, host, port |
| `langfuse-db-secret` | langfuse | postgresql role (copied) | username, password, database, host, port |

## Adding a New Database Consumer

To add a new service that needs PostgreSQL:

1. Add an entry to `postgresql_databases` in `env/<env>/global_config.yaml`. The setting
   replaces the whole list, so keep the three default entries (see
   [Configuration](#configuration)) and add yours:
   ```yaml
   postgresql_databases:
     # ... the keycloak, litellm and langfuse entries ...
     - name: myservice
       user: myservice
       target_namespace: myservice
   ```

2. Re-run the role with `./toolkit_manager.sh install postgresql --only --env <env>` — the postgresql role will:
   - Generate credentials
   - Create the user + database
   - Copy the secret to the target namespace

3. In your service's Helm values, reference the secret:
   ```yaml
   db:
     host: postgresql-rw.postgresql.svc.cluster.local
     port: 5432
     secret:
       name: myservice-db-secret
       usernameKey: username
       passwordKey: password
   ```

## Troubleshooting

### Check if database exists
```bash
kubectl get database -n postgresql
```

### Check database readiness
```bash
kubectl get database <name> -n postgresql -o jsonpath='{.status.applied}'
# Should return: true
```

### Check consumer secret
```bash
kubectl get secret <name>-db-secret -n <namespace> -o jsonpath='{.data.host}' | base64 -d
```

### Connect to PostgreSQL directly
```bash
kubectl exec -it postgresql-1 -n postgresql -c postgres -- psql -U postgres
\l          # list databases
\du         # list users
```

### Check CNPG cluster health
```bash
kubectl get cluster -n postgresql
kubectl describe cluster postgresql -n postgresql
```
