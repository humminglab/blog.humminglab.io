---
title: "HTTP SHA-256 Digest Authentication"
date: "2023-07-23T09:00:00+09:00"
lastmod: "2023-07-23T09:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["SHA", "HTTP", "Authentication"]
categories: ["Security"]
---

일반 웹 환경에서는 이제는 사양화 되어 거의 사용하지는 않지만, 
CCTV 표준인 [ONVIF](https://www.onvif.org)나 관련 보안 인증에서는 아직까지 HTTP Digest Authentication 을 사용한다.

이 글에서는 HTTP Digest Authentication의 전반적인 사항을 정리한다. 
목차는 다음과 같다.

- HTTP MD5, SHA-2 (SHA-256) Digest 인증 절차 
- Digest Authentication
- 웹 환경에서 더이상 사용하지 않는 이유
- 브라우저에서 SHA-256 Digest Authentication 구현 방법


## HTTP MD5, SHA-2 (SHA-256) Digest 인증 절차 

HTTP Authentication은 RFC 표준으로 다음과 같이 정의 되어 있다.

- [RFC 2617 - HTTP Authentication: Basic and Digest Access Authentication](https://datatracker.ietf.org/doc/html/rfc2617)
- [RFC 7616 - HTTP Digest Access Authentication](https://datatracker.ietf.org/doc/html/rfc7616)


웹 초창기에 사용자 인증이 필요한 페이지를 위하여  RFC 2617 로 표준이 정의 되었다. 
이 표준안에는 다음과 같은 2가지 인증을 정의한다.

- Basic Authentication
- Digest Access Authentication

Basic Authentication은 사용자 ID, password를 base64 로 encoding 하는 방식으로 보안이 취약할 수 밖에 없다. 

두 방식 모두 기본 보안 인증 절차는 다음과 같다. 

``` mermaid
sequenceDiagram
    Client ->> Server: HTTP Request
    activate Server
    Server ->> Client: 401 Unauthorized<br/>WWW-Authenticate: xxxx
    deactivate Server 

    Client ->> Server: HTTP Request<br/>Authorization: scheme yyyy
    activate Server
    Server ->> Client: 200 OK
    deactivate Server
```

- Client는 별도의 인증없이 GET/POST 와 같은 HTTP 메시지를 전송한다. 
- 해당 URL에 대하여 서버에서는 인증이 필요하다면, 401 Unauthorized 로 오류 메시지로 응답한다.
이때 서버는 `WWW-Authenticate` 로 어떤 종류의 인증이 필요한지와 인증에 필요한 추가 정보를 전달한다. 
  - RFC 2617 에서는 서버는 다음과 같이 두 응답 중 하나를 할 수 있다. 
    - Basic: `WWW-Authenticate: Basic realm="WallyWorld"`
    - Digest: `WWW-Authenticate: Digest realm="test" nonce="xxxxx" ...`
- 브라우저에서는 401 응답에 `WWW-Authenticate`가 basic 이나 digest scheme 이라면 아래 그림처럼 창을 띄워서 사용자 입력을 요청한다.
- 사용자가 입력하여 로그인을 수행하면 이 정보를 기반으로 인증정보를 생성하여 `Authorization` header에 scheme 과 인증 정보를 추가하여 다시 HTTP 요청을 전달한다. 
- 인증이 성공하면 이후로는 브라우저는 자동으로 해당 domain에 대하여 `Authorization` 정보를 자동으로 추가해서 요청을 하게 된다.


{{< image src="basic-input.png" width="400px" height="auto" caption="Basic Auth">}}


관련 필드의 설명은 RFC 2616 HTTP 프로토콜의 해당 필드를 참고할 수 있다. 

- [WWW-Authenticate](https://datatracker.ietf.org/doc/html/rfc2616#section-14.47)
- [Authorization](https://datatracker.ietf.org/doc/html/rfc2616#section-14.8)


2015년에 RFC 7616 으로 버전업을 하면서 가장 큰 변경 사항은 Digest 인증 시 필수 방식 이 MD5 에서 SHA-2 (SHA-256) 로 변경된 것이다.
Proxy, [Authenticateion-Info](https://datatracker.ietf.org/doc/html/rfc2617#section-3.2.3) 나 UTF-8 지원 등 변경된 것들이 있지만, 어쨋든 사람들이 일반적으로 말하는 기준으로 보면 RFC2617은 MD5 Digest 인증이고, RFC7616은 SHA-256 Digest 인증이라고 볼 수 있다.

참고로 이외에도 표준으로 정의된 다양한 인증방식이 있다. IANA에서는 아래 링크로 등록된 authentication scheme을 관리한다. 
- [HTTP Authentication Scheme Registry](https://www.iana.org/assignments/http-authschemes/http-authschemes.xhtml)

이 중 최근에 가장 자주 사용하는 것은 [OAuth 2.0](https://www.rfc-editor.org/rfc/rfc6750.html) 의 bearer scheme이다. 

## Digest Authentication

MD5, SHA-2 (SHA-256)에 대한 추가 설명은 이전 글 [TLS/암호 알고리즘 쉽게 이해하기(10) - Hash]({{< ref "posts/tls-cryptography-10-hash">}})을 참고 할 수 있다.


Digest 인증의 절차는 다음과 같다. (여기에서는 기본으로 사용하는 auth 모드만 설명한다)

- 서버가 응답하는 `WWW-Authenticate` 에는 인증 연산에 사용하는 정보는 다음과 같다.
  - `realm`: 리소스 식별용 문자열. 웹서버에서 "관리자", "사용자"를 구분한다면 이를 구분키 위하여 다른 문자열을 줄 수 있다. 
  - `nonce`: 서버에서 만든 랜덤 문자열
- 클라이언트는 인증 정보 생성 시 다음 값을 생성한다. 
  - `cnonce`: 클라이언트에서 생성한 랜덤 문자열 
  - `nc`: nonce count, 매번 전송 시 마다 1씩 증가하는 카운트 값


실제 예를 들어서 설명해 본다. 아래는 RFC7616의 예제로 서버의 401 응답과 이를 이용하여 digest를 만들어서 재전송한 것이다.
사용자 id는 "Mufasa"이고, 비밀번호는 "Circle of Life" 이다. 

```
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Digest
    realm="http-auth@example.org",
    qop="auth, auth-int",
    algorithm=SHA-256,
    nonce="7ypf/xlj9XXwfDPEoM4URrv/xwf94BcCAzFZH4GiTo0v",
    opaque="FQhe/qaU925kfnzjCev0ciny7QMkPqMAFRtzCUYo5tdS"


GET /dir/index.html HTTP/1.1 
Authorization: Digest username="Mufasa",
    realm="http-auth@example.org",
    uri="/dir/index.html",
    algorithm=SHA-256,
    nonce="7ypf/xlj9XXwfDPEoM4URrv/xwf94BcCAzFZH4GiTo0v",
    nc=00000001,
    cnonce="f2/wE4q74E6zIJEtWaHKaf5wv/H5QzzpXusqGemxURZJ",
    qop=auth,
    response="753927fa0e85d155564e2e272a28d1802ca10daf449
        6794697cf8db5856cb6c1",
    opaque="FQhe/qaU925kfnzjCev0ciny7QMkPqMAFRtzCUYo5tdS"
```

클라이언트는 다음과 같은 문자열을 붙여서 A1, A2를 생성한다. 

- A1 = username:realm:password 
- A2 = method:request-uri

위의 경우 다음과 같이 된다. 

- A1 = "Mufasa:http-auth@example.org:Circle of Life"
- A2 = "GET:/dir/index.html"

A1 과 A2를 각각 hash 함수를 돌려서 소문자 hexdecimal로 표현한 것을 각각 HA1, HA2 라고 하자. 
이를 다시 다음과 같이 문자열로 만들어서 hash 한 것이 최종 digest 값이다. 

- Digest_Input = HA1:nonce:nc:cnonce:qop:HA2

위 예를 직접 openssl 로 만들어 보면 다음과 같다.

```sh
$ echo -n 'Mufasa:http-auth@example.org:Circle of Life' | openssl sha256
SHA2-256(stdin)= 7987c64c30e25f1b74be53f966b49b90f2808aa92faf9a00262392d7b4794232

$ echo -n 'GET:/dir/index.html' | openssl sha256
SHA2-256(stdin)= 9a3fdae9a622fe8de177c24fa9c070f2b181ec85e15dcbdc32e10c82ad450b04

$ echo -n '7987c64c30e25f1b74be53f966b49b90f2808aa92faf9a00262392d7b4794232:\
7ypf/xlj9XXwfDPEoM4URrv/xwf94BcCAzFZH4GiTo0v:00000001:f2/wE4q74E6zIJEtWaHKaf5wv/\
H5QzzpXusqGemxURZJ:auth:9a3fdae9a622fe8de177c24fa9c070f2b181ec85e15dcbdc32e10c82ad450b04' | openssl sha256
SHA2-256(stdin)= 753927fa0e85d155564e2e272a28d1802ca10daf4496794697cf8db5856cb6c1
```

최종 결과 값이 위 예제의 response 결과와 같은 값이 된다. 위 정보들을 모두 추려서 `Authorization` 필드를 채워주면 정상적으로 인증이 된다.

위의 과정을 간단히 정리하면 다음과 같다. 

- HA1은 사용자 비밀번호를 단방향 해쉬하여 서버에 저장한 정보이다. 서버에는 username, realm, HA1 을 저장한다.
- 마지막 hash 과정은 HA1을 키로 포함한 것으로 결과값이 인증을 위한 digest가 된다.

## 웹 환경에서 더이상 사용하지 않는 이유

표준은 이렇게 정의 되어 있지만, 실제로 찾아보면 브라우저나 웹서버에서 digest 인증을 지원하는 경우가 많지 않다. 

대부분의 브라우저에서는 RFC2617 MD5 digest는 지원하지만 RFC7616 은 아예 구현이 되어 있지 않다. 또한 웹 서버의 경우도 마찬가지로 RFC7616 을 지원하지 않는 경우가 많고, MD5 digest의 경우도 지원은 하지만 기본 패키지를 설치하면 이 인증방식이 빠져 있는 경우도 있다.

2023년 7월 기준으로 브라우저를 확인해 보면 다음과 같다. 

| 브라우저 | Digest Auth | 설명 |
|--------|-------------|-----|
| Chrome | MD5 | |
| Edge | MD5 | |
| Safari | MD5 | |
| Firefox | MD5, SHA-256 | [2021년 FireFox 93](https://bugzilla.mozilla.org/show_bug.cgi?id=472823) 에서 SHA-256 지원 |

브라우저의 인증 관련 호환성은 아래 링크가 가장 잘 정리되어 있는 것 같다.
- [MDN Browser compatibility](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization#browser_compatibility)

HTTP Digest Authentication이 이렇게 사양화 된 이유는 크게 다음과 같은 두가지 이유 때문인 것 같다. 

- Man-In-the-Middle-Attach (중간자공격) 에 취약
  - 가장 결정적일 수 있는데, 아무리 강도가 높은 hash 함수를 사용하여도 서버를 검증할 방법이 없기 떄문에 중간자 공격을 대응할 방법이 없다.
  - 이를 근본적으로 해결할 수 있는 방법인 TLS 가 대세가 되고, 댜양한 용도로 활용이 가능한 OAuth 등으로 대체된 상태이다. 
- 서버 저장 방식의 취약성
  - 위 연산에서 HA1 (username:realm:password 조합의 hash 값)을 서버에 저장하게 된다. Linux의 password 저장처럼 단방향 해쉬를 사용한 저장방식이다.
  - 하지만 이 비밀번호를 저장한 파일이 공격에 의하여 탈취되면 피해 범위가 커질 수 밖에 없다. 예를들어 해당 파일에서 한 명의 비밀번호를 안다면, 동일한 비밀번호를 가진 다른 사용자들을 쉽게 찾을 수 있다. 
  - 최근의 비밀번호 저장은 각 사용자별 개별 salt(random 값)를 추가하고, hash 루프를 잔뜩 돌려서 brute-force attack 이나 rainbow table attack 등에 대응할 수 있도록 저장한다. 이런 방법으로 [PBKDF2](https://en.wikipedia.org/wiki/PBKDF2), [bcrypt](https://ko.wikipedia.org/wiki/Bcrypt) 등의 방식이 있다.


결과적으로 대세에 밀려서 RFC7616 은 실제 적용이 거의 되지 않은 상황이다.


## 브라우저에서 SHA-256 Digest Authentication 구현 방법

반드시 SHA-256 Digest Authentication을 브라우저에서 구현을 하여야 한다면 다음과 같은 방법을 고려해 볼 수 있다. 

우선 사용자가 URL을 주소창에 입력하는 방식의 구현은 브라우저에서 기본으로 제공하는 방식으로는 MD5 digest 이외에는 방법이 없다 (SHA-256을 지원하는 firefox 제외).

반드시 구현을 하여야 한다면 XMLHttpRequest 나 fetch API를 이용한 비동기 방식으로 구현하여야 한다. React와 같은 client side rendering 방식으로 구현 시 static data(HTML, CSS, javascript)는 별도의 인증없이 받아오고, 비동기로 얻어오는 RESTful API 에 대해서만 인증을 하는 방식으로 하면 된다.

하지만 실제로 구현을 해보면 401 Unauthorized 메시지를 javasript가 hooking 할 수 있는 방법이 없다. 

```javascript
var DigestAuthRequest = function(method, url) {
    var req = new XMLHttpRequest();
    req.open(method, url, true);
    req.onreadystatechange = function() {
        // 2: received headers,  3: loading, 4: done
        if (req.readyState === 2) {
            // 이곳에서 401 응답과 www-authenticate를 체크하여, authorization 헤더 생성하여 재전송
        } else if (req.readyState === 4) {
            // ...
        }
    }
    req.send();
}
```

위 처럼 구현을 해보아도 앞의 그림 처럼 브라우저에서 사용자정보 입력 창을 띄울때는 어떠한 callback도 javascript에서 받지 못한다. 

이유는 브라우저 구현에서 다음과 같은 조건이면 브라우저가 자체 사용자 정보 입력창을 띄우기 때문이다. 

- 401 응답이고, `www-authenticate` 가 basic 이거나 지원하는 digest 인증인 경우

이를 회피하기 위하여는 다음과 같은 방법 중 하나로 서버에서 수정을 해주어야 한다. 

- 401 Unauthorized 가 아니라 400 등의 다른 오류코드로 변경
- `www-authenticate`가 아니라 `x-www-authenticate` 처럼 header 이름을 변경하거나, `www-authentiate` 필드 digest scheme을 `xdigest`와 같이 비표준으로 변경 

이렇게 하면 브라우저에서 별도의 처리를하지 않아, 위 코드와 같은 방식으로 구현이 가능하다. 

### 그 외의 방법 

서버에서의 응답 수정 없이 대응 하려면 브라우저에 따라 처리를 다르게 하여야 한다. 

- 서버에서 digest 인증은 MD5는 제외하고, SHA-256만 사용하도록 설정한다.
  - 이렇게 하면 SHA-256 digest 인증을 지원하지 않는 browser는 인증창을 별도로 띄우지 않고, 에러 처리를 하므로 javascript에서 처리 할 수 있다. Firefox를 제외한 대부분의 브라우저가 해당된다.
- SHA-256 digest를 지원하는 Firefox 에서는 인증창을 띄우지 않도록 url 에 사용자 정보를 임베딩하여 보낸다. 
  - [URI](https://datatracker.ietf.org/doc/html/rfc3986) 에는 `http://username:password@hostname` 와 같이 사용자 정보를 포함할 수 있다. Fetch API 에서는 이 방식이 지원하지 않지만, XMLHttpRequest 는 이와 같은 사용자 정보를 URL에 추가하는 것이 가능하다. 
  - 해당 기능은 brwoser마다 동작 특성이 다르지만 chrome, firefox 에서는 URL 에 포함된 사용자 정보로 digest 인증을 시도하고 에러를 리턴한다. 하지만 safari 브라우저 에서는 기존 방식처럼 브라우저 자체의 인증창을 띄우게 된다. 

브라우저의 인증창을 열리고, 이곳에 사용자가 정보를 입력하게 되면 이 정보는 javascript에서는 엑세스가 블가능하므로 사용자 인증 절차가 꼬이게 되므로 충분한 검증을 하여야 할 것이다. 브라우저 종류나 버전마다 특성이 달라질 수 있어, 동작을 보장하기 위하여는 충분한 시험을 하여 검증 할 필요가 있다.
