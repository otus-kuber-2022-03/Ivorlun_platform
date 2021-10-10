# Ivorlun_platform
Ivorlun Platform repository

## Homework 2 (Intro)

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


## Homework 3 (Controllers)

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
* Liveness - Жив ли контейнер или же нужно его перезапустить. Например приложение запущено, но зависло. Можно ловить в выводе дедблок, который об этом свидетельствует. 
* Readiness - Готов ли контейнер полностью к работе, можно ли на него роутить трафик. Если под перестаёт быть готов, то его автоматом убирают из списка lb.
* Startup - задерживает предыдущие две, до тех пор пока его проверка не пройдёт. Нужно, чтобы другие probы не перезапустили контейнер пока приложение нормально не начнёт работу. Полезно для медленных приложений, БД и т.п.. 

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
*Node affinity* is a property of Pods that attracts (притягивать) them to a set of nodes (either as a preference or a hard requirement). *Taints* are the opposite -- they allow a node to repel (отталкивать, отражать) a set of pods.

*Tolerations* are applied to pods, and allow (but do not require) the pods to schedule onto nodes with matching taints.

#### Jobs
A Job creates one or more Pods and will continue to retry execution of the Pods until a specified number of them successfully terminate.

## Homework 4 (Security)

### Некоторые важные нюансы  

`Default Service Account` - Создаётся автоматически вместе с namespace-ом, он присваивается новым подам, чтобы они могли обращаться в Kube API.  
Когда создаёшь SA, то для него кубер автоматически создаёт secret, а именно - токен!  

Чтобы использовать RBAC (хороший акроним ККК - кого, как и кто) нужно: 
1. Иметь роль, которая позволяет проводить операции (глаголы) над ресурсами (объектами). Role/ClusterRole.
1. Иметь Субъект (т.е. кто совершает действия). Subjects (users, groups, or service accounts)
1. Связать Роль с Субъектом. RoleBinding/ClusterRoleBinding через roleRef. 

* RoleBinding - привязка внутри одного namespace
* ClusterRoleBinding - на весь кластер

В Binding секция roleRef отвечает за привязку. 


Важно, что в кубере множество механизмов защиты требуют неизменяемости ресурсов.  
Например roleRef:  
```
After you create a binding, you cannot change the Role or ClusterRole that it refers to. If you try to change a binding's roleRef, you get a validation error. If you do want to change the roleRef for a binding, you need to remove the binding object and create a replacement.
``` 
Т.е. как только у binding-а появились субъекты - нельязя менять роли, которые в связке roleRef перечислены.  
Это связано с тем, что 
1. Неизменность roleRef позволяет управлять только списком субъектов, но не менять права, которые им назначены ролью и binding-ом. 
1. Привязка к другой роли (т.е. другим правам для всей общности субъектов) - это фундаментально другой уровень асбракции. Требование пересоздания binding-а, для изменения связи между субъетом и ролью, гарантирует, что всем субъектам нужна новая роль, а не что права лишним субъектам выдадут случайно.

Грубо, это как в линуксе группе lol дать права записи на объект,  

С помощью `kubectl auth reconcile` можно создавать манифесты, которые позволяют пересоздавать привязки, если требуется.

Вопрос - а что тогда с ролью? Её можно менять и это ок, что все субъекты получат другие права на ресурсы?  
Звучит вроде нормально, примерно так, если бы в линуксе группе lol выдали бы права на новую директорию.  

Интересно, что есть ClusterRole, но это совсем не значит, что права будут на весь кластер - можно сделать привязку такой роли в пределах одного namespace.  
https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles

При этом, чтобы дать возможность всем service account-ам одного namespace-а права на доступ ко всему кластеру нужно использовать cluser role binding, но с ограничением `kind: Group name: system:serviceaccounts:namespace`.

Роли можно объединять в общности посредством aggregated clusterroles с помощью лейблов: https://kubernetes.io/docs/reference/access-authn-authz/rbac/#aggregated-clusterroles

### Admission controllers

AC может делать две важные функции:
* Изменять запросы к API (JSON Patch)
* Пропускать или отклонять запросы к API
Каждый контроллер может делать обе вещи, если захочет.  
❗ Но сначала - мутаторы, потом - валидаторы

Например, есть такие:    
NamespaceLifecycle:
* Запрещает создавать новые объекты в удаляемых Namespaces
* Не допускает указания несуществующих Namespaces
* Не дает удалить системные Namespaces

ResourceQuota (ns) ограничивает:
- кол-во объектов
- общий объем ресурсов
- объем дискового пространства для volumes

LimitRanger (ns) Возможность принудительно установить ограничения по ресурсам pod-а.

NodeRestriction - Ограничивает возможности kubelet по редактированию Node и Pod  
ServiceAccount - Автоматически подсовывает в Pod необходимые секреты для функционирования Service Accounts  
Mutating + Validating AdmissionWebhook - Позволяют внешним обработчикам вмешиваться в обработку запросов, идущих через AC  

## Homework 5 (Network)

