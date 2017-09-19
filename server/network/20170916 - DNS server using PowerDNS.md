# Build up DNS server using PowerDNS and PowerAdmin

## Background

* 내부 네트워크 Address: 192.168.11.0/24
* DNS를 올릴 서버의 주소: 192.168.11.70
* DNS를 통해 연결할 서버들의 주소
  * domain: 192.168.11.70
  * www: 192.168.11.101
  * virt: 192.168.11.102
* 내부 네트워크 이름: yourdomain.com

## Install Packages for PowerDNS

아래 명령어를 통해 epel, mariadb, pdns 를 설치. 또한, 네트워크 내 외부 시스템들의 DNS 쿼리를 지원하기 위해 firewall 에서 DNS 포트를 오픈함.

```bash
sudo yum install -y epel-release && sudo yum update -y;
sudo yum install -y mariadb mariadb-server;
sudo yum install -y pdns pdns-backend-mysql pdns-tools;
sudo firewall-cmd --zone=public --permanent --add-service=dns;
sudo firewall-cmd --reload;
```

## Setup PowerDNS

PowerDNS 설치를 위해서는 mariadb 초기화, PowerDNS용 DB 구축, PowerDNS 의 설정 값 변경 순서로 셋팅을 진행함.

### mariadb 초기화

우선 mariadb 의 서비스를 기동하여 초기화 하기 위한 사전작업 진행

```bash
systemctl enable mariadb.service
systemctl start mariadb.service
```

이후 mariadb 의 root 계정을 초기화 하여 셋업. 단, 설치 시 root 계정에 대한 계정암호를 입력하지 않았다면, root 계정에 암호를 설정해야 함.

```bash
mysql_secure_installation
```

아래가 실행 예제이며, 각 항목마다의 입력값의 설명은 아래와 같음

1. **Enter current password for root**: mariadb 의 root 계정 암호 입력.
    * 초기에는 아무런 암호가 설정되어 있지 않으므로 공란으로 엔터 입력.
1. **Set root password?**: root 계정 암호 리셋 여부 확인
1. **New Password / Re-enter new password**: root 계정의 암호 입력
1. **Remove anonymous users?**: DB 상 불필요한 익명 계정들 삭제
1. **Disallow root login remotely?**: root 계정의 원격 접속 허용 여부
1. **Remove test database and access to it?**: 임시 데이터베이스 삭제 여부
1. **Reload privilege table now?**: 권한정보 런타임 갱신 여부

```text
# mysql_secure_installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none):
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

Set root password? [Y/n] y
New password:
Re-enter new password:
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] y
 ... Success!

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```

### PowerDNS 용 DB 구축

데이터 베이스 작성을 위해 아래 링크의 SQL 스크립트 파일을 다운로드 받음.

* [pdns_db_create.sql](source/pdns_db_create.sql "PowerDNS DB Creation SQL file")

이후 PowerDNS 의 DB 생성을 위해, mariadb 에 접근함

```bash
mysql -u root -p mysql
```

root 계정으로 mariadb 에 로그인이 된 경우 아래의 계정정보 및 DB 를 생성하고, 앞선 다운로드 받은 SQL 파일을 사용하여 PowerDNS 에서 활용할 수 있도록 데이터베이스 내의 테이블 들을 생성함.

|Name    |Value   |
|:------:|:------:|
|DB Name |powerdns|
|DB User |powerdns|
|Password|password|

```sql
CREATE DATABASE powerdns;
GRANT ALL ON powerdns.* TO 'powerdns'@'localhost' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
SOURCE pdns_db_create.sql;
exit;
```

### PowerDNS 설정

PowerDNS 에서 앞서 구축한 mariadb 의 DB 를 활용할 수 있도록, /etc/pdns/pdns.conf 파일을 아래와 같이 수정함.

