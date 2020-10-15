# Informer 篇（上）

## 前言

众所周知，在 Kubernetes 中各组件是通过 HTTP 协议进行通信的，而组件间的通信也并没有依赖任何中间件，那么如何保证消息的实时性、可靠性、顺序性呢？**Informer 机制**很好的解决了这个问题。Kubernetes 中各组件与 API Server 的通信都是通过 client-go 的 informer 机制来保证和完成的。

## 控制器模式

控制器模式最核心的就是控制循环的概念。而 Informer 机制，也就是控制循环中负责观察系统的传感器（Sensor）主要由 Reflector、Informer、Indexer 三个组件构成。其与各种资源的 Controller 相配合，就可以完成完整的控制循环，不断的使系统向终态趋近 `status` -> `spec`。

![informer 机制](https://tva2.sinaimg.cn/large/ad5fbf65ly1gjme5nhuykj20mr0fmn6j.jpg)

### Informer

所谓 informer，其实就是一个带有本地缓存和索引机制的，可以注册 EventHandler 的 client，目的是为了减轻频繁通信 API Server 的压力而抽取出来的一层 cache，客户端对 API Server 数据的**读取**和**监测**操作都通过本地的 informer 来进行。

每一个 Kubernetes 资源上都实现了 informer 机制，每一个 informer 上都会实现 `Informer()` 和 `Lister()` 方法：

```go
// client-go/informers/core/v1/pod.go
type PodInformer interface {
  Informer() cache.SharedIndexInformer
  Lister() v1.PodLister
}
```

定义不同资源的 Informer，允许监控不同资源事件。同时为了避免同一资源的 Informer 被实例化多次，而每个 Informer 都会使用一个 Reflector，这样会运行过多相同的 ListAndWatch，从而加重 API Server 的压力，Informer 还提供了共享机制，多个 Informer 可以共享一个 Reflector，从而达到节约资源的目的。

```go
// client-go/informers/factory.go
type sharedInformerFactory struct {
  client           kubernetes.Interface
  namespace        string
  tweakListOptions internalinterfaces.TweakListOptionsFunc
  lock             sync.Mutex
  defaultResync    time.Duration
  customResync     map[reflect.Type]time.Duration

  informers map[reflect.Type]cache.SharedIndexInformer
  // startedInformers is used for tracking which informers have been started.
  // This allows Start() to be called multiple times safely.
  startedInformers map[reflect.Type]bool
}
...
// InternalInformerFor returns the SharedIndexInformer for obj using an internal client.
func (f *sharedInformerFactory) InformerFor(obj runtime.Object, newFunc internalinterfaces.NewInformerFunc) cache.SharedIndexInformer {
  f.lock.Lock()
  defer f.lock.Unlock()

  informerType := reflect.TypeOf(obj)
  informer, exists := f.informers[informerType]
  if exists {
    return informer
  }

  resyncPeriod, exists := f.customResync[informerType]
  if !exists {
    resyncPeriod = f.defaultResync
  }

  informer = newFunc(f.client, resyncPeriod)
  f.informers[informerType] = informer

  return informer
}
```

使用 map 数据结构实现共享 Informer 机制，在 `InformerFor()` 函数添加了不同资源的 Informer，在添加过程中如果已经存在同类型的 Informer，则返回当前 Informer，不再继续添加。如下就是 `deployment` 的 `Informer()` 方法，其中就调用了 `InformerFor()` 函数。

```go
// client-go/informers/apps/v1beta1/deployment.go
func (f *deploymentInformer) Informer() cache.SharedIndexInformer {
  return f.factory.InformerFor(&appsv1beta1.Deployment{}, f.defaultInformer)
}
```

### Reflector

Reflector 用于监测制定 Kubernetes 资源，当资源发生变化时，触发相应的事件，如：Added（资源添加）事件、Update（资源更新）事件、Delete（资源删除）事件，并将事件及资源名称添加到 DeltaFIFO 中。

#### ListAndWatch

在实例化 Reflector 时，必须传入 ListerWatcher 接口对象，其拥有 `List()` 和 `Watch()` 方法。Reflector 通过 `Run()` 方法启动监控并处理事件。在程序第一次运行时，会执行 `List()` 方法将所有的对象数据存入 DeltaFIFO 中，每次 Controller 重启，都会执行 `List()` 方法；同时，Reflector 实例中还有 `resyncPeriod` 参数，如果该参数不为 0，则会根据该参数值周期性的执行 `List()` 操作，此时这些资源对象会被设置为 Sync 操作类型（不同于 Add、Update 等）。

而 `Watch()` 则会根据 Reflector 实例 `period` 参数，周期性的监控资源对象是否有变更。如果发生变更，则通过 `r.watchHandler` 处理变更事件。

![Reflector](https://tva4.sinaimg.cn/large/ad5fbf65ly1gjmkxmiboej20mr0uwh9f.jpg)

#### DeltaFIFO

DeltaFIFO 顾名思义，Delta 是一个资源对象存储，可以保持操作类型（Add、Update、Delete、Sync等）；而 FIFO 则是一个先进先出的队列。其是一个生产者与消费者的队列，其中 Reflector 是生产者，消费者则调用 `Pop()` 方法取出最早进入队列的对象数据。

### Indexer

Indexer 是 client-go 用来存储资源对象并自带索引功能的本地存储，Reflector 从 DeltaFIFO 中将消费出来的资源对象存储至 Indexer。同时 Indexer 中的数据与 Etcd 中的数据保持完全一致。client-go 可以很方便的从本次存储中读取相应的资源对象数据，而无需每次都从远程 Etcd 集群中读取，从而降低了 API Server 和 Etcd 集群的压力。

## 结语

要了解 Kubernetes，Informer 是绕不过的内容，其在 Kubernetes 中非常重要。本文主要图解了 Informer 机制以及 Reflector，由于篇幅有限，DeltaFIFO，Indexer 等概念只做了简单介绍，这些内容会在后续的文章中进行详解，敬请期待。

## 参考

- 《Kubernetes 源码剖析 - 郑东旭著》
