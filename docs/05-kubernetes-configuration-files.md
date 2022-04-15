# 인증을 위한 쿠버네티스 설정 파일 생성

이번 실습에서는 kubeconfigs라고 알려진 [쿠버네티스 설정 파일](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)을 생성하고, 쿠버네티스 클라이언트가 API 서버를 찾아 인증할 수 있도록 합니다.

## 클라이언트 인증 설정

이제 `controller manater`, `kubelet`, `kube-proxy`, 그리고 `scheduler` 클라이언트와 `admin` 사용자 용 kubeconfig 파일을 생성해보겠습니다.



### 쿠버네티스 공인 IP 주소

각 kubeconfig 파일은 연결할 쿠버네티스 API 서버 주소를 필요로 합니다. 고가용성을 지원하기 위해 쿠버네티스 API 서버 앞에 외부 로드밸런서를 두고 IP 주소를 할당하여 사용합니다.

앞서 설정한 로드밸런서의 고정 IP 주소를 찾습니다.

```bash
KUBERNETES_PUBLIC_ADDRESS=`openstack floating ip list --port $LB_PORT -f value -c 'Floating IP Address'`
```



### Kubelet용 쿠버네티스 설정 파일

Kubelet용 kubeconfig 파일들을 생성할 때 Kubelet이 설치될 노드 이름과 매칭되는 클라이언트 인증서를 사용해야 합니다. 그래야만 쿠버네티스 [Node Authorizer](https://kubernetes.io/docs/admin/authorization/node/)가 Kubelet들을 승인할 수 있게 됩니다.

> 아래 명령들은 앞서 진행한 [TLS 인증서 생성하기](04-certificate-authority.md) 실습에서 SSL 인증서들을 생성하는데 사용한 것과 같은 디렉토리에서 실행해야 합니다.

각 워커 노드용 kubeconfig 파일을 생성합니다.

```bash
for instance in worker-0 worker-1 worker-2; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```

실행 결과:

```
worker-0.kubeconfig
worker-1.kubeconfig
worker-2.kubeconfig
```



### Kube-proxy용 쿠버네티스 설정 파일

`kube-proxy` 용 kubeconfig 파일을 생성합니다.

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

실행 결과:

```
kube-proxy.kubeconfig
```



### Kube-controller-manager용 쿠버네티스 설정 파일

`kube-controller-manager` 용 kubeconfig 파일을 생성합니다.

```bash
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```

실행 결과:

```
kube-controller-manager.kubeconfig
```



### `kube-scheduler`용 쿠버네티스 설정 파일

`kube-scheduler` 용 kubeconfig 파일을 생성합니다.

```bash
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```

실행 결과:

```
kube-scheduler.kubeconfig
```



### 관리자용 Kubernetes 설정 파일

`admin` 사용자를 위한 kubeconfig 파일을 생성합니다.

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem \
  --embed-certs=true \
  --kubeconfig=admin.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=admin \
  --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig
```

Results:

```
admin.kubeconfig
```



## Kubernetes 설정 파일 배포

`kubelet`과 `kube-proxy` kubeconfig 파일들을 각 Worker 노드 인스턴스로 복사합니다.

```bash
for instance in worker-0 worker-1 worker-2; do
  floating_ip=`openstack server show ${instance}.${DOMAIN} -c addresses -f value | cut -d ' ' -f 3`
  scp -i k8s.node-key.pem ${instance}.kubeconfig kube-proxy.kubeconfig ubuntu@${floating_ip}:~
done
```

`kube-controller-manager`과 `kube-scheduler` kubeconfig 파일들은 Master 노드 인스턴스들로 복사합니다.

```bash
for instance in controller-0 controller-1 controller-2; do
  scp -i k8s.node-key.pem admin.kubeconfig kube-controller-manager.kubeconfig \
    kube-scheduler.kubeconfig ${instance}.${DOMAIN}:~
done
```

Next: [Generating the Data Encryption Config and Key](06-data-encryption-keys.md)
