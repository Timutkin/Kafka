# Kafka

Репозиторий содержит docker-compose файлы, позволяющие развернуть кафку с использованием контейнеров Docker, а также определяет правила по определению продюсеров и консьюмеров.
На данный момент представлено два файла `docker-compose-cluster.yml` и `docker-compose-single.yml`
### `docker-compose-cluster.yml`
Данный файл позволяет поднять кластер кафки, состоящий из трех брокеров. 
Для них выставлены следующие параметры :
 - `KAFKA_DEFAULT_REPLICATION_FACTOR: 3`
 - `KAFKA_NUM_PARTITIONS: 3` 
 - `KAFKA_MIN_INSYNC_REPLICAS: 2` 
 - `KAFKA_AUTO_CREATE_TOPICS_ENABLE: true`
### Подключение к кластеру
Подключится к кластеру вы можете следующим образом : 
 1. Если вы поднимает кластер отдельно от других сервисов, то есть не определяете в docker-compose файле свои сервисы, то в файле application.yml необходимо указать следующие параметры : 
    ```yaml
    spring:
        kafka:
          bootstrap-servers: localhost:29092,localhost:29093,localhost:29094
    ```
 2. Вы определили в docker-compose файле свои сервисы :
     ``` yaml
    spring:
        kafka:
          bootstrap-servers: host.docker.internal:29092,host.docker.internal:29093,host.docker.internal:29094
    ```
### Cоздание топика
Если Вам по каким-то причинам вам необходимо изменить REPLICATION_FACTOR (он не может быть больше трех, так как количество брокеров равно трем) или уже увеличить количество партиций, то воспользуйтесь 
классом `org.apache.kafka.clients.admin.NewTopic`, определив его в файле конфигурации следующим образом :
```java
    @Bean
    public NewTopic newTopic(){
        return TopicBuilder.name(topic)
                .replicas(replicaCount)
                .partitions(partitionCount)
                .build();
    }
```
Таким образом вы переопределите дефолтные значения для количества партиций и репликаций.
Если дефолтные настройки вас устраивают, то ничего создавать не нужно, топик будет создан автоматически, все, что Вам нужно сделать,
это определить `topicName` при отправке или получении сообщений.
### Минимальная настройка продюсера c использованием `application.yaml`
```yaml
producer:
      client-id: example
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
```
Таким образом сконфигурируется ProducerFactory, которая внедрится в объект KafkaTemplate. Отправка сообщений осуществляется посредством KafkaTemplate :
```java
@RequiredArgsConstructor
@Component
public class KafkaProducerTest {

    @Value("${kafka.topics.test-topic}")
    private String topic;

    private final KafkaTemplate<Object, Object> kafkaTemplate;
    
    public void sendMessages() {
        kafkaTemplate.send("message", topic);
    }
}
```
Точно также можно выполнить настройку продюсера без использования файла конфигурации
```java
@Configuration
public class ApplicationConfig {

    @Bean
    public ObjectMapper objectMapper() {
        return JacksonUtils.enhancedObjectMapper();
    }

    @Bean
    public ProducerFactory<Object, Object> producerFactory(
            KafkaProperties kafkaProperties, ObjectMapper mapper) {
        var props = kafkaProperties.buildProducerProperties();
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);

        var kafkaProducerFactory = new DefaultKafkaProducerFactory<Object, Object>(props);
        kafkaProducerFactory.setValueSerializer(new JsonSerializer<>(mapper));
        return kafkaProducerFactory;
    }

    @Bean
    public KafkaTemplate<Object, Object> kafkaTemplate(
            ProducerFactory<Object, Object> producerFactory) {
        return new KafkaTemplate<>(producerFactory);
    }
}
```
Также хочу отметить, что Вы можете комбинировать оба способа, в случае, когда Вам нужно определить несколько ProducerFactory для разных объектов KafkaTemplate, 
при этом часть конфигурации не меняется. Например, объекты сериализации. Настоятельно рекомендую придерживаться этого подхода.

### Дополнительные настройки продюсера
#### Поверхностное описание функции sendMessage
1. fetch metadata - продюсеру нужно знать из чего состоит кластер, его нужно знать Leader реплику. Поэтому он обращается к кластеру, а кластер к Zookeeper’у. Функция send message объявлена асинхронной, но fetch metadata происходит синхронно. Но продюсер перед каждой отправкой сообщения не выполняет fetch metadata, он эти данные кеширует и периодически обновляет.
2. serialize message - сообщение сериализуется в нужный формат. На стороне продюсера указывается key.serializer и value.serializer.
3. define partition - выбор партиции, есть следующие опции :
explicit partition - выбор конкретной партиции
round-robin - запись в каждую доступную партицию
key-defined - партиция определяется по ключу (key_hash%n), где n - количество партиций
4. compress message - процесс сжатия сообщения
5. accumulate batch - сообщение сразу не отправляется какому-то брокеру, сообщения собираются в батч,существует две настройки:
batch size
linger.ms
#### Определение параметров для ProducerFactory
- `BATCH_SIZE_CONFIG` - кафка отправляет сообщения пачками, по достижению этой величины сообщения будут отправлены
- `LINGER_MS_CONFIG` - время по истечению которого сообщения будет отправлено, в случае, если не накоплено достаточное количество сообщений
- При желании можете также определить алгоритм сжатия сообщения при помощи параметра `COMPRESSION_TYPE_CONFIG` 

Если batch size и linger.ms не достигли граничных значений, но при этом у нас собраны батчи для одно брокера и разных партиций, суммарно для них превышен batch size или linger.ms, то данные будут отправлены.
#### Отправка сообщений 
 

