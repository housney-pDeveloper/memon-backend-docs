# 04. Clean Architecture 기반 이벤트 설계 표준

```
Document ID    : MQ-SPEC-001-04
Version        : 1.0.0
Classification : Internal / Confidential
```

---

## 1. Domain Event vs Integration Event 구분

### 1.1 정의 및 차이점

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Event Type Comparison                               │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                      Domain Event                                │    │
│  │                                                                  │    │
│  │  정의: 도메인 내부에서 발생한 비즈니스 사실의 기록                │    │
│  │                                                                  │    │
│  │  특징:                                                           │    │
│  │  - 동일 Bounded Context 내에서만 사용                            │    │
│  │  - 도메인 언어(Ubiquitous Language) 사용                         │    │
│  │  - 도메인 모델의 일부                                            │    │
│  │  - 동기/비동기 모두 가능                                         │    │
│  │  - 외부에 노출되지 않음                                          │    │
│  │                                                                  │    │
│  │  예시: OrderCreatedDomainEvent, PaymentCompletedDomainEvent      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                   │                                      │
│                                   │ 변환 (Anti-Corruption Layer)         │
│                                   ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    Integration Event                             │    │
│  │                                                                  │    │
│  │  정의: Bounded Context 간 통신을 위한 이벤트 계약                 │    │
│  │                                                                  │    │
│  │  특징:                                                           │    │
│  │  - 외부 시스템/서비스와 통신용                                   │    │
│  │  - 버전 관리 필수                                                │    │
│  │  - 스키마 계약 (Contract)                                        │    │
│  │  - 항상 비동기                                                   │    │
│  │  - MQ를 통해 전파                                                │    │
│  │                                                                  │    │
│  │  예시: order.created.v1, payment.completed.v1                    │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 변환 흐름

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Domain → Integration 변환                            │
│                                                                          │
│   Order Domain                                                           │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │  OrderService                                                   │     │
│  │  ┌──────────────────┐                                          │     │
│  │  │  createOrder()   │                                          │     │
│  │  │                  │                                          │     │
│  │  │  order.create()  │──► OrderCreatedDomainEvent               │     │
│  │  │  eventPublisher  │        │                                 │     │
│  │  │    .publish()    │        │                                 │     │
│  │  └──────────────────┘        │                                 │     │
│  └──────────────────────────────┼─────────────────────────────────┘     │
│                                 │                                        │
│                                 ▼                                        │
│   Application Layer (Domain Event Handler)                               │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │  OrderEventHandler                                              │     │
│  │  ┌───────────────────────────────────────────────────────────┐ │     │
│  │  │  @TransactionalEventListener                               │ │     │
│  │  │  void handle(OrderCreatedDomainEvent event) {              │ │     │
│  │  │      // 1. Integration Event로 변환                        │ │     │
│  │  │      OrderCreatedIntegrationEvent integration =            │ │     │
│  │  │          eventMapper.toIntegration(event);                 │ │     │
│  │  │                                                            │ │     │
│  │  │      // 2. Outbox에 저장 (같은 트랜잭션)                   │ │     │
│  │  │      outboxRepository.save(integration);                   │ │     │
│  │  │  }                                                         │ │     │
│  │  └───────────────────────────────────────────────────────────┘ │     │
│  └────────────────────────────────────────────────────────────────┘     │
│                                 │                                        │
│                                 ▼                                        │
│   Infrastructure Layer (Outbox Publisher)                                │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │  OutboxPublisher (별도 프로세스/스케줄러)                       │     │
│  │                                                                 │     │
│  │  1. Outbox 테이블에서 미발행 이벤트 조회                        │     │
│  │  2. RabbitMQ로 발행                                             │     │
│  │  3. Outbox 레코드 상태 업데이트                                 │     │
│  │                                                                 │     │
│  │  ──────────────────► RabbitMQ ────────────────────►             │     │
│  │                      order.created.v1                           │     │
│  └────────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Application Layer와 Messaging Adapter 분리

