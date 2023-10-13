# AdminClient

## 비동기적이고 최종적 일관성을 가지는 API

AdminClient는 비동기적으로 작동하고, 각 메서드는 요청을 클러스터 컨트롤러로 전송한 뒤 바로 1개 이상의 Future 객체를 리턴한다. <br>
Future 객체는 비동기 작업의 결과를 가리키며 비동기 작업의 결과를 확인하거나, 취소하거나, 완료될 때까지 대기하거나, 작업이 완료되었을 때 실행할 함수를 지정하는 메서드를 가지고 있다. <br>
카프카의 AdminClient는 Future 객체를 Result 객체 안에 감싸는데, Result 객체는 작업이 끝날때까지 대기하거나 작업 결과에 대해 일반적으로 뒤이어 쓰이는 작업을 수행하는 헬퍼 메서드를 가지고 있다. <br>
***ex) 카프카의 AdminClient.createTopics 메서드는 CreateTopicsResult 객체를 리턴하는데, 이 객체는 모든 토픽이 생성될 때까지 기다리거나, 각각의 토픽 상태를 하나씩 확인하거나, 아니면 특정한 토픽이 생성된 뒤 해당 토픽의 설정을 가져올 수 있도록 해준다.***

카프카 컨트롤러로부터 브로커로의 메타데이터 전파가 비동기적으로 이루어지기 때문에, AdminClient API가 리턴하는 Future 객체들은 컨트롤러의 상태가 완전히 업데이트된 시점에서 완료된 것으로 간주된다. <br>
이 시점에서 모든 브로커가 전부 다 새로운 상태에 대해 알고 있지는 못할 수 있기 때문에, listTopics 요청은 최신 상태를 전달받지 않은 브로커에 의해 처리될 수 있는 것이다. **(최종적 일관성)** <br>
**최종적으로 모든 브로커는 모든 토픽에 대해 알게 될 것이지만, 정확히 그게 언제가 될 지에 대해서는 아무런 보장도 할 수 없다.**

<br>
<hr>

#  AdminClient 사용법: 생성, 설정, 닫기

AdminClient를 사용하기 위해 가장 먼저 해야 할 일은 AdminClient 객체를 생성하는 것이다.

```java
Properties props = new Properties();
props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
AdminClient admin = AdminClient.create(props);
// TODO: AdminClient를 사용해서 필요한 작업을 수행한다.
//  admin.close(Duration.ofSeconds(30));
```

> 정적 메서드인 create 메서드는 설정값을 담고 있는 Properties 객체를 인수로 받는다. <br>
> 프로덕션 환경에서는 브로커 중 하나에 장애가 발생할 경우를 대비해서 최소한 3개 이상의 브로커를 지정하는 것이 보통이다.

> AdminClient를 시작했으면 결국엔 닫아야 한다. close를 호출할 때는 아직 진행중인 작업이 있을 수 있기 때문에, close 메서드 역시 타임아웃 매개변수를 받는다. <br>
> close를 호출하면 다른 메서드를 호출해서 요청을 보낼 수는 없지만, 클라이언트는 타임아웃이 만료 될 때까지 응답을 기다릴 것이다.

## client.dns.lookup

카프카는 부트스트랩 서버 설정에 포함된 호스트명을 기준으로 연결을 검증하고, 해석하고, 생성한다. <br>

### DNS 별칭을 사용하는 경우

broker1.hostname.com, broker2.hostname.com, ...와 같은 네이밍 컨벤션을 따르는 브로커들을 가지고 있다고 가정할 때, <br>
이 모든 브로커들을 부트스트랩 서버 설정에 일일이 지정하는 것보다 이 모든 브로커 전체를 가리킬 하나의 DNS 별칭을 만들 수 있다. <br>
이것은 편하지만, SASL을 사용할 경우 클라이언트는 all-brokers.hostname.com에 대해서 인증을 하려고 하는데, 서버의 보안 주체는 broker2.hostname.com이기 때문에 문제가 된다.

