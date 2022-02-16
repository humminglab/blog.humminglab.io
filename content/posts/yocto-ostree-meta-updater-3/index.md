---
title: "Yocto에 OSTree upgrade 적용(3) - 업그레이드/롤백 및 OSTree 리뷰"
date: "2022-02-16T22:00:00+09:00"
lastmod: "2022-02-16T22:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Yocto", "OSTree", "OTA", "Uptane", "U-Boot"]
categories: ["Development"]
---

이전글 [Yocto에 OSTree upgrade 적용(1)]({{< ref "posts/yocto-ostree-meta-updater-1" >}}) 에서 Yocto를 이용한 빌드 과정과,
[Yocto에 OSTree upgrade 적용(2)]({{< ref "posts/yocto-ostree-meta-updater-2" >}}) 에서 OSTree를 적용한 이미지의 부팅 과정에 대해서 설명하였다.

이번 글에서는 OSTree가 적용된 이미지를 실제로 업그레이드 하는 방법, 롤백 절차, 프로그램에서 이를 관리하는 방법에 대해서 설명한다.

이해를 돕고자 OSTree의 업그레이드 절차를 git과 비교하여 설명한다.
OSTree는 ostree CLI 명령을 이용하여 업그레이드 과정을 수행할 수 있고, libostree library 를 이용하여 프로그램으로 구현할 수 도 있다. 이 글에서는 CLI를 이용하는 업그레이드 방법을 설명한다.

### OSTree repository server

Yocto 빌드를 하면 ostree repository는 deploy 디렉토리에 ostree_repo 로 생성된다.

빌드된 deploy 디렉토리에서 `ostree log`를 실행해보면 지금까지 빌드된 log를 확인할 수 있다.

```shell
$ ostree --repo=ostree_repo refs
sc-gateway

$ ostree --repo=ostree_repo log sc-gateway
commit 931c8ec950226dc18613e3e32f6d4079356e74e543314ce34f07df93492e1f51
ContentChecksum:  a7bfe9ecec3dbc13014be7899706ac29a9db8bdab94d99c4291e0975156a4a84
Date:  1970-01-01 00:00:00 +0000
Version: 1.0

    Commit-id: sc-gateway-image-sc-gateway-20220211022907

commit 5d505a2b9ec10252e3db007f9d36551170eb5ab9f4961ca71377d96328565a6f
ContentChecksum:  c67060c2438059499c2c3941e46dd5c8104c94bc3666ea424037d1094f734272
Date:  1970-01-01 00:00:00 +0000
Version: 1.0

    Commit-id: sc-gateway-image-sc-gateway-20220210094454
```

이 디렉토리를 웹 서버로 엑세스 가능하게만 해주면, 기기에서 이를 이용하여 OTA를 수행할 수 있다.

시험을 위하여 웹 서버를 실행하는 간단한 방법 중 하나는 python을 이용하여 다음과 같이 실행하면 된다.

```shell
$ python -m SimpleHTTPServer 8081
```

상용으로 OTA 서버를 운영하는 경우에는 인증 매커니즘이 들어가야 한다.
이를 위해서 두가지 방법이 있는데, 첫번째는 ostree 에서 기본 제공하는 gpg 를 이용하여 정확한 서버의 데이타인 지를 클라이언트에서 검증하는 방법이고,
두번째는 웹 서버 연결 시 HTTPS 웹 인증을 수행하여 서버, 클라이언트(기기) 인증을 하는 방법이다.

일반적인 상용 서비스에서는 두번째와 같은 방식을 많이 사용한다. 여기에서는 시험 용도로 별도의 인증은 사용 하지 않는다.

### 기기에서 업그레이드 수행

초기 부팅을 한 상태에서는 타겟 내에는 빌드된 OSTree 의 최종 snapshot 만 가지고 있다.

`ostree log` 명령으로 로컬의 로그를 확인해 볼 수 있다 (`git log`와 유사). 타겟에서는 용량을 고려하여 모든 commit 정보를 가지고 있지 않고, 필요한 이력만 최소한으로 가지고 있다 (물론 필요하다면 로그 정보만 다 받아오는 것도 가능하다.)

```shell
$ ostree log poky:sc-gateway
commit 931c8ec950226dc18613e3e32f6d4079356e74e543314ce34f07df93492e1f51
ContentChecksum:  a7bfe9ecec3dbc13014be7899706ac29a9db8bdab94d99c4291e0975156a4a84
Date:  1970-01-01 00:00:00 +0000
Version: 1.0

    Commit-id: sc-gateway-image-sc-gateway-20220211022907
```

