# 01. 전체 아키텍처 설계

```
Document ID    : MQ-SPEC-001-01
Version        : 1.0.0
Classification : Internal / Confidential
```

---

## 1. RabbitMQ Cluster 구성

### 1.1 환경별 구성

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Production Cluster                              │
│                                                                          │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                 │
│   │   Node 1    │    │   Node 2    │    │   Node 3    │                 │
│   │   (Disc)    │◄──►│   (Disc)    │◄──►│   (Disc)    │                 │
│   │   Primary   │    │   Mirror    │    │   Mirror    │                 │
│   └─────────────┘    └─────────────┘    └─────────────┘                 │
│         │                  │                  │                          │
│         └──────────────────┼──────────────────┘                          │
│                            │                                             │
│                   ┌────────▼────────┐                                    │
│                   │  Quorum Queue   │                                    │
│                   │  (Raft-based)   │                                    │
│                   └─────────────────┘                                    │
└─────────────────────────────────────────────────────────────────────────┘
```

| 환경 | 노드 수 | Queue 타입 | 디스크 타입 | HA 전략 |
|------|---------|------------|-------------|---------|
| local | 1 | Classic | RAM | 없음 |
| dev | 1 | Classic | Disc | 없음 |
| staging | 3 | Quorum | Disc SSD | Quorum |
| prod | 3+ | Quorum | Disc SSD (NVMe) | Quorum + AZ 분산 |

### 1.2 Quorum Queue 선택 근거

| 항목 | Classic Mirrored | Quorum Queue |
|------|------------------|--------------|
| 일관성 | Eventual | Strong (Raft) |
| 데이터 안전성 | 낮음 | 높음 |
| 성능 | 높음 | 중간 |
| 운영 복잡도 | 높음 | 낮음 |
| RabbitMQ 권장 | Deprecated | Recommended |

**결론**: 금융권 데이터 일관성 요구사항 충족을 위해 **Quorum Queue** 사용

### 1.3 노드 스펙 권장사항

| 환경 | CPU | Memory | Disk | Network |
|------|-----|--------|------|---------|
| dev | 2 vCPU | 4GB | 50GB SSD | 1Gbps |
| staging | 4 vCPU | 8GB | 100GB SSD | 10Gbps |
| prod | 8 vCPU | 16GB | 500GB NVMe | 10Gbps |

---

## 2. Kubernetes 배포 구조

### 2.1 전체 구조

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                                │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    Namespace: platform-mq                        │    │
│  │                                                                  │    │
│  │  ┌────────────────────────────────────────────────────────────┐ │    │
│  │  │                    StatefulSet: rabbitmq                    │ │    │
│  │  │                                                             │ │    │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                  │ │    │
│  │  │  │ Pod-0    │  │ Pod-1    │  │ Pod-2    │                  │ │    │
│  │  │  │ rabbitmq │  │ rabbitmq │  │ rabbitmq │                  │ │    │
│  │  │  └────┬─────┘  └────┬─────┘  └────┬─────┘                  │ │    │
│  │  │       │             │             │                         │ │    │
│  │  │  ┌────▼─────┐  ┌────▼─────┐  ┌────▼─────┐                  │ │    │
│  │  │  │ PVC-0    │  │ PVC-1    │  │ PVC-2    │                  │ │    │
│  │  │  │ 500Gi    │  │ 500Gi    │  │ 500Gi    │                  │ │    │
│  │  │  └──────────┘  └──────────┘  └──────────┘                  │ │    │
│  │  └────────────────────────────────────────────────────────────┘ │    │
│  │                                                                  │    │
│  │  ┌─────────────────┐  ┌─────────────────┐                       │    │
│  │  │ Service:        │  │ Service:        │                       │    │
│  │  │ rabbitmq        │  │ rabbitmq-mgmt   │                       │    │
│  │  │ (ClusterIP)     │  │ (ClusterIP)     │                       │    │
│  │  │ Port: 5671      │  │ Port: 15672     │                       │    │
│  │  └─────────────────┘  └─────────────────┘                       │    │
│  │                                                                  │    │
│  │  ┌─────────────────┐  ┌─────────────────┐                       │    │
│  │  │ ConfigMap:      │  │ Secret:         │                       │    │
│  │  │ rabbitmq-config │  │ rabbitmq-secret │                       │    │
│  │  └─────────────────┘  └─────────────────┘                       │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                 Namespace: eventflow (Project)                   │    │
│  │                                                                  │    │
│  │  ┌─────────────────┐  ┌─────────────────┐                       │    │
│  │  │ ef-application  │  │ ef-system       │                       │    │
│  │  │ (Producer)      │  │ (Consumer)      │                       │    │
│  │  └─────────────────┘  └─────────────────┘                       │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Namespace 분리 전략

| Namespace | 용도 | 포함 리소스 |
|-----------|------|------------|
| platform-mq | MQ 클러스터 전용 | RabbitMQ StatefulSet, Service, ConfigMap |
| platform-mq-monitoring | MQ 모니터링 전용 | Prometheus, Grafana, Alertmanager |
| eventflow | eventflow 프로젝트 | Application, System, Gateway |
| {project-name} | 기타 프로젝트 | 각 프로젝트 워크로드 |

### 2.3 Pod Anti-Affinity 설정

```yaml
# RabbitMQ StatefulSet Pod Anti-Affinity
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: rabbitmq
          topologyKey: kubernetes.io/hostname
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: node-role
                operator: In
                values:
                  - mq
