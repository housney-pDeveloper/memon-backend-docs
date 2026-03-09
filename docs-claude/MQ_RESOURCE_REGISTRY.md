# MQ_RESOURCE_REGISTRY.md

RabbitMQ Exchange, Queue, Binding 리소스 등록 현황

이 문서는 EF 프로젝트에서 사용하는 모든 MQ 리소스를 정의한다.
Exchange 또는 Queue 추가/변경 시 반드시 이 문서를 함께 업데이트해야 한다.

---

## 1. vhost

```
/pf-ef
```

---

## 2. Exchanges

| Exchange 이름 | Type | Durable | 용도 | 등록일 |
|---------------|------|---------|------|--------|
| `ex.pf.ef.log.topic` | Topic | Yes | 로그 전용 | 2024-01 |
| `ex.pf.ef.message.topic` | Topic | Yes | 메시지 발송 전용 | 2024-01 |
| `ex.pf.ef.notification.topic` | Topic | Yes | 알림(SSE) 전용 | 2024-01 |
| `ex.pf.ef.commoncode.topic` | Topic | Yes | 공통코드 변경 이벤트 | 2025-03 |
| `ex.pf.ef.dlx` | Topic | Yes | Dead Letter Exchange | 2024-01 |

### 네이밍 규칙

```
ex.{플랫폼}.{프로젝트}.{도메인}.{타입}

예시:
- ex.pf.ef.message.topic
- ex.pf.ef.log.topic
```

---

## 3. Queues

| Queue 이름 | Type | DLX | TTL | Max Length | 용도 | 등록일 |
|------------|------|-----|-----|------------|------|--------|
| `q.pf.ef.log.write` | Quorum | - | - | - | 로그 기록 | 2024-01 |
| `q.pf.ef.message.send` | Quorum | `ex.pf.ef.dlx` | 24h | 50,000 | 메시지 발송 | 2024-01 |
| `q.pf.ef.dlq.message.send` | Quorum | - | 7d | - | 메시지 발송 DLQ | 2024-01 |
| `q.pf.ef.notification` | Quorum | `ex.pf.ef.dlx` | 24h | - | 알림 이벤트 | 2024-01 |
| `q.pf.ef.dlq.notification` | Quorum | - | 7d | - | 알림 이벤트 DLQ | 2025-03 |
| `q.pf.ef.commoncode.changed` | Quorum | `ex.pf.ef.dlx` | 24h | - | 공통코드 변경 | 2025-03 |
| `q.pf.ef.dlq.commoncode.changed` | Quorum | - | 7d | - | 공통코드 변경 DLQ | 2025-03 |

### 네이밍 규칙

```
q.{플랫폼}.{프로젝트}.{도메인}.{액션}

예시:
- q.pf.ef.message.send
- q.pf.ef.log.write

DLQ:
- q.pf.ef.dlq.{도메인}.{액션}
```

### Queue 속성 기준

| 속성 | 기본값 | 비고 |
|------|--------|------|
| Type | Quorum | 고가용성 보장 |
| TTL (일반) | 24시간 | 86,400,000ms |
| TTL (DLQ) | 7일 | 604,800,000ms |
| Max Length | 50,000 | 필요 시 조정 |
| Overflow | reject-publish | 큐 가득 시 발행 거부 |

---

## 4. Bindings

| Exchange | Routing Key | Queue | 용도 | 등록일 |
|----------|-------------|-------|------|--------|
| `ex.pf.ef.log.topic` | `log.access.v1` | `q.pf.ef.log.write` | 접근 로그 | 2024-01 |
| `ex.pf.ef.log.topic` | `log.error.v1` | `q.pf.ef.log.write` | 에러 로그 | 2024-01 |
| `ex.pf.ef.log.topic` | `log.audit.v1` | `q.pf.ef.log.write` | 감사 로그 | 2024-01 |
| `ex.pf.ef.message.topic` | `message.send.v1` | `q.pf.ef.message.send` | 메시지 발송 | 2024-01 |
| `ex.pf.ef.dlx` | `message.send.dlq` | `q.pf.ef.dlq.message.send` | 메시지 DLQ | 2024-01 |
| `ex.pf.ef.notification.topic` | `notification.event.v1` | `q.pf.ef.notification` | 알림 이벤트 | 2024-01 |
| `ex.pf.ef.dlx` | `notification.dlq` | `q.pf.ef.dlq.notification` | 알림 DLQ | 2025-03 |
| `ex.pf.ef.commoncode.topic` | `commoncode.changed.v1` | `q.pf.ef.commoncode.changed` | 공통코드 변경 | 2025-03 |
| `ex.pf.ef.dlx` | `commoncode.changed.dlq` | `q.pf.ef.dlq.commoncode.changed` | 공통코드 DLQ | 2025-03 |

