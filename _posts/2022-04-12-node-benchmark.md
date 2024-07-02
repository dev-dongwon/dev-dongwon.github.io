---
layout: post
title:  "[NODE] 노드의 내부를 파헤쳐보자 (4)"
excerpt: "multitask를 수행할 때 thread pool에서 일어나는 일들"
categories: [NODE]
tags: ['NODE']
author: dev-dongwon
comments: true
---



# multitask 상황에서 node의 동작

## 실험

node에서 서로 성격이 다른 여러가지 작업을 수행할 경우, 작업 순서는 어떻게 될까? 테스트를 하기 위해 다음과 같이 성격이 다른 작업들을 하나의 코드에 넣어 보았다.

- FS 모듈 (File IO task)
- http 모듈 (Network task)
- crypto 모듈 (CPU 연산 task)

```js
const https = require("https");
const crypto = require("crypto");
const fs = require("fs");

const start = Date.now();

function doRequest() {
  https
    .request("https://www.google.com", res => {
      res.on("data", () => {});
      res.on("end", () => {
        console.log("http: ", Date.now() - start);
      });
    })
    .end();
}

function doHash() {
  crypto.pbkdf2("a", "b", 100000, 512, "sha512", () => {
    console.log("hash: ", Date.now() - start);
  });
}

doRequest();

fs.readFile("multi.js", "utf8", () => {
  console.log("fs : ", Date.now() - start);
});

doHash();
doHash();
doHash();
doHash();

```



https 응답 속도에 따라 편차는 있었지만, 대부분 결과는 똑같았는데 뭔가 이상한 사실이 관찰된다. 왜 fs 모듈 작업은 hash 작업이 하나가 수행된 다음 완료되는가?

```js
// 1회차
https:  635
hash:  850
fs :  852
hash:  879
hash:  889
hash:  895

// 2회차
https:  637
hash:  854
fs :  855
hash:  868
hash:  888
hash:  952

// 3회차
https:  642
hash:  921
fs :  922
hash:  978
hash:  1037
hash:  1038
```



## 몇 가지 알 수 있는 사실과 이상한 점

### 관찰을 통해 알 수 있는 사실

- https는 libuv thread pool을 사용하지 않고, 바로 OS에 접근해서 사용하기 때문에  fs 및 hash 작업같이 스레드풀을 이용하는 작업의 속도와는 무관하다. 
- file IO 작업을 먼저 시행했음에도 불구하고, 늘 hash 작업이 하나 수행된 다음에 file IO 작업이 수행된다.



### 이상한 점

- 스레드풀의 default 스레드 개수는 4개다. 즉 최대 4개의 작업을 동시에 수행할 수 있는데, 시간을 보면 5개의 작업이 완료된 시간이 크게 차이가 안난다.
- 더 이상한 사실은 스레드풀을 사용하는 hash 작업을 제외하고 https 요청과 파일 읽기 작업만을 수행했을 때 `15 ms` 라는 엄청나게 빠른 속도를 보여준다. 뭔가 병목현상이 일어나고 있다.



결론적으로, 스레드풀 내부에서 뭔가 순서를 정하는 것 같은데... 이 순서는 어떻게 정해지는걸까?



## thread Pool 내부에서 일어나는 일들

### 파일 읽기 작업을 할 때 일어나는 일들

