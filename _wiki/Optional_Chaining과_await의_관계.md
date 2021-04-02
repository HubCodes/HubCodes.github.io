---
layout  : wiki
title   : Optional Chaining과 await의 관계
summary : 공통점을 찾아 추상화하자
date    : 2020-05-18 23:23:20 +0900
updated : 2021-04-02 19:24:50 +0900
tag     : coding
toc     : true
public  : true
parent  : [[index]]
latex   : false
---
* TOC
{:toc}


## ⛓ 옵셔널 체이닝?

[JavaScript에서는 실험적으로 존재하는 기능이고](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Operators/Optional_chaining), TypeScript에도 최근 도입되었으며 Swift나 Kotlin과 같은 언어들에서는 널리 사용되고 있는 기능이다.

Optional chaining은 이렇게 사용한다.

```js
function fetchNameFromAPI(apiURL) {
  const { data } = /* 외부에서 데이터를 불러온다 */
  return data?.person?.name;
}
```

외부 api에서 데이터를 받아와서 데이터가 있는지 없는지 확인하고, 만약 있다면 사람의 이름을 꺼내는 코드이다. 만약 Optional chaining이 없었다면 어떤 모습이었을까?

```js
function fetchNameFromAPI(apiURL) {
  const { data } = /* 외부에서 데이터를 불러온다 */
  if (data && data.person) {
    return data.person.name;
  }
  return null;
}
```

데이터가 `null` 이거나 `undefined` 일 수 있기 때문에, 데이터가 있는지 없는지 일일히 확인해야 한다. 만약 확인해주지 않는다면 `TypeError` 가 발생할 테니까.

## ⛔️ await?

await역시 최근 자바스크립트에서 빼놓을 수 없는 핵심 기능이다. 비동기 코드를 마치 _동기적인_ 것처럼 보이게 만들 수 있는 장점 덕분에 널리 사용되고 있기 때문이다.

await은 아래처럼 사용한다.

```js
async function fetchFromAPI(apiURL) {
  const { data } = await axios.get(apiURL);  // axios.get()은 프로미스를 반환한다
  /* 데이터를 이용해 작업을 진행한다 */
}
```

외부에서 _비동기적으로_ 데이터를 가지고 오는 코드이다. 프로미스를 이용했다면 어떤 모습이었을까?

```js
function fetchFromAPI(apiURL) {
  axios.get(apiURL)  // axios.get()은 프로미스를 반환한다
  	.then(data => {
    	/* 데이터를 이용해 작업을 진행한다 */
  	});
}
```

나쁘지는 않지만 썩 보기 좋지는 않다.

## 😠 하나도 안 똑같은데?

가만히 생각해보면, 옵셔널 체이닝은 if문을 축약해 놓은 문법이라고 생각할 수 있다. 실제로 `fetchNameFromAPI` 함수를 구현할 때 Optional chaining을 사용하던, 하지 않던, 결과는 똑같다. 데이터가 없으면 `null` 이나 `undefined` 를 반환할 것이고 데이터가 있으면 그것을 반환할 테니까. 다만 매번 `null` 체크를 하기 귀찮으므로 언어 차원에서 줄이는 기능을 만들어 준 것이다. 다른 말로 하자면, 옵셔널 체이닝은 데이터가 "있으면" 연결된다.

있으면 연결된다니, 무슨 뜻일까? 직관적인 이해를 위해  `?` 연산자 대신 `then` 을 그 자리에 집어넣어 보면:

```js
function fetchNameFromAPI(apiURL) {
  const { data } = /* 외부에서 데이터를 불러온다. data는 null이거나 undefined 일 수 있다! */
  return data.then(data => data.person).then(person => person.name);
}
```

말도 안 되는 의사코드지만 **모든** 데이터 (심지어 `null` 이나 `undefined` 조차도) 에 `then` 이라는 프로퍼티가 무조건 존재한다고 가정하고 코드를 보면 얼추 들어맞는다. 여기서 `null` 의 `then` 은 콜백을 받아도 데이터가 없으므로 아무 동작을 하지 않고 그냥 자기 자신을 리턴할 것이고, 제대로 된 데이터의 `then` 은 뭔가 보여줄 데이터가 있기 때문에 자기 자신을 꺼내서 콜백에게 넘겨줄 것이다.

await도 생각해 보자. await은 프로미스를 기다리는 문법이므로 프로미스를 가지고 설명한 다음 await으로 확장해도 무방하다.

```js
function fetchFromAPI(apiURL) {
  axios.get(apiURL)  // axios.get()은 프로미스를 반환한다
    .then(data => axios.get(data.nextFetchURL))
    .then(anotherData => /* 데이터를 활용해 작업을 진행한다 */);
}
```

`Promise`의 `then` 역시, 바꿔 말하면 if문을 축약해 놓은 문법이라고 생각할 수 있다. 프로미스에서 에러가 발생하는 경우 (reject되는 경우) 를 제외하고 생각하면, 프로미스의 `then` 은 데이터가 생길 때까지 기다렸다가 콜백을 호출해준다고 말할 수 있기 때문이다. 다만 매번 이렇게 콜백을 호출하는 것은 귀찮은 일이므로 await을 언어 차원에서 지원해 주는 것이다. 다른 말로 프로미스는 데이터가 "생기면" 연결다 .

### ✨ 마법의 연산자 `<-`

