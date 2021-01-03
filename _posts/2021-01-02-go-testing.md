---
title:  "Go언어에서의 테스팅"
last_modified_at: 
categories: 
  - Go
tags:
  - Testing 
toc: true
toc_label: "Golang"
---

The Go Programming Language 원서 11장 (11.1, 11.2)에 해당하는 내용입니다.

### Test files and Test functions

Go 에서 테스트코드는 “_test.go” 로 끝나는 파일들에 정리된다. 테스트파일 안에서 테스팅을 위해서는 testing 패키지를 임포트해야한다. 저장하는 위치는 테스트하려는 파일과 같은경로에 둔다. moby 같은 오픈소스 프로젝트들은 그렇게 정리되어있다.

`go test` 명령어는 _test.go 로 이름이 끝나는 파일들을 스캔하고 위에 언급된 함수들을 찾아내며, 이들을 빌드하고 실행해주는 임시 main package를 만들어낸다. 다 실행항 이후에는 결과를 보고하고 clean up 을 한다.

테스트케이스를 포함한 테스트함수는 이름이 Test로 시작한다. 그리고 성능측정을 위한 벤치마크 함수는 이름이 Benchmark로 시작한다. 그리고 예제를 Documentation에서 보여주는데 사용되는 예시함수(example function) 은 이름이 Example 로 시작한다.

### Test Functions

테스트함수는 아래와 같은 형식으로 정의한다.
```
func TestName(t *testing.T) {
    ...
}
```

여기에서 TestName 이라는 함수명에서 Name의 자리에 들어오는 suffix는 대문자로 시작해야한다.
예를들어 TestSin, TestCos 같은 형식으로 말이다.

테스트 실행하기 (`go test`)

`go test` 명령어를 이용해서 테스트를 빌드+실행할 수 있다.
`-v` 옵션을 주면 실패한 테스트뿐만 아니라 돌아간 모든 테스트케이스들에 대해 출력해준다.

`-run=<regex>` 옵션을 주면 주어진 정규식을 매치하는 테스트들만 선택적으로 돌아간다. 

예를들어 `go test -v -run="Me|You"` 라는 명령어를 돌리면 TestMe, TestYou 같은 테스트함수는 돌아가지만 TestThem 같은 함수는 돌아가지 않는다. 많이 쓸것 같지는 않지만 매우 느린 테스트들을 스킵하고 싶다면 쓸만하겠다.

### Standard IO를 사용하는 Commaned 테스팅

Python에서라면 IO관련 함수에서 mock testing 을 했을것 같은데 유사한 전략을 사용하긴한다. (물론 찾아보니 mocking을 하는 패키지도 github에 있다.)

Standard Output을 기본값으로 사용하는 경우가 많지만 코드의 testability를 위해 standard ouput을 사용하는 io.Writer 를 전역변수로 정의하고 이것을 테스팅할때 바꿔쓰는것이다. 테스트 할때는 io.Writer interface 를 만족하는 bytes.Buffer 같은것으로 바꿔주면서 출력값을 확인하면 된다. Production code에서 조금 번거로워질수 있긴한데 책에서는 이런 방안을 제시했다.

```
// 테스트 할때 이 변수를 바꿔준다.
var out io.Writer = os.Stdout
func sayHi() {
    fmt.Fprint(out, "Hi!")
}
```

```
func TestSayHi(t *testing.T) {
    // 결과 확인을 위해 바꿔치
    out = new(bytes.Buffer)
    sayHi()
    got := out.(*bytes.Buffer).String()
    if got != "Hi!" {
        t.Error(`SayHi() is not returning "Hi!"`)
    }
}
```

### Whitebox Testing

우선 whitebox testing 과 blackbox testing 의 차이는 내부구조를 알고 그것에 대해 테스트를 하는지이다. 예를 들어 내가 패키지를 만들고 GetStats() 라는 함수와 이 함수에서 사용하는 내부함수 getAverage() 가 있다고 치자. 그렇다면 getAverage()에 대해서 테스팅한다면 이것은 whitebox testing을 한것이고, 외부에 노출되는 GetStats 에 대해서 가능한 user input들로 테스트했다면 blackbox testing 을 한것이다. 

앞선 경우와 유사하다. 다만 앞에서는 유저입장에서 standard output을 쓸거라는걸 알고 있었지만 이번에 들 예시는 함수 내부적으로 email을 보내는 경우이다. 

```
// Testability를 위해 따로 빼서 변수에 담는다.
var sendMail = func(username, msg String) {
    ...
}

func CheckUser(username String) {
    ...
    sendMail(username, str)
    ...
}
```

변수로 저장해두었던것은 이 내부함수를 일시적으로 바꾸고 끝날때 되돌려주어야한다. 일종의 cleanup을 해주는 것이다. 그래야만 테스트 종료후 정상적으로 재사용이 가능하다. 테스트실패(fail, panic)의 경우에도 cleanup을 하기 위해서는 defer를 이용한다.

