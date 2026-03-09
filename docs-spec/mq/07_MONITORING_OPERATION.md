# 07. 모니터링 및 운영 설계

```
Document ID    : MQ-SPEC-001-07
Version        : 1.0.0
Classification : Internal / Confidential
```

---

## 1. 모니터링 아키텍처

### 1.1 전체 구조

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Monitoring Architecture                              │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    RabbitMQ Cluster                              │    │
│  │                                                                  │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                       │    │
│  │  │  Node-0  │  │  Node-1  │  │  Node-2  │                       │    │
│  │  │  :15692  │  │  :15692  │  │  :15692  │ Prometheus Metrics    │    │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘                       │    │
│  └───────┼─────────────┼─────────────┼─────────────────────────────┘    │
│          │             │             │                                   │
│          └─────────────┼─────────────┘                                   │
│                        │                                                 │
│                        ▼                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    Prometheus                                    │    │
│  │                                                                  │    │
│  │  ServiceMonitor: rabbitmq                                        │    │
│  │  Scrape Interval: 15s                                            │    │
│  │                                                                  │    │
│  │  ┌───────────────────────────────────────────────────────┐      │    │
│  │  │  Recording Rules                                       │      │    │
│  │  │  - rabbitmq:queue_messages:rate5m                      │      │    │
│  │  │  - rabbitmq:connection_count:sum                       │      │    │
│  │  │  - rabbitmq:message_publish_rate:rate1m                │      │    │
│  │  └───────────────────────────────────────────────────────┘      │    │
│  └───────────────────────────┬─────────────────────────────────────┘    │
│                              │                                           │
│          ┌───────────────────┼───────────────────┐                      │
│          │                   │                   │                      │
│          ▼                   ▼                   ▼                      │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐                │
│  │   Grafana    │   │ Alertmanager │   │   Logging    │                │
│  │              │   │              │   │   (ELK)      │                │
│  │  Dashboards  │   │  → Slack     │   │              │                │
│  │  - Overview  │   │  → PagerDuty │   │  - Access    │                │
│  │  - Queues    │   │  → Email     │   │  - Error     │                │
│  │  - Nodes     │   │              │   │  - Audit     │                │
│  └──────────────┘   └──────────────┘   └──────────────┘                │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 ServiceMonitor 설정

```yaml
# ServiceMonitor for RabbitMQ
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: rabbitmq
  namespace: platform-mq-monitoring
  labels:
    app: rabbitmq
    release: prometheus
spec:
  selector:
    matchLabels:
      app: rabbitmq
  namespaceSelector:
    matchNames:
      - platform-mq
  endpoints:
    - port: prometheus
      path: /metrics
      interval: 15s
      scrapeTimeout: 10s
      scheme: http
      honorLabels: true
      metricRelabelings:
        # vhost 라벨 정규화
        - sourceLabels: [vhost]
          regex: "/"
          replacement: "default"
          targetLabel: vhost
```

---

## 2. 필수 모니터링 지표

