---
layout: post
current: post
cover: assets/built/images/background/Go-Logo_Blue.png
navigation: True
title: Goroutines
date: 2022-10-16 00:00:00
tags: [backend, programming]
class: post-template
subclass: 'post tag-backend'
author: HeuristicWave
---

Goroutines, Concurrent Programming in Go

<br>

## Intro

해당 포스팅은 [Tucker의 Go 언어 프로그래밍](http://www.yes24.com/Product/Goods/99108736 ) 24장 고루틴과 동시성 프로그래밍 읽고 정리한 내용임을 알립니다.
미루고 미루던 Go 언어 학습을, [글또](https://heuristicwave.github.io/geultto2) 덕분에 올해 Go 언어 학습을 끝마칠 수 있을 것 같습니다. 😵‍💫

<br>

## Goroutines

고루틴은 Go 언어에서 관리하는 경량 스레드입니다. 함수나 명령을 동시에 수행할 때 사용하며, 여러 고루틴을 갖는 프로그램 코딩을 **동시성 프로그래밍**이라고 합니다.
고루틴을 이해하기 위해, 선수 지식들을 알아보겠습니다.

### Thread

메모리 공간에 로딩되어 동작하는 프로그램을 프로세스라고 합니다. 프로세스는 1개 이상의 작업 단위를 가지고 있으며, 이 작업 단위를 스레드라고 합니다.
스레드가 하나면 싱글 스레드 프로세스, 여럿이면 멀티 스레드 프로세스라 합니다.

원래 CPU 코어는 한 번에 한 명령밖에 수행할 수 없습니다. 그러나 스레드가 CPU 코어를 빠르게 교대로 점유하면 동시에 모든 스레드가 실행되는 것처럼 보입니다.

### Context switching

CPU 코어가 여러 스레드를 전환하는 것을 **컨텍스트 스위칭**이라고 합니다. 스레드를 전환하려면 현재 상태를 보관해야 다시 스레드가 전환되어 돌아올 때 마지막 실행 상태부터 이어서 실행이 가능합니다.
이를 위해 스레드의 명령 포인터(instruction pointer), 스택 메모리 등의 정보를 저장하는 데 이것을 **스레드 컨텍스트**라고 합니다.

스레드가 전환될 때마다 스레드 컨텍스트를 저장하고 복원하기 때문에 전환 비용이 발생하고 적정 개수를 넘어 너무 많은 스레드를 수행하면 성능이 저하됩니다.
하지만 **Go 언어에서는 CPU 코어마다 OS 스레드를 하나만 할당해 사용하므로 컨텍스트 스위칭 비용이 발생하지 않습니다.**

<br>

## Goroutines Example

모든 프로그램은 최소 하나의 고루틴을 가지고 있습니다. 이는 메인 루틴으로 `main()` 함수와 함께 고루틴이 시작되고 종료됩니다.
이미 하나의 고루틴이 있으며, 추가로 고루틴을 생성하는 방법은 다음과 같이, `go functionName()` go 키워드와 함께 함수를 호출하는 것입니다.

아래 코드는 2개의 서브 고루틴을 사용한 예시입니다. 어떤 결과가 나올지 예상해 보고, 하단의 결과를 열어 확인해 보세요 😎

```go
package main

import (
	"fmt"
	"time"
)

func PrintHangul() {
	hanguls := []rune{'가', '나', '다', '라', '마', '바', '사'}
	for _, v := range hanguls {
		time.Sleep(300 * time.Millisecond)
		fmt.Printf("%c", v)
	}
}

func PrintNumbers() {
	for i := 1; i <= 5; i++ {
		time.Sleep(400 * time.Millisecond)
		fmt.Printf("%d ", i)
	}
}

func main() {
	go PrintNumbers()
	go PrintHangul()
}
```

<details><summary markdown="span">👀 실행 결과 보기</summary>

해당 코드는 고루틴이 생성되어 있지만, 메인 함수가 먼저 종료되어 아무런 결과도 출력되지 않습니다.
결과를 출력하기 위해서는 서브 고루틴이 모두 실행되고 완료되는 2000ms 보다 많은 시간을 `main()` 함수에 넣으면 됩니다.

이렇게 `time.Sleep(3 * time.Second)` 3000ms를 보장하는 코드를 삽입하면 모든 실행을 보장합니다.

</details>

### 실행 시간 보장하기

생성한 서브 고루틴들의 실행을 보장하기 위해서는 `WaitGroup` 객체를 사용하면 됩니다.
```go
var wg sync.WaitGroup

wg.Add(3)   // 작업 개수 설정
wg.Done()   // 작업이 완료될 때마다 호출
wg.Wait()   // 모든 작업이 완료될 때까지 대기
```

해당 방법을 통해 위에 예시로 소개한 고루틴을 다음과 같이 수정하면 모든 실행을 보장할 수 있습니다.

<details><summary markdown="span">👀 서브 고루틴 기다리기</summary>

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

var wg sync.WaitGroup

func PrintHangul() {
	hanguls := []rune{'가', '나', '다', '라', '마', '바', '사'}
	for _, v := range hanguls {
		time.Sleep(300 * time.Millisecond)
		fmt.Printf("%c", v)
	}
	wg.Done()
}

func PrintNumbers() {
	for i := 1; i <= 5; i++ {
		time.Sleep(400 * time.Millisecond)
		fmt.Printf("%d ", i)
	}
	wg.Done()
}

func main() {
	wg.Add(2)
	go PrintNumbers()
	go PrintHangul()

	wg.Wait()
	// time.Sleep(3 * time.Second)
}
```

</details>

<br>

## Mechanism

고루틴은 명령을 수행하는 단일 흐름으로 **OS 스레드**를 이용하는 **경량 스레드**입니다. 해당 정의를 이해하기 위해 OS 스레드와 고루틴이 어떻게 다른지 알아보겠습니다.
2개의 코어에서 2개의 고루틴이 존재한다 가정하면, 아래 그림과 같이 각 코어 별, OS 스레드에 하나의 고루틴이 실행됩니다.

```shell
 ________           ______________         .''''''''.
|        |         /             /        /    Go    \
| CORE 1 |--------/ OS Thread 1 /---------\ routine1 /
|________|       /_____________/           '........'
 ________           ______________         .''''''''.
|        |         /             /        /    Go    \
| CORE 2 |--------/ OS Thread 2 /---------\ routine2 /
|________|       /_____________/           '........'
```

위 상황에서 고루틴을 하나 더 생성하면, 남는 코어가 없으므로 3번째 고루틴은 다른 고루틴이 실행 완료될 때까지 대기 상태로 멈춰 있습니다.
만약 고루틴 2가 실행 완료되면, 그제야 대기하던 고루틴 3이 실행됩니다.
```shell
 ________           ______________         .''''''''.
|        |         /             /        /    Go    \
| CORE 1 |--------/ OS Thread 1 /---------\ routine1 /
|________|       /_____________/           '........'
 ________           ______________         .''''''''.
|        |         /             /        /    Go    \
| CORE 2 |--------/ OS Thread 2 /---------\ routine2 /
|________|       /_____________/           '........'
                                                ^
 .'!Wait!'.                                     |
/    Go    \______After Goroutin 2 is removed___|
\ routine3 /
 '.!Wait!.'
```

### System Call

커널 서비스를 사용하기 위해 **시스템 콜**을 호출하면, 해당 서비스가 완료될 때까지 대기 상태가 됩니다.
앞선 예시에서는 실행 중인 고루틴이 완료되기까지 대기 상태를 유지했다면, 시스템 콜이 발생한 상황(고루틴 3)에서는 해당 고루틴을 대기열로 보내고
대기하던 다른 고루틴(고루틴 4)을 실행하며 **코어와 스레드 변경 없이** 고루틴만을 이동시킵니다.

```shell
 ________           ______________         .''''''''.
|        |         /             /        /    Go    \
| CORE 1 |--------/ OS Thread 1 /---------\ routine1 /
|________|       /_____________/           '........'
 ________           ______________         .''''''''.
|        |         /             /        /    Go    \
| CORE 2 |--------/ OS Thread 2 /---------\ routine3 /
|________|       /_____________/           '........'
                                                ^
 .'!Wait!'.                                     |
/    Go    \<------ Switch only Goroutin -------|
\ routine4 /  without changing cores and threads
 '.!Wait!.'
```

이와 같이 고루틴을 이용하면 컨텍스트 스위칭과 없이 오직 고루틴만 옮겨 다니므로, 컨텍스트 스위칭 비용이 증가하면서 발생하는 프로그램 성능 저하로부터 자유로워지게 됩니다.

<br>

## 동시성 프로그래밍 주의점

여러 고루틴이 동일한 메모리 자원에 접근하면 값을 변경시키면 **동시성 문제**를 일으킵니다. 이런 문제를 해결하기 위해 한 고루틴이 접근할 때,
**뮤텍스(mutex, 상호 배제)**를 이용하면 다른 고루틴이 자원에 접근하지 못하게 권한을 통제할 수 있습니다.

### Mutex

뮤텍스는 `Lock()` 메서드를 호출해 뮤텍스를 회득하면, 이후에 `Lock()` 메서드를 호출한 고루틴은 앞서 획득한 뮤텍스가 반납될 때까지 대기하게 됩니다.

```go
var mutex sync.Mutex        // 패키지 전역 변수 뮤텍스

func mutexExample() {
    mutex.Lock()            // 뮤텍스를 확보할 때까지 대기
    defer mutex.Unlock()    // 이하 로직은 뮤텍스를 확보한 단 하나의 고루틴만 실행
    ...
}
```

위 예시의 3줄만 작성한다면 프로그램에 뮤텍스를 이용해 **동시성 문제**를 해결할 수 있습니다. 그러나 또 다른 문제가 발생할 수 있습니다.

1. 오직 하나의 고루틴만 공유 자원에 접근하므로, 동시성 프로그래밍으로 얻는 성능 향상을 얻을 수 없음
2. 뮤텍스를 잘못 사용하면, **데드락(Deadlock, 교착 상태)**에 빠져 무한정 대기하게 됨
   
### Deadlock

하나의 프로세스가 2개 이상의 자원을 얻어야 하는 상황에서, 서로 원하는 자원이 상대방에 할당되어 무한히 다음 자원을 기다리는 데드락을 예시를 통해 발생시켜 보겠습니다.

```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

var wg sync.WaitGroup

func diningProblem(name string, first, second *sync.Mutex, firstName, secondName string) {
	for i := 0; i < 100; i++ {
		fmt.Printf("%s 밥을 먹으려 합니다.\n", name)
		first.Lock()
		fmt.Printf("%s %s 획득\n", name, firstName)
		second.Lock()
		fmt.Printf("%s %s 획득\n", name, secondName)

		fmt.Printf("%s 밥을 먹습니다.\n", name)
		time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond)

		second.Unlock()
		first.Unlock()
	}
	wg.Done()
}

func main() {
	rand.Seed(time.Now().UnixNano())

	wg.Add(2)
	fork := &sync.Mutex{}
	spoon := &sync.Mutex{}

	go diningProblem("A", fork, spoon, "포크", "수저")
	go diningProblem("B", spoon, fork, "수저", "포크")
	wg.Wait()
}
```

위 예제는 실행시키면 아래와 같이 어떤 고루틴도 원하는 만큼의 뮤텍스를 확보하지 못해 무한히 대기하게 됩니다. 

```shell
B 수저 획득
A 포크 획득
fatal error: all goroutines are asleep - deadlock!
```
   
### 서로 다른 자원에 접근하기

애초에 같은 자원을 여러 고루틴이 접근하지 않는다면, 멀티코어의 이점을 얻으면서 뮤텍스로 인해 발생하는 문제도 피할 수 있습니다.
각 고루틴에게 서로 다른 자원에 접근하도록 만들기 위해 아래 2가지 방법이 있습니다.

1. 영역 나누기 : 고루틴 간 간섭이 발생하지 않게 각각의 고루틴으로 할당된 작업만 실행
2. 역할 나누기 : **채널**을 활용해 고루틴 간의 간섭을 없애기

Go 언어에서 동시성 프로그래밍을 도와주는 채널과 컨텍스트에 대해서는 다음 포스팅에서 다루도록 하겠습니다.

<br>

## Outro

요약을 덧붙이며 이번 포스팅을 마무리 짓도록 하겠습니다.

1. 고루틴은 경량 스레드로 컨텍스트 스위칭 비용이 발생하지 않습니다.
2. 멀티 코어 머신에서 여러 고루틴을 사용해 성능을 증가시킬 수 있으나, 같은 메모리 영역을 조정하면 문제가 발생합니다.
3. 뮤텍스는 동시에 고루틴 하나만 자원에 접근하도록 조정합니다.
4. 뮤텍스를 잘못 사용하면 데드락 문제가 발생합니다.
5. 작업 분할 방식과 역할 분할 방식으로 뮤텍스 없이 동시 프로그래밍을 가능하게 할 수 있습니다.

소중한 시간을 내어 읽어주셔서 감사합니다! 잘못된 내용은 지적해 주세요! 😃

---