**이러한 경우 client.dns.lookup=resolve_canonical_bootstrap_servers_only 설정을 잡아 주면 된다.** <br>
이 설정이 되어 있을 경우 클라이언트는 DNS 별칭을 ‘펼치게’ 되기 때문에 DNS 별칭에 포함된 모든 브로커 이름을 일일이 부트스트랩 서버 목록에 넣어 준 것과 동일하게 작동하게 된다.


### 다수의 IP 주소로 연결되는 DNS 이름을 사용하는 경우

최근의 네트워크 아키텍처에서 모든 브로커를 프록시나 로드 밸런서 뒤로 숨기는 것은 매우 흔하다. <br>
외부로부터의 연결을 허용하기 위해서 로드 밸런서를 두어야 하는 쿠버네티스를 사용할 경우 특히나 그렇다. <br>
이 경우, 로드 밸런서가 단일 장애점이 되는 걸 보고 싶지는 않을 것이다. 이와 같은 이유 때 문에 broker1.hostname.com를 여러 개의 IP 주소로 연결하는 것은 매우 흔하다.

이 IP 주소들은 시간이 지남에 따라 변경될 수 있다. <br>
해석된 IP 주소가 사용 불능일 경우 브로커가 멀쩡하게 작동하고 있는데도 클라이언트는 연결에 실패할 수 있다는 얘기다. <br>
**클라이언트가 로드 밸런싱 계층의 고가용성을 충분히 활용할 수 있도록 client.dns.lookup=use_all_dns_ips를 사용하는 것이 권장된다.**


<br>
<hr>

# 필수적인 토픽 관리 기능

AdminClient의 가장 흔한 활용 사례는 토픽 관리다. (토픽 목록 조회, 상세 내역 조회, 생성 및 삭제)

```java
// 토픽 목록 조회
ListTopicsResult topics = admin.listTopics();
topics.names().get().forEach(System.out::println);
```

<br>

아래는 토픽이 존재하는지 확인하고, 없으면 만드는 코드이다. <br>
특정 토픽이 존재하는지를 확인하는 방법 중 하나는 모든 토픽의 목록을 받은 뒤 내가 원하는 토픽이 그 안에 있는지를 확인하는 것이다. 큰 클러스터에서 이것은 비효율적일 수 있다. <br>
때로는 단순한 토픽의 존재 여부 이상의 정보가 필요할 때가 있다. *(해당 토픽이 필요한 만큼의 파티션과 레플리카를 가지고 있는지 확인해야 하는 것과 같은 경우)* <br>

```java
// 토픽이 존재하는지 확인하고, 없으면 만든다.
DescribeTopicsResult demoTopic = admin.describeTopics(TOPIC_LIST);

try {
    topicDescription = demoTopic.values().get(TOPIC_NAME).get();
    
    if (topicDescription.partitions().size() != NUM_PARTITIONS) {
        System.out.println("Topic has wrong number of partitions. Exiting.");
        System.exit(-1);
    }
} catch (ExecutionException e) {
    if (! (e.getCause() instanceof UnknownTopicOrPartitionException)) {
        e.printStackTrace();
        throw e; 
    }

    // 여기까지 진행됐다면, 토픽은 존재하지 않는다.
    // 파티션 수와 레플리카 수는 선택사항
    CreateTopicsResult newTopic = admin.createTopics(Collections.singletonList(
        new NewTopic(TOPIC_NAME, NUM_PARTITIONS, REP.FACTOR)));

    // 토픽이 제대로 생성됐는지 확인한다.
    if (newTopic.numPartitions(TOPIC_NAME).get() != NUM_PARTITIONS) { 
        System.out.println("Topic has wrong number of partitions.");
        System.exit(-1); 
    }
}
```

<br>
<hr>

# 설정 관리

설정 관리는 ConfigResource 객체를 사용해서 할 수 있다. <br>

