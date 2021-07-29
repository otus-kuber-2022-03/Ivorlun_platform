# Ivorlun_platform
Ivorlun Platform repository

## Homework 1 (Intro)

### Unhealthy controller-manager and scheduler

After `docker rm -f $(docker ps -a -q)` and following `kubectl delete pod --all -n kube-system`  
I got:  
```
$ k get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                       ERROR
controller-manager   Unhealthy   Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection refused   
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused   
etcd-0               Healthy     {"health":"true"}
```

Удалил кластер, запустил заново начисто, без удалений проверил, что проблема действительно на стороне API.  
Нашёл такое https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.19.md  

```
Kube-apiserver: the componentstatus API is deprecated. This API provided status of etcd, kube-scheduler, and kube-controller-manager components, but only worked when those components were local to the API server, and when kube-scheduler and kube-controller-manager exposed unsecured health endpoints. Instead of this API, etcd health is included in the kube-apiserver health check and kube-scheduler/kube-controller-manager health checks can be made directly against those components' health endpoints. (#93570, @liggitt) [SIG API Machinery, Apps and Cluster Lifecycle]
```

Похоже, что теперь нужно обращаться вот так - https://kubernetes.io/docs/reference/using-api/health-checks/, однако не нашёл как не через curl обратиться к controller manager-у и scheduler-у напрямую за health-check-ом.

```
docker ps --format '{{.Names}}'
k8s_storage-provisioner_storage-provisioner_kube-system_8a83ad84-0320-48d3-a217-3bb817631e2d_3
k8s_kube-proxy_kube-proxy-hkwtf_kube-system_a6d0c436-4af7-4efa-af8b-7b6fc733393f_0
k8s_POD_kube-proxy-hkwtf_kube-system_a6d0c436-4af7-4efa-af8b-7b6fc733393f_0
k8s_coredns_coredns-558bd4d5db-78l77_kube-system_6198f362-4f3f-4b1b-8300-8bf397fa8a92_0
k8s_POD_coredns-558bd4d5db-78l77_kube-system_6198f362-4f3f-4b1b-8300-8bf397fa8a92_0
k8s_POD_storage-provisioner_kube-system_8a83ad84-0320-48d3-a217-3bb817631e2d_2
k8s_kube-controller-manager_kube-controller-manager-minikube_kube-system_a5754fbaabd2854e0e0cdce8400679ea_0
k8s_etcd_etcd-minikube_kube-system_be5cbc7ffcadbd4ffc776526843ee514_0
k8s_kube-apiserver_kube-apiserver-minikube_kube-system_cefbe66f503bf010430ec3521cf31be8_0
k8s_kube-scheduler_kube-scheduler-minikube_kube-system_a2acd1bccd50fd7790183537181f658e_0
k8s_POD_etcd-minikube_kube-system_be5cbc7ffcadbd4ffc776526843ee514_0
k8s_POD_kube-scheduler-minikube_kube-system_a2acd1bccd50fd7790183537181f658e_0
k8s_POD_kube-controller-manager-minikube_kube-system_a5754fbaabd2854e0e0cdce8400679ea_0
k8s_POD_kube-apiserver-minikube_kube-system_cefbe66f503bf010430ec3521cf31be8_0
```

### По каким причинам компоненты восстанавливаются

Полагаю, что поды восстанавливаются непосредственно миникубом, т.е. в моём случае (на Ubuntu по умолчанию докер) kubelet внутри docker-а отвечает за поддержание следующих сущностей:

```
ll manifests/
total 28
drwxr-xr-x 1 root root 4096 Jul 10 13:29 ./
drwxr-xr-x 1 root root 4096 Jul 10 13:29 ../
-rw------- 1 root root 2244 Jul 10 13:29 etcd.yaml
-rw------- 1 root root 4029 Jul 10 13:29 kube-apiserver.yaml
-rw------- 1 root root 3339 Jul 10 13:29 kube-controller-manager.yaml
-rw------- 1 root root 1385 Jul 10 13:29 kube-scheduler.yaml
```
Это можно увидеть посмотрев описание подов:

С помощью `k describe po kube-apiserver-minikube -n kube-system`  
`Controlled By:  Node/minikube`  
Как и все остальные компоненты, кроме    
CoreDNS:  
`k describe -n kube-system po coredns-558bd4d5db-k9lc5`
Где уже контроль происходит репликасетом,   
`Controlled By:  ReplicaSet/coredns-558bd4d5db`
и kube-proxy, который управляется kube-proxy =)  
```
k describe -n kube-system po kube-proxy-89sz9 | grep -i controlled
Controlled By:  DaemonSet/kube-proxy
```

### Pod frontend

Pod frontend не запускается, так как для контейнера не определены начальные переменные необходимые для запуска, что написано в логах и можно найти в исходном коде:  
https://github.com/GoogleCloudPlatform/microservices-demo/blob/v0.2.3/kubernetes-manifests/frontend.yaml


## Homework 2 (Controllers)

Неправильно в примере указан api для конфига kind - должен быть `apiVersion: kind.x-k8s.io/v1alpha4`.

Missing required field "selector" in io.k8s.api.apps.v1.ReplicaSetSpec
В манифесте не хватало обязательного блока с селекотром по лейблам.

ReplicaSet (так же как и устаревший replication controller) не заменяет поды при обновлении шаблона автоматически.  

