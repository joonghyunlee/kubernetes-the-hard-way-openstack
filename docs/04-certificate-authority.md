# 인증기관(CA) 구성 및 TLS 인증서 생성

이번 실습에서는 CloudFlare의 PKI 도구인 [cfssl](https://github.com/cloudflare/cfssl)를 이용해서 [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure)를 생성합니다. 그 다음 이를 이용하여 인증기관을 구성하고 TLS 인증서를 생성합니다. 생성한 TLS 인증서는 etcd, kube-apiserver, kube-controller-manager, kube-scheduler, kubelet, kube-proxy 등과 같은 컴포넌트들에서 사용됩니다.



## 인증 기관

먼저 TLS 인증서를 생성하기 위한 인증 기관을 생성하겠습니다. 다음 명령어들을 실행하여 CA 설정 파일, 인증서 그리고 비공개 키를 생성합니다.

```bash
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "KR",
      "ST": "Geonggi",
      "L": "Seongnam",
      "O": "NHN",
      "OU": "Kubernetes"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

실행 결과:

```
ca-key.pem
ca.pem
```



## 클라이언트와 서버 인증서

그 다음 쿠버네티스의 각 컴포넌트들을 위한 클라이언트/서버 인증서와 쿠버네티스의 `admin` 사용자를 위한 클라이언트 인증서를 생성합니다.

### Admin 클라이언트 인증서

`admin` 클라이언트 인증서와 사설키를 생성합니다:

```bash
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "KR",
      "ST": "Geonggi",
      "L": "Seongnam",
      "O": "NHN",
      "OU": "Kubernetes"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

실행 결과:

```
admin-key.pem
admin.pem
```

### 쿠버네티스 클라이언트 인증서

쿠버네티스는 [Node Authorizer](https://kubernetes.io/docs/admin/authorization/node/)라는 특수 목적 인증 방식을 사용합니다. 이 방식은 [Kubelet](https://kubernetes.io/docs/concepts/overview/components/#kubelet) 등 쿠버네티스 컴포넌트들이 호출하는 API 요청을 인증하는데 사용됩니다. Node Authorizer에게 인증을 받기 위해서 Kubelet들은 `system:nodes:<nodeName>` 사용자명을 가지고 있어 `system:nodes` 그룹에 속하는지를 나타내는 자격증명을 사용해야 합니다. 이 단계에서는 Node Authorizer의 요구 사항을 만족할 수 있는 쿠버네티스의 각 워커 노드 용 인증서를 생성합니다.

다음 명령을 통해 쿠버네티스의 워커 노드용 인증서와 사설 키를 생성합니다.

```bash
DOMAIN="k8s.nhn"
for instance in worker-0 worker-1 worker-2; do
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "KR",
      "ST": "Geonggi",
      "L": "Seongnam",
      "O": "NHN",
      "OU": "Kubernetes"
    }
  ]
}
EOF

EXTERNAL_IP=$(openstack server show ${instance}.${DOMAIN} -f value -c addresses | awk '{ print $2 }')
INTERNAL_IP=$(openstack server show ${instance}.${DOMAIN} -f value -c addresses | awk -F'[=,]' '{print $2}')

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done
```

실행 결과:

```
worker-0-key.pem
worker-0.pem
worker-1-key.pem
worker-1.pem
worker-2-key.pem
worker-2.pem
```

### Kube Controller Manager 클라이언트 인증서

`kube-controller-manager` 클라이언트 인증서와 사설키를 생성합니다.

```bash
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "KR",
      "ST": "Geonggi",
      "L": "Seongnam",
      "O": "NHN",
      "OU": "Kubernetes"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

실행 결과:

```
kube-controller-manager-key.pem
kube-controller-manager.pem
```


### Kube Proxy 클라이언트 인증서

`kube-proxy` 클라이언트 인증서와 사설키를 생성합니다.

```bash
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "KR",
      "ST": "Geonggi",
      "L": "Seongnam",
      "O": "NHN",
      "OU": "Kubernetes"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

실행 결과:

```
kube-proxy-key.pem
kube-proxy.pem
```

### Scheduler 클라이언트 인증서

`kube-scheduler` 클라이언트 인증서와 사설키를 생성합니다.

```bash
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "KR",
      "ST": "Geonggi",
      "L": "Seongnam",
      "O": "NHN",
      "OU": "Kubernetes"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```

실행 결과:

```
kube-scheduler-key.pem
kube-scheduler.pem
```


### 쿠버네티스 API Server 인증서

쿠버네티스 API 서버 인증서에는 `kubernetes-the-hard-way` 라는 이름을 가진 고정 IP 주소가 대상 이름 목록에 포함됩니다. 이를 통해 원격 클라이언트의 인증서를 검증할 수 있습니다.

`kube-apiserver`  인증서와 사설키를 생성합니다.

```bash
KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "KR",
      "ST": "Geonggi",
      "L": "Seongnam",
      "O": "NHN",
      "OU": "Kubernetes"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  - hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

> 쿠버네티스 API 서버는 자동적으로 `kubernetes` 라는 내부 도메인을 할당받습니다. 이 도메인은 쿠버네티스 내부 클러스터 서비스용으로 예약된 IP 주소 범위(10.32.0.1) 중 첫 번째 주소(10.32.0.1)를 가리키게 됩니다. 이 과정은 이후 [제어 영역 구축 과정](08-bootstrapping-kubernetes-controllers.md#configure-the-kubernetes-api-serve)에서 다루게 됩니다. 

실행 결과:

```
kubernetes-key.pem
kubernetes.pem
```



## 서비스 어카운트 키페어

쿠버네티스 Controller Manager는 키페어를 사용해 [서비스 어카운트 관리하기](https://kubernetes.io/docs/admin/service-accounts-admin/) 문서에서 설명한 대로 서비스 어카운트의 토큰을 생성하고 서명합니다.

`service-account` 인증서와 사설키를 생성합니다.

```bash
cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "KR",
      "ST": "Geonggi",
      "L": "Seongnam",
      "O": "NHN",
      "OU": "Kubernetes"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
```

실행 결과:

```
service-account-key.pem
service-account.pem
```



## 클라이언트와 서버 인증서 배포하기

쿠버네티스 클라이언트 인증서와 사설키들을 각 워커 인스턴스 내부로 복사합니다.

```bash
DOMAIN=k8s.nhn
for instance in worker-0 worker-1 worker-2; do
  floating_ip=`openstack server show ${instance}.${DOMAIN} -c addresses -f value | cut -d ' ' -f 3`
  
  scp -i k8s.node-key.pem ca.pem ${instance}-key.pem ${instance}.pem ubuntu@${floating_ip}:~
done
```

생성한 인증서와 사설키들을 각 마스터 인스턴스 내부로도 복사합니다.

```bash
DOMAIN=k8s.nhn
for instance in controller-0 controller-1 controller-2; do
  scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem ${instance}.${DOMAIN}:
done
```

> 다음 단계에서는 `kube-proxy`, `kube-controller-manager`, `kube-scheduler`, 그리고 `kubelet` 클라이언트 인증서들을 이용하여 클라이언트 인증 설정파일을 생성합니다.

Next: [Generating Kubernetes Configuration Files for Authentication](05-kubernetes-configuration-files.md)