```
func TestCheckUser(t *testing.T) {
    saved := sendMail
    defer func() {
         sendMail = saved
    }()
    sendMail = func(user String) {
        ... // 임시로 새로운 로직 적용
    }
    ... // 나머지 테스트 코드
}
```

### External Test Packages

이번에는 테스트를 하면서 발생할수 있는 의존성에 대해서 다뤄본다. 두개의 패키지 net/url 과 net/http를 보자. net/url 에서는 URL Parser를 제공하고 net/http 에서는 웹서버와 http클라이언트를 제공한다. net/http는 net/url에 의존한다. 

```
net/http -> net/url
```
근데 net/url을 테스트하기위해 net/http 를 사용한다고 하자. 그렇다면 의존성이 순환구조를 보인다. net/url을 패키지라고 명시한 테스트코드에서 net/http를 임포트해서 사용하기 때문이다.

```
net/http -> net/url -> net/http
```
Go 언어에서는 임포트 무한반복을 막기 위해 순환임포트가 일어나면 안된다.

이러한 순환임포트를 해결하기 위해 제목 말 그대로 외부에 테스트패키지를 하나 만들어준다. url_test 라는 패키지를 따로 만들어서 거기에 테스트를 추가한다면 순환 구조는 없어진다.

```
net/http -> net/url
net/url_test -> net/url
net/url_test -> net/http
```

하지만 외부패키지라면 외부로 export 되지 않는 부분들에 대해 어떻게 Whitebox Testing을 할까?

정답부터 말하자면 export 되지않는 요소들을 노출시켜주는 "백도어"를 만들어준다.
fmt 패키지를 살펴볼텐데, 여기에서 `export_test.go` 라는 파일이 이러한 역할을 한다. 파일을 열어보면 별거 없다.

```
// Copyright 2012 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

package fmt

var IsSpace = isSpace
var Parsenum = parsenum
```
우선 fmt 패키지의 일부라고 명시 되어있다. 따라서 여기 안에 정의된 것들은 fmt 패키지를 임포트했을때 사용 가능하다. 그러나 fmt패키지를 빌드할때는 이 파일이 테스트파일이기 때문에 포함되지 않는다. 프로덕션에서는 사용되지 않는다.

그리고 여기에서 내부함수 isSpace 와 parsenum 을 외부에 노출시켜주도록 public으로 만든것을 볼 수 있다. 이 파일에서는 테스트 케이스를 작성하지 않은것을 볼 수 있는데 역할은 단순히 fmt_test 같은 외부 테스트 패키지에서 이것을 통해 내부함수에 접근 가능하도록 해주는 것이다.

### 효율적인 테스트

다른 언어로 개발할때 테스트를 요이하게 하기위해 이미 정의된 assertion, 혹은 setup 등을 사용하는 경우가 많지만 Go에서는 그것을 사용한다기보다는 테스트코드도 프로그램처럼 생각하길 원한다. 기본적으로 제공되는 assertion들로는 체크하는것이 어떤 context 에서 체크하는것인지 알기 힘들다. 이것은 오히려 각 테스트케이스에서 적절한 컨택스트를 제공하면 개발에 더 편리하다. 

모든것을 풀어쓰라는것은 절대 아니다. 다만 우선은 usecase에 대한 컨택스트를 충분히 제공하면서 testcase를 쌓아가라는것이다. 따라서 이러한 setup&assertion 코드가 나중에 필요하다면 중복제거를 위해서 도입되면 되자만, premature abstraction 밖에 제공하지못하는 assertion 함수 등에 너무 의존해서는 안된다.

### 불안정한 테스트

The Go Programming Language 에서는 brittle test라는 표현을 쓴다. 예를 들자면 코드에 무언가를 수정할때마다 테스트케이스까지 같이 고쳐줘야하는 경우를 얘기한다. 

자주 마주하는 테스트로는 결과값으로 받은 긴 스트링을 비교하는 경우이다. 엄청 긴 스트링을 꼭 비교할것 없이 그 중 일부만 비교하면 될 경우, 필요한 부분들만 제대로 나오는지 체크하는것이 더 좋다. 유지 보수가 귀찮아질 수 있기 때문이다. 개인적으로는 이게 취향차이일수도 있다고 보지만 책에 나온 글로만 봤을때는 전혀 틀린 말이 아니다.

### 후기

테스트코드 작성이 때로는 시간을 적지않게 소요하는 작업이다. 시간 압박에 밀려서 테스트를 하면 다른 테스트케이스를 많이 참고해서 복붙을 할때도 있었고 테스팅 라이브러리에서 제공하는 편리한 기능으로 재미를 볼때도 많았다. 다만, 그러다 보니 의도가 불분명한 테스트들도 생겨난다.

Go언어에서는 이러한 불분명한 테스트를 최소한으로 줄이기 위해 설계를 한 것으로 보이는데 그러다보니 테스트케이스를 작성하는데 시간이 꽤나 걸릴것 같아 보이기도 한다. mock으로 처리하면 좀더 편리할것 같아보이는 부분들도 보이는데 실제 서비스/프로젝트에서는 어떻게 테스팅을 할지 궁금하다.