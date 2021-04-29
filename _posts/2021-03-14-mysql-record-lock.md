---
title: Innodb Storage Engine과 lock에 대한 연구 
date: 2021-03-14 23:00:00 +0900
categories: [DB, Mysql]
tags: [DB, Mysql]
---
## 서론
Innodb Storage Engine은 Row Level Locking이라는 특별한 개념을 가지고 있다. 이 Record Lock이라는 개념을 가지면서 Table Level Lock만을 가진 MyIsam 엔진보다 Lock이 걸리는 범위가 좁아졌기 때문에 다량의 CRUD 작업이 일어나는 App에서는 Innodb엔진을 많이 사용하고 있다. 하지만 실제로 lock이 걸렸을 때에만 Innodb Lock Monitor에 표시되기 때문에 디버깅이 어렵다. 이 포스트에서는 Row Level Lock의 종류에 대해서 공부한 부분을 정리하려고 한다. 

## Lock의 종류
Lock의 종류에는 Row-Level Lock과 Table-Level Lock이 존재한다. Innodb에서는 ```LOCK TABLES messages WRITE``` 같이 명시적으로 테이블을 잠그지 않는다면 Table-Level Lock에 대해서 고려할 필요가 없다. 

### Table-Level Lock
테이블락의 종류에는 **Regular lock**, **intention lock**, **auto-increment lock** 3가지가 있다. 