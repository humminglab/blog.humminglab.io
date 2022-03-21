---
title: "TLS/암호 알고리즘 쉽게 이해하기(8) - RSA"
date: "2022-03-21T09:00:00+09:00"
lastmod: "2022-03-21T09:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Cryptography, RSA, ECC, PCKS"]
categories: ["Security"]

series: ["TLS/암호 알고리즘 쉽게 이해하기"]
---

지난 번에 설명한 [이산대수]({{< ref "posts/tls-cryptography-6-math">}})를 이용하면 RSA 의 기본원리도 쉽게 이해할 수 있다.

RSA는 이름에 특별한 의미는 없고, 알고리즘을 발명한 사람들(Ron Rivest, Adi Shamir, Leonard Adleman)의 약자를 따서 만든 것이다.

[AES]({{< ref "posts/tls-cryptography-3-block-cipher">}})와는 달리 공개키(public key)와 개인키(private key) 두 벌로 구성된 키를 가지고 있는 비대칭키 암호화 알고리즘이다.

가장 일반적인 사용용도는 다음과 같이 Bob 이 Alice에게 암호 데이타 전달하는 방법이다.

- Alice의 공개키는 공개되어 누구나 알수 있다.
- Bob은 Alice의 공개키를 이용하여 암호화 하여 일반 채널로 전달한다.
- Alice는 자신의 개인키로 복호화 하여 Bob이 보낸 데이타를 얻을 수 있다.

{{< mermaid >}}sequenceDiagram
Participant Alice
Participant Bob
Alice ->> Bob: 공개키(K_pub) 배포
note over Bob: P=본문<br/>E=RSA(K_pub, P)
Bob ->> Alice: 암호화된 데이타(E) 전송
note over Alice: P=RSA(K_pri, E)
{{< /mermaid >}}

이 과정에서 암호화된 데이타를 다른 사람이 도청을 하더라도, Alice 의 private key가 없는 이상 복호화 할 방법은 없다.

반대로 개인키로 암호화 하는 것도 가능하다. 이 경우 공개키로만 복호화를 할 수 있게 된다.
이는 서명 용도로 다음과 같은 방법으로 활용한다.

- Alice는 문서를 SHA-1 등의 hash 알고리즘을 이용하여 짧은 데이터열(digest)를 만들어 낸다.
- 이 digest를 개인키로 암호화 한다.
- Alice는 원본 문서와 암호화한 데이타(서명)을 같이 전달한다.
- Alice의 공개키를 아는 Bob은 받은 데이타의 SHA-1 해쉬 값과, 서명데이타를 Alice의 공개키롤 복호화 해서 둘이 같은지를 확인한다.
- 이 둘이 같다면 원본 문서는 Alice가 만든 것이 맞다고 검증할 수 있다.

{{< mermaid >}}sequenceDiagram
Participant Alice
Participant Bob
Alice ->> Bob: 공개키 배포

note over Alice: P=본문<br/>S=RSA(K_pri, SHA_1(P))
Alice ->> Bob: P와 S를 전달
note over Bob: SHA_1(P) == RSA(K_pub, S)
{{< /mermaid >}}

즉, Alice의 개인키 없이는 암호화(서명)을 할 수 없는 것을 이용하여 디지털 서명을 만들어 낼 수 있다.

## 기본 원리

[Diffie-Hellman]({{< ref "posts/tls-cryptography-7-diffie-hellman">}})와 비슷 하게 RSA는 이산대수 문제의 어려움을 이용하고, 여기에 소인수분해의 어려움을 추가로 사용한다.

기본 원리는 다음과 같다. 관련된 수학적인 내용은 앞의 [이산수학]({{< ref "posts/tls-cryptography-6-math">}})에 설명되어 있으므로 이를 참고하여 보면 된다.

우선 두 소수 p, q를 선택한다. 그리고 p와 q를 곱하여 n을 구한다.

n 은 외부로 공개되는 값으로 p, q 가 아주 큰 소수이면 소인수분해로 p, q를 구하는 것은 실제적으로 불가능하다.

