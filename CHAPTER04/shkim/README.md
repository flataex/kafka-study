# 카프카 컨슈머 (개념)

## 컨슈머와 컨슈머 그룹

만약 프로듀서가 애플리케이션이 검사할 수 있는 속도보다 더 빠른 속도로 토픽에 메시지를 쓰게 된다면 <br>
데이터를 읽고 처리하는 컨슈머가 하나뿐이라면 애플리케이션은 새로 추가되는 메시지의 속도를 따라잡을 수 없기 때문에 메시지 처리가 계속해서 뒤로 밀리게 될 것이다. <br>
따라서, 우리는 토픽으로부터 데이터를 읽어 오는 작업을 확장할 수 있어야 한다. <br>
*여러 개의 프로듀서가 동일한 토픽에 메시지를 쓰듯이, 여러 개의 컨슈머가 같은 토픽으로부터 데이터를 분할해서 읽어올 수 있게 해야 하는 것이다.*

카프카 컨슈머는 보통 컨슈머 그룹의 일부로서 작동하고, 동일한 컨슈머 그룹에 속한 여러 개의 컨슈머들이 동일한 토픽을 구독할 경우, 각각의 컨슈머는 해당 토픽에서 서로 다른 파티션의 메시지를 받는다.

<img width="386" alt="스크린샷 2023-09-23 오후 4 12 36" src="https://github.com/flataex/kafka-study/assets/87420630/88c8a344-ee94-465f-80ed-5c6ad8438573">
<img width="431" alt="스크린샷 2023-09-23 오후 4 13 07" src="https://github.com/flataex/kafka-study/assets/87420630/ab38bc65-aaa7-4bf3-89e3-b1f1673d6725">
<img width="510" alt="스크린샷 2023-09-23 오후 4 13 26" src="https://github.com/flataex/kafka-study/assets/87420630/c9248e2e-909f-4eb9-bb05-53268d5f7275">
<img width="680" alt="스크린샷 2023-09-23 오후 4 15 40" src="https://github.com/flataex/kafka-study/assets/87420630/640540c7-2bee-451b-b47a-c4028828ba40">


> 카프카 컨슈머가 지연 시간이 긴 작업을 수행하는 것은 흔하다. <br>
> 이런 경우 하나의 컨슈머로 토픽에 들어오는 데이터의 속도를 감당할 수 없을 수도 있기 때문에 컨슈머를 추가함으로써 단위 컨슈머가 처리하는 파티션과 메시지의 수를 분산시키는 것이 일반적인 규모 확장 방식이다. <br>
> 이것은 토픽을 생성할 때 파티션 수를 크게 잡아주는 게 좋은 이유이기도 한데, 부하가 증가함에 따라서 더 많은 컨슈머를 추가 할 수 있게 해주기 때문이다. <br>
> 토픽에 설정된 파티션 수 이상으로 컨슈머를 투입하는 것이 아무 의미 없다. *(몇몇 컨슈머는 그냥 놀게 된다.)*

<br>

한 애플리케이션의 규모를 확장하기 위해 컨슈머 수를 늘리는 경우 이외에도 여러 애플리케이션이 동일한 토픽에서 데이터를 읽어와야 하는 경우 역시 매우 흔하다. <br>
이러한 경우 우리는 각각의 애플리케이션이 전체 메시지의 일부만 받는 게 아니라 전부 다 받도록 해야 한다. <br>
이렇게 하려면, 애플리케이션이 각자의 컨슈머 그룹을 갖도록 해야 하고, 카프카는 성능 저하 없이 많은 수의 컨슈머와 컨슈머 그룹으로 확장이 가능하다.

<img width="582" alt="스크린샷 2023-09-23 오후 4 18 05" src="https://github.com/flataex/kafka-study/assets/87420630/3c074920-6998-44e6-9c93-2a2b3fbd08f9">

## 컨슈머 그룹과 파티션 리밸런스

컨슈머 그룹에 속한 컨슈머들은 자신들이 구독하는 토픽의 파티션들에 대한 소유권을 공유한다. <br>
새로운 컨슈머를 컨슈머 그룹에 추가하면 이전에 다른 컨슈머가 읽고 있던 파티션으로부터 메시지를 읽기 시작한다. <br>
*컨슈머가 종료되거나 크래시가 났을 경우도 해당 컨슈머가 컨슈머 그룹에서 나가면 원래 이 컨슈머가 읽던 파티션들은 그룹에 잔류한 나머지 컨슈머 중 하나가 대신 받아서 읽기 시작하는 것*