### 2.1 지표 분류

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Monitoring Metrics Categories                       │
│                                                                          │
│  Category 1: Cluster Health (클러스터 상태)                              │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  rabbitmq_running                    - 노드 실행 상태           │    │
│  │  rabbitmq_cluster_nodes              - 클러스터 노드 수         │    │
│  │  rabbitmq_partitions                 - 네트워크 파티션 수       │    │
│  │  rabbitmq_alarms_*                   - 알람 상태 (memory/disk)  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Category 2: Resource Usage (리소스 사용량)                              │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  rabbitmq_process_resident_memory_bytes  - 메모리 사용량        │    │
│  │  rabbitmq_disk_space_available_bytes     - 디스크 여유 공간     │    │
│  │  rabbitmq_connections                    - 현재 연결 수         │    │
│  │  rabbitmq_channels                       - 현재 채널 수         │    │
│  │  process_cpu_seconds_total               - CPU 사용량           │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Category 3: Message Flow (메시지 흐름)                                   │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  rabbitmq_global_messages_published_total    - 발행 메시지 수   │    │
│  │  rabbitmq_global_messages_delivered_total    - 전달 메시지 수   │    │
│  │  rabbitmq_global_messages_acknowledged_total - ACK 메시지 수    │    │
│  │  rabbitmq_global_messages_redelivered_total  - 재전달 메시지 수 │    │
│  │  rabbitmq_global_messages_unroutable_*       - 라우팅 실패      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Category 4: Queue Metrics (큐 지표)                                      │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  rabbitmq_queue_messages                 - 큐 메시지 수         │    │
│  │  rabbitmq_queue_messages_ready           - 대기 중 메시지       │    │
│  │  rabbitmq_queue_messages_unacked         - 미확인 메시지        │    │
│  │  rabbitmq_queue_consumers                - Consumer 수          │    │
│  │  rabbitmq_queue_messages_published_total - 큐별 발행 수         │    │
│  │  rabbitmq_queue_messages_delivered_total - 큐별 전달 수         │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Category 5: Quorum Queue Specific                                       │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  rabbitmq_raft_term_total               - Raft term 수          │    │
│  │  rabbitmq_raft_log_snapshot_index       - 스냅샷 인덱스         │    │
│  │  rabbitmq_raft_log_last_applied_index   - 마지막 적용 인덱스    │    │
│  │  rabbitmq_raft_entry_commit_latency_*   - Raft 커밋 지연        │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 핵심 메트릭 대시보드

```yaml
# Grafana Dashboard JSON (핵심 패널)
panels:
  # 클러스터 상태 패널
  - title: "Cluster Status"
    type: stat
    targets:
      - expr: "count(rabbitmq_running == 1)"
        legendFormat: "Running Nodes"
    thresholds:
      - value: 3
        color: green
      - value: 2
        color: yellow
      - value: 1
        color: red

  # 메시지 처리율 패널
  - title: "Message Throughput"
    type: graph
    targets:
      - expr: "rate(rabbitmq_global_messages_published_total[5m])"
        legendFormat: "Published/s"
      - expr: "rate(rabbitmq_global_messages_delivered_total[5m])"
        legendFormat: "Delivered/s"
      - expr: "rate(rabbitmq_global_messages_acknowledged_total[5m])"
        legendFormat: "Acknowledged/s"

  # 큐 depth 패널
  - title: "Queue Depth by Queue"
    type: graph
    targets:
      - expr: "rabbitmq_queue_messages{vhost='/eventflow'}"
        legendFormat: "{{ queue }}"

  # 메모리 사용량 패널
  - title: "Memory Usage"
    type: gauge
    targets:
      - expr: |
          rabbitmq_process_resident_memory_bytes /
          rabbitmq_resident_memory_limit_bytes * 100
        legendFormat: "Memory %"
    thresholds:
      - value: 60
        color: green
      - value: 70
        color: yellow
      - value: 80
        color: red
```

---

## 3. 알람 기준

### 3.1 Critical 알람 (즉시 대응)

```yaml
# Critical Alerts - P1, 즉시 대응 필요
groups:
  - name: rabbitmq_critical
    rules:
      # 노드 다운
      - alert: RabbitMQNodeDown
        expr: up{job="rabbitmq"} == 0
        for: 1m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "RabbitMQ node {{ $labels.instance }} is down"
          runbook_url: "https://runbook.internal/rabbitmq/node-down"

      # 클러스터 파티션
      - alert: RabbitMQClusterPartition
        expr: rabbitmq_partitions > 0
        for: 0m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "RabbitMQ cluster partition detected"
          runbook_url: "https://runbook.internal/rabbitmq/partition"

      # Memory 알람 발생
      - alert: RabbitMQMemoryAlarm
        expr: rabbitmq_alarms_memory_used_watermark == 1
        for: 0m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "RabbitMQ memory alarm - publishers blocked"
          runbook_url: "https://runbook.internal/rabbitmq/memory-alarm"

      # Disk 알람 발생
      - alert: RabbitMQDiskAlarm
        expr: rabbitmq_alarms_free_disk_space_watermark == 1
        for: 0m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "RabbitMQ disk alarm - publishers blocked"
          runbook_url: "https://runbook.internal/rabbitmq/disk-alarm"

      # Quorum Queue 손실
      - alert: RabbitMQQuorumLost
        expr: |
          count by (queue, vhost) (
            rabbitmq_raft_members{role="leader"}
          ) == 0
        for: 1m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Quorum queue {{ $labels.queue }} has no leader"
```

