---
layout  : wiki
title   : Scala Type Lambda가 무엇일까
summary : 문자 식별하기
date    : 2020-04-12 21:46:12 +0900
updated : 2021-03-28 13:52:49 +0900
tag     : coding
toc     : true
public  : true
parent  : [[index]]
latex   : false
---
* TOC
{:toc}

인수 두 개를 받는 함수를 생각해보자. 아래 함수는 `(Int, Int) => Int` 타입이다.

```scala
val add = (x: Int, y: Int) => x + y
```

여기서 인수 하나를 원하는 대로 고정하면 `Int => Int` 타입이 된다.

```scala
val add2 = add(2, _)
println(add2(3))  // 5
```

`_` 를 통해서 부분 적용을 하고 있다. 그런데, 이런 메타포를 함수가 아닌 타입에도 적용할 수 있을까?

```scala
trait Functor[F[_]]

type OptionFunctor = Functor[Option]  // ok
type ListFunctor = Functor[List]  // ok
type EitherFunctor = Functor[Either]  // Error! Either takes two type parameters, expected: one
```

위 예시에서 F 는 한 개의 타입 인수를 받는 타입을 나타내고 있다. 그런데 `EitherFunctor` 타입을 만들 때, `Either` 라는 타입은 두 개의 타입 인수를 받고 있다.

그럼 함수에서 했던 것처럼 타입 하나를 `_` 로 묶어둘 수는 없을까?

```scala
type EitherFunctor = Functor[Either[Int, _]]  // Error! Either[Int, _] takes no type parameters, expected: one
```

그렇게는 할 수가 없다. :(

---

## λ Type lambdas λ

타입 람다는 이런 문제를 해결할 수 있게 해 준다.

```scala
type EitherFunctor = Functor[({ type IntOr[A] = Either[Int, A] })#IntOr]
```

문법은 조금 번잡하지만, 한 개의 타입 인수를 받는 T 라는 타입을 만들어서 Functor 에 넘겨주어야 한다는 목적은 달성했다. 문법이 조금 복잡하므로 한 단계씩 뜯어보자.

```scala
{ type IntOr[A] }  // 타입 파라미터 A를 받는 타입 IntOr를 만든다
{ type IntOr[A] = Either[Int, A] }  // Either에 의도한 타입 Int와 아직 모르는 타입 A를 넘긴다
({ type IntOr[A] = Either[Int, A] })#IntOr  // scala의 Type projection을 이용해 IntOr을 꺼낸다
```

결론적으로 함수에서 그랬던 것처럼 Int 만 Either 에 묶어놓고 나머지는 받고 싶을 때 받을 수 있게 됐다. 👍🏼

## `typelevel/kind-projector`

복잡한 문법은 가독성을 떨어뜨리는 일차적인 요인 중 하나이다. `kind-projector` 는 타입 람다의 복잡성을 덜어내기 위해 만들어진 컴파일러 플러그인이다. 이걸 쓰면 아래처럼 간단하게 처리할 수 있다. (아래 예시들은 모두 Simplify를 위해 Functor를 벗겨냈다. 아래 코드들은 컴파일 에러가 발생한다는 의미이다. 타입 파라미터를 받기 때문이다)

```scala
type IntOr = Either[Int, *]
```

만약 공변이나 반공변이 필요하다면 똑같이 하면 된다.

```scala
type IntOrCovariant = Either[Int, +*]
type ReturnsInt = Function1[-*, Int]
```

이런 식으로 쓰는 문법이 마음에 들지 않으면, 함수처럼 쓸 수도 있다.

```scala
type IntOr = Lambda[A => Either[Int, A]]
```

마찬가지로 공변이나 반공변이 필요하면 적용 가능하다.

```scala
type IntOrCovariant = Lambda[`+A` => Either[Int, A]]
type ReturnsInt = Lambda[`-A` => Function1[A, Int]]
```

## Scala 3

Scala 3에 접어들어서 문법이 조금 바뀌게 되었는데, 아래 내용을 참고해 보면 좋을 것 같다.

[Wildcard Arguments in Types](https://dotty.epfl.ch/docs/reference/changed-features/wildcards.html)

## References

- [underscore.io - Type lambdas](https://underscore.io/blog/posts/2016/12/05/type-lambdas.html)
- [Stack overflow - What is a kind projector](https://stackoverflow.com/questions/39905267/what-is-a-kind-projector)
