# ***WordPress Multi‑Server Deployment with Load Balancing***

> **작성자:** 백대홍
>
> **버전:** 1.0 (3번 문항)
> 
> **날짜:** 2025-07-31

## **목차**

- [***WordPress Multi‑Server Deployment with Load Balancing***](#wordpress-multiserver-deployment-with-load-balancing)
  - [**목차**](#목차)
  - [1. 개요](#1-개요)
  - [2. 아키텍처 구성도](#2-아키텍처-구성도)
    - [2.1 아키텍처 구성 요소별 역할 분석](#21-아키텍처-구성-요소별-역할-분석)
  - [3. 환경 정보](#3-환경-정보)
  - [4. 데이터베이스 서버 구성](#4-데이터베이스-서버-구성)
    - [4.1 MySQL 설치 및 서비스 설정](#41-mysql-설치-및-서비스-설정)
    - [4.2 방화벽 설정](#42-방화벽-설정)
    - [4.3 MySQL 내부 설정](#43-mysql-내부-설정)
  - [5. 웹 서버 구성](#5-웹-서버-구성)
    - [5.1 Apache 설치 및 서비스 구성](#51-apache-설치-및-서비스-구성)
    - [5.2 PHP 및 MySQL 클라이언트 설치](#52-php-및-mysql-클라이언트-설치)
    - [5.3 방화벽 설정](#53-방화벽-설정)
    - [5.4 WordPress 설치 및 배치](#54-wordpress-설치-및-배치)
    - [5.5 wp-config.php 내부 설정](#55-wp-configphp-내부-설정)
    - [5.6 Apache 가상 호스트 설정](#56-apache-가상-호스트-설정)
    - [5.7 health.php 상태 확인 페이지 작성](#57-healthphp-상태-확인-페이지-작성)
    - [5.8 SELinux 설정](#58-selinux-설정)
    - [5.9 서비스 재시작 및 검증](#59-서비스-재시작-및-검증)
  - [6. 추가 웹 서버 구성](#6-추가-웹-서버-구성)
  - [7. 로드 밸런서 구성](#7-로드-밸런서-구성)
    - [7.1 HAProxy 설치](#71-haproxy-설치)
    - [7.2 HAProxy 설정 파일 수정](#72-haproxy-설정-파일-수정)
    - [7.3 SELinux 및 서비스 시작](#73-selinux-및-서비스-시작)
    - [7.4 방화벽 구성](#74-방화벽-구성)
  - [8. 로드 밸런싱 검증](#8-로드-밸런싱-검증)
    - [8.1 X-Server-ID 헤더 이용](#81-x-server-id-헤더-이용)
    - [8.2 access\_log 실시간 확인](#82-access_log-실시간-확인)

---

## 1. 개요

이 문서는 2대의 웹 서버, 1대의 로드 밸런서, 1대의 데이터베이스 서버로 구성된 인프라 환경을 기반으로, 다음을 목표로 한다.

-   로드 밸런싱을 통해 다중 웹 서버 간의 트래픽 분산
-   내부/외부 네트워크 분리 구조의 이해
-   DB 연결 구조 및 웹-DB 통신 방식 설명

## 2. 아키텍처 구성도

<img src="ARC.png" alt="image.png" width="700"/>


---


### 2.1 아키텍처 구성 요소별 역할 분석

**클라이언트 (Client)**

-   서비스에 접속하는 사용자를 나타냅니다.
-   이들은 아키텍처의 복잡성을 전혀 인지하지 못하며, 오직 로드 밸런서의 단일 공인 IP 주소하고만 통신합니다.

**로드 밸런서 (Load Balancer - LB)**

-   트래픽 분산 (Traffic Distribution): 들어온 요청을 여러 웹 서버로 공평하게 분산시켜, 하나의 웹 서버에 트래픽이 몰리는 것을 방지하고 전체 시스템의 처리 효율을 높입니다.


**웹 서버 (Web Server 1 & 2)**

-   애플리케이션 엔진: 실제 웹 페이지를 만들고 사용자 요청을 처리하는 엔진입니다. 로드 밸런서로부터 전달받은 요청을 실행하여 비즈니스 로직(WordPress/PHP)을 처리합니다.
-   데이터 요청자: 동적인 웹 페이지를 생성하기 위해, 데이터베이스 서버에 접속하여 게시글, 사용자 정보 등의 데이터를 가져옵니다.
-   고가용성 (High Availability): 두 대 이상으로 구성되어 있어 한 서버에 장애가 발생하더라도 다른 서버가 서비스를 계속 이어갈 수 있습니다.

**데이터베이스 서버 (Database Server - DB)**

-   중앙 데이터 저장소: 웹사이트의 모든 영구적인 데이터(게시글, 댓글, 사용자 정보, 설정 등)를 저장하고 관리하는 중앙 저장소입니다.
-   데이터 제공자: 오직 신원이 확인된 웹 서버들의 데이터 요청(SQL 쿼리)에만 응답하여 데이터를 제공합니다.
-   보안의 핵심: 외부 인터넷과 완전히 분리되어 있어 데이터베이스를 직접 공격하는 것을 원천적으로 차단하며, 시스템의 보안을 크게 향상시킵니다.

**전체 시스템 동작 흐름**

1.  클라이언트가 로드 밸런서의 공인 IP로 웹 페이지를 요청합니다.
2.  로드 밸런서는 이 요청을 웹 서버 1 또는 웹 서버 2 중 한 곳으로 전달합니다.
3.  요청을 받은 웹 서버는 페이지를 만들기 위해 필요한 데이터를 데이터베이스 서버에 요청합니다.
4.  데이터베이스 서버는 데이터를 웹 서버에게 전달합니다.
5.  웹 서버는 받은 데이터로 최종 HTML 페이지를 완성하여 로드 밸런서를 통해 클라이언트에게 응답합니다.


---

## 3. 환경 정보

| 서버 종류       | 역할          | IP 주소         | 설명                                               |
| --------------- | ------------- | --------------- | -------------------------------------------------- |
| **Web Server 1**| Web 통신      | `192.168.56.44` | 로드 밸런서와 연결되는 내부 웹 인터페이스          |
|                 | DB 접속       | `192.168.57.44` | 데이터베이스 서버와 통신                           |
| **Web Server 2**| Web 통신      | `192.168.56.55` | 로드 밸런서와 연결되는 내부 웹 인터페이스          |
|                 | DB 접속       | `192.168.57.55` | 데이터베이스 서버와 통신                           |
| **Load Balancer** | Public 접속   | `192.168.55.66` | 외부 사용자가 접속하는 인터페이스 (Public IP)      |
|                 | 내부 통신     | `192.168.56.66` | 내부 웹 서버들과 통신하는 인터페이스 (Private IP)  |
| **Database Server** | DB 수신       | `192.168.57.33` | 웹 서버들로부터의 DB 연결을 수신                   |

---

**공통 환경**

-   OS: Rocky Linux 9 (RHEL 계열)
-   셸: Bash (root 또는 sudo 필요)

---

## 4. 데이터베이스 서버 구성

가장 먼저 모든 데이터가 저장될 DB 서버를 구축합니다.

### 4.1 MySQL 설치 및 서비스 설정

```bash
$ sudo dnf install -y mysql-server

$ sudo systemctl start mysqld
```

### 4.2 방화벽 설정

```bash
$ sudo firewall-cmd --add-service=mysql --permanent

$ sudo firewall-cmd --reload
```

### 4.3 MySQL 내부 설정

```bash

mysql

mysql> CREATE DATABASE wp;

mysql> CREATE USER 'web'@'192.168.57.%' IDENTIFIED BY 'password';

mysql> GRANT ALL PRIVILEGES ON wp.* TO 'web'@'192.168.57.%';

mysql> FLUSH PRIVILEGES;

mysql> EXIT;
```

---

## 5. 웹 서버 구성

첫 번째 웹 서버(Web Server 1)를 기준으로 순서대로 실행합니다.

### 5.1 Apache 설치 및 서비스 구성

```bash
$ sudo dnf install -y httpd             # Apache 설치

$ sudo systemctl start httpd            # 서비스 시작
```

### 5.2 PHP 및 MySQL 클라이언트 설치

```bash
$ sudo dnf install -y php php-mysqlnd   # PHP 및 MySQL 연동 모듈 설치
```

### 5.3 방화벽 설정

```bash
$ sudo firewall-cmd --add-service=http 

$ sudo firewall-cmd --add-port=80/tcp 

$ sudo firewall-cmd --add-port=3306/tcp 

$ sudo firewall-cmd --reload
```

### 5.4 WordPress 설치 및 배치

```bash
$ sudo curl -o wordpress.tar.gz https://wordpress.org/latest.tar.gz

$ sudo tar xvf wordpress.tar.gz -C /var/www/html

$ sudo cp /var/www/html/wordpress/wp-config-sample.php /var/www/html/wordpress/wp-config.php
```

### 5.5 wp-config.php 내부 설정

```bash
$ sudo vi /var/www/html/wordpress/wp-config.php
    
    #wp-config.php 내부에서 수정

    define( 'DB_NAME', 'wp' );    

    define( 'DB_USER', 'web' );

    define( 'DB_PASSWORD', 'password' );

    define( 'DB_HOST', '192.168.57.33' );
```

### 5.6 Apache 가상 호스트 설정

```bash
$ sudo vi /etc/httpd/conf.d/wordpress.conf

    #wordpress.conf 내부에서 수정

    <VirtualHost *:80>
    DocumentRoot /var/www/html/wordpress
        <Directory "/var/www/html/wordpress">
            AllowOverride All
        </Directory>
    </VirtualHost>
```

### 5.7 health.php 상태 확인 페이지 작성

```bash
$ sudo vi /var/www/html/wordpress/health.php

    #wordpress/health.php 내부에서 수정

    <?php
      echo "Hostname: " . gethostname() . "<br>";
      echo "Server IP: " . $_SERVER['SERVER_ADDR'] . "<br>";
      echo "Client IP: " . $_SERVER['REMOTE_ADDR'] . "<br>";
    ?>
```

### 5.8 SELinux 설정

```bash
$ sudo setsebool -P httpd_can_network_connect_db 1    #httpd_can_network_connect_db 정책을 영구적으로 활성화합니다.
```

### 5.9 서비스 재시작 및 검증

```bash
$ sudo systemctl restart httpd            # Apache 재시작

$ sudo nc -zv 192.168.57.33 3306          # DB 연결 테스트
```

> **health.php 확인:** http://<Web1_IP>/wordpress/health.php

---

## 6. 추가 웹 서버 구성 
두 번째 웹 서버(Web Server 2)는 **5장. 웹 서버 구성**의 절차를 동일하게 수행합니다.

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

```bash

/etc/haproxy/haproxy.cfg  
# 생략

#---------------------------------------------------------------------
# frontend main
#---------------------------------------------------------------------
frontend main
    bind *:80
    acl url_static       path_beg       -i /static /images /javascript /stylesheets
    acl url_static       path_end       -i .jpg .gif .png .css .js
    use_backend static          if url_static
    default_backend             app

#---------------------------------------------------------------------
# backend static
#---------------------------------------------------------------------
backend static
    balance     roundrobin                      // 부하 분산 방식: 라운드로빈
    server web1 192.168.56.44:80 check
    server web2 192.168.56.55:80 check

#---------------------------------------------------------------------
# backend app
#---------------------------------------------------------------------
backend app
    balance     roundrobin
    option httpchk GET /wp-login.php
    server web1 192.168.56.44:80 check
    server web2 192.168.56.55:80 check
    http-response add-header X-Server-ID %[srv_name]
```

### 7.3 SELinux 및 서비스 시작

```bash
$ sudo setsebool httpd_can_network_connect_db 1

$ sudo setsebool haproxy_connect_any 1

$ sudo systemctl restart haproxy.service

$ sudo systemctl start haproxy.service
```

### 7.4 방화벽 구성

```bash
$ sudo firewall-cmd --permanent --add-port=80/tcp

$ sudo firewall-cmd --reload
```

> 로드 밸런서 IP(192.168.55.66/192.168.56.66)로 접속 시 Web1/Web2로 분산됩니다.

---

## 8. 로드 밸런싱 검증 

### 8.1 X-Server-ID 헤더 이용

HAProxy 설정의 `http-response add-header X-Server-ID %[srv_name]` 라인은 응답 헤더에 실제 요청을 처리한 서버의 이름(예: `web1`, `web2`)을 추가해줍니다. 이를 통해 브라우저 개발자 도구의 '네트워크' 탭에서 로드 밸런싱이 올바르게 동작하는지 확인할 수 있습니다.

<img src="xserver1.png" alt="image.png" width="500"/>

> 위 그림처럼 응답 헤더의 `X-Server-Id` 값이 `web1` 또는 `web2`로 번갈아 나타나는지 확인합니다.

### 8.2 access_log 실시간 확인

각 웹 서버에서 아래 명령어를 실행하여 로드 밸런서로부터 오는 요청이 실시간으로 기록되는지 확인할 수 있습니다.

```bash
# 각 웹 서버에서 개별적으로 실행
$ sudo tail -f /var/log/httpd/access_log
```

> 로드 밸런서 주소로 새로고침 할 때마다 아래 그림처럼 각 서버에 로그가 번갈아 기록되는 것을 확인할 수 있습니다.

<img src="tail.png" alt="image.png" width="700"/>