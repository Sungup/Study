# PlantUML for GitLab

## Background

GitLab 내에서 UML을 통한 의견 공유가 필요한 경우 이를 간단히 표현하기 위한 인터페이스가 팔요했다. 이때, PlantUML 이라는 Java 기반의 Component를 발견했으며, 이를 WebApp형태로 올리면 URL을 통해서 GitLab과 PlantUML을 연결할 수 있는 GitLab의 내부 기능을 확인하였다.

설치하는 방법 자체는 어렵지 않으나, 관련된 사항을 모아두기 위해 CentOS 7 기준으로 정리한다.

## Install and configure the necessary dependencies

PlantUML을 설치해서 진행하는데 있어 기본적으로 아래와 같은 패키지들을 설치한다.

```bash
sudo yum install -y tomcat graphviz;
```

이때 사용자가 직접 git repository에서 clone을 해서 컴파일 하는 경우 아래 패키지들을 추가로 설치해야 한다

```bash
sudo yum install -y git maven;
```

## Package Download

### Git clone & user build

### Download .war package

## References

* [PlantUML & GitLab](https://gitlab.com/gitlab-org/gitlab-ce/blob/master/doc/administration/integration/plantuml.md "PlantUML & GitLab)