### 3.2 Warning 알람 (모니터링 필요)

```yaml
# Warning Alerts - P2, 모니터링 필요
groups:
  - name: rabbitmq_warning
    rules:
      # 높은 메모리 사용량
      - alert: RabbitMQHighMemory
        expr: |
          rabbitmq_process_resident_memory_bytes /
          rabbitmq_resident_memory_limit_bytes > 0.7
        for: 10m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "RabbitMQ memory usage > 70%"

      # 높은 Connection 수
      - alert: RabbitMQHighConnections
        expr: rabbitmq_connections > 3000
        for: 15m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "RabbitMQ connections > 3000"

      # 큐 메시지 적체
      - alert: RabbitMQQueueBacklog
        expr: rabbitmq_queue_messages_ready > 50000
        for: 15m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Queue {{ $labels.queue }} has > 50k messages"

      # Consumer 없는 큐 (메시지 있음)
      - alert: RabbitMQNoConsumer
        expr: |
          rabbitmq_queue_consumers == 0 and
          rabbitmq_queue_messages > 0
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Queue {{ $labels.queue }} has no consumers"

      # DLQ 메시지 증가
      - alert: RabbitMQDLQGrowing
        expr: |
          increase(rabbitmq_queue_messages{queue=~".*\\.dlq"}[1h]) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "DLQ {{ $labels.queue }} growing rapidly"

      # 높은 재전달률
      - alert: RabbitMQHighRedeliveryRate
        expr: |
          rate(rabbitmq_global_messages_redelivered_total[5m]) /
          rate(rabbitmq_global_messages_delivered_total[5m]) > 0.1
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "High message redelivery rate (>10%)"
```

### 3.3 알람 라우팅 설정

```yaml
# Alertmanager Configuration
route:
  receiver: platform-default
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    # Critical - PagerDuty + Slack
    - match:
        severity: critical
      receiver: platform-critical
      continue: true

    # Warning - Slack only
    - match:
        severity: warning
      receiver: platform-warning

receivers:
  - name: platform-default
    slack_configs:
      - channel: '#platform-alerts'
        send_resolved: true

  - name: platform-critical
    pagerduty_configs:
      - service_key: ${PAGERDUTY_SERVICE_KEY}
        description: '{{ .GroupLabels.alertname }}'
    slack_configs:
      - channel: '#platform-critical'
        send_resolved: true

  - name: platform-warning
    slack_configs:
      - channel: '#platform-alerts'
        send_resolved: true
```

---

## 4. DLQ 폭증 대응 전략

