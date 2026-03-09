# 06. 리소스 보호 전략

```
Document ID    : MQ-SPEC-001-06
Version        : 1.0.0
Classification : Internal / Confidential
```

---

## 1. Connection 제한 전략

### 1.1 Connection Limit 계층

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Connection Limit Hierarchy                           │
│                                                                          │
│  Level 1: Cluster Total                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Total Connections: 10,000                                       │    │
│  │  (Production Cluster 전체)                                       │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                              │                                           │
│                              ▼                                           │
│  Level 2: Node Limit                                                     │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐               │
│  │   Node-0       │ │   Node-1       │ │   Node-2       │               │
│  │   Max: 4,000   │ │   Max: 4,000   │ │   Max: 4,000   │               │
│  └────────────────┘ └────────────────┘ └────────────────┘               │
│                              │                                           │
│                              ▼                                           │
│  Level 3: vhost Limit                                                    │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │  vhost: /eventflow                                              │     │
│  │  Max Connections: 500                                           │     │
│  │                                                                 │     │
│  │  vhost: /billing                                                │     │
│  │  Max Connections: 300                                           │     │
│  └────────────────────────────────────────────────────────────────┘     │
│                              │                                           │
│                              ▼                                           │
│  Level 4: User Limit                                                     │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │  ef-producer-application: Max 100 connections                   │     │
│  │  ef-consumer-system: Max 200 connections                        │     │
│  │  ef-consumer-batch: Max 50 connections                          │     │
│  └────────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Connection Limit 설정

```erlang
# rabbitmq.conf - 노드 레벨 Connection Limit

# 최대 연결 수 (기본값: infinity)
# 노드당 4,000으로 제한
# tcp_listen_options.backlog = 128
# 연결 핸드셰이크 타임아웃 (10초)
handshake_timeout = 10000

# 하트비트 간격 (60초)
heartbeat = 60
```

```bash
# vhost 레벨 Connection Limit (RabbitMQ 3.12+)
rabbitmqctl set_vhost_limits -p /eventflow \
  '{"max-connections": 500}'

rabbitmqctl set_vhost_limits -p /billing \
  '{"max-connections": 300}'

# User 레벨 Connection Limit
rabbitmqctl set_user_limits ef-producer-application \
  '{"max-connections": 100}'

rabbitmqctl set_user_limits ef-consumer-system \
  '{"max-connections": 200}'

rabbitmqctl set_user_limits ef-consumer-batch \
  '{"max-connections": 50}'
```

### 1.3 Connection 모니터링

```yaml
# Prometheus Alert Rules
groups:
  - name: rabbitmq_connections
    rules:
      - alert: RabbitMQHighConnectionCount
        expr: rabbitmq_connections > 3000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High connection count on {{ $labels.instance }}"

      - alert: RabbitMQConnectionsNearLimit
        expr: rabbitmq_connections / rabbitmq_connection_limit > 0.8
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Connection count near limit"

      - alert: RabbitMQVhostConnectionsHigh
        expr: rabbitmq_vhost_connections > 400
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "vhost {{ $labels.vhost }} connections high"
```

---

## 2. Channel 제한 전략

### 2.1 Channel Limit 설정

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Channel Management                               │
│                                                                          │
│  Connection 1                                                            │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │  Channel-1   Channel-2   Channel-3   ... Channel-N               │    │
│  │  (Producer)  (Producer)  (Consumer)      (Max: 128)              │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Best Practices:                                                         │
│  - 1 Thread = 1 Channel                                                  │
│  - Connection당 최대 128 Channel 권장                                    │
│  - Channel 재사용 (Connection Pool처럼 Channel Pool 사용)                │
│  - 단기 작업에 Channel 생성/삭제 반복 금지                               │
└─────────────────────────────────────────────────────────────────────────┘
```

```erlang
# rabbitmq.conf - Channel 설정

# Connection당 최대 Channel 수
channel_max = 128

