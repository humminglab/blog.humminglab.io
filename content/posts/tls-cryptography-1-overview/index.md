---
title: "TLS/암호 알고리즘 쉽게 이해하기(1) - 개요"
date: "2022-02-09T09:00:00+09:00"
lastmod: "2022-02-09T09:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Cryptography", "TLS", "OpenSSL"]
categories: ["Security"]
series: ["TLS/암호 알고리즘 쉽게 이해하기"]
---

HTTPS의 SSL/TLS를 이해하는 것은 생각만큼 쉽지 않다. React, kubernetes 같이 오픈 생태계에서 핫 한 분야들은 다양한 시각으로 설명한 자료들이 많아 여러 방법으로 이해할 수도 있겠지만, 수십여년간 수학자, 암호학자들에게서 다듬어진 암호학에 대해서는 말랑말랑한 자료를 찾기가 쉽지는 않다. 좋은 자료라고 찾아 보아도 읽다 보면 이내 수많은 용어들과 수학 이론에 막혀 버리곤 한다.
그렇다고 제대로된 이해없이 SSL/TLS 나 암호화 알고리즘을 사용하게 되면 작은 실수로 인하여 보안에 심각한 문제를 만들 수도 있다.

여기에서는 일반 개발자를 위한 관점으로 암호 알고리즘과 TLS/SSL을 정리해 보기로 한다.
암호 알고리즘을 사용하는 입장이라면 이의 수학적인 지식에 대하여 모두 이해할 필요는 없다.
다만, 알고리즘이 어떤 원리로 구성 되는지를 이해하고, 이의 사용 방법과 사용해서는 안되는 취약 패턴을 안다면 충분하다.
동작 원리를 이해하는데 필요한 부분은 중학교 수학 수준에서 설명하고, 전체를 조합하여 TLS를 이해하는 것을 목표로 한다.

