# Install k8s using kubeadm

## 개요

## Trouble shooting

### Firewall issue

기본적으로 centos 는 firewalld 가 자동 시작으로 셋팅되어 있음. 단, k8s 를 실행할 때 열아야 할 포트가 상당히 많다는 부분이 문제가 될 수 있기 때문에 그냥 포기하고 firewalld 를 비활성화 시킴.

### native.cgoupdriver

설치가이드 에서는 `/etc/docker/daemon.json` 파일에 

```json
{
    "exec-opts": ["native.cgroupdriver=systemd"]
}
```

위와 같은 내용을 추가하라고 되어 있으나, 위 옵션을 `daemon.json` 에 추가하게 되는 경우 CentOS 7.3 에서는 서비스 재시작 시 문제가 발생. 원인은 CentOS 7.3 의 기본 docker 가 이미 위 옵션을 적용해 두었으며, 동일한 설정값을 입력했기 때문에 설정값 duplicated 오류가 발생하는 것으로 보임. 따라서, **CentOS 7.3 에서는 위의 설정을 추가할 필요가 처음부터 없음.**

## Reference

- https://kubernetes.io/docs/setup/independent/install-kubeadm/