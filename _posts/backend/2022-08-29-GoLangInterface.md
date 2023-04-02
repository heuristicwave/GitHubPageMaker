---
layout: post
current: post
cover: assets/built/images/background/Go-Logo_Blue.png
navigation: True
title: GO Lang Interface
date: 2022-08-29 00:00:00
tags: [backend, programming]
class: post-template
subclass: 'post tag-backend'
author: HeuristicWave
---

Explain the "interface" and "abstraction" of the go language.

<br>

## Intro

해당 포스팅은 [Tucker의 Go 언어 프로그래밍](http://www.yes24.com/Product/Goods/99108736 ) 20장 인터페이스를 읽고 정리한 내용임을 알립니다.
8월은 31일이고 해당 도서도 31개의 Chapter로 구성되어 있어, *하루에 1장씩 공부하면 Go 언어를 익힐 수 있을 것 같다는 호기로운 생각*이 인터페이스를 만나고 나서 사라졌습니다.
이렇게라도 하지 않으면 올해도 Go 언어 공부를 미룰 것 같아 작성하게 되었습니다. 😵‍💫

<br>

## Interface

인터페이스란 구현을 포함하지 않은 메서드 집합입니다. 구현을 포함하지 않았으므로 인터페이스는 구체화된 타입이 아닙니다. 즉, 추상화된 객체로 상호작용하기 위해 인터페이스를 사용합니다.

**선언 방법**

```go
type(타입 선언) DuckInterface(인터페이스 명) interface(인터페이스 키워드) {
    // 메서드 집합
    Fly()
    Walk(distance int) int
}
```

내부에 선언된 메서드는 반드시 메서드명(`_(x int)` 형태 불가)이 있어야 하며, 이름이 같은 메서드는 함께 있을 수 없습니다.

### 왜 사용할까?

예제를 통해 구체화된 객체가 아닌 인터페이스를 사용함으로써, 프로그램의 변경 요청에 유연하게 대응할 수 있는 방법에 대하여 알아보겠습니다. 

Fedex에서 아래와 같은 패키지 코드를 제공한다고 가정하겠습니다.

```go
package fedex
import "fmt"

type FedexSender struct {}

func (f *FedexSender) Send(parcel string) {
    fmt.Printf("Fedex sends" %v, parcel\n", parcel)
}
```

Fedex가 제공한 패키지를 이용해 상품 배송 기능을 만든다면 다음과 같습니다.

```go
package main

import "github.com/tuckersGo/musthaveGo/ch20/fedex"

func SendBook(name string, sender *fedex.FedexSender) {
    sender.Send(name)
}

func main() {
    sender := &fedex.FedexSender{}
    SendBook("Mastering Go", sender)
    SendBook("Mastering Rust", sender)
}
```

여기서 한국의 우체국이 Fedex의 패키지를 활용해 아래 코드를 작성했다고 가정하겠습니다.

```go
package main

import "github.com/tuckersGo/musthaveGo/ch20/koreaPost"
import "github.com/tuckersGo/musthaveGo/ch20/fedex"

func SendBook(name string, sender *fedex.FedexSender) {
    sender.Send(name)
}

func main() {
    sender := &koreaPost.PostSender{}
    SendBook("Mastering Go", sender)
    SendBook("Mastering Rust", sender)
}
```

해당 코드를 빌드 하면, 우체국과 Fedex의 타입이 달라 다음과 같은 에러를 발생시킵니다. `cannot use sender (variable of type *koreaPost.PostSender) as type *fedex.FedexSender in argument to SendBook`

### 인터페이스로 추상화 계층 만들기

Fedex 패키지를 오류 없이 사용하기 위해서는 `&fedex.FedexSender{}`와 같이 fedex 패키지의 타입과 동일하게 코드를 작성해야 합니다.
그러나 이런 방법은 Fedex 패키지에 의존성이 존재할뿐더러 관리 측면에서도 유연하지 못한 방법이므로, 인터페이스를 사용해 해당 문제를 해결해 보겠습니다. 

```go
type Sender interface {
    Send(parcel string)
}
```

우선, `Send()` 메서드만 포함하는 인터페이스를 작성해 한국의 우체국 코드에 포함합니다.
이어서 `SendBook()` 함수의 인수 `*fedex.FedexSender`를 Sender 인터페이스로 입력받을 수 있도록 코드를 수정하면,
기존의 `SendBook()` 함수는 Sender의 인수가 Fedex 인지, UPS 인지 **어떤 타입이든지 상관없이** 받아들이는 유연한 코드가 됩니다.

<details><summary markdown="span">👀 Interface를 적용한 코드 보기</summary>

```go
package main

import (
  "github.com/tuckersGo/musthaveGo/ch20/koreaPost"
)

type Sender interface {
  Send(parcel string)
}

func SendBook(name string, sender Sender) {
  sender.Send(name)
}

func main() {
  sender := &koreaPost.PostSender{}
  SendBook("Mastering Go", sender)
  SendBook("Mastering Rust", sender)
}
```

</details>

이처럼 Sender 인터페이스 정의 시 인터페이스 구현 여부를 명시적으로 드러내지 않고 메서드 포함 여부로만 결정하는 방식을 **duck typing**이라고 합니다.
덕 타이핑을 통해 내부 동작을 감춰 서비스 제공자(Fedex)와 사용자(우체국) 모두 자유도가 높아졌는데, 이런 방식을 **추상화(abstraction)**라고 합니다.
즉, 인터페이스는 추상화를 제공하는 **추상화 계층(abstraction layer)**이며, 기존의 의존 관계를 끊는 **디커플링(decoupling)**을 가능하게 해줍니다.

<br>

## 인터페이스 기능

지금까지 인터페이스의 기본 기능을 알아보았다면, 이제부터는 아래 3가지 기능에 대해 알아보겠습니다.

- 인터페이스를 포함하는 인터페이스
- 비어있는 인터페이스
- 인터페이스 기본값 nil

### Embedding Interface

구조체에서 다른 구조체를 포함된 필드로 가질 수 있듯이 인터페이스도 다른 인터페이스를 포함할 수 있습니다.

```go
type Reader interface {
    Read() (n int, err error)
    Close() error
}

type Writer interface {
    Write() (n int, err error)
    Close() error
}

// 2개의 인터페이스의 합쳐지면서, 같은 메서드 형식의 Close() error가 하나 메서드만 포합됩니다.
type ReadWriter interface {
    Reader
    Writer
}
```

위 인터페이스는 아래 각각의 타입에 따라, 사용할 수 있는 인터페이스가 다음과 같이 달라집니다.

1. `Read()`, `Write()`, `Close()` 메서드를 포함한 타입 : `Reader/Writer/ReadWriter` 모두 사용 가능
2. `Read()`, `Close()` 메서드를 포함한 타입 : `Reader` 만 사용 가능
3. `Write()`, `Close()` 메서드를 포함한 타입 : `Writer` 만 사용 가능
4. `Read()`, `Write()` 메서드를 포함한 타입 : `Close()` 메소드가 없으므로, `Reader/Writer/ReadWriter` 모두 사용 불가능

### Empty Interface

어떤 값이든 받을 수 있는 함수, 메서드, 변숫값을 만들 때 빈 인터페이스를 사용합니다.

```go
func Sample(s interface{}) {
    x := s.(type)
}
```

`Sample()` 함수는 빈 인터페이스를 인수로 받으므로, 모든 타입을 인수로 사용할 수 있습니다.
이런 특징을 활용하여 `switch` 구문에서 타입별로 다른 로직을 수행하도록 할 수 있습니다.

```go
func Sample(s interface{}) {
    switch x := s.(type) {
    case int:
        fmt.PrintF("s is int %d\n", int(x))
    case string:
        fmt.PrintF("s is string %s\n", string(x))
    default:
        fmt.PrintF("Not supported type: %T:%s\n", x, x)
    }
}
```

### nil Interface

인터페이스 변수의 기본값은 유효하지 않은 메모리 주소를 나타내는 `nil`입니다.
`Attacker`라는 인터페이스가 존재할 때, 아래와 같이 변수 att의 초깃값이 없으므로 해당 값은 **nil**이 됩니다.

```go
func main() {
    var att Attacker
    att.Attack()
}
```

> att의 메모리 주소는 nil이므로 **런타임 에러**가 발생하므로, 인터페이스를 사용할 때는 항상 인터페이스 값이 nil이 아닌지 확인해야 합니다.

<br>

## 인터페이스 변환하기

인터페이스 변수는 타입 변환을 통해서 **구체화된 다른 타입**이나 **다른 인터페이스**로 타입 변환이 가능합니다.

### 구체화된 다른 타입으로 타입 변환하기

**인터페이스 변수 a를 ConcreteType으로 변환하 법**

```go
var a Interface
t := a.(ConcreteType)
```

👀 구체화된 다른 타입으로 변환하는 예시

```go
package main

import "fmt"

type Stringer interface {
	String() string
}

type Student struct {
	Age int
}

func (s *Student) String() string {
	return fmt.Sprintf("Student Age:%d", s.Age)
}

func PrintAge(stringer Stringer) {
	s := stringer.(*Student)                // 3. 인터페이스 변수를 *Student 타입으로 변환
	fmt.Printf("Age: %d", s.Age)
}

func main() {
	s := &Student{15}                       // 1. *Student 타입 변수 s 선언
	PrintAge(s)                             // 2. 변수 s를 인터페이스 인수로 제공
}
```

`main()` 내부에 선언된 구조체 포인터 `*Student` 타입 변수 s를 선언하고(주석 1번), 주석 2번에서 `Stringer` 인터페이스 변수로 `PrintAge()` 함수를 호출했습니다.
이어서 `Stringer` 인터페이스 변수는 `Age`값에 접근할 수 없으므로 주석 3번에서 `*Student`로 타입이 변환되었습니다.
이어서 이러한 구조체 변환 시, 자주 만나는 컴파일 에러를 알아보겠습니다.

#### ❗️ 타입 변환 실패 (컴파일 타임)

인터페이스 변수를 구체화된 타입으로 변환하려면 해당 타입이 인터페이스 메서드 집합을 포함해야 합니다.
예를 들어 방금 예시에서 아래와 같이 가 **String() 메서드**를 포함하지 않는다면, **컴파일 타임** 에러가 발생합니다. 

```go
func (s *Student) String() string {
	return fmt.Sprintf("Student Age:%d", s.Age)
}
```

즉, 위 메서드가 없다면 주석 3번과 같은 `Stringer 인터페이스`에서 `*Student`로 타입 변환이 불가합니다.

### 다른 인터페이스로 타입 변환하기

`ConcreteType`이 `AInterface`와 `BInterface` 인터페이스 모두를 포함하고 있을 경우에는 아래와 같이 다른 인터페이스로 타입 변환이 가능합니다.

```go
var a AInterface = ConcreteType{}
b := a.(BInterface)
```

#### ❗️ 타입 변환 실패 (런 타임)

서로 다른 인터페이스로 타입 변환 시, 서로 다른 메서드 집합을 가지고 있어도 문법적으로 문제가 발생하지는 않습니다.
그러나 경우에 따라, 타입 변환에 실패하여 **런 타임** 에러가 발생할 수 있습니다.

<details><summary markdown="span">👀 다른 인터페이스로 타입 변환이 실패하는 예시</summary>

```go
package main

type Reader interface {
	Read()
}

type Closer interface {
	Close()
}

type File struct {
}

func (f *File) Read() {
}

func ReadFile(reader Reader) {
	c := reader.(Closer)
	c.Close()
}

func main() {
	file := &File{}
	ReadFile(file)
}
```
> Reader 인터페이스 변수를 Closer 인터페이스 타입으로 변환하려 하나, reader 인터페이스 변수가 가리키는 *File 타입이 `Close()` 메서드를 포함하지 않으므로 타입 변환에 실패

</details>

**런 타임** 에러가 발생하는 문제를 방지하기 위해, 아래와 같이 **타입 변환 성공 여부**를 반환하는 코드를 작성할 수 있습니다.

```go
var a Interface
t, ok := a.(ConcreteType)   // t: 타입 변환 결과, ok: 변환 성공 여부
```

<details><summary markdown="span">👀 타입 변환 성공 여부를 반영한 예시</summary>

```go
func ReadFile(reader Reader) {
	c, ok := reader.(Closer)
	if ok {
		c.Close()
	}
	
	/**
	* 한 줄로 표현
	* if c, ok := reader.(Closer); ok {}
	*/
}
```

</details>

## Outro

처음으로 책을 읽고 정리한 내용을 작성했는데, 이해한 내용을 바탕으로 재구성하는 것도 쉽지 않은 것 같습니다.
제가 레퍼런스로 차용한 도서의 저자가 유튜브에 공개한 강의([Tucker Programming](https://www.youtube.com/TuckerProgramming ))와
요약을 덧붙이며 이번 포스팅을 마무리 짓도록 하겠습니다.

1. 인터페이스는 구현을 포함하지 않은 메서드 집합니다.
2. 인터페이스에서 정의 시, 메서드 포함 여부로만 결정하는 덕 타이핑을 통해 자유도 높은 프로그래밍이 가능하다.
3. 인터페이스로 추상화 계층을 만들고 상호작용을 정의한다.
4. 인터페이스는 인터페이스 자체를 포함하거나 빈 상태로 사용할 수 있으며, 기본값은 nil이다.
5. 인터페이스는 구체화된 다른 타입이나 다른 인터페이스로 변환이 가능하며 타입 변환 시 에러를 고려해야 한다.

소중한 시간을 내어 읽어주셔서 감사합니다! 잘못된 내용은 지적해주세요! 😃

---
