# Spring Messaging & Integration

> **File:** `22_Spring_Messaging.md`
> **Part:** 7 — Advanced Topics
> **Prerequisites:** Spring Core, Spring Boot Fundamentals
> **Estimated Study Time:** 10–12 hours (read + code + practice)

---

## Table of Contents

1. [Messaging Concepts](#1-messaging-concepts)
2. [JMS with Spring](#2-jms-with-spring)
3. [AMQP with Spring (RabbitMQ)](#3-amqp-with-spring-rabbitmq)
4. [Apache Kafka with Spring](#4-apache-kafka-with-spring)
5. [Spring Cloud Stream](#5-spring-cloud-stream)
6. [Spring Integration](#6-spring-integration)
7. [Message Converters & Serialization](#7-message-converters--serialization)
8. [Error Handling & Dead Letter Queues](#8-error-handling--dead-letter-queues)
9. [Transactional Messaging](#9-transactional-messaging)
10. [Interview Questions](#10-interview-questions)
11. [Summary Cheat Sheet](#11-summary-cheat-sheet)

---

## 1. Messaging Concepts

### Why Messaging?

| Synchronous (HTTP) | Asynchronous (Messaging) |
|--------------------|--------------------------|
| Tight temporal coupling | Producer/consumer decoupled |
| Blocking | Non-blocking |
| Cascading failures | Fault isolation |
| Difficult to scale | Natural horizontal scaling |
| Direct request/response | Fire-and-forget, pub/sub |

### Core Terminology

| Term | Description |
|------|-------------|
| **Producer** | Sends messages |
| **Consumer** | Receives messages |
| **Broker** | Middleware storing/routing messages |
| **Queue** | Point-to-point (one consumer per message) |
| **Topic** | Publish-subscribe (many subscribers) |
| **Message** | Payload + headers |
| **Channel** | Logical pipe (Spring Integration) |
| **Exchange** | AMQP router (RabbitMQ) |
| **Binding** | Exchange → queue rule (AMQP) |
| **Partition** | Ordered log segment (Kafka) |
| **Consumer Group** | Kafka: consumers sharing partitions |

### Messaging Models

```
Point-to-Point (Queue):
  Producer ──► [Queue] ──► Consumer A
                       └─► Consumer B (each message to ONE consumer)

Publish-Subscribe (Topic):
  Producer ──► [Topic] ──┬─► Subscriber A
                         ├─► Subscriber B
                         └─► Subscriber C (each message to ALL)
```

### Delivery Guarantees

| Guarantee | Meaning |
|-----------|---------|
| **At most once** | May lose messages; never duplicate |
| **At least once** | Never lose; may duplicate |
| **Exactly once** | No loss, no duplicates (Kafka transactions) |

### Spring Messaging Abstraction

Spring provides:
- **`Message<T>`** — payload + headers
- **`MessageChannel`** — send messages
- **`MessageHandler`** — handle messages
- **`MessageConverter`** — serialize/deserialize
- **`@MessagingGateway`** — request/reply abstraction

---

## 2. JMS with Spring

### What is JMS?

**Java Message Service** — Java API for MOM (Message-Oriented Middleware). Brokers: ActiveMQ, Artemis, IBM MQ, HornetQ.

### Maven Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-artemis</artifactId>
</dependency>
<!-- or ActiveMQ -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-activemq</artifactId>
</dependency>
```

### Configuration

```properties
spring.artemis.mode=embedded         # or native
spring.artemis.host=localhost
spring.artemis.port=61616
spring.artemis.user=admin
spring.artemis.password=admin
```

### JmsTemplate — Sending

```java
@Service
public class JmsProducer {
    
    private final JmsTemplate jmsTemplate;
    
    public JmsProducer(JmsTemplate jmsTemplate) {
        this.jmsTemplate = jmsTemplate;
    }
    
    public void sendText(String queue, String message) {
        jmsTemplate.convertAndSend(queue, message);
    }
    
    public void sendObject(String queue, Order order) {
        jmsTemplate.convertAndSend(queue, order);
    }
    
    public void sendWithHeaders(String queue, String message) {
        jmsTemplate.convertAndSend(queue, message, msg -> {
            msg.setStringProperty("priority", "HIGH");
            msg.setIntProperty("retryCount", 0);
            return msg;
        });
    }
    
    // Receive (blocking)
    public String receive(String queue) {
        return (String) jmsTemplate.receiveAndConvert(queue);
    }
}
```

### @JmsListener — Receiving

```java
@Component
public class JmsConsumer {
    
    @JmsListener(destination = "order.queue")
    public void handleOrder(Order order) {
        System.out.println("Received: " + order);
    }
    
    @JmsListener(destination = "text.queue")
    public void handleText(String message,
                           @Header("priority") String priority) {
        System.out.println("[" + priority + "] " + message);
    }
    
    // Async reply
    @JmsListener(destination = "request.queue")
    @SendTo("response.queue")
    public String handleRequest(String request) {
        return "Processed: " + request;
    }
}
```

**Enable listeners:**

```java
@SpringBootApplication
@EnableJms
public class App { }
```

### Destination Types

```java
// Queue (point-to-point)
@JmsListener(destination = "myQueue")

// Topic (pub/sub)
@JmsListener(destination = "myTopic", 
             subscription = "durableSub1")  // durable subscription
```

### Listener Container Configuration

```java
@Configuration
public class JmsConfig {
    
    @Bean
    public DefaultJmsListenerContainerFactory jmsListenerContainerFactory(
            ConnectionFactory cf, DefaultJmsListenerContainerFactoryConfigurer configurer) {
        DefaultJmsListenerContainerFactory factory = 
            new DefaultJmsListenerContainerFactory();
        configurer.configure(factory, cf);
        factory.setConcurrency("3-10");
        factory.setSessionTransacted(true);
        factory.setErrorHandler(new MyErrorHandler());
        return factory;
    }
}
```

### Message Converter

```java
@Bean
public MessageConverter jmsConverter() {
    return new MappingJackson2MessageConverter();
}
```

Or per-template:

```java
jmsTemplate.setMessageConverter(new MappingJackson2MessageConverter());
```

### JmsTemplate vs JmsListener

| Aspect | JmsTemplate | @JmsListener |
|--------|-------------|--------------|
| Role | Producer / sync receive | Async consumer |
| Blocks | Yes (send/receive) | No (container-managed) |
| Transaction | Session transacted | Container transacted |
| Concurrency | N/A | Via container factory |

---

## 3. AMQP with Spring (RabbitMQ)

### What is AMQP?

**Advanced Message Queuing Protocol** — standardized messaging protocol with exchanges, bindings, routing keys.

### RabbitMQ Concepts

```
Producer ──► Exchange ──(binding)──► Queue ──► Consumer
                │
        Routing Key determines
        which queue(s) get the message
```

**Exchange types:**

| Type | Routing |
|------|---------|
| **Direct** | Exact match on routing key |
| **Topic** | Pattern match (`*`, `#`) |
| **Fanout** | All bound queues |
| **Headers** | Based on headers |

### Maven Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

### Configuration

```properties
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
spring.rabbitmq.virtual-host=/
```

### Declaring Queues/Exchanges

```java
@Configuration
public class RabbitConfig {
    
    @Bean
    public Queue orderQueue() {
        return QueueBuilder.durable("order.queue")
            .withArgument("x-dead-letter-exchange", "dlx.exchange")
            .withArgument("x-message-ttl", 60000)
            .build();
    }
    
    @Bean
    public DirectExchange orderExchange() {
        return new DirectExchange("order.exchange");
    }
    
    @Bean
    public Binding orderBinding(Queue orderQueue, DirectExchange orderExchange) {
        return BindingBuilder.bind(orderQueue)
            .to(orderExchange)
            .with("order.created");
    }
    
    @Bean
    public TopicExchange topicExchange() {
        return new TopicExchange("topic.exchange");
    }
    
    @Bean
    public FanoutExchange fanoutExchange() {
        return new FanoutExchange("fanout.exchange");
    }
    
    @Bean
    public Queue dlq() { return new Queue("dlq.queue"); }
    
    @Bean
    public DirectExchange dlx() { return new DirectExchange("dlx.exchange"); }
}
```

### RabbitTemplate — Sending

```java
@Service
public class OrderProducer {
    
    private final RabbitTemplate rabbitTemplate;
    
    public OrderProducer(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }
    
    public void sendOrder(Order order) {
        rabbitTemplate.convertAndSend(
            "order.exchange",
            "order.created",
            order);
    }
    
    public void sendWithHeaders(Order order) {
        rabbitTemplate.convertAndSend(
            "order.exchange",
            "order.created",
            order,
            msg -> {
                msg.getMessageProperties().setHeader("priority", "high");
                msg.getMessageProperties().setExpiration("60000");
                return msg;
            });
    }
}
```

### @RabbitListener — Receiving

```java
@Component
public class OrderConsumer {
    
    @RabbitListener(queues = "order.queue")
    public void handleOrder(Order order) {
        System.out.println("Received: " + order);
    }
    
    @RabbitListener(queues = "order.queue")
    public void handleWithHeaders(Order order,
                                  @Header("priority") String priority) {
        System.out.println("[" + priority + "] " + order);
    }
    
    // Async reply
    @RabbitListener(queues = "rpc.queue")
    public String handleRpc(String request) {
        return "Response: " + request;
    }
    
    // Multiple queues
    @RabbitListener(queues = {"q1", "q2"})
    public void handleMultiple(String message) { }
}
```

**Enable:**

```java
@SpringBootApplication
@EnableRabbit
public class App { }
```

### JSON Message Converter

```java
@Bean
public MessageConverter jsonConverter() {
    return new Jackson2JsonMessageConverter();
}

@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory cf,
                                     MessageConverter converter) {
    RabbitTemplate t = new RabbitTemplate(cf);
    t.setMessageConverter(converter);
    return t;
}

@Bean
public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
        ConnectionFactory cf, MessageConverter converter) {
    SimpleRabbitListenerContainerFactory f = 
        new SimpleRabbitListenerContainerFactory();
    f.setConnectionFactory(cf);
    f.setMessageConverter(converter);
    f.setConcurrentConsumers(3);
    f.setMaxConcurrentConsumers(10);
    f.setPrefetchCount(10);
    return f;
}
```

### Manual Acknowledgment

```java
@RabbitListener(queues = "order.queue", ackMode = "MANUAL")
public void handle(Order order, Channel channel,
                   @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
    try {
        process(order);
        channel.basicAck(tag, false);
    } catch (Exception e) {
        channel.basicNack(tag, false, false);   // → DLQ
    }
}
```

### AMQP vs JMS

| Feature | AMQP | JMS |
|---------|------|-----|
| Standard | Protocol-level | Java API |
| Brokers | RabbitMQ, Qpid | ActiveMQ, Artemis, IBM MQ |
| Model | Exchange/Binding | Queue/Topic |
| Routing | Flexible (topic, headers) | Simple |
| Interop | Cross-language | Java-focused |

---

## 4. Apache Kafka with Spring

### What is Kafka?

Distributed, partitioned, replicated **commit log** for high-throughput pub/sub streaming.

### Kafka Concepts

| Concept | Description |
|---------|-------------|
| **Topic** | Named log of messages |
| **Partition** | Ordered, append-only segment |
| **Offset** | Position in partition |
| **Producer** | Writes messages |
| **Consumer** | Reads messages |
| **Consumer Group** | Consumers sharing partitions |
| **Broker** | Kafka server |
| **Replication** | Copies of partitions across brokers |

### Maven Dependency

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
<!-- For Spring Boot -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-kafka</artifactId>
</dependency>
```

### Configuration

```properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
spring.kafka.consumer.group-id=my-group
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.properties.spring.json.trusted.packages=com.example.*
```

### KafkaTemplate — Sending

```java
@Service
public class OrderProducer {
    
    private final KafkaTemplate<String, Order> kafkaTemplate;
    
    public OrderProducer(KafkaTemplate<String, Order> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }
    
    public void sendOrder(Order order) {
        kafkaTemplate.send("orders", order.getId().toString(), order);
    }
    
    public void sendWithCallback(Order order) {
        CompletableFuture<SendResult<String, Order>> future = 
            kafkaTemplate.send("orders", order.getId().toString(), order);
        future.whenComplete((result, ex) -> {
            if (ex == null) {
                System.out.println("Sent: " + result.getRecordMetadata().offset());
            } else {
                System.err.println("Failed: " + ex.getMessage());
            }
        });
    }
    
    // With headers
    public void sendWithHeaders(Order order) {
        ProducerRecord<String, Order> record = 
            new ProducerRecord<>("orders", order.getId().toString(), order);
        record.headers().add("priority", "high".getBytes());
        kafkaTemplate.send(record);
    }
}
```

### @KafkaListener — Receiving

```java
@Component
public class OrderConsumer {
    
    @KafkaListener(topics = "orders", groupId = "order-processors")
    public void handleOrder(Order order) {
        System.out.println("Received: " + order);
    }
    
    @KafkaListener(topics = "orders", groupId = "order-processors")
    public void handleWithMetadata(
            Order order,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment ack) {
        System.out.println("P" + partition + "@" + offset + ": " + order);
        ack.acknowledge();   // manual ack
    }
    
    // Batch listener
    @KafkaListener(topics = "orders", groupId = "batch-group")
    public void handleBatch(List<Order> orders) {
        orders.forEach(System.out::println);
    }
}
```

**Enable:**

```java
@SpringBootApplication
@EnableKafka
public class App { }
```

### Consumer Configuration

```java
@Configuration
@EnableKafka
public class KafkaConfig {
    
    @Bean
    public ConsumerFactory<String, Order> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(BootstrapServersConfig, "localhost:9092");
        props.put(GroupIdConfig, "order-group");
        props.put(AutoOffsetResetConfig, "earliest");
        props.put(KeyDeserializerConfig, StringDeserializer.class);
        props.put(ValueDeserializerConfig, JsonDeserializer.class);
        return new DefaultKafkaConsumerFactory<>(props);
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Order> 
            kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Order> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3);            // 3 consumers
        factory.getContainerProperties()
            .setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        factory.getContainerProperties()
            .setErrorHandler(new SeekToCurrentErrorHandler(3));
        return factory;
    }
}
```

### Producer Configuration

```java
@Bean
public ProducerFactory<String, Order> producerFactory() {
    Map<String, Object> props = new HashMap<>();
    props.put(BootstrapServersConfig, "localhost:9092");
    props.put(KeySerializerConfig, StringSerializer.class);
    props.put(ValueSerializerConfig, JsonSerializer.class);
    props.put(RetriesConfig, 3);
    props.put(EnableIdempotenceConfig, true);      // exactly-once producer
    props.put(AcksConfig, "all");
    return new DefaultKafkaProducerFactory<>(props);
}

@Bean
public KafkaTemplate<String, Order> kafkaTemplate() {
    return new KafkaTemplate<>(producerFactory());
}
```

### Kafka Transactions (Exactly-Once)

```java
props.put(TransactionalIdConfig, "order-tx-1");

@Bean
public KafkaTransactionManager<String, Order> kafkaTransactionManager() {
    return new KafkaTransactionManager<>(producerFactory());
}

@Transactional("kafkaTransactionManager")
public void sendAtomically(List<Order> orders) {
    orders.forEach(o -> kafkaTemplate.send("orders", o.getId().toString(), o));
}
```

### Topic Auto-Creation

```java
@Bean
public NewTopic ordersTopic() {
    return TopicBuilder.name("orders")
        .partitions(6)
        .replicas(3)
        .config(TopicConfig.RETENTION_MS_CONFIG, "604800000")  // 7 days
        .build();
}
```

### Consumer Groups & Rebalancing

```
Topic "orders" (3 partitions)

Consumer Group A (order-processors):
  Consumer A1 → P0, P1
  Consumer A2 → P2

Consumer Group B (analytics):
  Consumer B1 → P0, P1, P2  (independent group, reads all)
```

- Each partition assigned to **one** consumer in a group
- Adding consumers triggers **rebalance**
- More consumers than partitions → idle consumers

### Offset Management

```properties
spring.kafka.consumer.auto-offset-reset=earliest   # or latest, none
spring.kafka.consumer.enable-auto-commit=false     # manual
```

**Seek to specific offset:**

```java
@KafkaListener(topics = "orders")
public void handle(ConsumerRecord<String, Order> record) {
    // process
}

// Or programmatically
consumer.seek(new TopicPartition("orders", 0), 100L);
```

### Kafka vs RabbitMQ vs JMS

| Feature | Kafka | RabbitMQ | JMS |
|---------|-------|----------|-----|
| Model | Distributed log | Exchange/Queue | Queue/Topic |
| Ordering | Per partition | Per queue | Per queue |
| Retention | Configurable (time/size) | Until consumed | Until consumed |
| Throughput | Very high (millions/s) | High (100Ks/s) | High |
| Replay | ✅ (offset) | ❌ | ❌ |
| Consumer groups | ✅ | ❌ (competing consumers) | ❌ |
| Best for | Streaming, event sourcing | Task queues, RPC | Enterprise Java |

---

## 5. Spring Cloud Stream

### What is Spring Cloud Stream?

A **framework for building message-driven microservices** with pluggable binders (Kafka, RabbitMQ, etc.). Same code works across brokers by swapping binders.

### Maven Dependency

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-kafka</artifactId>
</dependency>
<!-- or -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
```

### Functional Model (Modern — Spring Cloud Stream 3+)

```java
@Configuration
public class StreamConfig {
    
    // Producer: Supplier produces messages
    @Bean
    public Supplier<Order> orderSupplier() {
        return () -> new Order(UUID.randomUUID(), "new");
    }
    
    // Consumer: Consumer<Order> consumes messages
    @Bean
    public Consumer<Order> processOrder() {
        return order -> System.out.println("Processing: " + order);
    }
    
    // Processor: Function<Input, Output>
    @Bean
    public Function<Order, Receipt> generateReceipt() {
        return order -> new Receipt(order.getId(), order.getTotal());
    }
}
```

**Binding configuration:**

```properties
spring.cloud.stream.bindings.orderSupplier-out-0.destination=orders
spring.cloud.stream.bindings.processOrder-in-0.destination=orders
spring.cloud.stream.bindings.processOrder-in-0.group=order-processors
spring.cloud.stream.bindings.generateReceipt-in-0.destination=orders
spring.cloud.stream.bindings.generateReceipt-out-0.destination=receipts
```

> Naming: `<beanName>-<in|out>-<index>`

### Binders

```properties
spring.cloud.stream.default-binder=kafka
# or per-binding:
spring.cloud.stream.bindings.processOrder-in-0.binder=rabbit
```

### Error Handling

```java
@Bean
public Consumer<Order> processOrder() {
    return order -> {
        try {
            // process
        } catch (Exception e) {
            throw new RuntimeException(e);   // triggers retry/DLQ
        }
    };
}
```

**Configuration:**

```properties
spring.cloud.stream.bindings.processOrder-in-0.consumer.max-attempts=3
spring.cloud.stream.bindings.processOrder-in-0.consumer.back-off-initial-interval=1000
```

**DLQ:**

```properties
spring.cloud.stream.bindings.processOrder-in-0.consumer.enable-dlq=true
spring.cloud.stream.bindings.processOrder-in-0.consumer.dlq-name=orders-dlq
```

### Content Types

```properties
spring.cloud.stream.bindings.processOrder-in-0.content-type=application/json
```

### Why Spring Cloud Stream?

- **Broker-agnostic** — swap Kafka ↔ RabbitMQ without code changes
- **Declarative bindings** — properties-based
- **Built-in error handling, DLQ, retry**
- **Scales with Spring Cloud** ecosystem

---

## 6. Spring Integration

### What is Spring Integration?

Enterprise Integration Patterns (EIP) implementation in Spring — channels, endpoints, adapters, gateways.

### Core Concepts

| Component | Role |
|-----------|------|
| **Message** | Payload + headers |
| **MessageChannel** | Pipe between components |
| **MessageEndpoint** | Consumer/producer |
| **Channel Adapter** | Connect external system |
| **Gateway** | Request/reply abstraction |
| **Router** | Route messages |
| **Filter** | Pass/drop messages |
| **Transformer** | Convert payload |
| **Splitter** | Split into multiple |
| **Aggregator** | Combine messages |

### Java DSL Example

```java
@Configuration
@EnableIntegration
public class IntegrationConfig {
    
    @Bean
    public IntegrationFlow fileFlow() {
        return IntegrationFlow
            .from(Files.inboundAdapter(new File("/inbox"))
                    .patternFilter("*.csv"),
                  e -> e.poller(Pollers.fixedDelay(1000)))
            .transform(Files.toStringTransformer())
            .<String, Order>transform(payload -> parse(payload))
            .handle(orderService, "process")
            .get();
    }
    
    @Bean
    public IntegrationFlow httpFlow() {
        return IntegrationFlow
            .from(Http.inboundGateway("/api/orders")
                    .requestMapping(m -> m.methods(POST))
                    .requestPayloadType(Order.class))
            .handle(orderService, "create")
            .get();
    }
}
```

### Channels

```java
@Bean
public MessageChannel orderChannel() {
    return MessageChannels.direct("orderChannel").get();
}

@Bean
public MessageChannel publishSubscribeChannel() {
    return MessageChannels.publishSubscribe().get();
}

@Bean
public MessageChannel queueChannel() {
    return MessageChannels.queue(100).get();
}
```

### Gateways

```java
@MessagingGateway
public interface OrderGateway {
    @Gateway(requestChannel = "orderChannel")
    Receipt process(Order order);
}
```

### Annotations

```java
@Component
public class OrderHandler {
    
    @ServiceActivator(inputChannel = "orderChannel")
    public Receipt handle(Order order) {
        return new Receipt(order.getId());
    }
    
    @Transformer(inputChannel = "rawInput", outputChannel = "orderChannel")
    public Order transform(String raw) {
        return parse(raw);
    }
    
    @Router(inputChannel = "orderChannel")
    public String route(Order order) {
        return order.isPriority() ? "priorityChannel" : "normalChannel";
    }
    
    @Filter(inputChannel = "orderChannel")
    public boolean filter(Order order) {
        return order.getTotal() > 0;
    }
    
    @Splitter(inputChannel = "batchChannel", outputChannel = "orderChannel")
    public List<Order> split(Batch batch) {
        return batch.getOrders();
    }
}
```

### When to Use Spring Integration

- Complex routing/orchestration
- Integration with files, FTP, HTTP, JMS, mail, etc.
- Enterprise Integration Patterns
- Fine-grained control over message flow

---

## 7. Message Converters & Serialization

### Built-in Converters

| Converter | Content Type |
|-----------|-------------|
| `StringMessageConverter` | text/plain |
| `MappingJackson2MessageConverter` | application/json |
| `MarshallingMessageConverter` | XML (JAXB) |
| `ByteArrayMessageConverter` | byte[] |
| `SimpleMessageConverter` (JMS) | Serializable |
| `Jackson2JsonMessageConverter` (AMQP) | JSON |
| `JsonSerializer` / `JsonDeserializer` (Kafka) | JSON |

### Custom Converter (AMQP)

```java
public class OrderMessageConverter implements MessageConverter {
    
    private final ObjectMapper mapper = new ObjectMapper();
    
    @Override
    public Message toMessage(Object object, MessageProperties props) {
        try {
            byte[] body = mapper.writeValueAsBytes(object);
            return new Message(body, props);
        } catch (JsonProcessingException e) {
            throw new MessageConversionException("Failed", e);
        }
    }
    
    @Override
    public Object fromMessage(Message message) {
        try {
            return mapper.readValue(message.getBody(), Order.class);
        } catch (IOException e) {
            throw new MessageConversionException("Failed", e);
        }
    }
}
```

### Kafka SerDes

```properties
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.properties.spring.json.trusted.packages=com.example.model
spring.kafka.consumer.properties.spring.json.value.default.type=com.example.model.Order
```

### Avro / Schema Registry

```xml
<dependency>
    <groupId>io.confluent</groupId>
    <artifactId>kafka-avro-serializer</artifactId>
</dependency>
```

```properties
spring.kafka.producer.value-serializer=io.confluent.kafka.serializers.KafkaAvroSerializer
spring.kafka.producer.properties.schema.registry.url=http://localhost:8081
```

### Content Negotiation (Spring Integration)

```java
@Bean
public IntegrationFlow flow() {
    return IntegrationFlow
        .from(Amqp.inboundAdapter(cf, "queue"))
        .transform(Transformers.fromJson(Order.class))
        .handle(orderService, "process")
        .get();
}
```

---

## 8. Error Handling & Dead Letter Queues

### Error Handler Types

**JMS:**

```java
@Bean
public DefaultJmsListenerContainerFactory factory(ConnectionFactory cf) {
    DefaultJmsListenerContainerFactory f = new DefaultJmsListenerContainerFactory();
    f.setConnectionFactory(cf);
    f.setErrorHandler(new ErrorHandler() {
        @Override
        public void handleError(Throwable t) {
            log.error("JMS error", t);
        }
    });
    return f;
}
```

**AMQP:**

```java
@Bean
public SimpleRabbitListenerContainerFactory factory(ConnectionFactory cf) {
    SimpleRabbitListenerContainerFactory f = new SimpleRabbitListenerContainerFactory();
    f.setConnectionFactory(cf);
    f.setErrorHandler(new ConditionalRejectingErrorHandler(
        new FatalExceptionStrategy() {
            @Override
            public boolean isFatal(Throwable t) {
                return t instanceof MessageConversionException;
            }
        }));
    return f;
}
```

**Kafka:**

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, Order> factory() {
    ConcurrentKafkaListenerContainerFactory<String, Order> f = 
        new ConcurrentKafkaListenerContainerFactory<>();
    f.setConsumerFactory(consumerFactory());
    // Retry 3 times then send to DLT
    f.setCommonErrorHandler(new DefaultErrorHandler(
        new DeadLetterPublishingRecoverer(kafkaTemplate()),
        new FixedBackOff(1000L, 3L)));
    return f;
}
```

### Dead Letter Queue (RabbitMQ)

```java
@Bean
public Queue orderQueue() {
    return QueueBuilder.durable("order.queue")
        .withArgument("x-dead-letter-exchange", "dlx")
        .withArgument("x-dead-letter-routing-key", "order.failed")
        .withArgument("x-message-ttl", 60000)
        .withArgument("x-max-length", 10000)
        .build();
}

@Bean
public DirectExchange dlx() { return new DirectExchange("dlx"); }

@Bean
public Queue dlq() { return new Queue("dlq.queue"); }

@Bean
public Binding dlqBinding(Queue dlq, DirectExchange dlx) {
    return BindingBuilder.bind(dlq).to(dlx).with("order.failed");
}
```

### Dead Letter Topic (Kafka)

```java
@Bean
public DeadLetterPublishingRecoverer recoverer(KafkaTemplate<String, Object> t) {
    return new DeadLetterPublishingRecoverer(t,
        (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition()));
}

@Bean
public DefaultErrorHandler errorHandler(DeadLetterPublishingRecoverer recoverer) {
    return new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 2L));
}
```

### Retry with Backoff

**Spring Retry:**

```java
@Retryable(
    value = {TransientException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 1000, multiplier = 2))
public void process(Order order) {
    // ...
}

@Recover
public void recover(TransientException e, Order order) {
    log.error("Failed after retries: {}", order.getId());
}
```

**Enable:**

```java
@Configuration
@EnableRetry
public class RetryConfig { }
```

### Retry vs DLQ Decision

```
Transient failure?      → Retry with backoff
Permanent failure?      → Send to DLQ immediately
Unknown?                → Retry N times, then DLQ
Poison pill?            → Skip or DLQ, never retry forever
```

### Idempotency

Consumers **must be idempotent** because "at least once" delivery causes duplicates.

```java
@RabbitListener(queues = "order.queue")
public void handle(Order order) {
    if (processedOrders.contains(order.getId())) {
        return;   // already processed
    }
    process(order);
    processedOrders.add(order.getId());
}
```

Better: store processed IDs in DB with unique constraint.

---

## 9. Transactional Messaging

### The Dual-Write Problem

```
DB commit + Message send = 2 systems, no atomicity
  → DB succeeds, message fails → inconsistent
  → Message sent, DB fails → phantom message
```

### Solutions

1. **Local transaction + outbox pattern**
2. **Chained transactions** (best-effort)
3. **Kafka transactions** (exactly-once)
4. **XA transactions** (heavy, rare)

### Outbox Pattern (Recommended)

```
1. Same DB TX: [insert order] + [insert outbox_event]
2. Commit
3. Poller reads outbox → publishes to broker → marks published
```

```java
@Transactional
public void createOrder(Order order) {
    orderRepo.save(order);
    outboxRepo.save(new OutboxEvent("order.created", order));
}

@Scheduled(fixedDelay = 1000)
@Transactional
public void publishEvents() {
    outboxRepo.findUnpublished().forEach(event -> {
        kafkaTemplate.send("orders", event.getPayload());
        event.markPublished();
    });
}
```

### JMS Transaction

```java
@Bean
public JmsTransactionManager jmsTxManager(ConnectionFactory cf) {
    return new JmsTransactionManager(cf);
}

@Transactional("jmsTxManager")
public void sendAtomically() {
    jmsTemplate.convertAndSend("queue1", "msg1");
    jmsTemplate.convertAndSend("queue2", "msg2");
}
```

### AMQP Transaction

```java
@Bean
public RabbitTransactionManager rabbitTxManager(ConnectionFactory cf) {
    return new RabbitTransactionManager(cf);
}

// In listener
@RabbitListener(queues = "queue")
@Transactional
public void handle(String msg) {
    rabbitTemplate.convertAndSend("out.queue", msg);
    // Both receive + send are in same transaction
}
```

### Kafka Transaction (Exactly-Once)

```java
@Bean
public KafkaTransactionManager<String, Order> kafkaTxManager(
        ProducerFactory<String, Order> pf) {
    return new KafkaTransactionManager<>(pf);
}

@Transactional("kafkaTxManager")
public void processAndProduce(Order order) {
    // consume from input topic (in a listener)
    kafkaTemplate.send("output-topic", order);
    // Offset commit + produce in one transaction
}
```

Enable idempotent producer:

```properties
spring.kafka.producer.transaction-id-prefix=tx-
spring.kafka.producer.properties.enable.idempotence=true
spring.kafka.producer.acks=all
```

### Chained Transaction Manager (Deprecated)

```java
@Bean
public ChainedTransactionManager chainedTxManager(
        DataSourceTransactionManager dbTx,
        JmsTransactionManager jmsTx) {
    return new ChainedTransactionManager(jmsTx, dbTx);
}
```

> ⚠️ Deprecated in Spring 5.3+. Prefer outbox pattern.

### When to Use Which

| Pattern | Use When |
|---------|----------|
| Outbox | DB + any broker, high reliability |
| Kafka TX | Kafka-to-Kafka exactly-once |
| JMS/AMQP TX | Single broker, consumer-producer in same TX |
| XA | Legacy, strict atomicity across heterogeneous resources |

---

## 10. Interview Questions

### Q1. What is the difference between JMS and AMQP?

**Answer:** JMS is a **Java API** for MOM (works with any JMS broker: ActiveMQ, Artemis, IBM MQ). AMQP is a **wire protocol** (RabbitMQ, Qpid) with exchanges, bindings, and routing keys. JMS is Java-only; AMQP is language-agnostic with richer routing.

### Q2. What is the difference between a queue and a topic?

**Answer:** **Queue** = point-to-point: each message delivered to **one** consumer. **Topic** = pub/sub: each message delivered to **all** subscribers. In Kafka, partitions provide queue semantics within a consumer group and topic semantics across groups.

### Q3. How does @JmsListener work?

**Answer:** Spring's `JmsListenerContainerFactory` creates a `MessageListenerContainer` that polls the broker and dispatches messages to the annotated method. Concurrency, transactions, and error handling are configured on the container factory. Requires `@EnableJms`.

### Q4. Explain RabbitMQ exchange types.

**Answer:** 
- **Direct** — routing key exact match
- **Topic** — pattern match with `*` (one word) and `#` (zero+ words)
- **Fanout** — broadcasts to all bound queues
- **Headers** — routes based on message headers, not routing key

### Q5. How does Kafka achieve ordering?

**Answer:** Ordering is guaranteed **within a partition**, not across the topic. Use a consistent partition key (e.g., order ID) to ensure related messages land in the same partition.

### Q6. What is a Kafka consumer group?

**Answer:** A set of consumers that share partitions of a topic. Each partition is consumed by exactly one consumer in the group. Different groups read independently. Enables horizontal scaling and pub/sub semantics.

### Q7. Difference between auto-commit and manual commit in Kafka?

**Answer:** Auto-commit commits offsets periodically (may lose messages on crash). Manual commit (`AckMode.MANUAL`) commits after successful processing — at-least-once semantics. For exactly-once, use Kafka transactions.

### Q8. What is a Dead Letter Queue?

**Answer:** A queue that receives messages that couldn't be processed after retries. Prevents poison messages from blocking the main queue and allows manual inspection/reprocessing. Configured via `x-dead-letter-exchange` (RabbitMQ) or `DeadLetterPublishingRecoverer` (Kafka).

### Q9. How do you guarantee message delivery?

**Answer:** 
- **Producer**: enable `acks=all`, retries, idempotence
- **Broker**: replication factor ≥ 3, `min.insync.replicas=2`
- **Consumer**: manual acks after successful processing, idempotent handlers
- **Patterns**: outbox for DB+broker atomicity

### Q10. What is the outbox pattern?

**Answer:** To atomically update a DB and publish a message, write both into the same DB transaction (business table + outbox table). A poller reads the outbox and publishes to the broker, then marks as published. Avoids dual-write inconsistency.

### Q11. How does Spring Cloud Stream differ from Spring Kafka/Rabbit?

**Answer:** Spring Kafka/Rabbit are broker-specific. Spring Cloud Stream provides a **broker-agnostic** abstraction via binders. You write `Function`/`Consumer`/`Supplier` beans and configure bindings; swap Kafka ↔ RabbitMQ via properties.

### Q12. What is a Supplier/Function/Consumer in Spring Cloud Stream?

**Answer:** Functional programming model: `Supplier<T>` produces messages, `Consumer<T>` consumes, `Function<I, O>` transforms. Bound to broker destinations via `spring.cloud.stream.bindings.<bean>-<in|out>-0.destination`.

### Q13. Difference between Kafka and RabbitMQ?

**Answer:** Kafka = distributed log, high throughput, replayable, partitions, consumer groups. RabbitMQ = message broker with exchanges/queues, lower latency, flexible routing, per-message acks. Kafka for streaming/event sourcing; RabbitMQ for task queues/RPC.

### Q14. How do you handle poison messages?

**Answer:** Configure a DLQ after N retries. In Kafka, use `DefaultErrorHandler` with `DeadLetterPublishingRecoverer`. In RabbitMQ, set `x-dead-letter-exchange`. Also consider `ConditionalRejectingErrorHandler` for fatal conversion errors.

### Q15. What is idempotency and why is it important?

**Answer:** A handler is idempotent if processing the same message twice has the same effect as once. Important because "at-least-once" delivery can duplicate. Implement with idempotency keys (message ID) and DB unique constraints.

### Q16. How do you send a message transactionally with a DB update?

**Answer:** Use the **outbox pattern**: insert business data + outbox event in one DB TX. A separate poller publishes and marks events. Or use XA/chained transactions (discouraged). Kafka transactions work only for Kafka-to-Kafka exactly-once.

### Q17. What is @SendTo and @RabbitListener?

**Answer:** `@SendTo` on a listener method sends the return value to another destination — supports request/reply. Works with `@JmsListener`, `@RabbitListener`, `@KafkaListener`.

### Q18. Explain Kafka partitioning and keys.

**Answer:** A topic is split into partitions for parallelism. Messages with the same key go to the same partition (hash of key). Different keys distribute across partitions. Null key → round-robin. Ordering is per-partition.

### Q19. What is the difference between `enable-auto-commit` and `AckMode` in Kafka?

**Answer:** `enable-auto-commit=true` (default) commits periodically in the background. `AckMode.MANUAL`/`MANUAL_IMMEDIATE` requires calling `ack.acknowledge()` after processing. Manual gives at-least-once guarantees; auto may lose messages.

### Q20. How do you scale message consumers?

**Answer:** 
- **RabbitMQ/JMS**: increase `concurrency` on the listener container; use competing consumers.
- **Kafka**: add consumers to a group (up to partition count); increase `concurrency` on `ConcurrentKafkaListenerContainerFactory`.
- **Partition** topics to allow more consumers.

---

## 11. Summary Cheat Sheet

### Dependencies

```xml
<!-- JMS (Artemis) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-artemis</artifactId>
</dependency>

<!-- RabbitMQ -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>

<!-- Kafka -->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>

<!-- Spring Cloud Stream -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-kafka</artifactId>
</dependency>
```

### Annotations

| Annotation | Purpose |
|-----------|---------|
| `@EnableJms` | Enable JMS listeners |
| `@EnableRabbit` | Enable RabbitMQ listeners |
| `@EnableKafka` | Enable Kafka listeners |
| `@JmsListener(destination=...)` | JMS consumer |
| `@RabbitListener(queues=...)` | RabbitMQ consumer |
| `@KafkaListener(topics=..., groupId=...)` | Kafka consumer |
| `@SendTo("queue")` | Reply destination |
| `@Header("x")` | Access message header |
| `@Payload` | Access message body |
| `@EnableIntegration` | Spring Integration |
| `@MessagingGateway` | Gateway interface |
| `@ServiceActivator` | SI handler |
| `@Transformer` | SI transformer |
| `@Router` | SI router |
| `@Filter` | SI filter |
| `@Splitter` | SI splitter |
| `@Aggregator` | SI aggregator |
| `@EnableRetry` | Enable Spring Retry |
| `@Retryable` | Retry method |

### Core Templates

| Template | Broker |
|----------|--------|
| `JmsTemplate` | JMS |
| `RabbitTemplate` | RabbitMQ |
| `KafkaTemplate` | Kafka |
| `StreamBridge` | Spring Cloud Stream |

### Configuration Snippets

**JMS:**
```properties
spring.artemis.mode=native
spring.artemis.host=localhost
spring.artemis.port=61616
```

**RabbitMQ:**
```properties
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
```

**Kafka:**
```properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.consumer.group-id=my-group
spring.kafka.consumer.auto-offset-reset=earliest
```

### RabbitMQ Exchange Types

```
Direct  → exact routing key
Topic   → pattern (* = one word, # = zero+)
Fanout  → all bound queues
Headers → based on message headers
```

### Kafka Key Facts

```
Partitioning: key.hashCode() % numPartitions
Ordering: per-partition only
Consumer group: one consumer per partition
Retention: time or size based
Replay: seek to offset
```

### Error Handling Quick Reference

| Broker | DLQ Config |
|--------|-----------|
| RabbitMQ | `x-dead-letter-exchange` on queue |
| Kafka | `DeadLetterPublishingRecoverer` + `DefaultErrorHandler` |
| JMS | Broker-specific (ActiveMQ DLQ, Artemis address) |
| Spring Cloud Stream | `enable-dlq=true` |

### Transactional Messaging

```
Outbox pattern      → DB + broker atomically
Kafka TX            → exactly-once Kafka-to-Kafka
JMS/AMQP TX         → receive+send in same TX
XA                  → heavy, legacy
ChainedTx           → deprecated
```

### Delivery Guarantees

```
At most once   → auto-commit, ack before processing
At least once  → manual ack after processing (default)
Exactly once   → Kafka transactions + idempotent consumer
```

---

## Cross-References

- **Previous file:** `21_Spring_Batch.md` — Spring Batch
- **Next file:** `23_Spring_Web_Services.md` — SOAP & REST Web Services
- **Related:** `19_Spring_Microservices_Cloud.md` (Spring Cloud Stream), `05_Spring_Transaction_Management.md`

---

## Practice Exercises

1. Send and receive a JSON `Order` via RabbitMQ with `@RabbitListener`.
2. Configure a DLQ for failed messages in RabbitMQ.
3. Build a Kafka producer + consumer with manual offset commit.
4. Implement idempotent message consumption with a unique constraint.
5. Use Spring Cloud Stream `Function` to route messages between two topics.
6. Demonstrate the outbox pattern with JPA + Kafka.
7. Build a Spring Integration flow: file → transform → HTTP outbound.
8. Add retry with exponential backoff on a `@RabbitListener`.

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [21_Spring_Batch.md](./21_Spring_Batch.md)
- **Next →:** [23_Spring_Web_Services.md](./23_Spring_Web_Services.md)
- **Related:** [19_Spring_Microservices_Cloud.md](./19_Spring_Microservices_Cloud.md), [21_Spring_Batch.md](./21_Spring_Batch.md), [23_Spring_Web_Services.md](./23_Spring_Web_Services.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
