# 05. Exchange / Queue 표준 설계

```
Document ID    : MQ-SPEC-001-05
Version        : 1.0.0
Classification : Internal / Confidential
```

---

## 1. 프로젝트별 Exchange 구조

### 1.1 Exchange 네이밍 규칙

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Exchange Naming Convention                          │
│                                                                          │
│  Pattern: {project-id}.exchange.{domain}[.{purpose}]                     │
│                                                                          │
│  Examples:                                                               │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  ef.exchange.order          # Order 도메인 기본 Exchange         │    │
│  │  ef.exchange.order.dlx      # Order 도메인 DLX                   │    │
│  │  ef.exchange.order.delay    # Order 도메인 Delay Exchange        │    │
│  │  ef.exchange.user           # User 도메인 기본 Exchange          │    │
│  │  ef.exchange.notification   # Notification 도메인 기본 Exchange  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Reserved Suffixes:                                                      │
│  - .dlx     : Dead Letter Exchange                                       │
│  - .delay   : Delayed Message Exchange                                   │
│  - .retry   : Retry Exchange                                             │
│  - .fanout  : Fanout용 Exchange                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Exchange Type 선택 기준

| Type | 사용 케이스 | 예시 |
|------|------------|------|
| topic | 기본 (패턴 라우팅 필요) | ef.exchange.order |
| direct | 정확한 라우팅 키 매칭 | ef.exchange.order.specific |
| fanout | 모든 바인딩된 큐에 전달 | ef.exchange.order.dlx |
| headers | 헤더 기반 라우팅 (드물게 사용) | - |

### 1.3 Exchange 구조도

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   eventflow Project Exchange Structure                   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  ef.exchange.order (topic)                                       │    │
│  │                                                                  │    │
│  │  Routing Keys:                                                   │    │
│  │  ├── order.created      → ef.queue.order.created                │    │
│  │  ├── order.updated      → ef.queue.order.updated                │    │
│  │  ├── order.cancelled    → ef.queue.order.cancelled              │    │
│  │  └── order.*            → ef.queue.order.all (모니터링용)        │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                     │                                                    │
│                     │ DLX                                                │
│                     ▼                                                    │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  ef.exchange.order.dlx (fanout)                                  │    │
│  │                                                                  │    │
│  │  All failed messages → ef.queue.order.dlq                        │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  ef.exchange.user (topic)                                        │    │
│  │                                                                  │    │
│  │  Routing Keys:                                                   │    │
│  │  ├── user.registered    → ef.queue.user.registered              │    │
│  │  ├── user.updated       → ef.queue.user.updated                 │    │
│  │  └── user.deleted       → ef.queue.user.deleted                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                     │                                                    │
│                     │ DLX                                                │
│                     ▼                                                    │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  ef.exchange.user.dlx (fanout)                                   │    │
│  │                                                                  │    │
│  │  All failed messages → ef.queue.user.dlq                         │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  ef.exchange.notification (topic)                                │    │
│  │                                                                  │    │
│  │  Routing Keys:                                                   │    │
│  │  ├── notification.email → ef.queue.notification.email           │    │
│  │  ├── notification.sms   → ef.queue.notification.sms             │    │
│  │  └── notification.push  → ef.queue.notification.push            │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. DLX / DLQ 구조

