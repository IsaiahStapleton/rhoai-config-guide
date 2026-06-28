# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A step-by-step configuration guide (and companion manifests) for setting up Red Hat OpenShift AI (RHOAI) to deploy AI models using vLLM ServingRuntime for KServe on OpenShift. There is no build system, test suite, or application code — the deliverables are YAML manifests and documentation.

## Applying Manifests

All commands use the OpenShift CLI (`oc`). Apply manifests in numbered order:

```bash
# Step 1 — GPU support (NFD + NVIDIA GPU Operator)
oc apply -f manifests/01/nfd-operator.yaml
oc get pods -n openshift-nfd -w          # wait until ready
oc apply -f manifests/01/nfd-instance.yaml
oc apply -f manifests/01/nvidia-gpu-operator.yaml

# Retrieve and apply ClusterPolicy (generated into scratch/, not committed)
oc get csv -n nvidia-gpu-operator \
  -l operators.coreos.com/gpu-operator-certified.nvidia-gpu-operator \
  -ojsonpath='{.items[0].metadata.annotations.alm-examples}' | \
  jq '.[0]' > scratch/nvidia-gpu-clusterpolicy.json
oc apply -f scratch/nvidia-gpu-clusterpolicy.json

# Step 2 — KServe dependencies (Service Mesh, Serverless, Authorino, Cert-Manager, JobSet, Connectivity Link, LeaderWorkerSet)
oc apply -f manifests/02/servicemesh-subscription.yaml
oc apply -f manifests/02/serverless-operator.yaml
oc apply -f manifests/02/authorino-subscription.yaml
oc apply -f manifests/02/cert-manager-operator.yaml
oc apply -f manifests/02/jobset-operator-subscription.yaml
oc apply -f manifests/02/jobset-operator.yaml
oc apply -f manifests/02/connectivity-link-operator.yaml
oc patch console.operator.openshift.io cluster --type=json \
  -p '[{"op":"add","path":"/spec/plugins/-","value":"kuadrant-console-plugin"}]'
oc apply -f manifests/02/kuadrant-instance.yaml   # activates Kuadrant; required for AuthPolicy enforcement
# RHCL 1.4 Limitador workaround (three steps required to get MaaSSubscription to Active):
# Step A — Create the Limitador CR (the Kuadrant Operator does not auto-create it)
oc apply -f manifests/02/limitador-instance.yaml
oc get pods -n kuadrant-system -l app.kubernetes.io/name=limitador -w   # wait until Running
# Step B — Create a stub cluster-scoped CRD for LimitadorOperator (removed in Limitador Operator 1.x;
#   Kuadrant 1.4 still checks for this CRD when reconciling MaaSSubscription tokenRateLimits).
oc apply -f manifests/02/limitador-operator-stub-crd.yaml
# Step C — Create a LimitadorOperator CR (cluster-scoped) so the Kuadrant Operator sees it as present
oc apply -f manifests/02/limitador-operator-cr.yaml
# Step D — Delete the Kuadrant Operator pod to force re-reconciliation (OLM will restart it)
oc delete pod -n kuadrant-system -l app=kuadrant,control-plane=controller-manager
oc get pods -n kuadrant-system -l app=kuadrant,control-plane=controller-manager -w   # wait Running
oc apply -f manifests/02/leader-worker-set-subscription.yaml
oc apply -f manifests/02/leader-worker-set-operator.yaml

# Step 3 — RHOAI Operator + DataScienceCluster
oc apply -f manifests/03/rhoai-operator.yaml
oc apply -f manifests/03/rhoai-operator-dsc.yaml

# Step 4 — Shared PostgreSQL + Model Registry (Kustomize)
# Deploys PostgreSQL in redhat-ods-applications (shared by Model Registry and MaaS),
# creates the registry database via init script, and applies the ModelRegistry CR.
oc apply -k manifests/04/
oc get pods -n redhat-ods-applications -l app=maas-postgresql -w   # wait until Running

# Step 5 — Enable LlamaStack Operator + create llm-d inference Gateway
oc patch DataScienceCluster default-dsc \
  --type=merge \
  --patch-file manifests/05/rhoai-dsc-enable-llamastack.yaml
oc apply -f manifests/05/openshift-ai-inference-gateway.yaml

# Step 6 — GPU HardwareProfile for RHOAI workloads
oc apply -f manifests/06/gpu-hardware-profile.yaml

# Step 7 — Deploy a LLMInferenceService (llm-d)
# 7a — Create the dybbol project (makes it visible in RHOAI dashboard as a Data Science Project)
oc apply -f manifests/07/dybbol-project.yaml

# 7b — Primary: Qwen2.5-3B-Instruct from HuggingFace (namespace: dybbol, MaaS-ready, fits T4 GPU)
cp user.env.example user.env        # fill in your HF_TOKEN
oc create secret generic huggingface-token \
  -n dybbol \
  --from-env-file=user.env
oc apply -f manifests/07/llm-inference-qwen25-3b-instruct.yaml
# NOTE: If your cluster has GPUs with ≥20 GiB VRAM (A100, H100), use the 7B OCI variant instead:
# oc apply -f manifests/07/llm-inference-qwen25-7b-instruct.yaml

# 7c — (Optional) Quick-start: Qwen3-0.6B from HuggingFace (creates my-first-model namespace inline, NOT MaaS-ready)
oc create secret generic huggingface-token \
  -n my-first-model \
  --from-env-file=user.env
oc apply -f manifests/07/llm-inference.yaml

# Step 8 — Enable dashboard features
# 8a — Gen AI Studio (UI flag in OdhDashboardConfig)
oc patch OdhDashboardConfig odh-dashboard-config \
  -n redhat-ods-applications \
  --type=merge \
  --patch-file manifests/08/odh-dashboard-config-enable-aistudio.yaml
# 8b — Additional UI features: Training Jobs, LLM Gateway field, MCP Catalog, Prompt Management
oc patch OdhDashboardConfig odh-dashboard-config \
  -n redhat-ods-applications \
  --type=merge \
  --patch-file manifests/08/odh-dashboard-config-enable-features.yaml
# 8c — MLflow: (1) enable operator in DSC; (2) create server instance (operator does not auto-create it)
oc patch DataScienceCluster default-dsc \
  --type=merge \
  --patch-file manifests/08/rhoai-dsc-enable-mlflow.yaml
oc apply -f manifests/08/mlflow-instance.yaml
# 8d — Models-as-a-Service (MaaS): requires gateway + DSC change + UI flag
oc apply -f manifests/08/maas-gateway.yaml           # GatewayClass + maas-default-gateway (required before DSC reconcile)
oc patch DataScienceCluster default-dsc \
  --type=merge \
  --patch-file manifests/08/rhoai-dsc-enable-maas.yaml
oc patch OdhDashboardConfig odh-dashboard-config \
  -n redhat-ods-applications \
  --type=merge \
  --patch-file manifests/08/odh-dashboard-config-enable-maas.yaml

# Step 9 — MaaS infrastructure prerequisites: maas-db-config Secret + Authorino + optional User Workload Monitoring
# Note: Steps 8d gateway/DSC/UI must be applied before or after Step 9 — both must be complete for
# ModelsAsServiceReady to become True. The order between 8d and 9 does not matter.
# PostgreSQL is already running from Step 4 — no re-deploy needed.

# 9a — Create the maas-db-config Secret (PostgreSQL deployed in Step 4)
oc apply -f manifests/09/maas-db-config.yaml

# 9b — Deploy Authorino instance in redhat-ods-applications
oc apply -f manifests/09/authorino-instance.yaml
oc get pods -n redhat-ods-applications -l authorino-resource=authorino -w

# 9c — (Optional) Enable User Workload Monitoring
oc apply -f manifests/09/cluster-monitoring-config.yaml

# Verify ModelsAsService is Ready (allow 2-3 min after all prerequisites exist)
oc get DataScienceCluster default-dsc \
  -o jsonpath='{.status.conditions[?(@.type=="ModelsAsServiceReady")]}'

# 9d — Create DNS record so maas.<apps-domain> routes to the maas-default-gateway ELB
# The wildcard *.apps DNS points to the OpenShift Router, not the maas-default-gateway ELB.
# Without this record the dashboard API keys panel returns 503 (wrong backend).
MAAS_ELB=$(oc get svc maas-default-gateway-maas-gateway-class -n openshift-ingress \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
APPS_DOMAIN=$(oc get ingresses.config cluster -o jsonpath='{.spec.domain}')
cat <<EOF | oc apply -f -
apiVersion: ingress.operator.openshift.io/v1
kind: DNSRecord
metadata:
  name: maas-api-dns
  namespace: openshift-ingress-operator
spec:
  dnsManagementPolicy: Managed
  dnsName: "maas.${APPS_DOMAIN}."
  recordTTL: 30
  recordType: CNAME
  targets:
    - ${MAAS_ELB}
EOF
# Verify DNS is published to Route53 (status.zones[*].conditions[*].type=Published)
oc get dnsrecord maas-api-dns -n openshift-ingress-operator \
  -o jsonpath='{.status.zones[0].conditions[0]}'

# 9e — Register the deployed model with MaaS by creating a MaaSModelRef
# MaaSModelRef is the bridge between LLMInferenceService and the MaaS control plane.
# Without it, Subscriptions and Authorization Policies cannot reference any model.
oc apply -f manifests/09/maas-model-ref.yaml
oc get maasmodelref qwen25-3b-instruct -n dybbol \
  -o jsonpath='{.status.phase}'   # expect: Ready

# 9f — Create Subscription and AuthPolicy declaratively (no dashboard clicking needed)
# IMPORTANT: both CRs must go in the models-as-a-service namespace (not redhat-ods-applications).
# The MaaS controller watches models-as-a-service for these resources.
oc apply -f manifests/09/maas-subscription.yaml
oc apply -f manifests/09/maas-auth-policy.yaml
# Verify both reach Active phase (~10s)
oc get maassubscription,maasauthpolicies -n models-as-a-service

# 9g — Generate an API key in the dashboard (preferred) or via curl (quick test):
#   Dashboard: Gen AI Studio → API keys → Create API key → copy the Bearer token (sk-oai-* prefix)
# OR create an API key via curl (uses your OpenShift session to authenticate the creation request):
APPS_DOMAIN=$(oc get ingresses.config cluster -o jsonpath='{.spec.domain}')
API_KEY=$(curl -sk \
  -H "Authorization: Bearer $(oc whoami -t)" \
  -H "Content-Type: application/json" \
  -X POST \
  -d '{"name":"cli-test","description":"CLI test key","subscriptions":[{"name":"qwen25-3b-instruct-subscription"}]}' \
  "https://maas.${APPS_DOMAIN}/maas-api/v1/api-keys" | python3 -c "import sys,json;print(json.load(sys.stdin)['key'])")
echo "API key: $API_KEY"
# List available models (OpenShift token works for /v1/models)
curl -sk -H "Authorization: Bearer $(oc whoami -t)" \
  https://maas.${APPS_DOMAIN}/v1/models | jq .
# Chat completions — IMPORTANT: inference paths require a MaaS API key (sk-oai-*), not an OpenShift token.
# The MaaSAuthPolicy restricts kubernetes-tokens to /v1/models paths only; inference paths require sk-oai-* keys.
curl -sk -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen25-3b-instruct","messages":[{"role":"user","content":"What is 2+2?"}],"max_tokens":64}' \
  https://maas.${APPS_DOMAIN}/dybbol/qwen25-3b-instruct/v1/chat/completions | jq .
```

