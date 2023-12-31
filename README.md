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

Точно также можно выполнить настройку продюсера без использования файла конфигурации
```java
@Configuration
public class KafkaProducerConfig {
    /*
            Данный бин должен быть вынесен в отдельный класс
     */
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
sendMessage overloading methods - [Spring Dock](https://docs.spring.io/spring-kafka/reference/html/#kafka-template)

Хочу отметить, что метод sendMessage возвращает `CompletableFuture<SendResult<K, V>>`, поэтому возможна следующая обработка результата отправки сообщения:
```java
@RequiredArgsConstructor
@Component
public class KafkaProducerTest {

    @Value("${kafka.topics.test-topic}")
    private String topic;

    private final KafkaTemplate<Object, Object> kafkaTemplate;

    public void sendMessages() {
        kafkaTemplate.send("message", topic).whenComplete(
                (result, ex) -> {
                    if (ex == null){
                        // DO SMTH
                    }
                    else{
                        // DO SMTH
                    }
                }
        );
    }
}
```
Считаю, что неотправленное сообщение можно отправлять в специализированный топик, дабы обработать его позже. Также для отправки сообщения можно настроить ретраи (retry), 
об этом будем написано ниже.

Также хочу отметить, что Вы можете комбинировать оба способа, в случае, когда Вам нужно определить несколько ProducerFactory для разных объектов KafkaTemplate, 
при этом часть конфигурации не меняется. Например, объекты сериализации. Рекомендую придерживаться этого подхода.

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
- `CLIENT_ID_CONFIG` 
- `BATCH_SIZE_CONFIG` - кафка отправляет сообщения пачками, по достижению этой величины сообщения будут отправлены, этот параметр указывается в байтах (по умолчанию это 16 КБ).

  Малый размер пакета сделает пакетную обработку менее распространенной и может снизить пропускную способность (нулевой размер пакета полностью отключит пакетную обработку). 
  При очень большом размере пакета память может расходоваться более расточительно, так как мы всегда будем выделять буфер заданного размера в ожидании дополнительных записей.
- `LINGER_MS_CONFIG` - время по истечению которого сообщения будет отправлено, в случае, если не накоплено достаточное количество сообщений.

  Если batch size и linger.ms не достигли граничных значений, но при этом у нас собраны батчи для одно брокера и разных партиций, суммарно для них превышен batch size или linger.ms, то данные будут отправлены.
- При желании можете также определить алгоритм сжатия сообщения при помощи параметра `COMPRESSION_TYPE_CONFIG`. 

    Рекомендую использовать `snappy` или `lz4`, поскольку оба имеют оптимальную скорость и степень сжатия. С другой стороны, `Gzip` будет иметь самую высокую степень сжатия, но он не очень быстрый.


    snappy is very useful if your messages are text-based, for example, JSON documents or logs
    snappy has a good balance of compression ratio or CPU.

- `ACKS_CONFIG` - гарантии надежности и доставки сообщений.

    KafkaProducer обеспечивает надежность данных с помощью параметра конфигурации acks. Параметр acks указывает, сколько подтверждений должен получить продюсер, чтобы запись считалась доставленной брокеру. 
    Варианты значений:

  1. none (1) — продюсер считает записи успешно доставленными после их отправки на брокера. Никакого подтверждения он не ждет.

  2. one (0) — продюсер ждет от брокера лидера подтверждение того, что он занес запись в лог. 

  3. all (-1) — продюсер ждет подтверждения от брокера лидера и ISR реплик. 

    У разных приложений разные требования, и здесь нужно найти компромисс: или это будет высокая пропускная способность, но с риском потери данных, 
    или гарантия надежности в ущерб пропускной способности.


- `DELIVERY_TIMEOUT_MS_CONFIG` - время отправки сообщения, по истечению которого доставка сообщения считается неудачной, [подробнее](https://www.conduktor.io/kafka/kafka-producer-retries/), этот параметр по сути определяет таймаут для ретраев. По умолчанию delivery.timeout.ms равен двум минутам
  Параметр retries определяет, сколько раз производитель будет пытаться отправить сообщение, прежде чем пометить его как неудачное. 
  
  - По умолчанию установлены следующие значения:
      - 0 для Kafka <= 2.0

      - MAX_INT, т.е. 2147483647 для Kafka >= 2.1

  Обычно пользователи предпочитают оставлять этот параметр без настройки и вместо него использовать delivery.timeout.ms для управления поведением при повторных попытках.
- `ENABLE_IDEMPOTENCE_CONFIG` = true и `MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION` = 5 - обеспечивает идемпотентность и при этом высокую пропускную способность

Подробнее узнать о всех параметрах вы можете в [официальной документации](https://kafka.apache.org/documentation/#producerconfigs) 
### Отправка сообщения с использованием объекта Message
```Java
 public void sendFoo(String data){

       Message<String> message = MessageBuilder
                .withPayload(data)
                .setHeader(KafkaHeaders.TOPIC, topicFoo)
                .setHeader(KafkaHeaders.MESSAGE_KEY, "999")
                .setHeader(KafkaHeaders.PARTITION_ID, 0)
                .setHeader("X-Custom-Header", "Sending Custom Header with Spring Kafka")
                .build();
       
        kafkaTemplate.send(message);
    }
