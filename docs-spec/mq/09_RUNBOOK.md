# 09. 운영 Runbook

```
Document ID    : MQ-SPEC-001-09
Version        : 1.0.0
Classification : Internal / Confidential
```

---

## 1. 일상 운영 절차

### 1.1 클러스터 상태 확인

```bash
#!/bin/bash
# daily-health-check.sh

echo "=== RabbitMQ Daily Health Check ==="
echo "Date: $(date)"
echo ""

# 1. 클러스터 상태
echo "### 1. Cluster Status ###"
kubectl exec rabbitmq-0 -n platform-mq -- rabbitmqctl cluster_status

# 2. 노드 상태
echo ""
echo "### 2. Node Status ###"
kubectl exec rabbitmq-0 -n platform-mq -- rabbitmqctl status | grep -A 20 "Memory"

# 3. 알람 상태
echo ""
echo "### 3. Alarms ###"
kubectl exec rabbitmq-0 -n platform-mq -- rabbitmqctl list_alarms

# 4. Connection 수
echo ""
echo "### 4. Connections ###"
kubectl exec rabbitmq-0 -n platform-mq -- rabbitmqctl list_connections | wc -l

# 5. Queue 상태 (메시지 수 > 10000)
echo ""
echo "### 5. Queue Backlog (> 10000 messages) ###"
kubectl exec rabbitmq-0 -n platform-mq -- \
  rabbitmqctl list_queues -p /eventflow name messages consumers | \
  awk '$2 > 10000 {print $0}'

# 6. DLQ 상태
echo ""
echo "### 6. DLQ Status ###"
kubectl exec rabbitmq-0 -n platform-mq -- \
  rabbitmqctl list_queues -p /eventflow name messages | \
  grep "\.dlq"

# 7. 디스크 사용량
echo ""
echo "### 7. Disk Usage ###"
kubectl exec rabbitmq-0 -n platform-mq -- df -h /var/lib/rabbitmq
```

### 1.2 신규 프로젝트 온보딩

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  New Project Onboarding Checklist                        │
│                                                                          │
│  Project Name: _________________                                         │
│  Project ID: _________________                                           │
│  Requested by: _________________                                         │
│  Date: _________________                                                 │
│                                                                          │
│  [ ] Step 1: 요청서 검토                                                 │
│      - 프로젝트 정보 확인                                               │
│      - 리소스 요구사항 검토                                             │
│      - 보안 요구사항 검토                                               │
│                                                                          │
│  [ ] Step 2: 리소스 쿼터 결정                                            │
│      - Connection Limit: _____                                           │
│      - Queue Limit: _____                                                │
│      - 등급: [ ] Standard [ ] Premium [ ] Enterprise                    │
│                                                                          │
│  [ ] Step 3: Terraform 코드 작성                                         │
│      - environments/{env}/projects/{project}.tf 생성                    │
│      - PR 생성                                                           │
│                                                                          │
│  [ ] Step 4: Terraform Apply                                             │
│      - Dev 환경 적용                                                     │
│      - 연결 테스트                                                       │
│      - Staging 환경 적용                                                 │
│      - Prod 환경 적용                                                    │
│                                                                          │
│  [ ] Step 5: 검증                                                        │
│      - Producer 연결 테스트                                              │
│      - Consumer 연결 테스트                                              │
│      - DLQ 동작 테스트                                                   │
│      - 모니터링 대시보드 확인                                            │
│                                                                          │
│  [ ] Step 6: 인수인계                                                    │
│      - 프로젝트팀에 Credential 전달 방법 안내                            │
│      - 모니터링 대시보드 접근 권한 부여                                  │
│      - 운영 가이드 전달                                                  │
│                                                                          │
│  Sign-off: _________________                                             │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Credential Rotation

