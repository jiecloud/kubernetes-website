---
title: 高级 Pod 配置
api_metadata:
- apiVersion: "v1"
  kind: "Pod"
content_type: concept
weight: 180
---

<!--
title: Advanced Pod Configuration
api_metadata:
- apiVersion: "v1"
  kind: "Pod"
content_type: concept
weight: 180
-->

<!-- overview -->

本文介绍了高级 Pod 配置主题，包括 [PriorityClasses](#priorityclasses)、[RuntimeClasses](#runtimeclasses)、
Pod 内的 [安全上下文](#security-context)，以及介绍 [调度](/docs/concepts/scheduling-eviction/#scheduling) 的相关内容。

<!-- body -->

## PriorityClasses {#priorityclasses}

**PriorityClasses（优先级类）** 允许你设置 Pod 相对于其他 Pod 的重要性。
如果你为 Pod 分配了优先级类，Kubernetes 会根据你指定的 PriorityClass 为该 Pod 设置 `.spec.priority` 字段（你无法直接设置 `.spec.priority`）。
当或何时一个 Pod 无法被调度，且问题是因资源不足导致的，{{< glossary_tooltip term_id="kube-scheduler" text="kube-scheduler" >}}
会尝试 {{< glossary_tooltip text="preempt" term_id="preemption" >}} 较低优先级
的 Pod，以便让更高优先级的 Pod 能够被调度。

PriorityClass 是一个集群范围的 API 对象，它将优先级类名称映射到整数优先级值。数字越大表示优先级越高。

### 定义 PriorityClass

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 10000
globalDefault: false
description: "高优先级工作负载的优先级类"
```

### 使用 PriorityClass 指定 Pod 优先级

{{< highlight yaml "hl_lines=9" >}}
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  priorityClassName: high-priority
{{< /highlight >}}

### 内置 PriorityClasses

Kubernetes 提供了两个内置的 PriorityClasses：
- `system-cluster-critical`: 用于对集群至关重要的系统组件
- `system-node-critical`: 用于对单个节点至关重要的系统组件。这是 Kubernetes 中 Pod 可以拥有的最高优先级。

更多信息请参见 [Pod 优先级和抢占](/docs/concepts/scheduling-eviction/pod-priority-preemption/)。

## RuntimeClasses {#runtimeclasses}

**RuntimeClass（运行时类）** 允许你为 Pod 指定底层容器运行时。当你想为不同类型的 Pod 指定不同的容器运行时时非常有用，例如当你需要不同的隔离级别或运行时功能时。

### 示例 Pod {#runtimeclass-pod-example}

{{< highlight yaml "hl_lines=6" >}}
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  runtimeClassName: myclass
  containers:
  - name: mycontainer
    image: nginx
{{< /highlight >}}

[RuntimeClass](/docs/concepts/containers/runtime-class/) 是一个集群范围的对象，它表示在你的一些或所有节点上可用的容器运行时。

集群管理员安装和配置 backing RuntimeClass 的具体运行时。

他们可能会在所有节点上设置特殊的容器运行时配置，或者可能只是在部分节点上设置。

更多信息请参见 [RuntimeClass](/docs/concepts/containers/runtime-class/) 文档。

## Pod 和容器级别的安全上下文配置 {#security-context}

Pod 规范中的 `Security context` 字段提供了对 Pod 和容器的安全设置的细粒度控制。

### Pod 级别的 `securityContext` {#pod-level-security-context}

某些安全方面适用于整个 Pod；对于其他方面，你可能希望设置默认值，而无需任何容器级别的覆盖。

以下是在 Pod 级别使用 `securityContext` 的示例：

#### 示例 Pod {#pod-level-security-context-example}

{{< highlight yaml "hl_lines=5-9" >}}
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:  # This applies to the entire Pod
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: sec-ctx-demo
    image: registry.k8s.io/e2e-test-images/agnhost:2.45
    command: ["sh", "-c", "sleep 1h"]
{{< /highlight >}}

### 容器级别的安全上下文 {#container-level-security-context}

你可以只为特定的容器指定安全上下文。这是一个示例：

#### 示例 Pod {#container-level-security-context-example}

{{< highlight yaml "hl_lines=9-17" >}}
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-2
spec:
  containers:
  - name: sec-ctx-demo-2
    image: gcr.io/google-samples/node-hello:1.0
    securityContext:
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
      seccompProfile:
        type: RuntimeDefault
{{< /highlight >}}

### 安全上下文选项

- **用户和组 ID**: 控制容器以哪个用户/组运行
- **Capabilities**: 添加或丢弃 Linux capabilities
- **Seccomp Profiles**: 设置安全计算配置文件
- **SELinux Options**: 配置 SELinux 上下文
- **AppArmor**: 配置 AppArmor 配置文件以获得额外的访问控制
- **Windows Options**: 配置 Windows 特定的安全设置

{{< caution >}}
你也可以使用 Pod `securityContext` 允许 Linux 容器中的
[_privileged mode_](/docs/concepts/security/linux-kernel-security-constraints/#privileged-containers)。
特权模式会覆盖 `securityContext` 中的许多其他安全设置。
除非无法通过 `securityContext` 中的其他字段授予等效权限，否则避免使用此设置。
你可以通过在 Pod 级别的安全上下文中设置 `windowsOptions.hostProcess` 标志来类似地
在 Windows 容器中以特权模式运行。详细信息和说明请参见
[创建 Windows HostProcess Pod](/docs/tasks/configure-pod-container/create-hostprocess-pod/)。
{{< /caution >}}

更多信息请参见 [为 Pod 或容器配置安全上下文](/docs/tasks/configure-pod-container/security-context/)。

## 影响 Pod 调度决策 {#scheduling}

Kubernetes 提供了多种机制来控制你的 Pods 被调度到哪些节点上。

### Node selectors {#node-selectors}

最简单的节点选择约束：

{{< highlight yaml "hl_lines=9-11" >}}
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    disktype: ssd
{{< /highlight >}}

### Node affinity {#node-affinity}

Node affinity 允许你指定规则来约束你的 Pod 可以被调度的节点。以下是一个 Pod 的示例，该 Pod 偏好运行在标记为位于特定大陆的节点上，基于 [`topology.kubernetes.io/zone`](/docs/reference/labels-annotations-taints/#topologykubernetesiozone) 标签的值进行选择。

{{< highlight yaml "hl_lines=6-15" >}}
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
            - antarctica-west1
  containers:
  - name: with-node-affinity
    image: registry.k8s.io/pause:3.8
{{< /highlight >}}

### Pod affinity and anti-affinity {#pod-affinity-and-anti-affinity}

除了节点亲和性之外，你还可以根据已经运行在节点上的 _其他 Pods_ 的标签来约束 Pod 可以被调度到的节点。Pod affinity 允许你指定关于 Pod 相对于其他 Pod 应该放置在哪里的规则。

{{< highlight yaml "hl_lines=6-15" >}}
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - database
        topologyKey: topology.kubernetes.io/zone
  containers:
  - name: with-pod-affinity
    image: registry.k8s.io/pause:3.8
{{< /highlight >}}

### Tolerations {#tolerations}

_Tolerations（容忍度）_ 允许 Pod 被调度到有匹配污点的节点上：

{{< highlight yaml "hl_lines=9-13" >}}
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: myapp
    image: nginx
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
{{< /highlight >}}

更多信息请参见 [将 Pod 分配给节点](/docs/concepts/scheduling-eviction/assign-pod-node/)。

## Pod overhead {#pod-overhead}

Pod overhead 允许你在容器请求和限制之上，计入由 Pod 基础架构消耗的资源。

{{< highlight yaml "hl_lines=7-10" >}}
---
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kvisor-runtime
handler: kvisor-runtime
overhead:
  podFixed:
    memory: "2Gi"
    cpu: "500m"
---
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  runtimeClassName: kvisor-runtime
  containers:
  - name: myapp
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
{{< /highlight >}}


## {{% heading "whatsnext" %}}

* 阅读关于 [Pod 优先级和抢占](/docs/concepts/scheduling-eviction/pod-priority-preemption/)
* 阅读关于 [RuntimeClasses](/docs/concepts/containers/runtime-class/)
* 探索 [为 Pod 或容器配置安全上下文](/docs/tasks/configure-pod-container/security-context/)
* 了解 Kubernetes 如何 [将 Pod 分配给节点](/docs/concepts/scheduling-eviction/assign-pod-node/)
* [Pod Overhead](/docs/concepts/scheduling-eviction/pod-overhead/)
