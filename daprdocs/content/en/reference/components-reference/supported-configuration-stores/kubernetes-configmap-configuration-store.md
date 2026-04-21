---
type: docs
title: "Kubernetes ConfigMap"
linkTitle: "Kubernetes ConfigMap"
description: "Detailed information on the Kubernetes ConfigMap configuration store component"
aliases:
  - "/operations/components/setup-configuration-store/supported-configuration-stores/setup-kubernetes-configmap/"
---

## Component format

To setup a Kubernetes ConfigMap configuration store, create a component of type `configuration.kubernetes`. See [this guide]({{% ref "howto-manage-configuration.md#configure-a-dapr-configuration-store" %}}) on how to create and apply a configuration store configuration.

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: <NAME>
spec:
  type: configuration.kubernetes
  version: v1
  metadata:
  - name: configMapName
    value: <CONFIGMAP_NAME>
```

## Spec metadata fields

| Field              | Required | Details | Example |
|--------------------|:--------:|---------|---------|
| configMapName | Y | The name of the Kubernetes ConfigMap to use as the configuration source. Must be a valid Kubernetes resource name (DNS-1123 subdomain). | `"my-app-config"` |
| kubeconfigPath | N | Path to a kubeconfig file, passed directly to the Kubernetes client library. When running in-cluster, this is not needed. When running outside the cluster and not set, the client falls back to the `KUBECONFIG` env var, then to `~/.kube/config`. | `"/path/to/kubeconfig"` |
| resyncPeriod | N | How often the informer fully re-syncs the ConfigMap state from the API server as a consistency safety net, independent of watch events. Set to `"0"` (default) to disable periodic resync and rely solely on watch events. | `"10m"` |

## Behavior

### Namespace

The namespace is derived from the `NAMESPACE` environment variable, which is automatically set by the Dapr sidecar injector via the Kubernetes downward API. If the variable is not set, the component defaults to the `default` namespace.

The component only accesses ConfigMaps in its own namespace. Cross-namespace access is not supported.

### Watch mechanism

The component uses a Kubernetes [SharedIndexInformer](https://pkg.go.dev/k8s.io/client-go/tools/cache#SharedIndexInformer) with a field selector scoped to the specific ConfigMap. This means:

- Only a single watch connection to the API server is established, regardless of how many subscriptions exist.
- Changes are delivered in real time via the Kubernetes watch protocol.
- The `Get` operation reads from the local informer cache, not the API server.

### Initial state

When a new subscription is created, the current state of the ConfigMap is delivered immediately to the subscriber. This happens atomically within the informer, so no events can be missed between subscribing and receiving the initial state.

### Binary data

ConfigMap [`binaryData`](https://kubernetes.io/docs/concepts/configuration/configmap/#configmap-object) fields are returned as base64-encoded strings with an `encoding: base64` metadata tag on the item.

### Deletion

- When individual keys are removed from the ConfigMap, the subscriber receives items with an empty value and a `deleted: true` metadata tag.
- When the entire ConfigMap is deleted, all keys are reported as deleted.

## Setup Kubernetes ConfigMap

The Kubernetes ConfigMap configuration store runs in-cluster alongside the Dapr sidecar. No external infrastructure is required.

### 1. Create a ConfigMap

Create a ConfigMap in the same namespace where your Dapr application is running:

```bash
kubectl create configmap my-app-config \
  --from-literal=log.level=info \
  --from-literal=feature.enable-v2=true \
  -n <NAMESPACE>
```

### 2. Configure RBAC

The Dapr sidecar's service account needs permissions to `get`, `list`, and `watch` ConfigMaps. Create a Role and RoleBinding:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dapr-configmap-reader
  namespace: <NAMESPACE>
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dapr-configmap-reader-binding
  namespace: <NAMESPACE>
subjects:
- kind: ServiceAccount
  name: <SERVICE_ACCOUNT_NAME>
  namespace: <NAMESPACE>
roleRef:
  kind: Role
  name: dapr-configmap-reader
  apiGroup: rbac.authorization.k8s.io
```

### 3. Apply the component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: configstore
spec:
  type: configuration.kubernetes
  version: v1
  metadata:
  - name: configMapName
    value: "my-app-config"
```

## Related links
- [Basic schema for a Dapr component]({{% ref component-schema %}})
- Read [How-To: Manage configuration from a store]({{% ref "howto-manage-configuration" %}}) for instructions on how to use a configuration store.
- [Configuration building block]({{% ref configuration-api-overview %}})