```bash
#!/bin/bash
# credential-rotation.sh

set -euo pipefail

PROJECT_ID=$1
ENVIRONMENT=${2:-prod}

echo "=== Credential Rotation for ${PROJECT_ID} (${ENVIRONMENT}) ==="

# 1. 새 비밀번호 생성
NEW_PRODUCER_PASS=$(openssl rand -base64 32 | tr -d '=+/' | head -c 32)
NEW_CONSUMER_PASS=$(openssl rand -base64 32 | tr -d '=+/' | head -c 32)

# 2. RabbitMQ 비밀번호 변경
echo "Changing RabbitMQ passwords..."
kubectl exec rabbitmq-0 -n platform-mq -- \
  rabbitmqctl change_password "${PROJECT_ID}-producer-application" "${NEW_PRODUCER_PASS}"

kubectl exec rabbitmq-0 -n platform-mq -- \
  rabbitmqctl change_password "${PROJECT_ID}-consumer-system" "${NEW_CONSUMER_PASS}"

# 3. Vault 업데이트
echo "Updating Vault secrets..."
vault kv put "secret/projects/${PROJECT_ID}/mq/producer/application" \
  username="${PROJECT_ID}-producer-application" \
  password="${NEW_PRODUCER_PASS}" \
  rotated_at="$(date -u +%Y-%m-%dT%H:%M:%SZ)"

vault kv put "secret/projects/${PROJECT_ID}/mq/consumer/system" \
  username="${PROJECT_ID}-consumer-system" \
  password="${NEW_CONSUMER_PASS}" \
  rotated_at="$(date -u +%Y-%m-%dT%H:%M:%SZ)"

# 4. Pod 재시작 (새 Credential 적용)
echo "Restarting application pods..."
kubectl rollout restart deployment/${PROJECT_ID}-application -n ${PROJECT_ID}
kubectl rollout restart deployment/${PROJECT_ID}-system -n ${PROJECT_ID}

echo "Credential rotation completed for ${PROJECT_ID}"
```

---

## 2. 장애 대응 Runbook

### 2.1 RB-001: 노드 다운

