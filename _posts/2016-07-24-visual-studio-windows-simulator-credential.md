---
layout: post
title:  "비주얼 스튜디오에서 윈도우 시뮬레이터 자격증명 오류"
date:   2016-07-24 15:22:57 +0900
categories: windows-dev
tags: dev
---

윈도우10 환경에서 비주얼 스튜디오 Windows 8 앱 또는 유니버셜 앱 개발을 하면서 시뮬레이터로 테스트를 하려 했을 때, 아래와 같은 오류가 뜨면서 시뮬레이터 실행이 안 되는 경우를 겪었다.

```
Windows 시뮬레이터에서는 현재 자격 증명이 필요합니다. 이 컴퓨터를 잠근 다음 현재 암호를 사용하여 잠금을 해제하고(Windows 시뮬레이터는 사용자 이름 및 암호 자격 증명만 지원) Windows 시뮬레이터를 다시 실행하세요. 컴퓨터를 잠그려면 Ctrl+Alt+Del을 누르고 Enter 키를 누릅니다.
```

메시지에서 지시하는 대로 다시 로그인도 해 보고, PIN대신 암호 입력으로 로그인 옵션을 변경 해 봤는데도 여전히 안된다면, 계정 타입이 Microsoft 계정인지 확인한다.

1. `시작 버튼(윈도우 로고 버튼)`을 클릭하고 `설정`을 클릭한 후 `계정`을 클릭한다.
2. `내 메일 및 계정` 메뉴에서 `대신 로컬 계정으로 로그인`이라는 문구가 있는 지 확인한다.
3. `대신 로컬 계정으로 로그인`이라는 문구가 있으면 이 문구를 클릭해서 로컬 계정으로 변환한 후 다시 시뮬레이터를 실행한다.

윈도우를 설치하면 최초 설정 단계에서 Microsoft 계정을 사용할 권장하고, 일반 사용자라면 반드시 로컬 계정을 고집할 이유가 없다고 생각해서 로컬 계정을 생성하지 않았더니 개발자 기능에서 발목이 잡혔다.

### 덧붙이는 말

구글과 MSDN을 아무리 뒤져봐도 자격 증명 오류 현상과 해결책을 제대로 설명하는 글을 찾을 수 없어서 정리해서 올린다.
이런 문제로 발목 잡힐 거라고 생각한 사람은 그간 아무도 없었나 본데, 원래 오류란 누군가는 당연하다고 여기고 설명하지 않았던 부분에서부터 시작하는 법이다. :)