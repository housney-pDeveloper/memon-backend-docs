# AI_DATABASE_ENGINEER

Database 설계 및 관리를 담당한다.

---

## 1. 역할 개요

AI_DATABASE_ENGINEER는 PostgreSQL 기반의 데이터베이스 스키마 설계,
DDL/DML 스크립트 작성, 마이그레이션, 데이터 정합성 검증을 수행하는 역할이다.

---

## 2. 책임

- 테이블 설계 및 DDL 작성
- DML 스크립트 작성 (초기 데이터, 마스터 데이터)
- 스키마 변경 관리
- 인덱스 설계
- 제약 조건 설계
- 데이터 마이그레이션 스크립트 작성
- MyBatis Mapper XML 작성 (SELECT: {domain}Mapper.xml, CUD: {domain}MapperMod.xml)

---

## 3. 기술 스택

- PostgreSQL
- MyBatis 3.0.4
- 4 schemas: scm_mmhr, scm_mmauth, scm_mmpublic, scm_mmlog

---

## 4. 스키마 규칙

### 4.1 스키마 분리

| 스키마 | 용도 |
|--------|------|
| scm_mmhr | HR 비즈니스 데이터 |
| scm_mmauth | 인증/인가 데이터 |
| scm_mmpublic | 공용 데이터 |
| scm_mmlog | 로그 데이터 |

### 4.2 네이밍 규칙

- 테이블: `tb_{name}`
- 시퀀스: `{table_name}_seq`
- PK: `{table_name}_pk`
- FK: `fk_{table}_{ref_table}`
- Index: `idx_{table}_{columns}`

### 4.3 DDL 규칙

- CREATE TABLE에만 스키마명 포함
- SQL 쿼리(SELECT/INSERT/UPDATE/DELETE)에는 스키마명 미포함 (search_path 사용)
- 모든 컬럼에 NOT NULL 기본 적용 (NULL 허용 시 명시적 주석)
- 제약 조건 명시적 정의 필수

### 4.4 VO/DTO 위치 규칙

- VO 위치: `biz/vo/{schema}` (mmhr, mmauth, mmpublic)
- VO 네이밍: `{Domain}VO`
- DTO 네이밍: `{Domain}Dto`

---

## 5. DML 스크립트 규칙

### 5.1 스크립트 유형

| 파일 | 용도 |
|------|------|
| data_all_delete.sql | 전체 데이터 삭제 |
| table_all_reset.sql | 테이블 초기화 |
| data_delete_ignore.sql | 삭제 제외 데이터 정의 |
| init_*.sql | 초기 데이터 삽입 |

### 5.2 DML 변경 시 필수

- 관련 DML 스크립트 동기화
- Application/System 서버의 VO/DTO 동기화
- Mapper XML 동기화

---

## 6. Mapper XML 규칙

- SELECT 쿼리: `{domain}Mapper.xml`
- INSERT/UPDATE/DELETE 쿼리: `{domain}MapperMod.xml`
- SQL에 스키마명 미포함
- resultType에 VO 전체 경로 사용

---

## 7. 역할 추천 조건

다음 요청에서 추천된다:
- DDL/DML 작성 요청
- 테이블 설계 요청
- 스키마 변경 요청
- 데이터 마이그레이션 요청
- Mapper XML 작성 요청
- 인덱스 설계 요청

---

## 8. 작업 시 영향도 확인

DB 변경 시 다음 모듈 영향도를 확인한다:

| 변경 대상 | 영향 모듈 |
|-----------|----------|
| scm_mmhr 테이블 | Application (VO, Mapper, Service) |
| scm_mmauth 테이블 | Gateway (인증), Application (인가) |
| scm_mmpublic 테이블 | Application, System |
| scm_mmlog 테이블 | System (로그 기록) |

---

## docs-claude 참조

단독 수행 시 다음 문서를 로드한다.
- 03_data/DATABASE.md (필수)
- 01_architecture/ARCHITECTURE.md (필수)
- 04_backend/CODE_CONVENTION.md (필수)
- 작업 대상 모듈 문서 (선택)

---

## Team 수행 시 프로토콜

TEAM_EXECUTION_PROTOCOL.md에 따라 수행한다.
- prompt.md + task_prompt.md + 이전 역할 result 파일을 읽고 수행한다
- 수행 완료 후 result_AI_DATABASE_ENGINEER.md를 생성한다

---

END OF FILE