```java
        ConfigResource configResource = new ConfigResource(ConfigResource.Type.TOPIC,TOPIC_NAME);
        DescribeConfigsResult configsResult = admin.describeConfigs(Collections.singleton(configResource));
        Config configs = configsResult.all().get().get(configResource);

        // 기본값이 아닌 설정을 출력한다
        configs.entries().stream().filter(
                entry -> !entry.isDefault()).forEach(System.out::println);


        // 토픽에 압착 설정이 되어 있는지 확인한다
        ConfigEntry compaction = new ConfigEntry(TopicConfig.CLEANUP_POLICY_CONFIG,
                TopicConfig.CLEANUP_POLICY_COMPACT);
        if (! configs.entries().contains(compaction)) {
            // 토픽에 압착 설정이 되어 있지 않을 경우 해준다
            Collection<AlterConfigOp> configOp = new ArrayList<AlterConfigOp>();
            configOp.add(new AlterConfigOp(compaction, AlterConfigOp.OpType.SET));
            Map<ConfigResource, Collection<AlterConfigOp>> alterConfigs = new HashMap<>();
            alterConfigs.put(configResource, configOp);
            admin.incrementalAlterConfigs(alterConfigs).all().get();
        } else {
            System.out.println("Topic " + TOPIC_NAME + " is compacted topic");
        }
```

<br>
<hr>

# 컨슈머 그룹 관리

컨슈머 API를 사용해서 처리 지점을 되돌려서 오래된 메시지를 다시 토픽으로부터 읽어오는 방법이 있다. <br>
이러한 API를 사용한다는 것 자체가 애플리케이션에 데이터 재처리 기능을 미리 구현해 놓았다는 의미다. 그리고 애플리케이션 역시 재처리 기능을 사용 가능한 형태로 노출시켜 놓아야 한다. <br>
여기서는 AdminClient를 사용해서 프로그램적으로 컨슈머 그룹과 이 그룹들이 커밋한 오프셋을 조회하고 수정하는 방법에 대해서 살펴볼 것이다.

## 컨슈머 그룹 살펴보기

```java
admin.listConsumerGroups().valid().get().forEach(System.out::printin);
```

> 주의할 점은 valid() 메서드, get() 메서드를 호출함으로써 리턴되는 모음은 클러스터가 에러 없이 리턴한 컨슈머 그룹만을 포함한다는 점이다. <br>
> 이 과정에서 발생한 에러가 예외의 형태로 발생하지는 않는데. errors() 메서드를 사용해서 모든 예외를 가져올 수 있다.


<br>

우리는 컨슈머 그룹이 읽고 있는 각 파티션에 대해 마지막으로 커밋된 오프셋 값이 무엇인지, 최신 메시지에서 얼마나 뒤떨어졌는지를 알고 싶을 때가 있다. <br>
아래는 AdminClient를 사용해서 커밋 정보를 얻어오는 방법이다.


```java
        Map<TopicPartition, OffsetAndMetadata> offsets =
                admin.listConsumerGroupOffsets(CONSUMER_GROUP)
                        .partitionsToOffsetAndMetadata().get();

        Map<TopicPartition, OffsetSpec> requestLatestOffsets = new HashMap<>();
        Map<TopicPartition, OffsetSpec> requestEarliestOffsets = new HashMap<>();
        Map<TopicPartition, OffsetSpec> requestOlderOffsets = new HashMap<>();
        DateTime resetTo = new DateTime().minusHours(2);
        // For all topics and partitions that have offsets committed by the group, get their latest offsets, earliest offsets
        // and the offset for 2h ago. Note that I'm populating the request for 2h old offsets, but not using them.
        // You can swap the use of "Earliest" in the `alterConsumerGroupOffset` example with the offsets from 2h ago
        for(TopicPartition tp: offsets.keySet()) {
            requestLatestOffsets.put(tp, OffsetSpec.latest());
            requestEarliestOffsets.put(tp, OffsetSpec.earliest());
            requestOlderOffsets.put(tp, OffsetSpec.forTimestamp(resetTo.getMillis()));
        }

        Map<TopicPartition, ListOffsetsResult.ListOffsetsResultInfo> latestOffsets =
                admin.listOffsets(requestLatestOffsets).all().get();

        for (Map.Entry<TopicPartition, OffsetAndMetadata> e: offsets.entrySet()) {
            String topic = e.getKey().topic();
            int partition =  e.getKey().partition();
            long committedOffset = e.getValue().offset();
            long latestOffset = latestOffsets.get(e.getKey()).offset();

            System.out.println("Consumer group " + CONSUMER_GROUP
                    + " has committed offset " + committedOffset
                    + " to topic " + topic + " partition " + partition
                    + ". The latest offset in the partition is "
                    +  latestOffset + " so consumer group is "
                    + (latestOffset - committedOffset) + " records behind");
        }
```

