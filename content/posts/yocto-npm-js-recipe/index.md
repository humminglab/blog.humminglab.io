---
title: "Yocto 에서 NPM 기반의 Javascript 패키지 관리"
date: "2023-08-21T16:00:00+09:00"
lastmod: "2023-08-21T16:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Yocto", "Javascript", "NPM", "NodeJS", "React", "Webpack"]
categories: ["Development"]
---

[Yocto에서 Go 프로젝트 관리]({{< ref "posts/yocto-golang-recipe">}})에 추가하여 JavaScript 기반의 프로그램을 Yocto의 패키지로 관리하는 방법을 정리해 본다.

관련 사항은 Yocto Wiki 의 NPM 기반 패키지 관리 방법에 간략하게 설명되어 있다.

- [TipsAndTricks/NPM - Yocto Project](https://wiki.yoctoproject.org/wiki/TipsAndTricks/NPM)

Javascript 기반의 프로젝트도 Go 언어와 마찬가지로 패키지 관련한 문제가 있지만 이 부분은 어느정도 툴을 이용하여 해결된 상태이다.

임베디드 환경에서 Javascript NPM 를 사용하는 경우를 크게 보면 다음 두 경우가 있을 수 있다. 

- Node.js 기반의 프로젝트
- Webpack/React와 같은 static page 생성

Node.js와 같은 프로젝트는 빌드 시 `nodejs-native` 도 필요하지만 target에서 동작하는 `nodejs`도 필요하다. 
하지만 webpack과 같은 경우에는 `nodejs-native`만 있어서 configure, compile task에서 이를 이용하여 페이지를 생성하면 되고, 별도로 target에 `nodejs`를 설치할 필요가 없다. 

둘은 recipe를 만드는 방법이 차이가 있어 이 둘을 구분하여 설명한다. 

## eSDK and devtool

위 TipsAndTricks/NPM 문서를 보면 devtool 을 이용하여 초기에 bitbake recipe를 생성한다. 
이 devtool 이 만들어진 배경과 용도에 대해서 우선 간략히 설명한다. 

Yocto 의 초기에는 SDK 빌드만 제공하였다. SDK 자체는 이해 하기가 명확하다. 

간단하게 말하면 Yocto 기반의 rootfs image 위에서 개발하는 사람들을 위한 개발환경 이라고 할 수 있다. 
Embedded linux system에 대해서만 알고 있으면 SDK를 이용하여 필요한 C, C++ 프로그램을 타겟용으로 빌드하여 실행 시키는 용도로, 해당 개발자는 Yocto 에 대해서 몰라도 된다.

우리가 보통 말하는 다음과 같은 툴체인을 빌드하는 것이라고 생각하면 된다. 
- [Arm GNU Toolchain – Arm Developer](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads)
- [Builds & Downloads | Linaro](https://www.linaro.org/downloads/)

SDK 에는 다음과 같은 것들이 포함되어 있다.

- Cross-toolchain
- Header & libraries
- Debuggig & tracing tools
- System utilities 
- Package configuration tools
- Documentation

이들을 포함하여 설치 가능한 install script로 만들어 주는 것이 Yocto SDK 이다. 

빌드 방법은 아래 처럼 사용하는 이미지에 `populate_sdk` task로 빌드할 수 있다. 

```shell
$ bitbake core-image-miniaml -c populate_sdk
```

반면에 eSDK(Extensible Software Development Kit)는 C, C++ 프로그램 개발자가 아니라, 
Yocto package 개발자를 위한 것이라고 볼 수 있다. 

소규모 프로젝트라면 한 명의 Yocto 관리자가 있고, 나머지 개발자들은 SDK를 이용하여 프로그램을 개발 검증하여 repository에 넣어주면, yocto 관리자가 이를 위한 yocto 패키지를 업데이트하면 프로젝트 운영이 가능할 수 있다. 

하지만 yocto 로 개발하는 부분이 더 큰 규모의 프로젝트라면 yocto 패키지를 여러명이 개발 하여야 할 수 있다. eSDK는 이를 위한 개발 환경이라고 보면 된다. 
다수의 yocto 개발자가 원격 서버를 통하여 yocto 빌드를 관리하고, 로컬에서는 shared state(sstate) cache를 이용하여 사전에 빌드된 것은 패키지 관리자처럼 별도의 빌드없이 바로 사용 가능하고, 원하는 recipe를 편리하게 추가나 수정할 수 있는 기능을 제공한다.

빌드 방법은 SDK와 유사하지만 실제 사용방법은 그리 쉽지는 않다.

```shell
$ bitbake core-image-miniaml -c populate_sdk_ext
```

eSDK는 2016년에 릴리즈된 [Yocto Krogoth (2.1)](https://wiki.yoctoproject.org/wiki/Extensible_SDK) 부터 정식으로 포함되었고, 여기에 포함된 툴 중의 하나가 devtool 이다. 

Devtool은 eSDK 환경 만이 아니라 yocto bitbake 빌드환경이면 기본 설치되어 있어, 일반 용도로 사용도 가능하다. 

Devtool은 기본 동작은 다음과 같다.

- build/workspace 로 임시 작업 layer를 만들어서, 이를 build/conf/bblayer.conf에 추가한다. 
- `devtool add`, `devtool modify` 와 같은 명령을 하면 이 workspace 안에 .bb 또는 .bbappend 파일을 만들고 관련된 소스들도 이곳에 설치하여 개발을 할 수 있도록 한다. 
- 이때 cmake 처럼 표준화된 소스는 자동으로 recipe의 내용도 만들어 준다. 

Git 에 익숙한 사용자라면 git workspace 개념과 유사하다고 보면 된다. 수정은 workspace에서 하고, 최종적으로 작성이 완료되면 `devtool finish`를 이용하여 기존의 bitbake layer(local repository)로 commit 하는 구조라고 이해하면 된다.

NPM 기반의 소스코드도 devtool 에서 인식하여 recipe를 만들어 주어, 이를 이용하면 NPM 패키지를 쉽게 생성할 수 있다. 

## Node.js 기반의 프로젝트

Node.js 기반의 프로젝트는 위에서 설명한 devtool을 활용하는 것이 좋다. 
설치 절차는 [TipsAndTricks/NPM - Yocto Project](https://wiki.yoctoproject.org/wiki/TipsAndTricks/NPM) 와 같은 동일한 예제로 설명한다. 

초기에 recipe를 생성하는 것은 devtool을 이용하여 다음과 같이 실행한다.

```shell
$ devtool add https://github.com/martinaglv/cute-files.git
NOTE: No setscene tasks
NOTE: Executing Tasks
NOTE: Tasks Summary: Attempted 2 tasks of which 0 didn't need to be rerun and all succeeded.
NOTE: Checking if npm is available ...
NOTE: Generating shrinkwrap file ...
NOTE: Fetching npm dependencies ...
NOTE: Handling licences ...
...
```

실행이 성공하면 `build/workspace` 안에 파일들이 생성되고, `build/conf/bblayer.conf` 에도 workspace 가 추가된다. 

Workspace의 파일 구성을 보면 다음과 같다. 
```
workspace
├── README
├── appends
│   └── cute-files_git.bbappend
├── conf
│   └── layer.conf
├── recipes
│   └── cute-files
│       ├── cute-files
│       │   └── npm-shrinkwrap.json
│       └── cute-files_git.bb
└── sources
    └── cute-files
        ├── LICENSE
        ...
```

`recipes/cute-files/cute-files_git.bb`를 보면 다음과 같다. 내용을 이해하기 위하여 불필요한 줄은 삭제하였다.

```
SUMMARY = "Turn any folder on your computer into a cute file browser, available on the local network."
LICENSE = "MIT & Unknown & ISC"
LIC_FILES_CHKSUM = "file://LICENSE;md5=71d98c0a1db42956787b1909c74a86ca \
                    file://node_modules/has/LICENSE-MIT;md5=d000afc3c9ff3501a5610197db76a246 \
                    ...
                    file://node_modules/utils-merge/package.json;md5=0230ade39b9c19f5fcc29ed02dff4afe \
                    file://node_modules/vary/package.json;md5=3577fc17c1b964af7cfe2c17c73f84f3"

SRC_URI = " \
    git://github.com/martinaglv/cute-files.git;protocol=https;branch=master \
    npmsw://${THISDIR}/${BPN}/npm-shrinkwrap.json \
    "
PV = "1.0.2+git${SRCPV}"
SRCREV = "98fe76448b8367adf206de6809b4adb7189b05ee"

S = "${WORKDIR}/git"

inherit npm

LICENSE:${PN} = "MIT"
LICENSE:${PN}-accepts = "MIT"
...
LICENSE:${PN}-utils-merge = "MIT"
LICENSE:${PN}-vary = "MIT"
```

위 파일을 보면 다음과 같은 사항을 확인할 수 있다. 

- 버전 변경 문제를 해결하기 위하여 [npm-shrinkwrap](https://docs.npmjs.com/cli/v8/commands/npm-shrinkwrap) 을 이용하여 의존성있는 패키지의 버전을 고정시킨다. 
  파일은 `recipes/cute-files/cute-files/npm-shrinkwrap.json` 으로 생성된다. 
- 패키지의 라이센스도 자동으로 확인하여 패키지별로 명시된다.

이렇게 되면 [Reproducible Builds](https://docs.yoctoproject.org/test-manual/reproducible-builds.html) 를 보장할 수 있고, 라이센스도 관리할 수 있다.

`appends/cute-files_git.append` 를 보면 externsrc를 이용하여 로컬 소스로 빌드가 가능하도록 설정되어 있다. 
Exteralsrc 에 대해서는 이전에 정리한 [Yocto Project 개발하기(3) - 개발 시 로컬 패키지 관리하기]({{< ref "posts/yocto-project-on-orange-pi-3">}})를 참고할 수 있다.

```
inherit externalsrc
EXTERNALSRC = "/home/yslee/project/yocto-project/build/workspace/sources/cute-files"
EXTERNALSRC_BUILD = "/home/yslee/project/yocto-project/build/workspace/sources/cute-files"

# initial_rev: 98fe76448b8367adf206de6809b4adb7189b05ee
python do_configure:append() {
    pkgdir = d.getVar("NPM_PACKAGE")
    lockfile = os.path.join(pkgdir, "singletask.lock")
    bb.utils.remove(lockfile)
```

패키지 수정이 모두 완료 된 후에는 아래와 같이 필요한 layer로 옮길수 있다.

```shell
$ devtool finish cute-files ../meta-my-project/recipes-core 
```

### Node.js 빌드 과정

위와 같이 devtool을 이용하여 생성된 recipe는 `npm.bbclass`를 상속받아서 동작한다. 
이 부분의 절차가 복잡하기는 한데, 간단하게 설명하면 다음과 같다.

- configure: 작업 파일에 npm-package로 패키지를 설치하고, npm package manager의 역할과 유사하게 npm-cache에는 의존성 있는 모든 패키지를 받아온다. 
- compile: `npm install` 과정을 수행한다. 이때 의존성 있는 패키지는 npm-cache 폴더를 참조하여 설치한다. 최종 결과물은 npm-build에 생성된다. 
- install: npm-build 결과물을 기반으로 설치 파일 추출

```shell
$ ls tmp/work/cortexa7t2hf-neon-poky-linux-gnueabi/cute-files/1.0.2+git999-r0
npm-build  npm-cache  npm-package  recipe-sysroot  recipe-sysroot-native  temp
```

만일 devtool로 정상적으로 인식되지 않는 repository 라면 위 생성된 것을 참조하여 유사하게 recipe 를 직접 만들어야 한다.


## Webpack/React와 같은 static page 생성

이들 프로젝트에 대해서는 npm.bbclass 을 사용하기에는 오히려 더 복잡해진다. 
이 경우에는 아래처럼 간단하게 recipe를 만들어 사용할 수 있다. 

```
SRC_URI = " \
    git://git@gitlab.com/humminglab/test-react-frontend.git;protocol=ssh;branch=main \
    "
SRCREV = "${AUTOREV}"

S = "${WORKDIR}/git"

DEPENDS = "nodejs-native"

NPM_NODEDIR ?= "${RECIPE_SYSROOT_NATIVE}${prefix_native}"

do_configure() {
    ${NPM_NODEDIR}/bin/npm install
}

do_compile() {
    ${NPM_NODEDIR}/bin/npm run build
}

do_install() {
    install -d ${D}${datadir}/www-pages
    cp -r ${S}/build/* ${D}${datadir}/www-pages
}
```

NPM과는 다음과 같은 차이가 있다. 
- 의존성은 `nodejs-native`만 명시
- configure: 네트워크 접속이 가능하므로 `npm install` 로 webpack/react 등의 관련된 development package 설치 
- compile: `npm run build`로 static 결과물 생성
- install: 생성된 파일들 추리기 

위와 같이 하면 target 의 `/usr/share/www-pages`에 생성된 파일들이 설치된다.

## 정리

Javascript 는 Yocto에서 어느정도 잘 관리되는 상태로 보여진다. 
Go, Rust 등의 다른 패키지 관리자가 포함된 언어들도 Yocto 프로젝트가 버전업 되면서 이처럼 정리될 것으로 기대된다. 
