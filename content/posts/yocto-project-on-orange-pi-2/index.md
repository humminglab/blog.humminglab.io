---
title: "Yocto Project 개발하기(2) - Custom Layer 만들기"
date: "2022-01-07T19:00:00+09:00"
lastmod: "2022-01-07T19:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Yocto", "Linux", "OrangePi"]
categories: ["Development"]
series: ["Yocto Project 개발하기"]
---

[이전 글]({{< ref "posts/yocto-project-on-orage-pi-1">}}) 에서 meta-sunxi 를 추가하여 orage pi 용으로 빌드를 만들었고, 이번 과정은 project 용으로 meta layer를 만들어서 관리하는 방법을 설명한다.

실제 개발 과정을 이해하기 좋도록 meta layer 를 만들어 가는 과정을 설명한다.

- [Yocto Project 개발하기(1) - Orange Pi 보드 빌드]({{< ref "posts/yocto-project-on-orage-pi-1">}})
- **Yocto Project 개발하기(2) - Custom Layer 만들기**
- Yocto Project 개발하기(3) - 개발 시 로컬 패키지 관리하기
- Yocto Project 개발하기(4) - Yocto SDK 빌드
- Yocto Project 개발하기(5) - Yocto eSDK 이용한 개발 모델

## Layer 및 machine 생성

프로젝트를 sc-gateway 라고 명하고(이름에 특별한 의미는 없음), 이를 layer로 만든다. 다음과 같이 poky 폴더 안에 meta-sc-gateway를 생성한다.

```shell
$ mkdir -p meta-sc-gateway/conf/machine
```

Layer config 파일인 meta-sc-gateway/conf/layer.conf 를 다른 layer의 conf 를 참고하여 다음과 같이 만든다.

```
# We have a conf and classes directory, add to BBPATH
BBPATH .= ":${LAYERDIR}"

# We have recipes-* directories, add to BBFILES
BBFILES += "${LAYERDIR}/recipes-*/*/*.bb \
	${LAYERDIR}/recipes-*/*/*.bbappend"

BBFILE_COLLECTIONS += "sc-gateway"
BBFILE_PATTERN_sc-gateway = "^${LAYERDIR}/"
BBFILE_PRIORITY_sc-gateway = "10"

LAYERDEPENDS_sc-gateway += "meta-sunxi"

LAYERSERIES_COMPAT_sc-gateway = "honister"
```

- 10라인: layer의 priority를 10으로 높게 잡아 bb 파일 검색 시 우선 순위를 가지도록 함
- 12라인: meta-sunxi layer를 의존성에 추가
- 14라인: Yocto 호환 버전을 기록

meta-sunxi/conf/machine/orange-pi-zero.conf 와 같이 기존 machine을 그대로 이용해도 되지만, 프로젝트에 맞게 machine 이름도 sc-gateway 로 바꾸어 준다.

orange-pi-zero.conf를 meta-sc-gatway/conf/machine/sc-gateway.conf 로 복사한다. 아직은 kernel 설정이나 다른 것을 바꾸지 않았으므로 그대로 이용한다.

최종 이미지 제작용 recipe를 core-image-minimal.bb 를 참고하여 meta-sc-gateway/recipes-core/images/sc-gateway-image.bb 로 다음과 같이 만든다. SUMMARY만 변경하였고 내용은 동일하다.

```
SUMMARY = "SC Gateway Image"

IMAGE_INSTALL = "packagegroup-core-boot \
    ${CORE_IMAGE_EXTRA_INSTALL} \
    "

IMAGE_LINGUAS = " "

LICENSE = "MIT"

inherit core-image

IMAGE_ROOTFS_SIZE ?= "8192"
IMAGE_ROOTFS_EXTRA_SPACE:append = "${@bb.utils.contains("DISTRO_FEATURES", "systemd", " + 4096", "", d)}"
```

layer 가 적용되도록 다음과 같이 수정 한다.

build/conf/bblayer.conf 에 다음과 같이 meta-sc-gateway 를 추가한다.

```
BBLAYERS ?= " \
  /home/yslee/project/telesign/school-charger/poky/meta \
  /home/yslee/project/telesign/school-charger/poky/meta-poky \
  /home/yslee/project/telesign/school-charger/poky/meta-yocto-bsp \
  /home/yslee/project/telesign/school-charger/poky/meta-openembedded/meta-oe \
  /home/yslee/project/telesign/school-charger/poky/meta-sunxi \
  /home/yslee/project/telesign/school-charger/poky/meta-sc-gateway \
  "
```

