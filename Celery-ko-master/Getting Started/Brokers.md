# Brokers

## Broker Instructions

- [Using RabbitMQ]()
- [Using Redis](https://github.com/teachmesomething2580/Celery-ko/blob/master/Getting%20Started/Brokers/Using%20Redis.md)
- [Using Amazon SQS]()

## Broker Overview

|Name|Status|Monitoring|Remote Control|
|-|-|-|-|
|*RabbitMQ*|Stable|Yes|Yes|
|*Redis*|Stable|Yes|Yes|
|*Amazon SQS*|Stable|No|No|
|*Zookeeper*|Experimental|No|No|

실험브로커는 작동할 수 있지만 전용 관리자가 없다.

`Monitoring`을 지원하지 않는 것은 이벤트를 구현하지 않았음을 의미하며
따라서 `celery events`, `celerymon` 그리고 다른 이벤트 기반 모니터링 도구는 작동하지 않는다.

`Remote Control`은 셀러리 검사 및 셀러리 제어 명령을 사용하여 런타임에 작업자를 검사하고 관리할 수 있는 기능을 의미한다.