### 2.1 DLX/DLQ 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        DLX / DLQ Architecture                            │
│                                                                          │
│   Normal Flow                                                            │
│  ┌──────────┐     ┌──────────────────┐     ┌──────────────────┐         │
│  │ Producer │────►│  ef.exchange.*   │────►│ ef.queue.*       │         │
│  └──────────┘     └──────────────────┘     └──────────────────┘         │
│                                                   │                      │
│                                                   │ Consumer             │
│                                                   ▼                      │
│                                            ┌─────────────┐               │
│                                            │  Business   │               │
│                                            │   Logic     │               │
│                                            └─────────────┘               │
│                                                   │                      │
│                            ┌──────────────────────┼──────────────┐       │
│                            │                      │              │       │
│                         Success               Retry          Final Fail  │
│                            │                      │              │       │
│                            ▼                      ▼              ▼       │
│                        ┌──────┐            ┌──────────┐    ┌──────────┐  │
│                        │ ACK  │            │ Retry    │    │  NACK    │  │
│                        └──────┘            │ (requeue)│    │ (reject) │  │
│                                            └──────────┘    └──────────┘  │
│                                                                  │       │
│                                                                  │       │
│   Dead Letter Flow                                               │       │
│  ┌───────────────────────────────────────────────────────────────┘       │
│  │                                                                       │
│  │  ┌──────────────────┐     ┌──────────────────┐                       │
│  └─►│ ef.exchange.*.dlx│────►│ ef.queue.*.dlq   │                       │
│     │     (fanout)     │     │                  │                       │
│     └──────────────────┘     └──────────────────┘                       │
│                                      │                                   │
│                                      │ DLQ Consumer                      │
│                                      ▼ (수동 재처리 또는 알람)            │
│                               ┌─────────────┐                            │
│                               │  Alerting   │                            │
│                               │  / Manual   │                            │
│                               │  Reprocess  │                            │
│                               └─────────────┘                            │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 DLQ로 이동되는 조건

| 조건 | 설명 |
|------|------|
| Message Rejected | Consumer가 basicNack(requeue=false) 호출 |
| Message TTL 만료 | 메시지가 큐에서 TTL 초과 |
| Queue Max Length 초과 | overflow=reject-publish-dlx 설정 시 |
| 최대 재시도 초과 | Application에서 재시도 횟수 초과 판단 |

### 2.3 DLQ 설정 (Terraform)

```hcl
# DLX Exchange
resource "rabbitmq_exchange" "order_dlx" {
  name  = "ef.exchange.order.dlx"
  vhost = "/eventflow"

  settings {
    type        = "fanout"
    durable     = true
    auto_delete = false
  }
}

# DLQ Queue
resource "rabbitmq_queue" "order_dlq" {
  name  = "ef.queue.order.dlq"
  vhost = "/eventflow"

  settings {
    durable = true
    arguments = {
      "x-queue-type"      = "quorum"
      "x-message-ttl"     = 604800000  # 7일
      "x-max-length"      = 50000
    }
  }
}

# DLQ Binding
resource "rabbitmq_binding" "order_dlq_binding" {
  source           = rabbitmq_exchange.order_dlx.name
  vhost            = "/eventflow"
  destination      = rabbitmq_queue.order_dlq.name
  destination_type = "queue"
  routing_key      = "#"
}

# 일반 Queue에 DLX 설정
resource "rabbitmq_queue" "order_created" {
  name  = "ef.queue.order.created"
  vhost = "/eventflow"

  settings {
    durable = true
    arguments = {
      "x-queue-type"              = "quorum"
      "x-dead-letter-exchange"    = "ef.exchange.order.dlx"
      "x-dead-letter-routing-key" = "order.created.failed"
      "x-message-ttl"             = 86400000  # 24시간
      "x-max-length"              = 100000
    }
  }
}
```

### 2.4 재시도 전략 (Retry Exchange)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Retry Strategy with Delay                         │
│                                                                          │
│  Original Queue        Retry Exchange          Delay Queues             │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────────┐         │
│  │ef.queue.order│     │ef.exchange   │     │ef.queue.order    │         │
│  │   .created   │     │  .retry      │────►│ .retry.1s        │         │
│  │              │     │  (delayed)   │     │ (TTL: 1000ms)    │         │
│  └───────┬──────┘     └──────────────┘     └────────┬─────────┘         │
│          │                   │                      │                    │
│          │ NACK              │                      │ Expired            │
│          │ (retry)           │                      │                    │
│          └───────────────────┘                      │                    │
│                                                     │                    │
│                              ┌──────────────────────┘                    │
│                              ▼                                           │
│                   ┌──────────────────┐                                   │
│                   │ ef.exchange.order │                                  │
│                   │     (main)        │                                  │
│                   └────────┬─────────┘                                   │
│                            │                                             │
│                            ▼                                             │
│                   ┌──────────────────┐                                   │
│                   │ ef.queue.order   │                                   │
│                   │    .created      │                                   │
│                   │  (retry count+1) │                                   │
│                   └──────────────────┘                                   │
│                                                                          │
│  Retry Schedule:                                                         │
│  - 1st retry: 1s delay                                                   │
│  - 2nd retry: 5s delay                                                   │
│  - 3rd retry: 30s delay                                                  │
│  - Final fail: DLQ                                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Routing Key 네이밍 규칙