При попыктке обновить liveness для уже написанного манифеста, kubectl вернул ошибку:  
```
Forbidden: pod updates may not change fields other than `spec.containers[*].image`, `spec.initContainers[*].image`, `spec.activeDeadlineSeconds` or `spec.tolerations`
```
т.е. нельзя обновлять это поле - надо пересоздать под целиком.  

#### Осмысленность ps aux probe
Полагаю, что причина по которой конфигурация вида 
```
livenessProbe:
  exec:
    command:
      - 'sh'
      - '-c'
      - 'ps aux | grep my_web_server_process'
```
бессмысленна в том, что liveness определяет нужно ли перезапуситить контейнер, например из-за зависания приложения. А с учётом того, что процесс приложения вунтри контейнера как правило является основным и, соотвтественно, если контейнер запущен, то существует - описанный подход не поможет определить его истинную работоспособность, а проверка всегда будет успешно проходить.  
Но, скорее всего, имеет некоторый смысл для приложений у которых pid 1 - init, а целевой процесс будет дочерним.

### ClusterIP  
* ClusterIP выделяет IP-адрес для каждого сервиса из особого диапазона (этот адрес виртуален и даже не настраивается на сетевых интерфейсах)
* Когда pod внутри кластера пытается подключиться к виртуальному IP-адресу сервиса, то node, где запущен pod, меняет адрес получателя в сетевых пакетах на настоящий адрес pod-а.
* Нигде в сети, за пределами ноды, виртуальный ClusterIP не существует.

> IP-адрес для каждого сервиса из особого диапазона 
> (этот адрес виртуален и даже не настраивается на сетевых интерфейсах)  

Every node in a Kubernetes cluster runs a kube-proxy. kube-proxy is responsible for implementing a form of virtual IP for Services of type other than ExternalName.

То есть это функционал типа dnat, который заменяет виртуальный ip ClusterIP > Target Pod IP.  
Соответственно с физических нод этот IP не должен пинговаться - он хранится только в цепочках iptables.  

Вот он
```
iptables --list -nv -t nat
 pkts bytes target     prot opt in     out     source               destination  
    1    60 KUBE-MARK-MASQ  tcp  --  *      *      !10.244.0.0/16        10.102.189.119       /* default/web-svc-cip cluster IP */ tcp dpt:80
    1    60 KUBE-SVC-6CZTMAROCN3AQODZ  tcp  --  *      *       0.0.0.0/0            10.102.189.119       /* default/web-svc-cip cluster IP */ tcp dpt:80

```
Вот балансировка между 3мя enpoint-ами:  
```
Chain KUBE-SVC-6CZTMAROCN3AQODZ (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SEP-SLOPQOZW34M3DWKM  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/web-svc-cip */ statistic mode random probability 0.33333333349
    1    60 KUBE-SEP-JXVMOJ4WLIQT6I2K  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/web-svc-cip */ statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-HA42FWOOMUOBT5YR  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/web-svc-cip */

```

SEP - Service Endpoint

**Но!**
В случае работы через ipvs, а не iptables, clusterIP записывается на сетевой интерфейс и перестаёт быть виртуальным адресом и его можно пинговать!  
При этом правила в iptables построены по-другому. Вместо цепочки правил для каждого сервиса, теперь используются хэш-таблицы (ipset). Можно посмотреть их, установив утилиту ipset.  

То есть в iptables хранится минимум - основная инфа по правилам, а в быстрых хэш-таблицах, которые как раз хорошо работают при большом количестве нод - ip-адреса endpoint-ов.

```
iptables --list -nv -t nat
ip addr show kube-ipvs0
ipset list
Name: KUBE-CLUSTER-IP
Type: hash:ip,port
Revision: 5
Header: family inet hashsize 1024 maxelem 65536
Size in memory: 584
References: 2
Number of entries: 8
Members:
10.96.0.10,tcp:53
10.96.0.1,tcp:443
10.102.189.119,tcp:80
10.100.169.199,tcp:80
10.100.114.194,tcp:8000
10.100.124.99,tcp:80
10.96.0.10,tcp:9153
10.96.0.10,udp:53
```

### UDP > TCP localDNS

В localDNS, который располагается на ноде и кеширует соответствие, используется upgrade to tcp (from udp), чтобы располагаясь за NATом запросы не терялись.  

### Kube-proxy vs Calico and etc.

CNI cares about Pod IP.

CNI Plugin is focusing on building up an overlay network, without which Pods can't communicate with each other. The task of the CNI plugin is to assign Pod IP to the Pod when it's scheduled, and to build a virtual device for this IP, and make this IP accessable from every node of the cluster.  

kube-proxy

kube-proxy's job is rather simple, it just redirect requests from Cluster IP to Pod IP.  
kube-proxy has two mode, IPVS and iptables.  
https://stackoverflow.com/a/54881661


Kube-proxy process handles everything related to Services on each node. It ensures that connections to the service cluster IP and port go to a pod that backs the service. If backed by more than one service, kube-proxy load-balances traffic across pods.  
https://docs.projectcalico.org/networking/use-ipvs  
Calico gives you a choice of dataplanes, including a pure Linux eBPF dataplane, a standard Linux networking dataplane, and a Windows HNS dataplane.




