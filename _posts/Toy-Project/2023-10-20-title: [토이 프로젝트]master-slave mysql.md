---
title:  "master-slave in mysql"

categories:
  - Spring
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

==
// mysql 데이터 준비
$ mysql -u root -p

CREATE DATABASES demo;

CREATE TABLE MyGuests (
    id INT(6) AUTO_INCREMENT PRIMARY KEY,
    firstname VARCHAR(30) NOT NULL,
    lastname VARCHAR(30) NOT NULL,
    email VARCHAR(50) NOT NULL,
    reg_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

insert into MyGuests(firstname, lastname, email)
values ('lee', 'dong', 'dongjun.dev@gmail.com');

// slave 사용자 생성 및 권한 주기
CREATE USER 'slave_user'@'%' IDENTIFIED BY '1234';
GRANT REPLICATION SLAVE ON *.* TO 'slave_user'@'%';
FLUSH PRIVILEGES;

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

==
// mysql 데이터 준비
CREATE DATABASES demo;

CREATE TABLE MyGuests (
    id INT(6) AUTO_INCREMENT PRIMARY KEY,
    firstname VARCHAR(30) NOT NULL,
    lastname VARCHAR(30) NOT NULL,
    email VARCHAR(50) NOT NULL,
    reg_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

insert into MyGuests(firstname, lastname, email)
values ('lee', 'dong', 'dongjun.dev@gmail.com');

==
CHANGE MASTER TO MASTER_HOST='master db 서버 IP' MASTER_USER='slave_user' , MASTER_PASSWORD='1234' 
아까 마스터에서 확인했던 File, position 정보 -> MASTER_LOG_FILE='', MASTER_LOG_POS='';

START SLAVE;

SHOW SLAVE STATUS\G
```
