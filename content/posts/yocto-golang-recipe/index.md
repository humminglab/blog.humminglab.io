---
title: "Yocto 에서 Go 프로젝트 관리"
date: "2023-08-16T12:00:00+09:00"
lastmod: "2023-08-16T16:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Yocto", "Golang", "Go"]
categories: ["Development"]
---

Yocto recipe를 작성하다 보면 대부분의 프로젝트가 C, C++ 로 작성된 것들이라 이들은 참조할 것들이 많다.
하지만 Go, Rust, NodeJS 로 작성된 프로젝트는 Yocto에 추가하려다 보면 참고할 자료가 많지는 않은 편이다. 
이 글에서는 Go 언어로 작성된 프로젝트를 추가하는 방법을 정리한다.

## Go Module

우선 간단하게 Go 의 모듈 정책에 대해서 정리 해본다.

2009년에 Go 가 처음 나왔을때는 모듈관리는 단순했다. 
Go 프로젝트에서 참고하는 모듈은 `go get` 으로 다운로드하면 $GOPATH/src 디렉토리에 해당 모듈이 설치된다. 
하나의 예를 들면 다음과 같이 src 디렉토리에 모듈 경로를 포함해서 설치가 된다. 

```sh
$ find `go env GOPATH` -type d -depth 4
/Users/yslee/go/src/golang.org/x/sys
/Users/yslee/go/src/github.com/go-ble/ble
/Users/yslee/go/src/github.com/sirupsen/logrus
/Users/yslee/go/src/github.com/JuulLabs-OSS/cbgo
/Users/yslee/go/src/github.com/raff/goble
/Users/yslee/go/src/github.com/mattn/go-isatty
/Users/yslee/go/src/github.com/mattn/go-colorable
/Users/yslee/go/src/github.com/mgutz/ansi
/Users/yslee/go/src/github.com/mgutz/logxi
/Users/yslee/go/src/github.com/pkg/errors
```

이 방식의 가장 큰 단점은 모듈의 버전 관리가 안된다는 것이다. 
예를 들어 2개의 go 프로젝트에서 사용하는 모듈의 버전이 다르다면 문제가 발생한다
(Python의 PIP 패키지 방식의 문제와 유사하다).

2018년 릴리즈된 Go 1.11 부터는 모듈 패키지 관리 기능이 새로 추가되었다. `go mod`로 시작하는 명령이라고 보면 된다. 

패키지를 관리 기능이 지원하면서 프로젝트에는 `go.mod` 파일이 생성되어 패키지의 버전을 지정 할 수 있고, `go get`을 하면 아래 처럼 패키지가 설치된다.

```sh
$ find `go env GOPATH` -type d -depth 5
/Users/yslee/go/pkg/mod/cache/download/sumdb
/Users/yslee/go/pkg/mod/cache/download/gopkg.in
/Users/yslee/go/pkg/mod/cache/download/golang.org
/Users/yslee/go/pkg/mod/cache/download/github.com
/Users/yslee/go/pkg/mod/golang.org/x/sys@v0.0.0-20211204120058-94396e421777
/Users/yslee/go/pkg/mod/github.com/!juul!labs-!o!s!s/cbgo@v0.0.1
/Users/yslee/go/pkg/mod/github.com/go-ble/ble@v0.0.0-20230130210458-dd4b07d15402
/Users/yslee/go/pkg/mod/github.com/konsorten/go-windows-terminal-sequences@v1.0.1
/Users/yslee/go/pkg/mod/github.com/sirupsen/logrus@v1.5.0
/Users/yslee/go/pkg/mod/github.com/raff/goble@v0.0.0-20190909174656-72afc67d6a99
/Users/yslee/go/pkg/mod/github.com/mattn/go-colorable@v0.1.6
/Users/yslee/go/pkg/mod/github.com/mattn/go-isatty@v0.0.12
/Users/yslee/go/pkg/mod/github.com/mgutz/ansi@v0.0.0-20170206155736-9520e82c474b
/Users/yslee/go/pkg/mod/github.com/mgutz/logxi@v0.0.0-20161027140823-aebf8a7d67ab
/Users/yslee/go/pkg/mod/github.com/pkg/errors@v0.8.1
```

기존 방식과 달리 `pkg/mod` 에 설치되고, 디렉토리에 `@0.0.12` 와 같이 버전명이 추가된 것을 확인할 수 있다. 이렇게 하여 프로젝트들이 다른 모듈 버전을 사용하여도 문제없이 cache로 관리된다.

기존 방식을 사용할지 패키지 방식을 사용할지는 `GO111MODULE` 환경 변수로 설정할 수 있다. 이 환경 변수 설정도 Go 버전에 따라서 다르다. `GO111MODULE` 관련 설정은 아래 링크를 보면 Go 버전별로 정리가 된 글을 볼 수 있으나, 최신 버전에서는 계속 변하고 있다.

