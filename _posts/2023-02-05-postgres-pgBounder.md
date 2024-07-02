---
layout: post
title:  "postgresSQL - pgBouncer"
author: dev-dongwon
categories: [DB]
tags: ['DB']
comments: true
---
# postgresSQL - pgBouncer

### backend connection

- call fork()
- create address space
- create file descripter
- copy memory segment
- 위 작업이 계속 매초마다 일어난다면?

### connection pool
- application과 DB와의 connection을 맺을 때는 많은 communication이 발생
- connection pool이 적용된다면, login하는 절차 X
- 이미 connection을 connection pooler가 가지고 있기 때문
- DB로 들어가는 traffic도 제어 가능

### pgBouncer process

```bash
Client Side : CL_LOGIN, CL_WAITING_LOGIN, CL_ACTIVE, CL_WAITING

Server Side : SV_ACTIVE, SV_IDLE, SV_USED, SV_LOGIN, SV_TEST
```

- Client Side는 client로 부터 받는 요청들이 쌓이는 곳
- Client가 login을 요청했을 때, Query를 실행하려고 했을 때 모두 Client Side와 관련
- DB와 직접적으로 connection을 맺는곳은 X, 일종의 대기열

> CL_LOGIN : Client가 login 시도하는 상태
>
>
> ***CL_WAITING_LOGIN :** Client가 Login하기위해 대기하는 상태*
>
> ***CL_ACTIVE :** Client가 Server Side와 맵핑되어 있는 상태*
>
> ***CL_WAITING :** Client가 Server Side와 맵핑하기 위해 대기하는 상태*
>

- Server Side는 client의 요청을 받아 DB에 접속하는 파트
- DB의 active session들이 실제로 PgBouncer의 Server Side status와 동일

> ***SV_ACTIVE :** DB의 active session과 동일*
>
>
> ***SV_IDLE :** SV_ACTIVE로 사용하다가 query가 끝나고 남아 있는 상태. 일종의 Pool. DB에서도 idle로 connection 유지.*
>
>
> ***SV_USED :** SV_IDLE이 일정 시간(server_check_delay) 이후 미사용되면서 Pool로 전환된 상태. DB에서도 idle로 connection 유지.*
>
>
> ***SV_TEST :** SV_USED가 일정 시간동안 사용되지 않았던 connection이기 때문에 client에게 할당하기전 test query를 실행하는 상태. 일반적으로 select 1; 을 실행.*
>
>
> ***SV_LOGIN :** Server Side에서 DB로 login을 시도하는 상태.*
>

### pool config

***default pool :** PgBouncer의 database에서 pool을 별도로 설정하지 않았을 때, 적용받는 pool size. 일종의 global variable.*

***database pool :** 목적에 따라 분리해서 생성하는 pool로, database별로 pool size를 조절할 수 있음. 여기서 설정하면 default pool size에 영향을 받지 않음.*

***min_pool_size :** 앞서 말한 SV_IDLE + SV_USED 값으로, default pool size와 database pool size 안에서 pool을 확보하는 size.*

- min_pool_size를 반드시 설정해야 pool을 사용함. 걍 default_pool, database_pool은 최대 커넥션 수일 뿐, min_pool_size가 진짜 커넥션풀을 관장하는 ***SV_IDLE, SV_USED 의 총합이므로 주의.***
