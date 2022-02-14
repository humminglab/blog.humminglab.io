---
title: "Yocto에 OSTree upgrade 적용(2) - 부팅 절차"
date: "2022-02-14T09:00:00+09:00"
lastmod: "2022-02-14T09:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Yocto", "OSTree", "OTA"]
categories: ["Development"]
---

이전글 [Yocto에 OSTree upgrade 적용(1) - 이미지 생성]({{< ref "posts/yocto-ostree-meta-updater-1" >}}) 에서는 Yocto 빌드 과정을 통한 target 에 write 할 이미지를 만드는 과정까지 설명 하였다.

이번글에서는 부팅 이미지를 이용하여 부팅 절차를 설명한다.

## 디스크 이미지 파일

최종적으로 디스크에 쓰는 이미지를 Yocto 의 wic 툴을 이용하여 확인해 보면 다음과 같이 두개의 partition 으로 구성된다.

- Partiton 1(fat16): DOS FAT16 의 부팅 디스크로 u-boot 과 부팅에 필요한 설정 파일이 있다.
  - 파일 중 boot.scr 파일이 있는데, 이 파일로 u-boot 의 script를 대체하여 OSTree 이미지가 로드되도록 한다.
- Partition 2(ext4): OSTree용 이미지

```shell
$ wic ls  sc-gateway-image-sc-gateway.wic
Num     Start        End          Size      Fstype
 1       2097152     44040191     41943040  fat16
 2      44040192    132120575     88080384  ext4
```

이 중 EXT4 partition은 이전 글에 설명한 것과 같이 아래와 같은 디렉토리를 가진다.

```
.
├── boot
│   ├── boot -> .
│   ├── loader -> loader.1
│   ├── loader.1
│   └── ostree
│       └── poky-1f8...f7d
└── ostree
    ├── boot.1 -> boot.1.1
    ├── boot.1.1
    │   └── poky
    │       └── 1f8...f7d
    │           └── 0 -> ../../../deploy/poky/deploy/931...f51.0
    ├── deploy
    │   └── poky
    │       ├── deploy
    │       │   └── 931...f51.0
    │       │       ├── bin -> usr/bin
    │       │       ├── boot
    │       │       ├── dev
    │       │       ├── etc
    │       │       ├── home -> var/rootdirs/home
    │       │       ├── lib -> usr/lib
    │       │       ├── media -> var/rootdirs/media
    │       │       ├── mnt -> var/rootdirs/mnt
    │       │       ├── opt -> var/rootdirs/opt
    │       │       ├── ostree -> sysroot/ostree
    │       │       ├── proc
    │       │       ├── run
    │       │       ├── sbin -> usr/sbin
    │       │       ├── srv -> var/rootdirs/srv
    │       │       ├── sys
    │       │       ├── sysroot
    │       │       ├── tmp
    │       │       ├── usr
    │       │       └── var
    │       └── var
    │           ├── lib
    │           ├── local
    │           ├── log
    │           ├── rootdirs
    │           │   └── home
    │           │       └── root
    │           ├── sota
    │           │   └── import
    │           └── tmp
    └── repo
```

Bootloader에 관련된 파일들은 첫번째 FAT16 partition에 있고, 위의 디렉토리 구조 중 boot 디렉토리에는 kernel booting에 관련된 파일들이 있다.

특징적인 파일은 다음과 같다.

- /boot/loader/uEnv.txt: kernel 위치와 bootargs 옵션을 정의한다. 파일 내용은 다음과 같다.

```
kernel_image=/boot/ostree/poky-1f8...f7d/vmlinuz-5.4.69
ramdisk_image=/boot/ostree/poky-1f8...f7d/initramfs-5.4.69.img
bootargs=ostree=/ostree/boot.1/poky/1f8...f7d/0
```

- kernel, initramfs는 위 파일에 있는 /boot/ostree/poky-1f8...f7d/ 안에 있다.

## U-Boot 에서 커널 부팅 처리

