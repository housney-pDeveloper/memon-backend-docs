# INFRASTRUCTURE.md

MQ 리소스, 동기화, 미래 로드맵(WebSocket, Circuit Breaker, 모니터링)을 통합 정의한다.

---

## 1. MQ 리소스 레지스트리

RabbitMQ Exchange, Queue, Binding 리소스 등록 현황.
Exchange 또는 Queue 추가/변경 시 반드시 이 문서를 함께 업데이트해야 한다.

### 1.1 vhost

```
/pf-ef
```

### 1.2 Exchanges

| Exchange 이름 | Type | Durable | 용도 |
|---------------|------|---------|------|
| `ex.pf.ef.log.topic` | Topic | Yes | 로그 전용 |
| `ex.pf.ef.message.topic` | Topic | Yes | 메시지 발송 전용 |
| `ex.pf.ef.notification.topic` | Topic | Yes | 알림(SSE) 전용 |
| `ex.pf.ef.commoncode.topic` | Topic | Yes | 공통코드 변경 이벤트 |
| `ex.pf.ef.dlx` | Topic | Yes | Dead Letter Exchange |

네이밍: `ex.{플랫폼}.{프로젝트}.{도메인}.{타입}`

### 1.3 Queues

| Queue 이름 | Type | DLX | TTL | 용도 |
|------------|------|-----|-----|------|
| `q.pf.ef.log.write` | Quorum | - | - | 로그 기록 |
| `q.pf.ef.message.send` | Quorum | `ex.pf.ef.dlx` | 24h | 메시지 발송 |
| `q.pf.ef.dlq.message.send` | Quorum | - | 7d | 메시지 발송 DLQ |
| `q.pf.ef.notification` | Quorum | `ex.pf.ef.dlx` | 24h | 알림 이벤트 |
| `q.pf.ef.dlq.notification` | Quorum | - | 7d | 알림 이벤트 DLQ |
| `q.pf.ef.commoncode.changed` | Quorum | `ex.pf.ef.dlx` | 24h | 공통코드 변경 |
| `q.pf.ef.dlq.commoncode.changed` | Quorum | - | 7d | 공통코드 변경 DLQ |

네이밍: `q.{플랫폼}.{프로젝트}.{도메인}.{액션}` / DLQ: `q.{플랫폼}.{프로젝트}.dlq.{도메인}.{액션}`

Queue 속성 기준:

| 속성 | 기본값 |
|------|--------|
| Type | Quorum (고가용성) |
| TTL (일반) | 24시간 (86,400,000ms) |
| TTL (DLQ) | 7일 (604,800,000ms) |
| Max Length | 50,000 (필요 시 조정) |
| Overflow | reject-publish |

### 1.4 Bindings

| Exchange | Routing Key | Queue |
|----------|-------------|-------|
| `ex.pf.ef.log.topic` | `log.access.v1` | `q.pf.ef.log.write` |
| `ex.pf.ef.log.topic` | `log.error.v1` | `q.pf.ef.log.write` |
| `ex.pf.ef.log.topic` | `log.audit.v1` | `q.pf.ef.log.write` |
| `ex.pf.ef.message.topic` | `message.send.v1` | `q.pf.ef.message.send` |
| `ex.pf.ef.dlx` | `message.send.dlq` | `q.pf.ef.dlq.message.send` |
| `ex.pf.ef.notification.topic` | `notification.event.v1` | `q.pf.ef.notification` |
| `ex.pf.ef.dlx` | `notification.dlq` | `q.pf.ef.dlq.notification` |
| `ex.pf.ef.commoncode.topic` | `commoncode.changed.v1` | `q.pf.ef.commoncode.changed` |
| `ex.pf.ef.dlx` | `commoncode.changed.dlq` | `q.pf.ef.dlq.commoncode.changed` |

Routing Key 규칙: `{도메인}.{액션}.{버전}` / DLQ: `{도메인}.{액션}.dlq`

---

## 2. 메시지 흐름도

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              RabbitMQ (/pf-ef)                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ex.pf.ef.log.topic                                                        │
│    ├── log.access.v1 ──┐                                                   │
│    ├── log.error.v1 ───┼──> q.pf.ef.log.write → [worker-log]              │
│    └── log.audit.v1 ───┘                                                   │
│                                                                             │
│  ex.pf.ef.message.topic                                                    │
│    └── message.send.v1 ──> q.pf.ef.message.send → [worker-message]        │
│                              │ (NACK/Reject)                               │
│                              ▼                                             │
│    ex.pf.ef.dlx → message.send.dlq → q.pf.ef.dlq.message.send            │
│                                                                             │
│  ex.pf.ef.notification.topic                                               │
│    └── notification.event.v1 ──> q.pf.ef.notification → [api]             │
│                                    │ (NACK/Reject)                         │
│                                    ▼                                       │
│    ex.pf.ef.dlx → notification.dlq → q.pf.ef.dlq.notification            │
│                                                                             │
│  ex.pf.ef.commoncode.topic                                                 │
│    └── commoncode.changed.v1 ──> q.pf.ef.commoncode.changed → [worker-mq]│
│                                    │ (NACK/Reject)                         │
│                                    ▼                                       │
│    ex.pf.ef.dlx → commoncode.changed.dlq → q.pf.ef.dlq.commoncode.changed│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Producer / Consumer 매핑

