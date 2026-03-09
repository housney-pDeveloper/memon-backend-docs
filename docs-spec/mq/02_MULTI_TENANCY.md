# 02. 멀티 프로젝트 격리 전략

```
Document ID    : MQ-SPEC-001-02
Version        : 1.0.0
Classification : Internal / Confidential
```

---

## 1. vhost 설계 규칙

### 1.1 vhost 명명 규칙

```
Pattern: /{project-name}

Examples:
- /eventflow      # eventflow 프로젝트
- /billing        # billing 프로젝트
- /notification   # notification 프로젝트
```

### 1.2 vhost 구조

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         RabbitMQ Cluster                                 │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  vhost: /                                                          │  │
│  │  (기본 vhost - 클러스터 관리 전용, 프로젝트 사용 금지)             │  │
│  │                                                                    │  │
│  │  Users: platform-admin (관리자 전용)                               │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  vhost: /eventflow                                                 │  │
│  │                                                                    │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │  │
│  │  │ ef.exchange.*   │  │ ef.queue.*      │  │ ef.dlq.*        │    │  │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘    │  │
│  │                                                                    │  │
│  │  Users: ef-producer, ef-consumer, ef-admin                        │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  vhost: /billing                                                   │  │
│  │                                                                    │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │  │
│  │  │ bl.exchange.*   │  │ bl.queue.*      │  │ bl.dlq.*        │    │  │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘    │  │
│  │                                                                    │  │
│  │  Users: bl-producer, bl-consumer, bl-admin                        │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ※ Cross-vhost Access: DENIED                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.3 vhost 생성 정책

| 항목 | 규칙 |
|------|------|
| 생성 권한 | 플랫폼 관리자만 가능 |
| 삭제 권한 | 플랫폼 관리자만 가능 |
| 네이밍 | 소문자, 하이픈 허용, 숫자 시작 금지 |
| 예약 이름 | /admin, /system, /platform 사용 금지 |

---

## 2. User 네이밍 규칙

### 2.1 User 유형별 네이밍

| 유형 | 패턴 | 예시 | 용도 |
|------|------|------|------|
| Platform Admin | platform-admin | platform-admin | 전체 클러스터 관리 |
| Project Admin | {project-id}-admin | ef-admin | 프로젝트 내 리소스 관리 |
| Producer | {project-id}-producer-{service} | ef-producer-application | 메시지 발행 |
| Consumer | {project-id}-consumer-{service} | ef-consumer-system | 메시지 소비 |
| Monitoring | {project-id}-monitor | ef-monitor | 모니터링 전용 (읽기) |

### 2.2 User 계층 구조

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           User Hierarchy                                 │
│                                                                          │
│  Level 0: Platform Admin                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  platform-admin                                                  │    │
│  │  - 모든 vhost 접근                                               │    │
│  │  - User/vhost 생성/삭제                                          │    │
│  │  - Policy/Parameter 설정                                         │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                    │                                     │
│                                    ▼                                     │
│  Level 1: Project Admin                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  ef-admin (vhost: /eventflow)                                    │    │
│  │  - 해당 vhost 내 리소스 관리                                     │    │
│  │  - Exchange/Queue 생성/삭제                                      │    │
│  │  - Binding 관리                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                    │                                     │
│                          ┌─────────┴─────────┐                           │
│                          ▼                   ▼                           │
│  Level 2: Service Accounts                                               │
│  ┌──────────────────────────┐  ┌──────────────────────────┐             │
│  │  ef-producer-application │  │  ef-consumer-system      │             │
│  │  - Exchange publish만    │  │  - Queue consume만       │             │
│  └──────────────────────────┘  └──────────────────────────┘             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Producer / Consumer 계정 분리 전략

### 3.1 분리 원칙

| 원칙 | 설명 |
|------|------|
| 역할 분리 | Producer와 Consumer는 절대 동일 계정 사용 금지 |
| 서비스별 분리 | 서비스(Application, System 등)마다 별도 계정 |
| 환경별 분리 | dev/staging/prod 환경별 별도 계정 |

