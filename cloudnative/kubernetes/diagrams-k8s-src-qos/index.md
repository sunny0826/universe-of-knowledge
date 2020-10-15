# QoS 篇

## 前言

日常使用 Kubernetes 时，时长会出现 Node 节点中的 Pod 被 OOMKill 掉的情况，但 Node 节点中 Pod 众多，为什么单单选中这个 Pod  Kill 掉呢？这里就引出了 QoS 的概念，本篇文章就会从源码的角度介绍 QoS 的分类、打分机制，并简单介绍不同 QoS 的本质区别。看看这个机制是如何保证运行在 Kubernetes 中服务质量的。

## QoS

QoS(Quality of Service) 即服务质量，是 Kubernetes 中的一种控制机制，其会对运行在 Kubernetes 中的 Pod 进行一个质量划分，根据 Pod 中 container 的 Limit 和 request 将 Pod 分为 `Guaranteed`，`Burstable`，`BestEffort` 三类并对所有 Pod 进行一个打分。在资源尤其是内存这种不可压缩资源不够时，为保证整体质量的稳定，Kubernetes 就会根据 QoS 的不同优先级，对 Pod 进行资源回收。这也是有时集群中的 Pod 突然被 kill 掉的原因。

![Qos](https://tva2.sinaimg.cn/large/ad5fbf65ly1gjotxqvm4uj20mr0aw78n.jpg)

### QoS 分类

以下代码用来获取 Pod 的 QoS 类，用于区分不同 Pod 的 QoS。

```go
// github/kubernetes/pkg/apis/core/v1/helper/qos/qos.go
func GetPodQOS(pod *v1.Pod) v1.PodQOSClass {
  requests := v1.ResourceList{}
  limits := v1.ResourceList{}
  zeroQuantity := resource.MustParse("0")
  isGuaranteed := true
  allContainers := []v1.Container{}
  allContainers = append(allContainers, pod.Spec.Containers...)
  // InitContainers 容器也会被加入 QoS 的计算中
  allContainers = append(allContainers, pod.Spec.InitContainers...)
  for _, container := range allContainers {
    // 遍历 requests 中的 CPU 和 memory 值
    for name, quantity := range container.Resources.Requests {
      if !isSupportedQoSComputeResource(name) {
        continue
      }
      if quantity.Cmp(zeroQuantity) == 1 {
        delta := quantity.DeepCopy()
        if _, exists := requests[name]; !exists {
          requests[name] = delta
        } else {
          delta.Add(requests[name])
          requests[name] = delta
        }
      }
    }
    // 遍历 limits 中的 CPU 和 memory 值
    qosLimitsFound := sets.NewString()
    for name, quantity := range container.Resources.Limits {
      if !isSupportedQoSComputeResource(name) {
        continue
      }
      if quantity.Cmp(zeroQuantity) == 1 {
        qosLimitsFound.Insert(string(name))
        delta := quantity.DeepCopy()
        if _, exists := limits[name]; !exists {
          limits[name] = delta
        } else {
          delta.Add(limits[name])
          limits[name] = delta
        }
      }
    }

    // 判断是否同时设置了 limits 和 requests，如果没有则不是 Guaranteed
    if !qosLimitsFound.HasAll(string(v1.ResourceMemory), string(v1.ResourceCPU)) {
      isGuaranteed = false
    }
  }
  // 如果 requests 和 limits 都没有设置，则为 BestEffort
  if len(requests) == 0 && len(limits) == 0 {
    return v1.PodQOSBestEffort
  }
  // 检查所有资源的 requests 和 limits 是否都相等
  if isGuaranteed {
    for name, req := range requests {
      if lim, exists := limits[name]; !exists || lim.Cmp(req) != 0 {
        isGuaranteed = false
        break
      }
    }
  }
  // 都设置了 requests 和 limits，则为 Guaranteed
  if isGuaranteed &&
    len(requests) == len(limits) {
    return v1.PodQOSGuaranteed
  }
  return v1.PodQOSBurstable
}
```

### QoS 打分

QoS 会根据不同的分类进行 OOMScore 打分，当宿主机上内存不足时，系统会优先 kill 掉 OOMScore 分数高的进程。

![QoS 打分](https://tvax3.sinaimg.cn/large/ad5fbf65ly1gjoqlfe8cgj20mr0pcqmy.jpg)

值得注意的是不久之前 `guaranteedOOMScoreAdj` 的值还是 `-998`，今年 9 月 22 日才合并 [PR](https://github.com/kubernetes/kubernetes/pull/71269) 将其修改为 `-997`，而修改的 PR 及 [相关 ISSUE](https://github.com/kubernetes/kubernetes/issues/72294) 在 2018 年就已经提出了，感兴趣的同学可以去看看。这里附上源码：

```go
// github/kubernetes/pkg/kubelet/qos/policy.go
const (
  // KubeletOOMScoreAdj is the OOM score adjustment for Kubelet
  KubeletOOMScoreAdj int = -999
  // KubeProxyOOMScoreAdj is the OOM score adjustment for kube-proxy
  KubeProxyOOMScoreAdj  int = -999
  guaranteedOOMScoreAdj int = -997
  besteffortOOMScoreAdj int = 1000
)

func GetContainerOOMScoreAdjust(pod *v1.Pod, container *v1.Container, memoryCapacity int64) int {
  // 高优先级 Pod 直接返回 guaranteedOOMScoreAdj
  if types.IsCriticalPod(pod) {
    // Critical pods should be the last to get killed.
    return guaranteedOOMScoreAdj
  }

  // 根据 QoS 等级，返回 guaranteedOOMScoreAdj 或 besteffortOOMScoreAdj 的分数，这里只处理 Guaranteed 与 BestEffort
  switch v1qos.GetPodQOS(pod) {
  case v1.PodQOSGuaranteed:
    // Guaranteed containers should be the last to get killed.
    return guaranteedOOMScoreAdj
  case v1.PodQOSBestEffort:
    return besteffortOOMScoreAdj
  }

  memoryRequest := container.Resources.Requests.Memory().Value()
  // 内存占用越少，分数越高
  oomScoreAdjust := 1000 - (1000*memoryRequest)/memoryCapacity
  // 保证 Burstable 分数高于 Guaranteed
  if int(oomScoreAdjust) < (1000 + guaranteedOOMScoreAdj) {
    return (1000 + guaranteedOOMScoreAdj)
  }
  // 保证 Burstable 分数低于 BestEffect
  if int(oomScoreAdjust) == besteffortOOMScoreAdj {
    return int(oomScoreAdjust - 1)
  }
  return int(oomScoreAdjust)
}
```

### QoS 的本质区别

三种 QoS 在调度和实现都存在着区别：

- 调度时，调度器只会根据 request 值进行调度，这也就解释了有些 Node 节点 Resource Limit 超出 100% 的情况
- 当 OOM 时，系统会根据 `oom_score` 值来选择优先 kill 掉的进程，分数越高越先被 kill 掉。`oom_score` 由系统计算所得，用户是不能设置的。但是如上文所述，而根据 QoS 的类型，kubelet 会计算出 `oom_score_adj` 的值，通过 `oom_score_adj` 来调整 `oom_score` 的分数，从而影响 OOM 被 kill 进程的优先级。
- 对于资源的限制，是由 CGroup 来完成的。kubelet 会为三种 QoS 分别创建 QoS level CGroup：
  - `Guaranteed` Pod Qos 的 CGroup level 会直接创建在 `RootCgroup/kubepods` 下
  - `Burstable` Pod Qos 的创建在 `RootCgroup/kubepods/burstable` 下
  - `BestEffort` Pod Qos 的创建在 `RootCgroup/kubepods/BestEffort`下
- 而在 Pod level CGroup 中还会创建 Container level CGroup，其结构如下图所示：

![Qos-CGroup](https://tva3.sinaimg.cn/large/ad5fbf65ly1gjoxdsmfs0j20mr0e8tjg.jpg)

## 结语

本文我们讨论了 Kubernetes 中 QoS 机制的分类、打分及其本质，除了这些 QoS 的实现 `QOSContainerManager` 中还有三种 QoS 以宿主机上 allocatable 资源量为基础为 Pod 分配资源，并通过多个 level cgroup 进行层层限制的逻辑，由于篇幅有限，就不做详细介绍了。
