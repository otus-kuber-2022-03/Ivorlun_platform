# Ivorlun_platform
Ivorlun Platform repository

# Homework 2 (Intro)

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


# Homework 3 (Controllers)

Неправильно в примере указан api для конфига kind - должен быть `apiVersion: kind.x-k8s.io/v1alpha4`.

Missing required field "selector" in io.k8s.api.apps.v1.ReplicaSetSpec
В манифесте не хватало обязательного блока с селекотром по лейблам.

ReplicaSet (так же как и устаревший replication controller) не заменяет поды при обновлении шаблона автоматически.  

Официальная документация:  
https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/#deleting-just-a-replicaset  
```
Once the original is deleted, you can create a new ReplicaSet to replace it. As long as the old and new .spec.selector are the same, then the new one will adopt the old Pods. However, it will not make any effort to make existing Pods match a new, different pod template. To update Pods to a new spec in a controlled way, use a Deployment, as ReplicaSets do not support a rolling update directly.
```
## Rollout strategies
Recreate and rolling update.  

Указывается либо в uint либо в процентах:  
* maxSurge - максимальный оверхед подов
* maxUnavailable - понятно =)
### Blue-green rollout *
```
  strategy:
    type: RollingUpdate
    rollingUpdate: 
      maxSurge: 100%
      maxUnavailable: 0
```


### Reverse rolling update *

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
## Probes
* Liveness - Жив ли контейнер или же нужно его перезапустить. Например приложение запущено, но зависло. Можно ловить в выводе дедблок, который об этом свидетельствует. 
* Readiness - Готов ли контейнер полностью к работе, можно ли на него роутить трафик. Если под перестаёт быть готов, то его автоматом убирают из списка lb.
* Startup - задерживает предыдущие две, до тех пор пока его проверка не пройдёт. Нужно, чтобы другие probы не перезапустили контейнер пока приложение нормально не начнёт работу. Полезно для медленных приложений, БД и т.п.. 

## DaemonSet *

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

## Other Info

### Taints and Tolerations
*Node affinity* is a property of Pods that attracts (притягивать) them to a set of nodes (either as a preference or a hard requirement). *Taints* are the opposite -- they allow a node to repel (отталкивать, отражать) a set of pods.

*Tolerations* are applied to pods, and allow (but do not require) the pods to schedule onto nodes with matching taints.

### Jobs
A Job creates one or more Pods and will continue to retry execution of the Pods until a specified number of them successfully terminate.

# Homework 4 (Security)

## Некоторые важные нюансы  
### Default Service Account
`Default Service Account` - Создаётся автоматически вместе с namespace-ом, он присваивается новым подам, чтобы они могли обращаться в Kube API.  
Когда создаёшь SA, то для него кубер автоматически создаёт secret, а именно - токен!  
### RBAC
Чтобы использовать RBAC (хороший акроним ККК - кого, как и кто) нужно: 
1. Иметь роль, которая позволяет проводить операции (глаголы) над ресурсами (объектами). Role/ClusterRole.
1. Иметь Субъект (т.е. кто совершает действия). Subjects (users, groups, or service accounts)
1. Связать Роль с Субъектом. RoleBinding/ClusterRoleBinding через roleRef. 
### RoleBindings
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

### Пересоздание roleref
С помощью `kubectl auth reconcile` можно создавать манифесты, которые позволяют пересоздавать привязки, если требуется.

Вопрос - а что тогда с ролью? Её можно менять и это ок, что все субъекты получат другие права на ресурсы?  
Звучит вроде нормально, примерно так, если бы в линуксе группе lol выдали бы права на новую директорию.  

Интересно, что есть ClusterRole, но это совсем не значит, что права будут на весь кластер - можно сделать привязку такой роли в пределах одного namespace.  
https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles

При этом, чтобы дать возможность всем service account-ам одного namespace-а права на доступ ко всему кластеру нужно использовать cluser role binding, но с ограничением `kind: Group name: system:serviceaccounts:namespace`.

Роли можно объединять в общности посредством aggregated clusterroles с помощью лейблов: https://kubernetes.io/docs/reference/access-authn-authz/rbac/#aggregated-clusterroles

## Admission controllers

AC может делать две важные функции:
* Изменять запросы к API (JSON Patch)
* Пропускать или отклонять запросы к API
Каждый контроллер может делать обе вещи, если захочет.  
❗ Но сначала - мутаторы, потом - валидаторы

Например, есть такие:    
**NamespaceLifecycle**:
* Запрещает создавать новые объекты в удаляемых Namespaces
* Не допускает указания несуществующих Namespaces
* Не дает удалить системные Namespaces

**ResourceQuota** (ns) ограничивает:
- кол-во объектов
- общий объем ресурсов
- объем дискового пространства для volumes

**LimitRanger** (ns) Возможность принудительно установить ограничения по ресурсам pod-а.

NodeRestriction - Ограничивает возможности kubelet по редактированию Node и Pod  
ServiceAccount - Автоматически подсовывает в Pod необходимые секреты для функционирования Service Accounts  
Mutating + Validating AdmissionWebhook - Позволяют внешним обработчикам вмешиваться в обработку запросов, идущих через AC  

# Homework 5 (Network)

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

**SEP** - Service Endpoint

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

### Upgrade UDP > TCP localDNS

В localDNS, который располагается на ноде и кеширует соответствие, используется upgrade to tcp (from udp), чтобы располагаясь за NATом запросы не терялись.  

## Kube-proxy vs CNIs (Calico and etc.)

### CNI cares about Pod IP.

CNI Plugin is focusing on building up an overlay network, without which Pods can't communicate with each other. The task of the CNI plugin is to assign Pod IP to the Pod when it's scheduled, and to build a virtual device for this IP, and make this IP accessable from every node of the cluster.  

### kube-proxy

kube-proxy's job is rather simple, it just redirect requests from Cluster IP to Pod IP.  
kube-proxy has two mode, IPVS and iptables.  
https://stackoverflow.com/a/54881661


