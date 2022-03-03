---
title: "TLS/암호 알고리즘 쉽게 이해하기(5) - Stream Cipher"
date: "2022-03-02T18:00:00+09:00"
lastmod: "2022-03-02T18:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Cryptography"]
categories: ["Security"]
series: ["TLS/암호 알고리즘 쉽게 이해하기"]
---

일정 데이터 단위로 암호화를 하는 [블럭 암호]({{< ref "posts/tls-cryptography-3-block-cipher">}})와 비교하여 스트림 암호(Stream Cipher)는 비트 또는 바이트 단위로 암호화를 하는 방식이다.

쉽게 말해서 블럭 암호는 키를 사용하여 (필요하면 IV도 포함해서) 블럭단위로 전치와 치환을 통하여 암호하를 가하는 방식이라고 할 수 있다. 이와 비교하여 스트림 암호는 키와 IV(Initial Vector)로 다양한 연산을 이용하여 난수열을 만들고, 이를 이용하여 평문과 XOR 과정을 통하여 암호화를 하는 것이다. 블럭 암호에서 설명한 [AES-CTR]({{< ref "posts/tls-cryptography-4-block-cipher-mode#aes-ctr-counter">}})와 같은 것이 이와 같은 난수열을 만드는 방법 중 하나이다.

가장 이상적인 스트림 암호는 [True Randump Number Generator(TRNG)]({{< ref "posts/tls-cryptography-2-random">}})를 사용하여 만들어낸 난수열이다. 예를 들어 1MB의 난수열을 만들고, 이를 임의의 방식으로 상대방과 사전에 공유를 하였다고 하자. 이를 이용하면 1MB 까지의 평문을 XOR 하여 전달을 할 수 있고, 난수열을 가지고 있지 않은 어떠한 사람도 데이타 원본을 알 방법이 없다. 문제는 이와 같은 커다란 난수열을 교환하는 것이 쉽지 않다는 것이다. 아래에 설명하지만 그렇다고 한번 사용한 난수열을 다시 재 사용하여 다른 데이타를 암호화 할 수는 없다.

스트림 암호화 과정은 이와 같은 난수열을 키와 IV로 만들어 내는 것이라고 보면 된다.

참고로 IV는 nonce, salt 등으로 말하기도 하는데, 암호 연산에 추가하는 임의의 랜덤값이다. 이를 연산에 추가함으로써 동일한 평문을 동일한 키로 암호화 해도 항상 암호문이 생성된다. 키와는 달리 IV는 외부로 공개되는 값이다.

{{<mermaid>}}
graph TD;

subgraph Stream 암호
A1([평문]) --> B1[XOR] --> C1([암호문])
K1([난수열]) --> B1

C2([암호문]) --> B2[XOR] --> A2([평문])
K1 --> B2
end

style B1 fill:#FFF9C4
style B2 fill:#FFF9C4
{{</mermaid>}}

## Exclusive OR

암호 알고리즘에서 XOR(배타적 논리합, exclusive or) 을 자주 사용한다. 기호로는 $\oplus$ 를 사용한다.

XOR 결과는 아래 진리표와 같이 둘 중 하나의 값만이 1인 경우에만 결과가 1이 된다.

| $x_1$ | $x_2$ | $x_1 \oplus x_2$ |
| ----- | ----- | ---------------- |
| 0     | 0     | 0                |
| 0     | 1     | 1                |
| 1     | 0     | 1                |
| 1     | 1     | 0                |

이 XOR 특성은 다르게 보면 $x_2$ 가 1인 경우 $x_1$의 값이 반전되고,
$x_2$가 0인 경우 $x_1$의 값은 변하지 않는다.

임의의 데이터를 특정 값으로 XOR 한 결과를 다시 그 값으로 XOR 하면 원래 값이 되므로, 이 방식이 stream 암호의 기본적인 방법이다.

$$ P \oplus K = E $$
$$ E \oplus K = P $$

## XOR 의 보안 문제점 {#XOR-security-problem}

XOR는 연산은 간단하면서도 강력하여 암호화 알고리즘에서 자주 사용하지만 주의하여야 할 부분이 있다.

XOR 방식의 스트림 암호화는 절대로 IV(Initial Vertor)를 재사용하면 안된다는 것이다.

예를 들어 평문 A 를 키와 동일 IV로 만들어지는 난수열로 암호화하고, 평문 B 도 마찬가지로 동일한 난수열로 암호화 한다고 할때, 각각의 암호문을 다시 XOR 하면 어떻게 될까?

둘을 XOR을 하게 되면 아래처럼 키는 사라지고 두 평문의 XOR 한 결과가 된다.

$$ (A \oplus KEY) \oplus (B \oplus KEY) = A \oplus B $$

이전 글에서 설명한 [AES-CTR]({{< ref "posts/tls-cryptography-4-block-cipher-mode#aes-ctr-counter">}}) 로 확인을 해보면 다음과 같다.

- 아래 처럼 두개의 평문이 있다. 공격자는 p1.bin은 모르지만 p2.bin 의 내용을 알고 있다고 하자.

```shell
$ dd if=/dev/zero bs=16 count=1 | tr "\0" "\01" > p1.bin
$ hexdump -v p1.bin
0000000 0101 0101 0101 0101 0101 0101 0101 0101
$ dd if=/dev/zero bs=16 count=1 | tr "\0" "\02" > p2.bin
$ hexdump -v p2.bin
0000000 0202 0202 0202 0202 0202 0202 0202 0202
```