# Channel 작업 타임아웃
channel_operation_timeout = 15000
```

### 2.2 Application Channel Pool 설정 (Spring AMQP)

```yaml
# application.yml
spring:
  rabbitmq:
    cache:
      connection:
        mode: CHANNEL  # CONNECTION or CHANNEL
        size: 10       # Connection Pool Size
      channel:
        size: 25       # Channel Pool Size per Connection
        checkout-timeout: 5000  # ms

    # Publisher Confirms (신뢰성)
    publisher-confirm-type: correlated
    publisher-returns: true

    # Connection 설정
    connection-timeout: 10000
    requested-heartbeat: 60
```

```java
// RabbitConfig.java - 명시적 Channel Pool 설정
@Configuration
public class RabbitMqConfig {

    @Bean
    public CachingConnectionFactory connectionFactory() {
        CachingConnectionFactory factory = new CachingConnectionFactory();
        factory.setHost("rabbitmq.platform-mq");
        factory.setPort(5671);
        factory.setVirtualHost("/eventflow");

        // Connection Pool
        factory.setCacheMode(CachingConnectionFactory.CacheMode.CHANNEL);
        factory.setConnectionCacheSize(10);

        // Channel Pool
        factory.setChannelCacheSize(25);
        factory.setChannelCheckoutTimeout(5000);

        // Publisher Confirms
        factory.setPublisherConfirmType(
            CachingConnectionFactory.ConfirmType.CORRELATED
        );
        factory.setPublisherReturns(true);

        return factory;
    }
}
```

---

## 3. Queue Max-Length 전략

### 3.1 Max-Length 정책 유형

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Queue Max-Length Strategies                         │
│                                                                          │
│  Strategy 1: drop-head (기본값)                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │  [Msg1] [Msg2] [Msg3] ... [Msg99999] [Msg100000]                │    │
│  │    ↑                                                             │    │
│  │   DROP (가장 오래된 메시지 삭제)                                 │    │
│  │                                                                  │    │
│  │  + New Message = [Msg2] [Msg3] ... [Msg100000] [NewMsg]         │    │
│  │                                                                  │    │
│  │  ※ 메시지 유실 가능, Producer는 모름                            │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Strategy 2: reject-publish (권장)                                       │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │  [Msg1] [Msg2] [Msg3] ... [Msg99999] [Msg100000]                │    │
│  │                                               ↑                  │    │
│  │                                         FULL! REJECT             │    │
│  │                                                                  │    │
│  │  + New Message = NACK (Producer에게 알림)                        │    │
│  │                                                                  │    │
│  │  ※ 메시지 유실 방지, Producer가 재시도 가능                     │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Strategy 3: reject-publish-dlx (권장 + DLQ)                             │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │  Queue Full → 새 메시지를 DLQ로 라우팅                           │    │
│  │                                                                  │    │
│  │  ※ 메시지 보존, 나중에 재처리 가능                              │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Queue 유형별 Max-Length 권장값

| Queue 유형 | Max-Length | Max-Length-Bytes | Overflow 정책 |
|-----------|------------|------------------|---------------|
| 일반 업무 큐 | 100,000 | 500MB | reject-publish |
| 알림 큐 | 200,000 | 1GB | reject-publish |
| DLQ | 50,000 | 200MB | drop-head |
| 모니터링 큐 | 10,000 | 50MB | drop-head |
| 배치 큐 | 500,000 | 2GB | reject-publish |

### 3.3 Policy 설정 (Terraform)

```hcl
# 일반 큐 정책
resource "rabbitmq_policy" "ef_queue_limits_default" {
  name  = "ef-queue-limits-default"
  vhost = "/eventflow"

  policy {
    pattern  = "^ef\\.queue\\.(?!.*\\.dlq$).*"
    priority = 0
    apply_to = "queues"

    definition = {
      "x-max-length"       = 100000
      "x-max-length-bytes" = 524288000  # 500MB
      "x-overflow"         = "reject-publish"
    }
  }
}

