# 12. 카프카 운영하기

# 12.1 토픽 작업

`kafka-topics.sh` 스크립트는 Kafka 토픽을 관리하기 위한 다양한 명령어를 제공한다. 

### 토픽 생성

```
kafka-topics.sh --create --bootstrap-server <kafka-broker> --replication-factor <replication-factor> --partitions <num-partitions> --topic <topic-name>
```

- `-create`: 토픽 생성
- `-bootstrap-server`: Kafka 브로커의 주소입니다.
- `-replication-factor`: 토픽의 복제 팩
- `-partitions`: 토픽의 파티션 수
- `-topic`: 생성할 토픽의 이

### 토픽 목록 조회

```
kafka-topics.sh --list --bootstrap-server <kafka-broker>
```

- `-list`: 모든 토픽을 나열

### 토픽 설명 조회

```
kafka-topics.sh --describe --bootstrap-server <kafka-broker> --topic <topic-name>
```

- `-describe`: 특정 토픽의 세부 정보를 표시

### 토픽 삭제

```
kafka-topics.sh --delete --bootstrap-server <kafka-broker> --topic <topic-name>
```

- `-delete`: 특정 토픽을 삭제

### 토픽의 파티션 수 변경

```
kafka-topics.sh --alter --bootstrap-server <kafka-broker> --topic <topic-name> --partitions <new-num-partitions>
```

- `-alter`: 토픽의 구성을 변
- `-partitions`: 토픽의 새로운 파티션 수를 지정

---

# 12.2 컨슈머 그룹

### **모든 컨슈머 그룹 목록 조회**

```bash
kafka-consumer-groups.sh --bootstrap-server <bootstrap-server> --list
```

- **`<bootstrap-server>`**: Kafka 브로커의 주소와 포트 (예: **`localhost:9092`**)

### **특정 컨슈머 그룹의 상세 정보 조회**

```bash
kafka-consumer-groups.sh --bootstrap-server <bootstrap-server> --group <group-name> --describe
```

- **`<group-name>`**: 상세 정보를 조회하고 싶은 컨슈머 그룹의 이름

### **컨슈머 그룹 삭제**

```bash
kafka-consumer-groups.sh --bootstrap-server <bootstrap-server> --group <group-name> --delete
```

## 오프셋 관리

### 오프셋 내보내기

```
kafka-consumer-groups.sh --bootstrap-server <bootstrap-server> \
--export --group <group-name> --topic <topic-name> \
--reset-offsets --to-current --dry-run > offsets.csv
```

### 오프셋 가져오기

```
kafka-consumer-groups.sh --bootstrap-server <bootstrap-server> \
--reset-offsets --group <group-name> \
--from-file offsets.csv --execute
```

---

# 12.3 동적 설정 변경

### 토픽 설정 기본값 재정의하기

```
kafka-configs.sh --bootstrap-server <bootstrap-server> --entity-type topics --entity-name <topic-name> --alter --add-config <config-name>=<config-value>
```

### 클라이언트와 사용자 설정 기본값 재정의하기

```bash
kafka-configs.sh --bootstrap-server <bootstrap-server> 
--entity-type clients --entity-name <client-id> 
--alter --add-config <config-name>=<config-value>
```

- **`<bootstrap-server>`**: Kafka 브로커의 주소와 포트 (예: **`localhost:9092`**)
- **`<client-id>`**: 설정을 변경하려는 클라이언트의 ID
- **`<config-name>=<config-value>`**: 변경하려는 설정의 이름과 값

### 브로커 설정 기본값 재정의하기

```
kafka-configs.sh --bootstrap-server <bootstrap-server> --entity-type brokers --entity-name <broker-id> --alter --add-config <config-name>=<config-value>
```

---

# 12.5 파티션 관리

## 선호 레플리카 선출

자동 리밸런스 기능이 꺼져있을 경우, 크루즈 컨트롤과 같은 다른 오픈 소스 툴을 사용해서 언제나 카프카 클러스터의 균형을 맞춛록 해야한다.

```
kafka-leader-election.sh --bootstrap-server <bootstrap-server> \
--election-type PREFERRED \
--all-topic-partitions
```

## 파티션 레플리카 변경하기

`kafka-reassign-partitions.sh` 스크립트는 Kafka 토픽의 파티션을 재할당하거나, 즉 다른 브로커로 이동하거나, 복제본의 수를 변경하는 등의 작업을 수행할 때 사용된다. 이 스크립트를 사용하여 브로커간의 데이터를 재분배하고, 브로커의 스케일 업 또는 스케일 다운을 관리할 수 있다.

### 1. **파티션 재할당 계획 생성:**

먼저, 재할당을 위한 JSON 파일을 생성한다. 이 파일은 어떤 파티션을 어떻게 이동할지에 대한 계획을 포함한다.

```bash
kafka-reassign-partitions.sh --bootstrap-server <bootstrap-server> 
--topics-to-move-json-file <topics-to-move-json> 
--broker-list <broker-ids> --generate
```