### Routing Key 규칙

```
{도메인}.{액션}.{버전}

예시:
- message.send.v1
- log.access.v1
- notification.event.v1

DLQ:
- {도메인}.{액션}.dlq
```

---

## 5. 메시지 흐름도

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              RabbitMQ (/pf-ef)                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────┐                                                │
│  │ ex.pf.ef.log.topic      │                                                │
│  │      (Topic)            │                                                │
│  └───────────┬─────────────┘                                                │
│              │                                                              │
│              ├── log.access.v1 ──┐                                          │
│              ├── log.error.v1 ───┼──> q.pf.ef.log.write                     │
│              └── log.audit.v1 ───┘         │                                │
│                                            ▼                                │
│                                    [worker-log]                             │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────┐                                                │
│  │ ex.pf.ef.message.topic  │                                                │
│  │      (Topic)            │                                                │
│  └───────────┬─────────────┘                                                │
│              │                                                              │
│              └── message.send.v1 ──> q.pf.ef.message.send                   │
│                                            │                                │
│                                            ▼                                │
│                                    [worker-message]                         │
│                                            │                                │
│                                            │ (NACK/Reject)                  │
│                                            ▼                                │
│  ┌─────────────────────────┐      ┌─────────────────────────┐               │
│  │ ex.pf.ef.dlx            │◀─────│ DLX Routing             │               │
│  │   (Dead Letter)         │      └─────────────────────────┘               │
│  └───────────┬─────────────┘                                                │
│              │                                                              │
│              └── message.send.dlq ──> q.pf.ef.dlq.message.send              │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────┐                                                │
│  │ ex.pf.ef.notification   │                                                │
│  │      .topic             │                                                │
│  └───────────┬─────────────┘                                                │
│              │                                                              │
│              └── notification.event.v1 ──> q.pf.ef.notification             │
│                                                  │                          │
│                                                  ▼                          │
│                                              [api]                          │
│                                                  │                          │
│                                                  │ (NACK/Reject)            │
│                                                  ▼                          │
│              ┌── notification.dlq ──> q.pf.ef.dlq.notification              │
│              │                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────┐                                                │
│  │ ex.pf.ef.commoncode     │                                                │
│  │      .topic             │                                                │
│  └───────────┬─────────────┘                                                │
│              │                                                              │
│              └── commoncode.changed.v1 ──> q.pf.ef.commoncode.changed       │
│                                                  │                          │
│                                                  ▼                          │
│                                          [worker-mq]                        │
│                                                  │                          │
│                                                  │ (NACK/Reject)            │
│                                                  ▼                          │
│              ┌── commoncode.changed.dlq ──> q.pf.ef.dlq.commoncode.changed  │
│              │                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Producer / Consumer 매핑

| Producer | 위치 | Exchange | Routing Key |
|----------|------|----------|-------------|
| `CommonCodeEventPublisher` | Application | `ex.pf.ef.commoncode.topic` | `commoncode.changed.v1` |
| `MessageSendEventPublisher` | Application | `ex.pf.ef.message.topic` | `message.send.v1` |
| `OutboxPollingPublisher` | Application | (Outbox 테이블 기반) | - |
| `ImmediateOutboxPublisher` | Application | (즉시 발행) | - |

**참고**: Log/Notification 이벤트는 Gateway에서 직접 발행하거나 별도 Publisher 구현 필요

| Consumer | 위치 | Queue | Profile |
|----------|------|-------|---------|
| `CommonCodeEventListener` | System | `q.pf.ef.commoncode.changed` | worker-mq |
| `LogMessageListener` | System | `q.pf.ef.log.write` | worker-mq |
| `MessageSendListener` | System | `q.pf.ef.message.send` | worker-mq |
| `NotificationEventListener` | System | `q.pf.ef.notification` | worker-mq |
| `DeadLetterListener` | System | DLQ 처리 | worker-mq |

---

## 7. 코드 위치 및 서버 역할 분담

