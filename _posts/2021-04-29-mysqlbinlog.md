---
title: binlog와 mysqlbinlog
date: 2021-04-29 12:00:00 +0900
categories: [DB, Mysql]
tags: [DB, Mysql]
---

## Binlog
### Binlog란?
* binlog는 DML(update, insert, delete) 실행시 시간과 함께 기록되는 로그이다.
* 데이터 복구와 Replication에 사용된다.

### Binlog를 활성화하기 위한 옵션들
* my.cnf에 아래 옵션들을 넣어준다.     
    1. log-bin=[binlog가 저장될 디렉토리와 파일명] 
    2. binlog_format=[ROW, MIXED, STATEMENT] ([참고](http://channy.creation.net/project/dev.kthcorp.com/2011/09/16/mysql-replication-binlog-format-mixed-vs-row/index.html)) 
        * ROW
            * 변경된 행 자체를 BASE64로 Encoding하여 Binary Log에 기록하는 방식
            * Master 에서 실행된 SQL이 Slave 에서 재 실행되지 않으나, 변경된 행이 많은 경우 Binary Log 사이즈가 비약적으로 커질 수 있음
        * MIXED
            * 기본적으로 Statement-Based Type으로 기록되나, 필요에 따라 Row-base Type으로 Binary Log에 기록되는 방식
        * STATEMENT 
            *  실행된 SQL을 그대로 Binary Log에 기록하는 방식
            *  Binary Log 사이즈는 작으나, SQL 실행 시점에 따라 적용되는 결과가 달라질 수 있음 (Time Function, UUID, User Defined Function)
    
    3. expire_log_days=[binlog 보관 기간]
    4. max_binlog_size=[binlog 파일 최대 사이즈]
* 설명되지 않은 더 많은 옵션들은 [mysql 공식 문서](https://dev.mysql.com/doc/refman/5.7/en/replication-options-binary-log.html) 에서 확인하자.

## mysqlbinlog 

### mysqlbinlog란?
* mysqlbinlog는 binary log를 텍스트 형식으로 변환해서 사람이 읽을 수 있게 만들어주는 툴이다.
* 위 툴을 통해서 자신이 원하는 시점으로 복구(PITR)를 할 수 있다. 
* mysqlbinlog를 사용하기 위해서는 binlog가 활성화되어 있어야 한다. 
### 명령어와 옵션 설명
* 기본적인 명령어    
    ```mysqlbinlog [binlog 이름] [option들]```
    
* option들
    * ```--read-from-remote-server```: 원격서버로부터 binlog를 읽을 수 있게 한다. 아래 옵션들과 같이 사용해아 한다. 
        * ```--host```: 원격 서버의 hostname
        * ```--user```: 원격 서버의 user
        * ```--password```: 원격 서버의 password
    * ```--skip-gtids```: gtid 모드를 사용할 때, gtid 정보를 제거하고 출력한다.
        * ```--skip-gtids```를 제거하고 mysqlbinlog를 실행한 결과
            ```sql
              # at 812
              #210428 15:04:31 server id 1  end_log_pos 906 CRC32 0xc68b34a6 	Query	thread_id=18	exec_time=0	error_code=0
              SET TIMESTAMP=1619622271/*!*/;
              create database test
              /*!*/;
              # at 906
              #210428 15:04:47 server id 1  end_log_pos 971 CRC32 0x3ed4895a 	GTID	last_committed=4	sequence_number=5	rbr_only=no
              SET @@SESSION.GTID_NEXT= '3d899deb-a82c-11eb-8f72-3eedb690bbe0:5'/*!*/;
              # at 971
              #210428 15:04:47 server id 1  end_log_pos 1067 CRC32 0x65298f47 	Query	thread_id=18	exec_time=0	error_code=0
              use `test`/*!*/;
              SET TIMESTAMP=1619622287/*!*/;
              create table tt(i int)
              /*!*/;
            ```
        * ```--skip-gtids```를 붙이고 mysqlbinlog를 실행한 결과
           ```sql
            # at 812
            #210428 15:04:31 server id 1  end_log_pos 906 CRC32 0xc68b34a6 	Query	thread_id=18	exec_time=0	error_code=0
            SET TIMESTAMP=1619622271/*!*/;
            create database test
            /*!*/;
            # at 906
            # at 971
            #210428 15:04:47 server id 1  end_log_pos 1067 CRC32 0x65298f47 	Query	thread_id=18	exec_time=0	error_code=0
            use `test`/*!*/;
            SET TIMESTAMP=1619622287/*!*/;
            create table tt(i int)
            /*!*/;
           ```
          
## binlog로 부터 mysql 복구 실습

* 테스트 환경
    ```
    1. 1MASTER - 1SLAVE 구조 
    2. GTID_MODE=ON
    3. binlog_format=ROW
    ```
* 테스트 환경 준비
    1. minikube를 시작한다.
        ```
        minikube start
        ```
    2. minkube에서 [이 스크립트](https://github.com/hyanggeun/mysql-script-k8s) 를 실행한다. 
        * 스크립트를 실행하면 mysql-master-0, mysql-slave-0 pod이 실행될 것이다. 
        * 현재는 mysqlbinlog를 위한 실습이므로 Replication 설정은 건너뛰도록 한다.
    
    3. 먼저 mysqlbinlog가 설치되어 있는 pod을 하나 띄운다.
        ```bash
        kubectl run mysql-test --port="3306" --env="MYSQL_ROOT_PASSWORD=test" --image="mysql:5.7.33"
        ```

* 실제로 binlog로부터 새로운 mysql에 복구를 해보자
    1. master pod에 접속한다.
        ```bash
        kubectl exec -it mysql-master-0 bash
        mysql -uroot -pmaster # mysql master pw = master, slave pw = slave
        ```
    2. 데이터베이스를 만들고, table을 생성해보자.
        ```sql
        CREATE DATABASE test;
        USE test;
        CREATE TABLE tt(i int);
        INSERT INTO tt SET i=1;
        ```
    3. 현재 binlog의 index가 뭔지 확인해본다. 
        ```
        mysql> show binary logs;
        +-------------------+-----------+
        | Log_name          | File_size |
        +-------------------+-----------+
        | binary_log.000001 |       177 |
        | binary_log.000002 |       177 |
        | binary_log.000003 |      1320 |
        +-------------------+-----------+
        ```
    4. binlog를 가져올 수 있는 계정을 하나 만든다. 
          ```sql
          CREATE USER 'repl'@'%' IDENTIFIED WITH 'master';
          GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%'
          ```
    3. mysqlbinlog가 설치되어 있는 pod에 접속한다. 
        ```bash
       kubectl exec -it mysql-test bash
        ```
    4. mysqlbinlog를 실행해보자. gtid_mode가 ON이므로 ```--skip-gtid```를 줘야 gtid를 빼고 sql을 가져온다.
         ```bash
           mysqlbinlog --read-from-remote-server --skip-gtids --host=mysql-master-0.mysql --user=repl --password=master binary_log.000001 > restore.sql
           mysqlbinlog --read-from-remote-server --skip-gtids --host=mysql-master-0.mysql --user=repl --password=master binary_log.000002 >> restore.sql
           mysqlbinlog --read-from-remote-server --skip-gtids --host=mysql-master-0.mysql --user=repl --password=master binary_log.000003 >> restore.sql
         ```
    5. mysql-test pod에 있는 mysql에 restore.sql을 부어서 데이터가 다 있는지 확인한다. 
       ```bash
        mysql -uroot -ptest < restore.sql
       ```
       