# DATABASE.md

데이터베이스 설계, 스키마, 네이밍, DML 스크립트, MyBatis 매퍼 규칙을 통합 정의한다.

---

## 1. 스키마 구성

MM 프로젝트는 용도별로 스키마를 분리한다.

| 스키마 | 용도 | 설명 |
|--------|------|------|
| scm_mmhr | HR 도메인 데이터 | 사용자, 고객사 등 |
| scm_mmauth | 인증 데이터 | 인증, 권한 관련 |
| scm_mmpublic | 공통 데이터 | 공통코드, 약관 등 |
| scm_mmlog | 로그/이력 데이터 | 시스템 로그, 발송 이력, MQ 로그 |

### 1.1 스키마별 테이블 분류

**scm_mmhr (HR 도메인):**
- tb_user, tb_client, tb_user_client, tb_user_invite 등

**scm_mmauth (인증):**
- tb_user_auth, tb_user_client_auth, tb_user_terms_agree, tb_oauth_login_history

**scm_mmpublic (공통):**
- tb_code_mngmn, tb_code, tb_terms, tb_outbox_event

**scm_mmlog (로그):**
- tb_access_log, tb_error_log, tb_audit_log, tb_msg_send_log, tb_mq_consume_log, tb_mq_dead_letter

### 1.2 스키마 분리 목적

| 목적 | 설명 |
|------|------|
| 최소 권한 원칙 | 서버별로 필요한 스키마만 접근 |
| 데이터 보호 | 도메인 데이터 오염 방지 |
| 독립 운영 | 로그 백업/파티셔닝/아카이빙 분리 |
| 성능 격리 | 로그 I/O가 도메인에 영향 최소화 |
| 감사 용이 | 접근 경계 명확화 |

---

## 2. DB 사용자 구성

### 2.1 사용자별 권한

| 계정 | 용도 | 접근 스키마 |
|------|------|------------|
| manager | DB 소유자 | 전체 (DDL) |
| ddl_mmuser | DDL 전용 | 전체 (DDL) |
| mmuser | Application/Batch | 전체 (CRUD) |
| mmlog | System worker-log | scm_mmlog만 (CRUD) |

### 2.2 권한 부여

```sql
-- mmuser: 전체 스키마 CRUD
GRANT USAGE ON SCHEMA scm_mmhr, scm_mmauth, scm_mmpublic, scm_mmlog TO mmuser;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA scm_mmhr TO mmuser;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA scm_mmauth TO mmuser;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA scm_mmpublic TO mmuser;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA scm_mmlog TO mmuser;

-- mmlog: scm_mmlog만 접근
GRANT USAGE ON SCHEMA scm_mmlog TO mmlog;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA scm_mmlog TO mmlog;
```

### 2.3 search_path 설정

```sql
-- mmuser: 전체 스키마
ALTER ROLE mmuser IN DATABASE mmdb
    SET search_path = scm_mmhr, scm_mmauth, scm_mmpublic, scm_mmlog;

-- mmlog: scm_mmlog만
ALTER ROLE mmlog IN DATABASE mmdb
    SET search_path = scm_mmlog;
```

---

## 3. 테이블 정책

### 3.1 파일 위치

스키마별로 디렉토리를 분리한다.

```
database/tables/
├── scm_mmhr/
│   ├── tb_user.sql
│   └── tb_client.sql
├── scm_mmauth/
│   ├── tb_user_auth.sql
│   └── tb_user_client_auth.sql
├── scm_mmpublic/
│   ├── tb_code.sql
│   └── tb_terms.sql
└── scm_mmlog/
    ├── tb_access_log.sql
    ├── tb_msg_send_log.sql
    └── tb_mq_consume_log.sql
```

### 3.2 SQL 작성 규칙

- 파일 상단에 테이블 목적을 주석으로 명시
- CREATE TABLE 시 반드시 스키마를 명시

