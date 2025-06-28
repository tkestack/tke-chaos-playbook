# 演练场景说明

[English](README.md) | [中文](README_zh.md)

## kube-apiserver高负载

**playbook**：`workflow/apiserver-overload-scenario.yaml`

该场景构造`kube-apiserver`高负载，主要流程包括：
- **演练校验**：对被演练的`目标集群`做健康检查，检查演练集群中的`Node`和`Pod`的健康比例，低于阈值将不允许演练，您可以通过修改`precheck-pods-health-ratio`和`precheck-nodes-health-ratio`参数调整阈值。同时会校验`目标集群`中是否存在`tke-chaos-test/tke-chaos-precheck-resource ConfigMap`，如不存在将不允许演练。
- **资源预热**：在集群中创建资源(`pods/configmaps`)，模拟现网环境资源规模。
- **故障注入**：对apiserver发起洪泛`list pod/configmaps`请求，模拟`kube-apiserver`高负载压力。
- **资源清理**：演练测试完成后清理演练过程中创建的资源。

支持演练过程中开启`etcd过载保护`和`APF流控策略`[APF限流](https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/flow-control/)，您可以通过配置`enable-etcd-overload-protect`，`enable-apf`参数来开启和关闭这两个特性。`etcd过载保护`和`APF流控策略`可以有效的防护`kube-apiserver/etcd`被大流量冲击，您可以开启和禁用`etcd过载保护`、`APF流控策略`进行演练，以此来对比防护效果，并结合具体业务场景来决策是否对您的`K8s`集群开启防护策略。

**参数说明**

| 参数名称 | 类型 | 默认值 | 说明 |
|---------|---------|------|--------|
| `kubeconfig-secret-name` | `string` | `dest-cluster-kubeconfig` | `目标集群`接入`kubeconfig`的`secret名称`，如果为空，则演练当前集群 |
| `webhook-url` | `string` | "" | 企业微信群`webhook地址`，如果为空，将不进行企微通知 |
| `enable-apf` | `bool` | "false" | 演练时，是否启用`APF限速` |
| `enable-etcd-overload-protect` | `bool` | "false" | 演练时，是否启用`etcd`过载保护 |
| `precheck-pods-health-ratio` | `float` | "0.9" | `目标集群`中`Pod`健康率(0.9表示90%)，低于该值将不允许演练 |
| `precheck-nodes-health-ratio` | `float` | "0.9" | `目标集群`中`Node`健康率(0.9表示90%)，低于该值将不允许演练 |
| `enable-resource-create` | `bool` | "false" | 演练开始前是否进行资源预热，创建资源，模拟集群规模 |
| `resource-create-object-size-bytes` | `int` | "10000" | 预热阶段创建资源大小(单位字节) |
| `resource-create-object-count` | `int` | "10" | 预热阶段创建资源数量 |
| `resource-create-qps` | `int` | "10" | 预热创建资源`QPS` |
| `from-cache` | `bool` | "true" | 压测走`kube-apiserver`缓存时，设置为`true`；请求击穿到`etcd`时，设置为`false` |
| `inject-stress-concurrency` | `int` | "1" | 发压的`Pod`数量 |
| `inject-stress-list-qps` | `int` | "100" | 每个发压`Pod`的`QPS` |
| `inject-stress-total-duration` | `string` | "30s" | 发压执行总时长(如30s，5m等) |

**TKE集群推荐压测参数**

| 集群规格 | resource-create-object-size-bytes | resource-create-object-count | resource-create-qps | inject-stress-concurrency | inject-stress-list-qps |
|---------|----------------------------------|-----------------------------|---------------------|--------------------------|-----------------------|
| L5      | 10000                           | 100                         | 10                   | 6                        | 200                   |
| L50     | 10000                           | 300                         | 10                   | 6                        | 200                   |
| L100    | 50000                           | 500                         | 20                   | 6                        | 200                   |
| L200    | 100000                          | 1000                        | 50                   | 9                        | 200                   |
| L500    | 100000                          | 1000                        | 50                   | 12                       | 200                   |
| L1000   | 100000                          | 3000                        | 50                   | 12                       | 300                   |
| L3000   | 100000                          | 6000                        | 500                  | 18                       | 500                   |
| L5000   | 100000                          | 10000                       | 500                  | 21                       | 500                   |

