---
layout: post
title:  "[2019] NHN 200만 동접 게임을 위한 MySQL 샤딩 정리"
author: dev-dongwon
categories: [DB]
tags: ['DB']
comments: true
---

대용량 데이터를 어떻게 처리하는지 이것저것 뒤져보다 아래 영상을 봤는데 꽤 재밌었다.

[[2019] 200만 동접 게임을 위한 MySQL 샤딩](https://www.youtube.com/watch?v=8Eb_n7JA1yA&t=1531s&ab_channel=NHNCloud)

요점은
1. 대형 게임은 스케일이 다르다. 200개의 서버마다 30개의 db에 요청을 하면 커넥션 풀이 60000개가 된다. 이거 큰일남.
2. 최초 생각은 중간에 미들웨어를 둬서 뭔가 정리가 필요하다라는 생각이 듬.
3. 근데 이걸 구현하기에는 시간 빠듯. 대신할 걸 찾자.
4. 찾아보니 ProxySQL 이라는 게 있더라. 여러 개의 채널로 들어오는 커넥션을 멀티플렉싱 (대충 하나로 뭉뚱그린후 역다중화를 하면 원래의 채널 정보를 추출할 수 있다더라) 해서 적은 수의 백엔드 요청으로 바꿔준다.
5. ProxySQL은 쿼리 패턴 매칭을 통해 select는 slave로, master는 insert로 해서 보내는 것이 가능하다
6. 근데 insert, select가 동시에 나오면? 쿼리는 추가될 수 밖에 없다. 
7. 이럴 경우 MySql username을 사용할 수 밖에 없다고 결론냄. 이게 무슨 말이냐면, 각 쿼리를 어디로 보낼 지 따로 테이블을 만들어서 대표키가 3자리는 무조건 마스터로, 2자리는 슬레이브로 보내는 등의 조치를 취함.
8. 스키마 샤딩이라는 것도 있는데, 요거는 논리적으로 A, B 라는 db가 있으면 같은 논리적으로 통합되게 사용함. 한명의 유저가 들어오면 A에 없으면 B를 뒤져야 하는거지. 관리 측면에서 너무 여러개면 복잡하기 때문.
9. 적당히 스키마 샤딩을 해서 묶어놓고, 유저가 증가하면 따로 떼서 샤딩하면 되고, 유저가 적어지면 다시 디비를 하나로 통합하면 된다.
10. 게임은 무조건 유저 중심의 스키마! 유저가 200만 몰려서 병목 현상이 일어나면 이것도 문제다. 이를 위해 routeDB를 둬서 각각의 유저마다! 어디의 DB를 뒤져야 할 지 기록해둔다.
11. 근데! routeDB에 200만이 몰리면 그거도 위험하다! 그 앞에 redis를 둬서 캐싱해둔다. 근데! redis도 불안하다!
12. 이걸 해결하려고 유저가 unique한 값을 발급받으면 murmur hash (쫑 안나고 분포도가 좋다고 한다)를 돌려서 그걸 고정된 값으로 나눈 나머지값을 userDB index로 삼음.

내용이 다소 어려워서 이해가 절반쯤 된 것 같지만, 문제 안에 문제가 있고 그것을 돌파하는 뾰족함이 너무 멋있다. 효과를 설명해주시는데 나중에 확인해보니 3000개를 준비했지만 300개의 커넥션만 사용되었다고 한다 ㄷㄷ