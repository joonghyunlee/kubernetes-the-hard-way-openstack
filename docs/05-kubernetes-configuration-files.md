# 인증을 위한 쿠버네티스 설정 파일 생성

이번 실습에서는 kubeconfigs라고 알려진 [쿠버네티스 설정 파일](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)을 생성하고, 쿠버네티스 클라이언트가 API 서버를 찾아 인증할 수 있도록 합니다.

## 클라이언트 인증 설정

이제 `controller manater`, `kubelet`, `kube-proxy`, 그리고 `scheduler` 클라이언트와 `admin` 사용자 용 kubeconfig 파일을 생성해보겠습니다.



### 쿠버네티스 공인 IP 주소

각 kubeconfig 파일은 연결할 쿠버네티스 API 서버 주소를 필요로 합니다. 고가용성을 지원하기 위해 쿠버네티스 API 서버 앞에 외부 로드밸런서를 두고 IP 주소를 할당하여 사용합니다.

앞서 설정한 로드밸런서의 고정 IP 주소를 찾습니다.

```bash
KUBERNETES_PUBLIC_ADDRESS=$(openstack server show k8sosp.${DOMAIN} -f value -c addresses | awk '{ print $2 }')
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



### The admin Kubernetes Configuration File

Generate a kubeconfig file for the `admin` user:

```
{
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
}
```

Results:

```
admin.kubeconfig
```



## Distribute the Kubernetes Configuration Files

Copy the appropriate `kubelet` and `kube-proxy` kubeconfig files to each worker instance:

```
for instance in worker-0 worker-1 worker-2; do
  gcloud compute scp ${instance}.kubeconfig kube-proxy.kubeconfig ${instance}:~/
done
```

Copy the appropriate `kube-controller-manager` and `kube-scheduler` kubeconfig files to each controller instance:

```
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ${instance}:~/
done
```

Next: [Generating the Data Encryption Config and Key](06-data-encryption-keys.md)
