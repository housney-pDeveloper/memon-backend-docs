# 11. scm_eflog 스키마 분리 및 접근 제어 정책

## 1. 현황 분석

### 1.1 현재 데이터베이스 구조

```
Database: efdb
Owner: manager

Users:
├── ddl_efuser (DDL 전용, 스키마 생성/변경)
└── efuser (DML 전용, 모든 스키마 접근)

Schemas:
├── scm_efhr (HR 도메인)
├── scm_efauth (인증)
└── scm_efpublic (공통)
```

### 1.2 현재 efuser 권한

| 스키마 | SELECT | INSERT | UPDATE | DELETE |
|--------|--------|--------|--------|--------|
| scm_efhr | O | O | O | O |
| scm_efauth | O | O | O | O |
| scm_efpublic | O | O | O | O |

### 1.3 요구사항

1. **scm_eflog 스키마 신설**: 로그 관련 테이블 이관
2. **eflog 계정 신설**: worker-mq 전용, scm_eflog만 접근
3. **efuser 권한 유지**: scm_eflog 포함 전체 스키마 전권 보유

---

## 2. 설계 원칙

### 2.1 최소 권한 원칙 (Principle of Least Privilege)

| 계정 | 용도 | 접근 가능 스키마 |
|------|------|-----------------|
| efuser | Application, worker-message, worker-batch | 전체 스키마 |
| eflog | worker-log | scm_eflog만 |

### 2.2 스키마 분리 목적

| 목적 | 설명 |
|------|------|
| 보안 격리 | MQ Worker가 도메인 데이터 변경 불가 |
| 장애 격리 | 로그 테이블 문제가 도메인에 영향 없음 |
| 독립 운영 | 로그 파티셔닝/아카이빙 독립 수행 |
| 감사 용이 | 접근 로그 분석 시 명확한 경계 |

---

## 3. 스키마 설계

### 3.1 scm_eflog 테이블 구성

```
scm_eflog/
├── tb_access_log        # API 접근 로그
├── tb_error_log         # 에러 로그
├── tb_audit_log         # 감사 로그
├── tb_message_send_log      # 메시지 발송 로그
├── tb_message_send_result   # 메시지 발송 결과
├── tb_mq_consume_log    # MQ 소비 로그
└── tb_mq_dead_letter    # DLQ 메시지 기록
```

### 3.2 테이블 이관 대상 (scm_efpublic → scm_eflog)

현재 scm_efpublic에 로그성 테이블이 있다면 scm_eflog로 이관.

```sql
-- 예시: outbox_event는 도메인 연관이므로 scm_efpublic 유지
-- 순수 로그 테이블만 scm_eflog로 이관
```

---

## 4. 사용자 및 권한 설계

### 4.1 eflog 계정 생성

```sql
/* =========================================================
 * (manager)
 * MQ Worker 전용 계정 생성
 * ========================================================= */

CREATE USER eflog WITH PASSWORD '${MQ_WORKER_DB_PASSWORD}' LOGIN;

GRANT CONNECT ON DATABASE efdb TO eflog;
```

### 4.2 scm_eflog 스키마 생성 및 권한

