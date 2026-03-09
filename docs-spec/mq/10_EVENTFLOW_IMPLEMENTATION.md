# 10. eventflow 프로젝트 MQ 구현

```
Document ID    : MQ-SPEC-001-10
Version        : 1.0.0
Classification : Internal / Confidential
Project        : eventflow (최초 적용 프로젝트)
```

---

## 1. 아키텍처 개요

### 1.1 현재 단계 의도

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    eventflow MQ Architecture (Phase 1)                   │
│                                                                          │
│  현재 단계 목표:                                                         │
│  - MQ 인프라 기본 구조 구축                                              │
│  - Listener 기본 골격 구현 (로그 출력만)                                 │
│  - DLQ 정책 활성화 검증                                                  │
│  - 향후 메시지 발송 로직 추가 기반 마련                                  │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                         vhost: /pf-ef                             │   │
│  │                                                                   │   │
│  │   GATEWAY          APPLICATION           SYSTEM (MQ Worker)      │   │
│  │  ┌─────────┐       ┌─────────┐          ┌─────────────────┐      │   │
│  │  │ Access  │       │ SMS/LMS │          │ LogMessageListener│    │   │
│  │  │ Error   │──────▶│ 알림톡  │─────────▶│ MessageSendListener│   │   │
│  │  │ Audit   │       │ 요청    │          │ DeadLetterListener│    │   │
│  │  │ Log     │       └─────────┘          └─────────────────┘      │   │
│  │  └─────────┘                                    │                 │   │
│  │       │                                         │                 │   │
│  │       │ log.*                                   │ 현재: 로그 출력 │   │
│  │       ▼                                         │ 향후: 발송 처리 │   │
│  │  ┌─────────────────────────────────────────────────────────────┐ │   │
│  │  │                    RabbitMQ Cluster                          │ │   │
│  │  │                                                              │ │   │
│  │  │  ex.pf.ef.log.topic ──▶ q.pf.ef.log.write                   │ │   │
│  │  │                                                              │ │   │
│  │  │  ex.pf.ef.message.topic ──▶ q.pf.ef.message.send ───┐       │ │   │
│  │  │                                                      │ DLX   │ │   │
│  │  │  ex.pf.ef.notification.topic ──▶ q.pf.ef.notification─┤       │ │   │
│  │  │                                                      ▼       │ │   │
│  │  │  ex.pf.ef.dlx ─────────▶ q.pf.ef.dlq.message.send           │ │   │
│  │  │                    └────▶ q.pf.ef.dlq.notification           │ │   │
│  │  └─────────────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 네이밍 규칙

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Naming Convention                                │
│                                                                          │
│  접두어:                                                                 │
│  ├── pf     : Platform 공통                                             │
│  ├── ef     : eventflow 프로젝트                                        │
│  ├── ex     : Exchange                                                   │
│  └── q      : Queue                                                      │
│                                                                          │
│  vhost:                                                                  │
│  └── /pf-ef : Platform eventflow                                        │
│                                                                          │
│  Exchange:                                                               │
│  ├── ex.pf.ef.log.topic          : 로그 전용 (topic)                    │
│  ├── ex.pf.ef.message.topic      : 메시지 발송 전용 (topic)             │
│  ├── ex.pf.ef.notification.topic : 알림 전용 (topic)                    │
│  └── ex.pf.ef.dlx                : Dead Letter Exchange (topic)         │
│                                                                          │
│  Queue:                                                                  │
│  ├── q.pf.ef.log.write           : 로그 기록 큐                         │
│  ├── q.pf.ef.message.send        : 메시지 발송 큐                       │
│  ├── q.pf.ef.notification        : 알림 큐                              │
│  ├── q.pf.ef.dlq.message.send    : 메시지 발송 DLQ                      │
│  └── q.pf.ef.dlq.notification    : 알림 DLQ                             │
│                                                                          │
│  Routing Key:                                                            │
│  ├── log.access.v1         : 접근 로그                                  │
│  ├── log.error.v1          : 에러 로그                                  │
│  ├── log.audit.v1          : 감사 로그                                  │
│  ├── message.send.v1       : 메시지 발송                                │
│  └── notification.event.v1 : 알림 이벤트                                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Exchange / Queue / Binding 구성

