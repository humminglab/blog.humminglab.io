---
title: "TLS/암호 알고리즘 쉽게 이해하기(2) - Random"
date: "2022-012-11T09:00:00+09:00"
lastmod: "2022-02-11T09:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Cryptography", "TLS", "OpenSSL"]
categories: ["Security"]
series: ["TLS/암호 알고리즘 쉽게 이해하기"]
---

암호 알고리즘에서 난수는 중요한 요소이다.
예를 들어 암호 키 생성 시에도 난수로 만드는데, 생성된 난수가 편향성을 가지게 되면 암호 알고리즘이 아무리 좋아도 취약해 질 수 밖에 없다.

한 예로 오래전 일이지만, 2008년 [Debian linux OpenSSL 0.9.8 의 잘못된 patch](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-0166)로 seed를 current process ID 로만 사용하여 결과적으로 65,536개의 값 중 하나로 난수가 생성되어 brute force 공격으로 키를 찾을 수 있는 문제가 발생한 적도 있고, 이외에도 [CWE(Commn Weakness Enumeration)](https://cwe.mitre.org/find/index.html) 에서 random 으로 검색해보면 문제가 되었던 여러 케이스들을 찾아 볼 수 있다.

## 난수의 특징

위와 같은 문제를 가지지 않도록 암호에서 사용하는 난수는 다음과 같은 특성을 가져야 한다.

- 생성되는 난수가 확률적으로 균등분포로 무작위 하여야 한다.
- 이전에 발생한 난수 값에서 다음으로 생성하는 값이 예측 불가능하여야 한다.

말 그대로 난수는 어떤 방식으로든 예측 불가능하도록 무작위 하게 보여야 한다는 것이다.
이런 불확실성을 용어로 엔트로피(entropy)라고 한다.

디지털로 돌아가는 컴퓨터에서 이런 난수를 발생하기 위한 소스를 찾기가 쉽지 않다.
이런 엔트로피 소스로는 다음과 같은 것들을 통하여 만들 수도 있다.

- HDD의 엑세스 타임 정보
- 마우스, 키보드 패턴
- 인터럽트 발생 패턴
- 마이크 입력 노이즈

광전효과, 열잡음, 양자역한 현상등의 무작위성을 이용한 하드웨어로 만든 난수 발생기를 사용하는 경우도 있다.

Intel 이나 AMD 프로세서는 이런 하드웨어 난수 발생기가 추가되어 있고, [RDRAND](https://en.wikipedia.org/wiki/RDRAND) 어셈블리 명령어로 이를 얻을 수 있다.
ARM Cortex-M 과 같은 MCU의 경우에도 내부에 TRNG(True Random Number Generator)라고 하드웨어 난수 발생기를 제공하는 모듈들이 있다.
아래 예를 드는 [STM32L562](https://www.st.com/en/microcontrollers-microprocessors/stm32l5x2.html) 와 같이 암호 하드웨어 가속기가 있는 모듈 들은 TRNG도 필수라고 할 수 있다.
경량 RealTime OS를 사용하고, 주변기기가 통합된 MCU를 사용하는 이런 임베디드 기기에서는 TRNG가 없다면 엔트로피 소스를 만들어 내는게 큰 고역이다.

{{< figure src="stm32l562_512kb.jpg" width="400px" height="auto" caption="STM32L562 TRNG 예" >}}

## RNG vs PRNG

위처럼 엔트로피 원을 이용하여 무작위한 난수를 생성하는 것만 아니라, PRNG(Pseudo Random Number Generator, 의사난수 발생기)가 필요한 경우도 있다

C Library의 `rand()` 가 PRNG의 일종이다. 초기에 임의의 seed값을 `srand()`로 설정 하면 이 seed를 기반으로 난수를 생성하게 된다.

이 함수의 알고리즘은 [선형 합동 생성기(Linear congruential generator)](https://ko.wikipedia.org/wiki/%EC%84%A0%ED%98%95_%ED%95%A9%EB%8F%99_%EC%83%9D%EC%84%B1%EA%B8%B0) 라고 하는데, BSD 라이브러리의 경우 아래와 같은 공식으로 생성된다.

$$ \text{state}\_{n+1} = (1103515245 \cdot \text{state}\_n + 12345) \mod 2^{31} $$

초기 `srand()` 로 설정한 seed 값인 $\text{state}\_0$ 로 부터 시작하여 0~$2^{31}$ 범위의 난수($\text{state}_{n+1}$)를 생성한다.

이런 PRNG는 패턴은 무작위 해 보이지만, 초기 값을 동일하게 설정하면 동일한 난수열을 재연할 수 있다는 특징이 있다. 이런 특성 때문에 DRNG(Deterministic Random Number Generator)라고도 한다.

참고로 위와 같은 `rand()` 함수는 암호 용도로는 사용 불가능하다. 이유는 위 공식을 보면 알겠지만 중간값인 $\text{state}_{n}$ 을 알면 위 함수에 이 값을 대입하면 다음으로 생성되는 값을 알 수 있다. 또한 [위키피디아 페이지의 특성](https://ko.wikipedia.org/wiki/%EC%84%A0%ED%98%95_%ED%95%A9%EB%8F%99_%EC%83%9D%EC%84%B1%EA%B8%B0#%ED%8A%B9%EC%84%B1)에서 언급한 것과 같이 좋은 품질의 난수열을 만들어 내지는 못한다.

암호화 용도로 사용 가능한 PRNG는 Cryptographically-Secure Pseudo Random Number Generator(CPRNG 또는 CSPRNG)라고 말하기도 하는데, 선형합동생성기보다는 훨씬 복잡한 hash 함수 등을 이용하여 생성하며, 순방향(이전값)이든 역방향(다음값)이든 예측해 내지 못한다는 것을 검증받은 방법들이다.

정확한 수학적인 이론을 기반으로 만드는 것은 아니기 때문에 저마다 개별적으로 구현하거나, 미 국립표준기술연구소 [NIST](https://www.nist.gov/)에서 정의한 [SP 800-90A](https://csrc.nist.gov/publications/detail/sp/800-90a/rev-1/final), [90B](https://csrc.nist.gov/publications/detail/sp/800-90b/final), [90C](https://csrc.nist.gov/publications/detail/sp/800-90c/draft) 표준에서 추천한 방법에 따라 구현한다.

## Linux의 /dev/random, /dev/urandom

Linux 의 경우 /dev/random, /dev/urandom 디바이스 드라이버를 이용하여 난수를 제공한다.

기본적인 구조는 AMOSYS Security Blog의 [Linux RNG Architecture](https://blog.amossys.fr/linux-csprng-architecture.html) 를 보면 개괄적으로 이해할 수 있다.
아래 그림은 이 블로그에 있는 그림이다.

{{< figure src="https://blog.amossys.fr/content/images/article29/random.c.png" width="600px" height="auto" caption="Linux 5.4 CSPRNG Architecture">}}

해당 커널 버전까지는 /dev/urandom 과 /dev/random 드라이버는 다른 경로로 난수를 얻는다.

- 엔트로피 소스인 입력장치, 디바이스, HDD 등 에서 얻은 엔트로피를 input_pool 에 버퍼링을 한다.
- 부팅 직후에 엔트로피가 부족함을 대처하기 위하여 인터럽트로 부터 얻은 엔트로피도 fast_pool 로 관리한다.
- /dev/urandom

  - ChaCha20 암호 함수를 이용하여 CPRNG 의 난수를 생성한다.
  - 5분 가량의 간격으로 input_poll 에서 엔트로피를 꺼내서 CPRNG seed 를 재설정 하여, 혹시나 모를 난수 예측 방지
  - CPRNG로 생성하기 때문에 /dev/random에서 읽는 경우 blocking 되지 않는다.

- /dev/random

  - input_pool에서 얻은 entropy를 가져와 난수를 만들어 낸다.
  - Entropy 소스의 편향성을 없애기 위하여 SHA-1 hash 함수를 실행한 결과를 사용한다.
  - 만일 input_pool 에 entropy 데이타가 부족한 경우, /dev/random 읽기는 blocking 된다.

난수를 엔트로피 소스 값을 그대로 사용하지 않고, SHA-1 hash 나 ChaCha20 암호 함수를 이용하여 가공 하여 사용하는 것은, 이들 함수들의 특징을 이용한 것이다.
Hash 나 암호 함수 들의 기본 특징은 입력 데이타 중 1비트라도 바뀐 경우 결과 값은 이전 값과 어떤 관계성도 찾을 수 없을 정도로 전체가 무작위한 값처럼 변경되기 때문이다.

위 두 개의 드라이버 중 암호 용도로는 /dev/urandom 을 사용하는 것이 좋다.

PC는 엔트로피가 빠르게 모일 수 있을지 모르나, 하드웨어 TRNG가 없는 임베디드 리눅스의 경우 엔트로피가 부족하면 난수를 얻는 프로그램이 blocking될 수 있기 때문이다.
특히 부팅 직후 데몬으로 실행하는 프로그램이라면 더 쉽게 이런 문제가 발생할 수 있다.

Entropy가 어느 정도 모여 있는지는 다음과 같이 확인할 수 있다.

```shell
$ cat /proc/sys/kernel/random/entropy_avail
2395
$ cat /proc/sys/kernel/random/entropy_avail
2396
$ cat /proc/sys/kernel/random/poolsize
4096
```

최신 linux 버전의 경우 /dev/random 의 동작 방식이 제거되고, /dev/urandom 방식을 동일하게 사용한다. 즉, /dev/random 도 non-blocking 이다.

[독일 BSI](https://www.bsi.bund.de/) 페이지의 검색에서 "Documentation and Analysis of the Linux Random Number Generator"를 검색해보면 linux 버전별로 random number generator 의 자세한 구조를 찾아 볼 수 있다. 이 문서 중 [Documentation and Analysis of the Linux Random Number Generator v4.4](https://www.bsi.bund.de/SharedDocs/Downloads/EN/BSI/Publications/Studies/LinuxRNG/LinuxRNG_EN_V4_4.pdf?__blob=publicationFile&v=2)의 Figure 2를 보면 Linux 5.10 에서는 /dev/urandom, /dev/random 이 동일하게 구현된 것을 확인할 수 있다.

## MbedTLS 의 예

Mbed TLS도 [CPRNG](https://tls.mbed.org/module-level-design-rng) 로 암호 용도의 난수 발생기가 구현되어 있다.

Mbed TLS를 포팅 시 다른 라이브러리나 OS 의존성이 크지 않아 쉽게 빌드 되지만, 엔트로피 소스 설정은 신경 써주어야 한다.
특히, RTOS 를 사용하는 MCU에 Mbed TLS 가 포팅되어 있는 경우도 있는데, 동작만 되도록 단순히 rand() 함수로 엔트로피를 넣어주는 경우도 있다.

이 엔트로피 소스를 적절히 설정을 해주어야 하는데, [mbedtls_entropy_add_source()](https://tls.mbed.org/api/entropy_8h.html#ad1bf424d076142e9aeec9e68207f5aaa)를 사용하여 적절한 entropy source를 찾아서 넣어주면 된다.

만일, 하드웨어 상에서 엔트로피 소스가 전혀 없다면 [mbedtls_entropy_update_manual()](https://tls.mbed.org/api/entropy_8h.html#a81765f6cdf4e5111bcb9f4324f3234cb) 를 이용할 수도 있다.

사용 절차는 대략적으로 다음과 같다.

- FW 초기 상태는 기기마다 동일한 난수 데이타 블럭을 가지고 있다. 해당 영역은 write가 가능한 영역이다.
- Mbed TLS 초기화 시 `mbedtls_entropy_update_manual()`로 엔트로피 소스를 설정한다.
- 초기화 후 기기 종료 시점 또는 적절한 시점마다 이 난수 데이타 블럭의 값을 `mbedtls_entropy_func()` 등을 이용하여 새로운 난수 값으로 교체한다.

이와 같이 하면 난수 데이타 블럭은 부팅 할때 마다 다른 값을 유지할 수 있어서 어느 정도 수준의 엔트로피 원으로 사용할 수 있다.

## 보안 정책

처음에 언급한 [CWE(Commn Weakness Enumeration)](https://cwe.mitre.org/find/index.html)의 문제 사례를 보면 알 수 있듯이, 많은 이슈들이 엔트로피가 충분하지 않은 상태이거나 동일한 값으로 seed 를 선택하여 발생하는 것들이 많다.

특히 IoT와 같은 임베디드 장치에서 이와 같은 문제가 많이 발생하는데, 우리가 자주 사용하는 rand()의 seed 초기화를 생각해 보면 어떤 문제가 있을지 이해하기가 쉽다.

일반적인 application 의 경우 프로그램 시작 시 다음과 같이 현재시간을 seed 초기화를 하는 경우가 많다.

```c
#include <time.h>
#include <stdlib.h>

void init_seed(void) {
    srand(time(NULL));
}
```

만일 장비가 전원이 꺼진 동안 시간을 유지 시켜주는 RTC(Real Time Clock)이 없다면 어떻게 될까?

이 경우 기기 전원을 켜면 항상 1970년 1월 1일부터 시작되는데, 이렇게 되면 time() 의 리턴값은 항상 동일한 값이 나오거나, 다르게 나와도 수초 이내의 변화 밖에 가지지 않게 된다.

Linux 의 경우 이런 경우를 보완하기 위하여 init script 에서 다음과 같은 처리를 하는 경우가 많다.

- 종료 직전에 최종 시간을 파일에 저장하고, 부팅 직후 저장된 시간으로 현재 시간을 설정한다(나중에 네트워크 연결시 NTP로 재설정)
- 종료 직전에 /dev/urandom 도 일정 바이트를 읽어서 파일로 저장하고, 부팅 직후 이를 다시 /dev/urandom 에 넣어서 seed 로 반영한다.

이렇게 하면 매 부팅 시에는 현재시간도 urandom의 seed 도 다른 값으로 설정되어 문제를 해결할 수 있다.

하지만 정상적인 종료 절차를 거치지 않고 강제로 전원을 꺼버리면, 동일한 seed를 사용하는 문제가 발생한다. 이런 제품들의 경우 저장된 seed를 중간 중간 적절한 시점에서 바꾸어 주도록 보완을 하여야 한다.

이와 같이 잘못된 seed를 사용하게 되면 보안성도 크게 떨어지는 문제가 발생하기도 하지만, 제품의 동작에서 문제가 발생할 가능성도 있다.

예를 들어 부팅 시 동일한 seed를 사용하는 기기가 있고, 부팅 직후 지정된 서버로 TCP 연결을 수행한다고 하면, 이때 사용하는 local port, TCP initial sequence number, applicaiton의 transaction ID 등도 매번 동일한 값이 될 수 있다.
이 경우 중간에 패킷을 필터링하는 NAT/방화벽이나 transaction을 관리하는 서버측에서 이전 세션과 구분을 못하여 오동작을 할 가능성이 생길 수 있다.

임베디드 제품을 개발하게 된다면 초기 seed 가 적절한 방법으로 설정되는지를 반드시 한번은 짚고 넘어가야 한다.

## 마무리

난수는 말그대로 무작위한 TRNG를 원할 때도 있고, 스트림 암호와 같이 무작위 하지만 초기 seed에 따라 일련의 난수열이 만들어지는 PRNG 가 필요한 경우도 있다.
어떤 경우라도 초기 엔트로피/seed 가 어떻게 되는지를 주의해서 확인해 보고, 사용 용도와 보안성에 맞추어 공개된 적절한 랜덤 소스나 함수를 사용하면 된다.