**Producers (Application Server):**

| Producer | Exchange | Routing Key |
|----------|----------|-------------|
| `CommonCodeEventPublisher` | `ex.pf.ef.commoncode.topic` | `commoncode.changed.v1` |
| `MessageSendEventPublisher` | `ex.pf.ef.message.topic` | `message.send.v1` |
| `OutboxPollingPublisher` | (Outbox 테이블 기반) | - |
| `ImmediateOutboxPublisher` | (즉시 발행) | - |

**Consumers (System Server):**

| Consumer | Queue | Profile |
|----------|-------|---------|
| `CommonCodeEventListener` | `q.pf.ef.commoncode.changed` | worker-mq |
| `LogMessageListener` | `q.pf.ef.log.write` | worker-mq |
| `MessageSendListener` | `q.pf.ef.message.send` | worker-mq |
| `NotificationEventListener` | `q.pf.ef.notification` | worker-mq |
| `DeadLetterListener` | DLQ 처리 | worker-mq |

---

## 4. 서버 역할 분담

```
┌─────────────────────────────┐     ┌─────────────────────────────┐
│     Application Server      │     │       System Server         │
├─────────────────────────────┤     ├─────────────────────────────┤
│  역할: Publisher 전용        │     │  역할: Consumer + 리소스 관리 │
│  ✅ 메시지 발행              │     │  ✅ 메시지 소비              │
│  ✅ Exchange/Routing 상수   │     │  ✅ Exchange/Queue Bean 정의 │
│  ❌ Exchange/Queue Bean 없음│     │  ✅ 리소스 동기화            │
│  ❌ Consumer 없음           │     │  ✅ auto-declare            │
└─────────────────────────────┘     └─────────────────────────────┘
```

| 원칙 | 설명 |
|------|------|
| Single Source of Truth | System 서버의 Bean 정의가 유일한 리소스 정의 |
| Publisher는 상수만 | Application은 Exchange/Routing Key 상수만 참조 |
| Consumer는 System만 | @RabbitListener는 System 서버에서만 사용 |
| 동기화는 worker-mq만 | 리소스 삭제/재생성은 worker-mq 프로필에서만 수행 |

---

## 5. 코드 위치

**System Server (리소스 마스터):**

| 파일 | 설명 |
|------|------|
| `MqConstants.java` | Exchange/Queue/Routing Key 상수 |
| `MqResourceConfig.java` | Exchange/Queue/Binding Bean 정의 |
| `MqConsumerConfig.java` | Consumer 설정 |
| `MqResourceSynchronizer.java` | 리소스 동기화 로직 |
| `RabbitMqManagementClient.java` | Management API 클라이언트 |

**Application Server (Publisher 전용):**

| 파일 | 설명 |
|------|------|
| `RabbitMqConfig.java` | Exchange/Routing Key 상수 + RabbitTemplate |
| `CommonCodeEventPublisher.java` | 공통코드 변경 이벤트 발행 |
| `MessageSendEventPublisher.java` | 메시지 발송 이벤트 발행 |
| `OutboxPollingPublisher.java` | Outbox 폴링 발행 |
| `ImmediateOutboxPublisher.java` | 즉시 Outbox 발행 |

---

## 6. MQ 리소스 동기화 (Code as Infrastructure)

### 6.1 동작 방식

- Bean에 있고 RabbitMQ에 없음 → 생성 (auto-declare)
- Bean에 없고 RabbitMQ에 있음 → 삭제
- Arguments 불일치 → 삭제 후 재생성

### 6.2 설정

```yaml
spring:
  rabbitmq:
    host: ${rabbitmq.host}
    port: ${rabbitmq.port}
    username: ${rabbitmq.username.consumer}
    password: ${rabbitmq.password.consumer}
    virtual-host: /pf-ef
    management-url: ${rabbitmq.management-url}

mq:
  sync:
    enabled: true
    dry-run: false
    delete-non-empty-queues: false
    recreate-on-mismatch: true
    managed-exchange-prefixes:
      - "ex.pf.ef."
    managed-queue-prefixes:
      - "q.pf.ef."
```

### 6.3 환경별 설정

| 환경 | enabled | dry-run | 동작 |
|------|---------|---------|------|
| local | true | true | 로그만 출력 |
| develop | true | false | 실제 동기화 |
| staging | true | false | 실제 동기화 |
| production | false | - | 비활성화 |

### 6.4 안전장치

- prefix 필터: `ex.pf.ef.*`, `q.pf.ef.*` 만 관리
- `delete-non-empty-queues: false`: 메시지 있으면 스킵
- `dry-run: true`: 실제 삭제 없이 로그만
- production 비활성화

