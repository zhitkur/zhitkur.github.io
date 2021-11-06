---
layout: post
title: Windows driver - Callback routine
---

### Overview
윈도우 드라이버의 `콜백`루틴을 생성하여 로드되는 이미지에 대한 알림을 받는 코드를 작성.  
> 백신 프로그램이나, 안티치트등과 같은 프로그램에서도 사용되는 루틴입니다.  
> 이미지는 PE Image를 뜻하고, 프로세스 또는 쓰레드가 생성될 때를 의미합니다.  

여러가지 함수(`PsSetLoadImageNotifyRoutine`, `ObRegisterCallbacks`, ...)를 통해 루틴을 등록할 수 있는데, 저는 `PsSetLoadImageNotifyRoutine`함수를 사용하여 등록해 보겠습니다.  


### PsSetLoadImageNotifyRoutine

```c++
NTSTATUS PsSetLoadImageNotifyRoutine(
  PLOAD_IMAGE_NOTIFY_ROUTINE NotifyRoutine
);
```

`NotifyRoutine` 파라미터를 통해 로드되는 이미지(Driver, DLL, EXE)가 Virtual Memory에 Mapping될 때 호출됩니다.  

```c++
PLOAD_IMAGE_NOTIFY_ROUTINE PloadImageNotifyRoutine;

void PloadImageNotifyRoutine(
  PUNICODE_STRING FullImageName,
  HANDLE ProcessId,
  PIMAGE_INFO ImageInfo
)
```

- `FullImageName` : UNICODE_STRING으로 이루어진 이미지 파일 이름의 포인터 (e.g. notepad.exe)  
- `ProcessId` : 이미지가 Mapping된 프로세스의 식별 값, 드라이버의 경우 0  (e.g. 1548)  
- `ImageInfo` : 이미지 정보가 포함된 `IMAGE_INFO`구조체의 포인터  

```c++
typedef struct _IMAGE_INFO {
  union {
    ULONG Properties;
    struct {
      ULONG ImageAddressingMode : 8;
      ULONG SystemModeImage : 1;
      ULONG ImageMappedToAllPids : 1;
      ULONG ExtendedInfoPresent : 1;
      ULONG MachineTypeMismatch : 1;
      ULONG ImageSignatureLevel : 4;
      ULONG ImageSignatureType : 3;
      ULONG ImagePartialMap : 1;
      ULONG Reserved : 12;
    };
  };
  PVOID  ImageBase;
  ULONG  ImageSelector;
  SIZE_T ImageSize;
  ULONG  ImageSectionNumber;
} IMAGE_INFO, *PIMAGE_INFO;
```

- `SystemModeImage` : 드라이버와 같이 커널 모드의 구성요소의 경우 1(true), 유저모드에 매핑 된 이미지의 경우 0(false)  
- `ImageBase` : 해당 이미지의 ImageBase  

> `SystemModeImage`값을 통해 kernel <-> user-mode에 대한 이미지 검색을 설정

### Example Source
```c++
const wchar_t* target_process[] = {L"NOTEPAD.EXE", L"IDA64.EXE"};

VOID target_info(PUNICODE_STRING FullImageName, HANDLE ProcessId, PIMAGE_INFO ImageInfo)
{
    WCHAR upper_buffer[512] = { 0, };

    // user-mode image
    if (!ImageInfo->SystemModeImage)
	{
		memcpy(upper_buffer, FullImageName->Buffer, FullImageName->Length);
		_wcsupr(upper_buffer);

		for (int i = 0; i < sizeof(target_process) / sizeof(PVOID); i++)
		{
			if (wcsstr(upper_buffer, target_process[i]))
			{
				Log("[*] Target Image Loaded! -> %ws\n", target_process[i]);
				Log("[#] ImageBase -> [0x%X] | PID -> [%.4X]", ImageInfo->ImageBase, ProcessId);
        
        // Something... (e.g. Terminate target process)
			}
		}
	}
}

VOID unload_driver(PDRIVER_OBJECT pDrvObj)
{
    UNREFERENCED_PARAMETER(pDrvObj);

    PsRemoveLoadImageNotifyRoutine(&target_info);
    Log("[-] unload_driver called!\n");

}

NTSTATUS driver_entry(PDRIVER_OBJECT pDrvObj, PUNICODE_STRING pRegPath)
{
    UNREFERENCED_PARAMETER(pRegPath);

    PsSetLoadImageNotifyRoutine(&target_info);

    pDrvObj->DriverUnload = unload_driver;

    Log("[+] driver_entry called!\n");
    return STATUS_SUCCESS;
}
```

중요한 함수만 포함하여 구조체 정의에 필요한 헤더는 포함하지 않았습니다.  

### target_info
해당 콜백함수의 역할은 매우 간단한 target_process 배열에 존재하는 프로세스 이름과 로드되는 이미지를 비교하여 로드될 때 Log 매크로를 통해 타겟의 정보를 로그로 남깁니다.  

> upper_buffer 배열을 통해 로드되는 이미지를 대문자로 변경하여 오류를 줄임  

### unload_driver
Driver를 Unload할 때 정상적으로 콜백루틴을 제거하는 함수(`PsRemoveLoadImageNotifyRoutine`)가 포함되어 있습니다.  

### driver_entry
드라이버 진입점입니다.  
콜백루틴을 등록하는 함수(`PsSetLoadImageNotifyRoutine`)를 통해 등록합니다.  

## Conclusion
해당 콜백루틴을 어떻게 쓰냐에 따라서 다양한 방법으로 활용이 가능합니다.  
Target Image에 대한 알림을 받아, ImageBase값 + Offset을 얻어 특정 메모리 값을 조작한다던지..
`ObRegisterCallbacks`함수를 사용하여 Target Handle의 권한을 제어하여 비정상적인 접근을 방지하는 것도 가능합니다.