---
title:  "Go언어에서의 동시성 관리"
last_modified_at: 
categories: 
  - Go
tags:
  - Concurrency 
toc: true
toc_label: "Golang"
---

The Go Programming Language 원서 9장에 해당하는 내용입니다.

### Intro
Go를 Goroutine 때문에 쓴다는 사람들이 있을 정도로 동시성에 있어서 Go는 평이 좋은 언어다.

Go 에서는 Channel 을 사용해서 공유되는 변수들을 쉽게 관리 할 수 있는데 여기에서는 다른 언어들에서도 제공하는 Mutex Lock/Unlock 위주로 설명한다. Concurrency 철학이나 왜 Go가 동시성을 다루기 좋은지는 따로 포스트를 작성하겠다.

## Race conditions

Go로 작성한 절차지향적 프로그램에서는 하나의 goroutine만 돌아간다. main 함수가 하나의 goroutine 이며 여기에서는 실행순서가 프로그램 로직에 의해서 모두 결정된다. Goroutine이 2개 이상인 경우부터는 순서가 유지된다는 봊장이 없어진다. 같은 함수를 실한하는 Goroutine A 와 Goroutine B 가 있고 실행순서에 제약을 걸어놓지 않았다면 둘중 어느 Goroutine 이 먼저 실행될지 모른다. 이 경우, 2개의 이벤트가 concurrent 하다고 한다.

Concurrecny-safe 라는 용어도 존재하는데 이것은 concurrent 하게 돌려도 정확성이 보장된다는것을 의미한다. 따로 sync를 맞춰주는 작업을 해야한다던가 동시성으로 인해 프로그램의 정확성에 문제가 생긴다면 concurrency-safe 하지 못한것이다. Concurrency safe type은 타입의 메서드와 연산이 모두 안전할때를 지칭하는데, Concurrency-safe type은 일반적으로 기대해서는 안되는 속성이다. 문서에 안전하다고 명시되어있지 않다면 동시성을 위해 mutex를 사용하거나 하나의 goroutine에서만 사용하는등, 따로 조치를 취해줘야한다.

동시성으로 인해 발생하는 문제는 다양하다. Deadlock, livelock, starvation 등이 있는데 race condition을 보겠다. 이 문제는 여러개의 동시적 작업에서 만들어지는 작업순서가 틀린 값을 만들어 낼때를 얘기한다. 컴파일러, 플랫폼, 작업환경, 등 다양한 요인들이 영향을 미칠수 있어서 간헐적으로 일어나기도 하고, 따라서 디버깅 하기 어렵다.

책에서 나온 예제를 잠깐 보겠다.
```
func balance int
func Deposit(amount int) {balancee = balance + amount}
func Balance() int {return balance}
```

```
// Alice
go func() {
  bank.Deposit(200)
  fmt.Println("=", bank.Balance())
}

// Bob
go bank.Deposit(100)
```

Deposit 함수에는 balance를 한번 읽고 새로운 값을 쓰는작업이 한번씩 있다. 따라서 두개의 goroutine에서 총 일어나는 연산은 
Alice reads balance
Alice writes balance
Bob reads balance
Bob writes balance

그리고 Deposit 함수의 로직상, 한번의 호출 안에서 read 가 write 앞에 와야한다.
따라서 Alice reads가 Alice writes 앞에 오고, Bob reads 가 Bob writes 앞에 온다.
두개의 제약으로 인해 경우의 수는 4! / 4 = 6 이 되고 아래가 모든 경우의 수다.
```
AR -> AW -> BR -> BW  Deposit: 300
AR -> BR -> AW -> BW  Deposit: 100
AR -> BR -> BW -> AW  Deposit: 200
BR -> BW -> AR -> AW  Deposit: 300
BR -> AR -> BW -> AW  Deposit: 200
BR -> AR -> AW -> BW  Deposit: 100
```

그리고 결과는 일정하지 않다는 것을 볼 수 있다. 이렇게 최소 하나의 쓰기가 있고 두개 이상의 goroutine들이 동시적으로 같은 값에 접근하면서 생기는 문제가 data race다.

int 는 하나의 machine word (연산단위) 에 들어가지만 interface, string, slice 같은 변수를 동시적으로 다루면서 이러한 문제가 발생하면 경우의 수는 더 다양해진다.

## sync.Mutex
위에 race condition을 해결하는 방법은 순서에 강제성을 어느정도 부여하는 것이다.
추가적 제약없는 동시성 환경에서는 한명의 read 가 write 앞에 온다가 전부였지만 동시성 문제를 해결하기 위해 제약을 추가한다.

