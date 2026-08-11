# Dynamic Scoring Framework on ACM / OpenShift

End-to-end guide for installing the Dynamic Scoring Framework on an ACM hub running OpenShift, deploying the sample scorer, and verifying that scores flow back to the hub.

## Prerequisites

- An OpenShift cluster running ACM (Advanced Cluster Management) as the hub
- At least one managed cluster registered with ACM
- `oc` (or `kubectl`) and `helm` CLI tools
- OpenShift built-in monitoring enabled on managed clusters (`openshift-monitoring` namespace with `prometheus-k8s` service)

## 1. Install the Dynamic Scoring Framework

Install via Helm on the hub cluster:

```bash
helm repo add ocm https://open-cluster-management.io/helm-charts
helm repo update
helm upgrade --install dynamic-scoring-framework ocm/dynamic-scoring-framework \
  --namespace open-cluster-management --create-namespace
```

Verify that both controllers are running on the hub:

```bash
oc get pods -n open-cluster-management | grep dynamic
```

You should see two pods:
- `dynamic-scoring-framework-controller-*` (resource controller)
- `dynamic-scoring-addon-controller-*` (addon controller)

## 2. Fix the ManagedClusterSet Binding

The Helm chart creates a `ManagedClusterSetBinding` for the `global` cluster set, but ACM typically places managed clusters in the `default` set. If the addon agent is not deploying to your managed clusters, this is almost certainly the cause.

### Diagnose

```bash
# Check which cluster set your managed clusters belong to
oc get managedclusters -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels.cluster\.open-cluster-management\.io\/clusterset}{"\n"}{end}'

# Check how many clusters the Placement has selected (expect 0 if broken)
oc get placement dynamic-scoring-placement -n open-cluster-management -o jsonpath='{.status.numberOfSelectedClusters}'
```

If your clusters are in the `default` set and the Placement selects 0, apply the fix:

### Fix

Create a binding for the `default` cluster set:

```bash
oc apply -f - <<EOF
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSetBinding
metadata:
  name: default
  namespace: open-cluster-management
spec:
  clusterSet: default
EOF
```

Update both Placements to target `default`:

```bash
oc patch placement dynamic-scoring-placement -n open-cluster-management \
  --type merge -p '{"spec":{"clusterSets":["default"]}}'

oc patch placement dynamic-scoring-placement-aarch64 -n open-cluster-management \
  --type merge -p '{"spec":{"clusterSets":["default"]}}'
```

### Verify

```bash
# Placement should now select your managed clusters
oc get placement dynamic-scoring-placement -n open-cluster-management \
  -o jsonpath='{.status.numberOfSelectedClusters}'

# ManagedClusterAddOn should appear for each managed cluster
oc get managedclusteraddons -A | grep dynamic
```

Wait a minute or two for the addon framework to deploy the agent. Then confirm the agent pod is running on each managed cluster:

```bash
oc get pods -n dynamic-scoring --context <managed-cluster-context>
```

## 3. Deploy the Sample Scorer

The sample scorer is a simple FastAPI service that accepts Prometheus time-series data and returns an average CPU score. The recommended approach is to deploy it on the **hub cluster** with an HTTP Route — agents on all managed clusters POST their metrics to this single endpoint. No cross-cluster networking (Skupper) is required.

### Deploy on the hub cluster (recommended)

Create the namespace and apply the Deployment, Service, and Route directly on the hub:

```bash
oc create namespace dynamic-scoring
oc apply -f samples/sample-scorer/manifests/sample-scorer_ocp.yaml
```

### Alternative: Deploy on a managed cluster

If you prefer to run the scorer on a managed cluster instead, use ManifestWork to distribute the resources from the hub. Replace `<MANAGED_CLUSTER_NAME>` with your managed cluster's namespace on the hub (e.g., `dsf-mc`, `cluster1`):

```bash
export CLUSTER_NAME=<MANAGED_CLUSTER_NAME>
export SAMPLE_SCORER_IMAGE_NAME=quay.io/bjoydeep/sample-scorer:latest
envsubst < samples/sample-scorer/manifests/manifestwork_ocp.yaml | oc apply -f - -n open-cluster-management
```

### Build your own image (optional)

If you want to build the scorer image from source:

```bash
export SAMPLE_SCORER_IMAGE_NAME=quay.io/<your-org>/sample-scorer:latest
podman build --platform linux/amd64 -t $SAMPLE_SCORER_IMAGE_NAME samples/sample-scorer
podman push $SAMPLE_SCORER_IMAGE_NAME
```

**Note:** If building on an ARM machine (e.g., Apple Silicon Mac), you must pass `--platform linux/amd64` to produce an image compatible with x86_64 cluster nodes. Without it the pod will fail with `Exec format error`.

### Verify

Get the Route URL and test the scorer endpoints:

```bash
ROUTE_URL=$(oc get route sample-scorer -n dynamic-scoring -o jsonpath='http://{.spec.host}')
curl -sS $ROUTE_URL/healthz
curl -sS $ROUTE_URL/config | jq
```

The `/config` endpoint should return the scorer's configuration including its name, Prometheus source, and scoring parameters.

