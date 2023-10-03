# 카프카 운영하기

# 토픽 작업

kafka-topics.sh 툴을 사용해서 클러스터 내 토픽 생성, 변경, 삭제, 정보 조회를 할 수 있다. <br>
kafka-topics.sh를 사용하려면 --bootstrap-server 옵션에 연결 문자열과 포트를 넣어 줘야한다.

## 새 토픽 생성하기

--create 명령을 사용해서 새로운 토픽을 생성할 때 반드시 필요한 인수가 있다.

- **--topic**: 생성하려는 토픽의 이름
- **--replication-factor**: 클러스터 안에 유지되어야 할 레플리카의 개수
- **--partitions**: 토픽에서 생성할파티션의 개수

> 토픽 이름 짓기는 '_', '-', '.'를 사용할 수 있는데, '.'는 권장되지 않는다. <br>
> 카프카 내부적으로 사용하는 지표에서는 '.'를 '_'로 변환해서 처리하기 때문에 토픽 이름에 충돌이 발생할 수 있다.


파티션 각각이 2개의 레플리카를 가지는 8개의 파티션으로 이루어진 ‘my-topic’이라는 토픽을 생성하려면 다음과 같이 하면 된다.

```shell
$ bin/kafka-topics.sh --bootstrap-server localhost:9092 --create \
    --topic my-topic --replication-factor 2 --partitions 8 
Created topic "my-topic".
```

## 토픽 목록 조회하기

--list 명령은 클러스터 안의 모든 토픽을 보여준다.

```shell
$ bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
 __consumer_offsets
my-topic
other-topic
```

## 토픽 상세 내역 조회하기

클러스터 안에 있는 1개 이상의 토픽에 대해 상세한 정보를 보는 것 역시 가능하다. <br>
파티션 수, 재정의된 토피 설정, 파티션별 레플리카 할당 역시 함께 출력된다. <br>
하나의 토픽에 대해서만 보고 싶다면 --topic 인수를 지정해주면 된다.

```shell
$ bin/kafka-topics.sh --boostrap-server localhost:9092 --describe --topic my-topic
Topic: my-topic PartitionCount: 8 ReplicationFactor: 2 Configs: segment.
bytes=1073741824
        Topic: my-topic Partition: 0 Leader: 1 Replicas: 1,0 Isr: 1,0
        Topic: my-topic Partition: 1 Leader: 0 Replicas: 0,1 Isr: 0,1
        Topic: my-topic Partition: 2 Leader: 1 Replicas: 1,0 Isr: 1,0
        Topic: my-topic Partition: 3 Leader: 0 Replicas: 0,1 Isr: 0,1
        Topic: my-topic Partition: 4 Leader: 1 Replicas: 1,0 Isr: 1,0
        Topic: my-topic Partition: 5 Leader: 0 Replicas: 0,1 Isr: 0,1
        Topic: my-topic Partition: 6 Leader: 1 Replicas: 1,0 Isr: 1,0
        Topic: my-topic Partition: 7 Leader: 0 Replicas: 0,1 Isr: 0,1
$
```

> --describe 명령은 출력을 필터링할 수 있는 몇몇 유용한 옵션들 역시 가지고 있다.

- **--topics-with-overrides**: 설정 중 클러스터 기본값을 재정의한 것이 있는 토픽들을 보여준다.
- **--exclude-internal**: '__'로 시작하는(내부 토픽 앞에 붙는) 모든 토픽들을 결과에서 제외한다.
- **--under-replicated-partitions**: 1개 이상의 레플리카가 리더와 동기화되지 않고 있는 모든 파티션을 보여준다.
- **--at-min-isr-partitions**: 레플리카 수가 인-싱크 레플리카최소값과 같은 모든 파티션을 보여준다.
- **--under-min-isr-partitions**: ISR 수가 쓰기 작업이 성공하기 위해 필요한 최소 레플리카 수에 미달하는 모든 파티션을 보여준다.
- **--unavailable-partitions**: 리더가 없는 모든 파티션을 보여준다.

## 파티션 추가하기

파티션 수를 증가시키는 가장 일반적인 이유는 단일 파티션에 쏟아지는 처리량을 줄임으로써 토픽을 더 많은 브로커에 대해 수평적으로 확장시키기 위해서이다. <br>
나의 파티션은 컨슈머 그룹 내의 하나의 컨슈머만 읽을 수 있기 때문에, 컨슈머 그룹 안에서 더 많은 컨슈머를 활용해야 하는 경우에도 토픽의 파티션 수를 증가시킬 수 있다.

```shell
# ‘my-topic’의 파티션 수를 16개로 증가
$ bin/kafka-topics.sh --bootstrap-server localhost:9092 --alter --topic my-topic --partitions 16
```

> 키가 있는 메시지를 갖는 토픽에 파티션을 추가하는 것은 매우 어려울 수 있다.  <br>
> 파티션의 수가 변하면 키값에 대응되는 파티션도 달라지기 때문이다. <br>
> 이러한 이유 때문에 키가 포함된 메시지를 저장하는 토픽을 생성할 때는 미리 파티션의 개수를 정해 놓고, 일단 생성한 뒤에는 설정한 파티션의 수를 바꾸지 않는 것이 좋다.

