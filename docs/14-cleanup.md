# 정리

이 실습에서는 자습서를 진행하면서 만들었던 리소스를 삭제하겠습니다.

## 인스턴스

컨트롤러와 워커 인스턴스들을 삭제합니다.

```bash
openstack server delete \
  controller-0.${DOMAIN} controller-1.${DOMAIN} controller-2.${DOMAIN} \
  worker-0.${DOMAIN} worker-1.${DOMAIN} worker-2.${DOMAIN}
```

## 네트워킹

로드밸런서를 삭제합니다.

```bash
neutron lbaas-healthmonitor-delete kubernetes
neutron lbaas-pool-delete kubernetes
neutron lbaas-listener-delete kubernetes
neutron lbaas-loadbalancer-delete kubernetes
```

보안 그룹을 삭제 합니다.

```bash
openstack security group delete internal external
```

Floating IP들을 삭제합니다.

```bash
openstack floating ip delete $(openstack floating ip list -f value -c ID)
```