### 2.1 레이어 구조

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Clean Architecture Layers                           │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    Domain Layer                                  │    │
│  │                                                                  │    │
│  │  ├── domain/                                                     │    │
│  │  │   ├── order/                                                  │    │
│  │  │   │   ├── Order.java              (Aggregate Root)           │    │
│  │  │   │   ├── OrderItem.java          (Entity)                   │    │
│  │  │   │   ├── OrderStatus.java        (Value Object)             │    │
│  │  │   │   └── event/                                              │    │
│  │  │   │       └── OrderCreatedDomainEvent.java                   │    │
│  │  │   └── DomainEventPublisher.java   (Interface)                │    │
│  │  │                                                               │    │
│  │  │  ※ 외부 의존성 없음                                           │    │
│  │  │  ※ MQ, DB 등 인프라 개념 없음                                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                   │                                      │
│                                   ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                  Application Layer                               │    │
│  │                                                                  │    │
│  │  ├── application/                                                │    │
│  │  │   ├── order/                                                  │    │
│  │  │   │   ├── CreateOrderUseCase.java                            │    │
│  │  │   │   ├── OrderEventHandler.java                             │    │
│  │  │   │   └── OrderQueryService.java                             │    │
│  │  │   ├── port/                                                   │    │
│  │  │   │   ├── in/                                                 │    │
│  │  │   │   │   └── CreateOrderCommand.java                        │    │
│  │  │   │   └── out/                                                │    │
│  │  │   │       ├── OrderRepository.java       (Interface)         │    │
│  │  │   │       ├── EventOutboxPort.java       (Interface)         │    │
│  │  │   │       └── MessagePublisherPort.java  (Interface)         │    │
│  │  │   │                                                           │    │
│  │  │  ※ Use Case 오케스트레이션                                    │    │
│  │  │  ※ Port(인터페이스)만 정의, 구현 없음                         │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                   │                                      │
│                                   ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                Infrastructure Layer (Adapters)                   │    │
│  │                                                                  │    │
│  │  ├── infrastructure/                                             │    │
│  │  │   ├── persistence/                                            │    │
│  │  │   │   ├── OrderJpaRepository.java                            │    │
│  │  │   │   └── OutboxJpaRepository.java                           │    │
│  │  │   │                                                           │    │
│  │  │   └── messaging/                  <── Messaging Adapter       │    │
│  │  │       ├── rabbitmq/                                           │    │
│  │  │       │   ├── RabbitMqPublisher.java    (Adapter)            │    │
│  │  │       │   ├── RabbitMqConsumer.java     (Adapter)            │    │
│  │  │       │   ├── RabbitMqConfig.java                            │    │
│  │  │       │   └── MessageConverter.java                          │    │
│  │  │       ├── outbox/                                             │    │
│  │  │       │   ├── OutboxPublisher.java                           │    │
│  │  │       │   └── OutboxScheduler.java                           │    │
│  │  │       └── event/                                              │    │
│  │  │           ├── IntegrationEvent.java      (Base)              │    │
│  │  │           └── OrderCreatedIntegrationEvent.java              │    │
│  │  │                                                               │    │
│  │  │  ※ Port 구현체                                                │    │
│  │  │  ※ MQ 기술 의존성 캡슐화                                      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Port 인터페이스 정의

```java
// application/port/out/MessagePublisherPort.java
public interface MessagePublisherPort {

    /**
     * 메시지를 발행한다.
     *
     * @param exchange   대상 Exchange
     * @param routingKey 라우팅 키
     * @param event      Integration Event
     */
    void publish(String exchange, String routingKey, IntegrationEvent event);

    /**
     * 지연 메시지를 발행한다.
     *
     * @param exchange   대상 Exchange
     * @param routingKey 라우팅 키
     * @param event      Integration Event
     * @param delay      지연 시간 (밀리초)
     */
    void publishWithDelay(String exchange, String routingKey,
                          IntegrationEvent event, long delay);
}

// application/port/out/EventOutboxPort.java
public interface EventOutboxPort {

    /**
     * Outbox에 이벤트를 저장한다.
     */
    void save(OutboxEntry entry);

    /**
     * 미발행 이벤트를 조회한다.
     */
    List<OutboxEntry> findPendingEvents(int limit);

    /**
     * 이벤트 발행 완료를 표시한다.
     */
    void markAsPublished(String eventId);

    /**
     * 이벤트 발행 실패를 표시한다.
     */
    void markAsFailed(String eventId, String reason);
}
```

