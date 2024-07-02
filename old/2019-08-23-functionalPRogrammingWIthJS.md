---
layout: post
title:  "[JS] 함수형 프로그래밍 입문 #4"
excerpt: "함수형 프로그래밍과 ES6+ 정리 #4"
categories: [JS]
tags: ['JS']
author: dev-dongwon
comments: true

---

> 본 강좌는 유인동님의 **함수형 프로그래밍과 JavaScript ES6+** 강좌를 듣고 정리한 것입니다. 개인 학습용으로 유료 강좌 내용 전부가 아닌 유인동님의 깃허브에 공개된 자료를 바탕으로 코드를 요약 정리했습니다.



# 함수형 프로그래밍 입문 #4

## 활용

### 제품 총 수량 구하기

#### step1 :  go함수 활용

```js
go(
	products,
    map(p => p.quantity),
    reduce((a, b) => (a + b))
);
```



#### step2 : 단일 함수로 치환하기

```js
const total_quantity = products => go(products,
    map(p => p.quantity),
    reduce((a, b) => (a + b))
);
```



#### step3 : pipe 활용

```js
const total_quantity = pipe(
    map(p => p.quantity),
    reduce((a, b) => (a + b))
);
```



### 합산 금액 구하기

```js
const total_price = pipe(
    map(p => p.price * p.qauntity),
    reduce((a, b) => (a + b))
);
```



### REFACTORING

#### reduce를 add 함수로 빼기

```js
const add = (a, b) => a + b;

const total_quantity = pipe(
    map(p => p.quantity),
    reduce(add)
);

const total_price = pipe(
    map(p => p.price * p.qauntity),
    reduce(add)
);

```



#### 공통 부분 추상화해서 활용성 높이기

```js
const sum = (f, iter) => go(
	iter,
    map(f),
    reduce(add)
);

// 이렇게 하나를 추상화시켜 놓으면

log(sum(p => p.quantity, products));
log(sum(p => p.quantity * p.price, products));

// 다시 함수를 작성해보면

const total_quantity = products => sum(p => p.quantity, products);
const total_price = products => sum(p => p.price * p.quantity, products);
```



#### curry를 활용해 간결하게 만들기

```js
// sum에 curry를 wrapping 해 주면
const sum = curry((f, iter) => go(
	iter,
    map(f),
    reduce(add)
));

const total_quantity = products => 
	sum(p => p.quantity)(products)

// products는 그냥 전달만 해주므로
const total_quantity = sum(p => p.quantity);
const total_price = sum(p => p.price * p.quantity);

log(total_quantity(products));
log(totla_price(products));

// 추상화 레벨이 높으므로 다음과 같은 동작도 가능하다
// 나이 총합 구하기
log(sum(u => u.age, [
    {age: 40},
    {age: 50}
]));
```