참고로 개발자는 별도로 암호화 모듈을 만드는 일은 거의 없다고 보면 된다. 테스트 벡터가 충분하지 않은 상태에서 직접 만들어 동등한 암호화 수준을 유지하는 것은 쉬운 일이 아니다. 대부분은 오픈 소스인 [OpenSSL](https://www.openssl.org/) 또는 [Mbed TLS](https://tls.mbed.org/) 와 같이 오래 시간동안 검증된 암호화 모듈을 사용한다. Mbed TLS 의 경우 외부 라이브러리 의존성도 적고, 필요한 모듈만 빌드하여 사용하기도 쉽고, Apache 2.0 license로 상용 적용에도 문제가 없어, 특별한 사양을 요구하는 경우가 아니라면 이를 사용하는 것이 좋다.

## SSL/TLS History

SSL(Secure Sockets Layer)와 TLS(Transport Security Layer)는 SSL/TLS 라고 같이 표기하는데, 이는 버전업 되면서 SSL → TLS로 이름이 변경된 것이다. 다만 TLS 로 이름이 바뀌면서 서로 호환이 되지 않기 때문에 SSL/TLS 로 표기한다. 실제 사용 버전은 다음과 같다.

- SSL 2.0 (1995)
- SSL 3.0 (1996)
- TLS 1.0 (1999)
- TLS 1.1 (2006)
- TLS 1.2 (2008)
- TLS 1.3 (2018)

TLS는 버전업이 되는 이유가 이전의 보안 문제를 해결하거나 강화를 하는 것으로, 버전업이 된다면 이전 버전은 보통 취약점으로 인하여 사용 중단을 권고한다.

2021년 기준으로 보면 다음과 같다.

- SSL ~ TLS 1.1: 보안 문제로 사용 금지([RFC8996](https://datatracker.ietf.org/doc/html/rfc8996) 으로 공식 사용 금지하고, 2020년초 주요 브라우저에서도 사용 중단)
- TLS 1.2: 현재 가장 많이 사용
- TLS 1.3: 주요 브라우저에서 지원 ([링크](https://caniuse.com/tls1-3)에서 브라우저별 지원 여부 확인)

정리하면 현재는 TLS 1.2, TLS 1.3 만 사용한다.
2018년에 TLS 1.3 이 확정 되었지만 글을 쓰는 시점에서 아직까지는 TLS 1.2 가 주로 사용되고 있고, 점차적으로 TLS 1.3 으로 버전업 되고 있는 추세이다. 이 글에서 별도의 언급이 없는한 TLS 1.2를 기준으로 설명한다.

## OpenSSL 개요

OpenSSL의 주요 구성은 크게 3가지로 구분된다.

- **libcrypto** (Crypto Component): AES, RSA, AES 등 여러 암호 알고리즘
- **libssl** (TLS Component): 암호 알고리즘을 제외한 TLS 라이브러리
- **OpenSSL 실행파일** (Application Component): CLI(Command Line Interface)로 직접 실행이 가능한 유틸리티 프로그램

{{< figure src="openssl-architecture.png" width="600px" height="auto" caption="OpenSSL Architecture">}}

앞으로 설명하는 내용에서 실제 예를 드는 경우는 OpenSSL CLI를 사용한다.
프로그램은 Linux의 경우 패키지 매니저를 이용하면 쉽게 설치할 수 있으나, Windows의 경우에는 아래 링크를 참고하여 둘 중 하나의 방법으로 설치하여야 한다.

- 빌드된 프로그램 직접 설치하기: [How To Install OpenSSL on Windows](https://tecadmin.net/install-openssl-on-windows/)
- [Chocolatey로 설치하기](https://community.chocolatey.org/packages/openssl)

## Mbed TLS

Mbed TLS 도 아래 그림과 같이 OpenSSL의 Crypto, TLS component 에 해당하는 부분을 제공한다.

{{< figure src="mbedtls.png" width="600px" height="auto" caption="Mbed TLS Library" >}}

TLS 1.3 에 대해서는 Mbed TLS가 진도가 늦은 편이다. OpenSSL은 v1.1.1(2018년) 부터 지원하지만 MbedTLS은 2021년 12월 17일 릴리즈 한 [Mbed TLS 3.1.0](https://github.com/ARMmbed/mbedtls/releases/tag/v3.1.0) 에서 TLS 1.3 MVP(Minimum Viable Product)라고 TLS 1.3 중 일부분의 기능을 지원한다.
자세한 부분은 [docs/architecture/tls13-support.md](https://github.com/ARMmbed/mbedtls/blob/development/docs/architecture/tls13-support.md) 에서 확인할 수 있고, 간단히 말하면 TLS 1.3 표준 중 client 구현에 필요한 최소 기능만 지원한다고 보면 된다.

[Roadmap](https://developer.trustedfirmware.org/w/mbed-tls/roadmap/) 를 보면 TLS 1.3 전체 기능은 2022년 Q3 까지 구현될 모양이다.

## Cipher Suites

TLS에서는 초기 연결 시 client 에서 지원 가능한 Cipher Suites를 제공하고, 이 중 하나를 Server에서 선택하여 사용한다.

TLS 1.2 에서 보안성을 고려하여 권장하는 목록은 다음과 같다. 암호 강도가 높은 순으로 정리한 것이다.

- TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
- TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
- TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
- TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
- TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
- TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
- TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384
- TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384
- TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256
- TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256

복잡해 보이지만 이들 이름은 아래와 같이 4개 부분으로 나뉘어 진다.

{{< figure src="cipher-suite.png" width="600px" height="auto" caption="Cipher Suite의 구성" >}}

간단한 설명과 사용 가능한 알고리즘은 다음과 같다.

- **Key Exchange/Agreement**: 데이타 암호화를 위한 키를 교환하거나, 합의에 의하여 생성하는 방법
  - RSA, DHE, ECDHE
- **Authentication**: 인증서의 검증 방법
  - RSA, DSA, ECDSA
- **Block/Stream Cipher**: 데이타 암호화 알고리즘. 대칭키 방식으로 바이트 단위로 암호화 하는 스트림 방식이나 블럭 암호화를 사용. Key Exchange/Agreemen로 얻은 비밀키를 사용
  - RC4, 3DES, AES, CHACHA20
- **Message Authentication**: 데이타의 유효성 검증 방법
  - HMAC-SHA384, HMAC-SHA256, HMAC-SHA1, POLY1305

위 Cipher Suites를 표로 정리해 보면 다음과 같다.

| Key Exchange / Agreement | Authentication | Cipher     | Message Authentication |
| ------------------------ | -------------- | ---------- | ---------------------- |
| ECDHE                    | ECDSA          | AES256-GCM | SHA-384                |
| ECDHE                    | RSA            | AES256-GCM | SHA-384                |
| ECDHE                    | ECDSA          | CHACHA20   | POLY1305               |
| ECDHE                    | RSA            | CHACHA20   | POLY1305               |
| ECDHE                    | ECDSA          | AES128-GCM | SHA-256                |
| ECDHE                    | RSA            | AES128-GCM | SHA-256                |
| ECDHE                    | ECDSA          | AES256-CBC | SHA-384                |
| ECDHE                    | RSA            | AES256-CBC | SHA-384                |
| ECDHE                    | ECDSA          | AES128-CBC | SHA-384                |
| ECDHE                    | RSA            | AES128-CBC | SHA-384                |

## 정리

앞으로는 아래와 같은 순서로 주요 암호 알고리즘에 대하여 알아보고, 이를 기반으로 TLS를 설명할 것이다.

- RNG(Random Number Generator)
- Block Cipher
- Block Cipher Mode
- Stream Cipher
- Hash
- 모듈로 연산 / 이산 수학
- Diffie-Hellman Key Exchange
- RSA Public Key
- MAC
- Digital Signature