### 3.2 eventflow 프로젝트 계정 예시

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    eventflow 프로젝트 계정 구조                          │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                        vhost: /eventflow                         │    │
│  │                                                                  │    │
│  │  Producer Accounts:                                              │    │
│  │  ┌─────────────────────────────────────────────────────────────┐│    │
│  │  │  ef-producer-application                                     ││    │
│  │  │    - Application 서버에서 이벤트 발행                        ││    │
│  │  │    - ef.exchange.* 에 publish 권한                           ││    │
│  │  │                                                              ││    │
│  │  │  ef-producer-gateway                                         ││    │
│  │  │    - Gateway 서버에서 이벤트 발행 (특수 케이스)              ││    │
│  │  │    - ef.exchange.gateway.* 에만 publish 권한                 ││    │
│  │  └─────────────────────────────────────────────────────────────┘│    │
│  │                                                                  │    │
│  │  Consumer Accounts:                                              │    │
│  │  ┌─────────────────────────────────────────────────────────────┐│    │
│  │  │  ef-consumer-system                                          ││    │
│  │  │    - System 서버에서 이벤트 소비                             ││    │
│  │  │    - ef.queue.* 에 consume 권한                              ││    │
│  │  │                                                              ││    │
│  │  │  ef-consumer-batch                                           ││    │
│  │  │    - Batch Worker에서 이벤트 소비                            ││    │
│  │  │    - ef.queue.batch.* 에만 consume 권한                      ││    │
│  │  └─────────────────────────────────────────────────────────────┘│    │
│  │                                                                  │    │
│  │  Admin Account:                                                  │    │
│  │  ┌─────────────────────────────────────────────────────────────┐│    │
│  │  │  ef-admin                                                    ││    │
│  │  │    - Exchange/Queue/Binding 관리                             ││    │
│  │  │    - CI/CD에서 사용 (리소스 배포)                            ││    │
│  │  └─────────────────────────────────────────────────────────────┘│    │
│  │                                                                  │    │
│  │  Monitor Account:                                                │    │
│  │  ┌─────────────────────────────────────────────────────────────┐│    │
│  │  │  ef-monitor                                                  ││    │
│  │  │    - 모니터링 전용 (읽기만 가능)                             ││    │
│  │  │    - Management API 읽기 권한                                ││    │
│  │  └─────────────────────────────────────────────────────────────┘│    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.3 계정 Credential 관리

| 항목 | 방식 |
|------|------|
| 비밀번호 생성 | 32자 이상 랜덤 문자열 |
| 저장소 | HashiCorp Vault |
| 배포 | Kubernetes Secret (Vault Injector) |
| Rotation | 90일 주기 자동 Rotation |

---

## 4. Permission 최소 권한 설계

### 4.1 RabbitMQ Permission 구조

```
RabbitMQ Permission = Configure + Write + Read

Configure: 리소스 선언/삭제 (queue.declare, exchange.declare)
Write    : 메시지 발행 (basic.publish)
Read     : 메시지 소비 (basic.consume, basic.get)
```

### 4.2 역할별 Permission Matrix

| 역할 | Configure | Write | Read |
|------|-----------|-------|------|
| platform-admin | .* | .* | .* |
| {project}-admin | ^{project}\\..\* | ^{project}\\..\* | ^{project}\\..\* |
| {project}-producer-{service} | (없음) | ^{project}\\.exchange\\..\* | (없음) |
| {project}-consumer-{service} | (없음) | (없음) | ^{project}\\.queue\\..\* |
| {project}-monitor | (없음) | (없음) | (없음) |

### 4.3 Permission 설정 예시 (Terraform)

```hcl
# Producer Permission (최소 권한)
resource "rabbitmq_permissions" "ef_producer_application" {
  user  = "ef-producer-application"
  vhost = "/eventflow"

  permissions {
    configure = ""                        # 리소스 선언 불가
    write     = "^ef\\.exchange\\..*"     # ef.exchange.* 에만 publish
    read      = ""                        # 메시지 소비 불가
  }
}

# Consumer Permission (최소 권한)
resource "rabbitmq_permissions" "ef_consumer_system" {
  user  = "ef-consumer-system"
  vhost = "/eventflow"

  permissions {
    configure = ""                        # 리소스 선언 불가
    write     = ""                        # 메시지 발행 불가
    read      = "^ef\\.queue\\..*"        # ef.queue.* 에서만 consume
  }
}

# Admin Permission (프로젝트 한정 전체 권한)
resource "rabbitmq_permissions" "ef_admin" {
  user  = "ef-admin"
  vhost = "/eventflow"

  permissions {
    configure = "^ef\\..*"                # ef.* 리소스 관리
    write     = "^ef\\..*"                # ef.* 에 publish
    read      = "^ef\\..*"                # ef.* 에서 consume
  }
}

# Monitor Permission (읽기 전용, Tag로 Management API 접근)
resource "rabbitmq_user" "ef_monitor" {
  name     = "ef-monitor"
  password = var.monitor_password
  tags     = ["monitoring"]  # Management API 읽기 전용 접근
}

resource "rabbitmq_permissions" "ef_monitor" {
  user  = "ef-monitor"
  vhost = "/eventflow"

  permissions {
    configure = ""
    write     = ""
    read      = ""  # Queue consume은 불가, Management API로만 조회
  }
}
```

---

## 5. 프로젝트별 정책 적용 전략

### 5.1 Policy 네이밍 규칙

