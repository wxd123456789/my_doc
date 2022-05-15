# replicaset 控制器

> 引用 https://draveness.me/kubernetes-replicaset/
>
> https://cloud.tencent.com/developer/article/1554675

Kubernetes 中的 ReplicaSet 主要的作用是维持一组 [Pod](https://draveness.me/kubernetes-pod) 副本的运行，它的主要作用就是保证一定数量的 Pod 能够在集群中正常运行，它会持续监听这些 Pod 的运行状态，在 Pod 发生故障重启数量减少时重新运行新的 Pod 副本。这篇文章会介绍 ReplicaSet 的工作原理，其中包括在 Kubernetes 中是如何被创建的、如何创建并持有 Pod 并在出现问题时重启它们。

ReplicaSet中除了常见的 `apiVersion`、`kind` 和 `metadata` 属性之外，规格中总共包含三部分重要内容，也就是 Pod 副本数目 `replicas`、选择器 `selector` 和 Pod 模板 `template`，这三个部分共同定义了 ReplicaSet 的规格。

同一个 ReplicaSet 会使用选择器 `selector` 中的定义查找集群中自己持有的 `Pod` 对象，它们会根据标签的匹配获取能够获得的 Pod。被 ReplicaSet 持有的 Pod 有一个 `metadata.ownerReferences` 指针指向当前的 ReplicaSet，表示当前 Pod 的所有者，这个引用主要会被集群中的 [垃圾收集器](https://draveness.me/kubernetes-garbage-collector) 使用以清理失去所有者的 Pod 对象。



## 启动

在 startReplicaSetController 中有两个比较重要的变量：

- BurstReplicas：用来控制在一个 syncLoop 过程中 rs 最多能创建的 pod 数量，设置上限值是为了避免单个 rs 影响整个系统，默认值为 500；
- ConcurrentRSSyncs：指的是需要启动多少个 goroutine 处理 informer 队列中的对象，默认值为 5；

```go
func startReplicaSetController(ctx ControllerContext) (http.Handler, bool, error) {
   if !ctx.AvailableResources[schema.GroupVersionResource{Group: "apps", Version: "v1", Resource: "replicasets"}] {
      return nil, false, nil
   }
   go replicaset.NewReplicaSetController(
      ctx.InformerFactory.Apps().V1().ReplicaSets(),
      ctx.InformerFactory.Core().V1().Pods(),
      ctx.ClientBuilder.ClientOrDie("replicaset-controller"),
      replicaset.BurstReplicas,
   ).Run(int(ctx.ComponentConfig.ReplicaSetController.ConcurrentRSSyncs), ctx.Stop)
   return nil, true, nil
}

func NewReplicaSetController(rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int) *ReplicaSetController {
	eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartStructuredLogging(0)
	eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: kubeClient.CoreV1().Events("")})
	return NewBaseController(rsInformer, podInformer, kubeClient, burstReplicas,
		apps.SchemeGroupVersion.WithKind("ReplicaSet"),
		"replicaset_controller",
		"replicaset",
		controller.RealPodControl{
			KubeClient: kubeClient,
			Recorder:   eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "replicaset-controller"}),
		},
	)
}

// NewBaseController is the implementation of NewReplicaSetController with additional injected
// parameters so that it can also serve as the implementation of NewReplicationController.
func NewBaseController(rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int,
	gvk schema.GroupVersionKind, metricOwnerName, queueName string, podControl controller.PodControlInterface) *ReplicaSetController {
	if kubeClient != nil && kubeClient.CoreV1().RESTClient().GetRateLimiter() != nil {
		ratelimiter.RegisterMetricAndTrackRateLimiterUsage(metricOwnerName, kubeClient.CoreV1().RESTClient().GetRateLimiter())
	}

	rsc := &ReplicaSetController{
		GroupVersionKind: gvk,
		kubeClient:       kubeClient,
		podControl:       podControl,
		burstReplicas:    burstReplicas,
        // expectations 的初始化，缓存rs中pod的创建删除状态
		expectations:     controller.NewUIDTrackingControllerExpectations(controller.NewControllerExpectations()),
		queue:            workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), queueName),
	}

	rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    rsc.addRS,
		UpdateFunc: rsc.updateRS,
		DeleteFunc: rsc.deleteRS,
	})
	rsc.rsLister = rsInformer.Lister()
	rsc.rsListerSynced = rsInformer.Informer().HasSynced

	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: rsc.addPod,
		// This invokes the ReplicaSet for every pod change, eg: host assignment. Though this might seem like
		// overkill the most frequent pod update is status, and the associated ReplicaSet will only list from
		// local storage, so it should be ok.
		UpdateFunc: rsc.updatePod,
		DeleteFunc: rsc.deletePod,
	})
	rsc.podLister = podInformer.Lister()
	rsc.podListerSynced = podInformer.Informer().HasSynced

	rsc.syncHandler = rsc.syncReplicaSet

	return rsc
}

// Run begins watching and syncing.
func (rsc *ReplicaSetController) Run(workers int, stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	defer rsc.queue.ShutDown()

	controllerName := strings.ToLower(rsc.Kind)
	klog.Infof("Starting %v controller", controllerName)
	defer klog.Infof("Shutting down %v controller", controllerName)
 // 1、等待 informer 同步缓存
	if !cache.WaitForNamedCacheSync(rsc.Kind, stopCh, rsc.podListerSynced, rsc.rsListerSynced) {
		return
	}
 // 2、启动 5 个 goroutine 执行 worker 方法
	for i := 0; i < workers; i++ {
		go wait.Until(rsc.worker, time.Second, stopCh)
	}

	<-stopCh
}
```

replicaSetController 初始化完成后会调用 `Run` 方法启动 5 个 goroutine 处理 informer 队列中的事件并进行 sync 操作，kube-controller-manager 中每个 controller 的启动操作都是如下所示流程。 

## 事件处理

**addPod**

- 1、判断 pod 是否处于删除状态，处于删除状态删除；
- 2、获取该 pod 关联的 rs 以及 rsKey，入队 rs 并更新 rsKey 的 expectations；
- 3、若 pod 对象没体现出关联的 rs 则为孤儿 pod，遍历 rsList 查找匹配的 rs，若该 rs.Namespace == pod.Namespace 并且 rs.Spec.Selector 匹配 pod.Labels，则说明该 pod 应该与此 rs 关联，将匹配的 rs 入队；

```go
// When a pod is created, enqueue the replica set that manages it and update its expectations.
func (rsc *ReplicaSetController) addPod(obj interface{}) {
   pod := obj.(*v1.Pod)

   if pod.DeletionTimestamp != nil {
      // on a restart of the controller manager, it's possible a new pod shows up in a state that
      // is already pending deletion. Prevent the pod from being a creation observation.
      rsc.deletePod(pod)
      return
   }

   // If it has a ControllerRef, that's all that matters.
   if controllerRef := metav1.GetControllerOf(pod); controllerRef != nil {
      rs := rsc.resolveControllerRef(pod.Namespace, controllerRef)
      if rs == nil {
         return
      }
      rsKey, err := controller.KeyFunc(rs)
      if err != nil {
         return
      }
      klog.V(4).Infof("Pod %s created: %#v.", pod.Name, pod)
      rsc.expectations.CreationObserved(rsKey)
      rsc.queue.Add(rsKey)
      return
   }

   // Otherwise, it's an orphan. Get a list of all matching ReplicaSets and sync
   // them to see if anyone wants to adopt it.
   // DO NOT observe creation because no controller should be waiting for an
   // orphan.
   rss := rsc.getPodReplicaSets(pod)
   if len(rss) == 0 {
      return
   }
   klog.V(4).Infof("Orphan Pod %s created: %#v.", pod.Name, pod)
   for _, rs := range rss {
      rsc.enqueueRS(rs)
   }
}
```

**updatePod**

- 1、如果 pod label 改变或者处于删除状态，则直接删除；
- 2、如果 pod 的 OwnerReference 发生改变，此时 oldRS 需要创建 pod，将 oldRS 入队；
- 3、获取 pod 关联的 rs，入队 rs，若 pod 当前处于 ready 并非 available 状态，则会再次将该 rs 加入到延迟队列中，因为 pod 从 ready 到 available 状态需要触发一次 status 的更新；
- 4、否则为孤儿 pod，遍历 rsList 查找匹配的 rs，若找到则将 rs 入队；

```go
// When a pod is updated, figure out what replica set/s manage it and wake them
// up. If the labels of the pod have changed we need to awaken both the old
// and new replica set. old and cur must be *v1.Pod types.
func (rsc *ReplicaSetController) updatePod(old, cur interface{}) {
   curPod := cur.(*v1.Pod)
   oldPod := old.(*v1.Pod)
   if curPod.ResourceVersion == oldPod.ResourceVersion {
      // Periodic resync will send update events for all known pods.
      // Two different versions of the same pod will always have different RVs.
      return
   }

   labelChanged := !reflect.DeepEqual(curPod.Labels, oldPod.Labels)
   if curPod.DeletionTimestamp != nil {
      // when a pod is deleted gracefully it's deletion timestamp is first modified to reflect a grace period,
      // and after such time has passed, the kubelet actually deletes it from the store. We receive an update
      // for modification of the deletion timestamp and expect an rs to create more replicas asap, not wait
      // until the kubelet actually deletes the pod. This is different from the Phase of a pod changing, because
      // an rs never initiates a phase change, and so is never asleep waiting for the same.
      rsc.deletePod(curPod)
      if labelChanged {
         // we don't need to check the oldPod.DeletionTimestamp because DeletionTimestamp cannot be unset.
         rsc.deletePod(oldPod)
      }
      return
   }

   curControllerRef := metav1.GetControllerOf(curPod)
   oldControllerRef := metav1.GetControllerOf(oldPod)
   controllerRefChanged := !reflect.DeepEqual(curControllerRef, oldControllerRef)
   if controllerRefChanged && oldControllerRef != nil {
      // The ControllerRef was changed. Sync the old controller, if any.
      if rs := rsc.resolveControllerRef(oldPod.Namespace, oldControllerRef); rs != nil {
         rsc.enqueueRS(rs)
      }
   }

   // If it has a ControllerRef, that's all that matters.
   if curControllerRef != nil {
      rs := rsc.resolveControllerRef(curPod.Namespace, curControllerRef)
      if rs == nil {
         return
      }
      klog.V(4).Infof("Pod %s updated, objectMeta %+v -> %+v.", curPod.Name, oldPod.ObjectMeta, curPod.ObjectMeta)
      rsc.enqueueRS(rs)
      // TODO: MinReadySeconds in the Pod will generate an Available condition to be added in
      // the Pod status which in turn will trigger a requeue of the owning replica set thus
      // having its status updated with the newly available replica. For now, we can fake the
      // update by resyncing the controller MinReadySeconds after the it is requeued because
      // a Pod transitioned to Ready.
      // Note that this still suffers from #29229, we are just moving the problem one level
      // "closer" to kubelet (from the deployment to the replica set controller).
      if !podutil.IsPodReady(oldPod) && podutil.IsPodReady(curPod) && rs.Spec.MinReadySeconds > 0 {
         klog.V(2).Infof("%v %q will be enqueued after %ds for availability check", rsc.Kind, rs.Name, rs.Spec.MinReadySeconds)
         // Add a second to avoid milliseconds skew in AddAfter.
         // See https://github.com/kubernetes/kubernetes/issues/39785#issuecomment-279959133 for more info.
         rsc.enqueueRSAfter(rs, (time.Duration(rs.Spec.MinReadySeconds)*time.Second)+time.Second)
      }
      return
   }

   // Otherwise, it's an orphan. If anything changed, sync matching controllers
   // to see if anyone wants to adopt it now.
   if labelChanged || controllerRefChanged {
      rss := rsc.getPodReplicaSets(curPod)
      if len(rss) == 0 {
         return
      }
      klog.V(4).Infof("Orphan Pod %s updated, objectMeta %+v -> %+v.", curPod.Name, oldPod.ObjectMeta, curPod.ObjectMeta)
      for _, rs := range rss {
         rsc.enqueueRS(rs)
      }
   }
}
```

**deletePod**

- 1、确认该对象是否为 pod；
- 2、判断是否为孤儿 pod；
- 3、获取其对应的 rs 以及 rsKey；
- 4、更新 expectations 中 rsKey 的 del 值；
- 5、将 rs 入队；

```go
// When a pod is deleted, enqueue the replica set that manages the pod and update its expectations.
// obj could be an *v1.Pod, or a DeletionFinalStateUnknown marker item.
func (rsc *ReplicaSetController) deletePod(obj interface{}) {
   pod, ok := obj.(*v1.Pod)

   // When a delete is dropped, the relist will notice a pod in the store not
   // in the list, leading to the insertion of a tombstone object which contains
   // the deleted key/value. Note that this value might be stale. If the pod
   // changed labels the new ReplicaSet will not be woken up till the periodic resync.
   if !ok {
      tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
      if !ok {
         utilruntime.HandleError(fmt.Errorf("couldn't get object from tombstone %+v", obj))
         return
      }
      pod, ok = tombstone.Obj.(*v1.Pod)
      if !ok {
         utilruntime.HandleError(fmt.Errorf("tombstone contained object that is not a pod %#v", obj))
         return
      }
   }

   controllerRef := metav1.GetControllerOf(pod)
   if controllerRef == nil {
      // No controller should care about orphans being deleted.
      return
   }
   rs := rsc.resolveControllerRef(pod.Namespace, controllerRef)
   if rs == nil {
      return
   }
   rsKey, err := controller.KeyFunc(rs)
   if err != nil {
      utilruntime.HandleError(fmt.Errorf("couldn't get key for object %#v: %v", rs, err))
      return
   }
   klog.V(4).Infof("Pod %s/%s deleted through %v, timestamp %+v: %#v.", pod.Namespace, pod.Name, utilruntime.GetCaller(), pod.DeletionTimestamp, pod)
   rsc.expectations.DeletionObserved(rsKey, controller.PodKey(pod))
   rsc.queue.Add(rsKey)
}
```

**AddRS  DeleteRS updateRS**

```go
func (rsc *ReplicaSetController) addRS(obj interface{}) {
   rs := obj.(*apps.ReplicaSet)
   klog.V(4).Infof("Adding %s %s/%s", rsc.Kind, rs.Namespace, rs.Name)
   rsc.enqueueRS(rs)
}

func (rsc *ReplicaSetController) deleteRS(obj interface{}) {
	rs, ok := obj.(*apps.ReplicaSet)
	if !ok {
		tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("couldn't get object from tombstone %#v", obj))
			return
		}
		rs, ok = tombstone.Obj.(*apps.ReplicaSet)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("tombstone contained object that is not a ReplicaSet %#v", obj))
			return
		}
	}

	key, err := controller.KeyFunc(rs)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("couldn't get key for object %#v: %v", rs, err))
		return
	}

	klog.V(4).Infof("Deleting %s %q", rsc.Kind, key)

	// Delete expectations for the ReplicaSet so if we create a new one with the same name it starts clean
	rsc.expectations.DeleteExpectations(key)

	rsc.queue.Add(key)
}

// callback when RS is updated
func (rsc *ReplicaSetController) updateRS(old, cur interface{}) {
	oldRS := old.(*apps.ReplicaSet)
	curRS := cur.(*apps.ReplicaSet)

	// TODO: make a KEP and fix informers to always call the delete event handler on re-create
	if curRS.UID != oldRS.UID {
		key, err := controller.KeyFunc(oldRS)
		if err != nil {
			utilruntime.HandleError(fmt.Errorf("couldn't get key for object %#v: %v", oldRS, err))
			return
		}
		rsc.deleteRS(cache.DeletedFinalStateUnknown{
			Key: key,
			Obj: oldRS,
		})
	}

	// You might imagine that we only really need to enqueue the
	// replica set when Spec changes, but it is safer to sync any
	// time this function is triggered. That way a full informer
	// resync can requeue any replica set that don't yet have pods
	// but whose last attempts at creating a pod have failed (since
	// we don't block on creation of pods) instead of those
	// replica sets stalling indefinitely. Enqueueing every time
	// does result in some spurious syncs (like when Status.Replica
	// is updated and the watch notification from it retriggers
	// this function), but in general extra resyncs shouldn't be
	// that bad as ReplicaSets that haven't met expectations yet won't
	// sync, and all the listing is done using local stores.
	if *(oldRS.Spec.Replicas) != *(curRS.Spec.Replicas) {
		klog.V(4).Infof("%v %v updated. Desired pod count change: %d->%d", rsc.Kind, curRS.Name, *(oldRS.Spec.Replicas), *(curRS.Spec.Replicas))
	}
	rsc.enqueueRS(curRS)
}
```



## 调解流程

**syncReplicaSet**

`ReplicaSetController` 启动的多个 Goroutine 会从队列中取出待处理的事件，然后调用 `syncReplicaSet` 进行同步，这个方法会按照传入的 `key` 从 缓存 中取出 ReplicaSet 对象，获取ds的期望状态，然后取出全部 Active【非success failed且删除字段为空】 的 Pod，获取所有po的当前状态。

```go
func (rsc *ReplicaSetController) syncReplicaSet(key string) error {
   startTime := time.Now()
   defer func() {
      klog.V(4).Infof("Finished syncing %v %q (%v)", rsc.Kind, key, time.Since(startTime))
   }()

   namespace, name, err := cache.SplitMetaNamespaceKey(key)
   if err != nil {
      return err
   }
     // 1、根据 ns/name 从 informer cache 中获取 rs 对象，
    // 若 rs 已经被删除则直接删除 expectations 中的对象
   rs, err := rsc.rsLister.ReplicaSets(namespace).Get(name)
   if errors.IsNotFound(err) {
      klog.V(4).Infof("%v %v has been deleted", rsc.Kind, key)
      rsc.expectations.DeleteExpectations(key)
      return nil
   }
   if err != nil {
      return err
   }
 // 2、判断该 rs 是否需要执行 sync 操作
   rsNeedsSync := rsc.expectations.SatisfiedExpectations(key)
   selector, err := metav1.LabelSelectorAsSelector(rs.Spec.Selector)
   if err != nil {
      utilruntime.HandleError(fmt.Errorf("error converting pod selector to selector: %v", err))
      return nil
   }

   // list all pods to include the pods that don't match the rs`s selector
   // anymore but has the stale controller ref.
   // TODO: Do the List and Filter in a single pass, or use an index.
     // 3、获取所有 pod list
   allPods, err := rsc.podLister.Pods(rs.Namespace).List(labels.Everything())
   if err != nil {
      return err
   }
   // Ignore inactive pods.
     // 4、过滤掉异常 pod，处于删除状态或者 failed、successed 状态的 pod 都为非 active 状态
   filteredPods := controller.FilterActivePods(allPods)

   // NOTE: filteredPods are pointing to objects from cache - if you need to
   // modify them, you need to copy it first.
      // 5、根据 pod label 进行过滤获取与该 rs 关联的 pod 列表，对于其中的孤儿 pod 若与该 rs label 匹配则进行关联，若已关联的 pod 与 rs label 不匹配则解除关联关系；
   filteredPods, err = rsc.claimPods(rs, selector, filteredPods)
   if err != nil {
      return err
   }

   var manageReplicasErr error
   if rsNeedsSync && rs.DeletionTimestamp == nil {
// 6、若需要 sync 则执行 manageReplicas 创建/删除 pod
      manageReplicasErr = rsc.manageReplicas(filteredPods, rs)
   }
   rs = rs.DeepCopy()
    // 7、计算 rs 当前的 status
   newStatus := calculateStatus(rs, filteredPods, manageReplicasErr)
// 8、更新 rs status
   // Always updates status as pods come up or die.
   updatedRS, err := updateReplicaSetStatus(rsc.kubeClient.AppsV1().ReplicaSets(rs.Namespace), rs, newStatus)
   if err != nil {
      // Multiple things could lead to this update failing. Requeuing the replica set ensures
      // Returning an error causes a requeue without forcing a hotloop
      return err
   }
     // 9、判断是否需要将 rs 加入到延迟队列中，若 rs 设置了 MinReadySeconds 字段则将该 rs 加入到延迟队列中
   // Resync the ReplicaSet after MinReadySeconds as a last line of defense to guard against clock-skew.
   if manageReplicasErr == nil && updatedRS.Spec.MinReadySeconds > 0 &&
      updatedRS.Status.ReadyReplicas == *(updatedRS.Spec.Replicas) &&
      updatedRS.Status.AvailableReplicas != *(updatedRS.Spec.Replicas) {
      rsc.queue.AddAfter(key, time.Duration(updatedRS.Spec.MinReadySeconds)*time.Second)
   }
   return manageReplicasErr
}
```



**claimPods**

随后执行的 `ClaimPods` 方法会获取一系列 Pod 的所有权，**如果当前的 Pod 与 ReplicaSet 的选择器匹配就会建立从属关系，否则就会释放持有的对象**，或者直接忽视无关的 Pod，建立和释放关系的方法就是 `AdoptPod` 和 `ReleasePod`，`AdoptPod` 会设置目标对象的 `metadata.OwnerReferences` 字段。而 `ReleasePod` 删除目标 Pod 中的 `metadata.OwnerReferences` 属性。无论是建立还是释放从属关系，都是根据 ReplicaSet 的选择器配置进行的，它们根据匹配的标签执行不同的操作。

```go
func (rsc *ReplicaSetController) claimPods(rs *apps.ReplicaSet, selector labels.Selector, filteredPods []*v1.Pod) ([]*v1.Pod, error) {
   // If any adoptions are attempted, we should first recheck for deletion with
   // an uncached quorum read sometime after listing Pods (see #42639).
   canAdoptFunc := controller.RecheckDeletionTimestamp(func() (metav1.Object, error) {
      fresh, err := rsc.kubeClient.AppsV1().ReplicaSets(rs.Namespace).Get(context.TODO(), rs.Name, metav1.GetOptions{})
      if err != nil {
         return nil, err
      }
      if fresh.UID != rs.UID {
         return nil, fmt.Errorf("original %v %v/%v is gone: got uid %v, wanted %v", rsc.Kind, rs.Namespace, rs.Name, fresh.UID, rs.UID)
      }
      return fresh, nil
   })
   cm := controller.NewPodControllerRefManager(rsc.podControl, rs, selector, rsc.GroupVersionKind, canAdoptFunc)
   return cm.ClaimPods(filteredPods)
}

func (m *PodControllerRefManager) ClaimPods(pods []*v1.Pod, filters ...func(*v1.Pod) bool) ([]*v1.Pod, error) {
	var claimed []*v1.Pod
	var errlist []error

	match := func(obj metav1.Object) bool {
		pod := obj.(*v1.Pod)
		// Check selector first so filters only run on potentially matching Pods.
		if !m.Selector.Matches(labels.Set(pod.Labels)) {
			return false
		}
		for _, filter := range filters {
			if !filter(pod) {
				return false
			}
		}
		return true
	}
	adopt := func(obj metav1.Object) error {
		return m.AdoptPod(obj.(*v1.Pod))
	}
	release := func(obj metav1.Object) error {
		return m.ReleasePod(obj.(*v1.Pod))
	}

	for _, pod := range pods {
		ok, err := m.ClaimObject(pod, match, adopt, release)
		if err != nil {
			errlist = append(errlist, err)
			continue
		}
		if ok {
			claimed = append(claimed, pod)
		}
	}
	return claimed, utilerrors.NewAggregate(errlist)
}
```



**manageReplicas** 

manageReplicas 是最核心的方法，它会计算 replicaSet 需要创建或者删除多少个 pod 并调用 apiserver 的接口进行操作，在此阶段仅仅是调用 apiserver 的接口进行创建，并不保证 pod 成功运行，如果在某一轮，未能成功创建的所有 Pod 对象，则不再创建剩余的 pod。一个周期内最多只能创建或删除 500 个 pod，若超过上限值未创建完成的 pod 数会在下一个 syncLoop 继续进行处理。

该方法主要逻辑，首先计算已存在 pod 数与期望数的差异；

**1 创建**

如果 diff < 0 说明 rs 实际的 pod 数未达到期望值需要继续创建 pod，首先会**将需要创建的 pod 数在 expectations 中进行记录**，然后调用 slowStartBatch 创建所需要的 pod，slowStartBatch 以指数级增长的方式批量创建 pod，创建 pod 过程中若出现 为其他 err 则终止创建操作并更新 expectations，`slowStartBatch` 会批量创建出已计算出的 diff pod 数，创建的 pod 数依次为 1、2、4、8......，呈指数级增长。创建 pod 操作提交到 apiserver 后会经过认证、鉴权、以及动态访问控制三个步骤，使真的创建失败了，等到 expectations 过期后在下一个 syncLoop 时会重新创建。

```go
// manageReplicas checks and updates replicas for the given ReplicaSet.
// Does NOT modify <filteredPods>.
// It will requeue the replica set in case of an error while creating/deleting pods.
func (rsc *ReplicaSetController) manageReplicas(filteredPods []*v1.Pod, rs *apps.ReplicaSet) error {
	diff := len(filteredPods) - int(*(rs.Spec.Replicas))
	rsKey, err := controller.KeyFunc(rs)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("couldn't get key for %v %#v: %v", rsc.Kind, rs, err))
		return nil
	}
	if diff < 0 {
		diff *= -1
		if diff > rsc.burstReplicas {
			diff = rsc.burstReplicas
		}
		// TODO: Track UIDs of creates just like deletes. The problem currently
		// is we'd need to wait on the result of a create to record the pod's
		// UID, which would require locking *across* the create, which will turn
		// into a performance bottleneck. We should generate a UID for the pod
		// beforehand and store it via ExpectCreations.
		rsc.expectations.ExpectCreations(rsKey, diff)
		klog.V(2).InfoS("Too few replicas", "replicaSet", klog.KObj(rs), "need", *(rs.Spec.Replicas), "creating", diff)
		// Batch the pod creates. Batch sizes start at SlowStartInitialBatchSize
		// and double with each successful iteration in a kind of "slow start".
		// This handles attempts to start large numbers of pods that would
		// likely all fail with the same error. For example a project with a
		// low quota that attempts to create a large number of pods will be
		// prevented from spamming the API service with the pod create requests
		// after one of its pods fails.  Conveniently, this also prevents the
		// event spam that those failures would generate.
		successfulCreations, err := slowStartBatch(diff, controller.SlowStartInitialBatchSize, func() error {
			err := rsc.podControl.CreatePodsWithControllerRef(rs.Namespace, &rs.Spec.Template, rs, metav1.NewControllerRef(rs, rsc.GroupVersionKind))
			if err != nil {
				if errors.HasStatusCause(err, v1.NamespaceTerminatingCause) {
					// if the namespace is being terminated, we don't have to do
					// anything because any creation will fail
					return nil
				}
			}
			return err
		})

		// Any skipped pods that we never attempted to start shouldn't be expected.
		// The skipped pods will be retried later. The next controller resync will
		// retry the slow start process.
		if skippedPods := diff - successfulCreations; skippedPods > 0 {
			klog.V(2).Infof("Slow-start failure. Skipping creation of %d pods, decrementing expectations for %v %v/%v", skippedPods, rsc.Kind, rs.Namespace, rs.Name)
			for i := 0; i < skippedPods; i++ {
				// Decrement the expected number of creates because the informer won't observe this pod
				rsc.expectations.CreationObserved(rsKey)
			}
		}
		return err
	} else if diff > 0 {
	}

	return nil
}
```

**2 删除**

如果 diff > 0 说明可能是一次缩容操作需要删除多余的 pod，如果需要删除全部的 pod 则直接进行删除，否则会通过 getPodsToDelete 方法筛选出需要删除的 pod，具体的筛选策略如下，然后并发删除这些 pod，对于删除失败操作也会记录在 expectations 中；若 diff > 0 时再删除 pod 阶段会调用`getPodsToDelete` 对 pod 进行筛选操作，此阶段会选出最劣质的 pod，下面是用到的 6 种筛选方法，这几个排序规则遵循互斥原则，从上到下进行匹配，符合条件则排序完成。

- 1、判断是否绑定了 node：Unassigned < assigned；
- 2、判断 pod phase：PodPending < PodUnknown < PodRunning；
- 3、判断 pod 状态：Not ready < ready；
- 4、若 pod 都为 ready，则按运行时间排序，运行时间最短会被删除：empty time < less time < more time；
- 5、根据 pod 重启次数排序：higher restart counts < lower restart counts；
- 6、按 pod 创建时间进行排序：Empty creation time pods < newer pods < older pods；

```go
func (rsc *ReplicaSetController) manageReplicas(filteredPods []*v1.Pod, rs *apps.ReplicaSet) error {
   diff := len(filteredPods) - int(*(rs.Spec.Replicas))
   rsKey, err := controller.KeyFunc(rs)
   if err != nil {
      utilruntime.HandleError(fmt.Errorf("couldn't get key for %v %#v: %v", rsc.Kind, rs, err))
      return nil
   }
   if diff < 0 {
   } else if diff > 0 {
      if diff > rsc.burstReplicas {
         diff = rsc.burstReplicas
      }
      klog.V(2).InfoS("Too many replicas", "replicaSet", klog.KObj(rs), "need", *(rs.Spec.Replicas), "deleting", diff)

      relatedPods, err := rsc.getIndirectlyRelatedPods(rs)
      utilruntime.HandleError(err)

      // Choose which Pods to delete, preferring those in earlier phases of startup.
      podsToDelete := getPodsToDelete(filteredPods, relatedPods, diff)

      // Snapshot the UIDs (ns/name) of the pods we're expecting to see
      // deleted, so we know to record their expectations exactly once either
      // when we see it as an update of the deletion timestamp, or as a delete.
      // Note that if the labels on a pod/rs change in a way that the pod gets
      // orphaned, the rs will only wake up after the expectations have
      // expired even if other pods are deleted.
      rsc.expectations.ExpectDeletions(rsKey, getPodKeys(podsToDelete))

      errCh := make(chan error, diff)
      var wg sync.WaitGroup
      wg.Add(diff)
      for _, pod := range podsToDelete {
         go func(targetPod *v1.Pod) {
            defer wg.Done()
            if err := rsc.podControl.DeletePod(rs.Namespace, targetPod.Name, rs); err != nil {
               // Decrement the expected number of deletes because the informer won't observe this deletion
               podKey := controller.PodKey(targetPod)
               rsc.expectations.DeletionObserved(rsKey, podKey)
               if !apierrors.IsNotFound(err) {
                  klog.V(2).Infof("Failed to delete %v, decremented expectations for %v %s/%s", podKey, rsc.Kind, rs.Namespace, rs.Name)
                  errCh <- err
               }
            }
         }(pod)
      }
      wg.Wait()

      select {
      case err := <-errCh:
         // all errors have been reported before and they're likely to be the same, so we'll only return the first one we hit.
         if err != nil {
            return err
         }
      default:
      }
   }

   return nil
}