- [GO111MODULE](https://velog.io/@0xf4d3c0d3/GO111MODULE)

이 글에서는 패키지 관리 방식을 사용한 것만 대상으로 설명한다.

## Go for Yocto

Yocto에 C/C++ 언어가 아닌 Go 언어를 추가하는 것은 깔끔하지 않다. 

이는 Go 언어만의 문제가 아니라 패키지 관리자가 포함된 현대 프로그래밍 언어는 모두 해당되는 것으로 Yocto 가 지향하는 목표와 충돌하기 때문이다. 

첫번째로는 패키지 관리자 방식이 Yocto의 [Reproducible Builds](https://docs.yoctoproject.org/test-manual/reproducible-builds.html) 방식에 위배 된다. 

패키지 방식의 경우 프로젝트에서 참조하는 모든 하위 패키지들 버전이 정확히 패키지 관리자에 명시가 되지 않는 한 빌드 때마다 다른 버전을 참조하여 다른 결과물이 나올 수 있다. 

두번째로는 License 관리 문제이다. Yocto에서는 license를 엄격히 관리하여 최종 결과물에 참조된 사항들을 라이센스 조건에 맞도록 구분하여 관리할 수 있다. 하지만 이와 같이 패키지로 줄줄이 딸려 들어간 모듈들의 license 조건이 충돌 될 가능성이 있고, yocto 에서 이들 모듈을 라이센스 별로 분리가 안된다.

Go 프로젝트를 위하여 첫번째 문제를 해결하는 방법은, [Vendoring](https://go.dev/ref/mod#vendoring) 을 이용하는 것이다. 

`go mod vendor` 명령을 실행하면 아래처럼 참고하는 모든 패키지들이 프로젝트의 vendor 디렉토리 안에 모듈이 설치된다. 

```sh
$ go mod vendor 
$ find . -type d -depth 4
./vendor/golang.org/x/sys
./vendor/github.com/go-ble/ble
./vendor/github.com/konsorten/go-windows-terminal-sequences
./vendor/github.com/sirupsen/logrus
./vendor/github.com/JuulLabs-OSS/cbgo
./vendor/github.com/raff/goble
./vendor/github.com/mattn/go-isatty
./vendor/github.com/mattn/go-colorable
./vendor/github.com/mgutz/ansi
./vendor/github.com/mgutz/logxi
./vendor/github.com/pkg/errors
```

이와 같이 vendoring 기능을 이용하여 참조하는 모든 패키지를 repository 안에 포함하게 되면 항상 동일한 결과물을 얻을 수 있게 된다. 

이 경우 빌드를 하려면 아래처럼 `-mod`를 추가해 주어야 한다.

```sh
$ go build -mod=vendor
```

두번째 사항인 라이센스 관련 문제는 아직은 Yocto에서 Go 프로젝트는 해결 방안이 없는 것 같다. 위와 같이 vendoring을 하여 소스를 관리하는 것도 라이센스 조건에 위배될 수 있다. 

라이센스 조건은 이 글에서는 고민하지 않는 것으로 하나, Yocto 프로젝트 개발 시에는 이 부분에 대하여 고민을 해보아야 한다.


## Go Recipe 작성 방법

위에서 언급한 것과 같이 vendoring을 하는 경우에는 특별한 처리가 필요 없다. 

BB recipe를 다음과 같이 작성하면 된다.

```
SRC_URI = "git://git@${GO_IMPORT};branch=main"

SRCREV = "12345678"

GO_IMPORT = "github.com/humminglab/go-test" 

inherit go-mod
```

Go 언어는 go 패키지 처럼 위와 같이 git repository에서 받는 것만 고려되어 있다. 일반 C 프로젝트와 달리 위의 경우 github.com/humminglab/go-test 처럼 디렉토리가 생성되어 이곳에 소스가 설치된다.
그래서 local 에 있는 압축된 패키지를 사용하려는 경우에는 디렉토리 조정이 필요하다.

만일 vendoring 하지 않은 go 프로젝트를 yocto로 위처럼 만들어 빌드해보면 compile 시 패키지를 다운로드 받지 못한다는 proxy 관련 에러가 발생한다. 이유는 Yocto에서는 compile 시 network 접근은 금지하기 때문이다. 

이 부분은 임시로 patch 하려면 다음과 같이 BB 파일에 추가할 수 있다. 

```
# Workaround for network access issue during compile step
# this needs to be fixed in the recipes buildsystem to move
# this such that it can be accomplished during do_fetch task
do_compile[network] = "1"
```

위 라인을 추가하게 되면 compile 시 네트워크 사용이 가능해져, 의존성 있는 패키지를 다운로드 받을 수 있다. 
Yocto 내의 recipe를 찾아보면 `meta-openembedded/meta-oe/recipes-dbs/influxdb` 같은 경우도 이와 같은 방식으로 패키지를 설치하고 있다.

## 정리

NPM을 이용하는 NodeJS의 경우에는 패키지 설치에 대해서 yocto의 devtool 로 지원을 하고 있다. 

- [TipsAndTricks/NPM - Yocto Project](https://wiki.yoctoproject.org/wiki/TipsAndTricks/NPM)

점차 다른 언어들도 embedded system에서도 사용이 늘어나는 추세라 Yocto 에서도 일관된 관리 방식을 제공될 것으로 보인다. 
Yocto 프로젝트를 고려하면서 이들 사항에 대해서도 주기적으로 체크가 필요해 보인다.