# Ivorlun_platform
Ivorlun Platform repository

## Homework 1

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

`kubectl get event --namespace kube-system`

```
129m        Warning   Unhealthy          pod/kube-apiserver-minikube            Readiness probe failed: Get "https://192.168.49.2:8443/readyz": read tcp 192.168.49.2:56718->192.168.49.2:8443: read: connection reset by peer
129m        Warning   Unhealthy          pod/kube-apiserver-minikube            Readiness probe failed: Get "https://192.168.49.2:8443/readyz": dial tcp 192.168.49.2:8443: connect: connection refused
93m         Normal    SandboxChanged     pod/kube-apiserver-minikube            Pod sandbox changed, it will be killed and re-created.
93m         Normal    Pulled             pod/kube-apiserver-minikube            Container image "k8s.gcr.io/kube-apiserver:v1.20.7" already present on machine
93m         Normal    Created            pod/kube-apiserver-minikube            Created container kube-apiserver
93m         Normal    Started            pod/kube-apiserver-minikube            Started container kube-apiserver
93m         Warning   Unhealthy          pod/kube-apiserver-minikube            Startup probe failed: HTTP probe failed with statuscode: 500
```

С помощью `k describe po kube-apiserver-minikube -n kube-system`

Controlled By:  Node/minikube
