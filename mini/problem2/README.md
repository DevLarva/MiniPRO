# WordPress 이중 노드 분리형 서버 구축 가이드

> **작성자:** 백대홍  
> **작성일:** 2025-07-30  
> **버전:** 2.0  

---

# 목차

- [WordPress 이중 노드 분리형 서버 구축 가이드](#wordpress-이중-노드-분리형-서버-구축-가이드)
- [목차](#목차)
  - [1. 개요](#1-개요)
  - [2. 필수 패키지 설치 및 서비스 시작](#2-필수-패키지-설치-및-서비스-시작)
    - [2.1 Apache(httpd) 설치](#21-apachehttpd-설치)
    - [2.2 Apache(httpd) 서비스 시작 및 부팅 시 자동 실행](#22-apachehttpd-서비스-시작-및-부팅-시-자동-실행)
    - [2.5 PHP 및 연동 모듈 설치](#25-php-및-연동-모듈-설치)
  - [3. 방화벽 설정](#3-방화벽-설정)
    - [3.1 HTTP 포트 및 MySQL 포트 허용](#31-http-포트-및-mysql-포트-허용)
  - [4. 워드프레스 다운로드 및 배치](#4-워드프레스-다운로드-및-배치)
    - [4.1 WordPress 다운로드](#41-wordpress-다운로드)
    - [4.2 압축 해제 및 배치](#42-압축-해제-및-배치)
    - [4.3 디렉토리 확인](#43-디렉토리-확인)
  - [5. WordPress 설정](#5-wordpress-설정)
    - [5.1 wp-config.php 설정 복사](#51-wp-configphp-설정-복사)
    - [5.2 wp-config.php 편집](#52-wp-configphp-편집)
  - [6. Apache 설정](#6-apache-설정)
    - [6.1 WordPress용 가상 호스트 설정 추가](#61-wordpress용-가상-호스트-설정-추가)
  - [7. 데이터베이스 서버 설정](#7-데이터베이스-서버-설정)
    - [7.1 MySQL 설정](#71-mysql-설정)
  - [8. 방화벽 설정](#8-방화벽-설정)
    - [MySQL 포트 허용](#mysql-포트-허용)
  - [7. MySQL 설정](#7-mysql-설정)
    - [7.1 MySQL 접속](#71-mysql-접속)
    - [7.2 데이터베이스 및 사용자 생성 (MySQL 내부)](#72-데이터베이스-및-사용자-생성-mysql-내부)

---

## 1. 개요

이 문서는 **WordPress와 웹 서버(Apache)**를 실행하는 **웹 애플리케이션 서버**와, **MySQL 데이터베이스**를 운영하는 **데이터베이스 서버**로 분리하여 구성하는 방법을 설명합니다.  
해당 아키텍처는 **단일 서버보다 보안성과 확장성**이 높으며, 실습 환경뿐만 아니라 소규모 서비스 운영에도 적합합니다.  
서로 다른 두 서버 간의 통신을 설정하여, WordPress가 원격 MySQL 데이터베이스에 접근하도록 구성합니다.


---
## 2. 필수 패키지 설치 및 서비스 시작

### 2.1 Apache(httpd) 설치

```bash
sudo dnf install -y httpd
```

### 2.2 Apache(httpd) 서비스 시작 및 부팅 시 자동 실행

```bash
sudo systemctl start httpd
sudo systemctl enable httpd
```

### 2.5 PHP 및 연동 모듈 설치

```bash
sudo dnf install -y php php-mysqlnd
```

## 3. 방화벽 설정

### 3.1 HTTP 포트 및 MySQL 포트 허용

```bash
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --add-port=3306/tcp --permanent
sudo firewall-cmd --reload
```


## 4. 워드프레스 다운로드 및 배치

### 4.1 WordPress 다운로드

```bash
curl -o wordpress.tar.gz https://wordpress.org/latest.tar.gz
```

### 4.2 압축 해제 및 배치

```bash
tar xvf wordpress.tar.gz -C /var/www/html
```

### 4.3 디렉토리 확인

```bash
ls /var/www/html/
ls /var/www/html/wordpress/
```

---

## 5. WordPress 설정

### 5.1 wp-config.php 설정 복사

```bash
cp /var/www/html/wordpress/wp-config-sample.php /var/www/html/wordpress/wp-config.php
```

### 5.2 wp-config.php 편집

```bash
vi /var/www/html/wordpress/wp-config.php
```

```php
define( 'DB_NAME', 'wp' );
define( 'DB_USER', 'wp-user' );
define( 'DB_PASSWORD', 'password' );
define( 'DB_HOST', '192.168.56.33' ); 
// 이때는 DB의 서버 IP를 입력한다.
```

## 6. Apache 설정

### 6.1 WordPress용 가상 호스트 설정 추가

```bash
vi /etc/httpd/conf.d/wordpress.conf
```

```apache
<VirtualHost *:80>
    DocumentRoot /var/www/html/wordpress
    <Directory "/var/www/html/wordpress">
        AllowOverride All
    </Directory>
</VirtualHost>
```

---
## 7. 데이터베이스 서버 설정

### 7.1 MySQL 설정

```bash
MySQL 서버 설치
dnf install -y mysql-server
```
## 8. 방화벽 설정
### MySQL 포트 허용

```bash
sudo firewall-cmd --add-port=3306/tcp --permanent
sudo firewall-cmd --reload
```
## 7. MySQL 설정

### 7.1 MySQL 접속

```bash
mysql
```


### 7.2 데이터베이스 및 사용자 생성 (MySQL 내부)

```sql
CREATE DATABASE wp;

CREATE USER 'wp-user'@'192.168.56.44' IDENTIFIED BY 'password';

GRANT ALL PRIVILEGES ON wp.* TO 'wp-user'@'192.168.56.44';

mysql> SELECT user, host FROM mysql.user;

EXIT;
```

systemctl restart mysqld.service
systemctl enable mysqld.service