### Test scoring with sample data

You can test the scorer's `/scoring` endpoint without deploying the agent. In normal operation, the agent reads the scorer's `/config` to learn the Prometheus query, range, and step, queries Prometheus for matching time-series data, and POSTs that data to the scoring endpoint. The test script simulates this by skipping the Prometheus query and posting pre-built sample data directly — the data in `static/data.json` is shaped to match what Prometheus would return for the configured query (labels: `node`, `namespace`, `pod`; values: string-encoded floats at 60s intervals).

```bash
ROUTE_URL=$(oc get route sample-scorer -n dynamic-scoring -o jsonpath='http://{.spec.host}')
bash samples/sample-scorer/hack/test_scoring.sh $ROUTE_URL samples/sample-scorer/static/data.json
```

You should see a JSON response with computed scores (simple averages of the time-series values):

```json
{
  "results": [
    {
      "metric": {
        "node": "worker-1",
        "namespace": "default",
        "pod": "nginx-abc",
        "meta": "my_something_meta_by_sample_scorer"
      },
      "score": 0.07166666666666667
    },
    {
      "metric": {
        "node": "worker-1",
        "namespace": "kube-system",
        "pod": "coredns-xyz",
        "meta": "my_something_meta_by_sample_scorer"
      },
      "score": 1.6833333333333333
    }
  ]
}
```

## 4. Create a ServiceAccount Token Secret for Prometheus

OpenShift's built-in Prometheus (`prometheus-k8s` in `openshift-monitoring`) requires authentication. The Dynamic Scoring Agent needs a bearer token to query it.

The addon framework creates a ServiceAccount (`dynamic-scoring-agent-sa`) on each managed cluster with `cluster-admin` permissions, which covers Prometheus access. However, it does not create a long-lived token Secret for it. We need to create one so the agent can authenticate.

The Secret is **not mounted** into the agent pod — the agent reads it at runtime via the Kubernetes API using the secret name referenced in the DynamicScorer CR (Step 5). The `kubernetes.io/service-account-token` type with the `kubernetes.io/service-account.name` annotation tells Kubernetes to auto-populate the `token` data field with a valid bearer token for that SA.

Deploy the Secret to all managed clusters via ManifestWork from the hub (repeat for each cluster, or loop over them):

```bash
for CLUSTER in <MANAGED_CLUSTER_1> <MANAGED_CLUSTER_2>; do
oc apply -f - <<EOF
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: dynamic-scoring-agent-token
  namespace: ${CLUSTER}
spec:
  workload:
    manifests:
      - apiVersion: v1
        kind: Secret
        metadata:
          name: dynamic-scoring-agent-token
          namespace: dynamic-scoring
          annotations:
            kubernetes.io/service-account.name: dynamic-scoring-agent-sa
        type: kubernetes.io/service-account-token
EOF
done
```

Alternatively, apply the Secret directly on each managed cluster:

```bash
oc apply --context <managed-cluster-context> -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: dynamic-scoring-agent-token
  namespace: dynamic-scoring
  annotations:
    kubernetes.io/service-account.name: dynamic-scoring-agent-sa
type: kubernetes.io/service-account-token
EOF
```

## 5. Register the Scorer (DynamicScorer CR)

Create a `DynamicScorer` CR on the hub to register the sample scorer. This tells the framework where the scoring API is and how to query Prometheus.

Replace `<ROUTE_HOST>` with the Route URL from Step 3:

```bash
ROUTE_URL=$(oc get route sample-scorer -n dynamic-scoring -o jsonpath='http://{.spec.host}')
```

```yaml
apiVersion: dynamic-scoring.open-cluster-management.io/v1alpha1
kind: DynamicScorer
metadata:
  name: sample-scorer
  namespace: open-cluster-management
spec:
  description: A sample scorer for time series data
  configURL: <ROUTE_URL>/config
  configSyncMode: None
  location: Internal
  scoreDestination: AddOnPlacementScore
  scoreDimensionFormat: "${scoreName}"
  source:
    type: Prometheus
    host: https://prometheus-k8s.openshift-monitoring.svc.cluster.local:9091
    path: /api/v1/query_range
    params:
      query: 'avg by (node) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100'
      range: 3600
      step: 60
    auth:
      tokenSecretRef:
        name: dynamic-scoring-agent-token
        key: token
  scoring:
    host: <ROUTE_URL>
    path: /scoring
    params:
      name: sample_my_score
      interval: 30
```

### Why this query and dimension format

The default sample scorer config uses `sum by (node, namespace, pod) (rate(container_cpu_usage_seconds_total{...}[1m]))` with `scoreDimensionFormat: "${node};${namespace};${pod}"`. This has two problems on ACM/OCP:

1. **Scores are always zero.** The query returns per-second CPU usage rates — tiny floats like 0.001–0.08 on idle clusters. The agent truncates to `int32`, so everything below 1.0 becomes 0.

2. **One entry per pod.** A cluster with hundreds of pods produces hundreds of AddOnPlacementScore entries, which is noisy and not useful for cluster-level placement decisions.

The recommended query `avg by (node) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100` returns **CPU idle percentage** (0–100), giving meaningful integer scores — e.g., 86 means 86% idle capacity.