---

## 7. MQ User 정책

### 7.1 환경별 User 전략

| 환경 | MQ 유형 | User 전략 |
|------|---------|-----------|
| local/develop | Docker | 단일 계정 (pf_ef_admin) |
| staging/production | Amazon MQ | Publisher/Consumer 분리 |

### 7.2 Amazon MQ User 구성

| User | 용도 | 사용 서버 |
|------|------|----------|
| `pf_ef_publisher` | Exchange에 메시지 발행 | Application |
| `pf_ef_consumer` | Queue에서 메시지 소비 | System |
| `pf_ef_admin` | 리소스 생성/삭제/관리 | CI/CD, 운영 |

### 7.3 Parameter Store 구성

| Parameter | local/develop | staging/production |
|-----------|---------------|-------------------|
| `rabbitmq.host` | localhost | Amazon MQ Endpoint |
| `rabbitmq.port` | 5672 | 5671 (TLS) |
| `rabbitmq.vhost` | /pf-ef | /pf-ef |
| `rabbitmq.management-url` | http://localhost:15672 | https://{host}:15671 |
| `rabbitmq.username.publisher` | pf_ef_admin | pf_ef_publisher |
| `rabbitmq.username.consumer` | pf_ef_admin | pf_ef_consumer |

---

## 8. MQ 리소스 변경 시 체크리스트

- [ ] MqConstants.java에 상수 추가/수정
- [ ] MqResourceConfig.java에 Bean 정의
- [ ] 이 문서(INFRASTRUCTURE.md) 업데이트
- [ ] rabbitmq-definitions.json 파일 업데이트
- [ ] 변경 이력 기록

---

## 9. MQ 비교 분석

### 9.1 접속 정보

| 환경 | Host | Port | vhost |
|------|------|------|-------|
| develop | eventflow-web-dev.mq.fingate.kr | 15672 | /pf-ef |
| production | (TBD) | 15672 | /pf-ef |

### 9.2 간단 조회 명령어

```bash
# Exchange 목록
curl -s -u platform_ef:platform_ef \
  "http://eventflow-web-dev.mq.fingate.kr:15672/api/exchanges/%2Fpf-ef" | \
  jq -r '.[] | select(.name | startswith("ex.pf.ef")) | .name'

# Queue 목록
curl -s -u platform_ef:platform_ef \
  "http://eventflow-web-dev.mq.fingate.kr:15672/api/queues/%2Fpf-ef" | \
  jq -r '.[].name'

# Binding 목록
curl -s -u platform_ef:platform_ef \
  "http://eventflow-web-dev.mq.fingate.kr:15672/api/bindings/%2Fpf-ef" | \
  jq -r '.[] | select(.source != "") | "\(.source) --> \(.destination) [routing: \(.routing_key)]"'
```

---

## 10. 미래 로드맵

### 10.1 MQ SSL 활성화

```yaml
spring:
  rabbitmq:
    ssl:
      enabled: true
      algorithm: TLSv1.3
```
프로덕션 배포 전 필수. 금융권 보안 감사 기준 충족.

### 10.2 DLQ 모니터링 알림

- DlqMonitorService: 1분마다 DLQ 메시지 수 점검
- 임계값 초과 시 Slack/이메일 알림
- Grafana 대시보드에 DLQ 메트릭 추가

### 10.3 Circuit Breaker (WiseT API)

- Resilience4j @CircuitBreaker, @Retry 적용
- 50% 실패 시 OPEN, 30초 후 HALF_OPEN, 3회 성공 시 CLOSED
- Exponential Backoff: 1s, 2s, 4s

### 10.4 MQ 메트릭 수집

- Micrometer + Prometheus + Grafana
- 메트릭: 발행/실패 메시지 수, 처리 시간, 큐 적체량, 외부 API 응답 시간

### 10.5 WebSocket (v2 예정)

**현재 (v1):** SSE 기반 알림
- 30분 타임아웃, 30초 heartbeat
- 다중 디바이스 지원
- Last-Event-ID 기반 재연결 복구

**예정 (v2):** WebSocket
- Profile: worker-ws (8085)
- Raw WebSocket (STOMP 미사용)
- Redis Pub/Sub 기반 세션 동기화 (스케일아웃)
- 메시지 타입: CONNECT, CONNECTED, DISCONNECT, PING, PONG, CHAT_MESSAGE, ERROR
- 에러 코드: 75001-75006

---

## 11. 변경 이력

| 날짜 | 변경 내용 |
|------|----------|
| 2024-01 | 초기 구성 (log, message, notification) |
| 2025-03 | 문서 정리, MQ 리소스 비교 분석 정책 추가 |
| 2026-03 | notification DLQ, commoncode 리소스, 동기화 기능 추가 |
| 2026-03 | 서버 역할 분담 정책 (Application: Publisher, System: Consumer+리소스) |
| 2026-03 | Producer/Consumer 매핑 코드 동기화, management-url 반영 |

---

END OF FILE