### 7.1 서버별 역할

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           MQ 리소스 관리 역할 분담                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────┐     ┌─────────────────────────────┐        │
│  │     Application Server      │     │       System Server         │        │
│  │       (32_application)      │     │        (33_system)          │        │
│  ├─────────────────────────────┤     ├─────────────────────────────┤        │
│  │                             │     │                             │        │
│  │  역할: Publisher 전용        │     │  역할: Consumer + 리소스 관리 │        │
│  │                             │     │                             │        │
│  │  ✅ 메시지 발행              │     │  ✅ 메시지 소비              │        │
│  │  ✅ Exchange/Routing 상수   │     │  ✅ Exchange/Queue Bean 정의 │        │
│  │  ❌ Exchange/Queue Bean 없음│     │  ✅ 리소스 동기화 (삭제/재생성)│        │
│  │  ❌ Consumer 없음           │     │  ✅ auto-declare            │        │
│  │                             │     │                             │        │
│  └─────────────────────────────┘     └─────────────────────────────┘        │
│              │                                    │                         │
│              │ publish                            │ consume                 │
│              ▼                                    ▼                         │
│        ┌─────────────────────────────────────────────────┐                  │
│        │                   RabbitMQ                       │                  │
│        │              (Single Source: System)             │                  │
│        └─────────────────────────────────────────────────┘                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 코드 위치

**System Server (리소스 마스터)**

| 파일 | 설명 |
|------|------|
| `MqConstants.java` | Exchange/Queue/Routing Key 상수 정의 |
| `MqResourceConfig.java` | Exchange/Queue/Binding Bean 정의 (auto-declare) |
| `MqConsumerConfig.java` | Consumer 설정 (RabbitTemplate, RabbitAdmin) |
| `MqResourceSynchronizer.java` | 리소스 동기화 로직 (삭제/재생성) |
| `RabbitMqManagementClient.java` | Management API 클라이언트 |

```
33_system/src/main/java/kr/fingate/pf/system/mq/
├── config/
│   ├── MqConstants.java          ← 상수 정의
│   ├── MqResourceConfig.java     ← Bean 정의 (Exchange/Queue/Binding)
│   └── MqConsumerConfig.java     ← Consumer 설정 (RabbitTemplate, RabbitAdmin)
├── sync/
│   ├── MqResourceSyncProperties.java
│   ├── MqResourceSynchronizer.java
│   ├── MqResourceSyncRunner.java
│   └── RabbitMqManagementClient.java
└── listener/
    ├── CommonCodeEventListener.java   ← 공통코드 변경 Consumer
    ├── LogMessageListener.java        ← 로그 메시지 Consumer
    ├── MessageSendListener.java       ← 메시지 발송 Consumer
    ├── NotificationEventListener.java ← 알림 이벤트 Consumer
    └── DeadLetterListener.java        ← DLQ 처리
```

**Application Server (Publisher 전용)**

| 파일 | 설명 |
|------|------|
| `RabbitMqConfig.java` | Exchange/Routing Key 상수 + RabbitTemplate 설정 |
| `CommonCodeEventPublisher.java` | 공통코드 변경 이벤트 발행 |
| `MessageSendEventPublisher.java` | 메시지 발송 이벤트 발행 |
| `OutboxPollingPublisher.java` | Outbox 테이블 기반 폴링 발행 |
| `ImmediateOutboxPublisher.java` | 즉시 Outbox 발행 |

```
32_application/src/main/java/kr/fingate/pf/application/common/mq/
├── config/
│   └── RabbitMqConfig.java             ← 상수 + RabbitTemplate (Bean 선언 없음)
└── publisher/
    ├── CommonCodeEventPublisher.java   ← 공통코드 변경 이벤트
    ├── MessageSendEventPublisher.java  ← 메시지 발송 이벤트
    ├── OutboxPollingPublisher.java     ← Outbox 폴링 발행
    └── ImmediateOutboxPublisher.java   ← 즉시 Outbox 발행
```

### 7.3 리소스 관리 원칙

| 원칙 | 설명 |
|------|------|
| **Single Source of Truth** | System 서버의 Bean 정의가 유일한 리소스 정의 |
| **Publisher는 상수만** | Application은 Exchange/Routing Key 상수만 참조 |
| **Consumer는 System만** | @RabbitListener는 System 서버에서만 사용 |
| **동기화는 worker-mq만** | 리소스 삭제/재생성은 worker-mq 프로필에서만 수행 |

---

## 8. 변경 이력

