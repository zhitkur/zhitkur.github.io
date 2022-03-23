---
layout: post
title: Device Object
---

### Overview
Windows Driver를 개발할 때 응용 프로그램과의 의사소통이 가능하게 하려면 Device를 생성해줘야 합니다.  
이때 사용하는 함수가 `IoCreateDevice` 인데, 해당 함수 원형은 다음과 같습니다.

```c++
NTSTATUS IoCreateDevice(
  [in]           PDRIVER_OBJECT  DriverObject,
  [in]           ULONG           DeviceExtensionSize,
  [in, optional] PUNICODE_STRING DeviceName,
  [in]           DEVICE_TYPE     DeviceType,
  [in]           ULONG           DeviceCharacteristics,
  [in]           BOOLEAN         Exclusive,
  [out]          PDEVICE_OBJECT  *DeviceObject
);
```

여기서 DeviceExtensionSize 파라미터 값은 언제 사용하느냐가 이번 글의 핵심입니다.
해당 파라미터는 말 그대로 DeviceExtension의 크기를 지정해줍니다.  

Device Object마다 가지는 임의의 공간이기도 한데, Device를 생성할 때 해당 크기(DeviceExtensionSize)만큼 Device Extension을 생성하고 DeviceObject->DeviceExtension 포인터 변수에 그 주소가 저장이 됩니다.  

> Private Memory information 이라고도 하는데 드라이버 개발자가 정의하는 자신만의 고유공간을 의미합니다.  

### Why use it?  

그럼 Device Extension을 쓰는 이유는 무엇일까요?  

내가 어떠한 정보를 저장하고 싶은데 그 정보를 전역변수에 보관하는 게 아니라, 어딘가에 문맥으로 보관하고 싶다. 그 때 그 문맥은 내가 접근하고자 하는 하드웨어의 문맥으로 저장하고 싶을 때 사용합니다.  

예를 들면, 내가 드라이버를 하나 만들었다고 했을 때 접근하고 하는 하드웨어(Device)가 두 개라면 어떨까요?  

하나의 드라이버에서 두 개의 하드웨어에 접근하여야 하는데, 어떤 고유한 정보를 담아서 두 개의 하드웨어에 담고 싶을 때 사용할 수 있습니다.  

또한 각각의 device object를 통해 쉽게 접근할 수 있습니다.  

전역변수와 개념은 유사하지만 훨씬 더 접근하기 쉽고 Device에 종속적인 의미를 가지기에 실전에서 많이 사용된다고 합니다.  