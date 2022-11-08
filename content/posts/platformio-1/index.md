---
title: "PlatformIO (1) - 개요 및 특징"
date: "2022-06-24T09:00:00+09:00"
lastmod: "2022-06-24T09:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["PlatformIO", "STM32", "SMT32Cube", "VSCode"]
categories: ["Development"]
series: ["PlatformIO 개발하기"]
---

Cortex-M series 급을 이용한 임베디드 시스템 개발을 하다 보면, 지속적으로 사용할 수 있는 통합 개발 환경이 마땅치 않다는 문제가 있다.
Windows 나 Linux 라면 한번 익혀 두면 수년은 두고 두고 쓸수 있는 개발 환경들이 있지만 임베디드 개발 환경의 경우 MCU 가 바뀔 때마다 개발환경을 바꾸어야만 하는 경우가 생긴다. 
개발 환경의 범위를 최소 셋인 컴파일, 다운로드 만이 아닌 디버깅, unit test 까지로 고려한다면 범위가 더 좁아 질 수 밖에 없다. 

지금까지는 대부분의 프로젝트는 임베디드 Linux 와 마찬가지로 gcc, binutils, gdb, OpenOCD 을 이용하여 개발하였고, 디버깅을 지원하는 통합 개발 환경으로는 emacs를 사용하였다. 

