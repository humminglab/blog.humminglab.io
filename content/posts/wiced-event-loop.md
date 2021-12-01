---
title: "WICED eventloop library"
date: "2018-04-17T17:00:00+09:00"
lastmod: "2018-04-17T17:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["OrangePi"]
categories: ["Embedded Linux"]
aliases: [/wiced-event-loop/]
---


[WICED](http://www.cypress.com/products/wiced-software)와 같은 임베디드 디바이스용 SDK는 [FreeRTOS](https://freertos.org), [ThreadX](https://rtos.com/solutions/threadx/real-time-operating-system/)와 같은 RTOS의 multi tasking 기능을 이용하여 여러개의 task를 생성하여 주변기기를 제어하거나 네트워크로 데이타 송수신 한다. 일반적인 산업용 기기의 센서 동작은 realtime 요구 사항에 맞추어 task로 분리하여 작성하면 된다. 하지만 가정용 IoT 기기를 만들다 보면 이와 같은 multi task 방식 보다는 하나의 task에서 event driven 방식으로 구현을 하는 것이 편리할 때가 있다.

이 문서에서는 가정용 기기의 특징과 이를 task 방식으로 구현하였을 때의 단점을 설명하고, 구현한 event loop library를 설명한다.

## 가정용 기기의 user interface 및 task 구현

산업용 기기라면 경고 상황을 명확하게 인식할 수 있는 부저 알림, LED의 색상이나 깜빡임을 이용한 상태 표시가 사용자 인터페이스의 주요 수단이다. 하지만 가정용 기기의 경우에는 사용자의 감성을 고려하여 좀 더 다양한 인터페이스를 고려하여야 한다. 예를 들면 다음과 같은 사항이 있다.

- LED를 한번에 켜고 끄는 것이 아니라 timer를 이용하여 점차로 밝게 또는 어둡게 하기, 또는 이를 반복하여 breathing 효과 내기(PWM)
- LED display 상에서의 부드러운 메뉴 스크롤 또는 그래프 표시
- 부저를 이용한 부드러운 울림 소리 구현
- 사용자의 인터럽트(버튼, 네트워크 동작)에 따른 빠른 UI 반응성

이를 task로 구현하기 위하여는 WICED를 예로 들면 다음과 같이 구현되어야 한다.

```c
void led_task(wiced_thread_arg_t arg)
{
    ...
    while (1) {
       /* 다른 task에서 trigger */
       result = wiced_rtos_wait_for_event_flags(...);

       if (led_ramp_up) {
           for (i = 0; i < 100; i++) {
              led_pwm(i);
              wiced_rtos_delay_milliseconds(100);

              if (user_break) {
                  user_break = 0;
                  return;
               }
           }
       }
    }
}

void cancel_led_ramp_up() {
    user_break = 1;
    wiced_rtos_thread_force_awake(led_thread);
    wiced_rtos_thread_join(led_thread);
}
```

위 코드는 task 방식의 불편함을 예를 들기 위한 것으로 설명을 위한 주요 부분만 구성한 것이다. `led_task()`에서는 event 입력이 있으면 delay loop를 돌면서 LED를 점차로 밝게 하는 루틴이다.
만일 이와 같이 밝게 하는 도중에 사용자가 버튼을 누른 경우 빠르게 반응을 하기 위하여는 동작 중간에 중단 시켜야 한다. `wiced_rtos_delay_milliseconds()`와 같이 blocking 함수를 중단 시키기 위하여는 OS에서 강제로 리턴시키는 `wiced_rtos_thread_force_awake()`와 같은 함수를 이용하여 해당 task의 blocking 을 중단 시키고, 위의 예처럼 user_break와 같은 flag를 이용하여 task 실행을 중지 하여야 한다(system call이 retur 값이 있는 경우에는 error 값을 보고 판단도 가능하다). 중지를 시키는 task에서는 동기화가 필요하다면 `wiced_rtos_thread_join()`을 이용하여 task가 종료되는 것을 기다여야 한다.

여러 task 가 있는 경우 이들 task의 동기화도 고려하여야 하기 때문에 사용자 효과를 만족할 만하게 내기 위하여는 신경 쓸 부분이 많아진다. 또한 embedded 기기에서는 보통 RAM도 128KiB 이하이기 때문에 생성할 수 있는 task의 개수도 제한이 있다.

이런 부분을 GUI 프로그램처럼 timer와 event driven의 callback 방식으로 단일 task에서 구현한다면 빠른 반응성을 가지면서 동기화 문제 등 multi tasking시 고려하여야 할 사항들을 고민 하지 않아도 되어 좀더 쉽게 구현이 가능해진다.

## WICED event loop

Embedded SDK에 따라 event loop 라이브러리를 기본 제공하는 경우도 있으나 WICED에는 기본으로 제공하지 않아 별도로 구현을 하였다(참고로 [Mbed](https://www.mbed.com/en/)의 경우 [event loop library](https://os.mbed.com/blog/entry/Simplify-your-code-with-mbed-events/)를 제공한다). 소스는 아래 

* github: [https://github.com/humminglab/wiced-eventloop](https://github.com/humminglab/wiced-eventloop)

common/eventloop.c 에 구현된 event loop는 다음과 같은 특징을 가진다.
- WICED Event Flags를 이용한 event 처리
- Continuous timer 제공
- 동적인 메모리 사용을 하지 않음
- Nested inner loop 지원


기본 예는 다음과 같다.

```c
#define EVENT_SENSOR_FINISHED           (1 << 0)

eventloop_t evt;
eventloop_timer_node_t timer_node;
eventloop_event_node_t event_node;

int func()
{
    a_eventloop_init(&evt);
    a_eventloop_register_timer(&evt, &timer_node, timer_callback_function, 500, 0);
    a_eventloop_register_event(&evt, &event_node, event_callback_function, EVENT_SENSOR_FINISHED, 0);

    while (1) {
        a_eventloop(&evt, WICED_WAIT_FOREVER);
    }
}
```

timer나 event를 위한 handle의 경우에도 미리 staic으로 생성한 구조체를 사용하여 동적인 메모리 할당을 사용하지 않도록 구성되어 있다.

예제 코드는 이 event loop를 이용하여 센서나 버튼의 상태를 모니터링하고, MQTT 네트워크 송수신을 구현한 것이다. 코드에서 dns lookup을 제외하고는 모두 nonblocking 방식으로 구현하였다(DNS 처리 루틴도 work thread를 이용하여 background로 처리를 하여야 하나 자주 호출되는 구조는 아니라서 미구현). 예제에 있는 MQTT client 루틴은 [ThingsBoard](https://thingsboard.io)와 연동을 위한 코드이다.

Event driven, callback 구조로 코드에서 주의할 사항은 다음과 같이 두 가지 사항이 있다.
- Event loop 함수는 다른 task에서 호출할 경우 데이타가 깨지거나 정상적으로 callback이 호출되지 않을 수 있다.
- Callback function 이 blocking 된다면 모든 event loop가 blocking된다.

첫번째 사항은 event loop 가 single task 에서 사용하는 것을 기반으로 작성되어 있기 때문이다. 다른 task에서 `a_eventloop_set_flag()`를 호출하는 것은 가능하지만 `a_eventloop_register_timer()`은 list를 엑세스 하여야 하기 때문에 동기화 문제가 발생할 수 있다.

Blocking 함수는 sys_worker.h 에서 구현한 것처럼 작업을 work thread에서 처리하고 event flag로 event loop가 실행되는 task로 전달하여 완료 callback을 처리하는 방법으로 구현 할 수 있다.
sys_mqtt.c 에서는 이 callback 구조를 좀 더 쉽게 처리하기 위하여 nested inner loop 형식으로 `mqtt_inner_event_loop()`를 구현하였다. 이를 사용하면 coroutine과 비슷하게 callback 이 아닌 sequential한 코드로 처리가 가능해진다. 다만, 호출되는 함수내에서 해당 callback을 재호출하게 되면 stack overflow 가 발생하거나 race condition과 같이 이상한 동작이 발생할 수 있어서 제한된 범위로 용도를 한정하여야 한다.

추가적으로 event driven 구조의 장점으로 들 수 있는 것은 각각의 모듈이 sys_xxx.c 처럼 library화가 쉬어, 초기화 시에 해당 함수를 넣어 주거나 빼는 것으로 쉽게 모듈의 추가/삭제가 쉽다.