| 날짜 | 변경 내용 | 작성자 |
|------|----------|--------|
| 2024-01 | 초기 구성 (log, message, notification) | - |
| 2025-03 | 문서 정리 및 등록 | Claude |
| 2025-03 | MQ 리소스 비교 분석 정책 추가 | Claude |
| 2025-03 | User 및 권한 정책 추가 | Claude |
| 2025-03 | Publisher/Consumer 설정 파일 반영 | Claude |
| 2025-03 | 환경별 User 전략 정책 추가 (Docker: 단일계정, Amazon MQ: 분리) | Claude |
| 2026-03 | notification DLQ 추가 (q.pf.ef.dlq.notification) | Claude |
| 2026-03 | commoncode 리소스 추가 (네이밍 규칙 적용) | Claude |
| 2026-03 | MQ 리소스 동기화 기능 추가 (Code as Infrastructure) | Claude |
| 2026-03 | 서버 역할 분담 정책 추가 (Application: Publisher, System: Consumer+리소스) | Claude |
| 2026-03 | Application Bean 선언 제거, System 단일 관리로 변경 | Claude |
| 2026-03 | Management URL 설정 통합 (spring.rabbitmq.management-url) | Claude |
| 2026-03 | Producer/Consumer 매핑 실제 코드와 동기화, management-url 기본값 제거 반영 | Claude |

---

## 9. MQ 리소스 비교 분석

### 9.1 접속 정보

| 환경 | Host | Port | vhost |
|------|------|------|-------|
| develop | eventflow-web-dev.mq.fingate.kr | 15672 | /pf-ef |
| production | (TBD) | 15672 | /pf-ef |

**인증 정보**: AWS Parameter Store 또는 환경별 설정 참조

---

### 9.2 비교 분석 스크립트

아래 스크립트를 실행하여 문서와 실제 MQ 리소스를 비교한다.

#### 전체 비교 스크립트

```bash
#!/bin/bash
# mq-resource-compare.sh
# 사용법: ./mq-resource-compare.sh <host> <username> <password>

HOST=${1:-"eventflow-web-dev.mq.fingate.kr"}
PORT=${2:-"15672"}
USERNAME=${3:-"platform_ef"}
PASSWORD=${4:-"platform_ef"}
VHOST="%2Fpf-ef"  # URL encoded /pf-ef

echo "=========================================="
echo "MQ Resource Comparison Report"
echo "Host: $HOST:$PORT"
echo "vhost: /pf-ef"
echo "Time: $(date)"
echo "=========================================="

# 문서에 정의된 리소스
DOC_EXCHANGES=("ex.pf.ef.log.topic" "ex.pf.ef.message.topic" "ex.pf.ef.notification.topic" "ex.pf.ef.dlx")
DOC_QUEUES=("q.pf.ef.log.write" "q.pf.ef.message.send" "q.pf.ef.dlq.message.send" "q.pf.ef.notification")

echo ""
echo "[1] Exchange 비교"
echo "------------------------------------------"
ACTUAL_EXCHANGES=$(curl -s -u $USERNAME:$PASSWORD "http://$HOST:$PORT/api/exchanges/$VHOST" | jq -r '.[].name' | grep -v "^$" | grep -v "^amq\.")

echo "문서 정의:"
for ex in "${DOC_EXCHANGES[@]}"; do echo "  - $ex"; done

echo ""
echo "실제 등록:"
echo "$ACTUAL_EXCHANGES" | while read ex; do echo "  - $ex"; done

echo ""
echo "차이점:"
for ex in "${DOC_EXCHANGES[@]}"; do
    if ! echo "$ACTUAL_EXCHANGES" | grep -q "^$ex$"; then
        echo "  [MISSING] $ex (문서에 있으나 실제 없음)"
    fi
done
echo "$ACTUAL_EXCHANGES" | while read ex; do
    if [[ ! " ${DOC_EXCHANGES[@]} " =~ " $ex " ]] && [[ -n "$ex" ]]; then
        echo "  [EXTRA] $ex (실제에 있으나 문서에 없음)"
    fi
done

echo ""
echo "[2] Queue 비교"
echo "------------------------------------------"
ACTUAL_QUEUES=$(curl -s -u $USERNAME:$PASSWORD "http://$HOST:$PORT/api/queues/$VHOST" | jq -r '.[].name')

echo "문서 정의:"
for q in "${DOC_QUEUES[@]}"; do echo "  - $q"; done

echo ""
echo "실제 등록:"
echo "$ACTUAL_QUEUES" | while read q; do echo "  - $q"; done

echo ""
echo "차이점:"
for q in "${DOC_QUEUES[@]}"; do
    if ! echo "$ACTUAL_QUEUES" | grep -q "^$q$"; then
        echo "  [MISSING] $q (문서에 있으나 실제 없음)"
    fi
done
echo "$ACTUAL_QUEUES" | while read q; do
    if [[ ! " ${DOC_QUEUES[@]} " =~ " $q " ]] && [[ -n "$q" ]]; then
        echo "  [EXTRA] $q (실제에 있으나 문서에 없음)"
    fi
done

echo ""
echo "[3] Binding 비교"
echo "------------------------------------------"
curl -s -u $USERNAME:$PASSWORD "http://$HOST:$PORT/api/bindings/$VHOST" | \
    jq -r '.[] | select(.source != "") | "\(.source) --[\(.routing_key)]--> \(.destination)"'

echo ""
echo "=========================================="
echo "비교 완료"
echo "=========================================="
```

