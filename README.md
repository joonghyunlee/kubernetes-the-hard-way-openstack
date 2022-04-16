# Kubernetes The Hard Way

참고

* https://github.com/kelseyhightower/kubernetes-the-hard-way

* https://github.com/e-minguez/kubernetes-the-hard-way-openstack

* https://github.com/tienne/kubernetes-the-hard-way-aws-ko



이 자습서는 Kubernetes를 ~~고생 길~~ 밑바닥부터 설정하는 과정을 안내합니다. 이 문서는 완전히 자동화된 Kubernetes 클러스터 설치 방법을 찾고자 하는 사람에게는 맞지 않습니다. 만약 그런 방법을 원한다면 [NHN Kubernetes](https://www.toast.com/kr/service/container/kubernetes) ~~데헷~~ 또는 [Getting Started Guides](https://kubernetes.io/docs/setup)를 참고하길 바랍니다.

Kubernetes The Hard Way는 Kubernetes 클러스터를 스스로 구축하는데 필요한 각 작업들을 이해할 수 있도록 긴 과정을 거치게 함으로써 학습에 최적화되어 있다고 말할 수 있습니다.

> 이 자습서의 결과물은 상용 수준을 기대해서는 안되며, 커뮤니티로부터 지원은 제한적입니다. 그렇지만 학습을 멈추지는 마세요!

## Copyright

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License</a>.


## 이런 분들에게 좋아요

이 자습서는 Kubernetes 클러스터를 실제 서비스에 도입할 예정에 있고 어떻게 동작하는지 알고 싶어하는 사람들을 대상으로 합니다.



## 클러스터 상세

Kubernetes The Hard Way는 컴포넌트 수준 종단 간 암호화와 RBAC 인증이 적용된 고가용성 Kubernetes 클러스터를 구축하는 과정을 안내합니다.

* [kubernetes](https://github.com/kubernetes/kubernetes) v1.18.6
* [containerd](https://github.com/containerd/containerd) v1.3.6
* [coredns](https://github.com/coredns/coredns) v1.7.0
* [cni](https://github.com/containernetworking/cni) v0.8.6
* [etcd](https://github.com/coreos/etcd) v3.4.10



## 목차

이 자습서는 [NHN Cloud](www.toast.com)에 접근할 수 있다고 가정합니다. NHN Cloud를 기본 인프라스트럭쳐로 사용하지만, 이 자습서에서 배운 내용들은 다른 OpenStack 플랫폼에도 적용할 수 있습니다.

* [사전 준비](docs/01-prerequisites.md)
* [사용자 도구 설치](docs/02-client-tools.md)
* [인스턴스 준비](docs/03-compute-resources.md)
* [인증기관(CA) 구성 및 TLS 인증서 생성](docs/04-certificate-authority.md)
* [인증을 위한 쿠버네티스 설정 파일 생성](docs/05-kubernetes-configuration-files.md)
* [데이터 암호화 설정 및 키 생성](docs/06-data-encryption-keys.md)
* [etcd 클러스터 구성](docs/07-bootstrapping-etcd.md)
* [Kubernetes 컨트롤러 노드 구성](docs/08-bootstrapping-kubernetes-controllers.md)
* [Kubernetes 워커 노드 구성](docs/09-bootstrapping-kubernetes-workers.md)
* [원격 접근을 위한 kubectl 설정](docs/10-configuring-kubectl.md)
* [Pod Network 라우팅 구성](docs/11-pod-network-routes.md)
* [DNS 클러스터 애드온 설치](docs/12-dns-addon.md)
* [기능 검증](docs/13-smoke-test.md)
* [정리](docs/14-cleanup.md)