### 3.1 Routing Key 패턴

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Routing Key Naming Convention                       │
│                                                                          │
│  Pattern: {domain}.{action}[.{version}]                                  │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Examples:                                                       │    │
│  │                                                                  │    │
│  │  Basic:                                                          │    │
│  │  - order.created                                                 │    │
│  │  - order.updated                                                 │    │
│  │  - order.cancelled                                               │    │
│  │  - user.registered                                               │    │
│  │  - payment.completed                                             │    │
│  │                                                                  │    │
│  │  With Version:                                                   │    │
│  │  - order.created.v1                                              │    │
│  │  - order.created.v2                                              │    │
│  │                                                                  │    │
│  │  Wildcards (Consumer Binding):                                   │    │
│  │  - order.*          (all order events)                           │    │
│  │  - *.created        (all created events)                         │    │
│  │  - #                (all events)                                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Naming Rules:                                                           │
│  1. 소문자만 사용                                                        │
│  2. 단어 구분은 언더스코어 (_)                                           │
│  3. 도메인은 단수형 (orders X, order O)                                  │
│  4. 액션은 과거형 (create X, created O)                                  │
│  5. 계층은 마침표(.)로 구분                                              │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Routing Key 매핑 테이블

| Event Type | Routing Key | Target Queue(s) |
|------------|-------------|-----------------|
| order.created | order.created | ef.queue.order.created |
| order.updated | order.updated | ef.queue.order.updated |
| order.cancelled | order.cancelled | ef.queue.order.cancelled |
| order.* | order.* | ef.queue.order.all (모니터링) |
| user.registered | user.registered | ef.queue.user.registered |
| notification.email | notification.email | ef.queue.notification.email |
| notification.sms | notification.sms | ef.queue.notification.sms |

### 3.3 Binding 설정 예시

```hcl
# 기본 Binding
resource "rabbitmq_binding" "order_created" {
  source           = "ef.exchange.order"
  vhost            = "/eventflow"
  destination      = "ef.queue.order.created"
  destination_type = "queue"
  routing_key      = "order.created"
}

# Wildcard Binding (모니터링용)
resource "rabbitmq_binding" "order_all" {
  source           = "ef.exchange.order"
  vhost            = "/eventflow"
  destination      = "ef.queue.order.monitoring"
  destination_type = "queue"
  routing_key      = "order.*"
}

# Version별 Binding
resource "rabbitmq_binding" "order_created_v1" {
  source           = "ef.exchange.order"
  vhost            = "/eventflow"
  destination      = "ef.queue.order.created.v1"
  destination_type = "queue"
  routing_key      = "order.created.v1"
}

resource "rabbitmq_binding" "order_created_v2" {
  source           = "ef.exchange.order"
  vhost            = "/eventflow"
  destination      = "ef.queue.order.created.v2"
  destination_type = "queue"
  routing_key      = "order.created.v2"
}
```

---

## 4. Consumer별 Queue 분리 전략