#### 간단 조회 명령어

```bash
# Exchange 목록 조회
curl -s -u platform_ef:platform_ef \
  "http://eventflow-web-dev.mq.fingate.kr:15672/api/exchanges/%2Fpf-ef" | \
  jq -r '.[] | select(.name | startswith("ex.pf.ef")) | .name'

# Queue 목록 조회
curl -s -u platform_ef:platform_ef \
  "http://eventflow-web-dev.mq.fingate.kr:15672/api/queues/%2Fpf-ef" | \
  jq -r '.[].name'

# Binding 목록 조회
curl -s -u platform_ef:platform_ef \
  "http://eventflow-web-dev.mq.fingate.kr:15672/api/bindings/%2Fpf-ef" | \
  jq -r '.[] | select(.source != "") | "\(.source) --> \(.destination) [routing: \(.routing_key)]"'

# Queue 상세 정보 (메시지 수, 소비자 수)
curl -s -u platform_ef:platform_ef \
  "http://eventflow-web-dev.mq.fingate.kr:15672/api/queues/%2Fpf-ef" | \
  jq -r '.[] | "\(.name): messages=\(.messages), consumers=\(.consumers)"'
```

---

### 9.3 비교 분석 정책

#### 정기 점검

| 주기 | 점검 항목 | 담당 |
|------|----------|------|
| 배포 전 | 문서 vs 실제 리소스 일치 여부 | 개발자 |
| 주 1회 | DLQ 메시지 적체 여부 | 운영팀 |
| 월 1회 | 미사용 리소스 정리 | 인프라팀 |

#### 불일치 발견 시 조치

| 상황 | 조치 |
|------|------|
| 문서에 있으나 실제 없음 | MqResourceConfig.java 확인 후 리소스 생성 또는 문서 삭제 |
| 실제에 있으나 문서에 없음 | 사용 여부 확인 후 문서 추가 또는 리소스 삭제 |
| 속성 불일치 (TTL, DLX 등) | MqResourceConfig.java 기준으로 실제 리소스 재생성 |

#### Claude 작업 규칙

MQ 관련 작업 수행 시 Claude는 다음을 준수한다:

1. **리소스 추가 시**
   - MqConstants.java에 상수 추가
   - MqResourceConfig.java에 Bean 정의
   - 이 문서의 해당 섹션 업데이트
   - 변경 이력 기록

2. **리소스 삭제 시**
   - 사용처 확인 (Producer/Consumer 검색)
   - 코드에서 참조 제거
   - 이 문서에서 해당 항목 삭제
   - 변경 이력 기록

3. **비교 분석 요청 시**
   - 9.2 스크립트 안내 또는 실행
   - 불일치 항목 리포트
   - 조치 방안 제시

---

### 9.4 API Reference

RabbitMQ Management HTTP API 주요 엔드포인트:

| 엔드포인트 | 설명 |
|------------|------|
| `GET /api/overview` | 클러스터 개요 |
| `GET /api/exchanges/{vhost}` | Exchange 목록 |
| `GET /api/queues/{vhost}` | Queue 목록 |
| `GET /api/bindings/{vhost}` | Binding 목록 |
| `GET /api/queues/{vhost}/{queue}` | 특정 Queue 상세 |
| `DELETE /api/queues/{vhost}/{queue}` | Queue 삭제 |

**참고**: vhost는 URL 인코딩 필요 (`/pf-ef` → `%2Fpf-ef`)

---

## 10. User 및 권한 정책

---

### 10.1 환경별 User 전략

| 환경 | MQ 유형 | User 전략 | 비고 |
|------|---------|-----------|------|
| local | Docker (Self-hosted) | 단일 계정 (admin) | 환경변수 제약 |
| develop | Docker (Self-hosted) | 단일 계정 (admin) | 환경변수 제약 |
| staging | Amazon MQ | Publisher/Consumer 분리 | 관리형 서비스 |
| production | Amazon MQ | Publisher/Consumer 분리 | 관리형 서비스 |