### 4.1 DLQ 모니터링 대시보드

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        DLQ Monitoring Dashboard                          │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  DLQ Message Count (All vhosts)                                  │    │
│  │                                                                  │    │
│  │   1000 ┤                                          ●●●●●●●       │    │
│  │    800 ┤                                    ●●●●●●              │    │
│  │    600 ┤                              ●●●●●●                    │    │
│  │    400 ┤                        ●●●●●●                          │    │
│  │    200 ┤              ●●●●●●●●●●                                │    │
│  │      0 ┼──────────────────────────────────────────────────────  │    │
│  │        00:00  04:00  08:00  12:00  16:00  20:00  24:00          │    │
│  │                                                                  │    │
│  │  ⚠️ ALERT: DLQ growth rate > 100 msg/hour                        │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌────────────────────────────────────────┐ ┌────────────────────────┐  │
│  │  Top DLQ by Message Count              │ │  DLQ Death Reasons     │  │
│  │                                        │ │                        │  │
│  │  ef.queue.order.dlq         523        │ │  rejected       65%    │  │
│  │  ef.queue.notification.dlq  234        │ │  expired        25%    │  │
│  │  ef.queue.user.dlq           89        │ │  maxlen          10%    │  │
│  │                                        │ │                        │  │
│  └────────────────────────────────────────┘ └────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 DLQ 폭증 대응 절차

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      DLQ Overflow Response Procedure                     │
│                                                                          │
│  Phase 1: 탐지 (Detection)                                               │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Alert: RabbitMQDLQGrowing                                       │    │
│  │  Condition: increase(rabbitmq_queue_messages{dlq}[1h]) > 100     │    │
│  │                                                                  │    │
│  │  → Slack/PagerDuty 알림                                          │    │
│  │  → 당직자 확인                                                   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                   │                                      │
│                                   ▼                                      │
│  Phase 2: 분류 (Triage)                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  1. DLQ 메시지 샘플링 (최근 10개)                                │    │
│  │     $ rabbitmqadmin get queue=ef.queue.order.dlq count=10       │    │
│  │                                                                  │    │
│  │  2. x-death 헤더 분석                                            │    │
│  │     - reason: rejected/expired/maxlen                            │    │
│  │     - original-queue: 원래 큐                                    │    │
│  │     - count: 재시도 횟수                                         │    │
│  │                                                                  │    │
│  │  3. 에러 패턴 분류                                               │    │
│  │     A. 단일 이벤트 타입 → 특정 Consumer 문제                     │    │
│  │     B. 다양한 이벤트 → 시스템 장애                               │    │
│  │     C. 특정 시간대 집중 → 외부 시스템 장애                       │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                   │                                      │
│                          ┌───────┼───────┐                               │
│                          │       │       │                               │
│                          ▼       ▼       ▼                               │
│  Phase 3: 대응 (Response)                                                │
│  ┌────────────────────┐ ┌──────────────┐ ┌──────────────────────┐       │
│  │  A. Consumer 문제  │ │ B. 시스템 장애│ │ C. 외부 시스템 장애  │       │
│  │                    │ │              │ │                      │       │
│  │  1. Consumer 로그  │ │ 1. 장애 복구 │ │ 1. 외부 시스템 확인  │       │
│  │     확인          │ │    우선      │ │ 2. 장애 해소 대기    │       │
│  │  2. 코드 버그 확인│ │ 2. 스케일 아웃│ │ 3. 재처리 스케줄링   │       │
│  │  3. 핫픽스 배포   │ │    검토      │ │                      │       │
│  │  4. 재처리        │ │ 3. 재처리    │ │                      │       │
│  └────────────────────┘ └──────────────┘ └──────────────────────────┘   │
│                                   │                                      │
│                                   ▼                                      │
│  Phase 4: 재처리 (Reprocessing)                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  1. 문제 해결 확인                                               │    │
│  │  2. 재처리 스크립트 실행 (소량 테스트)                           │    │
│  │  3. 정상 확인 후 전체 재처리                                     │    │
│  │  4. DLQ 비우기                                                   │    │
│  │  5. 모니터링 지속                                                │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                   │                                      │
│                                   ▼                                      │
│  Phase 5: 사후 분석 (Post-mortem)                                        │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - 근본 원인 분석                                                │    │
│  │  - 재발 방지 대책 수립                                           │    │
│  │  - 알람 임계치 조정 검토                                         │    │
│  │  - 문서 업데이트                                                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.3 DLQ 재처리 스크립트

