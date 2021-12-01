---
title: "Yocto Project History"
date: "2019-01-06T09:00:00+09:00"
lastmod: "2019-01-06T09:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Yocto"]
categories: ["Embedded Linux"]
aliases: [/yocto-project-history/]
---

[Yocto Project](https://www.yoctoproject.org/)를 보면 OpenEmbedded, bitbake, poky 와 같은 용어들이 나온다. Bitbake는 Yocto Project의 make 툴이라고 이해하면 되는데, 다른 용어는 어떤 의미 인지 모호할 수 있다.
[OpenEmbedded](https://www.openembedded.org/wiki/Main_Page)와 Yocto Project와의 관계는 어떤 것인지, Poky는 범위가 어떤 것인지 메뉴얼을 보아도 정확히 감을 잡기가 어렵다.

개념적으로 잘 정리된 관계는 [Yocto Project Overview and Concepts Manual, 2.1 What is the Yocto Proejct?"](https://www.yoctoproject.org/docs/2.6/overview-manual/overview-manual.html#what-is-the-yocto-project)에 있는 아래 그림이다.

{{< image src="poky-reference-distribution.png" width="800px" height="auto" caption="Yocto Elements">}}


우선은 이를 이해하기 위하여는 Yocto Project가 발전한 변천사를 보는 것이 좋다.


## Yocto Project 변천사
OpenEmbedded는 iPAQ PDA 에 linux 를 올린 [Familiar Linux (2000년)](https://en.wikipedia.org/wiki/Familiar_Linux), 2001년 출시된 최초의 linux PDA인 [Sharp Zaurus](https://en.wikipedia.org/wiki/Sharp_Zaurus) SL-5000D 롬을 재패키징 한 [OpenZaurus](https://en.wikipedia.org/wiki/OpenZaurus) 프로젝트 부터 시작되었다고 할 수 있다.
2003년 이같은 리눅스 PDA 프로젝트를 코드를 기반으로 OpenEmbedded  프로젝트가 생성되었다.

2005년 bitkeeper에서 git으로 repository를 변경하고, 2010년 경에는 8000 개 가까이 recipe가 만들어질 정도로 활발하게 운영되었다. 하지만 pull request model이 아니라 개발자가 직접 repository에 commit 을 하도록 운영을 하다보니 안정성 등 여러가지 문제점들이 나오기 시작했다.

OpendHand 라는 스타트업에서는 2005년 부터 OpenEmbedded의 repository와는 별도의 repository를 만들어 PokyLinux 라는 프로젝트를 진행하였다.

{{< image src="poky-beaver.png" width="200px" height="auto" caption="PokyLinux 로고">}}


PokyLinux는 OpenEmbedded와는 다음과 같은 차별화를 가졌다.
* 양적으로 발산하는 OpenEmbedded와는 달리 제대로 관리하는 600여개의 recipe
* GMAE(Gnome Mobile and Embedded) 플랫폼으로 최적화된 패키지 관리
* QEMU emulation 및 GDB, profile, LTTng 와 같은 디버깅 환경을 지원
* 무엇보다도 지금의 Yoco Project 문서와 같이 깔끔하게 잘 정리된 매뉴얼 지원

인텔에서 2008년 Q3에 OpenedHand를 인수하여, 이를 기반으로 2010년에 Linux Foundation Collaborative Project인 [Yocto Project](https://www.yoctoproject.org/)를 만들었다.

Yocto Project가 되면서 유연하게 관리하기 위한 meta layer 등이 추가 되면서 최근에는 embedded Linux SDK를 배포하는 대부분의 업체들이 Yocto Project에 layer를 추가하여 SDK를 배포하고 것으로 변경하여, Yocto Project는 업계 표준이 되었다고 볼수 있다.

OpenEmbedded도 Yocto Project 가 만들어진 2010년에 기존 프로젝트는 “OpenEmbedded-Classic” 이라고 하고,  pull request 관리 방식으로 새로 만든 것이 “OpenEmbedded-Core”이다. OpenEmbedded라고 하면 이 “OpenEmbedded-Core” 를 말한다.

OpenEmbedded와 Yocto Project는 협력 관계로 목표점이 다르다.
* OpenEmbedded는 빌드 시스템이고, 별도로 배포본 (distribution)을 관리하지는 않는다.
* Yocto Project는 Umbrella project로 표준 배포본이라고 할 수 있다.


## Yocto, OpenEmbedded, Poky
위 히스토리를 이해하고, 아래 그림을 다시 보면 각각의 구분이 좀더 명확히 보인다.

{{< image src="poky-reference-distribution.png" width="800px" height="auto" caption="Yocto Elements">}}

* OpenEmbedded
	* 디렉토리: meta-openembedded
	* Bitbake를 포함한 빌드 시스템
	* OpenEmbedded-Core recipe는 특정 하드웨어를 타겟으로 하지 않고 QMEU machine을 타겟으로 함
* Poky
	* 디렉토리: meta-poky, meta-yocto-bsp
	* Yocto project의 레퍼런스 배포본
	* 타겟은 QEMU machine과 레퍼런스 보드 몇 종 (genericx86, [Freescale MPC8315E-RDB](https://old.yoctoproject.org/downloads/bsps/thud26/mpc8315e-rdb), [TI ARM Cortex-A8 Beaglebone](https://old.yoctoproject.org/downloads/bsps/daisy16/beaglebone), [Ubiquiti Networks EdgeRouter Lite](https://old.yoctoproject.org/downloads/bsps/fido18/edgerouter))
* Yocto Project
	* 다양한 BSP 및 시스템 환경을 위한 layer 모음
	* Yocto Project를 위한 개발 툴  및 문서


## 참고링크
* [Embedded Linux timeline](http://2net.co.uk/embedded-history)
* [Integrating Yocto and OpenEmbedded - ELC 2011(PDF)](http://elinux.org/images/d/de/Elc2011_kooi.pdf)
* [PokyLinux: Mobile GNOME at your fingertips(PDF)](https://elinux.org/images/a/a2/Poky.pdf)


## 마치며
위 사항을 이해한 상태에서 [Yocto Project - Software Overview](https://www.yoctoproject.org/software-overview/)를 보면 어느 정도 설명의 범주가 이해될 것이다. 이후로는 방대한 매뉴얼들을 차례대로 보면서 Yocto Proejct를 익히면 될 것이다.