```
Как именно отправлять сообщения решать именно вам - использовать объект Message или нет. 

Но обязательно при отправке указывать ключ сообщения, чтобы все сообщения, относящиеся к одному и той же сущности падали в одну и ту же партицию, дабы все возможные операции проходили в том порядке, в котором были отправлены сообщения. В нашем случае ключом может выступать id сущности.

`Вся дальнейшая настройка будет производится при помощи java-кода, точно такую же настройку вы можете выполнить при помощи файла application.yml `
### Минимальная настройка потребителя
- `KEY_DESERIALIZER_CLASS_CONFIG`
- `VALUE_DESERIALIZER_CLASS_CONFIG`
- `VALUE_DEFAULT_TYPE`
- `KEY_DEFAULT_TYPE`
  
  Когда десериализатору не удается десериализовать сообщение, Spring не имеет возможности обработать эту проблему, поскольку она возникает до возврата poll(). 
  Для решения этой проблемы был введен десериализатор ErrorHandlingDeserializer. Этот десериализатор делегирует реальному десериализатору (ключ или значение). 
  Если делегату не удается десериализовать содержимое записи, то ErrorHandlingDeserializer возвращает нулевое значение и DeserializationException в заголовке, 
  содержащем причину и необработанные байты.

  Можно использовать конструктор DefaultKafkaConsumerFactory, который принимает объекты Deserializer с ключом и значением и подключает к ним соответствующие 
  экземпляры ErrorHandlingDeserializer, которые вы сконфигурировали с соответствующими делегатами. В качестве альтернативы можно использовать свойства конфигурации 
  потребителя (которые используются ErrorHandlingDeserializer) для инстанцирования делегатов. Имена свойств ErrorHandlingDeserializer.KEY_DESERIALIZER_CLASS и ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS. 
  В качестве значения свойства может выступать класс или имя класса. В следующем примере показано, как установить эти свойства:
```java
// other props
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
props.put(ErrorHandlingDeserializer.KEY_DESERIALIZER_CLASS, JsonDeserializer.class);
props.put(JsonDeserializer.KEY_DEFAULT_TYPE, "com.example.MyKey")
props.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class.getName());
props.put(JsonDeserializer.VALUE_DEFAULT_TYPE, "com.example.MyValue")
props.put(JsonDeserializer.TRUSTED_PACKAGES, "com.example")
return new DefaultKafkaConsumerFactory<>(props);
```
  Более подробно об обработке исключений будет написано ниже


- `AUTO_OFFSET_RESET_CONFIG` -
  Когда только создается новая группа потребителей и ее потребители назначаются тематическим разделам, они должны решить, с какой точки начинать опрос сообщений. 
  Если потребителю не было дано указание опрашивать сообщения с определенного смещения (что встречается реже), то возможны два основных варианта. Во-первых, потребитель
  может читать сообщения с начала раздела, обрабатывая каждое сообщение, присутствующее в журнале раздела. Второй вариант - читать новые сообщения, записанные в тему, 
  только после того, как потребитель начал прослушивание.

  Параметры: 
    - `earliest` - с первых сообщений
    - `latest` - последние сообщения
    - `none`
Итоговая базовая настройка ConsumerFactor :
```java
    @Bean
    public ConsumerFactory<Object, Object> consumerFactory(KafkaProperties kafkaProperties){
        final var props = kafkaProperties.buildConsumerProperties();
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
        props.put(ErrorHandlingDeserializer.KEY_DESERIALIZER_CLASS, JsonDeserializer.class);
        props.put(JsonDeserializer.KEY_DEFAULT_TYPE, "com.example.MyKey");
        props.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class.getName());
        props.put(JsonDeserializer.VALUE_DEFAULT_TYPE, "com.example.MyValue");
        props.put(JsonDeserializer.TRUSTED_PACKAGES, "com.example");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        return new DefaultKafkaConsumerFactory<>(props);
    }
