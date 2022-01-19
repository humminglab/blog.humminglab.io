---
title: "Port Mirroring 기능 지원 저렴한 스위치 - GS108Ev3"
date: "2022-01-19T09:10:43+09:00"
lastmod: "2022-01-19T10:43:00+09:00"
draft: true
authors: ["YSLee"]
tags: ["Network", "Wireshark", "Port Mirroring", "Netgear"]
categories: ["Development"]
---

네트워크 프로토콜을 개발 하다보면 Wireshark 등으로 패킷을 캡쳐하여 분석할 일이 종종 생긴다.
Linux 장치라면 tcpdump나 wireshark을 실행해서 직접 패킷 캡쳐가 가능하지만, 그렇지 않은 경우에는 Network TAP 이나 이더넷 스위치의 port mirroring 기능을 사용하여서 패킷을 캡쳐하여야 한다.
Amazon 등에서 검색해보면 Network Tap 도 $200~300 정도로 비싼 편이고, 포트 미러링와 유사한 기능도 고급 스위치에서만 지원된다.

찾다 보니 Netgear [GS108Ev3](https://www.netgear.com/business/wired/switches/plus/gs108e/)(8포트), [GS105Ev2](https://www.netgear.com/business/wired/switches/plus/gs105ev2/)(5포트) 가 가격도 5~8만원 정도로 비싸지 않고, 포트 미러링 기능이 지원되어 소개한다.

외관은 금속으로 되어 있어 크기에 비하여 무게가 묵직하게 나간다.

{{< figure src="GS108E_productcarousel_hero_image_tcm148-109960.png" width="600px" height="auto" caption="GS108Ev3">}}

구매한 제품은 GS108Ev3 이다. 중간에 숫자 '8'은 포트 개수를 말하고, 'E'는 Enhanced Features 의미로 이 enhanced 기능 중 하나가 포트 미러링이다. 'v3'는 버전 의미로 8포트만 v3이고 5포트는 v2이다.

[GS108Ev3 FAQs](https://kb.netgear.com/25093/GS108Ev3-FAQs)를 보면 v3 에서 아래 처럼 해당 웹브라우저에서 관리를 할 수 있다는 데 무슨 의미인지 잘 모르겠다. v3 의미는 하드웨어 일텐데... 가능한 최신 버전일것 같아 GS108Ev3 를 구매하여 사용중이다.

> Compare to v2, is there any new feature for GS108Ev3?
>
> Yes, staring from v3, GS108E can be managed by web browser IE9 ~ 11, Firefox 26 ~ 29.0.1, Chrome 33.0.1750.117 ~ 35.0.1916.114 m, Safari 10.8.5.

제품은 아래처럼 Gigabit 을 지원한다. 

{{< figure src="gs108ev3-port-status.png" width="800px" height="auto" caption="지원 포트">}}

포트 미러링 설정을 보면 아래와 같이 원하는 source ports 를 destination port로 미러링 설정을 할 수 있다. 이와 같이 설정하면 포트 7, 8번으로 송수신되는 모든 트래픽은 포트 1번으로도 출력이 되어 해당 패킷을 1번 포트에서 캡쳐할 수 있다.

{{< figure src="gs108ev3-port-mirroring.png" width="800px" height="auto" caption="포트 미러링 설정">}}

## 정리

마땅한 장비가 없을때는 ipTime 공유기의 포트미러링을 사용했었다. 
이 기능은 외부포트를 내부포트로 미러링 해주는 기능이라 외부로 나가는 패킷은 캡쳐할 수 있지만 LAN 상의 패킷은 캡쳐를 하지 못한다. 

하지만 이 Netgear 스위치를 이용하면 원하는 포트의 패킷을 쉽게 캡쳐할 수 있어, 다음처럼 유선, 무선 모두 캡쳐 할 수 있는 환경을 구축하여 사용 중이다. 

{{< figure src="network.png" width="600px" height="auto" caption="개발용 네트워크 구성도">}}

- IPTime 유무선 공유기를 hub mode로 설정하여(AP only) 무선랜 시험용으로 사용 
- GS108Ev3 스위치에 AP와 유선으로 시험용 기기를 연결하여 Macbook 으로 포트 미러링
- Macbook에는 USB ethernet 2개를 사용
    - Ethernet 1: 일반 인터넷 사용 
    - Etehrnet 2: GS108Ev3 에 연결하여 ethernet packet 캡쳐 전용 
    - 내장 무선랜: IEEE 802.11 무선랜 패킷 캡쳐 및 무선랜 시험용

이와 같이하여 유선, 무선랜 IEEE 802.11 raw packet, 무선랜의 IP layer packet 과 같이 필요에 따라 적절한 방법으로 캡쳐할 수 있도록 하였다.