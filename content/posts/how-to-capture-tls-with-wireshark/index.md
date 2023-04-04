---
title: "Wireshark 으로 TLS 캡쳐 및 디코딩 하기"
date: "2021-12-21T16:00:00+09:00"
lastmod: "2021-12-22T10:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Wireshark", "TLS", "MbedTLS", "OpenSSL", "WebProxy"]
categories: ["Security"]
---

프로토콜을 개발하거나 검증하려고 할 때 SSL/TLS 암호화 채널로 전송되는 데이타를 디코딩하여 확인이 필요할 때가 있다.
이 글에서는 시험하려는 프로그램의 수정 없이 또는 최소한의 수정으로 디코딩 하는 방법을 설명한다.

## 개요

TLS 채널의 초기 셋업 절차는 크게 보면 다음과 같은 절차로 이루어진다.

- 서버 인증서를 받아서 검증하기
- 필요하면 클라이언트 인증서를 받아서 검증하기
- 암호화 방식을 이용하여 대칭키 교환
- 대칭키를 이용한 암호화된 데이타 송수신

패킷을 분석하기에 필요한 사항은 결국은 위 세번째 과정에서 교환한 대칭키(Master Secret)를 얻는 것이다. 다음은 이를 얻을 수 있는 방법 들을 정리한다.

## RSA Private key를 이용한 패킷 디코딩

대칭키를 교환하는 방법은 크게 다음과 같은 두가지 방식을 사용한다.

- RSA와 같은 비대칭키를 이용하여 대칭키 교환
- DHKE(Diffie-Hellman Key Exchange) 방식의 키 교환

이들 중 다음과 같은 조건을 만족하는 경우 서버의 RSA private key를 이용하여 디코딩이 가능하다.

- DH 방식의 키교환은 불가능
- TLS 1.2 이하 인 경우 가능. TLS 1.3은 불가능
- 서버 인증서로만 가능
- TLS handshake message 중 `ClientKeyExchange` 패킷이 캡쳐 된 경우

간단히 말해서 TLS1.2 이하이어야 하고, RSA를 이용한 키교환만 가능하다는 것이다.

예를들어, AWS IoT 서버와 통신을 하는 경우를 디코딩 하려면, 우선 지원되는 TLS cipher suites 목록을 아래에서 확인할 수 있다.