### 2.1 전체 구성도

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         vhost: /pf-ef                                    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  EXCHANGES                                                       │    │
│  │                                                                  │    │
│  │  ex.pf.ef.log.topic (type: topic, durable: true)                │    │
│  │      │                                                           │    │
│  │      ├── log.access.v1 ──▶ q.pf.ef.log.write                    │    │
│  │      ├── log.error.v1  ──▶ q.pf.ef.log.write                    │    │
│  │      └── log.audit.v1  ──▶ q.pf.ef.log.write                    │    │
│  │                                                                  │    │
│  │  ex.pf.ef.message.topic (type: topic, durable: true)                │    │
│  │      │                                                           │    │
│  │      └── message.send.v1 ──▶ q.pf.ef.message.send                │    │
│  │                                                                  │    │
│  │  ex.pf.ef.notification.topic (type: topic, durable: true)       │    │
│  │      │                                                           │    │
│  │      └── notification.event.v1 ──▶ q.pf.ef.notification          │    │
│  │                                                                  │    │
│  │  ex.pf.ef.dlx (type: topic, durable: true)                      │    │
│  │      │                                                           │    │
│  │      ├── message.send.dlq ──▶ q.pf.ef.dlq.message.send          │    │
│  │      └── notification.dlq ──▶ q.pf.ef.dlq.notification          │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  QUEUES                                                          │    │
│  │                                                                  │    │
│  │  q.pf.ef.log.write                                               │    │
│  │    ├── type: quorum                                              │    │
│  │    ├── durable: true                                             │    │
│  │    └── DLQ: 없음 (로그는 유실 허용)                              │    │
│  │                                                                  │    │
│  │  q.pf.ef.message.send                                                │    │
│  │    ├── type: quorum                                              │    │
│  │    ├── durable: true                                             │    │
│  │    ├── x-dead-letter-exchange: ex.pf.ef.dlx                      │    │
│  │    ├── x-dead-letter-routing-key: message.send.dlq                   │    │
│  │    ├── x-message-ttl: 86400000 (24시간)                          │    │
│  │    └── x-max-length: 50000                                       │    │
│  │                                                                  │    │
│  │  q.pf.ef.notification                                            │    │
│  │    ├── type: quorum                                              │    │
│  │    ├── durable: true                                             │    │
│  │    ├── x-dead-letter-exchange: ex.pf.ef.dlx                      │    │
│  │    ├── x-dead-letter-routing-key: notification.dlq               │    │
│  │    └── x-message-ttl: 86400000 (24시간)                          │    │
│  │                                                                  │    │
│  │  q.pf.ef.dlq.message.send                                        │    │
│  │    ├── type: quorum                                              │    │
│  │    ├── durable: true                                             │    │
│  │    └── x-message-ttl: 604800000 (7일)                            │    │
│  │                                                                  │    │
│  │  q.pf.ef.dlq.notification                                        │    │
│  │    ├── type: quorum                                              │    │
│  │    ├── durable: true                                             │    │
│  │    └── x-message-ttl: 604800000 (7일)                            │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 상세 스펙

| 리소스 유형 | 이름 | 타입 | 설정 |
|------------|------|------|------|
| Exchange | ex.pf.ef.log.topic | topic | durable=true |
| Exchange | ex.pf.ef.message.topic | topic | durable=true |
| Exchange | ex.pf.ef.notification.topic | topic | durable=true |
| Exchange | ex.pf.ef.dlx | topic | durable=true |
| Queue | q.pf.ef.log.write | quorum | durable=true |
| Queue | q.pf.ef.message.send | quorum | DLX, TTL 24h, max-length 50000 |
| Queue | q.pf.ef.notification | quorum | DLX, TTL 24h |
| Queue | q.pf.ef.dlq.message.send | quorum | TTL 7일 |
| Queue | q.pf.ef.dlq.notification | quorum | TTL 7일 |

### 2.3 Binding 매트릭스