// Less compares two pods with corresponding ranks and returns true if the first
// one should be preferred for deletion.
// 筛选策略
func (s ActivePodsWithRanks) Less(i, j int) bool {
	// 1. Unassigned < assigned
	// If only one of the pods is unassigned, the unassigned one is smaller
	if s.Pods[i].Spec.NodeName != s.Pods[j].Spec.NodeName && (len(s.Pods[i].Spec.NodeName) == 0 || len(s.Pods[j].Spec.NodeName) == 0) {
		return len(s.Pods[i].Spec.NodeName) == 0
	}
	// 2. PodPending < PodUnknown < PodRunning
	if podPhaseToOrdinal[s.Pods[i].Status.Phase] != podPhaseToOrdinal[s.Pods[j].Status.Phase] {
		return podPhaseToOrdinal[s.Pods[i].Status.Phase] < podPhaseToOrdinal[s.Pods[j].Status.Phase]
	}
	// 3. Not ready < ready
	// If only one of the pods is not ready, the not ready one is smaller
	if podutil.IsPodReady(s.Pods[i]) != podutil.IsPodReady(s.Pods[j]) {
		return !podutil.IsPodReady(s.Pods[i])
	}
	// 4. Doubled up < not doubled up
	// If one of the two pods is on the same node as one or more additional
	// ready pods that belong to the same replicaset, whichever pod has more
	// colocated ready pods is less
	if s.Rank[i] != s.Rank[j] {
		return s.Rank[i] > s.Rank[j]
	}
	// TODO: take availability into account when we push minReadySeconds information from deployment into pods,
	//       see https://github.com/kubernetes/kubernetes/issues/22065
	// 5. Been ready for empty time < less time < more time
	// If both pods are ready, the latest ready one is smaller
	if podutil.IsPodReady(s.Pods[i]) && podutil.IsPodReady(s.Pods[j]) {
		readyTime1 := podReadyTime(s.Pods[i])
		readyTime2 := podReadyTime(s.Pods[j])
		if !readyTime1.Equal(readyTime2) {
			return afterOrZero(readyTime1, readyTime2)
		}
	}
	// 6. Pods with containers with higher restart counts < lower restart counts
	if maxContainerRestarts(s.Pods[i]) != maxContainerRestarts(s.Pods[j]) {
		return maxContainerRestarts(s.Pods[i]) > maxContainerRestarts(s.Pods[j])
	}
	// 7. Empty creation time pods < newer pods < older pods
	if !s.Pods[i].CreationTimestamp.Equal(&s.Pods[j].CreationTimestamp) {
		return afterOrZero(&s.Pods[i].CreationTimestamp, &s.Pods[j].CreationTimestamp)
	}
	return false
}
```



## expectations 

？？？？？？？？？？？？？？？

通过上面的分析可知，在 rs 每次入队后进行 sync 操作时，首先需要判断该 rs 是否满足 expectations 机制，那么这个 expectations 的目的是什么？其实，**rs 除了有 informer 的缓存外，还有一个本地缓存就是 expectations，expectations 会记录 rs 所有对象需要 add/del 的 pod 数量，若两者都为 0 则说明该 rs 所期望创建的 pod 或者删除的 pod 数已经被满足，若不满足则说明某次在 syncLoop 中创建或者删除 pod 时有失败的操作，则需要等待 expectations 过期后再次同步该 rs。**

通过上面对 eventHandler 的分析，再来总结一下触发 replicaSet 对象发生同步事件的条件：

- 1、与 rs 相关的：AddRS、UpdateRS、DeleteRS；
- 2、与 pod 相关的：AddPod、UpdatePod、DeletePod；
- 3、informer 二级缓存的同步；

但是所有的更新事件是否都需要执行 sync 操作？对于除 rs.Spec.Replicas 之外的更新操作其实都没必要执行 sync 操作【manageReplicas?】，因为 spec 其他字段和 status 的更新都不需要创建或者删除 pod。

在 sync 操作真正开始之前，依据 expectations 机制进行判断，确定是否要真正地启动一次 sync，因为在 eventHandler 阶段也会更新 expectations 值，从上面的 eventHandler 中可以看到在 addPod 中会调用 rsc.expectations.CreationObserved 更新 rsKey 的  expectations，将其 add 值 -1，在 deletePod 中调用 rsc.expectations.DeletionObserved 将其 del 值 -1。所以等到 sync 时，若 controllerKey(name 或者 ns/name)满足 expectations 机制则进行 sync 操作，而 updatePod 并不会修改 expectations，所以，expectations 的设计就是当需要创建或删除 pod 才会触发对应的 sync 操作，expectations 机制的目的就是减少不必要的 sync 操作。

什么条件下 expectations 机制会满足？如下情况，需要sync，即manageReplicas

- 1、当 expectations 中不存在 rsKey 时，也就说首次创建 rs 时；
- 2、当 expectations 中 del 以及 add 值都为 0 时，即 rs 所需要创建或者删除的 pod 数都已满足；
- 3、当 expectations 过期时，即超过 5 分钟未进行 sync 操作；

```go
// SatisfiedExpectations returns true if the required adds/dels for the given controller have been observed.
// Add/del counts are established by the controller at sync time, and updated as controllees are observed by the controller
// manager.
func (r *ControllerExpectations) SatisfiedExpectations(controllerKey string) bool {
   if exp, exists, err := r.GetExpectations(controllerKey); exists {
      if exp.Fulfilled() {
         klog.V(4).Infof("Controller expectations fulfilled %#v", exp)
         return true
      } else if exp.isExpired() {
         klog.V(4).Infof("Controller expectations expired %#v", exp)
         return true
      } else {
         klog.V(4).Infof("Controller still waiting on expectations %#v", exp)
         return false
      }
   } else if err != nil {
      klog.V(2).Infof("Error encountered while checking expectations %#v, forcing sync", err)
   } else {
      // When a new controller is created, it doesn't have expectations.
      // When it doesn't see expected watch events for > TTL, the expectations expire.
      // - In this case it wakes up, creates/deletes controllees, and sets expectations again.
      // When it has satisfied expectations and no controllees need to be created/destroyed > TTL, the expectations expire.
      // - In this case it continues without setting expectations till it needs to create/delete controllees.
      klog.V(4).Infof("Controller %v either never recorded expectations, or the ttl expired.", controllerKey)
   }
   // Trigger a sync if we either encountered and error (which shouldn't happen since we're
   // getting from local store) or this controller hasn't established expectations.
   return true
}
```









# depolyment 控制器

> 引用：
>
> https://developer.aliyun.com/article/774817
>
> https://www.danielhu.cn/post/k8s/deployment-2/
>
> https://cloud.tencent.com/developer/article/1557568

Deployment 通过控制 ReplicaSet，ReplicaSet 再控制 Pod，最终由 Controller 驱动达到期望状态。在控制器模式下，每次操作对象都会触发一次事件，然后 controller 会进行一次 syncLoop 操作，controller 是通过 informer 监听事件以及进行 ListWatch 操作的。 deployment 的本质是控制 replicaSet，replicaSet 会控制 pod，然后由 controller 驱动各个对象达到期望状态。DeploymentController 是 Deployment 资源的控制器，其通过 DeploymentInformer、ReplicaSetInformer、PodInformer 监听三种资源，当三种资源变化时会触发 DeploymentController 中的 syncLoop 操作。

![deployment-controller](https://tvax4.sinaimg.cn/large/ad5fbf65gy1gj6twofn24j20es09s43a.jpg) 



Deployment 处理 Pod 的**滚动更新、回滚以及支持副本的水平扩缩容、暂停恢复**。 

Deployment 是 Kubernetes 中常用的对象类型，它解决了 ReplicaSet 更新的诸多问题，通过对 ReplicaSet 和 Pod 进行组装支持了滚动更新、回滚以及扩容等高级功能，通过对 Deployment 的学习既能让我们了解整个常见资源的实现也能帮助我们理解如何将 Kubernetes 内置的对象组合成更复杂的自定义资源。 

## 启动

`ctx.ComponentConfig.DeploymentController.ConcurrentDeploymentSyncs` 指定了 deployment controller 中工作的 goroutine 数量，默认值为 5，即会启动五个 goroutine 从 workqueue 中取出 object 并进行 sync 操作，该参数的默认值定义在 `k8s.io/kubernetes/pkg/controller/deployment/config/v1alpha1/defaults.go`  中。 

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

// NewDeploymentController creates a new DeploymentController.
func NewDeploymentController(dInformer appsinformers.DeploymentInformer, rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, client clientset.Interface) (*DeploymentController, error) {
	eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartStructuredLogging(0)
	eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: client.CoreV1().Events("")})

	if client != nil && client.CoreV1().RESTClient().GetRateLimiter() != nil {
		if err := ratelimiter.RegisterMetricAndTrackRateLimiterUsage("deployment_controller", client.CoreV1().RESTClient().GetRateLimiter()); err != nil {
			return nil, err
		}
	}
	dc := &DeploymentController{
		client:        client,
		eventRecorder: eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "deployment-controller"}),
		queue:         workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "deployment"),
	}
	dc.rsControl = controller.RealRSControl{
		KubeClient: client,
		Recorder:   dc.eventRecorder,
	}

	dInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    dc.addDeployment,
		UpdateFunc: dc.updateDeployment,
		// This will enter the sync loop and no-op, because the deployment has been deleted from the store.
		DeleteFunc: dc.deleteDeployment,
	})
	rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    dc.addReplicaSet,
		UpdateFunc: dc.updateReplicaSet,
		DeleteFunc: dc.deleteReplicaSet,
	})
	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		DeleteFunc: dc.deletePod,
	})

	dc.syncHandler = dc.syncDeployment
	dc.enqueueDeployment = dc.enqueue

	dc.dLister = dInformer.Lister()
	dc.rsLister = rsInformer.Lister()
	dc.podLister = podInformer.Lister()
	dc.dListerSynced = dInformer.Informer().HasSynced
	dc.rsListerSynced = rsInformer.Informer().HasSynced
	dc.podListerSynced = podInformer.Informer().HasSynced
	return dc, nil
}
```



## 调解流程

- 根据传入的键获取当前 Deployment 资源的最新状态；
- 调用 getReplicaSetsForDeployment 获取集群中与 Deployment 相关的全部 ReplicaSet；查找集群中的全部 ReplicaSet；根据 Deployment 的选择器对 ReplicaSet **建立或者释放**从属关系，释放不再控制的rs，收养关联的孤儿rs。通过patch rs中的OwnerReferences标记；
- 调用 getPodMapForDeployment 获取当前 Deployment 对象相关的从 ReplicaSet 到 Pod 的映射；根据选择器查找全部的 Pod；根据 Pod 的控制器 ReplicaSet 对上述 Pod 进行分类；
- 如果当前的 Deployment 处于暂停状态或者需要进行扩容，就会调用 sync 方法同步 Deployment;
- 在正常情况下会根据规格中的策略对 Deployment 进行更新；Recreate 策略会调用 rolloutRecreate 方法，它会先杀掉所有存在的 Pod 后启动新的 Pod 副本；RollingUpdate 策略会调用 rolloutRolling 方法，根据 maxSurge 和 maxUnavailable 配置对 Pod 进行滚动更新；