컨슈머에 파티션을 재할당하는 작업은 컨슈머 그룹이 읽고 있는 토픽이 변경되었을 때도 발생한다. *(ex. 운영자가 토픽에 새 파티션을 추가했을 경우)* <br>
컨슈머에 할당된 파티션을 다른 컨슈머에게 할당해주는 작업을 **리밸런스**라고 한다. <br>
**리밸런스는 컨슈머 그룹에 쉽고 안전하게 컨슈머를 제거할 수 있도록 해주고, 높은 가용성과 규모 가변성을 제공한다.**

### 조급한 리밸런스

**조급한 리밸런스**가 실행되는 와중에 모든 컨슈머는 읽기 작업을 멈추고 자신에게 할당된 모든 파티션에 대한 소유권을 포기한 뒤 컨슈머 그룹에 다시 참여하여 완전히 새로운 파티션 할당을 전달받는다. <br>
이러한 방식은 근본적으로 전체 컨슈머 그룹에 대해 짧은 시간 동안 작업을 멈추게 한다. <br>
*우선 모든 컨슈머가 자신에게 할당된 파티션을 포기하고, 파티션을 포기한 컨슈머 모두가 다시 그룹에 참여한 뒤에야 새로운 파티션을 할당받고 읽기 작업을 재개할 수 있다.*

<img width="772" alt="스크린샷 2023-09-23 오후 4 26 43" src="https://github.com/flataex/kafka-study/assets/87420630/ceab86f5-f7e6-455e-9d89-892bd5265594">

### 협력적 리밸런스

**협력적 리밸런스**는 한 컨슈머에게 할당되어 있던 파티션만을 다른 컨슈머에 재할당한다. *(재할당되지 않은 파티션에서 레코드를 읽어서 처리하던 컨슈머들은 작업에 방해받지 않고 하던 일을 계속할 수 있는 것)* <br>
이 경우 리밸런싱은 2개 이상의 단계에 걸쳐서 수행된다.

1. 컨슈머 그룹 리더가 다른 컨슈머들에게 각자에게 할당된 파티션 중 일부가 재할당될 것이라고 통보하면, 컨슈머들은 해당 파티션에서 데이터를 읽어오는 작업을 멈추고 해당 파티션에 대한 소유권을 포기한다.
2. 컨슈머 그룹 리더가 이 포기된 파티션들을 새로 할당한다.

이 점진적인 방식은 안정적으로 파티션이 할당될 때까지 몇 번 반복될 수 있지만, 조급한 리밸런스 방식에서 발생하는 전체 작업이 중단되는 사태는 발생하지 않는다.

<img width="731" alt="스크린샷 2023-09-23 오후 4 29 14" src="https://github.com/flataex/kafka-study/assets/87420630/f597da25-6fb3-4709-b486-12ec6057a130">

> 컨슈머는 해당 컨슈머 그룹의 **그룹 코디네이터** 역할을 지정받은 카프카 브로커에 **하트비트**를 전송함으로써 멤버십과 할당된 파티션에 대한 소유권을 유지한다. <br>
> 하트비트는 컨슈머의 백그라운드 스레드에 의해 전송되는데, 일정한 간격을 두고 전송되는 한 연결이 유지되고 있는 것으로 간주된다. <br>
> 만약 컨슈머가 일정 시간 이상 하트비트를 전송하지 않는다면, 세션 타임아웃이 발생하면서 그룹 코디네이터는 해당 컨슈머가 죽었다고 간주하고 리밸런스를 실행한다. <br>
> 컨슈머를 깔끔하게 닫아줄 경우 컨슈머는 그룹 코디네이터에게 그룹을 나간다고 통지하는데, 그러면 그룹 코디네이터는 즉시 리밸런스를 실행함으로써 처리가 정지되는 시간을 줄인다.


## 정적 그룹 멤버십

기본적으로, 컨슈머가 갖는 컨슈머 그룹의 멤버로서의 자격(멤버십)은 일시적이다. <br>
컨슈머가 컨슈머 그룹을 떠나는 순간 해당 컨슈머에 할당되어 있던 파티션들은 해제되고, 다시 참여하면 새로운 멤버 ID가 발급되면서 리밸런스 프로토콜에 의해 새로운 파티션들이 할당되는 것이다.

