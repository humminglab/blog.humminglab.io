---
title: "TLS/암호 알고리즘 쉽게 이해하기(4) - Block Cipher Mode"
date: "2022-02-28T19:00:00+09:00"
lastmod: "2022-04-25T20:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Cryptography, AES, HMAC"]
categories: ["Security"]
series: ["TLS/암호 알고리즘 쉽게 이해하기"]
---

이전 글의 [Block Cipher(블럭암호)]({{< ref "posts/tls-cryptography-3-block-cipher">}}) 암호화 방법을 그대로 사용하기에는 몇가지 문제가 있다
(아래 내용에서는 AES로 표기하나, 다른 블럭암호 방식에 공통적인 사항이다).

우선 첫번째 문제는 공격자가 암호키를 몰라도 암호 블럭을 순서를 바꾸어나 다른 내용으로 바꿀 수 있다는 것이다.
AES 암호의 경우 128bits(16bytes) 단위로 암호화 되는데, 예를 들어 다음과 같은 거래 정보를 암호화 한다고 해보자.

```C
struct {
    char from[16];
    char to[16];
    char amount[16];
} transaction;

struct transaction tx = {
    .from = "Alice",
    .to = "Bob",
    .amount = "100",
};
```

이를 AES로 암호를 하게 되면 16bytes 3개의 암호 블럭이 생성된다([패딩]({{< ref "posts/tls-cryptography-3-block-cipher#Padding" >}})을 하지 않는 경우). 만일 공격자가 이 블럭 순서 1,2,3을 2,1,3 으로 바꾸게 되면 from, to 가 바뀌어 거래가 반대로 되어 버린다.
또는 이전의 다른 거래내역에서 금액 부분을 저장해 두었다가 이것으로 교체하게 되면 거래 금액이 달라지게 된다.

두번째 문제는 재전송 공격이 가능하다는 것이다. 공격자가 위 메시지를 전체를 저장해 두었다가 다시 전송하게 되면 2번의 거래가 이루어 지게 된다.

다른 것들도 있지만 우선은 이 두가지 문제만 고려해보자.

첫번째 문제를 막을 수 있는 방법은 1,2,3 세개의 블럭을 서로 연결 시키는 방법이다. 예를 들어서 1,2,3 순번을 데이터와 같이 암호 시에 추가할 수 있다.
두번째 문제를 해결하기 위하여는 매 트랜젝션 마다 임의의 난수를 생성하여 이를 같이 암호화 하는 방법이 있다. 이렇게 하면 동일한 데이타를 암호화 하더라도 매번 다른 결과가 나오게 된다.

이와 같은 운용 모드로 다음과 같은 것이 있다.

- AES-ECB (Electronic Code Book)
- AES-CBC (Cipher Block Chaining)
- AES-CTR (Counter)

아래 설명에 있는 그림은 모두 [Wikipedia - Block cipher mode of operation](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation) 에 있는 그림이다.
조금 더 자세한 설명은 위키피디아 페이지를 참고할 수 있다.

위키피디아나 다른 문서를 보면 AES-CFB(Cipher FeedBack), AES-OFB(Output FeedBack) 등 다른 방법도 설명하고 있으나, 해당 방식들은 책에서만 있지 실사용은 거의 없는 방법들이다. 여기에서는 이들 방식은 언급하지 않는다.

## AES-ECB (Electronic Code Book) {#aes-ecb}

ECB 모드는 모드라고 말하기가 그렇다. 그냥 아무것도 하지 않고 AES 암호화 / 복호화를 하는 것이다.
실제로는 이와 같은 모드를 사용할 일은 거의 없다고 보아도 된다.

{{< figure src="1202px-ECB_encryption.svg.png" width="600px" height="auto">}}

{{< figure src="1202px-ECB_decryption.svg.png" width="600px" height="auto">}}

AES-128 로 ECB 모드를 암호화 하는 예를 들어 보자. 테스트는 리눅스 환경에서 실행하여야 한다.

실제 암호화 되는 패턴을 보면 다음과 같다.

- dd 를 이용하여 2개의 AES 블럭이 되는 36bytes 데이타를 생성한다. 이를 8진수 020 (십육진수 0x10) 으로 tr 로 변경하여 input.bin 으로 저장한다.

```shell
$ dd if=/dev/zero  bs=16 count=2 | tr "\0" "\020" > input.bin
$ hexdump -v input.bin
0000000 1010 1010 1010 1010 1010 1010 1010 1010
0000010 1010 1010 1010 1010 1010 1010 1010 1010
```

