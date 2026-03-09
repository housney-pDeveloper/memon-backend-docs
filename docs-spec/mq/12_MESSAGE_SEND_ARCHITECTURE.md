# 12. 메시지 발송 아키텍처 설계

## 1. 요구사항 분석

### 1.1 비즈니스 요구사항

```
┌─────────────────────────────────────────────────────────────┐
│                    메시지 발송 구조                          │
├─────────────────────────────────────────────────────────────┤
│  발송 요청 (1)  ────────────▶  수신 대상 (N)                │
│                                                              │
│  ┌──────────────┐            ┌──────────────┐               │
│  │ tb_message_send  │ 1      N   │tb_message_target │               │
│  │              │───────────▶│              │               │
│  │ - 발송정보   │            │ - 수신자정보 │               │
│  │ - 공통내용   │            │ - 개인화내용 │               │
│  │ - 템플릿참조 │            │ - 발송결과   │               │
│  └──────────────┘            └──────────────┘               │
│         │                                                    │
│         │ FK (optional)                                      │
│         ▼                                                    │
│  ┌──────────────┐                                           │
│  │tb_message_template│                                          │
│  │              │                                           │
│  │ - 양식정보   │                                           │
│  │ - 치환변수   │                                           │
│  └──────────────┘                                           │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 핵심 특징

| 항목 | 설명 |
|------|------|
| 1:N 관계 | 1건의 발송 요청에 N명의 수신자 |
| 개인화 | 수신자별 다른 메시지 내용 (이름, 금액 등) |
| 템플릿 | 사전 저장된 양식으로 빠른 발송 |
| 결과 관리 | 수신자별 발송 결과 추적 |
| 유형 관리 | SMS/LMS/알림톡 구분 |

### 1.3 데이터 성격

| 테이블 | 데이터 성격 | 적합 스키마 |
|--------|------------|-------------|
| tb_message_template | 비즈니스 마스터 | scm_efpublic |
| tb_message_send | 비즈니스 트랜잭션 | scm_efpublic |
| tb_message_target | 비즈니스 트랜잭션 | scm_efpublic |
| tb_message_send_log | 처리 로그 | scm_eflog |

---

## 2. 아키텍처 제약 조건

### 2.1 현재 스키마 정책

```
┌─────────────────────────────────────────────────────────────┐
│                    스키마 접근 권한                          │
├────────────────┬──────────────┬──────────────┬──────────────┤
│     계정       │ scm_efpublic │  scm_eflog   │   역할       │
├────────────────┼──────────────┼──────────────┼──────────────┤
│ efuser         │     CRUD     │     CRUD     │ Application  │
│ ef_mqworker    │      -       │     CRUD     │ worker-mq    │
└────────────────┴──────────────┴──────────────┴──────────────┘
```

### 2.2 충돌 지점

```
문제: worker-mq가 외부 API 호출 후 결과를 scm_efpublic에 기록해야 함
      그러나 ef_mqworker는 scm_efpublic 접근 불가

[Application]                    [worker-mq]
     │                                │
     │ 1. 발송요청 생성               │
     │    (scm_efpublic)              │
     │                                │
     │ 2. MQ 이벤트 발행 ────────────▶│
     │                                │
     │                                │ 3. 외부 SMS API 호출
     │                                │
     │                          ┌─────┴─────┐
     │                          │  결과 수신 │
     │                          └─────┬─────┘
     │                                │
     │                                │ 4. 결과 저장 필요
     │                                │    (scm_efpublic) ← 접근 불가!
     │                                │
     ▼                                ▼
```

---

## 3. 해결 방안 비교

### 3.1 Option A: 동기식 처리 (Application 직접 발송)

```
┌─────────────────────────────────────────────────────────────┐
│                    동기식 발송 흐름                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [Client] ──▶ [Gateway] ──▶ [Application]                   │
│                                    │                         │
│                                    │ 1. 발송요청 저장        │
│                                    │    (scm_efpublic)       │
│                                    │                         │
│                                    │ 2. SMS API 호출 (동기)  │
│                                    │                         │
│                                    │ 3. 결과 저장            │
│                                    │    (scm_efpublic)       │
│                                    │                         │
│                                    │ 4. 로그 이벤트 발행     │
│                                    │                         │
│  [worker-mq] ◀────────────────────┘                         │
│       │                                                      │
│       │ 5. 로그 저장 (scm_eflog)                            │
│       ▼                                                      │
└─────────────────────────────────────────────────────────────┘