이산수학의 [갈르와체와 확장]({{< ref "posts/tls-cryptography-6-math#갈르와체와-확장">}})를 보면 이와 같이 두 소수의 곱으로 만든 모듈로 n 의 경우 제한된 수를 추려내면 원소 내에서 모듈로 곱하기 연산이 가능하다.
가능한 수의 집합은 n과 서로소인 것으로 결국은 p와 q의 배수 들을 제외하면 되는 것으로 총 개수는 아래처럼 된다.

$$
\Phi(n) = (p - 1) \cdot (q - 1)
$$

앞의 이산 수학을 보면 이 모듈로 n 연산에서 임의의 수를 이 $\Phi(n)$ 개수 만큼 곱해보면(지수연산) 항상 1이 된다.

이것이 오일러의 정리로 다음과 같은 관계가 된다.

$$ k^{\Phi(n)} \equiv 1 \mod n$$

이 식의 양변에 k를 곱해서 다시 써보면 다음과 같이 된다.

$$ k^{\Phi(n)+1} \equiv k \mod n$$
$$ k^{\Phi(n) + 1} \equiv k^{x \cdot \Phi(n) + 1} \mod n $$

두 수를 곱해서 $(x \cdot \Phi(n) + 1)$ 가 되면 되는데(x는 임의의 정수), 이는 $\Phi(n)$ 모듈로 연산이라고 볼수 있다.

이 두개의 키 e와 d를 선정하는데, 이 둘의 관계를 모듈로 연산으로 정리하면 다음과 같은 관계가 되면 된다.

$$ e \cdot d = 1 \mod {\Phi(n)}$$

평문 k가 있을 때 공개키 e로 암호화 한것은 다음과 같다.

$$ E = k^{e} \mod n$$

복호화는 다음과 같이 d 로 할 수 있다.

$$ k = E^{d} \mod n$$

이와 같이 복호화가 되는 이유는 두 식을 합쳐 보면 된다.

$$ E^{d} = (k^{e})^d = k^{e\cdot d} = k^{1 + x \cdot \Phi(n)} = k^1 \cdot k^{x \cdot \Phi(n)} = k^1 = k\mod n$$

식으로 보면 e 로 암호화를 하냐, d 로 암호화를 하느냐는 차이가 없다. e로 암호화 하면 d로 풀수 있고, d로 암호화를 하면 e로 풀수 있다.

## 보안성

위의 식에서 외부로 공개되는 값은 모듈로 n 과 공개키 e 이다.

e를 이용하여 d를 얻고 싶으면 다음 공식의 $\Phi(n)$ 을 모르는 이상 구하기가 쉽지 않다 (이산대수 문제).

$$ e \cdot d = 1 \mod {\Phi(n)}$$

그러면 n에서 두개의 소수 p, q를 찾으면 $\Phi(n)$ 을 아래 처럼 구할 수 있는데, 이것도 큰 수의 소인수분해 문제로 아주아주 큰 수 인 경우 소인수분해를 하는 것은 실제적으로 불가능하다.

$$
\Phi(n) = (p - 1) \cdot (q - 1)
$$

## OpenSSL로 확인

OpenSSL utility 로 이를 확인해 보면 다음과 같다.

우선 다음과 같이 하여 개인키를 생성할 수 있다.

```shell
$ openssl genrsa -out private.pem 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
........................................+++++
..............................................................................................+++++
e is 65537 (0x010001)
```

생성된 PEM 파일은 다음과 같이 확인할 수 있다.