build/conf/local.conf 의 `MACHINE` 도 sc-gateway 로 바꾼다.

```
MACHINE ??= "sc-gateway"
```

이제 shell에서 sc-gateway-image를 빌드한다.

```shell
$ bitbake sc-gateway-image
```

최종 이미지는 build/tmp/deploy/images/sc-gateway/sc-gateway-image-sc-gateway.sunxi-sdimg 로 생성되고 이를 write 해서 실행해보면 기존과 동일하게 실행되는 것을 확인할 수 있다.

## Wi-Fi 추가 하기

Orange-Pi Zero는 무선랜이 내장되어 있고 해당 드라이버는 meta-sunxi/recipes-kernel/xradio/xradio.bb 로 있다.

이를 sc-gateway-image.bb 파일에 다음과 같이 추가한다. 그리고 무선랜 디바이스의 CLI인 iw와 Wi-Fi Protected Access (WPA) client 인 wpa_supplicant 를 같이 추가한다.

```
IMAGE_INSTALL = "packagegroup-core-boot \
    ${CORE_IMAGE_EXTRA_INSTALL} \
    xradio \
    iw \
    wpa_supplicant \
    "
```

다시 빌드를 해보면 xradio 관련하여 에러가 난다. 발생하는 이유는 xradio.bb 파일 내에 `COMPATIBLE_MACHINE` 으로 orange-pi-zero로 한정하였기 때문에 변경한 sc-gateway로는 xradio가 추가되지 않는다.

이 파일을 직접 수정해도 되지만 이 방식 보다는 bbappend 기능을 사용하여 모든 수정은 meta-sc-gateway 에 적용하도록 수정한다.

우선은 orange-pi-zero 로 한정된 recipe를 찾아본다.

```shell
$  grep -nHr 'COMPATIBLE_MACHINE.*=.*orange-pi-zero' meta*
meta-sunxi/recipes-kernel/xradio/xradio.bb:12:COMPATIBLE_MACHINE = "orange-pi-zero"
meta-sunxi/recipes-kernel/xradio-firmware/xradio-firmware.bb:10:COMPATIBLE_MACHINE = "orange-pi-zero"
$
```

위 처럼 xradio에 관련된 recipe만 해당된다.

동일한 디렉토리 구조로 다음과 걑이 bbappend를 만든다.

```shell
$ mkdir -p meta-sc-gateway/recipes-kernel/xradio
$ mkdir -p meta-sc-gateway/recipes-kernel/xradio-firmware
$ echo 'COMPATIBLE_MACHINE:append:sc-gateway = "|sc-gateway"' > meta-sc-gateway/recipes-kernel/xradio/xradio.bbappend
$ echo 'COMPATIBLE_MACHINE:append:sc-gateway = "|sc-gateway"' > meta-sc-gateway/recipes-kernel/xradio-firmware/xradio-firmware.bbappend
```

위와 같이 bbappend로 두 파일의 내용 중 COMPATIBLE_MACHINE 에 sc-gateway를 추가한다. 이렇게 하면 실제 적용되어 다음과 같이 `COMPATIBLE_MACHINE` 이 설정되는 셈이다.

```
COMPATIBLE_MACHINE = "orange-pi-zero|sc-gateway"
```

이와 관련된 사항은 Yocto 매뉴얼의 다음을 참고할 수 있다.

