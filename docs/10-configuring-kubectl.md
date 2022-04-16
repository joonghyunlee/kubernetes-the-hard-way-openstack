# 원격 접근을 위한 kubectl 설정

이 실습에서는 `admin` 사용자 자격 증명을 기반으로 `kubectl`용 kubeconfig 파일을 생성해보겠습니다.

> 이 실습은 앞 서 admin 사용자 자격 증명을 생성했던 디렉토리에서 진행해야 합니다.

## Admin용 Kubernetes 설정 파일

각 kubeconfig 파일에는 연결할 Kubernetes API 서버 주소가 필요합니다. 보통 고가용성을 지원하기 위해 Kubernetes API 서버 앞에 있는 로드밸런서에 연결된 Floating IP 주소가 사용됩니다.

`admin` 사용자를 인증할 수 있는 kubeconfig 파일을 생성해봅시다.

```bash
KUBERNETES_PUBLIC_ADDRESS=$(openstack floating ip list \
  --port `neutron lbaas-loadbalancer-show kubernetes -f value -c vip_port_id` \
  -f value -c 'Floating IP Address')

kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443

kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem

kubectl config set-context kubernetes-the-hard-way \
  --cluster=kubernetes-the-hard-way \
  --user=admin

kubectl config use-context kubernetes-the-hard-way
```

## 검증

원격 Kubernetes 클러스터의 상태를 확인해봅시다.

```bash
kubectl get componentstatuses
```

> 출력

```bash
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
etcd-1               Healthy   {"health":"true"}
etcd-2               Healthy   {"health":"true"}
```

원격 Kubernetes 클러스터의 노드 목록을 조회해봅시다.

```bash
kubectl get nodes
```

> 출력

```
NAME       STATUS   ROLES    AGE     VERSION
worker-0   Ready    <none>   2m30s   v1.18.6
worker-1   Ready    <none>   2m30s   v1.18.6
worker-2   Ready    <none>   2m30s   v1.18.6
```

다음: [Pod Network 라우팅 구성](11-pod-network-routes.md)