| Exchange | Routing Key | Queue |
|----------|-------------|-------|
| ex.pf.ef.log.topic | log.access.v1 | q.pf.ef.log.write |
| ex.pf.ef.log.topic | log.error.v1 | q.pf.ef.log.write |
| ex.pf.ef.log.topic | log.audit.v1 | q.pf.ef.log.write |
| ex.pf.ef.message.topic | message.send.v1 | q.pf.ef.message.send |
| ex.pf.ef.notification.topic | notification.event.v1 | q.pf.ef.notification |
| ex.pf.ef.dlx | message.send.dlq | q.pf.ef.dlq.message.send |
| ex.pf.ef.dlx | notification.dlq | q.pf.ef.dlq.notification |

---

## 3. Terraform 리소스 예시

```hcl
# terraform/environments/prod/projects/eventflow-mq.tf

terraform {
  required_providers {
    rabbitmq = {
      source  = "cyrilgdn/rabbitmq"
      version = "~> 1.8"
    }
  }
}

# ============================================================
# vhost
# ============================================================
resource "rabbitmq_vhost" "pf_ef" {
  name = "pf-ef"
}

# ============================================================
# Users
# ============================================================
resource "rabbitmq_user" "ef_producer" {
  name     = "ef-producer"
  password = var.ef_producer_password
  tags     = ["producer"]
}

resource "rabbitmq_user" "ef_consumer" {
  name     = "ef-consumer"
  password = var.ef_consumer_password
  tags     = ["consumer"]
}

# ============================================================
# Permissions
# ============================================================
resource "rabbitmq_permissions" "ef_producer" {
  user  = rabbitmq_user.ef_producer.name
  vhost = rabbitmq_vhost.pf_ef.name

  permissions {
    configure = ""
    write     = "^ex\\.pf\\.ef\\..*"
    read      = ""
  }
}

resource "rabbitmq_permissions" "ef_consumer" {
  user  = rabbitmq_user.ef_consumer.name
  vhost = rabbitmq_vhost.pf_ef.name

  permissions {
    configure = ""
    write     = ""
    read      = "^q\\.pf\\.ef\\..*"
  }
}

# ============================================================
# Exchanges
# ============================================================
resource "rabbitmq_exchange" "log_topic" {
  name  = "ex.pf.ef.log.topic"
  vhost = rabbitmq_vhost.pf_ef.name

  settings {
    type        = "topic"
    durable     = true
    auto_delete = false
  }
}

resource "rabbitmq_exchange" "msg_topic" {
  name  = "ex.pf.ef.message.topic"
  vhost = rabbitmq_vhost.pf_ef.name

  settings {
    type        = "topic"
    durable     = true
    auto_delete = false
  }
}

resource "rabbitmq_exchange" "notification_topic" {
  name  = "ex.pf.ef.notification.topic"
  vhost = rabbitmq_vhost.pf_ef.name

  settings {
    type        = "topic"
    durable     = true
    auto_delete = false
  }
}

resource "rabbitmq_exchange" "dlx" {
  name  = "ex.pf.ef.dlx"
  vhost = rabbitmq_vhost.pf_ef.name

  settings {
    type        = "topic"
    durable     = true
    auto_delete = false
  }
}

# ============================================================
# Queues
# ============================================================
resource "rabbitmq_queue" "log_write" {
  name  = "q.pf.ef.log.write"
  vhost = rabbitmq_vhost.pf_ef.name

  settings {
    durable = true
    arguments = {
      "x-queue-type" = "quorum"
    }
  }
}

resource "rabbitmq_queue" "msg_send" {
  name  = "q.pf.ef.message.send"
  vhost = rabbitmq_vhost.pf_ef.name

  settings {
    durable = true
    arguments = {
      "x-queue-type"              = "quorum"
      "x-dead-letter-exchange"    = "ex.pf.ef.dlx"
      "x-dead-letter-routing-key" = "message.send.dlq"
      "x-message-ttl"             = 86400000
      "x-max-length"              = 50000
      "x-overflow"                = "reject-publish"
    }
  }

  depends_on = [rabbitmq_exchange.dlx]
}

resource "rabbitmq_queue" "notification" {
  name  = "q.pf.ef.notification"
  vhost = rabbitmq_vhost.pf_ef.name

  settings {
    durable = true
    arguments = {
      "x-queue-type"              = "quorum"
      "x-dead-letter-exchange"    = "ex.pf.ef.dlx"
      "x-dead-letter-routing-key" = "notification.dlq"
      "x-message-ttl"             = 86400000
    }
  }

  depends_on = [rabbitmq_exchange.dlx]
}

resource "rabbitmq_queue" "dlq_msg_send" {
  name  = "q.pf.ef.dlq.message.send"
  vhost = rabbitmq_vhost.pf_ef.name

  settings {
    durable = true
    arguments = {
      "x-queue-type"  = "quorum"
      "x-message-ttl" = 604800000
    }
  }
}

resource "rabbitmq_queue" "dlq_notification" {
  name  = "q.pf.ef.dlq.notification"
  vhost = rabbitmq_vhost.pf_ef.name

  settings {
    durable = true
    arguments = {
      "x-queue-type"  = "quorum"
      "x-message-ttl" = 604800000
    }
  }
}

# ============================================================
# Bindings
# ============================================================
resource "rabbitmq_binding" "log_access" {
  source           = rabbitmq_exchange.log_topic.name
  vhost            = rabbitmq_vhost.pf_ef.name
  destination      = rabbitmq_queue.log_write.name
  destination_type = "queue"
  routing_key      = "log.access.v1"
}

resource "rabbitmq_binding" "log_error" {
  source           = rabbitmq_exchange.log_topic.name
  vhost            = rabbitmq_vhost.pf_ef.name
  destination      = rabbitmq_queue.log_write.name
  destination_type = "queue"
  routing_key      = "log.error.v1"
}

resource "rabbitmq_binding" "log_audit" {
  source           = rabbitmq_exchange.log_topic.name
  vhost            = rabbitmq_vhost.pf_ef.name
  destination      = rabbitmq_queue.log_write.name
  destination_type = "queue"
  routing_key      = "log.audit.v1"
}

resource "rabbitmq_binding" "msg_send" {
  source           = rabbitmq_exchange.msg_topic.name
  vhost            = rabbitmq_vhost.pf_ef.name
  destination      = rabbitmq_queue.msg_send.name
  destination_type = "queue"
  routing_key      = "message.send.v1"
}

resource "rabbitmq_binding" "notification" {
  source           = rabbitmq_exchange.notification_topic.name
  vhost            = rabbitmq_vhost.pf_ef.name
  destination      = rabbitmq_queue.notification.name
  destination_type = "queue"
  routing_key      = "notification.event.v1"
}

resource "rabbitmq_binding" "dlq_msg_send" {
  source           = rabbitmq_exchange.dlx.name
  vhost            = rabbitmq_vhost.pf_ef.name
  destination      = rabbitmq_queue.dlq_msg_send.name
  destination_type = "queue"
  routing_key      = "message.send.dlq"
}

resource "rabbitmq_binding" "dlq_notification" {
  source           = rabbitmq_exchange.dlx.name
  vhost            = rabbitmq_vhost.pf_ef.name
  destination      = rabbitmq_queue.dlq_notification.name
  destination_type = "queue"
  routing_key      = "notification.dlq"
}
```

