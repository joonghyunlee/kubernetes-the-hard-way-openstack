# 사용자 도구 설치하기

이 실습에서는 앞으로 진행하는데 필요한 명령줄 유틸리티인 [cfssl](https://github.com/cloudflare/cfssl), [cfssljson](https://github.com/cloudflare/cfssl), alc [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl)을 설치합니다.


## CFSSL 설치하기

`cfssl` 및 `cfssljson` 명령줄 유틸리티는 [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure)를 준비하고 TLS 인증서를 생성하는데 사용합니다.

다음과 같이 `cfssl`와 `cfssljson`를 설치합니다.

### Mac OS

```
curl -o cfssl https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/darwin/cfssl
curl -o cfssljson https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/darwin/cfssljson
```

```bash
chmod +x cfssl cfssljson
```

```bash
sudo mv cfssl cfssljson /usr/local/bin/
```

만약 바이너리를 받아 설치했을 때 문제가 생긴다면, 다음과 같이 [Homebrew](https://brew.sh)를 통해 설치하는 것도 좋은 방법입니다.

```bash
brew install cfssl
```

### Linux

```bash
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssl \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssljson
```

```bash
chmod +x cfssl cfssljson
```

```bash
sudo mv cfssl cfssljson /usr/local/bin/
```

### 검증

`cfssl`과 `cfssljson` 버전이 1.4.1 이상 임을 확인합니다.

```
cfssl version
```

> 출력

```
Version: 1.4.1
Runtime: go1.12.12
```

```
cfssljson --version
```
```
Version: 1.4.1
Runtime: go1.12.12
```

## kubectl 설치하기

`kubectl` 명령줄 유틸리티는 Kubernetes API와 상호 작용하는데 사용합니다. 공식 릴리즈 바이너리로부터 `kubectl`를 받아서 설치합니다.

### Mac OS

```
curl -o kubectl https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/darwin/amd64/kubectl
```

```
chmod +x kubectl
```

```
sudo mv kubectl /usr/local/bin/
```

### Linux

```
wget https://storage.googleapis.com/kubernetes-release/release/v1.18.6/bin/linux/amd64/kubectl
```

```
chmod +x kubectl
```

```
sudo mv kubectl /usr/local/bin/
```

### 검증

`kubectl` 버전이 1.18.6 이상인지 확인합니다.

```
kubectl version --client
```

> 출력

```
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.6", GitCommit:"dff82dc0de47299ab66c83c626e08b245ab19037", GitTreeState:"clean", BuildDate:"2020-07-15T16:58:53Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"linux/amd64"}
```

다음: [Provisioning Compute Resources](03-compute-resources.md)