### 2.3 Messaging Adapter 구현

```java
// infrastructure/messaging/rabbitmq/RabbitMqPublisher.java
@Component
@RequiredArgsConstructor
public class RabbitMqPublisher implements MessagePublisherPort {

    private final RabbitTemplate rabbitTemplate;
    private final ObjectMapper objectMapper;
    private final MeterRegistry meterRegistry;

    @Override
    public void publish(String exchange, String routingKey, IntegrationEvent event) {
        try {
            MessageProperties props = new MessageProperties();
            props.setMessageId(event.getEventId());
            props.setTimestamp(Date.from(event.getOccurredAt()));
            props.setContentType(MessageProperties.CONTENT_TYPE_JSON);
            props.setHeader("event_type", event.getEventType());
            props.setHeader("event_version", event.getVersion());
            props.setHeader("trace_id", event.getTraceId());

            byte[] body = objectMapper.writeValueAsBytes(event);
            Message message = new Message(body, props);

            rabbitTemplate.send(exchange, routingKey, message);

            meterRegistry.counter("mq.publish.success",
                "exchange", exchange,
                "event_type", event.getEventType()
            ).increment();

        } catch (Exception e) {
            meterRegistry.counter("mq.publish.failure",
                "exchange", exchange,
                "event_type", event.getEventType()
            ).increment();
            throw new MessagePublishException("Failed to publish message", e);
        }
    }

    @Override
    public void publishWithDelay(String exchange, String routingKey,
                                  IntegrationEvent event, long delay) {
        MessageProperties props = new MessageProperties();
        props.setMessageId(event.getEventId());
        props.setHeader("x-delay", delay);
        // ... delayed message exchange로 발행
    }
}
```

---

## 3. Outbox Pattern 설계

### 3.1 Outbox Pattern 개요

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Outbox Pattern Flow                               │
│                                                                          │
│   Application                    Database                   RabbitMQ    │
│  ┌───────────┐                ┌────────────┐              ┌──────────┐  │
│  │           │  Transaction  │            │              │          │  │
│  │  Service  │───────────────│   Order    │              │ Exchange │  │
│  │           │      BEGIN    │   Table    │              │          │  │
│  │           │               │            │              │          │  │
│  │  ┌─────┐  │               │  ┌──────┐  │              │          │  │
│  │  │ 1   │  │  INSERT       │  │Order │  │              │          │  │
│  │  └──┬──┘  │──────────────►│  │ Row  │  │              │          │  │
│  │     │     │               │  └──────┘  │              │          │  │
│  │     ▼     │               │            │              │          │  │
│  │  ┌─────┐  │               │  ┌──────┐  │              │          │  │
│  │  │ 2   │  │  INSERT       │  │Outbox│  │              │          │  │
│  │  └──┬──┘  │──────────────►│  │ Row  │  │              │          │  │
│  │     │     │               │  └──────┘  │              │          │  │
│  │     ▼     │               │            │              │          │  │
│  │  ┌─────┐  │               │            │              │          │  │
│  │  │ 3   │  │  COMMIT       │            │              │          │  │
│  │  └─────┘  │───────────────│            │              │          │  │
│  └───────────┘               └────────────┘              └──────────┘  │
│                                    │                                    │
│                                    │ (별도 프로세스)                    │
│                                    ▼                                    │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  Outbox Publisher (Polling / CDC)                                  │ │
│  │                                                                    │ │
│  │  ┌─────┐  SELECT WHERE     ┌──────┐       PUBLISH     ┌──────────┐│ │
│  │  │ 4   │  status=PENDING   │Outbox│ ─────────────────►│ RabbitMQ ││ │
│  │  └──┬──┘ ────────────────► │ Row  │                   │          ││ │
│  │     │                      └──────┘                   └──────────┘│ │
│  │     ▼                          │                                   │ │
│  │  ┌─────┐  UPDATE               │                                   │ │
│  │  │ 5   │  status=PUBLISHED     │                                   │ │
│  │  └─────┘ ◄─────────────────────┘                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ※ 트랜잭션 보장: Order와 Outbox가 같은 트랜잭션에서 저장                │
│  ※ At-Least-Once: MQ 발행 실패 시 재시도 가능                            │
│  ※ Idempotency: Consumer는 event_id 기반 중복 처리 필수                 │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Outbox 테이블 설계

