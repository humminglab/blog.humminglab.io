---
title: "Syslog and Journald"
date: "2022-05-27T09:00:00+09:00"
lastmod: "2022-05-28T09:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Yocto", "syslog", "rsyslog", "systemd", "journald"]
categories: ["Development"]
---

대부분의 최신 linux 배포본에서 systemd를 적용하면서 로그 시스템도 syslog 에서 systemd 의 journald로 변경되었다.

PC급 이상의 linux 배포본에서는 journald와 기존 호환성을 고려하여 syslog 데몬이 같이 사용하도록 기본 설정되어 있고,
상대적으로 광활한 저장장치과 메모리를 가지고 있고, 적절한 용량 선에서 log rotate가 되도록 설정되어 있어,
사용자가 설치 후 로그에 대해서는 신경을 쓸 필요가 거의 없다.

하지만 용량이 작은 저장장치와 메모리를 가진 embedded linux 제품을 개발하는 경우에는 시스템 로그를 어떤 식으로 관리 할지 충분히 고민하고 설정하여야 한다.
그렇지 않다면 저장장치나 메모리가 로그로 가득차 버려서 더이상 동작을 하지 못하는 문제가 발생할 수 있다.
Flash memory를 저장장치로 사용하는 경우에는 제품 수명 보다 저장장치의 수명이 길 수 있도록 erase 횟수도 감안하여 고려하여야 한다.

이 글에서는 다음과 같은 사항을 정리한다.

- Linux 시스템에서 kernel log, syslog의 관리 방법
- Syslog와 Journald의 장단점
- Yocto 기반 embedded linux에서 고려 사항

## Syslog daemon

우선 syslog의 운영 방식에 대해서 정리해 본다.
이 글에서는 linux를 기준으로 설명한다. 다른 Unix 계열도 비슷하지만 파일 위치나 디바이스 드라이버가 다를 수 있다.

System에 로그를 남기기 위해서 프로그램은 GNU C library내의 syslog 함수를 사용할 수 있다.

