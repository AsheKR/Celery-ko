# Using Redis

## Installation

`Redis` 지원을 위해 추가적인 의존성 설치가 필요하다. `celery[redis]` 번들을 사용하여 Celery와 이러한 종속성을 모두 설치할 수 있다.

```bash
$ pip install -U "celery[redis]"
```

## Configuration

설정은 간단하다. 단지 Redis database의 위치를 설정하기만 하면 된다.
```python
app.conf.broker_url = 'redis://localhost:6379/0'
```
URL 형식은 다음과 같다.
```
redis://:password@hostname:port/db_number
```

scheme 뒤 모든 필드는 선택사항이며  `localhost, 6379포트, 0번의 데이터베이스`r가 기본이된다.

Unix 소켓 연결을 사용하는 경우 URL은 다음의 형식을 사용한다.
```
redis+socker:///path/to/redis.sock
```

Unix 소켓을 사용할 때 다른 데이터베이스 번호를 지정하는 것은 `virtual_host` 매개변수를 URL에 추가하여 사용 가능하다.
```
redis+socker:///path/to/redis.sock?virtual_host=db_number
```

또한 쉽게 Redis Sentinel 목록에 직접적으로 연결할 수 있다.
> Redis Sentinel이란 운영중 예기치않게 마스터가 다운되었다면 관리자가 이를 감지해 슬레이브를 마스터로 올리고 클라이언트들이 새로운 마스터에 접속 할 수 있게 해준다.
[Redis Sentinel 소개](http://redisgate.jp/redis/sentinel/sentinel.php)

```python
app.conf.broker_url = 'sentinel://localhost:26379;sentinel://localhost:26380;sentinel://localhost:26381'
app.conf.broker_transport_options = { 'master_name': "cluster1" }
```

## Visibility Timeout

`visibility Timeout`은 메세지가 다른 `worker`에게 다시 전달되기 전 `worker`가 작업을 확인하기를 기다리는 시간이다. 아래 [주의사항](http://docs.celeryproject.org/en/latest/getting-started/brokers/redis.html#redis-caveats)을 확인한다.

이 옵션은 [broker_transport_options](http://docs.celeryproject.org/en/latest/userguide/configuration.html#std:setting-broker_transport_options) 설정이다.

```python
app.conf.broker_transport_options = {'visibility_timeout': 3600}  # 1 hour.
```

## Results

`Redis`의 작업의 상태와 값을 저장하고싶으면 다음 설정을 한다.
```python
app.conf.result_backend = 'redis://localhost:6379/0'
```
`Redis result backend`을 지원하는 모든 옵션은 [Redis backend settings](http://docs.celeryproject.org/en/latest/userguide/configuration.html#conf-redis-result-backend)를 참조한다.

Sentinel을 사용하는 경우 [result_backend_transport_options](http://docs.celeryproject.org/en/latest/userguide/configuration.html#std:setting-result_backend_transport_options)를 사용하여 `master_name`을 설정해야한다.

```python
app.conf.result_backend_transport_options = {'master_name': "mymaster"}
```

## 주의사항

### Fanout prefix
브로드캐스트 메세지는 기본적으로 모든 가상 호스트에서 볼 수 있다.

활성화된 가상 호스트에 의해서만 받아야하기때문에 transport option을 설정한다.

```python
app.conf.broker_transport_options = {'fanout_prefix': True}
```

이전 버전을 실행하는 `wokrer` 또는 이 설정을 사용하지 않는 `worker`와 통신 할 수 없다.

### Fanout patterns

`worker`는 기본적으로 모든 작업관련 이벤트를 받는다.

이를 방지하려면 `worker`가 `worker` 관련 이벤트만 받을 수 있도록 `fanout_patterns` 옵션을 설정해야한다.

```python
app.conf.broker_transport_options = {'fanout_patterns': True}
```

이 변경 사항은 이전 버전과 호환되지 않으므로 클러스터의 모든 `worker`는 이 옵션을 활성화해야한다. 그렇지 않으면 통신할 수 없다.

이 옵션은 나중에 default로 활성화 될것이다.

### Visibility timeout

`Visibility timeout`내 작업을 확인하지 않으면 작업이 다른 `worker` 에게 다시 전달되어 실행된다.

이로 인해 실행 시간이 `Visibility timeout`을 초과하는 `ETA/countdown/retry tasks`에 문제가 발생한다. 실제로 그런 일이 발생하면 다시 실행되고 반복된다.

따라서 사용할 시간이 가장 긴 `ETA` 시간과 일치하도록 `Visibility timeout`을 늘려야한다.

`Celery`는 `worker` 종료시 메세지를 재전송하므로 긴 `Visibility timeout`은 정전이나 강제 종료된 작업자의 경우 잃어버린 작업의 재전송을 지연시킬 뿐이다.

정기 작업은 `Visibility timeout`의 영향을 받지 않는다. `ETA/countdown`과는 별개의 개념이기 때문이다.

```python
app.conf.broker_transport_options = {'visibility_timeout': 43200}
```
값은 초로 이루어진 int 여야한다.

### Key Eviction

일부 상황에서 Redis가 데이터베이스에서 키를 제거할 수 있다.

다음과 같은 오류가 발생할 경우:
```
InconsistencyError: Probably the key ('_kombu.binding.celery') has been
removed from the Redis database.
```

redis 구성 파일에서 `timeout` 매개변수를 0으로 설정하여 `redis-server`가 키를 제거하지 않도록 구성할 수 있다.