**etcd过载保护&增强apf限流说明**

腾讯云TKE团队在社区版本基础上开发了以下核心保护特性：

1. **etcd过载保护**： 防止`etcd`因业务组件`List`请求（未查询`kube-apiserver`缓存）过载而不可用。
2. **增强APF(API优先级和公平性)限流**： 针对`expensive list`提供基于`List`开销的精细化流控。

支持的版本：

| TKE版本           |
|-------------------|
| v1.30.0-tke.9+    |
| v1.28.3-tke.14+   |
| v1.26.1-tke.17+   |
| v1.24.4-tke.26+   |
| v1.22.5-tke.35+   |
| v1.20.6-tke.54+   |

## coredns停服

**playbook**：`workflow/coredns-disruption-scenario.yaml`

该场景通过以下方式构造`coredns`服务中断：

1. **前置检查**：验证目标集群中存在`tke-chaos-test/tke-chaos-precheck-resource ConfigMap`，确保集群可用于演练
2. **组件停机**：登录argo Web UI，点击`coredns-disruption-scenario workflow`，点击`suspend-1`节点`SUMMARY`标签下的`RESUME`按钮，将`coredns Deployment`副本数缩容到`0`
3. **业务验证**：`coredns`停服期间，您可以去验证您的业务是否受到`cordns`停服的影响
4. **组件恢复**：点击`suspend-2`节点`SUMMARY`标签下的`RESUME`按钮，将`coredns Deployment`副本数恢复

**参数说明**

| 参数名称 | 类型 | 默认值 | 说明 |
|---------|------|--------|------|
| `kubeconfig-secret-name` | `string` | `dest-cluster-kubeconfig` | `目标集群kubeconfig secret`名称，如为空，则演练当前集群 |

## kubernetes-proxy停服

**playbook**：`workflow/kubernetes-proxy-disruption-scenario.yaml`

该场景通过以下方式构造`kubernetes-proxy`服务中断：

1. **前置检查**：验证目标集群中存在`tke-chaos-test/tke-chaos-precheck-resource ConfigMap`，确保集群可用于演练
2. **组件停机**：登录argo Web UI，点击`kubernetes-proxy-disruption-scenario workflow`，点击`suspend-1`节点`SUMMARY`标签下的`RESUME`按钮，将`kubernetes-proxy Deployment`副本数缩容到`0`
3. **业务验证**：`kubernetes-proxy`停服期间，您可以去验证您的业务是否受到`kubernetes-proxy`停服的影响
4. **组件恢复**：点击`suspend-2`节点`SUMMARY`标签下的`RESUME`按钮，将`kubernetes-proxy Deployment`副本数恢复

**参数说明**

| 参数名称 | 类型 | 默认值 | 说明 |
|---------|------|--------|------|
| `kubeconfig-secret-name` | `string` | `dest-cluster-kubeconfig` | `目标集群kubeconfig secret`名称，如为空，则演练当前集群 |

## 命名空间删除防护

**playbook**：`workflow/namespace-delete-scenario.yaml`

该场景测试腾讯云TKE集群的命名空间删除防护策略功能，主要流程包括：
- **创建保护策略**：创建命名空间删除约束策略，防止删除包含 Pod 的命名空间
- **创建测试资源**：创建测试命名空间 `tke-chaos-ns-76498` 和 Pod
- **验证保护机制**：尝试删除包含 Pod 的命名空间，验证保护策略是否生效
- **清理测试资源**：先删除 Pod 后再删除命名空间，最后移除保护策略

