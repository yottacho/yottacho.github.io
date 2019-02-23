---
layout: post
title:  "delegate의 이해와 람다식"
date:   2017-07-01 12:50:00 +0900
categories: windows-dev
tags: dev
---

### C#의 delegate 이해와 람다식

* [C#프로그래밍 가이드의 delegate에 대한 설명](https://docs.microsoft.com/ko-kr/dotnet/csharp/programming-guide/delegates/)

> 대리자는 특정 매개 변수 목록 및 반환 형식이 있는 메서드에 대한 참조를 나타내는 형식입니다. 대리자를 인스턴스화하면 모든 메서드가 있는 인스턴스를 호환되는 시그니처 및 반환 형식에 연결할 수 있습니다. 대리자 인스턴스를 통해 메서드를 호출할 수 있습니다

이렇게 보면 뭔 말인지 이해가 쉽지 않은데, 직접 다뤄보니 C의 함수포인터와 매우 유사하다는 느낌이다,

> 대리자는 메서드를 다른 메서드에 인수로 전달하는 데 사용됩니다.

이 설명이 delegate의 핵심이라고 생각한다.

#### 1. delegate는 어떤 경우에 사용하면 좋은가?
피호출 클래스에서 호출 클래스의 특정 메서드를 호출하고 싶을 때 매번 호출 클래스를 인자로 받아오지 않아도 된다. delegate를 활용하면 클래스간 결합이 interface 보다 더 느슨해져서 메서드 형식만 맞으면 어떤 클래스의 어떤 메서드를 대입하더라도 사용할 수 있다.

#### 2. delegate 선언하기
- 컴파일러가 검증할 수 있도록 메서드의 인자와 리턴값을 선언한다.
- 메서드 선언 중 메서드명에 해당하는 부분은 C의 `typedef`로 별칭(새로운 형식)을 정의하는 것과 동일한 용법으로 생각할 수 있다.

```csharp
delegate int PerformCalculation(int x, int y);
```

#### 3. delegate 할당하기
위의 코드에 의해 `PerformCalculation`라는 형식이 생겼다. 이제 이 형식으로 사용할 변수 `p`를 생성한다. 변수라고 한 이유는 `p`에 할당하는 메서드는 형식만 맞으면 어느 것이든지 사용할 수 있기 때문이다.

```csharp
// 어딘가에 있는 int Add(int x, int y) 메서드를 대입한다.
// static 메서드도, new로 생성한 instance의 메서드도 가능하다.
PerformCalculation p = Add;
```

#### 4. 메서드를 호출하자
`PerformCalculation p`로 정의한 `p`를 내부메서드(함수)를 호출하듯 호출한다. p.Add()로 호출하는 것이 아님에 유의한다.

```csharp
int result = p(1, 2);
```

#### 5. 그밖의 활용법

```csharp
// void 리턴형식의 delegate를 선언한다. 리턴값이 꼭 void여야 한다.
delegate void DelegateChain(int x, int y);
DelegateChain p += method1;
DelegateChain p += method2;
DelegateChain p += method3;

p(1, 2);
```

- 결과는 `method1`, `method2`, `method3`가 순서대로 실행된다.
- 마찬가지로 `-=` 연산자로 실행 목록에서 메서드를 제거할 수 있으며, `+=`으로 추가하지 않은 메서드를 `-=`으로 빼더라도 런타임 오류는 발생하지 않는다.
- 윈폼의 이벤트는 delegate로 구성되어 있다. 사용자가 `+=`으로 메서드를 추가해 두면 이벤트가 발생했을 때 메서드를 등록한 순서대로 호출한다.
- [참고사이트: http://tapito.tistory.com/45](http://tapito.tistory.com/45)

#### 6. 무명메서드
- C# 2.0부터는 delegate만으로도 간단하게 무명메서드를 정의할 수 있다. 이후에 등장할 람다를 추천한다.
- 자바는 익명클래스(=무명클래스)의 메서드 오버라이드를 통해 비슷하게 구현할 수 있는데 C#보다는 조금 더 많은 코딩이 필요하다. 자바도 8이후 도입된 람다를 사용하는 것을 추천.

```csharp
delegate int delegateMethod(int a);
delegateMethod m = null;

// 어딘가에 있는 메서드가 아니라 코드 중간에 메서드를 구현했다
m = delegate(int a) { return a++; };
int result1 = m(1);

m = delegate(int a) { return a--; };
int result2 = m(2);
```

#### 7. 람다표현식
이왕 여기까지 온 거, C# 3.0부터 도입된 람다도 다루어 보자.

```csharp
delegate int delegateMethod(int a);
delegateMethod m = null;

// 람다 표현식
m = a => a++;
int result1 = m(1);
```

- 람다 표현식은 `(매개변수) => { 내용; }` 또는 `(매개변수) => 반환식;`으로 정의한다. 매개변수가 하나일 경우 매개변수의 괄호를 생략할 수 있다.
- `(a) => a++;`는 `(a) => { return a++;}`와 동일하다.
- 무명메서드와 비교하면 delegate 키워드나 int 같은 매개변수의 형식 정의가 사라졌고 매우 심플한 코드로 압축되었다.
- [참고사이트: http://tapito.tistory.com/46](http://tapito.tistory.com/46)
- [람다식: https://docs.microsoft.com/ko-kr/dotnet/csharp/programming-guide/statements-expressions-operators/lambda-expressions](https://docs.microsoft.com/ko-kr/dotnet/csharp/programming-guide/statements-expressions-operators/lambda-expressions)

끝.