```sql
-- Outbox 테이블
CREATE TABLE tb_event_outbox (
    event_id        VARCHAR(36)     PRIMARY KEY,
    event_type      VARCHAR(100)    NOT NULL,
    event_version   VARCHAR(10)     NOT NULL DEFAULT 'v1',
    aggregate_type  VARCHAR(100)    NOT NULL,
    aggregate_id    VARCHAR(100)    NOT NULL,

    payload         JSONB           NOT NULL,

    exchange        VARCHAR(100)    NOT NULL,
    routing_key     VARCHAR(200)    NOT NULL,

    status          VARCHAR(20)     NOT NULL DEFAULT 'PENDING',
    -- PENDING, PUBLISHED, FAILED

    retry_count     INT             NOT NULL DEFAULT 0,
    max_retries     INT             NOT NULL DEFAULT 3,
    last_error      TEXT,

    trace_id        VARCHAR(36),
    client_no       BIGINT,
    user_no         BIGINT,

    created_at      TIMESTAMP       NOT NULL DEFAULT CURRENT_TIMESTAMP,
    published_at    TIMESTAMP,
    updated_at      TIMESTAMP       NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_status CHECK (status IN ('PENDING', 'PUBLISHED', 'FAILED'))
);

-- 인덱스
CREATE INDEX idx_outbox_status_created ON tb_event_outbox (status, created_at)
    WHERE status = 'PENDING';

CREATE INDEX idx_outbox_aggregate ON tb_event_outbox (aggregate_type, aggregate_id);

-- 파티셔닝 (월별)
CREATE TABLE tb_event_outbox_2026_02 PARTITION OF tb_event_outbox
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
```

### 3.3 Transactional Outbox 구현

```java
// application/order/OrderEventHandler.java
@Component
@RequiredArgsConstructor
public class OrderEventHandler {

    private final EventOutboxPort outboxPort;
    private final IntegrationEventMapper eventMapper;

    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void handle(OrderCreatedDomainEvent domainEvent) {
        // 1. Domain Event → Integration Event 변환
        OrderCreatedIntegrationEvent integrationEvent =
            eventMapper.toIntegrationEvent(domainEvent);

        // 2. Outbox Entry 생성
        OutboxEntry entry = OutboxEntry.builder()
            .eventId(integrationEvent.getEventId())
            .eventType(integrationEvent.getEventType())
            .eventVersion(integrationEvent.getVersion())
            .aggregateType("Order")
            .aggregateId(domainEvent.getOrderId())
            .payload(integrationEvent)
            .exchange("ef.exchange.order")
            .routingKey("order.created")
            .traceId(TraceContext.currentTraceId())
            .clientNo(SecurityContext.getCurrentClientNo())
            .userNo(SecurityContext.getCurrentUserNo())
            .build();

        // 3. Outbox에 저장 (같은 트랜잭션)
        outboxPort.save(entry);
    }
}

// infrastructure/messaging/outbox/OutboxPublisher.java
@Component
@RequiredArgsConstructor
@Slf4j
public class OutboxPublisher {

    private final EventOutboxPort outboxPort;
    private final MessagePublisherPort publisherPort;
    private final ObjectMapper objectMapper;

    @Scheduled(fixedDelay = 1000)  // 1초마다
    @SchedulerLock(name = "OutboxPublisher", lockAtLeastFor = "PT30S")
    public void publishPendingEvents() {
        List<OutboxEntry> pendingEvents = outboxPort.findPendingEvents(100);

        for (OutboxEntry entry : pendingEvents) {
            try {
                IntegrationEvent event = objectMapper.readValue(
                    entry.getPayload(),
                    IntegrationEvent.class
                );

                publisherPort.publish(
                    entry.getExchange(),
                    entry.getRoutingKey(),
                    event
                );

                outboxPort.markAsPublished(entry.getEventId());
                log.info("Published event: {} ({})",
                    entry.getEventId(), entry.getEventType());

            } catch (Exception e) {
                log.error("Failed to publish event: {}", entry.getEventId(), e);

                if (entry.getRetryCount() >= entry.getMaxRetries()) {
                    outboxPort.markAsFailed(entry.getEventId(), e.getMessage());
                } else {
                    outboxPort.incrementRetry(entry.getEventId());
                }
            }
        }
    }
}
```