- [GNU C Library - syslog](https://www.gnu.org/software/libc/manual/html_node/Syslog.html)

`openlog()` 를 이용하여 적절한 로그 구분이름(ident)를 설정하고, `syslog()`, `vsyslog()` 함수를 이용하여 원하는 로그를 출력하게 되면,
syslog daemon이 이를 받아서 `/var/log` 디렉토리에 text 형식으로 로그를 저장한다.

또는 [logger](https://man7.org/linux/man-pages/man1/logger.1.html) 프로그램을 이용하여 아래 처럼 로그를 남기는 것도 가능하다.

```shell
$ logger "hello world"

$ tail /var/log/syslog
...
May 26 11:43:39 humminglab yslee: hello world
```

Syslog daemon이 로그 메시지를 수신 받는 방법은 단순하다.
데몬은 실행 시 Unix domain socket을 `/dev/log` 로 생성하고, `syslog()` 에서는 이 소켓을 이용하여 로그를 전달한다.

아래처럼 이 소켓에 직접 로그 데이타를 넣어도 된다. 소켓 파일이므로 redirection은 안되고, netcat을 이용할 수 있다.

이 경우 데이터 형식은 [RFC 5424 - The Syslog Protocol](https://www.rfc-editor.org/rfc/rfc5424#section-6) 을 참고하면 된다.

```shell
$ echo '<13>Jan  1 00:00:00 test[1234]: hello' | nc -uU /dev/log

$ tail /var/log/syslog
...
May 26 12:08:51 humminglab test[1234]: hello
```

외부의 syslog 서버로 전달도 마찬가지로 동일한 방법을 사용한다. 다만 차이는 domain socket이 아니라 UDP/TCP 소켓으로 (Default syslog port: 514) 전달하는 것이다.

이와 같은 데몬으로는 syslog, syslog-ng, rsyslog 와 같은 프로그램이 있고, 순서대로 각각은 앞 제품을 개선한 것이라고 보면 된다. Ubuntu의 경우 rsyslog를 사용한다.

## Kernel log

Syslog의 로그 내용을 보면 kernel에서 출력하는 로그들도 기록 되는데 이는 klogd 데몬이 담당한다.

Klogd의 역할은 kernel log 가 발생하면 이를 읽어서 syslog daemon으로 전달하는 것이다.

이 부분의 작동 원리는 다음과 같다.

Linux kernel은 출력되는 로그를 메모리에 저장하고, 이를 접근하기 위한 `/proc/kmsg` 와 `/dev/kmsg` 를 제공한다.
둘의 특성은 차이가 있는데, `/proc/kmsg` 는 read-once 이다. 즉, 한번 `/proc/kmsg` 파일에서 읽으면 `/proc/kmsg` 는 비워진다.

`/dev/kmsg` 는 read-write 속성으로 여러번 읽어도 가능하다. dmesg 를 이용하여 kernel log를 읽는 경우와 같다고 보면 된다.
그리고 아래 처럼 `/dev/kmsg`로 kernel log를 쓰는 것도 가능하다.

```shell
$ cat /dev/kmsg

$ echo 'test: hello' | sudo tee -a /dev/kmsg
# root 권한이라면
# echo 'test: hello' > /dev/kmsg
```

동작 특성을 보면 알 수 있듯이 `/dev/kmsg` 는 dmesg 출력 용도로 사용하고, `/proc/kmsg` 는 klogd 에서 kernel log를 syslog로 전달하는 용도로 사용한다.

## Busybox klogd & syslogd

Yocto 의 기본 설정에서 syslogd, klogd는 busybox에 포함된 프로그램을 사용한다.

Klogd의 경우 단순히 syslog로만 전달하는 것이므로 특별히 신경쓸 것은 없다.

Busybox syslogd의 경우 200KB 의 용량으로 하나의 파일만 이용하여 log rotate 하는 것이 기본 설정이다.
로그를 남기는 `/var/log` 디렉토리도 보통은 tmpfs 로 마운트하여 사용하여 별도로 저장장치에 로그를 남기지 않는다.
결과적으로 메모리 200KB 가량만 로그 저장을 위하여 사용하므로 특별히 신경쓸 것은 없다.

기본 설정은 BusyBox 도움말에서 확인할 수 있다.

- [BusyBox - The Swiss Army Knife of Embedded Linux](https://busybox.net/downloads/BusyBox.html)

다만 반대로 로그를 저장장치에 저장하여 리부팅 된 이후에도 확인 가능토록 하려면 저장 위치를 바꾸거나, 외부 syslog로 전달하도록 설정하여야 한다.

## Journald 와 syslogd의 필요성

Journald는 기존 syslog의 단점을 보완한 것이라고 생각할 수 있는데, 먼저 syslog의 단점을 정리해 보면 다음과 같다.

- 텍스트 형식이라 로그 파일의 크기가 큼
- 별도의 툴 없이는 특정 데몬의 로그만 본다던지 하는 구조화된 분류가 안되고, 검색도 느리고 원하는 부분을 찾기가 쉽지 않음
- 텍스트 형식이라 해커의 로그 조작이 쉬움

Journald는 이를 다음과 같이 개선한 것이다.

- **Binary & Indexing**: Binary 형식으로 데이타를 저장하고 indexing 하여 검색이 빠름
- **Structure logging**: 기본으로 로그 분석이 가능한 툴(journalctl) 제공하고, 구조화된 로그 제공(특정 데몬만 보거나, 특정 시점만 확인하거나)
- **Access control**: 사용자에 따른 별도의 로그로 억세스 제어 가능
- **Automatic log rotation**: 기본 제공
- **Forward Secure Sealing(FSS)**: sealing key로 sealing 하고, verification key로 로그의 유효성 검증하여 로그 조작 방지

무엇보다고 가장 좋은 기능은 구조화된 로깅으로 mini command line 로그분석도구 라고도 할 수 있다.

하지만 journald의 단점은 반대로 binary 로 저장되어 텍스트로 저장하는 syslog처럼 다른 시스템과 연동이 어렵다는 것이다.
대부분 원격 로깅을 위한 log-shipper 에서는 syslog를 기본으로 사용하는 것이 많다.

이를 보완하기 위하여 보통 journald와 syslog daemon이 같이 동작하는 것을 제공한다.

우선 더 자세한 로그를 남기는 journald가 호스트에서 생성되는 로그를 수집하여 저장하고, 이를 syslog daemon에서 다음과 같이 두 방법 중 하나로 참조한다.

- syslog daemon이 직접 journal 로그 읽기: [rsyslogd imjournal](https://www.rsyslog.com/doc/v8-stable/configuration/modules/imjournal.html)
  - syslog 프로토콜로는 severity, hostname, message 정도 밖에 전송 되지 않으므로 직접 엑세스 하는 것이 더 많은 정보를 얻을 수 있음
  - 다른 프로그램이 journald 데이타를 참조하므로 불안정할 수 있고, 읽기 속도가 빠르지 않음
- journald에서 syslog로 메시지 forward 하기: [rsyslogd imuxsock](https://www.rsyslog.com/doc/v8-stable/configuration/modules/imuxsock.html)
  - syslog 프로토콜로 전달하므로 안정적. 전달은 /run/systemd/journal/syslog 로 생성한 unix domain socket을 이용
  - 로그가 두 곳에 각각 저장

Ubuntu의 경우 기본 설정은 imuxsock을 이용하여 syslog로 forward하는 방식으로 설정되어 있다.

## Yocto에서 journald를 적용 시 고려사항

어떤 로그시스템을 선택할지는 메모리와 저장장치의 상황을 감안하여 다음과 같이 결정할 수 있을 것이다.

- 로그를 볼일이 거의 없다면 journald 보다는 가벼운 busybox syslogd를 유지하는 것이 좋다.
- Log-shipper를 사용하지 않는다면 syslogd는 중지하고, journald 만 로그를 저장하고, syslog로 forward 하지 않도록 설정한다.
- Journald는 디렉토리 위치에 따라 디스크(/var/log/journal/), 메모리(/run/log/journal/)로 남길 수 있는데, 일반적인 경우 메모리로 하고, log rotate 적절히 설정을 하여 /run tmpfs 의 공간이 부족하지 않도록 한다. /run 영역이 부족하면 다른 프로그램의 동작도 문제가 발생할 수 있다.

Yocto 에서 journald 를 포함한 systemd 설정은 다음에 정리하기로 한다.

## Journalctl 사용 예

Journalctl 을 이용하여 로그를 볼때 자주 사용하는 패턴을 정리한다. 아래 옵션을 조합하여 사용할 수 있다.

- 로그 상의 부팅 이력 확인 하기
  - 특정 부팅 시점 이후를 확인하기 위하여 먼저 부팅 이력을 확인해 보기

```shell
$ journalctl --list-boots
-7 b3632d11641e40be8eec8b30d66859fe Wed 2022-03-30 13:30:51 KST—Wed 2022-03-30 14:55:40 KST
...
-1 7170362d223b4b829646e67ed618a875 Fri 2022-05-06 15:57:36 KST—Fri 2022-05-06 16:12:17 KST
 0 4d58600054b44d348e329361d91ef1c5 Fri 2022-05-06 16:14:27 KST—Fri 2022-05-27 13:56:58 KST
```

- 특정 부팅 이후의 로그 보기

```shell
# 이번 부팅 이후의 로그 보기
$ journalctl -b

# 이전 로그 보기 -2 는 위 --list-boots 에서의 앞의 번호
$ journalctl -b -2
```

- 특정 priority의 로그만 보기
  - 우선 순위
    - 0, emerge: Emergency
    - 1, alert: Alert
    - 2, crit: Critical
    - 3, err: Error
    - 4, warning: Warning
    - 5, notice: Notice
    - 6, info: Informational
    - 7, debug: Debug

```shell
# Alert level 이상만 보기
$ journalctl -p 1
$ journalctl -p alert

# Critical level 만 보기
$ journalctl -p 2..2
```

- JSON 형식으로 로그 보기
  - Text 보다 더 자세한 로그를 볼 수 있다.

```shell
$ journalctl -o json
$ journalctl -o json-pretty

{
        "_COMM" : "sshd",
        "_SYSTEMD_SLICE" : "user-1000.slice",
        "_SYSTEMD_UNIT" : "session-2.scope",
        "SYSLOG_PID" : "9247",
        "_GID" : "0",
        "MESSAGE" : "fatal: recv_rexec_state: buffer error: incomplete message",
        "_UID" : "0",
        "_SYSTEMD_SESSION" : "2",
        "SYSLOG_FACILITY" : "4",
        "_SYSTEMD_USER_SLICE" : "-.slice",
        "_AUDIT_SESSION" : "2",
        "_PID" : "9247",
        "_SOURCE_REALTIME_TIMESTAMP" : "1651816413129388",
        "PRIORITY" : "2",
        "__REALTIME_TIMESTAMP" : "1651816413129435",
        "_BOOT_ID" : "d379140462bd44b1a0742211ae80c5cd",
        "SYSLOG_IDENTIFIER" : "sshd",
        "__MONOTONIC_TIMESTAMP" : "1519765636",
        "_SELINUX_CONTEXT" : "unconfined\n",
        "_HOSTNAME" : "humminglab",
        "_SYSTEMD_INVOCATION_ID" : "b7e6f7a587064a5eaadb7e4f70e59ab8",
        "_SYSTEMD_OWNER_UID" : "1000",
        "_AUDIT_LOGINUID" : "1000",
        "_SYSTEMD_CGROUP" : "/user.slice/user-1000.slice/session-2.scope",
        "_MACHINE_ID" : "f705648c1b04425db65d5d1166c5ba16",
        "_CAP_EFFECTIVE" : "3fffffffff",
        "__CURSOR" : "s=947155d24c154008b7cfe329795bc58b;i=2710f88;b=d379140462bd44b1a0742211ae80c5cd;m=5a95c884;t=5de517a89eadb;x=75fabe2f188d11>
        "_TRANSPORT" : "syslog"
}
```

- 검색 필터
  - 위 JSON 의 key value로 검색 가능

```shell
$ journalctl _SYSTEMD_UNIT=apache2.service
# _SYSTEMD_UNIT 는 -u 옵션과 동일
$ journalctl -u apache2
```

- 특정 날짜 이후로 보기

```shell
$ journalctl -S today
$ journalctl -S yesterday
# 특정 시간 이후
$ journalctl -S "2022-05-14 00:00:00"
# 2일전
$ journalctl -S -2d
```

- 신규 Journal을 계속보기

```shell
$ journalctl -f
```

- 최신 순으로 보기

```shell
$ journalctl -r
```

이외에도 grep 패턴검색 등 도움말을 보면 유용한 것들이 많다.

### 마무리

Log-shipper를 사용하여 별도의 로그 서버에서 로그를 수집하는 것이 아니라면,
embedded linux 경우라도 journald를 적용하면 개발이나 디버깅 단계에서 시스템 로그를 적절한 필터를 설정하여 모니터링 할 수 있어,
syslogd를 사용하는 것 보다는 더 유용한 것 같다.
