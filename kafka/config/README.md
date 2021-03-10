## 개요

카프카 스프링에는 resources/application.yml 파일에서 설정을 읽어와 자동으로 설정을 할 수가 있다.

## application.yml

먼저, application.yml 파일을 효율적으로 작성하기 위하여
**intellij** 에서 **Spring Assistant** plugin을 설치한다.
Spring Assistant를 설치하면 application.yml에서 자동완성 기능을 사용할 수 있다.
application.yml 예제

```
spring:
  profiles:
    active: local ( 프로퍼티가 활성화되며 아래에 작성된 개별 프로퍼티에 profiles: xxx 와 같은 프로퍼티만 활성화됩니다. )
( 여기에 공통 프로퍼티를 작성합니다. )
server:
  port: 9093
---
( 개별 프로퍼티의 경우 --- 아래에 작성하며 profiles: 속성을 지정해줍니다. )
spring:
  profiles: local
  kafka:
    producer:
      bootstrap-servers: localhost:9092
      (localhost가 아닌 아이피 지정해야 함)
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
    consumer:
      bootstrap-servers: localhost:9092
      group-id: cloudit
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
    template:
      default-topic: cloudit.ace.volume
```

spring.profiles.active에 따라 지정된 profile로 자동설정을 할 수 있다.

## Producer

* 기본설정
application.yml에서 spring.kafka.producer에 설정을
KafkaAutoConfiguration이 자동으로 읽어와 Producer 설정을 해준다.
QueueSenderConfig 객체를 생성하고 @Configuration을 해준다.
application.yml설정을 그대로 사용하고 싶다면 내부에 아무런 메소드 구현을 하지 않아도 된다.
![Screenshot from 2020-05-18 14-16-56.png](/wikis/2479632164736111388/files/2747060547270827065)
* Producer 중요 옵션
spring,kafka.producer.properties에서 해당 옵션명을 입력하고 설정할 수 있다.
1. bootstrap.servers -> spring,kafka.producer에 설정
```
카프카 클러스터는 클러스터 마스터라는 개념이 없기 때문에 클러스터 내 모든 서버가 클라이언트의 요청을 받을 수 있다. 해당 옵션은 카프카 클러스터에 처음 연결을 하기 위한 호스트와 포트 정보로 구성된 리스트 정보를 나타낸다. 정의된 포맷은 "호스트 이름:포트, 호스트 이름:포트, 호스트 이름:포트"이다.
전체 카프카 리스트가 아닌 호스트 하나만 입력해 사용할 수 있지만, 이 방법을 추천하지는 않는다. 카프카 클러스터는 살아있는 상태이지만 해당 호스트만 장애가 발생하는 경우 접속이 불가하기 때문에, 리스트 전체를 입력하는 것을 권장한다. 만약 주어진 리스트의 서버 중 하나에서 장애가 발생할 경우 클라이언트는 자동으로 다른 서버로 재접속을 시도하기 때문에 사용자 프로그램에서는 문제없이 사용할 수 있게 된다.
```
2. acks
```
프로듀서가 카프카 토픽의 리더에게 메시지를 보낸 후 요청을 완료하기 전 ack의 수이다. 해당 오셥의 수가 작으면 성능이 좋지만, 메시지 손실 가능성이 있고, 반대로 수가 크면 성능이 좋지 않지만 메시지 손실 가능성도 줄어들거나 없어진다.
  > acks=0 : 만약 0으로 설정하는 경우 프로듀서는 서버로부터 어떠한 ack도 기다리지 않는다. 이경우 서버가 데이터를 받았는지 보장하지 않고, 클라이언트는 전송 실패에 대한 결과를 알지 못하기 때문에 재요청 설정도 적용되지 않는다. 메시지가 손실될 수 있지만, 서버로부터 ack에 대한 응답을 기다리지 않기 때문에 매우 빠르게 메시지를 보낼 수 있어 높은 처리량을 얻을 수 있다.
  > acks=1 : 만약 1로 설정하는 경우 리더는 데이터를 기록하지만, 모든 팔로워는 확인하지 않는다. 이 경우 일부 데이터의 손실이 발생할 수도 있다.
  > acks=all 또는 -1 : 만약 all 또는 -1로 설정하는 경우 리더는 ISR의 팔로워로부터 데이터에 대한 acks를 기다린다. 하나의 팔로워가 있는 한 데이터는 손실되지 않으며, 데이터 무손실에 대해 가장 강력하게 보장한다.
```
3. buffer.memory
```
프로듀서가 카프카 서버로 데이터를 보내기 위해 잠시 대기(배치 전송이나 딜레이 등)할 수 있는 전체 메모리 바이트(bytes)이다.
```
4. compression.type
```
프로듀서가 데이터를 압축해서 보낼 수 있는데, 어떤 타입으로 압축할지를 정할 수 있다. 옵션으로 none, gzip, snappy, lz4 같은 다양한 포맷 중 하나를 선택할 수 있다.
```
5. retries & request.timeout.ms
카프카 Producer Client는 메시지를 보내기전 버퍼메모리에 메시지를 저장 한 후 배치 사이즈에 도달했을 때 보내게 된다. 이때, 중간에 카프카 서버가 꺼져도 Client는 살아 있기에 버퍼에 남아있는 메시지가 재전송된다. Cloudit솔루션의 경우 메시지 전송 후 실패시 메시지는 재전송되면 안되기에 해당 옵션을 설정해야 한다.
> retries 옵션은 메시지 전송 후 실패시 재전송하는 횟수이다.
> request.timeout.ms 메시지 요청 후 얼마동안 기다릴 것인지에 대한 옵션이다. 단위는 millisecond 이므로 1000ms=1s 이다.
즉 메시지 전송 후 1초 동안 응답이 없으면 retries 옵션에 만큼 다시 전송하고 request.timeout.ms 시간 만큼 기다린다. 이 후에도 응답이 없으면 재전송을 하지 않는다.
테스트 화면을 보면,
retries = 1, request.timeout.ms=1000으로 설정했다.
![Screenshot from 2020-05-27 11-39-23.png](/wikis/2479632164736111388/files/2753503484644236462)
2020-05-27 11:26:26.873 메시지를 보냈다. 하지만 카프카 broker가 꺼져있어 java.util.concurrent.TimeoutException: null 이 발생한다. 이 후 27:26에 재전송을 시도하게 되고 이때도 서버가 꺼져있기에
11:28:26에 타임아웃 에러가 발생한다.
![Screenshot from 2020-05-27 11-41-00.png](/wikis/2479632164736111388/files/2753504418169323139)
그리고 나서 Broker서버가 정상 동작해도 메시지는 다시 전송되지 않는다.
하지만 retries옵션을 설정하지 않는 경우 재전송 횟수는 거의 무한대이기에 버퍼에 남아있어 전송 실패한 메시지가 재전송된다.
retries & request.timeout.ms 두 옵션을 적절하게 사용하여야 한다.
6. batch.size
```
프로듀서는 같은 파티션으로 보내는 여러 데이터를 함께 배치로 보내려고 시도한다. 이러한 동작은 클라이언트와 서버 양쪽에 성능적인 측면에서 도움이 된다.
이 설정으로 배치 크기 바이트(batch size byte) 단위를 조정할 수 있다. 정의된 크기보다는 큰 데이터는 배치를 시도하지 않게 된다. 배치를 보내기 전 클라이언트 장애가 발생하면, 배치 내에 있던 메시지는 전달되지 않는다. 만약 고가용성이 필요한 메시지의 경우라면 배치 사이즈를 주지 않는 것도 하나의 방법일 수 있다.
```
7. linger.ms
```
배치형태의 메시지를 보내기 전에 추가적인 메시지들을 기다리는 시간을 조정한다. 카프카 프로듀서는 지정된 배치 사이즈에 도달하면 이 옵션과 관계없이 즉시 메시지를 전송하고, 배치 사이즈에 도달하지 못한 상황에서 linger.ms 제한 시간이 도달했을 때 메시지들을 전송한다. 0이 기본값(지연 없음)이며, 0보다 큰 값을 설정하면 지연 시간은 조금 발생하지만 처리량은 좋아진다.
```
8. max.request.size	
```
프로듀서가 보낼 수 있는 최대 메시지 바이트 사이즈이다. 기본값은 1MB이다.
```
## Consumer