The `scratch/` directory is gitignored and used for generated/temporary cluster artifacts.
The `user.env` file is gitignored — copy `user.env.example` and fill in your token; never commit `user.env`.

## Manifest Structure

| Directory | Purpose |
|-----------|---------|
| `manifests/01/` | NFD Operator, NVIDIA GPU Operator, optional GPU sample workload |
| `manifests/02/` | KServe prerequisite operators: Service Mesh 3, Serverless, Authorino, Cert-Manager, JobSet, Red Hat Connectivity Link, LeaderWorkerSet |
| `manifests/03/` | RHOAI Operator (`rhods-operator`) and `DataScienceCluster` CRD |
| `manifests/04/` | Shared PostgreSQL instance (`redhat-ods-applications`) + Model Registry CR (`rhoai-model-registries`). PostgreSQL serves both the Model Registry (`registry` DB) and MaaS (`maasdb` DB). Kustomize. |
| `manifests/05/` | DSC patches and llm-d Gateway: LlamaStack enablement patch + `openshift-ai-inference` GatewayClass/Gateway |
| `manifests/06/` | GPU `HardwareProfile` for RHOAI workloads (`gpu-profile` in `redhat-ods-applications`) |
| `manifests/07/` | `dybbol` project namespace + `LLMInferenceService` examples: Qwen3-0.6B (HuggingFace) and Qwen2.5-7B-Instruct (Red Hat OCI) |
| `manifests/08/` | Dashboard feature enablement: Gen AI Studio, additional UI flags, MLflow (operator + instance), and MaaS (gateway + DSC + UI flag) |
| `manifests/09/` | MaaS infrastructure prerequisites: `maas-db-config` Secret, Authorino instance, optional User Workload Monitoring, DNS record template for maas gateway, `MaaSModelRef` to register the deployed model, `MaaSSubscription` and `MaaSAuthPolicy` to expose the model for API key issuance. PostgreSQL was moved to `manifests/04/`. |
| `manifests/10/` | MCP tools for the Playground: `gen-ai-aa-mcp-servers` ConfigMap (Fetch, Sequential-Thinking, Weather, Postgres-Airman) + `pg-airman-mcp.yaml` (in-cluster PostgreSQL MCP server in `dybbol`, restricted/read-only, Streamable HTTP `POST /mcp` on port 8000) |
| `manifests/mysql/` | Archived MySQL manifests (no longer in the main installation path; kept for reference) |

