---
title: "TLS/암호 알고리즘 쉽게 이해하기(7) - Diffie-Hellman Key Exchange"
date: "2022-03-10T16:00:00+09:00"
lastmod: "2022-08-19T10:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Cryptography, Modulo, Diffie-Hellman, DHE, ECDHE"]
categories: ["Security"]
series: ["TLS/암호 알고리즘 쉽게 이해하기"]
---

지난 번에 설명한 [이산대수]({{< ref "posts/tls-cryptography-6-math">}})를 이용하여 Diffie-Hellman Key Exchange(DHKE, 키교환 또는 키합의)을 이해해 보자.

TLS 암호화 채널을 절차를 간단하게 보면 다음과 같다.

- 서버를 믿을 수 있는지 검증, 필요시 클라이언트도 인증
- RSA 암호화 채널로 키를 전달하거나, Diffie-Hellman 방식으로 키교환
- 교환한 키로 AES, ChaCha20와 같은 대칭키로 암호화

Diffie-Hellman은 여기서 두번째 대칭키를 교환하는 방법이다.

RSA와 같은 비대칭키를 이용하여 암호화 채널을 만든 후 이를 이용하여 대칭키를 전달하는 것은 직관적이다.
하지만 Diffie-Hellman 방식은 이와 같은 암호화 채널 없이도, 서로 키를 교환할 수 있는 방식이다.

## 기본 원리

지수 연산에서 다음과 같은 수식은 모두 동일하다.

$$ {(g^m)}^n = g^{mn} = {(g^n)}^m $$

위 연산은 [이산대수-모듈로 연산]({{< ref "posts/tls-cryptography-6-math#모듈로-연산">}}) 에서 설명한 것과 같이 소수 모듈로 연산에서도 동일한 조건이 성립한다.

$$ {(g^m)}^n = g^{mn} = {(g^n)}^m \mod k, (\text{k 는 소수})$$

둘의 차이는 일반 연산에서는 이의 역연산인 로그 연산이 가능하지만, 모듈로 연산에서는 이산대수 문제인 역연산 자체가 어렵다는 것이다.

Diffie-Hellman의 기본 원리는 다음과 같다. Alice와 Bob이 키를 공유하려는 당사자들이고, Chuck는 이를 도청하는 사람이다.

- Bob 아주 큰 소수 $p$ 를 선택하고, 모듈로 $p$ 내의 수 중 적당한 수 $g$ 를 선택하여 Alice 에게 전달.
  - $p$ 는 모듈로 연산에 사용하는 값이고, $g$ 는 지수연산의 밑인 기수(generator) 이다. $g$ 는 보통 작은 수인 2를 사용한다.
- 모듈로 $p$ 내의 임의의 수를 각자 선정 (Bob 선정한 것은 $m$, Alice가 선정한 것은 $n$)
- 자신이 가지고 있는 수 로 모듈로 지수 연산한 결과를 상대방에게 전달.

{{< mermaid >}}sequenceDiagram
  Participant Alice
  Participant Bob
  Note over Bob: 소수 p, g 선정, m 선정
  Bob ->> Alice: p, g, gm mod k
  Note over Alice: n 선정
  Alice ->> Bob: gn mod k
{{< /mermaid >}}

암호화 되지 않은 채널로 위와 같은 전달하면, 각각이 알고 있는 값은 다음과 같다.

- 모두: $p, g, g^m \mod k, g^n \mod k$
- Alice: $n$
- Bob: $m$

여기서 결국 원하는 키는 $g^{mn} \mod k$ 의 결과 이다.

- Alice는 $m$ 을 알고 있으므로 $(g^n)^m \mod k$ 로 결과를 알수 있음
- Bob은 $n$ 을 알고 있으므로 $(g^m)^n \mod k$ 로 결과를 알수 있음
- 이 채널을 도청한 Chuck은 $g^m, g^n \mod k$은 알고 있으나 이로는 $g^{mn} \mod k$를 알수 없다.

알고보면 Diffie-Hellman 방식은 의외로 단순하다.

실제로는 2048bit 범위에서 큰 소수를 사용하는데, 여기에서는 간단하게 작은 소수 2017을 예를 든다.

Bob은 소수 $p$ 로 2017을 선택하고 $g$로 2를 선택 하였다고 하자.
Bob은 $m$으로 1273, Alice는 $n$ 으로 1721을 각각 선택하였다.