---

### 10.2 Docker 환경 (local/develop)

Docker RabbitMQ는 환경변수로 **단일 기본 계정만** 생성 가능하다.
추가 계정 생성은 definitions.json import가 필요하므로, 개발 환경에서는 단일 계정을 사용한다.

**docker-compose 설정:**
```yaml
environment:
  RABBITMQ_DEFAULT_USER: pf_ef_admin
  RABBITMQ_DEFAULT_PASS: pf_ef_admin
  RABBITMQ_DEFAULT_VHOST: /pf-ef
```

**AWS Parameter Store 설정 (local/develop):**

| Parameter Path | 값 | 비고 |
|----------------|-----|------|
| `rabbitmq.username.publisher` | pf_ef_admin | Application용 |
| `rabbitmq.password.publisher` | pf_ef_admin | Application용 |
| `rabbitmq.username.consumer` | pf_ef_admin | System용 |
| `rabbitmq.password.consumer` | pf_ef_admin | System용 |

---

### 10.3 Amazon MQ 환경 (staging/production)

Amazon MQ for RabbitMQ는 관리형 서비스로 사용자 관리 API를 지원한다.
Publisher/Consumer 분리를 통해 최소 권한 원칙을 적용한다.

**User 구성:**

| User | 용도 | 사용 서버 |
|------|------|----------|
| `pf_ef_publisher` | Exchange에 메시지 발행 | Application |
| `pf_ef_consumer` | Queue에서 메시지 소비 | System |
| `pf_ef_admin` | 리소스 생성/삭제/관리 | CI/CD, 운영 |

**권한 매트릭스:**

| User | configure | write | read | 설명 |
|------|-----------|-------|------|------|
| `pf_ef_publisher` | - | `ex\.pf\.ef\..*` | - | Exchange에 publish만 가능 |
| `pf_ef_consumer` | - | - | `q\.pf\.ef\..*` | Queue에서 consume만 가능 |
| `pf_ef_admin` | `.*` | `.*` | `.*` | 전체 관리 권한 |

**권한 설명:**
- **configure**: Exchange, Queue, Binding 생성/삭제
- **write**: Exchange에 메시지 발행 (publish)
- **read**: Queue에서 메시지 소비 (consume)

**AWS Parameter Store 설정 (staging/production):**

| Parameter Path | 값 | 비고 |
|----------------|-----|------|
| `rabbitmq.username.publisher` | pf_ef_publisher | Application용 |
| `rabbitmq.password.publisher` | (비밀번호) | Application용 |
| `rabbitmq.username.consumer` | pf_ef_consumer | System용 |
| `rabbitmq.password.consumer` | (비밀번호) | System용 |

---

### 10.4 Amazon MQ User 생성 방법

Amazon MQ 콘솔 또는 AWS CLI로 사용자를 생성한다.

```bash
# AWS CLI를 통한 사용자 생성
aws mq create-user \
  --broker-id <broker-id> \
  --username pf_ef_publisher \
  --password <password>

aws mq create-user \
  --broker-id <broker-id> \
  --username pf_ef_consumer \
  --password <password>
```

권한 설정은 RabbitMQ Management API를 통해 수행:

```bash
# Publisher 권한 설정
curl -u admin:<admin-password> -X PUT \
  "https://<broker-endpoint>:15671/api/permissions/%2Fpf-ef/pf_ef_publisher" \
  -H "Content-Type: application/json" \
  -d '{"configure":"","write":"ex\\.pf\\.ef\\..*","read":""}'

# Consumer 권한 설정
curl -u admin:<admin-password> -X PUT \
  "https://<broker-endpoint>:15671/api/permissions/%2Fpf-ef/pf_ef_consumer" \
  -H "Content-Type: application/json" \
  -d '{"configure":"","write":"","read":"q\\.pf\\.ef\\..*"}'
```

---

### 10.5 설정 파일

코드는 환경에 관계없이 동일한 키를 사용한다. 환경별 값은 Parameter Store에서 관리한다.

**Application Server (32_application/src/main/resources/application.yml)**

```yaml
spring:
  # RabbitMQ 설정 (Publisher 전용 계정)
  rabbitmq:
    host: ${rabbitmq.host}
    port: ${rabbitmq.port}
    username: ${rabbitmq.username.publisher}
    password: ${rabbitmq.password.publisher}
    virtual-host: ${rabbitmq.vhost}
```