## 컨슈머 그룹 수정하기

AdminClient는 컨슈머 그룹을 수정하기 위한 메서드들 역시 가지고 있다. (그룹 삭제, 멤버 제외, 커밋된 오프셋 삭제 혹은 변경) <br>
이것들은 SRE가 비상 상황에서 임기 응변으로 복구를 위한 툴을 제작할 때 자주 사용된다.

> 이들 중에서도 오프셋 변경 기능이 가장 유용하다. <br>
> 오프셋 삭제는 컨슈머를 맨 처음부터 실행시키는 가장 간단한 방법으로 보일 수 있지만, 이것은 컨슈머 설정에 의존한다. <br>
> 명시적으로 커밋된 오프셋을 맨 앞으로 변경하면 컨슈머는 토픽의 맨 앞에서부터 처리를 시작하게 된다.

```java
        Map<TopicPartition, ListOffsetsResult.ListOffsetsResultInfo> earliestOffsets =
                admin.listOffsets(requestEarliestOffsets).all().get();

        Map<TopicPartition, OffsetAndMetadata> resetOffsets = new HashMap<>();
        for (Map.Entry<TopicPartition, ListOffsetsResult.ListOffsetsResultInfo> e:
                earliestOffsets.entrySet()) {
            System.out.println("Will reset topic-partition " + e.getKey() + " to offset " + e.getValue().offset());
            resetOffsets.put(e.getKey(), new OffsetAndMetadata(e.getValue().offset()));
        }

        try {
            admin.alterConsumerGroupOffsets(CONSUMER_GROUP, resetOffsets).all().get();
        } catch (ExecutionException e) {
            System.out.println("Failed to update the offsets committed by group " + CONSUMER_GROUP +
                    " with error " + e.getMessage());
            if (e.getCause() instanceof UnknownMemberIdException)
                System.out.println("Check if consumer group is still active.");
        }
```

<br>
<hr>

# 고급 어드민 작업

## 토픽에 파티션 추가하기

토픽의 메시지들이 키를 가지고 있는 경우 같은 키를 가진 메시지들은 모두 동일한 파티션에 들어가 동일한 컨슈머에 의해 동일한 순서로 처리될 것이라고 생각할 수 있다. <br>
이러한 이유 때문에 토픽에 파티션을 추가해야 하는 경우는 드물며 위험할 수 있다. <br>
만약 토픽에 파티션을 추가해야 한다면 이것 때문에 토픽을 읽고 있는 애플리케이션들이 깨지지는 않을지 확인해야 할 것이다.

```java
        // add partitions
        Map<String, NewPartitions> newPartitions = new HashMap<>();
        newPartitions.put(TOPIC_NAME, NewPartitions.increaseTo(NUM_PARTITIONS+2));
        try {
            admin.createPartitions(newPartitions).all().get();
        } catch (ExecutionException e) {
            if (e.getCause() instanceof InvalidPartitionsException) {
                System.out.printf("Couldn't modify number of partitions in topic: " + e.getMessage());
            }
        }
```

## 토픽에서 레코드 삭제하기

토픽에 30일간의 보존 기한이 설정되어 있다 하더라도 파티션별로 모든 데이터가 하나의 세그먼트에 저장되어 있다면 보존 기한을 넘긴 데이터라 한들 삭제되지 않을 수다. <br>
deleteRecords 메서드는 호출 시점을 기준으로 지정된 오프셋보다 더 오래된 모든 레코드에 삭제 표시를 함으로써 컨슈머가 접근할 수 없도록 한다. <br>
이 메서드는 삭제된 레코드의 오프셋 중 가장 큰 값을 리턴하기 때문에 의도했던 대로 삭제가 이루어졌는지 확인할 수 있다.

