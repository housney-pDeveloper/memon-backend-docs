# 03. 금융권 수준 보안 설계

```
Document ID    : MQ-SPEC-001-03
Version        : 1.0.0
Classification : Internal / Confidential
```

---

## 1. 보안 아키텍처 개요

### 1.1 Defense in Depth 구조

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Security Layers                                  │
│                                                                          │
│  Layer 1: Network Security                                               │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - VPC Isolation                                                 │    │
│  │  - Private Subnet                                                │    │
│  │  - Security Group / Firewall                                     │    │
│  │  - Kubernetes NetworkPolicy                                      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                   │                                      │
│                                   ▼                                      │
│  Layer 2: Transport Security                                             │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - TLS 1.3 Mandatory                                             │    │
│  │  - mTLS (Client Certificate)                                     │    │
│  │  - Certificate Rotation                                          │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                   │                                      │
│                                   ▼                                      │
│  Layer 3: Authentication                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - Username/Password (Vault managed)                             │    │
│  │  - Client Certificate Auth (mTLS)                                │    │
│  │  - Service Account based                                         │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                   │                                      │
│                                   ▼                                      │
│  Layer 4: Authorization                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - vhost Isolation                                               │    │
│  │  - Resource Permission (configure/write/read)                    │    │
│  │  - Topic Authorization                                           │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                   │                                      │
│                                   ▼                                      │
│  Layer 5: Message Security                                               │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - Payload Encryption (Application Level)                        │    │
│  │  - Message Signing (Optional)                                    │    │
│  │  - PII Masking                                                   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                   │                                      │
│                                   ▼                                      │
│  Layer 6: Audit & Monitoring                                             │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - Connection Audit Log                                          │    │
│  │  - Operation Audit Log                                           │    │
│  │  - Security Event Alert                                          │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. TLS 강제 적용 구조

### 2.1 TLS 설정 요구사항

| 항목 | 요구사항 |
|------|----------|
| TLS 버전 | TLS 1.2 이상 필수, TLS 1.3 권장 |
| 암호화 스위트 | AES-256-GCM, ECDHE 기반 |
| 인증서 유형 | Internal PKI 발급 인증서 |
| 인증서 유효기간 | 1년 (자동 갱신) |
| Cipher Suites | ECDHE-ECDSA-AES256-GCM-SHA384, ECDHE-RSA-AES256-GCM-SHA384 |

### 2.2 RabbitMQ TLS 설정

```erlang
# rabbitmq.conf TLS 설정

# TLS 리스너만 사용 (평문 비활성화)
listeners.tcp = none
listeners.ssl.default = 5671

# TLS 인증서 설정
ssl_options.cacertfile = /etc/rabbitmq/certs/ca.crt
ssl_options.certfile = /etc/rabbitmq/certs/tls.crt
ssl_options.keyfile = /etc/rabbitmq/certs/tls.key

# TLS 버전 설정
ssl_options.versions.1 = tlsv1.3
ssl_options.versions.2 = tlsv1.2

# Cipher Suites (TLS 1.3)
ssl_options.ciphers.1 = TLS_AES_256_GCM_SHA384
ssl_options.ciphers.2 = TLS_CHACHA20_POLY1305_SHA256
ssl_options.ciphers.3 = TLS_AES_128_GCM_SHA256

# Cipher Suites (TLS 1.2 fallback)
ssl_options.ciphers.4 = ECDHE-ECDSA-AES256-GCM-SHA384
ssl_options.ciphers.5 = ECDHE-RSA-AES256-GCM-SHA384
ssl_options.ciphers.6 = ECDHE-ECDSA-AES128-GCM-SHA256
ssl_options.ciphers.7 = ECDHE-RSA-AES128-GCM-SHA256

# Client Certificate 검증 (mTLS)
ssl_options.verify = verify_peer
ssl_options.fail_if_no_peer_cert = true

# OCSP Stapling
ssl_options.honor_cipher_order = true
ssl_options.honor_ecc_order = true

# Management API TLS
management.ssl.port = 15671
management.ssl.cacertfile = /etc/rabbitmq/certs/ca.crt
management.ssl.certfile = /etc/rabbitmq/certs/tls.crt
management.ssl.keyfile = /etc/rabbitmq/certs/tls.key
management.ssl.versions.1 = tlsv1.3
management.ssl.versions.2 = tlsv1.2
```

