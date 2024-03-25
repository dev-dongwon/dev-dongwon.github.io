---
layout: post
title: system design - 1. 한계 극복의 역사
categories: [system design, study]
---
## 스터디를 시작하며

우연찮은 기회로 '가상 면접 사례로 배우는 대규모 시스템 설계 기초' 라는 책을 주제로 함께 스터디를 할 기회를 얻게 되었다.
한 주마다 서로 책을 읽고 발표를 하며 서로의 경험과 생각을 공유하는 스터디인데, 내용을 훑어보니 이 책... 만만찮다.
세부적으로 알아야 할 내용이 많고 말 그대로 책을 이해하기 위해 공부해야 하는 부분도 상당해서 어떻게 책을 제대로 이해할지 상당히 고민이 되었다.

고민을 하다 시스템 디자인의 본질이 뭘까라는 생각까지 가게 되었다.
시스템 디자인이란 결국 '한정된 자원으로 어떻게 현실의 한계를 어떻게 극복할 것인가?'를 다루는 것.
이것을 이해하기 위해선 '어떤 현실의 한계가 있는가?'를 이해하는 것이 가장 중요하다는 결론에 이르렀고, 이러한 관점에서 내용을 나의 언어로 정리하고자 한다.

책을 읽으며 내가 어떤 거인의 어깨에 올라타고 있는지 이해하는 시간이 되길 바라고,
함께하는 스터디원에게도 의미있고 즐거운 시간이 될 수 있게끔 많은 인사이트와 즐거움을 주는 내가 되길 바란다.

## 한대의 서버

*가장 쉬운 개념은 가장 중요하다*