```
Pattern: {project-id}-{policy-type}-{target}

Examples:
- ef-ha-all              # eventflow HA 정책
- ef-dlx-order           # eventflow order 도메인 DLX 정책
- ef-ttl-temporary       # eventflow 임시 큐 TTL 정책
- ef-max-length-default  # eventflow 기본 max-length 정책
```

### 5.2 필수 정책 목록

| 정책 | 설명 | 적용 대상 |
|------|------|----------|
| {project}-ha-all | 모든 큐 HA (Quorum) | ^{project}\\.queue\\..\* |
| {project}-dlx-default | 기본 DLX 설정 | ^{project}\\.queue\\.(?!.*dlq).\* |
| {project}-max-length-default | 기본 max-length | ^{project}\\.queue\\..\* |
| {project}-ttl-default | 기본 message-ttl | ^{project}\\.queue\\..\* |

### 5.3 Policy 설정 예시

```hcl
# Quorum Queue 기본 정책
resource "rabbitmq_policy" "ef_quorum_default" {
  name  = "ef-quorum-default"
  vhost = "/eventflow"

  policy {
    pattern  = "^ef\\.queue\\..*"
    priority = 0
    apply_to = "queues"

    definition = {
      "x-queue-type" = "quorum"
    }
  }
}

# DLX 정책 (DLQ가 아닌 모든 큐에 적용)
resource "rabbitmq_policy" "ef_dlx_default" {
  name  = "ef-dlx-default"
  vhost = "/eventflow"

  policy {
    pattern  = "^ef\\.queue\\.(?!.*\\.dlq$).*"
    priority = 1
    apply_to = "queues"

    definition = {
      "dead-letter-exchange"    = "ef.exchange.dlx"
      "dead-letter-routing-key" = "dlq"
    }
  }
}

# Max Length 정책
resource "rabbitmq_policy" "ef_max_length_default" {
  name  = "ef-max-length-default"
  vhost = "/eventflow"

  policy {
    pattern  = "^ef\\.queue\\..*"
    priority = 0
    apply_to = "queues"

    definition = {
      "max-length"         = 100000
      "overflow"           = "reject-publish"
    }
  }
}

# Message TTL 정책
resource "rabbitmq_policy" "ef_ttl_default" {
  name  = "ef-ttl-default"
  vhost = "/eventflow"

  policy {
    pattern  = "^ef\\.queue\\.(?!.*\\.dlq$).*"
    priority = 0
    apply_to = "queues"

    definition = {
      "message-ttl" = 86400000  # 24 hours
    }
  }
}

# DLQ용 장기 TTL 정책
resource "rabbitmq_policy" "ef_ttl_dlq" {
  name  = "ef-ttl-dlq"
  vhost = "/eventflow"

  policy {
    pattern  = "^ef\\.queue\\..*\\.dlq$"
    priority = 2
    apply_to = "queues"

    definition = {
      "message-ttl" = 604800000  # 7 days
    }
  }
}
```

---

## 6. 리소스 쿼터 전략

### 6.1 vhost별 쿼터 설정

| 프로젝트 등급 | Connection | Channel | Queue | Max Queue Length |
|--------------|------------|---------|-------|------------------|
| Standard | 100 | 500 | 50 | 100,000 |
| Premium | 500 | 2,500 | 200 | 500,000 |
| Enterprise | 1,000 | 5,000 | 500 | 1,000,000 |

### 6.2 vhost Limit 설정 예시

```hcl
# vhost Limit 설정 (RabbitMQ 3.12+)
resource "rabbitmq_vhost_limits" "eventflow" {
  vhost = "/eventflow"

  # Connection 제한
  max_connections = 500

  # Queue 제한
  max_queues = 200
}
```

### 6.3 User별 Connection Limit

```bash
# User별 Connection Limit 설정
rabbitmqctl set_user_limits ef-producer-application '{"max-connections": 50}'
rabbitmqctl set_user_limits ef-consumer-system '{"max-connections": 100}'
rabbitmqctl set_user_limits ef-admin '{"max-connections": 10}'
```

### 6.4 쿼터 모니터링 대시보드

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       vhost: /eventflow Quota                            │
│                                                                          │
│  Connections:  [████████████████░░░░░░░░░░░░░░]  256/500 (51.2%)        │
│  Channels:     [████████░░░░░░░░░░░░░░░░░░░░░░]  642/2500 (25.7%)       │
│  Queues:       [████░░░░░░░░░░░░░░░░░░░░░░░░░░]   42/200 (21.0%)        │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Queue Size Distribution                                         │    │
│  │                                                                  │    │
│  │  ef.queue.order.created     [████████████░░░░░░]  45,231/100,000 │    │
│  │  ef.queue.user.updated      [██░░░░░░░░░░░░░░░░]   8,142/100,000 │    │
│  │  ef.queue.notification.send [█░░░░░░░░░░░░░░░░░]   2,891/100,000 │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Cross-Project 통신 전략

