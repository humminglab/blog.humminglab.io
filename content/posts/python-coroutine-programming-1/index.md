---
title: "Python 비동기 프로그래밍 제대로 이해하기(1/2) - Asyncio, Coroutine"
date: "2018-03-26T23:00:00+09:00"
lastmod: "2018-03-26T23:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Python"]
categories: ["Python"]
aliases: [/python-coroutine-programming-1/]
---

Python2 와 비교하여 python3의 가장 돋보이는 killer feature 는 비동기 프로그래밍 지원이라고 할 수 있다. 이를 위하여 [python 3.4](https://www.python.org/download/releases/3.4.0/)에 asyncio 모듈이 추가되었고, [python 3.5](https://www.python.org/downloads/release/python-350/) 에는 native coroutine 지원을 위한 async, await 키워드가 추가되었다. 이들 기능을 이용하면 javascript나 다른 언어에서 지원하는 비동기 프로그래밍의 장점을 python 에서도 사용할 수 있다. 즉,  이벤트 방식이지만 blocking 방식의 프로그래밍 처럼 sequential 하게 코드를 작성할 수 있어, 단일 thread로 수만개의 네트워크 연결을 처리하는 서버를 오류 가능성을 최소화 하면서, 보다 편하게 개발할 수 있다. 

하지만 이 기능을 제대로 이해하려고 python 매뉴얼을 보다 보면 iterator, generator, yield, coroutine 정도 까지는 큰 어려움 없이 이해가 되나, asyncio, yield from, asyncio.coroutine, future, async, await, async with, async for, ... 비슷 비슷한 단어들의 설명이나 예제를 보다보면 각 기능의 차이점들이 무엇인지, coroutine이 event loop의 callback과 어떻게 같이 사용 되는지 알쏭달쏭 해지면서 머리 속이 점점 복잡해 진다. 

이 글에서는 이와 같은 모호함을 지나 어느 정도 구조를 이해할 수 있게 된 상태에서, 새로 접근하는 사람을 위하여 이해하기 쉽도록 다시 정리해 본다. 

다음과 같은 순서로 설명을 한다. 
- iterator 
- yield 키워드와 generator
- yield from
- asyncio
- async, await 
- future

이 글에서의 python 문법은 python3 으로 설명한다. Python2와의 차이점은 필요 시 언급은 되어 있으나 python2에서 실행하면 결과가 다르거나 실행이 되지 않을 수 있다. 

글은 총 두편으로 나누어서 작성할 예정이다. 
이 글은 yield from까지 정리하고, 나머지를 별도로 작성할 예정이다.
- [Python 비동기 프로그래밍 제대로 이해하기(2/2)]({{< ref "posts/python-coroutine-programming-2">}})

## Iterator
Coroutine과 직접적인 관련은 없지만, 이해를 쉽게 하기 위하여는 iterator 부터 설명을 시작하여야 한다.

기존의 list 형태는 각 원소가 실제 메모리에 할당된다. 예를 들어 `[1,2,3,4]`와 같이 list를 정의하면 4개의 원소를 저장할 공간이 필요하다. 하지만 수십만개의 저장공간이 필요한 아래와 같은 코드는 잠재적인 메모리 부족 문제를 만들 수 있다. 

````python
sum = 0
for i in range(100000):
    sum += i
````

물론 위 사항은 python2 에만 해당된다. python3의 경우 range() 함수의 경우 list가 아니라 iterator를 리턴하여 실제로 데이타가 할당되지 않는다.

이와 같이 이전 원소로 다음 원소를 계산할 수 있는 데이타 구조체라면 굳이 메모리를 할당하지 않고, 다음 값을 리턴 해주는 next() method를 제공해주는 객체가 있다면 유사한 기능을 될 것이다. 

이와 같은 생각에서 만들어 진 것이 iterator이다. Python에서 iterator라는 것은 `__iter__()`와 `__next__()` method를 가진 객체를 말한다. 
- `__iter__()`: Iterator object(`__next__()` method를 가진) 객체를 리턴
- `__next__()`: 호출될 때 마다 다음 값을 리턴

즉, 해당 객체에서 `__iter__()` method를 이용해서 iterator를 얻고, 이의 `__next__()` method를 이용해서 반복해서 값을 얻는 것이다. (Python2에서는 `__next__()` method가 아니라 `next()` method 이어야 한다)

```python
class Counter(object):
    def __iter__(self):
        iter = Iterator()
        return iter
	
class Iterator(object):
    def __init__(self):
        self.index = 0
	
    def __next__(self):
        if self.index > 10:
            raise StopIteration
        n = self.index * 2
        self.index += 1
        return n
```

위 예는 짝수를 리턴하는 iterator이다. 
실제 확인은 iterator의 다음 값을 얻는 builtin `next()` 함수를 사용할 수 있다.
```python
>>> c = Counter()
>>> next(c)
0
>>> next (c)
2
```

Iteration의 종료는 위 예와 같이 StopIteration exception을 발생 시키면 된다. 이와 같이 작성된 코드는 다음과 같이 for loop에 바로 사용할 수 있다.

```python
>>> c = Counter()
>>> for i in c:
        print(i)   # 0~20까지 짝수 출력 
``` 

즉, for 루프에 객체를 넘기는 경우 `__iter__` method가 있는 지를 확인하여 iterator로 동작되도록 언어 자체에서 지원하는 것이다. 

Iterator의 장점은 위와 같이 loop에 사용할 수 있다는 것이다. 이 의미는 iterator로 data structure와 algorithm을 분리할 수 있다는 것이다. 

이렇게 해서 Python 2.1 (참고: [PEP 234 -- Iterators](https://www.python.org/dev/peps/pep-0234/)) (PEP는 Python Enhancement Proposal이다. Python의 새로운 기능은 모두 이같은 proposal이 있다고 보면 된다) 부터 Iterator 기능을 지원하였다. 

정리를 해보면 다음과 같다. 

* _Iterator_
  - python for loop는 객체에 `__iter__()` method가 있으면 이를 이용 iterator를 얻음
  - Iterator의 `__next__()`로 StopIteration exception 이 발생할 때 까지 반복하여 값을 얻어 loop를 반복 수행
  - 동일한 동작은 built-in 함수인 `next()`를 이용하여 사용 가능

## Generator (yield 키워드)
Iterator를 좀더 편하게 사용하는 방법은 다른 언어처럼 yield 를 이용하여 coroutine을 지원하는 것이다. Python 2.2 ([PEP 255 -- Simple Generator](https://www.python.org/dev/peps/pep-0255/))에서는 이와 같은 lightweight coroutine 지원이 추가되었다. 이를 generator라고 부른다. 

Generator 코드 예는 다음과 같다.

```python
def test1():
    print('print 1')
    yield 1
    print('print 2')
    yield 2
	
def test2():
    for i in range(10):
        yield i * 2
``` 

Generator는 함수안에 yield keyword가 있다는 것을 제외하고는 일반 함수와 동일하지만, 이 함수를 호출하면 완전히 다른 동작을 한다. 

- 일반 함수 호출 시: 함수의 body 가 실행됨 
- Generator(yield가 있는 함수) 호출 시: Generator가 실행되는 것이 아니라 이 함수를 감싸는 'generator' 객체가 리턴된다. Generator 객체는 iterator와 동일하게 `__next__()` 를 가진 객체이다. 

위 첫번째 함수를 실행하면 결과는 다음과 같다.

```python
>>> g = test1()
>>> type(g)
generator
>>> dir(g)
['__class__',  '__del__',  '__delattr__',  '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__iter__', '__le__', '__lt__', '__name__', '__ne__', '__new__', '__next__', '__qualname__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', 'close', 'gi_code', 'gi_frame', 'gi_running', 'gi_yieldfrom', 'send', 'throw']
>>> next(g)
print 1
1
>>> next(g)
print 2
2
>>> next(g)
StopIteration
```

1. `test1()` 이 호출하여 리턴된 결과는 generator object이다 .
1. `dir()`로 확인해 보면 `__next__()`가 있는 것을 확인할 수 있다. 즉, iterator와 동일한 방법으로 실행될 수 있다는 것이다. 
1. `test1()`을 하며 바로 실행되지 않는 것은 'print 1' 문장이 출력되지 않는 것으로 확인할 수도 있다. 
1. 첫번째 next(g)를 실행하면 yield를 만날 때까지 실행된 후 여기에서 generator  동작은 멈추고 결과를 리턴한다. 결과는 yield 뒤에 있는 값이다. 출력을 보면 'print 1'이 실행되고 yield 1의 1리 리턴된 것을 확인 할 수 있다. 
1. 다시 `next(g)`를 실행하면 'print 2'가 출력되고, yield의 2가 리턴된다. 
1. 다시 `next(g)`를 호출하면 generator가 종료되면서 StopIteration exception이 발생한다. 

이와 같은 generator은 yield를 이용하여 중간에서 멈추고 결과를 받는 용도로, 실제 주용도는 위의 `test2()`와 같은 iterator를 쉽게 만드는 것일 것이다. 위 iterator의 Counter class 예와 비교해 보면 코드가 훨씬 간결해 진 것을 볼 수 있다. 

생성은 iterator와 generator가 다른지만, 생성된 후의 동작은 완전히 동일하다. 

```python
for i in x:
    print i
```

Python에서는 iterator, generator가 추가되면서  위의 예와 같은 for loop는 대략 아래와 같은 pseudo code로 번역된다고 이해하면 될 것 같다 (정확히는 iter()함수를 사용한다).

```python
if '__next__' in x:   # for iterator or generator
    try:
        while True:
            i = next(x)
            print  i
    except StopIteration:
        break
    else:
        raise
else:  # for list
    idx = 0
    while True
        i = x[idx]
        print i
        idx += 1
``` 

참고로 python 2.4 에서는 더 간결하게 generator를 만드는 것을 지원한다 ([PEP 289 -- Generator Expressions](https://www.python.org/dev/peps/pep-0289/)). 
 
Python에서 list를 만들 때 다음과 같이 만들 수 있다. 
```python
>>> [ x * 2 for x in range(10) ]
[0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
```

위의 대괄호를 소괄호로 바꾸면 list가 생성되는 것이 아니라 generator가 생성된다.
```python
>>> (x * 2 for x in range(10))
<generator object <genexpr> at 0x1072263b8> 
```

### Exception 처리
generator 안에서 동작 중 발생하는 exception은 일반 함수와 마찬가지로 처리된다. 
예를 들어 아래 코드를 실행하였을 때의 결과이다 .
```python
>>> def f():
        return 1/0
>>> def g():
        yield f()  # the zero division exception propagates
        yield 42   # and we'll never get here
>>> k = g()
>>> next(k)
ZeroDivisionError: integer division or modulo by zero
>>> next(k)  # and the generator cannot be resumed
StopIteration
```

위와 같이 generator안에서 0 으로 나누기를 실행하면 해당 exception은 next()를 호출한 부분으로 정상적으로 전달된다. 

일반 exception과 차이가 나는 부분은 다음과 같이 두가지 이다. 
- 위처럼 한번 exception이 발생한 generator는 더 이상 실행할 수 없다. 위 처럼 next()를 다시 호출해도 42가 리턴되는 것이 아니라 StopIteration으로 generator가 종료되었다고 exception을 발생한다. 
- generator 함수에서는 try/except 는 사용 가능하나, try/except/finally 처럼 finally를 사용할 수 없다. 

두번째는 실제 구현의 복잡함 때문이다. (다음에 설명하는 'Yield 기반의 coroutine'으로 기능이 확장되면서 try-finally도 정상적으로 지원한다). 
```python
>>> def f():
    try:
        yield 1
        try:
            yield 2
            1/0
            yield 3  # never get here
        except ZeroDivisionError:
            yield 4
            yield 5
            raise
        except:
            yield 6
            yield 7     # the "raise" above stops this
    except:
        yield 8
    yield 9
    try:
        x = 12
    finally:
        yield 10
    yield 11
>>> print list(f())
[1, 2, 4, 5, 8, 9, 10, 11]
```

위의 예를 보면 yield 문장이 있는 라인은 finally를 제외한 try/except/raise는 사용이 가능하고, generator에 yield가 없는 부분은 finally도 사용 가능하다. 

여기까지는 loop를 편하게 사용할 수 있는 generator에 대한 설명이었다. 이후부터는 본격적으로 coroutine을 설명한다.


## Yield 기반의 coroutine

위의 generator의 yield는 generator 함수내의 코드에서 호출하는 곳으로 데이타를 전달하는 것으로 볼수 있다. 즉 yield가 실행될 때 마다 해당 함수는 멈추고, 값을 `next(x)`를 호출한 함수로 전달하는 셈이다. 

참고: 이 부분부터 generator와 coroutine이 혼용되어 설명하되는데, yield 기반의 coroutine은 generator와 동일하다고 보면 된다. Python이 버전업 되면서 generator의 기능이 yield 기반의 generator로 확장되었다고 이해하면 된다. 나중에 설명하겠지만 python 3.5 에서는 async로 coroutine을 지원하는데, 이는 native coroutine이라고 한다. 

```python
def callee():
    yield 1
    yield 2
		
x = callee()
i = next(x)
i = next(x)
```

위의 그림을 도식적으로 그려보면 아래와 같이 된다(아래 부터는 표기를 편하게 하기 위하여 generator를 생성하거나 next()를 호출하는 곳을 caller라고 표기한다).
 
<div class="mermaid">
sequenceDiagram
    Participant Caller
    Participant Callee
    Note left of Caller: i=next(x)
    Caller->>Callee: 제어권 넘김
    Note right of Callee: yield 1
    Callee->>Caller: 값 1 반환 
    Note left of Caller: i=next(x)
    Caller->>Callee: 제어권 넘김
    Note right of Callee: yield 2
    Callee->>Caller: 값 2 반환 
</div>


- `next()`가 호출되는 시점에서 제어권을 generator로 넘김
- generator내의 코드는 yield를 만날때까지 실행되고, 이때의 값을 호출한 곳을 전달한다. 

결과적으로 보면 두개의 함수가 제어권을 핑퐁하면서 값을 전달하는 셈이다. 하나 빠진 것이 오른쪽에서 왼쪽으로 값은 전달할 수 있지만 반대 방향으로는 값을 전달하지 못한다. 

이와 같은 부분을 보완하여 상호간에 데이타와 함께 제어권을 전달하는 방법이 python 2.5에 yield 기반으로 확장된 coroutine 이다([PEP 342 -- Coroutines via Enhanced Generators](https://www.python.org/dev/peps/pep-0342/#new-generator-method-send-value))

여기서 부터 python은 단일 thread로 다수의 작업(coroutine)을 concurrent 하게 실행 할 수 있는 coroutine이 갖추어진 셈이다.

이를 위하여 다음과 같은 사항이 추가되었다. 
- `x = yield 1` 과 같이 yield 키워드에서도 값을 받을 수 있다. 
- `next()`가 아닌 `send()`함수가 추가 되어 이를 이용하여 caller는 coroutine의 yield 에 값을 전달 할 수 있다. 
- Coroutine 실행동안 exception 처리를 지원한다 .

다음은 간단한 yield와 send를 이용한 예이다.
```python
def coroutine1():
    print('callee 1')
    x = yield 1
    print('callee 2: %d' % x)
    x = yield 2
    print('callee 3: %d' % x)
	
task = coroutine1()
i = next(task)    # callee 1 출력, i는 1이 됨 
i = task.send(10) # callee 2: 10 출력, i는 2가 됨 
task.send(20)     # callee 3: 20 출력 후 StopIteration exception 발생
``` 

<div class="mermaid">
sequenceDiagram
    Participant Caller
    Participant C as coroutine1
    Note left of Caller: i=next(task)
    Caller->>C: 제어권 넘김
    Note right of C: print('callee 1')  x = yield 1
    C->>Caller: 값 1 반환 
    Note left of Caller: i=task.send(10)
    Caller->>C: 값 10과 제어권 넘김. Callee는 x로 값 받음
    Note right of C: print('callee 2: %d') x = yield 2
    C->>Caller: 값 2 반환
    Note left of Caller: i=task.send(20)
    Caller->>C: 값 20과 제어권 넘김. Callee는 x로 값 받음 
</div>

Coroutine은 generator와 yield 에서 값을 받을 수 있다는 것을 제외하고는 모든 것이 동일하다. `coroutine1()`을 실행하면 동일하게 코드가 실행되는 것이 아니라 generator 객체가 리턴된다.
이를 실행 시키려면 마찬가지로 처음에 `next(task)` 처럼 실행해 주어야 한다. 이렇게 되면 yield 를 만날때 까지 실행된다. 이때 'callee 1' 문장이 출력되고, `next()`로 값 1을 반환한다. 
이후 부터가 generator와 사용방법이 다른 부분이다. 처음 한번 `next()`를 호출한 이후로는 `coroutine.send()`를 이용하여 `next()` 와 유사하게 coroutine의 동작을 재개한다. `next()`와의 차이점은 값을 coroutine으로 전달 할 수 있다는 것이다. 위 예에서는 10을 coroutine에 전달하여 'callee 2: 10'이 출력되고 다음 yield를 만나서 다시 2를 리턴한다. 

처음에 `next()`를 호출하여야 한다는 부분이 조금 맘에 들지 않지만, 어쨋든 비동기 프로그램이 가능한 구조가 갖추어 졌다. 

하지만 이것만으로는 부족하다. Generator의 경우 generator 에서 caller로는 try-finally 제한은 있지만 exception을 전달할 수 있다. 또한 coroutine의 마지막 라인까지 실행되면 StopIteration exception으로 종료를 알릴 수 도 있다. 그렇지만 반대로 caller에서 coroutine으로 exception을 전달할 방법이 없고, caller를 parent task라는 개념으로 봤을 때 child인 coroutine을 종료시킬 방법도 없다. 
이를 지원하기 위하여 다음과 같은 사항이 추가되었다. 

- generator에서 yield문은 try-finally로 감쌀 수 없었으나, python 2.5부터는 이를 지원한다. 이렇게 되면 coroutine->caller 로의 예외 전달은 모두 지원하게 된다. 
- Caller에서 yield(또는 생성 직후)로 멈춰있는 coroutine에 excetion 전달 지원. `send()`와 비슷한 방법으로 `throw(type, value, traceback)` 처럼 exception을 coroutine에 전달할 수 있다. Parameter는 raise의 parameter와 동일하다. 
- Caller에서 coroutine을 종료 시킬 수 있는 `close()` 도 추가하였다. 이를 위하여 GeneratorExit exception이 추가되었고, `close()`는 `throw()`를 이용하여 GeneratorExit exception을 coroutine에 전달한다. 

위와 같이 exception과 종료가 추가되어서, 비동기 방식의 프로그래밍을 위한 부분이 완료된 셈이다. 


## Yield from

이제 부터는 본격적으로 python3에서 지원되는 기능이다. 

일반 함수나 task도 1:1 통신을 하는 경우 말고, a <-> b <-> c 와 같이 통신을 하는 경우도 많다(task가 sub task를 재 호출하는 방식). Coroutine도 마찬가지이다. Coroutine이 다시 sub-coroutine을 호출하는 구조가 될 수 있다. 예제로 보면 다음과 같이 될 것이다. 

```python
def subcoroutine():
    yield 1
    yield 2
    
def coroutine():
    for v in subcoroutine():
        yield v
        
x = coroutine()
print(next(x))    # 1 출력
print(next(x))    # 2 출력 
next(x)           # StopIteration 
``` 

Caller가 `next()`를 호출하면 `coroutine()`으로 제어가 넘어가고, 여기서 다시 `subcoroutine()`으로 넘어간다. 다시, 여기에서 yield로 돌려준 값이 caller까지 돌아오게 된다.

하지만 yield base coroutine에서 추가된 `send()`, `throw()`, `close()`를 지원하려면 단순히 이것으로만 되지 않는다. 예를 들어, 간단히 `send()`만 지원하게 하려면 중간의 `coroutine()`은 다음과 같이 수정되어야 한다. 

```python
def subcoroutine():
    print("Subcoroutine")
    x = yield 1
    print("Recv: " + str(x))
    x = yield 2
    print("Recv: " + str(x))

def coroutine():
    _i = subcoroutine()
    _x = next(_i)
    while True:
        _s = yield _x

        if _s is None:
            _x = next(_i)
        else:
            _x = _i.send(_s)
```

이를 실제 테스트 해보면 다음과 같다. 
```python
>>> from test import *
>>> x = coroutine()
>>> next(x)
Subcoroutine
1
>>> x.send(10)
Recv: 10
2
>>> x.send(20)
Recv: 20
StopIteration
```

중간에 _s의 값을 보고 `next()`와 `send()`를 구분하는데, 위 코드에서는 그냥 `_x = _i.send(_s)` 를 호출해도 된다. `send(None)`은 `next()`와 동일하다. 하지만 `_i`가 위처럼 generator가 아니라 iterator라면 반드시 next()를 해주어야 한다(iterator는 `__next__()`만 구현되고, `send()`는 없다). 여기에다 `throw()`, `close()` 등의 예외 사항을 추가한다면 복잡한 일이다.

이를 한줄로 지원하는 것이 python 3.3 ([PEP 380 -- Syntax for Delegating to a Subgenerator](https://www.python.org/dev/peps/pep-0380/))에 추가된 yield from 이다. 

위와 같이 coroutine에서 sub-coroutine을 호출하여 결과적으로 caller <=> sub-coroutine 이 데이타를 주고 받게 하려면 위와 같이 복잡하게 구현을 하지 않고 yield from을 사용하면 된다. 위 PEP 380을 보면 yield from 의 풀어쓴 코드 예가 있는데, 39줄이다. 기본적인 것은 위와 같은 루프에 exception 처리를 추가한 것이다. 

yield from을 이용하면 위 `coroutine()`은 다음과 같이 바뀐다. 
```python
def subcoroutine():
    print("Subcoroutine")
    x = yield 1
    print("Recv: " + str(x))
    x = yield 2
    print("Recv: " + str(x))

def coroutine():
    yield from subcoroutine()
```

yield from 오른쪽에 들어갈 수 있는 것은 iterable과 generator이다. 즉, 기존 for .. in 에 사용가능한 모든 iterable은 사용 가능하다. 

[PEP 380](https://www.python.org/dev/peps/pep-0380/)에 하나 더 추가된 기능이 있는데, generator에서 return 지원이다. Generator의 역사를 보면 iterator에서 부터 확장된 것이라, generator에서는 return을 사용하여 값을 반환 할 수 없고, 종료되면 StopIteration exception이 발생하는 방식이었다. 이를 return으로 값을 반환하도록 기능을 추가하였다. 

```python
def test():
    yield 1
    return 10
``` 

위와 같은 코드는 실제로 다음과 같다고 보면 된다. 
```python
def test():
    yield 1
    e = StopIteration()
    e.value = 10
    raise e
```

위 코드를 시험해 보면 다음과 같이 된다. 
```python
>>> def test():
    yield 1
    return 10
    
>>> x = test()
>>> next(x)
1
>>> next(x)
StopIteration: 10
``` 

Exception으로 10 이 같이 리턴되었다. 

위와 같이 return을 추가로 지원한 것은 yield from의 기능을 확장하기 위한 것이다. 이와 같이 return이 지원되면 다음과 같이 yield from 에서 직접 값을 받을 수 있다. 이런 구조가 되면 lightweight thread 형태가 된다. 즉 sub-corotine은 필요한 만큼 돌다가 결과가 나오면 그때 반환하는 방식이 된다. 

```python
def sum(max):
    tot = 0
    for i in range(max):
       tot += i
       yield tot
    return tot


def coroutine():
    x = yield from sum(10)
    print('Total: {}'.format(x))
```

하지만 여기서 부터 혼동이 될 수 있는데, `sum()` 과 같은 sub-coroutine에서 yield로 주는 값과, return 되는 값은 용도가 다르다는 것이다. `yield tot`로 전달한 것은 iterator 중간 값을 caller에서 받는다. 하지만 `yield from`으로 받는 값은 caller 가 아니라 중간의 parent coroutine에서 받는 것이다. 즉, 두가지는 사용하는 용도가 다르다. 
yield tot 와 같이 coroutine에서 값을 parent로 전달하는 것은 iterator의 확장이라고 볼수 있다. 하지만 보통 coroutine 기반으로 lightweigth thread를 작성하는 경우 이와 같이 중간값 보다는 다른 coroutine과 같이 돌다가 최종 결과를 받기를 원하는 경우이다. 이때 yield from의 return값을 사용하게 된다. 

다음 장에서 나오게 될 asynio의 eventloop로 돌아가는 coroutine에서 yield from의 리턴값을 제대로 사용하게 된다. asyncio는 비동기로 모든 함수들이 실행된다. 특정 이벤트에 따라서 호출되는 것은 일반 callback 함수 일 수 있고, coroutine 일 수 있다. Coroutine은 위와 같이 yield from으로 다른 coroutine을 호출하여 결과를 비동기로 받을 수 있다 (하지만 코드를 보면 yield from 이라는 문장만 있지 생긴것은 sequential하게 되는 셈이고, 예외 처리도 sequential한 코드와 동일한 방식으로 처리가 가능하다).


## 중간 정리 
Python의 coroutine이 복잡한 이유는 꾸준히 기능이 확장되었기 때문이라고 생각된다. 한번에 모든 기능이 정립되었으면 깔끔했을지 모르나, iterator부터 시작해서 하나씩 살이 붙다 보니, 개념을 정확히 이해하지 못하면 다양한 keyword의 활용 예를 보고 혼동에 빠지고 만다. 
가장 이해하기 좋은 방법은 위처럼 간단한 예를 직접 실행해 보고, 필요한 부분은 PEP 문서 등의 개요와 built in package source를 찾아 보는 것이다.  
