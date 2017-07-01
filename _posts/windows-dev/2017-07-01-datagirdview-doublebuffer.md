---
layout: post
title:  "(펌)데이터그리드뷰의 더블버퍼 설정"
date:   2017-07-01 11:10:00 +0900
categories: windows-dev
tags: dev
---

### DataGridView 에 DoubleBuffered 설정하기

데이터 그리드 뷰 컴포넌트는 표를 표현할 때 매우 유용한 컴퍼넌트지만 스크롤이 너무 느린 단점이 있다. 더블버퍼 기능을 켜면 속도가 개선되지만, 이 기능이 컴퍼넌트에 노출되어 있지 않기 때문에 더블버퍼 기능을 설정하기 위해서는 코딩이 필요하다.

```csharp
InitializeComponent(); // 생성자의 컴포넌트 생성 호출 이후 코딩 추가

DoubleBuffered = true; // Form의 DoubleBuffered 를 true로 변경

// Set Double buffering on the Grid using reflection and the bindingflags enum.
// DataGridView dataGridView1
typeof(DataGridView).InvokeMember("DoubleBuffered",
	BindingFlags.NonPublic | BindingFlags.Instance | BindingFlags.SetProperty,
	null, dataGridView1, new object[] { true });
```

* [출처: https://www.codeproject.com/Tips/654101/Double-Buffering-a-DataGridview](https://www.codeproject.com/Tips/654101/Double-Buffering-a-DataGridview)

또는 이런 방법도 있다.

```csharp
//Put this class at the end of the main class or you will have problems.
public static class ExtensionMethods
{
	public static void DoubleBuffered(this DataGridView dgv, bool setting)
	{
		Type dgvType = dgv.GetType();
		PropertyInfo pi = dgvType.GetProperty("DoubleBuffered",
			BindingFlags.Instance | BindingFlags.NonPublic | BindingFlags.SetProperty);
		pi.SetValue(dgv, setting, null);
	}
}

...

InitializeComponent(); // 생성자의 컴포넌트 생성 호출 이후 코딩 추가

// DataGridView 컴포넌트에 DoubleBuffered() 메서드가 없으므로, ExtensionMethods 의 DoubleBuffered()를 대신 호출한다.
dataGridView1.DoubleBuffered(true);
```

* [출처: http://bitmatic.com/c/fixing-a-slow-scrolling-datagridview](http://bitmatic.com/c/fixing-a-slow-scrolling-datagridview)


끝.