```diff
diff -urN a/pdns.conf b/pdns.conf
--- a/pdns.conf	2017-09-16 13:18:29.841720869 +0900
+++ b/pdns.conf	2017-09-16 13:21:48.370662061 +0900
@@ -21,6 +21,7 @@
 # allow-recursion	List of subnets that are allowed to recurse
 #
 # allow-recursion=0.0.0.0/0
+allow-recursion=0.0.0.0/0

 #################################
 # also-notify	When notifying a domain, also notify these nameservers
@@ -216,6 +217,12 @@
 # launch	Which backends to launch and order to query them in
 #
 # launch=
+launch=gmysql
+gmysql-host=localhost
+gmysql-user=powerdns
+gmysql-password=password
+gmysql-dbname=powerdns
+gmysql-dnssec=yes

 #################################
 # load-modules	Load this module - supply absolute or relative path
@@ -251,21 +258,25 @@
 # log-dns-details	If PDNS should log DNS non-erroneous details
 #
 # log-dns-details=no
+log-dns-details=on

 #################################
 # log-dns-queries	If PDNS should log all incoming DNS queries
 #
 # log-dns-queries=no
+log-dns-queries=yes

 #################################
 # logging-facility	Log under a specific facility
 #
 # logging-facility=
+logging-facility=0

 #################################
 # loglevel	Amount of logging. Higher is more. Do not set below 3
 #
 # loglevel=4
+loglevel=10

 #################################
 # lua-prequery-script	Lua script with prequery handler
```

상기와 같이 PowerDNS 의 설정을 완료한 경우 PowerDNS 서비스를 등록하고 기동함.

```bash
systemctl enable pdns.service
systemctl start pdns.service
```

## PowerAdmin 설치

Command line tool 을 활용하여 DNS 를 설정할 수 있으나, 웹 기반의 GUI Tool 인 PowerAdmin 으로 설정할 수 있음. 이를 위해서는 PHP + Apache 를 먼저 설치해 두어야 함.

### Install Prerequisit for PowerAdmin

PowerAdmin 설치를 위해 wget 과 unzip, 웹을 통한 제어화면을 제공하기 위해 http, php, php-mysql 을 설치하고, 해당 서비스들을 기동함. 또한 외부에서 PowerAdmin 에 접근할 수 있도록 Firewall 에서 http 포트를 개방함.

```bash
sudo yum install -y httpd php php-mysql php-mcrypt wget unzip;
sudo systemctl enable httpd;
sudo systemctl start httpd;
sudo firewall-cmd --zone=public --permanent --add-service=http;
sudo firewall-cmd --reload;
```

### Install PowerAdmin

아래의 GitHub 의 링크에서 master 버전의 PHP 서비스 다운로드.

