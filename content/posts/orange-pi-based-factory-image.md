---
title: "Orange Pi 보드용 이미지 수작업으로 만들기"
date: "2018-05-30T10:00:00+09:00"
lastmod: "2018-05-30T10:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["OrangePi"]
categories: ["Embedded Linux"]
aliases: [/orange-pi-based-factory-image/]
---

[Orange Pi](http://www.orangepi.org) 는 Allwinner([SUNXI](http://linux-sunxi.org/Main_Page)) 의 application chip으로 만들어진 single board computer이다. Raspberry Pi 와 비슷하다고 볼 수 있는데, 이것과 비교하여 주요 장단점은 다음과 같다. 



* 장점
  * Cortex A8 single core부터 octa core 까지 라인업
  * Mali400 GPU 내장
  * 무엇보다도 가격이 저렴. 아래에서 사용하는 [Orange Pi R1](http://www.orangepi.org/OrangePiR1/)의 경우 [소비자가가 $9.99](https://www.aliexpress.com/store/product/Orange-Pi-One-ubuntu-linux-and-android-mini-PC-Beyond-and-Compatible-with-Raspberry-Pi-2/1553371_32603308880.html). 
* 단점
  * CPU 사양이 제대로 공개가 안됨. 그나마 H3 정도까지는 인터넷 커뮤니티에 어느정도 공개 됨
  * 발열이 심함. Orange Pi R1의 경우도 별도로 방열판을 붙어야 안정적임



이 문서에서는 Orange Pi R1을 제품에 적용하기 위하여 보드 이미지 설정을 하는 과정을 정리한 것이다. 

소량 제작이기 때문에 별도로 install image를 작성하지는 않고, 배포본을 이용하여 필요한 설정 이나 패키지 설치를 한 후 다시 이를 image 로 만드는 방식으로 한다. 



## 이미지 다운로드 및 SD writing 

사용할 수 있는 이미지는 [Orange Pi 다운로드 페이지](http://www.orangepi.org/downloadresources/)에 debian, ubuntu, openwrt, android 이미지를 제공하지만 이들 모두 documentation이 부족하기 때문에 [armbian](https://www.armbian.com)에서 배포하는 ubuntu server 이미지를 사용한다.

Armbian은 raspberry pi용의 ubuntu image인 [Raspbian](https://www.raspberrypi.org/downloads/raspbian/)과 유사하게 여러 single board computer를 위한 ubuntu/debian image를 제공하는 프로젝트이다. 여기에서 지원하는 프로세서는 다음과 같다. 

- Allwinner A10, A20, A31, H2+, H3, H5, A64
- Amlogic S805 and S905 (Odroid boards), S802/S812, S805, S905, S905X and S912 (fork by `@balbes150`)
- Actionsemi S500
- Freescale / NXP iMx6
- Marvell Armada A380
- Rockchip RK3288
- Samsung Exynos 5422

Orange Pi R1은 Allwinner H2+ 를 사용하고 있다. 



Orange Pi R1에 해당하는 이미지 중 Armbian Xenial 이미지를 다운로드 받는다. Armbian Xenial 은 ubuntu image이고, Stretch 는 debian 이미지 이다. Ubuntu와 Debian의 버전명은 다음과 같은 [페이지](https://askubuntu.com/questions/445487/what-debian-version-are-the-different-ubuntu-versions-based-on)에서 확인할 수 있다. 



해당 파일을 linux machine에서 아래와 같이 sd card로 writing 한다. 최소 2GB 이상의 sd card가 필요하다. 

```sh
sudo fdisk -l 
sudo dd if=Armbian_5.38_Orangepi-r1_Ubuntu_xenial_next_4.14.14.img of=/dev/sdb bs=1M status=progress
```

of는 sdcard의 block device로 `fdisk -l` 명령 등으로 확인할 수 있다. 쓰기가 완료된 sd card를 orange pi 보드에 꽂은 후 전원을 넣으면 부팅한다. 



## 기본 설정

Orange Pi R1 이더넷 커텍터 부근의 커넥터 핀으로 시리얼로 연결하거나, ssh로 접속한다. 초기 계정은 root, 1234 이다. 

Root는 임시로 적절한 암호로 변경하고, 사용자 계정을 하나 만든다. 아래 예에는 pi로 계정을 만든 것으로 가정하고 설명한다. 사용자 계정은 sudoer에 포함되어 있으므로 root의 password는 삭제하여 불필요하게 로그인이 되지 않도록 막는다. 

```sh
sudo passwd -d root
```



Ubuntu 패키지를 최신으로 업데이트 한다. 

```sh
sudo apt update
sudo apt dist-upgrade 
sudo apt autoremove
```



apt를 ssh로 접속하여 실행하면 접속한 host locale과 설정이 다른 경우 warning 메시지가 뜰수 있다. 이를 막기 위하여 LC_CTYPE도 /etc/default/locale 에 추가한다. 

```sh
$ sudo -u root -s
$ echo LC_CTYPE=en_US.UTF-8 >> /etc/default/locale
```



서버를 pi 계정으로 띄우기 때문에 timezone 설정은 사용자 profile에 추가하였다. 

```sh
$ echo "TZ='Asia/Seoul'; export TZ" >> ~/.profile
```



다음과 같이 추가로 필요한 패키지를 설치한다. 

```sh
sudo apt i2c-tools ntpdate
```

각각의 패키지 용도는 다음과 같다. 

* i2c-tools: i2cdetect 프로그램으로 I2C에 연결되어 있는 장치를 확인할 때 사용
* ntpdate: 수동으로 NTP로 시간을 얻을 때 이용



보드에 있는 주변 장치 중 uart2와 i2c0를 사용하여 다음과 같이 설정한다. 

* /boot/armbianEnv.txt 파일의 overlays 항목에 아래와 같이 uart2와 i2c0 를 추가한다. 

  ```sh
  overlays=usbhost2 usbhost3 uart2 i2c0
  ```

* 다시 리부팅을 하면 uart2와 i2c0 를 사용할 수 있다. 장치 디바이스는 각각 /dev/ttyS2, /dev/i2c-0 로 생성된다. 



## RTC 설정(I2C)

기본 보드는 별도의 배터리로 구동되는 RTC(Real Time Clock)이 없어서 전원을 껏다 켜면 매번 시간이 초기화 된다. 다시 네트워크에 연결되어 NTP 등을 이용하여 시간을 받아오면 문제가 없으나, 인터넷 연결이 쉽지 않은 기기에서 시간이 매번 초기화 된다면 로그 시간 정보가 뒤죽박죽 되거나, 시간을 기준으로 처리하는 작업들이 오류가 발생할 수 있다. 

Armbian의 경우 기본적으로 fake-hwclock 패키지가 설정되어 이와 같이 시간이 다시 초기화 되는 문제를 최소화 하고 있다. Fake-hwclock의 동작은 다음과 같다. 

* 초기 부팅 시 /etc/fake-hwclock.data 에 있는 시간 정보를 읽어와 시스템시간을 설정한다. 
* 종료 시 시스템 시간을 /etc/fake-hwclock.data에 저장해 둔다. 
* 주기적으로(cron.hourly) /etc/fake-hwclock.data를 업데이트 한다. 

이와 같이 하면 어느정도는 재부팅 하였을 때 시간이 뒤로 가는 문제는 막을 수 있지만 완전하지는 않다 (중간에 전원이 꺼지는 경우).

이를 보완하기 위하여 I2C로 [MCP47910](http://www.microchip.com/wwwproducts/en/MCP79410) RTC를 연결하였다. 

I2C를 사용하지 않는다면 이 절은 건너띌 수 있다.



i2cdetect를 이용하여 다음과 같이 I2C에 연결된 디바이스를 찾을 수 있다. 

```sh
$ sudo i2cdetect -y 0
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- 57 -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 6f
70: -- -- -- -- -- -- -- --
```

0x57은 MCP79410의 EEPROM control address이고, 0x6f가 실제 RTC를 설정할 수 있는 번지 이다. 



I2C로 연결되는 이와 같은 device를 linux device driver로 등록하는 방법은 다음과 같이 두 가지 방법이 주로 사용된다. 

* /sys/class/i2c-adapter/i2c-0/new_device
* [Device tree overlay](https://www.raspberrypi.org/documentation/configuration/device-tree.md)

첫번째 방법이 간단하여 여기에서는 첫번째 방식을 사용하였다. 



### /sys/class/i2c-adapter/i2c-0/new_device

다음과 같이 RTC device를 시험해 볼 수 있다. 

```sh
$ sudo -u root -s 
$ echo mcp7941x 0x6f > /sys/class/i2c-adapter/i2c-0/new_device
$ hwclock -f /dev/rtc1 -w
$ hwclock -f /dev/rtc1 -r
```

* /sys/class/i2c-adapter/i2c-0/new_device에 쓰면 해당 드라이버가 로드되어 /dev/rtc1 이 생성된다. 참고로 /dev/rtc0는 H2 프로세서 내부의 RTC이다. 



만일 디바이스를 삭제하려면 아래와 같이 하면 된다. 

```sh 
$ echo 0x6f > /sys/class/i2c-adapter/i2c-0/delete_device
```



위와 같은 script를 rc.local 등 적절한 시점에 넣어 주면 된다. 



### Device Tree Overlay

Armbian은 rapbian과 마찬가지로 [device tree overlay](https://www.raspberrypi.org/documentation/configuration/device-tree.md)를 지원한다. ARM 계열 리눅스에서 제공하는 device tree를 동적으로 설정할 수 있도록 하여 타겟 보드의 설정에 따라서 필요한 디바이스를 추가하거나 제거할 수 있다. 

정확히 말하면 동적으로 로딩은 리눅스가 실행된 상태에서 동적으로 디바이스를 올렸다 내렸다 하는 것이 아니라 u-boot에서 kernel을 메모리에 로딩하여 실행되는 시점에서 /boot/armbianEnv.txt 를 이용하여 필요한 항목을 device tree에 추가하는 것이다.



MCP7941x RTC driver는 다음과 같이 dts 파일을 작성한다. 

```d
/dts-v1/;
/plugin/;

/ {
  compatible = "xunlong,orangepi-zeroallwinner,sun8i-h2-plus";

  /*
   * Aliases can be used to set the external RTC as rtc0
   * Needs supplying the correct path to the I2C controller RTC is connected to,
   * this example is for I2C0 on H3
   * NOTE: setting time at boot by the kernel
   * may not work in some cases if the external RTC module is loaded too late
   */
  fragment@0 {
    target-path = "/aliases";
    __overlay__ {
      rtc1 = "/soc/i2c@01c2ac00/mcp7941x@6f";
    };
  };

  fragment@1 {
    target = <&i2c0>;
    __overlay__ {
      #address-cells = <1>;
      #size-cells = <0>;
      mcp7941x@6f {
        compatible = "microchip,mcp7941x";
        reg = <0x6f>;
        status = "okay";
      };
    };
  };
};
```



위 문법을 보면 기존의 RTC와 유사하나 앞에 ```/plugin/``` 으로 overlay 형식을 알려주고, ```fragment``` 로 추가될 부분을 기록한다. Device tree 문법은 '[Device Tree for Dummies](https://elinux.org/images/f/f9/Petazzoni-device-tree-dummies_0.pdf)' 등의 참고 자료나 linux 소스의 Documentation/devicetree를 보고 이해하여야 한다. 

위 사항은 간략히 설명하면 fragment@0 은 새로 추가될 MCP7941x가 rtc1 alias로 등록되도록 하는 것이고, fragment@1 은 i2c0 버스 내dml 0x6f 주소에 'microchip,mcp7941x' driver를 등록하는 것이다. 해당 driver는 linux/drivers/rtc/rtc-ds1307.c 를 참고하면 된다. 



위와 같은 dts를 dtbo로 binary 형식으로 변환하기 위하여는 armbian에서 patch한 dtc를 사용하여야 한다. Overlay patch한 DTC를 아래처럼 받아서 빌드해서 사용한다. 

```sh
$ git clone -b dt-overaly8 https://github.com/pantoniou/dtc
$ cd dtc
$ sudo apt-get install flex bison 
$ make 
$ ./dtc -@ -O dtb -I dts -o sun8i-h3-rtc-mcp7941x.dtbo sun8i-h3-rtc-mcp7941x.dts
```



생성된 파일을 /boot/dtb/overlay/ 에 복사한 후 /boot/amrbianEnv.txt 의 overlay에 'rtc-mcp7941x' (overlay_prefix 문자열을 제외한)를 적어주고 리부팅 하면 된다. 

확인은 dmesg로 kernel log를 확인하여 RTC1 이 등록되어는지 확인 해보면 된다. 



###  Systemd 등록

위 두 방법 중 두번째 방법이 깔끔하기는 하지만 수정하여야 하는 범위가 넓어서 첫번째 방식으로 수정을 하여 systemd에 등록하는 것으로 한다. 



/etc/systemd/system/hwclock-i2c-mcp7941x.service 로 다음과 같이 파일을 생성한다.

```ini
[Unit]
Description=Restore / save MCP7941x RTC
DefaultDependencies=no
Before=sysinit.target
Conflicts=shutdown.target

[Service]
ExecStart=/sbin/hwclock-i2c-mcp7941x load
ExecStop=/sbin/hwclock-i2c-mcp7941x save
Type=oneshot
RemainAfterExit=yes

[Install]
WantedBy=sysinit.target
```



/sbin/hwclock-i2c-mcp7941x 로 다음과 같이 파일을 만든다. 

```sh
#!/bin/sh

RUNFILE=/run/hwclock-i2c-mcp7941x

COMMAND=$1
if [ "$COMMAND"x = ""x ] ; then
    COMMAND="save"
fi

case $COMMAND in
    load)
        if [ ! -e $RUNFILE ] ; then
            touch $RUNFILE
            echo mcp7941x 0x6f > /sys/class/i2c-adapter/i2c-0/new_device
            sleep 1
        fi

        /sbin/hwclock -f /dev/rtc1  -s
        ;;
    save)
        /sbin/hwclock -f /dev/rtc1 -w
        ;;
    *)
        echo $0: Unknown command $COMMAND
        exit 1
        ;;
esac
```



다음과 같이 실행하여 daemon을 등록하고 시작한다. 

```sh
$ sudo chmod +x /sbin/hwclock-i2c-mcp7941x
$ sudo systemctl daemon-reload
$ sudo systemctl enable hwclokc-i2c-mcp7941x
$ sudo systemctl start hwclokc-i2c-mcp7941x
```



정상적으로 동작되는지 확인은 lsmod로 rtc_ds1307 모듈(이 모듈내에 유사한 드라이버가 다 포함되어 있음)이 로딩되었는지를 확인하거나 아래와 같이 hwclock으로 시간을 설정해보고 읽어보면 된다. 



## 추가 필요한 모듈 설치 

기본 설정을 마친 상태에서 필요한 프로그램을 추가로 설치하여 부팅 시 호출되도록 설정해주면 된다. 

모든 작업을 마친 후에는 SD card를 이미지화 하기 위하여 정상적으로 shutdown 시키고 SD card를 분리한다. 

```sh
$ sudo shutdown -h now
```



## 디스크 이미지 shrink 하기

디스크 이미지의 rootfs 파티션은 보통 SD 카드의 전체 공간으로 할당되어 있다. 하지만 다른 SD card로 복제할 때 전체 공간을 그대로 복제하는 것 보다는 최소한으로 partition을 줄여서 읽어 들인 후, 이를 복제하고, 복제된 것으로 부팅 시 다시 SD card 전체 공간으로 파티션 크기를 넓히는 것이 시간면에서 유리하다. 



이 과정을 target에서 SD card로 하기에는 위험성도 있고, 시간도 걸려 host linux에서 하는 것으로 한다. 

위 과정으로 만들어진 SD 카드를 host PC에 USB 카드 리더기 등을 이용하여 꽂고 다음과 같이 전제 영역을 읽어 들인다. 

```sh
$ sudo fdisk -l
$ sudo dd if=/dev/sdb of=target.img bs=1M status=progress
```

SD card 읽고 쓰기 과정은 잘못해서 다른 디스크에 한다면 치명적일 수 있으므로, fdisk를 이용하여 해당 디스크가 맞는지 주의하여야 한다. 



읽어 들인 이미지를 loop device를 이용하여 block device로 등록시킨다. 관련 부분은 이전에 '[SD card 디스크 이미지 만들고 수정하는 방법 정리](https://blog.humminglab.io/how-to-make-sdcard-disk-image/)'를 참고할 수 있다. 

```sh
$ sudo losetup -Pf --show target.img
```

위와 같이 하면 /dev/loop0 과 파티션 개수만큼 /dev/loop0p1 처럼 생긴다. 



Armbian은 ex4 파일시스템을 사용하고 있다. 디스크의 파일 시스템이 정상인지 확인하고 필요하면 수정한다. 

```sh
$ sudo fsck -f /dev/loop0p1
```



우선 파일 시스템의 크기를 줄여야 한다. 다음과 같이 하여 최소 크기로 줄이도록 실행한다. 

```sh
$ sudo resize2fs -M -p  /dev/loop0p1
resize2fs 1.42.13 (17-May-2015)
Resizing the filesystem on /dev/loop0p1 to 466744 (4k) blocks.
Begin pass 2 (max = 112981)
Relocating blocks             XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
Begin pass 3 (max = 120)
Scanning inode table          XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
Begin pass 4 (max = 5486)
Updating inode references     XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
The filesystem on /dev/loop0p1 is now 466744 (4k) blocks long.
```

완료되면 위와 같이 446,744 개수의 4KiB 블럭만큼 줄었다고 알려준다.  466,744*4kiB=1,866,976kiB 크기가 된다. 위 과정은 file system을 줄인 과정이고, 다음은 전체 partiton 크기를 file system에 맞도록 줄여주어야 한다. 



다음과 같이 parted를 이용하여 파티션 공간을 확인한다(만일 GUI 환경이 있다면 gparted로 쉽게 할수도 있다).

```sh
$ sudo parted /dev/loop0
...
(parted) unit kiB
(parted) print
Model: Loopback device (loopback)
Disk /dev/loop0: 15793152kiB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start    End          Size         Type     File system  Flags
 1      4096kiB  15635216kiB  15631120kiB  primary  ext4
```

표시 단위를 kiB로 변경 후 본 파티션 공간이다. parted로는 resizepart 로 파티션 크기 변경 시에는 End를 변경할 수 있다. 새로 변경할 End 위치는 Start(4,096kiB) + Filesystem Size(1,866,976kiB) = 1,871,072kiB가 된다. 이 크기로 partition 크기를 줄인다. 

```sh
$ sudo parted /dev/loop0
(parted) unit kiB
(parted) resizepart
Partition number? 1
End?  [15635214kiB]? 1871072
Warning: Shrinking a partition can cause data loss, are you sure you want to continue?
Yes/No? y
(parted) print
Model: Loopback device (loopback)
Disk /dev/loop0: 15793152kiB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start    End         Size        Type     File system  Flags
 1      4096kiB  1871072kiB  1866977kiB  primary  ext4
```

위와 같이 정상적으로 줄어든 것을 볼 수 있다. 다시 fsck를 이용하여 해당 파티션이 문제가 없는지 확인해 보고, loop0 device를 제거한다. 

```sh
$ sudo fsck -f /dev/loop0p1
$ sudo losetup -d /dev/loop0
```



Truncate 를 사용하여 image 파일의 뒷 부분을 제거한다. 아래와 같이 파티션의 끝 위치에 조금 여유를 두고 잘라 낸다.

```sh
$ sudo truncate --size=$[(1871072+1)*1024] target.img
```



이와 같이 하면 해당 이미지를 target 용 SD 카드로 복제할 수 있는 수작업 이미지가 만들어 진다. 



## 마무리 

위와 같이 만들어진 이미지를 복제 후 부팅을 하여 정상동작 되는지 확인한다. 

초기화 설정이 안되어서 자동으로 파티션 영역이 확장되지는 않는다. 다음과 같이 하여 수동으로 전체 SD 공간으로 partition 및 파일 시스템 크기를 키울 수 있다. 

```sh
$ sudo /etc/init.d/resize2fs start
```



RTC를 사용한 다면 ntpdate 등으로 시간을 얻어온 후 RTC에 써주는 과정 등의 추가 초기 설정을 해주면 마무리 된다. 