### 4.1 분리 패턴

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Consumer Queue Separation Patterns                    │
│                                                                          │
│  Pattern 1: Service별 분리                                               │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │   ef.exchange.order                                              │    │
│  │         │                                                        │    │
│  │    order.created                                                 │    │
│  │         │                                                        │    │
│  │    ┌────┴────┐                                                   │    │
│  │    │         │                                                   │    │
│  │    ▼         ▼                                                   │    │
│  │  ┌─────────────────┐  ┌─────────────────┐                       │    │
│  │  │ef.queue.order   │  │ef.queue.order   │                       │    │
│  │  │.created.system  │  │.created.batch   │                       │    │
│  │  │                 │  │                 │                       │    │
│  │  │ Consumer:       │  │ Consumer:       │                       │    │
│  │  │ ef-system       │  │ ef-batch        │                       │    │
│  │  └─────────────────┘  └─────────────────┘                       │    │
│  │                                                                  │    │
│  │  ※ 동일 이벤트를 여러 서비스가 독립적으로 소비                   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Pattern 2: 처리 우선순위별 분리                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │   ef.exchange.order                                              │    │
│  │         │                                                        │    │
│  │    order.created                                                 │    │
│  │         │                                                        │    │
│  │    ┌────┴────┐                                                   │    │
│  │    │         │                                                   │    │
│  │    ▼         ▼                                                   │    │
│  │  ┌─────────────────┐  ┌─────────────────┐                       │    │
│  │  │ef.queue.order   │  │ef.queue.order   │                       │    │
│  │  │.created.high    │  │.created.normal  │                       │    │
│  │  │                 │  │                 │                       │    │
│  │  │ Priority: HIGH  │  │ Priority: NORMAL│                       │    │
│  │  │ Consumers: 10   │  │ Consumers: 5    │                       │    │
│  │  └─────────────────┘  └─────────────────┘                       │    │
│  │                                                                  │    │
│  │  ※ VIP 고객 주문은 HIGH 큐로 라우팅                              │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Pattern 3: Competing Consumers (부하 분산)                              │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                                                                  │    │
│  │   ef.queue.order.created                                         │    │
│  │              │                                                   │    │
│  │      ┌───────┼───────┬───────┐                                  │    │
│  │      │       │       │       │                                  │    │
│  │      ▼       ▼       ▼       ▼                                  │    │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                            │    │
│  │  │Pod-1 │ │Pod-2 │ │Pod-3 │ │Pod-4 │                            │    │
│  │  │      │ │      │ │      │ │      │                            │    │
│  │  │prefetch│ │prefetch│ │prefetch│ │prefetch│                    │    │
│  │  │  =10 │ │  =10 │ │  =10 │ │  =10 │                            │    │
│  │  └──────┘ └──────┘ └──────┘ └──────┘                            │    │
│  │                                                                  │    │
│  │  ※ 동일 큐에서 여러 Consumer가 메시지 분배                       │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Queue 네이밍 규칙

```
Pattern: {project-id}.queue.{domain}.{action}[.{consumer}][.{priority}]

Examples:
┌────────────────────────────────────────────────────────────────────────┐
│ Queue Name                          │ 설명                             │
├────────────────────────────────────────────────────────────────────────┤
│ ef.queue.order.created              │ Order Created 기본 큐            │
│ ef.queue.order.created.system       │ System 서비스 전용 큐            │
│ ef.queue.order.created.batch        │ Batch Worker 전용 큐             │
│ ef.queue.order.created.high         │ 높은 우선순위 큐                 │
│ ef.queue.order.created.dlq          │ Dead Letter Queue                │
│ ef.queue.order.monitoring           │ 모니터링 전용 (모든 order 이벤트)│
└────────────────────────────────────────────────────────────────────────┘
```

### 4.3 Prefetch 설정 권장값

| Consumer 유형 | Prefetch Count | 근거 |
|--------------|----------------|------|
| 빠른 처리 (< 100ms) | 50-100 | 처리량 극대화 |
| 보통 처리 (100ms-1s) | 10-30 | 균형 |
| 느린 처리 (> 1s) | 1-5 | 메시지 분배 공평성 |
| 순서 보장 필요 | 1 | 순차 처리 보장 |

---

## 5. 재처리 전략