### IPVS
IPVS (IP Virtual Server) implements transport-layer load balancing, usually called Layer 4 LAN switching, as part of Linux kernel.

IPVS runs on a host and acts as a load balancer in front of a cluster of real servers. IPVS can direct requests for TCP and UDP-based services to the real servers, and make services of real servers appear as virtual services on a single IP address.

Разница в балансировке сервисов между iptables и ipvs следующая. Допустим 3 пода, на которые может уходить трафик с сервиса:  
* iptables: последовательная цепочка правил 0.33 * ip pod1 > 0.5 * pod2 > pod3 - И эти вероятности выбора пода динамически изменяются в зависимости от масштабирования.
* ipvs: изначально балансер и в него вшит менее топорный механизм балансировки (least load, least connections, locality, weighted, etc. http://www.linuxvirtualserver.org/docs/scheduling.html) и достаточно добавить ip нового пода, а не обновлять правила каждый раз.  
Причём для балансера создаётся виртуальный сетевой интерфейс, на котором будут все адреса подов, которые подходят сервису. 
```
kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default 
    link/ether 2e:f7:f3:b2:35:a4 brd ff:ff:ff:ff:ff:ff
    inet 10.102.189.119/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.100.124.99/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.100.114.194/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.96.0.1/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.96.0.10/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
```
Вместо цепочки правил для каждого сервиса, теперь используются хэш-таблицы (ipset). Можно посмотреть их используя `ipset list`.  
```
Name: KUBE-CLUSTER-IP
Type: hash:ip,port
Revision: 5
Header: family inet hashsize 1024 maxelem 65536
Size in memory: 640
References: 2
Number of entries: 7
Members:
10.102.189.119,tcp:80
10.96.0.10,tcp:53
10.96.0.10,udp:53
10.100.124.99,tcp:80
10.96.0.10,tcp:9153
10.96.0.1,tcp:443
10.100.114.194,tcp:8000
```

### MetalLB  

После зачистки правил и уничтожения куб прокси, почему-то накрылся DNS внутри minikube-а.  
Но, может быть, из-за подключения к VPN-у, который переписал на хосте resolv.conf.

#### Share single ip for several services
```
  annotations:
    metallb.universe.tf/allow-shared-ip: "true"
```

### ARP ликбез  
ARP (address resolution protocol) используется для конвертации IP в MAC, соответственно работает, условно, на L2-канальном уровне, но для L3-сетевого.  
Для этого он отправляет на широковещательный адрес запрос, все участники подсети его получают, и нужный отправляет ответ.  
Важно, что можно узнать MAC-и только подсети, так как работает ниже L3-сетевого уровня и, соответственно, за маршрутизатор не выходит.  
При этом:  
* Участники сети, получив ARP запрос могут сразу составлять свою таблицу, чтобы в дальнейшем знать куда отправлять трафик
* Можно генерить запрос Gratuitous ARP (добровольный запрос) собственного ip, который позволит:  
    * оповестить других об изменении своего ip
    * проверить нет ли коллизий ip-адресов в сети

### Static routes to LB subnet

Если запустить металлб в миникубе, то с основной машины не получится достучаться до него.  
Это потому, что сеть кластера изолирована от основной ОС (а ОС
не знает ничего о подсети для балансировщиков)
Чтобы это поправить, добавим статический маршрут:  
* В реальном окружении это решается добавлением нужной подсети на интерфейс сетевого оборудования
* Или использованием L3-режима (что потребует усилий от сетевиков, но более предпочтительно)

### Ingress  
#### Ingress headless service  
Классная тема вязать ингресс на балансер 
1. Создаём сервис типа LB, который балансирует 80 и 443 в namespace: ingress-nginx, перехватывая трафик ingress-контроллера, выбирая его по селектору
1. Создаём сервис типа ClusterIP, но без clusterIP! (clusterIP: None), который выбирает приложение по селектору
1. Создаём ingress, который проксирует наше приложение, выбирая сервис ClusterIP по backend service name и port.

Из этого получится - единая точка входа, которая балансируется с полными возможностями metallb и openresty nginx-а.  


### Ingress creation code snippet out of date

Сейчас другой синтаксис и доступно в v1, а не v1beta1 версии: https://kubernetes.io/docs/concepts/services-networking/ingress/#the-ingress-resource

### Ingress context path shift to root problem  
```
spec:
  rules:
  - http:
      paths:
      - path: /web
```
Похоже, что ингресс nginx-а не должен переписывать / и сдвигать контекст: т.е. если ингресс имеет endpoint вида
https://ingress/web/index.html, то он не может, просто убрав префикс, заммапить запрос в контейнер, в котором в location / лежит index.html и работает по урлу https://endpoint/index.html. 



### Homework CNI

Документация Calico, кратко и системно излагая, хорошо описывает сетевую систему куба, сервисы и BPF.  
https://docs.projectcalico.org/about/about-k8s-networking

There are lots of different kinds of CNI plugins, but the two main ones are:
  * network plugins, which are responsible for connecting pod to the network
  * IPAM (IP Address Management) plugins, which are responsible for allocating pod IP addresses.
Both options can be provided simultaneously.