# DLQ 정책
resource "rabbitmq_policy" "ef_dlq_limits" {
  name  = "ef-dlq-limits"
  vhost = "/eventflow"

  policy {
    pattern  = "^ef\\.queue\\..*\\.dlq$"
    priority = 1
    apply_to = "queues"

    definition = {
      "x-max-length"       = 50000
      "x-max-length-bytes" = 209715200  # 200MB
      "x-overflow"         = "drop-head"  # 오래된 것부터 삭제
    }
  }
}

# 알림 큐 정책 (더 큰 용량)
resource "rabbitmq_policy" "ef_notification_limits" {
  name  = "ef-notification-limits"
  vhost = "/eventflow"

  policy {
    pattern  = "^ef\\.queue\\.notification\\..*"
    priority = 2
    apply_to = "queues"

    definition = {
      "x-max-length"       = 200000
      "x-max-length-bytes" = 1073741824  # 1GB
      "x-overflow"         = "reject-publish"
    }
  }
}
```

---

## 4. TTL 전략

### 4.1 TTL 유형

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           TTL Types                                      │
│                                                                          │
│  Type 1: Message TTL (x-message-ttl)                                     │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Queue 설정에서 지정                                             │    │
│  │  모든 메시지에 동일하게 적용                                     │    │
│  │                                                                  │    │
│  │  arguments: {"x-message-ttl": 86400000}  # 24시간                │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Type 2: Per-Message TTL (expiration)                                    │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  메시지 발행 시 개별 지정                                        │    │
│  │  메시지마다 다른 TTL 가능                                        │    │
│  │                                                                  │    │
│  │  properties.expiration = "60000"  # 60초                         │    │
│  │                                                                  │    │
│  │  ※ Queue TTL과 Per-Message TTL 중 짧은 것 적용                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Type 3: Queue TTL (x-expires)                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  사용되지 않는 Queue 자동 삭제                                   │    │
│  │  Consumer도 없고 메시지도 없을 때                                │    │
│  │                                                                  │    │
│  │  arguments: {"x-expires": 3600000}  # 1시간 미사용 시 삭제       │    │
│  │                                                                  │    │
│  │  ※ 임시 큐에만 사용, 운영 큐에는 설정하지 않음                   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 TTL 권장값

| Queue 유형 | Message TTL | 근거 |
|-----------|-------------|------|
| 일반 업무 큐 | 24시간 (86,400,000ms) | 1일 내 처리 필요 |
| 알림 큐 | 24시간 | 시의성 있는 알림 |
| 배치 큐 | 72시간 (259,200,000ms) | 주말 포함 처리 |
| DLQ | 7일 (604,800,000ms) | 분석 및 재처리 여유 |
| 임시/테스트 큐 | 1시간 (3,600,000ms) | 자동 정리 |

### 4.3 TTL 만료 처리 흐름

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      TTL Expiration Flow                                 │
│                                                                          │
│                    ┌──────────────────┐                                  │
│                    │ Message in Queue │                                  │
│                    │  (TTL: 24시간)   │                                  │
│                    └────────┬─────────┘                                  │
│                             │                                            │
│                             │ TTL 경과                                   │
│                             ▼                                            │
│             ┌───────────────────────────────────┐                        │
│             │     DLX 설정 여부?                │                        │
│             └───────────┬───────────────────────┘                        │
│                   Yes   │        No                                      │
│               ┌─────────┴──────────┐                                     │
│               ▼                    ▼                                     │
│     ┌─────────────────┐   ┌─────────────────┐                           │
│     │  DLQ로 이동     │   │  메시지 폐기    │                           │
│     │                 │   │  (영구 삭제)    │                           │
│     │ x-death 헤더:   │   │                 │                           │
│     │ - reason:expired│   │                 │                           │
│     │ - queue:원래큐  │   │                 │                           │
│     │ - time:만료시각 │   │                 │                           │
│     └─────────────────┘   └─────────────────┘                           │
│             │                                                            │
│             ▼                                                            │
│     ┌─────────────────┐                                                 │
│     │ DLQ에서 분석    │                                                 │
│     │ - 왜 처리 안됨? │                                                 │
│     │ - Consumer 문제?│                                                 │
│     │ - 시스템 장애?  │                                                 │
│     └─────────────────┘                                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Memory Watermark 전략

### 5.1 Memory 관리 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     RabbitMQ Memory Management                           │
│                                                                          │
│  Total System Memory: 16GB                                               │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │████████████████████████████████████████████░░░░░░░░░░░░░░░░░░░░│    │
│  │                                            │                    │    │
│  │◄────────── RabbitMQ Available ────────────►│◄── OS/Other ─────►│    │
│  │              (60% = 9.6GB)                 │    (40% = 6.4GB)  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  vm_memory_high_watermark.relative = 0.6                                 │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Memory Usage Levels                                             │    │
│  │                                                                  │    │
│  │  0%     40%        60%        70%         100%                  │    │
│  │  ├───────┼──────────┼──────────┼───────────┤                    │    │
│  │  │       │          │          │           │                    │    │
│  │  │ NORMAL│  WARNING │  PAGING  │   BLOCK   │                    │    │
│  │  │       │          │          │           │                    │    │
│  │  └───────┴──────────┴──────────┴───────────┘                    │    │
│  │                     ↑          ↑                                │    │
│  │              watermark   paging_ratio                            │    │
│  │                (0.6)       (0.7)                                 │    │
│  │                                                                  │    │
│  │  PAGING: 메시지를 디스크로 이동 시작                             │    │
│  │  BLOCK: 모든 Publisher 연결 차단                                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Memory 설정

```erlang
# rabbitmq.conf - Memory 설정

