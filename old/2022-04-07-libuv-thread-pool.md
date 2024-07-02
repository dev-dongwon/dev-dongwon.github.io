---
layout: post
title:  "[NODE] 노드의 내부를 파헤쳐보자 (3)"
excerpt: "libuv thread pool, OS delegation for network"
categories: [NODE]
tags: ['NODE']
author: dev-dongwon
comments: true
---



# 노드는 정말 싱글 스레드일까?

## 생각해 볼 만한 실험

밑의 결과는 어떻게 나올까?

```js
const crypto = require('crypto');

const start = Date.now();

// 1
crypto.pbkdf2('a', 'b', 100000, 512, 'sha512', () => {
  console.log('1: ', Date.now() - start);
})
// 2
crypto.pbkdf2('a', 'b', 100000, 512, 'sha512', () => {
  console.log('2: ', Date.now() - start);
})
```



조금씩 차이가 있긴 하지만, 대략 600ms로 거의 동일한 시간이 나온다. 만약 노드가 정말 작업을 처리하는 단위가 싱글 스레드라면 작업을 수행할 수 있는 단위가 하나이므로, 2는 1의 작업이 완료된 후 수행이 된다. 따라서 동일한 시간이 나올 수가 없다.

누군가 이 작업을 동시에 병렬적으로 수행한다는 결론이다. 이 작업을 수행하는 주체는 누구일까?



## libuv thread pool

![thread-pool](https://user-images.githubusercontent.com/43179397/64486834-59b7d980-d26d-11e9-83d1-4b971a2025ba.png)



- node는 cpu 자원을 소모하는 비싼 작업은 (offload task) libuv의 4개 스레드(default)로 구성된 스레드풀에 위임
- 스레드가 4개이므로 동시에 4개의 작업이 수행가능하다.
- 작업을 위임하고, 작업이 완료되면 callback을 호출하기 때문에 작업 수행을 순차적으로 기다리지 않아도 된다.



실험을 조금 수정해 6개의 offload task로 구성된 아래 작업으로 증명해보자. 4개의 스레드가 동시에 작업을 수행하므로 작업이 완료되는 순서가 일정하지 않은 상태로 4개의 작업은 거의 동일한 시간이 찍혀야 한다. 최대 스레드가 4개이므로 나머지 2개의 작업은 4개의 스레드 작업이 종료된 후 수행된다. 따라서 2개의 작업은 앞서 수행된 4개의 작업보다 시간이 유의미한 차이가 나야한다.



```js
const crypto = require('crypto');

const start = Date.now();

crypto.pbkdf2('a', 'b', 100000, 512, 'sha512', () => {
  console.log('1: ', Date.now() - start);
})

crypto.pbkdf2('a', 'b', 100000, 512, 'sha512', () => {
  console.log('2: ', Date.now() - start);
})

crypto.pbkdf2('a', 'b', 100000, 512, 'sha512', () => {
  console.log('3: ', Date.now() - start);
})

crypto.pbkdf2('a', 'b', 100000, 512, 'sha512', () => {
  console.log('4: ', Date.now() - start);
})

crypto.pbkdf2('a', 'b', 100000, 512, 'sha512', () => {
  console.log('5: ', Date.now() - start);
})

crypto.pbkdf2('a', 'b', 100000, 512, 'sha512', () => {
  console.log('6: ', Date.now() - start);
})

// 결과
// 1회차
4:  565
1:  568
2:  595
3:  596
5:  1107
6:  1109

// 2회차
4:  565
3:  568
1:  599
2:  600
5:  1108
6:  1108

// 3회차
1:  564
2:  567
4:  574
3:  574
5:  1102
6:  1102
```



예상했던대로, 4개의 작업은 순서와 상관없이 동일한 시간이 기록되었지만 나머지 2개의 작업은 2배의 시간이 걸렸다. 위 실험을 통해 node가 내부적으로 많은 연산이 필요한 작업은 **libuv thread pool**에 위임함을 알 수 있다.



스레드풀의 스레드 갯수를 조정해서 실험한다면?

```js
process.env.UV_THREADPOOL_SIZE = 2;

const crypto = require('crypto');

const start = Date.now();

crypto.pbkdf2('a', 'b', 100000, 512, 'sha512', () => {
  console.log('1: ', Date.now() - start);
})

crypto.pbkdf2('a', 'b', 100000, 512, 'sha512', () => {
  console.log('2: ', Date.now() - start);
})

crypto.pbkdf2('a', 'b', 100000, 512, 'sha512', () => {
  console.log('3: ', Date.now() - start);
})

crypto.pbkdf2('a', 'b', 100000, 512, 'sha512', () => {
  console.log('4: ', Date.now() - start);
})

// 결과
2:  548
1:  549
3:  1098
4:  1100
```



예상했던대로, 2개의 작업은 거의 동시에 끝나고, 나머지 2개의 작업은 앞선 작업이 모두 수행될 때까지 기다린 후, 동시에 작업이 끝난다.



## Network 작업은 OS에 위임한다! (OS delegation)

node의 네트워크 작업은 **OS에 위임**된다. 정확히는 libuv가 OS에 위임한다. 위의 실험과 마찬가지로 default thread가 4개인 상태로 네트워크 요청 8개를 동시에 보내보자.

```js
const https = require("https");

const start = Date.now();

function doRequest() {
  https.request('https://www.google.com', res => {
    res.on('data', () => {});
    res.on('end', () => {
      console.log(Date.now() - start);
    })
  })
  .end();
}

doRequest()
doRequest()
doRequest()
doRequest()
doRequest()
doRequest()
doRequest()
doRequest()

// 결과
402
418
450
462
463
463
463
463
```



libuv의 thread pool을 사용한다면 4개 단위로 끊겨서 결과가 동일하게 수행되어야 하지만, 실험 결과는 8개의 작업 모두 동일한 시간을 보인다. **따라서 거의 모든 네트워크 작업은 libuv thread pool이 아닌, libuv가 위임한 OS에서 일어난다. 다시 말해 node에서 네트워크로 분류된 모든 라이브러리는 OS에서 수행한다고 봐도 된다**.



> 출처
>
> https://www.udemy.com/advanced-node-for-developers