두 기능을 저렇게 놓고 비교하니 뭔가 유사해 보인다. 이왕 비슷하게 생긴 거, 같은 방법으로 쓸 수 있다면 더 좋지 않을까? 잠시 언어 설계자가 되었다고 생각하고 저 두 기능을 같은 방법으로 쓸 수 있게 해 보자.

핵심은 `then` 이다. `then` 이 if 문을 가지고 있으면서 **문맥**을 연결해 주기 때문이다. 여기서 문맥은 `Optional` 일 수도 있고, `Promise` 일 수도 있다.

`Promise` 를 펼치기 위해 `await` 이라는 문법을 추가한 것에서 착안해서 아래처럼 보이는 문법을 만들 수 있다. 이제부터 이런 함수를 문맥을 연결해준다는 의미로 `context function` 이라 하자.

```js
context function fetchFromAPI(apiURL) {
  const data <- axios.get(apiURL);
  const anotherData <- axios.get(data.nextFetchURL);
  /* 데이터를 활용해 작업을 진행한다 */
}
```

`await` 하는 것과 상당히 비슷하다. `context` 를 `async` 로 바꾸고 `<-` 부분을 `= await` 으로 바꾸면 비슷한 정도가 아니라 그냥 같다.

그렇다면 `Optional` 은 어떨까? 아까 위쪽에서 모든 데이터에 어거지로 `then` 을 만들었으므로 똑같이 할 수 있다.

```js
context function fetchNameFromAPI(apiURL) {
  const { maybeData } = /* 외부에서 데이터를 잘 불러온다 */
  const data <- maybeData;
  const person <- data.person;
  return person.name;
}
```

`Promise` 던, `Optional` 이던, `<-` 와 `then` 은 그대로 대응된다. 머리속으로 펼치는 상상을 하면서 다시 펼쳐보자. 콜백으로 넘겨줬던 코드를 읽기 편하게 하기 위해 화살표 왼쪽으로 밀어냈을 뿐이므로 화살표 왼쪽을 그대로 콜백으로 만들면 원상복구가 가능하다.

```js
function fetchFromAPI(apiURL) {
  axios.get(apiURL)
    .then(data => axios.get(data.nextFetchURL))
    .then(anotherData => /* 작업 */);
}
```

```js
function fetchNameFromAPI(apiURL) {
  const { maybeData } = /* 외부에서 데이터를 잘 불러온다 */
  maybeData
    .then(data => data)
    .then(data => data.person)
    .then(person => person.name);
}
```

하나는 비동기이고 하나는 옵셔널이라는 점을 제외하면 사용에 있어서 큰 차이를 보여주지는 않고 있다. 즉 정리해보면:

> "then" 메서드가 있으면 <- 연산자를 만들어서 적용할 수 있다

## 🌚 뭐야 그게 끝이야?

공통점을 찾았으니 추상화를 해서 다시 이런 중복이 생기지 않게 할 수 있다. 그런데 이런 추상화를 할 때는 타입스크립트가 유용하다. 약속을 머리속으로만 남기지 않고 눈에 보이는 코드로 남길 수 있기 때문이다.

```ts
interface Thenable<T> {
  then<U>(callback: (data: T) => Thenable<U>): Thenable<U>;
}
```

갑자기 복잡해졌지만 찬찬히 풀어서 보면 그냥 `then` 메서드가 있어요~ 라는 말을 타입스크립트로 표현한 것 뿐이다. `then` 은 "아무 타입 `T` 를 받아서 `Thenable` 로 감싸진 아무 타입 `U`를 반환하는 함수" 를 받아서 "`Thenable` 로 감싸진 아무 타입 `U`" 타입의 객체를 반환한다.

```ts
class Promise<T> implements Thenable<T> {
  then<U>(callback: (data: T) => Promise<U>): Promise<U> { ... }
}
```

비슷한 맥락으로 `Optional` 을 만들어 보자. 아까는 모든 데이터라고 뭉뚱그렸지만 모든 데이터가 `Optional` 은 아니기 때문에 좀 더 정확하게 만들 필요가 있다.

```ts
class Optional<T> implements Thenable<T> {
  constructor(private data: T | null | undefined) {}

  then<U>(callback: (data: T) => Optional<U>): Optional<U> {
    if (this.data) {
      return callback(this.data);
    }
    return new Optional<U>(null);
  }
}
```

### 결론

옵셔널과 프로미스는 비슷한 맥락을 공유하는 기능이라고 볼 수 있다. 이 글에서는 그런 기능을 추상화해서 `Thenable` 이라고 부르기로 약속했고, 비록 실제 js에는 없지만 머리속으로 마법의 연산자 `<-` 를 만들어 보기도 했다.

---

그런데 사실 이 글에 여태 써놓은 내용들은 선구자들(?) 이 이미 발견해서 더 체계적으로 정리해 둔 개념이 있다. 그에 따르면 `then` 역시 `bind` 나 `flatMap` 이라는 이름으로 흔히 불리고 있다. 거기까지 도달하려면 `Thenable` 을 조금만 더 확장하면 되지만, 이 글의 범위를 넘어서는 일이므로, 그 체계적인 개념이 혹시 궁금하다면 `Thenable` 을 좀 더 확장하면 `Monad` 라고 부른다는 점을 참고하면 좋을 것 같다.

### 예전에 작성한 최초 버전

- [velog](https://velog.io/@krlrhkstk/%EC%98%B5%EC%85%94%EB%84%90-%EC%B2%B4%EC%9D%B4%EB%8B%9D%EA%B3%BC-await%EC%9D%80-%EA%B0%99%EC%8A%B5%EB%8B%88%EB%8B%A4)