**System Server (33_system/src/main/resources/application.yml)**

```yaml
spring:
  # RabbitMQ 설정 (Consumer 전용 계정)
  rabbitmq:
    host: ${rabbitmq.host}
    port: ${rabbitmq.port}
    username: ${rabbitmq.username.consumer}
    password: ${rabbitmq.password.consumer}
    virtual-host: /pf-ef
    # Management API URL (리소스 동기화용) - Parameter Store에서 환경별 관리
    management-url: ${rabbitmq.management-url}
```

---

### 10.6 Parameter Store 구성

**경로 패턴**: `/fingate/config/platform/ef/{env}/rabbitmq.*`

| Parameter Path | local/develop | staging/production |
|----------------|---------------|-------------------|
| `rabbitmq.host` | localhost / MQ 호스트 | Amazon MQ Endpoint |
| `rabbitmq.port` | 5672 | 5671 (TLS) |
| `rabbitmq.vhost` | /pf-ef | /pf-ef |
| `rabbitmq.management-url` | http://localhost:15672 | https://{host}:15671 |
| `rabbitmq.username.publisher` | pf_ef_admin | pf_ef_publisher |
| `rabbitmq.password.publisher` | pf_ef_admin | (비밀번호) |
| `rabbitmq.username.consumer` | pf_ef_admin | pf_ef_consumer |
| `rabbitmq.password.consumer` | pf_ef_admin | (비밀번호) |

---

### 10.8 vhost 전략

**현행 유지**: 환경별 MQ 인스턴스 분리 방식

| 환경 | MQ 유형 | MQ Host | vhost |
|------|---------|---------|-------|
| local | Docker | localhost | `/pf-ef` |
| develop | Docker | eventflow-dev.mq.fingate.kr | `/pf-ef` |
| staging | Amazon MQ | (TBD) | `/pf-ef` |
| production | Amazon MQ | (TBD) | `/pf-ef` |

**환경 격리 방식**: vhost 분리가 아닌 **MQ 인스턴스 분리**로 구현
- 장점: 완전한 격리, 환경별 독립적 확장
- 단점: 인프라 비용 증가

---

## 11. MQ 리소스 동기화 (Code as Infrastructure)

### 11.1 개요

MQ 리소스를 코드(Bean)로 완전히 관리하는 선언적 동기화 기능.

```
┌─────────────────┐     ┌─────────────────┐
│   Bean 정의     │ ◀══▶│    RabbitMQ     │
│ (Single Source  │     │    (실제 상태)   │
│   of Truth)     │     │                 │
└─────────────────┘     └─────────────────┘
         │
         └── 생성 + 삭제 + 재생성 동기화
```

**동작 방식:**
- Bean에 있고 RabbitMQ에 없음 → 생성 (기존 auto-declare)
- Bean에 없고 RabbitMQ에 있음 → **삭제** (신규 기능)
- Arguments 불일치 → **삭제 후 재생성** (신규 기능)

---

### 11.2 설정

```yaml
# application.yml (System Server)
spring:
  rabbitmq:
    host: ${rabbitmq.host}
    port: ${rabbitmq.port}
    username: ${rabbitmq.username.consumer}
    password: ${rabbitmq.password.consumer}
    virtual-host: /pf-ef
    # Management API URL (리소스 동기화용) - Parameter Store에서 환경별 관리
    management-url: ${rabbitmq.management-url}

# 동기화 설정 (Management API 인증은 spring.rabbitmq.username/password 사용)
mq:
  sync:
    enabled: true                      # 동기화 활성화
    dry-run: false                     # false: 실제 삭제 수행
    delete-non-empty-queues: false     # 메시지 있는 큐 삭제 방지
    recreate-on-mismatch: true         # arguments 불일치 시 재생성
    managed-exchange-prefixes:
      - "ex.pf.ef."                    # 관리 대상 Exchange prefix
    managed-queue-prefixes:
      - "q.pf.ef."                     # 관리 대상 Queue prefix
```

---

### 11.3 환경별 설정

| 환경 | enabled | dry-run | 동작 |
|------|---------|---------|------|
| local | true | true | 로그만 출력 (테스트용) |
| develop | true | false | 실제 동기화 수행 |
| staging | true | false | 실제 동기화 수행 |
| production | false | - | 비활성화 (인프라 관리) |

---

### 11.4 안전장치

