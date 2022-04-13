# 데이터 암호화 설정 및 키 생성

Kubernetes는 클러스터 상태, 응용 프로그램 설정 및 비밀키 등 다양한 데이터들을 저장합니다. Kubernetes는 유휴 상태의 클러스터 데이터들을 [암호화](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data) 하는 기능을 제공합니다.

이번 실습에서는 Kubernetes Secret들을 암호화하는데 필요한 암호화 키와 [암호화 설정](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration)을 생성해보겠습니다.

## 암호화 키

암호화 키는 다음과 같이 생성합니다.

```bash
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## 암호화 설정 파일

암호화 설정 파일인 `encryption-config.yaml`는 다음과 같이 생성합니다.

```bash
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

이제 생성한 `encryption-config.yaml` 암호화 설정 파일을 각 마스터 노드 인스턴스들로 복사합니다.

```bash
for instance in controller-0 controller-1 controller-2; do
  scp -i k8s.node-key.pem encryption-config.yaml ${instance}.${DOMAIN}:~
done
```

Next: [Bootstrapping the etcd Cluster](07-bootstrapping-etcd.md)