---

## 4. Event ID 전략

### 4.1 Event ID 형식

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Event ID Format                                  │
│                                                                          │
│  Format: {timestamp}-{node-id}-{sequence}-{random}                       │
│                                                                          │
│  Example: 20260227103045-ef01-000001-a7b3c9d2                           │
│                                                                          │
│  ┌──────────────┐ ┌────────┐ ┌────────┐ ┌──────────┐                    │
│  │  timestamp   │ │ node   │ │  seq   │ │  random  │                    │
│  │ 14자리       │ │ 4자리  │ │ 6자리  │ │ 8자리    │                    │
│  │ yyyyMMdd     │ │ 서버ID │ │ 순번   │ │ 충돌방지 │                    │
│  │ HHmmss       │ │        │ │        │ │          │                    │
│  └──────────────┘ └────────┘ └────────┘ └──────────┘                    │
│                                                                          │
│  Alternative: UUID v7 (Time-ordered UUID)                                │
│  - 시간 순서 정렬 가능                                                   │
│  - 표준 UUID 형식 (36자)                                                 │
│                                                                          │
│  Example: 01920b9e-5a3c-7def-8a12-3456789abcde                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Event ID Generator 구현

```java
// infrastructure/messaging/event/EventIdGenerator.java
@Component
public class EventIdGenerator {

    private static final DateTimeFormatter TIMESTAMP_FORMAT =
        DateTimeFormatter.ofPattern("yyyyMMddHHmmss");

    private final String nodeId;
    private final AtomicLong sequence = new AtomicLong(0);
    private volatile String lastTimestamp = "";

    public EventIdGenerator(@Value("${server.node-id:ef01}") String nodeId) {
        this.nodeId = nodeId;
    }

    public synchronized String generate() {
        String timestamp = LocalDateTime.now().format(TIMESTAMP_FORMAT);

        // 같은 타임스탬프 내에서 순번 증가
        if (!timestamp.equals(lastTimestamp)) {
            lastTimestamp = timestamp;
            sequence.set(0);
        }

        long seq = sequence.incrementAndGet();
        String random = generateRandom(8);

        return String.format("%s-%s-%06d-%s",
            timestamp, nodeId, seq, random);
    }

    // UUID v7 방식 (권장)
    public String generateUuidV7() {
        return UuidCreator.getTimeOrderedEpoch().toString();
    }

    private String generateRandom(int length) {
        byte[] bytes = new byte[length / 2];
        ThreadLocalRandom.current().nextBytes(bytes);
        return HexFormat.of().formatHex(bytes);
    }
}
```

---

## 5. 멱등성 설계 전략

### 5.1 Consumer 멱등성 보장 패턴

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Idempotency Patterns                                │
│                                                                          │
│  Pattern 1: Event ID 기반 중복 체크                                       │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │   Consumer                     Processed Events Table           │    │
│  │  ┌──────────┐                 ┌────────────────────┐            │    │
│  │  │ Message  │  Check exists   │ event_id (PK)      │            │    │
│  │  │ Received │────────────────►│ processed_at       │            │    │
│  │  │          │                 │ consumer_id        │            │    │
│  │  │          │◄────────────────│                    │            │    │
│  │  │          │  Not exists     │                    │            │    │
│  │  │          │                 │                    │            │    │
│  │  │ Process  │  INSERT         │                    │            │    │
│  │  │ & INSERT │────────────────►│ [new row]          │            │    │
│  │  └──────────┘                 └────────────────────┘            │    │
│  │                                                                  │    │
│  │  ※ 이미 처리된 이벤트는 ACK만 보내고 스킵                        │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Pattern 2: Idempotency Key 기반 (비즈니스 키)                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │  Key: {aggregate_type}:{aggregate_id}:{event_type}              │    │
│  │                                                                  │    │
│  │  Example: "Order:12345:created"                                  │    │
│  │                                                                  │    │
│  │  ※ 동일 Aggregate의 동일 이벤트 타입 중복 방지                   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Pattern 3: 상태 기반 멱등성                                              │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │  if (entity.status == "already_processed") {                     │    │
│  │      log.info("Already processed, skipping");                    │    │
│  │      return;                                                     │    │
│  │  }                                                               │    │
│  │  // Process and update status                                    │    │
│  │                                                                  │    │
│  │  ※ Entity 상태로 중복 처리 판단                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Processed Events 테이블

