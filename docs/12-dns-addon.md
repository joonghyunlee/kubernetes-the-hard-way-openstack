# DNS 클러스터 애드온 설치

이 실습에서는 Kubernetes 클러스터 내에서 실행되는 애플리케이션을 위해 DNS 기반 서비스 디스커버리 기능을 제공하는 [DNS add-on](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)을 설치해보겠습니다.

## DNS Cluster 애드온

`coredns` 클러스터 애드온을 설치합니다.

```bash
kubectl apply -f https://storage.googleapis.com/kubernetes-the-hard-way/coredns-1.7.0.yaml
```

> 출력

```bash
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
```

`kube-dns` Deployment에 의해 생성된 파드들을 확인해보겠습니다.

```bash
kubectl get pods -l k8s-app=kube-dns -n kube-system
```

> 출력

```bash
NAME                       READY   STATUS    RESTARTS   AGE
coredns-5677dc4cdb-d8rtv   1/1     Running   0          30s
coredns-5677dc4cdb-m8n69   1/1     Running   0          30s
```

## 검증

`busybox` Deployment를 생성해봅시다.

```bash
kubectl run busybox --image=busybox:1.28 --command -- sleep 3600
```

`busybox` Deployment에 의해 생성된 파드 목록을 확인합니다.

```bash
kubectl get pods -l run=busybox
```

> 출력

```bash
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          3s
```

`busybox` 파드의 이름을 확인합니다.

```bash
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

`busybox` 파드 내에서 `kubernetes` 서비스에 대해 DNS 질의를 시도해보도록 합시다.

```bash
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

> 출력

```
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```

Next: [기능 검증](13-smoke-test.md)