```
┌─────────────────────────────────────────────────────────────────────────┐
│  RUNBOOK: RB-001 - RabbitMQ Node Down                                    │
│                                                                          │
│  Alert: RabbitMQNodeDown                                                │
│  Severity: CRITICAL                                                      │
│  SLA: 15분 내 대응                                                       │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. 영향 범위 확인                                                       │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  # 클러스터 상태 확인                                          │   │
│     │  kubectl get pods -n platform-mq                              │   │
│     │  kubectl exec rabbitmq-0 -n platform-mq -- \                  │   │
│     │    rabbitmqctl cluster_status                                 │   │
│     │                                                               │   │
│     │  # 영향 확인                                                   │   │
│     │  - 2/3 노드 정상: Quorum 유지, 서비스 정상                    │   │
│     │  - 1/3 노드 정상: Quorum 손실, 서비스 장애                    │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  2. Pod 상태 확인                                                        │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  kubectl describe pod rabbitmq-N -n platform-mq               │   │
│     │  kubectl logs rabbitmq-N -n platform-mq --tail=100            │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  3. 복구 시도                                                            │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  # Case A: Pod CrashLoopBackOff                               │   │
│     │  kubectl delete pod rabbitmq-N -n platform-mq                 │   │
│     │  # StatefulSet이 자동 재생성                                   │   │
│     │                                                               │   │
│     │  # Case B: Node 문제                                          │   │
│     │  kubectl drain <node-name> --ignore-daemonsets                │   │
│     │  # Pod가 다른 노드에서 재시작                                  │   │
│     │                                                               │   │
│     │  # Case C: PVC 문제                                           │   │
│     │  kubectl describe pvc rabbitmq-data-rabbitmq-N -n platform-mq │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  4. 복구 확인                                                            │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  # 노드 재참여 확인                                            │   │
│     │  kubectl exec rabbitmq-0 -n platform-mq -- \                  │   │
│     │    rabbitmqctl cluster_status                                 │   │
│     │                                                               │   │
│     │  # Queue 상태 확인                                             │   │
│     │  kubectl exec rabbitmq-0 -n platform-mq -- \                  │   │
│     │    rabbitmqctl list_queues name state leader                  │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  5. Escalation                                                           │
│     - 15분 내 복구 불가 시: 플랫폼 리더 호출                            │
│     - Quorum 손실 시: 즉시 긴급 대응팀 소집                             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 RB-002: Memory Alarm

```
┌─────────────────────────────────────────────────────────────────────────┐
│  RUNBOOK: RB-002 - RabbitMQ Memory Alarm                                 │
│                                                                          │
│  Alert: RabbitMQMemoryAlarm                                             │
│  Severity: CRITICAL                                                      │
│  SLA: 즉시 대응                                                          │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Impact: 모든 Publisher 차단됨                                           │
│                                                                          │
│  1. 현황 파악                                                            │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  # 메모리 상태 확인                                            │   │
│     │  kubectl exec rabbitmq-0 -n platform-mq -- \                  │   │
│     │    rabbitmqctl status | grep -A 30 "Memory"                   │   │
│     │                                                               │   │
│     │  # 큐별 메모리 사용량                                          │   │
│     │  kubectl exec rabbitmq-0 -n platform-mq -- \                  │   │
│     │    rabbitmqctl list_queues name messages memory | sort -k3 -rn│   │
│     │                                                               │   │
│     │  # Connection 수 확인                                          │   │
│     │  kubectl exec rabbitmq-0 -n platform-mq -- \                  │   │
│     │    rabbitmqctl list_connections | wc -l                       │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  2. 원인별 대응                                                          │
│                                                                          │
│     A. Queue 메시지 적체가 원인인 경우:                                  │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  # Consumer 스케일 아웃                                        │   │
│     │  kubectl scale deployment ef-system -n eventflow --replicas=20│   │
│     │                                                               │   │
│     │  # 또는 긴급 Consumer 배포                                     │   │
│     │  kubectl apply -f emergency-consumer.yaml                     │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
│     B. Connection 과다인 경우:                                           │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  # 비정상 Connection 확인                                      │   │
│     │  kubectl exec rabbitmq-0 -n platform-mq -- \                  │   │
│     │    rabbitmqctl list_connections user client_properties        │   │
│     │                                                               │   │
│     │  # 특정 User의 Connection 모두 종료                            │   │
│     │  for conn in $(kubectl exec rabbitmq-0 -n platform-mq -- \    │   │
│     │    rabbitmqctl list_connections pid user | \                  │   │
│     │    grep "problem-user" | awk '{print $1}'); do                │   │
│     │    kubectl exec rabbitmq-0 -n platform-mq -- \                │   │
│     │      rabbitmqctl close_connection "$conn" "memory alarm"      │   │
│     │  done                                                         │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
│     C. 메모리 Leak 의심:                                                 │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  # 노드별 메모리 분석                                          │   │
│     │  kubectl exec rabbitmq-0 -n platform-mq -- \                  │   │
│     │    rabbitmqctl eval 'erlang:memory().'                        │   │
│     │                                                               │   │
│     │  # 필요시 노드 재시작 (Rolling)                                │   │
│     │  kubectl delete pod rabbitmq-2 -n platform-mq                 │   │
│     │  # 복구 확인 후 다음 노드                                      │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  3. 알람 해제 확인                                                       │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  kubectl exec rabbitmq-0 -n platform-mq -- \                  │   │
│     │    rabbitmqctl list_alarms                                    │   │
│     │  # 출력이 비어있으면 알람 해제됨                               │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.3 RB-003: Network Partition