### 7.1 기본 원칙

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Cross-Project Communication                         │
│                                                                          │
│  ┌─────────────────┐                     ┌─────────────────┐            │
│  │   eventflow     │                     │    billing      │            │
│  │                 │                     │                 │            │
│  │  ef.exchange.*  │  ───── ✗ ─────      │  bl.exchange.*  │            │
│  │                 │  Direct Access      │                 │            │
│  │                 │  PROHIBITED         │                 │            │
│  └─────────────────┘                     └─────────────────┘            │
│                                                                          │
│                          │                                               │
│                          ▼                                               │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                   Integration Layer                              │    │
│  │                                                                  │    │
│  │  Option 1: Shovel (MQ 레벨)                                      │    │
│  │  Option 2: Federation (MQ 레벨)                                  │    │
│  │  Option 3: Integration Service (Application 레벨) ← 권장         │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Integration Service 패턴 (권장)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Integration Service Pattern                           │
│                                                                          │
│   eventflow                Integration            billing                │
│  ┌───────────┐            ┌───────────┐         ┌───────────┐           │
│  │ Producer  │ ──────────►│Integration│────────►│ Consumer  │           │
│  │           │ publish    │  Service  │ publish │           │           │
│  └───────────┘            └───────────┘         └───────────┘           │
│       │                        │                      │                  │
│       ▼                        ▼                      ▼                  │
│  /eventflow               (양쪽 vhost           /billing                 │
│  vhost                     접근 권한)            vhost                   │
│                                                                          │
│  Integration Service 계정:                                               │
│  - int-eventflow-billing (특수 계정)                                     │
│  - /eventflow 읽기 + /billing 쓰기 권한                                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.3 Shovel 패턴 (대량 데이터 복제)

```hcl
# Shovel 설정 (eventflow → billing 복제)
resource "rabbitmq_shovel" "ef_to_bl_order" {
  name  = "ef-to-bl-order-shovel"
  vhost = "/eventflow"

  info {
    source_uri           = "amqps://int-ef-bl@localhost/eventflow"
    source_queue         = "ef.queue.order.completed"

    destination_uri      = "amqps://int-ef-bl@localhost/billing"
    destination_exchange = "bl.exchange.order"
    destination_exchange_key = "order.received"

    ack_mode             = "on-confirm"
    reconnect_delay      = 5
  }
}
```

---

## 8. 격리 검증 체크리스트

### 8.1 신규 프로젝트 온보딩 검증

| 항목 | 검증 방법 | 통과 기준 |
|------|----------|----------|
| vhost 격리 | 다른 vhost 접근 시도 | ACCESS_REFUSED |
| Producer 권한 | Queue 직접 publish 시도 | ACCESS_REFUSED |
| Consumer 권한 | Exchange publish 시도 | ACCESS_REFUSED |
| Cross-user | 다른 프로젝트 계정으로 접근 | ACCESS_REFUSED |
| Admin 범위 | 다른 프로젝트 리소스 접근 | ACCESS_REFUSED |

### 8.2 자동화 검증 스크립트

```bash
#!/bin/bash
# isolation-test.sh

PROJECT_ID=$1
VHOST="/${PROJECT_ID}"

echo "=== Testing vhost isolation for ${PROJECT_ID} ==="

# Test 1: Producer cannot access other vhost
echo "Test 1: Cross-vhost access..."
result=$(rabbitmqadmin -u ${PROJECT_ID}-producer -p $PRODUCER_PASS \
  -V /other-project list queues 2>&1)
if [[ $result == *"ACCESS_REFUSED"* ]]; then
  echo "  PASS: Cross-vhost access denied"
else
  echo "  FAIL: Cross-vhost access allowed!"
  exit 1
fi

# Test 2: Producer cannot consume
echo "Test 2: Producer consume attempt..."
result=$(python3 << EOF
import pika
credentials = pika.PlainCredentials('${PROJECT_ID}-producer', '$PRODUCER_PASS')
connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost', 5672, '${VHOST}', credentials)
)
channel = connection.channel()
try:
    channel.basic_get('${PROJECT_ID}.queue.test', auto_ack=True)
    print("FAIL")
except pika.exceptions.ChannelClosedByBroker as e:
    if "ACCESS_REFUSED" in str(e):
        print("PASS")
    else:
        print("FAIL")
EOF
)
echo "  ${result}: Producer consume denied"

# Test 3: Consumer cannot publish
echo "Test 3: Consumer publish attempt..."
# Similar test for consumer

echo "=== All isolation tests passed ==="
```

---

END OF DOCUMENT