**参数说明**
| 参数名称 | 类型 | 默认值 | 说明 |
|---------|------|--------|------|
| `kubeconfig-secret-name` | `string` | `dest-cluster-kubeconfig` | 目标集群 kubeconfig secret 名称，如为空，则演练当前集群 |

腾讯云TKE支持大量的资源防护策略，如`CRD`删除保护、`PV`删除保护等，您可以访问腾讯云官方文档以查看详细信息[策略管理](https://cloud.tencent.com/document/product/457/103179)

## 托管集群master组件停服

1. 您的集群名称中需要包含`Chaos Experiment`或`混沌演练`字样且集群规模小于`L1000`，否则腾讯云API将会调用失败
2. 您需要修改演练`YAML`文件中`region`、`secret-id`、`secret-key`、`cluster-id`参数([参数说明](#托管集群master组件停服参数说明))

**playbook**
1. kube-apiserver停服&恢复：`workflow/managed-cluster-apiserver-shutdown-scenario.yaml`
2. kube-controller-manager停服&恢复：`workflow/managed-cluster-controller-manager-shutdown-scenario.yaml`
3. kube-scheduler停服&恢复：`workflow/managed-cluster-scheduler-shutdown-scenario.yaml`

该场景通过腾讯云API对托管集群的`master`组件进行停服演练，主要流程包括：

1. **前置检查**：验证目标集群中存在`tke-chaos-test/tke-chaos-precheck-resource ConfigMap`，确保集群可用于演练
2. **组件停机**：登录argo Web UI，点击`suspend-1`节点`SUMMARY`标签下的`RESUME`按钮，调用腾讯云API停止`master`组件
3. **状态验证**：延迟20秒后检查`master`状态，确保组件停机成功
4. **业务验证**：`apiserver`停服期间，您可以去验证您的业务是否受到`apiserver`停服的影响
5. **组件恢复**：点击`suspend-2`节点`SUMMARY`标签下的`RESUME`按钮，调用腾讯云API恢复`master`组件
6. **最终验证**：延迟20秒后，再次检查组件状态确保恢复成功，演练结束

**原子操作库**

`workflow/managed-cluster-master-component/`目录下是`​Master`组件停服演练的原子操作库，专为命令行环境设计，提供独立、可逆的管控单元。每个`YAML`文件对应一个最小化操作动作​（如停服/恢复），无需依赖 UI 或复杂编排。

若您的 Kubernetes 环境无法访问 Argo Web UI，可通过命令行直接调用原子化工作流执行组件演练。以`apiserver`停服演练为例，具体操作如下：

1. apiserver组件停服
```bash
kubectl create -f workflow/managed-cluster-master-component/shutdown-apiserver.yaml
```

2. apiserver组件恢复
```bash
kubectl create -f workflow/managed-cluster-master-component/restore-apiserver.yaml
```
<a id="托管集群master组件停服参数说明"></a>
**参数说明**

| 参数名称 | 类型 | 默认值 | 说明 |
|---------|------|--------|------|
| `region` | `string` | "" | 腾讯云地域，如`ap-guangzhou` [地域查询](https://www.tencentcloud.com/zh/document/product/213/6091) |
| `secret-id` | `string` | "" | 腾讯云API密钥ID, 密钥可前往官网控制台 [API密钥管理](https://console.cloud.tencent.com/cam/capi) 进行获取 |
| `secret-key` | `string` | "" | 腾讯云API密钥 |
| `cluster-id` | `string` | "" | 演练集群ID |
| `kubeconfig-secret-name` | `string` | `dest-cluster-kubeconfig` | 目标集群kubeconfig secret名称 |

**注意事项**

2. 演练过程中会影响集群`master`组件服务可用性
3. 建议在非生产环境或维护窗口期执行

## 自维护集群master组件停服
TODO