![fs-module](https://user-images.githubusercontent.com/43179397/64515139-fab69b00-d326-11e9-9506-211f5199dc78.png)

우선 thread pool 내부 동작을 살펴보기 전 **파일 읽기 작업**에서 일어나는 일들을 알아보자. 파일 읽기 작업은 fs 모듈이 바로 파일에 접근해서 파일을 읽을 수 없다. 파일 읽기 작업은 크게 2가지 요청을 기다리는 block 지점이 존재하는데



- 진짜 있어? 있으면 정보를 줘 : file 정보를 담은 stats 정보를 요구하고, **정보가 return 될 때까지 기다린다**

- 나 지금 읽는 중이야 : 정보가 있으면 파일 읽기 요청을 하고, **파일을 다 읽을 때까지 기다린다**



위 과정을 거쳐야 비로소 node는 file content를 우리에게 return 하게 된다.



### thread pool에서 일어나는 일들

위 사실을 기억하고 스레드풀 내부에서 일어나는 일들을 알아보자.



- 초기 스레드 할당 모습

![init-thread](https://user-images.githubusercontent.com/43179397/64515816-4a499680-d328-11e9-8008-9f6eca747942.png)

**HTTPS 작업은 스레드풀과 상관없이 OS가 독립적으로 작업을 수행**한다. 그리고 dafault thread가 4개니까 스레드 하나에 하나씩 순서대로 작업이 할당된다. 여기까진 우리가 예상했던 모습과 같다.



- HDD에서 stats 정보를 기다리는 스레드 1번

![thread-2](https://user-images.githubusercontent.com/43179397/64516638-c395b900-d329-11e9-8dc1-7a1f7db0cc6f.png)



스레드 2부터 4까지는 열심히 hash 값을 계산하고 있고, 스레드 1은 위에서 설명했던 프로세스를 수행하고 있다. 스레드 1은 HDD로부터 stats 정보가 오길 기다리고 있다. 아마 이 사이에 스레드풀과 관련없는 https 작업이 완료될 것이다.



- 스레드 1번의 변심 : 하드드라이브의 응답을 영원히 기다릴 수 없어. 다른 작업을 수행하자.

![thread-3](https://user-images.githubusercontent.com/43179397/64517051-8847ba00-d32a-11e9-8214-58c5d97b5647.png)

스레드 1이 HDD로부터 응답을 기다리는 동안, 잠깐 fs 작업은 잊어버린다. 그 대신 대기하고 있던 hash 4번 작업을 pick해서 작업을 수행한다. 그 사이 스레드 2-4 번이 수행하던 작업 중 하나가 끝나고 (아마도 가장 먼저 작업이 할당된 스레드 2번의 작업) console.log('hash')를 출력하게 된다. 이것이 fs 작업 이전 하나의 hash 작업이 완료된 이유이다.



- 스레드 1번 대신 현재 작업이 할당되지 않은 스레드 2번이 HDD 응답을 받는다.

![thread-4](https://user-images.githubusercontent.com/43179397/64517660-c09bc800-d32b-11e9-833a-c275d01125a3.png)

스레드 2번은 OS에게 시스템 콜을 요청해 파일 읽기 작업을 기다린다. 하지만 이전과는 달리 기다리고 있는 작업이 없으므로 작업이 완료되고 console.log('fs')를 출력한다. (fs 작업을 단독으로 수행할 시 `15 ms` 미만의 속도를 보여줬으니 굉장히 빠르게 처리될 것이다.) 그 다음 완료된 hash 작업이 출력된다. 



최종적으로 결과는 이렇게 된다. 

```js
https:  642	// 1. 스레드풀과 상관없이 OS가 작업 수행.
hash:  921  // 2. 스레드 1번이 블로킹 되어 남은 작업을 수행하는 동안 완료된 hash 작업
fs :  922   // 3. 2에서 작업을 완료한 스레드가 fs 작업 마무리
hash:  978  // 4. 나머지 hash 작업 완료
hash:  1037
hash:  1038
```



## thread pool 최적화

위 과정의 문제가 뭘까? fs 단독으로 수행하면 `15 ms` 미만의 빠른 속력을 보여주는 작업이 스레드풀의 내부 동작으로 인해 `922 ms`나 걸리게 된다. 만약, 똑같은 작업을 수행하는데 최소한의 시간을 보장하려면 어떻게 해야 할까? 

file IO 병목현상의 원인은 file IO 작업을 수행하는 스레드가 HDD의 응답을 기다리는 시점에 다른 작업을 처리하기 때문이다. 스레드가 다른 hash 작업을 수행하는 것을 막기 위해 hash 작업을 처리할 스레드를 하나 더 늘려보자.

thread pool의 default thread 개수를 4개에서 5개로 늘리면

```js
// thread dafult 개수 5개로 조정
process.env.UV_THREADPOOL_SIZE = 5;

// 동일한 작업 수행

// 결과
fs :  66
hash:  583
hash:  585
hash:  597
hash:  610
http:  917
```

 

짜잔! fs 모듈을 사용하는 file IO 작업의 속도가 획기적으로 개선된 모습을 볼 수 있다. 수행할 task의 성격과 스레드풀이 가진 스레드의 개수에 따라 효율이 차이가 난다는 사실! file IO와 관련해 병목현상이 나타나는 문제가 있다면 스레드풀의 개수를 조정하고 관찰하며 문제를 해결할 수도 있겠다.

