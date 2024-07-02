---
layout: post
title:  "[NODE] 노드의 내부를 파헤쳐보자 (2)"
excerpt: "이벤트 루프와 싱글 스레드"
categories: [NODE]
tags: ['NODE']
author: dev-dongwon
comments: true
---

# Event Loop

## What is the Event Loop?

node는 싱글스레드 기반이다. 싱글 스레드는 곧 동시에 하나의 작업만을 처리할 수 있다는 것을 의미하는데, node는 어떻게 동시에 많은 작업을 처리하는 것처럼 보이는 것일까? 그 답이 바로 **이벤트 루프**다.

대부분의 커널은 다중 스레드로 여러가지 작업을 처리할 수 있고, node는 이 똑똑한 커널에게 작업을 떠넘긴다. 작업 중 하나가 완료되면 커널이 '나 작업 완료했어!'라고 node에게 알려주게 되고, 이 때 콜백을 **poll**이라는 큐에 추가해서 실행되게 한다.



## Event Loop가 실행되는 조건

node는 최초 실행 후 자신이 실행할 작업들이 있는 지 우선 확인하게 된다. 이 작업들이 있으면 Event Loop가 실행되고, 목록이 비어있으면 바로 프로세스를 종료하게 된다.

node가 확인하는 작업 종류는 다음과 같은 3가지로 나눌 수 있다.

- timers : timer 함수가 있는 작업
- OS tasks : network 요청 같은 OS 기반 작업
- thread-pool : libuv의 thread-pool에 있는 cpu 기반 작업 (file IO 등)