## Key Configuration Details

- **RHOAI channel**: `stable-3.x` — must be set explicitly; the default `stable` channel still pins to v2.x.
- **Service Mesh**: uses `servicemeshoperator3` (not the legacy v2 operator name).
- **DataScienceCluster (`manifests/03/rhoai-operator-dsc.yaml`)**: controls which RHOAI components are `Managed` vs `Removed`. KServe raw deployment is set to `Headless`; Kueue and MLflow are `Removed` by default.
- **Shared PostgreSQL** (`manifests/04/`): one `registry.redhat.io/rhel9/postgresql-16` instance in `redhat-ods-applications` serves two databases — `maasdb` (MaaS) and `registry` (Model Registry). The `maas-postgresql-credentials` Secret (env vars `POSTGRESQL_USER/PASSWORD/DATABASE`) provisions the primary `maasdb`/`maas` user. An init script mounted at `/usr/share/container-scripts/postgresql/start/init-registry-db.sh` creates the `registry` database and `registry` role on first initialization. Data path: `/var/lib/pgsql/data`. `POSTGRESQL_ADMIN_PASSWORD` is set for emergency postgres superuser access.
- **Model Registry namespace**: `rhoai-model-registries` (matches `registriesNamespace` in the DSC). The `ModelRegistry` CR uses `spec.postgres` and connects cross-namespace via `maas-postgresql.redhat-ods-applications.svc.cluster.local`. The `registry-postgresql-credentials` Secret must exist in `rhoai-model-registries` so the operator can read it — a copy is deployed by the step 4 Kustomize bundle. Verify CRD support before applying: `oc explain ModelRegistry.spec --api-version=modelregistry.opendatahub.io/v1beta1` (expect a `postgres` field). Note: two CRDs share the `ModelRegistry` Kind — always specify the API version or use the fully-qualified resource `modelregistries.modelregistry.opendatahub.io` to avoid resolving to `components.platform.opendatahub.io` by mistake.
- **LlamaStack**: enabled via a `--type=merge` patch to `DataScienceCluster`; do NOT use `oc patch --patch-file` without `--type=merge` — the CRD only supports merge-patch and json-patch, not strategic-merge-patch.
- **llm-d Gateway** (`manifests/05/openshift-ai-inference-gateway.yaml`): `LLMInferenceService` defaults to a Gateway named `openshift-ingress/openshift-ai-inference` which must be created manually. Uses a dedicated `openshift-ai-inference` GatewayClass (controller: `openshift.io/gateway-controller/v1`) separate from the RHOAI data-science gateway. TLS reuses the `data-science-gateway-service-tls` secret. `allowedRoutes.from: All` permits HTTPRoutes from any model namespace.
- **GPU HardwareProfile** (`manifests/06/gpu-hardware-profile.yaml`): creates `gpu-profile` in `redhat-ods-applications`; referenced by `LLMInferenceService` via the `opendatahub.io/hardware-profile-name` annotation.
- **HuggingFace token**: `hf://` model URIs in `LLMInferenceService` require a `huggingface-token` Secret (key: `HF_TOKEN`) in the model namespace. Without it, anonymous downloads are rate-limited and the `storage-initializer` init container stalls silently. Copy `user.env.example` → `user.env`, fill in the token, then `oc create secret generic huggingface-token -n <ns> --from-env-file=user.env`. The `user.env` file is gitignored.
- **OdhDashboardConfig opt-in flags**: RHOAI 3.4 has many boolean flags in `OdhDashboardConfig.spec.dashboardConfig` that are hidden by default. The two categories are: `disable*` flags (default `false` = feature visible; set `true` to hide) and non-`disable` flags (opt-in; must be set `true` to appear). All the features in `manifests/08/` use one or both resources. Verify all current flags with `oc get OdhDashboardConfig odh-dashboard-config -n redhat-ods-applications -o jsonpath='{.spec.dashboardConfig}'`.
- **Gen AI Studio** (`manifests/08/odh-dashboard-config-enable-aistudio.yaml`): sets `genAiStudio: true` in the `OdhDashboardConfig` resource — **not** in `DataScienceCluster`. This is a common point of confusion: the DSC controls which operator components run; `OdhDashboardConfig` controls which UI features are visible. The flag defaults to `false` in RHOAI 3.4, so "AI Studio" never appears in the dashboard navigation until it is explicitly set. To verify: `oc get OdhDashboardConfig odh-dashboard-config -n redhat-ods-applications -o jsonpath='{.spec.dashboardConfig}'`. Apply with `--type=merge`.
- **Additional UI flags** (`manifests/08/odh-dashboard-config-enable-features.yaml`): enables `trainingJobs`, `llmGatewayField`, `mcpCatalog`, and `promptManagement` in one patch. All are opt-in booleans in `OdhDashboardConfig`. Prerequisites already met: `trainer: Managed`, llm-d gateway deployed, `llamastackoperator: Managed`.
- **MLflow** (`manifests/08/rhoai-dsc-enable-mlflow.yaml` + `manifests/08/mlflow-instance.yaml`): enabling MLflow requires **two separate steps**. First, set `mlflowoperator: Managed` in `DataScienceCluster` to deploy the operator (the `mlflow` flag in `OdhDashboardConfig` is deprecated — ignore it). Second, create the cluster-scoped `MLflow` CR (`mlflow-instance.yaml`) — the operator does not auto-create an instance. Without the `MLflow` CR, the dashboard shows "MLflow is currently unavailable" even though the operator pod is running. The instance uses SQLite + PVC for persistence (no S3 needed) and `workspaceLabelSelector: opendatahub.io/dashboard: "true"` to expose all RHOAI data science project namespaces. Verify: `oc get mlflow mlflow -o jsonpath='{.status}'`.
- **Models-as-a-Service / MaaS** (`manifests/08/maas-gateway.yaml` + `manifests/08/rhoai-dsc-enable-maas.yaml` + `manifests/08/odh-dashboard-config-enable-maas.yaml`): MaaS requires **three steps**. (1) Create `maas-default-gateway` in `openshift-ingress` — the name is hardcoded; the DSC reconcile fails with "GatewayNotReady" if it doesn't exist first. (2) Set `kserve.modelsAsService: Managed` in DSC. (3) Set `modelAsService: true` in `OdhDashboardConfig`. Kuadrant (RHCL) must be installed — it is (Step 2). Verify: `oc get DataScienceCluster default-dsc -o jsonpath='{.status.conditions[?(@.type=="ModelsAsServiceReady")]}'`.
- **MaaS infrastructure prerequisites** (`manifests/09/`): even after the three MaaS steps above, `ModelsAsServiceReady` stays `False (PrerequisitesNotMet)` until three more things exist in `redhat-ods-applications`: (1) the `maas-db-config` Secret with a valid PostgreSQL `DB_CONNECTION_URL`, (2) a running Authorino instance, and (3) optionally the `cluster-monitoring-config` ConfigMap in `openshift-monitoring`. The Secret and Authorino instance are in `manifests/09/`. PostgreSQL is already running from Step 4.
- **`maas-db-config` Secret** (`manifests/09/maas-db-config.yaml`): key `DB_CONNECTION_URL`, value format `postgresql://user:pass@host:5432/db?sslmode=disable`. References the shared PostgreSQL deployed in Step 4 at `maas-postgresql.redhat-ods-applications.svc.cluster.local:5432`, database `maasdb`, user `maas`.
- **Authorino instance for MaaS** (`manifests/09/authorino-instance.yaml`): The Authorino Operator (installed in Step 2.3) requires an `Authorino` CR (`operator.authorino.kuadrant.io/v1beta1`) deployed in `redhat-ods-applications`. `clusterWide: true` lets it watch `AuthConfig` CRs across all model namespaces. TLS is disabled on the listener — plain HTTP is sufficient for in-cluster MaaS communication.
- **PostgreSQL image (shared)**: `registry.redhat.io/rhel9/postgresql-16` — data path is `/var/lib/pgsql/data` (Red Hat convention). Must be PG16+: Model Registry migration 25 uses `IS JSON ARRAY` syntax unavailable in PostgreSQL 15. Primary credentials come from `maas-postgresql-credentials`; the `maas-db-config` Secret must use the same password for the `maas` user. The `registry` user credentials are in `registry-postgresql-credentials` (deployed to both `redhat-ods-applications` and `rhoai-model-registries`).
- **Kuadrant CR** (`manifests/02/kuadrant-instance.yaml`): The RHCL (Kuadrant) operator alone is not enough — a `Kuadrant` CR must be created in `kuadrant-system` to activate the operator and wire up its components (Authorino, Limitador, DNS Operator). Without the CR, `AuthPolicy` resources are accepted but never enforced (`"kuadrant is not installed, please create resource"`), meaning Kuadrant's Authorino is never started and no `AuthConfig` resources are generated. Apply immediately after the RHCL operator is ready. Verify: `oc get kuadrant kuadrant -n kuadrant-system -o jsonpath='{.status.conditions[0].message}'` → `"Kuadrant is ready"`.
- **`opendatahub.io/connections` annotation portability**: The RHOAI dashboard injects an `opendatahub.io/connections: <secret-name>` annotation when you create a `LLMInferenceService` or `InferenceService` through the UI. The mutating webhook `connection-llmisvc.opendatahub.io` (failurePolicy: Fail) validates that the named secret exists in the target namespace on every CREATE/UPDATE — if it doesn't, the apply is denied. Dashboard-generated secret names (e.g. `secret-6a153e`) are auto-generated and namespace-specific; never commit them to manifests. Remove the annotation for images pulled via the cluster pull secret (`registry.redhat.io`, `quay.io`, etc.). Only include it when you have pre-created a named data connection secret in the target namespace and want the dashboard to display the connection.
- **`dybbol` project namespace** (`manifests/07/dybbol-project.yaml`): The model deployment namespace must carry the label `opendatahub.io/dashboard: "true"` to appear as a Data Science Project in the RHOAI dashboard. Without this label the namespace exists but is invisible to the dashboard. The label `modelmesh-enabled: "false"` tells RHOAI to use KServe/llm-d rather than ModelMesh for serving. Apply with `oc apply -f manifests/07/dybbol-project.yaml` before deploying the `LLMInferenceService` — the namespace must exist first.
- **`MaaSModelRef`** (`manifests/09/maas-model-ref.yaml`): After the MaaS control plane is ready (`ModelsAsServiceReady: True`) and a `LLMInferenceService` is running, a `MaaSModelRef` CR must be created in the **same namespace** as the inference service to register it with MaaS. Without this CR, the Subscriptions and Authorization Policies forms in the dashboard show "No models available" and API keys cannot be issued. The spec uses `spec.modelRef.kind: LLMInferenceService` and `spec.modelRef.name: <service-name>`. Verify: `oc get maasmodelref -n dybbol -o jsonpath='{.items[0].status.phase}'` → `Ready`. Once Ready, the model appears in the Subscriptions dropdown and the full admin flow (Subscription → Authorization Policy → API key → curl) becomes available.
- **MaaS requires `maas-default-gateway` in `LLMInferenceService`**: A `LLMInferenceService` intended for MaaS must explicitly set `spec.router.gateway.refs` to `maas-default-gateway` in `openshift-ingress`. The default (`gateway: {}`) wires the HTTPRoute to `openshift-ai-inference` (the llm-d gateway from Step 5). The `MaaSModelRef` controller validates the HTTPRoute's `parentRefs` and sets phase `Failed` with "does not reference gateway (expected: openshift-ingress/maas-default-gateway)" if the default is left in place. See `manifests/07/llm-inference-qwen25-7b-instruct.yaml` for the correct `router.gateway.refs` pattern.
- **MaaS DNS routing** (`manifests/09/maas-dns-record.yaml`): The `maas-ui` sidecar inside the dashboard auto-discovers the MaaS API URL as `maas.<apps-domain>` (derived from `GATEWAY_DOMAIN` env var). On clusters where `*.apps` is a wildcard CNAME to the OpenShift Router, this URL resolves to the Router — not the `maas-default-gateway` ELB. The Router has no backend for this host and returns 503, causing the dashboard API keys panel to fail. Fix: create a specific `DNSRecord` (ingress.operator.openshift.io/v1) for `maas.<apps-domain>` as a CNAME to the `maas-default-gateway` ELB. This record overrides the wildcard in Route53 and is managed by the same ingress operator that manages the `*.apps` wildcard. The manifest in `manifests/09/maas-dns-record.yaml` is a template with `<APPS_DOMAIN>` and `<MAAS_ELB>` placeholders — use the dynamic command in Step 9d instead of applying the template directly.
- **Kuadrant Authorino missing RBAC (three ClusterRoleBindings needed)**: The Kuadrant operator deploys its own Authorino instance in `kuadrant-system` (SA: `kuadrant-system:authorino-authorino`) but is missing three critical ClusterRoleBindings that must be created manually. Note: the pre-existing `authorino-authorino` and `authorino-authorino-k8s-auth` CRBs both target `redhat-ods-applications:authorino-authorino` (the Step 9 Authorino), NOT the Kuadrant one. (1) **`tokenreviews` RBAC**: without `create tokenreviews` permission, every request through the MaaS gateway returns 401 with `"tokenreviews.authentication.k8s.io is forbidden"` in Authorino logs — `oc create clusterrolebinding authorino-authorino-kuadrant-k8s-auth --clusterrole=authorino-manager-k8s-auth-role --serviceaccount=kuadrant-system:authorino-authorino`. (2) **`authorino-manager-role` RBAC**: without this, the Kuadrant Authorino (running with `clusterWide: true`) cannot list `authconfigs` or `secrets` at the cluster scope — its in-memory cache stays empty and ALL ext-auth requests return `"Service not found"` (404). Fix: `oc create clusterrolebinding authorino-authorino-kuadrant-manager --clusterrole=authorino-manager-role --serviceaccount=kuadrant-system:authorino-authorino`, then `oc delete pod -n kuadrant-system -l control-plane=controller-manager` (kuadrant-system Authorino pod, not operator) to reload AuthConfigs. (3) **Service-serving CA trust**: even with correct RBAC, Authorino can't call `maas-api`'s `/internal/v1/subscriptions/select` (TLS) because it doesn't trust the OpenShift service-serving CA — `subscription-valid` authorization fails with 403 on inference paths. Fix: patch the Authorino Deployment to mount `openshift-service-ca.crt` ConfigMap at `/etc/ssl/certs/service-ca.crt` via `subPath` (Go's cert pool on Linux checks `/etc/ssl/certs/` for individual `.crt` files). Without this fix, `/v1/models` returns `{"data":[]}` even with a Ready subscription. The Authorino Operator reconciles the Deployment on CR change — the subPath patch may be reverted; re-apply if needed.
- **GPU sizing for LLMInferenceService**: The Tesla T4 has 15 GiB VRAM. Models up to ~6B parameters in BF16 (~6 GiB weights) fit comfortably with room for KV cache. 7B+ models in BF16 fill the T4 entirely — vLLM crashes during torch.compile autotuning because there is no headroom left after loading the weights. No `--gpu-memory-utilization` tuning can fix this. Use GPUs with ≥20 GiB VRAM (A100, H100) for 7B+ models. The rhelai1 OCI registry currently offers no modelcar images smaller than 7B; use `hf://` URIs for smaller models on T4 clusters.
- **MaaS admin flow — declarative (Step 9f)**: `MaaSSubscription` and `MaaSAuthPolicy` CRDs (`maas.opendatahub.io/v1alpha1`) exist and must be created in the **`models-as-a-service` namespace** — the MaaS controller only watches that namespace. Putting them in `redhat-ods-applications` leaves PHASE empty (never reconciled). `spec.owner.groups` in `MaaSSubscription` and `spec.subjects.groups` in `MaaSAuthPolicy` accept OpenShift group names (`system:authenticated` for all logged-in users, or any specific group). The `MaaSAuthPolicy` controller generates a Kuadrant `AuthPolicy` named `maas-auth-<model>` in the model namespace (`dybbol`). Verify: `oc get maassubscription,maasauthpolicies -n models-as-a-service` → both PHASE `Active`. After this, `/v1/models` returns `200` (with OpenShift token OR API key) and chat completions work at the namespace-prefixed path: `https://maas.<apps-domain>/<model-ns>/<model-name>/v1/chat/completions` with a `sk-oai-*` API key. **Note**: OpenShift tokens (`oc whoami -t`) only work for `/v1/models`; inference paths require a MaaS API key. Create one: `POST /maas-api/v1/api-keys` with Bearer `$(oc whoami -t)` (see Step 9g).
- **MaaSSubscription `Degraded` → fix via stub CRD + LimitadorOperator CR**: If `MaaSSubscription` shows `Degraded` with `[Limitador Operator] is not installed`, this is a known RHCL 1.4 bug. The Kuadrant Operator (rhcl-operator.v1.4.0) checks for the `limitadoroperators.limitador.kuadrant.io` CRD to confirm Limitador Operator readiness, but this CRD was removed in Limitador Operator 1.x. Complete fix: (1) `manifests/02/limitador-instance.yaml` creates the `Limitador` CR to start the Limitador pod; (2) create a stub cluster-scoped `limitadoroperators.limitador.kuadrant.io` CRD (`oc create -f` with `scope: Cluster`); (3) create a cluster-scoped `LimitadorOperator` CR (`apiVersion: limitador.kuadrant.io/v1alpha1, kind: LimitadorOperator, metadata.name: limitador`); (4) delete the Kuadrant Operator pod to force re-reconciliation: `oc delete pod -n kuadrant-system -l control-plane=controller-manager` **targeting only the kuadrant-operator pod** (not the Authorino pod — use the pod name prefix `kuadrant-operator-controller-manager-*`). After the operator restarts, the `TokenRateLimitPolicy` changes from `NotAccepted` to `Accepted` and `MaaSSubscription` transitions to `Active`. `oc rollout restart` is reverted by OLM; direct pod deletion is necessary. Note: without this fix, the subscription stays Degraded and `/v1/models` returns an empty list (the maas-api only lists models from Active subscriptions).
- **LlamaStack playground — vLLM endpoint scheme**: The RHOAI dashboard auto-generates the `llama-stack-config` ConfigMap with `http://` for the vLLM workload service URL (`<name>-kserve-workload-svc`). The llm-d workload service is TLS-only (`appProtocol: https`, port 8000) and requires `https://`. Fix: patch the `llama-stack-config` ConfigMap to replace `http://` with `https://` in `base_url`, then restart the `LlamaStackDistribution` deployment. The `VLLM_TLS_VERIFY=false` env var disables cert validation but does not fix the wrong scheme. Applied to `lsd-genai-playground` in `dybbol`.
- **LlamaStack playground — `VLLM_MAX_TOKENS` vs `max_model_len`**: The dashboard sets `VLLM_MAX_TOKENS=4096` and `--max-model-len=4096` to the same value. vLLM treats `max_tokens` as the maximum number of *output* tokens; it is added to the input token count and the sum must not exceed `max_model_len`. Setting both to 4096 leaves zero room for any input prompt — every request fails with HTTP 400 → Playground shows an empty response bubble with "Show metrics" but no text. Fix: patch `VLLM_MAX_TOKENS` in `spec.server.containerSpec.env` on the `LlamaStackDistribution` CR to 2048. With `--max-model-len=32768` already in the vLLM manifest, this leaves 30720 tokens of input budget — sufficient for both normal chat and RAG with multiple files. Patch via `oc patch llamastackdistribution <name> -n <ns> --type=json -p '[{"op":"replace","path":"/spec/server/containerSpec/env/<index>/value","value":"2048"}]'` — patching the Deployment directly is overwritten by the operator. Applied to `lsd-genai-playground` in `dybbol` (index 3).
- **LlamaStack playground — RAG requires vLLM tool-calling flags, wide context window, and VLLM_MAX_TOKENS bump**: (1) `--enable-auto-tool-choice` + `--tool-call-parser hermes` in LLMInferenceService: LlamaStack sends `tool_choice: "auto"` when invoking `file_search`; without these vLLM returns HTTP 400. (2) `--max-model-len=32768` (Qwen2.5-3B native limit): RAG retrieves chunks from ALL uploaded files simultaneously — with 3+ utility bill PDFs the injected context easily reaches 7000+ tokens. Even 8192 is insufficient for multi-file sessions. 32768 gives 30720 tokens of input budget (with VLLM_MAX_TOKENS=2048), covering the Playground's 10-file upload cap. T4 safety: KV cache for 32768 tokens ≈ 4.5 GiB vs ~6.75 GiB available; `--enforce-eager` prevents torch.compile OOM. Both vLLM flags are in `manifests/07/llm-inference-qwen25-3b-instruct.yaml`. (3) `VLLM_MAX_TOKENS=2048` in LlamaStackDistribution (patch index 3): set directly to 2048 in the initial fix step (README 11.2) — no separate RAG patch needed. Runtime patch — no manifest. The 7B manifest keeps `max-model-len=4096` — 7B exhausts T4 VRAM; RAG on 7B requires ≥20 GiB VRAM. System prompts cannot reduce input token consumption (chunks are injected before inference) but can reduce output tokens and improve focus.
- **LlamaStack playground — RAG vector stack is ready out of the box**: The `rh-dev` distribution used by the auto-generated `LlamaStackDistribution` already includes all RAG components: `inline::milvus` (vector IO), `inline::sentence-transformers` with IBM Granite `granite-embedding-125m-english` (768-dim embeddings), `inline::file-search` (tool runtime), and `inline::localfs` (file storage). The `vector_io` API and `vector_stores.default_provider_id: milvus` are pre-configured in the `llama-stack-config` ConfigMap. Files and embeddings are stored on ephemeral pod storage and lost on pod restart (no PVC configured).
- **MaaS authentication model — API key vs OpenShift token**: The MaaS gateway (`maas.<apps-domain>`) accepts different tokens per path: (1) `/v1/models` accepts both OpenShift tokens (`oc whoami -t`, via `openshift-identities` method) and `sk-oai-*` MaaS API keys; (2) inference paths (`/v1/chat/completions`, `/v1/completions`, etc.) **only** accept MaaS API keys (`sk-oai-*`). The `MaaSAuthPolicy` generated `AuthPolicy` restricts `kubernetes-tokens` auth to paths matching `.*/v1/models$` — this is intentional so inference calls use rate-limited API keys. Create an API key via dashboard (Gen AI Studio → API keys) or via `POST /maas-api/v1/api-keys` with an OpenShift Bearer token (see Step 9g). The **Playground** bypasses the MaaS gateway entirely — it connects directly to the LlamaStack distribution inside the cluster. No API key is needed for the Playground. OpenShift tokens expire (~24 h); MaaS API keys are long-lived (1–365 days) and better suited for persistent integrations.
- **pg-airman-mcp — in-cluster PostgreSQL MCP server** (`manifests/10/pg-airman-mcp.yaml`): Deploys EnterpriseDB's `pg-airman-mcp` as a Deployment+Service in `dybbol`, connecting to `maas-postgresql.redhat-ods-applications.svc.cluster.local:5432` (the shared PostgreSQL from Step 4). Runs with `--transport=streamable-http --streamable-http-port=8000 --access-mode=restricted` — Streamable HTTP endpoint at `POST /mcp` on port 8000, no write operations allowed. A dedicated `pg-airman-mcp-db` Secret in `dybbol` holds the `AIRMAN_MCP_DATABASE_URL` with `maasdb` credentials — created imperatively via `oc create secret generic` reading the password from `maas-postgresql-credentials` (not committed to the manifest to avoid credential leakage). This mirrors `maas-db-config` (in `redhat-ods-applications`) but must live in `dybbol` because Kubernetes does not allow Pod `env.valueFrom.secretKeyRef` to cross namespace boundaries. Registered in `gen-ai-aa-mcp-servers` as `Postgres-Airman` with URL `http://pg-airman-mcp.dybbol.svc.cluster.local:8000/mcp` (Streamable HTTP — RHOAI Playground uses POST /mcp, not SSE; using `--transport=sse` caused 405 errors). The entrypoint script auto-adds `--streamable-http-host=0.0.0.0`; `--streamable-http-port=8000` is set explicitly in args to pin port 8000 (otherwise entrypoint defaults to 8080). It also detects `KUBERNETES_SERVICE_HOST` and skips localhost remapping. The RHOAI Playground shows an "Authorize MCP server" dialog asking for an Access token when enabling any in-cluster MCP server; since pg-airman-mcp runs without `--auth-*` flags it accepts all connections without validating the Bearer token — enter any non-empty placeholder (e.g. `none`) and click Authorize. Image UID 1000 with world-readable venv files is compatible with OpenShift's namespace-assigned UID under `restricted-v2` SCC; if permission errors occur at runtime, add `securityContext.runAsUser: 1000` and bind a permissive SCC to the pod's ServiceAccount.
- **Workbench PyTorch GPU demo — YOLOv5 transfer learning** (`rh-aiservices-bu/yolov5-transfer-learning`): The recommended Workbench demo for RHOAI. Uses the `Jupyter | PyTorch | CUDA | Python 3.12` workbench image with 1 GPU (T4 is sufficient). Two notebooks: `01-data_preparation.ipynb` downloads 300 images/class (Bicycle, Car, Traffic sign) from Google Open Images and renders bounding boxes inline; `02-model_training.ipynb` clones Ultralytics YOLOv5, freezes the backbone (10 layers), and runs transfer learning — weights saved to `yolov5/runs/train/exp/weights/best.pt`. Training takes ~15–20 min on a T4. The demo already sets `num_workers=0` so the default container `/dev/shm` limit does not cause crashes. For custom PyTorch projects with multiple DataLoader workers, mount an `emptyDir` with `medium: Memory` at `/dev/shm` via the `Notebook` CR YAML (edit in OpenShift Console or patch with `oc`). PVC: 10 GiB in the `dybbol` namespace.
- **MCP servers in the Playground — `gen-ai-aa-mcp-servers` ConfigMap** (`manifests/10/gen-ai-aa-mcp-servers.yaml`): MCP (Model Context Protocol) tools are registered via a ConfigMap named `gen-ai-aa-mcp-servers` in `redhat-ods-applications`. Each key is the tool's display name in the Playground; each value is a JSON object with `url` (the MCP server's SSE or HTTP endpoint) and `description`. The `rh-dev` LlamaStack distribution ships the `remote::model-context-protocol` provider pre-installed — no changes to the `llama-stack-config` ConfigMap or `LlamaStackDistribution` spec are needed. Prerequisites: `mcpCatalog: true` in `OdhDashboardConfig` (Step 8.2) and `--enable-auto-tool-choice --tool-call-parser hermes` on vLLM (already in `manifests/07/llm-inference-qwen25-3b-instruct.yaml`). After applying the ConfigMap, restart the `lsd-genai-playground` Deployment for LlamaStack to connect to the new servers. In the Playground, MCP tools appear as toggles under **Configure → MCP** (or **Tools**). Verify registered tool groups: `oc exec -n dybbol deployment/lsd-genai-playground -- curl -s http://localhost:8321/v1/toolgroups | jq '.data[].identifier'`.