* [PowerAdmin GitHub package](https://github.com/poweradmin/poweradmin/archive/master.zip "Master version PowerAdmin GitHub link")

다운로드 받은 파일의 압축을 풀고, httpd 의 웹 홈디렉토리 /var/www/html 에 압축 푼 파일들을 설치.

> **주의**: CentOS 7 에서 해당 파일을 복사가 아니라 이동을 한 경우 httpd 에서 권한의 문제로 파일을 열 수 없는 문제가 발생. 내부의 Hidden Permission 권한에 의해 발생하는 문제로 예상되며, 이를 해결하기 위해서는 mv 가 아니라 cp 를 통해 파일을 <u>**복사**</u> 해야 올바로 웹페이지에 접근할 수 있는 권한이 생김.

```bash
wget https://github.com/poweradmin/poweradmin/archive/master.zip;
sudo cp -r poweradmin-master /var/www/html/poweradmin;
```

### PowerAdmin 설정

PowerAdmin 의 경우 웹 페이지에서 셋팅을 진행하지만, 이미지 링크가 크게 필요없어 입력해야 할 내용을 서술함.

우선 PowerAdmin 의 설정을 위해 웹브라우저에서 http://192.168.11.70/poweradmin/install/ 에 접근함.

### Installation Step 1

설치 시 진행하는 언어를 선택함. 지원되는 언어는 영어, 네덜란드어, 독일어, 일본어, 폴란드어, 프랑스어, 노르웨이어를 지원함.

### Installation Step 2

설치를 위한 사전 설명이 있음. 다음 단계로 이동

### Installation Step 3

PowerDNS 에서 mariadb 서버를 연결하기 위한 정보를 입력.

1. **UserAdmin**: PowerDNS DB 에 접근하기 위한 계정 이름 입력
1. **Paddword**: PowerDNS DB 에 접근하기 위한 계정의 암호 입력.
1. **Database type**: PowerDNS DB 가 설치된 데이터베이스 엔진 선택. MySQL (mariadb), PostgreSQL, SQLite 를 선택할 수 있으며, 이 경우에는 MySQL 을 선택
1. **Hostname**: PowerDNS DB 가 설치된 서버의 주소 입력. 같은 서버내에 있기 때문에 localhost 입력.
1. **DB Port**: 앞서 선택한 mariadb 와 통신하기 위한 port 번호 입력 (3306)
1. **Database**: PowerDNS 의 DB 명 입력.
1. **DB charset**: DB 에 사용되는 characterset 입력. 공란의 경우 DB 의 기본값이 선택됨
1. **DB collation**: DB 에서 character 를 비교하기 위한 rule 들을 입력. 공란의 경우 DB 의 기본값이 선택된
1. **Poweradmin administrator password**: PowerAdmin의 기본 관리자 계정에 대한 암호를 입력.

### Installation Step 4

PowerAdmin 에서 앞선 PowerDNS 의 접근정보와 다른 PowerAdmin 관리 툴이 DB 에 접근하기 위한 정보를 입력. PowerDNS 가 아닌 PowerAdmin 의 불필요한 DB 접근을 방지하기 위해 PowerDNS 서비스와 다른 PowerAdmin 의 독립적인 DB 계정을 입력할 수 있음.

1. **Username**: PowerAdmin 에서 DB에 접근하기 위한 DB 계정 명 입력.
1. **Password**: PowerAdmin 에서 DB에 접근하기 위한 DB 계정의 암호 입력.
1. **Hostmaster**: SOA 레코드를 만들때 없는 경우 기본값으로 선택되는 값.
    * 일반적으로 "hostmaster.yourdomain.com" 과 같은 방식으로 선택됨.
1. **Primary namesaerver**: 템플릿을 만들때 주 네임서버로 설정되는 값.
1. **Secondary nameserver**: 템플릿을 만들때 보조 네임서버로 설정되는값.
    * Primary/Secondary nameserver 는 일반적으로 아래와 같이 설정됨.
        * **Primary nameserver**: ns1.yourdomain.com
        * **Secondary nameserver**: ns2.yourdomain.com

### Installation Step 5

앞선 Step 에서 PowerAdmin 의 DB 접근을 위한 계정을 DB 상에서 생성하고 권한을 할당에 대한 작업사항을 알려줌. 서버에서 mariadb 에 접근한 다음 화면에 표시된 SQL 문을 입력하여 권한을 설정함. 단, PowerDNS 와 동일한 계정을 앞선 Step 4 에 설정한 경우 권한을 설정 작업을 할 필요 없음.

### Installation Step 6

PowerAdmin 에서 설정을 위한 값을 활용할 수 있도록 poweradmin 의 inc 디렉토리 내에 있는 config-me.inc.php 파일을 config.inc.php로 복사하여 설정값을 수정함.

```bash
cd /var/www/html/poweradmin/inc;
cp config-me.inc.php config.inc.php;
```

변경해야 하는 내용은 아래와 같음

```diff
--- config-me.inc.php   2017-09-17 10:16:03.568970503 +0900
+++ config.inc.php      2017-09-17 10:50:16.573658249 +0900
@@ -15,12 +15,12 @@
 // Better description of available configuration settings you can find here:
 // <https://github.com/poweradmin/poweradmin/wiki/Configuration-File>
 // Database settings
-$db_host = '';
-$db_port = '';
-$db_user = '';
-$db_pass = '';
-$db_name = '';
-$db_type = '';
+$db_host = 'localhost';
+$db_port = '3306';
+$db_user = '<Step 4 에서 입력한 DB 계정 명>';
+$db_pass = '<Step 4 에서 입력한 DB 계정 암호>';
+$db_name = 'powerdns';
+$db_type = 'mysql';
 //$db_charset = 'latin1'; // or utf8
 //$db_file             = '';           # used only for SQLite, provide full path to database file
 //$db_debug            = false;        # show all SQL queries
@@ -28,7 +28,7 @@

 // Security settings
 // This should be changed upon install
-$session_key = 'p0w3r4dm1n';
+$session_key = '<웹 화면상에 나온 Session Key 정보>';
 $password_encryption = 'md5'; // md5, md5salt or bcrypt
 //$password_encryption_cost = 12; // needed for bcrypt

@@ -42,9 +42,9 @@
 $iface_add_reverse_record = true;

 // Predefined DNS settings
-$dns_hostmaster = '';
-$dns_ns1 = '';
-$dns_ns2 = '';
+$dns_hostmaster = 'hostmaster.yourdomain.com';
+$dns_ns1 = 'ns1.yourdomain.com';
+$dns_ns2 = 'ns2.yourdomain.com';
 $dns_ttl = 86400;
 $dns_fancy = false;
 $dns_strict_tld_check = false;
```

### Installation Step 7

PowerAdmin 설정을 위한 마지막 작업 진행. 아래와 같은 2가지 작업을 추가로 해 주어야 함

1. install/htaccess.dist 파일을 poweradmin 의 루트 디렉토리에 .htaccess 파일로 복사
1. powerdns 의 install 디렉토리를 삭제 하여 이후 추가 설정하는 것을 방지.

```bash
cd /var/www/html/poweradmin;
sudo cp install/htaccess.dist .htaccess;
sudo rm -rf install;
```

기본적으로 CentOS 7 은 mod_rewrite 가 활성화 되어 있으나, 다른 설정의 문제로 비 활성화 되어 있을 가능성도 존재. 따라서 httpd 서버에서 mod\_rewrite 가 활성화 안된 경우 mod\_rewrite 를 활성화 함. 따라서 CentOS 7 기준으로 /etc/httpd/conf.modules.d/00-base.conf 내의 아래 정보가 있는 줄을 확인하고, 주석처리 된 경우 주석을 해제

* **설정 줄 내용**: LoadModule reqrite_module modules/mod_rewrite.so

## DNS 환경 설정

PowerAdmin 에 로그인 해서 DNS 환경을 구축해야 함. 기본적인 관리자 계정의 아이디는 admin 이며, step 3 에서 설정한 암호를 통해 로그인 할 수 있음.

우선, 로그인한 첫 화면이나 탑메뉴의 **Add master zone** 을 클릭하여 master zone 정보를 추가함. 이때, 사용하고자 하는 도메인을 **"Zone name"** 항목에 입력한 다음 Add zone 을 클릭함.

이후 탑메뉴에서 List zones 를 선택한 다음 추가된 도메인의 리스트를 확인함. 이때 리스트의 왼쪽 메뉴에서 수정 아이콘을 선택함. 선택해서 이동한 페이지에서는 앞서 추가한 도메인에 대한 정보를 조회/수정이 가능하며, zone 내에서 활용할 레코드들을 추가할 수 있음.

가장 앞에서 가정한 www 와 virt 의 A recorde 에 대해 중간에 위치한 레코드 입력창에 아래 항목에 맞춰 입력함. 이때 한번에 하나씩 입력이 가능하며, zone 화면에서 입력한 다음 추가적인 레코드는 동일한 방법으로 계속 입력할 수 있음.

* **www**
  * **Name**: www
  * **Type**: A
  * **Content**: 192.168.11.101 *(IP Address)*
* **virt**
  * **Name**: virt
  * **Type**: A
  * **Content**: 192.168.11.102 *(IP Address)*

## DNS 서버 테스트

### Install bind-utils and settings

DNS 설정을 확인용으로 필요한 dig 를 설치하기 위해 bind-utils 를 설치.

```bash
sudo yum install -y bind-utils;
```

만약 신규 설정한 DNS 서버를 기본 DNS 서버로 설정하기 않은 경우 /etc/resolv.conf 파일에 **nameserver <DNS 서버주소>** 를 추가하여 기본 DNS 서버로 사용하도록 변경함.

설정이 완료한 경우 아래와 같은 명령으로 설정한 Domain 주소의 정보를 조회함. 아래는 성공적으로 셋팅된 결과임.

```text
# dig www.yourdomain.com A @127.0.0.1

; <<>> DiG 9.9.4-RedHat-9.9.4-51.el7 <<>> www.yourdomain.com A @127.0.0.1
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 19025
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1680
;; QUESTION SECTION:
;www.yourdomain.com.		IN	A

;; ANSWER SECTION:
www.yourdomain.com.	86400	IN	A	192.168.11.101

;; Query time: 1 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Sun Sep 17 11:37:47 KST 2017
;; MSG SIZE  rcvd: 63

```

## References

* [CentOS7 に PowerDNS をインストールする by sig9](http://sig9.hatenablog.com/entry/2016/12/20/120000 "CentOS7 に PowerDNS をインストールする")
* [[install] powerdns 설치하기 by wapj2000](http://gyus.me/?p=560 "[install] powerdns 설치하기")
