---
layout: post
title: Windows GDI
---

### Overview
Reversing.kr ImagePrc 문제를 풀 때 GDI를 사용한 코드를 분석하는 지식이 부족한 것 같아서 궁금했던 GDI 정보에 대해 적습니다.

### GDI (Graphics Device Interface)
GDI는 Windows 운영체제가 응용 프로그램에 제공하는 그래픽 출력 API 입니다. 응용 프로그램과 Device Driver의 중간 역할은 한다고 보시면 됩니다. GDI를 통해 세세하게 제어(thickness, color, position ...)도 가능한데, 번거롭기 때문에 간편하게 제어하기 위해서 사용하는 구조체가 바로 `DC(Device Context)`입니다.

### DC (Device Context)
DC는 윈도우 프로그램에서 화면 출력을 위해 출력에 필요한 정보를 정의하는 `구조체(struct)`입니다. 또한, `모든 GDI 함수는 첫 번째 인자에 DC Handle을 요구합니다.`
DC의 기본 틀은 GetDC() ~ ... ~ ReleaseDC()로 작성됩니다.  

### More information
- `Get mouse position`  
마우스 메시지는 lParam의 상위 워드에 마우스 버튼이 눌러진 y 좌표, 하위 워드에 x 좌표를 가지며 좌표값을 검출해 내기 위해 HIWORD, LOWORD의 매크로 함수를 사용합니다.  
즉, x = LOWORD(lParam) / y = HIWORD(lParam)가 됩니다.

![xy](https://raw.githubusercontent.com/zhitkur/zhitkur.github.io/main/_screenshots/gdi/xy.png)

다음과 같이 사용하여 x, y 좌표를 구한 후 선을 그릴 수도 있습니다.

![lParam](https://raw.githubusercontent.com/zhitkur/zhitkur.github.io/main/_screenshots/gdi/lParam.png)
  
- `Enable double click`  
일반적으로 더블클릭도 인식이 바로 될거라고 생각하지만 아닙니다. (WM_LBUTTONDOWN -> WM_LBUTTONUP) 더블클릭 메시지를 정상적으로 받고자 한다면 윈도우 클래스의 스타일에 매크로 `CS_DBLCLKS` 플래그를 추가해줘야 합니다.  

![CS_DBLCLKS](https://raw.githubusercontent.com/zhitkur/zhitkur.github.io/main/_screenshots/gdi/CS_DBLCLKS.png)

스타일에 추가해주고 나면 정상적으로 인식합니다. 다음 사진은 예시입니다.

![wm_double_click](https://raw.githubusercontent.com/zhitkur/zhitkur.github.io/main/_screenshots/gdi/wm_double_click.png)

더블클릭 시 InvalidateRect()를 호출하여 작업영역 전체를 무효화시켜 화면을 지웁니다.

- `Timer`  
일반적인 키보드나, 마우스 메시지는 사용자로부터 입력이 되는데, 타이머 메시지(WM_TIMER)는 사용자의 동작과는 상관없이 발생합니다.  
주기적으로 같은 동작을 반복해야 한다거나, 여러 번 나누어 해야 할 일이 있을 때 이 메시지를 사용합니다. 

 > 하지만 WM_TIMER 메시지를 받아 실행하면 우선순위에 밀려 원하는 타이밍에 제대로 된 실행이 안될 수도 있습니다.
 > 그렇기에 콜백 함수로 등록하여 넘겨주면 가장 정확한 시간에 타이머를 실행시킬 수 있습니다.