기기에서 처음 할 일은 `ostree remote add`로 repository를 등록한다(`git remote add`와 유사).
다음과 같이 실행하면 remote 서버를 'poky'로 추가하게 된다.

```shell
$ ostree remote add --no-gpg-verify poky http://192.168.9.3/repo/
```

`ostree pull` 명령으로 서버의 최신 commit을 가져온다(`git fetch`와 유사).
모든 이력을 가져오지 않고 최신 하나의 commit 정보만 가지고 온다. 뒤에 붙는 REFS의 'poky' 는 remote 이름이고, 'sc-gateway'는 git branch로 이해할 수 있다.

```shell
$ ostree pull poky:sc-gateway
7 metadata, 11 content objects fetched; 382 KiB transferred in 0 seconds; 861.9 kB content written
```

`ostree log` 명령으로 가져온 로그를 확인할 수 있다.

```shell
$ ostree log poky:sc-gateway
commit 7113d71eec68e79ab6765d86487babdf74707fdfcb5f03e68adb01342e6059f2
Parent:  b10ffbc6124b8cf30f171ad8aa54cdb4722595ca740cfa7985916d50573b7efd
ContentChecksum:  eaef166c55855134b79d9b6cce01da2503f9375b8380b982b12c71695694e2df
Date:  1970-01-01 00:00:00 +0000
Version: 1.0

    Commit-id: sc-gateway-image-sc-gateway-20220215064144
```

가져온 commit을 deploy 하기 전에 우선 `ostree admin status`를 이용하여 현재 deploy 된 정보를 확인할 수 있다.

```shell
$ ostree admin status
* poky 931c8ec950226dc18613e3e32f6d4079356e74e543314ce34f07df93492e1f51.0
    Version: 1.0
    poky refspec: poky:sc-gateway
```

이제 `ostree admin deploy`를 이용하여 받아온 commit을 deploy 한다(`git checkout`와 유사).

```shell
$ ostree admin deploy poky:sc-gateway
note: Deploying commit f9afc4650235fc6657e57d7ff9c4d56a1fcb0bb9f4f5ccaa11058e4cc871ea0c which contains content in /var/local that will be ignored.
Copying /etc changes: 1 modified, 0 removed, 4 added
Transaction complete; bootconfig swap: yes; bootversion: boot.0.1, deployment count change: 1
```

이와 같이 실행하면 deploy 디렉토리(/sysroot/ostree/deploy/poky/deploy)에 새로 받은 버전이 추가된다. `ostree admin status`로 확인해 보면 다음과 같다.

기존에 있던 931...f51 이외에 f9a...a0c 가 추가 되었고 뒤에 (pending)이라고 되어 있다. 앞부분에 '\*'가 있는 것이 현재 사용하고 있는 deploy이다.

```shell
$ ostree admin status
  poky f9afc4650235fc6657e57d7ff9c4d56a1fcb0bb9f4f5ccaa11058e4cc871ea0c.0 (pending)
    Version: 1.0
    origin refspec: poky:sc-gateway
* poky 931c8ec950226dc18613e3e32f6d4079356e74e543314ce34f07df93492e1f51.0
    Version: 1.0
    origin refspec: poky:sc-gateway
```

이 상태에서 시스템을 리부팅하면 새로운 버전으로 변경되어 실행 된다. 시스템을 리부팅 후 status 를 확인해 보면 다음과 같다.

기존 931...f51은 (rollback) 로 표시되고, f9a...a0c 가 active 된다. Rollback은 필요 시 rollback 하기 위한 용도라고 보면 된다.

```shell
$ ostree admin status
* poky f9afc4650235fc6657e57d7ff9c4d56a1fcb0bb9f4f5ccaa11058e4cc871ea0c.0
    Version: 1.0
    origin refspec: origin:sc-gateway
  poky 931c8ec950226dc18613e3e32f6d4079356e74e543314ce34f07df93492e1f51.0 (rollback)
    Version: 1.0
    origin refspec: origing:sc-gateway
```

이상과 같이 간단하게 업그레이드 과정을 진행할 수 있다.