```bash
#!/bin/bash
# reprocess-dlq.sh

set -euo pipefail

DLQ_NAME="${1:-ef.queue.order.dlq}"
TARGET_EXCHANGE="${2:-ef.exchange.order}"
ROUTING_KEY="${3:-order.created}"
BATCH_SIZE="${4:-100}"

echo "=== DLQ Reprocessing ==="
echo "DLQ: ${DLQ_NAME}"
echo "Target: ${TARGET_EXCHANGE} / ${ROUTING_KEY}"
echo "Batch Size: ${BATCH_SIZE}"
echo ""

# DLQ 메시지 수 확인
MSG_COUNT=$(rabbitmqadmin -V /eventflow list queues name messages \
  | grep "${DLQ_NAME}" | awk '{print $4}')

echo "Messages in DLQ: ${MSG_COUNT}"
echo ""

read -p "Proceed with reprocessing? (yes/no): " CONFIRM
if [ "$CONFIRM" != "yes" ]; then
    echo "Aborted."
    exit 0
fi

# Shovel을 통한 재처리
rabbitmqctl set_parameter shovel dlq-reprocess "{
  \"src-uri\": \"amqp:///eventflow\",
  \"src-queue\": \"${DLQ_NAME}\",
  \"dest-uri\": \"amqp:///eventflow\",
  \"dest-exchange\": \"${TARGET_EXCHANGE}\",
  \"dest-exchange-key\": \"${ROUTING_KEY}\",
  \"src-prefetch-count\": ${BATCH_SIZE},
  \"ack-mode\": \"on-confirm\",
  \"src-delete-after\": \"never\"
}"

echo ""
echo "Shovel 'dlq-reprocess' created."
echo "Monitor progress: rabbitmqctl shovel_status"
echo ""
echo "To stop: rabbitmqctl clear_parameter shovel dlq-reprocess"
```

---

## 5. 메시지 적체 대응 전략

### 5.1 적체 탐지 알람

```yaml
# Queue Backlog Alerts
groups:
  - name: rabbitmq_backlog
    rules:
      # 절대값 기준 적체
      - alert: RabbitMQQueueBacklogHigh
        expr: rabbitmq_queue_messages_ready > 100000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Queue {{ $labels.queue }} backlog > 100k"

      # 증가율 기준 적체
      - alert: RabbitMQQueueGrowingFast
        expr: |
          rate(rabbitmq_queue_messages_published_total[5m]) >
          rate(rabbitmq_queue_messages_delivered_total[5m]) * 1.5
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Queue {{ $labels.queue }} growing faster than consuming"

      # Consumer 처리 지연
      - alert: RabbitMQSlowConsumer
        expr: |
          rabbitmq_queue_messages_unacked /
          rabbitmq_queue_consumers > 100
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Slow consumer on {{ $labels.queue }}"
```

### 5.2 적체 대응 절차

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Message Backlog Response Procedure                    │
│                                                                          │
│  Step 1: 현황 파악                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  $ rabbitmqadmin -V /eventflow list queues \                     │    │
│  │      name messages messages_ready consumers                      │    │
│  │                                                                  │    │
│  │  Queue                    Messages  Ready  Consumers             │    │
│  │  ef.queue.order.created   150000   148500      5                │    │
│  │  ef.queue.user.updated      2000     1800     10                │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                   │                                      │
│                                   ▼                                      │
│  Step 2: 원인 분석                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  A. Consumer 수 부족                                             │    │
│  │     → Consumer Pod 스케일 아웃                                   │    │
│  │                                                                  │    │
│  │  B. Consumer 처리 속도 느림                                      │    │
│  │     → 외부 의존성 확인 (DB, API)                                 │    │
│  │     → Consumer 로직 최적화                                       │    │
│  │                                                                  │    │
│  │  C. 트래픽 급증 (이벤트 폭발)                                    │    │
│  │     → Rate Limiting 검토                                         │    │
│  │     → 임시 Consumer 증설                                         │    │
│  │                                                                  │    │
│  │  D. Consumer 장애                                                │    │
│  │     → Consumer 재시작                                            │    │
│  │     → 에러 로그 확인                                             │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                   │                                      │
│                                   ▼                                      │
│  Step 3: 긴급 대응                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Option 1: Consumer 스케일 아웃                                  │    │
│  │  $ kubectl scale deployment ef-system --replicas=10              │    │
│  │                                                                  │    │
│  │  Option 2: Prefetch Count 조정 (처리량 증가)                     │    │
│  │  # application.yml 수정 후 재배포                                │    │
│  │  spring.rabbitmq.cache.channel.prefetch-count: 50                │    │
│  │                                                                  │    │
│  │  Option 3: 임시 Consumer 배포                                    │    │
│  │  $ kubectl apply -f emergency-consumer.yaml                      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                   │                                      │
│                                   ▼                                      │
│  Step 4: 모니터링 및 안정화                                              │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - 메시지 처리율 모니터링                                        │    │
│  │  - Queue depth 감소 확인                                         │    │
│  │  - 정상화 후 리소스 원복                                         │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.3 Emergency Consumer Deployment

