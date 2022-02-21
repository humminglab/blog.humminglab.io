---
title: "TLS/암호 알고리즘 쉽게 이해하기(3) - Block Cipher"
date: "2022-02-21T20:00:00+09:00"
lastmod: "2022-02-21T20:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Cryptography", "TLS", "OpenSSL", "AES", "DES", "3DES"]
categories: ["Security"]
series: ["TLS/암호 알고리즘 쉽게 이해하기"]
---

암호 알고리즘은 암호화/복호화 작업을 동일한 키 하나로 사용하는지, 아니면 암호화하는 키(공개키)와 복호화 하는 키(개인키)를 따로 사용하는지에 따라 각각 대칭키(Symmetric Key), 비대칭키(Asymmetric Key) 방식으로 나뉜다.

{{<mermaid>}}
graph TD;

subgraph Symmetric Key
A1([평문]) --> B1[암호화 모듈] --> C1([암호문])
K1([키]) --> B1

C2([암호문]) --> B2[복호화 모듈] --> A2([평문])
K1 --> B2
end

subgraph Asymmetric Key
A3([평문]) --> B3[암호화 모듈] --> C3([암호문])
K2([공개키]) --> B3

C4([암호문]) --> B4[복호화 모듈] --> A4([평문])
K3([개인키]) --> B4
end

style B1 fill:#FFF9C4
style B2 fill:#FFF9C4
style B3 fill:#FFF9C4
style B4 fill:#FFF9C4
{{</mermaid>}}

대칭키는 암호화 할때 일정 블럭 크기(예를 들어 128bits)를 한번에 암호화 할지, 아니면 비트/바이트 단위로 암호화 하는지에 따라서 블럭 암호(Block Cipher), 스트림 암호(Stream Cipher) 로 구분한다.

여기에서는 블럭 암호 방식에 대해서 설명한다.

## 블럭 암호 개요

비대칭키 방식은 역변환을 하려면 수십년, 수백년 이상 소요되어 실제적으로 역변환이 불가능한 수학적 이론을 바탕으로 구현되어 있지만,
대칭키인 블럭암호는 이리저리 뒤섞거나(전치), 연산이나 코드북을 이용해서 다른 값으로 변경(치환) 하는 작업을 복잡하게 반복하여 만들어낸 암호화 방법이다.

