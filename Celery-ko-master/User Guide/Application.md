# Application

- [Main Name](#Main-Name)
- [Configuration](#Configuration)
- [Laziness](#Laziness)
- [Breaking the chain](#Breaking-the-chain)
- [Abstract Tasks](#Abstract-Tasks)

Celery 라이브러리는 사용 전 인스턴스화되어야하며, 이 인스턴스를 application (간단히 `app`)이라고 한다.

application은 thread-safe이므로 다른 구성, 구성 요소 및 작업을 가진 여러 Celery 응용프로그램이 동일한 프로세스 공간에 공존할 수 있다.

하나 생성해보자.

```python
>>> from celery import Celery
>>> app = Celery()
>>> app
<Celery __main__:0x100469fd0>
```

마지막 줄에는 응용 프로그램 클래스(`Celery`)의 이름, 현재 주 모듈의 이름(`__main__`)및 객체의 메모리 주소 (`0x100469fd0`)을 비롯한 응용프로그램의 텍스트 표현이 표시된다.



## Main Name

이중 하나만 중요하며 그것은 주 모듈의 이름이다. 왜인지 살펴보자.

Celery에서 작업 메세지를 보내면 해당 메세지에는 소스코드가 포함되지 않고 실행하려는 작업의 이름만 포함된다. 이는 인터넷에서 호스트 이름이 작동하는 것과 유사하게 작동한다. 모든 작업자는 작업 레지스트리의 실제 기능에 대한 작업 이름 매핑을 유지 관리한다.

작업을 정의할때마다 해당 작업이 로컬 레지스트리에 추가된다.

```python
>>> @app.task
... def add(x, y):
... 	return x + y
>>> add
<@task: __main__.add>
>>> add.name
__main__.add
>>> app.tasks['__main__.add']
<@task: __main__.add>
```

그리고 `__main__`을 다시보자. Celery는 함수가 소속된 모듈을 감지할 수 없으며 작업 이름의 시작 부분을 생성한다.

이는 제한된 사용 사례에서만 문제가된다.

 	1. 작업 정의된 모듈이 프로그램으로 실행되는 경우
 	2. 응용 프로그램이 Python 쉘(REPL)에서 작성된 경우

예로 작업모듈을 사용하여 [app.worker_main()]()을 시작하는 경우

`tasks.py`

```python
from celery import Celery
app = Celery()

@app.task
def add(x, y): return x + y

if __name__ == '__main__':
    app.worker_main()
```

이 모듈이 실행될 때 `__main__`으로 시작하는 작업의 이름이 지정되지만 다른 프로세스에서 모듈을 가져오면 작업을 호출할 때 `tasks`(모듈의 실제 이름)로 시작하는 작업의 이름이 지정된다.

```python
>>> from tasks import add
>>> add.name
tasks.add
```

주 모듈에 다른 이름을 지정할 수 있다.

```python
>>> app = Celery('tasks')
>>> app.main
tasks

>>> @app.task
... def.add(x, y):
... 	return x + y

>>> add.name
tasks.add
```

>### See also:
>
>[Names]()

## Configuration

Celery의 작동하는 방식을 바꿀 수 있는 몇 옵션을 구성할 수 있다. 이러한 옵션은 app 인스턴스에서 직접 구성하거나 전용 구성 모듈을 사용할 수 있다.

구성은 [app.conf]()에서 사용할 수 있다.

```python
>>> app.conf.timezone
'Europe/London'
```

구성값을 직접 설정할 수 있다.

```python
app.conf.enable_utc = True
```

또는 `update` 메서드를 사용하여 여러 키를 한번에 업데이트할 수 있다.

```python
>>> app.conf.update(
...     enable_utc=True,
...     timezone='Europe/London',
...)
```

configuration 은 순서대로 참조되는 여러 dictionary로 되어있다.

1. run-time에 변경된 사항
2. 모듈 설정
3. 기본 값 참조 ([celery.app.defaults]())

[app.add_defaults()]()메서드를 사용하여 새 기본 소스를 추가할수도 있다.

>### See also:
>
>사용가능한 모든 설정의 전체와 기본값을 보려면 [Configuration reference]()를 참고하라.

#### config_from_object

[app.config_from_object()]() 메서드는 configuration을 로드한다.

구성 모듈이거나 구성 속성이 있는 모든 개체일 수 있다.

이전에 설정된 모든 구성은 `config_from_object()`가 호출될 때 재설정된다. 추가 구성을 설정하려면 나중에 해야한다.

### Example1: 모듈의 이름 사용하기

[app.config_from_object()]()메서드는 `celeryconfig`, `myproj.config.celery` 또는 `myproj.config: CeleryConfig`와 같이 Python 모듈의 정규화 된 이름 또는 Python 특성의 이름을 사용할 수 있다.

```python
from celery import Celery

app = Celery
app.config_from_object('celeryconfig')
```

`celeryconfig`는 다음과 같이 보일것이다.

`celeryconfig.py`:

```python
enable_utc = True
timezone = 'Europe/London'
```

`import celeryconfig`가 가능하다면 앱에서 사용할 수 있다.

### Example2: 실제 모듈 객체 전달하기

이미 가져온 모듈 객체를 전달할 수도 있지만 항상 권장되지는 않는다.

> ### Tip:
>
> `prefork pool`이 사용될 때 모듈을 serialize 할 필요가 없음을 의미하므로 모듈 이름을 사용하는 것이 좋다. configuration에 문제가 있거나 pickle 오류가 발생한다면 대신 모듈 이름을 사용해라.

```python
import celery

from celery import Celery

app = Celery()
app.config_from_object(celeryconfig)
```



### Example 3: 구성을 class/object로 사용해라.

```python
from celery import Celery

app = Celery()

class Config:
    enable_utc = True
    timezone = 'Europe/London'
    
app.config_from_object(Config)
# 또는 객체의 정규화된 이름을 사용한다.
# 	app.config_from_object('module:Config')
```



#### Config_from_envvar

[app.config_from_envvar()]()는 환경 변수에서 구성 모듈 이름을 가져온다.

예로 `CELERY_CONFIG_MODUEL` 환경 변수에 지정된 모듈에서 구성을 로드하려면 다음과 같이 작성한다.

```python
import os
from celery import Celery

#: Set default configuration module name
os.environ.setdefault('CELERY_CONFIG_MODULE', 'celeryconfig')

app = Celery()
app.config_from_envvar('CELERY_CONFIG_MODULE')
```

그런 다음 환경을 통해 사용할 구성 모듈 이름을 지정할 수 있다.

```bash
$ CELERY_CONFIG_MODULE="celeryconfig.prod" celery worker -l info
```

### 검열된 구성

디버깅 정보 또는 그와 비슷한 구성을 출력하려는 경우 암호 및 API 키와같은 중요 정보를 필터링 할 수 있다.

Celery는 구성을 표시하는데 유용한 여러 유틸리티를 제공한다. 하나는 [humanize()]()이다.

```python
>>> app.conf.humanize(with_defaults=False, censored=True)
```

이 메서드는 구성을 테이블 형식의 문자열로 반환한다. 여기에는 기본적으로 구성에 대한 변경사항만 포함되지만 `with_defaults` 인수를 사용하여 기본제공되는 기본 키와 값을 포함할 수 있다.

구성을 dictonary로 작업하려는 경우 [table()]()메서드를 사용할 수 있다.

```python
app.conf.table(with_defaults=False, censored=True)
```

Celery는 일반적으로 이름이 지정된 키를 검색하기위해 정규표현식을 사용하기 때문에 모든 민감한 정보를 제거할 수는 없다. 중요 정보가 포함된 사용자 지정 설정을 추가하는 경우 Celery가 비밀로 식별하는 이름을 사용하여 키 이름을 지정한다.

이름에 다음 하위 문자열이 포함되어있으면 구성 설정이 검열된다.

`API, TOKEN, KEY, SECRET, PASS, SIGNATURE, DATABASE`



## Laziness

application 인스턴스는 게으르다. 즉, 실제로 필요로하기전까지는 평가되지 않는다.

[Celery]()인스턴스를 생성하면 다음의 작업만 수행한다.

1. 이벤트에 사용될 논리적 클럭 인스턴스만 생성한다.
2. task registry를 생성한다.
3. 현재 응용 프로그램으로 설정한다. (`set_as_current` 인수가 비활성화된 경우 그렇지 않다.)
4. [app.on_init()]() 콜백을 호출한다. (기본적으로 아무것도 수행하지 않음).

[app.task()]() 데코레이터는 작업이 정의된 시점에 작업을 생성하지 않고 작업을 사용할 때 또는 응용 프로그램이 완료된 후 발생하도록 작업 생성을 지연한다.

이 예는 태스크를 사용하거나 속성(이 경우 repr())에 액세스 할 때 까지 태스크가 작성되지 않는 방법을 보여준다.

```python
>>> @app.task
>>> def add(x, y):
... 	return x + y

>>> type(add)
<class `celery.local.PromiseProxy`>

>>> add.__evaluated__()
False

>>> add  # <-- repr을 실행
<@task: __main__.add>
    
>>> add.__evaluated__()
True
```

앱의 종료는 [app.finalize()]()를 호출하여 명시적으로 발생시키거나 [app.tasks]()속성에 액세스하여 암시적으로 발생시킨다.

Finalizing 객체는 다음과 같은 역할을 한다.

1. 앱간 공유해야하는 작업 복사

   작업은 기본적으로 공유되지만 decorator의 `shared` 인수가 비활성화된 경우 작업은 바인딩된 앱에 비공개가 된다.

2. 대기중 작업을 모두 평가한다.

3. 모든 작업이 현재 앱에 바인딩되어있는지 확인한다.

   작업은 구성에서 기본 값을 읽을 수 있도록 바인딩된다.

> ### The "default app"
>
> Celery는 항상 application을 가지고 있지 않았고 모듈 기반 API만 있었고 이전 버전 호환성을 위해 Celery 5.0이 출시될 때 까지 이전 API가 그대로 남아있었다.
>
> Celery는 항상 기본 application인 "default app"을 만든다. 이 application은 사용자 지정 application이 인스턴스화되지 않은 경우 사용된다.
>
> `celery.task`모듈은 이전 API를 수용하기 위해 존재하며 사용자 정의 앱을 사용하는 경우 사용하지 않아야한다. 모듈 기반 API가 아닌 app 인스턴스의 메서드를 항상 사용해야한다.
>
> 예를 들어 이전 Task 기본 클래스는 태스크 메서드와 같은 새로운 기능과 호환되지 않는 호환성 기능을 여러 가지 사용할 수 있다.
>
> ```python
> from celery.task import Task  # << OLD Task base class.
> 
> from celery import Task  # << New base Class
> ```
>
> 이전 모듈 기반 API를 사용하는 경우에도 New base Class를 사용하는 것이 좋다.



## Breaking the chain

설정중인 현재 앱에 의존하는 것이 가능하지만, 가장 좋은 방법은 앱 인스턴스를 항상 필요한 곳에 전달하는 것이다.

이를 "app chain"이라 부르는데, 이유는 전달되는 app에 의존하는 일련의 인스턴스를 생성하기 때문이다.

다음 예제는 나쁜 습관을 보여준다.

```python
from celery import current_app

class Scheduler(object):
    
    def run(self):
        app = current_app
```

대신 앱을 인수로 받아야한다.

```python
class Scheduler(object):
    
    def __init__(self, app):
        self.app = app
```

내부적으로 Celery는 [celery.app.app_or_default()]()를 사용하여 모든것이 모듈 기반 호환성 API에서 작동하도록 한다.

```python
from celery.app import app_or_default

class Scheduler(object):
    def __init__(slef, app=None):
        self.app = app_or_default(app)
```

개발환경에서는 `CELERY_TRACE_APP` 환경 변수를 설정하여 앱 체인이 끊어지는 경우 예외를 발생시킬 수 있다.

```bash
$ CELERY_TRACE_APP=1 celery worker -l info
```

> ### API 진화
>
> Celery는 7년만에 많이 바뀌었다.
>
> 예를 들어, 처음에는 호출 가능 객체를 태스크로 사용가능했다.
>
> ```python
> def hello(to):
>     return 'hello {0}'.format(to)
> 
> >>> from celery.execute import apply_async
> >>> apply_async(hello, ('world!', ))
> ```
>
> 또는 `Task` 클래스를 만들어 특정 옵션을 설정하거나 다른 동작을 재정의 할 수 있다.
>
> ```python
> from celery.task import Task
> from celery.registry import tasks
> 
> class Hello(Task):
>     queue = 'hipri'
>     
>     def run(self, to):
>         return 'hello {0}'.format(to)
> task.register(Hello)
> 
> >>> Hello.delay('world')
> ```
>
> 나중에 pickle 이외에 serializer를 사용하기가 어려워졌기 때문에 임의의 호출 가능 함수를 전달하는 것이 anti-pattern이였고 2.0에서 기능이 제거 되어 task decorator로 대체되었다.



## Abstract Tasks

`task()` 데코레이터를 사용하여 만든 모든 작업은 응용프로그램의 기본 Task에서 상속받는다.

`base` 인수를 사용하여 다른 기본 클래스를 지정할 수 있다.

```python
@app.task(base=OtherTask)
def add(x, y):
    return x+ y
```

사용자 정의 태스크 클래스를 작성하려면 중립 기본 클래스인 `celery.Task`에서 상속해야한다.

```python
from celeery import Task

class DebugTask(Task):
    
    def __call__(self, *args, **kwrags):
        print('TASK STARTING: {0.name}[{0.request.id}]'.format(self))
        return super(DebugTask, self).__call__(*args, **kwargs)
```

> ### Tip:
>
> `__call__` 메서드를 재정의하는 경우 기본 호출 메서드가 작업이 직접 호출될 때 사용되는 기본 요청을 설정할 수 있도록 super를 호출하는 것이 매우 중요하다.

중립 기본 클래스는 특정 앱에 아직 바인딩 되지 않았기 때문에 특별하다. 일단 작업이 응용 프로그램에 바인딩되면 기본 값을 설정하는 등 구성을 읽는다.

기본 클래스를 구현하려면 `app.task()` 데코레이터를 사용하여 작업을 만들어야한다.

```python
@app.task(base=DebugTask)
def add(x, y):
    return x + y
```

`app.Task()` 속성을 변경하여 애플리케이션의 기본 클래스를 변경할수도 있다.

```python
>>> from celery import Celery, Task

>>> app = Celery()

>>> class MyBaseTask(Task):
...		queue = 'hipri'

>>> app.Task = MyBaseTask
>>> app.Task
<unbound MyBaseTask>

>>> @app.task
... def add(x, y):
...		return x + y

>>> add
<@task: __main__.add>
    
>>> add.__class__.mro()
[<class add of <Celery __main__:0x1012b4410>>,
 <unbound MyBaseTask>,
 <unbound Task>,
 <type 'object'>]
```