## 토픽 삭제하기

메시지가 하나도 없는 토픽이라 할지라도 디스크 공간이나 파일 핸들, 메모리와 같은 클러스터 자원을 잡아먹는다. <br>
컨트롤러 역시 아무 의미 없는 메타데이터에 대한 정보를 보유하고 있어야 하는데, 이는 대규모 클러스터에서는 성능을 하락으로 이어진다. <br>
만약 토픽이 더 이상 필요가 없다면 이러한 자원을 해제하기 위해 삭제할 수 있다. <br>
**이를 위해서는 클러스터 브로커의 delete.topic.enable 옵션이 true로 설정되어 있어야 한다.**

토픽 삭제는 비동기적인 작업이라, 명령을 실행하면 토픽이 삭제될 것이라고 표시만 될 뿐, 삭제 작업이 즉시 일어나지는 않는다.

```shell
# ‘my-topic’ 토픽을 삭제하는 예
$ bin/kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic my-topic
```

<br>
<hr>

# 컨슈머 그룹

컨슈머 그룹은 서로 협업해서 여러 개의 토픽 혹은 하나의 토픽에 속한 여러 파티션에서 데이터를 읽어오는 카프카 컨슈머의 집단을 가리킨다. <br>
**kafka-consumer-groups.sh** 툴을 사용하면 클러스터에서 토픽을 읽고 있는 컨슈머 그룹을 관리할 수 있다. 

## 컨슈머 그룹 목록 및 상세 내역 조회하기

```shell
$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list
console-consumer-95554
console-consumer-9581
my-consumer
$
```

목록에 포함된 모든 그룹에 대해서 --list 매개변수를 --describe로 바꾸고 --group 매개변수를 추가함으로써 상세한 정보를 조회할 수 있다.

```shell
$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 \ 
    --describe --group my-consumer
GROUP LAG HOST TOPIC PARTITION CURRENT-OFFSET LOG-END-OFFSET CONSUMER-ID CLIENT-ID
My-consumer my-topic 0 2 4 2 consumer-l-029af89c-873c-4751-a720-cefd41a669d6 127.0.0.1 consumer-1
my-consumer my-topic 1 2 3 1 consumer-l-029af89c-873c-4751-a720-cefd41a669d6 127.0.0.1 consumer-1
```

| 필드    | 설명                                                             |
|-------|----------------------------------------------------------------|
| GROUP | 컨슈머 그룹의 이름                                                     |
|TOPIC| 읽고 있는 토픽의 이름                                                   |
|PARTITION| 읽고 있는 파티션의 ID                                                  |
|CURRENT-OFFSET| 컨슈머 그룹이 이 파티션에서 다음번에 읽어올 메시지의 오프셋. 이 파티션에서의 컨슈머 위치라고 할 수 있다    |
|LOG-END-OFFSET| 브로커 토픽 파티션의 하이 워터마크 오프셋 현재값. 이 파티션에 쓰여질 다음번 메시지의 오프셋이라고 할 수 있다 |
|LAG|컨슈머의 CURRENT-OFFSET과 브로커의 LOG-END-OFFSET 간의 차이.|
|CONSUMER-ID|설정된 client-id 값을 기준으로 생성된 고유한 consumer-id.|
|HOST|컨슈머 그룹이 읽고 있는 호스트의 IP 주소.|
|CLIENT-ID|컨슈머 그룹에서 속한 클라이언트를 식별하기 위해 클라이언트에 설정된 문자열.|

## 컨슈머 그룹 삭제하기

```shell
# ‘my-consumer’라는 이름의 컨슈머 그룹을 삭제
$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --delete --group my-consumer
Deletion of requested consumer groups ('my-consumer') was successful. 
$
```

## 오프셋 관리

저장된 오프셋을 가져오거나 아니면 새로운 오프셋을 저장하는 것도 가능하다. <br>
이것은 뭔가 문제가 있어서 메시지를 다시 읽어와야 하거 나 뭔가 문제가 있는 메시지를 건너뛰기 위해 컨슈머의 오프셋을 리셋하는 경우 유용하다.

### 오프셋 내보내기

컨슈머 그룹을 csv 파일로 내보내려면 --dry-run 옵션과 함께 --reset-offsets 매개변수를 사용해 주면 된다.

```shell
# 'my-consumer' 컨슈머 그룹이 읽고 있는 'my-topic' 토픽의 오프셋을 offsets.csv 파일로 내보내는 방법
$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 \ 
    --export --group my-consumer --topic my-topic \
    --reset-offsets --to-current --dry-run > offsets.csv
```

### 오프셋 가져오기

```shell
# offsets.csv 파일로부터 my-consumer 컨슈머 그룹의 오프셋을 가져온다.
$ kafka-consumer-groups.sh --bootstrap-server \
    --reset-offsets  --group my-consumer \
    --from-file offsets.csv --execute
```

