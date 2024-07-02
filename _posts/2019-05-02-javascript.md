---
layout: post
title:  "[JS] JSON.stringfy()"
comments: true
excerpt: "JSON.stringfy()를 이해하자"
categories: [JS]
tags: ['JS']
author: dev-dongwon
---

JSON.stringfy()를 제대로 이해하고 써보자


# JSON.stringfy()

JavaScript 값이나 객체를 JSON 문자열로 변환

### 구문
```javascript
JSON.stringify(value[, replacer[, space]])
```
- `replacer`
	- 함수로 전달할 경우 변환 전 값을 변형
	- 배열로 전달할 경우 지정한 속성만 결과에 포함
- `space`
	- 가독성을 목적, JSON 문자열 출력에 공백을 삽입하는데 사용되는 [`String`](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/String "String 전역 객체는 문자열(문자의 나열)의 생성자입니다.") 또는 [`Number`](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Number "Number 객체는 숫자 값으로 작업할 수 있게 해주는 래퍼wrapper 객체입니다. Number 객체는 Number() 생성자를 사용하여 만듭니다. 원시 숫자 자료형은 Number() 함수를 사용해 생성합니다.")객체.
	- 최대값은 10이며, 10을 넘어갈 경우 10으로 작동

### 기본 예제
```js
JSON.stringify({});                  // '{}'
JSON.stringify(true);                // 'true'
JSON.stringify('foo');               // '"foo"'
JSON.stringify([1, 'false', false]); // '[1,"false",false]'
JSON.stringify({ x: 5 });            // '{"x":5}'

JSON.stringify(new Date(2006, 0, 2, 15, 4, 5)) 
// '"2006-01-02T15:04:05.000Z"'
```

### replacer
- `replacer` 매개변수는 함수 또는 배열
- 함수일 때는 key 와 value, 2개의 매개변수를 받음
- key 가 발견된 객체는 리플레이서의 `this` 매개변수로 제공

replacer 함수 활용
```js
function replacer(key, value) {
  if (typeof value === "string") {
    return undefined;
  }
  return value;
}

var foo = {foundation: "Mozilla", model: "box", week: 45, transport: "car", month: 7};
var jsonString = JSON.stringify(foo, replacer);
// JSON 문자열 결과는 `{"week":45,"month":7}`
```

replacer 배열 활용
- `replacer`  배열의 값은 JSON 문자열의 결과에 포함되는 속성의 이름
```js
JSON.stringify(foo, ['week', 'month']);  
// '{"week":45,"month":7}', 단지 "week" 와 "month" 속성을 포함한다
```

### space 
- number 일 경우 number 만큼 indent
- string일 경우 다음과 같이 동작
```js
JSON.stringify({ a: 2 }, null, ' ');
// '{
//  "a": 2
// }'
```
- tab도 쓸 수 있다
```js
JSON.stringify({ uno: 1, dos: 2 }, null, '\t');
// returns the string:
// '{
//     "uno": 1,
//     "dos": 2
// }'
```

>> 출처 : [`MDN : JSON.stringfy()`](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify)