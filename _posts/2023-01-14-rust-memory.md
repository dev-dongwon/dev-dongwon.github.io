---
layout: post
title:  "러스트의 소유권"
author: dev-dongwon
categories: [DB]
tags: ['DB']
comments: true
---

## 잠깐 메모리를 보고 가자
- 모든 프로그램은 컴퓨터 메모리의 사용 방법을 관리해야 함.
- 가비지 콜렉터가 있는 것도 있고, 없는 것도 있다. 없으면 직접 관리해줘야함.
- 한편, 메모리 구조는 다음과 같음
  - 코드: 코드 영역
  - 데이터: 정적 변수 및 전역 변수
  - 힙: 동적 할당
  - 스택: 정적 할당(매개 변수, 지역 변수)

- 스택은 따로 메모리 주소를 계산할 필요가 없어서 빠름. 왜? 그냥 일방적으로 쌓기만 하고, 마지막으로 쌓은 메모리 주소만 보고 계산하면 되니까. 그리고 스택에 들어가는 데이터는 이미 고정된 크기를 가지고 있음.
- 한편, 힙은 좀 복잡함. 런타임 시점이라 어떤게 들어갈지 모르니까 매번 저장할 공간이 있는지 따로 계산해야됨. 그러면 운영체제가 야! 여기 공간있다! 라고 하면서 지점을 표시해주는데 우린 이걸 얼로케이트, 할당이라고 부름. 힙에 저장된 데이터에 접근하는건 좀 느린데, 계속 포인터를 따라가야되서 그럼. 여기 가라 저기 가라 계속 돌리니까 메모리 속을 뛰어다니는거지.
- 곧 이 복잡하고 느린 힙을 관리하는게 러스트의 핵심인 '소유권'이라는 말.

## 소유권의 규칙 3가지
- 러스트의 각각의 값은 해당값의 오너(owner)라고 불리우는 변수를 갖고 있다.
- 한번에 딱 하나의 오너만 존재할 수 있다.
- 오너가 스코프 밖으로 벗어나는 때, 값은 버려진다(dropped).

```ts
{  // 아직 선언 안됐으므로 a는 유효하지 않음
  let s = "Hello, world"; // 유효
} // 스코프 끝. 유효하지 않음
```

- 스코프 안에서 s가 등장하면, 유효
- 이 유효기간은 스코프 밖으로 벗어날 때까지 지속

## String 타입의 소유권
- 스트링 리터럴 기억나지? 리터럴은 이뮤터블, 즉 변하지 않는다. 근데 String 타입은 아니야. 변경 가능하고 커질 수 있는 텍스트를 지원하기 위해 만들어진 거니까. 
- 이 말은 곧 런타임 시점에 얘 크기를 알 수 없다는 말이지. 그래서 메모리 공간 요청하고 할당받아야 하는데...
여기서 스트링의 메모리 요청 방법은 2개야.

  1. 직접 수행한다
     String::from을 호출하면 메모리 요청하고.
  2. 가비지 콜렉터가 지워준다

- 러스트는 '스코프'를 활용해서 스코프 밖으로 벗어나는 순간 바로 소유권을 뺏아옴. 괄호가 닫힐때 자동으로 drop을 호출한다네.

근데 이게 러스트 코드 작성하는 방법에 당연히 영향을 주나봐.
이걸 보자.

```rust
let s1 = String::from("hello");
let s2 = s1;
```

스트링은 그냥 스트링 리터럴이 아니라 문자열의 내용물을 담고 있는 메모리의 포인터, 길이, 그리고 용량을 가지고 있음. 이 메타데이터 그룹은 스택에 저장되고, 실제 데이터는 힙에 저장됨. 

근데 위 예제에서 s2 = s1 이라고 대입했을 때, 스택에 있는 데이터만 복사되고 힙에 있는 데이터는 복사되지 않음. 복사할 필요가 없지. 그냥 포인터가 힙에 있는 똑같은 데이터를 가리키면 되는거니까. 힙 데이터도 복사하는 순간 아까 말했던 메모리에서 이리 저리 뛸 수 밖에 없어서 느려짐.

근데 스코프를 벗어나면 소유권을 잃는다고 했는데, 그럼 s2 = s1 이고 힙에 있는 같은 데이터를 바라보는거니까 두 번 메모리를 해제하려고 하겠네? 

### 이동
- 하지만 그런 일은 일어나지 않음. s1이 소유권을 잃으면서 힙에 있는 데이터가 사라지는게 아니라 그대로 s2를 위해 남아줌. 우리는 이걸 'move', 이동이라고 부르기로 했음.

```rust
let s1 = String::from("hello");
let s2 = s1;
println!("{}, world!", s1);
```
당연히 이건 s1이 소유권을 잃고 스택에서 사라진 상태니까 오류가 날거임. 

### 클론
- 근데 우리가 힙데이터까지 깊은 복사 쓰고 싶으면 어떻게함? 이때 클론을 쓰면 됨. 나머지 프로그래밍 언어도 다 유사한듯.

```rust
let s1 = String::from("hello");
let s2 = s1.clone();

println!("s1 = {}, s2 = {}", s1, s2);
```
이건 힙까지 다 복사한거라 위처럼 오류가 나지 않음.

### 복사
```rust
let x = 5;
let y = x;

println!("x = {}, y = {}", x, y);
```
이건 어떻게 될까? 정상적으로 출력됨. 스트링이라는 다르게 정수형은 이미 정해져있는 데이터고, 그래서 스택에 저장되기 때문. 

카피가 가능한 데이터는 다음과 같다
```rust
u32와 같은 모든 정수형 타입들
true와 false값을 갖는 부울린 타입 bool
f64와 같은 모든 부동 소수점 타입들
Copy가 가능한 타입만으로 구성된 튜플들. (i32, i32)는 Copy가 되지만, (i32, String)은 안됩니다.
```

이 이상은 찬찬히 다시 보면서 어떻게 동작하는지 숙지해야지.