Официальная документация:  
https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/#deleting-just-a-replicaset  
```
Once the original is deleted, you can create a new ReplicaSet to replace it. As long as the old and new .spec.selector are the same, then the new one will adopt the old Pods. However, it will not make any effort to make existing Pods match a new, different pod template. To update Pods to a new spec in a controlled way, use a Deployment, as ReplicaSets do not support a rolling update directly.
```
### Rollout strategies
Recreate and rolling update.  

Указывается либо в uint либо в процентах:  
* maxSurge - максимальный оверхед подов
* maxUnavailable - понятно =)
#### Blue-green rollout *
```
  strategy:
    type: RollingUpdate
    rollingUpdate: 
      maxSurge: 100%
      maxUnavailable: 0
```


#### Reverse rolling update *

Попробовал применить другой манифест с ревёрс стратегией поверх blue-green той же версии, но, закономерно, ничего не произошло.
```
  strategy:
    type: RollingUpdate
    rollingUpdate: 
      maxSurge: 0
      maxUnavailable: 1
```
Однако, судя по логу, создание нового пода начинается до того как предыдущий был полностью удалён:
```
paymentservice-5f4bb9d75f-9tnlj   1/1     Running   0          4m35s
paymentservice-5f4bb9d75f-ss29x   1/1     Running   0          4m35s
paymentservice-5f4bb9d75f-vlmx7   1/1     Running   0          4m35s
paymentservice-5f4bb9d75f-ss29x   1/1     Terminating   0          8m37s
paymentservice-8477c5cfd4-jl5ct   0/1     Pending       0          0s
paymentservice-8477c5cfd4-jl5ct   0/1     Pending       0          0s
paymentservice-8477c5cfd4-jl5ct   0/1     ContainerCreating   0          0s
paymentservice-8477c5cfd4-jl5ct   1/1     Running             0          1s
paymentservice-5f4bb9d75f-vlmx7   1/1     Terminating         0          8m38s
paymentservice-8477c5cfd4-9cwkh   0/1     Pending             0          0s
paymentservice-8477c5cfd4-9cwkh   0/1     Pending             0          0s
paymentservice-8477c5cfd4-9cwkh   0/1     ContainerCreating   0          0s
paymentservice-8477c5cfd4-9cwkh   1/1     Running             0          2s
paymentservice-5f4bb9d75f-9tnlj   1/1     Terminating         0          8m40s
paymentservice-8477c5cfd4-5rnkj   0/1     Pending             0          0s
paymentservice-8477c5cfd4-5rnkj   0/1     Pending             0          0s
paymentservice-8477c5cfd4-5rnkj   0/1     ContainerCreating   0          0s
paymentservice-8477c5cfd4-5rnkj   1/1     Running             0          2s
paymentservice-5f4bb9d75f-ss29x   0/1     Terminating         0          9m7s
paymentservice-5f4bb9d75f-vlmx7   0/1     Terminating         0          9m9s
paymentservice-5f4bb9d75f-ss29x   0/1     Terminating         0          9m11s
paymentservice-5f4bb9d75f-ss29x   0/1     Terminating         0          9m11s
paymentservice-5f4bb9d75f-9tnlj   0/1     Terminating         0          9m11s
paymentservice-5f4bb9d75f-vlmx7   0/1     Terminating         0          9m21s
paymentservice-5f4bb9d75f-vlmx7   0/1     Terminating         0          9m21s
paymentservice-5f4bb9d75f-9tnlj   0/1     Terminating         0          9m21s
paymentservice-5f4bb9d75f-9tnlj   0/1     Terminating         0          9m21s
```
### Probes
* Liveness - Жив ли контейнер или же нужно его перезапустить. Например приложение запущено, но зависло. 
* Readyness - Готов ли контейнер полностью к работе, можно ли на него роутить трафик.
* Startup - отменяет предыдущие два, до тех пор пока его проверка не пройдёт. Нужно, чтобы другие проверки не перезапустили контейнер до тех пор, пока приложение нормально не начнёт работу. Полезно для медленных приложений, БД и т.п.. 

### DaemonSet *

A DaemonSet ensures that all (or some) Nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to them. As nodes are removed from the cluster, those Pods are garbage collected.

Типичные кейсы использования DaemonSet:
* Сетевые плагины
* Утилиты для сбора и отправки логов (Fluent Bit, Fluentd, etc...)
* Различные утилиты для мониторинга (Node Exporter, etc...)

В миникубе kube-proxy управляется daemonset-ом.

Node exporter daemonset взят отсюда - https://github.com/bibinwilson/kubernetes-node-exporter

### DaemonSet on master **
```
  template:
    spec:
      tolerations:
      # this toleration is to have the daemonset runnable on master nodes
      # remove it if your masters can't run pods
      - key: node-role.kubernetes.io/master
```
https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/#writing-a-daemonset-spec

### Other Info

### Taints and Tolerations
*Node affinity* is a property of Pods that attracts them to a set of nodes (either as a preference or a hard requirement). *Taints* are the opposite -- they allow a node to repel a set of pods.

*Tolerations* are applied to pods, and allow (but do not require) the pods to schedule onto nodes with matching taints.

#### Jobs
A Job creates one or more Pods and will continue to retry execution of the Pods until a specified number of them successfully terminate.

## Homework 3 (Security)