### 2.3 인증서 구조

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Certificate Hierarchy                             │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────┐      │
│  │                    Platform Root CA                            │      │
│  │               CN=Platform Root CA                              │      │
│  │               Validity: 10 years                               │      │
│  │               Usage: CA only                                   │      │
│  └───────────────────────────────────────────────────────────────┘      │
│                                │                                         │
│                                ▼                                         │
│  ┌───────────────────────────────────────────────────────────────┐      │
│  │                Platform Intermediate CA                        │      │
│  │           CN=Platform Infrastructure CA                        │      │
│  │           Validity: 5 years                                    │      │
│  │           Usage: CA only                                       │      │
│  └───────────────────────────────────────────────────────────────┘      │
│                    │                   │                                 │
│                    ▼                   ▼                                 │
│  ┌──────────────────────┐  ┌──────────────────────┐                     │
│  │   Server Cert        │  │   Client Cert        │                     │
│  │ CN=rabbitmq.internal │  │ CN=ef-producer-app   │                     │
│  │ SAN=rabbitmq-0,1,2   │  │ Validity: 1 year     │                     │
│  │ Validity: 1 year     │  │ Usage: Client Auth   │                     │
│  │ Usage: Server Auth   │  │                      │                     │
│  └──────────────────────┘  └──────────────────────┘                     │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.4 cert-manager 설정

```yaml
# Certificate resource for RabbitMQ
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: rabbitmq-server-cert
  namespace: platform-mq
spec:
  secretName: rabbitmq-tls-secret
  duration: 8760h  # 1 year
  renewBefore: 720h  # 30 days before

  subject:
    organizations:
      - Platform Infrastructure

  commonName: rabbitmq.platform-mq.svc.cluster.local

  dnsNames:
    - rabbitmq.platform-mq.svc.cluster.local
    - rabbitmq.platform-mq.svc
    - rabbitmq-0.rabbitmq-headless.platform-mq.svc.cluster.local
    - rabbitmq-1.rabbitmq-headless.platform-mq.svc.cluster.local
    - rabbitmq-2.rabbitmq-headless.platform-mq.svc.cluster.local
    - "*.rabbitmq-headless.platform-mq.svc.cluster.local"

  privateKey:
    algorithm: ECDSA
    size: 384

  usages:
    - server auth
    - key encipherment
    - digital signature

  issuerRef:
    name: platform-intermediate-ca
    kind: ClusterIssuer
    group: cert-manager.io
```

---

## 3. Secret 관리 전략

### 3.1 Secret 관리 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Secret Management Architecture                      │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                     HashiCorp Vault                              │    │
│  │                                                                  │    │
│  │  Secret Engine: kv-v2                                            │    │
│  │  Path Structure:                                                 │    │
│  │  ├── platform/mq/                                                │    │
│  │  │   ├── admin/                                                  │    │
│  │  │   │   └── credentials (platform-admin password)               │    │
│  │  │   └── erlang-cookie                                           │    │
│  │  │                                                               │    │
│  │  ├── projects/eventflow/mq/                                      │    │
│  │  │   ├── producer/application (producer credentials)            │    │
│  │  │   ├── consumer/system (consumer credentials)                  │    │
│  │  │   └── admin (project admin credentials)                       │    │
│  │  │                                                               │    │
│  │  └── projects/billing/mq/                                        │    │
│  │      └── ...                                                     │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                   │                                      │
│                                   │ Vault Agent Injector                 │
│                                   ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                  Kubernetes Cluster                              │    │
│  │                                                                  │    │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐        │    │
│  │  │ platform-mq   │  │  eventflow    │  │   billing     │        │    │
│  │  │  namespace    │  │  namespace    │  │  namespace    │        │    │
│  │  │               │  │               │  │               │        │    │
│  │  │ Secret:       │  │ Secret:       │  │ Secret:       │        │    │
│  │  │ rabbitmq-     │  │ ef-mq-        │  │ bl-mq-        │        │    │
│  │  │ admin-secret  │  │ credentials   │  │ credentials   │        │    │
│  │  └───────────────┘  └───────────────┘  └───────────────┘        │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Vault Policy 설정