group.instance.id 설정으로 컨슈머가 컨슈머 그룹의 정적인 멤버가 되도록 할 수 있다. <br>
컨슈머가 정적 멤버로서 컨슈머 그룹에 처음 참여하면 평소와 같이 해당 그룹이 사용하고 있는 파티션 할당 전략에 따라 파티션이 할당된다. <br>
하지만 이 컨슈머가 꺼질 경우, 자동으로 그룹을 떠나지는 않는다. <br>
그리고 컨슈머가 다시 그룹에 조인하면 멤버십이 그대로 유지되기 때문에 리밸런스가 발생할 필요 없이 예전에 할당받았던 파티션들을 그대로 재할당받는다.

> 정적 그룹 멤버십은 애플리케이션이 각 컨슈머에 할당된 파티션의 내용물을 사용해서 로컬 상태나 캐시를 유지해야 할 때 편리하다. <br>
> 캐시를 재생성하는 것이 시간이 오래 걸릴 때, 컨슈머가 재시작할 때마다 이 작업을 반복하고 싶지는 않을 것이다. <br>
> 반대로 생각하면, 각 컨슈머에 할당된 파티션들이 해당 컨슈머가 재시작한다고해서 다른 컨슈머로 재할당되지는 않는다는 점을 기억할 필요가 있다. <br>
> *정지되었던 컨슈머가 다시 돌아오면 이 파티션에 저장된 최신 메 시지에서 한참 뒤에 있는 밀린 메시지부터 처리하게 된다.*


<br>
<hr>

# 카프카 컨슈머 생성하기

카프카 레코드를 읽어오기 위한 첫 번째 단계는 KafkaConsumer 인스턴스를 생성하는 것이다. <br>

```java
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9092,broker2:9092");
props.put("group.id", "CountryCounter"); 
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka•common.serialization.StringDeserializer");
KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(props);
```

<br>
<hr>

# 토픽 구독하기

컨슈머를 생성하고 나서 다음으로 할 일은 1개 이상의 토픽을 구독하는 것이다. <br>

```java
consumer.subscribe(Collections.singletonList("customerCountries"));
```

<br>

정규식을 매개변수로 사용해서 subscribe를 호출하는 것 역시 가능하다. <br>
정규식은 다수의 토픽 이름에 매치될 수도 있으며, 만약 누군가가 정규식과 매치되는 이름을 가진 새로운 토픽을 생성할 경우, 거의 즉시 리밸런스가 발생하면서 컨슈머들은 새로운 토픽으로부터의 읽기 작업을 시작하게 된다. <br>
*(이것은 다수의 토픽에서 레코드를 읽어와서 토픽이 포함하는 서로 다른 유형의 데이터를 처리해야 하는 애플리케이션의 경우 편리하다.)*

```java
consumer.subscribe(Pattern.compile("test.*"));
```

<br>
<hr>

# 폴링 루프

컨슈머 API의 핵심은 서버에 추가 데이터가 들어왔는지 폴링하는 단순한 루프다. <br>
컨슈머 애플리케이션의 주요 코드는 대략 다음과 같다.

```java
Duration timeout = Duration.ofMillis(100);

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(timeout);

    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("topic = %s, partition = %d, offset = %d, " +
                        "customer = %s, country = %s\n",
        record.topic()), record.partition(), record.offset(),
                record.key(), record.value());
        int updatedCount = 1;
        if (custCountryMap.containsKey(record.value())) {
            updatedCount = custCountryMap.get(record.value()) + 1;
        }
        custCountryMap.put(record.value(), updatedCount);

        JSONObject json = new JSONObject(custCountryMap);
        System.out.println(json.toString());
    }
}
```

> 이 루프는 무한 루프이기 때문에 종료되지 않않는다. 컨슈머 애플리케이션은 보통 계속해서 카프카에 추가 데이터를 폴링하는, 오랫동안 돌아가는 애플리케이션이다.

> 컨슈머는 카프카를 계속해서 폴링하지 않으면 죽은 것으로 간주되어 이 컨슈머가 읽어오고 있던 파티션들은 그룹 내의 다른 컨슈머에게 넘겨진다. <br>
> 우리가 poll()에 전달하는 매개변수는 컨슈머 버퍼에 데이터가 없을 경우 poll()이 블록될 수 있는 최대 시간을 결정한다. <br>
> 만약 이 값이 0으로 지정되거나 버퍼 안에 이미 레코드가 준비되어 있을 경우 poll()은 즉시 리턴된다.

> poll()은 레코드들이 저장된 List 객체를 리턴한다. <br>
> 각각의 레코드는 레코드가 저장되어 있던 토픽, 파티션, 파티션에서의 오프셋 그리고 키값과 밸류값을 포함한다.

