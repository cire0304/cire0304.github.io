---
title:  "master-slave in mysql"

categories:
  - Ayu-coupon
tags:
  - [Spring, Java, JPA]

toc: true
toc_sticky: true

date: 2023-10-19
---

```text
== master ==

// mysql 설정 파일
$ vim etc/mysql/mysql.conf.d/mysqld.cnf

bind-adress = 0.0.0.0
server-id = 1 # 유일한 숫자면 아무 숫자나 ok
log_bin = /var/log/mysql/mysql-bin.log
max_binlog_size = 100M

// 특정 DB 동기화
binlog_do_db = demo 

// slave 사용자 생성 및 권한 주기
CREATE USER 'slave'@'%' IDENTIFIED BY '1234';
GRANT REPLICATION SLAVE ON *.* TO 'slave'@'%';
FLUSH PRIVILEGES;

//
create database coupon_db;

// slave 에게 공유할 master 정보 
FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;

-> File, position, Binlog_Do_DB 정보 저장 -> slave 설정에서 다시 사용

UNLOCK TABLES;

== slave ==

// mysql 설정 파일
$ vim /etc/mysql/mysql.conf.d/mysqld.cnf

server-id = 2
log-bin = /var/log/mysql/mysql-bin.log
expire_logs_days = 10
max_binlog_size = 100M

binlog_do_db = demo // master에서 설정했던 이름
relay_log = /var/log/mysql/mysql-relay-bin.log

// msyql
STOP SLAVE;
RESET SLAVE;

CHANGE MASTER TO MASTER_HOST={master db 서버 IP}, MASTER_USER='slave' , MASTER_PASSWORD='1234' , MASTER_LOG_FILE={master에서 확인했던 File}, MASTER_LOG_POS={마스터에서 확인 했떤 position};

START SLAVE;

SHOW SLAVE STATUS\G

// 확인
Slave_IO_Running: Yes 
Slave_SQL_Running: Yes
```
