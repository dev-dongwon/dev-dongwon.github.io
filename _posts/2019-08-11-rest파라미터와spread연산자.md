---
layout: post
title:  "[JS] Rest parameter vs spread operator"
excerpt: "나만 헷갈리는 게 아닐거야... 레스트 파라미터와 스프레드 연산자를 비교해보자"
categories: [JS]
tags: ['JS']
author: dev-dongwon
comments: true
---







함수형 프로그래밍을 배우다 보니, 함수를 합성하고 분해하는 과정에서 굉장히 높은 빈도로 `rest parameter`와 `spread operator`를 쓰는 것을 알 수 있었다. 하지만 ... 으로 시작하는 둘의 모습이 매우 비슷한고로, 난 굉장히 헷갈리기 시작했다.

```js
const f = (a, ...args) => {
    return args;
}
f(1,2,3,4,5); // [2,3,4,5];

function add(a, b, c) {
  return a + b + c ;
}
const args = [1, 2, 3];

add(...args);
// add(...args) === add(1,2,3) true
```



### rest parameter

rest parameter 는 인자에서 나머지 요소들을 모아 **배열로 반환**해준다. `rest parameter`를 선언하기 전 인수를 별도로 정의할 수 있고, 나머지 인수는 개수에 상관없이 rest parameter로 반환된다.

```js
function xyz(x, y, ...z) {
    console.log(x, y, z);
}

xyz(1,2,3,4,5);
// result : 1, 2, [3,4,5]
```



주의할 점은 `rest parameter`를 이용할 경우 항상 마지막 인수에 있어야 한다는 사실이다. 이를 어기면 오류가 발생한다.

```js
function popo (x, ...y, z) {
	console.log(x, y, z);
}

// result : Uncaught SyntaxError: Rest parameter must be last formal parameter
```



기존 ES5에서 가변 인자의 경우, arguments 객체를 통해 인자값을 확인할 수 있었지만 아쉽게도 ES6의 화살표 함수에서는 this, super, arguments가 바인딩 되지 않으므로 쓸 수 없다. 하지만 arguments가 유사 배열로 반환되어 쓰기 사알짝 불편하다는 점과 달리 `rest parameter`와 화살표 함수를 함께 쓸 경우 예쁘게 배열로 받을 수 있다.

```js
var foo = function () {
  console.log(arguments);
};
foo(1, 2); // { '0': 1, '1': 2 }


const bar = (...args) => {
    console.log(args);
}
bar(1,2); // [1, 2];
```



함수형 프로그래밍의 `pipe` 함수를 구현할 때, 첫번째 함수를 따로 클로져로 뺀 다음, `go` 함수를 통해 실행할 때 이렇게 쓰인다.

```js
const pipe = (f, ...fs) => (...as) => go(f(...as), ...fs);
```



### spread operator

배열, 문자열 또는 iterable 객체를 개별 요소로 분리한다. 이게 강력한 이유는 기존에 splice()나 concat() 으로 배열을 조작하거나 붙이는 작업을 더 간편하게 할 수 있기 때문이다.

```js
const arr1 = [0, 1, 2];
const arr2 = [3, 4, 5];
// arr2 의 모든 항목을 arr1 에 붙임
const arr3 = arr1.concat(arr2);


const arr1 = [0, 1, 2];
const arr2 = [3, 4, 5];
const arr3 = [...arr1, ...arr2]; // arr1 은 이제 [0, 1, 2, 3, 4, 5]
```



밑의 예제에서 기존 배열을 변형하지 않고 불변성을 유지한 채, 새로운 배열을 반환한다. side effect가 없다는 점이 가장 큰 매력!

```js
var arr1 = [0, 1, 2];
var arr2 = [3, 4, 5];
// arr2 의 모든 항목을 arr1 의 앞에 붙임
Array.prototype.unshift.apply(arr1, arr2) // arr1 은 이제 [3, 4, 5, 0, 1, 2] 가 됨

var arr1 = [0, 1, 2];
var arr2 = [3, 4, 5];
arr1 = [...arr2, ...arr1]; // arr1 은 이제 [3, 4, 5, 0, 1, 2] 가 됨
```



이렇게 함수의 파라미터로 간편하게 할당할 수도 있다.

```js
// ES6
function foo(x, y, z) {
  console.log(x); // 1
  console.log(y); // 2
  console.log(z); // 3
}
const arr = [1, 2, 3];
foo(...arr);
```



### 그래서 뭐가 다른데?

`rest parameter`는 함수의 파라미터에 가변인자를 받아 배열로 사용하는 것!

`spread operator`는 iterable한 객체를 개별 요소로 분리하는 것!

모양은 다르지만, 구문이 사용되는 context가 다르니 충분히 구분해서 쓸 수 있겠다.



> 출처 : 
>
> [Javascript's three dots ( ... ): Spread vs rest operators](https://scotch.io/bar-talk/javascripts-three-dots-spread-vs-rest-operators543)
>
> MDN 문서