# Deployment Controller 篇

## 前言

Kubernetes 最为云原生领域的绝对 leader，可以说是当下最著名开源项目之一，拥有着庞大的贡献者群体以及更庞大的用户群体。作为使用 Go 语言开发的明星项目，其源码也是非常有趣的。笔者在研究 Kubernetes 源码时，常常发现很多让人眼前一亮的设计和拍案叫绝的逻辑。但由于 Kubernetes 的代码量十分庞大，函数间的调用也十分复杂，在阅读源码时常常被绕的找不着北，正好手边有一本《图解算法》，于是就萌生了图解 Kubernetes 源码的想法。本文为本系列第一篇文章，尝试使用流程图来分析 Kubernetes Controller Manager 中 的 Deployment Controller 逻辑。

## Deployment Controller

Deployment Controller 是 Kube-Controller-Manager 中最常用的 Controller 之一管理 Deployment 资源。而 Deployment 的本质就是通过管理 ReplicaSet 和 Pod 在 Kubernetes 集群中部署 **无状态** Workload。

### Deployment、ReplicaSet 和 Pod

![deployment-controller](https://tvax4.sinaimg.cn/large/ad5fbf65gy1gj6twofn24j20es09s43a.jpg)

Deployment 通过控制 ReplicaSet，ReplicaSet 再控制 Pod，最终由 Controller 驱动达到期望状态。在控制器模式下，每次操作对象都会触发一次事件，然后 controller 会进行一次 syncLoop 操作，controller 是通过 informer 监听事件以及进行 ListWatch 操作的。

Deployment Controller 会监听 DeploymentInformer、ReplicaSetInformer、PodInformer 三种资源。这三种资源变化时，都会触发 syncLoop 也就是下面代码 `dc.Run()` 中的 `dc.syncDeployment` 操作，来进行状态更新逻辑。

```go
func startDeploymentController(ctx ControllerContext) (http.Handler, bool, error) {
    if !ctx.AvailableResources[schema.GroupVersionResource{Group: "apps", Version: "v1", Resource: "deployments"}] {
      return nil, false, nil
    }
    dc, err := deployment.NewDeploymentController(
      ctx.InformerFactory.Apps().V1().Deployments(),
      ctx.InformerFactory.Apps().V1().ReplicaSets(),
      ctx.InformerFactory.Core().V1().Pods(),
      ctx.ClientBuilder.ClientOrDie("deployment-controller"),
    )
    if err != nil {
      return nil, true, fmt.Errorf("error creating Deployment controller: %v", err)
    }
    go dc.Run(int(ctx.ComponentConfig.DeploymentController.ConcurrentDeploymentSyncs), ctx.Stop)
    return nil, true, nil
}
```

### Deployment Controller 启动流程

那么先从启动逻辑开始，Kube-Controller-Manager 中所有的 Controller 的启动逻辑都差不多，都是在 `Run()` 方法中完成初始化并启动，`NewControllerInitializers` 会初始化所有 Controller，而 `startXXXXController()` 则会启动对应的 Controller。

![deployment-controller-启动流程](https://tva3.sinaimg.cn/large/ad5fbf65gy1gj6rw439nrj20mh12o7wh.jpg)

### 核心逻辑 syncHandler

Deployment Controller 在初始化时指定了 `dc.syncHandler = dc.syncDeployment`，所以核心逻辑就是围绕 `syncDeployment()` 来展开的。

![deployment-controller-核心逻辑](https://tvax4.sinaimg.cn/large/ad5fbf65gy1gj6s4tfuynj20my1zq7wi.jpg)

从源码可以看出，删除、暂停、回滚、扩缩容、更新策略的优先级为 `delete > pause > rollback > scale > rollout`。而最终都不是直接更新或修改对应资源，而是通过 `dc.client.AppsV1().Deployments().UpdateStatus()` 更新 Deployment Status。

## 结语

以上就是 Deployment Controller 代码逻辑，通过流程图，希望能描述的更加清晰。因为是第一次尝试图解，可能有遗漏和不足，欢迎留言指正。图解 Kubernetes 源码将作为一个系列继续下去，后续会带来更多的源码图解。