그러다 최근에 다른 MCU를 사용하다 보니 칩 업체에서 [Keil MDK](https://www2.keil.com/mdk5), [IAR](https://www.iar.com/kr/products/architectures/arm/) 과 같은 상용 프로그램 용으로만 SDK를 제공하는 경우를 종종 만나게 된다. 굳이 gcc 환경으로 포팅할 만한 정도의 프로젝트가 아니다 보니, 코딩은 맥 emacs 에서, 컴파일, 디버깅은 윈도우 VM에서 하는 불편한 반복작업을 해야 하는 경우가 생긴다.

임베디드 개발에서도 PC 와 같은 통합 개발 환경은 없을까?

이런 고민에서 출발한 것이 [PlatformIO](https://platformio.org/) 일 것이다. 

PlatformIO는 한마디로 말하면 [Arduino](https://www.arduino.cc) 부터 다양한 MCU와 RTOS 를 지원하는 통합 개발 환경이라고 할 수 있다. 
물론 Linux 도 지원을 하지만 주 타겟은 RTOS를 사용하는 MCU 급 이다.

홈페이지를 보면 다음과 같이 2022년 6월 현재 48개의 플랫폼과 1,390여개의 EV board 를 지원한다고 한다.

{{< image src="platformio.png" width="600px" height="auto" caption="PlatformIO 홈페이지">}}

ESP, Atmel, STM32 Cube, Nordic NRF52, Linux ARM, ... 등 다양한 플랫폼을 지원하기 때문에 새로운 하드웨어 플랫폼을 사용할 때는 PlatformIO를 고려해 볼 수 있을 것이다.

이번 연재에서는 다음과 같이 나누어 설명을 할 예정이다. 

- PlatformIO (1) - 개요 및 특징
- PlatformIO (2) - STM32Cube Platform 개발
- PlatformIO (3) - STM32Cube FreeRTOS 적용
- PlatformIO (4) - PlatformIO 디버깅 
- PlatformIO (5) - PlatformIO Unit Test

이 글에서 예로 사용하는 보드는 Cortex-M3 STM32F103 MCU 를 이용한 [NUCLEO-F103RB](https://www.st.com/en/evaluation-tools/nucleo-f103rb.html) 을 예로 들어서 설명한다. 

## PlatformIO 의 범위

위의 캡쳐 이미지와 같이 홈페이지에서 PlatformIO를 다음과 같이 소개 하고 있다. 

- Prefessional collaborative platform for embedded development
    - A place where Developers and Teams have true Freedom!
    - No more vendor lock-in!

조금 과장 광고 같아 보이는 문구이지만, 직접 사용하다 보면 이 말에 수긍하게 될 것이다.

PlatformIO 를 사용하는 임베디드 개발자는 CLI(Command Line Interface) 툴과, Visual Studio Code Extension 를 이용한다. 
[Platform IDE for VSCode](https://platformio.org/install/ide?install=vscode)를 설치하면 CLI도 같이 설치가 되므로 보통 CLI는 따로 설치할 필요가 없다. 

개발 예로 드는 STM32 NUCLEO-F103RB는 PlatformIO를 이용하기는 하지만, 처음 초기화 코드 생성은 [STM32CubeIDE](https://www.st.com/en/development-tools/stm32cubeide.html)를 사용하여 시작하여 사용한다.
이렇게 되면 굳이 PlatformIO IDE를 사용치 않고, STM32CubeIDE를 사용해서 개발하면 되지 않을까 생각이 들지만 PlatformIO를 이용하면 그만큼의 장점이 있다. 

우선 장점을 나열해 보면 다음과 같다. 

- JAVA 기반의 Eclipse IDE 를 기반으로 만들어진 STM32CubeIDE 의 느린 반응 속도에 답답할 필요가 없다. 
- 익숙한 통합개발환경인 VSCode 에서 개발, 다운로드, 디버깅, 유니테스트 등 임베디드 개발에 필요한 모든 것을 할 수 있다.
- CI/CD 를 위한 Host Unit Test, Remote Development 를 이용한 안정적인 팀 프로젝트 관리가 가능하다. 

물론 하나의 통일된 개발환경이다 보니, STM32CubeIDE만 제공하는 기능을 사용하지 못한다는 단점이 있다. 
가장 대표적인 것은 SWO(Serial Wire Output), ITM(Instrumentation Trace Macrocell) 을 이용한 Serial Wire Viewer(SWV) real-time tracing 기능이다.

Hard realtime 을 요하는 환경에서는 real-time tracing 기능이 큰 장점을 발휘하겠지만, 
일반적인 IoT 임베디드 기기라면 팀 프로젝트를 위한 PlatformIO 의 기능이 더 부각될 수도 있어 보인다.

우선 PlatformIO Core의 기능부터 정리해 보면 다음과 같다.

- platformio.ini 파일의 설정에 따라서 툴체인, 플랫폼 SDK, 예제코드 등 자동 설치
- 별도의 make 파일 생성없이 자동으로 빌드 환경 구성
- Library Dependency Finder(LDF) 기능으로 소스 파일의 include를 참고하여 자동으로 의존성 있는 라이브러리 빌드
- J-Link 등 다양한 디버깅 툴을 이용한 다운로드, 디버깅 지원
- Serial Monitor 기능 제공하여 시리얼 디버깅 콘솔 출력 기능
- Unit Test 지원하여 host/target hibrid unit test 환경 제공 
- 위의 사항을 조합한 통합 개발 환경

간단히 말해서 platformio.ini 로 초기화를 잡아주고 CLI를 실행만 해주면 초기 틀이 잡혀지고, 
여기에 코드만 작성하면 바로 빌드, 로딩, 실행, 디버깅이 가능하다는 것이다. 

위와 같은 CLI 기능이 VSCode frontend 로 감싸진 형태라 VSCode 에 익숙한 개발자라면 러닝 커브없이 임베디드 개발환경을 익힐 수 있다.

## VSCode Extension 설치하기

VSCode Extension에서 아래 패키지를 설치하면 된다.

- [PlatformIO IDE](https://marketplace.visualstudio.com/items?itemName=platformio.platformio-ide)

설치가 되면 PlatformIO는 다음과 같이 접근 할 수 있다. 

- VSCode 좌측 Active Bar의 개미얼굴 모양의 아이콘으로 PIO Quick Access 열기
- VSCode 하단 Status Bar의 집 모양 아이콘으로 PIO Home 열기

열려는 프로젝트의 루트 폴더에 platformio.ini 가 있으면 자동으로 PlatformIO 가 인식되어 빌드 등의 동작을 수행할 수 있다.

만일 CLI를 직접 실행 해 보려면 다음과 같이 실행할 수 있다.

- PIO Quick Access -> Miscellaneous -> PlatformIO Core CLI
- VSCode 명령창에서 "PlatformIO: Open PlatformIO Core CLI"

PlatformIO는 필요한 파일들을 홈디렉토리의 .platformio 폴더에 생성한다. 

NUCLEO-F103RB 용으로 프로젝트를 생성한 이후에는 해당 폴더에 다음과 같이 파일과 디렉토리가 생성된다. 

```shell
$ cd ~/.platformio
$ tree -L 2
.
├── appstate.json
├── homestate.json
├── packages
│   ├── contrib-piohome
│   ├── framework-stm32cubef1
│   ├── tool-cppcheck
│   ├── tool-dfuutil
│   ├── tool-ldscripts-ststm32
│   ├── tool-openocd
│   ├── tool-scons
│   ├── tool-stm32duino
│   └── toolchain-gccarmnoneeabi
├── penv
│   ├── bin
│   ├── include
│   ├── lib
│   ├── pip.conf
│   ├── pyvenv.cfg
│   └── state.json
├── platforms
│   ├── native
│   └── ststm32
└── python3
    ├── bin
    ├── include
    ├── lib
    ├── package.json
    └── share
```

각각의 폴더의 특징을 설명하면 다음과 같다.

- **python3, penv**: Platform Core CLI를 Python으로 작성되어 있다. Python 과 PlatformIO 툴이 설치
- **platforms**: 사용하는 플랫폼이 다운로드 됨. ststm32는 STM32 platform 이고, native 는 host 에서 unit test 를 위하여 설치한 것이다.
- **packages**: Framework와 필요한 툴들이 설치 
    - **framework-stm32cubef1**: STM32CubeF1 Framework 
    - **tool-???**: 툴체인, OpenOCd, 다운로드 프로그램, SCons 등 필요한 툴이 자동으로 설치됨.

## 정리

이상으로 간단하게 PlatformIO의 특징을 정리하고, 
다음 글 PlatformIO (2) - STM32Cube Platform 개발 에서는 PlatformIO를 이용하여 NUCLEO-F103RB 용 프로젝트를 생성하고 개발하는 과정을 설명한다.