```sql
-- 처리 완료 이벤트 테이블
CREATE TABLE tb_processed_events (
    event_id        VARCHAR(36)     PRIMARY KEY,
    consumer_id     VARCHAR(50)     NOT NULL,
    event_type      VARCHAR(100)    NOT NULL,
    aggregate_type  VARCHAR(100),
    aggregate_id    VARCHAR(100),
    processed_at    TIMESTAMP       NOT NULL DEFAULT CURRENT_TIMESTAMP,

    -- 보관 기간 후 자동 삭제를 위한 인덱스
    CONSTRAINT idx_processed_events_date
        CHECK (processed_at > CURRENT_TIMESTAMP - INTERVAL '30 days')
);

-- 파티셔닝 (일별, 자동 삭제)
CREATE TABLE tb_processed_events_2026_02_27
    PARTITION OF tb_processed_events
    FOR VALUES FROM ('2026-02-27') TO ('2026-02-28');

-- 30일 지난 파티션 자동 삭제 (pg_cron)
SELECT cron.schedule('0 1 * * *',
    $$DROP TABLE IF EXISTS tb_processed_events_$$ ||
    to_char(CURRENT_DATE - INTERVAL '30 days', 'YYYY_MM_DD')
);
```

### 5.3 Consumer 멱등성 구현

```java
// infrastructure/messaging/rabbitmq/IdempotentMessageConsumer.java
@Component
@RequiredArgsConstructor
@Slf4j
public abstract class IdempotentMessageConsumer<T extends IntegrationEvent> {

    private final ProcessedEventRepository processedEventRepository;
    private final TransactionTemplate transactionTemplate;

    @RabbitListener(queues = "#{queueName}")
    public void onMessage(Message message, Channel channel) throws IOException {
        String eventId = message.getMessageProperties().getMessageId();
        String eventType = (String) message.getMessageProperties()
            .getHeader("event_type");

        try {
            // 1. 중복 체크
            if (processedEventRepository.existsById(eventId)) {
                log.info("Event already processed, skipping: {} ({})",
                    eventId, eventType);
                channel.basicAck(message.getMessageProperties()
                    .getDeliveryTag(), false);
                return;
            }

            // 2. 메시지 역직렬화
            T event = deserialize(message);

            // 3. 처리 + 처리 완료 기록 (같은 트랜잭션)
            transactionTemplate.execute(status -> {
                // 비즈니스 로직 실행
                processEvent(event);

                // 처리 완료 기록
                ProcessedEvent processed = ProcessedEvent.builder()
                    .eventId(eventId)
                    .consumerId(getConsumerId())
                    .eventType(eventType)
                    .aggregateType(event.getAggregateType())
                    .aggregateId(event.getAggregateId())
                    .build();

                processedEventRepository.save(processed);
                return null;
            });

            // 4. ACK
            channel.basicAck(message.getMessageProperties()
                .getDeliveryTag(), false);

            log.info("Event processed successfully: {} ({})",
                eventId, eventType);

        } catch (DuplicateKeyException e) {
            // 동시에 같은 이벤트 처리 시도 (Race Condition)
            log.warn("Duplicate event processing detected: {}", eventId);
            channel.basicAck(message.getMessageProperties()
                .getDeliveryTag(), false);

        } catch (Exception e) {
            log.error("Failed to process event: {} ({})",
                eventId, eventType, e);
            // NACK - DLQ로 이동
            channel.basicNack(message.getMessageProperties()
                .getDeliveryTag(), false, false);
        }
    }

    protected abstract T deserialize(Message message);
    protected abstract void processEvent(T event);
    protected abstract String getConsumerId();
}
```

---

## 6. 이벤트 버저닝 전략

