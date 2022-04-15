# etcd 클러스터 구성하기

Kubernetes 컴포넌트들은 클러스터 상태 정보를 [etcd](https://github.com/etcd-io/etcd)에 저장합니다. 이번 실습에서는 고가용성 및 안전한 원격 접근을 위해 3개 노드로 구성된 etcd 클러스터를 구축해보겠습니다.

## 사전 준비

이번 실습에서 진행할 내용들은 컨트롤러 인스턴스 `controller-0.k8s.nhn`, `controller-1.k8s.nhn`, 그리고 `controller-2.k8s.nhn`에서 각각 실행해야 합니다. 다음 명령을 실행하여 컨트롤러 인스턴스에 접근해 봅시다.

```
ssh controller-0.k8s.nhn
```

### tmux를 활용한 병렬 작업

[tmux](https://github.com/tmux/tmux/wiki)는 동시에 여러 인스턴스에서 명령을 실행할 수 있게 도와주는 도구 입니다. 실습 전 사전 준비 장에 있는 [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux) 항목을 참고하세요.

## etcd 클러스터 멤버 준비하기

### etcd 바이너리 내려받기 및 설치하기

[etcd](https://github.com/etcd-io/etcd) GitHub 프로젝트 페이지에서 etcd 공식 바이너리 배포 파일을 내려받도록 합니다.

```bash
wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.4.10/etcd-v3.4.10-linux-amd64.tar.gz"
```

내려받은 파일의 압축을 풀고 `etcd` 서버와 `etcdctl` 명령줄 도구를 설치합니다.

```bash
tar -xvf etcd-v3.4.10-linux-amd64.tar.gz
sudo mv etcd-v3.4.10-linux-amd64/etcd* /usr/local/bin/
```

### etcd 서버 설정하기

우선 다음 명령을 실행합시다.

```bash
sudo mkdir -p /etc/etcd /var/lib/etcd
sudo chmod 700 /var/lib/etcd
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```

인스턴스의 내부 IP주소는 클라이언트 요청을 처리하고 다른 etcd 클러스터 형제 노드들과 통신하는데 사용합니다. 내부 IP 주소는 인스턴스 내에서 아래 명령을 통해 확인할 수 있습니다.

```bash
INTERNAL_IP=$(curl -sX GET http://169.254.169.254/latest/meta-data/local-ipv4)
```

각 etcd 멤버들은 클러스터 내에서 고유한 이름을 가져야 합니다. 이 실습에서는 인스턴스의 호스트명과 일치하도록 etcd 이름을 설정합시다.

```
ETCD_NAME=$(hostname -s)
```

다음으로 `etcd.service` systemd unit 파일을 생성합니다.

```bash
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://192.168.0.10:2380,controller-1=https://192.168.0.11:2380,controller-2=https://192.168.0.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### etcd 서버 시작

```bash
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

> 지금까지 나온 명령은 각 컨트롤러 노드 `controller-0`, `controller-1`, 그리고 `controller-2`에서 각각 실행해야 한다.

## 검증

다음과 같이 etcd 클러스터 멤버들을 조회합니다.

```bash
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

> 출력

```bash
27005c80257589, started, controller-1, https://192.168.0.11:2380, https://192.168.0.11:2379, false
dc6a42cb0de5d24c, started, controller-2, https://192.168.0.12:2380, https://192.168.0.12:2379, false
f6cc087b9f0b9fa3, started, controller-0, https://192.168.0.10:2380, https://192.168.0.10:2379, false
```

Next: [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)
