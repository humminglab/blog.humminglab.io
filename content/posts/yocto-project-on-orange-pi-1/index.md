---
title: "Yocto Project 개발하기(1) - Orange Pi 보드 빌드"
date: "2021-12-30T22:00:00+09:00"
lastmod: "2022-01-09T13:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Yocto", "Linux", "OrangePi"]
categories: ["Development"]
series: ["Yocto Project 개발하기"]
---

이 문서는 [Orange Pi Zero](http://www.orangepi.org/orangepizero/) 보드에서 [Yocto Project](https://www.yoctoproject.org/)를 이용하여 배포본을 만들고, 개별 패키지를 관리하는 방법에 대하여 설명한다.

설명은 다음과 같이 나누어서 설명을 할 에정이다.

- **Yocto Project 개발하기(1) - Orange Pi 보드 빌드**
- [Yocto Project 개발하기(2) - Custom Layer 만들기]({{< ref "posts/yocto-project-on-orange-pi-2">}})
- Yocto Project 개발하기(3) - 개발 시 로컬 패키지 관리하기
- Yocto Project 개발하기(4) - Yocto SDK 빌드
- Yocto Project 개발하기(5) - Yocto eSDK 이용한 개발 모델

이 문서에서는 첫번째 과정인 필요한 layer를 추가해서 빌드하여 타겟에 올려서 동작을 확인하는 과정을 설명한다.

## Orange Pi Zero

Orage Pi Zero 보드는 아래와 같은 기능을 제공한다. [Allwinner](https://www.allwinnertech.com/) H2+ Cortex-A7 Quad-Core CPU, Mali400MP2 GPU, 256MB SDRAM, 2MB SFlash, 그리고 TF card 메모리 인터페이스를 지원하여 작은 gateway 기능으로 적합할 수 있다. 배터리를 고려한 power management가 필요치 않다면 가성비로는 좋은 편이다.

{{< image src="orangepizero.png" width="600px" height="auto" caption="Orange Pi Zero Board">}}

보드에 대한 정보는 다음에서 찾아볼 수 있다.

- [Orage Pi Resource](http://www.orangepi.org/downloadresources/)
- [Linux Sunxi](https://linux-sunxi.org/H3#Variants)

Android 나 [Armbian](https://www.armbian.com/) 패키지는 제공하지만 gateway box로 구성하기에는 이보다는 BuildRoot 또는 Yocto를 이용하여 커스텀 롬을 만들어 내는 것이 OTA 등의 관리적인 측면에서 효과적이다.

## Buildroot vs. Yocto Project

[Buildroot](https://buildroot.org/) 는 패키지를 위한 빌드 툴로 사용 방법이 직관적이지만, [Yocto Project](https://www.yoctoproject.org/)는 개발 단계에서 어떻게 사용할지 명확치 않고 너무 복잡하다고 느낄 수 있다.

간단하게 설명하면 Buildroot는 루트 이미지를 만드는 것이 목적인 빌드 툴이라고 보면 된다. 의존성에 따라서 패키지들을 빌드하여 root image를 결과물로 만들어 낸다.

Yocto의 경우는 임베디드판 Ubuntu/Debian 과 같은 배포본 빌더라고 볼 수 있다. 즉, 각 패키지를 빌드하여 rpm/deb 를 만들고 이를 패키지 매니저를 이용하여 배포본을 만들어 낸다.
Ubuntu와 차별점이라면 하드웨어 사양이 제 각각인 임베디드 시스템에 맞도록 커스터마이징을 쉽게 할 수 있다는 것이다.

여기에서는 Yocto를 선정하여 배포본을 만들고, 개발하는 패키지를 개발 단계에서 Yocto project로 어떻게 관리하는지를 정리한다.

## Yocto 개발 환경 구성

Linux kernel 빌드는 아직까지는 linux 컴퓨터에 gcc 로만 가능하다. 또한 Yocto 빌드 시 사용하는 utility도 linux 용이라 결과적으로 linux 이외에서 개발은 불가능하다.
Yocto 빌드는 메모리, CPU 자원을 많이 사용하므로 성능이 어느정도 받쳐 주는 linux 환경을 준비하여야 한다.

Yocto는 빌드하는 기기 마다의 결과 차이를 없애기 위하여 gcc 나 빌드에 필요한 모든 것을 직접 빌드하여 사용한다. 관련 사항은 [Quick Build](https://docs.yoctoproject.org/brief-yoctoprojectqs/index.html#compatible-linux-distribution) 문서를 참고할 수 있다.

Ubuntu 라면 다음과 같이 yocto에 필요한 툴을 설치한다.

```shell
$ sudo apt install gawk wget git diffstat unzip texinfo gcc build-essential \
 chrpath socat cpio python3 python3-pip python3-pexpect xz-utils debianutils \
 iputils-ping python3-git python3-jinja2 libegl1-mesa libsdl1.2-dev pylint3 \
 xterm python3-subunit mesa-common-dev zstd liblz4-tool
```

최신 버전인 Yocto 3.4.1 (honister) 을 다운로드 한다.

```shell
$ git clone git://git.yoctoproject.org/poky -b honister
```

Yoco 내에 Allwinner layer인 meta-sunxi 를 설치한다. 또한 meta-sunxi 가 참조하는 meta-openembedded/meta-oe 도 추가로 설치한다.

```shell
$ cd poky
$ git clone git@github.com:linux-sunxi/meta-sunxi.git -b honister
$ git clone git://git.openembedded.org/meta-openembedded -b honister
```

위와 같이 meta layer는 poky 디렉토리 안에 설치를 한다면 이를 git submodule 등을 이용하여 관리를 하는 것이 좋을 수 있다.

```shell
$ source ./oe-init-build-env
```

위를 실행하면 메시지가 나오면서 poky 빌드에 필요한 환경 변수가 적절히 설정되고, 처음 실행이라면 `./build` 디렉토리를 만들고 기본 설정 파일이 다음과 같이 생성된다.

```shell
➜  build git:(honister) ✗ tree
.
└── conf
    ├── bblayers.conf
    ├── local.conf
    └── templateconf.cfg
```

`conf/bblayers.conf`를 에디터로 열어서 BBLAYERS 에 다음과 같이 meta-oe, meta-sunxi 를 추가한다.

```
BBLAYERS ?= " \
  /home/yslee/project/poky/meta \
  /home/yslee/project/poky/meta-poky \
  /home/yslee/project/poky/meta-yocto-bsp \
  /home/yslee/project/poky/meta-openembedded/meta-oe \
  /home/yslee/project/poky/meta-sunxi \
  "
```

`conf/local.conf`의 default MACHINE 을 orage-pi-zero 로 변경한다. 해당 설정은 `meta-sunxi/conf/machine/orange-pi-zero.conf` 에 위치한다.

```
MACHINE ??= "orange-pi-zero"
```

최소 셋으로 배포본을 빌드해 본다.

```shell
$ bitbake core-image-minimal
```

최종 결과물은 `build/tmp/deploy/images/orange-pi-zero` 디렉토리에 생성된다. 이 중 `core-image-minimal-orange-pi-zero.sunxi-sdimg` 파일이 sd card 용으로 생성된 이미지 이다.
이 파일을 sd card 에 다음과 같이 write 한다.

```shell
$ sudo dd if=core-image-minimal-orange-pi-zero.sunxi-sdimg of=/dev/sdc status=progress
```

좀 더 안전하게 쓰려면 다음과 같이 by-path 나 by-id 경로의 디바이스 파일을 사용하는 것이 실수로 컴퓨터 디스크에 쓰는 것을 막을 수 있다.

```shell
$ sudo dd if=core-image-minimal-orange-pi-zero.sunxi-sdimg of=/dev/disk/by-id/usb-Generic_STORAGE_DEVICE_000000001532-0:0 status=progress
```

SD card를 orange pi zero에 꽂고 부팅 시켜 보면 다음과 같은 로그가 출력된다.

[Orage Pi Boot Log](orange-pi-boot.log)

로그를 보면 ethernet 까지는 정상적으로 동작되는 것을 확인할 수 있다.
Wi-Fi 드라이버는 아직 설치가 안된 상태이다.

## 마무리

기본 동작을 확인하였고 다음은 프로젝트의 custom layer를 만들어 관리하는 방법을 설명한다.

이전에 작성한 글 [Yocto Project History]({{< ref "yocto-project-history" >}}) 도 한번 읽어 보면 전체적으로 Yocto의 발전 과정을 볼 수 있다.