```yaml
# emergency-consumer.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ef-emergency-consumer
  namespace: eventflow
  labels:
    app: ef-emergency-consumer
    type: emergency
spec:
  replicas: 10  # 긴급 증설
  selector:
    matchLabels:
      app: ef-emergency-consumer
  template:
    metadata:
      labels:
        app: ef-emergency-consumer
        mq-client: "true"
    spec:
      containers:
        - name: consumer
          image: harbor.platform.internal/eventflow/system:latest
          resources:
            requests:
              cpu: "1"
              memory: "2Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "worker-mq,emergency"
            - name: RABBITMQ_PREFETCH_COUNT
              value: "50"
          envFrom:
            - secretRef:
                name: ef-mq-credentials
---
# 긴급 대응 후 삭제
# kubectl delete -f emergency-consumer.yaml
```

---

## 6. 장애 시나리오별 대응 절차

### 6.1 시나리오 1: 단일 노드 장애

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   Scenario: Single Node Failure                          │
│                                                                          │
│  증상:                                                                   │
│  - RabbitMQNodeDown 알람 발생                                           │
│  - 클러스터 3노드 중 1노드 다운                                         │
│  - Quorum Queue 정상 (과반수 유지)                                      │
│                                                                          │
│  영향:                                                                   │
│  - 연결 일시 끊김 (자동 재연결)                                         │
│  - 해당 노드 Leader 큐 → 다른 노드로 전환                               │
│  - 성능 일시 저하 가능                                                  │
│                                                                          │
│  대응:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  1. [자동] Kubernetes가 Pod 재시작 시도                          │    │
│  │                                                                  │    │
│  │  2. [확인] Pod 상태 확인                                         │    │
│  │     $ kubectl get pods -n platform-mq                            │    │
│  │                                                                  │    │
│  │  3. [확인] 클러스터 상태 확인                                    │    │
│  │     $ kubectl exec rabbitmq-0 -- rabbitmqctl cluster_status      │    │
│  │                                                                  │    │
│  │  4. [조치] Pod 재시작 안 되면 수동 개입                          │    │
│  │     $ kubectl delete pod rabbitmq-N --force                      │    │
│  │                                                                  │    │
│  │  5. [확인] 노드 복구 후 큐 상태 확인                             │    │
│  │     $ rabbitmqctl list_queues name state leader                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  복구 시간: 5-10분 (자동 복구)                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 시나리오 2: 네트워크 파티션

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   Scenario: Network Partition                            │
│                                                                          │
│  증상:                                                                   │
│  - RabbitMQClusterPartition 알람 발생                                   │
│  - rabbitmq_partitions > 0                                               │
│  - 일부 노드 간 통신 불가                                               │
│                                                                          │
│  영향:                                                                   │
│  - Quorum Queue: pause_minority 정책 → 소수 파티션 일시 정지            │
│  - 데이터 불일치 가능성                                                 │
│  - 메시지 발행/소비 일부 실패                                           │
│                                                                          │
│  대응:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  1. [긴급] 파티션 상태 확인                                      │    │
│  │     $ rabbitmqctl cluster_status                                 │    │
│  │                                                                  │    │
│  │  2. [분석] 네트워크 문제 원인 파악                               │    │
│  │     - 노드 간 ping 테스트                                        │    │
│  │     - Kubernetes 네트워크 정책 확인                              │    │
│  │     - CNI 플러그인 상태 확인                                     │    │
│  │                                                                  │    │
│  │  3. [조치] 네트워크 문제 해결                                    │    │
│  │                                                                  │    │
│  │  4. [확인] 파티션 자동 복구 확인                                 │    │
│  │     (pause_minority → 자동 재참여)                               │    │
│  │                                                                  │    │
│  │  5. [필요시] 수동 동기화                                         │    │
│  │     $ rabbitmqctl forget_cluster_node rabbit@rabbitmq-N          │    │
│  │     $ rabbitmqctl join_cluster rabbit@rabbitmq-0                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  복구 시간: 네트워크 복구 후 자동 (수분)                                │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.3 시나리오 3: Memory Alarm

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   Scenario: Memory Alarm                                 │
│                                                                          │
│  증상:                                                                   │
│  - RabbitMQMemoryAlarm 알람 발생                                        │
│  - 모든 Publisher 차단됨                                                │
│  - Connection은 유지되나 publish 불가                                   │
│                                                                          │
│  영향:                                                                   │
│  - 신규 메시지 발행 불가                                                │
│  - 기존 메시지 소비는 정상                                              │
│  - Application에서 publish 타임아웃/실패                                │
│                                                                          │
│  대응:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  1. [확인] 메모리 사용 현황                                      │    │
│  │     $ rabbitmqctl status | grep memory                           │    │
│  │                                                                  │    │
│  │  2. [분석] 메모리 소비 원인 파악                                 │    │
│  │     - Queue별 메시지 수 확인                                     │    │
│  │     - Connection/Channel 수 확인                                 │    │
│  │     $ rabbitmqadmin list queues name messages memory             │    │
│  │                                                                  │    │
│  │  3. [긴급 조치] Consumer 증설                                    │    │
│  │     - Queue depth 줄이기                                         │    │
│  │                                                                  │    │
│  │  4. [긴급 조치] 불필요한 Connection 정리                         │    │
│  │     $ rabbitmqctl close_connection <conn_id> "memory alarm"      │    │
│  │                                                                  │    │
│  │  5. [확인] 알람 해제 확인                                        │    │
│  │     $ rabbitmqctl status | grep alarms                           │    │
│  │                                                                  │    │
│  │  6. [후속] 용량 계획 재검토                                      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  복구 시간: 메모리 확보 즉시 (자동)                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.4 시나리오 4: Disk Alarm

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   Scenario: Disk Alarm                                   │
│                                                                          │
│  증상:                                                                   │
│  - RabbitMQDiskAlarm 알람 발생                                          │
│  - 모든 Publisher 차단됨                                                │
│  - 디스크 공간 부족                                                     │
│                                                                          │
│  영향:                                                                   │
│  - Memory Alarm과 동일 (신규 발행 불가)                                 │
│  - 추가로: 메시지 paging 불가 (디스크 쓰기 안됨)                        │
│                                                                          │
│  대응:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  1. [확인] 디스크 사용량 확인                                    │    │
│  │     $ kubectl exec rabbitmq-0 -- df -h /var/lib/rabbitmq         │    │
│  │                                                                  │    │
│  │  2. [긴급 조치] 오래된 로그 삭제                                 │    │
│  │     $ kubectl exec rabbitmq-0 -- \                               │    │
│  │         find /var/log/rabbitmq -mtime +7 -delete                 │    │
│  │                                                                  │    │
│  │  3. [긴급 조치] 처리된 메시지 정리 (Quorum Queue snapshot)       │    │
│  │     - 자동 정리 대기                                             │    │
│  │                                                                  │    │
│  │  4. [조치] Consumer 증설하여 Queue depth 줄이기                  │    │
│  │                                                                  │    │
│  │  5. [후속] PVC 확장 검토                                         │    │
│  │     $ kubectl patch pvc rabbitmq-data-rabbitmq-0 \               │    │
│  │         -p '{"spec":{"resources":{"requests":{"storage":"1Ti"}}}'│    │
│  │                                                                  │    │
│  │  6. [확인] 알람 해제 확인                                        │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  복구 시간: 공간 확보 즉시 (자동)                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

END OF DOCUMENT