장점:
- 단순한 아키텍처
- 스키마 정책 준수
- 트랜잭션 보장 용이

단점:
- Application 스레드 블로킹
- 대량 발송 시 응답 지연
- SMS API 장애가 Application에 영향
```

### 3.2 Option B: 비동기 + 콜백 MQ 패턴

```
┌─────────────────────────────────────────────────────────────┐
│                    비동기 콜백 흐름                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [Application]                                               │
│       │                                                      │
│       │ 1. 발송요청 저장 (PENDING)                          │
│       │    + Outbox 이벤트 저장                              │
│       │                                                      │
│       │ 2. MQ 발행: message.send.request                        │
│       │                                                      │
│  [worker-mq] ◀──────────────────────────────────────────────┤
│       │                                                      │
│       │ 3. SMS API 호출                                     │
│       │                                                      │
│       │ 4. 결과 MQ 발행: message.send.result                    │
│       │                                                      │
│       │ 5. 로그 저장 (scm_eflog)                            │
│       │                                                      │
│  [Application] ◀────────────────────────────────────────────┤
│       │                                                      │
│       │ 6. 결과 수신 및 저장                                │
│       │    (scm_efpublic)                                   │
│       │                                                      │
└─────────────────────────────────────────────────────────────┘

장점:
- 완전 비동기 처리
- 스키마 정책 준수
- Application 응답 지연 없음

단점:
- MQ 2홉 (복잡도 증가)
- Application이 MQ Consumer 역할 추가
- 결과 수신 지연 가능
```

### 3.3 Option C: 내부 API 콜백 패턴

```
┌─────────────────────────────────────────────────────────────┐
│                    내부 API 콜백 흐름                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [Application]                                               │
│       │                                                      │
│       │ 1. 발송요청 저장 (PENDING)                          │
│       │    + Outbox 이벤트 저장                              │
│       │                                                      │
│       │ 2. MQ 발행: message.send.request                        │
│       │                                                      │
│  [worker-mq] ◀──────────────────────────────────────────────┤
│       │                                                      │
│       │ 3. SMS API 호출                                     │
│       │                                                      │
│       │ 4. Application 내부 API 호출                        │
│       │    POST /internal/msg/result                        │
│       │                                                      │
│  [Application] ◀────────────────────────────────────────────┤
│       │                                                      │
│       │ 5. 결과 저장 (scm_efpublic)                         │
│       │                                                      │
│  [worker-mq]                                                 │
│       │                                                      │
│       │ 6. 로그 저장 (scm_eflog)                            │
│       │                                                      │
└─────────────────────────────────────────────────────────────┘

장점:
- 비동기 처리
- 스키마 정책 준수
- MQ 토폴로지 단순

단점:
- Application 가용성 의존
- 내부 API 추가 개발 필요
- HTTP 오버헤드
```

### 3.4 Option D: 전용 계정 확장 (권장)

```
┌─────────────────────────────────────────────────────────────┐
│                    전용 계정 확장                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  새로운 계정: ef_msgworker                                   │
│                                                              │
│  ┌────────────────┬──────────────┬──────────────┬─────────┐ │
│  │     계정       │ scm_efpublic │  scm_eflog   │  역할   │ │
│  ├────────────────┼──────────────┼──────────────┼─────────┤ │
│  │ ef_mqworker    │      -       │     CRUD     │ 로그    │ │
│  │ ef_msgworker   │  메시지 CRUD │     CRUD     │ 메시지  │ │
│  └────────────────┴──────────────┴──────────────┴─────────┘ │
│                                                              │
│  ef_msgworker 권한 범위:                                     │
│  - scm_efpublic.tb_message_send (CRUD)                          │
│  - scm_efpublic.tb_message_target (CRUD)                        │
│  - scm_efpublic.tb_message_template (SELECT)                    │
│  - scm_eflog.* (CRUD)                                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘

장점:
- 단순한 발송 흐름
- 최소 권한 원칙 (메시지 테이블만)
- 확장 유연성

