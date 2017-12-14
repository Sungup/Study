# DevStack on Xenial (Ubuntu 16.04.3)

## Background

2017년 기준 최신 LTS 버전인 Ubuntu 16.04.3 상에서 DevStack 을 통한 개발자 버전의 OpenStack 설치.

## Install

### Add user

DevStack 을 설치하기 위한 사용자 계정 stack을 아래와 같이 생성.

```bash
sudo useradd -m -c "DevStack User" stack -s /bin/bash
sudo passwd stack
```

사용자 계정을 생성한 다음, DevStack을 통해 설치 시 활용해야 하는 sudo 명령어를 stack 유저가 암호 없이 활용해야 하기 때문에, /etc/sudoers 파일에 아래와 같이 stack 유저에 대한 권한 추가.

```diff
$ sudo diff -uprN /etc/sudoers{.org,}
--- /etc/sudoers.org    2017-12-15 00:04:00.557476488 +0900
+++ /etc/sudoers        2017-12-15 00:05:08.290400731 +0900
@@ -25,6 +25,9 @@ root  ALL=(ALL:ALL) ALL
 # Allow members of group sudo to execute any command
 %sudo  ALL=(ALL:ALL) ALL
 
+# Specified user account for OpenStack
+stack  ALL=(ALL) NOPASSWD:ALL
+
 # See sudoers(5) for more information on "#include" directives:
 
 #includedir /etc/sudoers.d
```

### Clone DevStack

stack 유저로 로그인 한 다음 git을 사용하여 DevStack 을 clone 함. 2017년 12월 현재 최신 버전은 Pike 지만, 안정성을 위해 하나 전 버전인 Okata 를 활용. 기본적인 설정값은 samples/local.conf 에 있으며 이를 devstack 의 root 디렉토리에 저장함.

```bash
su - stack
git clone https://github.com/openstack-dev/devstack -b stable/ocata
cp ~/devstack/samples/local.conf ~/devstack
```

### Config and Install

저장한 local.conf 에서 아래와 같은 항목에 대해 파일 내용을 수정한다.

```diff
$ diff -uprN local.conf{.org,}
--- local.conf.org      2017-12-15 00:16:34.355979670 +0900
+++ local.conf  2017-12-15 00:15:27.126949365 +0900
@@ -25,9 +25,9 @@
 
 # If the ``*_PASSWORD`` variables are not set here you will be prompted to enter
 # values for them by ``stack.sh``and they will be added to ``local.conf``.
-ADMIN_PASSWORD=nomoresecret
-DATABASE_PASSWORD=stackdb
-RABBIT_PASSWORD=stackqueue
+ADMIN_PASSWORD=openstack
+DATABASE_PASSWORD=$ADMIN_PASSWORD
+RABBIT_PASSWORD=$ADMIN_PASSWORD
 SERVICE_PASSWORD=$ADMIN_PASSWORD
 
 # ``HOST_IP`` and ``HOST_IPV6`` should be set manually for best results if
@@ -98,3 +98,7 @@ SWIFT_REPLICAS=1
 # moved by setting ``SWIFT_DATA_DIR``. The directory will be created
 # if it does not exist.
 SWIFT_DATA_DIR=$DEST/data
+
+HOST_IP=10.0.1.21
+FIXED_RANGE=10.0.100.0/24
+
```

수정한 다음, HOST\_IP 설정이 적용 안되는 일이 발생할 수 있기 때문에 환경변수에 HOST\_IP 를 등록하고 설치 시작.

```bash
export HOST_IP=<HOST_IP>
cd ~/devstack
./stach.sh
```

올바르게 설치되었는지 확인하기 위해 Admin 토큰을 활용하여 network 환경 확인

```bash
cd ~/devstack
source openrc admin admin
openstack network list
```

### Check the status using GNU Screen (until Ocata)

DevStack의 경우 개발자를 위해 Screen 을 통해 실시간 동작을 확인할 수 있도록 지원함. 단, Pike 버전 이후부터는 지원하지 않으며, systemctl 을 활용하여 상태확인을 할 수 있음.

Screen 을 통해 제어하는 방법은 아래와 같음.

* `screen -ls`: screen 에서 동작하고 있는 Session 확인
* `screen -r`: screen 에 접근
  * _Cannot open /dev/pts0_ 가 발생한 경우 `script /dev/null` 을 실행한 다음 재접속
* screen 내의 내비게이션
  * `Ctrl-A + "`: list tabs
  * `Ctrl-A + n`: next tab
  * `Ctrl-A + p`: previous tab

## Trouble Shooting