<br>
<hr>

# 동적 설정 변경

토픽, 클라이언트, 브로커 등 많은 설정이 클러스터를 끄거나 재설치할 필요 없이 돌아가는 와중에 동적으로 바꿀 수 있는 설정은 굉장히 많다. <br>
이러한 설정들을 수정할 때는 kafka-configs.sh가 주로 사용된다.

## 토픽 설정 기본값 재정의하기

```shell
$ bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter --entity-type topics --entity-name {토픽 이름} \
  --add-config {key}={value}[,{key}={value}...]
```

```shell
# my-topic 토픽의 보존 기한을 1시간으로 설정
$ bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter --entity-type topics --entity-name my-topic \
  --add-config retention.ms=3600000
```

## 브로커 설정 기본값 재정의하기

브로커와 클러스터 수준 설정은 주로 클러스터 설정 파일에 정적으로 지정되지만, 카프카를 재설치할 필요 없이 프로세스가 돌아가는 중에 재정의가 가능한 설정들도 많다. <br>
[재정의 가능한 항목](https://kafka.apache.org/documentation/#brokerconfigs)

## 재정의된 설정 상세 조회

```shell
$ bin/kafka-configs.sh --bootstrap-server localhost:9092 \
 --describe --entity-type topics --entity-name my-topic
Configs for topics:my-topic are
retention.ms=3600000
$
```

## 재정의된 설정 삭제

```shell
$ bin/kafka-configs.sh --bootstrap-server localhost:9092 \
 --alter --entity-type topics --entity-name my-topic 
 --delete-config retention.ms
Updated config for topic: "my-topic". 
$
```

<br>
<hr>

# 쓰기 작업과 읽기 작업

카프카를 사용할 때 애플리케이션이 제대로 돌아가는지를 확인하기 위해 수동으로 메시지를 쓰거나 샘플 메시지를 읽어와야 하는 경우가 있다. <br>
이러한 작업을 위해 kafka-console-consumer.sh와 kafka-console-producer.sh가 제공된다

## 콘솔 프로듀서

kafka-console-producer.sh 툴을 사용해서 카프카 토픽에 메시지를 써넣을 수 있다.

```shell
$ bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic my-topic
>Message 1
>Test Message 2
>Test Message 3
>Message 4 
>AD
$
```

## 콘솔 컨슈머

kafka-console-consumer.sh 툴을 사용하면 카프카 클러스터 안의 1개 이상의 토픽에 대해 메시지를 읽어올 수 있다.

```shell
$ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092
  --whitelist 'my.*' \
  --from-beginning
Message 1
Test Message 2
Test Message 3
Message 4
^C
$
```

<br>
<hr>

# 파티션 관리

카프카 클러스터 안의 브로커 간에 메시지 트래픽의 균형을 직접 맞춰 줘야 할 때 요긴하게 사용할수 있다.

## 선호 레플리카 선출

각 파티션은 신뢰성을 보장하기 위해 여러 개의 레플리카를 가질 수 있다. <br>
전체 카프카 클러스터에 대해 부하를 고르게 나눠주려면 리더 레플리카를 전체 브로커에 걸쳐 균형 있게 분산해줄 필요가 있다.

> 리더 레플리카는 레플리카 목록에 있는 첫 번째 인-싱크 레플리카로 정의된다. <br>
> 하지만, 만약 브로커가 중단되거나 나머지 브로커와의 네트워크 연결이 끊어지면 다른 인-싱크 레플리카 중 하나가 리더 역할을 인계받게 되지만, 리더 역할이 원래 리더를 맡고 있던 레플리카로 자동으로 복구되지는 않는다. <br>
> 그렇기 때문에 자동 리더 밸런싱 기능이 꺼져 있을 경우, 처음 설치했을 때는 잘 균형을 이루던 것이 나중에는 엉망진창이 될 수 있다. <br>
> 만약 카프카 클러스터에 균형이 맞지 않을 경우, 선호 레플리카 선출을 실행시킬 수 있다

```shell
# 클러스터 내 모든 선호 레플리카 선출을 시작하는 명령
$ bin/kafka-leader-election.sh --bootstrap-server localhost:9092 \
  --election-type PREFERRED \
  --all-topic-partitions
$
```

## 파티션 레플리카 변경하기

아래의 경우 파티션의 레플리카 할당을 수동으로 변경해줘야 한다.

- 자동으로 리더 레플리카를 분산시켜 주었는데도 브로커간 부하가 불균등할 때
- 브로커가 내려가서 파티션이 불완전 복제되고 있을 때
- 새로 추가된 브로커에 파티션을 빠르게 분산시켜주고 싶을 때
- 토픽의 복제 팩터를 변경해주고 싶을 경우

이런 경우 kafka-reassign-partitions.sh를 사용한다. <br>
[사용법](https://kyeongseo.tistory.com/entry/kafka-kafka-reassign-partitionssh%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%95%9C-%ED%8C%8C%ED%8B%B0%EC%85%98-%EC%9E%AC%ED%95%A0%EB%8B%B9)