```
#### Определение ConcurrentKafkaListenerContainerFactory
Позволяет создавать консюмеров, которые могут обрабатывать сообщения из нескольких партиций Kafka одновременно,
также настраивать параметры такие как количество потоков, хэндлинг и т.д.
```java
@Bean
    public ConcurrentKafkaListenerContainerFactory<Object, Object> kafkaListenerContainerFactory(ConsumerFactory consumerFactory) {
        ConcurrentKafkaListenerContainerFactory<Object, Object> kafkaListenerContainerFactory = new ConcurrentKafkaListenerContainerFactory<>();
        // Настройка фабрики для создания консьюмера Kafka
        kafkaListenerContainerFactory.setConsumerFactory(consumerFactory);
        // Обработка сообщений в 4 потока
        kafkaListenerContainerFactory.setConcurrency(4);
        return kafkaListenerContainerFactory;
    }
```

#### Потребление сообщений
```java
@KafkaListener(
            // Определяет группу консюмера
            id = "consumer-group-1",
            // Определяет топик откуда читаем
            topics = "${kafka.topics.test-topic}",
            // ВАЖНО: определяет фабрику, которую мы используем. Иначе используется фабрика по умолчанию и чтение происходит в однопоточном режиме
            containerFactory = "kafkaListenerContainerFactory")
    public void handle(@Payload JsonMessage message) {
        readMessage(message);
    }
```
#### Подтверждение получения сообщения
```java
@KafkaListener(
            // Определяет группу консюмера
            id = "consumer-group-1",
            // Определяет топик откуда читаем
            topics = "${kafka.topics.test-topic}",
            // ВАЖНО: определяет фабрику, которую мы используем. Иначе используется фабрика по умолчанию и чтение происходит в однопоточном режиме
            containerFactory = "kafkaListenerContainerFactory")
    public void handle(
            @Payload JsonMessage message,
            Acknowledgment ack
        ) {
        readMessage(message);
        ack.acknowledge();
    }
```

#### Обработка исключений
```java

@RequiredArgsConstructor
public class KafkaConfiguration {

    private static final String DLT_TOPIC_SUFFIX = ".dlt";

    
    @Bean
    public KafkaTemplate<Object, Object> kafkaTemplate(ProducerFactory<Object, Object> producerFactory) {
        return new KafkaTemplate<>(producerFactory);
    }