```java
        // delete records
        Map<TopicPartition, ListOffsetsResult.ListOffsetsResultInfo> olderOffsets =
                admin.listOffsets(requestOlderOffsets).all().get();
        Map<TopicPartition, RecordsToDelete> recordsToDelete = new HashMap<>();
        for (Map.Entry<TopicPartition, ListOffsetsResult.ListOffsetsResultInfo>  e:
                olderOffsets.entrySet())
            recordsToDelete.put(e.getKey(), RecordsToDelete.beforeOffset(e.getValue().offset()));
        admin.deleteRecords(recordsToDelete).all().get();
```

## 리더 선출

이 메서드는 두 가지 서로 다른 형태의 리더 선출을 할 수 있게 해 준다.

### 선호 리더 선줄

각 파티션은 선호 리더라 불리는 레플리카를 하나씩 가진다. <br>
기본적으로, 카프카는 5분마다 선호 리더 레플리카가 실제로 리더를 맡고 있는지를 확인해서 리더를 맡을 수 있는데도 맡고 있지 않은 경우 해당 레플리카를 리더로 삼는다.

### 언클린 리더 선출

만약 어느 파티션의 리더 레플리카가 사용 불능 상태가 되었는데 다른 레플리카들은 리더를 맡을수 없는 상황이라면 해당 파티션은 리더가 없게 되고 따라서 사용 불능 상태가 된다. *(대개 데이터가 없어서 그렇다)* <br>
이 문제를 해결하는 방법 중 하나가 리더가 될 수 없는 레플리카를 그냥 리더로 삼아버리는 언클린 리더 선출을 작동시키는 것이다. <br>
이것은 데이터 유실을 초래한다. *(예전 리더에 쓰여졌지만 새 리더로 복제되지 않은 모든 이벤트는 유실된다)*

```java
        Set<TopicPartition> electableTopics = new HashSet<>();
        electableTopics.add(new TopicPartition(TOPIC_NAME, 0));
        try {
            admin.electLeaders(ElectionType.PREFERRED, electableTopics).all().get();
        } catch (ExecutionException e) {
            if (e.getCause() instanceof ElectionNotNeededException) {
                System.out.println("All leaders are preferred leaders, no need to do anything");
            }
        }
```

## 레플리카 재할당

브로커에 너무 많은 레플리카가 올라가 있어서 몇 개를 다른 데로 옮기고 싶을 수도 있고, 레플리카를 추가하고 싶을 수도 있으며, 아니면 장비를 내리기 위해 모든 레플리카를 다른 장비로 내보내야 할 수도 있다.
alterPartitionReassignments를 사용하면 파티션에 속한 각각의 레플리카의 위치를 정밀하게 제어할 수 있다.

*레플리카를 하나의 브로커에서 다른 브로커로 재할당하는 일은 브로커 간에 대량의 데이터 복제를 초래한다*

```java
        // reassign partitions to new broker
        Map<TopicPartition, Optional<NewPartitionReassignment>> reassignment = new HashMap<>();
        reassignment.put(new TopicPartition(TOPIC_NAME, 0),
                Optional.of(new NewPartitionReassignment(Arrays.asList(0,1))));
        reassignment.put(new TopicPartition(TOPIC_NAME, 1),
                Optional.of(new NewPartitionReassignment(Arrays.asList(0))));
        reassignment.put(new TopicPartition(TOPIC_NAME, 2),
                Optional.of(new NewPartitionReassignment(Arrays.asList(1,0))));
        reassignment.put(new TopicPartition(TOPIC_NAME, 3),
                Optional.empty());
        try {
            admin.alterPartitionReassignments(reassignment).all().get();
        } catch (ExecutionException e) {
            if (e.getCause() instanceof NoReassignmentInProgressException) {
                System.out.println("We tried cancelling a reassignment that was not happening anyway. Lets ignore this.");
            }
        }

        System.out.println("currently reassigning: " +
                admin.listPartitionReassignments().reassignments().get());
        demoTopic = admin.describeTopics(TOPIC_LIST);
        topicDescription = demoTopic.values().get(TOPIC_NAME).get();
        System.out.println("Description of demo topic:" + topicDescription);
```



