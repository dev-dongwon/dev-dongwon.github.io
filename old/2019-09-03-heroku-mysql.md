---
layout: post
title:  "[ENVIRONMENT] HEROKU에서 clearDB로 MySQL 연동하기"
excerpt: "HEROKU에서 MySQL을 쓰는 방법"
categories: [ENVIRONMENT]
tags: ['ENVIRONMENT']
author: dev-dongwon
comments: true
---



# HEROKU에서 MySQL을 써보자

## 여는 글

최근 **Express**와 **mongoDB**, **MySQL**를 이용해 사진을 공유하는 SNS를 만드는 개인 프로젝트를 진행했다. 

프로젝트 이름은 **[DailyFrame](https://dailyframe.herokuapp.com/)**! 일상 속에서 자신을 잡아끄는 사진을 공유하고, 서로 커뮤니케이션 할 수 있는 흔하디 흔한 SNS다. (그래도 일단 예쁘고... 있을 건 다 있다!) 배포 환경은 비교적 쉽고 진입 장벽이 낮은 **HEROKU**를 선택했다.



![image](https://user-images.githubusercontent.com/43179397/64172139-3cf75c80-ce8f-11e9-8116-78e90f1dc617.png)

​																				[DailyFrame 구경하기](https://dailyframe.herokuapp.com/)



Heroku에선 친절하게도 배포 후 DB를 연동하기 위한 수고를 덜어주기 위한 add-on을 제공한다. mongoDB를 이용할 때는 **mLab MongoDB** 라는 add-on을 이용해 DB 서버와 연동 작업을 진행했다. **mLab MongoDB**는 비교적 친절한 UI를 제공하는데, 굳이 Cli 환경이 아니더라도 손쉽게 콜렉션과 도큐먼트를 생성할 수 있다. (구글님의 도움으로 자료도 넘쳐난다!)



## MySQL로 마이그레이션!

배포는 끝났지만 관계형 데이터베이스를 익히기 위해 **mongoDB**에서 **MySQL**로 마이그레이션을 최근에 진행했다. 당연히 DB 서버도 바꿔야 했다. HEROKU에서는 MySQL을 지원하는 DB 서버도 친절하게 만들어주는데 **clearDB**라는 add-on으로 이용할 수 있다.



공식 문서에 친절하게 나와있지만, 구글에 검색해보니 mongoDB와 비교해 비교적 자료가 많지 않았다. 아마도 처음 배포 작업을 시작하는 분들은 여러가지로 막막하고 시행착오를 겪을 수 있다는 생각이 들어 본인이 생각한 가장 손쉬운 이용 방법을 소개하고자 한다. 



## 준비물

- **HEROKU CLI** : 홈페이지의 add-on market에서도 연동할 수 있지만, CLI를 활용하는 것이 훨씬 편리하다. 아직 HEROKU CLI가 설치가 되지 않은 상황이면 [여기](https://devcenter.heroku.com/articles/heroku-cli)를 참고해서 설치해보자.

- **MySQL workbench** : 계정을 만든 후, 이곳에서 remote 접속해 확인할 것이다. (MySql Shell로도 가능)



## 1. clearDB를 연동하자

다시 한 번 설명하자면, HEROKU에서는 MySQL을 연동하는 **clearDB**라는 Add-on을 제공한다. Add-on을 이용하면 클라우드로 이용할 수 있는 DB 서버를 친절하게도 세팅해준다. **[HEROKU DEV CENTER](https://devcenter.heroku.com/articles/cleardb)** 에서 친절하게도 설명을 잘해놨으니 한 번 따라가보자.



- heroku에 접속한다

```shell
$ heroku login
```



- clearDB add-on의 ignite를 설치한다 (ignite가 무료로 사용할 수 있는 커뮤니티 버전이다)

```shell
$ heroku addons:create cleardb:ignite
```



- heroku는 똑똑하게 이미 환경변수로 db remote 주소를 받아놨다. grep으로 주소를 확보해주자.

```shell
$ heroku config | grep CLEARDB_DATABASE_URL
```



- 위 과정을 거쳐 이렇게 DB 서버의 주소가 확보되었다. 끝!

```
mysql://adffdadf2341:adf4234@us-cdbr-east.cleardb.com/heroku_db?reconnect=true
```



## 2. workbench로 연결을 확인하자

1의 과정에서 확보한 주소는 workbench를 접속할 때 필요한 정보들을 모두 포함하고 있다. 워크벤치에 접속하기 위해서는 몇 가지 정보를 입력해야 한다. 새로운 커넥션을 생성하면 다음과 같은 창이 뜬다.



![workbench](https://user-images.githubusercontent.com/43179397/64175653-b9da0480-ce96-11e9-96ca-06591d2b7952.png)



만약 아래의 주소를 1의 과정에서 획득했다고 가정할 경우

```
mysql://adffdadf2341:adf4234@us-cdbr-east.cleardb.com/heroku_db?reconnect=true
```

- Hostname : `us-cdbr-east.cleardb.com`

- Port :  `3306` (공식 문서에 나와있지 않지만 default 값이 적용된다)
- Username :  `adffdadf2341` (임의로 바꿀 수 없다)

- Password : `adf4234` (패스워드는 다시 재발급이 가능하다)



이렇게 입력 후 세팅하고 연결하면 끝! 이제 접속해서 local server에 있는 스키마와 테이블을 옮기자.



## 3. 환경변수 세팅

.env.local에서 개발환경을 위한 MySQL 정보를 이미 세팅해놨을 것이다. HEROKU에서 DB를 이용하기 위해서 위의 정보들을 이용해 환경 변수를 설정해줘야 하는데 이것도 역시 CLI를 이용하면 매우 쉽다.



로컬 환경 변수에 MYSQL_HOSTNAME 이라고 세팅해놨다면 이렇게 CLI를 활용해 환경 변수를 설정해주면 된다.

```shell
$ heroku config:set MYSQL_HOSTNAME=us-cdbr-east.cleardb.com

# Adding config vars and restarting myapp... done, v12
# MYSQL_HOSTNAME: us-cdbr-east.cleardb.com
```



이렇게 환경 변수까지 설정해주면 끝이다! 위의 과정을 모두 정상적으로 진행했다면 다시 **HEROKU APP**에 접속해보면 DB를 이용할 수 있다!



## 마무리

정말 쉽지 않은가? 사실 문서로 옮길 필요가 없을 정도로 **HEROKU**는 공식문서가 정말 친절하게 되어있다. 

하지만 처음 배포를 진행하는 초심자의 입장에서는 그 공식문서를 찾아볼 생각이 나지 않을 뿐더러, 진입 장벽이 만만치 않다는 것을 나 스스로가 너무나 잘 알고 있다.  HEROKU는 비교적 쉽게 환경을 세팅할 수 있고, 그래서 자신의 어플리케이션을 처음 배포하고자 하는 초심자라면 굉장히 훌륭한 선택지다. 

그저 까만 화면이 무섭고, 뭐 하나 잘못하면 터뜨릴까봐 노심초사하는 초심자의 불안한 마음을 이 문서가 조금이라도 덜어줬으면 좋겠다. 나도 AWS로 다이빙하며 겪는 시행착오를 누군가의 친절한 문서가 덜어줄테니까.