- Bob이 전달한 값은 다음과 같다.
  - $p = 2017$, $g = 2$, $M = 2^{1273} \mod{2017} = 1859$
- Alice가 전달한 값은 다음과 같다.
  - $N = 2^{1721} \mod 2017 = 219$
- Alice는 Bob이 보낸 1859 값에 자신이 알고 있는 $n$ 1721로 지수연산을 하면 된다.
  - $1859^{1721} \mod 2017$: 1427
- Bob은 Alice가 보낸 219 값에 자신이 알고 있는 $m$ 1273로 지수연산을 하면 된다.
  - $219^{1273} \mod 2017$: 1427

이와 같이 둘은 비밀번호로 이용할 1427 이라는 값을 교환하게 되고, 중간에 이를 도청하여, p 2017, g 2, M 1859, N 219 값을 모두 알고 있어도 이를 이용하여 1427 이라는 값을 얻을 수 없다.

실제 연산은 온라인 [Big Number Calculator](https://www.boxentriq.com/code-breaking/big-number-calculator) 등을 이용하여 계산해 보면 된다.

## Safe Prime

소수 $p$ 의 모듈로 지수 연산 내에서 모든 기수 조건에 대해서 지수 연산을 해보면 다음처럼 순환 구조가 일정치 않다.

예를 들어 $p=13$ 인 경우에 원소 1~12 까지를 각각 지수 연산을 해보면 다음과 같이 순환한다.

```
g:  1, [1]
g:  2, [1, 2, 4, 8, 3, 6, 12, 11, 9, 5, 10, 7]
g:  3, [1, 3, 9]
g:  4, [1, 4, 3, 12, 9, 10]
g:  5, [1, 5, 12, 8]
g:  6, [1, 6, 10, 8, 9, 2, 12, 7, 3, 5, 4, 11]
g:  7, [1, 7, 10, 5, 9, 11, 12, 6, 3, 8, 4, 2]
g:  8, [1, 8, 12, 5]
g:  9, [1, 9, 3]
g: 10, [1, 10, 9, 12, 3, 4]
g: 11, [1, 11, 4, 5, 3, 7, 12, 2, 9, 8, 10, 6]
g: 12, [1, 12]
```

위에서 g가 3 인 경우 모듈로 연산으로 $3^1=3, 3^2=9, 3^3=1, 3^4=3, ...$ 으로 3개의 원소 1,3,9 만 순환하게 된다.

p를 13 으로 DH 키교환을 할 때 Alice 와 Bob 이 보낸 M, N의 값이 2,6,7,11 의 값이라면 이 값으로 생성 가능한 수는 p-1 개로 답을 찾는데 어려울 수 있으나,
M 또는 N의 값이 3 이라면 경우의 수는 1,3,9 총 3개 밖에 없다. 경우의 수가 12 개에서 3개로 줄어드니 보안에 취약해 질 수 있다.

소수 중에서도 위와 같인 순환 그룹이 짧게 되는 문제가 없는 소수들이 있다. 이들 소수를 [Safe prime(안전한 소수)](https://en.wikipedia.org/wiki/Safe_and_Sophie_Germain_primes#safe_prime)라고 한다.
이 소수의 조건은 소수 p 가 있으면 이보다 1 작은 수는 항상 짝수이다. 이 짝수의 반 값이 소수 인경우 이를 safe prime 이라고 한다 ($q=(p-1)/2$).

예를 들어 소수 $p=23$ 인 경우 $q=(23-1)/2=11$ 도 소수 이므로 safe prime이다. 이때의 지수 연산은 다음과 같이 순환한다.

```
g:  1, len:  1, [1]
g:  2, len: 11, [1, 2, 4, 8, 16, 9, 18, 13, 3, 6, 12]
g:  3, len: 11, [1, 3, 9, 4, 12, 13, 16, 2, 6, 18, 8]
g:  4, len: 11, [1, 4, 16, 18, 3, 12, 2, 8, 9, 13, 6]
g:  5, len: 22, [1, 5, 2, 10, 4, 20, 8, 17, 16, 11, 9, 22, 18, 21, 13, 19, 3, 15, 6, 7, 12, 14]
g:  6, len: 11, [1, 6, 13, 9, 8, 2, 12, 3, 18, 16, 4]
g:  7, len: 22, [1, 7, 3, 21, 9, 17, 4, 5, 12, 15, 13, 22, 16, 20, 2, 14, 6, 19, 18, 11, 8, 10]
g:  8, len: 11, [1, 8, 18, 6, 2, 16, 13, 12, 4, 9, 3]
g:  9, len: 11, [1, 9, 12, 16, 6, 8, 3, 4, 13, 2, 18]
g: 10, len: 22, [1, 10, 8, 11, 18, 19, 6, 14, 2, 20, 16, 22, 13, 15, 12, 5, 4, 17, 9, 21, 3, 7]
g: 11, len: 22, [1, 11, 6, 20, 13, 5, 9, 7, 8, 19, 2, 22, 12, 17, 3, 10, 18, 14, 16, 15, 4, 21]
g: 12, len: 11, [1, 12, 6, 3, 13, 18, 9, 16, 8, 4, 2]
g: 13, len: 11, [1, 13, 8, 12, 18, 4, 6, 9, 2, 3, 16]
g: 14, len: 22, [1, 14, 12, 7, 6, 15, 3, 19, 13, 21, 18, 22, 9, 11, 16, 17, 8, 20, 4, 10, 2, 5]
g: 15, len: 22, [1, 15, 18, 17, 2, 7, 13, 11, 4, 14, 3, 22, 8, 5, 6, 21, 16, 10, 12, 19, 9, 20]
g: 16, len: 11, [1, 16, 3, 2, 9, 6, 4, 18, 12, 8, 13]
g: 17, len: 22, [1, 17, 13, 14, 8, 21, 12, 20, 18, 7, 4, 22, 6, 10, 9, 15, 2, 11, 3, 5, 16, 19]
g: 18, len: 11, [1, 18, 2, 13, 4, 3, 8, 6, 16, 12, 9]
g: 19, len: 22, [1, 19, 16, 5, 3, 11, 2, 15, 9, 10, 6, 22, 4, 7, 18, 20, 12, 21, 8, 14, 13, 17]
g: 20, len: 22, [1, 20, 9, 19, 12, 10, 16, 21, 6, 5, 8, 22, 3, 14, 4, 11, 13, 7, 2, 17, 18, 15]
g: 21, len: 22, [1, 21, 4, 15, 16, 14, 18, 10, 3, 17, 12, 22, 2, 19, 8, 7, 9, 5, 13, 20, 6, 11]
g: 22, len:  2, [1, 22]
```

1과 p-1 인 22 를 제외하고는 순환하는 개수가 q 인 11이거나, 2q 인 22 이다. 앞의 소수 11과 비교하여 일정한 형태를 가지므로 DH에 사용하면 좀더 안전할 것이다.

위의 계산 결과나 safe prime 에 대해서는 아래 링크에 좀 더 자세하게 나와 있다.

- [Safe primes in Diffie-Hellman](https://securitypitfalls.wordpress.com/2017/05/05/safe-primes-in-diffie-hellman/) (위 순환군 계산은 링크에 있는 python script를 이용하였다)
- [On Generators of Diffie-Hellman-Groups](https://florianjw.de/en/insecure_generators.html)

## Diffie-Hellman Parameter 생성

OpenSSL을 이용하여 Diffie-Hellman parameter인 소수 p와 기수(generator라고도 한다) g를 얻는 것은 다음과 같이 할 수 있다. TLS 에서는 보통 g 로 2를 사용한다.

```shell
$ time openssl dhparam -out dhparam.pem 2048
Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time
...
openssl dhparam -out dhparam.pem 2048  34.29s user 0.01s system 99% cpu 34.304 total

$ openssl dhparam -in dhparam.pem -text -noout
    DH Parameters: (2048 bit)
        prime:
            00:9a:3a:c8:3a:f8:c9:af:1e:19:02:e2:8a:2b:0c:
            ac:60:8d:df:47:46:d8:dd:08:8c:af:6a:fa:0a:d6:
            55:21:56:0f:ee:7d:84:a9:25:96:1c:cf:f0:07:bb:
            44:40:2c:3a:60:1a:93:7e:86:fb:0e:63:7a:fe:5b:
            67:36:ee:15:81:4d:d5:60:52:ef:cf:f4:22:d0:1d:
            6a:77:ff:10:0d:05:69:c8:5c:cb:d8:a5:b3:98:e8:
            31:a3:12:f3:a1:a0:19:02:3f:72:ee:52:08:fa:ca:
            b3:f1:ec:5e:35:ae:4b:3f:3a:c3:b4:7e:bc:81:6e:
            be:86:97:0d:01:b2:ad:89:dd:10:c5:a9:d6:ef:39:
            f0:2f:b6:d7:6b:9c:8e:b7:2d:07:33:a1:4e:22:11:
            13:c1:e4:1c:40:09:c8:be:7b:1e:a8:c2:27:76:35:
            6c:33:a2:33:7f:03:e5:62:33:da:89:58:48:e1:b1:
            ce:ec:7a:2f:3e:91:82:23:8f:43:67:56:34:2b:1d:
            37:cb:29:0b:c0:09:66:b5:34:d6:c1:95:5c:cc:ce:
            76:d3:92:02:32:f2:44:de:ca:72:37:12:57:42:ad:
            8e:aa:9b:7f:fd:e3:79:38:e7:e5:12:6a:45:09:92:
            94:11:00:ab:dc:75:3b:29:eb:11:f2:77:36:e8:7b:
            36:6b
        generator: 2 (0x2)
```

컴퓨터에 따라서 다르나, 위의 linux machine에서는 2048 bit 길이의 DH 용 소수를 찾는데 34.29초가 소요되었다.

이것과 비교하여 비대칭키 RSA를 생성하는 것은 4096 bit의 경우에도 0.23초 밖에 소요되지 않았다.

```shell
$ time openssl genrsa 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
...
openssl genrsa 2048  0.03s user 0.00s system 94% cpu 0.034 total

$ time openssl genrsa 4096
Generating RSA private key, 4096 bit long modulus (2 primes)
...
openssl genrsa 4096  0.23s user 0.00s system 98% cpu 0.233 total
```

일반적으로 RSA 2048bit 와 DH 2048bit 의 경우 비슷한 암호 강도를 가진다고 볼수 있으나 생성은 차이가 많이 발생한다.

이와 같은 이유는 dhparam으로 소수를 생성할 때는 위의 "safe prime"을 찾기 때문이다. 소수 $p$ 에 대해서 $(p-1)/2$ 도 소수이면 $p$ 를 safe prime이라고 한다.

Safe prime을 찾는데 시간이 걸리는 이유는 큰 수에서 소수를 찾을 확률은 1/2000 정도 된다. 하지만 safe prime은 $(p-1)/2$ 도 소수인 경우를 찾아야 하므로 다시 1/2000 확률 이므로 찾을 확률이 1/2000 x 1/2000 가량 되기 때문이다.

TLS web server 를 설정 시 DHE 키교환을 지원하는 경우 위와 같이 'openssl dhparam'을 이용하여 등록할 수 있다.
[Nginx](https://www.nginx.com/) 웹서버를 예로 들면 [ssl_dhparam](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_dhparam) 으로 등록하면 된다.

라즈베리파이와 같은 저사양 기기에서는 DH param을 찾는데 수 시간이 소요될 수 있다. 이 경우에는 PC에서 생성하여 복사하거나, 아니면 [openssl dhparam -dsaparam](https://www.openssl.org/docs/man1.0.2/man1/dhparam.html) 옵션으로 만들면 safe prime 이 아니라 좀 더 빠르게 만들 수 있다.

DH 키 생성 시 소수 크기는 현재는 2048 bit 이상을 권고하고 있다. 그 이하인 경우에는 Logjam attack 등으로 취약 해 질 수 있다. 관련 사항은 아래 링크를 참고할 수 있다.

- [Weak Diffie-Hellman and the Logjam Attack](https://weakdh.org/)
- [디피 헬만 키 (Diffie-Hellman Key) 를 2048 bit 로 바꿔야 하는 이유 - RSEC.KR](https://rsec.kr/?p=242)

## 정리

최근에는 DHE 와 유사한 타원암호알고리즘(ECC)를 Diffie-Hellman 방식으로 적용한 ECDHE 를 더 많이 사용한다.

다시 정리하면 TLS 에서 주로 사용하는 키 교환 방법은 다음과 같다.

- DH 방식: DHE, ECDHE
- RSA 채널로 키전달 방식: RSA

DH 키 교환 방식을 사용한 경우 서버의 private key를 안다고 하여도 중간에서 키를 얻을낼 방법이 없다.
개발 단계에서는 [Wireshark 으로 TLS 캡쳐 및 디코딩 하기]({{< ref "posts/how-to-capture-tls-with-wireshark">}})에서 설명한 것 처럼 TLS library에서 로그 채널로 알려주는 대칭키 값(CLIENT_RANDOM)을 얻어서 디코딩 하여야 한다.

이상으로 DHKE는 마무리한다.