```go
func (dc *DeploymentController) syncDeployment(key string) error {
	startTime := time.Now()
	klog.V(4).Infof("Started syncing deployment %q (%v)", key, startTime)
	defer func() {
		klog.V(4).Infof("Finished syncing deployment %q (%v)", key, time.Since(startTime))
	}()

	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}
	deployment, err := dc.dLister.Deployments(namespace).Get(name)
	if errors.IsNotFound(err) {
		klog.V(2).Infof("Deployment %v has been deleted", key)
		return nil
	}
	if err != nil {
		return err
	}

	// Deep-copy otherwise we are mutating our cache.
	// TODO: Deep-copy only when needed.
	d := deployment.DeepCopy()

	everything := metav1.LabelSelector{}
	if reflect.DeepEqual(d.Spec.Selector, &everything) {
		dc.eventRecorder.Eventf(d, v1.EventTypeWarning, "SelectingAll", "This deployment is selecting all pods. A non-empty selector is required.")
		if d.Status.ObservedGeneration < d.Generation {
			d.Status.ObservedGeneration = d.Generation
			dc.client.AppsV1().Deployments(d.Namespace).UpdateStatus(context.TODO(), d, metav1.UpdateOptions{})
		}
		return nil
	}

	// List ReplicaSets owned by this Deployment, while reconciling ControllerRef
	// through adoption/orphaning.
	rsList, err := dc.getReplicaSetsForDeployment(d)
	if err != nil {
		return err
	}
	// List all Pods owned by this Deployment, grouped by their ReplicaSet.
	// Current uses of the podMap are:
	//
	// * check if a Pod is labeled correctly with the pod-template-hash label.
	// * check that no old Pods are running in the middle of Recreate Deployments.
	podMap, err := dc.getPodMapForDeployment(d, rsList)
	if err != nil {
		return err
	}

	if d.DeletionTimestamp != nil {
		return dc.syncStatusOnly(d, rsList)
	}

	// Update deployment conditions with an Unknown condition when pausing/resuming
	// a deployment. In this way, we can be sure that we won't timeout when a user
	// resumes a Deployment with a set progressDeadlineSeconds.
	if err = dc.checkPausedConditions(d); err != nil {
		return err
	}

	if d.Spec.Paused {
		return dc.sync(d, rsList)
	}

	// rollback is not re-entrant in case the underlying replica sets are updated with a new
	// revision so we should ensure that we won't proceed to update replica sets until we
	// make sure that the deployment has cleaned up its rollback spec in subsequent enqueues.
	if getRollbackTo(d) != nil {
		return dc.rollback(d, rsList)
	}

	scalingEvent, err := dc.isScalingEvent(d, rsList)
	if err != nil {
		return err
	}
	if scalingEvent {
		return dc.sync(d, rsList)
	}

	switch d.Spec.Strategy.Type {
	case apps.RecreateDeploymentStrategyType:
		return dc.rolloutRecreate(d, rsList, podMap)
	case apps.RollingUpdateDeploymentStrategyType:
		return dc.rolloutRolling(d, rsList)
	}
	return fmt.Errorf("unexpected deployment strategy type: %s", d.Spec.Strategy.Type)
}

```

其中的getReplicaSetsForDeployment 方法，该方法获取 Deployment 持有的 ReplicaSet 时会重新与集群中符合条件的 ReplicaSet 通过 ownerReferences 建立或者释放从属关系，执行的逻辑与 ReplicaSet 中调用 AdoptPod/ReleasePod 几乎完全相同。调用关系链如下。

```go
// getReplicaSetsForDeployment uses ControllerRefManager to reconcile
// ControllerRef by adopting and orphaning.
// It returns the list of ReplicaSets that this Deployment should manage.
func (dc *DeploymentController) getReplicaSetsForDeployment(d *apps.Deployment) ([]*apps.ReplicaSet, error) {
	// List all ReplicaSets to find those we own but that no longer match our
	// selector. They will be orphaned by ClaimReplicaSets().
	rsList, err := dc.rsLister.ReplicaSets(d.Namespace).List(labels.Everything())
	if err != nil {
		return nil, err
	}
	deploymentSelector, err := metav1.LabelSelectorAsSelector(d.Spec.Selector)
	if err != nil {
		return nil, fmt.Errorf("deployment %s/%s has invalid label selector: %v", d.Namespace, d.Name, err)
	}
	// If any adoptions are attempted, we should first recheck for deletion with
	// an uncached quorum read sometime after listing ReplicaSets (see #42639).
	canAdoptFunc := controller.RecheckDeletionTimestamp(func() (metav1.Object, error) {
		fresh, err := dc.client.AppsV1().Deployments(d.Namespace).Get(context.TODO(), d.Name, metav1.GetOptions{})
		if err != nil {
			return nil, err
		}
		if fresh.UID != d.UID {
			return nil, fmt.Errorf("original Deployment %v/%v is gone: got uid %v, wanted %v", d.Namespace, d.Name, fresh.UID, d.UID)
		}
		return fresh, nil
	})
	cm := controller.NewReplicaSetControllerRefManager(dc.rsControl, d, deploymentSelector, controllerKind, canAdoptFunc)
	return cm.ClaimReplicaSets(rsList)
}

// ClaimReplicaSets tries to take ownership of a list of ReplicaSets.
//
// It will reconcile the following:
//   * Adopt orphans if the selector matches.
//   * Release owned objects if the selector no longer matches.
//
// A non-nil error is returned if some form of reconciliation was attempted and
// failed. Usually, controllers should try again later in case reconciliation
// is still needed.
//
// If the error is nil, either the reconciliation succeeded, or no
// reconciliation was necessary. The list of ReplicaSets that you now own is
// returned.
func (m *ReplicaSetControllerRefManager) ClaimReplicaSets(sets []*apps.ReplicaSet) ([]*apps.ReplicaSet, error) {
	var claimed []*apps.ReplicaSet
	var errlist []error

	match := func(obj metav1.Object) bool {
		return m.Selector.Matches(labels.Set(obj.GetLabels()))
	}
	adopt := func(obj metav1.Object) error {
		return m.AdoptReplicaSet(obj.(*apps.ReplicaSet))
	}
	release := func(obj metav1.Object) error {
		return m.ReleaseReplicaSet(obj.(*apps.ReplicaSet))
	}

	for _, rs := range sets {
		ok, err := m.ClaimObject(rs, match, adopt, release)
		if err != nil {
			errlist = append(errlist, err)
			continue
		}
		if ok {
			claimed = append(claimed, rs)
		}
	}
	return claimed, utilerrors.NewAggregate(errlist)
}

// ClaimObject tries to take ownership of an object for this controller.
//
// It will reconcile the following:
//   * Adopt orphans if the match function returns true.
//   * Release owned objects if the match function returns false.
//
// A non-nil error is returned if some form of reconciliation was attempted and
// failed. Usually, controllers should try again later in case reconciliation
// is still needed.
//
// If the error is nil, either the reconciliation succeeded, or no
// reconciliation was necessary. The returned boolean indicates whether you now
// own the object.
//
// No reconciliation will be attempted if the controller is being deleted.
func (m *BaseControllerRefManager) ClaimObject(obj metav1.Object, match func(metav1.Object) bool, adopt, release func(metav1.Object) error) (bool, error) {
	controllerRef := metav1.GetControllerOfNoCopy(obj)
	if controllerRef != nil {
		if controllerRef.UID != m.Controller.GetUID() {
			// Owned by someone else. Ignore.
			return false, nil
		}
		if match(obj) {
			// We already own it and the selector matches.
			// Return true (successfully claimed) before checking deletion timestamp.
			// We're still allowed to claim things we already own while being deleted
			// because doing so requires taking no actions.
			return true, nil
		}
		// Owned by us but selector doesn't match.
		// Try to release, unless we're being deleted.
		if m.Controller.GetDeletionTimestamp() != nil {
			return false, nil
		}
		if err := release(obj); err != nil {
			// If the pod no longer exists, ignore the error.
			if errors.IsNotFound(err) {
				return false, nil
			}
			// Either someone else released it, or there was a transient error.
			// The controller should requeue and try again if it's still stale.
			return false, err
		}
		// Successfully released.
		return false, nil
	}

	// It's an orphan.
	if m.Controller.GetDeletionTimestamp() != nil || !match(obj) {
		// Ignore if we're being deleted or selector doesn't match.
		return false, nil
	}
	if obj.GetDeletionTimestamp() != nil {
		// Ignore if the object is being deleted
		return false, nil
	}
	// Selector matches. Try to adopt.
	if err := adopt(obj); err != nil {
		// If the pod no longer exists, ignore the error.
		if errors.IsNotFound(err) {
			return false, nil
		}
		// Either someone else claimed it first, or there was a transient error.
		// The controller should requeue and try again if it's still orphaned.
		return false, err
	}
	// Successfully adopted.
	return true, nil
}
```





## 扩容缩容

> 引用 https://draveness.me/kubernetes-deployment/



如果当前需要更新的 Deployment 经过 `isScalingEvent` 的检查发现更新事件实际上是一次扩容或者缩容，也就是 ReplicaSet 持有的 Pod 数量和规格中的 `Replicas` 字段并不一致，那么就会调用 `sync` 方法对 Deployment 进行同步。

同步的过程其实比较简单，该方法会从 apiserver 中拿到当前 Deployment 对应的最新的一个 ReplicaSet 【按照创建时间排序，且rs的Spec.Template与deploy的Spec.Template相同】和历史的 ReplicaSet 并调用 `scale` 方法开始扩容。

```go
// sync is responsible for reconciling deployments on scaling events or when they
// are paused.
func (dc *DeploymentController) sync(d *apps.Deployment, rsList []*apps.ReplicaSet) error {
   newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(d, rsList, false)
   if err != nil {
      return err
   }
   if err := dc.scale(d, newRS, oldRSs); err != nil {
      // If we get an error while trying to scale, the deployment will be requeued
      // so we can abort this resync
      return err
   }

   // Clean up the deployment when it's paused and no rollback is in flight.
   if d.Spec.Paused && getRollbackTo(d) == nil {
      if err := dc.cleanupDeployment(oldRSs, d); err != nil {
         return err
      }
   }

   allRSs := append(oldRSs, newRS)
   return dc.syncDeploymentStatus(allRSs, newRS, d)
}
```

如果集群中只有一个活跃的 ReplicaSet【活跃的rs即其副本大于0】，那么就会对该 ReplicaSet 进行扩缩容；

如果不存在活跃的 ReplicaSet 对象，就会选择最新的 ReplicaSet 进行操作，当调用 `IsSaturated` 方法发现当前的 Deployment 对应的副本数量已经饱和时就会删除所有历史版本 ReplicaSet 持有的 Pod 副本，将副本数置0。

```go
func (dc *DeploymentController) scale(deployment *apps.Deployment, newRS *apps.ReplicaSet, oldRSs []*apps.ReplicaSet) error {
   // If there is only one active replica set then we should scale that up to the full count of the
   // deployment. If there is no active replica set, then we should scale up the newest replica set.
   if activeOrLatest := deploymentutil.FindActiveOrLatest(newRS, oldRSs); activeOrLatest != nil {
      if *(activeOrLatest.Spec.Replicas) == *(deployment.Spec.Replicas) {
         return nil
      }
      _, _, err := dc.scaleReplicaSetAndRecordEvent(activeOrLatest, *(deployment.Spec.Replicas), deployment)
      return err
   }

   // If the new replica set is saturated, old replica sets should be fully scaled down.
   // This case handles replica set adoption during a saturated new replica set.
   if deploymentutil.IsSaturated(deployment, newRS) {
      for _, old := range controller.FilterActiveReplicaSets(oldRSs) {
         if _, _, err := dc.scaleReplicaSetAndRecordEvent(old, 0, deployment); err != nil {
            return err
         }
      }
      return nil
   }
  }      
```

但是在 Deployment 使用滚动更新策略时，如果发现当前的 ReplicaSet 并没有饱和并且存在多个活跃的 ReplicaSet 对象就会按照比例分别对各个活跃的 ReplicaSet 进行扩容或者缩容： 

- 通过 FilterActiveReplicaSets 获取所有活跃的 ReplicaSet 对象；

- 调用 GetReplicaCountForReplicaSets 计算当前 Deployment 对应 ReplicaSet 持有的全部 Pod 副本个数；

- 根据 Deployment 对象配置的 Replicas 和最大额外可以存在的副本数 maxSurge 以计算 Deployment 允许创建的 Pod 数量；

- 通过 allowedSize 和 allRSsReplicas 计算出需要增加或者删除的副本数；

- 根据 deploymentReplicasToAdd 变量的符号对 ReplicaSet 数组进行排序并确定当前的操作时扩容还是缩容；如果 deploymentReplicasToAdd > 0，ReplicaSet 将按照从新到旧的顺序依次进行扩容；如果 deploymentReplicasToAdd < 0，ReplicaSet 将按照从旧到新的顺序依次进行缩容；

  maxSurge、maxUnavailable 是两个处理滚动更新时需要关注的参数。

```go
// There are old replica sets with pods and the new replica set is not saturated.
// We need to proportionally scale all replica sets (new and old) in case of a
// rolling deployment.
if deploymentutil.IsRollingUpdate(deployment) {
   allRSs := controller.FilterActiveReplicaSets(append(oldRSs, newRS))
   allRSsReplicas := deploymentutil.GetReplicaCountForReplicaSets(allRSs)

   allowedSize := int32(0)
   if *(deployment.Spec.Replicas) > 0 {
      allowedSize = *(deployment.Spec.Replicas) + deploymentutil.MaxSurge(*deployment)
   }

   // Number of additional replicas that can be either added or removed from the total
   // replicas count. These replicas should be distributed proportionally to the active
   // replica sets.
   deploymentReplicasToAdd := allowedSize - allRSsReplicas

   // The additional replicas should be distributed proportionally amongst the active
   // replica sets from the larger to the smaller in size replica set. Scaling direction
   // drives what happens in case we are trying to scale replica sets of the same size.
   // In such a case when scaling up, we should scale up newer replica sets first, and
   // when scaling down, we should scale down older replica sets first.
   var scalingOperation string
   switch {
   case deploymentReplicasToAdd > 0:
      sort.Sort(controller.ReplicaSetsBySizeNewer(allRSs))
      scalingOperation = "up"

   case deploymentReplicasToAdd < 0:
      sort.Sort(controller.ReplicaSetsBySizeOlder(allRSs))
      scalingOperation = "down"
   }
```

因为当前的 Deployment 持有了多个活跃的 ReplicaSet，所以在计算了需要增加或者删除的副本个数 `deploymentReplicasToAdd` 之后，就会为多个活跃的 ReplicaSet 分配每个 ReplicaSet 需要改变的副本数，`GetProportion` 会根据以下几个参数决定最后的结果:

1. Deployment 期望的 Pod 副本数量；
2. 需要新增或者减少的副本数量；
3. Deployment 当前通过 ReplicaSet 持有 Pod 的总数量；

Kubernetes 会在 `getReplicaSetFraction` 使用下面的公式计算每一个 ReplicaSet 在 Deployment 资源中的占比，最后会返回该 ReplicaSet 需要改变的副本数：

![deployment-replicaset-get-proportion](https://img.draveness.me/2019-02-24-deployment-replicaset-get-proportion.png)

该结果又会与目前期望的剩余变化量进行对比，保证变化的副本数量不会超过期望值。

最后会直接调用 `scaleReplicaSet` 将每一个 ReplicaSet 都扩容或者缩容到我们期望的副本数： 

```go
// Iterate over all active replica sets and estimate proportions for each of them.
// The absolute value of deploymentReplicasAdded should never exceed the absolute
// value of deploymentReplicasToAdd.
deploymentReplicasAdded := int32(0)
nameToSize := make(map[string]int32)
for i := range allRSs {
   rs := allRSs[i]

   // Estimate proportions if we have replicas to add, otherwise simply populate
   // nameToSize with the current sizes for each replica set.
   if deploymentReplicasToAdd != 0 {
      proportion := deploymentutil.GetProportion(rs, *deployment, deploymentReplicasToAdd, deploymentReplicasAdded)

      nameToSize[rs.Name] = *(rs.Spec.Replicas) + proportion
      deploymentReplicasAdded += proportion
   } else {
      nameToSize[rs.Name] = *(rs.Spec.Replicas)
   }
}

// Update all replica sets
for i := range allRSs {
   rs := allRSs[i]

   // Add/remove any leftovers to the largest replica set.
   if i == 0 && deploymentReplicasToAdd != 0 {
      leftover := deploymentReplicasToAdd - deploymentReplicasAdded
      nameToSize[rs.Name] = nameToSize[rs.Name] + leftover
      if nameToSize[rs.Name] < 0 {
         nameToSize[rs.Name] = 0
      }
   }

   // TODO: Use transactions when we have them.
   if _, _, err := dc.scaleReplicaSet(rs, nameToSize[rs.Name], deployment, scalingOperation); err != nil {
      // Return as soon as we fail, the deployment is requeued
      return err
   }
}
```

scaleReplicaSet方法，直接修改目标 ReplicaSet 规格中的 `Replicas` 参数和注解 deployment.kubernetes.io/desired-replicas、 deployment.kubernetes.io/max-replicas的值并通过 API 请求更新当前的 ReplicaSet 对象，并记录扩容事件。

```go
func (dc *DeploymentController) scaleReplicaSet(rs *apps.ReplicaSet, newScale int32, deployment *apps.Deployment, scalingOperation string) (bool, *apps.ReplicaSet, error) {

   sizeNeedsUpdate := *(rs.Spec.Replicas) != newScale

   annotationsNeedUpdate := deploymentutil.ReplicasAnnotationsNeedUpdate(rs, *(deployment.Spec.Replicas), *(deployment.Spec.Replicas)+deploymentutil.MaxSurge(*deployment))

   scaled := false
   var err error
   if sizeNeedsUpdate || annotationsNeedUpdate {
      rsCopy := rs.DeepCopy()
      *(rsCopy.Spec.Replicas) = newScale
      deploymentutil.SetReplicasAnnotations(rsCopy, *(deployment.Spec.Replicas), *(deployment.Spec.Replicas)+deploymentutil.MaxSurge(*deployment))
      rs, err = dc.client.AppsV1().ReplicaSets(rsCopy.Namespace).Update(context.TODO(), rsCopy, metav1.UpdateOptions{})
      if err == nil && sizeNeedsUpdate {
         scaled = true
         dc.eventRecorder.Eventf(deployment, v1.EventTypeNormal, "ScalingReplicaSet", "Scaled %s replica set %s to %d", scalingOperation, rs.Name, newScale)
      }
   }
   return scaled, rs, err
}
```



## 重建更新

当 Deployment 使用的更新策略类型是 `Recreate` 时，`DeploymentController` 就会使用如下的 `rolloutRecreate` 方法对 Deployment 进行更新： 

```go
// rolloutRecreate implements the logic for recreating a replica set.
func (dc *DeploymentController) rolloutRecreate(d *apps.Deployment, rsList []*apps.ReplicaSet, podMap map[types.UID][]*v1.Pod) error {
   // Don't create a new RS if not already existed, so that we avoid scaling up before scaling down.
   newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(d, rsList, false)
   if err != nil {
      return err
   }
   allRSs := append(oldRSs, newRS)
   activeOldRSs := controller.FilterActiveReplicaSets(oldRSs)

   // scale down old replica sets.
   scaledDown, err := dc.scaleDownOldReplicaSetsForRecreate(activeOldRSs, d)
   if err != nil {
      return err
   }
   if scaledDown {
      // Update DeploymentStatus.
      return dc.syncRolloutStatus(allRSs, newRS, d)
   }

   // Do not process a deployment when it has old pods running.
   if oldPodsRunning(newRS, oldRSs, podMap) {
      return dc.syncRolloutStatus(allRSs, newRS, d)
   }

   // If we need to create a new RS, create it now.
   if newRS == nil {
      newRS, oldRSs, err = dc.getAllReplicaSetsAndSyncRevision(d, rsList, true)
      if err != nil {
         return err
      }
      allRSs = append(oldRSs, newRS)
   }

   // scale up new replica set.
   if _, err := dc.scaleUpNewReplicaSetForRecreate(newRS, d); err != nil {
      return err
   }

   if util.DeploymentComplete(d, &d.Status) {
      if err := dc.cleanupDeployment(oldRSs, d); err != nil {
         return err
      }
   }

   // Sync deployment status.
   return dc.syncRolloutStatus(allRSs, newRS, d)
}
```

1. 利用 `getAllReplicaSetsAndSyncRevision` 和 `FilterActiveReplicaSets` 两个方法获取 Deployment 中所有的 ReplicaSet 以及其中活跃的 ReplicaSet 对象；

2. 调用 `scaleDownOldReplicaSetsForRecreate` 方法将所有活跃的历史 ReplicaSet 持有的副本数目降至 0；

3. 同步 Deployment 的最新状态并等待 Pod 的终止；

4. 在需要时通过 `getAllReplicaSetsAndSyncRevision` 方法创建新的 ReplicaSet 并调用 `scaleUpNewReplicaSetForRecreate` 函数对 ReplicaSet 进行扩容；

5. 更新完成之后会调用 `cleanupDeployment` 方法删除历史全部的 ReplicaSet 对象【保留RevisionHistoryLimit

   个历史版本的rs】并更新 Deployment 的状态；

![kubernetes-deployment-rollout-recreate](https://img.draveness.me/2019-02-24-kubernetes-deployment-rollout-recreate.png)

也就是说在更新的过程中，之前创建的 ReplicaSet 和 Pod 资源全部都会被删除，只是 Pod 会先被删除而 ReplicaSet 会后被删除；上述方法也会创建新的 ReplicaSet 和 Pod 对象，需要注意的是在这个过程中旧的 Pod 副本一定会先被删除，所以会有一段时间不存在可用的 Pod。



## 滚动更新

Deployment 的另一个更新策略 `RollingUpdate` 其实更加常见，在具体介绍滚动更新的流程之前，我们首先需要了解滚动更新策略使用的两个参数 `maxUnavailable` 和 `maxSurge`：

- `maxUnavailable` 表示在更新过程中能够进入不可用状态的 Pod 的最大值；
- `maxSurge` 表示能够额外创建的 Pod 个数；

`maxUnavailable` 和 `maxSurge` 这两个滚动更新的配置都可以使用绝对值或者百分比表示，使用百分比时需要用 `Replicas * Strategy.RollingUpdate.MaxSurge` 公式计算相应的数值。

![kubernetes-deployment-rolling-update-spe](https://img.draveness.me/2019-02-24-kubernetes-deployment-rolling-update-spec.png)

`rolloutRolling` 方法就是 `DeploymentController` 用于处理滚动更新的方法：

```go
// rolloutRolling implements the logic for rolling a new replica set.
func (dc *DeploymentController) rolloutRolling(d *apps.Deployment, rsList []*apps.ReplicaSet) error {
   newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(d, rsList, true)
   if err != nil {
      return err
   }
   allRSs := append(oldRSs, newRS)

   // Scale up, if we can.
   scaledUp, err := dc.reconcileNewReplicaSet(allRSs, newRS, d)
   if err != nil {
      return err
   }
   if scaledUp {
      // Update DeploymentStatus
      return dc.syncRolloutStatus(allRSs, newRS, d)
   }

   // Scale down, if we can.
   scaledDown, err := dc.reconcileOldReplicaSets(allRSs, controller.FilterActiveReplicaSets(oldRSs), newRS, d)
   if err != nil {
      return err
   }
   if scaledDown {
      // Update DeploymentStatus
      return dc.syncRolloutStatus(allRSs, newRS, d)
   }

   if deploymentutil.DeploymentComplete(d, &d.Status) {
      if err := dc.cleanupDeployment(oldRSs, d); err != nil {
         return err
      }
   }

   // Sync deployment status
   return dc.syncRolloutStatus(allRSs, newRS, d)
}
```

1. 首先获取 Deployment 对应的全部 ReplicaSet 资源；
2. 通过 `reconcileNewReplicaSet` 调解新 ReplicaSet 的副本数，创建新的 Pod 并保证额外的副本数量不超过 `maxSurge`；
3. 通过 `reconcileOldReplicaSets` 调解历史 ReplicaSet 的副本数，删除旧的 Pod 并保证不可用的部分数不会超过 `maxUnavailable`；
4. 最后删除无用的 ReplicaSet 并更新 Deployment 的状态；

需要注意的是，在滚动更新的过程中，Kubernetes 并不是一次性就切换到期望的状态，即『新 ReplicaSet 运行指定数量的副本』，而是会先启动新的 ReplicaSet 以及一定数量的 Pod 副本，然后删除历史 ReplicaSet 中的副本，再启动一些新 ReplicaSet 的副本，不断对新 ReplicaSet 进行扩容并对旧 ReplicaSet 进行缩容最终达到了集群期望的状态。

1 当我们使用如下的 `reconcileNewReplicaSet` 方法对新 ReplicaSet 进行调节时，我们会发现在新 ReplicaSet 中副本数量满足期望时会直接返回，在超过期望时会进行缩容，如果 ReplicaSet 的数量不够就会调用 `NewRSNewReplicas` 函数计算新的副本个数。该过程总共需要考虑 Deployment 期望的副本数量、当前可用的副本数量以及新 ReplicaSet 持有的副本，还有一些最大值和最小值的限制，例如额外 Pod 数量不能超过 `maxSurge`、新 ReplicaSet 的 Pod 数量不能超过 Deployment 的期望数量，遵循这些规则我们就能计算出 `newRSNewReplicas`。

```go
func (dc *DeploymentController) reconcileNewReplicaSet(allRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet, deployment *apps.Deployment) (bool, error) {
   if *(newRS.Spec.Replicas) == *(deployment.Spec.Replicas) {
      // Scaling not required.
      return false, nil
   }
   if *(newRS.Spec.Replicas) > *(deployment.Spec.Replicas) {
      // Scale down.
      scaled, _, err := dc.scaleReplicaSetAndRecordEvent(newRS, *(deployment.Spec.Replicas), deployment)
      return scaled, err
   }
    //计算新的副本个数
   newReplicasCount, err := deploymentutil.NewRSNewReplicas(deployment, allRSs, newRS)
   if err != nil {
      return false, err
   }
   scaled, _, err := dc.scaleReplicaSetAndRecordEvent(newRS, newReplicasCount, deployment)
   return scaled, err
}
```

2 另一个滚动更新中使用的方法 `reconcileOldReplicaSets` 主要作用就是对历史 ReplicaSet 对象持有的副本数量进行缩容： 

1. 计算历史 ReplicaSet 持有的副本总数量；
2. 计算全部 ReplicaSet 持有的副本总数量；
3. 根据 Deployment 期望的副本数、最大不可用副本数以及新 ReplicaSet 中不可用的 Pod 数量计算最大缩容的副本个数；
4. 通过 `cleanupUnhealthyReplicas` 方法清理 ReplicaSet 中处于不健康状态的副本；
5. 调用 `scaleDownOldReplicaSetsForRollingUpdate` 方法对历史 ReplicaSet 中的副本进行缩容；

该方法会使用上述简化后的公式计算这次总共能够在历史 ReplicaSet 中删除的最大 Pod 数量，并调用 `cleanupUnhealthyReplicas` 和 `scaleDownOldReplicaSetsForRollingUpdate` 两个方法进行缩容，这两个方法的实现都相对简单，它们都对历史 ReplicaSet 按照创建时间进行排序依次对这些资源进行缩容，两者的区别在于前者主要用于删除不健康的副本。

```go
func (dc *DeploymentController) reconcileOldReplicaSets(allRSs []*apps.ReplicaSet, oldRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet, deployment *apps.Deployment) (bool, error) {
   oldPodsCount := deploymentutil.GetReplicaCountForReplicaSets(oldRSs)
   if oldPodsCount == 0 {
      // Can't scale down further
      return false, nil
   }

   allPodsCount := deploymentutil.GetReplicaCountForReplicaSets(allRSs)
   klog.V(4).Infof("New replica set %s/%s has %d available pods.", newRS.Namespace, newRS.Name, newRS.Status.AvailableReplicas)
   maxUnavailable := deploymentutil.MaxUnavailable(*deployment)

   minAvailable := *(deployment.Spec.Replicas) - maxUnavailable
   newRSUnavailablePodCount := *(newRS.Spec.Replicas) - newRS.Status.AvailableReplicas
   maxScaledDown := allPodsCount - minAvailable - newRSUnavailablePodCount
   if maxScaledDown <= 0 {
      return false, nil
   }

   // Clean up unhealthy replicas first, otherwise unhealthy replicas will block deployment
   // and cause timeout. See https://github.com/kubernetes/kubernetes/issues/16737
   oldRSs, cleanupCount, err := dc.cleanupUnhealthyReplicas(oldRSs, deployment, maxScaledDown)
   if err != nil {
      return false, nil
   }
   klog.V(4).Infof("Cleaned up unhealthy replicas from old RSes by %d", cleanupCount)

   // Scale down old replica sets, need check maxUnavailable to ensure we can scale down
   allRSs = append(oldRSs, newRS)
   scaledDownCount, err := dc.scaleDownOldReplicaSetsForRollingUpdate(allRSs, oldRSs, deployment)
   if err != nil {
      return false, nil
   }
   klog.V(4).Infof("Scaled down old RSes of deployment %s by %d", deployment.Name, scaledDownCount)

   totalScaledDown := cleanupCount + scaledDownCount
   return totalScaledDown > 0, nil
}
```

## 回退

Kubernetes 中的每一个 Deployment 资源都包含有 `revision` 这个概念，版本的引入可以让我们在更新发生问题时及时通过 Deployment 的版本对其进行回滚，当我们在更新 Deployment 时，之前 Deployment 持有的 ReplicaSet 其实会被 `cleanupDeployment` 方法清理：

```go
func (dc *DeploymentController) cleanupDeployment(oldRSs []*apps.ReplicaSet, deployment *apps.Deployment) error {
	aliveFilter := func(rs *apps.ReplicaSet) bool {
		return rs != nil && rs.ObjectMeta.DeletionTimestamp == nil
	}
	cleanableRSes := controller.FilterReplicaSets(oldRSs, aliveFilter)

	diff := int32(len(cleanableRSes)) - *deployment.Spec.RevisionHistoryLimit
	if diff <= 0 {
		return nil
	}
	sort.Sort(controller.ReplicaSetsByCreationTimestamp(cleanableRSes))

	for i := int32(0); i < diff; i++ {
		rs := cleanableRSes[i]
		if rs.Status.Replicas != 0 || *(rs.Spec.Replicas) != 0 || rs.Generation > rs.Status.ObservedGeneration || rs.DeletionTimestamp != nil {
			continue
		}
		dc.client.AppsV1().ReplicaSets(rs.Namespace).Delete(rs.Name, nil)
	}

	return nil
}
```

Deployment 资源在规格中由一个 `spec.revisionHistoryLimit` 的配置，这个配置决定了 Kubernetes 会保存多少个 ReplicaSet 的历史版本，这些历史上的 ReplicaSet 并不会被删除，它们只是不再持有任何的 Pod 副本了，假设我们有一个 `spec.revisionHistoryLimit=2` 的 Deployment 对象，那么当前资源最多持有两个历史的 ReplicaSet 版本：

![kubernetes-deployment-revision](https://img.draveness.me/2019-02-24-kubernetes-deployment-revision.png)

这些资源的保留能够方便 Deployment 的回滚，而回滚其实是通过 kubectl 在客户端实现的，我们可以使用如下的命令将 Deployment 回滚到上一个版本：

```bash
$ kubectl rollout undo deployment.v1.apps/nginx-deployment
deployment.apps/nginx-deployment
```

上述 kubectl 命令没有指定回滚到的版本号，所以在默认情况下会回滚到上一个版本，在回滚时会直接根据传入的版本查找历史的 ReplicaSet 资源，拿到这个 ReplicaSet 对应的 Pod 模板后会触发一个资源更新的请求：

回滚对于 Kubernetes 服务端来说其实与其他的更新操作没有太多的区别，在每次更新时都会在 `FindNewReplicaSet` 函数中根据 Deployment 的 Pod 模板在历史 ReplicaSet 中查询是否有相同的 ReplicaSet 存在：

如果存在规格完全相同的 ReplicaSet，就会保留这个 ReplicaSet 历史上使用的版本号并对该 ReplicaSet 重新进行扩容并对正在工作的 ReplicaSet 进行缩容以实现集群的期望状态。

```go
// rollback the deployment to the specified revision. In any case cleanup the rollback spec.
func (dc *DeploymentController) rollback(d *apps.Deployment, rsList []*apps.ReplicaSet) error {
	newRS, allOldRSs, err := dc.getAllReplicaSetsAndSyncRevision(d, rsList, true)
	if err != nil {
		return err
	}

	allRSs := append(allOldRSs, newRS)
	rollbackTo := getRollbackTo(d)
	// If rollback revision is 0, rollback to the last revision
	if rollbackTo.Revision == 0 {
		if rollbackTo.Revision = deploymentutil.LastRevision(allRSs); rollbackTo.Revision == 0 {
			// If we still can't find the last revision, gives up rollback
			dc.emitRollbackWarningEvent(d, deploymentutil.RollbackRevisionNotFound, "Unable to find last revision.")
			// Gives up rollback
			return dc.updateDeploymentAndClearRollbackTo(d)
		}
	}
	for _, rs := range allRSs {
		v, err := deploymentutil.Revision(rs)
		if err != nil {
			klog.V(4).Infof("Unable to extract revision from deployment's replica set %q: %v", rs.Name, err)
			continue
		}
		if v == rollbackTo.Revision {
			klog.V(4).Infof("Found replica set %q with desired revision %d", rs.Name, v)
			// rollback by copying podTemplate.Spec from the replica set
			// revision number will be incremented during the next getAllReplicaSetsAndSyncRevision call
			// no-op if the spec matches current deployment's podTemplate.Spec
			performedRollback, err := dc.rollbackToTemplate(d, rs)
			if performedRollback && err == nil {
				dc.emitRollbackNormalEvent(d, fmt.Sprintf("Rolled back deployment %q to revision %d", d.Name, rollbackTo.Revision))
			}
			return err
		}
	}
	dc.emitRollbackWarningEvent(d, deploymentutil.RollbackRevisionNotFound, "Unable to find the revision to rollback to.")
	// Gives up rollback
	return dc.updateDeploymentAndClearRollbackTo(d)
}
```



## 其他

**暂停恢复**

Deployment 中有一个不是特别常用的功能，也就是 Deployment 进行暂停，暂停之后的 Deployment 哪怕发生了改动也不会被 Kubernetes 更新，这时我们可以对 Deployment 资源进行更新或者修复，随后当重新恢复 Deployment 时，`DeploymentController` 才会重新对其进行滚动更新向期望状态迁移。我们只需要在文件里对资源进行修改并进行一次更新就可以了，但是我们可以在出现问题时，暂停一次正在进行的滚动更新以防止错误的扩散。 

**删除**

如果我们在 Kubernetes 集群中删除了一个 Deployment 资源，那么 Deployment 持有的 ReplicaSet 以及 ReplicaSet 持有的副本都会被 Kubernetes 中的 [垃圾收集器](https://draveness.me/kubernetes-garbage-collector) 删除：

```bash
$ kubectl delete deployments.apps nginx-deployment
deployment.apps "nginx-deployment" deleted

$ kubectl get replicasets --watch
nginx-deployment-7c6cf994f6   0     0     0     2d1h
nginx-deployment-5cc74f885d   0     0     0     2d1h
nginx-deployment-c5d875444   3     3     3     30h

$ kubectl get pods --watch
nginx-deployment-c5d875444-6r4q6   1/1   Terminating   2     30h
nginx-deployment-c5d875444-7ssgj   1/1   Terminating   2     30h
nginx-deployment-c5d875444-4xvvz   1/1   Terminating   2     30h
```

由于与当前 Deployment 有关的 ReplicaSet 历史和最新版本都会被删除，所以对应的 Pod 副本也都会随之被删除，这些对象之间的关系都是通过 `metadata.ownerReference` 这一字段关联的，



# 垃圾收集器

> 引用 https://draveness.me/kubernetes-garbage-collector/
>
> https://cloud.tencent.com/developer/article/1562130

**删除策略**

Kubernetes 中的垃圾收集器会负责删除**以前有所有者但是现在没有**的对象，`metadata.ownerReference` 属性标识了一个对象的所有者，当垃圾收集器发现对象的所有者被删除时，就会自动删除这些无用的对象，这也是 ReplicaSet 持有的 Pod 被自动删除的原因，。

kubernetes 中有三种删除策略：`Orphan`、`Foreground` 和 `Background`，三种删除策略的意义分别为：

- `Orphan` 策略：非级联删除，删除对象时，不会自动删除**它的依赖或者是子对象**，这些依赖被称作是原对象的孤儿对象；

- `Background` 策略：在该模式下，kubernetes 会立即删除该对象，然后垃圾收集器会在后台删除这些该对象的依赖对象；  
- `Foreground` 策略：在该模式下，对象首先进入“删除中”状态，即会设置对象的 `deletionTimestamp` 字段并且对象的 `metadata.finalizers` 字段包含了值 “foregroundDeletion”，此时该对象依然存在，然后垃圾收集器会删除该对象的所有依赖对象，垃圾收集器在删除了所有“Blocking” 状态的依赖对象（指其子对象中 `ownerReference.blockOwnerDeletion=true`的对象）之后，然后才会删除对象本身。



**finalizer 机制**

**finalizer 是在删除对象时设置的一个 hook，其目的是为了让对象在删除前确认其子对象已经被完全删除，k8s 中默认有两种 finalizer：`OrphanFinalizer` 和 `ForegroundFinalizer`**，finalizer 存在于对象的 ObjectMeta 中，当一个对象的依赖对象被删除后其对应的 finalizers 字段也会被移除，只有 finalizers 字段为空时，apiserver 才会删除该对象。此外，finalizer 不仅仅支持以上两种字段，在使用自定义 controller 时也可以在 CR 中设置自定义的 finalizer 标识。

```
{
	......
	"metadata": {
	   ......
		"finalizers": [
			"foregroundDeletion"
		]
	}
	......
}
```



## 数据结构

`GarbageCollector` 中包含一个 `GraphBuilder` 结构体，这个结构体会以 Goroutine 的形式运行并使用 Informer 监听集群中几乎全部资源的变动list+watch，一旦发现任何的变更事件 — 增删改，就会将事件交给主循环处理，主循环会根据事件的不同选择将待处理对象加入不同的队列，与此同时 `GarbageCollector` 持有的另外两组队列会负责删除或者孤立目标对象。 

```go
type GarbageCollector struct {
   restMapper     resettableRESTMapper
   metadataClient metadata.Interface
   // garbage collector attempts to delete the items in attemptToDelete queue when the time is ripe.
   attemptToDelete workqueue.RateLimitingInterface
   // garbage collector attempts to orphan the dependents of the items in the attemptToOrphan queue, then deletes the items.
   attemptToOrphan        workqueue.RateLimitingInterface
   dependencyGraphBuilder *GraphBuilder
   // GC caches the owners that do not exist according to the API server.
   absentOwnerCache *UIDCache

   workerLock sync.RWMutex
}

// GraphBuilder processes events supplied by the informers, updates uidToNode,
// a graph that caches the dependencies as we know, and enqueues
// items to the attemptToDelete and attemptToOrphan.
type GraphBuilder struct {
    restMapper meta.RESTMapper

    // informers
    monitors    monitors
    monitorLock sync.RWMutex

    // 当 kube-controller-manager 中所有的 controllers 都启动后，informersStarted 会被 close 掉
    // informersStarted 会被 close 掉的调用程序在 kube-controller-manager 的启动流程中
    informersStarted <-chan struct{}

    stopCh <-chan struct{}

    // 当调用 GraphBuilder 的 run 方法时，running 会被设置为 true
    running bool

    metadataClient metadata.Interface
    
    // informers 监听到所有资源的事件会放在 graphChanges 中
    graphChanges workqueue.RateLimitingInterface
    
    // 维护所有资源对象的依赖关系 内存中 dag?
    uidToNode *concurrentUIDToNode
    
    // GarbageCollector 作为消费者要处理 attemptToDelete 和 attemptToOrphan 两个队列中的事件
    attemptToDelete workqueue.RateLimitingInterface
    attemptToOrphan workqueue.RateLimitingInterface

    absentOwnerCache *UIDCache
    sharedInformers  controller.InformerFactory
    // 不需要被 gc 的资源
    ignoredResources map[schema.GroupResource]struct{}
}
```

**GraphBuilder** 

GraphBuilder 主要有三个功能：

- 1、监控集群中所有的可删除资源；
- 2、基于 informers 中的资源在 uidToNode 数据结构中维护着所有对象的依赖关系；
- 3、处理 graphChanges 中的事件并放到 attemptToDelete 和 attemptToOrphan 两个队列中；



**uidToNode**

此处有必要先说明一下 uidToNode 的功能，uidToNode 数据结构中**维护着所有对象的依赖关系**，此处的依赖关系是指比如当创建一个 deployment 时会创建对应的 rs 以及 pod，**pod 的 owner 就是 rs，rs 的 owner 是 deployment，rs 的 dependents 是其关联的所有 pod，deployment 的 dependents 是其关联的所有 rs**。

uidToNode 中的 node 不是指 k8s 中的 node 节点，而是将 graphChanges 中的 event 转换为 node 对象，k8s 中所有 object 之间的级联关系是通过 node 的概念来维护的，garbageCollector 在后续的处理中会直接使用 node 对象，node 对象定义如下：

```go
type concurrentUIDToNode struct {
   uidToNodeLock sync.RWMutex
   uidToNode     map[types.UID]*node
}

// The single-threaded GraphBuilder.processGraphChanges() is the sole writer of the
// nodes. The multi-threaded GarbageCollector.attemptToDeleteItem() reads the nodes.
// WARNING: node has different locks on different fields. setters and getters
// use the respective locks, so the return values of the getters can be
// inconsistent.
type node struct {
	identity objectReference
	// dependents will be read by the orphan() routine, we need to protect it with a lock.
	dependentsLock sync.RWMutex
	// dependents are the nodes that have node.identity as a
	// metadata.ownerReference.
	dependents map[*node]struct{}
	// this is set by processGraphChanges() if the object has non-nil DeletionTimestamp
	// and has the FinalizerDeleteDependents.
	deletingDependents     bool
	deletingDependentsLock sync.RWMutex
	// this records if the object's deletionTimestamp is non-nil.
	beingDeleted     bool
	beingDeletedLock sync.RWMutex
	// this records if the object was constructed virtually and never observed via informer event
    // 当 virtual 值为 true 时，此时不确定该对象是否存在于 apiserver 中
	virtual     bool
	virtualLock sync.RWMutex
	// when processing an Update event, we need to compare the updated
	// ownerReferences with the owners recorded in the graph.
	owners []metav1.OwnerReference
}
```

## 启动

`startGarbageCollectorController`，其主要逻辑为：

- 1、初始化 discoveryClient，discoveryClient 主要用来获取集群中的所有资源； 
- 2、调用 `garbagecollector.GetDeletableResources` 获取集群内所有可删除的资源对象，支持 “delete”, “list”, “watch” 三种操作的 resource 称为 `deletableResource`；
- 3、调用 `garbagecollector.NewGarbageCollector` 初始化 garbageCollector 对象；
- 4、调用 `garbageCollector.Run` 启动 garbageCollector；
- 5、调用 `garbageCollector.Sync` 监听集群中的 `DeletableResources` ，当出现新的  `DeletableResources` 时同步到 monitors 中，确保监控集群中的所有资源；
- 6、调用 `garbagecollector.NewDebugHandler` 注册 debug 接口，用来提供集群内所有对象的关联关系；

```go
func startGarbageCollectorController(ctx ControllerContext) (http.Handler, bool, error) {
   if !ctx.ComponentConfig.GarbageCollectorController.EnableGarbageCollector {
      return nil, false, nil
   }

   gcClientset := ctx.ClientBuilder.ClientOrDie("generic-garbage-collector")

   config := ctx.ClientBuilder.ConfigOrDie("generic-garbage-collector")
   metadataClient, err := metadata.NewForConfig(config)
   if err != nil {
      return nil, true, err
   }

   ignoredResources := make(map[schema.GroupResource]struct{})
   for _, r := range ctx.ComponentConfig.GarbageCollectorController.GCIgnoredResources {
      ignoredResources[schema.GroupResource{Group: r.Group, Resource: r.Resource}] = struct{}{}
   }

   garbageCollector, err := garbagecollector.NewGarbageCollector(
      metadataClient,
      ctx.RESTMapper,
      ignoredResources,
      ctx.ObjectOrMetadataInformerFactory,
      ctx.InformersStarted,
   )
   if err != nil {
      return nil, true, fmt.Errorf("failed to start the generic garbage collector: %v", err)
   }

   // Start the garbage collector.
     // 启动 garbage collector
   workers := int(ctx.ComponentConfig.GarbageCollectorController.ConcurrentGCSyncs)
   go garbageCollector.Run(workers, ctx.Stop)

   // Periodically refresh the RESTMapper with new discovery information and sync
   // the garbage collector.
    // 监听集群中的 DeletableResources
   go garbageCollector.Sync(gcClientset.Discovery(), 30*time.Second, ctx.Stop)
// 注册 debug 接口
   return garbagecollector.NewDebugHandler(garbageCollector), true, nil
}
```



## garbageCollector.Sync

`garbageCollector.Sync` 是 `startGarbageCollectorController` 中的第三个核心方法，主要功能是周期性的查询集群中所有的资源，过滤出 `deletableResources`，然后对比已经监控的 `deletableResources` 和当前获取到的 `deletableResources` 是否一致，若不一致则更新 GraphBuilder 的 monitors 并重新启动 monitors 监控所有的  `deletableResources`，该方法的主要逻辑为：

- 1、通过调用 `GetDeletableResources` 获取集群内所有的 `deletableResources` 作为 newResources，`deletableResources` 指支持 “delete”, “list”, “watch” 三种操作的 resource，包括 CR；
- 2、检查 oldResources, newResources 是否一致，不一致则需要同步；
- 3、调用 `gc.resyncMonitors` 同步 newResources，在 `gc.resyncMonitors` 中会重新调用 GraphBuilder 的 `syncMonitors` 和 `startMonitors` 两个方法完成 monitors 的刷新；
- 4、等待 newResources informer 中的 cache 同步完成；
- 5、将 newResources 作为 oldResources，继续进行下一轮的同步； 

```go
func (gc *GarbageCollector) Sync(discoveryClient discovery.ServerResourcesInterface, period time.Duration, stopCh <-chan struct{}) {
	oldResources := make(map[schema.GroupVersionResource]struct{})
	wait.Until(func() {
		// Get the current resource list from discovery.
		newResources := GetDeletableResources(discoveryClient)

		// This can occur if there is an internal error in GetDeletableResources.
		if len(newResources) == 0 {
			klog.V(2).Infof("no resources reported by discovery, skipping garbage collector sync")
			return
		}

		// Decide whether discovery has reported a change.
		if reflect.DeepEqual(oldResources, newResources) {
			klog.V(5).Infof("no resource updates from discovery, skipping garbage collector sync")
			return
		}

		// Ensure workers are paused to avoid processing events before informers
		// have resynced.
		gc.workerLock.Lock()
		defer gc.workerLock.Unlock()

		// Once we get here, we should not unpause workers until we've successfully synced
		attempt := 0
		wait.PollImmediateUntil(100*time.Millisecond, func() (bool, error) {
			attempt++

			// On a reattempt, check if available resources have changed
			if attempt > 1 {
				newResources = GetDeletableResources(discoveryClient)
				if len(newResources) == 0 {
					klog.V(2).Infof("no resources reported by discovery (attempt %d)", attempt)
					return false, nil
				}
			}

			klog.V(2).Infof("syncing garbage collector with updated resources from discovery (attempt %d): %s", attempt, printDiff(oldResources, newResources))

			// Resetting the REST mapper will also invalidate the underlying discovery
			// client. This is a leaky abstraction and assumes behavior about the REST
			// mapper, but we'll deal with it for now.
			gc.restMapper.Reset()
			klog.V(4).Infof("reset restmapper")

			// Perform the monitor resync and wait for controllers to report cache sync.
			//
			// NOTE: It's possible that newResources will diverge from the resources
			// discovered by restMapper during the call to Reset, since they are
			// distinct discovery clients invalidated at different times. For example,
			// newResources may contain resources not returned in the restMapper's
			// discovery call if the resources appeared in-between the calls. In that
			// case, the restMapper will fail to map some of newResources until the next
			// attempt.
			if err := gc.resyncMonitors(newResources); err != nil {
				utilruntime.HandleError(fmt.Errorf("failed to sync resource monitors (attempt %d): %v", attempt, err))
				return false, nil
			}
			klog.V(4).Infof("resynced monitors")

			// wait for caches to fill for a while (our sync period) before attempting to rediscover resources and retry syncing.
			// this protects us from deadlocks where available resources changed and one of our informer caches will never fill.
			// informers keep attempting to sync in the background, so retrying doesn't interrupt them.
			// the call to resyncMonitors on the reattempt will no-op for resources that still exist.
			// note that workers stay paused until we successfully resync.
			if !cache.WaitForNamedCacheSync("garbage collector", waitForStopOrTimeout(stopCh, period), gc.dependencyGraphBuilder.IsSynced) {
				utilruntime.HandleError(fmt.Errorf("timed out waiting for dependency graph builder sync during GC sync (attempt %d)", attempt))
				return false, nil
			}

			// success, break out of the loop
			return true, nil
		}, stopCh)

		// Finally, keep track of our new state. Do this after all preceding steps
		// have succeeded to ensure we'll retry on subsequent syncs if an error
		// occurred.
		oldResources = newResources
		klog.V(2).Infof("synced garbage collector")
	}, period, stopCh)
}

```



**gb.syncMonitors**

`syncMonitors` 的主要作用是初始化各个资源对象的 informer，并调用 `gb.controllerFor` 为每种资源注册 eventHandler，此处每种资源被称为 monitors，因为为每种资源注册 eventHandler 时，对于 AddFunc、UpdateFunc 和 DeleteFunc 都会将对应的 event push 到 graphChanges 队列中，每种资源对象的 informer 都作为生产者。

`gb.syncMonitors` 中初始化了所有 deletableResources 的 informer，为每个 informer 添加 eventHandler 并将监听到的所有 event push 到 graphChanges 队列中，此处每个 informer 都被称为 monitor，所有 informer 都被称为生产者。graphChanges 是 GraphBuilder 中的一个对象，GraphBuilder 的主要功能是作为一个生产者，**其会处理 graphChanges 中的所有事件并进行分类，将事件放入到 attemptToDelete 和 attemptToOrphan 两个队列中** 

```go
// syncMonitors rebuilds the monitor set according to the supplied resources,
// creating or deleting monitors as necessary. It will return any error
// encountered, but will make an attempt to create a monitor for each resource
// instead of immediately exiting on an error. It may be called before or after
// Run. Monitors are NOT started as part of the sync. To ensure all existing
// monitors are started, call startMonitors.
func (gb *GraphBuilder) syncMonitors(resources map[schema.GroupVersionResource]struct{}) error {
   gb.monitorLock.Lock()
   defer gb.monitorLock.Unlock()

   toRemove := gb.monitors
   if toRemove == nil {
      toRemove = monitors{}
   }
   current := monitors{}
   errs := []error{}
   kept := 0
   added := 0
   for resource := range resources {
      if _, ok := gb.ignoredResources[resource.GroupResource()]; ok {
         continue
      }
      if m, ok := toRemove[resource]; ok {
         current[resource] = m
         delete(toRemove, resource)
         kept++
         continue
      }
      kind, err := gb.restMapper.KindFor(resource)
      if err != nil {
         errs = append(errs, fmt.Errorf("couldn't look up resource %q: %v", resource, err))
         continue
      }
       // 为 每种resource 的 controller 注册 eventHandler 
      c, s, err := gb.controllerFor(resource, kind)
      if err != nil {
         errs = append(errs, fmt.Errorf("couldn't start monitor for resource %q: %v", resource, err))
         continue
      }
      current[resource] = &monitor{store: s, controller: c}
      added++
   }
   gb.monitors = current

   for _, monitor := range toRemove {
      if monitor.stopCh != nil {
         close(monitor.stopCh)
      }
   }

   klog.V(4).Infof("synced monitors; added %d, kept %d, removed %d", added, kept, len(toRemove))
   // NewAggregate returns nil if errs is 0-length
   return utilerrors.NewAggregate(errs)
}

// 为每个 deletableResources 的 informer 注册 eventHandler，此处就可以看到真正的生产者了。
func (gb *GraphBuilder) controllerFor(resource schema.GroupVersionResource, kind schema.GroupVersionKind) (cache.Controller, cache.Store, error) {
	handlers := cache.ResourceEventHandlerFuncs{
		// add the event to the dependencyGraphBuilder's graphChanges.
		AddFunc: func(obj interface{}) {
			event := &event{
				eventType: addEvent,
				obj:       obj,
				gvk:       kind,
			}
			gb.graphChanges.Add(event)
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			// TODO: check if there are differences in the ownerRefs,
			// finalizers, and DeletionTimestamp; if not, ignore the update.
			event := &event{
				eventType: updateEvent,
				obj:       newObj,
				oldObj:    oldObj,
				gvk:       kind,
			}
			gb.graphChanges.Add(event)
		},
		DeleteFunc: func(obj interface{}) {
			// delta fifo may wrap the object in a cache.DeletedFinalStateUnknown, unwrap it
			if deletedFinalStateUnknown, ok := obj.(cache.DeletedFinalStateUnknown); ok {
				obj = deletedFinalStateUnknown.Obj
			}
			event := &event{
				eventType: deleteEvent,
				obj:       obj,
				gvk:       kind,
			}
			gb.graphChanges.Add(event)
		},
	}
	shared, err := gb.sharedInformers.ForResource(resource)
	if err != nil {
		klog.V(4).Infof("unable to use a shared informer for resource %q, kind %q: %v", resource.String(), kind.String(), err)
		return nil, nil, err
	}
	klog.V(4).Infof("using a shared informer for resource %q, kind %q", resource.String(), kind.String())
	// need to clone because it's from a shared cache
	shared.Informer().AddEventHandlerWithResyncPeriod(handlers, ResourceResyncTime)
	return shared.Informer().GetController(), shared.Informer().GetStore(), nil
}
```







## garbageCollector.Run

`garbageCollector.Run` 调用 `gc.dependencyGraphBuilder.Run` 启动一个 goroutine 处理  graphChanges 队列中的事件并分别放到 attemptToDelete 和 attemptToOrphan 两个队列中 ；

run` 方法会调用 `gc.runAttemptToDeleteWorker` 和 `gc.runAttemptToOrphanWorker` 启动多个 goroutine 处理 attemptToDelete 和 attemptToOrphan 两个队列中的事件。 

```go
// Run starts garbage collector workers.
func (gc *GarbageCollector) Run(workers int, stopCh <-chan struct{}) {
   defer utilruntime.HandleCrash()
   defer gc.attemptToDelete.ShutDown()
   defer gc.attemptToOrphan.ShutDown()
   defer gc.dependencyGraphBuilder.graphChanges.ShutDown()

   klog.Infof("Starting garbage collector controller")
   defer klog.Infof("Shutting down garbage collector controller")
// 启动一个 goroutine 处理 graphChanges 中的事件将其分别放到 GraphBuilder 的 attemptToDelete 和 attemptToOrphan 两个 队列中
   go gc.dependencyGraphBuilder.Run(stopCh)
// 2、等待 informers 的 cache 同步完成
   if !cache.WaitForNamedCacheSync("garbage collector", stopCh, gc.dependencyGraphBuilder.IsSynced) {
      return
   }

   klog.Infof("Garbage collector: all resource monitors have synced. Proceeding to collect garbage")

   // gc workers
   for i := 0; i < workers; i++ {
       // 3、启动多个 goroutine 调用 gc.runAttemptToDeleteWorker 处理 attemptToDelete 中的事件
      go wait.Until(gc.runAttemptToDeleteWorker, 1*time.Second, stopCh)
      // 4、启动多个 goroutine 调用 gc.runAttemptToOrphanWorker 处理 attemptToDelete 中的事件
       go wait.Until(gc.runAttemptToOrphanWorker, 1*time.Second, stopCh)
   }

   <-stopCh
}
```

**gc.dependencyGraphBuilder.Run**

`gc.dependencyGraphBuilder.Run`的核心是调用了 `gb.startMonitors` 和 `gb.runProcessGraphChanges` 两个方法来完成主要功能，继续看这两个方法的主要逻辑

`startMonitors` 的功能很简单就是启动所有的 informers  

```go
// Run sets the stop channel and starts monitor execution until stopCh is
// closed. Any running monitors will be stopped before Run returns.
func (gb *GraphBuilder) Run(stopCh <-chan struct{}) {
   klog.Infof("GraphBuilder running")
   defer klog.Infof("GraphBuilder stopping")

   // Set up the stop channel.
   gb.monitorLock.Lock()
   gb.stopCh = stopCh
   gb.running = true
   gb.monitorLock.Unlock()

   // Start monitors and begin change processing until the stop channel is
   // closed.
   gb.startMonitors()
   wait.Until(gb.runProcessGraphChanges, 1*time.Second, stopCh)

   // Stop any running monitors.
   gb.monitorLock.Lock()
   defer gb.monitorLock.Unlock()
   monitors := gb.monitors
   stopped := 0
   for _, monitor := range monitors {
      if monitor.stopCh != nil {
         stopped++
         close(monitor.stopCh)
      }
   }

   // reset monitors so that the graph builder can be safely re-run/synced.
   gb.monitors = nil
   klog.Infof("stopped %d of %d monitors", stopped, len(monitors))
}
```

**gb.runProcessGraphChanges**

`runProcessGraphChanges` 方法的主要功能是处理 graphChanges 中的事件将其分别放到 GraphBuilder 的 attemptToDelete 和 attemptToOrphan 两个队列中，代码主要逻辑为：

- 1、从 graphChanges 队列中取出一个 item 即 event；
- 2、获取 event 的 accessor，accessor 是一个 object 的 meta.Interface，里面包含访问 object meta 中所有字段的方法；
- 3、通过 accessor 获取 UID 判断 uidToNode 中是否存在该 object；
- 4、若 uidToNode 中不存在该 node 且该事件是 addEvent 或 updateEvent，则为该 object 创建对应的 node，并调用 `gb.insertNode` 将该 node 加到 uidToNode 中，然后将该 node 添加到其 owner 的 dependents 中，执行完 `gb.insertNode` 中的操作后再调用 `gb.processTransitions` 方法判断该对象是否处于删除状态，若处于删除状态会判断该对象是以 `orphan` 模式删除还是以 `foreground` 模式删除，若以 `orphan` 模式删除，则将该 node 加入到 attemptToOrphan 队列中，若以 `foreground` 模式删除则将该对象以及其所有 dependents 都加入到 attemptToDelete 队列中；
- 5、若 uidToNode 中存在该 node 且该事件是 addEvent 或 updateEvent 时，此时可能是一个 update 操作，调用 `referencesDiffs` 方法检查该对象的 `OwnerReferences` 字段是否有变化，若有变化(1)调用 `gb.addUnblockedOwnersToDeleteQueue` 将被删除以及更新的 owner 对应的 node 加入到 attemptToDelete 中，因为此时该 node 中已被删除或更新的 owner 可能处于删除状态且阻塞在该 node 处，此时有三种方式避免该 node 的 owner 处于删除阻塞状态，一是等待该 node 被删除，二是将该 node 自身对应 owner 的 `OwnerReferences` 字段删除，三是将该 node `OwnerReferences` 字段中对应 owner 的 `BlockOwnerDeletion` 设置为 false；(2)更新该 node 的 owners 列表；(3)若有新增的 owner，将该 node 加入到新 owner 的 dependents 中；(4) 若有被删除的 owner，将该 node 从已删除 owner 的 dependents 中删除；以上操作完成后，检查该 node 是否处于删除状态并进行标记，最后调用 `gb.processTransitions` 方法检查该 node 是否要被删除；  举个例子，若以 `foreground` 模式删除 deployment 时，deployment 的 dependents 列表中有对应的 rs，那么 deployment 的删除会阻塞住等待其依赖 rs 的删除，此时 rs 有三种方法不阻塞 deployment 的删除操作，一是 rs 对象被删除，二是删除 rs 对象 `OwnerReferences` 字段中对应的 deployment，三是将 rs 对象`OwnerReferences` 字段中对应的 deployment 配置 `BlockOwnerDeletion` 设置为 false。  
- 6、若该事件为 deleteEvent，首先从 uidToNode 中删除该对象，然后从该 node 所有 owners 的 dependents 中删除该对象，将该 node 所有的 dependents 加入到 attemptToDelete 队列中，最后检查该 node 的所有 owners，若有处于删除状态的 owner，此时该 owner 可能处于删除阻塞状态正在等待该 node 的删除，将该 owner 加入到 attemptToDelete 中；  

总结一下，当从 graphChanges 中取出 event 时，不管是什么 event，主要完成三件事，首先都会将 event 转化为 uidToNode 中的 node 对象，其次一是更新 uidToNode 中维护的依赖关系，二是更新该 node 的 owners 以及 owners 的 dependents，三是检查该 node 的 owners 是否要被删除以及该 node 的 dependents 是否要被删除，**若需要删除则根据 node 的删除策略将其添加到 attemptToOrphan 或者 attemptToDelete 队列**中

`k8s.io/kubernetes/pkg/controller/garbagecollector/graph_builder.go:526`

```go
func (gb *GraphBuilder) runProcessGraphChanges() {
    for gb.processGraphChanges() {
    }
}

func (gb *GraphBuilder) processGraphChanges() bool {
    // 1、从 graphChanges 取出一个 event
    item, quit := gb.graphChanges.Get()
    if quit {
        return false
    }
    defer gb.graphChanges.Done(item)
    event, ok := item.(*event)
    if !ok {
        utilruntime.HandleError(fmt.Errorf("expect a *event, got %v", item))
        return true
    }
    obj := event.obj
    accessor, err := meta.Accessor(obj)
    if err != nil {
        utilruntime.HandleError(fmt.Errorf("cannot access obj: %v", err))
        return true
    }

    // 2、若存在 node 对象，从 uidToNode 中取出该 event 的 node 对象
    existingNode, found := gb.uidToNode.Read(accessor.GetUID())
    if found {
        existingNode.markObserved()
    }
    switch {
    // 3、若 event 为 add 或 update 类型以及对应的 node 对象不存在时
    case (event.eventType == addEvent || event.eventType == updateEvent) && !found:
        // 4、为 node 创建 event 对象
        newNode := &node{
            ......
        }
        // 5、在 uidToNode 中添加该 node 对象
        gb.insertNode(newNode)

        // 6、检查并处理 node 的删除操作 
        gb.processTransitions(event.oldObj, accessor, newNode)
        
    // 7、若 event 为 add 或 update 类型以及对应的 node 对象存在时
    case (event.eventType == addEvent || event.eventType == updateEvent) && found:
        added, removed, changed := referencesDiffs(existingNode.owners, accessor.GetOwnerReferences())
        // 8、若 node 的 owners 有变化
        if len(added) != 0 || len(removed) != 0 || len(changed) != 0 {

            gb.addUnblockedOwnersToDeleteQueue(removed, changed)
            // 9、更新 uidToNode 中的 owners
            existingNode.owners = accessor.GetOwnerReferences()
            // 10、添加更新后 Owners 对应的 dependent
            gb.addDependentToOwners(existingNode, added)
            // 11、移除旧 owners 对应的 dependents
            gb.removeDependentFromOwners(existingNode, removed)
        }
				
        // 12、检查是否处于删除状态
        if beingDeleted(accessor) {
            existingNode.markBeingDeleted()
        }
        // 13、检查并处理 node 的删除操作 
        gb.processTransitions(event.oldObj, accessor, existingNode)
        
    // 14、若为 delete event
    case event.eventType == deleteEvent:
        if !found {
            return true
        }
        // 15、从 uidToNode 中删除该 node
        gb.removeNode(existingNode)
        existingNode.dependentsLock.RLock()
        defer existingNode.dependentsLock.RUnlock()
        if len(existingNode.dependents) > 0 {
            gb.absentOwnerCache.Add(accessor.GetUID())
        }
        // 16、删除该 node 的 dependents
        for dep := range existingNode.dependents {
            gb.attemptToDelete.Add(dep)
        }
        // 17、删除该 node 处于删除阻塞状态的 owner 
        for _, owner := range existingNode.owners {
            ownerNode, found := gb.uidToNode.Read(owner.UID)
            if !found || !ownerNode.isDeletingDependents() {
                continue
            }
            gb.attemptToDelete.Add(ownerNode)
        }
    }
    return true
}

func (gb *GraphBuilder) processTransitions(oldObj interface{}, newAccessor metav1.Object, n *node) {
	if startsWaitingForDependentsOrphaned(oldObj, newAccessor) {
		klog.V(5).Infof("add %s to the attemptToOrphan", n.identity)
		gb.attemptToOrphan.Add(n)
		return
	}
	if startsWaitingForDependentsDeleted(oldObj, newAccessor) {
		klog.V(2).Infof("add %s to the attemptToDelete, because it's waiting for its dependents to be deleted", n.identity)
		// if the n is added as a "virtual" node, its deletingDependents field is not properly set, so always set it here.
		n.markDeletingDependents()
		for dep := range n.dependents {
			gb.attemptToDelete.Add(dep)
		}
		gb.attemptToDelete.Add(n)
	}
}
```



**gc.attemptToDeleteItem**

 `gc.attemptToDeleteItem` 的主要逻辑为：

- 1、判断 node 是否处于删除状态；
- 2、从 apiserver 获取该 node 最新的状态，该 node 可能为 virtual node，若为 virtual node 则从 apiserver 中获取不到该 node 的对象，此时会将该 node 重新加入到 graphChanges 队列中，再次处理该 node 时会将其从 uidToNode 中删除；
- 3、判断该 node 最新状态的 uid 是否等于本地缓存中的 uid，若不匹配说明该 node 已更新过此时将其设置为 virtual node 并重新加入到 graphChanges 队列中，再次处理该 node 时会将其从 uidToNode 中删除；
- 4、通过 node 的 `deletingDependents` 字段判断该 node 当前是否处于删除 dependents 的状态，若该 node 处于删除 dependents 的状态则调用 `processDeletingDependentsItem` 方法检查 node 的 `blockingDependents` 是否被完全删除，若 `blockingDependents` 已完全被删除则删除该 node 对应的 finalizer，若 `blockingDependents` 还未删除完，将未删除的 `blockingDependents` 加入到 attemptToDelete 中；  上文中在 GraphBuilder 处理 graphChanges 中的事件时，若发现 node 处于删除状态，会将 node 的 dependents 加入到 attemptToDelete 中并标记 node 的 `deletingDependents` 为 true；  
- 5、调用 `gc.classifyReferences` 将 node 的 `ownerReferences` 分类为 `solid`, `dangling`, `waitingForDependentsDeletion` 三类：`dangling`(owner 不存在)、`waitingForDependentsDeletion`(owner 存在，owner 处于删除状态且正在等待其 dependents 被删除)、`solid`(至少有一个 owner 存在且不处于删除状态)；
- 6、对以上分类进行不同的处理，若 `solid`不为 0 即当前 node 至少存在一个 owner，该对象还不能被回收，此时需要将 `dangling` 和 `waitingForDependentsDeletion` 列表中的 owner 从 node 的 `ownerReferences` 删除，即已经被删除或等待删除的引用从对象中删掉；
- 7、第二种情况是该 node 的 owner 处于 `waitingForDependentsDeletion` 状态并且 node 的 dependents 未被完全删除，该 node 需要等待删除完所有的 dependents 后才能被删除；
- 8、第三种情况就是该 node 已经没有任何 dependents 了，此时按照 node 中声明的删除策略调用 apiserver 的接口删除即可；

 

```go
func (gc *GarbageCollector) attemptToDeleteItem(item *node) error {
	klog.V(2).InfoS("Processing object", "object", klog.KRef(item.identity.Namespace, item.identity.Name),
		"objectUID", item.identity.UID, "kind", item.identity.Kind)

	// "being deleted" is an one-way trip to the final deletion. We'll just wait for the final deletion, and then process the object's dependents.
// 1、判断 node 是否处于删除状态
	if item.isBeingDeleted() && !item.isDeletingDependents() {
		klog.V(5).Infof("processing item %s returned at once, because its DeletionTimestamp is non-nil", item.identity)
		return nil
	}
	// TODO: It's only necessary to talk to the API server if this is a
	// "virtual" node. The local graph could lag behind the real status, but in
	// practice, the difference is small.
// 2、从 apiserver 获取该 node 最新的状态
	latest, err := gc.getObject(item.identity)
	switch {
	case errors.IsNotFound(err):
		// the GraphBuilder can add "virtual" node for an owner that doesn't
		// exist yet, so we need to enqueue a virtual Delete event to remove
		// the virtual node from GraphBuilder.uidToNode.
		klog.V(5).Infof("item %v not found, generating a virtual delete event", item.identity)
		gc.dependencyGraphBuilder.enqueueVirtualDeleteEvent(item.identity)
		// since we're manually inserting a delete event to remove this node,
		// we don't need to keep tracking it as a virtual node and requeueing in attemptToDelete
		item.markObserved()
		return nil
	case err != nil:
		return err
	}
 // 3、判断该 node 最新状态的 uid 是否等于本地缓存中的 uid
	if latest.GetUID() != item.identity.UID {
		klog.V(5).Infof("UID doesn't match, item %v not found, generating a virtual delete event", item.identity)
		gc.dependencyGraphBuilder.enqueueVirtualDeleteEvent(item.identity)
		// since we're manually inserting a delete event to remove this node,
		// we don't need to keep tracking it as a virtual node and requeueing in attemptToDelete
		item.markObserved()
		return nil
	}

	// TODO: attemptToOrphanWorker() routine is similar. Consider merging
	// attemptToOrphanWorker() into attemptToDeleteItem() as well.
 // 4、判断该 node 当前是否处于删除 dependents 状态中
	if item.isDeletingDependents() {
		return gc.processDeletingDependentsItem(item)
	}

	// compute if we should delete the item
 // 5、检查 node 是否还存在 ownerReferences 
	ownerReferences := latest.GetOwnerReferences()
	if len(ownerReferences) == 0 {
		klog.V(2).Infof("object %s's doesn't have an owner, continue on next item", item.identity)
		return nil
	}
// 6、对 ownerReferences 进行分类
	solid, dangling, waitingForDependentsDeletion, err := gc.classifyReferences(item, ownerReferences)
	if err != nil {
		return err
	}
	klog.V(5).Infof("classify references of %s.\nsolid: %#v\ndangling: %#v\nwaitingForDependentsDeletion: %#v\n", item.identity, solid, dangling, waitingForDependentsDeletion)

	switch {
  // 7、存在不处于删除状态的 owner
	case len(solid) != 0:
		klog.V(2).Infof("object %#v has at least one existing owner: %#v, will not garbage collect", item.identity, solid)
		if len(dangling) == 0 && len(waitingForDependentsDeletion) == 0 {
			return nil
		}
		klog.V(2).Infof("remove dangling references %#v and waiting references %#v for object %s", dangling, waitingForDependentsDeletion, item.identity)
		// waitingForDependentsDeletion needs to be deleted from the
		// ownerReferences, otherwise the referenced objects will be stuck with
		// the FinalizerDeletingDependents and never get deleted.
		ownerUIDs := append(ownerRefsToUIDs(dangling), ownerRefsToUIDs(waitingForDependentsDeletion)...)
		patch := deleteOwnerRefStrategicMergePatch(item.identity.UID, ownerUIDs...)
		_, err = gc.patch(item, patch, func(n *node) ([]byte, error) {
			return gc.deleteOwnerRefJSONMergePatch(n, ownerUIDs...)
		})
		return err
 // 8、node 的 owner 处于 waitingForDependentsDeletion 状态并且 node的 dependents 未被完全删除
	case len(waitingForDependentsDeletion) != 0 && item.dependentsLength() != 0:
		deps := item.getDependents()
 // 9、删除 dependents
		for _, dep := range deps {
			if dep.isDeletingDependents() {
				// this circle detection has false positives, we need to
				// apply a more rigorous detection if this turns out to be a
				// problem.
				// there are multiple workers run attemptToDeleteItem in
				// parallel, the circle detection can fail in a race condition.
				klog.V(2).Infof("processing object %s, some of its owners and its dependent [%s] have FinalizerDeletingDependents, to prevent potential cycle, its ownerReferences are going to be modified to be non-blocking, then the object is going to be deleted with Foreground", item.identity, dep.identity)
				patch, err := item.unblockOwnerReferencesStrategicMergePatch()
				if err != nil {
					return err
				}
				if _, err := gc.patch(item, patch, gc.unblockOwnerReferencesJSONMergePatch); err != nil {
					return err
				}
				break
			}
		}
		klog.V(2).Infof("at least one owner of object %s has FinalizerDeletingDependents, and the object itself has dependents, so it is going to be deleted in Foreground", item.identity)
		// the deletion event will be observed by the graphBuilder, so the item
		// will be processed again in processDeletingDependentsItem. If it
		// doesn't have dependents, the function will remove the
		// FinalizerDeletingDependents from the item, resulting in the final
		// deletion of the item.
// 10、以 Foreground 模式删除 node 对象
		policy := metav1.DeletePropagationForeground
		return gc.deleteObject(item.identity, &policy)
        // 11、该 node 已经没有任何依赖了，按照 node 中声明的删除策略调用 apiserver 的接口删除
	default:
		// item doesn't have any solid owner, so it needs to be garbage
		// collected. Also, none of item's owners is waiting for the deletion of
		// the dependents, so set propagationPolicy based on existing finalizers.
		var policy metav1.DeletionPropagation
		switch {
		case hasOrphanFinalizer(latest):
			// if an existing orphan finalizer is already on the object, honor it.
			policy = metav1.DeletePropagationOrphan
		case hasDeleteDependentsFinalizer(latest):
			// if an existing foreground finalizer is already on the object, honor it.
			policy = metav1.DeletePropagationForeground
		default:
			// otherwise, default to background.
			policy = metav1.DeletePropagationBackground
		}
		klog.V(2).InfoS("Deleting object", "object", klog.KRef(item.identity.Namespace, item.identity.Name),
			"objectUID", item.identity.UID, "kind", item.identity.Kind, "propagationPolicy", policy)
		return gc.deleteObject(item.identity, &policy)
	}
}
```



**gc.runAttemptToOrphanWorker**

`runAttemptToOrphanWorker` 是处理以 `orphan` 模式删除的 node，主要逻辑为：

- 1、调用 `gc.orphanDependents` 删除 owner 所有 dependents `OwnerReferences` 中的 owner 字段；
- 2、调用 `gc.removeFinalizer` 删除 owner 的 `orphan` Finalizer；
- 3、以上两步中若有失败的会进行重试；

```go
// attemptToOrphanWorker dequeues a node from the attemptToOrphan, then finds its
// dependents based on the graph maintained by the GC, then removes it from the
// OwnerReferences of its dependents, and finally updates the owner to remove
// the "Orphan" finalizer. The node is added back into the attemptToOrphan if any of
// these steps fail.
func (gc *GarbageCollector) attemptToOrphanWorker() bool {
	item, quit := gc.attemptToOrphan.Get()
	gc.workerLock.RLock()
	defer gc.workerLock.RUnlock()
	if quit {
		return false
	}
	defer gc.attemptToOrphan.Done(item)
	owner, ok := item.(*node)
	if !ok {
		utilruntime.HandleError(fmt.Errorf("expect *node, got %#v", item))
		return true
	}
	// we don't need to lock each element, because they never get updated
	owner.dependentsLock.RLock()
	dependents := make([]*node, 0, len(owner.dependents))
	for dependent := range owner.dependents {
		dependents = append(dependents, dependent)
	}
	owner.dependentsLock.RUnlock()

	err := gc.orphanDependents(owner.identity, dependents)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("orphanDependents for %s failed with %v", owner.identity, err))
		gc.attemptToOrphan.AddRateLimited(item)
		return true
	}
	// update the owner, remove "orphaningFinalizer" from its finalizers list
    // 更新 owner, 从 finalizers 列表中移除 "orphaningFinalizer"
	err = gc.removeFinalizer(owner, metav1.FinalizerOrphanDependents)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("removeOrphanFinalizer for %s failed with %v", owner.identity, err))
		gc.attemptToOrphan.AddRateLimited(item)
	}
	return true
}
// dependents are copies of pointers to the owner's dependents, they don't need to be locked.
func (gc *GarbageCollector) orphanDependents(owner objectReference, dependents []*node) error {
	errCh := make(chan error, len(dependents))
	wg := sync.WaitGroup{}
	wg.Add(len(dependents))
	for i := range dependents {
		go func(dependent *node) {
			defer wg.Done()
			// the dependent.identity.UID is used as precondition
			patch := deleteOwnerRefStrategicMergePatch(dependent.identity.UID, owner.UID)
			_, err := gc.patch(dependent, patch, func(n *node) ([]byte, error) {
				return gc.deleteOwnerRefJSONMergePatch(n, owner.UID)
			})
			// note that if the target ownerReference doesn't exist in the
			// dependent, strategic merge patch will NOT return an error.
			if err != nil && !errors.IsNotFound(err) {
				errCh <- fmt.Errorf("orphaning %s failed, %v", dependent.identity, err)
			}
		}(dependents[i])
	}
	wg.Wait()
	close(errCh)

	var errorsSlice []error
	for e := range errCh {
		errorsSlice = append(errorsSlice, e)
	}

	if len(errorsSlice) != 0 {
		return fmt.Errorf("failed to orphan dependents of owner %s, got errors: %s", owner, utilerrors.NewAggregate(errorsSlice).Error())
	}
	klog.V(5).Infof("successfully updated all dependents of owner %s", owner)
	return nil
}
```

## 总结

GarbageCollectorController 是一种典型的生产者消费者模型，所有 `deletableResources` 的 informer 都是生产者，每种资源的 informer 监听到变化后都会将对应的事件 push 到 graphChanges 中，graphChanges 是 GraphBuilder 对象中的一个数据结构，**GraphBuilder 会启动另外的 goroutine 对 graphChanges 中的事件进行分类并放在其 attemptToDelete 和 attemptToOrphan 两个队列中，garbageCollector 会启动多个 goroutine 对 attemptToDelete 和 attemptToOrphan 两个队列中的事件进行处理，处理的结果就是回收一些需要被删除的对象**。最后，再用一个流程图总结一下 GarbageCollectorController 的主要流程:

```
                    monitors (producer)
                          |
                          |
                          ∨
                  graphChanges queue
                          |
                          |
                          ∨
                  processGraphChanges
                          |
                          |
                          ∨
          -------------------------------
          |                             |
          |                             |
          ∨                             ∨
attemptToDelete queue         attemptToOrphan queue
          |                             |
          |                             |
          ∨                             ∨
  AttemptToDeleteWorker       AttemptToOrphanWorker
      (consumer)                    (consumer)
```



# job 控制器

如下job。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  completions: 10
  parallelism: 5
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

由于任务 `pi` 在配置时指定了 `spec.completions=10`，所以当前的任务需要等待 10 个 Pod 的成功执行，另一个比较重要的 `spec.parallelism=5` 表示最多有多少个并发执行的任务，如果 `spec.parallelism=1` 那么所有的任务都会依次顺序执行，只有前一个任务执行成功时，后一个任务才会开始工作。 

## 主流程

用于处理 Job 资源的 `JobController` 控制器会监听 Pod 和 Job 这两个资源的变更事件，而资源的同步还是需要运行 `syncJob` 方法。 `syncJob` 是 `JobController` 中主要用于同步 Job 资源的方法，这个方法的主要作用就是对资源进行同步，它的大体架构就是先获取一些当前同步的基本信息，然后调用 `manageJob` 方法管理 Job 对应的 Pod 对象，最后计算出处于 `active`、`failed` 和 `succeed` 三种状态的 Pod 数量并更新 Job 的状态。

```go
// syncJob will sync the job with the given key if it has had its expectations fulfilled, meaning
// it did not expect to see any more of its pods created or deleted. This function is not meant to be invoked
// concurrently with the same key.
func (jm *Controller) syncJob(key string) (bool, error) {
	startTime := time.Now()
	defer func() {
		klog.V(4).Infof("Finished syncing job %q (%v)", key, time.Since(startTime))
	}()

	ns, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return false, err
	}
	if len(ns) == 0 || len(name) == 0 {
		return false, fmt.Errorf("invalid job key %q: either namespace or name is missing", key)
	}
	sharedJob, err := jm.jobLister.Jobs(ns).Get(name)
	if err != nil {
		if errors.IsNotFound(err) {
			klog.V(4).Infof("Job has been deleted: %v", key)
			jm.expectations.DeleteExpectations(key)
			return true, nil
		}
		return false, err
	}
	// make a copy so we don't mutate the shared cache
	job := *sharedJob.DeepCopy()

	// if job was finished previously, we don't want to redo the termination
	if IsJobFinished(&job) {
		return true, nil
	}

	// retrieve the previous number of retry
	previousRetry := jm.queue.NumRequeues(key)

	// Check the expectations of the job before counting active pods, otherwise a new pod can sneak in
	// and update the expectations after we've retrieved active pods from the store. If a new pod enters
	// the store after we've checked the expectation, the job sync is just deferred till the next relist.
	jobNeedsSync := jm.expectations.SatisfiedExpectations(key)

	pods, err := jm.getPodsForJob(&job)
	if err != nil {
		return false, err
	}

	activePods := controller.FilterActivePods(pods)
	active := int32(len(activePods))
	succeeded, failed := getStatus(pods)
	conditions := len(job.Status.Conditions)
	// job first start
	if job.Status.StartTime == nil {
		now := metav1.Now()
		job.Status.StartTime = &now
		// enqueue a sync to check if job past ActiveDeadlineSeconds
		if job.Spec.ActiveDeadlineSeconds != nil {
			klog.V(4).Infof("Job %s has ActiveDeadlineSeconds will sync after %d seconds",
				key, *job.Spec.ActiveDeadlineSeconds)
			jm.queue.AddAfter(key, time.Duration(*job.Spec.ActiveDeadlineSeconds)*time.Second)
		}
	}

	var manageJobErr error
	jobFailed := false
	var failureReason string
	var failureMessage string

	jobHaveNewFailure := failed > job.Status.Failed
	// new failures happen when status does not reflect the failures and active
	// is different than parallelism, otherwise the previous controller loop
	// failed updating status so even if we pick up failure it is not a new one
	exceedsBackoffLimit := jobHaveNewFailure && (active != *job.Spec.Parallelism) &&
		(int32(previousRetry)+1 > *job.Spec.BackoffLimit)

	if exceedsBackoffLimit || pastBackoffLimitOnFailure(&job, pods) {
		// check if the number of pod restart exceeds backoff (for restart OnFailure only)
		// OR if the number of failed jobs increased since the last syncJob
		jobFailed = true
		failureReason = "BackoffLimitExceeded"
		failureMessage = "Job has reached the specified backoff limit"
	} else if pastActiveDeadline(&job) {
		jobFailed = true
		failureReason = "DeadlineExceeded"
		failureMessage = "Job was active longer than specified deadline"
	}

	if jobFailed {
		errCh := make(chan error, active)
		jm.deleteJobPods(&job, activePods, errCh)
		select {
		case manageJobErr = <-errCh:
			if manageJobErr != nil {
				break
			}
		default:
		}

		// update status values accordingly
		failed += active
		active = 0
		job.Status.Conditions = append(job.Status.Conditions, newCondition(batch.JobFailed, failureReason, failureMessage))
		jm.recorder.Event(&job, v1.EventTypeWarning, failureReason, failureMessage)
	} else {
		if jobNeedsSync && job.DeletionTimestamp == nil {
			active, manageJobErr = jm.manageJob(activePods, succeeded, &job)
		}
		completions := succeeded
		complete := false
		if job.Spec.Completions == nil {
			// This type of job is complete when any pod exits with success.
			// Each pod is capable of
			// determining whether or not the entire Job is done.  Subsequent pods are
			// not expected to fail, but if they do, the failure is ignored.  Once any
			// pod succeeds, the controller waits for remaining pods to finish, and
			// then the job is complete.
			if succeeded > 0 && active == 0 {
				complete = true
			}
		} else {
			// Job specifies a number of completions.  This type of job signals
			// success by having that number of successes.  Since we do not
			// start more pods than there are remaining completions, there should
			// not be any remaining active pods once this count is reached.
			if completions >= *job.Spec.Completions {
				complete = true
				if active > 0 {
					jm.recorder.Event(&job, v1.EventTypeWarning, "TooManyActivePods", "Too many active pods running after completion count reached")
				}
				if completions > *job.Spec.Completions {
					jm.recorder.Event(&job, v1.EventTypeWarning, "TooManySucceededPods", "Too many succeeded pods running after completion count reached")
				}
			}
		}
		if complete {
			job.Status.Conditions = append(job.Status.Conditions, newCondition(batch.JobComplete, "", ""))
			now := metav1.Now()
			job.Status.CompletionTime = &now
			jm.recorder.Event(&job, v1.EventTypeNormal, "Completed", "Job completed")
		}
	}

	forget := false
	// Check if the number of jobs succeeded increased since the last check. If yes "forget" should be true
	// This logic is linked to the issue: https://github.com/kubernetes/kubernetes/issues/56853 that aims to
	// improve the Job backoff policy when parallelism > 1 and few Jobs failed but others succeed.
	// In this case, we should clear the backoff delay.
	if job.Status.Succeeded < succeeded {
		forget = true
	}

	// no need to update the job if the status hasn't changed since last time
	if job.Status.Active != active || job.Status.Succeeded != succeeded || job.Status.Failed != failed || len(job.Status.Conditions) != conditions {
		job.Status.Active = active
		job.Status.Succeeded = succeeded
		job.Status.Failed = failed

		if err := jm.updateHandler(&job); err != nil {
			return forget, err
		}

		if jobHaveNewFailure && !IsJobFinished(&job) {
			// returning an error will re-enqueue Job after the backoff period
			return forget, fmt.Errorf("failed pod(s) detected for job key %q", key)
		}

		forget = true
	}

	return forget, manageJobErr
}
```

## manageJob

准备工作完成之后，会调用 `manageJob` 方法，该方法主要负责管理 Job 持有的一系列 Pod 的运行，它最终会返回目前集群中当前 Job 对应的活跃任务的数量。 

将 Job 规格中设置的 `spec.completions` 和已经完成的任务数量进行比对，确认当前的 Job 是否已经结束运行，如果任务已经结束运行就会更新当前 Job 的完成时间，同时当 `JobController` 发现有一些状态没有正确同步时，也会调用 `updateHandler` 更新资源的状态。 

Pod 的创建和删除都是由 `manageJob` 这个方法负责的，这个方法根据 Job 的 `spec.parallelism` 配置对目前集群中的节点进行创建和删除。

如果当前正在执行的活跃节点数量超过了 `spec.parallelism`，那么就会按照一定的排序策略删除多余的任务，删除任务时会使用 `DeletePod` 方法：  

当正在活跃的节点数量小于 `spec.parallelism` 时，我们就会根据当前未完成的任务数和并行度计算出最大可以处于活跃的 Pod 个数 `wantActive` 以及与当前的活跃 Pod 数相差的 `diff`： 

在方法的最后就会以批量的方式并行创建 Pod，所有 Pod 的创建都是通过 `CreatePodsWithControllerRef` 方法在 Goroutine 中执行的。 这里会使用 `WaitGroup` 等待每个 Batch 中创建的结果返回才会执行下一个 Batch，Batch 的大小是从 1 开始指数增加的，以冷启动的方式避免首次创建的任务过多造成失败： 通过对 `manageJob` 的分析，我们其实能够看出这个方法就是根据规格中的配置对 Pod 进行管理，它在较多并行时删除 Pod，较少并行时创建 Pod，也算是一个简单的资源利用和调度机制 。

```go
// manageJob is the core method responsible for managing the number of running
// pods according to what is specified in the job.Spec.
// Does NOT modify <activePods>.
func (jm *Controller) manageJob(activePods []*v1.Pod, succeeded int32, job *batch.Job) (int32, error) {
   var activeLock sync.Mutex
   active := int32(len(activePods))
   parallelism := *job.Spec.Parallelism
   jobKey, err := controller.KeyFunc(job)
   if err != nil {
      utilruntime.HandleError(fmt.Errorf("Couldn't get key for job %#v: %v", job, err))
      return 0, nil
   }

   var errCh chan error
   if active > parallelism {
      diff := active - parallelism
      errCh = make(chan error, diff)
      jm.expectations.ExpectDeletions(jobKey, int(diff))
      klog.V(4).Infof("Too many pods running job %q, need %d, deleting %d", jobKey, parallelism, diff)
      // Sort the pods in the order such that not-ready < ready, unscheduled
      // < scheduled, and pending < running. This ensures that we delete pods
      // in the earlier stages whenever possible.
      sort.Sort(controller.ActivePods(activePods))

      active -= diff
      wait := sync.WaitGroup{}
      wait.Add(int(diff))
      for i := int32(0); i < diff; i++ {
         go func(ix int32) {
            defer wait.Done()
            if err := jm.podControl.DeletePod(job.Namespace, activePods[ix].Name, job); err != nil {
               // Decrement the expected number of deletes because the informer won't observe this deletion
               jm.expectations.DeletionObserved(jobKey)
               if !apierrors.IsNotFound(err) {
                  klog.V(2).Infof("Failed to delete %v, decremented expectations for job %q/%q", activePods[ix].Name, job.Namespace, job.Name)
                  activeLock.Lock()
                  active++
                  activeLock.Unlock()
                  errCh <- err
                  utilruntime.HandleError(err)
               }

            }
         }(i)
      }
      wait.Wait()

   } else if active < parallelism {
      wantActive := int32(0)
      if job.Spec.Completions == nil {
         // Job does not specify a number of completions.  Therefore, number active
         // should be equal to parallelism, unless the job has seen at least
         // once success, in which leave whatever is running, running.
         if succeeded > 0 {
            wantActive = active
         } else {
            wantActive = parallelism
         }
      } else {
         // Job specifies a specific number of completions.  Therefore, number
         // active should not ever exceed number of remaining completions.
         wantActive = *job.Spec.Completions - succeeded
         if wantActive > parallelism {
            wantActive = parallelism
         }
      }
      diff := wantActive - active
      if diff < 0 {
         utilruntime.HandleError(fmt.Errorf("More active than wanted: job %q, want %d, have %d", jobKey, wantActive, active))
         diff = 0
      }
      if diff == 0 {
         return active, nil
      }
      jm.expectations.ExpectCreations(jobKey, int(diff))
      errCh = make(chan error, diff)
      klog.V(4).Infof("Too few pods running job %q, need %d, creating %d", jobKey, wantActive, diff)

      active += diff
      wait := sync.WaitGroup{}

      // Batch the pod creates. Batch sizes start at SlowStartInitialBatchSize
      // and double with each successful iteration in a kind of "slow start".
      // This handles attempts to start large numbers of pods that would
      // likely all fail with the same error. For example a project with a
      // low quota that attempts to create a large number of pods will be
      // prevented from spamming the API service with the pod create requests
      // after one of its pods fails.  Conveniently, this also prevents the
      // event spam that those failures would generate.
      for batchSize := int32(integer.IntMin(int(diff), controller.SlowStartInitialBatchSize)); diff > 0; batchSize = integer.Int32Min(2*batchSize, diff) {
         errorCount := len(errCh)
         wait.Add(int(batchSize))
         for i := int32(0); i < batchSize; i++ {
            go func() {
               defer wait.Done()
               err := jm.podControl.CreatePodsWithControllerRef(job.Namespace, &job.Spec.Template, job, metav1.NewControllerRef(job, controllerKind))
               if err != nil {
                  if errors.HasStatusCause(err, v1.NamespaceTerminatingCause) {
                     // If the namespace is being torn down, we can safely ignore
                     // this error since all subsequent creations will fail.
                     return
                  }
               }
               if err != nil {
                  defer utilruntime.HandleError(err)
                  // Decrement the expected number of creates because the informer won't observe this pod
                  klog.V(2).Infof("Failed creation, decrementing expectations for job %q/%q", job.Namespace, job.Name)
                  jm.expectations.CreationObserved(jobKey)
                  activeLock.Lock()
                  active--
                  activeLock.Unlock()
                  errCh <- err
               }
            }()
         }
         wait.Wait()
         // any skipped pods that we never attempted to start shouldn't be expected.
         skippedPods := diff - batchSize
         if errorCount < len(errCh) && skippedPods > 0 {
            klog.V(2).Infof("Slow-start failure. Skipping creation of %d pods, decrementing expectations for job %q/%q", skippedPods, job.Namespace, job.Name)
            active -= skippedPods
            for i := int32(0); i < skippedPods; i++ {
               // Decrement the expected number of creates because the informer won't observe this pod
               jm.expectations.CreationObserved(jobKey)
            }
            // The skipped pods will be retried later. The next controller resync will
            // retry the slow start process.
            break
         }
         diff -= batchSize
      }
   }

   select {
   case err := <-errCh:
      // all errors have been reported before, we only need to inform the controller that there was an error and it should re-try this job once more next time.
      if err != nil {
         return active, err
      }
   default:
   }

   return active, nil
}
```



# cronjob控制器

每一个 Job 对象都会持有一个或者多个 Pod，而每一个 CronJob 就会持有多个 Job 对象，CronJob 能够按照时间对任务进行调度，我们可以使用 Cron 格式快速指定任务的调度时间。CronJob 控制job。

用于管理 CronJob 资源的 `CronJobController` 虽然也使用了控制器模式，但是它的实现与其他的控制器不太一样，他没有从 Informer 中接受其他消息变动的通知，而是直接访问 apiserver 中的数据 

```go
// Controller is a controller for CronJobs.
type Controller struct {
   kubeClient clientset.Interface
   jobControl jobControlInterface
   cjControl  cjControlInterface
   podControl podControlInterface
   recorder   record.EventRecorder
}