    @Bean
    public ProducerFactory<Object, Object> producerFactory(KafkaProperties kafkaProperties){
        final var properties = kafkaProperties.buildProducerProperties();
        properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, DltMessageSerializer.class);
        return new DefaultKafkaProducerFactory<>(properties);
    }

    @Bean
    public ConsumerFactory<Object, Object> consumerFactory(KafkaProperties kafkaProperties){
        final var props = kafkaProperties.buildConsumerProperties();
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
        props.put(ErrorHandlingDeserializer.KEY_DESERIALIZER_CLASS, JsonDeserializer.class);
        props.put(JsonDeserializer.KEY_DEFAULT_TYPE, "com.example.MyKey");
        props.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class.getName());
        props.put(JsonDeserializer.VALUE_DEFAULT_TYPE, "com.example.MyValue");
        props.put(JsonDeserializer.TRUSTED_PACKAGES, "com.example");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<Object, Object> kafkaListenerContainerFactory(
            DefaultErrorHandler errorHandler,
            ConsumerFactory<Object, Object> consumerFactory
    ) {
        // Позволяет создавать консюмеров, которые могут обрабатывать сообщения из нескольких партиций Kafka одновременно,
        // а также настраивать параметры такие как количество потоков, хэндлинг и т.д.
        ConcurrentKafkaListenerContainerFactory<Object, Object> kafkaListenerContainerFactory = new ConcurrentKafkaListenerContainerFactory<>();
        // Настройка фабрики для создания консьюмера Kafka
        kafkaListenerContainerFactory.setConsumerFactory(consumerFactory);
        // Возврат сообщений в DLT очередь
        kafkaListenerContainerFactory.setCommonErrorHandler(errorHandler);
        // Обработка сообщений в 4 потока
        kafkaListenerContainerFactory.setConcurrency(4);
        return kafkaListenerContainerFactory;
    }

    /**
     * Публикатор в dead-letter topic.
     */
    @Bean
    public DeadLetterPublishingRecoverer publisher(KafkaTemplate<Object, Object> bytesTemplate) {
        //  Определяем логику выбора партиции для отправки сообщения в DLT.
        //  В данном случае, создаём новый объект TopicPartition, используя имя топика (consumerRecord.topic()) и добавляя суффикс DLT_TOPIC_SUFFIX,
        //  а также номер партиции (consumerRecord.partition()).
        //  Следовательно в DLT топике должно быть столько партиций сколько и в топике откуда читаем
        return new DeadLetterPublishingRecoverer(bytesTemplate, (consumerRecord, exception) ->
                new TopicPartition(consumerRecord.topic() + DLT_TOPIC_SUFFIX, consumerRecord.partition()));
    }

    /**
     * Обработчик исключений при получении сообщений из kafka по умолчанию.
     */
    @Bean
    public DefaultErrorHandler errorHandler(DeadLetterPublishingRecoverer deadLetterPublishingRecoverer) {
        final var handler = new DefaultErrorHandler(deadLetterPublishingRecoverer);
        // Обрабатываем любые исключения и отправляем в DLT
        handler.addNotRetryableExceptions(Exception.class);
        return handler;
    }
}
/**
 * Кастомный сериалайзер.
 */
public class DltMessageSerializer<T> implements Serializer<T> {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public void configure(Map<String, ?> configs, boolean isKey) {
        // Дополнительная конфигурация не требуется.
    }

    @Override
    public byte[] serialize(String topic, T data) {
        try {
            return objectMapper.writeValueAsBytes(data);
        } catch (JsonProcessingException e) {
            throw new SerializationException("Error when serializing to JSON", e);
        }
    }

    @Override
    public void close() {
        // Дополнительная конфигурация не требуется.
    }
}
```

### Семантика доставки
 - #### At-Most-Once Delivery - Возможна потеря сообщений
   Дабы реализовать данный тип семантики необходимо на стороне потребителя необходимо установить параметр `ENABLE_AUTO_COMMIT_CONFIG` = true
 - #### At-Most-Once Delivery - Возможна повторная обработка сообщений
   Дабы реализовать данный тип семантики необходимо на стороне производителя указать следующие параметры:
   - `ACKS_CONFIG` = 1
   - `ENABLE_IDEMPOTENCE_CONFIG` = true
   
   На стороне потребителя:
   - `ENABLE_AUTO_COMMIT_CONFIG` = false
 
### [Работа с заголовками](https://memorynotfound.com/spring-kafka-adding-custom-header-kafka-message-example/)

### [Optimizing Kafka consumers](https://strimzi.io/blog/2021/01/07/consumer-tuning/)


 