- 각각을 AES-CTR 로 동일한 IV를 사용하여 암호화 한 결과는 다음과 같다. 여기서 AES 키는 중요치 않다.

```shell
$ openssl enc -aes-128-ctr -in p1.bin -K 000102030405060708090a0b0c0d0e0f -iv 1234567890abcdef1234567890abcdef | hexdump -v
0000000 a35a b0af b63f 27b8 9cbf c3fd 9376 2c05
$ openssl enc -aes-128-ctr -in p2.bin -K 000102030405060708090a0b0c0d0e0f -iv 1234567890abcdef1234567890abcdef | hexdump -v
0000000 a059 b3ac b53c 24bb 9fbc c0fe 9075 2f06
```

- 두 결과값을 XOR 하면 다음과 같다.

  - p1_enc: a35a b0af b63f 27b8 9cbf c3fd 9376 2c05
  - p2_enc: a059 b3ac b53c 24bb 9fbc c0fe 9075 2f06
  - p1_enc $\oplus$ p2_enc: 0303 0303 0303 0303 0303 0303 0303 0303

- 둘을 XOR 하면 이미 패턴이 보이기 시작한다. 이 결과값을 공격자가 알고 있는 p2.bin 과 XOR를 하면 p1.bin 을 얻을 수 있다.

  - p1_enc $\oplus$ p2_enc: 0303 0303 0303 0303 0303 0303 0303 0303
  - p2.bin: 0202 0202 0202 0202 0202 0202 0202 0202
  - 결과: 0101 0101 0101 0101 0101 0101 0101 0101

블럭 암호도 IV 값을 재사용하면 replay attack 등이 가능할 수 있으나, 스트림 암호의 경우 IV 를 재사용하게 되면 이와 같이 평문이 노출될 수 있는 문제가 발생할 수 있으므로 반드시 주의하여 사용하여야 한다.

이와 같은 문제는 Wi-Fi 의 초기 암호화 방식인 WEP 에서도 실제로 발생하였다. WEP은 RC4 스트림 암호를 사용하였는데, 문제 중 하나가 IV를 24bit 로 사용하는 것이다. IV는 패킷마다 1씩 증가하여 중복되지 않도록 하였으나, 트래픽이 많은 사무실 환경에서는 $2^{24}$ 의 카운트도 몇시간이면 모두 사용하여 다시 초기 값을 재사용하게 된다. 이를 분석적 공격을 이용하여 하루 정도의 트래픽 분석으로 모든 데이타를 복호화 하는 것이 가능하다.

관련된 사항은 아래 글을 참고할 수 있다.

- [WiFi security: history of insecurities in WEP, WPA and WPA2, 2013-08-28](https://security.blogoverflow.com/2013/08/wifi-security-history-of-insecurities-in-wep-wpa-and-wpa2/)

## ChaCha20

스트림 암호는 여러 가지가 있지만, TLS 에서는 거의 [ChaCha20](https://datatracker.ietf.org/doc/html/rfc7539) 만 사용한다고 보면 될 듯 하다.
TLS 전체로 보면 대칭키 암호는 다음과 같은 2개가 주로 사용된다.

- 블럭암호 AES
- 스트림암호 ChaCha20

MAC과 같이 데이타의 무결성을 보장을 포함해서 AES-GCM 과 ChaCha20-Poly1305 두 가지 모드를 사용한다고 보면 된다
(이 부분은 나중에 다시 설명키로 한다).

AES 비교하여 ChaCha20은 연산이 단순하여 4배 정도 빠르다는 장점이 있어, 별도의 AES 하드웨어 엑셀러레이터가 없어도 빠른 속도로 암호화 할 수 있어,
최근에 많이 사용되는 암호화 방식이다.

위키피디아의 [ChaCha](https://en.wikipedia.org/wiki/Salsa20#ChaCha_variant)를 보면 아래 그림과 같이 상대적으로 단순한 XOR, Shift 의 quater-round 연산으로 구성된다.

{{< figure src="500px-ChaCha_Cipher_Quarter_Round_Function.svg.png" width="400px" height="auto" caption="ChaCha quater-round">}}

ChaCha20은 AES-256과 같이 256bit의 키와 128bit의 initial vector를 사용한다.

XOR 스트림 암호이기 때문에 이도 마찬가지로 동일한 IV를 사용하면 아래처럼 평문이 노출될 가능성이 있다.

```shell
$ openssl enc -chacha20 -in p1.bin -K 000102030405060708090a0b0c0d0e0f000102030405060708090a0b0c0d0e0f -iv 1234567890abcdef1234567890abcdef | hexdump -v
0000000 b331 f016 7570 a55a 061b 69c1 f880 3819
$ openssl enc -chacha20 -in p2.bin -K 000102030405060708090a0b0c0d0e0f000102030405060708090a0b0c0d0e0f -iv 1234567890abcdef1234567890abcdef | hexdump -v
0000000 b032 f315 7673 a659 0518 6ac2 fb83 3b1a
```

참고로 예전에 자주 사용하던 RC4 는 2015년에 [RFC 7465](https://datatracker.ietf.org/doc/html/rfc7465)로 사용이 금지되었다.

이상으로 스트림 암호는 마무리한다.