// NewController creates and initializes a new Controller.
func NewController(kubeClient clientset.Interface) (*Controller, error) {
   eventBroadcaster := record.NewBroadcaster()
   eventBroadcaster.StartStructuredLogging(0)
   eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: kubeClient.CoreV1().Events("")})

   if kubeClient != nil && kubeClient.CoreV1().RESTClient().GetRateLimiter() != nil {
      if err := ratelimiter.RegisterMetricAndTrackRateLimiterUsage("cronjob_controller", kubeClient.CoreV1().RESTClient().GetRateLimiter()); err != nil {
         return nil, err
      }
   }

   jm := &Controller{
      kubeClient: kubeClient,
      jobControl: realJobControl{KubeClient: kubeClient},
      cjControl:  &realCJControl{KubeClient: kubeClient},
      podControl: &realPodControl{KubeClient: kubeClient},
      recorder:   eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "cronjob-controller"}),
   }

   return jm, nil
}

// Run starts the main goroutine responsible for watching and syncing jobs.
func (jm *Controller) Run(stopCh <-chan struct{}) {
   defer utilruntime.HandleCrash()
   klog.Infof("Starting CronJob Manager")
   // Check things every 10 second.
   go wait.Until(jm.syncAll, 10*time.Second, stopCh)
   <-stopCh
   klog.Infof("Shutting down CronJob Manager")
}