- [Transport security in AWS IoT](https://docs.aws.amazon.com/iot/latest/developerguide/transport-security.html)

이들 목록 중 앞부분의 'ECDHE-' 로 시작하는 것을 제외한 AES128-GCM-SHA256, AES128-SHA256, AES128-SHA, AES256-GCM-SHA384, AES256-SHA256 만 사용하도록 제한하여야 한다.

Wireshark의 설정 -> RSA Keys 에 RSA private key를 등록하면 된다. 관련된 설명은 [Wireshark - Transport Layer Security (TLS)](https://wiki.wireshark.org/TLS)에 자세히 설명되어 있다.

하지만 이 방식은 서버인증서의 개인키를 가지고 있어야 하는데, 임시 시험용 서버가 아닌 이상 서버의 private key를 얻을 수 있는 방법은 없으므로 실제적으로 활용 가능한 경우는 많이 없다.

## Master Secret 을 얻어서 디코딩 하기

### 웹 브라우저

Firefox browser 의 경우 NSS (Network Security Services) 모듈에서 TLS 관련 기능을 처리한다.
참고로 [NSS layer](https://developer.mozilla.org/en-US/docs/Mozilla/Projects/NSS/NSS_API_Guidelines)는 아래와 같은 레이어링을 제공한다.

{{< image src="nss-layer.gif" width="300px" height="auto" caption="NSS Layer">}}

이 NSS 에서 디버깅을 위한 [NSS Key Log](https://developer.mozilla.org/en-US/docs/Mozilla/Projects/NSS/Key_Log_Format) 기능을 제공한다.
이 기능을 켜기 위하여는 환경 변수에 `SSLKEYLOGFILE` 로 로그를 저장할 파일을 지정해 주면 된다. Firefox 이외에도 Chrome 브라우저도 동일한 기능을 제공한다.

아래 예는 mac 의 경우이고, linux 나 windows 도 동일한 방법으로 환경 변수를 설정해서 사용하면 된다. ([설정관련 참고](https://support.f5.com/csp/article/K50557518))

```shell
$ SSLKEYLOGFILE=/tmp/tlskey.log open -n /Applications/Firefox.app
$ SSLKEYLOGFILE=/tmp/tlskey.log open -n /Applications/Google\ Chrome.app
```

위와 같이 환경 변수를 설정하여 브라우저를 실행하면 웹 페이지를 열 때마다 로그파일에 로그가 추가된다. 이 중 대칭키 디코딩에 필요한 라인은 `CLIENT_RANDOM` 으로 아래와 같이 랜덤 값과, master secret 으로 구성된다.

```
CLIENT_RANDOM 5ab65015dc125e2aa00daececfa52ff37489daf6a20375cf140d5f14e291e2ec 64df1871ca4a4be463872aef5f29caff98cfeff4b2a0f0193951947533212dc30766f03b4d202de43f93a265a828e393
```

Wireshark의 설정 -> Protocols -> TLS 의 '(Pre)-Master-Secret log fiename' 에 해당 파일을 설정하면 자동으로 TLS 데이타가 디코딩 된다.

{{< image src="wireshark-tls.png" width="600px" height="auto" caption="Master-Secret 설정">}}

### Node.js

개발하는 프로그램이 node.js 로 작성한다면 위의 웹 브라우저와 동일한 기능을 수행하는 [node-sslkeylog](https://www.npmjs.com/package/sslkeylog) 패키지를 사용할 수 있다.

코드 상에 아래처럼 추가를 하면 SSLKEYLOGFILE 환경 변수로 컨트롤이 가능하다.

```javascript
require("sslkeylog").hookAll();
```

### MbedTLS library

사용하는 기기가 RTOS 기반 IoT 모듈 이라면 TLS 관련 library를 수정하여야 한다. 모듈이 mbedTLS 를 사용한다면 아래 링크과 같이 patch를 적용한다.

- [Example code to produce key log file compatible with Wireshark](https://github.com/Lekensteyn/mbedtls/commit/68aea15)

위 patch 는 위와 유사한 방식으로 fputs 를 이용하여 `CLIENT_RANDOM` 을 출력 시킨다.

이와 같은 시나리오로 WiFI IoT 모듈의 TLS 패킷 캡쳐는 다음과 같이 할 수 있다.

{{< image src="iot-capture.png" width="400px" height="auto" caption="IoT 캡쳐">}}


- IoT 모듈의 디버깅 로그를 UART로 PC와 연결하여 디버깅 로그 출력
- 로그를 [picocom](https://linux.die.net/man/8/picocom)와 같이 terminal 프로그램 이용하여 CLIENT_RANDOM만 추출
- 아래와 같이 두개의 스크립트를 각각 실행

```shell
$ picocom -b 115200 /dev/ttyUSB0 | tee /tmp/test.log
$ tail -f /tmp/test.log | grep --line-buffered CLIENT_RANDOM | sed -l -e "s/\r//g" >> /tmp/tlskey.log
```

첫번째 줄은 로그 출력 결과를 test.log에 임시로 저장. 두번째 줄은 이 로그 중 `CLIENT_RANDOM` 이 들어간 문장만 tlskey.log로 추가 하는 것이다.

위와 같이 한 상태에서 IoT 모듈의 패킷을 캡쳐하면 디코딩이 가능하다.

### OpenSSL library

OpenSSL 도 비슷한 방식으로 캡쳐가 가능하다. OpenSSL 버전에 따라 대응 방법이 다르다.

- OpenSSL v1.1.1 이상인 경우

  - [SSL_CTX_set_keylog_callback()](https://www.openssl.org/docs/man1.1.1/man3/SSL_CTX_set_keylog_callback.html) 함수로 출력 가능

  ```C
  SSL_CTX_set_keylog_callback(mosq->ssl_ctx, SSL_CTX_keylog_cb_func_cb);
  ```

  ```C
  void SSL_CTX_keylog_cb_func_cb(const SSL *ssl, const char *line){
    printf("%s\r\n",line);
  }
  ```

- 이전 버전인 경우 ssl/ssl_lib.c 의 SSL_read() 에서 `(SSL *)s->session->master_key` 를 적절하게 출력시키기

IoT 기기의 경우 가벼운 mbedTLS 를 주로 사용하여 OpenSSL을 이용한 시험은 실제로 해보지는 못하였다.

관련하여 실제 사용예는 아래 링크를 참고할 수 있다.

- [Hijacking openssl renegotiated keys for server wiretaps](https://embeddedinn.xyz/articles/tutorial/hijacking-openssl-renegotiated-keys-for-server-wiretaps/)

## 마무리

지금까지 Master Secret을 얻어서 패킷을 디코딩 하는 방법을 정리하였다.

물론 이와 같이 Master Secret을 구해서 디코딩 하지 않고, Web Proxy tool을 이용하여 중간에서 패킷을 가로채서 확인하는 방법도 있고, 상황에 따라서는 이 방법이 더 편리할 수도 있다.
유명한 툴로는 [Burp Suite](https://portswigger.net/burp)가 있고, 검색을 해보면 [mtimproxy](https://mitmproxy.org)와 같은 오픈소스 툴들도 있다.
이들 툴은 HTTPS 프로토콜만을 대상으로 하고 있어 MQTT over TLS 와 같은 다른 프로그램은 분석이 불가능하다.

시험 타겟이 HTTP proxy 기능 지원 여부에 따라서 다음과 같이 모니터링이 가능하다.

- HTTP proxy 지원하는 경우: proxy 주소만 web proxy 툴이 설정된 곳으로 설정
- HTTP proxy를 미지원하는 경우: DNS 또는 Gateway 주소를 web proxy 툴의 주소로 설정하고, transparent proxy 모드로 모니터링

TLS 를 위와 같은 방법을 이용하여 분석 환경을 구성한다면 좀 더 쉽게 검증하면서 개발이 가능할 것이다.
