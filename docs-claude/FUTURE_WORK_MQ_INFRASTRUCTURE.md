# FUTURE_WORK_MQ_INFRASTRUCTURE.md

MQ 인프라 추후 개선 작업 목록

---

## 1. MQ SSL 활성화

### 현황
- 현재 RabbitMQ 연결이 평문 통신 (SSL 미적용)
- 개발/스테이징 환경에서는 허용 가능하나 프로덕션에서는 보안 취약점

### 기대효과
- 메시지 전송 구간 암호화로 데이터 탈취 방지
- MITM (Man-in-the-Middle) 공격 차단
- 금융권 보안 감사 기준 충족

### 설계

```yaml
# application-prod.yml
spring:
  rabbitmq:
    ssl:
      enabled: true
      key-store: classpath:ssl/rabbitmq-client.p12
      key-store-password: ${MQ_SSL_KEYSTORE_PASSWORD}
      trust-store: classpath:ssl/rabbitmq-truststore.jks
      trust-store-password: ${MQ_SSL_TRUSTSTORE_PASSWORD}
      algorithm: TLSv1.3
```

### 작업 범위
- [ ] RabbitMQ 서버 SSL 인증서 발급 및 설정
- [ ] Application/System 서버 클라이언트 인증서 설정
- [ ] AWS Parameter Store에 인증서 비밀번호 등록
- [ ] 환경별 설정 분리 (dev: false, prod: true)

---

## 2. DLQ 모니터링 알림

### 현황
- DLQ (Dead Letter Queue)에 실패 메시지가 쌓여도 감지 불가
- 7일 후 자동 삭제되어 장애 추적 어려움

### 기대효과
- 메시지 처리 실패 즉시 감지
- 장애 대응 시간 단축 (MTTR 감소)
- 실패 패턴 분석을 통한 사전 예방

### 설계

```java
@Component
@Profile("worker-mq")
@RequiredArgsConstructor
public class DlqMonitorService {

    private final RabbitAdmin rabbitAdmin;
    private final AlertService alertService;

    @Value("${dlq.monitor.threshold:10}")
    private int threshold;

    @Scheduled(fixedRate = 60000)  // 1분마다
    public void monitorDlq() {
        QueueInformation queueInfo = rabbitAdmin.getQueueInfo(
            MqConstants.QUEUE_DLQ_MESSAGE_SEND
        );

        if (queueInfo != null && queueInfo.getMessageCount() > threshold) {
            alertService.sendDlqAlert(
                MqConstants.QUEUE_DLQ_MESSAGE_SEND,
                queueInfo.getMessageCount()
            );
        }
    }
}
```

### 알림 채널
- Slack Webhook 연동
- 이메일 알림 (운영팀)
- Prometheus AlertManager (옵션)

### 작업 범위
- [ ] DlqMonitorService 구현
- [ ] AlertService 구현 (Slack/Email)
- [ ] 임계값 설정 (application.yml)
- [ ] Grafana 대시보드에 DLQ 메트릭 추가

---

## 3. Circuit Breaker (WiseT API)

### 현황
- WiseT API 장애 시 모든 요청이 타임아웃까지 대기
- 연쇄 장애 발생 가능 (Thread Pool 고갈)

### 기대효과
- 외부 API 장애 시 빠른 실패 (Fail Fast)
- 시스템 리소스 보호
- 자동 복구 시도 (Half-Open 상태)

### 설계

```java
@Component
@Profile("worker-mq")
public class WiseTClient {

    @CircuitBreaker(name = "wiset", fallbackMethod = "fallbackSend")
    @Retry(name = "wiset")
    public WiseTResponse sendSms(MessageSendRequest request) {
        return restTemplate.postForObject(url, request, WiseTResponse.class);
    }

    public WiseTResponse fallbackSend(MessageSendRequest request, Throwable t) {
        log.error("WiseT API unavailable, circuit open: {}", t.getMessage());
        throw new ExternalApiUnavailableException("WiseT service unavailable");
    }
}
```

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      wiset:
        slidingWindowSize: 10
        failureRateThreshold: 50        # 50% 실패 시 OPEN
        waitDurationInOpenState: 30s    # 30초 후 HALF_OPEN
        permittedNumberOfCallsInHalfOpenState: 3

  retry:
    instances:
      wiset:
        maxAttempts: 3
        waitDuration: 1s
        exponentialBackoffMultiplier: 2  # 1s, 2s, 4s
        retryExceptions:
          - java.net.SocketTimeoutException
          - org.springframework.web.client.ResourceAccessException