```hcl
# Vault Policy for eventflow project
path "secret/data/projects/eventflow/mq/*" {
  capabilities = ["read"]
}

# Vault Policy for platform-mq admin
path "secret/data/platform/mq/*" {
  capabilities = ["read"]
}

# Terraform용 Policy (provisioning)
path "secret/data/projects/+/mq/*" {
  capabilities = ["create", "update", "read", "delete", "list"]
}
```

### 3.3 Vault Agent Injector 설정

```yaml
# eventflow Application Deployment에 Vault annotation 추가
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ef-application
  namespace: eventflow
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "eventflow-producer"
        vault.hashicorp.com/agent-inject-secret-mq-credentials: "secret/data/projects/eventflow/mq/producer/application"
        vault.hashicorp.com/agent-inject-template-mq-credentials: |
          {{- with secret "secret/data/projects/eventflow/mq/producer/application" -}}
          export MQ_USERNAME="{{ .Data.data.username }}"
          export MQ_PASSWORD="{{ .Data.data.password }}"
          {{- end -}}
    spec:
      serviceAccountName: ef-application
      containers:
        - name: application
          # credentials는 /vault/secrets/mq-credentials 에 주입됨
```

### 3.4 KMS 연동 (Cloud 환경)

```yaml
# AWS KMS를 사용한 Vault Auto-Unseal
storage "raft" {
  path = "/vault/data"
}

seal "awskms" {
  region     = "ap-northeast-2"
  kms_key_id = "alias/platform-vault-unseal"
}
```

---

## 4. IAM 연계 전략

### 4.1 Kubernetes RBAC 연계

```yaml
# RabbitMQ 접근용 ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ef-application
  namespace: eventflow
  annotations:
    # Vault 연동
    vault.hashicorp.com/role: "eventflow-producer"

---
# NetworkPolicy와 연계된 RBAC
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: mq-secret-reader
  namespace: eventflow
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["ef-mq-credentials"]
    verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ef-application-mq-secret
  namespace: eventflow
subjects:
  - kind: ServiceAccount
    name: ef-application
roleRef:
  kind: Role
  name: mq-secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### 4.2 Service Account 매핑

| Kubernetes SA | Vault Role | RabbitMQ User |
|---------------|------------|---------------|
| ef-application | eventflow-producer | ef-producer-application |
| ef-system | eventflow-consumer | ef-consumer-system |
| ef-batch-worker | eventflow-consumer | ef-consumer-batch |
| rabbitmq-admin | platform-mq-admin | platform-admin |

---

## 5. Network Policy

### 5.1 RabbitMQ Namespace NetworkPolicy

```yaml
# platform-mq namespace NetworkPolicy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: rabbitmq-network-policy
  namespace: platform-mq
spec:
  podSelector:
    matchLabels:
      app: rabbitmq

  policyTypes:
    - Ingress
    - Egress

  ingress:
    # AMQPS: 허용된 프로젝트 namespace에서만 접근
    - from:
        - namespaceSelector:
            matchLabels:
              mq-access: "allowed"
          podSelector:
            matchLabels:
              mq-client: "true"
      ports:
        - protocol: TCP
          port: 5671

    # Management API: 모니터링 namespace에서만
    - from:
        - namespaceSelector:
            matchLabels:
              name: platform-mq-monitoring
      ports:
        - protocol: TCP
          port: 15672
        - protocol: TCP
          port: 15692  # Prometheus metrics

    # 클러스터 내부 통신
    - from:
        - podSelector:
            matchLabels:
              app: rabbitmq
      ports:
        - protocol: TCP
          port: 4369   # epmd
        - protocol: TCP
          port: 25672  # distribution

  egress:
    # DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53

    # 클러스터 내부 통신
    - to:
        - podSelector:
            matchLabels:
              app: rabbitmq
      ports:
        - protocol: TCP
          port: 4369
        - protocol: TCP
          port: 25672
```

### 5.2 프로젝트 Namespace NetworkPolicy

```yaml
# eventflow namespace에서 MQ 접근 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-mq-egress
  namespace: eventflow
spec:
  podSelector:
    matchLabels:
      mq-client: "true"

  policyTypes:
    - Egress

  egress:
    # RabbitMQ AMQPS
    - to:
        - namespaceSelector:
            matchLabels:
              name: platform-mq
          podSelector:
            matchLabels:
              app: rabbitmq
      ports:
        - protocol: TCP
          port: 5671

    # DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

### 5.3 Namespace Label 설정

