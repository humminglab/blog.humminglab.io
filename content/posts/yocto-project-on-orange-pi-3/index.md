---
title: "Yocto Project 개발하기(3) - 개발 시 로컬 패키지 관리하기"
date: "2022-01-18T23:00:00+09:00"
lastmod: "2022-01-18T23:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Yocto", "Linux", "OrangePi"]
categories: ["Development"]
series: ["Yocto Project 개발하기"]
---

Yocto Project 개발 절차는 기본적으로 이미 개발이 완료된 패키지들의 recipe를 만들고, 이를 bitbake로 전체 이미지를 빌드하는 과정이다. 

하지만 개발 진행 중인 소프트웨어 패키지를 yocto에 추가하고, 이를 계속 수정 개발을 할 때 어떤 방식으로 관리를 할지 고민을 하여야 한다. 
여기에서는 이와 같이 개발 중인 패키지를 yocto 에 추가하여 빌드 하는 방법을 알아본다.

- [Yocto Project 개발하기(1) - Orange Pi 보드 빌드]({{< ref "posts/yocto-project-on-orange-pi-1">}})
- [Yocto Project 개발하기(2) - Custom Layer 만들기]({{< ref "posts/yocto-project-on-orange-pi-2">}})
- **Yocto Project 개발하기(3) - 개발 시 로컬 패키지 관리하기**
- Yocto Project 개발하기(4) - Yocto SDK 빌드
- Yocto Project 개발하기(5) - Yocto eSDK 이용한 개발 모델

## 방법들 

예를 들어 u-boot 를 수정 하는 경우를 예로 들어 보자. 
수정되어 commit 된 경우라면 bb(append) recipe만 수정해서 재실행하면 되지만, commit 전에 yocto 에 추가하여 빌드하려면 다음과 같은 방법을 고려해 볼 수 있다. 


1. yocto build 디렉토리에 있는 소스를 수정하고 빌드하기 
2. devshell task 를 실행하여 직접 수작업 빌드하기
3. externalsrc.bbclass 를 이용하여 poky 외부 소스를 참조하여 빌드하기


첫번째 방법은 u-boot의 경우 소스코드는 `tmp/work/sc_gateway-poky-linux-gnueabi/u-boot/1_2021.07-r0/git` 에 다운로드 받아서 빌드된다. 이 디렉토리에 코드를 수정 후 아래 처럼 강제로 compile 부터 재실행하면 된다. 

```shell
$ bitbake u-boot -c compile -f && bitbake u-boot 
```

이 방식의 단점은 실수로 clean, fetch, patch 등의 task를 수행하면 수정한 부분이 사라질 수 있다. 

두번쨰 방식은 아래처럼 u-boot 빌드하는 shell 환경을 직접 실행하는 것이다.

```shell
$ bitbake u-boot -c devshell
```

Shell 은 위 소스 디렉토리에서 실행되고, `export`로 보면 빌드에 필요한 환경 변수들이 모두 잡혀 있는 것을 확인할 수 있다. 이 shell에서 다음처럼 하면 bitbake로 compile 하는 것과 동일한 작업을 수행한다. 

```shell
$ ../temp/run.do_compile
```

이 shell에서 다른 소스 코드를 빌드해 볼 수 있는데 번거롭다.


마지막 방법이 여기에서 설명할 방법으로 bitbake 빌드 과정과 동일하지만 소스 디렉토리, 빌드 디렉토리만 외부 폴더로 바꾸는 방법이다. 


## Externalsrc

관련 설명은 아래에서 찾을 수 있다. 
- [3.10.6 Building Software from an External Source](https://docs.yoctoproject.org/dev-manual/common-tasks.html#building-software-from-an-external-source)
- [5.31 externalsrc.bbclass](https://docs.yoctoproject.org/ref-manual/classes.html?highlight=externalsrc#externalsrc-bbclass)
- [EXTERNALSRC](https://docs.yoctoproject.org/ref-manual/variables.html?highlight=externalsrc#term-EXTERNALSRC)


externalsrc.bbclass 의 기능을 이용하여 특정 recipe의 소스 디렉토리를 외부로 변경할 수 있다. 

예를 들어 /home/yslee/u-boot 으로 clone 한 소스가 있고, 이를 bitbake 로 빌드를 하고 싶다면 local.conf 에 아래와 같이 추가하면 된다. 

```
INHERIT += "externalsrc"

EXTERNALSRC:pn-u-boot = "/home/yslee/u-boot"
EXTERNALSRC_BUILD:pn-u-boot = "/home/yslee/u-boot"
```

- 첫번쨰 줄은 externalsrc.bbclass를 추가하는 것이다. 이와 같은 conf 파일에는 [INHERIT](https://docs.yoctoproject.org/bitbake/bitbake-user-manual/bitbake-user-manual-metadata.html#inherit-configuration-directive) 문법을 사용하고, .bb recipe 파일에는 [inherit directive](https://docs.yoctoproject.org/bitbake/bitbake-user-manual/bitbake-user-manual-metadata.html#inherit-directive) 를 사용한다. 
- `EXTERNALSRC:pn-`{PACKAGE_NAME} 로 패키지의 소스 디렉토리를 절대 경로로 기록한다. 이와 같이 하면 u-boot 는 빌드 시 recipe 에 정의된 소스 파일을 사용하지 않고, 이 디렉토리의 소스로 빌드를 수행한다. 
- `EXTERNALSRC_BUILD:pn-`{PACKAGE_NAME} 로 패키지의 빌드 디렉토리를 절대 경로로 기록한다. 이 부분은 필요시에 추가하면 된다. 이를 추가하게 되면 compile 시 생성되는 파일도 해당 디렉토리에 생성된다. 실제 object 파일 등을 확인하려고 할때는 이와 같이 source 디렉토리에 생성되도록 하면 편리할 수도 있다.

## 사용 예

EXTERNALSRC 는 패키지 소스 수정이 필요한 경우 다음과 같은 절차로 진행한다. 

- 패키지의 소스를 외부 디렉토리로 다운로드 한다. 이와 같은 외부 빌드를 하는 경우 bitbake fetch, unpack, patch 과정 대신에 외부 소스를 참조하는 것이기 떄문에, 만일 recipe에서 소스코드를 patch 한다면 이는 수작업으로 외부 소스에 반영하여야 한다. 
- local.conf 파일에 위와 같인 externalsrc를 추가하고, 외부 소스도 패키지 이름으로 EXTERNALSRC 으로 등록해 준다. 
- `bitbake package_name` 와 같이 실행하면 해당 패키지가 컴파일 되고 다음 과정으로 진행한다. 
- 코드 수정이 완료되면 git commit을 하고, 해당 tag를 recipe에 업데이트 한다. 
- 최종적으로 local.conf의 EXTERNALSRC 부분을 삭제하고, 재빌드 해서 recipe에 정상적으로 반영되는지 확인한다.

위와 같이 externalsrc 로 등록한 패키지의 경우 해당 compile task는 tained mode로 설정되어 `bitbake -f -c compile` 과 같이 force 옵션을 주지 않아도 매번 compile을 수행한다.

이와 같이 externalsrc 로 가장 자주 사용하는 부분은 kernel 수정 작업 시 이다. 커널의 경우 externalsrc로 설정하고 수정 후 빌드를 하면 의존성 있는 kernel module package 나 initramfs 와 같은 recipe도 재빌드 되어 수작업으로 패키지을 할 필요가 없다.