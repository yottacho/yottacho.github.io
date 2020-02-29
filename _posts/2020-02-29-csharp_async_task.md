---
layout: post
title:  "C# 비동기 Task로 실행하기"
date:   2020-02-29 11:10:00 +0900
categories: windows-dev
tags: dev
---

## C# Async Task로 실행하기

대부분의 I/O 메서드에는 Async 구현이 포함되어 있는데, I/O구현이 아닌 직접 작성한 로직에서 Async 를 구현하고자 할 경우 아래와 같이 작성할 수 있다.


```csharp
Task t = Task.Run(() => user_function(arg1, arg2));

// 메서드에 async 키워드를 추가하고 await을 사용할 수도 있지만 아래처럼 ContinueWith()를 사용해도 된다.
// ContinueWith를 쓸 때는 async 키워드를 추가하지 않아도 된다.

t.ContinueWith((x) => 
{
    // 메서드가 종료되었을 때 실행할 내용
}, TaskScheduler.FromCurrentSynchronizationContext());
```

await으로 대기할때는 ConfigureAwait(true/false)로 현재 스레드로 실행할 것인지(true), 새 스레드로 실행할 것인지(false)를 선택할 것이 권장된다.


## 스레드에서 UI 업데이트하기

별도 스레드에서 UI를 갱신하려고 하면 예외가 발생하는데 Invoke로 처리하는 방법

```csharp
if (ui_control.InvokeRequired)
{
    ui_control.Invoke(new Action(() => { ui_control.Items.Add(message); }));
}
else
{
    ui_control.Items.Add(message);
}
```

InvokeRequired 및 Invoke() 호출은 ui_control 대신에 form을 사용해도 된다.


끝.