![event-loop](https://user-images.githubusercontent.com/43179397/64487809-0ac47100-d27a-11e9-96cf-3a73fdfb14cc.png)



## Event Loop 구조

**이벤트 루프**는 다음과 같은 몇 개의 **phase**(이하 페이즈)로 구성되어 있다. 각 페이즈 별로 콜백 작업을 수행하기 위한  **FIFO QUEUE**가 존재한다. 이 큐에는 페이즈가 처리할 특정한 이벤트의 콜백들을 넣고, 콜백은 이벤트 루프가 해당 페이즈를 호출 할 때 실행된다. 



```
   ┌───────────────────────────┐
┌─>│           timers          │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │ poll  <─────┤  connections │──────
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │
   └───────────────────────────┘
```



### timers

- setTimeout(), setInterval()과 같이 타이머 함수들의 콜백을 실행

- 이벤트 루프가 순회하며 timers를 보고, timers의 큐에서 작업을 꺼내면서 실제 콜백을 poll 큐에 등록한다.
-  이 말인즉슨 타이머 콜백들은 poll 큐에 등록된 콜백이 처리되고 나중에 처리될 수도 있다는 말. (반대로 poll 큐가 비어있다면 정확히 그 시간에 실행되겠지?)
- 그래서 timer 함수에 지정된 시간은 '몇 분 뒤에 정확히 실행되요'와 같은 개념이 아니라 '적어도 이 시간은 기다리고 최대한 빨리 실행할게요'와 같은 말과 같다.

### pending callbacks

- 페이즈 별로 지정된 콜백이 아닌 모든 콜백들을 실행
- 역시 이벤트 루프가 pending callbacks의 큐에서 작업들을 꺼내고, 실제 콜백을 poll 큐에 등록한다. 따라서 이벤트 루프가 poll 영역을 처리할 때 콜백 내부 로직이 실행된다

### Idle, prepare

- 내부적으로 실행됨. 모든 큐가 비어있으면 이벤트 루프의 주기를 바꾼다고 카더라.(확인 필요)

### poll

- poll 큐가 비어있지 않으면 이벤트 루프가 큐를 순회하며 처리한다
- poll 큐가 비어있고, setImmediate()가 있으면 check 페이즈로 넘어간다. 없으면 페이즈를 돌아다니며 콜백을 기다린다.

### check

- setImmediate() 콜백이 여기서 호출되고 집행

### close callbacks

socket.on('close') 같은 close 콜백 실행



## 예제로 확인해보자

### #1

```js
fs.readFile(filename, () => {
	setTimeout(() => {
		console.log('A');
	}, 0);
	setImmediate(() => {
		console.log('B');
	})
})

// 출처: https://sjh836.tistory.com/149 [빨간색코딩]
```

- 파일 읽기 작업은 블로킹 작업이다. 이벤트 루프는 블로킹 작업을 만나는 즉시 **libuv의 워커 쓰레드 풀**로 작업을 떠넘긴다. (node IO 작업을 저수준으로 이해하려면 **김정환** 님이 번역한 [노드 개발자가 IO 작업을 시작하면 무슨일이 일어날까?](http://jeonghwan-kim.github.io/node/2017/01/27/node-io-deep.html)를 참고해보자. 설명이 자세히 되어있다)

- 워커 쓰레드가 작업을 마친 후, **pending callbacks** 영역의 큐에 콜백을 등록한다.

- 이벤트 루프가 돌아다니면서 **pending callbacks** 큐에 콜백이 있는 것을 발견하고 실행한다. 실행되면 다시 콜백을 **poll** 큐에 콜백이 등록된다.

- 그 다음 페이즈 순서는 **poll** 큐지? 이 **poll** 큐에 등록된 작업이 존재하므로 실행한다.



실행됨과 동시에

- 실행되면 콜백 내부가 실행된다. 콜백 내부에서 처음 만나는 함수가 setTimeout()이지? setTimeout()은 아까 말했듯이 **timers** 페이즈에 등록된다. 등록하고 다음 코드 영역으로 넘어간다.

- 다음 코드 영역은 setImmediate() 함수를 만난다. setImmediate() 함수는 **check** 페이즈에 등록된다.



자, 아까 poll 큐를 비운 시점부터 다시 추적해보자.

- **poll** 페이즈 다음은 뭐지? **check** 페이즈다. **check** 페이즈에서는 큐에 이미 setImmediate() 콜백이 등록되어 있으므로, 이것을 큐에서 꺼내면서 console.log('B')를 실행한다.

- **close callbacks**에서는 할 일이 없으므로 이벤트 루프는 처음 페이즈부터 다시 순회한다.
- **timers** 를 순회할 때, 큐에 setTimeout()이 있음을 발견한다. 큐에서 꺼내면서 콜백 함수를 **poll** 큐에 등록한다.
- **pending callbacks** 페이즈에는 큐가 비워져있다. 건너뛰고 **poll** 페이즈로 가면 큐에 setTimeout()의 콜백이 등록되어 있다. 실행하면서 console.log('A')를 찍는다.
- 이제 모든 큐가 비워졌다. 프로세스가 반환되면서 끝.



### #2

```js
setTimeout(() => { console.log('A'); }, 0);
setImmediate(() => { console.log('B'); });

// 출처: https://sjh836.tistory.com/149 [빨간색코딩]
```

- setTimeout()을 만났으니 **timers**에 등록한다
- 그 다음 setImmediate()을 만났으니 **check** 영역에 등록한다



근데 두 함수 중 무엇이 먼저 시행될지는 **정해지지 않았다**. (슈뢰딩거의 고양이!!) 지금 이벤트 루프가 어느 시점을 지나고 있는 지 모르기 때문에, 만약 timers 페이즈를 지나치지 않았을 경우에는

- **timers** 에 있는 큐에서 작업을 빼내면서 **poll**에 등록한다

- **poll** 영역을 지나며 큐에서 setTimeout() 콜백을 실행하면서 console.log('A')가 찍힌다
- 이후 **check** 영역의 큐에 등록된 setImmediate() 콜백을 실행하며 console.log('B')가 찍힌다.



timers 페이즈를 이미 지났을 경우에는 **check**를 먼저 순회하므로, console.log('B')가 먼저 찍히겠지?



> 출처
>
> [node.js 공식 문서- The Node.js Event Loop, Timers, and process.nextTick()](https://nodejs.org/uk/docs/guides/event-loop-timers-and-nexttick/)
>
> [nodejs의 내부동작원리](https://sjh836.tistory.com/149)