* 기본설정
application.yml에서 spring.kafka.consumer 설정을
KafkaAutoConfiguration이 자동으로 읽어와 Consumer 설정을 해준다.
QueueReceiverConfig 객체를 생성하고 @Configuration, @EnableKafka을 해준다.
application.yml설정을 그대로 사용하고 싶다면 내부에 아무런 메소드 구현을 하지 않아도 된다.
![Screenshot from 2020-05-18 14-22-07.png](/wikis/2479632164736111388/files/2747062484356862429)
* Consumer 추가설정
KafkaListenerContainerFactory와 같이 확장이 필요한 경우
@Override를 통해서 추가적인 설정을 할 수 있다.
spring,kafka.consumer.properties에서 해당 옵션명을 입력하고 설정할 수 있다.
1. fetch.min.bytes
```
한번에 가져올 수 있는 최소 데이터 사이즈이다. 만약 지정한 사이즈보다 작은 경우, 요청에 대해 응답하지 않고 데이터가 누적될때까지 기다린다.
```
2. enable.auto.commit
```
백그라운드로 주기적으로 오프셋을 커밋한다.
```
3. auto.offset.reset
```
카프카에서 초기 오프셋이 없거나 현재 오프셋이 더이상 존재하지 않은 경우(데이터가 삭제)에 다음 옵션으로 리셋한다.
- earliest : 가장 초기의 오프셋값으로 설정한다.
- lastest : 가장 마지막의 오프셋값으로 설정한다.
- none : 이전 오프셋값을 찾지 못하면 에러를 나타낸다.
```
4. fetch.max.bytes	
```
한번에 가져올 수 있는 최대 데이터 사이즈
```
5. request.timeout.ms
```
요청에 대해 응답을 기다리는 최대 시간
```
6. session.timeout.ms
```
컨슈머와 브로커 사이의 세션 타임 아웃 시간.
브로커가 컨슈머가 살아있는 것으로 판단하는 시간(기본값 10초) 만약 컨슈머가 그룹 코디네이터에게 하트비트를 보내지 않고 session.timeout.ms이 지나면 해당 컨슈머는 종료되거나 장애가 발생한 것으로 판단하고 컨슈머 그룹은 rebalance를 시도합니다.
session.timeout.ms은 하트비트 없이 얼마나 오랫동안 컨슈머가 있을 수 있는지를 제어하며, 이 속성은 heartbeat.interval.ms와 밀접한 관련이 있다. 일반적인 경우 두 속성이 함께 수정된다.
session.timeout.ms를 기본값보다 낮게 설정하면 실패를 빨리 감지할 수 있지만, 가비지 컬렉션이나 poll 루프를 완료하는 시간이 길어지게 되면 원하지 않게 리밸런스가 일어나기도 한다. 반대로 session.timeout.ms를 높게 설정하면 원하지 않는 리밸런스가 일어날 가능성은 줄지만 실제 오류를 감지하는데 시간이 오래걸릴 수 있다.
```
7. heartbeat.interval.ms
```
그룹 코디네이터에게 얼마나 자주 KafkaConsumer poll() 메소드로 하트비트를 보낼 것인지 조정한다. session.timeout.ms와 밀접한 관계가 있으며 session.timeout.ms보다 낮아야 한다. 일반적으로 3분의 1 정도로 설정한다. (기본값은 3초)
```
8. max.poll.records
```
단일 호출 poll()에 대한 최대 레코드 수를 조정한다. 이 옵션을 통해 애플리케이션이 폴링 루프에서 데이터 양을 조정할 수 있다.
```
9. max.poll.interval.ms
```
컨슈머가 살아있는지를 체크하기 위해 하트비트를 주기적으로 보내는데, 컨슈머가 계속해서 하트비트만 보내고 실제로 메시지를 가져가지 않는 경우가 있을 수도 있다. 이러한 경우 컨슈머가 무한정 해당 파티션을 점유할 수 없도록 주기적으로 poll을 호출하지 않으면 장애라고 판단하고 컨슈머 그룹에서 제외한 후 다른 컨슈머가 해당 파티션에서 메시지를 가져갈 수 있게 한다.
```
10. auto.commit.interval.ms	
```
주기적으로 오프셋을 커밋하는 시간
```
11. fetch.max.wait.ms	
```
fetch.min.bytes에 의해 설정된 데이터보다 적은 경우 요청에 응답을 기다리는 최대 시간
```