```yaml
# eventflow namespace - MQ 접근 허용
apiVersion: v1
kind: Namespace
metadata:
  name: eventflow
  labels:
    name: eventflow
    mq-access: "allowed"

---
# 비허용 namespace 예시
apiVersion: v1
kind: Namespace
metadata:
  name: external-service
  labels:
    name: external-service
    # mq-access label 없음 → MQ 접근 불가
```

---

## 6. 감사 로그 전략

### 6.1 감사 대상 이벤트

| 카테고리 | 이벤트 | 심각도 |
|----------|--------|--------|
| Authentication | 로그인 성공 | INFO |
| Authentication | 로그인 실패 | WARNING |
| Authentication | 계정 잠금 | CRITICAL |
| Authorization | 권한 거부 | WARNING |
| Authorization | 권한 변경 | CRITICAL |
| Resource | vhost 생성/삭제 | CRITICAL |
| Resource | User 생성/삭제 | CRITICAL |
| Resource | Exchange/Queue 생성/삭제 | INFO |
| Operation | Policy 변경 | CRITICAL |
| Operation | Parameter 변경 | WARNING |
| Connection | 비정상 연결 패턴 | WARNING |

### 6.2 RabbitMQ 감사 로그 설정

```erlang
# rabbitmq.conf - 감사 로그 설정

# 파일 로깅
log.file = /var/log/rabbitmq/rabbit.log
log.file.level = info
log.file.rotation.date = $D0
log.file.rotation.count = 30

# 연결 이벤트 로깅
log.connection.level = info

# 채널 이벤트 로깅
log.channel.level = warning

# 인증 감사 로그
log.default.level = info

# JSON 형식 (ELK 연동)
log.console.formatter = json
log.file.formatter = json
```

### 6.3 감사 로그 포맷

```json
{
  "timestamp": "2026-02-27T10:30:45.123Z",
  "level": "warning",
  "source": "rabbitmq",
  "node": "rabbit@rabbitmq-0",
  "event_type": "authentication_failure",
  "details": {
    "user": "ef-producer-application",
    "vhost": "/eventflow",
    "source_ip": "10.0.1.45",
    "reason": "invalid_credentials",
    "mechanism": "PLAIN"
  },
  "trace_id": "abc123def456"
}
```

### 6.4 감사 로그 수집 파이프라인

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Audit Log Pipeline                                  │
│                                                                          │
│  RabbitMQ Nodes          Fluentd              Elasticsearch            │
│  ┌──────────────┐       ┌──────────┐         ┌──────────────┐          │
│  │ rabbit.log   │──────►│ Fluentd  │────────►│ audit-*      │          │
│  │ (JSON)       │       │ DaemonSet │        │ index        │          │
│  └──────────────┘       └──────────┘         └──────────────┘          │
│                                                      │                   │
│                                                      ▼                   │
│                              ┌───────────────────────────────────────┐  │
│                              │            Kibana                      │  │
│                              │                                        │  │
│                              │  Dashboard: MQ Security Audit          │  │
│                              │  - Authentication failures             │  │
│                              │  - Permission denied events            │  │
│                              │  - Resource changes                    │  │
│                              └───────────────────────────────────────┘  │
│                                                      │                   │
│                                                      ▼                   │
│                              ┌───────────────────────────────────────┐  │
│                              │        Alert Rules                     │  │
│                              │                                        │  │
│                              │  - 5분 내 인증 실패 10회 → CRITICAL    │  │
│                              │  - 권한 변경 → NOTIFY                  │  │
│                              │  - 비정상 시간대 접근 → WARNING        │  │
│                              └───────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 계정 Rotation 전략

### 7.1 Rotation 정책

| 계정 유형 | Rotation 주기 | 방식 |
|----------|--------------|------|
| platform-admin | 30일 | 수동 (Vault) |
| {project}-admin | 60일 | 자동 (Vault) |
| {project}-producer-* | 90일 | 자동 (Vault) |
| {project}-consumer-* | 90일 | 자동 (Vault) |
| erlang-cookie | 변경 금지 | N/A |

