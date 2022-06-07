---
title: "Systemd의 특징과 Yocto에 적용하기"
date: "2022-06-07T09:00:00+09:00"
lastmod: "2022-06-07T09:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Yocto", "Linux", "systemd", "SysV", "journald"]
categories: ["Development"]
---

Yocto project에서 기본 설정으로 빌드하면 SysV Init를 사용한다.
개발하는 제품이 이더넷 네트워크로 연결되고, 부팅 이후에는 네트워크 환경이 변하지 않는다면 SysV Init를 이용하는 것이 구조도 단순해서 더 좋을 수 있다.

하지만 다음과 같은 사항을 고려하고 있다면 systemd를 적용하는 것을 검토해 볼 수 있다.

- Daemon 이 죽는 경우를 검출하여 재시작 관리가 필요한 경우
- Wi-Fi 와 같이 동적으로 변경될 수 있는 네트워크 관리가 필요한 경우
- 불규칙하게 네트워크가 끊길 수 있는 조건에서 시간 동기화가 필요한 경우
- 프로그램에 CPU 또는 메모리 자원을 제한하기 위하여 [cgroups](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)를 사용하려는 경우
- 효과적인 로그 관리를 위하여 journald를 사용하고 싶은 경우
- 부팅 직후 초기 프로세스의 실행 시간을 줄여 보려는 경우

물론 위의 기능을 사용하기 위해서 systemd만 가능한 것은 아니지만, systemd를 사용하는 경우 별도의 프로그램 없이 위 기능을 쉽게 적용할 수 있다.

## Yocto 에서 systemd 적용

