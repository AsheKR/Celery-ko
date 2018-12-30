# First Steps with Celery

`Celery`는 배터리가 포함된 작업 대기열이다. 사용하기 쉽기 때문에 복잡한 문서를 배우지 않고 시작할 수 있다. 제품이 다른 언어로 확장 및 통합될 수 있도록 모범 사례를 바탕으로 설계되었으며 프로덕션 환경에서 이러한 시스템을 실행하는 데 필요한 도구와 자원이 제공된다.

이 튜토리얼에서는 `Celery`의 기초 사용법을 배운다.

- 메세지 전송 (브로커) 선택 및 설치
- `Celery` 설치 및 첫 작업 만들기
- `worker` 실행 및 작업 호출
- 작업이 다른 상태로 전환 할 때 작업을 추적하고 반환값을 검사

`Celery`는 힘들어보일지 몰라도 이 튜토리얼을 통해 바로 시작할 수 있다. 의도적으로 단순하게 유지하여 고급 기능으로 혼란하지 않게 한다. 이 튜토리얼을 끝내고 나면 나머지 문서를 찾아보는 것이 좋다. 예를 들면 다음 단계 자습서에서는 `Celery`의 기능을 보여준다.

## 문서의 목차

- [Choosing a Broker](#Choosing a Broker)
	- [RabbitMQ](#RabbitMQ)
	- [Redis]()
	- [Other brokers]()
- [Installing Celery]()
- [Application]()
- [Running the Celery worker server]()
- [Calling the task]()
- [Keeping Results]()
- [Configuration]()
- [Where to go from here]()
- [Troubleshooting]()
	- [Worker doesn't start: Permission Error]()
	- [Result backend doesn't work to tasks are always in **PENDING** state]()

## Choosing a Broker

`Celery`는 메세지를 보내고 받기위한 솔루션이 필요하다. 대개 이것은 메세지 브로커라는 별도의 서비스 형태로 제공된다.

몇가지 선택이 가능하다.

[브로커 비교](https://stackshare.io/stackups/amazon-sqs-vs-rabbitmq-vs-redis)

### RabbitMQ

`RabbitMQ`는 기능이 완벽하고 안정적이며 내구성이 뛰어나고 설치가 쉽다. Production 환경에서 훌륭한 선택이다. 더 자세한 내용은 :[Using RabbitMQ]()

우분투 또는 데비안을 사용하는 경우 이 명령을 실행하여 `RabbitMQ`를 설치한다.

설치 완료 후 브로커가 이미 백그라운드에서 실행중이며 메세지를 이동할 준비가 된다.
`Starting rabbitmq-server: SUCCESS`

우분투 또는 데비안 환경을 사용하지 않는다고 걱정하지 않아도 된다. 이 웹사이트에서 Microsoft Windows를 포함한 다른 플랫폼에 대한 간단한 설명 지침을 찾을 수 있다.
	[http://www.rabbitmq.com/download.html](http://www.rabbitmq.com/download.html)

### Redis

`Redis`는 기능이 완벽하지만 급격한 종류, 전원 장애시 데이터 손실에 보다 취약하다.
더 자세한 내용은: [Using Redis](https://github.com/teachmesomething2580/Celery-ko/blob/master/Getting%20Started/Brokers/Using%20Redis.md)

### Other brokers

위 내용 외에도 `Amazon SQS`를 비록한 선택 목록이 있다.
더 자세한 내용은: [Broker Overview](http://docs.celeryproject.org/en/latest/getting-started/brokers/index.html#broker-overview)

## Installing Celery

`Celery`는 Python Package Index(PyPI)에 있으므로 `pip` 또는 `easy_install`과 같은 표준 Python 도구로 설치할 수 있다.

```bash
$ pip install celery
```

## Application

가장 먼저 필요한 것은 `Celery Instance`이다. 우리는 이를 `Celery Application` 또는 간단히 `app`이라고 부른다.  이 인스턴스는 `Celery`에서 수행하려는 모든 작업(예: `task` 작성 및 `worker` 관리)의 시작점으로 사용되므로 다른 모듈이 가져오는 것이 가능해야한다.

이 튜토리얼에서는 단일 모듈에 모든것을  포함시키고 있지만 큰 프로젝트의 경우 전용 모듈을 만들어 사용할 수 있다.

`tasks.py`를 생성한다.
```python
from celery import Celery

app = Celery('tasks', broker='pyamqp://guest@localhost//')

@app.task
def add(x, y):
	return x + y
```

`Celery`의 첫 인자는 현재 모듈의 이름이다. 이는 `__main__` 모듈에서 작업을 정의할 때 이름을 자동으로 생성할 수 있도록하기위해 필요하다.

두번째 인자는 `broker`의 인자이며 사용하려는 메세지 브로커의 URL을 지정한다. 여기 `RabbitMQ`(기본 옵션)을 사용한다.

자세한 선택은 위의 [Choosing a Broker]()를 참조한다. `RabbitMQ`의 경우 `amqp://localhost`를 사용하고 Redis의 경우 `redis://localhost`를 사용할 수 있다.

`add`라고하는 단일 태스크를 정의하여 두 숫자의 합계를 리턴한다.

## Running the Celery worker server

`worker` 인자를 통해 프로그램을 실행하여 `worker`를 실행할 수 있다.
```bash
$ celery -A tasks worker --loglevel=info
```
> Note:
> `worker`가 실행이되지 않는다면 [Troubleshooting]() 섹션을 참고하라.

프로덕션에서는 백그라운드에서 작업자를 데몬으로 실행하려한다. 이렇게하려면 플랫폼에서 제공하는 도구 또는 `supervisord`와 같은 도구를 사용해야한다. (자세한 내용은 [Daemonization]()을 참고한다.)

사용가능한 명령 옵션의 전체 목록을 보려면 다음을 수행한다.
```bash
$ celery worker --help
```
다른 몇가지 명령을 사용할수도 있다.
```bash
$ celery help
```

## Calling the task

작업을 부르기위해선 `delay()` 메서드를 실행한다.

태스크 실행을 더 잘 제어할 수 있는 `apply_async()` 메소드의 편리한 바로가기이다. ([Calling Tasks]() 참조)
```python
>>> from tasks improt add
>>> add.delay(4, 4)
```
작업은 내가 이전에 시작한 `worker`에 의해 처리된다. `worker`의 콘솔을 보면 이를 확인 할 수 있다.

작업을 호출하면 `AsyncResult` 인스턴스가 리턴된다. 이는 작업의 상태를 확인하거나, 작업이 끝나기를 기다리거나 리턴값을 얻는데 (또는 태스크가 실패한 경우 예외 및 추적을 얻는 데)사용될 수 있다.

Results는 기본적으로 사용하지 않도록 되어있다. 원격 프로시저를 호출을 수행하거나 데이터베이스에서 작업 결과를 추적하려면 Result backend를 사용하도록 `Celery`를 구성해야한다. 이는 다음 섹션에서 설명한다.

## Keeping Results

태스크의 상태를 추적하려면 Celery가 상태를 저장하거나 어딘가로 보낼 필요가 있다. 선택할 수 있는 몇가지 백엔드가 있다. [SQLAlchemy/Django]() ORM, [Memcached](), [Redis](), [RPC]() ([RabbitMQ]()/AMQP), 또는 직접 정의할 수 있다.

이 예는 _rpc_ result backend를 사용하여 상태를 일시적인 메세지로 다시 보낸다. 백엔드는 Celery에 대한 백엔드 인수를 통해 (또는 구성 모듈을 사용하도록 선택한 경우 [result_backend]() 설정을통해) 지정된다.

```python
app = Celery('tasks', backend='rpc://', broker='pyamqp://')
```

또는 Redis를 result backend로 사용하고 RabbitMQ를 메세지 브로커로 사용하는 경우 (자주 사용되는 조합)
```python
app = Celery('tasks', backend='redis://localhost', broker='pyamqp://')
```

더 자세한 result backend에 대한 정보는 [Result Backends]()를 참조한다.

이제 백엔드가 구성된 상태에서 작업을 다시 호출한다. 이번에는 작업을 호출할 때 반환되는 [AsyncResult]()가 유지된다.

```python
>>> result = add.delay(4, 4)
```

[ready()]() 메서드는 task가 처리를 완료했는지 여부를 리턴한다.

결과가 완료될 때까지 기다릴 수는 있지만 비동기 호출을 동기식 호출로 전환하기 때문에 거의 사용되지 않는다.

```python
>>> result.get(timeout=1)
8
```

task가 예외를 발생시키는 경우 `get()`은 예외를 다시 발생시키지만 `propagate`인자를 지정하여 예외를 재정의 할 수 있다.

```python
>>> result.get(propagate=False)
```

task에서 예외가 발생하면 traceback에 액세스 할 수 있다.
```python
>>> result.traceback
```

> Warning:
> 백엔드는 결과를 저장하고 전송하기 위해 리소스를 사용한다. 리소스가 해제되도록 작업 호출 후 반환되는 모든 _AysncResult_ 인스턴스에서 _get()_ 또는 _forget()_을 호출해야한다.

result object는 [celery.result]() 참조한다.

## Configuration

Celery는 가전제품과 마찬가지로 작동하기때문에 많은 구성요소를 필요로 하지 않는다. 이는 입력과 출력을 가지고 있다. 입력은 브로커에 연결되어야하며 출력은 선택적으로 결과 백엔드에 연결될 수 있다. 그러나 뒤를 자세히 보면 많은 수의 슬라이더, 다이얼 및 버튼이 있는 뚜껑이 있다.

기본 구성은 대부분의 유즈 케이스에 대해 충분해야하지만 Celery를 필요에 따라 정확하게 작동하도록 구성할 수 있는 많은 옵션이 있다. 사용 가능한 옵션에 대해 구성 할 수 있는 것을 숙지하는 것이 좋다. [Configuration and defaults]()에서 읽을 수 있다.

설정은 응용 프로그램에서 직접 설정하거나 전용 구성 모듈을 사용하여 설정할 수 있다. 예를 들어 _task_serializer_ 설정을 변경하여 작업 페이로드를 serialize하는데 사용되는 기본 serializer를 구성 할 수 있다.

```python
app.conf.task_serializer = 'json'
```

한번에 여러 설정을 구성하는 경우 `update`:
```python
app.conf.update(
    task_serializer='json',
    accept_content=['json'],  # Ignore other content
    result_serializer='json',
    timezone='Europe/Oslo',
    enable_utc=True,
)
```

대규모 프로젝트에서는 전용 구성 모듈을 사용할것을 추천한다. 정기 작업, 작업 라우팅 옵션을 하드코딩하는 것은 바람직하지 않다. 중앙 집중식으로 보관하는 것이 좋다. 이는 사용자가 작업 방식을 제어할 수 있으므로 라이브러리 경우 특히 그렇다. 중앙 집중식 구성을 사용하면 관리자가 시스템 문제 발생시 간단한 변경 작업을 수행할 수 있다.

Celery 인스턴스가 _app.config_from_object()_ 메소드를 사용하여 구성 모듈을 사용한다고 설정할 수 있다.
```python
app.config_from_object('celeryconfig')
```

이 모듈의 이름은 보통 _"celeryconfig"_라고 불린다. 하지만 이름은 아무렇게 작성해도 좋다.

위의 경우 `celeryconfig.py` 모듈은 현재 디렉토리 또는 Python 경로에서 로드할 수 있어야한다.

_celeryconfig.py:_
```python
broker_url = 'pyamqp://'
result_backend = 'rpc://'

task_serializer = 'json'
result_serializer = 'json'
accept_content = ['json']
timezone = 'Europe/Oslo'
enable_utc = True
```

구성파일이 올바르게 작동하고 구문 오류가 없는지 확인하려면 파일을 가져올 수 있다.

```bash
$ python -m celeryconfig
```

구성 옵션에 대한 자세한 내용은 [Configuration and default]()를 참조한다.

구성 파일의 성능을 보여주기 위해 오작동하는 작업을 전용 대기열로 라우팅하는 방법은 다음과 같다.

_celeryconfig.py:_
```python
task_routes = {
	'tasks.add': 'low-priority',
}
```

또는 라우팅 대신 작업의 속도를 제한할 수 있으므로 이 유형은 오직 10개의 작업만을 1분에 처리하게 한다.

_celeryconfig.py:_
```python
task_routes = {
	'tasks.add': {'rate_limit': '10/m'}
}
```

브로커로 RabbitMQ 또는 Redis를 사용하는 경우 런타임 작업에 대한 새로운 속도 제한을 설정하도록 `worker`에게 지시할수도 있다.

```bash
$ celery -A tasks control rate_limit tasks.add 10/m
worker@example.com: OK
	new rate limit set successfully
```

task routing의 더 자세한 내용은 Routing Tasks]()을 보고 annotations 설정은 [task_annotations](), 원격 제어 명령 및 `worker`의 작업 모니터링 방법에 대한 자세한 내용은 [Monitoring and Management Guide]()를 참조하도록한다.

## Where to go from here

더 많은 것을 배우고 싶다면 [Next Steps] 튜토리얼읽고 그 후 [User Guide]()를 읽을 수 있다.