### 7.2 자동 Rotation 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Credential Rotation Flow                             │
│                                                                          │
│   ┌────────────┐    ┌────────────┐    ┌────────────┐                    │
│   │   Vault    │    │ Rotation   │    │ RabbitMQ   │                    │
│   │  (Secret)  │◄───│   Job      │───►│  Cluster   │                    │
│   └────────────┘    └────────────┘    └────────────┘                    │
│         │                                    │                           │
│         │ Secret Update                      │ Password Change           │
│         ▼                                    ▼                           │
│   ┌────────────────────────────────────────────────────────────────┐    │
│   │                    Kubernetes Pods                              │    │
│   │                                                                 │    │
│   │  Phase 1: New credentials injected (Vault Agent)                │    │
│   │  Phase 2: Application reconnects with new credentials          │    │
│   │  Phase 3: Old credentials invalidated (grace period 후)        │    │
│   │                                                                 │    │
│   └────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.3 Rotation CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mq-credential-rotation
  namespace: platform-mq
spec:
  schedule: "0 2 1 */3 *"  # 분기별 첫째 날 02:00
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: credential-rotator
          containers:
            - name: rotator
              image: platform/credential-rotator:1.0.0
              command:
                - /rotate.sh
              env:
                - name: VAULT_ADDR
                  value: "https://vault.internal:8200"
                - name: RABBITMQ_API_URL
                  value: "https://rabbitmq.platform-mq:15672"
              volumeMounts:
                - name: vault-token
                  mountPath: /var/run/secrets/vault
          volumes:
            - name: vault-token
              projected:
                sources:
                  - serviceAccountToken:
                      path: token
                      expirationSeconds: 3600
                      audience: vault
          restartPolicy: OnFailure
```

### 7.4 Rotation 스크립트

```bash
#!/bin/bash
# rotate.sh - Credential Rotation Script

set -euo pipefail

VAULT_ADDR="${VAULT_ADDR}"
RABBITMQ_API="${RABBITMQ_API_URL}"

# Vault 인증
vault login -method=kubernetes role=credential-rotator

# 각 프로젝트별 rotation
for project in $(vault kv list -format=json secret/projects | jq -r '.[]'); do
  echo "Rotating credentials for project: ${project}"

  for account_type in producer consumer; do
    # 새 비밀번호 생성
    NEW_PASSWORD=$(openssl rand -base64 32)

    # RabbitMQ에 비밀번호 변경
    USER="${project}-${account_type}"
    curl -sf -X PUT "${RABBITMQ_API}/api/users/${USER}" \
      -u "platform-admin:${ADMIN_PASSWORD}" \
      -H "Content-Type: application/json" \
      -d "{\"password\": \"${NEW_PASSWORD}\", \"tags\": \"${account_type}\"}"

    # Vault에 새 비밀번호 저장
    vault kv put "secret/projects/${project}/mq/${account_type}" \
      username="${USER}" \
      password="${NEW_PASSWORD}" \
      rotated_at="$(date -u +%Y-%m-%dT%H:%M:%SZ)"

    echo "  Rotated: ${USER}"
  done
done

echo "Credential rotation completed"
```

---

## 8. 메시지 암호화 전략

### 8.1 암호화 레이어

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Message Encryption Layers                             │
│                                                                          │
│  Layer 1: Transport Encryption (TLS)                                     │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - 전송 중 암호화                                                │    │
│  │  - 자동 적용 (TLS 강제)                                          │    │
│  │  - 모든 메시지에 적용                                            │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Layer 2: Payload Encryption (Application Level)                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - 저장 중 암호화                                                │    │
│  │  - 민감 데이터만 선택적 적용                                     │    │
│  │  - Application이 암/복호화 담당                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Layer 3: Field-Level Encryption                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - PII 필드만 개별 암호화                                        │    │
│  │  - 로그/감사에서 복호화 없이 추적 가능                           │    │
│  │  - 키 분리 관리                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Payload 암호화 구현 (Java)

```java
// MessageEncryptionService.java
@Service
public class MessageEncryptionService {

    private static final String ALGORITHM = "AES/GCM/NoPadding";
    private static final int GCM_IV_LENGTH = 12;
    private static final int GCM_TAG_LENGTH = 128;

    private final SecretKey encryptionKey;

    public MessageEncryptionService(
        @Value("${mq.encryption.key-id}") String keyId,
        VaultTemplate vaultTemplate
    ) {
        // Vault에서 암호화 키 조회
        VaultResponse response = vaultTemplate.read("secret/data/platform/mq/encryption-keys");
        String keyBase64 = (String) response.getData().get(keyId);
        this.encryptionKey = new SecretKeySpec(Base64.getDecoder().decode(keyBase64), "AES");
    }