---

## 4. Helm values.yaml 예시

```yaml
# helm/releases/eventflow/rabbitmq-values.yaml

# RabbitMQ 연결 설정
rabbitmq:
  host: rabbitmq.platform-mq.svc.cluster.local
  port: 5672
  virtualHost: /pf-ef

  # TLS 설정 (Production)
  ssl:
    enabled: true
    port: 5671

  # Credentials (Vault에서 주입)
  auth:
    existingSecret: ef-mq-credentials
    usernameKey: username
    passwordKey: password

# eventflow MQ 리소스 정의
resources:
  exchanges:
    - name: ex.pf.ef.log.topic
      type: topic
      durable: true

    - name: ex.pf.ef.message.topic
      type: topic
      durable: true

    - name: ex.pf.ef.notification.topic
      type: topic
      durable: true

    - name: ex.pf.ef.dlx
      type: topic
      durable: true

  queues:
    - name: q.pf.ef.log.write
      durable: true
      arguments:
        x-queue-type: quorum

    - name: q.pf.ef.message.send
      durable: true
      arguments:
        x-queue-type: quorum
        x-dead-letter-exchange: ex.pf.ef.dlx
        x-dead-letter-routing-key: message.send.dlq
        x-message-ttl: 86400000
        x-max-length: 50000
        x-overflow: reject-publish

    - name: q.pf.ef.notification
      durable: true
      arguments:
        x-queue-type: quorum
        x-dead-letter-exchange: ex.pf.ef.dlx
        x-dead-letter-routing-key: notification.dlq
        x-message-ttl: 86400000

    - name: q.pf.ef.dlq.message.send
      durable: true
      arguments:
        x-queue-type: quorum
        x-message-ttl: 604800000

    - name: q.pf.ef.dlq.notification
      durable: true
      arguments:
        x-queue-type: quorum
        x-message-ttl: 604800000

  bindings:
    - source: ex.pf.ef.log.topic
      destination: q.pf.ef.log.write
      routingKey: "log.#"

    - source: ex.pf.ef.message.topic
      destination: q.pf.ef.message.send
      routingKey: message.send.v1

    - source: ex.pf.ef.notification.topic
      destination: q.pf.ef.notification
      routingKey: notification.event.v1

    - source: ex.pf.ef.dlx
      destination: q.pf.ef.dlq.message.send
      routingKey: message.send.dlq

    - source: ex.pf.ef.dlx
      destination: q.pf.ef.dlq.notification
      routingKey: notification.dlq
```

