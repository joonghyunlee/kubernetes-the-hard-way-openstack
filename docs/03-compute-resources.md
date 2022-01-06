# 인스턴스 준비하기

쿠버네티스는 컨트롤 플레인(API 서버)가 호스팅되는 노드들과 컨테이너가 궁극적으로 실행되는 워커 노드들이 필요합니다. 이 실습에서는 단일 [가용성 영역](https://docs.toast.com/ko/Compute/Instance/ko/overview/#availability-zone)에서 고가용성 쿠버네티스를 구축할 수 있도록 인스턴스들을 생성해보도록 하겠습니다.



## Networking

쿠버네티스의 [네트워킹 모델](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model)은 컨테이너와 노드들이 서로 통신할 수 있는 flat 네트워크를 사용한다고 가정합니다. 이러한 가정이 여의치 않을 경우 [네트워크 정책](https://kubernetes.io/docs/concepts/services-networking/network-policies/)으로 컨테이너들이 상호간에 통신하는 방법과 외부 네트워크 엔드포인트들과 통신하는 방법을 제한할 수 있습니다.

> 이 자습서는 쿠버네티스의 네트워크 정책에 대해 다루지 않습니다.



### 공용 네트워크

공용 네트워크(Public network)는 Internet 망과 통신을 할 수 있도록 외부에서 접근이 가능한 네트워크를 말합니다. OpenStack의 공용 네트워크는 OpenStack 관리자 또는 서비스 프로바이더에 의해 제공됩니다. 

NHN Cloud에서는 아래 명령을 통해 이미 존재하는 공용 네트워크를 확인할 수 있습니다.

```bash
openstack network list --external -c ID -c Name
```

위 명령의 실행 결과는 다음과 같습니다.

```bash
+--------------------------------------+----------------+
| ID                                   | Name           |
+--------------------------------------+----------------+
| a858742a-245b-41d3-9a05-617e1b069eb9 | Public Network |
+--------------------------------------+----------------+
```



### 사설 테넌트 네트워크

사설 테넌트 네트워크(Private tenant network, 이하 테넌트 네트워크)는 사용자의 테넌트 내에서만 사용할 수 있는 사설 네트워크 입니다. 사설 네트워크에서 외부 인터넷 망으로 연결되려면 Neutron 라우터를 통해 공용 네트워크에 연결되어야 합니다. NHN Cloud의 기본 인프라 서비스에는 서비스 활성화시 기본 네트워크가 생성되기 때문에 이를 사용하도록 하겠습니다.

다음 명령으로 기본 네트워크를 확인합니다.

```bash
. openrc.sh

NETWORK_ID=$(openstack network list --no-share -f value -c ID)
EXT_NETWORK_ID=$(openstack network list  -f value -c ID)
```

기본 네트워크의 서브넷 기본 정보는 다음과 같이 확인합니다. 사용하려는 서브넷은 쿠버네티스의 각 노드에 사설 IP 주소를 부여할 수 있도록 충분히 넓은 대역을 가지고 있어야 합니다. NHN Cloud에서 기본 서브넷의 CIDR는 /24 이므로 이 자습서가 다루는 수준에서는 충분하다고 할 수 있습니다. 

```bash
openstack subnet list --network $NETWORK_ID
```

> CIDR가 `192.168.0.0/24` 인 경우 최대 254개의 IP 주소를 사용할 수 있습니다. 다만 Router 및 DHCP 서버 등이 점유하는 IP가 있어서 사용할 수 있는 IP는 총 252개 입니다.



### 보안 그룹

NHN Cloud에서는 네트워크와 마찬가지로 보안그룹 역시 기본적으로 제공합니다. 이 보안그룹은 같은 그룹내 TCP, UDP 및 ICMP 통신을 허용하도록 되어 있습니다. 기본 보안그룹은 다음과 같이 확인합니다.

```bash
openstack security group list
```

위 명령의 실행 결과는 다음과 같습니다.

```bash
+--------------------------------------+----------+
| ID                                   | Name     |
+--------------------------------------+----------+
| e63abd05-06ed-4ee7-a603-b5cb12180bd3 | default  |
+--------------------------------------+----------+
```

이제 쿠버네티스 노드 사이의 통신을 위한 보안 그룹을 생성합니다. 이 보안 그룹에는 `tcp`, `udp`, `icmp`를 모두 허용하는 규칙을 지정합니다.

```bash
openstack security group create internal
openstack security group rule create --ingress --protocol tcp --remote-group internal internal
openstack security group rule create --ingress --protocol udp --remote-group internal internal
openstack security group rule create --ingress --protocol icmp --remote-group internal internal
```

다음으로 쿠버네티스 클러스터 구축 및 사용을 위해 외부에서 TCP 22, 6443 포트와 ICMP를 열어주도록 합니다. 이를 위해 외부 접근에 대한 규칙을 담은 새로운 보안 그룹을 생성합니다.

```bash
openstack security group create external
```

```bash
openstack security group rule create --ingress --protocol icmp external
openstack security group rule create --ingress --protocol tcp --dst-port 22 external
openstack security group rule create --ingress --protocol tcp --dst-port 6443 external
```

생성한 보안그룹을 보려면 다음과 같이 명령어를 실행합니다.

```bash
openstack security group list -c ID -c Name
```

위 명령의 실행 결과는 다음과 같습니다.

```
+--------------------------------------+----------+
| ID                                   | Name     |
+--------------------------------------+----------+
| 2f04bc01-fe46-4b81-8789-9385def84cfb | internal |
| 7f5115f8-b097-4da4-8500-ff8570be57fd | external |
| e63abd05-06ed-4ee7-a603-b5cb12180bd3 | default  |
+--------------------------------------+----------+
```



### 로드밸런서 생성

서브넷 ID 조회

```bash
SUBNET_ID=`openstack subnet list --network $NETWORK_ID -c ID -f value`
```



```bash
neutron lbaas-loadbalancer-create --name kubernetes --vip-address 192.168.0.9 $SUBNET_ID
neutron lbaas-listener-create --loadbalancer kubernetes --protocol TCP --protocol-port 6443 --name kubernetes
neutron lbaas-pool-create --name kubernetes --loadbalancer kubernetes --protocol TCP --lb-algorithm ROUND_ROBIN --listener kubernetes --member-port 6443
neutron lbaas-healthmonitor-create --name kubernetes --type TCP --pool kubernetes --delay 30 --timeout 5 --max-retries 2 --health-check-port 6443
```



```bash
LB_PORT=`neutron lbaas-loadbalancer-show kubernetes -f value -c vip_port_id`
openstack floating ip create $EXT_NETWORK_ID --port $LB_PORT
KUBERNETES_PUBLIC_ADDRESS=`openstack floating ip list --port $LB_PORT -f value -c 'Floating IP Address'`
```





## 인스턴스

### 인스턴스 이미지

쿠버네티스 노드들이 사용할 OS 이미지를 조회합니다. 이 자습서에서는 [containerd container runtime](https://github.com/containerd/containerd)를 지원하는 [Ubuntu Server](https://www.ubuntu.com/server) 20.04를 OS로 선택해서 인스턴스를 생성합니다. 

```bash
IMAGE_ID=$(openstack image list -f value -c ID --public \
    --property os_distro=Ubuntu --property os_version='Server 20.04 LTS')
```



### 키 페어

쿠버네티스 구축을 위해 각 노드에 접근하기 위한 키 페어를 생성합니다.

```bash
openstack keypair create k8s-node-key > ./k8s.node-key.pem
```



### 쿠버네티스 컨트롤러

이제 쿠버네티스 컨트롤러 노드 3대를 생성합니다. 여기서는 쿠버네티스 구축 과정을 간소화하기 위해 각 인스턴스들의 고정 IP를 지정해서 생성합니다.

```bash
DOMAIN="k8s.nhn"
for i in 0 1 2;
do
  nova boot \
    --nic net-id=$NETWORK_ID,v4-fixed-ip=192.168.0.1${i} \
    --flavor m2.c4m8 \
    --key-name k8s-node-key \
    --security-groups external,internal \
    --block-device source=image,id=$IMAGE_ID,dest=volume,size=100,shutdown=remove,bootindex=0 \
    controller-${i}.${DOMAIN};
  openstack server add floating ip controller-${i}.${DOMAIN} $(openstack floating ip create $EXT_NETWORK_ID -f value -c floating_ip_address)
done
```



### 쿠버네티스 워커

다음으로 쿠버네티스 워커 노드를 생성합니다. 컨트롤러 노드와 마찬가지로 3대를 만들겠습니다.

```bash
for i in 0 1 2;
do
  nova boot \
    --nic net-id=$NETWORK_ID,v4-fixed-ip=192.168.0.2${i} \
    --flavor m2.c4m8 \
    --key-name k8s-node-key \
    --security-groups external,internal \
    --block-device source=image,id=$IMAGE_ID,dest=volume,size=200,shutdown=remove,bootindex=0 \
    worker-${i}.${DOMAIN};
    openstack server add floating ip worker-${i}.${DOMAIN} $(openstack floating ip create $EXT_NETWORK_ID -f value -c floating_ip_address)
done
```

### 생성 확인

이제 다음 명령을 실행하여 생성된 인스턴스들을 확인합니다.

```bash
openstack server list -f table -c Name -c Networks -c Flavor -c Status
```

위 명령의 실행 결과는 다음과 같습니다.

```
+----------------------+--------+------------------------------+---------+
| Name                 | Status | Networks                     | Flavor  |
+----------------------+--------+------------------------------+---------+
| worker-2.k8s.nhn     | ACTIVE | Default Network=192.168.0.22 | m2.c4m8 |
| worker-1.k8s.nhn     | ACTIVE | Default Network=192.168.0.21 | m2.c4m8 |
| worker-0.k8s.nhn     | ACTIVE | Default Network=192.168.0.20 | m2.c4m8 |
| controller-2.k8s.nhn | ACTIVE | Default Network=192.168.0.12 | m2.c4m8 |
| controller-1.k8s.nhn | ACTIVE | Default Network=192.168.0.11 | m2.c4m8 |
| controller-0.k8s.nhn | ACTIVE | Default Network=192.168.0.10 | m2.c4m8 |
+----------------------+--------+------------------------------+---------+
```



## SSH 접속 설정

컨트롤러 및 워커 인스턴스에 접근하기 위해서 SSH 설정을 구성합니다. 인스턴스 생성시 지정한 키페어는 인스턴스에 자동으로 주입되어 비밀번호 없이 접근할 수 있도록 설정됩니다. 아래 명령은 `~/.ssh/known_hosts`에 키를 추가하라는 요청을 통과할 수 있도록 합니다.

```bash
for host in $(openstack server list -f value -c Name); do
  ssh-keyscan -H ${host} >> ~/.ssh/known_hosts
done
```

인스턴스 접속 시 매번 user와 keyfile을 지정하는 것이 불편하다면 아래와 같이 `~/.ssh/config` 파일에 설정을 추가합니다.

```config
Host *.k8s.nhn
    User centos
    IdentityFile ~/.ssh/k8s.node-key.pem
```



Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
