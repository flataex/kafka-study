# 프로듀서 개요

애플리케이션이 카프카에 메시지를 써야 하는 상황에는 여러 가지가 있을 수 있다. <br>
다른 애플리케이션과의 비동기적 통신 수행, 임의의 정보를 데이터베이스에 저장하기 전 버퍼링, 로그메시지 저장 등..

이러한 사용 사례들은 목적이 다양한 만큼 요구 조건 역시 다양하다. <br>
메시지 유실이 용납되지 않는지, 아니면 유실이 허용되는지, 중복이 허용되도 상관없는지, 반드시 지켜야 할 지연나 처리율이 있는지..

> 예를 들어, 신용카드 결제 시스템은 메시지에 어떠한 유실이나 중복 도 허용되지 않는다. <br>
> 지연 시간은 낮아야 하지만 500밀리초 정도까지는 허용될 수 있으며, 처리율은 매우 높아야 한다. <br>
> 다른 예로는, 웹사이트에서 생성되는 클릭 정보를 저장하는 경우 메시지가 조금 유실되거나 중복되는 것은 문제가 되지 않는다. <br>
> 사용자 경험에 영향을 주지 않는 한, 지연 역시 높아도 상관없다. 처리율 역시 웹사이트에서 예상되는 수준의 사용자 행동에 따라 달라질 수 있다.


<br>

<img width="583" alt="스크린샷 2023-09-22 오후 4 30 34" src="https://github.com/flataex/kafka-study/assets/87420630/806c4fa2-0bfb-44f1-948e-a1ebbcf50de1">

카프카에 메시지를 쓰는 작업은 ProducerRecord 객체를 생성함으로써 시작된다. *(레코드가 저장될 토픽과 밸류 지정은 필수사항, 키와 파티션 지정은 선택사항이다)* <br>
ProducerRecord를 전송하는 API를 호출했을 때 프로듀서가 가장 먼저 하는 일은 키와 값 객체가 네트워크 상에서 전송될 수 있도록 직렬화해서 바이트 배열로 변환한다. <br>
그 이후 만약 파티션을 명시적으로 지정하지 않았다면 해당 데이터를 파티셔너에게로 보낸다. *(파티셔너는 파티션을 결정하는 역할을 하는데, 그 기준은 보통 ProduderRecord 객체의 키의 값이다.)* <br>
파티션이 결정되어 메시지가 전송될 토픽과 파티션이 확정되면 프로듀서는 이 레코드를 같은 토픽 파티션으로 전송될 레코드들을 모은 레코드 배치에 추가한다. <br>

> 브로커가 메시지를 받으면 응답을 돌려주게 되어 있다. <br>
> 메시지가 성공적으로 저장되었을 경우 브로커는 토픽, 파티션, 그리고 해당 파티션 안에서의 레코드의 오프셋을 담은 RecordMetadata 객체를 리턴한다. <br>
> 메시지가 저장에 실패했을 경우에는 에러가 리턴된다. <br>
> 프로듀서가 에러를 수신했을 경우, 메시지 쓰기를 포기하고 사용자에게 에러를 리턴하기 전까지 몇 번 더 재전송을 시도할 수 있다.

<br>
<hr>

# 카프카로 메시지 전달하기

```java
ProducerRecord<String, String> record = new ProducerRecord<>("CustomerCountry", "Precision Products", "France");

try {
    producer.send(record);  
} catch (Exception e) {
    e.printStackTrace();
}
```

1. 프로듀서는 ProducerRecord 객체를 받으므로 이 객체를 생성하는 것에서부터 시작한다.
2. ProducerRecord를 전송하기 위해 프로듀서 객체의 send 메서드를 사용한다. send() 메서드는 RecordMetadata를 포함한 자바 Future 객체를 리턴하지만, 여기서는 리턴값을 무시하기 때문에 메시지 전송의 성공 여 부를 알아낼 방법은 없다. 이러한 방법은 메시지가 조용히 누락되어도 상관없는 경우 사용될 수 있다.


## 동기적으로 메시지 전송하기

카프카 브로커가 쓰기 요청에 에러 응답을 내놓거나 재전송 횟수가 소진되었을 때 발생되는 예외를 받아서 처리할 수 있다. <br>
동기적으로 메시지를 전송할 경우 전송을 요청하는 스레드는 이 시간 동안 아무것도 안하면서 기다려야 한다. *(성능이 크게 낮아지기 때문에 실제로 사용되는 애플리케이션에서는 잘 사용되지 않는다.)*

## 비동기적으로 메시지 전송하기

실제로 대부분의 경우 굳이 응답이 필요 없다. <br>
카프카는 레코드를 쓴 뒤 해당 레코드의 토픽, 파티션 그리고 오프셋을 리턴하는데, 대부분의 애플리케이션에서는 이런 메타데이터가 필요 없기 때문이다. <br>
하지만, 메시지 전송에 완전히 실패했을 경우에는 그런 내용을 알아야 한다.