단점:
- 계정 추가 관리 부담
- 테이블 수준 권한 관리 복잡
```

---

## 4. 권장 방안: Option C + D 하이브리드

### 4.1 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                    권장 아키텍처                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [Application Server]                                        │
│  ├── efuser 계정 (전체 스키마)                              │
│  ├── 발송요청 생성/조회 API                                 │
│  └── scm_efpublic CRUD                                      │
│                                                              │
│  [System worker-log] ─── 로그 전용                          │
│  ├── eflog 계정 (scm_eflog만)                               │
│  ├── 로그 이벤트 소비                                       │
│  └── scm_eflog CRUD                                         │
│                                                              │
│  [System worker-message] ─── 메시지 발송 전용                   │
│  ├── efuser 계정 (전체 스키마)                              │
│  ├── 발송 이벤트 소비                                       │
│  ├── 외부 SMS API 연동                                      │
│  └── scm_efpublic(메시지) + scm_eflog CRUD                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 발송 흐름

```
┌─────────────────────────────────────────────────────────────┐
│                    메시지 발송 흐름                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. [Client] ──▶ POST /api/v1/msg/send                      │
│                                                              │
│  2. [Application]                                            │
│     ├── 발송요청 저장 (tb_message_send, tb_message_target)          │
│     ├── 상태: PENDING                                        │
│     └── Outbox 이벤트 저장                                   │
│                                                              │
│  3. [Outbox Publisher] ──▶ MQ (ex.pf.ef.message.topic)          │
│     └── Routing Key: message.send.request.v1                    │
│                                                              │
│  4. [worker-message] ◀── MQ 수신                                │
│     ├── tb_message_send 조회 (SELECT)                           │
│     ├── tb_message_target 조회 (SELECT)                         │
│     ├── 개인화 메시지 생성                                   │
│     ├── 외부 SMS API 호출                                   │
│     ├── tb_message_target UPDATE (결과 저장)                    │
│     ├── tb_message_send UPDATE (집계 갱신)                      │
│     └── 로그 저장 (scm_eflog)                               │
│                                                              │
│  5. [Client] ──▶ GET /api/v1/msg/send/{sendNo}/status       │
│     └── 발송 결과 조회                                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 Profile 구성

```yaml
# worker-log Profile (로그 전용)
# 포트: 8083
spring:
  profiles: worker-log
  datasource:
    username: eflog          # scm_eflog만 접근
    password: ${LOG_DB_PASSWORD}

---

# worker-message Profile (메시지 발송)
# 포트: 8085
spring:
  profiles: worker-message
  datasource:
    username: efuser         # 전체 스키마 접근
    password: ${DB_PASSWORD}
```

---

## 5. 스키마 설계

### 5.1 테이블 배치

```
┌─────────────────────────────────────────────────────────────┐
│                    테이블 배치                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  scm_efpublic/                                              │
│  ├── tb_message_template    # 메시지 양식                       │
│  ├── tb_message_send        # 발송 요청 (마스터)                │
│  └── tb_message_target      # 발송 대상 (디테일)                │
│                                                              │
│  scm_eflog/                                                 │
│  └── tb_message_send_log    # 발송 처리 로그 (이력)             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 테이블 관계

```
┌──────────────────┐
│ tb_message_template  │
├──────────────────┤
│ template_no (PK) │◀─┐
│ client_no        │  │
│ template_code    │  │
│ message_type     │  │
│ title            │  │
│ content          │  │
│ variables        │  │ FK (optional)
└──────────────────┘  │
                      │
┌──────────────────┐  │
│ tb_message_send      │  │
├──────────────────┤  │
│ send_no (PK)     │──┼──────────────────────┐
│ client_no        │  │                      │
│ template_no (FK) │──┘                      │
│ message_type     │                         │
│ sender_phone     │                         │
│ title            │                         │
│ content          │                         │
│ total_count      │                         │ FK
│ success_count    │                         │
│ fail_count       │                         │
│ send_status      │                         │
│ scheduled_at     │                         │
│ reg_user_no      │                         │
│ reg_date         │                         │
└──────────────────┘                         │
                                             │