```

### 2.4 AZ 분산 배치 (Production)

```
┌───────────────────────────────────────────────────────────────┐
│                     Availability Zones                         │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │    AZ-A      │  │    AZ-B      │  │    AZ-C      │         │
│  │              │  │              │  │              │         │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │         │
│  │ │ RMQ-0    │ │  │ │ RMQ-1    │ │  │ │ RMQ-2    │ │         │
│  │ │ (Leader) │ │  │ │(Follower)│ │  │ │(Follower)│ │         │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                                │
│  ※ Quorum: 과반수(2/3) 노드 생존 시 서비스 정상                 │
└───────────────────────────────────────────────────────────────┘
```

---

## 3. Helm Chart 구조

### 3.1 Chart 디렉토리 구조

```
platform-mq-helm/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-staging.yaml
├── values-prod.yaml
├── templates/
│   ├── _helpers.tpl
│   ├── statefulset.yaml
│   ├── service.yaml
│   ├── service-headless.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── pdb.yaml
│   ├── networkpolicy.yaml
│   ├── serviceaccount.yaml
│   ├── rbac.yaml
│   ├── servicemonitor.yaml
│   └── tests/
│       └── test-connection.yaml
├── charts/
│   └── rabbitmq-cluster-operator/
└── crds/
    └── rabbitmqcluster.yaml
```

### 3.2 values.yaml 예시

```yaml
# values.yaml
global:
  imageRegistry: "harbor.platform.internal"
  imagePullSecrets:
    - name: harbor-secret

rabbitmq:
  image:
    repository: rabbitmq
    tag: "3.13.1-management-alpine"
    pullPolicy: IfNotPresent

  replicas: 3

  resources:
    requests:
      cpu: "2"
      memory: "4Gi"
    limits:
      cpu: "4"
      memory: "8Gi"

  persistence:
    enabled: true
    storageClass: "ssd-retain"
    size: 500Gi

  clustering:
    enabled: true
    rebalance: false

  auth:
    existingPasswordSecret: "rabbitmq-admin-secret"
    existingPasswordSecretKey: "password"
    existingErlangCookieSecret: "rabbitmq-erlang-cookie"

  tls:
    enabled: true
    existingSecret: "rabbitmq-tls-secret"
    failIfNoPeerCert: true
    sslOptionsVerify: verify_peer

  configuration: |
    cluster_formation.peer_discovery_backend = rabbit_peer_discovery_k8s
    cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
    cluster_formation.k8s.address_type = hostname
    cluster_formation.node_cleanup.interval = 30
    cluster_formation.node_cleanup.only_log_warning = true
    cluster_partition_handling = pause_minority
    queue_master_locator = min-masters

    # Quorum Queue 기본 설정
    default_queue_type = quorum

    # Memory 설정
    vm_memory_high_watermark.relative = 0.6
    vm_memory_high_watermark_paging_ratio = 0.7

    # Disk 설정
    disk_free_limit.absolute = 5GB

    # Connection 설정
    heartbeat = 60

    # Logging
    log.file.level = info
    log.console = true
    log.console.level = info

  plugins:
    - rabbitmq_management
    - rabbitmq_prometheus
    - rabbitmq_shovel
    - rabbitmq_shovel_management
    - rabbitmq_federation
    - rabbitmq_federation_management
    - rabbitmq_peer_discovery_k8s