# 전체 시스템 메모리의 60%까지 사용
vm_memory_high_watermark.relative = 0.6

# 또는 절대값으로 설정
# vm_memory_high_watermark.absolute = 8GB

# Paging 시작 비율 (watermark의 70% 도달 시)
# 즉, 전체의 42% 사용 시 paging 시작
vm_memory_high_watermark_paging_ratio = 0.7

# Memory 계산 방식 (allocated가 더 정확)
vm_memory_calculation_strategy = allocated
```

### 5.3 Memory 알람 설정

```yaml
# Prometheus Alert Rules
groups:
  - name: rabbitmq_memory
    rules:
      - alert: RabbitMQMemoryWarning
        expr: |
          rabbitmq_process_resident_memory_bytes /
          rabbitmq_resident_memory_limit_bytes > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "RabbitMQ memory usage > 50%"

      - alert: RabbitMQMemoryHigh
        expr: |
          rabbitmq_process_resident_memory_bytes /
          rabbitmq_resident_memory_limit_bytes > 0.7
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "RabbitMQ memory usage > 70% - paging active"

      - alert: RabbitMQMemoryAlarm
        expr: rabbitmq_alarms_memory_used_watermark == 1
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "RabbitMQ memory alarm triggered - publishers blocked"
```

---

## 6. Backpressure 전략

### 6.1 Backpressure 메커니즘

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Backpressure Mechanisms                             │
│                                                                          │
│  Level 1: Connection-level Flow Control                                  │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │  Producer              RabbitMQ                                  │    │
│  │  ┌──────────┐         ┌──────────┐                              │    │
│  │  │ Channel  │◄────────│ Flow     │                              │    │
│  │  │          │  BLOCK  │ Control  │                              │    │
│  │  │ blocked  │         │ Active   │                              │    │
│  │  └──────────┘         └──────────┘                              │    │
│  │                                                                  │    │
│  │  ※ Memory/Disk 알람 시 모든 Publisher 차단                      │    │
│  │  ※ channel.flow 메서드로 알림                                   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Level 2: Credit-based Flow Control (Consumer)                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │  RabbitMQ              Consumer                                  │    │
│  │  ┌──────────┐         ┌──────────┐                              │    │
│  │  │ Queue    │────────►│ Channel  │                              │    │
│  │  │          │ prefetch│          │                              │    │
│  │  │ waiting  │◄────────│ ACK      │                              │    │
│  │  └──────────┘ credit  └──────────┘                              │    │
│  │                                                                  │    │
│  │  ※ prefetch_count만큼만 전송                                    │    │
│  │  ※ ACK 받으면 추가 메시지 전송                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Level 3: Queue Max-Length (Producer)                                    │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │  Producer              Queue                                     │    │
│  │  ┌──────────┐         ┌──────────┐                              │    │
│  │  │ publish  │────X────│ FULL     │                              │    │
│  │  │          │  NACK   │          │                              │    │
│  │  │ retry    │◄────────│ overflow │                              │    │
│  │  └──────────┘         │ reject   │                              │    │
│  │                       └──────────┘                              │    │
│  │                                                                  │    │
│  │  ※ reject-publish 정책으로 Producer에게 알림                    │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Publisher Confirms 설정

```java
// RabbitMQ Publisher with Backpressure Handling
@Component
@RequiredArgsConstructor
@Slf4j
public class ReliableMessagePublisher {