**메시지를 비동기적으로 전송하고도 여전히 에러를 처리하는 경우를 위해 프로듀서는 레코드를 전송 할 때 콜백을 지정할 수 있도록 한다.**

```java
private class DemoProducerCallback implements Callback { 
    @Override
    public void onCompletion(RecordMetadata recordMetadata, Exception e) { 
        if (e != null) {
            e.printStackTrace(); 
        }
    } 
}

ProducerRecord<String, String> record = new ProducerRecord<>("CustomerCountry", "Biomedical. Materials", "USA");
producer.send(record, new DemoProducerCallback());
```

1. 콜백을 사용하려면 org.apache.kafka.clients.producer.Callback 인터페이스를 구현하는 클래스가 필요하다.
2. 만약 카프카가 에러를 리턴한다면 onCompletionO 메서드가 null이 아닌 Exception 객체를 받 게 된다.

<br>
<hr>


# 프로듀서 설정하기

## client.id

프로듀서와 그것을 사용하는 애플리케이션을 구분하기 위한 논리적 식별자. <br>
브로커는 프로듀서가 보내온 메시지를 서로 구분하기 위해 이 값을 사용한다. *(이 값을 잘 선택하는 것은 문제가 발생했을 때 트러블슈팅을 쉽게 한다.)*

## acks

acks 매개변수는 프로듀서가 임의의 쓰기 작업이 성공했다고 판별하기 위해 얼마나 많은 파티션 레플리카가 해당 레코드를 받아야 하는지를 결정한다. <br>
이 매개변수는 메시지가 유실될 가능성에 큰 영향을 미친다.

- **acks = 0**: 프로듀서는 메시지가 성공적으로 전달되었다고 간주하고 브로커의 응답을 기다리지 않는다. 프로듀서가 서버로부터 응답을 기다리지 않는 만큼 네트워크가 허용하는 한 빠르게 메시지를 보낼 수 있다.
- **acks = 1**: 프로듀서는 리더 레플리카가 메시지를 받는 순간 브로커로부터 성공했다는 응답을 받는다. 만약 리더에 메시지를 쓸 수 없다면 프로듀서는 에러 응답을 받을 것이고 데이터 유실을 피하기 위해 메시지 재전송을 시도한다.
- **acks = all**: 프로듀서는 메시지가 모든 인-싱크 레플리카에 전달되고 나서 브로커로부터 성공했다는 응답을 받는다. 단순히 브로커 하나가 메시지를 받는 것보다 더 기다려야 하기 때문에 지연 시간은 더 길어질 것이다.


<br>
<hr>

# 파티션

ProduceRecord 객체는 토픽, 키, 밸류의 값을 포함한다. <br>
키의 역할은 두 가지로, 그 자체로 메시지에 함께 저장되는 추가적인 정보이고, 하나의 토픽에 속한 여러 개의 파티션 중 해당 메시지가 저장될 파티션을 결정짓는 기준점이기도 하다. <br>
같은 키값을 가진 모든 메시지는 같은 파티션에 저장되는 것이다. 

```java
// key-value 레코드 생성 예제
ProducerRecord<String, String> record = new ProducerRecord<>("CustomerCountry", "Laboratory Equipment", "USA");

// key 없는 예
ProducerRecord<String, String> record = new ProducerRecord<>("CustomerCountry", "USA");
```

> key 값이 null이면 레코드는 현재 사용 가능한 토픽의 파티션 중 하나에 랜덤하게 저장된다. *(sticky 처리를 위해 라운드 로빈 알고리즘 사용)* <br>

<img width="729" alt="스크린샷 2023-09-22 오후 5 16 43" src="https://github.com/flataex/kafka-study/assets/87420630/34a7dfb2-3caa-4c62-bddb-e3ca0613dd20">

<br>
<hr>


# 인터셉터

카프카 클라이언트의 코드를 고치지 않으면서 그 작동을 변경해야 하는 경우가 있다. <br>
이럴 때 사용하는 것이 카프카의 ProducerInterceptor 인터셉터다.

**ProducerRecord<K, V> onSend(ProducerRecord<K, V> record)** <br>
이 메서드는 프로듀서가 레코드를 브로커로 보내기 전, 직렬화되기 직전에 호출된다. <br>
이 메서드를 재정의할 때는 보내질 레코드에 담긴 정보를 볼 수 있을 뿐만 아니라 고칠 수도 있다. <br>
이 메서드가 리턴한 레코드가 직 렬화되어 카프카로 보내질 것이다.


**void onAcknowledgement(RecordMetadata metadata, Exception exception)** <br>
이 메서드는 카프카 브로커가 보낸 응답을 클라이언트가 받았을 때 호출된다. 브로커가 보낸 응답 을 변경할 수는 없지만, 그 안에 담긴 정보는 읽을 수 있다.

<br>
<hr>


