# Kubernetes 워커 노드 구성

이 실습에서는 세 대의 Kubernetes 워커 노드를 준비하겠습니다. 각 워커 노드에는 [runc](https://github.com/opencontainers/runc), [container networking plugins](https://github.com/containernetworking/cni), [containerd](https://github.com/containerd/containerd), [kubelet](https://kubernetes.io/docs/admin/kubelet), 그리고 [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies) 등이 설치됩니다.

## 사전 준비

이 실습에 나온 명령들은 각 워커 인스턴스 `worker-0`, `worker-1`, 그리고 `worker-2`에서 실행해야 합니다. 다음과 같이 워커 인스턴스에 접속합니다.

```
ssh worker-0.${DOMAIN}
```

## Kubernetes 워커 노드 구성하기

OS 의존성 패키지들을 설치합니다.

```bash
sudo apt-get update
sudo apt-get -y install socat conntrack ipset
```

> socat 바이너리는 `kubectl port-forward` 명령을 지원하는데 필요합니다.

### Swap 비활성화

기본적으로 kubelet는 [스왑](https://help.ubuntu.com/community/SwapFaq)이 활성화되어 있다면 구동에 실패합니다. 그러므로 Kubernetes가 적절한 리소스 할당 기능 및 서비스 품질을 제공할 수 있도록 스왑을 비활성화하는 것을 [권장합니다](https://github.com/kubernetes/kubernetes/issues/7294).

스왑이 활성화되어 있는지 확인해봅니다.

```bash
sudo swapon --show
```

만약 아무 것도 출력되지 않는다면 스왑은 비활성화된 상태입니다. 스왑이 활성화되어 있는 경우 다음 명령을 통해 즉시 비활성화시키도록 합시다.

```bash
sudo swapoff -a
```

> 위 명령은 재부팅하면 그 효과가 사라집니다. 재부팅 후에도 스왑이 비활성화되게 하는 방법은 사용하고 있는 Linux 배포판 문서를 참고하시기 바랍니다.

### Worker 바이너리 다운로드 및 설치

```bash
wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.18.0/crictl-v1.18.0-linux-amd64.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc91/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v0.8.6/cni-plugins-linux-amd64-v0.8.6.tgz \
  https://github.com/containerd/containerd/releases/download/v1.3.6/containerd-1.3.6-linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kubelet
```

설치를 위한 디렉토리를 생성합니다.

```bash
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

워커 바이너리들을 설치합니다.

```bash
mkdir containerd
tar -xvf crictl-v1.18.0-linux-amd64.tar.gz
tar -xvf containerd-1.3.6-linux-amd64.tar.gz -C containerd
sudo tar -xvf cni-plugins-linux-amd64-v0.8.6.tgz -C /opt/cni/bin/
sudo mv runc.amd64 runc
chmod +x crictl kubectl kube-proxy kubelet runc 
sudo mv crictl kubectl kube-proxy kubelet runc /usr/local/bin/
sudo mv containerd/bin/* /bin/
```

### CNI 네트워킹 구성

이 과정은 `kube-router` 설치 과정에서 자동으로 완료된다.

### containerd 구성

`containerd` 설정 디렉토리를 생성합니다.

```bash
sudo mkdir -p /etc/containerd/
```

```bash
cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
EOF
```

`containerd.service` systemd unit 파일을 생성합니다.

```bash
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
```

### Kubelet 구성

```bash
sudo mv ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
sudo mv ca.pem /var/lib/kubernetes/
```

`kubelet-config.yaml` 설정 파일을 생성합니다.

```bash
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}-key.pem"
EOF
```

> `resolvConf` 설정은 `systemd-resolved`을 사용하는 시스템에서 서비스 디스커버리 용도로 CoreDNS를 사용할 때 루프를 방지하기 위해 사용합니다.

`kubelet.service` systemd unit 파일을 생성합니다.

```bash
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Kubernetes Proxy 구성

```bash
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

`kube-proxy-config.yaml` 설정 파일을 생성합니다.

```bash
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```

`kube-proxy.service` systemd unit 파일을 생성합니다.

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Worker 서비스 구동

```bash
sudo systemctl daemon-reload
sudo systemctl enable containerd kubelet kube-proxy
sudo systemctl start containerd kubelet kube-proxy
```

> Remember to run the above commands on each worker node: `worker-0`, `worker-1`, and `worker-2`.

## 검증

> 이 자습서에서 생성된 인스턴스에서는 이 단원의 내용을 진행할 수 없습니다. 인스턴스를 생성하는데 사용된 장비에서 아래 명령을 실행하도록 합니다.

등록된 Kubernetes 노드 목록을 확인합니다.

```bash
ssh controller-0.${DOMAIN} \
  "kubectl get nodes --kubeconfig admin.kubeconfig"
```

> 출력

```
NAME       STATUS   ROLES    AGE   VERSION
worker-0   Ready    <none>   24s   v1.18.6
worker-1   Ready    <none>   24s   v1.18.6
worker-2   Ready    <none>   24s   v1.18.6
```

**주의:** 출력 결과에서 노드들의 상태가 'NotReady'로 되어 있는 것은 아직 CNI 구성이 끝나지 않았기 때문입니다. 이 과정은 이후 "Pod Network Routes" 실습에서 진행됩니다.

Next: [원격 접근을 위한 kubectl 설정](10-configuring-kubectl.md)