> 처리가 끝날 때는 결과물을 데이터 저장소에 쓰거나 이미 저장된 레코드를 갱신한다.


새 컨슈머에서 처음으로 poll()을 호출하면 컨슈머는 GroupCoordinator를 찾아서 컨슈머 그룹에 참가하고, 파티션을 할당받는다. <br>
리밸런스 역시 연관된 콜백들과 함께 여기서 처리된다. <br>
**즉, 컨슈머 혹은 콜백에서 뭔가 잘못 될 수 있는 거의 모든 것들은 poll()에서 예외의 형태로 발생되는 것이다.**

## 스레드 안정성

하나의 스레드에서 동일한 그룹 내에 여러 개의 컨슈머를 생성할 수는 없으며, 같은 컨슈머를 다수의 스레드가 안전하게 사용할 수도 없다. ***(하나의 스레드당 하나의 컨슈머가 원칙)*** <br>
하나의 애플리케이션에서 동일한 그룹에 속하는 여러 개의 컨슈머를 운용하고 싶다면 스레드를 여러 개 띄워서 각각에 컨슈머를 하나씩 돌리는 수밖에 없다. <br>
**컨슈머 로직을 자체적인 객체로 감싼 다음 자바의 ExecutorService를 사용해서 각자의 컨슈머를 가지는 다수의 스레드를 시작시키면 좋다.** <br>
*참고) (https://www.confluent.io/blog/tutorial-getting-started-with-the-new-apache-kafka-0-9-consumer-client/)*

**또 다른 방법은 이벤트를 받아서 큐에 넣는 컨슈머 하나와 이 큐에서 이벤트를 꺼내서 처리하는 여러 개의 워커 스레드를 사용하는 것이다.** <br>
*참고) (https://www.confluent.io/blog/kafka-consumer-multi-threaded-messaging/)*

<br>
<hr>

# 컨슈머 설정하기

대부분의 매개변수는 합리적인 기본값을 가지고 있기 때 문에 딱히 변경할 필요는 없다. <br>
하지만 몇몇 매개변수는 컨슈머의 성능과 가용성에 영향을 준다.

## fetch.min.bytes

**이 속성은 컨슈머가 브로커로부터 레코드를 얻어올 때 받는 데이터의 최소량(바이트)를 지정할 수 있게 해준다.** <br>
만약 브로커가 컨슈머로부터 레코드 요청을 받았는데 새로 보낼 레코의 양이 fetch.min.bytes보다 작으면, 브로커는 충분한 메시지를 보낼 수 있을 때까지 기다린 뒤 컨슈머에게 레코드를 보내준다. <br>
이것은 토픽에 새로운 메시지가 많이 들어오지 않거나 하루 중 쓰기 요청이 적은 시간대일 때와 같은 상황에서 오가는 메시지 수를 줄임으로써 컨슈머와 브로커 양쪽에 대해 부하를 줄여주는 효과가 있다.

## fetch.max.wait.ms

**카프카가 컨슈머에게 응답하기 전 충분한 데이터가 모일 때까지 기다리도록 할 수 있다.** *(기본적으로는 500밀리초)* <br>
카프카는 토픽에 컨슈머에게 리턴할 데이터가 부족할 경우 리턴할 데이터 최소량 조건을 맞추기 위해 fetch.max.wait.ms만큼 기다린다

## fetch.max.bytes

**컨슈머가 브로커를 폴링할 때 카프카가 리턴하는 최대 바이트 수를 지정한다.** <br>
이것은 컨슈머가 서버로부터 받은 데이터를 저장하기 위해 사용하는 메모리의 양을 제한하기 위해 사용된다. 

## max.poll.records

**poll()을 호출할 때마다 리턴되는 최대 레코드 수를 지정한다.** <br>
애플리케이션이 폴링 루프를 반복할 때마다 처리해야 하는 레코드의 개수를 제어할 때 사용한다.

## max.partition.fetch.bytes

**서버가 파티션별로 리턴하는 최대 바이트 수를 결정한다.** *(기본값은 1MB)* <br>
KafkaConsumer.poll()가 ConsumerRecords를 리턴할 때, 메모리 상에 저장된 레코드 객체의 크기는 컨슈머에 할당된 파티션별로 최대 max.partition.fetch.bytes까지 차지할 수 있다.

## session.timeout.ms 그리고 heartbeat.interval.ms

컨슈머가 브로커와 신호를 주고받지 않고도 살아 있는 것으로 판정되는 최대 시간의 기본값은 10초이다.

만약 컨슈머가 그룹 코디네이터에게 하트비트를 보내지 않은 채로 session.timeout.ms가 지나가면 그룹 코디네이터는 해당 컨슈머를 죽은 것으로 간주하고 <br>
죽은 컨슈머에게 할당되어 있던 파티션들을 다른 컨슈머에게 할당해주기 위해 리밸런스를 실행시킨다. <br>
이 속성은 카프카 컨슈머가 얼마나 자주 그룹 코디네이터에게 하트비트를 보내는지를 결정하는 heartbeat.interval.ms 속성과 밀접하게 연관되어 있다. <br>
session.timeout.ms가 컨슈머가 하트비트를 보내지 않을 수 있는 최대 시간을 결정하기 때문이다.

## max.poll.interval.ms

**이 속성은 컨슈머가 폴링을 하지 않고도 죽은 것으로 판정되지 않을 수 있는 최대 시간을 지정할 수 있게 해준다.** <br>

하트비트는 백그라운드 스레드에 의해 전송되는데, 카프카에서 레코드를 읽어오는 메인 스레드는 데드락이 걸렸는데 백그라운드 스레드는 멀쩡히 하트비트를 전송하고 있을 수도 있다. <br>
이는 이 컨슈머에 할당된 파티션의 레코드들이 처리되고 있지 않음을 의미한다.

## default.api.timeout.ms

**이것은 API를 호출할 때 명시적인 타임아웃을 지정하지 않는 한. 거의 모든 컨슈머 API 호출에 적용되는 타임아웃 값이다.** *(기본값은 1분)*

## request.timeout.ms

**컨슈머가 브로커로부터의 응답을 기다릴 수 있는 최대 시간이다.** *(기본값은 30초)* <br>
만약 브로커가 이 설정에 지정된 시간 사이에 응답하지 않을 경우, 클라이언트는 브로커가 완전히 응답하지 않을 것이라고 간주하고 연결을 닫은 뒤 재연결을 시도한다.

## auto.offset.reset

**컨슈머가 예전에 오프셋을 커밋한 적이 없거나. 커밋된 오프셋이 유효하지 않을 때 파티션을 읽기 시작 할 때의 작동을 정의한다.** (대개 컨슈머가 오랫동안 읽은 적이 없어서 오프셋의 레코드가 이미 브로커에서 삭제된 경우) <br>
기본값은 **latest**인데 만약 유효한 오프셋이 없을 경우 컨슈머는 가장 최신 레코드부터 읽기 시작한다.

## enable.auto.commit

**컨슈머가 자동으로 오프셋을 거밋할지의 여부를 결정한다.** <br>
언제 오프셋 을 커밋할지를 직접 결정하고 싶다면 이 값을 false로 놓으면 된다. *(중복을 최소화하고 유실되는 데이터를 방지하려면 필요하다)*

## partition.assignment.strategy

**어느 컨슈머에게 어느 파티션이 할당될지를 결정하는 역할을 한다.** <br>
종류로는 Range, RoundRobin, Sticky, Cooperative Sticky가 있다.

<br>
<hr>

# 오프셋과 커밋

우리가 poll()을 호출할 때마다 카프카에 쓰여진 메시지 중에서 컨슈머 그룹에 속한 컨슈머들이 아직 읽지 않은 레코드가 리턴된다. <br>
뒤집어 말하면, 이를 이용해서 그룹 내의 컨슈머가 어떤 레코드를 읽었는지를 판단할 수 있다는 얘기다.

카프카에서는 파티션에서의 현재 위치를 업데이트하는 작업을 **오프셋 커밋**이라고 한다. <br>
카프카는 레코드를 개별적으로 커밋하지 않고, 파티션에서 성공적으로 처리해 낸 마지막 메시지를 커밋함으로써 그 앞의 모든 메시지들 역시 성공적으로 처리되었음을 암묵적으로 나타낸다.

> 컨슈머는 오프셋을 커밋할 때, 카프카에 특수 토픽인 __consumer_offsets 토픽에 각 파티션별로 커밋된 오프셋을 업데이트하도록 하는 메시지를 보냄으로써 이루어진다.

## 자동 커밋

오프셋을 커밋하는 가장 쉬운 방법은 컨슈머가 대신하도록 하는 것이다. <br>
enable.auto.commit 설정을 true로 잡아주면 컨슈머는 5초에 한 번, poll()을 통해 받은 메시지 중 마지막 메시지의 오프셋을 커밋한다. <br>
5초 간격은 기본값으로, auto.commit.interval.ms 설정을 잡아줌으로써 바꿀 수 있다.

> 기본적으로 자동 커밋은 5초에 한 번 발생한다. 마지막으로 커밋한 지 3초 뒤에 컨슈머가 크래시되었다고 해보자. <br>
> 리밸런싱이 완료된 뒤부터 남은 컨슈머들은 크래시된 컨슈머가 읽고 있던 파티션들을 이어받아서 읽기 시작한다. <br>
> 문제는 남은 컨슈머들이 마지막으로 커밋된 오프셋부터 작업을 시작할 때, 커밋되어 있는 오프셋은 3초 전의 것이기 때문에 크래시되기 3초 전까지 읽혔던 이벤트들은 두 번 처리되게 된다.
> **오프셋을 더 자주 커밋하여 레코드가 중복될 수 있는 윈도우를 줄어들도록 커밋 간격을 줄여서 설정해 줄 수도 있지만, 중복을 완전히 없애는 것은 불가능하다.**

**자동 커밋은 편리하다. 그러나 개발자가 중복 메시지를 방지하기엔 충분하지 않다.**

## 현재 오프셋 커밋하기

대부분의 개발자들은 오프셋이 거밋되는 시각을 제어하고자 한다. <br>
컨슈머 API는 타이머 시간이 아닌, 애플리케이션 개발자가 원하는 시간에 현재 오프셋을 커밋하는 옵션을 제공한다.

> **enable.auto.commit=false**로 설정해 줌으로써 애플리케이션이 명시적으로 거밋하려 할 때만 오프셋이 커밋되게 할 수 있다. <br>
> 가장 간단하고 또 신뢰성 있는 커밋 API는 commitSync()이다. <br>
> 이 API는 poll()이 리턴한 마지막 오프셋을 커밋한 뒤 커밋이 성공적으로 완료되면 리턴, 어떠한 이유로 실패 하면 예외를 발생시킨다.

> commitSync()는 poll()에 의해 리턴된 마지막 오프셋을 커밋하는데, 만약 poll()에서 리턴된 모든 레코드의 처리가 완료되기 전 commitSync()를 호출하면 <br>
> 애플리케이션이 크래시되었을 때 커밋은 되었지만 아직 처리되지 않은 메시지들이 누락될 위험을 감수해야 할 것이다. <br>
> 만약 애플리케이션이 아직 레코드들을 처리하는 와중에 크래시가 날 경우, 마지막 메시지 배치의 맨 앞 레코드에서부터 리밸런스 시작 시점까지의 모든 레코드들은 두 번 처리될 것이다. <br>
> *이것이 메시지 유실보다 더 나을지 아닐지는 상황에 따라 다르다*

아래는 가장 최근의 메시지 배치를 처리한 뒤 commitSync()를 호출해서 오프셋을 커밋하는 예이다.

```java
Duration timeout = Duration.ofMillis(100);

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(timeout);

    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("topic = %s, partition = %d, offset = %d, " +
                        "customer = %s, country = %s\n",
        record.topic()), record.partition(), record.offset(),
                record.key(), record.value());
    }
    try {
        consumer.commitSync();
    } catch (CommitFailedException e) {
        log.error("commit failed", e)
    }
}
```

현재 배치의 모든 레코드에 대한 처리가 완료되면 추가 메시지를 폴링하기 전에 commitSync를 호출해서 해당 배치의 마지막 오프셋을 커밋한다.

## 비동기적 커밋

수동 커밋의 단점 중 하나는 브로커가 커밋 요청에 응답할 때까지 애플리케이션이 블록된다는 점이다. *(처리량 하락)* <br>
비동기적 커밋 api를 사용하면 브로커가 거밋에 응답할 때까지 기다리는 대신 요청만 보내고 처리를 계속한다.

```java
Duration timeout = Duration.ofMillis(100);

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(timeout);

    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("topic = %s, partition = %d, offset = %d, " +
                        "customer = %s, country = %s\n",
        record.topic()), record.partition(), record.offset(),
                record.key(), record.value());
    }
    consumer.commitAsync();
}
```

> 이 방식의 단점은 commitSync()가 성공하거나 재시도 불가능한 실패가 발생할 때까지 재시도하는 반면, commitAsync()는 재시도를 하지 않는다는 점이다.<br>
> 왜냐하면 commitAsync()가 서버로부터 응답을 받은 시점에는 이미 다른 커밋 시도가 성공했을 수도 있기 때문이다.

commitAsync()에는 브로커가 보낸 응답을 받았을 때 호출되는 콜백을 지정할 수 있는 옵션이 있는데, <br>
이 콜백은 커밋 에러를 로깅하거나 커밋 에러 수를 지표 형태로 집계하기 위해 사용되는 것이 보통이지만 재시도를 하기 위해 콜백을 사용하고자 할 경우 커밋 순서 관련된 문제에 주의를 기울이는 것이 좋다.

```java
Duration timeout = Duration.ofMillis(100);

while (true) {
        ConsumerRecords<String, String> records = consumer.poll(timeout);

        for (ConsumerRecord<String, String> record : records) {
            System.out.printf("topic = %s, partition = %d, offset = %d, " +
            "customer = %s, country = %s\n",
            record.topic()), record.partition(), record.offset(),
            record.key(), record.value());
        }
        consumer.commitAsync() {
            public void onComplete(Map<TopicPartition, OffsetAndMetadata> offsets, Exception e) {
                if (e != null)
                log.error("Commit failed for offsets {}", offsets, e);
            }
        }
}
```

## 동기적 커밋과 비동기적 커밋을 함께 사용하기

대체로 재시도 없는 커밋이 이따금 실패한다고 해서 큰 문제가 되지는 않는다. 일시적인 문제일 경우 뒤이은 커밋이 성공할 것이기 때문이다. <br>
하지만 이것이 컨슈머를 닫기 전 혹은 리밸런스 전 마지막 커밋이라면, 성공 여부를 추가로 확인할 필요가 있을 것이다.

이런 경우에는 commitAsync()와 commitSync()를 함께 사용할 수 있다.

```java
Duration timeout = Duration.ofMillis(100);

try {
    while (!closing) {
        ConsumerRecords<String, String> records = consumer.poll(timeout);

        for (ConsumerRecord<String, String> record : records) {
            System.out.printf("topic = %s, partition = %d, offset = %d, " +
            "customer = %s, country = %s\n",
            record.topic()), record.partition(), record.offset(),
            record.key(), record.value());
        }
        consumer.commitAsync()
    }
    consumer.commitSync();
} catch (Exception e) {
    log.error("Unexpected error", e);
} finally {
    consumer.close();
}
```

1. 정상적인 상황에서는 commitAsync를 사용한다. 더 빠를 뿐더러 설령 커밋이 실패하더라도 다음 커밋이 재시도 기능을 하게 된다.
2. 하지만 컨슈머를 닫는 상황에서는 다음 커밋이라는 것이 있을 수 없으므로 comitSync()를 호출한다. 커밋이 성공하거나 회복 불가능한 에러가 발생할 때까지 재시도할 것이다.

## 특정 오프셋 커밋하기

가장 최근 오프셋을 커밋하는 것은 메시지 배치의 처리가 끝날 때만 수행이 가능하다. <br>
만약 더 자주 커밋하고 싶다면 컨슈머 API에 커밋하고자 하는 파티션과 오프셋의 맵을 전달할 수 있다.

```java
private Map<TopicPartition, OffsetAndMetadata> currentoffsets = new HashMap<>();
int count = 0;

Duration timeout = Duration.ofMillis(100);

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(timeout);

    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("topic = %s, partition = %d, offset = %d, " +
        "customer = %s, country = %s\n",
        record.topic()), record.partition(), record.offset(),
        record.key(), record.value());
        currentoffsets.put(
            new TopicPartition(record.topic(), record.partition()),
            new OffsetAndMetadata(record.offset()+1, "no metadata"));
        if(count % 1000==0)
            consumer.commitAsync(currentOffsets, null);
        count++;
    }
```

1. 각 레코드를 읽은 뒤, 맵을 다음 번에 처리할 것으로 예상되는 메시지의 오프셋으로 업데이트한다.
2. 여기서는 1000개의 레코드마다 현재 오프셋을 커밋한다. 실제 애플리케이션에는 시간 혹은 레코드의 내용물을 기준으로 커밋해야 할 것이다.

<br>
<hr>

# 리밸런스 리스너

컨슈머는 종료하기 전이나 리밸런싱이 시작되 기 전에 정리 작업을 해줘야 한다. <br>
만약 컨슈머에 할당된 파티션이 해제될 것이라는 걸 알게 된다면 해당 파티션에서 마지막으로 처리한 이벤트의 오프셋을 커밋해야 할 것이다.

컨슈머 API는 컨슈머에 파티션이 할당되거나 해제될 때 사용자의 코드가 실행되도록 하는 메커니즘을 제공한다. <br>
subscribe()를 호출할 때 ConsumerRebalanceListener를 전달해주면 된다.

### public void onPartitionsAssigned(Collection<TopicPartition> partitions)

파티션이 컨슈머에게 재할당된 후에, 컨슈머가 메시지를 읽기 시작하기 전에 호출된다. <br>
파티션과 함께 사용할 상태를 적재하거나, 필요한 오프셋을 탐색하거나 등과 같은 준비 작업을 수행 하는 곳다.

### public void onPartitionsRevoked(Collection<TopicPartition> partitions)

컨슈머가 할당받았던 파티션이 할당 해제될 때 호출된다. <br>
이 메서드는 컨슈머가 메시지 읽기를 멈춘 뒤에, 그리고 리밸런스가 시작되기 전에 호출된다. <br>
여기서 오프셋을 커밋해주어야 이 파티션을 다음에 할당받는 컨슈머가 시작할 지점을 알아낼 수 있다.

### public void onPartitionsLost(Collection<TopicPartition> partitions)

협력적 리밸런스 알고리즘이 사용되었을 경우, 할당된 파티션이 리밸런스 알고리즘에 의해 해제되기 전에 다른 컨슈머에 먼저 할당된 예외적인 상황에서만 호출된다. <br>
여기서는 파티션과 함께 사용되었던 상태나 자원들을 정리해주어야 한다. <br>
이러한 작업을 수행할 때는 주의할 필요가 있는데, 파티션을 새로 할당 받은 컨슈머가 이미 상태를 저장했을 수도 있기 때문에 충돌을 피해야 할 것이다.

<br>
<hr>

# standalone consumer: 컨슈머 그룹 없이 컨슈머를 사용해야 하는 이유와 방법

컨슈머 그룹은 컨슈머들에게 파티션을 자동으로 할당해주고 해당 그룹에 컨슈머가 추가되거나 제거될 경우 자동으로 리밸런싱을 해준다. <br>
대개는 이것이 우리가 원하는 것이지만, 경우에 따라서는 훨씬 더 단순한 것이 필요할 수도 있다. <br>
**하나의 컨슈머가 토픽의 모든 파티션으로부터 모든 데이터를 읽어와야 하거나, 토픽의 특정 파티션으로부터 데이터를 읽어와야 할 때가 있다.** <br>
이러한 경우 컨슈머 그룹이나 리밸런스 기능이 필요하지는 않다. 그냥 컨슈머에게 특정한 토픽과 파티션을 할당해주고, 메시지를 읽어서 처리하고, 필요할 경우 오프셋을 커밋하면 되는 것이다.

만약 컨슈머가 어떤 파티션을 읽어야 하는지 정확히 알고 있을 경우 토픽을 구독할 필요 없이 그냥 파티션을 스스로 할당받으면 된다. <br>
**컨슈머는 토픽을 구독하거나 스스로 파티션을 할당할 수 있지만, 두 가지를 동시에 할 수는 없다.**

아래는 컨슈머 스스로가 특정 토픽의 모든 파티션을 할당한 뒤 메시지를 읽고 처리하는 방법을 보여준다.

```java
Duration timeout = Duration.ofMillis(100);
List<PartitionInfo> partitionInfos = consumer.partitionsFor("topic");

if (partitioninfos != null) {
    for (Partitioninfo partition : partitioninfos) {
        partitions.add(
            new TopicPartition(partition.topic(), partition.partition())
        ); 
    }
    
consumer.assign(partitions);
    
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(timeout);

    for (ConsumerRecord<String, String> record: records) {
        System.out.printf("topic = %s, partition = %d, offset = %d, " +
        "customer = %s, country = %s\n",
        record.topic()), record.partition(), record.offset(),
        record.key(), record.value());
    }
    consumer.commitSync();
}
```

1. 카프카 클러스터에 해당 토픽에 대해 사용 가능한 파티션들을 요청하면서 시작한다. 만약 특정 파티션의 레코드만 읽어 올 생각이라면 이 부분은 건너뛸 수 있다.
2. 읽고자 하는 파티션을 알았다면 해당 목록에 대해 assignO을 호출한다.

> 만약 누군가가 토픽에 새로운 파티션을 추가할 경우 컨슈머에게 알림이 오지는 않는다는 걸 명심하자. <br>
> 주기적으로 consumer.partitionsFor()를 호출해서 파티션 정보를 확인하거나 파티션이 추가될 때마다 애플리케이션을 재시작함으로써 이러한 상황에 대처해 줄 필요가 있을 것이다.