### 5.1 재처리 흐름

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Message Reprocessing Flow                           │
│                                                                          │
│   Step 1: DLQ 메시지 확인                                                │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  $ rabbitmqadmin get queue=ef.queue.order.dlq count=10          │   │
│   │                                                                  │   │
│   │  event_id: 20260227-ef01-000042-abc123                          │   │
│   │  event_type: order.created                                       │   │
│   │  original_queue: ef.queue.order.created                          │   │
│   │  death_reason: rejected                                          │   │
│   │  death_count: 3                                                  │   │
│   │  first_death_at: 2026-02-27T10:30:45Z                           │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                      │
│                                   ▼                                      │
│   Step 2: 원인 분석                                                      │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  - Application 로그 확인                                         │   │
│   │  - 에러 타입 분류:                                               │   │
│   │    ① 일시적 오류 (외부 API 장애) → 재처리 가능                   │   │
│   │    ② 데이터 오류 (잘못된 형식) → 수정 후 재처리                  │   │
│   │    ③ 비즈니스 오류 (규칙 위반) → 검토 후 폐기/수정               │   │
│   │    ④ 버그 (코드 결함) → 코드 수정 후 재처리                      │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                      │
│                                   ▼                                      │
│   Step 3: 재처리 실행                                                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Option A: DLQ → 원래 Queue로 이동 (shovel)                      │   │
│   │                                                                  │   │
│   │  $ rabbitmqadmin declare shovel name=dlq-reprocess \             │   │
│   │      src-queue=ef.queue.order.dlq \                              │   │
│   │      dest-exchange=ef.exchange.order \                           │   │
│   │      dest-exchange-key=order.created                             │   │
│   │                                                                  │   │
│   │  Option B: 관리 UI에서 개별 메시지 이동                          │   │
│   │                                                                  │   │
│   │  Option C: 재처리 스크립트 실행                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                   │                                      │
│                                   ▼                                      │
│   Step 4: 검증                                                           │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  - 재처리 결과 모니터링                                          │   │
│   │  - 다시 DLQ로 가는지 확인                                        │   │
│   │  - 비즈니스 처리 완료 확인                                       │   │
│   └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 재처리 스크립트

```python
#!/usr/bin/env python3
# reprocess_dlq.py

import pika
import json
import sys
from datetime import datetime

def reprocess_dlq_messages(
    dlq_name: str,
    target_exchange: str,
    routing_key: str,
    count: int = 10,
    filter_event_type: str = None
):
    """DLQ 메시지를 원래 큐로 재발행"""

    credentials = pika.PlainCredentials(
        os.environ['MQ_USERNAME'],
        os.environ['MQ_PASSWORD']
    )

    connection = pika.BlockingConnection(
        pika.ConnectionParameters(
            host='rabbitmq.platform-mq',
            port=5671,
            virtual_host='/eventflow',
            credentials=credentials,
            ssl_options=pika.SSLOptions(context)
        )
    )

    channel = connection.channel()
    reprocessed = 0
    skipped = 0

    for _ in range(count):
        method, properties, body = channel.basic_get(
            queue=dlq_name,
            auto_ack=False
        )

        if not method:
            print(f"No more messages in {dlq_name}")
            break

        message = json.loads(body)
        event_type = message.get('event_type')

        # 필터 적용
        if filter_event_type and event_type != filter_event_type:
            channel.basic_nack(
                delivery_tag=method.delivery_tag,
                requeue=True
            )
            skipped += 1
            continue

        # 재발행 헤더 추가
        headers = dict(properties.headers or {})
        headers['x-reprocessed'] = True
        headers['x-reprocessed-at'] = datetime.utcnow().isoformat()
        headers['x-original-dlq'] = dlq_name

        new_properties = pika.BasicProperties(
            message_id=properties.message_id,
            content_type='application/json',
            headers=headers,
            delivery_mode=2  # persistent
        )

        # 원래 Exchange로 재발행
        channel.basic_publish(
            exchange=target_exchange,
            routing_key=routing_key,
            body=body,
            properties=new_properties
        )

        # DLQ에서 제거
        channel.basic_ack(delivery_tag=method.delivery_tag)
        reprocessed += 1

        print(f"Reprocessed: {properties.message_id} ({event_type})")

    connection.close()
    print(f"\nSummary: Reprocessed={reprocessed}, Skipped={skipped}")

if __name__ == '__main__':
    reprocess_dlq_messages(
        dlq_name='ef.queue.order.dlq',
        target_exchange='ef.exchange.order',
        routing_key='order.created',
        count=int(sys.argv[1]) if len(sys.argv) > 1 else 10
    )
```

### 5.3 자동 재처리 정책

