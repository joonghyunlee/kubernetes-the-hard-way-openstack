# 기능 검증

이 실습에서는 여러분들이 설치한 Kubernetes 클러스터가 정확히 동작하는지 확인하는 과정을 진행하겠습니다.

## 데이터 암호화

이 단원에서는 [저장 시 secret 데이터 암호화](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted) 기능을 검증해보겠습니다.

Generic Secret을 하나 생성합니다.

```bash
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

Print a hexdump of the `kubernetes-the-hard-way` secret stored in etcd:

etcd에 저장된 `kubernetes-the-hard-way` Secretd의 덤프를 가져옵니다.

```bash
ssh controller-0.${DOMAIN} \
  "sudo ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem\
  /registry/secrets/default/kubernetes-the-hard-way | hexdump -C"
```

> 출력

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a 8c 7b 16 f3 26 59 d5  |:v1:key1:.{..&Y.|
00000050  c9 65 1c f0 3a 04 e7 66  2a f6 50 93 4e d4 d7 8c  |.e..:..f*.P.N...|
00000060  ca 24 ab 68 54 5f 31 f6  5c e5 5c c6 29 1d cc da  |.$.hT_1.\.\.)...|
00000070  22 fc c9 be 23 8a 26 b4  9b 38 1d 57 65 87 2a ac  |"...#.&..8.We.*.|
00000080  70 11 ea 06 93 b7 de ba  12 83 42 94 9d 27 8f ee  |p.........B..'..|
00000090  95 05 b0 77 31 ab 66 3d  d9 e2 38 85 f9 a5 59 3a  |...w1.f=..8...Y:|
000000a0  90 c1 46 ae b4 9d 13 05  82 58 71 4e 5b cb ac e2  |..F......XqN[...|
000000b0  3b 6e d7 10 ab 7c fc fe  dd f0 e6 0a 7b 24 2e 68  |;n...|......{$.h|
000000c0  5e 78 98 5f 33 40 f8 d2  10 30 1f de 17 3f 06 a1  |^x._3@...0...?..|
000000d0  81 bd 1f 2e be e9 35 26  2c be 39 16 cf ac c2 6d  |......5&,.9....m|
000000e0  32 56 05 7d 80 39 5d c0  a4 43 46 75 96 0c 87 49  |2V.}.9]..CFu...I|
000000f0  3c 17 1a 1c 8e 52 b1 e8  42 6b a5 e8 b2 b3 27 bc  |<....R..Bk....'.|
00000100  80 a6 53 2a 9f 57 d2 de  a3 f8 7f 84 2c 01 c9 d9  |..S*.W......,...|
00000110  4f e0 3f e7 a7 1e 46 b7  47 dc f0 53 d2 d2 e1 99  |O.?...F.G..S....|
00000120  0b b7 b3 49 d0 3c a5 e8  26 ce 2c 51 42 2c 0f 48  |...I.<..&.,QB,.H|
00000130  b1 9a 1a dd 24 d1 06 d8  34 bf 09 2e 20 cc 3d 3d  |....$...4... .==|
00000140  e2 5a e5 e4 44 b7 ae 57  49 0a                    |.Z..D..WI.|
0000014a
```

etcd 키는 `k8s:enc:aescbc:v1:key1` 접두사를 사용해야 합니다. 이는 `aescbc` 공급자가 `key` 암호화 키로 데이터를 암호화하는데 사용되었음을 나타냅니다.

## Deployment

이 단원에서는 [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)를 생성하고 관리하는 기능을 검증합니다.

[nginx](https://nginx.org/en/) 웹 서버용 Deployment를 생성합니다.

```bash
kubectl create deployment nginx --image=nginx
```

`nginx` Deployment에 의해 생성된 파드 목록을 가져옵니다.

```bash
kubectl get pods -l app=nginx
```

> 출력

```
NAME                    READY   STATUS    RESTARTS   AGE
nginx-f89759699-kpn5m   1/1     Running   0          10s
```

### 포트 포워딩

이 단원에서는 [포트 포워딩](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)을 사용해 어플리케이션에 원격으로 접근하는 기능을 검증합니다.

`nginx` 파드의 이름을 가져옵니다.

```bash
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
```

로컬 장비의 `8080` 포트를 `nginx` 파드의 `80` 포트로 포워딩합니다.

```bash
kubectl port-forward $POD_NAME 8080:80
```

> 출력

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

새로운 터미널 창을 열고 포워딩하는 주소로 HTTP 요청을 보내봅니다.

```bash
curl --head http://127.0.0.1:8080
```

> 출력

```
HTTP/1.1 200 OK
Server: nginx/1.21.6
Date: Sat, 16 Apr 2022 12:11:34 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 25 Jan 2022 15:03:52 GMT
Connection: keep-alive
ETag: "61f01158-267"
Accept-Ranges: bytes
```

이전 터미널 창으로 돌아가 `nginx` 파드로의 포트 포워딩을 중단합니다.

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

### 로그

이 단원에서는 [컨테이너 로그 보기](https://kubernetes.io/docs/concepts/cluster-administration/logging/) 기능을 검증하겠습니다.

`nginx` 파드의 로그를 출력해봅시다.

```bash
kubectl logs $POD_NAME
```

> 출력

```
...
127.0.0.1 - - [16/Apr/2022:12:11:34 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.68.0" "-""-"
```

### 실행하기

이 단원에서는 [컨테이너 내부에서 명령 실행하기](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container) 기능을 검증하겠습니다.

다음과 같이 `nginx` 컨테이너 내부에서 `nginx -v` 명령을 실행하여 nginx 버전을 확인하겠습니다.

```bash
kubectl exec -ti $POD_NAME -- nginx -v
```

> 출력

```
nginx version: nginx/1.21.6
```

## Service

이 단원에서는 [Service](https://kubernetes.io/docs/concepts/services-networking/service/)를 사용하여 어플리케이션을 외부로 노출하는 기능을 검증하겠습니다.

[NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) 타입 Service를 사용하여 `nginx` Deployment를 외부로 노출합니다.

```bash
kubectl expose deployment nginx --port 80 --type NodePort
```

> 여러분들이 구성한 Kubernetes 클러스터는 [클라우드 공급자 통합](https://kubernetes.io/docs/getting-started-guides/scratch/#cloud-provider) 구성이 되어 있지 않기 때문에 LoadBalancer 타입 Service는 사용할 수 없습니다. 클라우드 공급자 통합 설정은 이 자습서의 범위를 벗어납니다.

`nginx` Service에 할당된 노드 포트를 확인합니다.

```bash
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

`nginx` 노드 포트로의 원격 접근을 허용하는 보안 그룹 규칙을 생성합니다.

```bash
openstack security group rule create \
  --ingress \
  --protocol tcp \
  --dst-port ${NODE_PORT} \
  external
```

워커 인스턴스의 외부 IP 주소를 확인합니다.

```bash
EXTERNAL_IP=$(openstack server show worker-0.${DOMAIN} -f value -c addresses | cut -d',' -f2 | tr -d '[:space:]')
```

외부 IP 주소와 `nginx` 노드 포트를 사용해서 HTTP 요청을 보냅니다.

```
curl -I http://${EXTERNAL_IP}:${NODE_PORT}
```

> 출력

```
HTTP/1.1 200 OK
Server: nginx/1.21.6
Date: Sat, 16 Apr 2022 13:38:54 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 25 Jan 2022 15:03:52 GMT
Connection: keep-alive
ETag: "61f01158-267"
Accept-Ranges: bytes
```

Next: [정리](14-cleanup.md)
