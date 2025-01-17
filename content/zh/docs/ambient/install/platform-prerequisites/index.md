---
title: 平台特定的前提条件
description: 安装 Ambient 模式的 Istio 时平台特定的前提条件。
weight: 2
aliases:
  - /zh/docs/ops/ambient/install/platform-prerequisites
  - /zh/latest/docs/ops/ambient/install/platform-prerequisites
owner: istio/wg-environments-maintainers
test: no
---

本文档涵盖了安装 Ambient 模式的 Istio 时各类平台或环境特定的前提条件。

## 平台 {#platform}

某些 Kubernetes 环境需要您设置各种配置选项才能支持 Istio。

### Google Kubernetes Engine（GKE） {#google-kubernetes-engine-gke}

在 GKE 上，具有 [system-node-critical](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/)
`priorityClassName` 的 Istio 组件只能安装在定义了
[ResourceQuota](https://kubernetes.io/zh-cn/docs/concepts/policy/resource-quotas/)
的命名空间中。默认情况下，在 GKE 中，只有 `kube-system` 为 `node-critical` 类定义了 ResourceQuota。
Istio CNI 节点代理和 `ztunnel` 都需要 `node-critical` 类，因此在 GKE 中，两个组件都必须满足以下任一条件：

- 安装到 `kube-system`（**不是** `istio-system`）
- 安装到另一个已手动创建 ResourceQuota 的命名空间（如 `istio-system`），例如：

{{< text syntax=yaml >}}
apiVersion: v1
kind: ResourceQuota
metadata:
  name: gcp-critical-pods
  namespace: istio-system
spec:
  hard:
    pods: 1000
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values:
      - system-node-critical
{{< /text >}}

### k3d

当使用 [k3d](https://k3d.io/) 与默认的 Flannel CNI 时，
您必须将正确的 `platform` 值附加到安装命令中，
因为 k3d 使用非标准位置进行 CNI 配置和二进制文件，这需要一些 Helm 覆盖。

1. 创建一个禁用 Traefik 的集群，以免与 Istio 的入口网关冲突：

    {{< text bash >}}
    $ k3d cluster create --api-port 6550 -p '9080:80@loadbalancer' -p '9443:443@loadbalancer' --agents 2 --k3s-arg '--disable=traefik@server:*'
    {{< /text >}}

1. 安装 Istio Chart 时设置 `global.platform=k3d`。例如：

    {{< tabset category-name="install-method" >}}

    {{< tab name="Helm" category-value="helm" >}}

        {{< text syntax=bash >}}
        $ helm install istio-cni istio/cni -n istio-system --set profile=ambient --set global.platform=k3d --wait
        {{< /text >}}

    {{< /tab >}}

    {{< tab name="istioctl" category-value="istioctl" >}}

        {{< text syntax=bash >}}
        $ istioctl install --set profile=ambient --set values.global.platform=k3d
        {{< /text >}}

    {{< /tab >}}

    {{< /tabset >}}

### K3s

使用 [K3s](https://k3s.io/) 及其绑定的 CNI 之一时，
您必须将正确的 `platform` 值附加到安装命令中，
因为 K3s 对 CNI 配置和二进制文件使用非标准位置，这需要一些 Helm 覆盖。
对于默认的 K3s 路径，Istio 根据 `global.platform` 值提供内置覆盖。

{{< tabset category-name="install-method" >}}

{{< tab name="Helm" category-value="helm" >}}

    {{< text syntax=bash >}}
    $ helm install istio-cni istio/cni -n istio-system --set profile=ambient --set global.platform=k3s --wait
    {{< /text >}}

{{< /tab >}}

{{< tab name="istioctl" category-value="istioctl" >}}

    {{< text syntax=bash >}}
    $ istioctl install --set profile=ambient --set values.global.platform=k3s
    {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

但是，根据 K3s 文档，这些位置可能会在 K3s 中被覆盖。
如果您将 K3s 与自定义、非捆绑的 CNI 一起使用，则必须手动为这些 CNI 指定正确的路径，
比如 `/etc/cni/net.d` - [有关详细信息，请参阅 K3s 文档](https://docs.k3s.io/zh/networking/basic-network-options#custom-cni)。例如：

{{< tabset category-name="install-method" >}}

{{< tab name="Helm" category-value="helm" >}}

    {{< text syntax=bash >}}
    $ helm install istio-cni istio/cni -n istio-system --set profile=ambient --wait --set cniConfDir=/var/lib/rancher/k3s/agent/etc/cni/net.d --set cniBinDir=/var/lib/rancher/k3s/data/current/bin/
    {{< /text >}}

{{< /tab >}}

{{< tab name="istioctl" category-value="istioctl" >}}

    {{< text syntax=bash >}}
    $ istioctl install --set profile=ambient --set values.cni.cniConfDir=/var/lib/rancher/k3s/agent/etc/cni/net.d --set values.cni.cniBinDir=/var/lib/rancher/k3s/data/current/bin/
    {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

### MicroK8s

如果您要在 [MicroK8s](https://microk8s.io/) 上安装 Istio，
则必须在安装命令中附加正确的 `platform` 值，
因为 MicroK8s [使用非标准位置来存放 CNI 配置和二进制文件](https://microk8s.io/docs/change-cidr)。例如：

{{< tabset category-name="install-method" >}}

{{< tab name="Helm" category-value="helm" >}}

    {{< text syntax=bash >}}
    $ helm install istio-cni istio/cni -n istio-system --set profile=ambient --set global.platform=microk8s --wait

    {{< /text >}}

{{< /tab >}}

{{< tab name="istioctl" category-value="istioctl" >}}

    {{< text syntax=bash >}}
    $ istioctl install --set profile=ambient --set values.global.platform=microk8s
    {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

### minikube

如果您正在使用 [minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)
和 [Docker 驱动程序](https://minikube.sigs.k8s.io/docs/drivers/docker/)，
您必须将正确的 `platform` 值附加到安装命令中，
因为带有 Docker 的 minikube 使用非标准的容器绑定挂载路径。例如：

{{< tabset category-name="install-method" >}}

{{< tab name="Helm" category-value="helm" >}}

    {{< text syntax=bash >}}
    $ helm install istio-cni istio/cni -n istio-system --set profile=ambient --set global.platform=minikube --wait"
    {{< /text >}}

{{< /tab >}}

{{< tab name="istioctl" category-value="istioctl" >}}

    {{< text syntax=bash >}}
    $ istioctl install --set profile=ambient --set values.global.platform=minikube"
    {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

### Red Hat OpenShift {#red-hat-openshift}

OpenShift 要求在 `kube-system` 命名空间中安装 `ztunnel` 和 `istio-cni` 组件，
并且要求为所有 Chart 设置 `global.platform=openshift`。

如果您使用 `helm`，您可以直接设置目标命名空间和 `global.platform` 值。

如果您使用 `istioctl`，则必须使用名为 `openshift-ambient` 的特殊配置文件来完成相同的操作。

{{< tabset category-name="install-method" >}}

{{< tab name="Helm" category-value="helm" >}}

    {{< text syntax=bash >}}
    $ helm install istio-cni istio/cni -n kube-system --set profile=ambient --set global.platform=openshift --wait
    {{< /text >}}

{{< /tab >}}

{{< tab name="istioctl" category-value="istioctl" >}}

    {{< text syntax=bash >}}
    $ istioctl install --set profile=openshift-ambient --skip-confirmation
    {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

## CNI 插件 {#cni-plugins}

当使用某些 {{< gloss "CNI" >}}CNI 插件{{< /gloss >}}时，以下配置适用于所有平台：

### Cilium

1. Cilium 目前默认会主动删除其他 CNI 插件及其配置，
   并且必须配置 `cni.exclusive = false` 才能正确支持链式。
   更多细节请参阅 [Cilium 文档](https://docs.cilium.io/en/stable/helm-reference/)。
1. Cilium 的 BPF 伪装目前默认处于禁用状态，
   并且在 Istio 使用本地链路 IP 进行 Kubernetes 健康检查时存在问题。
   目前不支持通过 `bpf.masquerade=true` 启用 BPF 伪装，
   这会导致 Istio Ambient 中的 Pod 健康检查无法正常工作。
   Cilium 的默认 iptables 伪装实现应该可以继续正常运行。
1. 由于 Cilium 管理节点身份的方式以及在内部将节点级健康探针列入 Pod
   的白名单的方式，在 Ambient 模式下在 Istio 底层的 Cilium CNI
   安装中应用任何默认 DENY `NetworkPolicy` 都将导致 `kubelet`
   健康探测（默认情况下 Cilium 会默默地免除所有策略实施）被阻止。
   这是因为 Istio 使用 Cilium 无法识别的链路本地 SNAT 地址，并且 Cilium 没有免除链路本地地址执行策略的选项。

    这可以通过应用以下 `CiliumClusterWideNetworkPolicy` 来解决：

    {{< text syntax=yaml >}}
    apiVersion: "cilium.io/v2"
    kind: CiliumClusterwideNetworkPolicy
    metadata:
      name: "allow-ambient-hostprobes"
    spec:
      description: "Allows SNAT-ed kubelet health check probes into ambient pods"
      enableDefaultDeny:
        egress: false
        ingress: false
      endpointSelector: {}
      ingress:
      - fromCIDR:
        - "169.254.7.127/32"
    {{< /text >}}

    除非您的集群中已经应用了其他默认拒绝的 `NetworkPolicies` 或 `CiliumNetworkPolicies`，否则不需要覆盖此策略。

    更多细节请参阅 [Issue #49277](https://github.com/istio/istio/issues/49277)
    和 [CiliumClusterWideNetworkPolicy](https://docs.cilium.io/en/stable/network/kubernetes/policy/#ciliumclusterwidenetworkpolicy)。