Kube-proxy process handles everything related to Services on each node. It ensures that connections to the service cluster IP and port go to a pod that backs the service. If backed by more than one service, kube-proxy load-balances traffic across pods.  
https://docs.projectcalico.org/networking/use-ipvs  
Calico gives you a choice of dataplanes, including a pure Linux eBPF dataplane, a standard Linux networking dataplane, and a Windows HNS dataplane.




## IPVS
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

#### Snippet for cleaning rules and creating route  
```
kubectl get configmap kube-proxy -n kube-system -o yaml | \
  sed -e "s/mode: \"\"/mode: \"ipvs\"/" | \
  kubectl apply -f - -n kube-system
kubectl get configmap kube-proxy -n kube-system -o yaml | \
  sed -e "s/strictARP: false/strictARP: true/" | \
  kubectl apply -f - -n kube-system

kubectl --namespace kube-system delete pod --selector='k8s-app=kube-proxy'

minikube ssh "sudo -i"
sed -i "s/nameserver 192.168.49.1/nameserver 192.168.49.1\nnameserver 1.1.1.1/g" /etc/resolv.conf > /etc/resolv.conf.new
mv /etc/resolv.conf.new /etc/resolv.conf

cat <<EOF >> /tmp/iptables.cleanup
*nat
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
COMMIT
*filter
COMMIT
*mangle
COMMIT
EOF
iptables-restore /tmp/iptables.cleanup

sudo ip route add 172.17.255.0/24 via 192.168.49.2
```

### MetalLB  

После зачистки правил и уничтожения куб прокси, почему-то накрылся DNS внутри minikube-а.  
Но, может быть, из-за подключения к VPN-у, который переписал на хосте resolv.conf.

В общем постоянная проблема в миникубе - приходится изменять внутри контейнера миникуба resolv или изменять cm coredns примерно так https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/#example-1:  
```
apiVersion: v1
data:
  stubDomains: |
        {"abc.com" : ["1.2.3.4"], "my.cluster.local" : ["2.3.4.5"]}
  upstreamNameservers: |
        ["8.8.8.8", "8.8.4.4"]
kind: ConfigMap
```

#### Share single ip for several services
```
  annotations:
    metallb.universe.tf/allow-shared-ip: "true"
```

## ARP ликбез  
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

## Ingress  
### Ingress headless service  
Классная тема вязать ингресс на балансер 
1. Создаём сервис типа LB, который балансирует 80 и 443 в namespace: ingress-nginx, перехватывая трафик ingress-контроллера, выбирая его по селектору
1. Создаём сервис типа ClusterIP, но без clusterIP! (clusterIP: None), который выбирает приложение по селектору https://kubernetes.io/docs/concepts/services-networking/service/#headless-services
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