```shell
openssl rsa -in private.pem -text -noout
RSA Private-Key: (2048 bit, 2 primes)
modulus:
    00:cb:cb:a0:cf:9a:0e:14:8c:61:fa:5e:ae:f3:e4:
    28:77:03:85:05:7c:cd:b9:aa:5a:60:a8:f3:5e:e9:
    e8:ad:9a:93:34:9c:4f:4e:03:94:80:18:71:ab:e3:
    b0:a0:3b:37:b8:d8:16:cf:ed:91:f9:a2:57:1d:58:
    54:f1:e6:5b:11:a6:86:ba:8b:61:ea:ae:5e:cc:96:
    df:c9:2f:86:71:ca:1c:8c:8e:fb:5a:9a:3a:39:30:
    f0:96:d7:52:5a:49:56:ad:76:59:a4:a3:5f:01:dc:
    9f:00:71:da:02:ca:eb:37:69:f8:14:30:fe:f0:ab:
    dc:46:a0:0d:9e:05:ff:0b:b3:78:c1:a0:6c:51:47:
    44:66:7c:c7:6e:f6:a1:10:49:04:a7:de:7d:66:3d:
    83:fc:1d:98:c7:d7:ae:b3:cd:93:15:cd:c4:2c:02:
    e0:e1:3a:90:2a:b9:e0:fe:bc:78:7f:88:18:5f:a4:
    aa:86:5e:df:cc:e4:e2:8c:23:c5:49:9d:01:53:5c:
    39:1d:30:bf:02:36:74:41:32:45:5c:72:a0:5a:82:
    07:59:ac:63:be:7f:11:3e:01:d5:9f:84:24:4c:37:
    69:d3:73:75:fa:83:b9:e3:b7:4f:c5:bc:f0:d2:c1:
    0f:80:34:30:3e:0d:bf:dc:3e:4c:60:1b:ee:2a:2a:
    bb:8b
publicExponent: 65537 (0x10001)
privateExponent:
    2e:6e:07:06:25:2f:fe:08:79:ae:03:f2:52:08:72:
    1b:a3:46:a4:18:69:fa:59:d0:5b:63:42:87:26:3d:
    67:87:e6:ef:be:88:e6:da:33:f3:f7:1d:b6:ae:9a:
    27:f7:35:db:bc:07:7e:79:be:9f:24:18:3a:cc:4c:
    16:0c:88:44:fe:2e:85:c3:89:9c:60:fb:a2:1a:e1:
    83:41:7b:9c:e3:12:1c:07:db:46:2a:0b:07:ca:99:
    95:94:1a:e4:0c:ff:5d:67:b0:46:ad:1d:d1:1b:c5:
    71:e1:7e:6c:d2:74:42:5c:b7:33:4a:72:5a:bc:9c:
    e3:ce:45:2b:f2:6b:c7:eb:44:4b:54:ea:0e:76:22:
    d7:1f:8b:94:8d:79:2b:f5:95:79:da:c4:fe:2d:d3:
    53:58:95:73:fd:5c:bd:f5:79:50:44:fd:25:06:1b:
    e1:44:45:7c:39:eb:3a:c4:79:37:71:9e:bc:bb:17:
    c9:35:f7:91:28:90:3b:47:d2:90:0d:61:53:6d:72:
    4d:fb:4c:7d:17:31:da:ad:d9:78:ce:94:52:c0:84:
    8d:02:00:ff:8e:bb:a5:32:d9:e7:51:c5:2b:3f:60:
    d4:0e:68:9d:b9:aa:b4:53:b0:a8:b2:a8:8a:4f:b3:
    a3:b3:56:78:58:49:76:32:bb:6a:91:5e:66:8c:74:
    81
prime1:
    00:f5:03:c6:e1:e6:e8:13:71:72:49:6d:b6:e5:b5:
    cc:4b:db:2b:6e:74:a4:3a:d9:9b:95:39:1c:ef:41:
    94:f9:5e:34:3b:de:4c:1c:3c:90:34:44:16:ff:a7:
    c4:23:35:72:3b:8d:27:82:39:0b:d9:02:b6:f4:f1:
    e2:42:56:68:00:de:f8:2a:b0:55:d5:96:c9:2c:d0:
    eb:ba:8a:dd:42:6b:85:d4:11:de:a3:4d:60:c9:05:
    9c:7f:fb:30:c6:5a:79:6e:ac:9a:cd:27:ef:2f:91:
    7c:47:31:c8:2a:8d:ce:ec:16:29:e6:f4:ac:b5:88:
    3b:36:6a:b1:b6:75:be:4c:4b
prime2:
    00:d4:ee:be:c9:78:69:25:ca:7f:0d:7a:1f:c2:99:
    3e:79:e2:97:0f:f4:ca:7c:16:a0:a8:0a:bf:0e:53:
    c8:b8:9e:2a:ef:c6:d3:5e:58:b5:48:f2:52:94:72:
    87:2c:42:f4:a6:09:45:94:73:6d:0a:44:a5:d1:f7:
    75:46:ce:3e:03:8b:4d:1a:62:ad:6f:0b:96:b8:83:
    fb:19:6c:8b:3f:b3:45:1d:76:5c:8a:26:43:f6:ea:
    d6:8b:75:74:91:ca:c7:2f:d9:84:11:aa:4c:ee:e5:
    a4:22:c9:6d:79:98:52:08:be:fa:c4:42:86:67:5e:
    61:bc:59:d7:0a:75:89:45:c1
exponent1:
    00:a5:c3:b7:63:90:a8:44:b7:45:1e:1e:a7:56:04:
    48:42:ad:f6:55:55:7e:e2:fd:e4:7f:f1:d2:fc:9f:
    ff:1d:33:39:dd:a3:49:14:f5:78:8e:93:de:87:7a:
    c6:7d:17:a4:c0:5b:80:76:5f:07:ff:fb:11:32:e9:
    0f:2d:d8:6d:a6:e1:33:3f:16:6c:0c:04:66:f8:f6:
    23:f5:e2:0b:4d:eb:96:f0:62:62:a1:53:31:7e:ef:
    57:f1:52:4d:ae:74:f9:a1:02:0f:fd:6a:de:2c:ed:
    9e:0a:40:c8:ee:d9:60:3c:63:c6:57:a6:03:cf:11:
    6b:16:26:db:32:d9:b8:34:bf
exponent2:
    29:ad:c1:b2:75:db:3f:06:6f:f0:17:63:78:17:be:
    de:e4:b7:64:ec:29:66:38:97:a1:cc:d8:b0:d9:3d:
    84:c5:90:e9:f6:25:11:66:93:b5:7f:99:22:6d:78:
    7f:f5:6b:25:c4:d2:d5:c7:f2:23:fc:63:e8:c1:63:
    37:44:cf:66:aa:31:a1:64:87:46:21:22:93:63:62:
    17:0b:e4:05:c7:f5:53:5b:03:aa:16:eb:5e:bd:80:
    d9:33:58:69:e1:23:33:fe:83:97:61:9a:45:78:b5:
    b4:09:71:60:47:ac:67:01:da:db:e7:99:9f:4a:1e:
    1f:5c:06:77:89:a2:21:01
coefficient:
    43:30:cc:bb:d4:5f:fa:d4:97:f3:c5:ed:72:ab:01:
    1d:8b:b8:2e:57:55:02:37:a2:12:cc:06:2b:00:2f:
    1c:fa:4f:d4:36:9d:12:ef:87:6f:ec:07:3b:c0:a0:
    dc:89:61:d3:c0:d8:c7:ad:8c:d4:9c:fe:bb:9c:49:
    72:a1:b8:c2:8e:1f:84:d4:dc:ae:a0:30:a0:ab:09:
    f9:c5:11:88:1c:cc:2b:9d:5f:48:95:fd:b8:7a:77:
    7f:1a:3b:b5:96:05:e4:9c:1f:48:3f:a3:bc:e3:40:
    ad:9c:67:4e:78:03:36:39:15:fb:6a:b9:1b:34:98:
    31:15:83:2d:ab:81:ff:66
```