## Pending Work

- [x] **Test MaaS end-to-end** — Subscription + AuthPolicy created declaratively; `curl /dybbol/qwen25-3b-instruct/v1/chat/completions` returns `200` with a valid response using `oc whoami -t` as Bearer token. Dashboard API keys page should now load correctly.
- [x] **Declarative Subscription / Authorization Policy manifests** — `MaaSSubscription` and `MaaSAuthPolicy` CRDs confirmed present. Manifests added to `manifests/09/maas-subscription.yaml` and `manifests/09/maas-auth-policy.yaml`. **Critical**: both must be in `models-as-a-service` namespace (not `redhat-ods-applications`) — the MaaS controller only watches `models-as-a-service`.
- [x] **`llm-inference.yaml` (HuggingFace / Qwen3-0.6B) namespace alignment** — decided: stays in `my-first-model` (inline in the manifest) as a standalone optional quick-start. `dybbol` is the primary MaaS-ready namespace for `llm-inference-qwen25-7b-instruct.yaml`.
- [x] **HuggingFace example MaaS-readiness** — decided: out of scope. `llm-inference.yaml` is a community quick-start using the default `openshift-ai-inference` gateway; it is not intended for MaaS. Only `llm-inference-qwen25-7b-instruct.yaml` is MaaS-ready.

## Contributing

Work on the `contribution` branch (see `contribute.md`). PRs go to `IsaiahStapleton/rhoai-config-guide` upstream. The `scratch/` directory must not be committed.