Alice가 연산을 시작하면 Alice가 끝날때 까진 Alice 만 작업할수 있고,
Bob도 마찬가지로 연산을 시작하면 본인의 작업이 끝날때까지는 본인만 변수를 다룰 수 있다.

따라서 AR 다음엔 무조건 AW, BR 다음은 무조건 BW 가 나와야한다. 이러면 경우의 수는 단 두개로 줄어든다.

```
AR -> AW -> BR -> BW  Deposit: 300
BR -> BW -> AR -> AW  Deposit: 300
```

그리고 같은 결과값을 보장할 수 있게된다.

이 동시성 관리는 mutex로 해결한다. Mutex는 Mutual Exclusion 을 의미하는데, 하나의 goroutine 에서 접근을 하면 나머지는 접근이 불가능하게 되는것이다. 아래 코드를 보자. 이걸 어떻게 구현했는가에 대한 답은 단순히 변수에 접근할 수 있는 권한을 한번의 호출에 우선 제공해주는 것이다.

```
var (
  mu sync.Mutex
  balance int
)

func Deposit(amount int) {
  mu.Lock()
  balance = balance + amount
  mu.Unlock()
}

func Balance() int {
  mu.Lock()
  b := balance
  mu.Unlock()
  return b
}
```
여러개의 고루틴이 동시에 돌아가면 먼저 mu.Lock() 을 호출하는 고루틴이 Lock을 걸고 그 고루틴이 Unlock을 하기 전까진 그 고루틴만 Lock과 Unlock 사이의 로직을 수행할 수 있다. 이 Lock 과 Unlock 사이에 보호반는 구간을 critical section 이라고도 한다.

Mutex Lock/Unlock은 channel 로도 구현이 가능한데 channel로 표현하면 문을 걸어잠그는 것 보다는 열쇠를 얻어서 들어가는 것처럼 보인다.

```
var (
    sema  = make(chan struct{}, 1) // 플래그 같은 역할을 한다.
    balance int
)

func Deposit(amount int) {
  sema <- struct{}{} // 키를 얻는다.
  balance = balance + amount // 키를 얻으면 잔고를 바꾼다.
  <-sema // 키를 버린다. 이제 다른 고루틴에서 이 키를 가져가서 쓸 수 있게된다.
}

func Balance() int {
  sema <- struct{}{} // 키 받기
  b := balance
  <-sema // 키 버리기
  return b
}
```

이제 Balance()와 Deposit()이 concurrency-safe 하니 이걸 사용한 프로그램들 또한 concurrency-safe 할까?

쓰기 나름이다. 그래서 항상 안전하지는 않다.

```
func Withdraw(amount int) bool {
  Deposit(-amount)
  if Balance() < 0 {
    Deposit(amount)
    return false
  }
}
```
이러한 함수를 여러개의 고루틴에서 동시에 돌린다면? 각 Deposit 과 Balance 안에서는 순서유지가 되지만 여러개의 Balance() 호출과 Deposit() 호출이 뒤죽박죽될것이다.

위 함수에 또 통째로 Lock/Unlock을 걸어버리면 어떨까?
```
func Withdraw(amount int) bool {
  mu.Lock()
  defer mu.Unlock()
  Deposit(-amount)
  if Balance() < 0 {
    Deposit(amount)
    return false
  }
}
```
안타깝게도 Deadlock 이 걸린다. 이 함수에서 락을 한번 걸고 Deposit 에서 락을 걸려고하지만 이전에 호출한 Mutex Lock 이 Unlock 되지 않았기 때문이다. 결국 풀리지 않는 Lock을 영원히 기다리고 Deadlock이 걸리게 된다.

이러한 경우 해결책은 Withdraw함수 에서 balance 를 직접 다루게 하고 거기에 mutex Lock 을 사용하는 것이다.

## sync.RWMutex -  Read/Write mutexes

근데 데이터를 바꾸지 않는다면 여러개의 고루틴이 동시에 접근하는건 괜찮지 않나? 라는 생각을 하는 사람들도 있는데 맞다, 읽기만 한다면 문제 될게 없고, 쓰기또한 일어난다고 해도 쓰기 작업만 피해서 읽으면 된다.
따라서 여러개의 일기작업이 동시에 돌아가는것은 막을 필요가 없을 수 있다.
여러개의 동시 읽기는 허용하고 쓰기작업은 한번에 한개만 허용하는 mutex가 따로 있다. sync.RWMutex 다.

```
var mu sync.RWMutex
var balance int

func Balance() int {
  mu.RLock()
  defer mu.RUnlock()
  return balance
}
```