---
layout: post
title:  "도대체 dynamoDB에서 local secondary index는 언제 사용하는거야?"
categories: [SQL]
tags: ['SQL']
author: dev-dongwon
comments: true
---

dynamoDB에는 다음과 같은 key를 가지고 있다.

### primary key
- 파티셔닝을 하는 기준이 되는 키
- 파티션 키 수만큼 파티셔닝해서 들어오는 데이터를 각 파티션에 나눠서 적재
- 키 잘못 나누면 특정 파티션에 요청이 몰려서 성능에 영향
- 따라서 카디널리티 높은 걸로 설정하는게 유리

### sort key
- 범위 조회 시 사용
- 모범 사례는 아례와 같음
```
[big#medium#small]
계층적으로 키를 설정하여 범위 검색이 용이하게끔
```

### Index
- 위 key로는 부족하다, 리얼월드는 더 많은 인덱스가 필요함
- index는 local secondary index, global secondary index로 나뉘어짐
- 이게 좀 이해가 안되는 부분이 있는데, 후술함

1. local secondary index
    - 테이블과 동일한 파티션키 사용, sort key 지정
    - 삭제 불가능, 테이블 생성시에만 생성
    - 심지어 10기가 용량 제한도 있음, aws 문서에서는 지속적으로 체크하는게 좋다고 하는데 그게 어디 말처럼 쉽나?
2. global secondary index
    - primary key 필수, sort는 옵션
    - 테이블 생성 후에도 삭제, 생성 가능
    - 용량 제한 없음


### 그래서 언제 local secondary index를 사용해야 되나? 단점밖에 보이지 않는데?
- 더 찾아보니 이런게 있더라
    - dynamoDB는 3개의 복제본을 가지고 있음. 이 복제본들은 같이 데이터가 동기화되는데, 보통의 경우 이 3개 중 가장 빠른 놈을 가지고 와 데이터를 가져오게됨. 이걸 Eventual Consistent Read 라고 함.
    - 당연히 정확하지 않을 수 있음.
    - 근데 dynamoDB는 Strong Consistent Read도 지원함. 상대적으로 응답 속도는 느리지만 쓰기 작업 완료가 최종적으로 된 후, 데이터를 가져오므로 정확함.
    - 이 기능은 primary key, sort key, 그리고 **local secondary index** 이 세가지 인덱스만 지원함.
    - 아주 민감하고 정확한 데이터를 원한다면 local을 고려해야 하는 상황도 올 수 있을듯.
    - 더불어 global secondary index와 테이블은 각각 따로 용량 유닛을 사용한다. 글로벌 보조 인덱스로 뒀을 때 지나치게 많은 데이터를 쿼리해야 할 경우, 로컬을 쓰면 성능 저하 이슈도 막을 수 있다. 