service:
  type: ClusterIP
  ports:
    amqp: 5671
    amqpPlain: 5672
    management: 15672
    prometheus: 15692

ingress:
  enabled: false  # Management UI는 internal만

networkPolicy:
  enabled: true
  allowedNamespaces:
    - eventflow
    - platform-mq-monitoring

pdb:
  enabled: true
  minAvailable: 2

serviceMonitor:
  enabled: true
  namespace: platform-mq-monitoring
  interval: 15s
```

---

## 4. Terraform 모듈 구조 설계

### 4.1 모듈 디렉토리 구조

```
terraform/
├── modules/
│   ├── rabbitmq-cluster/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── versions.tf
│   │   └── README.md
│   │
│   ├── rabbitmq-vhost/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   │
│   ├── rabbitmq-user/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   │
│   ├── rabbitmq-policy/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   └── rabbitmq-exchange-queue/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── backend.tf
│   │   ├── terraform.tfvars
│   │   └── projects/
│   │       └── eventflow.tf
│   │
│   ├── staging/
│   │   ├── main.tf
│   │   ├── backend.tf
│   │   ├── terraform.tfvars
│   │   └── projects/
│   │       └── eventflow.tf
│   │
│   └── prod/
│       ├── main.tf
│       ├── backend.tf
│       ├── terraform.tfvars
│       └── projects/
│           └── eventflow.tf
│
└── shared/
    ├── providers.tf
    └── versions.tf
```

### 4.2 rabbitmq-vhost 모듈 예시

```hcl
# modules/rabbitmq-vhost/main.tf

terraform {
  required_providers {
    rabbitmq = {
      source  = "cyrilgdn/rabbitmq"
      version = "~> 1.8"
    }
  }
}

resource "rabbitmq_vhost" "vhost" {
  name = var.vhost_name
}

# Producer 계정
resource "rabbitmq_user" "producer" {
  name     = "${var.project_id}-producer"
  password = var.producer_password
  tags     = ["producer"]
}

# Consumer 계정
resource "rabbitmq_user" "consumer" {
  name     = "${var.project_id}-consumer"
  password = var.consumer_password
  tags     = ["consumer"]
}

# Admin 계정 (프로젝트 관리용)
resource "rabbitmq_user" "admin" {
  name     = "${var.project_id}-admin"
  password = var.admin_password
  tags     = ["administrator", "management"]
}

# Producer 권한: Exchange에만 publish
resource "rabbitmq_permissions" "producer" {
  user  = rabbitmq_user.producer.name
  vhost = rabbitmq_vhost.vhost.name

  permissions {
    configure = ""
    write     = "^${var.project_id}\\.exchange\\..*"
    read      = ""
  }
}

# Consumer 권한: Queue에서만 consume
resource "rabbitmq_permissions" "consumer" {
  user  = rabbitmq_user.consumer.name
  vhost = rabbitmq_vhost.vhost.name

  permissions {
    configure = ""
    write     = ""
    read      = "^${var.project_id}\\.queue\\..*"
  }
}

# Admin 권한: 프로젝트 리소스 전체 관리
resource "rabbitmq_permissions" "admin" {
  user  = rabbitmq_user.admin.name
  vhost = rabbitmq_vhost.vhost.name

  permissions {
    configure = "^${var.project_id}\\..*"
    write     = "^${var.project_id}\\..*"
    read      = "^${var.project_id}\\..*"
  }
}

# 기본 정책
resource "rabbitmq_policy" "ha" {
  name  = "${var.project_id}-ha-policy"
  vhost = rabbitmq_vhost.vhost.name

  policy {
    pattern  = "^${var.project_id}\\.queue\\..*"
    priority = 0
    apply_to = "queues"

    definition = {
      "ha-mode"      = "all"
      "ha-sync-mode" = "automatic"
    }
  }
}

# variables.tf
variable "vhost_name" {
  description = "Name of the vhost"
  type        = string
}

