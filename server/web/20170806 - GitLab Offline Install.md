# CentOS 7.3: Gitlab Offline Install

## Background

특정 개발망에서는 외부와의 네트워크 문제로 개발을 위한 Git 서버 구축의 어려움이 존재한다. 단, 이 경우 Offline Install이 필요하지만, 기본적인 Git 소스서버들의 경우 Django나 Rails와 같은 Web Framework를 사용하는 경우가 많은 관계로 Offline 설치가 가능한지 여부를 확인해야 할 필요가 있다.

2017/08/06 기준으로 2개의 가상머신을 Host Only Network상에서 묶어둔 상태에서 1개는 RPM Repository 서버, 다른 1개는 GitLab을 설치하기 위한 서버로 구성하였으며, 테스트 결과, 별도의 외부 Ruby Gems 의 다운로드 없이 순수하게 내부에 내장된 Package들 만으로 설치가 되며, 안정적으로 동작하는 것을 확인하였다.

## Install Environment

* OS: CentOS 7.3 Latest Update (2017/08/06)
* Package: GitLab Community Edition 9.4.3 (gitlab-ce-9.4.3-ce.0.el7.x86_64.rpm)

## Installatio Process

### Install and configure the necessary dependencies

email 전송을 위해 Postfix의 설지가 필요하면, Setup 시 Internet Site를 반드시 선택. (하지만 아직 찾지 못했음). Postfix외에 다른 Sendmail 이나 자체 SMTP 서버를 설정한 경우 해당 서버를 SMTP 서버로 설정할 수 있음.

CentOS에서 아래의 명령으로 SSH 서버를 열며, HTTP 및 SSH로 접근이 가능하도록 Firewall에서 Port를 개방한다.

```bash
sudo yum install -y curl policycoreutils openssh-server openssh-clients
sudo systemctl enable sshd
sudo systemctl start sshd
sudo yum install postfix
sudo systemctl enable postfix
sudo systemctl start postfix
sudo firewall-cmd --permanent --add-service=http
sudo systemctl reload firewalld
```

### Install the package

기본적으로 GitLab에서는 YUM Repository 서버를 통해 Maintenance 통해 Update를 제공한다. 단, 이 경우에는 Git Server가 외부 네트워크로 단절되어 있으므로, 외부 네트워크 상에서 존재하는 GitLab의 Repository에 직접적인 접근이 불가능하다. 따라서 GitLab에서 제공하는 RPM 패키지를 다운받아 설치해야 한다.

* 설치파일 경로: https://packages.gitlab.com/gitlab/gitlab-ce/

상기 주소를 통해 접근이 가능하며, 각 OS 별 설치에 필요한 RPM 및 DEB 파일들이 존재한다.

다운로드 받은 파일의 크기는 약 350MB 전후이며, 다운로드 받은 파일의 설치를 위해 아래와 같은 명령어로 GitLab을 설치한다. 이때 파일명의 XXX는 다운로드 받은 GitLab의 버전이다.

```bash
sudo yum localinstall -y gitlab-ce-XXX.rpm
```

### Configure and start GitLab

하기의 경로로 가서 GitLab에 대한 설정을 확인한 다음 GitLab의 초기화 및 설치를 진행한다.

* 설장파일 경로: /etc/gitlab/gitlab.rb

```bash
sudo gitlab-ctl reconfigure
```

### Browse to the hostname and login

설치한 서버를 Web Browser에서 접근하면, 관리자의 암호를 초기화 하도록 하는 화면을 만날 수 있다. 해당 화면에서 원하는 관리자의 암호를 설정하면 다시 로그인 화면으로 이동하게 된다.

이후 관리자 화면으로 들어가기 위해 **root** 계정에 대해 앞서 초기화 한 암호를 통해 로그인 한다. 이후 관리자의 이름을 원하는 형태로 변경할 수 있다.

## References

* [GitLab Installation](https://about.gitlab.com/installation/#centos-7 "GitLab Installation for CentOS 7")
