# 카프카 입문

카프카에 저장된 데이터는 순서를 유지한 채로 **지속성**있게 보관되며 읽을 수 있다. <br>
또한, 확장시 성능을 향상시키고 실패가 발생하더라도 데이터 사용에는 문제가 없도록 시스템 안에서 데이터를 분산시켜 저장할 수 있다.

## 메시지와 배치

카프카에서 데이터의 기본 단위는 메시지이다. <br>
메시지는 **키**라 불리는 메타데이터를 포함할 수도 있는데, 이는 메시지를 저장할 파티션을 결정하기 위해 사용된다.

> 키값에서 일정한 해시값을 생성한 뒤 이 값을 토픽의 파티션 수로 나눴을 때 나오는 나머지 값에 해당하는 파티션에 메시지를 저장해서, <br>
> 같은 키값을 가진 메시지는(파티션 수가 변하지 않는 한) 항상 같은 파티션에 저장된다.

<br>

카프카는 효율성을 위해 메시지를 **배치** 단위로 저장한다. <br>
배치는 같은 토픽의 파티션에 쓰여지는 메시지들의 집합이고, 메시지를 쓸 때마다 네트워크상에서 신호가 오가는 것은 막대한 오버헤드를 발생시키는데, 메시지를 배치 단위로 모아서 쓰면 이것을 줄일 수 있다. <br>
물론 이것은 **지연(latency)** 과 **처리량(throughput)** 사이에 트레이드오프를 발생시킨다. <br>
배치 크기가 커질수록 시간당 처리되는 메시지의 수는 늘어나지만, 각각의 메시지가 전달되는 데 걸리는 시간은 늘어나는 것이다. <br>
*배치는 더 효율적인 데이터 전송과 저장을 위해 약간의 처리 능력을 들여서 압축되는 경우가 많다.*

## 스키마

각 애플리케이션의 필요에 따라 사용 가능한 메시지 스키마에는 여러 가지가 있는데, 가장 간단한 방법으로는 쓰기 쉽고 사람이 알아볼 수 있는 **JSON**, **XML**이 있다. <br>
하지만 이 방식들은 타입 처리 기능이나 스키마 버전 간의 호환성 유지 기능이 떨어져서 아파치 카프카 개발자들은 **아파치 에이브로**를 선호한다.

## 토픽과 파티션

카프카에 저장되는 메시지는 **토픽** 단위로 분류되고, 토픽은 다시 여러 개의 **파티션**으로 나뉘어진다.

토픽에 여러 개의 파티션이 있는 만큼 토픽 안의 메시지 전체에 대해 순서는 보장되지 않고, 단일 파티션 안에서만 순서가 보장된다. <br>
각 파티션은 서로 다른 서버에 저장될 수 있기 때문에 하나의 토픽이 여러 개의 서버로 수평적으로 확장되어 하나의 서버의 용량을 넘어가는 성능을 보여 줄 수 있다.

**파티션은 복제될 수 있다.** <br>
서로 다른 서버들이 동일한 파티션의 복제본을 저장하고 있기 때문에 서버 중 하나에 장애가 발생한다고 해서 읽거나 쓸 수 없는 상황이 벌어지지는 않는다.

<img width="693" alt="스크린샷 2023-09-22 오후 4 01 47" src="https://github.com/flataex/kafka-study/assets/87420630/cf8094d4-a7c5-4726-a008-af7b5ca89312">

> 카프카에는 **스트림**이라는 용어가 자주 사용된다. <br>
> 스트림은(파티션의 개수와 상관없이) 하나의 토픽에 저장된 데이터로 간주되고, producer -> consumer로의 데이터 흐름을 나타낸다.


## 프로듀서와 컨슈머

**프로듀서**는 새로운 메시지를 생성한다. <br>
기본적으로 프로듀서는 메시지를 쓸 때 토픽에 속한 파티션들 사이에 고르게 나눠서 쓰도록 되어 있다. <br>
하지만 어떠한 경우에는, 프로듀서가 특정한 파티션을 지정해서 메시지를 쓰기도 하는데, 이것은 보통 메시지 키와 키값의 해시를 특정 파티션으로 대응시켜 주는 파티셔너를 사용해서 구현된다. <br>
이렇게 하면 동일한 키 값을 가진 모든 메시지는 같은 파티션에 저장된다.

**컨슈머**는 메시지를 읽는다. <br>
컨슈머는 1개 이상의 토픽을 구독해서 여기에 저장된 메시지들을 각 파티션에 쓰여진 순서대로 읽어 온다. <br>
컨슈머는 메시지의 오프셋을 기록함으로써 어느 메시지까지 읽었는지를 유지한다. <br>
주어진 파티션의 각 메시지는 고유한 오프셋을 가지며, 뒤에 오는 메시지가 앞의 메시지보다 더 큰 오프셋을 가진다.

컨슈머는 컨슈머 그룹의 일원으로서 작동한다. <br>
컨슈머 그룹은 토픽에 저장된 데이터를 읽어오기 위해 협업하는 하나 이상의 컨슈머로 이루어지고, 각 파티션이 하나의 컨슈머에 의해서만 읽히도록 한다.

<img width="733" alt="스크린샷 2023-09-22 오후 4 08 31" src="https://github.com/flataex/kafka-study/assets/87420630/3cd871c7-bfe7-4f9c-8a34-d1b219d58989">

<br>

## 브로커와 클러스터

하나의 카프카 서버를 **브로커**라고 부르고, 브로커는 프로듀서로부터 메시지를 전달받아 오프셋을 할당한 뒤 디스크 저장소에 쓴다. <br>
브로커는 컨슈머의 파티션 읽기 요청 역시 처리하고 발행된 메시지를 보내준다. <br>
*시스템 하드웨어의 성능에 따라 다르겠지만, 하나의 브로커는 초당 수천 개의 파티션과 수백만 개의 메시지를 쉽게 처리할 수 있다.*

카프카 브로커는 **클러스터**의 일부로서 작동하도록 설계되었다. <br>
하나의 클러스터 안에 여러 개의 브로커가 포함될 수 있으며, 그 중 하나의 브로커가 클러스터 컨트롤러의 역할을 하게 된다 <br>
컨트롤러는 파티션을 브로커에 할당해주거나 장애가 발생한 브로커를 모니터링하는 등의 관리 기능을 담당한다. <br>
파티션은 클러스터 안의 브로커 중 하나가 담당하며, 그 브로커를 **파티션 리더**라고 부른다. <br>
복제된 파티션이 여러 브로커에 할당될 수도 있는데 이것들은 파티션의 **팔로워**라고 부른다.

> **복제** 기능은 파티션의 메시지를 중복 저장함으로써 리더 브로커에 장애가 발생했을 때 팔로워 중 하나가 리더 역할을 이어받을 수 있도록 한다. <br>
> 모든 프로듀서는 리더 브로커에 메시지를 발행해야 하지만, 컨슈머는 리더나 팔로워 중 하나로부터 데이터를 읽어올 수 있다.

<img width="712" alt="스크린샷 2023-09-22 오후 4 13 16" src="https://github.com/flataex/kafka-study/assets/87420630/6f7be0a1-b2a9-4b1f-9dbd-183ae0c4fe71">

## 다중 클러스터

다수의 클러스터를 운용하면 다음과 같은 장점이 있다.

- 데이터 유형별 분리
- 보안 요구사항을 충족시키기 위한 격리
- 재해 복구를 대비한 다중 데이터센터





