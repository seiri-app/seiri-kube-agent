# seiri-kube-agent

A Helm chart for the **Seiri HealthCheck Operator** — automated Kubernetes service health monitoring and incident detection, powered by [seiri.app](https://cloud.seiri.app).

## Overview

`seiri-kube-agent` deploys a lightweight operator into your cluster that watches `HealthCheck` custom resources and continuously monitors the health of your Deployments, Pods, StatefulSets, DaemonSets, and Jobs. Check results are reported back to the Seiri cloud platform for dashboarding, alerting, and incident management.

### What gets installed

- A **CustomResourceDefinition** (`healthchecks.monitoring.seiri.app`) in API group `monitoring.seiri.app/v1alpha1`
- A **Deployment** running the `ghcr.io/seiri-app/kubeagent` container
- A **ServiceAccount**, **ClusterRole**, and **ClusterRoleBinding** with least-privilege RBAC
- A **Secret** holding your Seiri authentication token

## Prerequisites

- Kubernetes 1.21+
- Helm 3.x
- A [seiri.app](https://cloud.seiri.app) account with a registered cluster (to obtain `authToken` and `clusterId`)

## Installation

### Add the chart (from local source)

```bash
helm install seiri-kube-agent ./helm-main \
  --namespace seiri-system --create-namespace \
  --set seiri.authToken="<YOUR_AUTH_TOKEN>" \
  --set seiri.clusterId="<YOUR_CLUSTER_ID>"
```

### Verify the deployment

```bash
kubectl get pods -n seiri-system
kubectl get crd healthchecks.monitoring.seiri.app
```

## Configuration

All configurable values live in `values.yaml`. Override them with `--set` flags or a custom values file (`-f my-values.yaml`).

### Required parameters

| Parameter | Description |
|---|---|
| `seiri.authToken` | Authentication token from your Seiri dashboard |
| `seiri.clusterId` | Cluster ID registered in seiri.app |

### Full values reference

| Parameter | Default | Description |
|---|---|---|
| `replicaCount` | `1` | Number of operator replicas |
| `image.repository` | `ghcr.io/seiri-app/kubeagent` | Container image repository |
| `image.tag` | `"1.0"` | Image tag |
| `image.pullPolicy` | `Always` | Image pull policy |
| `seiri.backendUrl` | `https://cloud.seiri.app` | Seiri backend API endpoint |
| `seiri.authToken` | `""` | **Required.** Auth token from seiri.app |
| `seiri.clusterId` | `""` | **Required.** Cluster ID from seiri.app |
| `serviceAccount.create` | `true` | Create a dedicated ServiceAccount |
| `serviceAccount.name` | `seiri-kube-agent` | ServiceAccount name |
| `securityContext.runAsNonRoot` | `true` | Run as non-root user |
| `securityContext.runAsUser` | `65532` | UID for the container process |
| `securityContext.readOnlyRootFilesystem` | `true` | Read-only root filesystem |
| `securityContext.allowPrivilegeEscalation` | `false` | Deny privilege escalation |
| `resources.requests.cpu` | `100m` | CPU request |
| `resources.requests.memory` | `128Mi` | Memory request |
| `resources.limits.cpu` | `200m` | CPU limit |
| `resources.limits.memory` | `256Mi` | Memory limit |
| `probes.liveness.path` | `/healthz` | Liveness probe endpoint |
| `probes.readiness.path` | `/readyz` | Readiness probe endpoint |
| `nodeSelector` | `{}` | Node selector constraints |
| `tolerations` | `[]` | Pod tolerations |
| `affinity` | `{}` | Pod affinity rules |

## Usage

Once the operator is running, create `HealthCheck` resources to start monitoring workloads.

### Example: Monitor a Deployment

```yaml
apiVersion: monitoring.seiri.app/v1alpha1
kind: HealthCheck
metadata:
  name: check-test
  namespace: default
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  interval: 10s
  checkID: "chk-test-001"
  tags:
    env: development
    region: local
```

```bash
kubectl apply -f check-test.yaml
```

### Inspecting check status

```bash
kubectl get healthchecks -A
```

The CRD exposes additional printer columns for quick visibility:

| Column | Description |
|---|---|
| `Phase` | Current health phase — `Healthy`, `Unhealthy`, `Missing`, `Error`, `Unknown`, or `Suspended` |
| `Last Check` | Timestamp of the most recent check execution |
| `Last Success` | Timestamp of the last successful check |
| `Last Failed` | Timestamp of the last failed check |
| `Age` | Time since the resource was created |

### HealthCheck spec fields

| Field | Required | Description |
|---|---|---|
| `spec.targetRef.apiVersion` | Yes | API version of the target (e.g. `apps/v1`) |
| `spec.targetRef.kind` | Yes | Resource kind — `Deployment`, `Pod`, `Job`, `StatefulSet`, `DaemonSet` |
| `spec.targetRef.name` | Yes | Name of the target resource |
| `spec.targetRef.namespace` | No | Namespace of the target (defaults to the HealthCheck's namespace) |
| `spec.checkID` | Yes | Identifier displayed in the seiri.app dashboard |
| `spec.interval` | No | Check interval (e.g. `10s`, `1m`, `5m`) |
| `spec.suspend` | No | Set to `true` to pause the check |
| `spec.tags` | No | Key-value pairs for filtering and organization in seiri.app |

## RBAC permissions

The operator requests only the minimum permissions needed:

| API Group | Resources | Verbs |
|---|---|---|
| `monitoring.seiri.app` | `healthchecks` | get, list, watch |
| `monitoring.seiri.app` | `healthchecks/status` | get, update, patch |
| `apps` | `deployments`, `statefulsets`, `daemonsets` | get, list, watch |
| `""` (core) | `pods` | get, list, watch |
| `batch` | `jobs` | get, list, watch |

## Security

The default security posture follows container hardening best practices: the container runs as non-root (UID 65532), uses a read-only root filesystem, drops all Linux capabilities, disables privilege escalation, and applies the `RuntimeDefault` seccomp profile. The auth token is stored in a Kubernetes Secret and injected as an environment variable — never mounted as a file.

## Uninstalling

```bash
helm uninstall seiri-kube-agent -n seiri-system
```

> **Note:** The CRD and any existing `HealthCheck` resources are **not** removed automatically. To fully clean up:
>
> ```bash
> kubectl delete healthchecks --all -A
> kubectl delete crd healthchecks.monitoring.seiri.app
> ```

## Chart info

| | |
|---|---|
| **Chart version** | 0.1.0 |
| **App version** | 1.0.0 |
| **Type** | application |
| **Source** | [seiri.app](https://cloud.seiri.app) |