```sql
/* =========================================================
 * (manager)
 * scm_eflog 스키마 생성
 * ========================================================= */

CREATE SCHEMA IF NOT EXISTS scm_eflog AUTHORIZATION manager;


/* =========================================================
 * (manager)
 * efuser 권한 부여 (전권)
 * ========================================================= */

-- 스키마 접근
GRANT USAGE ON SCHEMA scm_eflog TO ddl_efuser, efuser;
GRANT ALL ON SCHEMA scm_eflog TO ddl_efuser;

-- 기존 테이블
GRANT SELECT, INSERT, UPDATE, DELETE
    ON ALL TABLES IN SCHEMA scm_eflog
    TO efuser;

GRANT ALL
    ON ALL TABLES IN SCHEMA scm_eflog
    TO ddl_efuser;

-- 신규 테이블 자동 권한
ALTER DEFAULT PRIVILEGES IN SCHEMA scm_eflog
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO efuser;

ALTER DEFAULT PRIVILEGES IN SCHEMA scm_eflog
    GRANT ALL ON TABLES TO ddl_efuser;

-- 시퀀스
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA scm_eflog TO efuser;
GRANT ALL ON ALL SEQUENCES IN SCHEMA scm_eflog TO ddl_efuser;

ALTER DEFAULT PRIVILEGES IN SCHEMA scm_eflog
    GRANT USAGE, SELECT ON SEQUENCES TO efuser;

ALTER DEFAULT PRIVILEGES IN SCHEMA scm_eflog
    GRANT ALL ON SEQUENCES TO ddl_efuser;

-- search_path 갱신
ALTER ROLE efuser IN DATABASE efdb
    SET search_path = scm_efhr, scm_efauth, scm_efpublic, scm_eflog;

ALTER ROLE ddl_efuser IN DATABASE efdb
    SET search_path = scm_efhr, scm_efauth, scm_efpublic, scm_eflog;


/* =========================================================
 * (manager)
 * eflog 권한 부여 (scm_eflog만)
 * ========================================================= */

-- scm_eflog 스키마만 접근 허용
GRANT USAGE ON SCHEMA scm_eflog TO eflog;

-- CRUD 권한
GRANT SELECT, INSERT, UPDATE, DELETE
    ON ALL TABLES IN SCHEMA scm_eflog
    TO eflog;

-- 신규 테이블 자동 권한
ALTER DEFAULT PRIVILEGES IN SCHEMA scm_eflog
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO eflog;

-- 시퀀스 권한
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA scm_eflog TO eflog;

ALTER DEFAULT PRIVILEGES IN SCHEMA scm_eflog
    GRANT USAGE, SELECT ON SEQUENCES TO eflog;

-- search_path (scm_eflog만)
ALTER ROLE eflog IN DATABASE efdb
    SET search_path = scm_eflog;
```

### 4.3 권한 매트릭스 (최종)

| 계정 | scm_efhr | scm_efauth | scm_efpublic | scm_eflog |
|------|----------|------------|--------------|-----------|
| ddl_efuser | ALL | ALL | ALL | ALL |
| efuser | CRUD | CRUD | CRUD | CRUD |
| eflog | - | - | - | CRUD |

---

## 5. Application 설정 변경

### 5.1 application-worker-mq.yml 수정

```yaml
# MQ Worker Profile - 전용 DataSource 설정
spring:
  datasource:
    # eflog 계정 사용 (scm_eflog만 접근 가능)
    url: ${MQ_DB_URL:jdbc:postgresql://localhost:5432/efdb}
    username: ${MQ_DB_USERNAME:eflog}
    password: ${MQ_DB_PASSWORD:}
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 5
      minimum-idle: 2
      connection-timeout: 10000
      # 연결 검증
      connection-test-query: SELECT 1
```

### 5.2 환경변수 (AWS Parameter Store)

```
/fingate/config/platform/ef/mq.db.url       = jdbc:postgresql://{host}:5432/efdb
/fingate/config/platform/ef/mq.db.username  = eflog
/fingate/config/platform/ef/mq.db.password  = {encrypted}
```

### 5.3 MyBatis Mapper 경로 분리 (선택)

```yaml
# worker-mq는 scm_eflog 관련 mapper만 로드
mybatis:
  mapper-locations: classpath:mapper/eflog/**/*.xml
```

---

## 6. 코드 수준 보호

### 6.1 Schema 명시 필수

Mapper XML에서 스키마를 명시하여 실수 방지.

```xml
<!-- 올바른 예 -->
<insert id="insertAccessLog">
    INSERT INTO scm_eflog.tb_access_log (...)
    VALUES (...)
</insert>

<!-- 금지: 스키마 누락 -->
<insert id="insertAccessLog">
    INSERT INTO tb_access_log (...)
</insert>
```

### 6.2 접근 제어 검증

eflog가 다른 스키마 접근 시 PostgreSQL 에러 발생.

```
ERROR: permission denied for schema scm_efhr
```

이 에러는 코드 버그 조기 발견에 도움.

---

## 7. 보안 고려사항

### 7.1 Credential 관리

