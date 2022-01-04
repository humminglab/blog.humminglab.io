---
title: "Nearley 로 설정용 파서 만들기"
date: "2022-01-04T19:00:00+09:00"
lastmod: "2021-01-04T19:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Nearley", "Parser", "JavaScript", "C"]
categories: ["Development"]
---

이 문서에서는 [Nearley](https://nearley.js.org/) parsing toolkit 을 이용하여 IoT 기기에서 사용할 설정 정보의 binary pack 및 loader 를 생성하는 방법을 설명한다.

예를 들어 아래와 같은 간단한 문법을 정의하고, 이를 Nearley 로 parser를 만들어 구분 분석을 하여, 디바이스에 로드할 수 있는 바이너리 데이터로 변환을 한다.
그리고, 변환된 바이너리 파일을 장치에서 로드하여 설정 정보를 얻는다.

```
topic/test1 {
    temperature I8;
    humidity U8;
    pressure U16;
    timestamp U32;
    name STR[12];
}
```



## 배경

아래 그림과 같이 IoT 기기는 기기 고유의 컨트롤을 담당하는 Host MCU와 별도의 Wi-Fi 모듈로 IoT 기능을 구현하는 경우가 많다.
이 경우 Host MCU와 IoT 모듈 사이에는 AT 명령 등을 이용하여 제어를 한다.

{{< image src="iot-general.png" width="600px" height="auto" caption="IoT 연결 구성도">}}

IoT 서비스의 경우 보통 외부 서버와 MQTT protocol 을 통하여 JSON 형식의 데이타를 주고 받는다. Host MCU에서 이를 직접 처리를 하면 좋겠지만, 수 Kbytes의 작은 메모리를 가진 MCU 로 한계가 있을 수 있다. 대안으로 MQTT, JSON 처리를 모두 Wi-Fi 에서 처리하여 Host에는 C 구조체로 바로 매핑가능한 binary 형식으로 전달할 수 있다. 하지만 JSON-to-C 또는 C-to-JSON 변환 룰이 고정된 경우 사양이 변경될 때마다 매번 HOST, WiFi를 업그레이드를 하여야 한다.

여기에서는 빌드된 바이너리 설정 정보를 WiFi 모듈에 로드하여, 이를 이용하여 동적으로 C <=> JSON 을 변환을 하는 기능을 만든다. 이렇게 된다면 WiFi 모듈은 변경없이 HOST MCU 만 변경하여 새로운 데이타 형식 지원이 가능하게 된다.

간단하게 절차를 정리하면 다음과 같이 된다.

- 개발용 PC 에서 설정파일 작성
- 이를 빌드하여 Wi-Fi 모듈에 로드할 binary 데이타와 C 참조 소스 생성
- Host MCU에서 이 정보를 가지고 있고, 이를 초기 실행 시 WiFi 모듈에 로드

이와는 용도가 다르지만 parser 사용이 필요한 경우 아래 내용을 참고해 볼 수 있을 것이다.

## Nearley

[Nearley](https://nearley.js.org/) 는 Javascript 용 parsing toolkit 으로, 이를 이용하면 작성한 구문을 원하는 형식에 맞추어 token 으로 분리할 수 있다.
특히 Nearley의 경우 이들 토큰에 대한 post processing이 간편하여 짧은 코드로 분리한 token을 기반으로 원하는 정보를 만들어 낼 수 있다.

사용 방법은 다음과 같은 단계를 거쳐서 진행한다.

- [Nearley syntax](https://nearley.js.org/docs/grammar) 형식으로 parser의 문법을 작성하여 확장자 .ne 로 저장
- 이를 `nearleyc` 를 이용하여 compile 하여 javascript 소스 생성
- `nearley` 패키지를 이용하여 위에서 생성한 javascript를 로드하여 이를 이용하여 문법을 파싱하는 기능을 만든다.

기본적인 사용 방법은 Nearley의 [Getting Started](https://nearley.js.org/docs/getting-started#nearley-in-3-steps) 부터 참고해 볼 수 있다.

## Parser 구현

관련 소스는 [Github nearley-dev-conf](https://github.com/humminglab/nearley-dev-conf) 를 참조할 수 있다.

### Parser의 입렵 및 출력 결과물

작성하려는 parser는 아래와 같이 MQTT topic 이름과 C 구조체와 유사한 형식의 데이타로 구성된 데이타의 구문 분석기 이다. 
특별한 의미는 없지만 C 와는 다르게 data type을 뒤에 있도록 하였다.

```
topic/test1 {
    temperature I8;
    humidity U8;
    pressure U16;
    timestamp U32;
    name STR[12];
}

topic/array_object {
    name STR[12];

    node[4] {
        subname STR[12];
        info {
            temperature I8;
            humidity U8;
        }
    }
}
```

이를 이용하여 WiFi 모듈에서 JSON <=> C 로 변환할 정보를 생성하고, Host MCU 에서 참고할 C 소스 코드를 생성한다.

### Parser 작성

위 입력을 parsing 하기 위한 ne 파일은 다음과 같다.

```
input -> _ templates _ 
    {% 
        (data) => {
            return { templates: data[1] };
        } 
    %}

templates 
    -> template {% (data) => [data[0]] %}
    | template _ templates {% (data) => [data[0], ...data[2]] %}

template 
    -> key _ "{" _ statements _ "}"
    {% 
        (data) => { return {type: "TEMPLATE", name: data[0], data: data[4]}; }
    %}

statements 
    -> statement {% (data) => [data[0]] %}
    | statement _ statements {% (data) => [data[0], ...data[2]] %}

statement 
    -> key _ basic_data_type _ ";" 
    {% 
        (data) => {
            data[2]['name'] = data[0];
            return data[2];
        }
    %}
    | key _ "{" _ statements _ "}" _ ";"
    {%
        (data) => {
            return {type: "OBJECT", name: data[0], data: data[4]}; 
        }
    %}
    | key _ "[" _ number _ "]" _ "{" _ statements _ "}" _ ";" 
    {% 
        (data) => {
            return {type: "ARRAY", name: data[0], length: data[4], data: data[10]};
        }
    %}

basic_data_type
    -> basic_type {% (data) => { return {type: data[0][0], length: 0}; } %}
    | basic_type _ "(" _ number _ ")" {% (data) => { return {type: data[0][0], length: Number(data[4])}; } %}
    | fixed_type_prefix _ "[" _ number _ "]" {% (data) => { data[0].length = Number(data[4]); return data[0]; } %}

fixed_type_prefix
    -> "STR" {% () => { return {type: "FIX_STR"}; } %}

basic_type
    -> "I8"
    | "U8"
    | "I16"
    | "U16"
    | "I32"
    | "U32"

key -> key_character_first key_character:*  {% (data) => data[0] + data[1].join("") %}
key_character_first -> [_\-a-zA-Z]   {% id %}
key_character -> [_\-a-zA-Z0-9/] {% id %}

number 
    -> digits {% (data) => Number(data[0])  %}
    | digits "." digits {% (data) => Number(data[0] + "." + data[2]) %}
    | "0" [xX] hexs {% (data) => parseInt(data[2], 16) %}

hexs 
    -> hex {% id %}
    | hex hexs {% (data) => data.join("") %}

hex -> [0-9a-fA-F] {% (data) => data[0].toLowerCase() %}

digits 
    -> digit {% id %}
    | digit digits {% (data) => data.join("") %}

digit -> [0-9]   {% id %}

_ 
    -> [ \t\r\n]:*
    | _ "#" [^\r\n]:* "\n" _
    | _ "#" [^\r\n]:* "\r" _
```

이를 nearleyc 를 이용하여 빌드를 하면 js 파일이 생성된다. 

```shell
$ nearleyc -o parser.js parser.ne
```

### Parser 동작 검증

생성된 parser.js 가 정상적으로 수행하는지 확인하려면 [nearley-test](https://nearley.js.org/docs/tooling#nearley-test-exploring-a-parser-interactively) 로 확인해 볼 수 있다.

```shell
$ cat input/test.conf | nearley-test parser.js
...
Parse results:
[
  {
    templates: [
      {
        type: 'TEMPLATE',
        name: 'topic/test1',
        data: [
          { type: 'I8', length: 0, name: 'temperature' },
          { type: 'U8', length: 0, name: 'humidity' },
          { type: 'U16', length: 0, name: 'pressure' },
          { type: 'U32', length: 0, name: 'timestamp' },
          { type: 'FIX_STR', length: 12, name: 'name' }
        ]
      },
      {
        type: 'TEMPLATE',
        name: 'topic/array_object',
        data: [
          { type: 'FIX_STR', length: 12, name: 'name' },
          {
            type: 'ARRAY',
            name: 'node',
            length: 4,
            data: [
              { type: 'FIX_STR', length: 12, name: 'subname' },
              {
                type: 'OBJECT',
                name: 'info',
                data: [
                  { type: 'I8', length: 0, name: 'temperature' },
                  { type: 'U8', length: 0, name: 'humidity' }
                ]
              }
            ]
          }
        ]
      }
    ]
  }
]
```

위의 결과와 같이 Nearley [postprocessor](https://nearley.js.org/docs/grammar#postprocessors) 에 `{% %}` 문 안의 코드 추가로 간단하게 JSON 형식으로 구조화된 데이타를 생성할 수 있다.
이를 이용하여 바이너리 형식으로 패킹된 데이타나 C 소스 파일 형식으로 출력만 하면 된다.

[nearley-railroad](https://nearley.js.org/docs/tooling#nearley-railroad-automagical-railroad-diagrams)로 비주얼하게 보면 아래 링크와 같다.

```shell
nearley-railroad parser.ne -o grammar.html
```

[Railroad Diagram](grammar.html)

### 바이너리로 변환

위와 같이 파싱된 정보를 이용하여 다음과 같은 바이너리 데이타로 변환한다.

- String 만 추출하여 별도의 string table 만듬
- 파싱된 각 element 정보를 패킹
- 위 정보를 헤더를 추가하여 바이너리로 패킹

위 코드는 [index.js](https://github.com/humminglab/nearley-dev-conf/blob/main/builder/index.js) 로 구현되어 있다. 참조를 위해서는 이 파일 보다는 빌더안에 있는 [loader.c](https://github.com/humminglab/nearley-dev-conf/blob/main/loader/loader.c)로 읽어들이는 부분을 더 이해하기 좋다.

## 빌더 만들기

입력 파일로 최종 원하는 C 소스 코드와 바이너리로 패킹된 데이타는 아래 링크에 있는 소스코드와 같이 구현한다.

- [loader.c](https://github.com/humminglab/nearley-dev-conf/blob/main/loader/loader.c)

코드는 간단하게 바이너리 정보를 읽어서 원 소스 파일과 유사한 형태로 결과를 출력한다.

```
Header ID: 0x5aa5
Total size: 226
CRC16: 0x3244
Number of strings: 10
Number of templates: 2
Offset string index: 18
Offset string table: 40
Offset template index: 134
Offset template table: 140

========================================

topic/test1
{
 temperature I8;
 humidity U8;
 pressure U16;
 timestamp U32;
 name STR[12];
}
topic/array_object
{
 name STR[12];
 node ARRAY[4] {
  subname STR[12];
  info OBJECT {
   temperature I8;
   humidity U8;
  };
 };
}
```

Host, WiFi IoT 모듈 모두 제한된 기기에서 상대적으로 복잡한 동적 설정 정보를 사용하는 경우 위와 같은 방법을 사용하게 되면 적은 메모리에서 효율적으로 사용 할 수 있을 것이다.