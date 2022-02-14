---
title: "Yocto에 OSTree upgrade 적용(1) - 이미지 생성"
date: "2022-01-26T21:00:00+09:00"
lastmod: "2022-02-14T09:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Yocto", "OSTree", "OTA"]
categories: ["Development"]
---

Linux PC 의 경우 각 배포본 마다 yum, rpm, dpkg 등의 package manager를 제공하여, 이를 이용하여 패키지를 최신 버전으로 유지 관리할 수 있다.
임베디드의 경우도 [Raspberry Pi OS](https://www.raspberrypi.com/software/), [armbian](https://www.armbian.com/) 은 PC 에서 사용하는 package manager 방식을 제공하고, Yocto 도 rpm 등을 이용하여 패키지 관리가 가능하다.

이들 패키지 매니저는 패키지 데이타베이스를 업데이트 하고, 패키지 업그레이드 시 의존성 있는 추가 패키지도 다운로드 받아서 설치/삭제하고, 설치 전/후처리를 위한 script를 자동으로 실행시켜서 최종 상태를 만들어 준다. 하지만 임베디드 제품의 경우 이같은 방식은 다음과 같은 유지 관리 문제를 가져 올 수 있다.

- 언제든지 전원이 꺼질 수 있는 기기 특성 상, 기존 패키지 매니저를 이용한 업그레이드 도중 전원 차단 시 시스템 손상 가능성
- 업그레이드 시점에 따라 기기마다 다른 패키지 버전 설치로 고객 센터에서 장애 대응 어려움
- 자체 검증이 안된 외부 공개 패키지 업그레이드 시 장애 발생 가능성
- 모니터, 키보드 등의 사용자 인터페이스가 없는 제품의 경우 장애 발생 시 장애 원인 확인 및 수동 복구 어려움

이와 같은 단점으로 많은 임베디드 기기의 경우 보통 루트 파일 시스템 전체를 업그레이드 하는 방법을 취한다. 다만 이와 같은 업그레이드 방식은 작은 수정 시에도 전체 이미지를 다운로드 설치하여야 하여 비용 및 시간이 소요되는 불편한 점이 있다.

여기에서는 다른 대안으로 [OSTree](https://ostreedev.github.io/ostree/)를 이용한 OTA (Over the Air) Firmrware Update 하는 방법에 대한 이해와 Yocto project 를 이용한 구축 예를 설명한다.

이 방식은 간단하게 말하면 git 과 같은 delta 업데이트 방식으로 전체 이미지가 아닌 변경된 파일만 다운로드 받는 방식이다.
Connected Car 솔루션과 같이 지속적으로 성능 개선 및 유지 보수가 필요한 경우, 이와 같은 OTA 방식을 적용한다면 더욱 빠르게 업그레이드가 가능하고, rollback 등의 관리도 유연하게 대응할 수 있게 된다.

## Git 을 이용한 업그레이드

우리가 많이 사용하는 [git](https://git-scm.com/) 버전 관리 시스템은 [Pro Git 기초](https://git-scm.com/book/ko/v2/%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0-Git-%EA%B8%B0%EC%B4%88)에 있는 그림과 같이 그 기반은 분산형 파일 버전 관리 파일 시스템이라고 볼 수가 있다.

{{< figure src="https://git-scm.com/book/en/v2/images/snapshots.png" width="600px" height="auto" caption="git version">}}

위 그림처럼 repository에는 각 버전별로 snapshot 파일들을 관리한다. `git-checkout`은 원하는 snapshot 을 작업 디레토리로 가져오는 과정이라고 볼 수 있다.

git 에서 repository에 snapshot 정보를 관리하는 방법은 [Pro Git 10.2 Git의 내부](https://git-scm.com/book/ko/v2/Git%EC%9D%98-%EB%82%B4%EB%B6%80-Git-%EA%B0%9C%EC%B2%B4)를 보면 좀 더 자세히 알 수 있다.

간단하게 요약하면, tree(디렉토리 정보), blob(파일 내용), commit(commit 내용), tag(태그 정보) 등의 각 개별정보를 각각의 파일로 만들어서 아래 그림처럼 snapshot 정보를 repository에 관리하는 것이다.

{{< figure src="https://git-scm.com/book/en/v2/images/data-model-3.png" width="600px" height="auto" caption="Git 저장소 내의 모든 개체">}}

이를 업그레이드에 활용하면 어떻게 될까?

파일의 내용을 모두 git 에 추가한 다음 기기에서는 대략적인 과정으로는 아래처럼 운용을 하면 될 것이다.

```shell
# 현재 production-v1 branch 를 v1 디렉토리에 설치하기
$ git clone -n https://github.com/xxxxxxxxxxx
$ git fetch
$ mkdir v1
$ git --work-tree=v1 origin/production-v1 -- .
$ ln -s v1 current

# production-v2 branch를 v2 디렉토리에 설치하기
$ mkdir v2
$ git fetch
$ git --work-tree=v2 origin/production-v2 -- .

# 신규 FW로 교체하기
$ ln -sf v2 current
```

- 1: git clone은 하는데 checkout은 하지 않음.
- 4~5: v1 디렉토리로 prodution-v1을 checkout
- 6: current를 v1으로 심볼릭 링크
- 9~14: 신규 v2로 업그레이드 과정. 서버에서 fetch로 받아오고, 이를 v2에 설치하고, current로 링크

문제는 루트파일 시스템을 이와같이 운용하기에는 다음과 같은 관점에서 비효율적이다.

- checkout 과정이 파일 복사로 위 경우 동일파일이 git repository, v1, v2 디렉토리에 3개 유지
- Git repository에 모든 이력을 가지고 있어야 함(Shallow clone 이라도 v1 ~ v2 이력은 가지고 있어야 함)

## OSTree Overview

Git 과 유사한 방식으로 루트 파일시스템의 업그레이드 기능을 만든 것이 OSTree 라고 할 수 있다.

[OSTree(libostree)](https://github.com/ostreedev/ostree)는 GNU license 오픈소스프로젝트로 [Fedora CoreOS](https://getfedora.org/en/coreos) 등에서 사용하고 있다. OSTree(libostree)는 이름처럼 library 형태와 ostree CLI 두가지 모두 제공한다. 일반적인 용도에서는 ostree CLI 를 이용하여 OTA 기능을 구현한다.

OSTree가 git 과의 가장 큰 차이점은 deploy(checkout) 시 repository 내의 파일을 hard link 를 통한 복제를 하고, repository 내의 정보도 현재 버전과 새로운 버전을 위한 정보만 local 에 유지하여 전체적인 저장공간 사용량을 최소화 한다.

## OSTree CLI

OSTree CLI는 git과 유사하게 subcommand를 지원하는데, 이 명령은 크게 admin 명령과 다른 명령들로 구분된다.

```shell
$  ostree

Builtin Commands:
  admin             Commands for managing a host system booted with ostree
  cat               Concatenate contents of files
  checkout          Check out a commit into a filesystem tree
  checksum          Checksum a file or directory
  commit            Commit a new revision
  config            Change repo configuration settings
  diff              Compare directory TARGETDIR against revision REV
  export            Stream COMMIT to stdout in tar format
  find-remotes      Find remotes to serve the given refs
  create-usb        Copy the refs to a USB stick
  fsck              Check the repository for consistency
  gpg-sign          Sign a commit
  init              Initialize a new empty repository
  log               Show log starting at commit or ref
  ls                List file paths
  prune             Search for unreachable objects
  pull-local        Copy data from SRC_REPO
  pull              Download data from remote repository
  refs              List refs
  remote            Remote commands that may involve internet access
  reset             Reset a REF to a previous COMMIT
  rev-parse         Output the target of a rev
  show              Output a metadata object
  static-delta      Static delta related commands
  summary           Manage summary metadata
```

- admin 이외의 명령들: `--repo` 로 지정한 repository 에 대해서 git 명령들과 유사한 작업을 수행한다. 즉 repository를 관리하는 명령이라고 보면 된다.
- admin 명령: admin은 실제 루트 파일시스템에서 업그레이드를 수행할 때 사용하는 명령이다. Target 파일 시스템에는 `/ostree/repo` 로 repository 디렉토리가 지정되어 있고, 이 명령 수행 시에는 `--sysroot` 로 실제 루트 위치를 지정하여 사용한다.

## OSTree 운영 방법 {#ostree-operating-method}

타겟에서 실행 시 ROOTFS는 read only로 마운트 하고, 업그레이드 시 ROOTFS snapshot을 교체하는 방식이다.
기기 설정과 동작 중 생성되는 파일을 위하여 `/etc`와 `/var`는 read/write 로 생성되고, 다음과 같이 별도로 관리한다.

- `/var`: 처음 생성한 이후로는 OSTree에서는 관여하지 않는다. 기기에서 생성되는 파일들을 저장하고, 업그레이드 시에도 이 디렉토리는 유지된다.
- `/etc`: 초기에 read/write 영역에 별도로 복사하여 사용하고, 동작 중 수정 될 수도 이다. 업그레이드 시에는 3-way merge 를 수행하여 업그레이드 되도록 한다.

OSTree 로 관리가 가능한 파일 시스템을 만들어야 하는데 이와 같은 절차는 다음과 같다.

1. 기존에 생성된 루트 파일 시스템을 OSTree 정책에 맞도록 정리한다. 예를 들어 `/mnt` 디렉토리는 삭제하고, `/var/rootdirs/mnt` 로 symlink를 생성해둔다 (부팅 시 ramfs 등을 이용하여 마운트하여 사용).
1. 정리된 rootfs 디렉토리를 ostree CLI를 이용하여 원격 repository에 push 한다.
1. ostree CLI의 admin 명령을 이용하여 target 에 write 할 rootfs 이미지를 만든다. 이 이미지는 ostree repository와 1)의 과정에서 만든 snapshot을 deploy 디렉토리에 생생한다.
1. initramfs를 이용하여 deploy 디렉토리가 root로 마운트 되도록 [bind mount](https://www.baeldung.com/linux/bind-mounts)하여 적절히 구성을 하고 [switch_root](https://man7.org/linux/man-pages/man8/switch_root.8.html)로 root 를 전환하고 정상 부팅과정을 진행한다.

위와 같은 과정을 OTRree가 관여 하다보니 다른 OTA 방식과 비교하여 조금은 복잡한 편이다.

## OSTree On Yocto

[Yocto Project](https://www.yoctoproject.org/)에 OSTree를 적용하는 절차는 recipe 에 따라 빌드, 생성해 주기 때문에 간단하지만, OSTree의 운영 방법에 대한 이해가 없다면 실제로 어떻게 관리되는지를 알수가 없게 된다.
또한 Yocto 빌드 과정에서 생성된 OSTree repository를 실수로 지우기라도 한다면 OTA snapshot 을 복구하는 험난한 과정을 거쳐야 한다. 그러므로 정확한 동작 방식을 이해한 후에 사용하여야만 된다.

Yocto에서 OSTree를 적용하기 위하여는 잘 관리되고 있는 [meta-updater](https://github.com/advancedtelematic/meta-updater) layer를 사용하면 된다.

[기본으로 제공하는 board](https://docs.ota.here.com/ota-client/latest/supported-boards.html)는 몇 가지가 있는데, 이 중 Raspberry Pi 3를 사용하는 경우를 예로 들어 설명한다.

Raspberry Pi 로 빌드하는 경우는 아래 링크를 참고하면 된다.

- [Build a Raspberry Pi image in OTA Connect Developer Guide](https://docs.ota.here.com/ota-client/latest/build-raspberry.html)

### Build 절차 요약

위 링크에 나온 것과 같이 repo 를 이용하여 이미지를 받아서 빌드 할 수 있다. 지원되는 Yocto 버전 중 가장 최신인 [Yocto 3.1 (Dunfell)](https://wiki.yoctoproject.org/wiki/Releases) 을 사용하였다.

Processor/Board 업체에서는 여러 layer를 포함하여 repository를 편하게 배포하려고 이와 같이 [repo](https://source.android.com/setup/develop/repo?hl=ko) 를 이용하는 경우가 많다.

```shell
$ mkdir myproject
$ cd myproject
$ repo init -u https://github.com/advancedtelematic/updater-repo.git -m dunfell.xml
$ repo sync

$ source meta-updater/scripts/envsetup.sh raspberrypi3
$ bitbake core-image-minimal
```

위와 같이 빌드를 마치면 build/tmp/deploy/images/raspberrypi3/ 에는 다음과 같은 파일이 생성된다.

- `core-image-minimal-raspberrypi3.ext3`: OSTree 적용 전의 rootfs
- `core-image-minimal-raspberrypi3.ostree`: [OSTree 운영 방법]({{< ref "#ostree-operating-method" >}})의 1) 과정. core-image-minimal-raspberrypi3.ext3 를 OSTree 운영 정책대로 수정 (정확히는 해당파일은 더미이고, core-image-minimal work 디렉토리에서 찾을 수 있음)
- `ostree_repo`: OSTree repository. [OSTree 운영 방법]({{< ref "#ostree-operating-method" >}})의 2) 과정. core-image-minimal-raspberrypi3.ostree 를 commit 하여 만듬. 이를 외부 HTTP server 로 운영하여 업그레이드 시 다운로드 받을 수 있도록 한다.
- `core-image-minimal-raspberrypi3.ota-ext4`: [OSTree 운영 방법]({{< ref "#ostree-operating-method" >}})의 3) 과정. `ostree_repo`의 최종 snapshot을 이용하여 deploy 구조로 만듬
- `initramfs-ostree-image-raspberrypi3.cpio.gz`: 부팅시 사용하는 initramfs. 다음과 같은 두개의 모듈을 실행
  - [ostree-prepare-root](https://github.com/ostreedev/ostree/tree/main/src/switchroot): 링크에 있는 switchroot.sh 파일의 절차와 같이 bind, move mount를 이용하여 root 디렉토리 구성
  - [ostree-initrd](https://github.com/advancedtelematic/meta-updater/tree/master/recipes-sota/ostree-initrd): tmpfs 등을 적절히 mount 하고 switch_root를 이용하여 root 디렉토리를 변경
- `fitImage`: u-boot, kernel, initramfs 이미지를 합쳐서 생성한 파일
- `core-image-minimal-raspberrypi3.wic`: 위 전체를 다 포함해서 만든 SD Card 에 쓰기 위한 이미지 파일

위 파일 중 wic 파일을 dd 등을 이용하여 SD Card에 쓴 후 부팅하면 된다.

### image_types_ostree.bbclass

위 파일 중 `core-image-minimal-raspberrypi3.ostree` 를 만드는 절차로 기존의 rootfs 을 다음과 같이 수정한다.

- `var/local` 을 제외한 `var` 내의 디렉토리 삭제
- `syslroot/`, `usr/rootdirs` 생성
- `home/`, `opt/`, `mnt/`, `media/`, `srv/`, `root/`, `usr/local` 디렉토리 삭제하고 `var/rootdirs`로 symlink 생성
  - init script에 부팅 시 `/var` ramdisk 에 디렉토리가 생성되도록 설정 추가
- `etc`를 `usr/etc`로 이동

위와 같이 하면 기본적인 OSTree 의 read only rootfs를 구성한다.

### image_types_ota.bbclass

image_types_ostree 로 생성되는 rootfs는 initramfs에서 최종적으로 bind mount로 이리저리 옮겨서 실제로 실행되는 rootfs 형태이고, image_types_ota 는 local repository와 이를 deploy한 디렉토리로 구성된, 실제로 디스크에 쓰는 파일 시스템을 생성한다. 이를 이용하여 생성한 파일은 `core-image-minimal-raspberrypi3.ota-ext4` 가 된다.

다음과 같은 절차로 만들어 간다.

- ostree admin init-fs 서브 명령으로 파일시스템의 기본 틀을 생성한다.

```shell
$ export OTA_SYSROOT="ota-sysroot"
$ export OSTREE_OSNAME="poky"
$ export OSTREE_BRANCHNAME="raspberrypi3"
$ export OSTREE_REPO="../../../../deploy/images/raspberrypi3/ostree_repo"
$ mkdir ${OTA_SYSROOT}
$ ostree admin --sysroot=${OTA_SYSROOT} init-fs --modern ${OTA_SYSROOT}
```

이를 거치면 다음과 같은 타겟용 rootfs 디렉토리 구조가 생성된다.

```shell
.
├── boot
└── ostree
    ├── deploy
    └── repo
        ├── config
        ├── extensions
        ├── objects
        ├── refs
        │   ├── heads
        │   ├── mirrors
        │   └── remotes
        ├── state
        └── tmp
            └── cache
```

- ostree admin os-init 로 deploy 용 기본 구조 생성.
  - ostree/deploy/${OSTREE_OSNAME} 으로 deploy 디렉토리 생성 및 초기 /var 구조 생성

```shell
$ ostree admin --sysroot=${OTA_SYSROOT} os-init ${OSTREE_OSNAME}
```

```shell
.
├── boot
└── ostree
    ├── deploy
    │   └── poky
    │       └── var
    │           ├── lib
    │           ├── lock -> ../run/lock
    │           ├── log
    │           ├── run -> ../run
    │           └── tmp
    └── repo
        ├── config
        ├── extensions
        ├── objects
        ├── refs
        │   ├── heads
        │   ├── mirrors
        │   └── remotes
        ├── state
        └── tmp
            └── cache
```

- ostree pull-local 로 REPO 를 복사하기 filesystem 안으로 복사하고 reference 생성

```shell
$ export ostree_target_hash=`ostree --repo=${OSTREE_REPO} rev-parse ${OSTREE_BRANCHNAME}`
$ ostree --repo=${OTA_SYSROOT}/ostree/repo pull-local --remote=${OSTREE_OSNAME} ${OSTREE_REPO} ${ostree_target_hash}
$ ostree --repo=${OTA_SYSROOT}/ostree/repo refs --create=${OSTREE_OSNAME}:${OSTREE_BRANCHNAME} ${ostree_target_hash}
```

repo 디렉토리 안에 repository가 복제되고, refs/remotes 에 reference가 생성된다.

```shell
.
├── boot
└── ostree
    ├── deploy
    │   └── poky
    │       └── var
    │           ├── lib
    │           ├── lock -> ../run/lock
    │           ├── log
    │           ├── run -> ../run
    │           └── tmp
    └── repo
        ├── config
        ├── extensions
        ├── objects
        │   ├── 00
        │   │   ├── 013...164.file
        │   ...
        │   ├── fd
        │   ...
        │       └── f84...237.file
        ├── refs
        │   ├── heads
        │   ├── mirrors
        │   └── remotes
        │       └── poky
        │           └── raspberrypi3
        ├── state
        └── tmp
            └── cache
```

- ostree admin deploy 로 deploy 수행

```shell
$ mkdir -p ${OTA_SYSROOT}/boot/loader.0
$ ln -s loader.0 ${OTA_SYSROOT}/boot/loader
$ touch ${OTA_SYSROOT}/boot/loader/uEnv.txt

$ export kargs_list="--karg-append=kernel_option1"
$ ostree admin --sysroot=${OTA_SYSROOT} deploy ${kargs_list} --os=${OSTREE_OSNAME} ${OSTREE_OSNAME}:${OSTREE_BRANCHNAME}
```

deploy 를 실행하면 ostre/deploy/${OSTREE_OSNAME}/deploy/ 로 reference로 deploy 되고, boot 디렉토리에는 kernel 파일이 복사된다.

```shell
.
├── boot
│   ├── loader -> loader.1
│   ├── loader.1
│   │   ├── entries
│   │   │   └── ostree-1-poky.conf
│   │   └── uEnv.txt
│   └── ostree
│       └── poky-925...009
│           ├── initramfs-4.19.126-v7.img
│           └── vmlinuz-4.19.126-v7
└── ostree
    ├── boot.1 -> boot.1.1
    ├── boot.1.1
    │   └── poky
    │       └── 925...009
    │           └── 0 -> ../../../deploy/poky/deploy/4cb...4fd.0
    ├── deploy
    │   └── poky
    │       ├── deploy
    │       │   ├── 4cb...4fd.0
    │       │   │   ├── bin -> usr/bin
    │       │   │   ├── boot
    │       │   │   ├── dev
    ...
    │       │   │   └── var
    │       │   │       └── local
    │       │   └── 4cb...4fd.0.origin
    │       └── var
    │           ├── lib
    │           ├── lock -> ../run/lock
    │           ├── log
    │           ├── run -> ../run
    │           └── tmp
    └── repo
        ├── config
        ├── extensions
        ├── objects
        │   ├── 00
        │   │   ├── 013...164.file
        │   ...
        │   ├── fd
        │   ...
        │       └── f84...237.file
        ├── refs
        │   ├── heads
        │   │   └── ostree
        │   │       └── 1
        │   │           └── 1
        │   │               └── 0
        │   ├── mirrors
        │   └── remotes
        │       └── poky
        │           └── raspberrypi3
        ├── state
        └── tmp
            └── cache
```

deploy 전에 boot/loader/uEnv.txt 를 임시로 만들었는데, 이렇게 uEnv.txt 파일이 있으면 deploy 시 uEnv.txt 파일도 자동으로 업데이트 하여 u-boot 에서 이를 참조 할 수 있도록 한다.

boot/loader/uEnv.txt 를 보면 다음과 같이 생성된다. deploy시 추가한 kernel argument 도 같이 추가되어 있다.
u-boot 에서는 이를 읽어들여 kernel, ramdisk 이미지를 로드하고, 적절한 kernel arguments를 추가하면 된다.

```
kernel_image=/ostree/poky-925...009/vmlinuz-4.19.126-v7
ramdisk_image=/ostree/poky-925...009/initramfs-4.19.126-v7.img
bootargs=kernel_option1 ostree=/ostree/boot.1/poky/925...009/0
```

- 추가로 ${OTA_SYSROOT}/ostree/deploy/poky/var 디렉토리의 초기 구성을 잡아준다.

## 정리

이상으로 OSTree 를 이용한 업그레이드 시스템 구성 시 파일 시스템이 어떻게 구성되는지를 정리하였다.

다음에는 생성된 Yocto Image 로 실행중인 장치에서 OSTree를 이용하여 업그레이드, rollback을 하는 방법을 정리하기로 한다.