```
┌─────────────────────────────────────────────────────────────────────────┐
│  RUNBOOK: RB-003 - RabbitMQ Network Partition                            │
│                                                                          │
│  Alert: RabbitMQClusterPartition                                        │
│  Severity: CRITICAL                                                      │
│  SLA: 즉시 대응                                                          │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Impact: 클러스터 분리, Quorum Queue 소수 파티션 일시 정지              │
│                                                                          │
│  ⚠️  주의: 잘못된 대응은 데이터 손실 가능                                │
│                                                                          │
│  1. 파티션 상태 확인                                                     │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  kubectl exec rabbitmq-0 -n platform-mq -- \                  │   │
│     │    rabbitmqctl cluster_status                                 │   │
│     │                                                               │   │
│     │  # 출력에서 확인:                                              │   │
│     │  # - Running Nodes 목록                                        │   │
│     │  # - Network Partitions 섹션                                   │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  2. 네트워크 문제 진단                                                   │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  # 노드 간 통신 테스트                                         │   │
│     │  kubectl exec rabbitmq-0 -n platform-mq -- \                  │   │
│     │    ping -c 3 rabbitmq-1.rabbitmq-headless                     │   │
│     │                                                               │   │
│     │  kubectl exec rabbitmq-0 -n platform-mq -- \                  │   │
│     │    ping -c 3 rabbitmq-2.rabbitmq-headless                     │   │
│     │                                                               │   │
│     │  # Kubernetes 노드 상태                                        │   │
│     │  kubectl get nodes -o wide                                    │   │
│     │                                                               │   │
│     │  # CNI 상태 확인 (Calico 예시)                                 │   │
│     │  kubectl get pods -n kube-system -l k8s-app=calico-node       │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  3. 네트워크 문제 해결                                                   │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  # CNI Pod 재시작 (해당 노드)                                  │   │
│     │  kubectl delete pod calico-node-xxxxx -n kube-system          │   │
│     │                                                               │   │
│     │  # 또는 노드 재부팅 (최후 수단)                                │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  4. 파티션 자동 복구 확인                                                │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  # pause_minority 정책: 네트워크 복구 시 자동 재참여           │   │
│     │  kubectl exec rabbitmq-0 -n platform-mq -- \                  │   │
│     │    rabbitmqctl cluster_status                                 │   │
│     │                                                               │   │
│     │  # Network Partitions가 빈 목록이면 복구됨                     │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  5. 수동 복구 (자동 복구 안 될 경우)                                     │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  ⚠️  데이터 손실 가능 - 반드시 확인 후 진행                     │   │
│     │                                                               │   │
│     │  # 소수 파티션 노드에서 실행:                                  │   │
│     │  kubectl exec rabbitmq-2 -n platform-mq -- \                  │   │
│     │    rabbitmqctl stop_app                                       │   │
│     │  kubectl exec rabbitmq-2 -n platform-mq -- \                  │   │
│     │    rabbitmqctl reset                                          │   │
│     │  kubectl exec rabbitmq-2 -n platform-mq -- \                  │   │
│     │    rabbitmqctl join_cluster rabbit@rabbitmq-0                 │   │
│     │  kubectl exec rabbitmq-2 -n platform-mq -- \                  │   │
│     │    rabbitmqctl start_app                                      │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  6. 복구 후 검증                                                         │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  # Quorum Queue 상태 확인                                      │   │
│     │  kubectl exec rabbitmq-0 -n platform-mq -- \                  │   │
│     │    rabbitmqctl list_queues name state leader members          │   │
│     │                                                               │   │
│     │  # 메시지 정합성 확인 (Application 레벨)                       │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.4 RB-004: DLQ 폭증

```
┌─────────────────────────────────────────────────────────────────────────┐
│  RUNBOOK: RB-004 - DLQ Message Overflow                                  │
│                                                                          │
│  Alert: RabbitMQDLQGrowing                                              │
│  Severity: WARNING → CRITICAL (증가 지속 시)                            │
│  SLA: 30분 내 분석 시작                                                  │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. 현황 파악                                                            │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  # DLQ 메시지 수 확인                                          │   │
│     │  kubectl exec rabbitmq-0 -n platform-mq -- \                  │   │
│     │    rabbitmqctl list_queues -p /eventflow name messages | \    │   │
│     │    grep "\.dlq"                                               │   │
│     │                                                               │   │
│     │  # 증가율 확인 (Grafana 또는)                                  │   │
│     │  # 5분 전 대비 증가량                                          │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  2. 메시지 샘플 분석                                                     │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  # 최근 메시지 10개 조회                                       │   │
│     │  kubectl exec rabbitmq-0 -n platform-mq -- \                  │   │
│     │    rabbitmqadmin get queue=ef.queue.order.dlq count=10        │   │
│     │                                                               │   │
│     │  # x-death 헤더 분석                                           │   │
│     │  # - reason: 왜 DLQ로 왔는지                                   │   │
│     │  # - queue: 원래 큐                                            │   │
│     │  # - count: 재시도 횟수                                        │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  3. 원인별 대응                                                          │
│                                                                          │
│     A. reason: rejected (Consumer 에러)                                  │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  # Consumer 로그 확인                                          │   │
│     │  kubectl logs -l app=ef-system -n eventflow --tail=500 | \    │   │
│     │    grep -i error                                              │   │
│     │                                                               │   │
│     │  # 코드 버그 → 핫픽스 후 재처리                                │   │
│     │  # 외부 시스템 장애 → 장애 해소 후 재처리                      │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
│     B. reason: expired (TTL 만료)                                        │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  # Consumer 처리 속도 부족                                     │   │
│     │  kubectl scale deployment ef-system -n eventflow --replicas=20│   │
│     │                                                               │   │
│     │  # 또는 TTL 증가 검토                                          │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
│     C. reason: maxlen (큐 오버플로우)                                    │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  # Consumer 스케일 아웃                                        │   │
│     │  # 또는 Queue max-length 증가 검토                             │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  4. 재처리                                                               │
│     ┌───────────────────────────────────────────────────────────────┐   │
│     │  # 문제 해결 확인 후                                           │   │
│     │  ./reprocess-dlq.sh ef.queue.order.dlq ef.exchange.order \    │   │
│     │    order.created 100                                          │   │
│     │                                                               │   │
│     │  # 소량 테스트 후 전체 재처리                                  │   │
│     └───────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  5. 재발 방지                                                            │
│     - 근본 원인 문서화                                                  │
│     - 필요 시 알람 임계치 조정                                          │
│     - Consumer 안정성 개선                                              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 유지보수 절차

