# SYSTEM.md

System 서버 고유 구현 가이드. 공통 컨벤션은 CODE_CONVENTION.md 참조.

---

## 1. Profile 정책

### 1.1 Profile 종류

| Profile | 포트 | 역할 |
|---------|------|------|
| api | 8082 | 공통코드 API, 알림(SSE), Health |
| worker-mq | 8083 | MQ 리스닝 및 이벤트 처리, 로그 적재 |
| worker-batch | 8084 | 배치 작업, 비즈니스 로직 수행 |
| worker-ws | 8085 | WebSocket 연결 관리 (v2 예정) |

### 1.2 실행 예시

```bash
java -jar system.jar --spring.profiles.active=api,dev
java -jar system.jar --spring.profiles.active=worker-mq,dev
java -jar system.jar --spring.profiles.active=worker-batch,dev
```

### 1.3 Profile별 스키마 접근 권한

| Profile | scm_mmhr | scm_mmauth | scm_mmpublic | scm_mmlog |
|---------|----------|------------|--------------|-----------|
| api | SELECT | SELECT | SELECT | SELECT |
| worker-mq | SELECT | - | CRUD | CRUD |
| worker-batch | ALL | ALL | ALL | ALL |
| worker-log | - | - | - | CRUD |

### 1.4 Profile별 허용 작업

**api Profile:**
- 공통코드 조회, SSE 알림 발송, Health Check
- 금지: 스키마 쓰기, 비즈니스 로직, MQ/배치 처리

**worker-mq Profile:**
- MQ 이벤트 소비, 외부 API 연동 (SMS, 알림톡, 이메일)
- 로그 테이블 INSERT/UPDATE (TB_QUE_LOG, TB_SEND_LOG, TB_EVENT_HISTORY)
- 금지: 도메인 엔티티 생성/삭제, HTTP 엔드포인트 노출 (Health 제외)

**worker-batch Profile:**
- 배치 작업, 대용량 데이터 처리, 정기 집계/동기화
- 전체 테이블 접근, 배치 전용 비즈니스 로직 수행
- 금지: HTTP 엔드포인트 노출 (Health 제외), MQ 리스닝

### 1.5 DB 사용자 구성

```sql
-- api Profile 전용
CREATE USER system_api_user WITH PASSWORD '...';
GRANT SELECT ON ALL TABLES IN SCHEMA scm_mmhr, scm_mmlog TO system_api_user;

-- worker-mq Profile 전용
CREATE USER system_mq_user WITH PASSWORD '...';
GRANT SELECT ON ALL TABLES IN SCHEMA scm_mmhr TO system_mq_user;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA scm_mmlog TO system_mq_user;

-- worker-batch Profile 전용
CREATE USER system_batch_user WITH PASSWORD '...';
GRANT ALL ON ALL TABLES IN SCHEMA scm_mmhr, scm_mmlog TO system_batch_user;
```

---

## 2. 이벤트 처리 정책

### 2.1 멱등성 보장

- event_id 기준 중복 체크
- 이미 처리된 이벤트는 ack 후 스킵

### 2.2 재시도 정책

- 최대 3회 재시도
- Exponential Backoff (1s, 2s, 4s)
- 최종 실패 시 DLQ 이동

### 2.3 메시지 스키마

```json
{
  "event_id": "전역 유니크",
  "event_type": "이벤트명",
  "occurred_at": "UTC",
  "trace_id": "추적ID",
  "client_no": 숫자,
  "user_no": 숫자,
  "payload": { ... }
}
```

### 2.4 로그 적재 정책

| 테이블 | 용도 |
|--------|------|
| TB_QUE_LOG | 큐 처리 로그 (RECEIVED/COMPLETED/FAILED/DLQ) |
| TB_SEND_LOG | 발송 로그 (SMS, 알림톡, 이메일) |
| TB_EVENT_HISTORY | 이벤트 처리 이력 |

### 2.5 트랜잭션 원칙

- 이벤트 처리와 로그 적재는 별도 트랜잭션
- 로그 적재 실패가 이벤트 처리를 롤백하지 않음

---

## 3. 디렉토리 구조

```
root
├─ config
│  └─ profile
│     ├─ ApiConfig.java
│     ├─ WorkerMqConfig.java
│     └─ WorkerBatchConfig.java
│
├─ common
│  ├─ annotations
│  ├─ constant
│  ├─ context
│  ├─ filter
│  ├─ exception
│  ├─ response
│  └─ util
│
├─ api
│  ├─ controller
│  └─ service
│
├─ mq
│  ├─ listener
│  ├─ service
│  ├─ dao
│  └─ vo
│
├─ batch
│  ├─ scheduler
│  ├─ job
│  ├─ service
│  ├─ dao
│  └─ vo
│
├─ client
│  └─ ApplicationApiClient
│
└─ SystemApplication.java
```

### 3.1 VO 위치 규칙

- mq VO: mq/vo 디렉토리
- batch VO: batch/vo 디렉토리
- 공통 VO: common/vo (필요 시)

### 3.2 DAO 규칙

**mq DAO:** 로그/이력 테이블 접근 전용 (TB_QUE_LOG, TB_SEND_LOG, TB_EVENT_HISTORY)
**batch DAO:** 모든 테이블 읽기/쓰기 허용

### 3.3 Mapper 위치

```
resources/mapper
├─ mq/
│  └─ MqLogMapper.xml
└─ batch/
   └─ BatchMapper.xml
```

---

## 4. 에러 코드 (7XXXX)

| 범위 | 영역 |
|------|------|
| 7001-7099 | 공통 오류 |
| 7100-7199 | 공통코드 |
| 7200-7299 | 알림 |
| 7300-7399 | MQ |
| 7400-7499 | Batch |
| 7500-7599 | WebSocket (v2 예정) |

공통 코드 (9XXXX): hs-common CommonErrorCode 사용

신규 코드 추가:
1. 7XXXX → System ErrorCode.java 추가
2. 9XXXX → hs-common CommonErrorCode.java 추가

---

## 5. Application 서버 통신

### 5.1 동기 호출 (REST)

ApplicationApiClient를 통해 Application API 호출 가능.
필수 헤더: X-Gateway-Verified, X-Gateway-Secret

### 5.2 비동기 호출 (MQ)

MQ를 통해 Application에 이벤트 발행 가능.

---

## 6. 부하 분리 원칙

- Application: 실시간 사용자 트래픽 전담 (24시간 유입 처리)
- System (worker-mq): 비동기 이벤트 처리 전담
- System (worker-batch): 배치 작업 전담 (수 초 ~ 수 시간)

목적: Application 서버의 응답 지연 방지

---

## 7. 공통 금지 사항

- 하나의 JVM에서 복수 worker Profile 동시 실행 금지
- JWT 생성/검증 금지
- Gateway 인증 헤더 검증 누락 금지
- WebFlux 도입 금지
- 스키마 경계 위반 금지

---

END OF FILE
