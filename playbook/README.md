# Chaos Engineering Scenarios

[English](README.md) | [中文](README_zh.md)

## kube-apiserver High Load

**playbook**: `workflow/apiserver-overload-scenario.yaml`

This scenario simulates high load on `kube-apiserver` with the following workflow:
- **Pre-check**: Performs health checks on the target cluster, verifying the health ratio of Nodes and Pods. If below threshold, the test will be aborted. You can adjust thresholds via `precheck-pods-health-ratio` and `precheck-nodes-health-ratio` parameters. Also checks for existence of `tke-chaos-test/tke-chaos-precheck-resource ConfigMap`.
- **Resource Warm-up**: Creates resources (`pods/configmaps`) to simulate production environment scale.
- **Fault Injection**: Floods apiserver with `list pod/configmaps` requests to simulate high load.
- **Cleanup**: Cleans up resources created during the test.

Supports enabling `etcd Overload Protection` and `APF Flow Control` [APF Rate Limiting](https://kubernetes.io/docs/concepts/cluster-administration/flow-control/) during testing via `enable-etcd-overload-protect` and `enable-apf` parameters. These protections can effectively guard against traffic spikes to `kube-apiserver/etcd`. You can compare results with/without protections to decide whether to enable them for your cluster.

**Parameters**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `kubeconfig-secret-name` | `string` | `dest-cluster-kubeconfig` | Secret name containing target cluster kubeconfig. If empty, tests current cluster |
| `webhook-url` | `string` | "" | WeChat work group webhook URL. If empty, no notifications will be sent |
| `enable-apf` | `bool` | "false" | Whether to enable APF rate limiting |
| `enable-etcd-overload-protect` | `bool` | "false" | Whether to enable etcd overload protection |
| `precheck-pods-health-ratio` | `float` | "0.9" | Minimum healthy Pod ratio (0.9=90%) to allow test |
| `precheck-nodes-health-ratio` | `float` | "0.9" | Minimum healthy Node ratio (0.9=90%) to allow test |
| `enable-resource-create` | `bool` | "false" | Whether to create resources to simulate cluster scale |
| `resource-create-object-size-bytes` | `int` | "10000" | Resource size in bytes |
| `resource-create-object-count` | `int` | "10" | Number of resources to create |
| `resource-create-qps` | `int` | "10" | QPS for resource creation |
| `from-cache` | `bool` | "true" | Set `true` to test kube-apiserver cache, `false` to test etcd directly |
| `inject-stress-concurrency` | `int` | "1" | Number of stress test Pods |
| `inject-stress-list-qps` | `int` | "100" | QPS per stress test Pod |
| `inject-stress-total-duration` | `string` | "30s" | Total test duration (e.g. 30s, 5m) |

**Recommended Parameters for TKE Clusters**

| Cluseter Level | resource-create-object-size-bytes | resource-create-object-count | resource-create-qps | inject-stress-concurrency | inject-stress-list-qps |
|---------|----------------------------------|-----------------------------|---------------------|--------------------------|-----------------------|
| L5      | 10000                           | 100                         | 10                   | 6                        | 200                   |
| L50     | 10000                           | 300                         | 10                   | 6                        | 200                   |
| L100    | 50000                           | 500                         | 20                   | 6                        | 200                   |
| L200    | 100000                          | 1000                        | 50                   | 9                        | 200                   |
| L500    | 100000                          | 1000                        | 50                   | 12                       | 200                   |
| L1000   | 100000                          | 3000                        | 50                   | 12                       | 300                   |
| L3000   | 100000                          | 6000                        | 500                  | 18                       | 500                   |
| L5000   | 100000                          | 10000                       | 500                  | 21                       | 500                   |

**etcd Overload Protection & Enhanced APF**

Tencent Cloud TKE team has developed these core protection features:

1. **etcd Overload Protection**: Prevent etcd from becoming unavailable due to overload of user component expensive `List` requests (without querying kube-apiserver cache).
2. **Enhanced APF (API Priority and Fairness)**: Provides fine-grained flow control for expensive `List` operations.

Supported versions:

| TKE Version       |
|-------------------|
| v1.30.0-tke.9+    |
| v1.28.3-tke.14+   |
| v1.26.1-tke.17+   |
| v1.24.4-tke.26+   |
| v1.22.5-tke.35+   |
| v1.20.6-tke.54+   |

## coredns Disruption

**playbook**: `workflow/coredns-disruption-scenario.yaml`

This scenario simulates coredns service disruption by:

1. **Pre-check**: Verify the existence of the `tke-chaos-test/tke-chaos-precheck-resource ConfigMap` in the target cluster to ensure the cluster is available for testing

2. **Component Shutdown**: Log in to the Argo Web UI, click on `coredns-disruption-scenario workflow`, then click the `RESUME` button under the `SUMMARY` tab of the `suspend-1` node to scale down the coredns Deployment replicas to 0

3. **Service Validation**: During the coredns disruption, you can verify whether your services are affected by the coredns disruption

4. **Component Recovery**: Click the `RESUME` button under the `SUMMARY` tab of the `suspend-2` node to restore the coredns Deployment replicas

**Parameters**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `kubeconfig-secret-name` | `string` | `dest-cluster-kubeconfig` | Target cluster kubeconfig secret name |

## kubernetes-proxy Disruption

**playbook**: `workflow/kubernetes-proxy-disruption-scenario.yaml`

This scenario simulates kubernetes-proxy service disruption by:

1. **Pre-check**: Verify the existence of the `tke-chaos-test/tke-chaos-precheck-resource ConfigMap` in the target cluster to ensure the cluster is available for testing

2. **Component Shutdown**: Log in to the Argo Web UI, click on `kubernetes-proxy-disruption-scenario workflow`, then click the `RESUME` button under the `SUMMARY` tab of the `suspend-1` node to scale down the kubernetes-proxy Deployment replicas to 0

3. **Service Validation**: During the kubernetes-proxy disruption, you can verify whether your services are affected by the kubernetes-proxy disruption

4. **Component Recovery**: Click the `RESUME` button under the `SUMMARY` tab of the `suspend-2` node to restore the kubernetes-proxy Deployment replicas

**Parameters**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `kubeconfig-secret-name` | `string` | `dest-cluster-kubeconfig` | Target cluster kubeconfig secret name |

## Namespace Deletion Protection

**playbook**: `workflow/namespace-delete-scenario.yaml`

This scenario tests Tencent Cloud TKE's namespace deletion block policy with the following workflow:
- **Create namespace deletion block policy**: Create a namespace deletion constraint policy to prevent deletion of namespaces containing Pods
- **Create test resources**: Creates test namespace `tke-chaos-ns-76498` and Pod
- **Verify protection**: Attempts to delete namespace with Pod to verify protection works
- **Cleanup**: Deletes Pod first, then namespace, and finally removes namespace deletion block policy

**Parameters**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `kubeconfig-secret-name` | `string` | `dest-cluster-kubeconfig` | Target cluster kubeconfig secret name |

Tencent Cloud TKE supports various resource protection policies, such as CRD deletion protection, PV deletion protection, etc. You can refer to the official Tencent Cloud documentation for more details: [Policy Management](https://cloud.tencent.com/document/product/457/103179)

## Managed Cluster Master Component Disruption

1. Your cluster name must contain the words `Chaos Experiment` or `混沌演练` and the cluster size must be smaller than `L1000`, otherwise the Tencent Cloud API call will fail
2. You need to modify the `region`, `secret-id`, `secret-key`, and `cluster-id` parameters in the YAML file ([Parameter Explanation](#managed-cluster-master-component-parameters))

**playbooks**:
1. kube-apiserver disruption: `workflow/managed-cluster-apiserver-shutdown-scenario.yaml`
2. kube-controller-manager disruption: `workflow/managed-cluster-controller-manager-shutdown-scenario.yaml` 
3. kube-scheduler disruption: `workflow/managed-cluster-scheduler-shutdown-scenario.yaml`

This scenario tests the disruption of managed cluster master components via Tencent Cloud API with the following workflow:

1. **Pre-check**: Verify the existence of `tke-chaos-test/tke-chaos-precheck-resource ConfigMap` in the target cluster to ensure the cluster is ready for testing
2. **Component Shutdown**: Log in to Argo Web UI, click the `RESUME` button under the `SUMMARY` tab of the `suspend-1` node to call Tencent Cloud API for stopping the master component
3. **Status Verification**: After a 20-second delay, check the master status to confirm successful shutdown
4. **Business Verification**: During apiserver shutdown, verify business impact
5. **Component Recovery**: Click the `RESUME` button under the `SUMMARY` tab of the `suspend-2` node to call Tencent Cloud API for restoring the master component
6. **Final Verification**: After another 20-second delay, recheck the component status to confirm successful restoration, marking the end of the test

**Atomic Operations Library**

The `workflow/managed-cluster-master-component/` directory contains atomic operations for command-line environments. Each YAML file corresponds to a minimal operation (e.g. shutdown/recovery) without UI dependency.

For Kubernetes environments without Argo Web UI access, execute component tests directly via CLI. Example for apiserver:

1. apiserver shutdown:
```bash
kubectl create -f workflow/managed-cluster-master-component/shutdown-apiserver.yaml
```

2. apiserver recovery:
```bash
kubectl create -f workflow/managed-cluster-master-component/restore-apiserver.yaml
```

<a id="managed-cluster-master-component-parameters"></a>
**Parameters**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `region` | `string` | "" | Tencent Cloud region, e.g. `ap-guangzhou` [Region List](https://www.tencentcloud.com/document/product/213/6091?lang=en&pg=) |
| `secret-id` | `string` | "" | Tencent Cloud API secret ID, obtain from [API Key Management](https://console.cloud.tencent.com/cam/capi) |
| `secret-key` | `string` | "" | Tencent Cloud API secret key |
| `cluster-id` | `string` | "" | Target cluster ID |
| `kubeconfig-secret-name` | `string` | `dest-cluster-kubeconfig` | Secret name containing target cluster kubeconfig |

**Notes**
1. Will affect master component availability during test
2. Recommended to execute in non-production environments or maintenance windows

## Self-Maintenance Cluster Master Component Disruption
TODO
