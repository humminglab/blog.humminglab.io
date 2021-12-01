---
title: "Python 비동기 프로그래밍 제대로 이해하기(2/2) - Asyncio, Coroutine"
date: "2018-03-30T17:00:00+09:00"
lastmod: "2018-03-30T17:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Python"]
categories: ["Python"]
aliases: [/python-coroutine-programming-2/]
---


이글은 [Python 비동기 프로그래밍 제대로 이해하기(1/2)]({{< ref "posts/python-coroutine-programming-1">}}) 에 이어서 작성한 글이다.

## Asyncio

Python 3.4 에서는 그동안 [Twisted](https://twistedmatrix.com/), [Tornado](http://www.tornadoweb.org/)와 같이 별도의 library로 제공되던 event loop 방식의 비동기 프로그래밍이 asyncio ([PEP 3156 -- Asynchronous IO Support Rebooted: the "asyncio" Module](https://www.python.org/dev/peps/pep-3156/)) 표준 라이브러리로 새로 추가되었다.

각각의 event loop 구현이 비슷하지만 약간의 차이가 있어서 이들을 혼용하여 사용할 때 차이점을 이해하는데 부담이 있지만, 시간이 지나며 이들도 asyncio로 통합 지원하는 방향으로 되는 것 같다. 참고로 2018년 3월에 새로 릴리즈된 [Tornado 5.0](http://www.tornadoweb.org/en/stable/releases/v5.0.0.html) 부터는 asyncio가 통합되어 단일 interface로 사용이 가능해졌다.

이 event loop는 C 언어에서 사용하는 것과 같은 callback 방식으로도 사용은 가능하지만 coroutine과 같이 사용한다면 큰 장점을 발휘하게 된다. 단일 thread 에 마치 multi tasking을 하는 것과 유사한 기능을 수행할 수 있게 된다. 

이를 이해하기 위하여는 먼저 기존 python의 thread나 process 에서의 concurrent 프로그래밍 방식으로 python 3.2에 추가된 future ([PEP 3148 -- futures - execute computations asynchronously](https://www.python.org/dev/peps/pep-3148/)) 를 이해할 필요가 있다. 이후에 만들어진 asyncio도 이와 동일한 API로 만들어진 것이다.

Future는 쉽게 말해서 work thread(process)의 핸들이라고 볼수 있다. 이를 `future.result()`와 같이 종료가 끝날때 까지 기다리게 되면, 해당 work funtion에서 결과를 완료하거나, exception이 발생한 경우 이를 받을 수 있다.

간단한 예제로 [PEP 3148](https://www.python.org/dev/peps/pep-3148/)에 다음과 같은 web crawler 가 있다.
```python
from concurrent import futures
import urllib.request

URLS = ['http://www.foxnews.com/',
        'http://www.cnn.com/',
        'http://some-made-up-domain.com/']

def load_url(url, timeout):
    return urllib.request.urlopen(url, timeout=timeout).read()

def main():
    with futures.ThreadPoolExecutor(max_workers=5) as executor:
        future_to_url = dict(
            (executor.submit(load_url, url, 60), url)
             for url in URLS)

        for future in futures.as_completed(future_to_url):
            url = future_to_url[future]
            try:
                print('%r page is %d bytes' % (
                          url, len(future.result())))
            except Exception as e:
                print('%r generated an exception: %s' % (
                          url, e))

if __name__ == '__main__':
    main()
```

- `excutor.submit()`으로 thread pool에서 돌릴 함수를 등록하면 future를 리턴한다. 등록된 함수는 thread pool에서 비동기로 실행된다. 
- `futures.as_completed()` 처럼 결과가 완료된 순서되로 리턴되는 generator를 리턴 받을 수 있다. 위와 같이 for..in 에 넣어 loop를 돌릴 수 있다. 완료되거나 비정상 종료된 future가 차례대로 나오게 된다. 
- `future.result()`로 결과를 받을 수 있다. 만일 future내의 함수(`load_url()`)에서 exception이 발생한 것도 future를 통하여 호출한 thread에서 받을 수 있게 된다. 위 예제와 같이 `future.result()`가 try..except 문으로 감싸서 해당 작업에서 발생한 예외도 받을 수 있다.

이와 같이 future가 있으면 child thread에서 발생한 exception도 쉽게 처리가 가능해진다. 

(이 글은 asyncio의 개념을 이해하는데 목적을 둔 것이라 실제 사용법은 자세하게 설명하지 않는다. 사용방법은 링크로 걸어둔 PEP 문서 등을 참조하면 된다)

Asyncio에서도 concurrent.futures.Future와 유사한 asyncio.future를 제공한다. 차이점은 일반 함수가 아니라 coroutine을 전달하는 것이고, `future.result()` 함수가 blocking이 되지 않는다는 정도(단일 thread에서 event loop로 돌기 때문에 blocking 되면 안되므로)의 차이가 있다.  

Asyncio를 이해하는데 혼동이 되는 부분은 이 future 때문이다. 사용 방법을 정리해 보면 다음과 같다.  
- Asyncio는 future 없이도 callback만 사용 가능하다. 이 경우 `call_later()`등의 함수를 이용하여 callback을 등록하고 사용할 수 있다. 하지만 이것만 사용하게 되면 진정한 asyncio의 장점을 살릴 수 없게 된다. `call_later()`등을 이용하여 callback을 등록하면 event loop에서 적절한 시점에 callback을 호출해 준다. 
- Coroutine은 future를 이용하여 사용한다. 실제적으로 future를 직접 사용하지 않고, 이를 상속받은 Task class를 사용한다. Future와 Task의 차이점은 Future는 coroutine을 예외 처리들을 위해 감싼 것이고, Task는 여기에 event loop와 같이 연계한 것이라고 보면 된다.

이제 여기에서 나오는 future, Task, coroutine, yield from, @asyncio.coroutine 등의 용어들을 명확히 이해하기 위하여 정리해보자.

우선 `@asyncio.coroutine` 부터 보면, 이와 같이 @가 붙어 있고, 함수 앞에 쓰면 decorator([PEP 318 -- Decorators for Functions and Methods](https://www.python.org/dev/peps/pep-0318/)]라고 한다.
예를 들어 다음과 같이 decorator를 사용한다. 
```python
@dec1
def func(arg1, arg2, ...):
    pass
```

위 함수는 아래와 동일한 코드가 된다. 
```python
def func(arg1, arg2, ...):
    pass
func = dec1(func)
```

즉 decorator도 함수이다. 함수를 parameter로 받아서 다시 함수를 리턴하는 함수이다. 기존 함수를 변형하는 용도로 사용한다. 예를 들어 기존 함수를 이와 같은 decorator를 이용하여 함수의 입출력을 바꾸거나, trace등을 할 수 있다.
`@asyncio.coroutine`도 이런 decorator이다. 하지만 이 decoreator는 실제로 특별한 기능은 수행치 않고, asyncio와 같이 사용하는 coroutine이라고 표기하는 documentation 목적이다. 즉, 빼고 사용해도 특별히 문제될 것은 없다.

Coroutine은 일반적으로 호출 함수(caller)에서 반복적으로 `next()`, `send()`를 이용하여 yield에 멈추어 있는 coroutine을 재개 시킨다. Coroutine(coroutine A)에서는 내부 적으로 다시 coroutine(coroutine B)을 호출 할 수 있다. 이때는 편리하게 yield from 으로 호출하면 호출된 coroutine B가 yield가 반복되어 최종 리턴될 때 까지 coroutine A는 기다리게 된다. Caller, coroutine A, coroutine B를 같이 놓고 보면 Caller 가 `send()`를 호출 될 때마다 coroutine B의 yield가 풀리는 셈이 된다. 

`send()`를 반복적으로 호출하는 것을 asyncio의 event loop에서 한다고 보면 된다. 이렇게 되면 coroutine도 event loop에서 마치 별도의 thread에서 도는 것과 같이 실행되는 셈이 된다. 이들 coroutine을 event loop에서 관리 하기 위하여는 future에서 상속받은 Task를 사용하는 것이다. 일반 callback 함수는 `call_later()`를 이용하여 event loop에 등록하고, coroutine은 `ensure_future()`나 `loop.create_task()`를 사용하여 등록한다고 보면 된다.

여기에 하나 더 추가된 것이 기존의 yield from에는 iterator, generator(coroutine)이 사용 가능 했는데, 여기에 future도 사용 가능하도록 추가된 것이다. 

이정도면 asyncio에 추가된 coroutine의 개념은 정리된 셈이다.

Asyncio를 사용한 예제는 [asyncio.readthedocs.io](http://asyncio.readthedocs.io/) 를 참조하면 다양한 사용 예제가 있어 이해하기 편하다. 다음절에서 설명하는 async/await를 사용치 않고 python 3.4 기능만으로 사용한다면 callback, coroutine을 혼용하여 사용하는 예는 다음과 같다.

```python
import asyncio

@asyncio.coroutine
def print_every_second_coroutine(type):
    "Print seconds"
    while True:
        for i in range(10):
            print(i, 's (corotine {})'.format(type))
            yield from asyncio.sleep(1)
        loop = asyncio.get_event_loop()
        loop.stop()

def print_every_seconds_callback(i):
    print (i, 's (callback)')
    loop = asyncio.get_event_loop()
    loop.call_later(1.0, print_every_seconds_callback, i+1)

def print_every_seconds_callback_to_coroutine():
    asyncio.ensure_future(print_every_second_coroutine('B'))

loop = asyncio.get_event_loop()
loop.call_soon(print_every_seconds_callback, 0)
loop.call_soon(print_every_seconds_callback_to_coroutine)
asyncio.ensure_future(print_every_second_coroutine('A'))

loop.run_forever()
loop.close()
```

- `print_every_second_coroutine()`은 `asyncio.ensure_future()`를 이용하여 default event handler에 coroutine을 등록한다. 이때는 generator나 future를 등록하여야 하기 때문에 `print_every_second_coroutine('A')` 와 같이 generator를 리턴 받아서 등록한다. 이는 바로 event loop (`loop.run_forever()`)에서 호출된다.
- callback은 print_every_seconds_callback 와 같이 함수 이름을 전달한다. 만일 함수에 parameter 전달 조건이 맞지 않는다면 [`functiontools.partial`](https://docs.python.org/3/library/functools.html)을 이용 할 수 있다. 기본적으로 one shot 이기 때문에 callback 함수에서는 call_later() 등의 method를 이용하여 반복해서 호출해준다.
- `print_every_seconds_callback_to_coroutine()`과 같이 일반 callback 함수에서는 coroutine을 직접 호출할 수 없다(직접 호출하려면 이 함수가 next()를 반복해서 호출하여야 하기때문에 event loop가 blocking된다). 대신, coroutine을 등록 하는 것과 동일하게 `asyncio.ensure_future()`(또는 `loop.create_task()`)를 사용한다.

일반 함수와 generator의 차이점, 그리고 event loop에서 일반 함수 callback 처리와, coroutine의 반복적인 실행 처리의 차이점만 정확히 이해한다면 asyncio에서 제공하는 다른 network나 동기화 관련 nonbocking 함수들에 대해서 쉽게 이해할 수 있을 것이다.

## async, await 
Python 3.5에서는 coroutine을 명시적으로 지정하는 async와 yield를 대체하는 await keyword가 추가 되었다 ([PEP 492 -- Coroutines with async and await syntax](https://www.python.org/dev/peps/pep-0492/)). 이를 기존의 yield를 하는 generator based corourinte과 비교하기 위하여 native coroutine이라고 한다. 앞 절에서 설명한 것과 같이 python의 coroutine (generator based coroutine)은 iterator부터 시작하여 generator를 확장한 것이라, 그 자체가 history를 정확히 모르면 기능 자체가 모호해질 수 밖에 없다. 이를 명확히 정리하고자 새로 native coroutine을 정의한 것 이라고 보면 된다.

기존 generator based coroutine은 함수 내에 yield의 유무로 결정되나, native coroutine은 함수 앞에 async def 키워드를 붙여서 사용한다.
```python
async def read_data(db):
    pass
```

async함수에는 기존 문법인 yield, yield from을 사용할 수 없고, await를 사용한다. 또한 위와 같이 함수안에 await를 사용치 않아도 async def 로 정의된 함수는 coroutine이 된다 (generator based coroutine은 함수안에 yield 여부에 따라서 function인지 generator인지 구분된다).

사용방법은 coroutine에는 def 대신에 async def를 붙이고, 기존의 yield, yield from을 사용하는 자리에 await를 사용하면 된다. 참고로 yeild from과 await와 연산자간의 우선 순위가 차이가 있어, 뒤에 다른 조건문이 붙으면 달라지므로 주의를 하여야 한다. 해당 사항은 [PEP 492](https://www.python.org/dev/peps/pep-0492/)를 보면 잘 나와있다.

기존의 yield from을 대체 하기 위하여 다음과 같은 사항이 await 오른쪽에 올 수 있다.
- native coroutine object
- 기존 generator based coroutine object (정확히는 새로 추가된 `@types.coroutine` decorator를 붙인 generator이어야 함)
- `__await__` method를 가진 object를 리턴하는 iterator
- CPython API를 위한 `tp_as_async.am_await`

위를 자세히 보면 await는 기존의 yield from은 대체가 되나, yield의 완전히 대체할 수 없고, 아래와 같이 generator의 용도로는 사용이 불가능 하다.
```python
yield
yield 10
yield rand()   # 일반 함수
```

즉, generator의 기능은 빼고, asyncio와 같이 비동기 concurrent 프로그래밍을 위한 것이라고 이해하면 된다. Asyncio와 사용하는 용도로만 고려한다면 이정도만 이해하고, 아래에 설명하는 async for, async with만 간략히 보면 사용하는데 별 문제가 없다.

하지만 좀 더 파고들다 보면 또 async for, asynchronous iterator 등 으로 또 혼동이 생길 소지가 있다. 우선 async/await를 사용하여 event loop와 같은 concurrent programming은 어떻게 되는지 봐보자. 아래 예는 Benno blog의 [Playing around with await/async in Python 3.5](http://benno.id.au/blog/2015/05/25/await1)를 이용하여 설명하였다. 이 링크를 보고 직접 이해하는 것도 좋을 것이다.

일단 yield로 두개가 concurrent하게 돌아가는 event loop를 최소로 만들어 보면 다음과 같다.
```python
def coro1():
    print('C1: Start')
    yield
    print('C1: a')
    yield
    print('C1: b')
    yield
    print('C1: end')

def coro2():
    print('C2: Start')
    yield
    print('C2: a')
    yield
    print('C2: b')
    yield
    print('C2: end')

def run(coros):
    coros = list(coros)

    while coros:
        for coro in list(coros):
            try:
                coro.send(None)
            except StopIteration:
                coros.remove(coro)

c1 = coro1()
c2 = coro2()
run([c1, c2])
```


이를 async/await로 바꾸어 보면 우선 첫번째 문제는 위 yield 처럼 뒤에 operand없이 그냥 await만을 사용이 안된다. Await 만으로는 동일한 switch logic을 만들수 없어, 기존 generator base coroutine으로 task switching을 하도록 하여 구현한다.
```python
import types

@types.coroutine
def switch():
    yield

async def coro1():
    print('C1: Start')
    await switch()
    print('C1: a')
    await switch()
    print('C1: b')
    await switch()
    print('C1: end')

async def coro2():
    print('C2: Start')
    await switch()
    print('C2: a')
    await switch()
    print('C2: b')
    await switch()
    print('C2: end')

def run(coros):
    coros = list(coros)

    while coros:
        for coro in list(coros):
            try:
                coro.send(None)
            except StopIteration:
                coros.remove(coro)

c1 = coro1()
c2 = coro2()
run([c1, c2])
```

Python 3.5에서 예외 처리도 보완이 되었는데, 중첩된 coroutine에서 StopIteration이 발생 시 어느 것의 exception 인지 처리가 모호해지는 문제를 위하여 coroutine 밖으로 전파될때는 StopIteration이 RuntimeError로 변경([PEP 479 -- Change StopIteration handling inside generators](https://www.python.org/dev/peps/pep-0479/))되었다.


### Async Internal

Async, await를 좀 더 깊게 들어 가보자.

우선 이와 같이 async/await가 추가되면서 확장된 [data model](https://docs.python.org/3/reference/datamodel.html?#coroutines)을 보면 다음과 같다.

- Awaitable object
  - `__await__()`가 구현된 객체. async def 함수을 호출하여 리턴되는 native coroutine 이 awaitable 객체이다.
  - `object.__await__(self)`에서 iterator가 리턴되어, await에서 사용된다. Future의 경우도 `__await__()`가 구현되어서 await에 사용할 수 있는 것이다.
- Coroutine object
  - Awaitable object 이다. 여기에 `coroutine.send(value)`, `coroutine.throw(type[, value[, traceback]])`, `coroutine.close()`이 구현되어 있다.
- Asynchronous Iterators
  - 기존의 iterator와 비슷하게 `__aiter__()`, `__anext__()` method 가 구현된 객체이다 (기존 iterator는 `__iter__()`, `__next__()`가 구현된 객체이다).
  - 이 객체는 새로 추가된 async for 에 사용할 수 있다.
- Asynchronous Context Managers
  - 기존에 with에서 사용하던 객체와 비슷하게 `__aenter__()`, `__aexit()` method가 구현된 객체이 (기존은 `__enter__()`, `__exit__()`가 구현된 객체이다).
  - 이 객체는 새로 추가된 async with 에 사용할 수 있다.

Awatiable object는 명확하다. 기존의 generator based coroutine과 유사하게 `__await__()`로 iterator를 얻은 후 이를 `send()`를 이용하여 반복되는 구조가 된다.

Asynchronous Context Manager도 개념이 그리 복잡하지 않다. 기존에 사용하던 with를 async 버전으로 만든 것이라고 보면 된다. 이를 기존의 yield from을 with와 같이 사용 시에는 아래와 같다.
```python
with (yield from lock):
    ...
```

이를 async에서는 다음과 같이 사용할 수 있다.
```python
async with lock:
    ...
```

이 기능이 어떻게 풀어지는지 보면,
```python
async with EXPR as VAR:
    BLOCK
````

위와 같은 문장은 다음과 같은 code와 동일하게 실행된다.
```python
mgr = (EXPR)
aexit = type(mgr).__aexit__
aenter = type(mgr).__aenter__(mgr)

VAR = await aenter
try:
    BLOCK
except:
    if not await aexit(mgr, *sys.exc_info()):
        raise
else:
    await aexit(mgr, None, None, None)
```

with 문장이 시작될 때 `__aenter__()`가 호출되어 전처리 작업이 실행되고, exception 등의 모든 조건에서도 자동으로 `__aexit__()`가 호출되어 후처리가 되는 것이다. 위 처럼 Lock과 같은 기능처럼 다시 lock을 풀어주는 작업을 명시적으로 하지 않아도 되기 때문에 편리할 것이다.


하지만 async for를 위한 asynchronous iterator는 좀 불명확하다. await도 iterator인데, 이것 말고 또 다른 iterator를 돌려야 한다는 것이다. 이 부분은 우선 python 3.6 에서 보완된 asynchronous generator([PEP 525 -- Asynchronous Generators](https://www.python.org/dev/peps/pep-0525/))를 같이 보는 것이 이해하기가 쉽다. Asynchronous generator는 간단히 말해서 async 문에도 yield를 사용하여 data producer를 할 수 있는 generator 기능을 추가하자는 것이다. 위에서도 언급한 바와 같이 await로는 `yield 10`과 같이 데이타를 생성하는 로직을 구현 할수가 없다. Python 3.5([PEP 492](https://www.python.org/dev/peps/pep-0492/))에서 이와 같은 기능을 고려한 것이, async for에 사용하는 asynchronous iterator이다.

아래 예는 Quentin Pradet의 [Using asynchronous for loops in Python](https://quentin.pradet.me/blog/using-asynchronous-for-loops-in-python.html) 블로그 글에서 발췌 하였다. 한글로 설명한 부분이 불명확하면 이 블로그 글을 읽어보는 것이 도움이 될 것이다.

우선 기존의 yield 문법으로 network에서 데이타를 받는 generator를 만들면 다음과 같은 구조일 것이다. 네트워크 함수는 blocking mode인 셈이다.
```python
def get_docs():
    page = fetch_page()
    while page:
        for doc in page:
            yield doc
        page = fetch_page()

for doc in get_docs():
    pass  # work on doc
```

for문이 generator를 받아서 루프를 돌면 doc은 `get_docs()`에서 보내오는 doc 데이타를 전달 받게 된다. 이를 비동기 함수로 교체하고 적절히 async를 붙이면 다음과 같이 될 것이다.
```python
async def get_docs():
    page = await fetch_page()
    while page:
        for doc in page:
            yield doc
        page = await fetch_page()

async for doc in get_docs():
    pass  # work on doc
```

이 문장이 python 3.6(PEP 525)에서 지원되는 asynchronouse generator이다. await로 비동기 루프를 도는 것은 돌고, 데이타를 전달하는 것은 yield를 이용하여 전달한다.

이를 python 3.5에서 구현하려면 위와 같이 yield를 사용하지 않고, asynchronouse iterator를 생성하여야 한다. 이는 기존 iterator와 비슷하지만 `__aiter__()`, `__anext__()` method가 있고, iterator가 종료되면 StopAsyncIteration exception이 발생하는 것이다. 이를 사용해서 구현해 보면 다음과 같이 된다.
```python
import collections

class AsyncGetDocs:
    def __init__(self):
        self.buffer = collections.deque()

    def __aiter__(self):
        return self

    async def __anext__(self):
        if not self.buffer:
            await self._prefetch()
            if not self.buffer:
                raise StopAsyncIteration
        return self.buffer.popleft()

    async def _prefetch(self):
        for doc in await fetch_page():
            self.buffer.append(doc)

async for doc in AsycnGetDocs():
    pass  # work on doc
```

새로 구현을 하는 경우라면 python 3.6 문법을 이용하는 것이 편리할 것이다.

참고로 async for는 다음과 같은 문장과 같은 의미를 가진다.
```python
async for TARGET in ITER:
    BLOCK
else:
    BLOCK2
```

```python
iter = (ITER)
iter = type(iter).__aiter__(iter)
running = True
while running:
    try:
        TARGET = await type(iter).__anext__(iter)
    except StopAsyncIteration:
        running = False
    else:
        BLOCK
else:
    BLOCK2
```


## 마치며

이상이 python의 asynchronous와 관련된 문법들이다.
가장 이해하기 좋은 방법은 버전업된 절차를 따라가면서 직접

Python 디버거에 익숙치 않다면 JetBrains의 python IDE 인 [PyCharm Community Edition](https://www.jetbrains.com/pycharm/features/)을 이용해 보는 것도 좋다. Community 버전도 python code를 작성, code inspection, graphical debuger가 지원되면서 무료로 사용할 수 있다. 이를 이용해서 간단하게 코드를 작성하면서 실제 flow가 어떻게 실행되는지, 리턴 되는 값이 어떻게 되는지를 확인해 보면 비동기 절차가 명확히 그려질 것이다.