- 이를 openssl CLI를 이용하여 AES-128 ECB 모드로 암호화 한다.
  - AES-128 은 128bit 키를 사용하고, 여기에서는 hexdecimal 로 "000102030405060708090a0b0c0d0e0f" 로 입력하였다.
  - '-nopad' 로 추가적인 패딩을 하지 않도록 함.

```shell
$ openssl aes-128-ecb -e -in input.bin -K 000102030405060708090a0b0c0d0e0f -nopad | hexdump -v
0000000 4f95 f264 e8e4 9e6e 82ee 02d2 6816 9948
0000010 4f95 f264 e8e4 9e6e 82ee 02d2 6816 9948
```

- 결과를 보면 0x10 으로 채워진 2개의 블럭은 암호화시 동일한 결과를 가진다.

- '-nopad' 옵션을 빼고 암호화를 하면 다음과 같다.
  - 암호화 결과가 동일한 한개의 블럭이 더 추가되었는데, 이유는 데이타가 16bytes로 나누어 떨어지는 경우 0x10 으로 추가 한블럭의 padding이 추가 되기 때문이다. 관련 설명은 이전글 [패딩]({{< ref "posts/tls-cryptography-3-block-cipher#Padding" >}}) 을 참고할 수 있다.

```shell
$ openssl aes-128-ecb -e -in input.bin -K 000102030405060708090a0b0c0d0e0f | hexdump -v
0000000 4f95 f264 e8e4 9e6e 82ee 02d2 6816 9948
0000010 4f95 f264 e8e4 9e6e 82ee 02d2 6816 9948
0000020 4f95 f264 e8e4 9e6e 82ee 02d2 6816 9948
```

- 암호화 한것을 다시 복호화 해보면 정상적으로 복호화 되는 것을 확인할 수 있다.

```shell
$ openssl aes-128-ecb -e -in input.bin -K 000102030405060708090a0b0c0d0e0f | openssl aes-128-ecb -d -K 000102030405060708090a0b0c0d0e0f |
 hexdump -v
0000000 1010 1010 1010 1010 1010 1010 1010 1010
0000010 1010 1010 1010 1010 1010 1010 1010 1010
```

## AES-CBC (Cipher Block Chaining) {#aes-cbc}

AES-ECB 방식에 다음과 같은 2가지 사항이 추가되었다.

- IV(Initialization Vector)라는 초기 랜덤값을 추가하여 동일한 데이타에 대해서도 다른 암호화 결과가 나오도록 함
- 앞 블럭의 암호화 결과를 다음 블럭을 암호화 할때 XOR 하여 체인으로 연결

{{< figure src="1200px-CBC_encryption.svg.png" width="600px" height="auto">}}

{{< figure src="1200px-CBC_decryption.svg.png" width="600px" height="auto">}}

XOR의 경우 동일한 XOR를 반복하는 경우 다시 원래의 데이터가 되는 특성이 있어, 이를 사용한 것이다. 암호화 시에는 먼저, 복호화 시 암호화 절차와 반대로 뒷 부분에 XOR를 해준다.

통신 시, 클라이언트나 서버, 또는 둘 모두 에서 생성한 랜덤값을 IV로 사용하여 통신을 하게되면, 동일한 데이타를 보내도 다른 암호 결과값이 되고, 데이타 순서가 바뀌는 경우에도 정상적으로 복호화 되지 않아 처음에 언급한 두 가지 문제를 해결할 수 있다.

OpenSSL로 암호화 되는 패턴을 보면 다음과 같다.

- IV를 0 으로 해서 실행해보면 첫 블럭은 AES-ECB 방식을 사용한 것과 동일하다. 하지만 두번째 블럭은 앞 블럭의 암호화 결과가 XOR 되어 암호화 결과가 다른 값이 된다.

```shell
$ openssl aes-128-cbc -e -in input.bin -K 000102030405060708090a0b0c0d0e0f -nopad -iv 00000000000000000000000000000000 | hexdump -v
0000000 4f95 f264 e8e4 9e6e 82ee 02d2 6816 9948
0000010 93db 8ae4 f2e2 6326 3ce6 38df cd70 a808
```

하지만 이 방식은 다음과 같은 단점이 있다.

- 구조적으로 병렬화가 되지 않아 성능이 떨어진다. 암호화시 앞 블럭의 결과가 나와야만 다음 블럭을 암호화 할수 있으므로 멀티 프로세서를 사용한 병렬화가 불가능하다.
- 패딩과 복호화 시 XOR 가 뒤에 있는 구조로 인하여, 패딩 오라클 공격에 취약할 수 있다.

