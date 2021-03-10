### 개요
Client가 Kafak broker에게 메시지를 보내기 위한 Client이다.

### Kafka Template
클라이언트가 카프가 브로커에게 메시지를 보내기 위해서는 Kafka Template가 필요하다.
Kafka Template를 요약하자면 아래와 같다.
* Produce records in to Kafka Topic
* Similar to JDBCTemplate for DB

Kafka Template가 메시지를 보낼 때에는 여러가지 메소드가 존재한다.
Intellij에서 KafkaTemplate를 상세하게 조회할 수 있다.
아래에는 메시지를 보내는 메소드이다.
```
ListenableFuture<SendResult<K, V>> sendDefault(V data);

ListenableFuture<SendResult<K, V>> sendDefault(K key, V data);

ListenableFuture<SendResult<K, V>> sendDefault(Integer partition, K key, V data);

ListenableFuture<SendResult<K, V>> sendDefault(Integer partition, Long timestamp, K key, V data);

ListenableFuture<SendResult<K, V>> send(String topic, V data);

ListenableFuture<SendResult<K, V>> send(String topic, K key, V data);

ListenableFuture<SendResult<K, V>> send(String topic, Integer partition, K key, V data);

ListenableFuture<SendResult<K, V>> send(String topic, Integer partition, Long timestamp, K key, V data);

ListenableFuture<SendResult<K, V>> send(ProducerRecord<K, V> record);

ListenableFuture<SendResult<K, V>> send(Message<?> message);
```
가장 많이 사용하는 두 가지를 소개하면, 
send()는 토픽과 메시지를 지정하고
sendDefault는 application.yml파일에 지정된 토픽에 대해서 메세지를 보내는 메소드이다.

KafkaTemplate.send()의 내부 구조를 소개하면 아래와 같다.
![Screenshot from 2020-05-18 16-36-00.png](/wikis/2479632164736111388/files/2747130734236888003)
메시지를 보내게 되면 Serializer를 거치고 Pationing을 한다음 LinkedList에 쌓인다음 순차적으로 브로커에 전송된다.

KafkaTemplate에 대한 설정은 2가지의 방법으로 할 수 있다.
1. 코드에서 수동설정
2. application.yml파일을 통한 AutoConfig
![Screenshot from 2020-05-18 16-40-28.png](/wikis/2479632164736111388/files/2747132298344909630)