블럭 암호의 대표적인 [DES](https://en.wikipedia.org/wiki/Data_Encryption_Standard), [AES](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard) 방식은 위키피디아의 그림(아래 참고)과 같이 복잡한 연산을 수십번의 반복(라운드)을 통해 구현된다.

| {{< image src="https://upload.wikimedia.org/wikipedia/commons/thumb/6/6a/DES-main-network.png/500px-DES-main-network.png" width="200px" height="auto" caption="DES의 반복적인 라운드 구조" >}} | {{< image src="https://upload.wikimedia.org/wikipedia/commons/5/50/AES_%28Rijndael%29_Round_Function.png" width="300px" height="auto" caption="AES 라운드 함수의 구성" >}} |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

입력 키와 암호문 관계를 모호하게 하는 혼동(Confusion)과 평문 입력의 작은 변경이 암호문의 변경과 관련성을 없애는 확산(Diffusion)을 통하는 방식으로,
수학자, 암호학자들에 의하여 이 암호화를 풀어내는 것이 불가능에 가깝다고 평가된 것들이다.

암호화 모듈을 개발하지 않는다면 블록암호화 방법의 내부를 굳이 이해하려고 노력할 필요는 없다. 다만 각 암호 마다의 특징만 알고 있으면 사용에는 큰 문제가 없다.

| 구분             | DES             | 3DES            | AES                              |
| :--------------- | :-------------- | :-------------- | :------------------------------- |
| Key sizes(bit)   | 56              | 168             | 128 / 192 / 256                  |
| Block sizes(bit) | 64              | 64              | 128                              |
| 구조             | Feistel network | Feistel network | Substitution–permutation network |
| 라운드 횟수      | 16              | 48              | 10 / 12 / 14                     |
| 공개년도         | 1977            | 1981            | 1998                             |

## DES (Data Encryption Standard, FIPS PUB 46)

DES는 상당히 오래전에 만들어진 것으로 56bit의 짧은 키 길이로 현재는 더 이상 안전하지 않은 방식이다. 키 길이를 64bit로 표기된 경우도 있는데, 이는 parity bit 로 8bit가 추가된 것으로 유효한 키는 56bit 이다.

지금은 DES를 무차별 대입 공격(Brute-Force Attack)으로 특정 장비로 수 십시간 내에 풀 수 있어, 짧은 시간의 안전성을 보장하는 경우가 아니라면 사용하지 않는 것이 좋다.
[TLS 1.2 (2008)](https://datatracker.ietf.org/doc/html/rfc5246#appendix-B) 부터 DES 가 공식적으로 제외되었다.

## 3DES

3DES는 DES의 짧은 키 길이를 보완하기 위하여 각각의 다른키로 3번 DES 를 한 것이라고 보면 된다. 암호화 블럭 크기는 DES와 동일하게 64bit 이다.

2016년 [CVE-2016-2183](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2016-2183), [Sweet32](https://sweet32.info/) attack 으로 64bit 블럭 크기를 사용하는 암호화 방식의 취약점이 공개되어, 대부분의 서버에서 사용이 중단되었고,
[TLS 1.3 (2018)](https://datatracker.ietf.org/doc/html/rfc8446)에서 공식으로 제외되었다.

## AES (Advanced Encryption Standard, FIPS PUB 197)

결과적으로 공식적으로 사용하는 블럭 암호화 중 유일하게 사용가능한 것은 AES 이다.

미국에서 폐쇄적으로 만든 DES에 비하여, AES는 미국 NIST(National Institute of Standards and Technology)에서 개방적으로 표준화를 진행한 것이다.
여러 국가의 연구자들이 제출한 알고리즘 중 [Rijndael](https://csrc.nist.gov/csrc/media/projects/cryptographic-standards-and-guidelines/documents/aes-development/rijndael-ammended.pdf)이 최종적으로 선정되었다.

표준 제정 시 요구 사항 중 하나로 향후 양자 컴퓨팅 공격을 대비한 암호화 키 128, 192, 256 bit 길이 지원이다.
AES는 이를 지원하기 위하여 내부적으로 키 길이에 따라 라운드 횟수가 추가된다.

## 암호화 키 길이

양자 컴퓨팅 공격이 가능한 미래까지도 기밀성을 고려한 데이터라면 256bit 를 사용하는 것이 좋다.
예를 들면 스토리지에 저장되는 파일과 같은 경우는 AES 256 사용을 검토할 수 있다.
일반적인 통신 프로토콜의 세션 암호화와 같은 경우에는 128 bit를 사용하여도 적절할 것이다.

참고로 알고리즘에 따른 암호화 키 길이 비교는 아래와 같다.
AES 128bit는 비대칭키인 RSA 3072bit와 유사한 암호 강도라고 보면 된다.

이 의미는 AES 128bit 인 경우 이를 Bruteforce 공격으로 키를 찾으려면 $2^{128}$ 의 시도를 해보아야 한다는 의미이다(각 1번의 시도에 걸리는 시간은 고려치 않고 횟수만으로 가정한 것이다).
인수분해를 이용하는 RSA 비대칭키 방식은 키를 3072bit를 이용하지만 이를 bruteforce 공격으로 찾는다면 이도 $2^{128}$ 가량의 시도를 해보면 찾을 수 있다는 의미이다.

2021년 기준으로 일반적인 보안 표준에서는 AES 128bit, RSA 2048bit 이상의 사용을 권장하고 있다.

| 계열     | 암호시스템         | 80bit | 128bit | 192bit | 256bit |
| -------- | ------------------ | ----- | ------ | ------ | ------ |
| 인수분해 | RSA                | 1024  | 3072   | 7680   | 15360  |
| 이산대수 | DHKE, DSA, Elgamal | 1024  | 3072   | 7680   | 15360  |
| 타원곡선 | ECDH, ECDSA        | 160   | 256    | 384    | 512    |
| 대칭암호 | AES                | 80    | 128    | 192    | 256    |

## 패딩

블럭 암호화를 하는 경우 암호화 하려는 데이타가 AES block size인 128bit(16bytes)로 딱 맞지 않는 경우라면 어떻게 해야 할까?
단순히 뒤의 공간을 0으로 패딩을 하게 되면 아래 두 데이타는 암호화를 수행하면 동일한 결과를 가지게 되어 구분을 할 수 없게 된다.

```C
uint8_t data1[1] = { 0x55 };
uint8_t data2[16] = { 0x55, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
                      0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00 };
```

이 부분은 [RFC 2315](https://datatracker.ietf.org/doc/html/rfc2315) PKCS#7 에 정의된 패딩방법을 사용한다.
PKCS는 RSA 암호를 만든 [RSA 회사](https://www.rsa.com/)에서 정의한 암호화 표준으로 [Public-Key Cryptography Standards(PKCS)](https://web.archive.org/web/20061209135809/http://www.rsasecurity.com/rsalabs/node.asp?id=2124) #1 ~ #15 까지 있다.

이 방법은 남는 공간을 남은 개수 만큼의 값으로 채워주는 방식이다.
예를 들어 15bytes의 데이타는 마지막 1바이트가 0x01 로 채워지고, 14bytes 데이타는 마지막 2바이트가 0x02로 채워진다.
만일 블럭의 끝이 16bytes로 나누어 떨어진다면, 하나의 블럭을 더 추가하여 모두 0x10 으로 패딩하여 구분한다.

위와 같이 하면 padding 으로 데이타의 끝 부분을 알 수 있게 된다.

참고로, 암호화에서는 PKCS #7 에 정의한 패딩방식 이외에도 다른 방식들도 사용한다.
관련 방식들은 [Wikipedia Padding](<https://en.wikipedia.org/wiki/Padding_(cryptography)>)에서 찾아볼 수 있다.

## 마무리

이상으로 블럭 암호 방법의 특징에 대해서 정리하고, 다음 글인 블럭암호의 운용 모드에서 실제 사용 방법도 같이 설명한다.
