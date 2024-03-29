# 7. 신뢰성 있는 데이터 전달

# 7.1 신뢰성 보장

### 카프카가 보장하는 것들

- 카프카는 파티션 안의 메시지들 간에 순서를 보장한다.
- 클라이언트가 쓴 메시지는 모든 인-싱크 레플리카의 파티션에 쓰여진 뒤에야 ‘커밋’된 것으로 간주된다.
- 커밋된 메시지들은 최소 1개의 작동 가능한 레프리카가 남아 있는 한 유실되지 않는다.
- 컨슈머는 커밋된 메시지만 읽을 수 있다.

---

# 7.2 복제

### 팔로워 레플리카 인-싱크 상태

- 주키퍼와의 활성 세션이 있다. 즉, 최근 6초 사이에 주키퍼로 하트비트를 전송했다.
- 최근 10초 사이 리더로부터 메시지를 읽어 왔다.
- 최근 10초 사이에 리더로부터 읽어 온 메시지들이 가장 최근 메시지이다.

⇒ 이 상태를 유지하지 않은 팔로워 레플리카는 아웃-오브-싱크 상태로 간주된다.

---

# 7.3 브로커 설정

## 7.3.1 복제 팩터

- replication.factor : 토픽 단위 설정
- default.replication.factor : 자동으로 생성되는 토픽들이 적용되는 브로커 단위 설정

### 트레이드오프

가용성과 하드웨어 사용량 사이에 트레이드 오프가 있다.

- 복제 팩터가 클수록 가용성과 신뢰성 ⬆️
- 복제 팩터가 클수록 N배의 디스크 공간 사용

### 핵심 고려 사항

토픽에 몇 개의 레플리카가 적절한지 결정하기 위해 몇 가지 핵심 고려 사항이 있다.

- **가용성**
    
    레플리카 수가 더 많을소록 가용성은 더 늘어난다.
    
- **지속성**
    
    복사본이 더 많을수록 모든 데이터가 유실될 가능성은 줄어든다.
    
- **처리량**
    
    레플리카가 추가될 때마다 브로커간 트래픽 역시 늘어난다.
    
- **종단 지연**
    
    레플리카 수가 많을수록 이들 중 하나가 느려짐으로써 컨슈머까지 함께 느려질 가능이 높아진다.
    
- **비용**
    
    더 많은 레플리카를 가질수록 저장소와 네트워크에 들어가는 비용 역시 증가한다.
    

랙 단위 사고를 방지하기 위해 브로커들을 서로 다른 랙에 배치한 뒤 broker.rack 브로커 설정 매개변수에 랙 이름을 잡아줄 것을 권장한다. ( 클라우드의 경우, 가용 영역 설정 )

## 7.3.2 언클린 리더 선출

- unclean.leader.election.enable ( default: false )

파티션의 리더가 더 이상 사용 가능하지 않을 경우 인-싱크 레플리카 중 하나가 새 리더가 된다.

### 인-싱크 레플리카가 없는 경우

아웃-오브-싱크 레플리카가 리더가 될 수 있도록 허용할 경우 데이터 유실과 일관성 깨짐의 위험성이 있다.

그렇지 않을 경우, 파티션이 다시 온라인 상태가 될 때가지 원래 리더가 복구되는 것을 기다려야 하는 만큼 가용성이 줄어든다.

⇒ unclean.leader.election.enable의 기본값이 false로 잡혀 있어, 아웃-오브-싱크-레플리카는 리더가 될 수 없다.

## 7.3.3 최소 인-싱크 레플리카

- min.insync.replicas

경우1 ) 토픽 레플리카 = 3 / min.insync.replicas = 3

레플리카 하나라도 작동 불능에 빠지면, 브로커는 더 이상 요청을 받을 수 없다.

컨슈머는 계속해서 존재하는 데이터를 읽을 수 있다.

## 7.3.4 레플리카를 인-싱크 상태로 유지하기

카프카는 카프카 클러스터의 민감도를 조절할 수 있는 브로커 설정을 두 개 가지고 있다.