variable "project_id" {
  description = "Project identifier (e.g., ef for eventflow)"
  type        = string
}

variable "producer_password" {
  description = "Producer account password"
  type        = string
  sensitive   = true
}

variable "consumer_password" {
  description = "Consumer account password"
  type        = string
  sensitive   = true
}

variable "admin_password" {
  description = "Admin account password"
  type        = string
  sensitive   = true
}

# outputs.tf
output "vhost_name" {
  value = rabbitmq_vhost.vhost.name
}

output "producer_user" {
  value = rabbitmq_user.producer.name
}

output "consumer_user" {
  value = rabbitmq_user.consumer.name
}

output "admin_user" {
  value = rabbitmq_user.admin.name
}
```

### 4.3 프로젝트별 Terraform 예시

```hcl
# environments/prod/projects/eventflow.tf

module "eventflow_vhost" {
  source = "../../../modules/rabbitmq-vhost"

  vhost_name        = "/eventflow"
  project_id        = "ef"
  producer_password = data.vault_generic_secret.ef_mq.data["producer_password"]
  consumer_password = data.vault_generic_secret.ef_mq.data["consumer_password"]
  admin_password    = data.vault_generic_secret.ef_mq.data["admin_password"]
}

module "eventflow_exchanges" {
  source = "../../../modules/rabbitmq-exchange-queue"

  vhost      = module.eventflow_vhost.vhost_name
  project_id = "ef"

  exchanges = {
    "ef.exchange.order" = {
      type    = "topic"
      durable = true
    }
    "ef.exchange.order.dlx" = {
      type    = "fanout"
      durable = true
    }
    "ef.exchange.user" = {
      type    = "topic"
      durable = true
    }
    "ef.exchange.user.dlx" = {
      type    = "fanout"
      durable = true
    }
  }

  queues = {
    "ef.queue.order.created" = {
      type        = "quorum"
      durable     = true
      dlx         = "ef.exchange.order.dlx"
      max_length  = 100000
      message_ttl = 86400000  # 24 hours
    }
    "ef.queue.order.created.dlq" = {
      type        = "quorum"
      durable     = true
      max_length  = 10000
      message_ttl = 604800000  # 7 days
    }
  }

  bindings = {
    "ef.queue.order.created" = {
      exchange    = "ef.exchange.order"
      routing_key = "order.created"
    }
    "ef.queue.order.created.dlq" = {
      exchange    = "ef.exchange.order.dlx"
      routing_key = "#"
    }
  }
}
```

---

## 5. 프로젝트 증가 시 확장 전략

### 5.1 확장 프로세스

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      신규 프로젝트 온보딩 프로세스                        │
│                                                                          │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐              │
│   │ 1.신청  │ -> │ 2.검토  │ -> │ 3.생성  │ -> │ 4.검증  │              │
│   └─────────┘    └─────────┘    └─────────┘    └─────────┘              │
│                                                                          │
│   - 프로젝트 ID  - 리소스 요구사항  - Terraform    - 연결 테스트         │
│   - 담당자       - 보안 검토        - vhost 생성   - 권한 검증           │
│   - 용도 설명    - 쿼터 결정        - User 생성    - DLQ 테스트          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 온보딩 체크리스트

| 단계 | 항목 | 담당 |
|------|------|------|
| 신청 | 프로젝트 신청서 제출 | 프로젝트팀 |
| 신청 | 프로젝트 ID 예약 | 플랫폼팀 |
| 검토 | 리소스 요구사항 검토 | 플랫폼팀 |
| 검토 | 보안 요구사항 검토 | 보안팀 |
| 검토 | 쿼터 결정 | 플랫폼팀 |
| 생성 | Terraform PR 생성 | 프로젝트팀 |
| 생성 | Terraform 코드 리뷰 | 플랫폼팀 |
| 생성 | Terraform Apply | 플랫폼팀 |
| 생성 | Vault Secret 등록 | 보안팀 |
| 검증 | 연결 테스트 | 프로젝트팀 |
| 검증 | DLQ 테스트 | 프로젝트팀 |
| 검증 | 모니터링 대시보드 확인 | 프로젝트팀 |

### 5.3 수직 확장 기준

| 지표 | 임계치 | 조치 |
|------|--------|------|
| 클러스터 CPU | > 70% 지속 | 노드 스케일업 검토 |
| 클러스터 Memory | > 60% 지속 | 노드 스케일업 검토 |
| 클러스터 Disk | > 70% | 디스크 확장 |
| Connection 수 | > 80% of limit | Connection limit 증가 |
| Queue 수 | > 500 | 구조 검토 |

---

## 6. 특정 프로젝트 분리 전략

### 6.1 분리 판단 기준

| 기준 | 임계치 | 설명 |
|------|--------|------|
| 메시지 처리량 | > 전체의 50% | 단일 프로젝트가 과반 점유 |
| 장애 영향도 | Critical | 다른 프로젝트에 영향 |
| 보안 요구사항 | 격리 필수 | 규제 또는 정책 요구 |
| SLA 요구사항 | 차별화 필요 | 다른 SLA 적용 필요 |

### 6.2 분리 프로세스

```
Phase 1: 준비 (1-2주)
┌─────────────────────────────────────────────────────────────────────────┐
│  1. 독립 클러스터 프로비저닝                                             │
│  2. 독립 클러스터 설정 및 보안 적용                                      │
│  3. Federation Plugin 설정                                               │
│  4. 마이그레이션 테스트                                                  │
└─────────────────────────────────────────────────────────────────────────┘
                                   ↓
