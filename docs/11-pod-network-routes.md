# Pod Network 라우팅 구성

노드로 스케줄링된 파드는 노드의 파드 CIDR 범위내에서 IP를 하나 할당받게 됩니다. 우리가 구성한 클러스터는 아직 네트워크 라우팅이 설정되어 있지 않아 파드가 다른 노드에서 구동 중인 다른 파드들과 통신할 수 없습니다.

이 실습에서는 네트워크 라우팅을 제어할 수 있도록 [kube-router](https://github.com/cloudnativelabs/kube-router)를 설치해보겠습니다.

> Kubernetes 네트워킹 모델을 구현하는 [다른 방법](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this)도 있습니다.

## 파드간 트래픽 허용

OpenStack의 네트워크는 스푸핑을 방지하기 위해 자신이 알지 못하는 IP에서 온 모든 패킷을 걸러버립니다. 이 때 파드 및 서비스의 CIDR 범위의 IP 역시 버려집니다.

파드 트래픽을 허용하기 위해서는 이들 IP를 허용해주어야 합니다.

```bash
openstack port list -f value -c device_owner -c id | grep compute | awk '{print $2}' | xargs -tI@ openstack port set @ --allowed-address ip-address=10.200.0.0/16 --allowed-address ip-address=10.32.0.0/24
```

## kube-router 설치

각 워커 노드들은 컨트롤러 매니저로부터 고유의 파드 서브넷을 할당받습니다. kube-router는 이를 이용하여 컨테이너 네트워킹을 구성합니다.

> 또는 각 노드에 대해 특정 어노테이션을 추가해서 파드 서브넷을 지정할 수 있습니다.
> ```bash
>  kubectl annotate node/$(hostname -s) "kube-router.io/pod-cidr=<POD_CIDR>"
> ```
> `POD_CIDR`는 `cluster-cidr` 범위 안에 있어야 하고 다른 파드들과 충돌하지 않아야 합니다.

이제 `kube-router` 파드들을 설치해보겠습니다.

```bash
DOMAIN="k8s.nhn" \
CLUSTERCIDR=10.200.0.0/16 \
APISERVER="https://${KUBERNETES_PUBLIC_ADDRESS}:6443" \
sh -c 'curl https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/generic-kuberouter-all-features.yaml -o - | \
sed -e "s;%APISERVER%;$APISERVER;g" -e "s;%CLUSTERCIDR%;$CLUSTERCIDR;g"' | \
kubectl apply -f -
```

> 출력

```
configmap/kube-router-cfg created
daemonset.apps/kube-router created
serviceaccount/kube-router created
clusterrole.rbac.authorization.k8s.io/kube-router created
clusterrolebinding.rbac.authorization.k8s.io/kube-router created
```

**중요:** 어떠한 이유로 kube-router를 재설치하거나 설정을 변경하는 경우, 워커 노드들의 `/var/lib/kube-router/kubeconfig` 파일을 확인하세요. 이 파일들은 kube-router가 설치될 때 초기화되어 삭제되어도 남게 됩니다. 따라서 주소를 변경하게 된다면, 해당 파일이 최신 상태인지 확인해야 합니다.

이제 모든 노드들이 사용가능한 상태가 되었습니다. 한 번 확인해봅시다.

```bash
kubectl get nodes
```

> output

```
NAME               STATUS   ROLES    AGE     VERSION
worker-0.k8s.lan   Ready    <none>   3m55s   v1.18.6
worker-1.k8s.lan   Ready    <none>   3m55s   v1.18.6
worker-2.k8s.lan   Ready    <none>   3m55s   v1.18.6
```

다음: [DNS 클러스터 애드온 설치](12-dns-addon.md)