// syncAll lists all the CronJobs and Jobs and reconciles them.
func (jm *Controller) syncAll() {
   // List children (Jobs) before parents (CronJob).
   // This guarantees that if we see any Job that got orphaned by the GC orphan finalizer,
   // we must also see that the parent CronJob has non-nil DeletionTimestamp (see #42639).
   // Note that this only works because we are NOT using any caches here.
   jobListFunc := func(opts metav1.ListOptions) (runtime.Object, error) {
      return jm.kubeClient.BatchV1().Jobs(metav1.NamespaceAll).List(context.TODO(), opts)
   }

   js := make([]batchv1.Job, 0)
   err := pager.New(pager.SimplePageFunc(jobListFunc)).EachListItem(context.Background(), metav1.ListOptions{}, func(object runtime.Object) error {
      jobTmp, ok := object.(*batchv1.Job)
      if !ok {
         return fmt.Errorf("expected type *batchv1.Job, got type %T", jobTmp)
      }
      js = append(js, *jobTmp)
      return nil
   })

   if err != nil {
      utilruntime.HandleError(fmt.Errorf("Failed to extract job list: %v", err))
      return
   }

   klog.V(4).Infof("Found %d jobs", len(js))
   cronJobListFunc := func(opts metav1.ListOptions) (runtime.Object, error) {
      return jm.kubeClient.BatchV1beta1().CronJobs(metav1.NamespaceAll).List(context.TODO(), opts)
   }

   jobsByCj := groupJobsByParent(js)
   klog.V(4).Infof("Found %d groups", len(jobsByCj))
   err = pager.New(pager.SimplePageFunc(cronJobListFunc)).EachListItem(context.Background(), metav1.ListOptions{}, func(object runtime.Object) error {
      cj, ok := object.(*batchv1beta1.CronJob)
      if !ok {
         return fmt.Errorf("expected type *batchv1beta1.CronJob, got type %T", cj)
      }
      syncOne(cj, jobsByCj[cj.UID], time.Now(), jm.jobControl, jm.cjControl, jm.recorder)
      cleanupFinishedJobs(cj, jobsByCj[cj.UID], jm.jobControl, jm.cjControl, jm.recorder)
      return nil
   })

   if err != nil {
      utilruntime.HandleError(fmt.Errorf("Failed to extract cronJobs list: %v", err))
      return
   }
}