각각의 필드는 다음과 같다.

- **prime1**, **prime2**: 소수 p와 q를 말한다. 두 수의 비트를 보면 총 앞의 '00:' 을 제외하면 1024bit (128bytes) 이다.
- **modulus**: $n = p \cdot q$를 말한다. 길이는 2048bit (256bytes) 이다.
- **publicExponent**: 공개키인 e 를 말한다. 일반적인 경우 공개키는 65537 (0x10001) 와 같은 고정된 값을 사용한다.
- **privateExponent**: 개인키인 d 를 말한다. 2048bit (256bytes) 에 가까운 값이 된다.
- **exponent1**, **exponent2**, **coefficient**: [중국인의 나머지 정리](<https://en.wikipedia.org/wiki/RSA_(cryptosystem)#Using_the_Chinese_remainder_algorithm>)에 따라 성능 빠른 연산을 위한 추가적인 파라메타들 이다.

공개키 (publicExponent)가 짧은 것은, 서명 용도로 사용할 때 특히나 장점이 될 수 있다.

- 암호화 용도로 사용할 때에도 private 키에는 추가적인 parameter가 있어, 공개키만 알고 있는 사람에 비하여 빠른 연산이 가능하다.
- 서명 용도로 사용할 때는 서명자가 1회 서명한 문서를, 다수의 사람들이 필요한 시점에 매번 검증을 하는 경우가 많다(인증서 서명 검증용). 이 경우 다수의 사람들이 검증을 할때 연산량을 줄일 수 가 있다.

개인키 파일에서 공개키 파일은 다음과 같이 생성할 수 있다.

```shell
$ openssl rsa  -in private.pem -out public.pem -pubout
writing RSA key

$ openssl rsa -in public.pem -pubin -text -noout
RSA Public-Key: (2048 bit)
Modulus:
    00:cb:cb:a0:cf:9a:0e:14:8c:61:fa:5e:ae:f3:e4:
    28:77:03:85:05:7c:cd:b9:aa:5a:60:a8:f3:5e:e9:
    e8:ad:9a:93:34:9c:4f:4e:03:94:80:18:71:ab:e3:
    b0:a0:3b:37:b8:d8:16:cf:ed:91:f9:a2:57:1d:58:
    54:f1:e6:5b:11:a6:86:ba:8b:61:ea:ae:5e:cc:96:
    df:c9:2f:86:71:ca:1c:8c:8e:fb:5a:9a:3a:39:30:
    f0:96:d7:52:5a:49:56:ad:76:59:a4:a3:5f:01:dc:
    9f:00:71:da:02:ca:eb:37:69:f8:14:30:fe:f0:ab:
    dc:46:a0:0d:9e:05:ff:0b:b3:78:c1:a0:6c:51:47:
    44:66:7c:c7:6e:f6:a1:10:49:04:a7:de:7d:66:3d:
    83:fc:1d:98:c7:d7:ae:b3:cd:93:15:cd:c4:2c:02:
    e0:e1:3a:90:2a:b9:e0:fe:bc:78:7f:88:18:5f:a4:
    aa:86:5e:df:cc:e4:e2:8c:23:c5:49:9d:01:53:5c:
    39:1d:30:bf:02:36:74:41:32:45:5c:72:a0:5a:82:
    07:59:ac:63:be:7f:11:3e:01:d5:9f:84:24:4c:37:
    69:d3:73:75:fa:83:b9:e3:b7:4f:c5:bc:f0:d2:c1:
    0f:80:34:30:3e:0d:bf:dc:3e:4c:60:1b:ee:2a:2a:
    bb:8b
Exponent: 65537 (0x10001)
```

위와 같이 Modulus와 Public Exponent만 추출된 것을 확인할 수 있다.

다음과 같이 몇개의 경우를 암호화 복호화 해보자.

RSA-2048은 256bytes(2048bit)의 데이타를 2048bit의 큰 숫자로 생각하고, RSA 연산을 수행하는 것이다.
우선 0으로 패딩된 데이타를 암호화 해보자.

```shell
$ dd if=/dev/zero if=zeros.bin bs=256 count=1

$ openssl rsautl -encrypt -raw -in zeros.bin -out enc.bin -pubin -inkey public.pem

$ hexdump -v enc.bin
0000000 0000 0000 0000 0000 0000 0000 0000 0000
...
00000f0 0000 0000 0000 0000 0000 0000 0000 0000
```

암호화한 데이타도 원본과 동일하게 0이다.

마찬가지로 1을 가진 값도 암호화를 해보면 그 자신이 된다.

```shell
$ < /dev/zero tr '\000' '\001' | head -c 1 > one.bin
$ dd if=/dev/zero of=ones.bin  bs=255 count=1
$ cat one.bin >> ones.bin

$ openssl rsautl -encrypt -raw -in ones.bin -out enc.bin -pubin -inkey public.pem
$ hexdump -v -C enc.bin
00000000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
...
000000f0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 01  |................|
```

이유는 RSA 의 모듈러 지수연산의 특성 상 0 이나 1을 지수연산 해보아도 그 자신이 되기 때문이다.

이외에도 RSA는 HASH나 AES와 같이 암호화 결과가 랜덤 특성을 가지지 못하므로, 중복 가능성이 높은 일반 평문 텍스트 문장을 RSA로 암호화를 하는 경우 분석적인 공격에 의하여 뚫릴 가능성도 높아진다.

RSA는 이와 같은 약점을 가지고 있으므로 텍스트 문장을 암호화 하는 용도로는 사용하지 않는다. 다만 이런 경우에는 [OEAP, PKCS#1](https://en.wikipedia.org/wiki/Optimal_asymmetric_encryption_padding) 패딩을 추가하여 사용한다. 위 rsautl을 사용하여 암호화 할 때 -raw 옵션을 제외하면 [PKCS#1 v1.5](https://datatracker.ietf.org/doc/html/rfc2313) 방법을 이용하여 패딩한다.

RSA를 암호화 용도로 사용하는 경우 다음 룰을 따라야 한다.

- RSA로는 일반 텍스트 문장을 암호화 하지 않고, hash, 임시 키 등의 random 특성을 가진 데이타를 암호화는 용도로만 사용한다.
- 텍스트를 암호화 하기 위하여는 RSA로 임시 비밀키를 전달하여 이를 사용하여 AES 등으로 대칭키 알고리즘으로 암호화 한다.

참고로 RSA와 같은 비대칭키 알고리즘인 ECC(Elliptic Curve Cryptography)의 경우에는 이와 같은 암호/복호화 용도로는 기능을 제공하지 않는다. Diffie-Hellman 방식의 키교환 ECDH 이나, 서명용 ECDSA 와 같은 용도로만 사용한다.

## 정리

이상으로 RSA를 사용한 암호화/복호화 방법에 대하여 정리하였다.

[암호화 키 길이]({{< ref "posts/tls-cryptography-3-block-cipher#KeyLength">}})에 정리한 것과 같이 인수분해 문제를 이용한 RSA는 AES 와 같은 것에 비하여 암호 강도가 낮다. 현재 기술로 AES-128 은 아직은 안전하다고 볼수 있는데, 이를 만족할만한 것은 RSA-3072 이상이 되어야 한다.

아직은 일반적인 경우 RSA-2048 을 많이 사용되나, 최근에는 RSA-4096 이상도 사용하는 경우가 있다.

이와 같이 키가 길어지면 작은 메모리를 가진 IoT 기기에서 사용 시 문제가 될 수도 있다.
뒤에서 다시 설명하지만, 보통은 공개키는 상위기관의 서명이 포함된 인증서 형태로 가지고 있게 된다.

IoT 기기에서는 최소한 루트인증서를 저장하여야 하는데, RSA-4096 인 경우 인증서의 modulus 길이가 4096bit(512bytes)가 된다.
이외에도 인증서가 체인으로 엮여 있는 경우 외부 기기와 통신을 위하여는 해당 인증서를 모두 통신 채널로 전달하여야 한다.
예를 들어 3단계 인증이 되어 있다면 각 공개키의 modulus의 길이만 1,532bytes가 된다.
이를 BLE, Zigbee, LoRA 등의 저속 통신 프로토콜을 이용하는 경우 이 전송 비용도 만만치 않게 된다.

이런 이유로 점차적으로 타원알고리즘(EC)을 적용한 암호화 시스템을 IoT 기기에서는 많이 사용하는 편이나, 일반 웹 서비스의 경우에는 아직은 기존에 오랜 기간 사용하였던 RSA 서명 시스템을 많이 사용한다.
