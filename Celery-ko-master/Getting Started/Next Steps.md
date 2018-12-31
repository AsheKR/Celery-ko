# Next Steps

[First Steps with Celery]() 가이드는 의도적으로 최소화했다. 이 가이드에서는 응용 프로그램 및 라이브러리에 Celery 지원을 추가하는 방법을 비롯하여 Celery가 제공하는 것을 보다 자세히 설명한다.



이 문서에는 Celery의 모든 기능과 모범 사례가 나와있지 않으므로 [User Guide]()를 읽는 것이 좋다.



- [Using Celery in your Application](#Using-Celery-in-your-Application)
- [Calling Tasks](#Calling_Tasks)
- [Canvas: Designing Work-flows](#Canvas:-Designing-Work-flows)
- [Routing](#Routing)
- [Remote Control](#Remote-Control)
- [Timezone](#Timezone)
- [Optimization](#Optimization)
- [What to do now?](#What-to-do-now?)



## Using Celery in your Application

### Out Project

Project layout:

```bash
proj/__init__.py
	/celery.py
	/tasks.py
```



proj/celery.py

```python
from __future__ import absolute_import, unicode_literals
from celery import Celery

app = Celery('proj',
             broker='amqp://',
             backend='amqp://',
             include=['proj.tasks'])

# Optional configuration, see the application user guide.
app.conf.update(
    result_expires=3600,
)

if __name__ == '__main__':
    app.start()
```

이 모듈에서는 Celery 인스턴스(_app_이라고도 함)를 만들었다. Celery를 프로젝트에서 사용하려면 인스턴스를 가져오기만 하면 된다.

- `broker` 인자는 사용할 broker의 URL을 지정한다.

  더 자세한 정보는 [Choosing a Broker]()을 보자.

- `backend`인자는 result backend로 사용할 것을 지정한다.

  작업 상태와 결과를 추적하는데 사용된다. 결과가 나중에 작동하는 방법을 보여주기 때문에 기본적으로 결과가 사용되지 않지만 RPC result backend를 사용하므로 응용 프로그램에 다른 백엔드를 사용할 수 있다. 모두 다른 강약점을 가지고 있다. result가 필요하지 않은 경우 result backend를 사용하지 않는 것이 좋다. `@task(ignore_result=True)`옵션을 설정하여 개별 작업에 대한 결과를 비활성화 할 수도 있다.

  더 자세한 정보는 [Keeping Results]()를 보자.

- `include` 인자는 worker가 시작할 때 가져올 모듈 목록이다. worker가 작업을 찾을 수 있도록 여기에 작업 모듈을 추가한다.



proj/tasks.py

```python
from __future__ import absolute_import, unicode_literals
from .celery import app


@app.task
def add(x, y):
    return x + y


@app.task
def mul(x, y):
    return x * y


@app.task
def xsum(numbers):
    return sum(numbers)
```



### Starting the worker

celery 프로그램을 사용하여 worker를 시작할 수 있다. (worker는 proj 위의 디렉터리에서 실행해야한다.)

```bash
$ celery -A proj worker -l info
```

worker가 시작하면 배너와 메세지가 나타난다.

```bash
 -------------- celery@ubuntu v4.2.1 (windowlicker)
---- **** ----- 
--- * ***  * -- Linux-4.15.0-43-generic-x86_64-with-debian-buster-sid 2018-12-30 02:51:04
-- * - **** --- 
- ** ---------- [config]
- ** ---------- .> app:         proj:0x7f42fab831d0
- ** ---------- .> transport:   amqp://guest:**@localhost:5672//
- ** ---------- .> results:     amqp://
- *** --- * --- .> concurrency: 4 (prefork)
-- ******* ---- .> task events: OFF (enable -E to monitor tasks in this worker)
--- ***** ----- 
 -------------- [queues]
                .> celery           exchange=celery(direct) key=celery
                

[tasks]
  . proj.tasks.add
  . proj.tasks.mul
  . proj.tasks.xsum

[2018-12-30 02:51:04,187: INFO/MainProcess] Connected to amqp://guest:**@127.0.0.1:5672//
[2018-12-30 02:51:04,212: INFO/MainProcess] mingle: searching for neighbors
[2018-12-30 02:51:05,235: INFO/MainProcess] mingle: all alone
[2018-12-30 02:51:05,251: INFO/MainProcess] celery@ubuntu ready.

```

- _broker_는 Celery 모듈의 브로커 인수에 지정한 URL이며 **-b** 옵션을통해 명령행에 다른 브로커를 지정할 수 있다.
- _Concurrency_는 작업을 동시에 처리하는데 사용되는 worker 프로세스 수이다. 작업이 모두 수행중인경우 새 작업은 작업 중 하나가 완료될 때까지 기다려야 처리된다.

기본 concurrency 수는 해당 컴퓨터의 CPU 수이며 `celery worker -c`를 사용하여 커스텀 할 수 있다.  최적의 수는 여러 요소에 따라 달라지므로 권장값이 없다. 그러나 작업이 대부분 I/O를 사용한다면 수를 증가시키려고 할 수 있는데 이는 효과적이지 않으며 성능저하를 일으킬수도 있다.

기본 prefork 풀을 포함하여 Celery는 Eventlet, Gevent 사용 및 단일 스레드에서의 실행도 지원한다.

- _Events_가 활성화된 경우 Celery가 worker에서 발생하는 작업에 대한 모니터링 메세지를 보내도록 하는 옵션이다. `celery Event`` Flower와 같은 실시간 모니터 프로그램에서 사용할 수 있으며  [Monitoring and Management guide]()에서 참고할 수 있다.
- _Queue_는 worker가 작업을 사용하는 대기열 목록이다. worker는 여러 Queue에서 한번에 소비하라는 메세지를 받을 수 있으며 이는 [Routing Guide]()에 설명된 Quality of Service, 우선 순위 분리 및 우선 순위 지정을 위한 수단으로 특정 작업자에게 메세지를 라우팅하는데 사용된다.



[--help]() 플래그를 사용하여 전체 명령 리스트를 얻을 수 있다.

```bash
$ celery worker --help
```

이러한 옵션에 대해서는 [Workers Guide]()에서 자세히 설명한다.



### Stopping the worker

worker는 `Control-c`로 멈출 수 있다. worker가 지원하는 signal들은 [Workers Guide]()에서 참조할 수 있다.



### In the background

프로덕션에서는 백그라운드에서 worker를 실행하려고한다. 자세한내용은 [daemonization tutorial]()을 참고한다.

데몬 스크립트는 `celery multi` 명령을 사용하여 백그라운드에서 하나 이상의 worker를 실행한다.

```bash
$ celery multi start w1 -A proj -l info
celery multi v4.2.1 (windowlicker)
> Starting nodes...
	> w1@ubuntu: OK

```

다시시작할수도 있다.

```bash
$ celery multi restart w1 -A proj -l info
celery multi v4.2.1 (windowlicker)
> Stopping nodes...
	> w1@ubuntu: TERM -> 23219
> Waiting for 1 node -> 23219.....
	> w1@ubuntu: OK
> Restarting node w1@ubuntu: OK
> Waiting for 1 node -> None...

```

또는 멈추기

```bash
celery multi stop -A proj -l info
```

`stop` 명령은 비동기적이므로 worker가종료될때까지 기다리지 않는다. `stopwait` 명령을 사용하여 현재 실행중인 모든 작업을 종료되기 전에 완료한다.

```bash
celery multi stopwait w1 -A proj -l info
```

> ### Note:
>
> **celery multi**는 worker에대한 정보를 저장하지 않으므로 다시 시작할 때 동일한 명령줄 인수를 사용해야한다. 중지할 때 동일한 pidfile 및 logfile인수를 사용해야한다.

기본적으로 현재 디렉터리에 pid 및 로그 파일을 생성한다. 전용 디렉터리를 사용하길 권장한다.

```bash
$ mkdir -p /var/run/celery
$ mkdir -p /var/log/celery
$ celery multi start w1 -A proj -l info --pidfile=/var/run/celery/%n.pid \
--logfile=/var/log/celery/%n%I.log
```

`multi` 명령을 사용하면 여러 작업자를 실행할 수 있으며 강력한 command-line 문법을 사용하여 다른 worker에 대한 인자도 지정할 수 있다.

```bash
$ celery multi start 10 -A proj -l info -Q:1-3 images,video -Q:4,5 data \
-Q default -L:4,5 debug
```

`multi` 모듈에대한 더 많은 예제는 [multi]()문서를 참조한다.



### About the --app argument

__--app__ 인자는 사용할 Celery 응용 프로그램 인스턴스를 지정하며 `module.path:attribute` 형식이여야한다.

그러나 패키지 이름만 지정되어있는 경우 앱 인스턴스를 검색할 위치는 다음과 같은 순서로 지원한다.

[--app=proj]():

 	1. `proj.app` 또는
 	2. `proj.celery` 또는
 	3. 값이 Celery 응용 프로그램인 모듈 `proj` 의 모든 속성 또는

이중 어느것도 발견되지 않으면 `proj.celery`라는 서브모듈에 시도한다.

1. `proj.celery.app`의 이름을 가진 속성 또는
2. `proj.celery.celery`의 이름을 가진 속성 또는
3. 값이 Celery 응용 프로그램인 `proj.celery`의 모든 속성



## Calling Tasks

task를 `delay()`메서드를 사용해 호출할 수 있다.

```python
>>> add.delay(2, 2)
```

이 메서드는 실제로 다른 메서드인 `apply_async()`에 대한 star-argument 단축 메서드이다.

```python
>>> add.apply_async((2, 2))
```

후자를 사용하면 실행시간(카운트 다운), 보내야할 큐 등과 같은 실행 옵션을 지정할 수 있다.

```python
>>> add.apply_async((2, 2), queue='lopri', countdown=10)
```

위 예에서 작업은 `lopri`라는 Queue로 보내지고 작업은 메세지가 전송된 후 빠른시간 10초 이내에 실행된다.

작업을 직접 실행하면 메세지가 전송되지 않도록 현재 프로세스에서 작업이 실행된다.

```python
>>> add(2, 2)
4
```

`delay()`, `apply_async()`, 그리고 apply(\_\_call\_\_)의 세가지 메서드는 Celery API를 나타내며 서명에서도 사용된다.

더 자세한내용은 [Calling User Guide]()에서 찾을 수 있다.

모든 작업 호출에는 고유 식별자(UUID)가 주어지며, 이것은 작업 ID이다.

`delay`와 `apply_async` 메서드는 작업 실행 상태를 추적하는 데 사용할 수 있는 `AsyncResult` 인스턴스를 반환한다. 그러나 이것을 위해서는 상태가 어딘가 저장될 수 있도록 Result backend를 활성화해야한다.

Results는 기본적으로 비활성화되어있다. 모든 응용 프로그램에 적합한 Result backend가 없으므로 각 백엔드의 단점을 고려해 하나를 선택해야한다. 많은 작업의 경우 반환 값을 유지하는 것이 매우 유용하지 않으므로 합리적인 기본값이다. 또한 Celery가 전용 이벤트 메세지를 사용하기 때문에 Result backend는 작업 및 worker를 모니터링하는데 사용되지 않는다.

Result backend가 구성된 작업의 경우 작업의 반환 값을 검색할 수 있다.

```python
>>> res = add.delay(2, 2)
>>> res.get(timeout=1)
4
```

또한 id 속성을 보고 작업의 ID를 찾을 수 있다.

```python
>>> res.id
d6b3aea2-fb9b-4ebc-8da4-848818db9114
```

또한 작업이 예외를 발생시킨 경우 예외 및 traceback이 가능하다. 사실 result.get()은 기본적으로 모든 오류를 전달한다.

```python
>>> res = add.delay(2)
>>> res.get(timeout=1)
```

```python
Traceback (most recent call last):
File "<stdin>", line 1, in <module>
File "/opt/devel/celery/celery/result.py", line 113, in get
    interval=interval)
File "/opt/devel/celery/celery/backends/rpc.py", line 138, in wait_for
    raise meta['result']
TypeError: add() takes exactly 2 arguments (1 given)
```

에러가 전파되는 것을 원치않으면 `propagate`인수를 전달하려 오류를 비활성화 시킬 수 있다.

```python
>>> res.get(propagate=False)
TypeError('add() takes exactly 2 arguments (1 given)',)
```

이 경우 대신 발생한 예외 인스턴스가 반환되므로 작업의 성공실패 여부를 확인하려면 결과 인스턴스에서 메서드를 사용해야한다.

```python
>>> res.failed()
True

>>> res.successful()
False
```

작업이 실패했는지 여부는 작업 상태를 보면알 수 있다.

```python
>>> res.state
'FAILURE'
```

작업은 단일 상태일 수 있지만 여러 상태를 통해 진행될 수 있다. 일반적 작업 단계는 다음과 같다.

```wiki
PENDING -> STARTED -> SUCCESS
```

시작 상태는 [task_track_started]()설정이 사용가능하거나 작업에 `@task(track_started=True)` 옵션이 설정된 경우에만 기록되는 특별한 상태이다.

보류 상태는 실제로 기록된 상태가 아닌 알수 없는 작업 ID의 기본 상태이다.

```python
>>>from proj.celery import app

>>> res = app.AsyncResult('this-id-does-not-exist')
>>> res.state
'PENDING'
```

작업이 재시도되면 단계가 더욱 복잡해질 수 있다. 두번째 시도된 작업의 경우 단계는 다음과 같다.

```wiki
PENDING -> STARTED -> RETRY -> STARTED -> RETRY -> STARTED -> SUCCESS
```

작업 상태에 대한 자세한 내용은 [States]()섹션 간이드에 자세히 설명되어 있다.

작업 호출은 [Calling Guide]()에 자세히 설명되어있다.



## Canvas: Designing Work-flows

방금 `delay()` 메서드를 사용하여 task를 호출하는 방법을 배웠고,  때로는 작업 호출의 signature를 다른 프로세스에 전달하거나 다른 함수의 인수로 전달할 수 있다.

서명은 단일 태스트 호출의 인수 및 실행 옵션을 함수에 전달하거나 직렬화하여 전선을 통해 전송할 수 있는 방식으로 래핑한다.

인수 (2, 2)와 10초 countdown을 사용하여 `add` task의 서명을 생성할 수 있다.

```python
>>> add.signature((2, 2), countdown=10)
task.add(2, 2)
```

단축 명령도 사용할 수 있다.

```python
>>> add.s(2, 2)
tasks.add(2, 2)
```



### And there's that calling API again...

서명 인스턴스는 API 호출도 지원한다. 즉, `delay` 및 `apply_async` 메서드를 가지고 있음을 의미한다.

그러나 서명에는 이미 인수가 지정되어있다는점에서 차이가 있다. `add`task는 두개의 인수를 취하므로 두 개의 인수를 지정하여 완전한 서명을 작성할 수 있다.

```python
>>> s1 = add.s(2, 2)
>>> res = s1.delay()
>>> res.get()
4
```

부분적으로 불완전한 서명을 생성할 수 있다.

```python
# incomplete partial: add(?, 2)
>>> s2 = add.s(2)
```

`s2`를 완료된 서명으로 만들기 위해 또 다른 인수를 필요로하는 부분 서명이며 서명을 호출할 때 해결할 수 있다.

```python
# resolves the partial: add(8, 2)
>>> res = s2.delay(8)
>>> res.get()
10
```

여기서 기존 인수 2앞에 추가된 인수 8을 추가하여 `add(8, 2)`의 완전한 서명을 작성한다.

키워드 인수는 나중에 추가 할 수도 있다. 기존 키워드 인수와 병합되지만 새로운 인수가 우선 적용된다.

```python
>>> s3 = add.s(2, 2, debug=True)
>>> s3.delay(debug=False)  # debug is now False.
```

값이 채워진 서명은 다음의 API 호출을 지원한다.

- `sig.apply_async(args=(), kwargs={}, **options)`

  선택전 부분 인수 및 부분 키워드 인수를 사용하여 호출한다. 부분 실행 옵션도 지원한다.

- `sig.delay(*args, **kwargs)`

  `apply_async`의 star argument 버전. 

이 모든게 유용하게 보이지만 실제로 이것들로 무엇을 할 수 있는가?  이를 알려면 `canvas primitives`를 소개해야한다...



### The Primitives

- [group]()
- [chain]()
- [chord]()
- [map]()
- [starmap]()
- [chunks]()

이러한 Primitives는 서명 객체 자체이며 여러가지 방법을 결합하여 복잡한 작업 흐름을 구성할 수 있다.

> ### Note:
>
> 이 예제는 결과를 검색하므로 result backend를 구성해야한다.

몇가지 예제를 살펴본다.



#### Groups

그룹은 병렬 작업의 목록을 호출하고 그룹으로 결과를 검사하고 반환 값을 검색할 수 있다. 이는 특별한 결과 인스턴스를 반환한다.

```python
>>> from celery import group
>>> from proj.tasks import add

>>> group(add.s(i, i) for i in xrange(10))().get()
[0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
```

- 부분 그룹

```python
>>> g = group(add.s(i) for i in xrange(10))
# 부분 그룹에 빈 곳에 10 채운거!
>>> g(10).get()
[10, 11, 12, 13, 14, 15, 16, 17, 18, 19]
```



#### Chains

하나의 작업이 돌아오면 다른 작업이 호출되도록 작업을 함께 연결할 수 있다.

```python
>>> from celery import chain
>>> from proj.tasks import add, mul

# (4 + 4) * 8
>>> chain(add.s(4, 4) | mul.s(8))().get()
64
```

부분 체인

```python
>>> # (? + 4) * 8
>>> g = chain(add.s(4) | mul.s(8))
>>> g(4).get()
64
```

체인은 다음과 같이 작성 할 수도 있다.

```python
>>> (add.s(4, 4) | mul.s(8))().get()
64
```

#### Chords

chord는 콜백이 있는 그룹이다.

```python
>>> from celery import chord
>>> from proj.tasks import add, xsum

>>> chord((add.s(i, i) for i in xrange(10)), xsum.s())().get()
90
```

다른 작업에 연결된 그룹은 자동으로 코드를 변환한다.

```python
>>> (group(add.s(i, i) for i in xrange(10)) | xsum.s())().get()
90
```

이러한 primitives는 모두 서명 유형이므로 원하는 경우 모두 결합 가능하다.

```python
>>> upload_document.s(file) | group(apply_filter.s() for filter in filters)
```

[Canvas]() 사용자 가이드에서 작업 흐름에 대해 자세히 읽어보자.



## Routing

Celery는 AMQP가 제공하는 모든 라우팅 기능을 지원하지만 메세지가 명명된 대기열로 보내지는 간단한 라우팅도 지원한다.

[tasks_routes]()설정을 사용하면 작업을 이름순으로 라우팅하고 모든 것을 한 곳에 중앙 집중식으로 유지할 수 있다.

```python
app.conf.update(
    task_routes = {
    	'proj.tasks.add': {'queue': 'hipri'},  
    },
)
```

또는 `apply_async`에 `queue`인수를 사용하여 런타임에 대기열을 지정할 수 있다.

```python
>>> from proj.tasks import add
>>> add.apply_async((2, 2), queue='hipri')
```

그런 다음 celery worker -Q 옵션을 지정하여 큐를 사용할 수 있게 한다.

```python
$ celery -A proj worker -Q hipri
```

쉼표로 구분된 목록을 사용하여 여러대기열을 지정할 수 있다. 예로 `hipri`와 기본 대기열을 사용하는 큐를 사용한다. 기본 대기열의 이름은 `celery`를 사용한다.

```python
$ celery -A proj worker -Q hipri,celery
```

worker가 대기열에 동일한 가중치를 주므로 대기열을 순서는 중요하지 않다.

AMQP라우팅의 모든 기능을 포함하여 라우팅에 대한 자세한 내용은 [Routing Guide]()를 참조하라.



## Remote Control

RabbitMQ (AMQP), Redis 또는 Qpid를 브로커로 사용하는 경우 런타임에 worker를 제어하고 검사할 수 있다.

예를 들어  worker가 현재 작업하고 있는 작업을 볼 수 있다.

```bash
$ celery -A proj inspect active
```

이는 브로드캐스트 메세징을 사용하여 구현되므로 원격 제어 명령은 모든 클러스터의 worker가 수신한다.

[--destination]()옵션을 사용하여 요청에 대해 작업할 하나 이상의 작업자를 설정할 수 있다. 쉼표로 구분된 작업자 호스트의 이름 목록을 사용한다.

```bash
$ celery -A proj inspect active --destination=celery@example.com
```

목적지가 제공되지 않으면 모든 작업자가 행동하고 요청에 응답한다.

__celery instpect__ 명령에는 worker를 변경하지 않는 명령이 포함되어 있으며 worker 내부에서 일어나는 일에 대한 정보와 통계만 응답한다. 검사 명령 목록을 보려면 다음을 수행한다.

```bash
$ celery -A proj inspect --help
```

__celery control__ 명령에는 실제로 worker의 Runtime 정보를 변경하는 명령을 포함하는 명령을 포함한다.

```bash
$ celery -A proj control --help
```

예를 들어 worker가 (모니터링과 작업과 작업자를 사용하면) 이벤트 메세지를 사용하도록 설정할 수 있다.

```bash
$ celery -A proj control enable_events
```

이벤트가 활성화되면 이벤트 dumper를 사용하여 worker가 수행중인 작업을 볼 수 있다.

```bash
$ celery -A proj events --dump
```

또는 _curses_ 인터페이스를 시작할 수 있다.

```bash
$ celery -A proj events
```

모니터링이 끝나면 이벤트를 다시 비활성화 할 수 있다.

```bash
$ celery -A proj control disable_events
```

__celery status__ 명령은 원격 제어 명령을 사용하고 클러스터의 온라인 작업자 목록을 표시한다.

```bash
$ celery -A proj status
```

__celery__ 명령 및 모니터링에 대한 자세한 내용은 [Monitoring Guide]()를 참조하자.



## Timezone

모든 시간과 날짜, 내부 및 메세지는 UTC 시간대를 사용한다.

worker가 카운트다운 설정과 같은 메세지를 받으면 UTC 시간을 현지 시간으로 변환한다. 시스템 시간대와 다른 시간대를 사용하려면 timezone 설정을 사용하여 시간대를 구성해야한다.

```python
app.conf.timezone = 'Europe/London'
```

## Optimization

기본 구성은 기본적으로 처리량에 최적화되어있지 않으므로 짧은 작업과 긴 작업 사이, 공정 스케쥴링과 처리량 사이를 타협하여 설정해야한다.

엄격한 공정 스케쥴링이나 처리량을 최적화하고 싶으면 [Optimizing Guide]()를 읽는다.

RabbitMQ를 사용하고 있다면 librabbitmq 모듈을 설치할 수 있다: 이는 AMQP 클라이언트를 C로 구현한것이다.

```bash
$ pip install librabbitmq
```