// cleanupFinishedJobs cleanups finished jobs created by a CronJob
func cleanupFinishedJobs(cj *batchv1beta1.CronJob, js []batchv1.Job, jc jobControlInterface,
   cjc cjControlInterface, recorder record.EventRecorder) {
   // If neither limits are active, there is no need to do anything.
   if cj.Spec.FailedJobsHistoryLimit == nil && cj.Spec.SuccessfulJobsHistoryLimit == nil {
      return
   }

   failedJobs := []batchv1.Job{}
   successfulJobs := []batchv1.Job{}

   for _, job := range js {
      isFinished, finishedStatus := getFinishedStatus(&job)
      if isFinished && finishedStatus == batchv1.JobComplete {
         successfulJobs = append(successfulJobs, job)
      } else if isFinished && finishedStatus == batchv1.JobFailed {
         failedJobs = append(failedJobs, job)
      }
   }

   if cj.Spec.SuccessfulJobsHistoryLimit != nil {
      removeOldestJobs(cj,
         successfulJobs,
         jc,
         *cj.Spec.SuccessfulJobsHistoryLimit,
         recorder)
   }

   if cj.Spec.FailedJobsHistoryLimit != nil {
      removeOldestJobs(cj,
         failedJobs,
         jc,
         *cj.Spec.FailedJobsHistoryLimit,
         recorder)
   }

   // Update the CronJob, in case jobs were removed from the list.
   if _, err := cjc.UpdateStatus(cj); err != nil {
      nameForLog := fmt.Sprintf("%s/%s", cj.Namespace, cj.Name)
      klog.Infof("Unable to update status for %s (rv = %s): %v", nameForLog, cj.ResourceVersion, err)
   }
}

// removeOldestJobs removes the oldest jobs from a list of jobs
func removeOldestJobs(cj *batchv1beta1.CronJob, js []batchv1.Job, jc jobControlInterface, maxJobs int32, recorder record.EventRecorder) {
   numToDelete := len(js) - int(maxJobs)
   if numToDelete <= 0 {
      return
   }

   nameForLog := fmt.Sprintf("%s/%s", cj.Namespace, cj.Name)
   klog.V(4).Infof("Cleaning up %d/%d jobs from %s", numToDelete, len(js), nameForLog)

   sort.Sort(byJobStartTime(js))
   for i := 0; i < numToDelete; i++ {
      klog.V(4).Infof("Removing job %s from %s", js[i].Name, nameForLog)
      deleteJob(cj, &js[i], jc, recorder)
   }
}

// syncOne reconciles a CronJob with a list of any Jobs that it created.
// All known jobs created by "cj" should be included in "js".
// The current time is passed in to facilitate testing.
// It has no receiver, to facilitate testing.
func syncOne(cj *batchv1beta1.CronJob, js []batchv1.Job, now time.Time, jc jobControlInterface, cjc cjControlInterface, recorder record.EventRecorder) {
   nameForLog := fmt.Sprintf("%s/%s", cj.Namespace, cj.Name)

   childrenJobs := make(map[types.UID]bool)
   for _, j := range js {
      childrenJobs[j.ObjectMeta.UID] = true
      found := inActiveList(*cj, j.ObjectMeta.UID)
      if !found && !IsJobFinished(&j) {
         recorder.Eventf(cj, v1.EventTypeWarning, "UnexpectedJob", "Saw a job that the controller did not create or forgot: %s", j.Name)
         // We found an unfinished job that has us as the parent, but it is not in our Active list.
         // This could happen if we crashed right after creating the Job and before updating the status,
         // or if our jobs list is newer than our cj status after a relist, or if someone intentionally created
         // a job that they wanted us to adopt.

         // TODO: maybe handle the adoption case?  Concurrency/suspend rules will not apply in that case, obviously, since we can't
         // stop users from creating jobs if they have permission.  It is assumed that if a
         // user has permission to create a job within a namespace, then they have permission to make any cronJob
         // in the same namespace "adopt" that job.  ReplicaSets and their Pods work the same way.
         // TBS: how to update cj.Status.LastScheduleTime if the adopted job is newer than any we knew about?
      } else if found && IsJobFinished(&j) {
         _, status := getFinishedStatus(&j)
         deleteFromActiveList(cj, j.ObjectMeta.UID)
         recorder.Eventf(cj, v1.EventTypeNormal, "SawCompletedJob", "Saw completed job: %s, status: %v", j.Name, status)
      }
   }

   // Remove any job reference from the active list if the corresponding job does not exist any more.
   // Otherwise, the cronjob may be stuck in active mode forever even though there is no matching
   // job running.
   for _, j := range cj.Status.Active {
      if found := childrenJobs[j.UID]; !found {
         recorder.Eventf(cj, v1.EventTypeNormal, "MissingJob", "Active job went missing: %v", j.Name)
         deleteFromActiveList(cj, j.UID)
      }
   }

   updatedCJ, err := cjc.UpdateStatus(cj)
   if err != nil {
      klog.Errorf("Unable to update status for %s (rv = %s): %v", nameForLog, cj.ResourceVersion, err)
      return
   }
   *cj = *updatedCJ

   if cj.DeletionTimestamp != nil {
      // The CronJob is being deleted.
      // Don't do anything other than updating status.
      return
   }

   if cj.Spec.Suspend != nil && *cj.Spec.Suspend {
      klog.V(4).Infof("Not starting job for %s because it is suspended", nameForLog)
      return
   }

   times, err := getRecentUnmetScheduleTimes(*cj, now)
   if err != nil {
      recorder.Eventf(cj, v1.EventTypeWarning, "FailedNeedsStart", "Cannot determine if job needs to be started: %v", err)
      klog.Errorf("Cannot determine if %s needs to be started: %v", nameForLog, err)
      return
   }
   // TODO: handle multiple unmet start times, from oldest to newest, updating status as needed.
   if len(times) == 0 {
      klog.V(4).Infof("No unmet start times for %s", nameForLog)
      return
   }
   if len(times) > 1 {
      klog.V(4).Infof("Multiple unmet start times for %s so only starting last one", nameForLog)
   }

   scheduledTime := times[len(times)-1]
   tooLate := false
   if cj.Spec.StartingDeadlineSeconds != nil {
      tooLate = scheduledTime.Add(time.Second * time.Duration(*cj.Spec.StartingDeadlineSeconds)).Before(now)
   }
   if tooLate {
      klog.V(4).Infof("Missed starting window for %s", nameForLog)
      recorder.Eventf(cj, v1.EventTypeWarning, "MissSchedule", "Missed scheduled time to start a job: %s", scheduledTime.Format(time.RFC1123Z))
      // TODO: Since we don't set LastScheduleTime when not scheduling, we are going to keep noticing
      // the miss every cycle.  In order to avoid sending multiple events, and to avoid processing
      // the cj again and again, we could set a Status.LastMissedTime when we notice a miss.
      // Then, when we call getRecentUnmetScheduleTimes, we can take max(creationTimestamp,
      // Status.LastScheduleTime, Status.LastMissedTime), and then so we won't generate
      // and event the next time we process it, and also so the user looking at the status
      // can see easily that there was a missed execution.
      return
   }
   if cj.Spec.ConcurrencyPolicy == batchv1beta1.ForbidConcurrent && len(cj.Status.Active) > 0 {
      // Regardless which source of information we use for the set of active jobs,
      // there is some risk that we won't see an active job when there is one.
      // (because we haven't seen the status update to the SJ or the created pod).
      // So it is theoretically possible to have concurrency with Forbid.
      // As long the as the invocations are "far enough apart in time", this usually won't happen.
      //
      // TODO: for Forbid, we could use the same name for every execution, as a lock.
      // With replace, we could use a name that is deterministic per execution time.
      // But that would mean that you could not inspect prior successes or failures of Forbid jobs.
      klog.V(4).Infof("Not starting job for %s because of prior execution still running and concurrency policy is Forbid", nameForLog)
      return
   }
   if cj.Spec.ConcurrencyPolicy == batchv1beta1.ReplaceConcurrent {
      for _, j := range cj.Status.Active {
         klog.V(4).Infof("Deleting job %s of %s that was still running at next scheduled start time", j.Name, nameForLog)

         job, err := jc.GetJob(j.Namespace, j.Name)
         if err != nil {
            recorder.Eventf(cj, v1.EventTypeWarning, "FailedGet", "Get job: %v", err)
            return
         }
         if !deleteJob(cj, job, jc, recorder) {
            return
         }
      }
   }

   jobReq, err := getJobFromTemplate(cj, scheduledTime)
   if err != nil {
      klog.Errorf("Unable to make Job from template in %s: %v", nameForLog, err)
      return
   }
   jobResp, err := jc.CreateJob(cj.Namespace, jobReq)
   if err != nil {
      // If the namespace is being torn down, we can safely ignore
      // this error since all subsequent creations will fail.
      if !errors.HasStatusCause(err, v1.NamespaceTerminatingCause) {
         recorder.Eventf(cj, v1.EventTypeWarning, "FailedCreate", "Error creating job: %v", err)
      }
      return
   }
   klog.V(4).Infof("Created Job %s for %s", jobResp.Name, nameForLog)
   recorder.Eventf(cj, v1.EventTypeNormal, "SuccessfulCreate", "Created job %v", jobResp.Name)

   // ------------------------------------------------------------------ //

   // If this process restarts at this point (after posting a job, but
   // before updating the status), then we might try to start the job on
   // the next time.  Actually, if we re-list the SJs and Jobs on the next
   // iteration of syncAll, we might not see our own status update, and
   // then post one again.  So, we need to use the job name as a lock to
   // prevent us from making the job twice (name the job with hash of its
   // scheduled time).

   // Add the just-started job to the status list.
   ref, err := getRef(jobResp)
   if err != nil {
      klog.V(2).Infof("Unable to make object reference for job for %s", nameForLog)
   } else {
      cj.Status.Active = append(cj.Status.Active, *ref)
   }
   cj.Status.LastScheduleTime = &metav1.Time{Time: scheduledTime}
   if _, err := cjc.UpdateStatus(cj); err != nil {
      klog.Infof("Unable to update status for %s (rv = %s): %v", nameForLog, cj.ResourceVersion, err)
   }

   return
}