    private final RabbitTemplate rabbitTemplate;
    private final MeterRegistry meterRegistry;

    // 동시 발행 제한
    private final Semaphore publishSemaphore = new Semaphore(1000);

    // 재시도 설정
    private static final int MAX_RETRIES = 3;
    private static final long RETRY_DELAY_MS = 1000;

    @PostConstruct
    public void setupConfirmCallback() {
        rabbitTemplate.setConfirmCallback((correlation, ack, cause) -> {
            if (ack) {
                publishSemaphore.release();
                meterRegistry.counter("mq.publish.confirmed").increment();
            } else {
                log.warn("Message not confirmed: {}", cause);
                meterRegistry.counter("mq.publish.nacked").increment();
                // 재시도 로직
                handlePublishFailure(correlation);
            }
        });

        rabbitTemplate.setReturnsCallback(returned -> {
            log.error("Message returned: exchange={}, routingKey={}, replyText={}",
                returned.getExchange(),
                returned.getRoutingKey(),
                returned.getReplyText()
            );
            meterRegistry.counter("mq.publish.returned").increment();
        });
    }

    public void publish(String exchange, String routingKey, Object message) {
        int retries = 0;

        while (retries < MAX_RETRIES) {
            try {
                // Backpressure: 동시 발행 제한
                if (!publishSemaphore.tryAcquire(5, TimeUnit.SECONDS)) {
                    throw new BackpressureException(
                        "Too many pending publishes"
                    );
                }

                CorrelationData correlation = new CorrelationData(
                    UUID.randomUUID().toString()
                );

                rabbitTemplate.convertAndSend(
                    exchange,
                    routingKey,
                    message,
                    correlation
                );

                return;

            } catch (AmqpException e) {
                retries++;
                log.warn("Publish failed, retry {}/{}: {}",
                    retries, MAX_RETRIES, e.getMessage());

                if (retries >= MAX_RETRIES) {
                    throw new MessagePublishException(
                        "Failed after " + MAX_RETRIES + " retries", e
                    );
                }

                try {
                    Thread.sleep(RETRY_DELAY_MS * retries);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new MessagePublishException("Interrupted", ie);
                }
            }
        }
    }
}
```

### 6.3 Consumer Prefetch 설정

```java
// Consumer Prefetch 설정
@Configuration
public class ConsumerConfig {

    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
        ConnectionFactory connectionFactory
    ) {
        SimpleRabbitListenerContainerFactory factory =
            new SimpleRabbitListenerContainerFactory();

        factory.setConnectionFactory(connectionFactory);

        // Prefetch Count: Consumer가 한 번에 가져올 메시지 수
        // 처리 시간에 따라 조정
        factory.setPrefetchCount(10);

        // Concurrent Consumers
        factory.setConcurrentConsumers(5);
        factory.setMaxConcurrentConsumers(20);

        // ACK Mode
        factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);

        // 에러 처리
        factory.setDefaultRequeueRejected(false);

        return factory;
    }
}
```

---

## 7. Disk 보호 전략

### 7.1 Disk Free Limit 설정

```erlang
# rabbitmq.conf - Disk 설정

