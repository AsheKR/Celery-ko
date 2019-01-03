# Tasks

작업은 Celery 응용 프로그램의 기본 요소이다.

A task is a class that can be created out of any callable. 작업이 호출될 때 (메세지를 전송할 때), 작업자가 그 메세지를 받았을 때 일어나는 일 모두를 정의한다는 점에서 이중 역할을 수행한다.

모든 task class는 고유한 이름을 가지며 작업자가 올바른 기능을 찾을 수 있도록 이름이 메세지에서 참조된다.
작업 메세지는 작업자가 해당 메세지를 확인한 후 대기열에서 제거되지 않는다. 작업자는 미리 많은 메세지를 예약할 수 있으며 정전이나 다른 이유로 작업자가 죽더라도 메세지가 다른 작업자에게 전송된다.

이상적인 작업 기능은 같은 인수를 여러번 반복해도 기능이 의도하지 않은 결과를 야기하지 않는 것을 의미한다. 작업자가 작업이 얼마나 멱등한지 알 수 없기 때문에, 기본 행동은 메세지가 실행되기 직전 미리 메세지를 확인하는 것이고, 이미 시작된 작업 호출이 다시 실행되지 않도록 하는 것이다.

태스크가 멱등할경우 [acks_late]() 옵션을 설정하여 작업이 대신 반환된 후 작업자가 메세지를 확인하도록 할 수 있다. FAQ 항목의 [Should I use retry or acks_late?]()를 참고하자.

[acks_late]()가 활성화 된 경우에도 작업을 실행하는 하위 프로세스가 [sys.exit()]() 호출 작업 또는 신호에 의해 종료되면 작업자가 메세지를 확인한다. 이 행동은 다음의 목적이 있다.

1. 커널이 `SIGSEGV(segmentation fault)`나 유사한 시그널 프로세스에 보내도록 하는 작업을 다시하고싶지 않다.
2. 시스템 관리자가 의도적으로 작업을 종료한다고해서 자동으로 다시시작되지 않기를 원하지 않는다고 가정한다.
3. 너무 많은 메모리를 할당하는 작업은 커널 OOM 킬러를 트리거 할 위험이 있다. 같은 일이 다시 발생할 수 있다.
4. 재 전달할 때 항상 실패하는 작업으로 인해 시스템을 중단시키는 고주파수 메세지 루프가 발생할 수 있다.

이러한 시나리오에서 작업을 다시 전달하려는 경우 [task_reject_on_worker_lost]() 설정을 사용하도록 고려해야한다.

> ### 경고
>
> 무기한 차단되는 작업으로 인해 결국 다른 작업을 수행하지 못하게 될 수 있다.
>
> 작업에서 I / O를 수행하는 경우 [requests](https://pypi.python.org/pypi/requests/) 라이브러리를 사용하여 웹 요청에 시간 초과를 추가하는 것과 같이 이러한 작업에 시간 초과를 추가해야한다.
>
> ```
> connect_timeout, read_timeout = 5.0, 30.0
> response = requests.get(URL, timeout=(connect_timeout, read_timeout))
> ```
>
> [Time limits]() 은 모든 작업을시기 적절하게 반환하는 데 편리하지만 시간 제한 이벤트는 실제로 강제로 프로세스를 종료하므로 수동 시간 제한을 아직 사용하지 않은 경우를 감지하는 데만 사용할 수 있다.
>
> 기본 prefork 풀 스케줄러는 장시간 실행 작업에는 적합하지 않으므로 분당 / 시간으로 실행되는 작업을 수행하는 경우 **셀러리 작업자** 에게 [`-Ofair`]() 명령 줄 인수를 사용하도록 설정한다. 자세한 내용은 [Prefork pool prefetch settings]() 을 참조하고 장기간 및 단기 실행 작업을 전용 작업자에게 최적의 성능으로 라우팅하려면  [Automatic routing]()을 참조하자.
>
> 작업자가 멈추면 문제를 제출하기 전 실행중인 작업을 조사하라. 네트워크 작업에 매달린 하나 이상의 작업으로 인해 일시 중단될 가능성이 높기 때문이다.



현재 챕터에서 작업 정의에 대한 모든 내용을 배우며 목차는 다음과 같다.

- [Basics](#Basics)
- [Names](#Names)
- [Task Request](#Task-Request)
- [Logging](#Logging)
- [Retrying](#Retrying)
- [List of Options](#List-Of-Options)
- [States](#States)
- [Semipredicates](#Semipredicates)
- [Custom task classes](#Custom-Task-Classes)
- [How it works](#how-it-works)
- [Tips and Best Practices](#tips-and-best-practices)
- [Performance and Strategies](#performance-and-strategies)
- [Example](#task-example)



## Basics

[task()]() 데코레이터를 사용하여 모든 호출 가능 태스크를 쉽게 작성할 수 있다.

```python
from .models import User

@app.task
def create_user(username, password):
    User.objects.create(username=username, password=password)
```

또한 태스크에 사용할 수 있는 많은 옵션이 있으며 이들은 데코레이터 인수로 지정할 수 있다.

> ### Multiple decorators
>
> 작업 데코레이터와 함께 여러 데코레이터를 사용하는 경우 작업 데코레이터가 마지막으로 적용되었는지 확인해야한다. (이상하게도 Python에서는 목록의 첫번째여야함)
>
> ```python
> @app.task
> @decorator1
> @decorator2
> def add(x, y):
>     return x + y
> ```



### Bound tasks

바인드 된 작업은 Python 바운드 메서드와 마찬가지로 작업의 첫 인수가 항상 작업 인스턴스(자체)임을 의미한다.

```python
logger = get_task_logger(__name__)

@task(bind=True)
def add(self, x, y):
    logger.info(self.request.id)
```

바운드된 작업은 재시도 ([app.Task.retry()]()), 현재 작업 요청에 대한 정보 액세스 및 사용자 정의 작업 기본 클래스에 추가하는 모든 기능이 필요하다.



### Task inheritance

`base`인수는 태스크의 기본 클래스를 지정한다.

```python
import celery

class MyTask(celery.Task):
    
    def on_failure(self, exc, task_id, args, kwargs, einfo):
        print('{0!r} failed: {1!r}'.format(task_id, exc))
        
@task(base=MyTask)
def add(x, y):
    raise KeyError()
```