Phase 2: 이중화 (1주)
┌─────────────────────────────────────────────────────────────────────────┐
│  1. Federation 활성화 (메시지 복제)                                      │
│  2. Consumer 이중 구독                                                   │
│  3. 데이터 정합성 검증                                                   │
│  4. 모니터링 이중화                                                      │
└─────────────────────────────────────────────────────────────────────────┘
                                   ↓
Phase 3: 전환 (1일)
┌─────────────────────────────────────────────────────────────────────────┐
│  1. Producer 전환 (신규 클러스터)                                        │
│  2. 구 클러스터 Consumer 유지 (잔여 메시지 처리)                         │
│  3. Federation 중단                                                      │
│  4. 구 클러스터 Consumer 종료                                            │
└─────────────────────────────────────────────────────────────────────────┘
                                   ↓
Phase 4: 정리 (1주)
┌─────────────────────────────────────────────────────────────────────────┐
│  1. 구 클러스터 vhost 삭제                                               │
│  2. 구 클러스터 User 삭제                                                │
│  3. Terraform 정리                                                       │
│  4. 문서 업데이트                                                        │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.3 Federation 설정 예시

```erlang
# Federation Upstream 설정 (신규 클러스터에서)
rabbitmqctl set_parameter federation-upstream platform-mq \
  '{"uri":"amqps://federation:password@platform-mq.internal:5671/eventflow","ack-mode":"on-confirm"}'

# Policy 설정 (메시지 복제)
rabbitmqctl set_policy federate-all "^ef\.queue\\..*" \
  '{"federation-upstream":"platform-mq"}' \
  --priority 1 \
  --apply-to queues
```

---

## 7. 클러스터 운영 정책

### 7.1 버전 업그레이드 정책

| 환경 | 업그레이드 방식 | 다운타임 |
|------|----------------|----------|
| dev | Rolling Update | 무중단 |
| staging | Rolling Update | 무중단 |
| prod | Rolling Update (Blue-Green 준비) | 무중단 |

### 7.2 백업 정책

| 대상 | 주기 | 보관 기간 | 저장소 |
|------|------|----------|--------|
| 정의 (Exchange/Queue/Binding) | 일 1회 | 30일 | S3 |
| 메시지 (DLQ만) | 일 1회 | 7일 | S3 |
| 설정 | 변경 시 | 영구 | Git |

### 7.3 Rolling Update 절차

```
1. Pod-2 (Follower) drain 및 업데이트
   - kubectl drain rabbitmq-2
   - StatefulSet image 업데이트
   - Pod 재시작 대기 (Ready)
   - Quorum 복구 확인
                    ↓
2. Pod-1 (Follower) drain 및 업데이트
   - 동일 절차
                    ↓
3. Pod-0 (Leader) drain 및 업데이트
   - Leader 전환 발생 (자동)
   - 동일 절차
                    ↓
4. 검증
   - 클러스터 상태 확인
   - 모든 Queue 정상 확인
   - 메시지 처리 확인
```

---

END OF DOCUMENT