```sql
-- 고객사 정보를 저장하는 테이블
CREATE TABLE scm_mmhr.tb_client (
    client_no BIGSERIAL PRIMARY KEY,
    client_name VARCHAR(100) NOT NULL,
    ...
);
```

### 3.3 기본 작성 원칙

- PK는 명시적으로 constraint 생성
- sequence는 별도 생성
- 모든 column에는 comment 작성
- index는 필요한 경우 명시적으로 생성
- cascade 옵션은 명확히 검토 후 사용

### 3.4 로그 테이블 특별 규칙 (scm_mmlog)

**파티셔닝 권장:**
```sql
CREATE TABLE scm_mmlog.tb_access_log (
    log_seq BIGSERIAL,
    created_at TIMESTAMP NOT NULL,
    ...
) PARTITION BY RANGE (created_at);
```

**필수 컬럼:** created_at (생성일시), created_by (생성자, 선택)
**필수 인덱스:** created_at 기준 인덱스

**아카이빙:**
- 90일 이상 로그: 아카이브 테이블 이동
- 365일 이상 로그: 삭제 또는 외부 스토리지 이관

**백업:**
- scm_mmhr, scm_mmauth, scm_mmpublic: 일일 전체 백업
- scm_mmlog: 주간 전체 백업 (용량 고려)

---

## 4. 네이밍 규칙

| 대상 | 패턴 | 예시 |
|------|------|------|
| 테이블 | tb_{entity} | tb_client |
| 시퀀스 | {tableName}_seq | tb_client_seq |
| Primary Key | {tableName}_pk | tb_client_pk |
| Unique Key | {tableName}_uk01, uk02 ... | tb_client_uk01 |
| Index | {tableName}_idx01, idx02 ... | tb_client_idx01 |

- 모든 constraint는 명시적으로 이름을 지정한다.
- 자동 생성 constraint 사용 금지

---

## 5. DML 스크립트 정책

### 5.1 위치

database/dml_script/

### 5.2 파일 목적

| 파일 | 목적 |
|------|------|
| data_all_delete.sql | 모든 테이블 데이터 삭제 (create 직후 상태로 복구) |
| data_delete_ignore.sql | data_all_delete에서 제외할 테이블 정의 (예: tb_code, tb_terms) |
| table_all_reset.sql | 모든 테이블 drop 후 create (schema 포함 필수) |

### 5.3 변경 프로세스

database/tables/** 중 하나라도 변경이 발생한 경우 필수 후속 작업:

1. data_all_delete.sql 수정 여부 확인
2. data_delete_ignore.sql 수정 여부 확인
3. table_all_reset.sql 수정 여부 확인

> 테이블 변경 후 dml_script 미검토는 운영 장애로 이어질 수 있다.

---

## 6. MyBatis 매퍼 규칙

### 6.1 Mapper 파일 분리

- {domain}Mapper.xml → SELECT 전용
- {domain}MapperMod.xml → INSERT/UPDATE/DELETE/CALL

### 6.2 스키마 명시 금지

SQL에서 스키마명 작성 금지 (search_path 설정 전제)

```sql
-- X
FROM scm_mmhr.tb_client
-- O
FROM tb_client
```

### 6.3 날짜 포맷 규칙

- 날짜 문자열: YYYYMMDD
- 시간 포함: YYYYMMDDHH24MISS

```sql
TO_CHAR(reg_date, 'YYYYMMDD')
```

---

## 7. 금지 사항

- 스키마 없이 테이블 생성 금지
- 잘못된 스키마에 테이블 생성 금지
- 로그 테이블을 scm_mmhr/scm_mmauth/scm_mmpublic에 생성 금지
- 도메인 테이블을 scm_mmlog에 생성 금지
- worker-log에서 scm_mmlog 외 스키마 접근 시도 금지
- SQL에서 스키마명 명시 금지 (CREATE TABLE 제외)
- 자동 생성 constraint 사용 금지
- 테이블 변경 후 dml_script 동기화 누락 금지

---

END OF FILE
