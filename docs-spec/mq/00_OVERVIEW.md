# Platform MQ Architecture Specification

```
Document ID    : MQ-SPEC-001
Version        : 1.0.0
Classification : Internal / Confidential
Author         : Platform Architecture Team
Created        : 2026-02-27
Last Updated   : 2026-02-27
```

---

## 1. 문서 목적

이 문서는 플랫폼사업부의 **RabbitMQ 기반 메시징 인프라 표준 아키텍처**를 정의한다.

본 설계는 다음을 목표로 한다:

- 단일 RabbitMQ 클러스터 기반 멀티 프로젝트 운영
- 금융권 수준 보안 및 감사 체계
- Clean Architecture 기반 이벤트 설계 표준
- Terraform + Helm 기반 IaC 운영
- 향후 프로젝트 독립 분리 가능 구조

---

## 2. 문서 구성

| 번호 | 문서명 | 내용 |
|------|--------|------|
| 00 | OVERVIEW.md | 전체 개요 (본 문서) |
| 01 | ARCHITECTURE.md | 전체 아키텍처 설계 |
| 02 | MULTI_TENANCY.md | 멀티 프로젝트 격리 전략 |
| 03 | SECURITY.md | 금융권 수준 보안 설계 |
| 04 | CLEAN_ARCHITECTURE_EVENT.md | Clean Architecture 기반 이벤트 설계 표준 |
| 05 | EXCHANGE_QUEUE_STANDARD.md | Exchange / Queue 표준 설계 |
| 06 | RESOURCE_PROTECTION.md | 리소스 보호 전략 |
| 07 | MONITORING_OPERATION.md | 모니터링 및 운영 설계 |
| 08 | TERRAFORM_HELM.md | IaC 구조 설계 |
| 09 | RUNBOOK.md | 운영 Runbook |
| 10 | EVENTFLOW_IMPLEMENTATION.md | eventflow 프로젝트 MQ 구현 |
| 11 | SCHEMA_ISOLATION_POLICY.md | scm_eflog 스키마 분리 및 접근 제어 정책 |
| 12 | MESSAGE_SEND_ARCHITECTURE.md | 메시지 발송 아키텍처 설계 |
| json/ | rabbitmq-definitions.json | RabbitMQ definitions import용 JSON |

---

## 3. 적용 범위

### 3.1 대상 프로젝트

```
┌─────────────────────────────────────────────────────────────┐
│                   Platform MQ Cluster                        │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  eventflow   │  │  project-b   │  │  project-c   │  ...  │
│  │   (v1.0)     │  │   (future)   │  │   (future)   │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 적용 환경

| 환경 | 클러스터 | 용도 |
|------|----------|------|
| local | 단일 노드 | 개발자 로컬 개발 |
| dev | 단일 노드 | 개발 통합 테스트 |
| staging | 3노드 Quorum | 운영 전 검증 |
| prod | 3노드 Quorum + HA | 운영 환경 |

---

## 4. 핵심 설계 원칙

### 4.1 격리 원칙

```
┌─────────────────────────────────────────────────────────────┐
│                     Single MQ Cluster                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   vhost: /eventflow          vhost: /project-b              │
│   ┌───────────────────┐      ┌───────────────────┐          │
│   │ Exchange/Queue    │      │ Exchange/Queue    │          │
│   │ User: ef-*        │      │ User: pb-*        │          │
│   │ Policy: ef-*      │      │ Policy: pb-*      │          │
│   └───────────────────┘      └───────────────────┘          │
│                                                              │
│   ✗ Cross-vhost Access Denied                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 보안 원칙

| 원칙 | 설명 |
|------|------|
| Defense in Depth | 다층 보안 적용 (Network → TLS → Auth → Permission) |
| Least Privilege | 최소 권한 원칙 (Producer/Consumer 분리) |
| Zero Trust | vhost 간 완전 격리 |
| Audit Everything | 모든 작업 감사 로그 기록 |

### 4.3 운영 원칙

| 원칙 | 설명 |
|------|------|
| Immutable Infrastructure | IaC 기반 변경 관리 |
| Observability First | 메트릭/로그/트레이스 통합 |
| Graceful Degradation | 장애 시 우아한 저하 |
| Automated Recovery | 자동 복구 우선 |

---

## 5. 기술 스택

### 5.1 Core

| 구성요소 | 기술 | 버전 |
|----------|------|------|
| Message Broker | RabbitMQ | 3.13.x |
| Container Orchestration | Kubernetes | 1.28+ |
| IaC | Terraform | 1.7+ |
| Package Manager | Helm | 3.14+ |

### 5.2 보안

| 구성요소 | 기술 |
|----------|------|
| Secret Management | HashiCorp Vault |
| Certificate Management | cert-manager |
| Network Policy | Cilium / Calico |

### 5.3 모니터링

| 구성요소 | 기술 |
|----------|------|
| Metrics | Prometheus + Grafana |
| Logging | ELK Stack / Loki |
| Tracing | OpenTelemetry + Jaeger |
| Alerting | Alertmanager + PagerDuty |

---

## 6. 프로젝트 네이밍 규칙

### 6.1 프로젝트 식별자

| 프로젝트 | 플랫폼 | 프로젝트 | vhost |
|----------|--------|----------|-------|
| eventflow | pf (platform) | ef | /pf-ef |
| (예시) billing | pf | bl | /pf-bl |
| (예시) notification | pf | nf | /pf-nf |

### 6.2 네이밍 패턴

```
Exchange: ex.{platform}.{project}.{domain}.{type}
Queue:    q.{platform}.{project}.{domain}.{action}
DLQ:      q.{platform}.{project}.dlq.{domain}.{action}
Routing:  {domain}.{action}.{version}

Examples (eventflow):
- ex.pf.ef.log.topic           # 로그 전용 Exchange (topic)
- ex.pf.ef.message.topic       # 메시지 발송 Exchange (topic)
- ex.pf.ef.notification.topic  # 알림 Exchange (topic)
- ex.pf.ef.dlx                 # Dead Letter Exchange

- q.pf.ef.log.write            # 로그 기록 Queue
- q.pf.ef.message.send         # 메시지 발송 Queue
- q.pf.ef.notification         # 알림 Queue
- q.pf.ef.dlq.message.send     # 메시지 발송 DLQ
- q.pf.ef.dlq.notification     # 알림 DLQ

- log.access.v1                # 접근 로그 Routing Key
- message.send.v1              # 메시지 발송 Routing Key
- notification.event.v1        # 알림 이벤트 Routing Key
```

---

## 7. 문서 변경 이력

| 버전 | 일자 | 변경 내용 | 작성자 |
|------|------|----------|--------|
| 1.0.0 | 2026-02-27 | 초안 작성 | Platform Architecture Team |

---

END OF DOCUMENT
