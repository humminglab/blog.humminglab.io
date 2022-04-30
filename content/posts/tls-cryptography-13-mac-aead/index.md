---
title: "TLS/암호 알고리즘 쉽게 이해하기(13) - MAC, AE, AEAD"
date: "2022-04-30T09:00:00+09:00"
lastmod: "2022-04-30T09:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Cryptography, MAC, AE, AEHD, AES-GCM, ChaCha20-Poly1305"]
categories: ["Security"]

series: ["TLS/암호 알고리즘 쉽게 이해하기"]
---

지금까지 설명한 암호화 알고리즘을 조합하여 확장을 해보기로 한다.

이 글에서 설명할 내용은 다음과 같다.

- MAC(Message Authentication Code)
- AE(Authenticated Encryption)
- AEAD(Authenticated Encryption with Associated Data)

## MAC (Message Authentication Code)

MAC은 한마디로 정리하면 [Hash]({{< ref "posts/tls-cryptography-10-hash">}})에 비밀키를 추가한 버전이라고 볼 수 있다.

키를 사용한다는 것으로 보면 [DSA]({{< ref "posts/tls-cryptography-9-dsa">}}) 디지털 서명과 비슷한 기능을 수행하지만, 공유키를 사용한다는 것이 다르다.

DSA는 Alice의 공개키를 가진 다수의 사람이 검증을 위한 용도이고, MAC은 키를 공유한 사람 간에 검증을 하기 위한 용도이다. 물론 연산량도 DSA와 비교하여 더 적고 빠르다.

가장 간단한 MAC 동작은 본문과 키를 합쳐서 SHA-1 등의 hash 를 실행하는 것이다. 이렇게 하면 키를 아는 사람만 hash 값의 유효성을 검증할 수 있다.

MAC은 hash 함수만으로 구성하거나, AES-CBC 암호화 모드를 적용하거나, 뒤에서 설명한 GCM 모드를 적용하느냐에 따라서 각각 HMAC, CMAC, GMAC 등으로 구분하기도 한다.

