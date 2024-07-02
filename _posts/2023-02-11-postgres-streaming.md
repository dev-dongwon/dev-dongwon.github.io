---
layout: post
title:  "postgresSQL - streaming/logical replication"
author: dev-dongwon
categories: [DB]
tags: ['DB']
comments: true
---
# postgresSQL - streaming/logical replication

### Streaming replication

- DB 들어오기 전, 중간에 로드밸런서를 둬서 primary - replica 분배
- Master DB의 WAL에 기록된 내용을 실시간으로 Slave DB에게 전달
- Config 파일로 설정
- 프로세스 확인 시 Master DB에는 walsender, Standby DB에는 Walreceiver가 떠있음
- pirmary - replica 사이는 아주 작은 지연 시간
- WAL Log를 거의 실시간성으로 전달함으로써 별다른 지연없이 모든 DB가 동일한 값을 저장할 수 있게 하는 것

### async vs sync replication

- 보통 asynchronous 구조.
- async
    - fast response time
    - but, 데이터 정합성 보장 못함
- async
    - longer response time
    - read on replica 일 때 정합성 보장
- 결론은 상황에 따라 다르게 적용해야 됨

### set up

```bash

# postgressql.conf 설정

listen_addresses= "*" => listen to any connection
port=5433 => 5432 쓰는 기존 서버랑 겹칠까봐 포트 다르게

이후 pg_ctl -D /tmp/primary_db start (/tmp/primary_db에 initdb로 우선 설정해야 함)

create user repuser replication; => 레플 유저 만들어주고
vim /tmp/primary_db/pg_hba.conf => pg_hba 설정 들어가서 repuser가 접근 가능하게 주소 추가

pg_basebackup -h localhost -U repuser --checkpoint=fast\
-D /tmp/replica_db/ -R --slot=some_name -C --port=5433
=> 데이터 복사

이후 cd /tmp/replica_db/ 로 찾아들어가서
cat postgresql.auto.conf 로 확인

postgresql.conf 에서 여기도 port 바꿔주고
pg_ctl -D /tmp/replica_db start 하면 레플리카 db 서버도 구동 확인.

이까지 셋업 ㄱㄱ

psql postgres --port=5433 => primary
psql postgres --port=5434 => replica

요렇게 각각 들어가서
primary에 테이블 만들고 생성, 삭제 해보면
replica에도 똑같이 생성됨을 확인할 수 있다
```

## logical replication

- 변경된 데이터를 스트림으로 다른 서버로 보냄
- 테이블 단위로 리플리케이션 될 수도 있음
- master나 slave를 구분해서 디자인된게 아니라 양방향으로 스트림 될 수 있음
- 장점
    - 서로 다른 버전끼리의 복제가 가능
    - OS가 달라도 된다!
    - 15버전부터는 pubsub에 where조건도 추가하여 원하는 데이터만 pub / sub 가능
    - PubSub의 기본원리와 마찬가지로 하나의 Publication에 여러개의 Subscription이 가능
- 단점
    - DDL은 복제되지 않음
        - 이거는 pg_dumb —schema-only 요거로 수동 복제가 가능
    - 스키마가 맞지 않으면 충돌남
    - 시퀀스 데이터는 replicate가 안됨