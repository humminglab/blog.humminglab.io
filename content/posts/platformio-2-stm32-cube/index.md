---
title: "PlatformIO (2) - STM32Cube Platform 개발"
date: "2022-06-24T09:00:00+09:00"
lastmod: "2022-06-24T09:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["PlatformIO", "STM32", "SMT32Cube", "VSCode"]
categories: ["Development"]
series: ["PlatformIO 개발하기"]
---

이전 글에 이어 PlatformIO를 이용하여 STM32Cube SDK 로 개발환경을 구성하는 것을 정리해 본다. 

- [PlatformIO (1) - 개요 및 특징]({{< ref "posts/platformio-1">}})
- **PlatformIO (2) - STM32Cube Platform 개발**
- PlatformIO (3) - STM32Cube FreeRTOS 적용
- PlatformIO (4) - PlatformIO 디버깅 
- PlatformIO (5) - PlatformIO Unit Test

## PlatformIO로 STM32 platform 설치하기

VSCode에서 아래와 같이 PlatformIO Home을 연다.

{{< image src="pio-01.png" width="600px" height="auto">}}

New Project 선택하여 다음과 같이 설정 

- Name은 stm32test 와 같이 적절히 설정 
- Board는 "ST Nucleo F103RB" 선택 
- Framework는 "STM32Cube" 선택
- Finish 로 생성

{{< image src="pio-02.png" width="400px" height="auto">}}

우선 동작이 되는지만 확인키 위하여 다음과 같이 src/main.c 로 main() 함수를 만든다. 

{{< image src="pio-03.png" width="400px" height="auto">}}

VSCode "PlatformIO: Build" 를 하면 정상적으로 빌드된다. 

[NUCLEO-F103RB](https://www.st.com/en/evaluation-tools/nucleo-f103rb.html) EV board를 USB로 연결 후, 
"PlatformIO: Upload"를 실행하면 다음과 같이 터미널 창에 ST-Link 를 이용하여 정상적으로 FW가 write 된것을 확인할 수 있다. 

{{< image src="pio-04.png" width="600px" height="auto">}}

하는 김에 "PaltformIO: Start Debugging" 을 실행하면 다음과 같은 동작이 수행된다. 
- 코드를 디버깅 모드로 다시 빌드 및 upload 실행
- OpenOCD 등 디버깅에 필요한 툴 자동 설치 
- VSCode 디버깅 화면 전환 후 main에 임시 break point 설정 후 실행

{{< image src="pio-05.png" width="600px" height="auto">}}

이렇게 간편하게 STM32 개발 및 디버깅 환경 설치를 하였다. 

현재까지 구성된 소스는 아래와 같다. 

- https://github.com/aqwerf/platformio-stm32test/tree/v1-init

## STM32Cube IDE 로 코드 생성하기

main 함수에서 하여야 할일은 시스템 클럭, 주변 장치 등을 초기화 하는 일부터 시작하여야 한다. 

이를 예제 코드를 참조하여 수정하는 것도 방법이지만 가장 편한 방법은 STM32Cube IDE를 이용하여 GUI 환경에서 포트 초기화 등을 설정 후, 
생성된 소스를 복사해서 사용하는 것이다.

[STM32CubeIDE](https://www.st.com/en/development-tools/stm32cubeide.html) 페이지에서 STM32Cube IDE를 다운로드 받아서 설치한다.

다음과 같은 절차로 프로젝트를 생성한다.

- File -> New -> STM32 Project 를 선택
- Board Select 탭에서 NUCLEO-F103RB 를 입력 후 Next 를 누른다. 
- Project Name을 stm32cube-test 처럼 적절히 설정 후 Finish 를 눌러 코드 생성

생성되면 좌측에서 확장자가 ioc 인 stm32cube-test.iot를 더블 클릭하면 Pinout & Configuration 창이 열린다. 

FreeRTOS 까지 실행되는 것을 만들려고 하므로 다음과 같이 설정 

- Middleware -> FREERTOS -> Interface -> CMSIS_V2
- Middleware -> FREERTOS -> Advanced settings -> USE_NEWLIB_REENTRANT -> Enabled

설정 후 저장하면 아래와 같은 경고가 뜨는데 이 부분은 무시하고 Yes 를 선택한다. 

{{< image src="pio-06.png" width="500px" height="auto">}}

위의 경고가 뜨는 이유는 HAL에서도 Tick을 이용하여 tick count를 증가시키고 있고, FreeRTOS에서도 tick을 사용하는데, 둘을 따로 사용하라는 말이다.

HAL의 tick 관련 함수를 사용할 일이 없어 둘이 같이 Systick을 공유하여 사용하도록 한다.

Yes를 선택하면 위의 설정대로 코드가 생성된다.

{{< image src="pio-07.png" width="600px" height="auto">}}

Project Explorer에 보이는 파일은 다음과 같다. 

- Core: 생성된 초기화 파일로 이 파일을 PlatformIO 로 복사해서 사용하면 된다. 
- Drivers: 이 부분은 PlatformIO에서 STM32Cube framework를 사용하므로 이미 ~/.platformio/packages/framework-stm32cubef1/ 에 설치되어 있다. (윈도우즈의 경우 )
- Middleware/Third_Party/FreeRTOS: FreeROTS 용 소스로 복사해서 사용하면 된다.

이렇게 기본 초기화 소스를 생성하고, STM32Cube는 종료한다.
나중에 GPIO, UART 등의 수정이 있을 때는 다시 여기에서 설정해주고, 생성된 파일을 다시 복사하는 방식으로 하면 된다.

## PlatformIO 에 STM32Cube 코드 옮기기

PlatformIO는 프로그램 소스를 src 와 lib 두 곳에 있을 수 있다. 
src에 있는 코드는 무조건 컴파일 되어서 추가되는 코드들이고, lib 에 있는 소스들은 src 에 있는 파일들의 의존성에 따라서 빌드되고 추가된다. 

Makefile 없이도 의존성을 찾아서 자동으로 빌드해 주는 것이, PlatformIO의 [Library Dependency Finder (LDF)](https://docs.platformio.org/en/stable/librarymanager/ldf.html) 기능이다.

LDF는 C/C++ 소스의 `#include` 문을 분석하여, library의 의존성을 찾아서 빌드에 자동으로 추가한다. 
이 기능으로 인하여 별도의 빌드 파일없이도 소스만으로 빌드가 가능하다. 

PlatformIO 내부적으로는 [SCons](https://scons.org/) 빌드 툴을 이용하여 이를 구현하고 있다. SCons을 사용한 이유는 PlatformIO와 같이 Python 으로 작성되었기 때문인 것 같다. 
어쨋든 개발자에게는 빌드 script를 노출하지 않으므로 SCons 사용 방법을 볼 필요는 없다.

STM32Cube에서 생성한 소스 코드를 옮길때 가장 간단한 방법은 관련된 모든 코드를 src 디렉토리 안으로 옮기는 것이다. 
이렇게 되면 모두 컴파일 되고, 링크로 합쳐지게 되어 특별히 문제가 될 것이 없다.