| 에러 유형 | 자동 재처리 | 재시도 간격 | 최대 재시도 |
|----------|------------|------------|------------|
| 일시적 네트워크 오류 | O | 1s, 5s, 30s | 3 |
| 외부 API 타임아웃 | O | 5s, 30s, 2m | 3 |
| DB 연결 오류 | O | 1s, 5s, 30s | 3 |
| 데이터 검증 실패 | X | - | DLQ 직행 |
| 비즈니스 규칙 위반 | X | - | DLQ 직행 |
| 알 수 없는 예외 | O | 1s, 5s, 30s | 2 |

---

## 6. 전체 Exchange/Queue 설계 예시 (eventflow)

### 6.1 리소스 목록

```yaml
# eventflow MQ Resources

exchanges:
  # Order Domain
  - name: ef.exchange.order
    type: topic
    durable: true
  - name: ef.exchange.order.dlx
    type: fanout
    durable: true

  # User Domain
  - name: ef.exchange.user
    type: topic
    durable: true
  - name: ef.exchange.user.dlx
    type: fanout
    durable: true

  # Notification Domain
  - name: ef.exchange.notification
    type: topic
    durable: true
  - name: ef.exchange.notification.dlx
    type: fanout
    durable: true

queues:
  # Order Queues
  - name: ef.queue.order.created
    type: quorum
    dlx: ef.exchange.order.dlx
    max_length: 100000
    message_ttl: 86400000
  - name: ef.queue.order.updated
    type: quorum
    dlx: ef.exchange.order.dlx
    max_length: 100000
    message_ttl: 86400000
  - name: ef.queue.order.cancelled
    type: quorum
    dlx: ef.exchange.order.dlx
    max_length: 50000
    message_ttl: 86400000
  - name: ef.queue.order.dlq
    type: quorum
    max_length: 50000
    message_ttl: 604800000  # 7 days

  # User Queues
  - name: ef.queue.user.registered
    type: quorum
    dlx: ef.exchange.user.dlx
    max_length: 50000
    message_ttl: 86400000
  - name: ef.queue.user.updated
    type: quorum
    dlx: ef.exchange.user.dlx
    max_length: 50000
    message_ttl: 86400000
  - name: ef.queue.user.dlq
    type: quorum
    max_length: 10000
    message_ttl: 604800000

  # Notification Queues
  - name: ef.queue.notification.email
    type: quorum
    dlx: ef.exchange.notification.dlx
    max_length: 200000
    message_ttl: 86400000
  - name: ef.queue.notification.sms
    type: quorum
    dlx: ef.exchange.notification.dlx
    max_length: 100000
    message_ttl: 86400000
  - name: ef.queue.notification.push
    type: quorum
    dlx: ef.exchange.notification.dlx
    max_length: 200000
    message_ttl: 86400000
  - name: ef.queue.notification.dlq
    type: quorum
    max_length: 20000
    message_ttl: 604800000

bindings:
  # Order Bindings
  - source: ef.exchange.order
    destination: ef.queue.order.created
    routing_key: order.created
  - source: ef.exchange.order
    destination: ef.queue.order.updated
    routing_key: order.updated
  - source: ef.exchange.order
    destination: ef.queue.order.cancelled
    routing_key: order.cancelled
  - source: ef.exchange.order.dlx
    destination: ef.queue.order.dlq
    routing_key: "#"

  # User Bindings
  - source: ef.exchange.user
    destination: ef.queue.user.registered
    routing_key: user.registered
  - source: ef.exchange.user
    destination: ef.queue.user.updated
    routing_key: user.updated
  - source: ef.exchange.user.dlx
    destination: ef.queue.user.dlq
    routing_key: "#"

  # Notification Bindings
  - source: ef.exchange.notification
    destination: ef.queue.notification.email
    routing_key: notification.email
  - source: ef.exchange.notification
    destination: ef.queue.notification.sms
    routing_key: notification.sms
  - source: ef.exchange.notification
    destination: ef.queue.notification.push
    routing_key: notification.push
  - source: ef.exchange.notification.dlx
    destination: ef.queue.notification.dlq
    routing_key: "#"
```

---

END OF DOCUMENT
