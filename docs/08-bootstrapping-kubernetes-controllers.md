# Kubernetes 컨트롤러 노드 구성

이번 실습에서는 세 개의 인스턴스에 걸쳐 Kubernetes 마스터 노드를 준비하고 고가용성 구성을 하는 과정을 다룹니다. 이 과정에서 Kubernetes API 서버를 클라이언트에게 노출하기 위한 로드밸런서를 생성합니다. 최종적으로 Kubernetes API Server, Scheduler, 그리고 Controller Manager와 같은 구성요소들이 각 노드들에 설치됩니다.

## 사전 준비

이번 실습의 명령들은 컨트롤러 인스턴스 `controller-0`, `controller-1`, 그리고 `controller-2`에서 각각 실행되어야 합니다. 다음 명령을 통해 각 컨트롤러 인스턴스에 접속합니다.

```bash
ssh controller-0.${DOMAIN}
```

컨트롤러 인스턴스의 `/etc/hosts` 파일에 워커 인스턴스들의 별칭과 IP 주소를 추가합니다.

```
192.168.0.20 worker-0
192.168.0.21 worker-1
192.168.0.22 worker-2
```

## Kubernetes 컨트롤러 노드 구성

먼저 Kubernetes 설정들을 저장하는 디렉토리를 생성합니다.

```bash
sudo mkdir -p /etc/kubernetes/config
```

### Kubernetes 컨트롤러 바이너리 다운로드 및 설치

Kubernetes 공식 배포 바이너리를 내려받습니다.

```bash
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kubectl"
```

Kubernetes 바이너리 설치하기

```bash
chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
```

### Kubernetes API 서버 설정

```bash
sudo mkdir -p /var/lib/kubernetes/

sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
  service-account-key.pem service-account.pem \
  encryption-config.yaml /var/lib/kubernetes/
```

인스턴스 내부 IP 주소는 API 서버를 클러스터 내 나머지 멤버들에게 알려주기 위해 사용됩니다. 내부 IP 주소는 인스턴스 내에서 아래 명령을 통해 확인할 수 있습니다.

```bash
INTERNAL_IP=$(curl -sX GET http://169.254.169.254/latest/meta-data/local-ipv4)
```

`kube-apiserver.service` systemd unit 파일을 생성합니다.

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://192.168.0.10:2379,https://192.168.0.11:2379,https://192.168.0.12:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config='api/all=true' \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Kubernetes Controller Manager 설정

`kube-controller-manager` kubeconfig 파일을 다음과 같이 옮깁니다.

```bash
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

`kube-controller-manager.service` systemd unit 파일을 생성합니다.

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=0.0.0.0 \\
  --allocate-node-cidrs=true \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Kubernetes Scheduler 설정

`kube-scheduler` kubeconfig 파일을 다음과 같이 옮깁니다.

```bash
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

`kube-scheduler.yaml` 설정 파일을 생성합니다.

```bash
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

`kube-scheduler.service` systemd unit 파일을 생성합니다.

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 컨트롤러 서비스 구동

```bash
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

> Kubernetes API 서버가 완전히 초기화되는데 약 30초 정도 소요됩니다.


### 검증

다음 명령을 통해 실행 상태를 확인합니다.

```bash
kubectl get componentstatuses
```

```bash
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-1               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}
```

## Kubelet 승인을 위한 RBAC

이 단락에서 Kubernetes API 서버가 각 워커 노드의 Kubelet API 서버에 접근할 수 있도록 RBAC 권한을 설정합니다. 메트릭 및 로그를 조회하고 파드에서 명령을 실행하려면 Kubelet API에 접근해야 합니다.

> 이 자습서에서는 Kubelet의 `--authorization-mode` 설정을 `Webhook`으로 설정합니다. Webhook 모드는 [SubjectAccessReview](https://kubernetes.io/docs/admin/authorization/#checking-api-access)를 사용하여 권한을 결정합니다.

이 단락에 있는 명령들은 전체 클러스터에 영향을 주므로 컨트롤러 노드 중 한 대에서 한 번만 실행하면 됩니다.

```bash
ssh controller-0.${DOMAIN}
```

Kubelet API에 접근하고 파드 관리와 관련된 가장 일반적인 작업을 수행할 권한이 있는 `system:kube-apiserver-to-kubelet` [ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole)을 생성합니다.

```bash
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

Kubernetes API 서버는 `--kubelet-client-certificate` 옵션으로 정의된 클라이언트 인증서를 사용하여 Kubelet에 `kubernetes` 사용자로 인증합니다. 

이제 `system:kube-apiserver-to-kubelet` ClusterRole을 `kubernetes` 사용자에게 적용합니다.

```bash
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

### 검증

> 이 자습서에서 생성된 인스턴스에서는 이 단원의 내용을 진행할 수 없습니다. 인스턴스를 생성하는데 사용된 장비에서 아래 명령을 실행하도록 합니다.

`kubernetes` 로드밸런서에 연결된 Floating IP를 조회합니다.

```bash
LB_PORT=`neutron lbaas-loadbalancer-show kubernetes -f value -c vip_port_id`
KUBERNETES_PUBLIC_ADDRESS=`openstack floating ip list --port $LB_PORT -f value -c 'Floating IP Address'`
```

Kubernetes 버전을 확인하는 HTTP 요청을 확인차 보내봅니다.

```bash
curl --cacert ca.pem https://${KUBERNETES_PUBLIC_ADDRESS}:6443/version
```

> 출력

```bash
{
  "major": "1",
  "minor": "18",
  "gitVersion": "v1.18.6",
  "gitCommit": "dff82dc0de47299ab66c83c626e08b245ab19037",
  "gitTreeState": "clean",
  "buildDate": "2020-07-15T16:51:04Z",
  "goVersion": "go1.13.9",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

다음: [Kubernetes 워커 노드 구성](09-bootstrapping-kubernetes-workers.md)