### 3.1 Rolling Update (버전 업그레이드)

```bash
#!/bin/bash
# rolling-update.sh

set -euo pipefail

NEW_VERSION=${1:?Usage: rolling-update.sh <new-version>}
NAMESPACE="platform-mq"

echo "=== RabbitMQ Rolling Update to ${NEW_VERSION} ==="

# 1. 현재 상태 확인
echo "Step 1: Pre-flight checks"
kubectl exec rabbitmq-0 -n ${NAMESPACE} -- rabbitmqctl cluster_status
kubectl exec rabbitmq-0 -n ${NAMESPACE} -- rabbitmqctl list_alarms

# 알람 있으면 중단
ALARMS=$(kubectl exec rabbitmq-0 -n ${NAMESPACE} -- rabbitmqctl list_alarms 2>/dev/null)
if [ -n "$ALARMS" ]; then
    echo "ERROR: Active alarms detected. Aborting upgrade."
    exit 1
fi

# 2. Node 2 (Follower) 업데이트
echo ""
echo "Step 2: Updating rabbitmq-2"
kubectl exec rabbitmq-0 -n ${NAMESPACE} -- rabbitmqctl await_online_nodes 3

# Drain
kubectl drain rabbitmq-2.rabbitmq-headless.${NAMESPACE} --ignore-daemonsets --delete-emptydir-data || true

# Image 업데이트
kubectl set image statefulset/rabbitmq rabbitmq=rabbitmq:${NEW_VERSION} -n ${NAMESPACE}

# Pod 2만 재시작 (StatefulSet은 역순)
kubectl delete pod rabbitmq-2 -n ${NAMESPACE}
kubectl wait --for=condition=Ready pod/rabbitmq-2 -n ${NAMESPACE} --timeout=300s

# 클러스터 상태 확인
kubectl exec rabbitmq-0 -n ${NAMESPACE} -- rabbitmqctl await_online_nodes 3
echo "rabbitmq-2 updated successfully"

# 3. Node 1 (Follower) 업데이트
echo ""
echo "Step 3: Updating rabbitmq-1"
kubectl delete pod rabbitmq-1 -n ${NAMESPACE}
kubectl wait --for=condition=Ready pod/rabbitmq-1 -n ${NAMESPACE} --timeout=300s
kubectl exec rabbitmq-0 -n ${NAMESPACE} -- rabbitmqctl await_online_nodes 3
echo "rabbitmq-1 updated successfully"

# 4. Node 0 (Leader) 업데이트
echo ""
echo "Step 4: Updating rabbitmq-0 (Leader)"
echo "Note: Leader election will occur"
kubectl delete pod rabbitmq-0 -n ${NAMESPACE}
kubectl wait --for=condition=Ready pod/rabbitmq-0 -n ${NAMESPACE} --timeout=300s
kubectl exec rabbitmq-0 -n ${NAMESPACE} -- rabbitmqctl await_online_nodes 3
echo "rabbitmq-0 updated successfully"

# 5. 최종 확인
echo ""
echo "Step 5: Final verification"
kubectl exec rabbitmq-0 -n ${NAMESPACE} -- rabbitmqctl cluster_status
kubectl exec rabbitmq-0 -n ${NAMESPACE} -- rabbitmqctl list_queues name state leader

echo ""
echo "=== Rolling update completed successfully ==="
```