U-Boot는 [Hush Shell](https://www.denx.de/wiki/DULG/CommandLineParsing) 이라는 shell script를 지원한다.
이를 이용하여 기본적인 부팅 정책이 설정되어 있는데, 보통은 다음과 같은 정책을 가진다.

- Block device의 bootable partition 에서 `boot.scr` 을 찾는다. 이 파일은 U-Boot의 mkimage를 이용하여 패킹한 U-Boot용 script 파일이다. 이 파일이 있으면 이를 로드 후 script 를 실행한다.
- 만일 `boot.scr`이 없으면 미리 지정된 kernel booting 절차를 수행한다.

Yocto OSTree 이미지에서는 이 boot.scr을 다음과 같이 설정하여 OSTree에 맞도록 부팅을 수행한다.

```
setenv loadaddr ${kernel_addr_r}
setenv bootcmd_resetvars 'setenv kernel_image; setenv bootargs; setenv kernel_image2; setenv bootargs2'
setenv bootcmd_otenv 'run bootcmd_resetvars; load mmc 0:2 $loadaddr /boot/loader/uEnv.txt; env import -t ${loadaddr} ${filesize}'
setenv bootcmd_rollbackenv 'setenv kernel_image ${kernel_image2}; setenv bootargs ${bootargs2}'

setenv bootcmd_args 'setenv bootargs "${bootargs} ${bootargs_fdt} ostree_root=/dev/mmcblk0p2 root=/dev/ram0 rw rootwait rootdelay=2 ramdisk_size=8192 panic=1"'

setenv bootcmd_getroot 'setexpr ostree_root gsub "^.*ostree=([^ ]*).*$" "\\\\1" "${bootargs}";'

setenv bootcmd_load 'load mmc 0:2 ${ramdisk_addr_r} "/boot"${kernel_image}'
setenv bootcmd_run 'bootm "${ramdisk_addr_r}"'

setenv bootcmd_create_envfile 'if test ! -e mmc 0:1 uboot.env; then saveenv; fi;'

setenv bootlimit 3

setenv bootcmd 'if test "${rollback}" = "1"; then run altbootcmd; else run bootcmd_create_envfile; run bootcmd_otenv; run bootcmd_args; run bootcmd_load; run bootcmd_run; if ! "${upgrade_available}" = "1"; then setenv upgrade_available 1; saveenv; fi; reset; fi'

setenv bootcmd_set_rollback 'if test ! "${rollback}" = "1"; then setenv rollback 1; setenv upgrade_available 0; saveenv; fi'
setenv altbootcmd 'run bootcmd_create_envfile; run bootcmd_otenv; run bootcmd_set_rollback; if test -n "${kernel_image2}"; then run bootcmd_rollbackenv; fi; run bootcmd_args; run bootcmd_load; run bootcmd_run; reset'

run bootcmd
```

위 script를 간단하게 정리하면 다음과 같다.

- EXT4 Partition의 /boot/loader/uEnv.txt 를 읽음. 이곳에는 위에 설명한 것과 같이 `kernel_image`, `ramdisk_image`, `bootargs` 값이 있음
  - ramdisk_image 변수는 U-Boot에서는 사용하지 않고, kernel_image만 사용한다. 여기에서 가리키는 kernel는 [U-Boot FIT Image](https://www.thegoodpenguin.co.uk/blog/u-boot-fit-image-overview/)로 kernel, DTB, initramfs 파일이 패킹되어 있다.
- 아래와 같이 bootargs 에 `ostree`, `ostreee_root` 를 설정하고 ramdisk를 rootfs로 initramfs 로 부팅한다.

```
setenv bootargs "${bootargs} ${bootargs_fdt} ostree_root=/dev/mmcblk0p2 root=/dev/ram0 rw rootwait rootdelay=2 ramdisk_size=8192 panic=1"'
```

## Initramfs의 동작

Initramfs로 부팅이 되면 init script에서는 다음과 같은 동작을 수행한다.

- 환경 변수로 다음과 같이 설정

  - `ostree_root`: /dev/mmcblk0p2 로 EXT4 partition
  - `ostree`: /ostree/boot.1/poky/1f8...f7d/0 로 OSTree image가 deploy된 디렉토리

- `ostreee_root` 를 /sysroot 디렉토리에 마운트
- /sysroot/`${ostree}`/usr를 read-only 가 되도록 bind remount 수행
- /sysroot/`${ostree}`/deploy/poky/var를 /sysroot/`${ostree}`/var 에 마운트 (/var 는 업그레이드에도 바뀌지 않도록 별도의 폴더를 사용)
- 초기 디스크 이미지를 참조할 수 있도록 /sysroot를 /sysroot/`${ostree}`/sysroot 로 bind mount
- /sysroot/`${ostree}` 를 /sysroot 로 mount (Deploy 된 OSTree 이미지)

이렇게 되면 /sysroot 는 `ostree`가 가리키는 /ostree/boot.1/poky/1f8...f7d/0 가 되고, 고정된 /var 도 이 안에 마운트 되고, 전체 EXT4 파티션도 symbolic link를 통해서 /sysroot/sysroot 로 된다.

이 상태에서 [switch_root](https://man7.org/linux/man-pages/man8/switch_root.8.html) 명령으로 /sysroot 를 루트로 전환하여 정상적인 init script가 실행되도록 한다.

이 과정을 거치면 정상적인 부팅이 된다.

디스크 내의 디렉토리가 최종적으로 어떻게 마운트 되었는지를 정리해보면 다음과 같다.

| 디스크이미지                    | 최종 실행된 시스템   | 설명                                        |
| :------------------------------ | :------------------- | :------------------------------------------ |
| /                               | /sysroot             | 전체 디스크 이미지                          |
| /ostree/deploy/poky/var         | /var                 | Upgrade 에도 영향을 받지 않는 /var 디렉토리 |
| /ostree/boot.1/poky/1f8...f7d/0 | /                    | Deploy된 OSTree 이미지                      |
| /ostree/repo                    | /sysroot/ostree/repo | OSTree local repository                     |

bind mount 관계를 shell 로 확인해 보면 다음과 같다.

```shell
$ cat /proc/self/mountinfo
21 22 179:2 / /sysroot rw,relatime - ext4 /dev/mmcblk0p2 rw
22  1 179:2 /ostree/deploy/poky/deploy/931...f51.0 / rw,relatime - ext4 /dev/mmcblk0p2 rw
23 22 179:2 /ostree/deploy/poky/var /var rw,relatime - ext4 /dev/mmcblk0p2 rw
24 22 179:2 /boot /boot rw,relatime - ext4 /dev/mmcblk0p2 rw
25 22 179:2 /ostree/deploy/poky/deploy/931...f51.0/usr /usr ro,relatime - ext4 /dev/mmcblk0p2 rw
26 22 0:5   / /proc rw,relatime - proc proc rw
27 22 0:14  / /sys rw,relatime - sysfs sysfs rw
28 22 0:6   / /dev rw,relatime - devtmpfs devtmpfs rw,size=123408k,nr_inodes=30852,mode=755
29 28 0:18  / /dev/pts rw,relatime - devpts devpts rw,gid=5,mode=620,ptmxmode=666
30 22 0:19  / /run rw,nosuid,nodev - tmpfs tmpfs rw,mode=755
31 23 0:20  / /var/volatile rw,relatime - tmpfs tmpfs rw
```

참고로 /etc 를 생성한 소스는 /usr/etc 에 있다. 이를 3-way merge를 통하여 OTA 시 신규 /etc를 만든다.

## Target 에 Yocto 적용하기

meta-updater layer에서 Raspberry Pi 와 같이 기존 제공하는 보드가 아닌, 개발하고 있는 제품에 적용하기 위한 절차를 정리하면 다음과 같다.

- layer.conf 에 meta-updater(sota) 의존성 추가

```
LAYERDEPENDS_sc-gateway += "sota"
```

- bblayers.conf 에 `BBLAYERS` 에 meta-updater 추가
- local.conf 에 `DISTRO` 를 poky에서 poky-sota로 변경

```
DISTRO ?= "poky-sota"
```

- machine conf (conf/machine/sc-gateway.conf)에 `SOTA_MACHINE` 추가

```
SOTA_MACHINE:sc-gateway = "sc-gateway"
```

- classes/sota_sc-gateway.bbclass 와 같이 `SOTA_MACHINE` 에 맞는 bbclass 추가 및 적절히 수정. WKS_FILES 로 wks script 변경

```
KERNEL_CLASSES:append:sota = " kernel-fitimage"
KERNEL_IMAGETYPE:sota = "fitImage"
INITRAMFS_FSTYPES = "cpio.gz"
OSTREE_KERNEL = "${KERNEL_IMAGETYPE}-${INITRAMFS_IMAGE}-${MACHINE}-${KERNEL_FIT_LINK_NAME}"

OSTREE_BOOTLOADER ?= "u-boot"

WKS_FILES:sc-gateway ?= "sc-gateway-sdcard-image.wks.in"
```

- wic/sc-gateway-sdcard-image.wks.in 으로 root parition을 otaimge로 변경

```
part u-boot --source rawcopy --sourceparams="file=${SPL_BINARY}" --ondisk mmcblk0 --no-table --align 8
part /boot --source bootimg-partition --ondisk mmcblk0 --fstype=vfat --label boot --active --align 2048 --fixed-size ${SUNXI_BOOT_SPACE}
part /     --source otaimage --ondisk mmcblk0 --fstype=ext4 --align 2048
```

- u-boot 을 변경하여 boot.scr을 대체

  - u-boot\_%.bbappend

  ```
  FILESEXTRAPATHS:prepend:sc-gateway := "${THISDIR}/files:"
  ```

  - u-boot/boot.cmd

  ```
  setenv loadaddr ${kernel_addr_r}
  setenv bootcmd_resetvars 'setenv kernel_image; setenv bootargs; setenv kernel_image2; setenv bootargs2'
  setenv bootcmd_otenv 'run bootcmd_resetvars; load mmc 0:2 $loadaddr /boot/loader/uEnv.txt; env import -t ${loadaddr} ${filesize}'
  setenv bootcmd_rollbackenv 'setenv kernel_image ${kernel_image2}; setenv bootargs ${bootargs2}'

  setenv bootcmd_args 'setenv bootargs "${bootargs} ${bootargs_fdt} ostree_root=/dev/mmcblk0p2 root=/dev/ram0 rw rootwait rootdelay=2 ramdisk_size=8192 panic=1"'

  setenv bootcmd_getroot 'setexpr ostree_root gsub "^.*ostree=([^ ]*).*$" "\\\\1" "${bootargs}";'

  setenv bootcmd_load 'load mmc 0:2 ${ramdisk_addr_r} "/boot"${kernel_image}'
  setenv bootcmd_run 'bootm "${ramdisk_addr_r}"'

  setenv bootcmd_create_envfile 'if test ! -e mmc 0:1 uboot.env; then saveenv; fi;'

  setenv bootlimit 3

  setenv bootcmd 'if test "${rollback}" = "1"; then run altbootcmd; else run bootcmd_create_envfile; run bootcmd_otenv; run bootcmd_args; run bootcmd_load; run bootcmd_run; if ! "${upgrade_available}" = "1"; then setenv upgrade_available 1; saveenv; fi; reset; fi'

  setenv bootcmd_set_rollback 'if test ! "${rollback}" = "1"; then setenv rollback 1; setenv upgrade_available 0; saveenv; fi'
  setenv altbootcmd 'run bootcmd_create_envfile; run bootcmd_otenv; run bootcmd_set_rollback; if test -n "${kernel_image2}"; then run bootcmd_rollbackenv; fi; run bootcmd_args; run bootcmd_load; run bootcmd_run; reset'

  run bootcmd
  ```

위와 같이 변경한 후 빌드를 하면 OSTRee가 적용된 이미지를 생성할 수 있다.

## 정리

이번 과정은 OSTree 의 부팅과정과 이를 다른 타겟에 적용하는 방법을 알아보았다.
다음으로는 ostree를 이용하여 업그레이드, 롤백하는 과정을 알아보기로 한다.