모든 나무는 하나의 씨앗에서 시작되고, 만갈래의 시작에는 하나의 처음이 있으니 이를 근본이라 이른다.
웹의 근본은 html이고, 초기 웹은 이러한 html 형식의 정적 페이지를 서빙하는 것이었다.
팀 버너스 리부터 시작한 세계 최초의 웹 사이트는 [https://info.cern.ch/](https://info.cern.ch/) 인데, h1 태그의 투박한 글씨체로 **home of the first website** 라고 써져있는게 아주 인상적이다.

<div class="mermaid">
sequenceDiagram
    participant Client
    participant DNS
		participant Web Server
    Client->>DNS: 1. Query (https://google.com)
		DNS->>Client: 2. Response (000.000.000.000)
		Client->>Web Server: 3. Request http
		Web Server->>Client: 3. Response html, json
</div>

> 1. DNS 요청
2. IP 주소 반환
3. HTTP 요청 전달
4. 웹 서버가 HTML이나 JSON 응답 반환

DNS는 인간이 가지고 있는 커뮤니케이션의 욕구가 아니었다면 생기지 않았을 것이라고 생각한다.
숫자로 된 주소에 이름을 붙여주고, 그 이름을 말하고, 그 이름으로 함께 커뮤니케이션하는 세상을 꿈꾸는 사람들이 분명히 DNS를 만들었을 것이다.
점점 이름이 많아지며 하나의 네임 서버로는 한계가 올 것임을 일찌감치 깨달았을 것이고, 아래와 링크에서 설명하는 멋진 구조를 통해 현재 우리는 이 블로그를 보고 있다.

[Amazon - what is dns?](https://aws.amazon.com/ko/route53/what-is-dns)

## 데이터베이스 서버 추가

인간이 이토록 진보할 수 있었던 이유는 기록의 힘이다.
지혜를 다음 세대에 전해주는 구전도 일종의 뇌새김을 통한 일종의 기록이다.
다만 이러한 기록은 쉬이 증발되며, 하나의 고정된 형태로 전해질 수 밖에 없는데 이는 static한 페이지를 서빙하는 원시의 서버와 같다.

모든 것을 구전으로 전할 수 없으니, 문자를 통해 사람들은 책을 쓰며 세대를 건너뛰며 지혜를 축적할 수 있게 되었고 비로소 문명을 가지게 되었다.

<div class="mermaid">
sequenceDiagram
    participant Client
    participant DNS
		participant Web Server
		participant DB Server
    Client->>DNS: 1. Query (https://google.com)
		DNS->>Client: 2. Response (000.000.000.000)
		Client->>Web Server: 3. Request http
		Web Server->>DB Server: 4. Request CRUD
		DB Server->>Web Server: 5. Response Data
		Web Server->>Client: 6. Response html, json
</div>

부족사회 단위의 사용자를 넘어 서비스가 문명 단위로 나아가기 위해서 웹 계층과 데이터 계층을 분리해 좀 더 유연하게 외연을 확장하는 방식을 선택했다.
데이터베이스 서버는 RDB라는 도서관에 장르 별로 차곡차곡 데이터를 쌓고, 이 모든게 끝일줄 알았지만...
RDB의 한계는 명확했다.

관계형 데이터베이스라는 말이 무엇이겠는가. 각 테이블은 하나의 서버 안에서 고유한 관계를 가진다.
사용자가 폭발적으로 늘어서 하나의 데이터베이스 서버로 버거우면? 다른 데이터베이스 서버를 구축하면 원래의 데이터베이스 서버에 있던 테이블과 관계를 맺을 수 있는가?
데이터베이스 서버가 달라지면 join 연산이 불가능해지므로 이 같은 경우 scale out 대신 scale up이 강제된다. 그리고 우리는 무한정 탑을 높이 쌓을 수 없다.

불행하게도 세상은 점점 더 많은 데이터 속에서 점점 더 빠른 응답 속도를 요구하게 되었고, 사람들은 데이터를 분산 저장하되 빠른 응답 속도를 보장하는 방법을 찾기 시작했다.
관계가 필요없이 key-value로 저장하고, 한번에 필요한 데이터를 역정규화해서 저장하고, 이러한 데이터를 여러 대의 데이터베이스 서버에 분산해서 저장한다.
이것이 NoSql이다. (그래프, 도큐먼트 등 다양한 방식이 있지만 key-value만 놓고 본다면!)

[NoSql 데이터베이스란](https://www.ibm.com/kr-ko/topics/nosql-databases)

## 로드밸런서

데이터베이스 서버가 여러대 필요한 이유는 트래픽이 몰려서일테다. DB는 잠시 잊고, 이러한 트래픽을 처리하려면 여러대의 웹서버를 둬야 할텐데...
여러 대의 웹서버를 두면 어떻게 이 서버에게 골고루 사용자 요청을 분산할 것인가?

로드밸런서는 이러한 문제를 해결해준다. public IP 로 사용자 요청을 받는 댐을 하나 건설해두고, 이 댐에서 private IP 로 어떤 웹서버로 트래픽이 흐를지 제어해주는 것이다.
하나가 고장나도 또 다른 하나가 대체해주니, 이것을 가용성이 향상된다라고 책에서는 표현하고 있다.
로드 밸런서를 세부적으로 보면 L4 레벨, L7 레벨로 나뉘어진다.

### L4 로드밸런서
- OSI 7계층에서 레이어4가 뭐더라? => Transport layer (TCP, UDP ...) => IP와 PORT를 알고 있다
- IP + port 기준으로 들어오는 트래픽을 부하 분산
- 동일한 기능과 동작을 가진 서버를 여러대 두었을 때, 각각의 서버로 부하 분산 시켜준다 => round robin 방식 주로 씀
- private IP 를 가지고 있으니 외부에서 직접 서버를 공격하기도 어렵다
- 아하, 그래서 virtual server 라고도 하는구나

### L7 로드밸런서
- 레이어7은 뭐더라? => Application layer (우리가 아는 http, https ...)
- 애플리케이션 레벨이니까 url path 로도 부하 분산이 가능하겠네?
- 예를 들어, 특정 url이 /readonly 요런 식으로 끝나면 읽기 전용 서버 => 읽기 전용 레플리카 db => 응답 요런 식으로 사용 가능하겠지?
- 좀 더 구체적인 레벨에서 필요한 서비스로 부하 분산을 할 수 있다

## DB 다중화

사람들은 읽기를 많이 할까, 쓰기를 많이 할까? 답은 명백히 읽기다.
사람들의 행동은 서비스 내에서도 변하지 않아서, 특별한 경우를 제외하고는 읽기 요청이 쓰기 요청보다 압도적으로 많다.

하나의 master를 두고 원본 데이터를 저장하고 얘는 원본 데이터에 영향을 주는 CUD 만 시키자. 나머지는 replica DB를 여러대 두고 읽기만 시키자.
master가 뻗을 경우, replica 중 하나가 자동으로 master로 승격해 새로운 주 데이터베이스 서버가 되게하자.


-- 힝... 정리가 너무 시간이 많이 걸려... 블로깅은 나중에 하고 우선 키워드 중심으로 정리해봅니다.

## 캐시
- 자주 참조되는 데이터를 메모리에 둠 (disk I/O 보다 속도가 월등)
  - redis의 경우 한번에 100,000개 쿼리 처리 가능
- 사용 용도를 고려해야 한다. 단순히 휘발성 데이터라서 캐시에만 둬도 되는지, 혹은 읽기 성능의 향상을 위해서 DB를 따로 두고 잠깐 캐싱하는 용도인지.
  - 후자의 경우 보통 이렇게 많이 쓴다. 캐시에서 먼저 읽고, 없으면 DB 에서 읽고. 여기서 중요한 건 무조건 원본 데이터는 DB 등의 영속적인 저장 공간에 있어야 함
- 만료 정책 중요하다. 너무 짧으면 DB read가 계속되게 되어 캐시 두는 효과가 없고, 너무 길면 DB 원본과 차이가 나고, 만료를 하지 않으면 결국 한정된 용량이 꽉 차게 된다 (늘리기엔 비싸다..!!)
- DB랑 일관성 유지는 시스템 확장 시에 어려운 포인트가 될 수 있다
- 캐시는 캐시일 뿐. 얘가 고장난다고 해서 단일 장애 포인트가 되어서는 안된다.
- 캐시가 다 찼어. 그러면 어떤 기준으로 앞에 있는 데이터를 지울거야? 보통은 LRU (사용된 시점이 가장 오래된 데이터) 정책을 둔다


## CDN

## 무상태 웹계층
-