# 절대값으로 설정 (5GB)
disk_free_limit.absolute = 5GB

# 또는 상대값 (메모리의 1.5배)
# disk_free_limit.relative = 1.5

# Disk 알람 발생 시:
# - 모든 Producer 차단
# - 메시지 Paging 중단
# - 클러스터 재구성 차단
```

### 7.2 Disk 모니터링 알람

```yaml
# Prometheus Alert Rules
groups:
  - name: rabbitmq_disk
    rules:
      - alert: RabbitMQDiskSpaceWarning
        expr: |
          rabbitmq_disk_space_available_bytes /
          rabbitmq_disk_space_available_limit_bytes < 2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "RabbitMQ disk space < 2x limit"

      - alert: RabbitMQDiskAlarm
        expr: rabbitmq_alarms_free_disk_space_watermark == 1
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "RabbitMQ disk alarm - publishers blocked"

      - alert: RabbitMQDiskSpaceLow
        expr: |
          node_filesystem_avail_bytes{mountpoint="/var/lib/rabbitmq"} /
          node_filesystem_size_bytes{mountpoint="/var/lib/rabbitmq"} < 0.2
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "RabbitMQ data disk < 20% free"
```

---

## 8. 리소스 보호 종합 체크리스트

### 8.1 프로덕션 설정 검증

| 항목 | 설정값 | 검증 명령 |
|------|--------|----------|
| vhost Connection Limit | 500 | `rabbitmqctl list_vhost_limits` |
| User Connection Limit | 100-200 | `rabbitmqctl list_user_limits` |
| Channel per Connection | 128 | `rabbitmq.conf: channel_max` |
| Memory Watermark | 60% | `rabbitmqctl environment` |
| Disk Free Limit | 5GB | `rabbitmqctl environment` |
| Queue Max Length | 100,000 | Policy 확인 |
| Message TTL | 24h | Policy 확인 |
| Prefetch Count | 10-50 | Application 설정 |

### 8.2 용량 계획 가이드

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Capacity Planning Guide                             │
│                                                                          │
│  메시지 크기 평균: 1KB                                                   │
│  일일 메시지 수: 10,000,000                                              │
│  피크 시간 배수: 3x                                                      │
│                                                                          │
│  계산:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  피크 메시지/초 = 10,000,000 / 86,400 * 3 ≈ 347 msg/s           │    │
│  │  피크 처리량 = 347 * 1KB = 347 KB/s                              │    │
│  │                                                                  │    │
│  │  Queue 적체 허용 시간: 1시간                                     │    │
│  │  최대 적체량 = 347 * 3600 = 1,249,200 messages                   │    │
│  │  최대 적체 용량 = 1.25GB                                         │    │
│  │                                                                  │    │
│  │  권장 Queue Max Length: 2,000,000                                │    │
│  │  권장 Queue Max Length Bytes: 2GB                                │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Memory 계산:                                                            │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  메시지 오버헤드 (Quorum Queue): ~150 bytes                      │    │
│  │  Queue 1개당 메모리 = 2,000,000 * (1KB + 150B) ≈ 2.3GB          │    │
│  │                                                                  │    │
│  │  활성 Queue 10개 예상                                            │    │
│  │  필요 메모리 = 2.3GB * 10 = 23GB                                 │    │
│  │                                                                  │    │
│  │  권장 노드 메모리: 16GB * 3 = 48GB (클러스터)                    │    │
│  │  Watermark 60% = 28.8GB 사용 가능                                │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

END OF DOCUMENT
