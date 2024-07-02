---
layout: post
title:  "[NODE] 노드의 내부를 파헤쳐보자 (1)"
excerpt: "노드는 무엇이고 어떻게 동작하는가"
categories: [NODE]
tags: ['NODE']
author: dev-dongwon
comments: true

---



## node internals

![internal-node](https://user-images.githubusercontent.com/43179397/64483701-16944100-d242-11e9-8ced-e971db57a21c.png)



### node가 사용하는 것들

#### libuv 라이브러리 (C++)

- access file system
- networking



#### V8 엔진

- JS 코드 실행



### node의 역할

그냥 직접 V8이나 libuv 라이브러리를 사용하면 되잖아? node의 역할이 뭔데?



#### 내부 코드의 언어 구성

- javascript code : 당연히 100% JS
- node.js : 50% JS, 50% C++
- V8 : 30% JS, 70% C++
- libuv : 100% C++



#### node는 인터페이스다

다 다르지? node를 거치지 않고 직접 엔진이나 라이브러리를 조작하려면 C++를 작성해야 된다는 소리. 우리가 js로 V8과 libuv를 사용하게 하기 위해 node는 존재한다. 정리하면

- **node.js**는 V8 엔진과 libuv 라이브러리를 JS로 조작하기 위한 **인터페이스** 역할을 수행

- libuv에 있는 기능들을 한 번 더 wrapping해서 우리가 쓸 수 있도록 편리하게 **모듈화** 해놓음



### 내부 동작



![internal-node1](https://user-images.githubusercontent.com/43179397/64483559-d4b5cb80-d23e-11e9-8f82-7f8226527113.png)



#### git에 있는 node 폴더 구조를 보면

- lib : 모든 js function과 module을 정의
- src : lib 에 정의해놓은 것들을 C++로 구현



#### internal flow

- 우리가 js 코드를 작성하면 내부에 있는 node library를 통해 정의된 동작을 찾게 되고
- 이 과정에서 실제 구현되어 있는 C++ 코드와 바인딩된다 (상단에 internalBinding이라고 명시되어 있다)
- V8 엔진을 통해 JS를 C++로 convert 한다 (C++ 구현 코드를 보면 using v8::something 이 엄청나게 많이 있다)
- libuv로 OS에 접근해 작업을 수행한다



> 출처 : [udemy - Node.js : Advanced Concepts](https://www.udemy.com/advanced-node-for-developers/)