- `<bootstrap-server>`: Kafka 브로커의 주소와 포트 (예: `localhost:9092`)
- `<topics-to-move-json>`: 이동할 토픽과 파티션을 지정하는 JSON 파일의 경로
- `<broker-ids>`: 파티션을 이동할 대상 브로커의 ID 목록 (쉼표로 구분)

### 2. **파티션 재할당 실행**

생성된 계획을 바탕으로 실제 파티션의 재할당을 수행한다.

```bash
kafka-reassign-partitions.sh --bootstrap-server <bootstrap-server> 
--reassignment-json-file <reassignment-json> --execute
```

- `<reassignment-json>`: `-generate` 옵션을 사용하여 생성된 재할당 계획 JSON 파일의 경로

### 3. **재할당 상태 확인**

재할당의 진행 상태를 확인한다.

```bash
kafka-reassign-partitions.sh --bootstrap-server <bootstrap-server> 
--reassignment-json-file <reassignment-json> --verify
```

### 예제:

1. **재할당 계획 생성:**
    
    `topics-to-move.json` 파일:
    
    ```json
    {
      "topics": [{"topic": "my-topic"}],
      "version":1
    }
    
    ```
    
    명령어:
    
    ```bash
    kafka-reassign-partitions.sh --bootstrap-server localhost:9092 --topics-to-move-json-file topics-to-move.json --broker-list "1,2,3" --generate
    
    ```
    
2. **재할당 실행 및 확인:**
    
    위 명령어의 출력으로 생성된 `reassignment-plan.json` 파일을 사용하여 재할당을 실행하고 상태를 확인한다.
    
    ```bash
    kafka-reassign-partitions.sh --bootstrap-server localhost:9092 --reassignment-json-file reassignment-plan.json --execute
    kafka-reassign-partitions.sh --bootstrap-server localhost:9092 --reassignment-json-file reassignment-plan.json --verify
    ```
    

## 로그 세그먼트 덤프 뜨기

Kafka에서 로그 세그먼트를 덤프(dump)하는 경우, 일반적으로 `DumpLogSegments` 도구를 사용한다. 이 도구는 Kafka의 로그 세그먼트 파일의 내용을 텍스트 형식으로 변환하여 콘솔에 출력하므로, 사용자가 이해하고 분석하기 쉽게 해준다.

### `DumpLogSegments` 사용 예제:

1. **Kafka 브로커의 로그 디렉터리로 이동:**
    
    브로커의 로그 디렉터리는 `server.properties` 파일의 `log.dirs` 설정에 지정되어 있다. 해당 디렉터리로 이동한다.
    
2. **`DumpLogSegments` 명령어 실행:**
    
    ```bash
    kafka-dump-log.sh --files <log-segment-file> --print-data-log
    ```
    
    - `<log-segment-file>`: 덤프를 뜨려는 로그 세그먼트 파일의 경로. 로그 세그먼트 파일은 토픽의 파티션 디렉터리 내에서 찾을 수 있으며, `.log` 확장자를 가진다.

### 상세 예제:

```bash
# Kafka 설치 디렉토리로 이동
cd /path/to/kafka

# DumpLogSegments 스크립트 실행
bin/kafka-dump-log.sh --files /tmp/kafka-logs/my-topic-0/0000000000000000000.log --print-data-log
```

### 설명:

- `-files`: 덤프를 뜨려는 로그 세그먼트 파일의 경로를 지정한다.
- `-print-data-log`: 로그 세그먼트의 레코드를 출력합니다. 이 옵션 없이 명령어를 실행하면, 로그 세그먼트의 인덱스 및 시간 인덱스 정보만 출력된다.

---

# 12.7 안전하지 않은 작업

## 클러스터 컨트롤러 이전하기

오작동하고 있는 클러스터나 브로커를 트러블 슈팅 중일 경우 컨트롤러 역할을 하는 브로커를 끄는 대신 컨트롤러 역할을 강제로 다른 브로커로 넘겨주는게 유용할 수 있다.

컨트롤러를 강제로 옮기려면 주키퍼의 /admin/controller 노드를 수동으로 삭제해주면 된다.

## 삭제될 토픽 제거하기

삭제하다 멈추는 상태를 벗어나고 싶으면 주키퍼에서 /admin/delete_topic/{topic} 노드를 지우면 된다.

- 완료 되지 않은 삭제 요청이 삭제된다.
- 만약 삭제 요청이 컨트롤러에 캐시되어 있다가 다시 올라온다면, 주키퍼에서 노드를 지운 직후 앞에서 살펴본 것과 같이 컨트롤러 역할을 강제로 옮겨줌으로써 컨트롤러에 캐시된 요청이 남지 않도록 해줘야 한다.

## 수동으로 토픽 삭제하기

1. 클러스터의 모든 브로커를 내린다.
2. 주키퍼의 카프카 클러스터 설정 노드 아래에서 /brokers/topics/{topic}을 삭제한다. 자식 노드들 먼저 삭제해야 한다는 점에 주의
3. 각 브로커의 로그 디렉토리에서 토픽에 해당하는 파티션의 디렉토리를 삭제한다. {topic}-{int}형식으로 되어 있다.
4. 모든 브로커를 재시작한다.