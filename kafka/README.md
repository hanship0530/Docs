## Kafka 요약

### 기존 메시지 시스템과 카프카 비교
![Screenshot from 2020-05-18 15-35-33.png](/wikis/2479632164736111388/files/2747099536821229333)

### Kafka Topic과 Partition
Partition is where the message lives inside the topic
Each Topic will be create with one or more partitions
![Screenshot from 2020-05-18 15-37-22.png](/wikis/2479632164736111388/files/2747100262028212572)
[그림1]
![Screenshot from 2020-05-18 15-37-56.png](/wikis/2479632164736111388/files/2747100524880620242)
[그림2]

[그림2]에서 Kafka Producer가 메시지를 Topic A에 전송하면 지정된 파티션에 따라 Offset이 순차적으로 증가하면서 파티션에 메시지가 쌓이게 된다.

### Key, Value
Kafka Message these sent from producer has two properties
* Key (optional)
* Value
* 키없이 메시지 전송
![Screenshot from 2020-05-18 16-11-17.png](/wikis/2479632164736111388/files/2747117390184481221)
토픽 내부 파티션에 메시지 전송 순서와 상관없이 메시지가 쌓인다.
파티션이 한 개 일 경우 순서대로 쌓인다.
* 키와 함께 메시지 전송
![Screenshot from 2020-05-18 16-12-33.png](/wikis/2479632164736111388/files/2747117936883839076)
Key값을 Kafka가 hash를 이용하여 Key가 할당될 파티션을 정해주고
해당 파티션에 전송순서에 따라 지정된 Key 메시지가 순서대로 쌓인다.

### Consumer Offsets
카프카 토픽내부에는 메시지가 오프셋이 증가하면서 쌓이게 된다.
메시지를 조회할때에 3가지 옵션으로 조회할 수 있다.
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
1. from-beginning
![Screenshot from 2020-05-18 16-19-46.png](/wikis/2479632164736111388/files/2747121585578157266)
토픽 메시지 맨 처음부터 조회
2. latest
3. specific offset

> Consumer offsets behaves like a bookmark for the consumer to start
reading the messages from the point it left off.

### Consumer Groups
카프카에서 Consume을 하기 위해서는 필수적으로 **group.id**를 지정해야 된다.
> group.id plays a major role when it comes
to scalable message consumption.
![Screenshot from 2020-05-18 16-23-00.png](/wikis/2479632164736111388/files/2747123209396958008)
group.id를 지정하게 되면 지정된 group.id에 대한 Consumer를 broker가 스케쥴링하여 처리하여 작업의 일관성이 보장된다.
![Screenshot from 2020-05-18 16-25-40.png](/wikis/2479632164736111388/files/2747124560502503481)

요약
- Consumer Group은 메시지 소비 확장에 사용된다.
- 각각의 다른 서비스는 유니크한 그룹을 사용한다.
- 그룹은 kafka broker가 관리한다.

### Commit Log
![Screenshot from 2020-05-18 16-27-25.png](/wikis/2479632164736111388/files/2747125496804059447)

### Retention Policy
* 메시지가 얼마나 보관될지는 정하는 정책
* server.properties파일에서 log.retention.hours 옵션에 설정할 수 있다.
* 기본 값은 168시간(7일)이다.

### Kafka Distribution