HMAC (Hash-based Message Authentication Code) 만 간단히 설명하면 [RFC 2104 HMAC](https://tools.ietf.org/html/rfc2104)로 정의되어 있고, 원리는 다음과 같은 단순하다.

$$
\begin{align*}
\text{HMAC}(K, m) &= \text{H} ((K \oplus \text{opad}) || H((K \oplus \text{ipad}) || m)) \\
\end{align*}
$$

opad는 0x5c 로 반복된 데이타이고, ipad는 0x36으로 반복되는 데이타이다.

다음과 같은 데이타를 합쳐서 hash함수를 수행하는 것이다.

- 키(K)와 opad(outer padding)를 xor 한 값
- 키(K)와 ipad(inner padding)를 xor 한 값을 hash 연산
- 메시지(m)

해시 함수는 이전에 언급한 MD5, SHA-1, SHA-256, ... 을 사용할 수 있고, 이들 hash 함수의 블럭 크기보다 키가 길다면 키도 hash 함수를 실행시켜 블럭 크기로 변형하여 사용한다.

## MAC 의 용도

FTPS, SFTP와 같은 데이타를 전송 시 MAC 을 추가로 전달하여 파일 전송 후 정상적으로 전송이 되었는지 검증 용도로 사용할 수 있을 것이다.

또는 반대의 경우도 가능할 것이다. 서버에 저장된 데이타가 삭제되지 않고 유효하게 저장하고 있는 지를 확인해 볼 수도 있다. 이런 경우 키를 변경하여 서버에게 MAC 검증을 요청하면 서버가 실제 데이타를 가지고 있지 않다면 MAC을 정상적으로 생성하지 못할 것이다.

## AE (Authenticated Encryption)

AES와 같은 암호화를 사용하는 경우 암호화된 데이타가 정상인지를 확인 할 방법은 없다. [AES-ECB]({{< ref "posts/tls-cryptography-4-block-cipher-mode#aes-ecb">}})의 padding으로 일부 검출이 가능할 수는 있지만 이것으로 암호화된 데이타가 정상인지를 확인하는 것은 안전하지 않다.

이와 같이 암호화된 데이타의 유효성을 검증하기 위하여 MAC 를 사용한다. 이를 AE (Authenticated Encryption) 라고 한다.

AE도 암호화와 MAC의 순서에 따라서 다음과 같이 나뉘어 진다. 아래 그림은 [Wikipedia](https://en.wikipedia.org/wiki/Authenticated_encryption)의 그림으로 해당 페이지를 보면 부연설명을 볼 수 있다.

| 구분   | Encrypt-then-MAC                                                                 | Encrypt-and-MAC                                                                  | MAC-then-Encrypt                                                                 |
| :----- | :------------------------------------------------------------------------------- | :------------------------------------------------------------------------------- | :------------------------------------------------------------------------------- |
| 구조   | {{< figure src="Authenticated_Encryption_EtM.png" width="300px" height="auto">}} | {{< figure src="Authenticated_Encryption_EaM.png" width="300px" height="auto">}} | {{< figure src="Authenticated_Encryption_MtE.png" width="300px" height="auto">}} |
| 사용예 | IPsec                                                                            | SSH                                                                              | TLS                                                                              |

## AEAD (Authenticated Encryption with Associated Data)

TLS로 데이타 통신을 하는 경우 AE 만으로 완전하지 않을 수 있다. 예를 들어 Alice가 Bob에게 보내는 메시지를 AE로 암호화 하여 전달하는 경우, 공격자는 이 데이타를 그대로 다른 대상으로 replay attack을 할 수 있다.
좀더 안전한 방법은 IP, TCP/UDP header에 포함된 주소 정보 등을 MAC 연산에 추가를 하는 것이다. 이렇게 하면 해당 데이타를 다른 주소로 전달하는 경우 무효화 할 수 있다.

이와 같이 암호화 되지 않은 데이타를 추가하여 검증하는 것을 AEAD (Authenticated Encryption with Associated Data) 라고 한다.

TLS cipher suite를 보면 AES-GCM, ChaCha20-Poly1305 이 이와 같은 AEAD 방식이다. 다른 조합도 가능하겠지만 이와 같은 두 조합이 가장 널리 사용한다.

둘 모두 기본 구성은 동일하고, 다음과 같은 절차를 거친다.

- 스트림암호화 방식으로 Key와 Nonce를 이용하여 난수열을 생성한다.
- 난수열의 앞의 일정 부분을 AEAD 용으로 사용하고, 나머지는 암호용도로 사용한다.
- 본문 데이타를 암호화열과 XOR 하여 암호화 한다.
- 같이 엮을 데이타(AD, Associated Data), 암호화 결과, 이들의 길이 등을 합쳐서 키있는 해시 함수를 실행하여 결과를 얻는다.

## AES-GCM (AES Galois/Counter Mode)

GCM mode는 아래 그림과 같은 구성을 가진다. (그림출처: [Wikipedia](https://en.wikipedia.org/wiki/Galois/Counter_Mode))

{{< figure src="1920px-GCM-Galois_Counter_Mode_with_IV.svg.png" width="600px" height="auto">}}

기본 구성이 예전에 설명한 [block cipher CTR mode]({{< ref "posts/tls-cryptography-4-block-cipher-mode#aes-ctr-counter">}})와 유사하다. 해당 그림을 다시 보면 아래 그림과 같다.

{{< figure src="1202px-CTR_encryption_2.svg.png" width="600px" height="auto">}}

CTR 모드와 비교하면 다음과 같다.

- 기본적인 CTR 모드로 암호화 하는 것은 동일하다. 다만, Counter 0을 암호화 용도로 사용하지 않고, 해시의 키로 사용한다.
- 평문과 추가 데이타(Auth Data)는 $mult_H$ 라고 표시된 해시 함수를 이용하여 연산을 하여 최종 Auth Tag (MAC과 같은 용도) 를 생성한다.

GCM 모드의 특징은 난수의 특성을 가져야 하는 암호화 해시 함수가 아닌 일반 해시를 사용하여 연산량이 상대적으로 작고, CTR 모드의 특징으로 각 블럭의 암호화를 병렬화 가능하다는 장점이 있어, 가장 많이 사용하는 AEAD 모드이다.

## ChaCha20-Poly1305

ChaCha20-Poly1305 는 스트림 암호인 ChaCha20 에 Poly1305 MAC을 조합한 AEAD 모드 이다.

[RFC 7539](https://datatracker.ietf.org/doc/html/rfc7539)으로 정의 되어 있고, [RFC 7905](https://datatracker.ietf.org/doc/html/rfc7905)로 TLS/DTLS 1.2 의 cipher suite에 포함되었다.

기본 구성은 아래 그림과 같다. (그림출처: [Wikipedia](https://en.wikipedia.org/wiki/ChaCha20-Poly1305))
{{< figure src="ChaCha20-Poly1305_Encryption.svg.png" width="600px" height="auto">}}

CTR 모드와 유사한 방식으로 Counter 0의 값은 MAC의 키로 사용하고, AD 데이타와 평문을 포함하여 Poly1305 MAC을 생성한다.

키길이는 256bit이고, 96bit의 nonce를 사용한다.

## OpenSSL 로 동작 확인

HMAC 는 다음과 같이 dgst subcommand로 확인 가능하다.

```shell
> echo -n '1234567890' > mydata.txt
> openssl dgst -sha256 -hmac 1234567890 mydata.txt
HMAC-SHA256(mydata.txt)= 8a27c85f02d1b8d2b3df658be130bb818656a3bd08620f9254ee73ba37f971eb
```

AES-GCM 과 ChaCha20-Poly1305 는 OpenSSL command line으로는 제공하지 않는다. 이유는 아래 링크에 설명되어 있고, 직접 OpenSSL EVP API 를 이용하여 구현하는 것은 가능하다.

- [from openssl docs](https://github.com/openssl/openssl/blob/14d3bb06c9c11b3e13c64611913757c27bc057f2/doc/man1/openssl-enc.pod.in#L268)

## 마무리

최신 TLS 1.3 에서는 보안에 문제가 있는 AES-CBC 등이 제거되고, 아래와 같이 5개의 cipher suite로 정리되었다.

- TLS\_**AES_128_GCM**\_SHA256
- TLS\_**AES_256_GCM**\_SHA384
- TLS\_**CHACHA20_POLY1305**\_SHA256
- TLS\_**AES_128_CCM**\_SHA256
- TLS\_**AES_128_CCM_8**\_SHA256

[AES_CCM](https://datatracker.ietf.org/doc/html/rfc3610) 도 위에 AE에서 언급한 MAC-then-Encrypt 로 AEAD 방식의 일종이지만 실제로는 많이는 사용하지 않는다.

[The 2021 TLS Telemetry Report - F5 Labs](https://www.f5.com/labs/articles/threat-intelligence/the-2021-tls-telemetry-report) 을 보면 2021년에 조사한 백만개 사이트 중 94% 가까운 연결이 AES-GCM 방법을 사용하고, ChaCha20-Poly1305 가 2% 정도 사용 되었다고 한다.

시간이 지나면 보안이 취약한 기존 cipher suite 들의 사용이 줄어든다면 이 두 방식이 주를 이룰 것으로 보여진다.

두 방식의 차이점은 AES hardware acceleration 이 있는 경우에는 AES-GCM 방식이 장점이 있으나, 순수하게 소프트웨어로 구동을 하는 경우 ChaCha20-Poly1305 방식이 빠르기 때문에 IoT 등 저사양 기기들이 늘어나면서 ChaCha20-Poly1305 방식도 점차로 인기가 많아질 듯 싶다.

이상으로 MAC, AE, AEAD의 특징 및 알고리즘 정리를 마무리 한다.