```

### 상태 전이
```
CLOSED (정상) → 50% 실패 → OPEN (차단)
    ↑                          ↓
    └── 3회 성공 ← HALF_OPEN (30초 후)
```

### 작업 범위
- [ ] Resilience4j 의존성 추가 (build.gradle)
- [ ] WiseTClient에 @CircuitBreaker, @Retry 적용
- [ ] fallback 메서드 구현
- [ ] Circuit 상태 메트릭 수집 (Actuator)
- [ ] Grafana 대시보드에 Circuit 상태 추가

---

## 4. MQ 메트릭 수집

### 현황
- MQ 처리량, 지연 시간 등 가시성 부재
- 장애 발생 시 원인 분석 어려움

### 기대효과
- 실시간 처리량 모니터링
- 성능 병목 구간 식별
- 용량 계획 수립 근거 확보
- SLA 준수 여부 측정

### 설계

```java
@Component
@RequiredArgsConstructor
public class MqMetricsService {

    private final MeterRegistry meterRegistry;

    public void recordPublished(String messageType) {
        meterRegistry.counter("mq.message.published",
            "type", messageType
        ).increment();
    }

    public void recordFailed(String messageType, String reason) {
        meterRegistry.counter("mq.message.failed",
            "type", messageType,
            "reason", reason
        ).increment();
    }

    public void recordProcessDuration(String messageType, long durationMs) {
        meterRegistry.timer("mq.message.process.duration",
            "type", messageType
        ).record(Duration.ofMillis(durationMs));
    }

    public void recordExternalApiDuration(String api, long durationMs) {
        meterRegistry.timer("external.api.duration",
            "api", api
        ).record(Duration.ofMillis(durationMs));
    }
}
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active}
```

### 수집 메트릭

| 메트릭 | 설명 | 타입 |
|--------|------|------|
| `mq.message.published` | 발행된 메시지 수 | Counter |
| `mq.message.failed` | 실패한 메시지 수 | Counter |
| `mq.message.process.duration` | 처리 소요 시간 | Timer |
| `rabbitmq_queue_messages` | 큐 메시지 수 | Gauge |
| `rabbitmq_queue_messages_ready` | 대기 메시지 수 | Gauge |
| `external.api.duration` | 외부 API 응답 시간 | Timer |
| `wiset.api.success.rate` | WiseT API 성공률 | Gauge |

### Grafana 대시보드 구성
```
┌─────────────────────────────────────────────────────────┐
│ MQ Dashboard                                            │
├─────────────────┬─────────────────┬─────────────────────┤
│ 발행 메시지/분   │ 실패 메시지/분   │ 평균 처리 시간      │
│    [Graph]      │    [Graph]      │    [Graph]          │
├─────────────────┼─────────────────┼─────────────────────┤
│ 큐 적체량        │ DLQ 메시지 수    │ WiseT API 응답 시간 │
│    [Gauge]      │    [Stat]       │    [Graph]          │
└─────────────────┴─────────────────┴─────────────────────┘
```

### 작업 범위
- [ ] Micrometer 의존성 확인 (Spring Boot Actuator 포함)
- [ ] MqMetricsService 구현
- [ ] MessageSendListener에 메트릭 수집 로직 추가
- [ ] WiseTClient에 응답 시간 메트릭 추가
- [ ] Prometheus scrape config 설정
- [ ] Grafana 대시보드 JSON 작성

---

## 우선순위

| 순위 | 항목 | 중요도 | 긴급도 | 비고 |
|------|------|--------|--------|------|
| 1 | MQ SSL | 높음 | 중간 | 프로덕션 배포 전 필수 |
| 2 | DLQ 모니터링 | 높음 | 높음 | 장애 감지 필수 |
| 3 | 메트릭 수집 | 중간 | 중간 | 운영 가시성 확보 |
| 4 | Circuit Breaker | 중간 | 낮음 | 안정성 강화 |

---

END OF FILE