### 6.1 버저닝 정책

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Event Versioning Strategy                           │
│                                                                          │
│  Version Format: v{major}.{minor}                                        │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Breaking Changes (Major Version)                                │    │
│  │                                                                  │    │
│  │  - 필드 삭제                                                     │    │
│  │  - 필드 타입 변경                                                │    │
│  │  - 필수 필드 추가                                                │    │
│  │  - 의미 변경                                                     │    │
│  │                                                                  │    │
│  │  v1.0 → v2.0                                                     │    │
│  │  기존 Consumer 호환 불가                                         │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Non-Breaking Changes (Minor Version)                            │    │
│  │                                                                  │    │
│  │  - 선택적 필드 추가                                              │    │
│  │  - 문서/설명 변경                                                │    │
│  │                                                                  │    │
│  │  v1.0 → v1.1                                                     │    │
│  │  기존 Consumer 호환 가능                                         │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 버전 공존 전략

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Version Coexistence Strategy                         │
│                                                                          │
│  Producer                         RabbitMQ                Consumer      │
│  ┌──────────┐                  ┌────────────┐            ┌──────────┐   │
│  │ v1 + v2  │                  │            │            │ v1 only  │   │
│  │ events   │ ────────────────►│  Exchange  │───────────►│ consumer │   │
│  │          │                  │            │            │          │   │
│  │          │                  │ routing:   │            │          │   │
│  │          │                  │ *.v1       │            │          │   │
│  │          │                  │ *.v2       │            │          │   │
│  └──────────┘                  └────────────┘            └──────────┘   │
│                                       │                                  │
│                                       │ routing: *.v2                    │
│                                       ▼                                  │
│                                ┌──────────┐                              │
│                                │ v2 only  │                              │
│                                │ consumer │                              │
│                                │          │                              │
│                                └──────────┘                              │
│                                                                          │
│  Routing Key Examples:                                                   │
│  - order.created.v1                                                      │
│  - order.created.v2                                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.3 마이그레이션 절차

```
Phase 1: 신규 버전 Consumer 배포
┌─────────────────────────────────────────────────────────────────────────┐
│  - v2 Consumer 배포 (v2 큐 구독)                                        │
│  - v1 Consumer 유지                                                      │
│  - Producer: v1만 발행                                                   │
└─────────────────────────────────────────────────────────────────────────┘
                                   ↓
Phase 2: Producer 이중 발행
┌─────────────────────────────────────────────────────────────────────────┐
│  - Producer: v1 + v2 동시 발행                                          │
│  - v1 Consumer: v1 처리                                                  │
│  - v2 Consumer: v2 처리                                                  │
│  - 모니터링: v2 정상 동작 확인                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                   ↓
Phase 3: v1 Consumer 제거
┌─────────────────────────────────────────────────────────────────────────┐
│  - v1 Consumer 제거                                                      │
│  - v1 큐 삭제                                                            │
│  - Producer: v2만 발행                                                   │
│  - v1 Exchange binding 제거                                              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 이벤트 스키마 관리 전략

### 7.1 스키마 관리 구조

```
event-schemas/
├── order/
│   ├── order.created.v1.json
│   ├── order.created.v2.json
│   ├── order.updated.v1.json
│   └── order.cancelled.v1.json
├── payment/
│   ├── payment.completed.v1.json
│   └── payment.failed.v1.json
├── user/
│   └── user.registered.v1.json
└── _common/
    ├── base-event.json
    └── address.json
```

### 7.2 JSON Schema 예시

```json
// order/order.created.v1.json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://platform.internal/schemas/order/order.created.v1.json",
  "title": "OrderCreatedEvent",
  "description": "주문 생성 이벤트 (v1)",
  "type": "object",

  "allOf": [
    { "$ref": "../_common/base-event.json" }
  ],

  "properties": {
    "event_type": {
      "const": "order.created"
    },
    "version": {
      "const": "v1"
    },
    "payload": {
      "type": "object",
      "properties": {
        "order_id": {
          "type": "string",
          "format": "uuid",
          "description": "주문 ID"
        },
        "customer_id": {
          "type": "string",
          "format": "uuid",
          "description": "고객 ID"
        },
        "items": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/OrderItem"
          },
          "minItems": 1
        },
        "total_amount": {
          "type": "number",
          "minimum": 0,
          "description": "총 주문 금액"
        },
        "currency": {
          "type": "string",
          "enum": ["KRW", "USD"],
          "default": "KRW"
        },
        "ordered_at": {
          "type": "string",
          "format": "date-time"
        }
      },
      "required": ["order_id", "customer_id", "items", "total_amount", "ordered_at"]
    }
  },

  "$defs": {
    "OrderItem": {
      "type": "object",
      "properties": {
        "product_id": { "type": "string" },
        "product_name": { "type": "string" },
        "quantity": { "type": "integer", "minimum": 1 },
        "unit_price": { "type": "number", "minimum": 0 }
      },
      "required": ["product_id", "quantity", "unit_price"]
    }
  }
}

