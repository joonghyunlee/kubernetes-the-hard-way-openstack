# 사전 준비

## NHN Cloud Platform

이 가이드에서는 [NHN Cloud](www.toast.com)을 활용하여 Kubernetes 클러스터를 구축하는데 필요한 컴퓨팅 인프라스트럭쳐 구축 과정을 간소화합니다.

## OpenStack 명령줄 도구 설치

### OpenStack 명령줄 도구 설치

다음 [OpenStack 문서](https://docs.openstack.org/newton/user-guide/common/cli-install-openstack-command-line-clients.html)를 참고하여 `python-openstackclient` 도구를 설치하고 사용할 수 있도록 준비합니다. 여기서는 3.12.0 버전을 설치합니다.

```bash
pip install python-openstackclient==3.12.0
pip install python-neutronclient==6.19.0
```

실행하면 아래와 같은 에러가 발생할 수 있습니다.

```bash
  File "$PYTHON-LIB-PATH/python2.7/site-packages/openstack/utils.py", line 13, in <module>
    import queue
ImportError: No module named queue
```

이 경우 아래와 같이 코드를 수정해야 합니다.

```bash
sed -i 's/^import queue$/import Queue as queue/' $PYTHON-LIB-PATH/python2.7/site-packages/openstack/utils.py
sed -i 's/^import queue$/import Queue as queue/' $PYTHON-LIB-PATH/python2.7/site-packages/openstack/cloud/openstackcloud.py
```

이 후 다음과 같이 확인합니다.

```bash
$ openstack --version
openstack 3.19.0
```

### OpenStack 환경 변수 설정

[OpenStack 명령줄 도구를 사용하기 위한 환경 변수 설정](https://docs.openstack.org/newton/user-guide/common/cli-set-environment-variables-using-openstack-rc.html) 문서와 NHN Cloud의 [Compute > API 사용준비](https://docs.toast.com/ko/Compute/Compute/ko/identity-api/) 문서를 참고하여 아래와 같이 OpenStack `rc` 파일을 준비합니다. 이 파일을 통해 OpenStack 관련 환경 변수들을 설정해야 OpenStack CLI를 사용할 수 있습니다.

```bash
export OS_USERNAME=joonghyun.lee@nhn.com
export OS_PASSWORD=$PASSWORD
export OS_AUTH_URL=	https://api-identity.infrastructure.cloud.toast.com/v2.0
export OS_TENANT_ID=$TENANT_ID
export OS_REGION_NAME=KR1
```

파일이 준비되면 `rc` 파일을 불러들여 환경 변수들을 설정합니다.

```
. openrc.sh
```

## tmux를 이용해 명령어를 동시에 실행하기

[tmux](https://github.com/tmux/tmux/wiki)를 사용하면 동시에 여러 인스턴스에서 명령을 실행할 수 있습니다. 이 자습서의 실습에서는 이러한 경우가 종종 있는데, 이 때는 tmux를 사용해서 사용해 여러 화면으로 분할하고 동기화 기능를 통해 동시에 명령을 내려 보다 빠르게 진행할 수 있습니다.

> tmux 사용이 반드시 자습서 내용을 진행하는데 반드시 필요하지는 않습니다.

![tmux screenshot](images/tmux-screenshot.png)

> Enable synchronize-panes by pressing `ctrl+b` followed by `shift+:`. Next type `set synchronize-panes on` at the prompt. To disable synchronization: `set synchronize-panes off`.

Next: [Installing the Client Tools](02-client-tools.md)