- [Conditional Syntax (Overrides)](https://docs.yoctoproject.org/bitbake/bitbake-user-manual/bitbake-user-manual-metadata.html#conditional-syntax-overrides)
- [Append Files](https://docs.yoctoproject.org/bitbake/bitbake-user-manual/bitbake-user-manual-intro.html?highlight=bbappend#append-files)

이제 다시 빌드해 보면 정상적으로 xradio가 빌드된다. Target에 복사해서 실행 시켜보면 다음과 같이 wlan0 이 추가된 것을 확인할 수 있다.

```shell
$ ifconfig -a wlan0
wlan0     Link encap:Ethernet  HWaddr 12:42:4F:3B:42:7D
          BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

패스워드 설정이 안된 AP 가 있는 경우 다음과 같이 연결을 시킬 수 있다.

```shell
$ ifconfig wlan0 up

# 무선랜 스캔 결과 확인
$ iw dev wlan0 scan

# password가 없는 SSID testap 에 연결하기
$ iw dev wlan0 connect testap
$ udhcpc -i wlan0
$ ifconfig wlan0

# ping 으로 정상 여부 확인
$ ping 1.1.1.1
```

## wpa_supplicant 로 무선랜 연결하기

일반적인 AP의 경우 WPA password 가 설정되어 있어 이는 wpa_supplicant를 사용하여 연결을 하여야 한다.

SSID가 testap, passphrase가 1234567890 이라고 하면 다음과 같이 /etc/wpa_supplicant.conf 파일에 추가한다.

```shell
$ wpa_passphrase "testap" "1234567890" >> /etc/wpa_supplicant.conf
$ cat /etc/wpa_supplicant.conf
...
network={
	ssid="testap"
	#psk="1234567890"
	psk=83cb935df586238f2ead7d9c69b702507f81e9d213dd3879a640cfc0c1faafb5
}
```

설정한 AP로 연결 시도를 해본다.

```shell
$ wpa_supplicant -Dnl80211 -iwlan0 -c/etc/wpa_supplicant.conf
...
wlan0: WPA: Failed to set PTK to the driver (alg=3 keylen=16 bssid=8a:36:6c:4f:38:a4)
[ 2951.993580] wlan0: deauthenticating from 8a:36:6c:4f:38:a4 by local choice (Reason: 1=UNSPECIFIED)
```

PTK (Pairwise Transient Key)는 데이타 전송에 사용하는 암호키를 말한다. SSID, passphrase 를 이용하여 생성된 PSK(Pre-Shared Key)로 EAPOL handshake로 교환한 키이다.

다음과 같이 kernel의 설정을 확인해 본다.

```shell
$ bitbake virtual/kernel -c menuconfig
```

Menuconfig에서 '/' 로 검색에서 LIB80211 을 쳐보면 `LIB80211_CRYPT_CCMP` 등이 모두 모듈로 빌드된다. 관련된 모듈이 패키지에 포함되어 있지 않다.

{{< figure src="kernel-menuconfig.png" width="800px" height="auto" caption="Menuconfig">}}

모듈을 추가하는 방법은 아래 문서를 보고서 적절한 packagegroup 를 추가해 주면 된다.

- [7.2.4. Customizing Images Using Custom Package Groups](https://www.yoctoproject.org/docs/current/mega-manual/mega-manual.html#usingpoky-extend-customimage-customtasks)

하지만 사용하는 meta-sunxi의 경우 kernel 버전이 낮아서 모듈 이름이 다르다. 그리고 몇번의 확인 끝에 wpa_supplicant 에서 CRYPTO_CCM 모듈만 사용하는 것을 확인하였다.

```
IMAGE_INSTALL = "packagegroup-core-boot \
    ${CORE_IMAGE_EXTRA_INSTALL} \
    kernel-module-ccm \
    xradio \
    iw \
    wpa-supplicant \
    "
```

이와 같이 CCM 모듈만 추가를 하고 다시 이미지 만든 후에, 위의 /etc/wpa_supplicant.conf를 설정하고 실행해 보면 정상적으로 작동되는 것을 확인할 수 있다. 이번에는 `-B` 옵션을 주어서 background 로 실행하였다.

```shell
$ wpa_passphrase "testap" "1234567890" >> /etc/wpa_supplicant.conf
$ wpa_supplicant -Dnl80211 -iwlan0 -c/etc/wpa_supplicant.conf -B
$ udhcpc -i wlan0
$ ping 8.8.8.8
```

## 마무리

보드를 살리는 과정은 이렇게 하나 하나 yocto package 내의 설정 들을 찾아서 잡아 주는 과정이다.

wpa_supplicant.conf 의 초기 설정 등을 이미지 생성 전에 하려면 sc-gateway-image.bb 파일에 [ROOTFS_POSTPROCESS_COMMAND](https://docs.yoctoproject.org/ref-manual/variables.html?highlight=rootfs_postprocess_command#term-ROOTFS_POSTPROCESS_COMMAND) 에 스크립트를 추가할 수 있다.

이렇게 yocto layer를 만들어 가는 과정은 마무리하고, 다음으로는 커널이나 개발하는 프로그램을 yocto 에서 관리하고 빌드하는 방법을 알아보도록 한다.
