# Docker with LVM

## Introduction

CentOS 에 Docker 올리기. 기본적인 설치를 위한 계정은 root 계정으로 진행한다.

## Installation

### lvm volume 생성

lvm2 volume 생성 이전에 lvm2 volume 으로 활용하기 위한 가상 디스크를 추가함. VirtualBox 에 새로운 가상디스크를 sata 포트에 추가하게 되면, OS 부팅 시 /dev/sdb 로 보여지게 됨.

lvm2 를 위한 디스크를 추가하고 부팅에 성공하면 lvm2 패키지를 설치하여 lvm2 및 thin-provisioning-tools 패키지를 설치함.

```bash
yum install -y lvm2;
```

이후, 새로 추가된 /dev/sdb에 대해 아래의 순서대로 작업을 진행한다.

1. pvcreate 로 /dev/sdb 에 물리적 볼륨을 생성
1. docker 를 위한 볼륨그룹을 생성.
1. thinpool 과 thinpoolmeta 라는 논리적 볼륨을 생성함.
    - 이때 예시로 docker를 위한 thinpool 과 thinpoolmeta 는 각각 95%, 1%를 할당하고 자동확장이 가능하도록 일부 공간 (4%)를 남김.
1. pool 을 thinpool 로 변환함.

위의 작업에 대해 아래와 같은 command 로 작업을 진행.

```bash
pvcreate /dev/sdb;
vgcreate docker /dev/sdb;
lvcreate --wipesignatures y -n thinpool docker -l 95%VG;
lvcreate --wipesignatures y -n thinpoolmeta docker -l 1%VG;
lvconvert -y --zero n -c 512K --thinpool docker/thinpool --poolmetadata docker/thinpoolmeta;
```

lvm 에 대해 앞서 생성한 docker 용 thinpool 에 대해 `/etc/lvm/profile/docker-thinpool.profile` 설정파일을 만들어 자동확장 옵션을 아래와 같이 설정함.

```text
activation {
thin_pool_autoextend_threshold=80
thin_pool_autoextend_percent=20
}
```

위와 같은 profile 을 만들고 난 다음 해당 profile 내용을 앞서 생성한 docker/thinpool 에 적용함. 이때, `lvs` 명령어를 통해 앞서 생성한 logical volume 이 정상적으로 셋팅되었는지 확인.

```bash
lvchange --metadataprofile docker-thinpool docker/thinpool;
lvs -o+seg_monitor;
```

### Install docker using direct-lvm mode

아래의 명령으로 docker 를 설치한 다음 device mode 를 direct-lvm 으로 변경시키기 위해 docker 를 정지시키고 기존의 Graph Device Directory 를 제거함. 단, 필요시 이전 버전으로 복구가 필요한 경우 다른 이름의 디렉토리로 변경함.

```bash
yum install -y docker;
systemctl stop docker;
mkdir /var/lib/docker.backup;
mv /var/lib/docker/* /var/lib/docker.backup/;
```

`/etc/docker/daemon.json` 파일에 아래와 같은 설정값을 추가함.

```json
{
    "storage-driver": "devicemapper",
    "storage-opts":[
        "dm.thinpooldev=/dev/mapper/docker-thinpool",
        "dm.use_deferred_removal=true",
        "dm.use_deferred_deletion=true"
    ]
}
```

> 단, 아래의 옵션중에서 dm.use\_deferred\_deletion 설정은 kernel version 3.18 이후에서만 활용이 가능하므로 CentOS 및 구버전의 Linux Distribution 에서는 `/etc/docker/daemon.json` 파일에 추가 하지 않음.
>
> **Kernel Version 3.18 이하**
>
> ```json
> {
>     "storage-driver": "devicemapper",
>     "storage-opts":[
>         "dm.thinpooldev=/dev/mapper/docker-thinpool",
>         "dm.use_deferred_removal=true"
>     ]
> }
> ```

`/etc/docker/daemon.json` 파일에 변경사항을 저장한 다음 docker 데몬을 다시 실행시킴. 이때 logical volume 의 상태를 확인하기 위해 `lvs` 혹은 `lvs -a` 명령을 사용하며, 여유공간을 확인은 `vgs` 명령을 사용함.

또한, thinpool 에 대한 자동확장 로그는 아래의 명령어로 확인 함.

```bash
systemctl enable docker && systemctl start docker;
lvs -a;
vgs;
journalctl -fu dm-event.service;
```

## References

- http://xoit.tistory.com/18
- https://docs.docker.com/engine/reference/commandline/dockerd/
