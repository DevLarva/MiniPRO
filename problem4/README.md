# WordPress 다중 서버 배포와 로드 밸런싱 및 NFS 연동

> 작성자: 백대홍
>
> **버전:** 1.0 (4번 문항)
>
> **날짜:** 2025-07-31

---

## 목차

## 목차

- [1. 개요](#1-개요)
- [2. 아키텍처 구성도](#2-아키텍처-구성도)
  - [2.1 아키텍처 구성 요소별 역할 분석](#21-아키텍처-구성-요소별-역할-분석)
- [3. 환경 정보](#3-환경-정보)
  - [시스템 환경](#시스템-환경)
  - [네트워크 구성](#네트워크-구성)
  - [네트워크 대역별 용도](#네트워크-대역별-용도)
- [4. NFS 서버 구성](#4-nfs-서버-구성)
  - [4.1 NFS 설치 및 서비스 설정](#41-nfs-설치-및-서비스-설정)
  - [4.2 WordPress 파일 준비 및 배치](#42-wordpress-파일-준비-및-배치)
  - [4.3 NFS 공유 설정](#43-nfs-공유-설정)
  - [4.4 방화벽 설정](#44-방화벽-설정)
  - [4.5 WordPress 설정 파일 준비](#45-wordpress-설정-파일-준비)
  - [4.6 서비스 재시작 및 확인](#46-서비스-재시작-및-확인)
- [5. 데이터베이스 서버 구성](#5-데이터베이스-서버-구성)
  - [5.1 MySQL 설치 및 서비스 설정](#51-mysql-설치-및-서비스-설정)
  - [5.2 방화벽 설정](#52-방화벽-설정)
  - [5.3 MySQL 내부 설정](#53-mysql-내부-설정)
- [6. 웹 서버 구성](#6-웹-서버-구성)
  - [6.1 Apache, PHP 설치](#61-apache-php-설치)
  - [6.2 방화벽 설정](#62-방화벽-설정)
  - [6.3 NFS 공유 디렉토리 마운트](#63-nfs-공유-디렉토리-마운트)
  - [6.4 Apache 가상 호스트 설정](#64-apache-가상-호스트-설정)
  - [6.5 SELinux 정책 설정](#65-selinux-정책-설정)
  - [6.6 서비스 재시작](#66-서비스-재시작)
- [7. 로드 밸런서 구성](#7-로드-밸런서-구성)
  - [7.1 HAProxy 설치](#71-haproxy-설치)
  - [7.2 HAProxy 설정 파일 수정](#72-haproxy-설정-파일-수정)
  - [7.3 SELinux 및 서비스 시작](#73-selinux-및-서비스-시작)
  - [7.4 방화벽 구성](#74-방화벽-구성)
- [8. 구성 검증](#8-구성-검증)
  - [8.1 X-Server-ID 헤더 이용](#81-x-server-id-헤더-이용)
  - [8.2 access_log 실시간 확인](#82-access_log-실시간-확인)
  - [8.3 NFS 일관성 확인](#83-nfs-일관성-확인)

---

## 1. 개요

이 문서는 2대의 웹 서버, 1대의 로드 밸런서, 1대의 데이터베이스 서버, 그리고 1대의 NFS 서버로 구성된 고가용성 인프라 환경 구축을 목표로 합니다.

**주요 목표:**

- **로드 밸런싱**: 다중 웹 서버 간의 트래픽을 효율적으로 분산
- **데이터 일관성**: NFS를 통해 여러 웹 서버가 동일한 WordPress 파일 공유
- **네트워크 분리**: 보안 강화를 위한 내부/외부 네트워크 분리
- **통합 연동**: Web-DB-NFS 간 통신 및 데이터 흐름 구현

---

## 2. 아키텍처 구성도

![image.png](%EB%B3%B4%EA%B3%A0%EC%84%9C%202954f143198880aca323e9c76653dc1c/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-10-23_21.08.27.png)

### 2.1 아키텍처 구성 요소별 역할 분석

**클라이언트 (Client)**

서비스에 접속하는 최종 사용자입니다. 로드 밸런서의 공인 IP 주소로만 통신하며 내부의 복잡한 구조는 인지하지 못합니다.

**로드 밸런서 (Load Balancer - LB)**

- **트래픽 분산**: 외부의 모든 HTTP 요청을 받아 내부 웹 서버(Web 1, Web 2)들에게 라운드로빈 방식으로 균등하게 분배합니다.
- **고가용성**: 웹 서버의 상태를 주기적으로 확인(Health Check)하여 장애가 발생한 서버로는 트래픽을 보내지 않습니다.

**웹 서버 (Web Server 1 & 2)**

- **애플리케이션 엔진**: 로드 밸런서로부터 전달받은 요청을 처리하여 PHP 코드를 실행하고 동적인 웹 페이지를 생성합니다.
- **중앙 파일 시스템 연결**: 모든 WordPress 소스 파일과 미디어 업로드 파일을 NFS 서버에서 마운트하여 사용합니다. 이를 통해 모든 웹 서버가 항상 동일한 데이터를 보도록 보장합니다.
- **데이터 요청자**: 동적 콘텐츠 생성을 위해 중앙 데이터베이스 서버에 접속하여 데이터를 요청합니다.

**NFS 서버 (Network File System Server)**

- **중앙 파일 저장소**: WordPress의 핵심 파일, 테마, 플러그인, 업로드된 이미지 등 웹 콘텐츠 전체를 저장하고 관리합니다.
- **파일 공유**: 웹 서버들에게 `/var/www/html` 디렉토리를 네트워크를 통해 공유하여 데이터의 일관성을 유지합니다.

**데이터베이스 서버 (Database Server - DB)**

- **중앙 데이터 저장소**: 웹사이트의 모든 영구 데이터(게시글, 사용자 정보, 설정 등)를 저장하고 관리합니다.
- **데이터 제공자**: 내부망에 위치한 웹 서버들의 SQL 쿼리 요청에만 응답하여 데이터를 제공합니다.
- **보안**: 외부 인터넷과 완전히 분리되어 데이터베이스 직접 공격을 원천 차단합니다.

---

## 3. 환경 정보

### 시스템 환경

- **가상화**: Vagrant + VirtualBox
- **OS**: Rocky Linux 9 (RHEL 계열)
- **셸**: Bash

### 네트워크 구성

| 서버 종류           | 역할        | IP 주소       | 설명                                              |
| ------------------- | ----------- | ------------- | ------------------------------------------------- |
| **Load Balancer**   | Public 접속 | 192.168.55.66 | 외부 사용자가 접속하는 인터페이스 (Public IP)     |
|                     | 내부 통신   | 192.168.56.66 | 내부 웹 서버들과 통신하는 인터페이스 (Private IP) |
| **Web Server 1**    | Web 통신    | 192.168.56.44 | 로드 밸런서와 연결되는 내부 웹 인터페이스         |
|                     | DB/NFS 접속 | 192.168.57.44 | 데이터베이스 및 NFS 서버와 통신                   |
| **Web Server 2**    | Web 통신    | 192.168.56.55 | 로드 밸런서와 연결되는 내부 웹 인터페이스         |
|                     | DB/NFS 접속 | 192.168.57.55 | 데이터베이스 및 NFS 서버와 통신                   |
| **Database Server** | DB 접속     | 192.168.57.33 | 웹 서버들로부터의 DB 연결 수신                    |
| **NFS Server**      | NFS 접속    | 192.168.57.66 | 웹 서버들에게 파일 공유 서비스 제공               |

### 네트워크 대역별 용도

| 네트워크            | CIDR            | 용도        | 연결 서버            |
| ------------------- | --------------- | ----------- | -------------------- |
| **Public Network**  | 192.168.55.0/24 | 외부 접속   | Client ↔ LB          |
| **Web Network**     | 192.168.56.0/24 | 웹 트래픽   | LB ↔ Web1, Web2      |
| **Backend Network** | 192.168.57.0/24 | 내부 서비스 | Web1, Web2 ↔ DB, NFS |

---

## 4. NFS 서버 구성

모든 웹 서버가 공유할 WordPress 파일을 중앙에서 관리하기 위해 NFS 서버를 먼저 구성합니다.

### 4.1 NFS 설치 및 서비스 설정

```bash
# NFS 관련 유틸리티 설치
$ sudo dnf install -y nfs-utils

# NFS 서비스를 시작하고 부팅 시 자동 실행되도록 설정
$ sudo systemctl enable --now nfs-server

```

### 4.2 WordPress 파일 준비 및 배치

```bash
# WordPress 최신 버전 다운로드 및 압축 해제
$ sudo curl -o wordpress.tar.gz https://wordpress.org/latest.tar.gz
$ sudo mkdir -p /var/www/html
$ sudo tar xvf wordpress.tar.gz -C /var/www/html

# 공유할 디렉토리 권한 설정 (웹 서버 프로세스가 접근 가능하도록)
$ sudo chown -R apache:apache /var/www/html/wordpress

```

### 4.3 NFS 공유 설정

```bash
# /etc/exports 파일에 공유할 디렉토리와 옵션을 추가합니다.
$ sudo vi /etc/exports
```

**추가 내용:**

```
/var/www/html 192.168.57.0/24(rw,sync,no_root_squash)
```

### 4.4 방화벽 설정

```bash
# NFS 서비스에 필요한 방화벽 포트를 영구적으로 개방합니다.
$ sudo firewall-cmd --add-service=nfs --permanent
$ sudo firewall-cmd --add-service=rpc-bind --permanent
$ sudo firewall-cmd --add-service=mountd --permanent
$ sudo firewall-cmd --reload
```

### 4.5 WordPress 설정 파일 준비

```bash
# DB 연결 정보를 담을 설정 파일을 샘플 파일로부터 복사합니다.
$ sudo cp /var/www/html/wordpress/wp-config-sample.php /var/www/html/wordpress/wp-config.php

# DB 연결 정보를 수정합니다. 이 파일은 모든 웹 서버가 공통으로 사용합니다.
$ sudo vi /var/www/html/wordpress/wp-config.php
```

**수정 내용:**

```php
define( 'DB_NAME', 'wp' );
define( 'DB_USER', 'web' );
define( 'DB_PASSWORD', 'password' );
define( 'DB_HOST', '192.168.57.33' ); // DB 서버의 내부 IP
```

**로드 밸런서의 상태 확인을 위한 health.php 페이지 작성:**

```bash
$ sudo vi /var/www/html/wordpress/health.php

```

**작성 내용:**

```php
<?php
  echo "Hostname: " . gethostname() . "<br>";
  echo "Server IP: " . $_SERVER['SERVER_ADDR'] . "<br>";
?>
```

### 4.6 서비스 재시작 및 확인

```bash
# 설정한 공유 정보를 시스템에 적용합니다.
$ sudo exportfs -ra

# 공유 상태를 확인합니다.
$ sudo showmount -e localhost
```

---

## 5. 데이터베이스 서버 구성

WordPress의 모든 데이터를 저장할 DB 서버를 구성합니다.

### 5.1 MySQL 설치 및 서비스 설정

```bash
$ sudo dnf install -y mysql-server
$ sudo systemctl enable --now mysqld
```

### 5.2 방화벽 설정

```bash
$ sudo firewall-cmd --add-service=mysql --permanent
$ sudo firewall-cmd --reload
```

### 5.3 MySQL 내부 설정

```bash
# MySQL 서버에 접속합니다.
$ sudo mysql
```

**SQL 명령어:**

```sql
-- WordPress용 데이터베이스 및 사용자를 생성합니다.
mysql> CREATE DATABASE wp;
mysql> CREATE USER 'web'@'192.168.57.%' IDENTIFIED BY 'password';
mysql> GRANT ALL PRIVILEGES ON wp.* TO 'web'@'192.168.57.%';
mysql> FLUSH PRIVILEGES;
mysql> EXIT;
```

---

## 6. 웹 서버 구성

두 웹 서버에 동일한 설정을 적용합니다. 아래 절차를 Web 1, Web 2 서버에서 각각 수행합니다.

### 6.1 Apache, PHP 설치

```bash
$ sudo dnf install -y httpd php php-mysqlnd
$ sudo systemctl enable --now httpd
```

### 6.2 방화벽 설정

```bash
$ sudo firewall-cmd --permanent --add-service=http
$ sudo firewall-cmd --permanent --add-service=nfs
$ sudo firewall-cmd --permanent --add-service=mountd
$ sudo firewall-cmd --permanent --add-service=rpc-bind
$ sudo firewall-cmd --reload
```

### 6.3 NFS 공유 디렉토리 마운트

```bash
# NFS 서버의 공유 디렉토리를 웹 서버의 /var/www/html 디렉토리에 마운트합니다.
$ sudo mount -t nfs 192.168.57.66:/var/www/html /var/www/html

# 재부팅 시에도 자동으로 마운트되도록 /etc/fstab에 등록합니다.
$ sudo vi /etc/fstab
```

**추가 내용:**

```
192.168.57.66:/var/www/html     /var/www/html   nfs    defaults,sec=sys 0 0
```

**설정 적용 및 확인:**

```bash
# fstab 설정을 시스템에 즉시 적용하고 마운트 상태를 확인합니다.
$ sudo mount -a
$ mount | grep html
```

### 6.4 Apache 가상 호스트 설정

```bash
# WordPress 디렉토리를 DocumentRoot로 사용하는 가상 호스트를 설정합니다.
$ sudo vi /etc/httpd/conf.d/wordpress.conf
```

**작성 내용:**

```
<VirtualHost *:80>
    DocumentRoot /var/www/html/wordpress
    <Directory "/var/www/html/wordpress">
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

### 6.5 SELinux 정책 설정

```bash
# Apache(httpd)가 네트워크를 통해 DB에 접속하고, NFS를 사용할 수 있도록 SELinux 정책을 영구적으로 허용합니다.
$ sudo setsebool -P httpd_can_network_connect_db 1
$ sudo setsebool -P httpd_use_nfs 1
```

### 6.6 서비스 재시작

```bash
$ sudo systemctl restart httpd
```

---

## 7. 로드 밸런서 구성

웹 서버들 앞단에서 트래픽을 분산할 로드 밸런서를 설정합니다.

### 7.1 HAProxy 설치

```bash
$ sudo dnf install -y haproxy
```

### 7.2 HAProxy 설정 파일 수정

```bash
$ sudo vi /etc/haproxy/haproxy.cfg
```

**수정 내용 (frontend 및 backend 섹션):**

```
#---------------------------------------------------------------------
# frontend main
#---------------------------------------------------------------------
frontend main
    bind *:80
    default_backend             app

#---------------------------------------------------------------------
# backend app
#---------------------------------------------------------------------
backend app
    balance     roundrobin
    # 웹 서버의 로그인 페이지를 통해 상태를 확인합니다.
    option httpchk GET /wp-login.php
    server web1 192.168.56.44:80 check
    server web2 192.168.56.55:80 check
    # 응답 헤더에 실제 처리한 서버의 이름을 추가하여 확인을 용이하게 합니다.
    http-response add-header X-Server-ID %[srv_name]

```

### 7.3 SELinux 및 서비스 시작

```bash
# HAProxy가 모든 네트워크에 연결할 수 있도록 SELinux 정책을 설정합니다.
$ sudo setsebool -P haproxy_connect_any 1

# HAProxy 서비스를 시작하고 부팅 시 자동 실행되도록 설정합니다.
$ sudo systemctl enable --now haproxy.service

# 설정 파일의 문법이 올바른지 확인합니다.
$ haproxy -c -f /etc/haproxy/haproxy.cfg
```

### 7.4 방화벽 구성

```bash
# 외부에서 80번 포트로 접속할 수 있도록 방화벽을 영구적으로 개방합니다.
$ sudo firewall-cmd --permanent --add-port=80/tcp
$ sudo firewall-cmd --reload
```

> 로드 밸런서의 공인 IP(192.168.55.66)로 접속 시, 요청이 Web 1과 Web 2로 분산됩니다.

---

## 8. 구성 검증

### 8.1 X-Server-ID 헤더 이용

HAProxy 설정의 `http-response add-header X-Server-ID %[srv_name]` 라인은 응답 헤더에 실제 요청을 처리한 서버의 이름(예: `web1`, `web2`)을 추가해줍니다. 이를 통해 브라우저 개발자 도구의 '네트워크' 탭에서 로드 밸런싱이 올바르게 동작하는지 확인할 수 있습니다.

![image.png](%EB%B3%B4%EA%B3%A0%EC%84%9C%202954f143198880aca323e9c76653dc1c/image2.png)

> 위 그림처럼 응답 헤더의 X-Server-Id 값이 web1 또는 web2로 번갈아 나타나는지 확인합니다.

### 8.2 access_log 실시간 확인

각 웹 서버에서 아래 명령어를 실행하여 로드 밸런서로부터 오는 요청이 실시간으로 기록되는지 확인할 수 있습니다.

```bash
# 각 웹 서버(Web 1, Web 2)에서 개별적으로 실행
$ sudo tail -f /var/log/httpd/access_log
```

> 로드 밸런서 주소로 새로고침 할 때마다 아래 그림처럼 각 서버에 로그가 번갈아 기록되는 것을 확인할 수 있습니다.

### 8.3 NFS 일관성 확인

Web1에서 업로드한 파일이 Web2에서도 확인되면 성공입니다.

```bash
# Web1에서 테스트 파일 작성
$ echo "hello from web1" | sudo tee /var/www/html/test-nfs.txt

# Web2에서 확인
$ cat /var/www/html/test-nfs.txt
```

**예상 출력:**

```
hello from web1
```