| 위험 | 안전장치 |
|------|---------|
| 다른 서비스 리소스 삭제 | prefix 필터 (`ex.pf.ef.*`, `q.pf.ef.*` 만 관리) |
| 메시지 유실 | `delete-non-empty-queues: false` (메시지 있으면 스킵) |
| 실수로 삭제 | `dry-run: true` (실제 삭제 없이 로그만 출력) |
| 운영 사고 | production에서 비활성화 |

---

### 11.5 실행 로그 예시

```
[MQ-SYNC] ================================================
[MQ-SYNC] MQ Resource Synchronization Starting...
[MQ-SYNC] ================================================
[MQ-SYNC] Configuration:
[MQ-SYNC]   - enabled: true
[MQ-SYNC]   - dry-run: false
[MQ-SYNC]   - delete-non-empty-queues: false
[MQ-SYNC]   - managed-exchange-prefixes: [ex.pf.ef.]
[MQ-SYNC]   - managed-queue-prefixes: [q.pf.ef.]
[MQ-SYNC] ------------------------------------------------
[MQ-SYNC] Declared resources - Exchanges: 5, Queues: 7, Bindings: 9
[MQ-SYNC] Actual resources - Exchanges: 7, Queues: 10, Bindings: 12
[MQ-SYNC] ========== RESOURCE DIFF ==========
[MQ-SYNC] Exchanges to DELETE (2):
[MQ-SYNC]   - common-code.exchange
[MQ-SYNC]   - common-code.exchange.dlx
[MQ-SYNC] Queues to DELETE (3):
[MQ-SYNC]   - common-code.changed.queue
[MQ-SYNC]   - common-code.changed.queue.dlq
[MQ-SYNC]   - q.pf.ef.notification (arguments mismatch)
[MQ-SYNC] =====================================
[MQ-SYNC] Deleted: 2 exchanges, 3 queues, 3 bindings
[MQ-SYNC] Recreated: 1 queues
[MQ-SYNC] ================================================
```

---

### 11.6 동기화 클래스 상세

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         MQ 리소스 동기화 클래스 구조                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────┐                                                │
│  │  MqResourceSyncRunner   │  ← 진입점 (ApplicationRunner, @Order 100)      │
│  │                         │     auto-declare 이후 실행                      │
│  └───────────┬─────────────┘                                                │
│              │                                                              │
│              ▼                                                              │
│  ┌─────────────────────────┐                                                │
│  │ MqResourceSynchronizer  │  ← 핵심 로직                                   │
│  │                         │     Bean 수집 → 실제 조회 → 비교 → 삭제/재생성   │
│  └───────────┬─────────────┘                                                │
│              │                                                              │
│      ┌───────┴───────┐                                                      │
│      │               │                                                      │
│      ▼               ▼                                                      │
│  ┌─────────┐   ┌─────────────────────────┐                                  │
│  │ Spring  │   │ RabbitMqManagementClient │  ← Management API 호출          │
│  │ Context │   │                         │     GET /api/queues (조회)       │
│  │ (Beans) │   │                         │     DELETE /api/queues (삭제)    │
│  └─────────┘   └─────────────────────────┘                                  │
│                         │                                                   │
│                         ▼                                                   │
│                  ┌─────────────┐                                            │
│                  │  RabbitMQ   │                                            │
│                  │  HTTP API   │                                            │
│                  │   :15672    │                                            │
│                  └─────────────┘                                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

| 클래스 | 역할 | 주요 메서드 |
|--------|------|------------|
| `MqResourceSyncRunner` | 서버 시작 시 동기화 실행 | `run()` |
| `MqResourceSynchronizer` | Bean/실제 비교 및 동기화 로직 | `synchronize()`, `collectDeclaredResources()`, `fetchActualResources()`, `applyChanges()` |
| `RabbitMqManagementClient` | Management API 호출 | `getQueues()`, `deleteQueue()`, `getExchanges()`, `deleteExchange()` |
| `MqResourceSyncProperties` | 동기화 설정 | enabled, dryRun, managedPrefixes |

---

### 11.7 운영 가이드

**리소스 추가:**
1. `MqConstants.java`에 상수 추가
2. `MqResourceConfig.java`에 Bean 추가
3. 서버 재시작 → 자동 생성

**리소스 삭제:**
1. `MqConstants.java`에서 상수 삭제
2. `MqResourceConfig.java`에서 Bean 삭제
3. 서버 재시작 → 자동 삭제 (sync 활성화 시)

**Arguments 변경:**
1. Bean arguments 수정
2. 서버 재시작 → 자동 삭제 후 재생성

---

END OF FILE