### 3.2 백업 및 복원

```bash
#!/bin/bash
# backup-definitions.sh

set -euo pipefail

BACKUP_DIR=${1:-/backup/rabbitmq}
NAMESPACE="platform-mq"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

echo "=== RabbitMQ Definitions Backup ==="

mkdir -p ${BACKUP_DIR}

# Definitions 내보내기
kubectl exec rabbitmq-0 -n ${NAMESPACE} -- \
  rabbitmqctl export_definitions /tmp/definitions.json

kubectl cp ${NAMESPACE}/rabbitmq-0:/tmp/definitions.json \
  ${BACKUP_DIR}/definitions_${TIMESTAMP}.json

echo "Backup saved to: ${BACKUP_DIR}/definitions_${TIMESTAMP}.json"

# 오래된 백업 정리 (30일)
find ${BACKUP_DIR} -name "definitions_*.json" -mtime +30 -delete

echo "Backup completed"
```

```bash
#!/bin/bash
# restore-definitions.sh

set -euo pipefail

BACKUP_FILE=${1:?Usage: restore-definitions.sh <backup-file>}
NAMESPACE="platform-mq"

echo "=== RabbitMQ Definitions Restore ==="
echo "Restoring from: ${BACKUP_FILE}"

# 백업 파일 복사
kubectl cp ${BACKUP_FILE} ${NAMESPACE}/rabbitmq-0:/tmp/definitions.json

# Definitions 복원
kubectl exec rabbitmq-0 -n ${NAMESPACE} -- \
  rabbitmqctl import_definitions /tmp/definitions.json

echo "Restore completed"

# 확인
kubectl exec rabbitmq-0 -n ${NAMESPACE} -- rabbitmqctl list_vhosts
kubectl exec rabbitmq-0 -n ${NAMESPACE} -- rabbitmqctl list_users
```

---

## 4. 연락처 및 Escalation

### 4.1 Escalation Matrix

| Level | 조건 | 담당자 | 연락처 |
|-------|------|--------|--------|
| L1 | 일반 알람 (Warning) | Platform 당직자 | Slack: #platform-oncall |
| L2 | Critical 알람 (15분 미복구) | Platform 리더 | PagerDuty |
| L3 | 클러스터 전체 장애 | Platform CTO | 긴급 연락망 |

### 4.2 주요 연락처

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Emergency Contacts                                │
│                                                                          │
│  Platform Team                                                           │
│  - Slack: #platform-mq                                                   │
│  - PagerDuty: platform-mq-oncall                                         │
│  - Email: platform-mq@company.com                                        │
│                                                                          │
│  Security Team (보안 이슈)                                               │
│  - Slack: #security-incident                                             │
│  - Email: security@company.com                                           │
│                                                                          │
│  Cloud Infrastructure (Kubernetes/Network)                               │
│  - Slack: #cloud-infra                                                   │
│  - PagerDuty: cloud-infra-oncall                                         │
│                                                                          │
│  Vendor Support (RabbitMQ/VMware)                                        │
│  - Support Portal: https://support.vmware.com                            │
│  - Contract ID: XXXXX-XXXXX                                              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 문서 변경 이력

| 버전 | 일자 | 변경 내용 | 작성자 |
|------|------|----------|--------|
| 1.0.0 | 2026-02-27 | 초안 작성 | Platform Architecture Team |

---

END OF DOCUMENT
