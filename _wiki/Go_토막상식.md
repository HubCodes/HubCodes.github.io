---
layout  : wiki
title   : Go 토막상식
summary : Go의 빌딩블럭 가지고 놀기
date    : 2021-04-01 20:13:20 +0900
updated : 2021-04-01 20:13:20 +0900
tag     : coding
toc     : true
public  : true
parent  : [[index]]
latex   : false
---
* TOC
{:toc}

WIP

# Struct

구조체는 C처럼 Go도 메모리 공간의 한 영역을 추상화한다고 생각하고 접근했다.

## 생성

Go 책을 보면서 공부하고 있었는데, 이런 식으로 코딩한 부분이 나왔다.

```go
func foo() []*Result {
	var results []*Result
	// ...
	results = append(results, &Result{
		...
	})
	// ...
	return results
}
```

처음에는 C처럼 생각하고 접근해서 의아했다. 언뜻 보기에는 전혀 힙 메모리에 구조체를 할당하는 것처럼
보이지 않았기 때문에, 저렇게 만들면 스택에 생성된 객체를 가리키는 포인터를 넣는 꼴이 아닌가? 싶어서.

근데 아니었다. 이런저런 방법으로 확인해 본 결과 엄연히 저렇게 코딩하면 동적할당이 된다.

```go
type foo struct {
	data int64
}

func bar() *foo {
	return &foo{data:1023}
}

func main() {
	fo := bar()
	println(fo)
}
```

```
  main.go:8		0x9e8			e800000000		CALL 0x9ed		[1:5]R_CALL:runtime.newobject<1>	
  main.go:8		0x9ed			488b442408		MOVQ 0x8(SP), AX	
  main.go:8		0x9f2			48c700ff030000		MOVQ $0x3ff, 0(AX)	
```

스택오버플로우가 알려준 대로 `go tool objdump` 한 결과를 눈치껏 잘라낸 코드인데, 
아마 `runtime.newobject` 함수? 에서 반환된 포인터가 gc에 의해 트래킹되는 메모리 영역이 아닐까 싶다.
그걸 받아서 그 자리에 1023을 넣고 있다. (근데 레지스터 이름이 왜 저런지는 모르겠다)

물론 애시당초부터 다른 언어이므로 당연히 다른게 맞지만 직접 눈으로 확인하고 나니 마음이 편하다.