그리고, `ostree pull` 과 `ostree admin deploy` 를 합치 `ostree admin upgrade` 명령이 있어서 이를 이용한 경우 위 절차를 하나로 줄일 수 있다(`git pull`과 유사).

```shell
$ ostree admin upgrade
6 metadata, 2 content objects fetched; 5 KiB transferred in 0 seconds; 14 bytes content written
note: Deploying commit 0e90b885b48e4142dae209f1964046c54da75c840de1aca02ff1f19e923ec64f which contains content in /var/local that will be ignored.
Copying /etc changes: 1 modified, 0 removed, 4 added
Transaction complete; bootconfig swap: no; bootversion: boot.0.0, deployment count change: 0
Freed objects: 19.1?kB
```

### 롤백 하기

만일 새로 받은 버전이 문제가 있는 경우 이전 버전으로 롤백을 할 수 있다.

위의 `ostree admin status` 에서 (rollback) 이 있는 상태에서, /sysroot/boot/loader/uEnv.txt 를 보면 다음과 같이 kernel_image 이외에 kernel_image2가 추가되어 있다. 이 2번 항목들이 rollback을 위한 것이다 (아래 에서는 kernel이 변경되지는 않아 동일한 것을 가리키고 있다).

```shell
$ cat /sysroot/boot/loader/uEnv.txt
kernel_image=/boot/ostree/poky-1f81ac983fb706e6034016a81ed3b5d727700c2284f8cc52f4b5323740293f7d/vmlinuz-5.4.69
ramdisk_image=/boot/ostree/poky-1f81ac983fb706e6034016a81ed3b5d727700c2284f8cc52f4b5323740293f7d/initramfs-5.4.69.img
bootargs=ostree=/ostree/boot.0/poky/1f81ac983fb706e6034016a81ed3b5d727700c2284f8cc52f4b5323740293f7d/0
kernel_image2=/boot/ostree/poky-1f81ac983fb706e6034016a81ed3b5d727700c2284f8cc52f4b5323740293f7d/vmlinuz-5.4.69
ramdisk_image2=/boot/ostree/poky-1f81ac983fb706e6034016a81ed3b5d727700c2284f8cc52f4b5323740293f7d/initramfs-5.4.69.img
bootargs2=ostree=/ostree/boot.0/poky/1f81ac983fb706e6034016a81ed3b5d727700c2284f8cc52f4b5323740293f7d/1
```

롤백 절차를 간단히 정리해 보면 다음과 같다.

- u-boot shell 에서 u-boot 변수 `rollback`을 1로 설정하고 부팅한다.

```shell
=> setenv rollback 1
=> run bootcmd
```

- `rollback`이 1인 경우 [Yocto에 OSTree upgrade 적용(2)]({{< ref "posts/yocto-ostree-meta-updater-2#target-%EC%97%90-yocto-%EC%A0%81%EC%9A%A9%ED%95%98%EA%B8%B0" >}}) 의 u-boot/boot.cmd 와 같이 `altbootcmd` 를 수행한다. 이 과정은 kernel_image2를 사용하여 부팅되도록 되어 있다.

- initramfs에서도 `rollback` 인 경우 rootfs를 이전 상태로 하여 실행한다.

이와 같이 rollback이 된 상태에서 보면 아래와 같이 deploy 된 직후의 상태로 돌아온다.

이 과정은 완전하게 rollback이 된 것은 아니고, 다시 리부팅을 하면 (peding) 되어 있던 것이 다시 active가 된다.

```shell
$ ostree admin status
  poky f9afc4650235fc6657e57d7ff9c4d56a1fcb0bb9f4f5ccaa11058e4cc871ea0c.0 (pending)
    Version: 1.0
    origin refspec: poky:sc-gateway
* poky 931c8ec950226dc18613e3e32f6d4079356e74e543314ce34f07df93492e1f51.0
    Version: 1.0
    origin refspec: poky:sc-gateway
```

롤백된 상태를 계속 유지하려면 다음과 같이 `ostree admin undeply`로 제거하여야 한다. 첫번째 (0번 인덱스)를 제거하면 rollback 과정이 완료된다.

```shell
$ ostree admin undeploy 0

$ ostree admin status
* poky 931c8ec950226dc18613e3e32f6d4079356e74e543314ce34f07df93492e1f51.0
    Version: 1.0
    origin refspec: poky:sc-gateway
```

## 자동 롤백 구현하기