오라클 공격에 대해서는 아래 첫번째 블로그를 보면 이해하기 쉽게 원리가 설명되어 있고, 두번째 링크에서는 이를 기반으로한 다양한 공격 방법들에 대해서 찾아 볼 수 있다.

- [오라클 패딩 공격 기초 설명](https://bperhaps.tistory.com/entry/%EC%98%A4%EB%9D%BC%ED%81%B4-%ED%8C%A8%EB%94%A9-%EA%B3%B5%EA%B2%A9-%EA%B8%B0%EC%B4%88-%EC%84%A4%EB%AA%85-Oracle-Padding-Attack)
- [Lucky 13, BEAST, CRIME,... Is TLS dead, or just resting?](https://www.ietf.org/proceedings/89/slides/slides-89-irtfopen-1.pdf)

참고로 AES-CBC 모드는 데이타 검증을 위한 HMAC 과 같이 사용하면 이와 같은 공격에 대한 방어를 할 수 있고, TLS 1.2 에서도 이를 사용하고 있다.
하지만 TLS 1.3 에서는 AES-CBC 가 제외 되었고, AES-GCM 를 사용하여야 한다. 이는 나중에 MAC 관련된 부분에서 다시 설명키로 한다.

## AES-CTR (Counter)

CTR 모드는 CBC 모드와 달리 AES를 이용하여 데이타를 암호화 하는 것이 아니라, Nonce 와 Counter 값을 암호화 후, 이의 결과 값과 평문을 XOR 로 암호화 한다.

참고로 Stream 암호화에서도 다시 언급하겠지만, 완전한 난수열와 XOR를 한 데이타는 난수열을 알지 못하는 이상 어떤 방식으로든 푸는 것이 불가능하다.

{{< figure src="1202px-CTR_encryption_2.svg.png" width="600px" height="auto">}}
{{< figure src="1202px-CTR_decryption_2.svg.png" width="600px" height="auto">}}

이와 같은 방식으로 하면 다음과 같은 장점이 생긴다.

- CBC와는 달리 이전 블럭과 직접적인 상관 관계가 없어 병렬화가 가능하다.
- 블럭 암호화이기도 하지만 그림을 잘 보면 스트림 암호화이기도 하다. (Nonce || Counter)를 암호화한 난수열로 plaintext가 암호화 되는 stream 암호화이기도 하다.
- 스트림 암호이므로 별도의 패딩이 필요없다. 필요한 바이트만 암호화/복호화 하면 된다.

동작 확인은 다음과 같이 해볼 수 있다.

```shell
$ openssl enc -aes-128-ctr -in input.bin -K 000102030405060708090a0b0c0d0e0f -iv 00000000000000000000000000000000 | hexdump -v
0000000 b1d6 272b 9f97 924b 5f7f 7291 d8b1 69c8
0000010 5663 8503 d085 0ea4 6b59 f3ad e475 1a3d

$ openssl enc -aes-128-ctr -in input.bin -K 000102030405060708090a0b0c0d0e0f -iv 00000000000000000000000000000001 | hexdump -v
0000000 5663 8503 d085 0ea4 6b59 f3ad e475 1a3d
0000010 c659 4397 8b89 9cb6 99f3 786a 9170 8da0
```

위의 결과를 보면 IV가 0인 경우의 두번째 블럭(counter가 1이 되는)과 IV가 1인 경우의 첫번째 블럭이 동일하게 암호화 된다. OpenSSL 에서는 IV $\oplus$ Counter 와 같이 XOR 하여 이를 키로 암호화 하는 것을 확인할 수 있다.

아래와 같이 AES 블럭 단위인 16바이트가 아니어도 암호화 가능하다.

```shell
$ dd if=/dev/zero  bs=8 count=1 | tr "\0" "\020" > input2.bin
$ openssl enc -aes-128-ctr -in input2.bin -K 000102030405060708090a0b0c0d0e0f -iv 00000000000000000000000000000000 | hexdump -v
0000000 b1d6 272b 9f97 924b
```

## 정리

이상으로 블럭 암호화 방식의 운용 모드에 대해서 정리해 보았다.

이 방식들은 처음에 언급한 것과 같이 암호화한 데이타의 조작이나, 재사용을 방지하기 위한 방법이다. 하지만 이 방식은 정상적인 데이터 인지는 확인할 수 없다. Hash 등을 이용한 체크섬이 추가된다면 이에 대한 보완이 될 것이다. 암호화에서는 key가 추가된 hash 방식인 HMAC(Hash-based Message Authentication Code)와 같이 사용한다. 이 부분은 다음에 설명키로 하고 블럭암호화 모드는 이것으로 마무리한다.