// deleteJob reaps a job, deleting the job, the pods and the reference in the active list
func deleteJob(cj *batchv1beta1.CronJob, job *batchv1.Job, jc jobControlInterface, recorder record.EventRecorder) bool {
   nameForLog := fmt.Sprintf("%s/%s", cj.Namespace, cj.Name)

   // delete the job itself...
   if err := jc.DeleteJob(job.Namespace, job.Name); err != nil {
      recorder.Eventf(cj, v1.EventTypeWarning, "FailedDelete", "Deleted job: %v", err)
      klog.Errorf("Error deleting job %s from %s: %v", job.Name, nameForLog, err)
      return false
   }
   // ... and its reference from active list
   deleteFromActiveList(cj, job.ObjectMeta.UID)
   recorder.Eventf(cj, v1.EventTypeNormal, "SuccessfulDelete", "Deleted job %v", job.Name)

   return true
}

func getRef(object runtime.Object) (*v1.ObjectReference, error) {
   return ref.GetReference(scheme.Scheme, object)
}
```



# ns控制器

## 初始化

ns的删除

```go
func startModifiedNamespaceController(ctx ControllerContext, namespaceKubeClient clientset.Interface, nsKubeconfig *restclient.Config) (http.Handler, bool, error) {

   metadataClient, err := metadata.NewForConfig(nsKubeconfig)
   if err != nil {
      return nil, true, err
   }

   discoverResourcesFn := namespaceKubeClient.Discovery().ServerPreferredNamespacedResources

   namespaceController := namespacecontroller.NewNamespaceController(
      namespaceKubeClient,
      metadataClient,
      discoverResourcesFn,
      ctx.InformerFactory.Core().V1().Namespaces(),
      ctx.ComponentConfig.NamespaceController.NamespaceSyncPeriod.Duration,
      v1.FinalizerKubernetes,
   )
   go namespaceController.Run(int(ctx.ComponentConfig.NamespaceController.ConcurrentNamespaceSyncs), ctx.Stop)

   return nil, true, nil
}

// NewNamespaceController creates a new NamespaceController
func NewNamespaceController(
	kubeClient clientset.Interface,
	metadataClient metadata.Interface,
	discoverResourcesFn func() ([]*metav1.APIResourceList, error),
	namespaceInformer coreinformers.NamespaceInformer,
	resyncPeriod time.Duration,
	finalizerToken v1.FinalizerName) *NamespaceController {

	// create the controller so we can inject the enqueue function
	namespaceController := &NamespaceController{
		queue:                      workqueue.NewNamedRateLimitingQueue(nsControllerRateLimiter(), "namespace"),
		namespacedResourcesDeleter: deletion.NewNamespacedResourcesDeleter(kubeClient.CoreV1().Namespaces(), metadataClient, kubeClient.CoreV1(), discoverResourcesFn, finalizerToken),
	}

	if kubeClient != nil && kubeClient.CoreV1().RESTClient().GetRateLimiter() != nil {
		ratelimiter.RegisterMetricAndTrackRateLimiterUsage("namespace_controller", kubeClient.CoreV1().RESTClient().GetRateLimiter())
	}

	// configure the namespace informer event handlers
	namespaceInformer.Informer().AddEventHandlerWithResyncPeriod(
		cache.ResourceEventHandlerFuncs{
			AddFunc: func(obj interface{}) {
				namespace := obj.(*v1.Namespace)
				namespaceController.enqueueNamespace(namespace)
			},
			UpdateFunc: func(oldObj, newObj interface{}) {
				namespace := newObj.(*v1.Namespace)
				namespaceController.enqueueNamespace(namespace)
			},
		},
		resyncPeriod,
	)
	namespaceController.lister = namespaceInformer.Lister()
	namespaceController.listerSynced = namespaceInformer.Informer().HasSynced

	return namespaceController
}

// NewNamespacedResourcesDeleter returns a new NamespacedResourcesDeleter.
func NewNamespacedResourcesDeleter(nsClient v1clientset.NamespaceInterface,
	metadataClient metadata.Interface, podsGetter v1clientset.PodsGetter,
	discoverResourcesFn func() ([]*metav1.APIResourceList, error),
	finalizerToken v1.FinalizerName) NamespacedResourcesDeleterInterface {
	d := &namespacedResourcesDeleter{
		nsClient:       nsClient,
		metadataClient: metadataClient,
		podsGetter:     podsGetter,
		opCache: &operationNotSupportedCache{
			m: make(map[operationKey]bool),
		},
		discoverResourcesFn: discoverResourcesFn,
		finalizerToken:      finalizerToken,
	}
    // 初始化缓存，不支持列表、批量删除的资源
	d.initOpCache()
	return d
}
```



初始化缓存，不支持列表、批量删除的资源

```go
func (d *namespacedResourcesDeleter) initOpCache() {
   // pre-fill opCache with the discovery info
   //
   // TODO(sttts): get rid of opCache and http 405 logic around it and trust discovery info
   resources, err := d.discoverResourcesFn()
   if err != nil {
      utilruntime.HandleError(fmt.Errorf("unable to get all supported resources from server: %v", err))
   }
   if len(resources) == 0 {
      klog.Fatalf("Unable to get any supported resources from server: %v", err)
   }

   for _, rl := range resources {
      gv, err := schema.ParseGroupVersion(rl.GroupVersion)
      if err != nil {
         klog.Errorf("Failed to parse GroupVersion %q, skipping: %v", rl.GroupVersion, err)
         continue
      }

      for _, r := range rl.APIResources {
         gvr := schema.GroupVersionResource{Group: gv.Group, Version: gv.Version, Resource: r.Name}
         verbs := sets.NewString([]string(r.Verbs)...)

         if !verbs.Has("delete") {
            klog.V(6).Infof("Skipping resource %v because it cannot be deleted.", gvr)
         }

          // list deletecollection
         for _, op := range []operation{operationList, operationDeleteCollection} {
            if !verbs.Has(string(op)) {
               d.opCache.setNotSupported(operationKey{operation: op, gvr: gvr})
            }
         }
      }
   }
}
```

## 主流程

```go
func (d *namespacedResourcesDeleter) Delete(nsName string) error {
   // Multiple controllers may edit a namespace during termination
   // first get the latest state of the namespace before proceeding
   // if the namespace was deleted already, don't do anything
   namespace, err := d.nsClient.Get(context.TODO(), nsName, metav1.GetOptions{})
   if err != nil {
      if errors.IsNotFound(err) {
         return nil
      }
      return err
   }
   if namespace.DeletionTimestamp == nil {
      return nil
   }

   klog.V(5).Infof("namespace controller - syncNamespace - namespace: %s, finalizerToken: %s", namespace.Name, d.finalizerToken)

   // ensure that the status is up to date on the namespace
   // if we get a not found error, we assume the namespace is truly gone
    // 更新ns状态为终止中
   namespace, err = d.retryOnConflictError(namespace, d.updateNamespaceStatusFunc)
   if err != nil {
      if errors.IsNotFound(err) {
         return nil
      }
      return err
   }

   // the latest view of the namespace asserts that namespace is no longer deleting..
   if namespace.DeletionTimestamp.IsZero() {
      return nil
   }

   // return if it is already finalized. finalized
   if finalized(namespace) {
      return nil
   }

   // there may still be content for us to remove
    // 删除ns下所有gvr主流程
   estimate, err := d.deleteAllContent(namespace)
   if err != nil {
      return err
   }
   if estimate > 0 {
      return &ResourcesRemainingError{estimate}
   }

   // we have removed content, so mark it finalized by us
    // specified finalizerToken and finalizes the namespace
   _, err = d.retryOnConflictError(namespace, d.finalizeNamespace)
   if err != nil {
      // in normal practice, this should not be possible, but if a deployment is running
      // two controllers to do namespace deletion that share a common finalizer token it's
      // possible that a not found could occur since the other controller would have finished the delete.
      if errors.IsNotFound(err) {
         return nil
      }
      return err
   }
   return nil
}
```





```go
// deleteAllContent will use the dynamic client to delete each resource identified in groupVersionResources.
// It returns an estimate of the time remaining before the remaining resources are deleted.
// If estimate > 0, not all resources are guaranteed to be gone.
func (d *namespacedResourcesDeleter) deleteAllContent(ns *v1.Namespace) (int64, error) {
   namespace := ns.Name
   namespaceDeletedAt := *ns.DeletionTimestamp
   var errs []error
   conditionUpdater := namespaceConditionUpdater{}
   estimate := int64(0)
   klog.V(4).Infof("namespace controller - deleteAllContent - namespace: %s", namespace)

   resources, err := d.discoverResourcesFn()
   if err != nil {
      // discovery errors are not fatal.  We often have some set of resources we can operate against even if we don't have a complete list
      errs = append(errs, err)
      conditionUpdater.ProcessDiscoverResourcesErr(err)
   }
   // TODO(sttts): get rid of opCache and pass the verbs (especially "deletecollection") down into the deleter
   deletableResources := discovery.FilteredBy(discovery.SupportsAllVerbs{Verbs: []string{"delete"}}, resources)
   groupVersionResources, err := discovery.GroupVersionResources(deletableResources)
   if err != nil {
      // discovery errors are not fatal.  We often have some set of resources we can operate against even if we don't have a complete list
      errs = append(errs, err)
      conditionUpdater.ProcessGroupVersionErr(err)
   }

   numRemainingTotals := allGVRDeletionMetadata{
      gvrToNumRemaining:        map[schema.GroupVersionResource]int{},
      finalizersToNumRemaining: map[string]int{},
   }
   for gvr := range groupVersionResources {
       // 批量删除或者循环单个删除gvr
      gvrDeletionMetadata, err := d.deleteAllContentForGroupVersionResource(gvr, namespace, namespaceDeletedAt)
      if err != nil {
         // If there is an error, hold on to it but proceed with all the remaining
         // groupVersionResources.
         errs = append(errs, err)
         conditionUpdater.ProcessDeleteContentErr(err)
      }
      if gvrDeletionMetadata.finalizerEstimateSeconds > estimate {
         estimate = gvrDeletionMetadata.finalizerEstimateSeconds
      }
      if gvrDeletionMetadata.numRemaining > 0 {
         numRemainingTotals.gvrToNumRemaining[gvr] = gvrDeletionMetadata.numRemaining
         for finalizer, numRemaining := range gvrDeletionMetadata.finalizersToNumRemaining {
            if numRemaining == 0 {
               continue
            }
            numRemainingTotals.finalizersToNumRemaining[finalizer] = numRemainingTotals.finalizersToNumRemaining[finalizer] + numRemaining
         }
      }
   }
   conditionUpdater.ProcessContentTotals(numRemainingTotals)

   // we always want to update the conditions because if we have set a condition to "it worked" after it was previously, "it didn't work",
   // we need to reflect that information.  Recall that additional finalizers can be set on namespaces, so this finalizer may clear itself and
   // NOT remove the resource instance.
   if hasChanged := conditionUpdater.Update(ns); hasChanged {
      if _, err = d.nsClient.UpdateStatus(context.TODO(), ns, metav1.UpdateOptions{}); err != nil {
         utilruntime.HandleError(fmt.Errorf("couldn't update status condition for namespace %q: %v", namespace, err))
      }
   }

   // if len(errs)==0, NewAggregate returns nil.
   klog.V(4).Infof("namespace controller - deleteAllContent - namespace: %s, estimate: %v, errors: %v", namespace, estimate, utilerrors.NewAggregate(errs))
   return estimate, utilerrors.NewAggregate(errs)
}

// 批量删除或者循环单个删除gvr
// deleteAllContentForGroupVersionResource will use the dynamic client to delete each resource identified in gvr.
// It returns an estimate of the time remaining before the remaining resources are deleted.
// If estimate > 0, not all resources are guaranteed to be gone.
func (d *namespacedResourcesDeleter) deleteAllContentForGroupVersionResource(
	gvr schema.GroupVersionResource, namespace string,
	namespaceDeletedAt metav1.Time) (gvrDeletionMetadata, error) {
	klog.V(5).Infof("namespace controller - deleteAllContentForGroupVersionResource - namespace: %s, gvr: %v", namespace, gvr)

	// estimate how long it will take for the resource to be deleted (needed for objects that support graceful delete)
	estimate, err := d.estimateGracefulTermination(gvr, namespace, namespaceDeletedAt)
	if err != nil {
		klog.V(5).Infof("namespace controller - deleteAllContentForGroupVersionResource - unable to estimate - namespace: %s, gvr: %v, err: %v", namespace, gvr, err)
		return gvrDeletionMetadata{}, err
	}
	klog.V(5).Infof("namespace controller - deleteAllContentForGroupVersionResource - estimate - namespace: %s, gvr: %v, estimate: %v", namespace, gvr, estimate)

	// first try to delete the entire collection
	deleteCollectionSupported, err := d.deleteCollection(gvr, namespace)
	if err != nil {
		return gvrDeletionMetadata{finalizerEstimateSeconds: estimate}, err
	}

	// delete collection was not supported, so we list and delete each item...
	if !deleteCollectionSupported {
		err = d.deleteEachItem(gvr, namespace)
		if err != nil {
			return gvrDeletionMetadata{finalizerEstimateSeconds: estimate}, err
		}
	}

	// verify there are no more remaining items
	// it is not an error condition for there to be remaining items if local estimate is non-zero
	klog.V(5).Infof("namespace controller - deleteAllContentForGroupVersionResource - checking for no more items in namespace: %s, gvr: %v", namespace, gvr)
	unstructuredList, listSupported, err := d.listCollection(gvr, namespace)
	if err != nil {
		klog.V(5).Infof("namespace controller - deleteAllContentForGroupVersionResource - error verifying no items in namespace: %s, gvr: %v, err: %v", namespace, gvr, err)
		return gvrDeletionMetadata{finalizerEstimateSeconds: estimate}, err
	}
	if !listSupported {
		return gvrDeletionMetadata{finalizerEstimateSeconds: estimate}, nil
	}
	klog.V(5).Infof("namespace controller - deleteAllContentForGroupVersionResource - items remaining - namespace: %s, gvr: %v, items: %v", namespace, gvr, len(unstructuredList.Items))
	if len(unstructuredList.Items) == 0 {
		// we're done
		return gvrDeletionMetadata{finalizerEstimateSeconds: 0, numRemaining: 0}, nil
	}

	// use the list to find the finalizers
	finalizersToNumRemaining := map[string]int{}
	for _, item := range unstructuredList.Items {
		for _, finalizer := range item.GetFinalizers() {
			finalizersToNumRemaining[finalizer] = finalizersToNumRemaining[finalizer] + 1
		}
	}

	if estimate != int64(0) {
		klog.V(5).Infof("namespace controller - deleteAllContentForGroupVersionResource - estimate is present - namespace: %s, gvr: %v, finalizers: %v", namespace, gvr, finalizersToNumRemaining)
		return gvrDeletionMetadata{
			finalizerEstimateSeconds: estimate,
			numRemaining:             len(unstructuredList.Items),
			finalizersToNumRemaining: finalizersToNumRemaining,
		}, nil
	}

	// if any item has a finalizer, we treat that as a normal condition, and use a default estimation to allow for GC to complete.
	if len(finalizersToNumRemaining) > 0 {
		klog.V(5).Infof("namespace controller - deleteAllContentForGroupVersionResource - items remaining with finalizers - namespace: %s, gvr: %v, finalizers: %v", namespace, gvr, finalizersToNumRemaining)
		return gvrDeletionMetadata{
			finalizerEstimateSeconds: finalizerEstimateSeconds,
			numRemaining:             len(unstructuredList.Items),
			finalizersToNumRemaining: finalizersToNumRemaining,
		}, nil
	}

	// nothing reported a finalizer, so something was unexpected as it should have been deleted.
	return gvrDeletionMetadata{
		finalizerEstimateSeconds: estimate,
		numRemaining:             len(unstructuredList.Items),
	}, fmt.Errorf("unexpected items still remain in namespace: %s for gvr: %v", namespace, gvr)
}
```





# 有状态服务控制器

`StatefulSet` 类似于 `ReplicaSet`，但是它可以处理 Pod 的启动顺序，为保留每个 Pod 的状态设置唯一标识，具有以下几个功能特性： 

- 稳定的、唯一的网络标识符
- 稳定的、持久化的存储
- 有序的、优雅的部署和缩放
- 有序的、优雅的删除和终止
- 有序的、自动滚动更新

`StatefulSet` 资源清单中和 `volumeMounts` 进行关联的不是 `volumes` 而是一个新的属性：`volumeClaimTemplates`，该属性会自动创建一个 PVC 对象，其实这里就是一个 PVC 的模板，和 Pod 模板类似，PVC 被创建后会自动去关联当前系统中和他合适的 PV 进行绑定。除此之外，还多了一个 `serviceName: "nginx"` 的字段，`serviceName` 就是管理当前 `StatefulSet` 的服务名称，该服务必须在 StatefulSet 之前存在，并且负责该集合的网络标识，Pod 会遵循以下格式获取 DNS/主机名：`pod-specific-string.serviceName.default.svc.cluster.local`，其中 `pod-specific-string` 由 StatefulSet 控制器管理。 

`StatefulSet` 的拓扑结构和其他用于部署的资源对象其实比较类似，比较大的区别在于 `StatefulSet` 引入了 PV 和 PVC 对象来持久存储服务产生的状态，这样所有的服务虽然可以被杀掉或者重启，但是其中的数据由于 PV 的原因不会丢失。 





# 守护服务控制器

DaemonSet 可以保证集群中所有的或者部分的节点都能够运行同一份 Pod 副本，每当有新的节点被加入到集群时，Pod 就会在目标的节点上启动，如果节点被从集群中剔除，节点上的 Pod 也会被垃圾收集器清除；DaemonSet 的作用就像是计算机中的守护进程，它能够运行集群存储、日志收集和监控等『守护进程』，这些服务一般是集群中必备的基础服务。 

# volume控制器

https://rootdeep.github.io/posts/volume-management-part1/