실제 제품으로 사용하는 경우에는 위와 같은 수동 롤백이 아니라, 업그레이드 후 문제 발생 시 자동 롤백 기능이 필요하다.

[meta-updateer](https://github.com/advancedtelematic/meta-updater) 에서는 [Aktualizr](https://advancedtelematic.github.io/aktualizr/index.html) 라는 프로그램으로 전용 OTA 서버를 이용하는 기능이 구현되어 있다.

Aktualizr 를 사용하지 않고, 직접 구현하여 사용하는 경우에 필요한 처리 사항을 정리하면 다음과 같다.

U-Boot 에는 [bootcount](https://github.com/u-boot/u-boot/blob/master/doc/README.bootcount) 기능을 제공한다(자세한 방법은 이 링크의 설명을 참고).

관련된 환경변수는 다음과 같다.

- `bootcount`: 부팅 시 마다 1씩 자동으로 증가한다.
- `bootlimit`: `bootcount`가 `bootlimit` 값을 넘어서는 경우 `bootcmd`가 아닌 `altbootcmd` 를 실행한다. altbootcmd에서 rollback을 구현한다.
- `upgrade_available`: 만일 이 값이 0 이면 `bootcount`는 증가하지 않고, 1 인 경우에만 증가한다. 즉, upgrade 한 직후에만 1로 하여 문제 발생 시 rollback 될 수 있도록 할 수 있다.

U-Boot 의 이 기능만으로 운영은 하지 못하고, 정상적으로 부팅이 완료된 후 데몬과 같은 프로그램에서 추가적인 처리를 해주어야 한다.

이 데몬과 U-Boot는 다음과 같이 동작한다.

- 업그레이드 시
  - 데몬 프로그램
    - `ostree admin upgrade`와 같은 절차를 수행하여 새로운 deploy를 생성한다.
    - U-Boot [fw_setenv](https://github.com/u-boot/u-boot/tree/master/tools/env) 등을 이용하여 U-Boot 환경변수 `upgrade_available`를 1 로 설정하고, `bootcount` 는 0, `bootlimit`는 적절한 값으로 설정한다.
    - 리부팅
  - U-Boot
    - 부팅 시마다 `bootcount`를 1씩 증가한다.
    - linux 부팅 중 문제가 발생하는 경우 watchdog 등으로 인하여 리부팅 되면, `bootcount`는 1씩 증가한다.
    - 만일 `bootcount`가 `bootlimit`만큼 되면 `altbootcmd`가 실행되어 롤백 부팅이 된다.
  - 데몬 프로그램
    - 정상 부팅이 되면 `upgrade_available`을 0으로 설정하여 u-boot 에서 rollback 되지 않도록 한다.
    - 만일 rollback 부팅 되었다면 문제가 되는 deployment를 제거한다.

[Aktualizr](https://advancedtelematic.github.io/aktualizr/index.html) 의 기능 중 일부가 위와 같은 데몬 프로그램의 역할을 하는 것이다.

이와 유사한 기능은 libostree를 이용하여 프로그램으로 구현을 하거나, demon script로 구현을 하면 자동 롤백 기능을 구현할 수 있다.

## 마무리

OSTree를 사용하는 [Fedora CoreOS](https://getfedora.org/en/coreos) 는 Kubernetes와 같이 수많은 클러스터를 위한 Container 전용 OS 이다. OSTree 기반의 rpm-ostree를 이용하여 다수의 클러스터를 동일한 snapshot 으로 유지 관리를 할 수 있다.
위에서 언급한 [Aktualizr](https://advancedtelematic.github.io/aktualizr/index.html) 도 자동차의 Secure OTA를 위한 [Uptane Alliance](https://uptane.github.io)의 표준을 만족하는 OSTree 방식의 업그레이드 시스템이다.

OSTree는 이와 같이 수많은 기기들을 동일한 스냅샷으로 유지 관리할 필요가 있는 Clustering, 자동차, IoT OS 등의 upgrade 기능으로 장점을 가진다.

전체 파일 시스템을 업그레이드 하는 방식과 비교하여, 빠른 업그레이드 속도, overlayfs 방식의 임시수정, hotfix, mutlple delta 스위칭 등 다양한 기능을 활용할 수 있어 제품의 라이프 사이클이 길고, 지속적으로 기능이 향상되는 기기에서는 이의 적용을 검토해 보는 것도 좋을 것이다.