| 항목 | 정책 |
|------|------|
| 비밀번호 복잡도 | 최소 16자, 특수문자 포함 |
| 저장 위치 | AWS Parameter Store (SecureString) |
| 로테이션 | 90일 주기 |
| 접근 로그 | CloudTrail 기록 |

### 7.2 네트워크 격리

```
┌─────────────────────────────────────────────────────┐
│                    VPC                              │
│  ┌─────────────┐       ┌──────────────────────┐    │
│  │ EKS Cluster │       │   RDS PostgreSQL     │    │
│  │             │       │                      │    │
│  │ worker-mq   │──────▶│ eflog 계정     │    │
│  │ Pod         │  5432 │ (scm_eflog만 접근)   │    │
│  └─────────────┘       └──────────────────────┘    │
│                              │                      │
│  ┌─────────────┐            │                      │
│  │ Application │────────────┘                      │
│  │ Server      │  efuser 계정                      │
│  │             │  (전체 스키마)                    │
│  └─────────────┘                                   │
└─────────────────────────────────────────────────────┘
```

### 7.3 감사 로그

```sql
-- PostgreSQL 로그 설정
ALTER SYSTEM SET log_statement = 'mod';
ALTER SYSTEM SET log_connections = on;
ALTER SYSTEM SET log_disconnections = on;

-- 연결 모니터링
SELECT usename, client_addr, state, query
FROM pg_stat_activity
WHERE usename = 'eflog';
```

---

## 8. 마이그레이션 계획

### Phase 1: 스키마 및 계정 생성 (Day 1)

1. scm_eflog 스키마 생성
2. eflog 계정 생성
3. 권한 부여
4. 연결 테스트

### Phase 2: 테이블 생성 (Day 2)

1. scm_eflog 테이블 DDL 실행
2. 인덱스 생성
3. 시퀀스 생성

### Phase 3: Application 배포 (Day 3)

1. Parameter Store 설정
2. worker-mq 신규 배포
3. 연결 검증
4. 로그 적재 테스트

### Phase 4: 모니터링 (Day 4+)

1. 권한 에러 모니터링
2. 성능 모니터링
3. 로그 볼륨 확인

---

## 9. Rollback 계획

### 9.1 Application Rollback

```bash
# worker-mq를 efuser로 롤백
kubectl set env deployment/system-worker-mq \
  MQ_DB_USERNAME=efuser \
  MQ_DB_PASSWORD=${EFUSER_PASSWORD}
```

### 9.2 Database Rollback

```sql
-- eflog 권한 회수
REVOKE ALL ON SCHEMA scm_eflog FROM eflog;
REVOKE CONNECT ON DATABASE efdb FROM eflog;

-- 계정 삭제 (옵션)
DROP USER IF EXISTS eflog;
```

---

## 10. 체크리스트

### 10.1 사전 검토

- [ ] efuser의 현재 사용 쿼리 중 scm_eflog 참조 확인
- [ ] Application 서버에서 로그 테이블 접근 여부 확인
- [ ] 기존 로그 테이블 이관 대상 확정

### 10.2 구현

- [ ] scm_eflog 스키마 DDL 작성
- [ ] eflog 계정 생성 스크립트 작성
- [ ] 권한 부여 스크립트 작성
- [ ] application-worker-mq.yml 수정
- [ ] Parameter Store 설정

### 10.3 검증

- [ ] eflog로 scm_eflog 접근 성공
- [ ] eflog로 scm_efhr 접근 실패 (기대 동작)
- [ ] efuser로 scm_eflog 접근 성공
- [ ] worker-mq 로그 적재 성공

---

## 11. 결론

### 11.1 정책 요약

| 항목 | 결정 |
|------|------|
| 신규 스키마 | scm_eflog |
| 신규 계정 | eflog |
| eflog 권한 | scm_eflog CRUD만 |
| efuser 권한 | 전체 스키마 (변경 없음) |
| 적용 대상 | worker-mq profile |

### 11.2 기대 효과

1. **보안 강화**: MQ Worker가 도메인 데이터 변경 불가
2. **장애 격리**: 로그 테이블 이슈가 도메인에 영향 없음
3. **감사 용이**: 계정별 접근 로그 분석 가능
4. **운영 독립**: 로그 테이블 독립 운영 가능

---

END OF FILE