// _common/base-event.json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://platform.internal/schemas/_common/base-event.json",
  "title": "BaseEvent",
  "description": "모든 Integration Event의 기본 스키마",
  "type": "object",

  "properties": {
    "event_id": {
      "type": "string",
      "description": "이벤트 고유 식별자"
    },
    "event_type": {
      "type": "string",
      "pattern": "^[a-z]+\\.[a-z_]+$",
      "description": "이벤트 타입 (domain.action)"
    },
    "version": {
      "type": "string",
      "pattern": "^v\\d+$",
      "description": "이벤트 버전"
    },
    "occurred_at": {
      "type": "string",
      "format": "date-time",
      "description": "이벤트 발생 시각 (UTC)"
    },
    "trace_id": {
      "type": "string",
      "description": "분산 추적 ID"
    },
    "client_no": {
      "type": "integer",
      "description": "클라이언트 번호"
    },
    "user_no": {
      "type": "integer",
      "description": "사용자 번호"
    },
    "payload": {
      "type": "object",
      "description": "이벤트 상세 데이터"
    }
  },

  "required": ["event_id", "event_type", "version", "occurred_at", "payload"]
}
```

### 7.3 스키마 검증

```java
// infrastructure/messaging/event/EventSchemaValidator.java
@Component
@Slf4j
public class EventSchemaValidator {

    private final Map<String, JsonSchema> schemas = new ConcurrentHashMap<>();
    private final JsonSchemaFactory schemaFactory;

    public EventSchemaValidator() {
        this.schemaFactory = JsonSchemaFactory.getInstance(SpecVersion.VersionFlag.V202012);
        loadSchemas();
    }

    private void loadSchemas() {
        try {
            PathMatchingResourcePatternResolver resolver =
                new PathMatchingResourcePatternResolver();
            Resource[] resources = resolver.getResources(
                "classpath:event-schemas/**/*.json"
            );

            for (Resource resource : resources) {
                if (resource.getFilename().startsWith("_")) continue;

                String content = StreamUtils.copyToString(
                    resource.getInputStream(), StandardCharsets.UTF_8
                );
                JsonSchema schema = schemaFactory.getSchema(content);
                String schemaId = extractSchemaId(resource.getFilename());
                schemas.put(schemaId, schema);

                log.info("Loaded event schema: {}", schemaId);
            }
        } catch (Exception e) {
            throw new IllegalStateException("Failed to load event schemas", e);
        }
    }

    public ValidationResult validate(IntegrationEvent event) {
        String schemaId = event.getEventType() + "." + event.getVersion();
        JsonSchema schema = schemas.get(schemaId);

        if (schema == null) {
            return ValidationResult.error("Schema not found: " + schemaId);
        }

        try {
            JsonNode eventNode = objectMapper.valueToTree(event);
            Set<ValidationMessage> errors = schema.validate(eventNode);

            if (errors.isEmpty()) {
                return ValidationResult.valid();
            } else {
                return ValidationResult.invalid(errors);
            }
        } catch (Exception e) {
            return ValidationResult.error(e.getMessage());
        }
    }
}
```

### 7.4 스키마 레지스트리 통합 (선택)

```yaml
# Confluent Schema Registry 대신 간단한 스키마 저장소 사용 가능
# S3 또는 Git 기반 스키마 관리

schema-registry:
  type: git  # git | s3 | http
  git:
    repository: https://github.com/platform/event-schemas.git
    branch: main
    sync-interval: 60s
  s3:
    bucket: platform-event-schemas
    region: ap-northeast-2
```

---

END OF DOCUMENT
