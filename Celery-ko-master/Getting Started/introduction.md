# Introduction to Celery
```
- Task Queue란?
- 내가 필요한 것
- 시작하기
- 샐러리는..
- 기능
- 프레임워크 통합
- 설치
```

## Task Queue란?
`Task queue`는 스레드 또는 시스템에서 작업을 분배하는 매커니즘으로 사용된다.

`Task queue`의 입력은 `task`라고하는 작업 단위이다.  `worker process`는 새로운 작업을 수행하기 위해 지속적으로 작업 큐를 모니터링한다.

`Celery`는 메세지를 통해 통신하며 대개 `broker`를 사용하여 클라이언트와 `worker`를 중재한다. 클라이언트가 작업을 시작하기 위해 클라이언트가 대기열에 메세지를 추가하면 `broker`는 해당 메세지를 `worker`에게 전달한다.

`Celery` 시스템은 여러 `worker`와 `broker`로 구성되어있고 높은 가용성 및 수평 확장을 제공한다.

`Celery`는 Python으로 작성되었지만 프로토콜은 모든 언어로 구현될 수 있다.파이썬 이외에도 Node.js용 [node-celery](https://github.com/mher/node-celery)와 [PHP client](https://github.com/gjedeer/celery-php)가 있다.

언어 상호 운용성은 HTTP endpoint를 노출하고 이를 요청하는 태스크를 가질 수 있다. (webhooks).

## 내가 필요한 것

`Celery`는 메세지를 주고받으려면 `message transport`가 필요하다. RabbitMQ 및 Redis 브로커 전송은 기능이 완벽하지만 local 개발을 위해 SQLite를 사용하는 것을 포함하여 다른 많은 실험 솔루션에 대한 지원도 있다.

`Celery`는 단일 시스템, 여러 시스템 또는 데이터센터에서 실행될 수 있다.

## 시작하기

`Celery`를 처음 사용하는 경우이거나 3.1 버전에서 개발을 계속하지 않고 이전 버전에서 제공하는 경우, `getting started tutorial`을 읽어야한다.

- [First Steps with Celery](https://github.com/teachmesomething2580/Celery-ko/blob/master/Getting%20Started/First%20Steps%20with%20Celery.md)
- [Next Steps]()

## 샐러리는..

**- 간단하다**
	`Celery`는 사용하기 쉽고 유지보수가 용이하며 구성파일이 필요하지 않다.

	[maling-list]()와 [IRC channel]을 포함하여 지원을 위해 적극적이고 친숙한 커뮤니티가 있다.

	다음은 가장 간단한 응용 프로그램 중 하나이다.
	```python
	from celery import Celery

	app = Celery('hello', broker='amqp://guest@localhost//')

	@app.task
	def hello():
		return 'hello world'
	```

**- 높은 가용성**
	`woker`와 클라이언트는 연결이 끊어지거나 실패한경우 자동으로 재시도하며, 일부 `broker`는 *Primary/Primary or Primary/Replica*  복제 방식에서 높은 가용성을 지원한다.

**-빠르다**
	단일 샐러리 프로세스는 밀리 초 단위의 왕복 대기 시간 (RabbitMQ, librabbitmq 및 최적화 된 설정 사용)으로 분당 수백만 개의 작업을 처리 할 수 있다.

**-유연성**
	`Celery`의 거의 모든 부분을 자체적으로 확장하거나 Custom pool 구현, serializers, compression schemes, logging, schedulers, consumers, producers, broker transports 등과 같이 사용할 수 있다.

### 지원하는 것
- Brokers
	- RabbitMQ, Redis,
	- Amazon SQS, and more...

- Concurrency
	- prefork(multiprocessing),
	- Eventlet, gevent
	- solo (single threaded)

- Result Stores
	- AMQP, Redis
	- Memcached,
	- SQLAlchemy, Django ORM
	- Apache Cassandra, Elasticsearch

- Serialization
	- pickle, json, yaml, msgpack.
	- zlib, bzip2 compression.
	- Cryptographic message signing.

## 기능
**- Monitoring**
	모니터링 이벤트 스트림은 `worker`에 의해 생성되며 클러스터에서 수행중인 작업을 실시간으로 알려주는 내장 및 외부 도구에 의해 사용된다.

**- Work-flows**
	간단하고 복잡한 작업 흐름은 grouping, chaining, chunking 등 `canvas`라고 하는 강력한 기본 요소 세트를 사용하여 구성할 수 있다.

**- Time & Rate Limits **
	초 / 분 / 시간당 작업 수 또는 작업 실행 허용 시간을 제어할 수 있으며 특정 `worker` 또는 각 작업 유형에 대해 개별적으로 설정할 수 있다.

**- Scheduling**
	초 또는 **datetime**으로 작업을 실행할 시간을 지정하거나 간단한 간격을 기준으로 반복 이벤트에 주기적 작업을 사용하거나 분, 시간, 요일, 주, 월을 지원하는 Crontab 표현식을 사용할 수 있다.

**- Resource Leak Protection**
	**--max-tasks-per-child** 옵션은 메모리나 파일 디스크립터 같은 리소스를 누설하는 사용자 작업에 사용된다.

**- User Components**
	각 `worker`의 구성 요소는 사용자 정의 할 수 있으며 추가 구성 요소는 사용자가 정의할 수 있다. 작업자는 내부 구조를 세밀하게 제어할 수 있는 종속성 그래프인 "bootsteps"를 사용하여 빌드된다.

## 프레임워크 통합
`Celery`는 웹 프레임워크와 통합하기 쉽고, 일부는 통합 패키지를 가지고 있다.

|||
|:---------:|:----------------:|
|[Pyramid]()|[pyramid_celery]()|
|[Pylons]()|[celery_pylons]()|
|[Flask]()|not needed|
|[web2py]()|[web2py-celery]()|
|[Tornado]()|[tornado-celery]()|
|[Tryton]()|[celery_tryton]()|

[Django]()의 경우 [First steps with Django]()를 참고한다.
통합 패키지는 반드시 필요한것은 아니지만 개발을 쉽게 할 수 있으며 *fork (2)*에서 데이터베이스 연결을 종료하는 것과 같은 중요한 후크를 추가하는 경우가 있다.

## 설치

`Celery`는 Python Package Index (PyPI) 또는 소스를 통해 설치가 가능하다.
**pip**로 설치하기
```bash
$ pip install -U Celery
```

### Bundles
`Celery`는 주어진 기능에 대한 종속성을 설치하는데 사용할 수 있는 번들 그룹도 정의한다.

대괄호를 사용하여 **pip** 명령줄에서 지정 할 수 있다. 여러 번들은 쉼표로 구분하여 지정할 수 있다.

```bash
$ pip install "celery[librabbitmq]"
$ pip install "celery[librabbitmq,redis,auth,msgpack]"
```

다음의 번들 목록을 지원한다.
[**번들목록**](http://docs.celeryproject.org/en/latest/getting-started/introduction.html#serializers)