## Broker: server.properties

* auto.create.topics.enable
```
토픽 자동생성 옵션: true/false
```
* log.dirs=/tmp/kafka-logs-1
```
각 브로커에 대하여 로그를 저장 할 곳
해당 로그에 메시지에 대한 백업, 오프셋, 커밋이 저장되며 메시지 손실 시 해당 로그에서 메시지를 복구할 수 있다.
하지만 이 옵션을 통해 용량이 큰 별도의 디스크 경로로 설정해주지 않는다면, 기본값이 /tmp/kafka-logs으로 설정되어 있어 OS 영역의 디스크를 사용하게 되고, 결국 용량이 가득 차는 경우가 발생할 수 있다.
```
* log.retention.hours
```
기본값은 72이다. 하지만 기본값으로 사용을 하게 되면 결국 디스크가 full 나서 대응이 되지 않는 경우가 생긴다.
해당 옵션은 카프카에서 토픽으로 저장되는 모든 메시지를 해당 시간만큼 보관한다는 의미이다. 기본값이 168시간으로 7일을 의미한다. 예제를 위해 계산하기 쉽게 메시지 하나가 1KB라고 가정하고 10,000/sec의 메시지가 카프카로 유입된다고 가정하고 여기에 총 7일을 보관한다고 계산보면 아래와 같다.
1KB(메시지 사이즈) * 10,000(초당 메시지 수) * 60(1분) * 60(1시간) * 24(1일) * 7(일)이다. 만약 리플리카 3으로 되어 있다면 *3이다. 게다가 이런 토픽이 하나가 아니라 10개만 되어도 *10을 해야한다. 계산을 해보니 약 168TB라는 숫자가 나온다. 하지만 retetion 옵션을 7일이 아닌 3일로 변경했다면, 72TB의 공간만 있으면 된다. 필요로 하는 디스크 공간을 많이 줄일 수 있다.
```
* delete.topic.enable=true
```
기본값은 true이다. 
이 옵션은 카프카의 토픽 삭제와 관련된 옵션이다. 만약 허용하지 않는다면 삭제를 하더라도 삭제되지 않고 삭제 표시만 남아 있게 된다. 만약 위의 경우처럼 디스크가 가득 차서 빠르게 토픽을 삭제한다는 생각이 들었는데, 해당 옵션이 적용되어있지 않다면 토픽을 삭제할 수 없는 상황이 발생한다. 그래서 해당 옵션을 확인해 토픽을 삭제할 수 있도록 변경하는 것이 좋다. 필요할 때 언제든 토픽을 삭제할 수 있도록 해야한다.
```