---

## 5. 패키지 구조

```
system/src/main/java/kr/fingate/gs/system/
├── mq/
│   ├── config/
│   │   ├── MqConstants.java          # Exchange/Queue/RoutingKey 상수
│   │   ├── MqConsumerConfig.java     # RabbitMQ Consumer 설정
│   │   └── MqResourceConfig.java     # Exchange/Queue/Binding Bean 정의
│   │
│   ├── dto/
│   │   ├── BaseMessage.java          # 메시지 공통 필드
│   │   ├── LogMessage.java           # 로그 메시지 DTO
│   │   └── MessageSendRequest.java   # 메시지 발송 요청 DTO
│   │
│   ├── listener/
│   │   ├── LogMessageListener.java       # 로그 이벤트 리스너
│   │   ├── MessageSendListener.java      # 메시지 발송 리스너
│   │   ├── NotificationEventListener.java # 알림 이벤트 리스너
│   │   └── DeadLetterListener.java       # DLQ 리스너
│   │
│   └── handler/
│       ├── LogMessageHandler.java        # 로그 처리 핸들러
│       └── MessageSendHandler.java       # 메시지 발송 핸들러 (Stub)
```

---

## 6. 향후 확장 전략

### 6.1 Phase 2: 메시지 발송 로직 추가

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Phase 2 확장 계획                                │
│                                                                          │
│  현재 (Phase 1)              향후 (Phase 2)                             │
│  ┌─────────────────┐        ┌─────────────────────────────────────────┐ │
│  │ MessageSend     │        │ MessageSendHandler                      │ │
│  │ Listener        │        │                                         │ │
│  │                 │        │ ┌─────────────────┐                     │ │
│  │ - 로그 출력     │  ──▶   │ │ SMS Provider    │ (외부 API 연동)     │ │
│  │ - ACK 처리      │        │ ├─────────────────┤                     │ │
│  │                 │        │ │ LMS Provider    │                     │ │
│  └─────────────────┘        │ ├─────────────────┤                     │ │
│                             │ │ 알림톡 Provider │                     │ │
│                             │ └─────────────────┘                     │ │
│                             │                                         │ │
│                             │ + 발송 결과 저장                        │ │
│                             │ + 재시도 정책                           │ │
│                             │ + 발송 이력 관리                        │ │
│                             └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Queue 분리 가능성

```
현재: q.pf.ef.message.send (통합)
          │
          ▼
향후:  q.pf.ef.message.sms        (SMS 전용)
       q.pf.ef.message.lms        (LMS 전용)
       q.pf.ef.message.alimtalk   (알림톡 전용)

※ 정책 확정 후 결정
```

---

## 7. 변경 이력

| 버전 | 일자 | 변경 내용 | 작성자 |
|------|------|----------|--------|
| 1.0.0 | 2026-02-27 | 초안 작성 | Platform Architecture Team |
| 1.1.0 | 2026-03-06 | notification 리소스 추가 (MQ_RESOURCE_REGISTRY.md 동기화) | Claude |

---

END OF DOCUMENT
