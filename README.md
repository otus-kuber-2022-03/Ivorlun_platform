# Ivorlun_platform
Ivorlun Platform repository

## Homework 1

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


## Homework 2
