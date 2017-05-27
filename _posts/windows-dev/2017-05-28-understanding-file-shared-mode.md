---
layout: post
title:  "파일 공유모드의 이해"
date:   2017-05-28 00:30:00 +0900
categories: windows-dev
tags: dev
---

### 윈도우에서 다른 프로세스가 사용중인 파일에 접근하기

윈도우에서 다른 프로세스가 사용 중인 파일에 접근해야 할 경우가 있다. 로그파일이라던지.

`File.Open("파일명", FileMode.Open, FileAccess.Read, FileShare.Read)` 등으로 접근해도 파일 접근에 실패하여, `File.Copy()`로 임시파일로 복사해서 여는 바보짓으로 구현했다가 좀 더 알아보니 `FileShare`가 생각과 다르게 동작한다는 사실을 알았다.

API 문서를 잘 보면 알 수 있었겠지만 인터넷이 제한된 환경에서 도움말도 접근할 수 없으니 감으로 코딩해서 발생한 오류였다.

결론은 `FileShare`는 내가 사용하고자 하는 모드로 지정하는게 아니라, 파일을 Lock하고 있는 프로세스 기준으로 지정하면 된다.

`FileMode`나 `FileAccess`는 내가 사용하고자 하는 모드로 지정하지만 `FileShare`는 그 반대로 지정하면 된다.

```csharp
FileStream fs = File.Open("파일명", FileMode.Open, FileAccess.Read, FileShare.ReadWrite)
```

매우 간단한 것이었다.

끝.