- [zookeeper.session.timeout.ms](http://zookeeper.session.timeout.ms) : 브로커가 주키퍼로 하트비트 전송을 멈출 수 있는 최대 시간
- [replica.lag.time.max.ms](http://replica.lag.time.max.ms) : 리더로 부터 데이터를 읽지 못하거나 최신 메시지를 따라잡지 못하는 경우 동기화가 풀리는 시간
    - 컨슈머의 최대 지연에도 영향을 준다.

## 7.3.5 디스크에 저장하기

카프카는 메시지를 받은 레플리카 수에만 의존할 뿐, 디스크에 저장되지 않은 메시지에 대해서도 응답한다.

카프카는 세그먼트를 교체할 때 **재시작 직전**에만 메시지를 디스크에 플러시한다.

그 외의 경우에는 **리눅스의 페이지 캐시 기능**에 의존한다.

- 페이지 캐시 공간이 다 찼을 경우에만 메시지를 플러시한다.

flush.messages : 디스크에 저장되지 않은 최대 메시지 수

flush.ms : 얼마나 자주 디스크에 메시지를 저장하는지

---

# 7.4 신뢰성 있는 시스템에서 프로듀서 사용하기

- 신뢰성 요구 조건에 맞는 올바른 acks 설정을 사용한다.
- 설정과 코드 모두에서 에러를 올바르게 처리한다.

## 7.4.1 응답 보내기

### acks=0

- 성공 시점 : 프로듀서 메시지 전송 시점
- 지연은 낮지만, 위험성이 높다.

### acks=1

- 성공 시점 : 리더가 메시지를 받아서 파티션 데이터 파일에 쓴 직후
- 메시지를 팔로워가 복제하는 속도보다 더 빨리 리더에 쓸 수 있어 불완전 복제 파티션이 발생할 수 있다.

### acks=all

- 성공 시점 : 리더와 모든 인-싱크 레플리카가 메시지를 받아갈 때
- min.insync.replicas

## 7.4.2 프로듀서 재시도 설정하기

### 메시지 유실되지 않도록 설정하기

- 재시도 수를 기본 설정값(MAX_INT)로 설정
- 메시지 전송을 포기할 때까지 대기할 수 있는 시간을 지정하는 [delivery.timeout.ms](http://delivery.timeout.ms) 설정값을 최대로 설정

⇒ 재시도는 메시지 중복될 위험을 내포한다.

🔥 재시도와 주의 깊은 에러 처리는 각 메시지가 ‘최소 한 번’ 저장되도록 보장할 수는 있지만, ‘정확히 한 번’은 보장할 수 없다. enable.idempotence=true 설정을 잡아줌으로써 프로듀서가 추가적인 정보를 레코드에 포함할 수 있도록, 중복된 메시지를 건너뛸 수 있도록 할 수 있다.

---

# 7.5 신뢰성 있는 시스템에서 컨슈머 사용하기

파티션으로부터 데이터를 읽어 올 때, 컨슈머는 메시지를 배치 단위로 읽어온 뒤 배치별로 마지막 오프셋을 확인한 뒤, 브로커로부터 받은 마지막 오프셋 값에서 시작하는 다른 메시지 배치를 요청한다.

정지한 컨슈머가 마지막으로 읽은 오프셋을 저장하기 위해 **커밋**을 해야 한다.

## 7.5.1 신뢰성 있는 처리를 위해 중요한 컨슈머 설정

### [group.id](http://group.id)

- 컨슈머가 구독한 토픽의 모든 메시지를 읽어야 한다면 고유한 그룹 ID가 필요하다.

### auto.offset.reset

- 커밋된 오프셋이 없을 때 컨슈머가 브로커에 없는 오프셋을 요청할 때 어떻게 해야할지 결정한다.
- earliest : 유효한 오프셋이 없는 한 파티션의 맨 앞에서부터 읽기 시작
- latest : 파티션의 끝에서부터 읽기 시작

### enable.auto.commit

- 자동 오프셋 커밋 기능
    - 우리가 처리하지 않은 오프셋을 실수로 커밋하는 사태를 방지한다.
    - 메시지 중복 처리를 피할 수 없다.

### auto.commit.interval.ms

- 자동 오프셋 커밋 주기

---

# 7.6 시스템 신뢰성 검증하기

## 7.6.1 설정 검증하기

[org.apache.kafka.tools](http://org.apache.kafka.tools) 패키지에는 VerifiableProducer와 VerifiableConsumer 클래스가 포함되어 있다.

### 테스트 선정

- 리더 선출 : 재개 시간 측정
- 컨트롤러 선출 : 재개 시간 측정
- 롤링 재시작 : 메시지 유실 여부 판단
- 언클린 리더 선출 테스트

## 7.6.2 애플리케이션 검증하기

### 권장 테스트

- 클라이언트가 브로커 중 하나와 연결이 끊어짐
- 클라이언트와 브로커 사이의 긴 지연
- 디스크 꽉 참
- 디스크 멈춤
- 리더 선출
- 브로커 롤링 재시작
- 컨슈머 롤링 재시작
- 프로듀서 롤링 재시작

트록도르 테스트 프레임워크 제공한다.

### 7.6.3 프로덕션 환경에서 신뢰성 모니터링하기

카프카 자바 클라이언트들은 클라이언트 쪽 상태와 이벤트를 모니터링할 수 있게 해주는 JMX 지표를 포함한다.

### 프로듀서

- 레코드별 에러율, 재시도율
- 프로듀서의 ERROR 레벨 로그 메시지들
    - 재시도 불가능한 에러
    - 재시도 횟수가 고갈된 재시도 가능한 에러
    - 타임아웃 메시지 전송 실패

### 컨슈머

- 컨슈머 랙 (consumer lag)
    - 컨슈머가 브로커 내 파티션에 커밋된 가장 최산 메시지에서 얼마나 뒤떨어져 있는지
- 링크드인 버로우 툴 추천

### 타임스탬프

데이터의 흐름을 모니터링한다는 것은 곧 모든 쓰여진 데이터가 적절한 시기에 읽혀진다는 것을 의미한다.

- 언제 데이터가 생성되었는지를 알아야 한다.
- 모든 메시지는 이벤트가 생성된 시점을 가리키는 타임스탬프를 포함한다.

컨슈머는 단위 시간당 읽은 이벤트 수, 이벤트가 쓰여진 시점과 읽힌 시점 사이의 랙을 기록해야 한다.

카프카는 브로커가 클라이언트로 보내는 에러 응답률을 보여주는 지표를 포함한다.

- kafka.server:type=BrokerTopicMetrics,name=FailedProduceRequestsPerSec
- kafka.server:type=BrokerTopicMetrics,name=FailedFetchRequestsPerSec