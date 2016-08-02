---
layout: post
title:  "C#에서 COM 모듈 불러오기"
date:   2016-08-02 14:00:00 +0900
categories: windows-dev
tags: dev
---

### 1. COM`(Component Object Model)`

* [Microsoft 설명](https://www.microsoft.com/com/default.mspx)
* [위키(한국어)](https://ko.wikipedia.org/wiki/%EC%BB%B4%ED%8F%AC%EB%84%8C%ED%8A%B8_%EC%98%A4%EB%B8%8C%EC%A0%9D%ED%8A%B8_%EB%AA%A8%EB%8D%B8)

(응용프로그래머 입장에서) 하드웨어 제어나 DirectX 같은 특정 기능을 구현한 API 개발자가 제공한 DLL 또는 OCX 모듈을 `regsvr32`로 레지스트리에 등록하고, 응용프로그램에서 Guid를 통해 제공받은 모듈과 연결하고 모듈의 기능을 사용하는 인터페이스 개발 방법

이 글은 기존의 COM 기반 모듈을 쉽게 불러와서 쉽게 활용하는 방법을 소개한다.


### 2. C#에서 COM DLL을 불러오는 방법
[MSDN 참조링크](https://msdn.microsoft.com/ko-kr/library/aa288455(v=vs.71).aspx)

1. Visual Studio의 참조추가 메뉴에서 COM 검색 (완전 자동화)
   - 장점: 개발PC에 설치된 COM을 바로 불러와서 코드에 활용
   - 단점: 개발PC에 모듈을 설치할 수 없는 경우 작업 불가 (재배포할 수 없는 모듈 등)

2. `TlbImp` 프로그램 이용 (Wrapper DLL 생성)
    - TlbImp는 비주얼 스튜디오 설치 시 Windows SDK 설치여부에 따라 아래 위치에서 찾을 수 있다.
        * exe만 복사해서 Windows SDK가 설치되지 않은 PC에서도 실행할 수 있다.
        * C:\Program Files (x86)\Microsoft SDKs\Windows\v7.0A\Bin
        * C:\Program Files (x86)\Microsoft SDKs\Windows\v8.1A\bin\NETFX 4.5.1 Tools
        * C:\Program Files (x86)\Microsoft SDKs\Windows\v10.0A\bin\NETFX 4.6 Tools
    - `TlbImp "파일명.DLL(OCX)"`으로 실행하면 "파일명`Lib`.dll"이 생성된다.
        * TlbImp의 옵션은 [`MSDN`](https://msdn.microsoft.com/ko-kr/library/tt0cf3sx(v=vs.110).aspx)을 참조한다.
        * TlbImp로 생성한 "파일명Lib.dll"을 `Ildasm`으로 열면 구조를 확인할 수 있다.
        * 생성된 파일을 RegAsm을 통해 닷넷 공유 라이브러리로 등록할 수 있다고 하는데 안 해봤다.
    - TlbImp로 생성한 "파일명Lib.dll"을 비주얼 스튜디오 참조 항목에 추가한다.
        * COM 모듈이 없더라도 Wrapper만 있으면 컴파일이 가능하다.
        * Wrapper 파일 속성에 `출력 디렉터리에 복사` 기능을 켜 둔다. (Wrapper DLL을 관리해야 함)
        * 실행하려면 "파일명Lib.dll"이 필요하고 COM 모듈이 regsvr32로 등록되어 있어야 한다. (COM 모듈을 찾을 수 없으면 'System.Runtime.InteropServices.COMException' 발생한다.)

3. COM 매핑을 수동으로 수행하는 C# 코드를 작성
    - [MSDN 참조링크](https://msdn.microsoft.com/ko-kr/library/aa288455(v=vs.71).aspx) 예제를 확인한다.
    - Wrapper DLL을 생성할 필요가 없고 배포 부담이 덜지만 Wrapper API를 코딩으로 커버해야 한다.

### 3. COM DLL이 준비됐으면 사용해 보자
(네임스페이스를 따로 지정하지 않았으면) `모듈명Lib.클래스명 변수명 = new 모듈명Lib.클래스명()`으로 간단하게 생성하고 메서드를 호출할 수 있다.

> 대신증권 Cybos Plus의 cputil.dll 호출 예제<br/>
> 대신증권 Cybos 5를 다운로드하고 Cybos Plus 탭을 클릭하고 로그인하면 `C:\DAISHIN\CYBOSPLUS` 폴더에 DLL들이 생성된다.<br/>
> cputil.dll을 TlbImp로 래퍼를 생성하여 참조에 추가하고 아래 코드를 입력해서 테스트한다.<br/>
> 참고: 거래처리를 하려면 `1846` 메뉴에서 시스템 트레이딩을 신청해야 한다.

```csharp
    CPUTILLib.CpCybos cybos = new CPUTILLib.CpCybos();
    int connected = cybos.IsConnect;
    textBox1.Text = "Connect " + connected;

    CPUTILLib.CpStockCode stock = new CPUTILLib.CpStockCode();
    string name = stock.CodeToName("A003220");
    textBox2.Text = name;
```

끝.