Yocto의 init manager를 설정하는 방법은 [Yocto Project Development Tasks Manual, 3.25 Selecting an Initialization Manager](https://docs.yoctoproject.org/dev-manual/common-tasks.html#selecting-an-initialization-manager)를 참고할 수 있다.

Systemd를 적용하기 위하여는 local.conf 파일이나 적절한 곳에 아래와 같이 추가하면 된다.

```
DISTRO_FEATURES:append = " systemd"
VIRTUAL-RUNTIME_init_manager = "systemd"
VIRTUAL-RUNTIME_initscripts = "systemd-compat-units"
```

또는 아래 처럼 한줄을 추가해주는 것도 가능하다.

```
INIT_MANAGER = "systemd"
```

위와 같이 추가하면 `poky/meta/conf/distro/defaultsetup.conf` 파일에서 적절한 설정 파일을 추가한다.

Systemd인 경우 `poky/meta/conf/distro/include/init-manager-systemd.inc` 파일이 추가되고, 이 파일에 위의 설정이 포함되어 있다. login_manager, dev_manager도 추가로 설정한다.

빌드 후 타겟에 설치하여 실행하면 SysV init 설정 중 필요없는 몇가지를 추가로 제거할 수 있다.

sysvinit 관련한 것들이 의존성으로 추가될 가능성을 막으려면 아래처럼 설정을 추가하면 된다.
관련 설정은 [Feature Backfilling](https://docs.yoctoproject.org/ref-manual/features.html#ref-features-backfill)을 참고하면 된다.

```
DISTRO_FEATURES_BACKFILL_CONSIDERED:append = " sysvinit"
```

## systemd 개요

Systemd 관련하여 인터넷 체계적으로 잘 정리된 것을 찾기는 힘들다.

책으로 보기를 원한다면 원서이지만 최근에 출간된 아래 책을 추천한다. Systemd 의 전체적인 사항들에 대하여 실제 서버에서 설정 예를 참고로 쉽게 정리되어 있다.

- [Linux Service Management Made Easy with systemd, Donald A. Tevault, Packt Publishing (February 3, 2022)](https://www.amazon.com/Linux-Service-Management-Made-systemd/dp/1801811644/ref=sr_1_1?keywords=Linux+Service+Management+Made+Easy+with+systemd&qid=1652500911&sr=8-1)

Systemd는 boot script만이 아니라, 아래에서 설명할 networkd, resolved, timesysncd, logind, journald 등 여러 서비스들을 기본으로 제공한다.
한마디로 Linux 시스템 운영에 필요한 전반적인 것을 통합 관리하는 툴이라고 말할 수 있다.

하나만 잘하자는 UNIX의 철학에 위배 된다고도 하지만, 실제 사용 관점에서 어떤 배포본을 사용하더라도 일관된 시스템 운영을 할 수 있다는 것과
systemd에서만 제공하는 장점들이 있어 최근 대부분의 리눅스 배포본이 systemd를 적용하고 있다.

## systemd 서비스

먼저 기존 SysV init script의 단점을 정리해 보면 다음과 같다.

- Script가 파일이름 앞 숫자에 따라 순차적으로 실행되어 느림
- Run level로 그래픽, 콘솔 모드를 설정할 수 있으나, 배포본 마다 설정이 다름
- Bash script로 구성 되어 있어 동작을 이해하려면 script 코드를 분석해야 함. 같은 프로그램이더라도 배포본 마다 실행 script가 다름

Systemd를 사용하게 되면 위의 단점을 해결하고, 추가적인 장점을 얻을 수 있다.

- 파일이름의 번호 순이 아니라, 설정 파일에 명시된 의존성에 따라 실행된다. 의존성이 없는 경우 병렬로 실행이 되어 부팅 속도도 빨라진다.
- Script가 아닌 INI 형식의 설정 파일로 구성되어 일관된 설정을 할 수 있다.
- 데몬이 죽는 경우 등의 예외 처리등을 설정으로 적용할 수 있다.
- Namespace, cgroups를 이용하여 프로세스의 권한 및 시스템 자원을 제한할 수 있어 보안성이 높고, 시스템 자원을 효율적으로 분배할 수 있다.

Systemd에 실행파일을 포한한 설정은 /lib/systemd 디렉토리 안에 있다.

설정 파일은 해당 디렉토리 안의 /lib/systemd/system 디렉토리에 저장된다. 이 설정 파일들을 unit file 이라고 하는데 확장자에 따라 다음과 같이 구분된다.

- **service**: service 실행
- **socket**: inetd, [xinetd](https://en.wikipedia.org/wiki/Xinetd) 대체. 슈퍼데몬 방식으로 포트로 실제 입력이 있는 경우 해당 daemon 실행 시키기
- **slice**: CGgroups 설정용
- **mount**, **automount**: 마운트 포인트
- **target**: 초기 기동 시 unit들을 그룹핑 하기.
- **timer**: cron system 대체
- **path**: path-based activation
- **swap**: swap partition에 대한 정보

SysV init script에 대응되는 것은 service unit file이고, runlevel과 대체 되는 것은 target 이다.

SysV runlevel과 비교해 보면 target은 다음과 같다.

| SysV runlevel | systemd target    |
| ------------- | ----------------- |
| 0             | poweroff.target   |
| 1             | rescue.target     |
| 2, 3, 4       | multi-user.target |
| 5             | graphical.target  |
| 6             | reboot.target     |

이외에도 매핑은 안되지만 emergency.target, hibernate.target, suspend.target 등이 있다.

Systemd 에 대한 설명을 하기에는 내용이 커져서 여기에서는 실제 사용에 필요한 사항만 정리해본다.

처음에 어떤 target으로 설정할 지는 아래와 같이 systemctl 명령이나 직접 파일을 확인 가능하다. set-default 서브 명령으로 변경도 가능하다.

```shell
$ systemctl get-default
multi-user.target

$ ls -l /etc/systemd/system/default.target
lrwxrwxrwx    1 root     root   41 May 16 02:46 /etc/systemd/system/default.target -> /usr/lib/systemd/system/multi-user.target

$ sudo systemctl set-default multi-user
```

임시로 target 을 변경하는 것도 가능하다. 이와 같이 하면 의존성이 없는 서비스는 자동으로 중단되고 필요한 서비스들이 실행된다.

```shell
$ sudo systemctl isolate multi-user
$ sudo systemctl isolate graphical
```

참고로 systemd는 대부분 기존의 SysV 와도 호환 가능하도록 되어 있다. 예를 들어 위 isolate는 init 명령으로도 사용할 수 있다.
아래와 같이 sysv 호환되는 target도 생성되어 있다.

```shell
$ sudo init 3

$ ls -l /lib/systemd/system/runlevel*
lrwxrwxrwx    1 root     root            15 May 16 02:46 /lib/systemd/system/runlevel0.target -> poweroff.target
lrwxrwxrwx    1 root     root            13 May 16 02:46 /lib/systemd/system/runlevel1.target -> rescue.target
lrwxrwxrwx    1 root     root            17 May 16 02:46 /lib/systemd/system/runlevel2.target -> multi-user.target
lrwxrwxrwx    1 root     root            17 May 16 02:46 /lib/systemd/system/runlevel3.target -> multi-user.target
lrwxrwxrwx    1 root     root            17 May 16 02:46 /lib/systemd/system/runlevel4.target -> multi-user.target
lrwxrwxrwx    1 root     root            16 May 16 02:46 /lib/systemd/system/runlevel5.target -> graphical.target
lrwxrwxrwx    1 root     root            13 May 16 02:46 /lib/systemd/system/runlevel6.target -> reboot.target
```

Unit 파일을 확인해 보려면 다음과 같이 확인 할 수 있다. list-units는 메모리에 로드된 것을 list-unit-files은 설치된 파일을 확인한다.

```shell
$ systemctl list-units
$ systemctl list-units --all
# service unit 만 보기
$ systemctl list-units -t service
# 죽은 서비스만 보기
$ systemctl list-units -t service --state=dead
# state 목록 보기
$ systemctl --state=help
# 설치된 unit files 보기
$ systemctl list-unit-files
# 설정 확인
$ systemctl is-enabled systemd-networkd
$ systemctl is-active systemd-networkd
# config 보기
$ systemctl show
```

Service를 사용할지 말지 여부는 enable/disable로 설정 가능하다. Enable 되면 매 부팅 시 자동으로 실행된다.
Start/stop으로 임시로 서비스를 실행 중지 시킬 수 있다.

```shell
$ sudo systemctl enable systemd-networkd
$ sudo systemctl disable systemd-networkd
# 바로 서비스도 시작하기
$ sudo systemctl enable --now systemd-networkd

$ sudo systemctl start systemd-networkd
$ sudo systemctl stop systemd-networkd
```

어떤 service가 실행된 이유를 보려면 dependency로 확인 가능하다. `--all, --before, --after` 등을 사용해서 의존성을 구분해서 볼수 있다.

```shell
$ systemctl list-dependencies systemd-networkd
$ systemctl list-dependencies --after systemd-networkd
```

Systemd에 관련된 사항은 man 으로 확인을 할 수 있다. 목차에 해당하는 부분이 systemd.directives 이다.
이 부분 부터 확인하여 필요한 부분의 man 페이지 명을 확인할 수 있다.

```shell
$ man systemd.directives
```

## systemd-networkd, systemd-resolved

각각의 역할은 다음과 같은 기능을 제공한다.

- systemd-networkd
  - 사전에 설정한 네트워크 정보로 자동 연결 수행
  - 연결 단절 시 systemd-networkd-wait-online 을 실행하여 재연결 수행
- systemd-resolved
  - DHCP, IPv6 advertisement 로 얻은 DNS 정보로 /etc/resolv.conf 파일을 업데이트

Ubuntu 등에서는 systemd-networkd를 사용하지 않고 [NetworkManager](https://networkmanager.dev/)를 주로 사용한다. 이 NetworkManager는 [nmcli](https://developer-old.gnome.org/NetworkManager/stable/nmcli.html), [nmtui](https://developer-old.gnome.org/NetworkManager/stable/nmtui.html) 와 같은 설정 프로그램을 이용하여 사용자가 쉽게 관리할 수 있다.

이것과 비교하여 systemd-networkd는 다음과 같은 차이가 있다.

- 설정은 NetworkManager에 비하여 UI 툴은 제공하지 않고, networkctl 설정 프로그램을 제공한다.
- NetworkManager와 비교하여 Container를 위한 bridge 설정, CAN(Controller Area Network) 를 지원한다.

systemd-networkd, systemd-resolved 를 사용하면 유무선 네트워크 상황이 바뀌더라도 자동으로 연결하여 적절하게 네트워크 설정이 가능해진다.

SysV 자체로는 초기에 dhcp client 데몬을 실행시키는 정도 밖에는 제공되지 않아 별도의 network manager가 필요하다.

## systemd-timesyncd

SysV init 를 yocto 에 적용 시 시간 관련된 기본 동작은 init script에서 다음과 같은 부분만 수행한다.

- 부팅 시 RTC driver에서 시간을 가져와 시스템 시간 설정
- Shutdown 시 시스템 시간을 RTC 에 저장하여 시간 유지

주기적으로 SNTP 등을 이용한 시간 동기를 맞추는 부분과, 강제로 power off가 될 경우를 고려하여 RTC 동기화 부분을 별도로 구현하여야 한다.

PC 급에서는 초기에는 [ntpd](https://www.eecis.udel.edu/~mills/ntp/html/index.html)를 사용하였으나, 단점도 있고 2017년 code audit으로 보안 취약점들이 발견되었다. ([참고](https://wiki.mozilla.org/images/e/ea/Ntp-report.pdf))

최근에는 [chrony](https://chrony.tuxfamily.org/)를 많이 사용하고 있다. Chrony는 다음과 같은 특징이 있다.

- [NTP(Network Time Protocol)](https://datatracker.ietf.org/doc/html/rfc5905) 지원
- 네트워크가 불규칙하게 연결되는 환경에서도 시간 동기화가 잘 됨
- 온도 변화에 따른 HW clock osillator의 변화 보정
- HW timestamping, HW reference clock 사용하여 sub-microsecond 정밀도도 가능

같은 존에서 운영하는 network service 들의 경우 chrony를 사용하여 정밀하게 시간 동기화를 하는 것이 필요할 수 있다.

Yocto 기반의 edge gateway를 만들 때 100ms 정도의 오차여도 문제가 없다면 [SNTP(Simple NTP)](https://datatracker.ietf.org/doc/html/rfc4330) 여도 충분할 것이다. 이런 경우 systemd-timesyncd 를 사용할 수 있다.

systemd-timesyncd 는 다음과 같은 사항을 지원한다.

- SNTP client
- 주기적인, 네트워크 연결 시 자동 시간 동기화
- RTC 동기화

Yocto에서 systemd를 추가하면 자동으로 service로 실행된다.
보통은 켜놓기만 하면 알아서 시간동기가 되므로 신경 쓸 필요가 없고, 필요하다면 `/etc/systemd/timesyncd.conf` 를 수정하면 된다.

## journald

기존의 rsyslog를 대체 하는 systemd의 로깅 시스템이다.

Desktop이나 Server Linux에서는 호환성과 편리성을 위하여 journald와 기존 syslog 둘 모두 같이 실행되도록 설정하여 사용한다. 
임베디드 환경에서는 저장공간을 감안하여 둘 중 어느 것을 사용할지 선택을 할 수 있다. 

Syslog와 journald 에 대한 부분은 내용이 길어져서 먼저 [Syslog and Journald]({{< ref "posts/syslog-journald">}}) 로 정리하였다. 

자세한 설명은 해당 글을 참조하면 된다.

## Embedded Yocto 에서 고려사항

Yocto 에 systemd를 추가하면 기본적으로 다음과 같은 service가 실행된다. 

- system-getty.slice
- systemd-udevd.service
- systemd-timesyncd.service
- systemd-networkd.service
- systemd-resolved.service
- systemd-journald.service

위에서 언급한 것 이외에 udevd device management와 getty teminal 데몬이 추가로 실행된다. 
위 서비스 중 journald를 제외하고는 기본설정을 그대로 사용하여도 특별히 문제가 되지 않는다. 

하지만 journald 의 경우 로그가 메모리나 저장장치에 남게 되어 시스템의 자원을 소모하기 때문에 신경 써서 설정하여야 한다. 

Busybox klogd, syslogd도 기본으로 실행되는데, 불필요하다고 판단되면 해당 데몬의 실행을 중지하여야 한다.
중지를 하는 경우 journald 에서 syslog로 forward하는 것을 끄는 것이 좋다. 

Journald 기본 설정은 Sotrage=auto 로 `/var/log/journal/` 디렉토리가 있으면 storage로 보고 영구저장을 하고, 없으면 `/run/log/journal/` 에 임시로 저장한다. 보통 이 디렉토리는 tmpfs 이기도 하고, 파일은 리부팅하면 journald가 자동으로 기존 로그를 삭제한다. 기본 설정에서 `/var/log/journal/` 디렉토리가 없으므로 임시 로그가 설정되고, 이에 관련된 설정은 RuntimeMaxUse, RuntimeKeepFree, ... 등의 필드가 있다.

Yocto에서는 `poky/meta/recipes-core/systemd/systemd-conf/journald.conf` 로 다음과 같이 기본 설정을 변경하고 있다. 

```
[Journal]
ForwardToSyslog=yes
RuntimeMaxUse=64M
```

이로 인하여 메모리가 적은 시스템에서는 `/var` 공간이 로그로 가득차는 문제가 발생할 수 있다. 

이를 bbappend 등으로 적절하게 수정하여 저장 용량을 조정하는 것이 필요할 수 있다. 실제 타겟 이미지에는 `/lib/systemd/journald.conf.d/00-systemd-conf.conf` 파일로 생성된다.


## 마무리

이상으로 systemd의 주요 모듈에 대한 설명과 사용법을 정리하였고, Yocto에서 설정하는 법을 설명하였다.

Systemd에는 위에 언급한 것 이외에도 여러 서비스들이 있다. 
`/lib/systemd` 폴더의 실행파일들이 제공되는 실행파일들의 이름을 보면 대략적으로 어떤 일을 하는지 알 수 있을 것이다.

임베디드 시스템 이미지를 직접 구축을 하는 경우라면 위에 추천한 도서를 참고하여 전체적인 systemd의 설정 파일을 하나하나 차분히 점검하고 튜닝하는 것이 좋을 것이다.