For the dimension format, use `${scoreName}` rather than `${node}`. The `${node}` placeholder resolves from the scorer's response metric labels, but on OCP the `node_cpu_seconds_total` metric may not carry a `node` label (it uses `instance` instead). Since `instance` is not a supported placeholder in the agent, `${node}` resolves to an empty string and scores are silently dropped. Using `${scoreName}` always resolves to the score name (`sample_my_score`) and produces one score per cluster — which is the right granularity for placement.

Key fields:
- **`configURL`** — the Route URL of the scorer's `/config` endpoint
- **`configSyncMode: None`** — all config is defined in this CR (use `Full` if you want the controller to fetch config from the scorer's `/config` endpoint instead)
- **`source.host`** — OpenShift's built-in Prometheus endpoint (port 9091, HTTPS)
- **`source.auth.tokenSecretRef`** — references the token Secret created in Step 4
- **`scoring.host`** — the Route URL of the scorer (agents on all managed clusters POST here)
- **`scoring.params.name`** — the score name used in `AddOnPlacementScore` and Prometheus metrics

## 6. Create DynamicScoringConfig

The `DynamicScoringConfig` singleton triggers the framework to distribute scorer configurations to managed clusters via ConfigMaps.

```bash
oc apply -n open-cluster-management -f - <<EOF
apiVersion: dynamic-scoring.open-cluster-management.io/v1alpha1
kind: DynamicScoringConfig
metadata:
  name: dynamic-scoring-config
spec:
  masks:
    - clusterName: "local-cluster"
      scoreName: "sample_my_score"
EOF
```

Important:
- The name **must** be `dynamic-scoring-config`.
- Apply it in the same namespace as the framework (`open-cluster-management`).
- The `masks` entry excludes the hub cluster (`local-cluster`) from receiving this scorer — adjust as needed.

### Verify distribution

Check that the ConfigMap arrived on the managed cluster:

```bash
oc get configmap dynamic-scoring-config -n dynamic-scoring --context <managed-cluster-context> \
  -o jsonpath='{.data.summaries}' | jq
```

You should see a JSON array containing the scorer summary with the Prometheus endpoint, query, scoring endpoint, and interval.

## 7. Verify End-to-End

### Check the agent logs

```bash
oc logs -n dynamic-scoring deployment/dynamic-scoring-agent --context <managed-cluster-context> --tail=30
```

Look for log lines showing successful metric fetches and scoring API calls.

### Check the sample-scorer logs

```bash
oc logs -n dynamic-scoring deployment/sample-scorer --context <managed-cluster-context> --tail=20
```

You should see incoming POST requests to `/scoring` with HTTP 200 responses.

### Check AddOnPlacementScore on the hub

```bash
oc get addonplacementscore sample-my-score -n <MANAGED_CLUSTER_NAME> -o yaml
```

Expected output includes a `status.scores` array with a single entry per cluster:

```yaml
status:
  scores:
    - name: sample_my_score
      value: 86
```

The `name` field follows the `scoreDimensionFormat` from the DynamicScorer (`${scoreName}`). The `value` is the integer-truncated average CPU idle percentage — higher means more spare capacity.

## Troubleshooting

### Addon agent not deploying to managed clusters

**Symptom:** `oc get managedclusteraddons -A | grep dynamic` returns nothing.

**Cause:** ManagedClusterSetBinding mismatch. See [Step 2](#2-fix-the-managedclusterset-binding).

### Prometheus authentication failures

**Symptom:** Agent logs show 401/403 errors when querying Prometheus.

**Fix:** Ensure the token Secret exists in the `dynamic-scoring` namespace on the managed cluster and is annotated with the correct ServiceAccount name (`dynamic-scoring-agent-sa`). The ServiceAccount must have permission to query Prometheus — on OpenShift, `cluster-admin` binding (created by the addon) covers this.

## Next Steps

### Explore other scorer types

The sample scorer computes a simple average — it exists to validate the end-to-end flow. The framework is designed to be pluggable: swap in a different scorer without changing anything else. See [Scoring API Samples](./scoring-api-samples.md) for implementations that go beyond averages:

- **Simple Prediction Scorer** — trains a linear regression model on CPU time-series data and predicts usage 5 minutes ahead. Returns a high score when predicted usage is likely to exceed a threshold.
- **LLM Forecast Scorer** — sends time-series data to an OpenAI-compatible LLM for forecasting. Useful when you want natural-language reasoning about trends.
- **AI Workload Scorer** — scores GPU power headroom and token generation throughput for AI inference workloads.
- **Static Scorer** — returns pre-defined scores from configuration, no computation. Useful for encoding known hardware characteristics or business rules.

### Build a placement optimizer

At this point, `AddOnPlacementScores` exist on the hub for each managed cluster — but nothing acts on them yet. The next step is a **placement optimizer** that takes a workload definition, reads the scores across clusters, and decides where to place it. See the [DSF MCP Server](../samples/dynamic-scoring-framework-mcp/) for an example that uses PuLP linear programming to optimize placement and exposes the logic as MCP tools for AI-assisted decision-making.