    public EncryptedPayload encrypt(Object payload) throws Exception {
        byte[] plaintext = objectMapper.writeValueAsBytes(payload);

        byte[] iv = new byte[GCM_IV_LENGTH];
        SecureRandom.getInstanceStrong().nextBytes(iv);

        Cipher cipher = Cipher.getInstance(ALGORITHM);
        GCMParameterSpec parameterSpec = new GCMParameterSpec(GCM_TAG_LENGTH, iv);
        cipher.init(Cipher.ENCRYPT_MODE, encryptionKey, parameterSpec);

        byte[] ciphertext = cipher.doFinal(plaintext);

        return new EncryptedPayload(
            Base64.getEncoder().encodeToString(iv),
            Base64.getEncoder().encodeToString(ciphertext),
            "AES-256-GCM"
        );
    }

    public <T> T decrypt(EncryptedPayload encrypted, Class<T> type) throws Exception {
        byte[] iv = Base64.getDecoder().decode(encrypted.getIv());
        byte[] ciphertext = Base64.getDecoder().decode(encrypted.getCiphertext());

        Cipher cipher = Cipher.getInstance(ALGORITHM);
        GCMParameterSpec parameterSpec = new GCMParameterSpec(GCM_TAG_LENGTH, iv);
        cipher.init(Cipher.DECRYPT_MODE, encryptionKey, parameterSpec);

        byte[] plaintext = cipher.doFinal(ciphertext);

        return objectMapper.readValue(plaintext, type);
    }
}

// EncryptedPayload.java
public record EncryptedPayload(
    String iv,
    String ciphertext,
    String algorithm
) {}
```

### 8.3 암호화 대상 필드 정의

| 필드 유형 | 암호화 필수 | 예시 |
|----------|------------|------|
| 개인 식별 정보 (PII) | 필수 | 이름, 이메일, 전화번호 |
| 금융 정보 | 필수 | 계좌번호, 카드번호 |
| 인증 정보 | 필수 | 비밀번호, 토큰 |
| 비즈니스 민감 정보 | 선택 | 거래 금액, 계약 조건 |
| 일반 정보 | 불필요 | 이벤트 타입, 타임스탬프 |

---

## 9. 외부 연동 시 보안 고려사항

### 9.1 외부 시스템 연동 패턴

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   External System Integration                            │
│                                                                          │
│  Internal Network                         External Network               │
│  ┌────────────────────────┐              ┌────────────────────────┐     │
│  │                        │              │                        │     │
│  │  RabbitMQ Cluster      │              │  External System       │     │
│  │  (Private Subnet)      │              │                        │     │
│  │                        │              │                        │     │
│  └───────────┬────────────┘              └────────────────────────┘     │
│              │                                       ▲                   │
│              │ Internal Only                         │                   │
│              ▼                                       │                   │
│  ┌────────────────────────┐                          │                   │
│  │  Integration Gateway   │──────────────────────────┘                   │
│  │  (DMZ)                 │     API Call (HTTPS)                         │
│  │                        │     mTLS Required                            │
│  │  - IP Whitelist        │     Rate Limited                             │
│  │  - mTLS Validation     │                                              │
│  │  - Request Signing     │                                              │
│  └────────────────────────┘                                              │
│                                                                          │
│  ※ RabbitMQ는 절대 외부에 직접 노출하지 않음                              │
│  ※ 외부 연동은 반드시 Integration Gateway를 통해서만                      │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.2 외부 연동 보안 체크리스트

| 항목 | 요구사항 | 검증 방법 |
|------|----------|----------|
| 네트워크 격리 | MQ 직접 접근 불가 | NetworkPolicy 검증 |
| API Gateway | 모든 외부 요청 통과 | 트래픽 로그 분석 |
| mTLS | 외부 시스템 인증서 검증 | 인증서 체인 확인 |
| IP Whitelist | 허용된 IP만 접근 | 방화벽 규칙 검증 |
| Rate Limiting | 초당 요청 수 제한 | 부하 테스트 |
| Request Signing | 요청 무결성 검증 | 서명 검증 로직 |
| 감사 로그 | 모든 외부 요청 기록 | 로그 샘플링 |

---

END OF DOCUMENT