В общем получилось исправить следующей конфигурацией, наподобие примеру https://github.com/kubernetes/ingress-nginx/blob/main/docs/examples/rewrite/README.md#rewrite-target:  

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /web(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80

```
Суть связки в том, что ЛБ даёт возможность получить доступ извне внутрь кластера и резервирует для этого адрес, а ingress позволяет разделить доступ к разным сервисам, например, по контекстному пути.  
Т.е. если удалить ингресс выше, то доступ к сервису пропадёт, несмотря на то, что балансировщик будет работать:  
```
❯ k delete -f ../web-ingress.yaml 
ingress.networking.k8s.io "web" deleted
$ 
❯ curl -k https://172.17.255.2/web/index.html
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```
### Kuber Dashboard context ingress
Дашборд легко привязался к тому же ингрессу и его эндпоинту, однако важны были аннотации:
```
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
```
Так как первая переключала протокол на безопасный, а **rewrite-target: /$1 позволяла не конфликтовать с существующим префиксом для другого сервиса** - т.е. на одном бэкенде $1, на другом - $2.

## Canary Ingress и как взаимодействуют компоненты при использовании ingress и MetalLB 
Итого важный момент, как это всё взаимодействует в конкретном примере и в принципе.  

Существуют 2 deployment-а в namespace default:
* web - 3 replicas
* web-canary - 2 replicas

Существуют для каждого из них headless ClusterIP service-ы, которые по селектору `label=web` или `label=web-canary` выбирают на какую группу подов обращаться.
* web-svc
* web-canary-svc

Для того, чтобы сделать содержание их index.html доступным по контексту `/web/` (т.е. http://address/web/index.html) используются для каждого ingress-ы, то есть правила маршрутизации трафика извне, в данном случае, на сервисы ClusterIP.  
* web
* web-canary

Причём, ingress canary включается только при наличии в запросе header-а `canary=forsure`:  
```
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$3
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "canary"
    nginx.ingress.kubernetes.io/canary-by-header-value: "forsure"
```

Наконец, есть сервис типа load balancer (то есть metallb, настоящий L4-балансировщик, установленный как отдельный контроллер - 172.17.255.2), который связан с классом ingress-nginx
```
...
kind: Service
spec:
  externalTrafficPolicy: Local
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/component: controller
...
```
и, связывающий все обращения на него, с правилами ingress-ов, которые используют `ingressClassName: nginx`.

То есть, все обращения вида `curl http(s)://172.17.255.2/*` физически попадают на балансировщик, а дальше маршрутизируются согласно правилам ingress-ов на целевые бэкенды.  

В данном примере, все запросы, которые имеют header `canary=forsure`, перенаправляются на поды canary, а те, которые не имеют, на предыдущие.
```
❯ curl -k https://172.17.255.2/web/index.html | tail
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 83783    0 83783    0     0  4812k      0 --:--:-- --:--:-- --:--:-- 5113k
<pre># Kubernetes-managed hosts file.
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
fe00::0	ip6-mcastprefix
fe00::1	ip6-allnodes
fe00::2	ip6-allrouters
172.17.0.3	web-59c5c547b7-q87sq</pre>
</body>
</html>
 
❯ curl -k -H "canary: forsure" https://172.17.255.2/web/index.html | tail
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 83811    0 83811    0     0  5456k      0 --:--:-- --:--:-- --:--:-- 5456k
<pre># Kubernetes-managed hosts file.
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
fe00::0	ip6-mcastprefix
fe00::1	ip6-allnodes
fe00::2	ip6-allrouters
172.17.0.11	web-canary-54cdcc6d8-xd85x</pre>
</body>
</html>
```

### Полезные варианты работы с Canary-аннотациями

Дальше, выбор ограничивается только параметрами запросов - крутой вариант использования, например по языкам или странам:  

```
    nginx.ingress.kubernetes.io/canary-by-header: "Region"
    nginx.ingress.kubernetes.io/canary-by-header-pattern: "cd|sz"
```
Как предлагают тут - https://intl.cloud.tencent.com/document/product/457/38413.  

Возможны и другие аннотации, помимо header-ов:  
https://github.com/kubernetes/ingress-nginx/blob/main/docs/user-guide/nginx-configuration/annotations.md#canary  

Более того самые полезные, имхо, `nginx.ingress.kubernetes.io/canary-weight: "10"`, так как они позволяют без изменения запросов просто бесшовно 10% траффика перенаправлять на выбранную группу подов.
https://mcs.mail.ru/help/ru_RU/cases-bestpractive/k8s-canary  
https://v2-1.docs.kubesphere.io/docs/quick-start/ingress-canary/

## Полезные ссылки
Инструмент автоматизации развёртывания приложений в кубере  и его пример работы с canary и ингрессом:
https://docs.flagger.app/tutorials/nginx-progressive-delivery

```
Flagger implements several deployment strategies (Canary releases, A/B testing, Blue/Green mirroring) using a service mesh (App Mesh, Istio, Linkerd, Open Service Mesh) or an ingress controller (Contour, Gloo, NGINX, Skipper, Traefik) for traffic routing. For release analysis, Flagger can query Prometheus, Datadog, New Relic, CloudWatch or Graphite and for alerting it uses Slack, MS Teams, Discord and Rocket.
```


Примеры работы с ингрессом и сервис для тестов

https://github.com/kubernetes/ingress-nginx/blob/main/docs/examples/http-svc.yaml

# Homework 6 (Volumes and Storages)

## Synopsis  
Данные внутри контейнеров и подов эфемерны, поэтому, чтобы они сохранялись между перезапусками, используется механизм volume-ов, как в docker-е.   
Также часто необходимо иметь возможность 2м контейнерам обращаться к одим и тем же файлам.  
Volume-ы решают обе проблемы.  
## Volumes
**Volume** - абстракция реального хранилища (A directory containing data, accessible to the containers in a pod) 
* Volume создается и удаляется вместе с подом
* Один и тот же Volume может использоваться одновременно несколькими контейнерами в поде
Далее все volumes делятся на 2 вида - volume и persistent.

### subPath
Можно один и тот же вольюм в двух контейнерах, но при этом разбивать его на поддиректории.  
Например все данные приложения для бэкапа хранить в одном вольюме, но по разным маунтпоинтам и путям:  
```

      image: mysql
      env:
      - name: MYSQL_ROOT_PASSWORD
        value: "rootpasswd"
      volumeMounts:
      - mountPath: /var/lib/mysql
        name: site-data
        subPath: mysql
    - name: php
      image: php:7.0-apache
      volumeMounts:
      - mountPath: /var/www/html
        name: site-data
        subPath: html
```
### Volume types  
Их целое множество - cephfs volume, azureFile CSI migration, glusterfs, iscsi, etc.  
Kubernetes supports two volumeModes of PersistentVolumes: Filesystem and Block
#### emptyDir
* Существует пока под запущен
* Изначально пустой каталог на хосте
* Все контейнеры в поде могут читать и записывать внутри файлы, причём монтирование может быть по разным путям
* Данные могут храниться в tmpfs (чревато OOM)
#### hostPath
* Возможность монтировать файл или директорию с хоста
* Часто используется для служебных сервисов 
    * Node Exporter
    * Fluentd/Fluent Bit
    * running cAdvisor in a container; use a hostPath of /sys
    * running a container that needs access to Docker internals; use a hostPath of /var/lib/docker
* Типов монтирования много:  
    * DirectoryOrCreate
    * Directory
    * Socket
    * CharDevice
    * BlockDevice
    * FileOrCreate
    * File
* Кубер не рекомендует, так как очень небезопасно как с точки зрения привилегий, так и с точки зрения разницы сред 
#### downwardAPI  
##### Expose Pod Information to Containers Through Files
There are two ways to expose Pod and Container fields to a running Container:
* Environment variables
* Volume Files  

### projected
A projected volume maps several existing volume sources into the same directory.
```
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: mysecret
          items:
            - key: username
              path: my-group/my-username
      - downwardAPI:
          items:
            - path: "labels"
              fieldRef:
                fieldPath: metadata.labels
            - path: "cpu_limit"
              resourceFieldRef:
                containerName: container-test
                resource: limits.cpu
```
Together, these two ways of exposing Pod and Container fields are called the Downward API.
### local
PV являющийся примонтированным локальным хранилищем - директорией, разделом или диском.
Не поддерживает динамический провижининг.  
Лучше, чем hostpath, так как не нужно явно указывать привзяку подов к ноде - система сама знает куда его назначить.  
То есть это более надёжное и гибкое решение, однако, ограниченное тем, что диск физически привязан к хосту ноды и поломка ноды означает поломку работы пода.  

## Out-of-tree volume plugins
Всё это, конечно, не полный список, а с помощью  Container Storage Interface (CSI) и FlexVolume кто угодно может создавать плагины для хранилищ без необходимости менять код кубера.  

## Container Storage Interface (CSI) 

Defines a standard interface for container orchestration systems (like Kubernetes) to expose arbitrary storage systems to their container workloads.
Once a CSI compatible volume driver is deployed on a Kubernetes cluster, users may use the csi volume type to attach or mount the volumes exposed by the CSI driver.

A csi volume can be used in a Pod in three different ways:

* through a reference to a PersistentVolumeClaim
* with a generic ephemeral volume (alpha feature)
* with a CSI ephemeral volume if the driver supports that (beta feature)

## Persistent Volumes
* Создаются на уровне кластера
* PV похожи на обычные Volume, но имеют отдельный от сервисов жизненный цикл  

Но их уже нельзя просто "объявить" - нужно реализовать привязку нагрузки к PV через PVC.

Отдельно, стоит выделить local volume - так как он привязывается к ноде. https://kubernetes.io/docs/concepts/storage/_print/#local

## persistentVolumeClaim  
Запрос на использование какого-либо PV для POD-а.  
То есть это способ привязки без необходимости углубления в детали конкретной технологии фс и её реализации.

### Claims As Volumes
Вообще PVC это отдельный объект и может объявляться в самостоятельных манифестах, однако возможно объявление прямо в pod.spec:  
```
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```
### Expanding Persistent Volumes Claims  
Поддержка авторасширения pvc доступна с 1.11 и включена по умолчанию, но работает далеко не со всеми storage class.  
You can only expand a PVC if its storage class's allowVolumeExpansion field is set to true.
### CSI Volume expansion  
То же доступно и для CSI - должно поддерживаться целевым драйвером.  
You can only resize volumes containing a file system if the file system is XFS, Ext3, or Ext4.  

### PVC & PV lifecycle
Provisioning > binding > using  
Provisioning - статический (выдали все pv заранее и pvc привязывается к существующим) и динамический (реализуется через default storage class - по запросу pvc кластер сам создаёт необходимый PV под его запрос)  
Для следующих этапов есть разные инструменты защиты от переиспользования и перезаписи.  

### PV Reclaiming 
PV может иметь несколько разных политик переиспользования ресурсов хранилища:
* **Retain** - после удаления PVC, PV переходит в состояние “released”, чтобы переиспользовать ресурс, администратор должен вручную удалить PV, освободить место во внешнем хранилище (удалить данные или сделать их резервную копию)
* **Delete** - (плагин должен поддерживать эту политику) PV удаляется вместе с PVC и высвобождается ресурс во внешнем хранилище
* **Recycle** (deprecated в пользу dynamic provisioning-а) - удаляет все содержимое PV и делает его доступным для использования 

### PV Access Modes
Тома монтируются к кластеру с помощью различных провайдеров, они имеют различные разрешения доступа чтения/записи, PV дает общие для всех провайдеров режимы.  
PV монтируется на хост с одним их трех режимов доступа:  
* **ReadWriteOnce** - **RWO** - только один узел может монтировать том для чтения и записи. ReadWriteOnce может предоставлять доступ нескольким подам, если они запущены на одной node-е.
* **ReadOnlyMany** - **ROX** - несколько узлов могут монтировать том для чтения
* **ReadWriteMany** - **RWX** - несколько узлов могут монтировать том для чтения и записи
* **ReadWriteOncePod** - **RWOP** - Только для единственного pod-а в рамках всего кластера. Поддержка только для CSI k8s 1.22+

### ConfigMap & Secret

Надо отметить, что эти два типа ресурсов так же являются PV.

**СonfigMap** - хранят:  
* конфигурацию приложений
* значения переменных окружения отдельно от конфигурации пода  

**Secret** - хранят чувствительные данные (возможно шифрование содержимого в etcd, но в манифестах - base64)  

You can store secrets in the Kubernetes API and mount them as files for use by pods without coupling to Kubernetes directly. secret volumes are backed by tmpfs (a RAM-backed filesystem) so they are never written to non-volatile storage.

Оба типа функционируют схожим образом:
1. Сначала создаем соответствующий ресурс (ConfigMap, Secret)  
2. В конфигурации пода в описании volumes или переменных окружения ссылаемся на созданный ресурс  

## PVC earning lifecycle 
Стандартный путь:  
1. Создаётся StorageClass, который позволяет привязать реальное хранилище к pv
2. Создаётся PV
3. Создаётся PVC пользователем
4. Кубер находит подходящий под PVC PV
5. Создаётся POD с volume-ом, который ссылается на PVC

Кстати надо будет руками потом подчищать ненужные PV - это место на всякий случай навсегда занимается.
### В какой момент происходит монтирование
1. Kubernetes монтирует сетевой диск на ноду
2. Runtime пробрасывает том в контейнер

## The StorageClass Resource
Описание "классов" различных систем хранения
Разные классы могут использоваться для:
* Произвольных политик (например переиспользования?)
* Динамического provisioning

У каждого StorageClass есть provisioner, который определяет какой плагин используется для работы с PVs. 

### Provisioner  
Для того, чтобы storage class мог физически управлять выданным ему хранилищем существует Provisioner - т.е. код, который непосредственно отправляет ему вызовы.  

### Dynamic Volume Provisioning
Dynamic volume provisioning allows storage volumes to be created on-demand.  
The implementation of dynamic volume provisioning is based on the API object StorageClass from the API group storage.k8s.io.  
A cluster administrator can define as many StorageClass objects as needed, each specifying a volume plugin (aka provisioner) that provisions a volume and the set of parameters to pass to that provisioner when provisioning.
#### Resizing a volume containing a file system
You can only resize volumes containing a file system if the file system is XFS, Ext3, or Ext4 in RWX.

## StatefulSet
PODы в StatefulSet отличаются от других нагрузок:
* Каждый под имеет уникальное состояние (имя, сетевой адрес и volume-ы)
* Для каждого pod-а создается отдельный PVC
* Volume-ы для подов должны создаваться через PersistentVolume
* Удаление/масштабирование подов не удаляет тома, связанные с ними
## Homework part  

### MinIO StatefulSet

Интересный момент, что сначала создаётся PVC, а затем создаётся PV под него в данном случае  
```
❯ k get ev
LAST SEEN   TYPE     REASON                  OBJECT                               MESSAGE
5m21s       Normal   WaitForFirstConsumer    persistentvolumeclaim/data-minio-0   waiting for first consumer to be created before binding
5m21s       Normal   ExternalProvisioning    persistentvolumeclaim/data-minio-0   waiting for a volume to be created, either by external provisioner "rancher.io/local-path" or manually created by system administrator
5m21s       Normal   Provisioning            persistentvolumeclaim/data-minio-0   External provisioner is provisioning volume for claim "default/data-minio-0"
5m19s       Normal   ProvisioningSucceeded   persistentvolumeclaim/data-minio-0   Successfully provisioned volume pvc-f3a65b28-a1ba-4b83-a109-22326015c61f
5m18s       Normal   Scheduled               pod/minio-0                          Successfully assigned default/minio-0 to kind-control-plane
5m17s       Normal   Pulling                 pod/minio-0                          Pulling image "minio/minio:RELEASE.2019-07-10T00-34-56Z"
3m26s       Normal   Pulled                  pod/minio-0                          Successfully pulled image "minio/minio:RELEASE.2019-07-10T00-34-56Z" in 1m51.278097132s
3m26s       Normal   Created                 pod/minio-0                          Created container minio
3m26s       Normal   Started                 pod/minio-0                          Started container minio
5m21s       Normal   SuccessfulCreate        statefulset/minio                    create Claim data-minio-0 Pod minio-0 in StatefulSet minio success
5m21s       Normal   SuccessfulCreate        statefulset/minio                    create Pod minio-0 in StatefulSet minio successful
```
В ДЗ предлагают использовать для просмотра mc, но есть ui, в котором можно работать с бакетами по 9000 порту. Забавно, что он показывает использованное пространство для всего сегмента фс, а не для pvc в 10 гигабайт, который ему по идее должен быть выдан.


То есть порядок действий следующий:  
* Создаётся statefulset с MinIO, у которого есть просто volume, в котором он будет хранить данные
* Вместе с ним создаётся RWO volumeClaim, который запрашивает у дефолтного storage class квоту на pv со storageClass=standard и volumeMode=filesystem, т.е. дефолтные значения  
* А всё потому что в kind-е по умолчанию установлен rancher/local-path-provisioner в одноимённом ns.  
То есть он выдаёт на каждый PVC PV, выделяя целиком (тк лимиты пока игнорирует) `hostPath` диск динамически - очень удобная штука в случае использованя bare-metal и отсутствия network storage.

## Secret  

Данные для секретов записываются в 2х форматах - `data` и `stringData`.  
data - base64 encoded.  


| Builtin Type | Usage |
| --- | --- |
| Opaque (Generic)                      | arbitrary user-defined data |
| kubernetes.io/service-account-token   | service account token |
| kubernetes.io/dockercfg               | serialized ~/.dockercfg file |
| kubernetes.io/dockerconfigjson        | serialized ~/.docker/config.json file |
| kubernetes.io/basic-auth              | credentials for basic authentication |
| kubernetes.io/ssh-auth                | credentials for SSH authentication |
| kubernetes.io/tls                     | data for a TLS client or server |
| bootstrap.kubernetes.io/token         | bootstrap token data |

Secrets могут быть примонтированы как data volumes или как environment variables, чтобы использоваться контейнером в Pod.  

### Immutable Secrets 
    * protects you from accidental (or unwanted) updates that could cause applications outages
    * improves performance of your cluster by significantly reducing load on kube-apiserver, by closing watches for secrets marked as immutable.

### Risks
  * In the API server, secret data is stored in etcd; therefore:
    * Administrators should enable encryption at rest for cluster data (requires v1.13 or later).
    * Administrators should limit access to etcd to admin users.
    * Administrators may want to wipe/shred disks used by etcd when no longer in use.
    * If running etcd in a cluster, administrators should make sure to use SSL/TLS for etcd peer-to-peer communication.
  * If you configure the secret through a manifest (JSON or YAML) file which has the secret data encoded as base64, sharing this file or checking it in to a source repository means the secret is compromised. Base64 encoding is not an encryption method and is considered the same as plain text.
  * Applications still need to protect the value of secret after reading it from the volume, such as not accidentally logging it or transmitting it to an untrusted party.
  * A user who can create a Pod that uses a secret can also see the value of that secret. Even if the API server policy does not allow that user to read the Secret, the user could run a Pod which exposes the secret.


В общем base64 - это норм, если хочется скрыть от беглого взгляда, но в идеале, лучше шифровать.  

Примечательно, что директория, в которой находятся загруженные в бакет файлы, находится буквально за 2 команды:  
```
❯ k describe pv | grep Path
    Type:          HostPath (bare host directory volume)
    Path:          /var/local-path-provisioner/pvc-f3a65b28-a1ba-4b83-a109-22326015c61f_default_data-minio-0
```
и переходя в контейнер kind-а:  
```
root@kind-control-plane:~# ls -lahF /var/local-path-provisioner/pvc-f3a65b28-a1ba-4b83-a109-22326015c61f_default_data-minio-0
total 16K
drwxrwxrwx 4 root root 4.0K Oct 28 23:04 ./
drwxr-xr-x 3 root root 4.0K Oct 24 15:07 ../
drwxr-xr-x 6 root root 4.0K Oct 28 23:04 .minio.sys/
drwxr-xr-x 2 root root 4.0K Oct 28 23:04 123/
```
# Homework 7 (Templating and helm)

## Helm  
Возможности  
* Упаковка нескольких манифестов Kubernetes в пакет - Chart
* Установка пакета в Kubernetes (установленный Chart называется Release)
* Шаблонизация во время установки пакета
* Upgrade (обновления) и Rollback (откаты) установленных пакетов
* Управление зависимостями между пакетами
* Xранение пакетов в удаленных репозиториях

```
example/ 
  Chart.yaml         # описание пакета 
  README.md 
  requirements.yaml  # список зависимостей 
  values.yaml        # переменные
  charts/            # загруженные зависимости 
  templates/         # шаблоны описания ресурсов Kubernetes
```
### Встрооенные переменные

* Release - информация об устанавливаемом release
* Chart - информация о chart, из которого происходит установка
* Files - возможность загружать в шаблон данные из файлов (например, в
* configMap )
* Capabilities - информация о возможностях кластера (например, версия
* Kubernetes)
* Templates - информация о манифесте, из которого был создан ресурс

### Использование публичных чартов, но своих переменных  
`helm install chartmuseum chartmuseum/chartmuseum -f kubernetes-templating/chartmuseum/values.yaml --namespace=chartmuseum --create-namespace`
### Циклы, условия и функции
В основе Helm лежит шаблонизатор Go с 50+ встроенными функциями.  
Например.  
Условия:
``` 
{{- if .Values.server.persistentVolume.enabled }} 
    persistentVolumeClaim: 
      ... 
{{- else }}
```
Циклы:
```
{{- range $key, $value := .Values.server.annotations }} 
  {{ $key: }} {{ $value }} 
{{- end }}
```
### Hooks
Определенные действия, выполняемые в различные моменты
жизненного цикла поставки. Hook, как правило, запускает Job (но это не
обязательно).
Виды hooks:
* `pre/post-install`
* `pre/post-delete`
* `pre/post-upgrade`
* `pre/post-rollback`

### Helm Secrets
* Плагин для Helm
* Механизм удобного* хранения и деплоя секретов для тех, у кого нет HashiCorp Vault
* Реализован поверх другого решения - Mozilla Sops
* Возможность сохранить структуру зашифрованного файла (YAML, JSON)
* Поддержка PGP и KMS (AWS, GCP)

### Best practices  
* Указывайте все используемые в шаблонах переменные в values.yaml, выбирайте адекватные значения по умолчанию
* Используйте команду helm create для генерации структуры своего chart
* Пользуйтесь плагином helm docs для документирования своего chart

https://helm.sh/docs/chart_best_practices/

## Helmfile  
* Надстройка над helm - шаблонизатор шаблонизатора.  
* Управление развертыванием нескольких Helm Charts на нескольких окружениях
* Возможность устанавливать Helm Charts в необходимом порядке
* Больше шаблонизации, в том числе и в values.yaml
* Поддержка различных плагинов (helm-tiller, helm-secret, helm-diﬀ)
* Главное - не увлечься шаблонизацией и сохранять прозрачность решения
```
releases:
- name: prometheus-operator
  chart: stable/prometheus-operator
  version: 6.11.0
  <<: *template
- name: prometheus-telegram-bot
  chart: express42/prometheus-telegram-bot
  version: 0.0.1
  <<: *template
...
    - ./values/{{`{{ .Release.Name }}`}}.yaml.gotmpl
```
## Jsonnet

* Продукт от Google
* Расширение JSON (как YAML - нужно помнить, что любой json представим в виде yaml)
* Любой валидный JSON - валидный Jsonnet (как YAML)
* Полноценный язык программирования* (заточенный под шаблонизацию) (не как YAML)

### Зачем (в большинстве - незачем)

* Для генерации и применения манифестов множества однотипных ресурсов, отличающихся несколькими параметрами
* Если есть ненависть к YAML, многострочным портянкам на YAML и отступам в YAML
* Для генерации YAML и передачи его в другие утилиты (например - kubectl):
* `kubecfg show workers.jsonnet | kubectl apply -f -`

### Kubecfg  
Самый лучший на данный момент тул для работы с Jsonnet-ом  
Общий workﬂow следующий: 
1. Импортируем подготовленную библиотеку с описанием ресурсов
1. Пишем общий для сервисов шаблон
1. Наследуемся от шаблона, указывая конкретные параметры

## Kustomize

* Поддержка встроена в kubectl
* Кастомизация готовых манифестов
* Все больше приложений начинают использовать kustomize как альтернативный вариант поставки (istio, nginx-ingress, etc...)
* Почти как Jsonnet, только YAML (но kustomize - это не templating)
* Нет параметризации при вызове, но можно делать так: `kustomize edit set image ...`

### Общая логика работы:
1. Создаем базовые манифесты ресурсов
1. Создаем файл kustomization.yaml с описанием общих значений
1. Кастомизируем манифесты и применяем их (Можно делать как на лету, так и по очереди)
1. Отлично подходит для labels , environment variables и много чего еще

## cert-manager  
Интересный инструмент, позволяющий автоматизировать работу с сертификатами.  
Причём провайдерами могут выступать как Let’s Encrypt, HashiCorp Vault и Venafi, так и private PKI.  

Туториал для LetsEncrypt + Ingress-Nginx https://cert-manager.io/docs/tutorials/acme/ingress/

Для отладки используется великолепное приложение kuard, которое позволяет получить исчёрпывающую инфу о кластере и работе сети - https://github.com/kubernetes-up-and-running/kuard

Для того, чтобы с помощью let's encrypt-а можно было получить сертификат, нужно выполнить следующее:  

1. Иметь в кластере ingress-контроллер (простейший пример - helm nginx)
2. Установить в кластер сам cert-manager
3. Иметь публичное (доступное для LE) доменное имя или зону, которая будет подтверждать сертификат - пришлось в GCP зарегать google domains
4. Создать ingress для целевого сервиса, в котором будут аннотации на эмитента сертификата и блок с секретом tls:  

```
  annotations:
    kubernetes.io/ingress.class: "nginx"    
    #cert-manager.io/issuer: "letsencrypt-staging"
spec:
  tls:
  - hosts:
    - example.example.com
    secretName: quickstart-example-tls
```
5. Дальше работа идёт с 2мя видами эмитентов LE - staging и prod, потому что можно легко выйти за лимиты попыток подтверждения сертификатов у LE и нужно сначала протестировать корректность всей связки.  

Создаётся 
```
   apiVersion: cert-manager.io/v1
   kind: Issuer
   metadata:
     name: letsencrypt-prod
```
через который на ингресс и делается фактический запрос у LE.  

6. Deploy a TLS Ingress Resource

Если выполнены все требования, то можно делать запрос на TLS сертификат.  
Для этого существуют 2 способа:  
1. Аннотации в ingress через ingress-shim
2. Прямое создание ресурса сертификата  

Проще первый путь, так как простое раскомментирование строки `#cert-manager.io/issuer: "letsencrypt-staging"` позволяет cert-manager
* Создать автоматом сертификат
* Создаст или изменит ingress чтобы использовать его для подтверждения домена (что обычно в html/.well-known/token.html)
* Подтвердить домен
* После того как домен подтверждён и выдан, cert-manager создаст и/или обновит секрет в сертификате

Проверяем состояние выдачи `kubectl describe certificate quickstart-example-tls` и сам секрет серта `kubectl describe secret quickstart-example-tls`  

## ChartMuseum (upload and install chart package)

Для того, чтобы включить загрузку с помощью логина и пароля:  
```
env:
  open:
  DISABLE_API: false
  secret:
    # username for basic http authentication
    BASIC_AUTH_USER:user
    # password for basic http authentication
    BASIC_AUTH_PASS:password
```

Docs - https://chartmuseum.com/docs/#uploading-a-chart-package

### Создание чарта 
Осуществляется командой Helm:
```
cd mychart/
helm package .
```
Также можно загрузить уже готовый чарт для prometheus-оператора:  
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm pull prometheus-community/kube-prometheus-stack -d ./
```
После этого появится файл `kube-prometheus-stack-19.2.3.tgz`

### Загрузка

Осуществляется командой:
`curl -u user:password --data-binary "@kube-prometheus-stack-19.2.3.tgz" https://chartmuseum.34.88.82.193.nip.io/api/charts`  

Если же пакет подписан и для него существует provenance file, нужно его загрузить рядом:
`curl -u user:password --data-binary "@kube-prometheus-stack-19.2.3.tgz.prov" https://chartmuseum.34.88.82.193.nip.io/api/prov`  

Можно загрузить одновременно оба файла используя /api/charts и multipart/form-data тип:
`curl -u user:password -F "chart=@kube-prometheus-stack-19.2.3.tgz" -F "prov=@kube-prometheus-stack-19.2.3.tgz.prov" https://chartmuseum.34.88.82.193.nip.io/api/charts`  

Но можно использовать просто helm-push:

`helm push mychart/ chartmuseum`

### Установка чарта в k8s:  
Add the URL to your ChartMuseum installation to the local repository list:

```
helm repo add chartmuseum-nip https://chartmuseum.34.88.82.193.nip.io --username=user --password=password
"chartmuseum-nip" has been added to your repositories
```
После этого выполняется поиск по чартам музея:  

`helm search repo chartmuseum-nip`

Выбирается нужный и устанавливается:
`helm install chartmuseum-nip/kube-prometheus-stack`

## Harbor

После того, как в кластере развёрнут ingress-nginx и cert-manager через helm, деплоим issuer для автогенерации сертов.  

Далее проходим шаги по установке самого harbor    
https://github.com/goharbor/harbor-helm/tree/v1.1.2#installation  

Однако в процессе деплоя выясняется, что харбор пытается подтянуть secret ca, который не генериться cert-manager-ом: 

```
Events:
  Type     Reason       Age                  From               Message
  ----     ------       ----                 ----               -------
  Normal   Scheduled    4m18s                default-scheduler  Successfully assigned harbor/harbor-harbor-core-c6f66f696-r5qck to gke-templating-default-pool-9ed53396-gxp5
  Warning  FailedMount  2m15s                kubelet            Unable to attach or mount volumes: unmounted volumes=[ca-download], unattached volumes=[token-service-private-key ca-download psc kube-api-access-hpcjr config secret-key]: timed out waiting for the condition
  Warning  FailedMount  8s (x10 over 4m18s)  kubelet            MountVolume.SetUp failed for volume "ca-download" : references non-existent secret key: ca.crt
```

Как описано в этой issue:  
https://github.com/goharbor/harbor-helm/issues/229#issuecomment-788519847.  
Решилось апдейтом на версию 1.5.4 и `expose.tls.certSource: secret`, как показывает влитый фикс выше.  

Snippet for fast cluster config:  

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx && helm repo update && helm install nginx-ingress ingress-nginx/ingress-nginx --namespace nginx-ingress --create-namespace
k get -n nginx-ingress svc nginx-ingress-ingress-nginx-controller -o jsonpath="{.status.loadBalancer.ingress..ip}"
helm repo add jetstack https://charts.jetstack.io && helm repo update && helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.6.0 --set installCRDs=true
k apply -f kubernetes-templating/harbor/issuer.yaml
helm repo add harbor https://helm.goharbor.io && helm repo update && helm install harbor harbor/harbor --wait --namespace=harbor --create-namespace -f kubernetes-templating/harbor/values.yaml --version v1.5.4
```

## Helmfile 

* Докуметация:  https://github.com/roboll/helmfile/tree/v0.142.0
* Best practices: https://github.com/roboll/helmfile/blob/master/docs/writing-helmfile.md

Если положить манифест в директорию и директорию указать как чарт, то можно с помощью хелмфайла накатывать одиночные манифесты:  
https://github.com/roboll/helmfile/pull/673

Интересно, что в качестве источника хелм чарта можно использовать даже гит с указанием ветки

`docker run --rm --net=host -v "${HOME}/.kube:/root/.kube" -v "${HOME}/.config/helm:/root/.config/helm" -v "${PWD}:/wd" --workdir /wd quay.io/roboll/helmfile:helm3-v0.142.0 helmfile sync`

`helmfile apply`

## Helm 

Chart - пакет, включающий
* Метаданные
* Шаблоны описания ресурсов Kubernetes
* Конфигурация установки (values.yaml)
* Документация
* Список зависимостей

Release
* Установленный в Kubernetes Chart
* Хранятся в configMaps и Secrets
* Chart + Values = Release
* 1 Upgrade = 1 Release

A chart can be either an 'application' or a 'library' chart.
* Application charts are a collection of templates that can be packaged into versioned archives to be deployed.
* Library charts provide useful utilities or functions for the chart developer. They're included as a dependency of application charts to inject those utilities and functions into the rendering pipeline. Library charts do not define any templates and therefore cannot be deployed.

Работает Helm 3 следующим образом:

1. Получает на вход Chart (локально или из репозитория, при этом чарты могут использовать друг друга) и генерирует манифест релиза.
1. Получает текст предыдущего релиза.
1. Получает текущее стостояние примитивов из namespace-релиза.
1. Сравнивает эти три вещи, делает patch и передает его в KubeAPI.
1. Дожидается выката релиза (опциональный шаг).

Эта схема называется 3-way merge. Таким образом Helm приведет конфигурацию приложения к состоянию, которое описано в git, но не тронет другие изменения. Т. е., если у вас в кластере есть какая-то сущность, которая трансформирует ваши примитивы (например, Service Mesh), то Helm отнесется к ним бережно.

У чарта могут быть тесты - например connection test после установки.  
А так же постинсталл инфа формируется через NOTES.txt

“.” означает текущую область значений (current scope), далее идет зарезервированное слово Values и путь до ключа. При рендере релиза сюда подставится значение, которое было определено по этому пути.

Следующая аннотация позволяет развертывать новые развертывания (Deployments) при изменении ConfigMap:
```
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```
Отказ от удаления ресурсов с помощью политик ресурсов (PVC for example):
```
metadata:
  annotations:
    "helm.sh/resource-policy": keep
```
Плагин helm-secrets предлагает секретное управление и защиту вашей важной информации. Он делегирует секретное шифрование Mozilla SOPS

Виды hooks:
* pre/post-install
* pre/post-delete
* pre/post-upgrade
* pre/post-rollback

Для того, чтобы переопределить values для subchart-а необходимо в основном файле values прописать секцию с именем зависимости и в ней переопределить значения.
### Создаем свой helm chart | Задание со ⭐  
Реализовал задание с зависимостью не через requirements.yaml, а через dependencies, так как 1й вариант является устаревшим:  
https://helm.sh/docs/faq/changes_since_helm2/#consolidation-of-requirementsyaml-into-chartyaml  
> The apiVersion field should be v2 for Helm charts that require at least Helm 3. Charts supporting previous Helm versions have an apiVersion set to v1 and are still installable by Helm 3.

> Changes from v1 to v2:
> * A dependencies field defining chart dependencies, which were located in a separate requirements.yaml file for v1 charts (see Chart Dependencies).
То есть в Chart.yaml объявляем:  
```
dependencies:
  - name: frontend
    version: 0.1.0
    repository: "file://../frontend"
  - name: redis
    repository: https://charts.bitnami.com/bitnami
    version: 15.6.4
```
Далее смотрим на манифест hipster-shop и https://artifacthub.io/packages/helm/bitnami/redis,  
и переопределяем необходимые значения в values.yaml самого hipster-shop, чтобы чарт работал в связке с приложением.  
```
redis: 
  fullnameOverride: "redis-cart"
  nameOverride: "redis-cart"
  commonLabels: 
    app: "redis-cart"
  architecture: "standalone"
  auth:
    enabled: false
```
Так как нет возможности изменить имя сервиса в values helm chart redis, а только его префикс, то в манифесте hipster-shop пришлось сменить:  
`REDIS_ADDR` с `redis-cart` на `redis-cart-master`  

## Helm Secrets (Mozilla SOPS with gpg)  
Помимо PGP есть поддержка KMS (AWS, GCP) и реализует возможность сохранить структуру зашифрованного файла (YAML, JSON).  

Для работы требуются утилиты GPG, Mozilla SOPS https://github.com/mozilla/sops и плагин helm-secrets https://github.com/jkroepke/helm-secrets.  
Создаём ключ для подписи  
`gpg --full-generate-key`  
Создадим в корне директории чарта новый файл secrets.yaml со значениями, которые надо будет шифровать.  
Шифруем его  
`sops -e -i --pgp <$ID> secrets.yaml`  
Можно посмотреть содержание через  
`helm secrets view secrets.yaml` или `sops -d secrets.yaml`  
Создаём secrets.yaml уже в директории templates и помещаем туда значения  
```
type: Opaque
data:
 visibleKey: {{ .Values.visibleKey | b64enc | quote }}
```
Далее можно работать через команду  
`helm secrets upgrade --install`

## "Sealed Secrets" for Kubernetes by bitnami 
https://github.com/bitnami-labs/sealed-secrets  
Encrypt your Secret into a SealedSecret, which is safe to store - even to a public repository. The SealedSecret can be decrypted only by the controller running in the target cluster and nobody else (not even the original author) is able to obtain the original Secret from the SealedSecret.

### Предложите способ использования плагина helm-secrets в CI/CD  
Простейшим способом кажется добавление gpg-ключа в secret variables или аналогичные системы.  
Также можно записать ключ в образ, который будет производить работу с helm в CI.   

helm repo add templating <Ссылка на ваш репозиторий> будет работать только после того как изменения попадут в мастер ветку.  

## JSONnet Kubecfg (язык шаблонов для json)  
Так как используемая в домашнем задании библиотека устарела и для деплоймента имела версию kube api apps/v1beta2, пришлось её обновить до коммита  
https://github.com/bitnami-labs/kube-libsonnet/raw/96b30825c33b7286894c095be19b7b90687b1ede/kube.libsonnet  

Позволяет делать шаблоны манифестов в виде json-шаблонов используя библиотеку для интерпретации, например - libsonnet.  

# Homework 21 (CNI)

Документация Calico, кратко и системно излагая, хорошо описывает сетевую систему куба, сервисы и BPF.  
https://docs.projectcalico.org/about/about-k8s-networking

There are lots of different kinds of CNI plugins, but the two main ones are:
  * network plugins, which are responsible for connecting pod to the network
  * IPAM (IP Address Management) plugins, which are responsible for allocating pod IP addresses.
Both options can be provided simultaneously.

# Homework 22 (CSI)  

Вообще у CSI куча преимуществ против стандартных PV.

### Volume Snapshot and Restore  
A VolumeSnapshot**Content** is a snapshot taken from a volume in the cluster that has been provisioned by an administrator. It is a resource in the cluster just like a PersistentVolume is a cluster resource.

A VolumeSnapshot is a request for snapshot of a volume by a user. It is similar to a PersistentVolumeClaim.  

Volume snapshots provide Kubernetes users with a standardized way to copy a volume's contents at a particular point in time without creating an entirely new volume. 
### Volume Cloning
The CSI Volume Cloning feature adds support for specifying existing PVCs in the dataSource field to indicate a user would like to clone a Volume.  
### Volume populators and data sources  
Штука, которая заполняет данными PV из другого PV?  
Volume populators are controllers that can create non-empty volumes, where the contents of the volume are determined by a Custom Resource. Users create a populated volume by referring to a Custom Resource using the dataSourceRef field

### Volume Health Monitoring

CSI volume health monitoring allows CSI Drivers to detect abnormal volume conditions from the underlying storage systems and report them as events on PVCs or Pods.