┌──────────────────┐                         │
│ tb_message_target    │                         │
├──────────────────┤                         │
│ target_no (PK)   │                         │
│ send_no (FK)     │◀────────────────────────┘
│ recipient_phone  │
│ recipient_name   │
│ variable_data    │  JSON: {"name":"홍길동","amount":"50,000원"}
│ actual_content   │  개인화된 최종 메시지
│ send_status      │  PENDING/SENDING/SUCCESS/FAILED
│ provider_message_id  │
│ provider_result  │
│ sent_at          │
│ result_at        │
└──────────────────┘
```

---

## 6. 계정 권한 설계

### 6.1 계정 구조 (단순화)

```
┌─────────────────────────────────────────────────────────────┐
│                    계정 구조                                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  efuser (전체 권한)                                         │
│  ├── Application                                            │
│  ├── worker-message                                             │
│  └── worker-batch                                           │
│                                                              │
│  eflog (scm_eflog만)                                        │
│  └── worker-log                                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘

worker-message는 efuser를 사용하므로 별도 계정 생성 불필요.
메시지 테이블(scm_efpublic) + 로그 테이블(scm_eflog) 모두 접근 가능.
```

### 6.2 권한 매트릭스 (최종)

| 계정 | scm_efhr | scm_efauth | scm_efpublic | scm_eflog | 사용 Profile |
|------|----------|------------|--------------|-----------|--------------|
| efuser | CRUD | CRUD | CRUD | CRUD | Application, worker-message, worker-batch |
| eflog | - | - | - | CRUD | worker-log |

---

## 7. MQ 토폴로지

### 7.1 Exchange/Queue 추가

```
┌─────────────────────────────────────────────────────────────┐
│                    MQ 토폴로지                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Exchange: ex.pf.ef.message.topic (기존)                        │
│  │                                                           │
│  ├── message.send.request.v1 ──▶ q.pf.ef.message.send.request       │
│  │                           └── worker-message 소비            │
│  │                                                           │
│  └── message.send.# ──▶ q.pf.ef.message.send (기존, 로그용)         │
│                     └── worker-mq 소비 (로그만)             │
│                                                              │
│  Exchange: ex.pf.ef.dlx (기존)                              │
│  │                                                           │
│  └── dlq.message.* ──▶ q.pf.ef.dlq.message.send                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 Queue 설정

```java
// MqConstants.java 추가
public static final String QUEUE_MESSAGE_SEND_REQUEST = "q.pf.ef.message.send.request";
public static final String RK_MESSAGE_SEND_REQUEST = "message.send.request.v1";
```

---

## 8. 대안 비교 요약

| 항목 | Option A (동기) | Option B (MQ 콜백) | Option C (API 콜백) | Option D (전용계정) |
|------|-----------------|-------------------|--------------------|--------------------|
| 복잡도 | 낮음 | 높음 | 중간 | 중간 |
| 스키마 정책 | 준수 | 준수 | 준수 | 부분 확장 |
| 응답 지연 | 높음 | 낮음 | 낮음 | 낮음 |
| 장애 격리 | 낮음 | 높음 | 중간 | 높음 |
| 확장성 | 낮음 | 높음 | 중간 | 높음 |
| 권장 상황 | 소량 발송 | 대규모 | 중규모 | 중규모+ |

### 8.1 권장 조합

**중소규모 (일 1만건 미만)**: Option C (내부 API 콜백)
- worker-mq가 SMS API 호출 후 Application 내부 API 콜백
- 계정 추가 없이 기존 구조 활용

**중대규모 (일 1만건 이상)**: Option D (전용 계정)
- worker-message 신규 Profile + ef_msgworker 계정
- 메시지 테이블 직접 접근으로 성능 최적화

---

## 9. 결론

### 9.1 권장 아키텍처

```
[Phase 1] 즉시 적용
├── 테이블 생성 (scm_efpublic)
│   ├── tb_message_template
│   ├── tb_message_send
│   └── tb_message_target
├── 로그 테이블 유지 (scm_eflog)
│   └── tb_message_send_log (처리 이력)
└── Option C 적용 (내부 API 콜백)

[Phase 2] 트래픽 증가 시
├── ef_msgworker 계정 생성
├── worker-message Profile 추가
└── Option D 적용 (전용 계정)
```

### 9.2 다음 단계

1. scm_efpublic 메시지 테이블 DDL 작성
2. 기존 scm_eflog.tb_message_send_log 역할 재정의 (처리 로그로 한정)
3. Application 내부 API 설계
4. MQ 이벤트 스키마 정의

---

END OF FILE
