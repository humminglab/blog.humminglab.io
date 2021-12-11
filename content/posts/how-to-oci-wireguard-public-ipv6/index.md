---
title: "OCI, WireGuard 로 무료 공인 IPv6 주소 사용하기"
date: "2021-12-10T09:00:00+09:00"
lastmod: "2021-12-11T09:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["OCI", "WireGuard", "IPv6"]
#featuredImage: "oci-wireguard.png"
#featuredImagePreview: "oci-wireguard.png"
categories: ["Development"]
---

IPv6 서비스를 개발하거나 시험하려는 경우 국내에서 기업용 인터넷이 아닌 경우에는 IPv6 주소를 이용하기가 쉽지 않다.
IPv4 주소만 제공하는 가정용 인터넷이나 LTE 망에서
[Oracle Cloud Infrastruct(OCI)](https://www.oracle.com/kr/cloud/) Free Tier 를 사용하여 무료로 공인 IPv6 망을 구성하는 방법을 설명한다.

{{< image src="oci-wireguard.png" width="600px" height="auto" caption="시스템 구성">}}

## 다른 방법들

공인 IPv6 를 할당받는 가장 쉬운 방법은 IPv6 [TunnelBrokwer](https://datatracker.ietf.org/doc/html/rfc3053) 를 이용하는 것이다.
미국 ISP 업체인 Hurricane Electric 에서 제공하는 Tunnel Broker는 아래 주소로 접속하여 가입하면 무료로 IPv6 주소를 할당 받아 사용할 수 있다.

- https://tunnelbroker.net/

이곳은 [6in4](https://en.wikipedia.org/wiki/6in4) tunnel 방식인데, 사용하려는 NAT 환경 에서는 문제가 있어 포기하였다. 관련 사항은 [HE FAQ](https://ipv6.he.net/certification/faq.php) 에서 확인할 수 있다.

[Wikipedia TunnerBroker 서비스 목록](https://en.wikipedia.org/wiki/List_of_IPv6_tunnel_brokers) 중 [6project](https://6project.org/) 와 같은 경우는 OpenVPN을 제공한다고 하여 시도는 해보았으나, 국내 신용카드로는 Donation 결제가 안되어 이곳도 포기하였다.

그래서 2개 까지 무료 인스턴스를 제공하는 [OCI](https://www.oracle.com/kr/cloud/)의 가상머신에 [WireGuard](https://www.wireguard.com/)
VPN 을 이용하여 공인 IPv6 망을 구성하기로 하고, 이 설정 방법을 정리한다.

## 개요

설정은 다음과 같은 절차로 수행한다.

- VM instance 설치하고, IPv6 할당하기
- Wiregaurd 설치
- IP Masquerade로 가상 IPv4, IPv6 VPN tunneling 설정
- Macbook에 WireGuard Peer 설치
- Private IPv6 tunneling 동작 확인
- Public IPv6 tunneling으로 설정 변경
- Public IPv6 동작 확인

## OCI Ubntu 설치

OCI Compute VM 에 Ubuntu 20.4 를 설치하다. OCI의 경우 IPv6 가 자동으로 할당되지 않아 설치 후 VCN(Virtual Cloud Networks)에 IPv6를 할당하여야 한다.

설치 과정은 다음과 같은 문서를 참고할 수 있다.

- [Free Tier: Install Apache and PHP on an Ubuntu Instance](https://docs.oracle.com/en-us/iaas/developer-tutorials/tutorials/apache-on-ubuntu/01oci-ubuntu-apache-summary.htm)
- [Enable IPv6 on Oracle Cloud Infrastructure & Asiign it to CentOS](https://blog.51sec.org/2021/09/enable-ipv6-on-oracle-cloud.html)

다음과 같은 절차로 설치한다.

- **Compute** -> **Instances** -> **Create Instance** 로 무료 인스턴스를 생성한다.
  - Image는 **Canonical Ubuntu 20.04** 로 변경하여 설치한다.
- **Networking** -> **Virtual Cloud Networks** 에서 VCN의 **CIDR Blocks**를 선택하여 IPv6 Subnet을 추가한다.
  - **Add IPv6 CIDR Block** 으로 IPv6 block을 한다. VCN에 `2603:cafe:cafe:ca00::/56` 와 같이 public 대역을 할당 받을 수 있다.
- VCN 내의 Subnet 항목에도 Edit를 선택하여 **Enable IPv6 CIDR Block** 를 선택하고, 00~FF 사이의 값을 설정하여 해당 subnet에 /64 subnet을 등록한다. 이번 예에서는 01 을 설정하여 `2603:cafe:cafe:ca01::/64` 와 같은 subnet을 구성하였다.
- Subnet의 **Security Lists** 항목을 선택하여 Ingress, Egress Rules에 각각 모든 IPv6 를 허용하도록 설정한다. 필요하다면 IPv6 주소나 프로토콜 중 허용 범위를 제한할 수도 있다.
  - CIDR, ::/0, All Protocols
- **VCN** -> **Route Tables** -> route rules 항목에 IPv6 도 추가한다.
  - IPv6, Internet Gateway, ::/0
- **Compute** -> **Instances** -> Instance -> **Attached VNIC** -> **IPv6 Address** 항목에서 생성한 Ubuntu Instances 에 IPv6 주소를 할당한다.
  - `2603:cafe:cafe:ca01::1001` 와 같이 적절한 IP를 설정한다.

위와 같은 과정을 거치면 IPv4, IPv6가 설정된 intance를 생성할 수 있다. 세부 과정은 위 링크를 참고하면 도움이 된다.

절차를 정리해 보면 다음과 같다.

- Ubuntu 가상 머신을 만들기
- VCN IPv6 /56 block 할당
- Subnet에도 /64 block 할당
- 네트워크 라우터와 Security rule에 IPv6 설정
- Ubuntu 가상 머신에 subnet 내의 IPv6 주소 할당

## IPv6 사설망 터널링 설정

이제 부터는 Ubuntu machine에 WireGuard를 설치하고 IPv4, IPv6 VPN 터널링을 설치한다.
Ubuntu VM에 IPv6 주소는 `2603:cafe:cafe:ca01::1001` 와 같이 하나의 주소가 설정되어 있고, IPv6 도 IPv4와 같은 방식으로 NAT 방식으로 이용한다.

설정 과정은 다음 문서를 참고할 수 있다.

- [How To Set Up WireGuard on Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-wireguard-on-ubuntu-20-04)

위 설치 과정에서는 ufw linux filewall을 설정하지만, OCI에서는 iptables로 기본 패킷 필터링이 설정되어 있어, ufw를 이용치 않고, 그냥 iptables로 설정을 변경한다.

다음과 같은 절차로 설치한다.

- APT 로 wireguard를 설치한다.

```shell
$ sudo apt update
$ sudo apt install wireguard
```

- WireGuard는 비대칭키를 생성하여 이를 이용하여 인증 및 암호화를 한다.

  - 생성하는 비밀번호의 읽기 권한을 제한토록 설정

  ```shell
  $ umask 077
  ```

  - Private 키를 생성하고, 이를 이용하여 public key 생성

  ```shell
  $ wg genkey | sudo tee /etc/wireguard/private.key
  $ sudo cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key
  ```

- VPN 터널링은 사설 IP를 이용하므로 각각을 임의로 다음과 같이 사용한다. IPv6 사설 IP 생성은 위 링크에 보면 나와 있으나 fd 로 시작하는 적당한 subnet을 선택하여 사용하면 된다.

  - `10.8.0.0/24`
  - `fd4f:c479:8c81::/64`

- 다음과 같이 /etc/wireguadd/wg0.conf 로 파일을 생성한다.
  - Address는 Wireguard 서버쪽의 주소와 서브넷이다.
  - SaveConfig는 설정이 변경된 경우 자동 저장토록 한다.
  - PostUp, PreDown을 이용하여 wireguard가 시작될 때 자동으로 IP Masquerade가 설정되어 NAT 동작이 되도록 한다.
  - ListenPort는 Remote Peer에서 wireguard 연결하는 포트를 설정한다.

```
[Interface]
Address = 10.8.0.1/24
Address = fd4f:c479:8c81::1/64
SaveConfig = true
PostUp = iptables -t nat -I POSTROUTING -o ens3 -j MASQUERADE
PostUp = ip6tables -t nat -I POSTROUTING -o end3 -j MASQUERADE
PreDown = iptables -t nat -D POSTROUTING -o ens3 -j MASQUERADE
PreDown = ip6tables -t nat -D POSTROUTING -o end3 -j MASQUERADE
ListenPort = 51820
PrivateKey = 2ZDTT0ebU4/pyLwg5CKnNNw2oLUcNIpwgjV7eyhtGQ0=
```

- 패킷 필터링 중 적절한 부분을 해제 한다.

  - iptables 를 이용하여 다음과 같이 WireGuard 접속에 사용하는 UDP 51820 포트를 허용하고, Forward 가능하도록 FORWARD 에 등록된 것을 삭제한다.

  ```shell
  $ sudo iptables -L -v
  $ sudo iptables -I INPUT 4 -p udp --dport 51820 -j ACCEPT
  $ sudo iptables -D FORWARD 1
  ```

  - iptables 설정이 제대로 반영된 것을 확인하고, 재부팅 후에도 유지되도록 저장한다.

  ```shell
  $ sudo iptables -L -v
  $ sudo netfilter-persistent save
  ```

- Ubuntu 머신에서 패킷 라우팅이 되도록 /etc/sysctl.conf 에 다음을 추가한다.

```
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```

- 설정이 반영되도록 sysctl 을 실행한다.

```shell
$ sudo sysctl -p
```

- OCI의 subnet에도 UDP 51820 포트가 수신되도록 설정한다.

  - **Networking** -> **Virtual Cloud Netowrks** -> vcn -> subnet -> security list -> **Ingress Rules**
    - CIDR, 0.0.0.0/0, UDP, Dest Port: 51820

- wg0.conf 에 대한 데모을 enable 하고 실행한다.

```shell
$ sudo systemctl enable wg-quick@wg0.service
$ sudo systemctl start wg-quick@wg0.service
$ sudo systemctl status wg-quick@wg0.service
```

위와 같은 과정을 거치면 WireGuard의 기본설정은 마무리 된다.
실제 연결한 PC (Peer) 에서 사용할 비대칭키와 IP 정보를 추가로 설정하면 바로 사용 가능하다.

## MacBook 에서 연결

macOS 용으로는 GUI 앱이 있어 이를 이용하여 사용한다. 다른 OS를 사용하는 경우에도 유사한 과정을 수행하면 된다.

- 다음 주소에서 맥용 앱을 설치한다.

  - https://www.wireguard.com/install/

- 앱을 실행 후 **Add Empty Tunnel** 을 누르면 다음과 같이 설정 창이 뜬다.
  - 앱에서 자동으로 Peer용 private를 생성하고, 자동으로 등록을 해준다.

{{< image src="wireguard-01.png" caption="WireGuard Peer 설정창">}}

- 나머지 설정을 채워넣는다.
  - Address는 맥북에서 사용할 IP목록이다. IPv4, IPv6 각각 1개를 선택하였다. 주소는 서버의 설정에 있는 subnet 내 이어야 한다.
  - Peer의 PublicKey는 서버의 PublicKey 값이다. 위의 설정대로라면 서버 /etc/wireguard/public.key 에 저장되어 있다.
  - AllowedIPs는 VPN으로 터널링할 target IP 목록이다. 여기에서는 IPv6 용 public IP만 터널링 하도록 설정하였다.
  - 만일 IPv4도 터널링 하려면 0.0.0.0/0 을 추가로 적으면, 하단에 'Exclude private IPs' 체크박스가 표시된다. 이를 선택하면 private IP를 제외한 주소만 터널링 된다. 모든 주소를 터널링 시키면 맥북이 사용하는 LAN의 사설 IP도 터널링 되어 정상적인 동작이 되지 않는다.
  - Endpoint에는 서버의 공인 IP와 포트를 적는다.

```
[Interface]
PrivateKey = sFDjqz+39nOR1kASq2AyZjm8eZOm9RIPTSoBqThBDGo=
Address = 10.8.0.2/32, fd4f:c479:8c81::2/128

[Peer]
PublicKey = RXqX80EgZn1xTDjPNUr482qruhEDvzf9dbhmVrKqwhQ=
AllowedIPs = 2000::/4
Endpoint = 131.111.111.111:51820
```

참고로 WireGuard에서는 client/server라고 구분하지 않고, 리모트를 Peer라고 한다.

- 서버의 /etc/wireguard/wg0.conf 에도 맥북을 Peer로 추가한다.
  - Peer PublicKey는 맥북의 public key를 적는다.
  - AllowedIPs는 허용할 IP 주소를 적는다. 맥북의 Interface-Address에 적은것과 동일하게 적으면 된다.

```shell
[Interface]
Address = 10.8.0.1/24
Address = fd4f:c479:8c81::1/64
SaveConfig = true
PostUp = iptables -t nat -I POSTROUTING -o ens3 -j MASQUERADE
PostUp = ip6tables -t nat -I POSTROUTING -o end3 -j MASQUERADE
PreDown = iptables -t nat -D POSTROUTING -o ens3 -j MASQUERADE
PreDown = ip6tables -t nat -D POSTROUTING -o end3 -j MASQUERADE
ListenPort = 51820
PrivateKey = 2ZDTT0ebU4/pyLwg5CKnNNw2oLUcNIpwgjV7eyhtGQ0=

[Peer]
PublicKey = 6mRvCsDgqYqVldlaFABW8NrpFRG3pWFVqBj8xynEGWl=
AllowedIPs = 10.8.0.2/32, fd4f:c479:8c81::2/128
```

- 서버 설정 변경 후 재기동 한다.

```shell
$ sudo systemctl restart wg-quick@wg0.service
```

- 맥북에서 Active를 해보면 다음과 같이 연결되었다고 표시된다.

{{< image src="wireguard-02.png" caption="WireGuard 연결">}}

정상적으로 설정이 되면 다음과 같이 IPv6 로 연결이 된다.

```shell
$ ping6 v6.myip.tf
16 bytes from 2001:19f0:7001:3623:5400:1ff:feb4:ae7a, icmp_seq=0 hlim=54 time=66.299 ms
```

하지만 curl로 연결을 해보면 아래처럼 DNS 에러가 발생한다.

```shell
$ curl -6 'http://v6.myip.tf'
curl: (6) Could not resolve host: v6.myip.tf
```

아래 처럼 직접 IP로 연결 해야만 정상적으로 연결이된다.

```shell
$ host v6.myip.tf
v6.myip.tf has IPv6 address 2001:19f0:9002:1f1c:5400:1ff:fe72:abe5
v6.myip.tf has IPv6 address 2001:19f0:7001:3623:5400:1ff:feb4:ae7a

curl -6 -s 'http://[2001:19f0:9002:1f1c:5400:1ff:fe72:abe5]' -H 'Host: v6.myip.tf'
```

### MAC 에서 IPv6 DNS 문제 수정

위와 같이 DNS query 가 정상적으로 안되는 문제는 macOS 에만 한정된 것으로 아래 링크에서 도움을 받을 수 있다.

- [How to convince macOS to do IPv6 DNS lookups when your only IPv6 address is via a VPN or tunnel of some sort](https://gist.github.com/smammy/3247b5114d717d12b68c201000ab163d)

간단하게 정리하면 맥에서는 물리적인 네트워크 인터페이스가 라우팅 가능한 IPv6 주소를 할당 받지 않는 경우 DNS 로 A 만 찾고, IPv6 주소인 AAAA 를 찾지 않는다.

scutil로 확인 가능하다. IPv6 query 가 되려면 'Request A records, Request AAAA records' 와 같이 보여야 한다.

```shell
scutil --dns
DNS configuration

resolver #1
  nameserver[0] : 8.8.8.8
  nameserver[1] : 1.1.1.1
  if_index : 13 (en10)
  flags    : Request A records
  reach    : 0x00000002 (Reachable)
```

curl의 경우 `getaddrinfo` 를 기본 설정으로 사용하여 `AI_ADDRCONFIG`로 되어 있고, 이로 인하여 A 만 query를 한다.
ping6 는 이를 overwrite 하여 정상적으로 IPv6 주소를 얻게 된다.

이를 해결하기 위하여 수동으로 설정하거나 위 링크에서 제공하는 python script를 등록하여 맥북 wireguard 설정의 PostUp/PostDown 에 등록하여 사용하면 된다.

### IP 구성 확인

네트워크 구성을 보면 아래 그림과 같이 Ubuntu 서버가 NAT 라우터로 동작이 되는 구조이다.

전체적인 라우팅은 다음과 같은 기능을 동작하게 된다.

- iptable은 이용한 IP Masquerade(NAT) 수행
- sysctl 의 라우팅 설정(`net.ipv4.ip_forward`, `net.ipv6.conf.all.forwarding`)

{{< image src="network1.png" caption="사설 IP VPN 네트워크 구성">}}

## 공인 IPv6로 변경

외부 서버와 연결하는 경우라면 IPv6 사설망이어도 문제 없지만,
Server를 실행하고 외부에서 IPv6 로 client 접속을 한다면 public IPv6 주소가 필요하다.

기존 설정을 변경하여 공인 IPv6 가 터널링 되도록 설정한다.

### 공인 IP를 라우팅 하기

공인 IP로 라우팅은 다음과 같이 설정한다. 즉, 외부에서 공인 IPv6 주소인 `2603:cafe:cafe:ca01::2002` 에 접속을 할 수 있도록 한다.

{{< image src="network2.png" caption="공인 IP VPN 네트워크 구성">}}

Internet Gateway에 적절한 라우팅 테이블 설정이 가능하다면 subnet을 구분하여 설정하면 되겠지만,
무료 범주에서는 이런 구조를 사용하지 않는다.

WireGuard가 설치된 Ubuntu 서버에 NDP Proxy를 설정하여 특정 IP에 대해서 Neighbor Discovery 에 대해서 응답하도록 하면 해당 주소의 패킷을 수신할 수 있게 된다.

이 설정을 하려면 다음과 같이 하면 된다.

- sysctl로 IPv6 NDP proxy를 허용

```shell
$ sudo sysctl -w net.ipv6.conf.all.proxy_ndp=1
```

- ip command로 neigobor proxy 수행하기

```shell
$ sudo ip -6 neigh add proxy 2603:cafe:cafe:ca01::2002 dev ens3
```

위와 같이만 설정되면 Internet Gateway에서 `2603:cafe:cafe:ca01::2002` Neighbor Discovery 에 대해서 응답을 하게
되어 패킷을 수신받게 되고, 내부 라우팅 룰에 따라 VPN Peer로 forwarding 된다.

아래 링크는 관련 설정을 정리한 것이다.

- [DigitalOcean, assign public ipv6 to wireguard clients](https://gist.github.com/MartinBrugnara/cb0cd5b53a55861d92ecba77c80ba729)

하지만, OCI 에서 설정을 해보면 등록되지 않은 IP 주소에 대해서는 Neighbor Discovery 를 수행하지 않아, 위와 같이 설정해도 패킷을 수신받지 못한다.

### OCI 에서 설정

패킷을 수신받기 위하여 다음과 같이 설정한다.

- OCI 설정에서 Ubuntu VM에 `2603:cafe:cafe:ca01::2002` IP를 추가 등록. Uubntu는 `2603:cafe:cafe:ca01::1001`과 2개의 IPv6를 가짐

  - **Compute** -> **Instances** -> Instance -> **Attached VNICs** -> vnic -> **IPv6 Address**

- VNIC에서 주소체크를 하지 않도록 수정

  - **Compute** -> **Instances** -> Instance -> **Attached VNICs** -> Edit VNIC -> **Skip source/destination check** 체크 

- Wireguard 중지

```shell
$ sudo systemctl stop wg-quick@wg0.service
```

- /etc/wirdguard/wg0.conf 수정
  - Address를 임의의 주소로 변경. Subnet은 범위를 줄임
  - 기존 IPv6 masquerade 설정 삭제
  - DHCP client를 강제 종료하고, 2603:cafe:cafe:ca01::2002 는 삭제

```
[Interface]
Address = 10.8.0.1/24
Address = 2603:cafe:cafe:ca01::2001/112
SaveConfig = true
PostUp = iptables -t nat -I POSTROUTING -o ens3 -j MASQUERADE
PostUp = killall dhcient
PostUp = ip -6 addr del 2603:cafe:cafe:ca01::2002 dev ens3
PreDown = iptables -t nat -D POSTROUTING -o ens3 -j MASQUERADE
ListenPort = 51820
PrivateKey = 2ZDTT0ebU4/pyLwg5CKnNNw2oLUcNIpwgjV7eyhtGQ0=
```

- 재기동

```shell
$ sudo systemctl start wg-quick@wg0.service
```

- 맥북에서는 Interface의 Address 를 public IPv6 로 변경

```
[Interface]
Address = 10.8.0.2/32, 2603:cafe:cafe:ca01::2002/128
```

이렇게 하면 Internet Gateway에서는 패킷을 전달하게 되고, 나머지는 Ubuntu 내부 라우팅 룰로 전달된다.

## 공인 IPv6 주소 확인

위의 설정으로 하면 다음과 같이 맥북에서 wireshark로 utun 인터페이스를 캡쳐해보면 정상적으로 ping 수신되는 것을 확인할 수 있다.

{{< image src="ping.png" caption="ping 수행 결과">}}

## 정리

DigitalOcean 과 같이 NDP proxy가 가능한 구조면 좀 더 깔끔하게 설정이 되었을 텐데, 이것이 안되어 조금은 설정이 복잡해졌다.

아직은 놔두면 다시 2603:cafe:cafe:ca01::2002 가 할당되어 주기적으로 삭제를 해주어야만 하지만, 급한데로 IPv6 를 테스트 할 수 있는 환경은 구축 완료 하였다.
