# MEMON 요구사항 기술 검토서 v1

> 역할: AI_TECH_LEAD
> 작성일: 2026-03-09
> 대상: MEMON Backend - 홈 대시보드 및 핵심 비즈니스 API
> 상태: Critical 보완 완료 / 개발 착수 가능

---

## 1. 문서 개요

본 문서는 MEMON 서비스의 핵심 비즈니스 요구사항(비즈니스1~11)을 기술적으로 검토한 결과물이다.
기존 아키텍처(Gateway / Application / System), 스키마 정책, 코드 컨벤션, 에러코드 체계를 기반으로
요구사항의 타당성, 누락 사항, 보완 필요 사항, 데이터 모델 설계 방향, API 설계 방향을 정리한다.

---

## 2. 핵심 비즈니스 모델 분석

### 2.1 MEMON의 두 가지 핵심 흐름

```
[흐름 1] 나 → 타인 행사 (보낸 기록, SENT)
┌──────┐    금액 지불    ┌──────────┐
│  나  │ ──────────────→ │ 타인 행사 │
└──────┘                 └──────────┘
특징: 1행사 = 1기록 (일반적)

[흐름 2] 타인 → 나의 행사 (받은 기록, RECEIVED)
┌──────┐    금액 지불    ┌──────────┐
│ 타인A │ ──────────────→ │          │
│ 타인B │ ──────────────→ │ 나의 행사 │
│ 타인C │ ──────────────→ │          │
└──────┘                 └──────────┘
특징: 1행사 = N기록 (다건 등록)
```

### 2.2 핵심 가치

| 항목 | 설명 |
|------|------|
| 기록 단위 | 나 ↔ 특정 관계 간의 금전 이동 1건 |
| 관계 흐름 | 특정 관계와의 시간별 주고받은 금전 히스토리 |
| 전체 통계 | 나 자신의 전체 보낸/받은 금액, 관계별/유형별 분포 |
| 인사이트 | 기록 패턴에서 도출한 관계 흐름 해석 |

### 2.3 요구사항 보완 항목 (Critical - 확정)

**[C-01] 기록 방향(Direction) 개념 추가 ✅ 확정**

요구사항의 기록(tb_record) 정의에 **보낸/받은 구분(Direction)**이 명시되지 않았다.
이는 MEMON의 가장 핵심적인 비즈니스 속성이다.

- 보낸 기록 (SENT): 나 → 타인 행사에 금액 지불
- 받은 기록 (RECEIVED): 타인 → 나의 행사에 금액 지불

> **확정**: tb_record에 `record_direction_cd` 컬럼 추가
> - 코드 그룹: RECORD_DIRECTION (SENT / RECEIVED)
> - NOT NULL 필수값
> - 모든 기록 관련 API 필수값에 포함
> - 통계/대시보드 집계 시 방향별 분리 집계 적용

**[C-02] 행사(Event) 엔티티 1급 승격 ✅ 확정**

요구사항에서 tb_event는 대시보드 API(비즈니스1 API2)에만 등장하고,
기록 관리(비즈니스3)에서는 독립 CRUD가 정의되지 않았다.

그러나 사용자의 핵심 시나리오에 의하면:
- 흐름 2(받은 기록)에서 **1행사에 N건 기록**이 등록됨
- 이는 기록을 그룹핑할 **행사 엔티티**가 필요함을 의미

> **확정**: tb_event를 1급 엔티티로 승격, CRUD API 신설
> - 비즈니스3-A "행사 관리" 도메인 신설
> - tb_event 테이블: event_no, user_no, event_type_cd, event_owner_cd, event_nm, event_dt, relationship_no, memo, deleted_yn
> - event_owner_cd: MINE(내 행사) / OTHERS(타인 행사)
> - OTHERS일 때 relationship_no 필수
> - tb_record.event_no FK로 행사-기록 연결 (nullable, 단독 기록 허용)
> - URL: /api/{version}/event/* (addEvent, getEventList, getEventDetail, updateEvent, deleteEvent)

**[C-03] 코드 테이블 통합 ✅ 확정**

요구사항에서 별도 테이블로 정의된 항목:
- tb_event_type_code
- tb_relationship_type_code
- tb_record_type_code
- tb_schedule_type_code

그러나 프로젝트에는 이미 **tb_code_mngmn + tb_code** 공통 코드 체계가 존재한다.

> **확정**: 별도 _type_code 테이블 생성 금지. 기존 tb_code 체계 활용
> - 요구사항의 모든 _type_code 참조를 tb_code(code_group_id, code_id)로 대체
> - 신규 코드 그룹 9개 추가 (3.2절 참조)
> - 테이블 컬럼에는 code_id 값을 VARCHAR(20) _cd 접미사로 저장
> - 프론트엔드는 공통코드 API(System api profile)로 코드 목록 조회

---

## 3. 데이터 모델 설계 방향

### 3.1 스키마 배치 계획

| 테이블 | 스키마 | 사유 |
|--------|--------|------|
| tb_relationship | scm_mmhr | 핵심 도메인 데이터 |
| tb_event | scm_mmhr | 핵심 도메인 데이터 |
| tb_record | scm_mmhr | 핵심 도메인 데이터 |
| tb_record_attachment | scm_mmhr | 기록 종속 데이터 |
| tb_schedule | scm_mmhr | 핵심 도메인 데이터 |
| tb_user_profile | scm_mmhr | 사용자 확장 정보 |
| tb_user_dashboard_setting | scm_mmhr | 사용자 설정 |
| tb_notification | scm_mmpublic | 시스템 공통 기능 |
| tb_dashboard_insight_snapshot | scm_mmpublic | 집계/캐시 데이터 |
| tb_share_group (v2) | scm_mmhr | 도메인 데이터 |
| tb_share_member (v2) | scm_mmhr | 도메인 데이터 |

### 3.2 공통 코드 그룹 추가 계획

기존 tb_code_mngmn / tb_code 체계에 아래 코드 그룹을 추가한다.

| code_group_id | code_group_nm | 코드 예시 |
|---------------|---------------|-----------|
| RELATIONSHIP_TYPE | 관계 유형 | FAMILY, FRIEND, WORK, ACQUAINTANCE, OTHER |
| EVENT_TYPE | 행사 유형 | WEDDING, FUNERAL, BIRTHDAY, ANNIVERSARY, BABY_FIRST, OTHER |
| EVENT_OWNER_TYPE | 행사 주체 | MINE, OTHERS |
| RECORD_DIRECTION | 기록 방향 | SENT, RECEIVED |
| SCHEDULE_TYPE | 일정 유형 | BIRTHDAY, ANNIVERSARY, REMINDER, OTHER |
| SCHEDULE_REPEAT | 일정 반복 | NONE, YEARLY |
| NOTIFICATION_TYPE | 알림 유형 | SCHEDULE, RECORD, SYSTEM |
| NOTIFICATION_TARGET | 알림 대상 유형 | RECORD, SCHEDULE, RELATIONSHIP, EVENT |
| DASHBOARD_SCOPE | 대시보드 범위 | SELF, SHARED |

### 3.3 핵심 테이블 설계 (확정)

#### 3.3.1 tb_relationship (관계)

```
스키마: scm_mmhr

컬럼:
- relationship_no    BIGINT PK (시퀀스)
- user_no            BIGINT FK → tb_user (소유자)
- relationship_nm    VARCHAR(50) NOT NULL (관계 대상 이름)
- relationship_type_cd VARCHAR(20) NOT NULL (코드: RELATIONSHIP_TYPE)
- phone_no           VARCHAR(20) (선택)
- memo               TEXT (선택)
- importance_level   INTEGER DEFAULT 0 (중요도 0~5)
- birthday           DATE (생일)
- birthday_lunar_yn  CHAR(1) DEFAULT 'N' (음력 여부)
- deleted_yn         CHAR(1) DEFAULT 'N'
- reg_dt, reg_id, mod_dt, mod_id

인덱스:
- idx01: (user_no, deleted_yn)
- idx02: (user_no, relationship_type_cd, deleted_yn)
- uk01: (user_no, relationship_nm, phone_no) -- 중복 방지 기준
```

#### 3.3.2 tb_event (행사)

```
스키마: scm_mmhr

컬럼:
- event_no           BIGINT PK (시퀀스)
- user_no            BIGINT FK → tb_user (등록자)
- event_type_cd      VARCHAR(20) NOT NULL (코드: EVENT_TYPE)
- event_owner_cd     VARCHAR(10) NOT NULL (코드: EVENT_OWNER_TYPE - MINE/OTHERS)
- event_nm           VARCHAR(100) NOT NULL (행사명)
- event_dt           DATE NOT NULL (행사일)
- relationship_no    BIGINT FK → tb_relationship (타인 행사일 때 해당 관계, nullable)
- memo               TEXT
- deleted_yn         CHAR(1) DEFAULT 'N'
- reg_dt, reg_id, mod_dt, mod_id

인덱스:
- idx01: (user_no, deleted_yn, event_dt)
- idx02: (user_no, event_owner_cd, deleted_yn)
- idx03: (relationship_no)

비즈니스 규칙:
- event_owner_cd = 'OTHERS'이면 relationship_no 필수
- event_owner_cd = 'MINE'이면 relationship_no NULL
```

#### 3.3.3 tb_record (기록)

```
스키마: scm_mmhr

컬럼:
- record_no          BIGINT PK (시퀀스)
- user_no            BIGINT FK → tb_user (등록자)
- event_no           BIGINT FK → tb_event (행사 연결, nullable)
- relationship_no    BIGINT FK → tb_relationship (관계 대상)
- record_type_cd     VARCHAR(20) NOT NULL (코드: EVENT_TYPE과 동일 체계)
- record_direction_cd VARCHAR(10) NOT NULL (코드: RECORD_DIRECTION - SENT/RECEIVED)
- amount             BIGINT DEFAULT 0 (금액, 원 단위)
- record_dt          DATE NOT NULL (기록일)
- memo               TEXT
- deleted_yn         CHAR(1) DEFAULT 'N'
- reg_dt, reg_id, mod_dt, mod_id

인덱스:
- idx01: (user_no, deleted_yn, record_dt)
- idx02: (user_no, relationship_no, deleted_yn)
- idx03: (user_no, record_direction_cd, deleted_yn, record_dt)
- idx04: (user_no, record_type_cd, deleted_yn)
- idx05: (event_no)

비즈니스 규칙:
- event_no가 있으면 event의 event_type_cd와 record_type_cd 일치 검증
- SENT: 내가 타인 행사에 보낸 금액
- RECEIVED: 타인이 내 행사에 보낸 금액
- amount는 유형에 따라 선택적 (기념일 메모 등은 0 허용)
```

#### 3.3.4 tb_record_attachment (기록 첨부파일)

```
스키마: scm_mmhr

컬럼:
- attachment_no      BIGINT PK (시퀀스)
- record_no          BIGINT FK → tb_record
- file_nm            VARCHAR(255) NOT NULL
- file_path          VARCHAR(500) NOT NULL
- file_size          BIGINT DEFAULT 0
- file_type          VARCHAR(50)
- deleted_yn         CHAR(1) DEFAULT 'N'
- reg_dt, reg_id

인덱스:
- idx01: (record_no, deleted_yn)
```

#### 3.3.5 tb_schedule (일정)

```
스키마: scm_mmhr

컬럼:
- schedule_no        BIGINT PK (시퀀스)
- user_no            BIGINT FK → tb_user
- relationship_no    BIGINT FK → tb_relationship (nullable)
- schedule_type_cd   VARCHAR(20) NOT NULL (코드: SCHEDULE_TYPE)
- title              VARCHAR(200) NOT NULL
- schedule_dt        DATE NOT NULL
- lunar_yn           CHAR(1) DEFAULT 'N' (음력 여부)
- repeat_cd          VARCHAR(20) DEFAULT 'NONE' (코드: SCHEDULE_REPEAT)
- alarm_yn           CHAR(1) DEFAULT 'Y'
- alarm_days_before  INTEGER DEFAULT 3 (D-day 기준 사전 알림 일수)
- memo               TEXT
- completed_yn       CHAR(1) DEFAULT 'N' (확인 완료 여부)
- deleted_yn         CHAR(1) DEFAULT 'N'
- reg_dt, reg_id, mod_dt, mod_id

인덱스:
- idx01: (user_no, deleted_yn, schedule_dt)
- idx02: (user_no, schedule_type_cd, deleted_yn)
- idx03: (relationship_no)
```

#### 3.3.6 tb_notification (알림)

```
스키마: scm_mmpublic

컬럼:
- notification_no    BIGINT PK (시퀀스)
- user_no            BIGINT FK → tb_user
- notification_type_cd VARCHAR(20) NOT NULL (코드: NOTIFICATION_TYPE)
- title              VARCHAR(200) NOT NULL
- content            TEXT
- target_type_cd     VARCHAR(20) (코드: NOTIFICATION_TARGET - RECORD/SCHEDULE/...)
- target_no          BIGINT (대상 엔티티 PK)
- read_yn            CHAR(1) DEFAULT 'N'
- read_dt            TIMESTAMP
- reg_dt             TIMESTAMP NOT NULL
- reg_id             VARCHAR(50)

인덱스:
- idx01: (user_no, read_yn, reg_dt DESC)
- idx02: (user_no, notification_type_cd, read_yn)
```

#### 3.3.7 tb_dashboard_insight_snapshot (인사이트 스냅샷)

```
스키마: scm_mmpublic

컬럼:
- snapshot_no        BIGINT PK (시퀀스)
- user_no            BIGINT FK → tb_user
- insight_type_cd    VARCHAR(20) NOT NULL (인사이트 유형)
- insight_text       TEXT NOT NULL (인사이트 문구)
- reference_data     JSONB (근거 데이터)
- valid_from         DATE NOT NULL
- valid_to           DATE NOT NULL
- reg_dt, reg_id, mod_dt, mod_id

인덱스:
- idx01: (user_no, valid_from, valid_to)
```

#### 3.3.8 tb_user_profile (사용자 프로필 확장)

```
스키마: scm_mmhr

컬럼:
- profile_no         BIGINT PK (시퀀스)
- user_no            BIGINT FK → tb_user (UNIQUE)
- nickname           VARCHAR(50)
- profile_image_path VARCHAR(500)
- timezone           VARCHAR(50) DEFAULT 'Asia/Seoul'
- reg_dt, reg_id, mod_dt, mod_id

인덱스:
- uk01: (user_no) UNIQUE
```

#### 3.3.9 tb_user_dashboard_setting (대시보드 설정)

```
스키마: scm_mmhr

컬럼:
- setting_no              BIGINT PK (시퀀스)
- user_no                 BIGINT FK → tb_user (UNIQUE)
- dashboard_scope_cd      VARCHAR(10) DEFAULT 'SELF' (코드: DASHBOARD_SCOPE)
- recent_record_count     INTEGER DEFAULT 5 (최근 기록 표시 수)
- upcoming_schedule_days  INTEGER DEFAULT 30 (다가오는 일정 조회 범위)
- reg_dt, reg_id, mod_dt, mod_id

인덱스:
- uk01: (user_no) UNIQUE
```

### 3.4 테이블 관계도 (ERD 요약)

```
tb_user (1)
  ├── (1:N) tb_relationship
  │         ├── (1:N) tb_record
  │         ├── (1:N) tb_event
  │         └── (1:N) tb_schedule
  ├── (1:N) tb_event
  │         └── (1:N) tb_record
  │                   └── (1:N) tb_record_attachment
  ├── (1:N) tb_record
  ├── (1:N) tb_schedule
  ├── (1:N) tb_notification
  ├── (1:1) tb_user_profile
  ├── (1:1) tb_user_dashboard_setting
  └── (1:N) tb_dashboard_insight_snapshot
```

### 3.5 v2 대비 테이블 (현재 미생성, 구조만 정의)

| 테이블 | 용도 | 비고 |
|--------|------|------|
| tb_share_group | 공유 그룹 | 가족/배우자 그룹 |
| tb_share_member | 공유 멤버 | 그룹 내 멤버 매핑 |
| tb_relationship_group | 관계 그룹 | 관계 분류 (선택 기능) |
| tb_relationship_tag | 관계 태그 | 관계에 붙이는 자유 태그 |
| tb_record_tag | 기록 태그 | 기록에 붙이는 자유 태그 |

---

## 4. 비즈니스별 요구사항 검토

### 4.1 비즈니스1: 홈 대시보드 집계

#### API 1: 메인 대시보드 요약 조회

| 항목 | 검토 결과 |
|------|-----------|
| 테이블 | tb_event_type_code, tb_relationship_type_code → **tb_code로 대체 확정** (C-03 반영) |
| 수행 기능 | 적절함 |
| 성능 | Redis 캐시 권장 (TTL 5분 또는 이벤트 기반 무효화) |
| 조회 범위 | 초기 SELF 고정, 향후 SHARED 확장 가능 |

**보완 사항:**

- [B-01] 보낸 총액 / 받은 총액 집계 추가 필요 (MEMON 핵심 가치)
- [B-02] 환영 메시지용 데이터는 tb_user_profile에서 nickname 우선 조회, 없으면 user_nm
- [B-03] 최근 기록은 5건 제한 (tb_user_dashboard_setting.recent_record_count)
- [B-04] 다가오는 일정은 30일 이내 제한 (tb_user_dashboard_setting.upcoming_schedule_days)

**응답 DTO 구조 (권고):**

```
DashboardSummaryDto {
    userName: String          // 표시명
    totalRelationshipCount: Integer
    totalRecordCount: Integer
    thisMonthRecordCount: Integer
    upcomingScheduleCount: Integer
    totalSentAmount: Long     // [추가] 보낸 총액
    totalReceivedAmount: Long // [추가] 받은 총액
    recentRecords: List<RecentRecordDto>  // 최대 5건
    upcomingSchedules: List<UpcomingScheduleDto>  // 최대 5건
}
```

#### API 2: 홈 대시보드 상세 섹션 일괄 조회

| 항목 | 검토 결과 |
|------|-----------|
| 테이블 | tb_event 활용 → **tb_event CRUD 신설 확정** (C-02 반영) |
| 수행 기능 | 적절하나 섹션 단위 null-safe 설계 필수 |
| 기간 | 기본 6개월, 최대 12개월 |

**보완 사항:**

- [B-05] 월별 기록 추이에 **방향별(SENT/RECEIVED) 분리 집계** 추가
- [B-06] 각 섹션은 독립적으로 실패 가능하므로 섹션별 status 필드 권고
- [B-07] 인사이트 문구는 tb_dashboard_insight_snapshot에서 조회, 없으면 fallback 문구 반환

**응답 DTO 구조 (권고):**

```
DashboardDetailDto {
    monthlyTrend: MonthlyTrendSection      // 월별 기록 추이
    recordTypeDistribution: DistributionSection  // 기록 유형 분포
    relationshipTypeDistribution: DistributionSection  // 관계 유형 분포
    topRelationships: TopRankSection        // 자주 기록한 관계
    topEventTypes: TopRankSection           // 자주 기록한 유형
    recentRecords: List<RecordSummaryDto>   // 최근 기록
    upcomingSchedules: List<ScheduleSummaryDto>  // 다가오는 일정
    insight: InsightDto                     // 인사이트 문구
}
```

---

### 4.2 비즈니스2: 관계 관리

#### API 1: 관계 등록

| 항목 | 검토 결과 |
|------|-----------|
| 중복 기준 | **(user_no + relationship_nm + phone_no)** 조합 권고 |
| 삭제 정책 | soft delete (deleted_yn) 확정 |
| 테이블 | tb_relationship_group, tb_relationship_tag → **v2 이관** |

**보완 사항:**

- [B-08] phone_no가 NULL인 경우 이름만으로 중복 검증 → 동명이인 허용 여부 정책 필요
- [B-09] 관계 등록 시 생일 입력 → 자동 일정 생성 여부 정책 필요 (v2 권장)
- [B-10] importance_level 기본값 0, 범위 0~5 제한

#### API 2: 관계 목록 조회

| 항목 | 검토 결과 |
|------|-----------|
| 인덱스 | (user_no, relationship_type_cd, deleted_yn) 인덱스 필수 |
| 정렬 | 기본 최근 기록일 DESC, 이름순 ASC 선택 가능 |

**보완 사항:**

- [B-11] 각 관계의 최근 기록일/금액 요약을 조인으로 가져올 때 성능 주의
- [B-12] 홈용 "최근 사용 관계"는 별도 쿼리 또는 record 기반 정렬로 제공

#### API 3: 관계 상세 조회

| 항목 | 검토 결과 |
|------|-----------|
| 보안 | **user_no 소유권 검증 필수** (타 사용자 접근 차단) |

**보완 사항:**

- [B-13] 상세 조회 시 해당 관계의 **보낸/받은 총액 집계** 포함 권고
- [B-14] 최근 기록은 최대 10건으로 제한

#### API 4: 관계 수정/삭제

| 항목 | 검토 결과 |
|------|-----------|
| 삭제 정책 | soft delete 확정 |

**보완 사항:**

- [B-15] 관계 삭제 시 연결된 기록/일정 처리 정책:
  - 기록: 관계 soft delete 시 기록은 유지 (관계 복원 가능성)
  - 일정: 관계 soft delete 시 일정은 비활성화
  - 행사: 관계 soft delete 시 행사는 유지

---

### 4.3 비즈니스3: 기록 관리 + 행사 관리

> **확정**: 비즈니스3에 행사(Event) CRUD를 추가한다. (C-02 반영)

#### [신설] API 0: 행사 등록

```
필수값: userNo, eventTypeCd, eventOwnerCd, eventNm, eventDt
조건부 필수: relationshipNo (eventOwnerCd = 'OTHERS'일 때)
```

**흐름 1 (나 → 타인 행사):**
1. 행사 등록 (owner=OTHERS, relationship=김철수)
2. 기록 등록 (direction=SENT, event 연결)

**흐름 2 (타인 → 나의 행사):**
1. 행사 등록 (owner=MINE, eventNm="내 결혼식")
2. 기록 N건 등록 (각각 direction=RECEIVED, event 연결)

#### API 1: 기록 등록

| 항목 | 검토 결과 |
|------|-----------|
| 필수값 | **record_direction_cd 추가 확정** (C-01 반영) |
| event_no | nullable (행사 없이 단독 기록 가능) |
| amount | 유형별 필수/선택 정책 필요 |

**보완 사항:**

- [B-16] amount 정책:
  - 결혼/장례/돌잔치 → amount 필수 (0 이상)
  - 생일/기념일/기타 → amount 선택 (0 허용)
- [B-17] event_no가 있을 때 event의 event_type_cd와 record_type_cd 일치 검증
- [B-18] record_direction_cd는 event_owner_cd와 논리적 일관성 검증:
  - event_owner_cd = 'MINE' → record_direction_cd = 'RECEIVED'
  - event_owner_cd = 'OTHERS' → record_direction_cd = 'SENT'

#### API 2: 기록 목록 조회

| 항목 | 검토 결과 |
|------|-----------|
| 분리 | 홈용 / 전체용 응답 구조 분리 적절 |

**보완 사항:**

- [B-19] 홈용 최근 기록: RecentRecordDto (간략)
- [B-20] 전체 내역: RecordListDto (상세) + 페이징
- [B-21] 필터 조건에 **record_direction_cd (SENT/RECEIVED)** 추가 필수
- [B-22] 금액 마스킹 불필요 (본인 데이터만 조회하므로)

#### API 3: 기록 상세 조회

| 항목 | 검토 결과 |
|------|-----------|
| 첨부파일 | 접근 권한 검증은 record 소유권으로 충분 |

#### API 4: 기록 수정/삭제

| 항목 | 검토 결과 |
|------|-----------|
| 후처리 | 통계/인사이트 재계산 필요 |

**보완 사항:**

- [B-23] 기록 CUD 후 대시보드 캐시 무효화 이벤트 발행 권고
- [B-24] soft delete (deleted_yn) 적용

---

### 4.4 비즈니스4: 일정 관리

#### API 1: 일정 등록

| 항목 | 검토 결과 |
|------|-----------|
| 음력 | **v1에서 정책 결정 필수** |
| 반복 | NONE / YEARLY 2가지만 v1 지원 |

**보완 사항:**

- [B-25] 음력 지원 방안:
  - **권고안**: lunar_yn 필드로 음력 여부 저장, 양력 변환은 서버에서 연산
  - 음력→양력 변환 라이브러리 필요 (Korean Lunar Calendar)
  - 매년 양력 날짜가 달라지므로 반복 일정은 연초 배치에서 당해 양력 날짜로 전개
- [B-26] 관계 등록 시 생일 자동 일정 생성은 v2로 이관

#### API 2: 다가오는 일정 조회

| 항목 | 검토 결과 |
|------|-----------|
| D-day | 서버 기준 today ~ today+N일 범위 조회 |

**보완 사항:**

- [B-27] 지난 일정 중 미완료(completed_yn='N') 항목도 함께 반환 권고
  - 별도 섹션 "놓친 일정"으로 분리
  - 최대 5건 제한
- [B-28] D-day 계산은 한국 시간(Asia/Seoul) 기준

#### API 3: 일정 캘린더 조회

| 항목 | 검토 결과 |
|------|-----------|
| 반복 전개 | **계산형 전개 권고** (배치 불필요) |

**보완 사항:**

- [B-29] YEARLY 반복 일정은 조회 시 동적으로 해당 연도에 매핑하여 반환
  - DB에는 원본 1건만 저장
  - 조회 응답에서 해당 연도 날짜로 계산하여 포함
- [B-30] 음력 반복 일정은 양력 변환 후 계산형 전개

#### API 4: 일정 수정/삭제

| 항목 | 검토 결과 |
|------|-----------|
| 반복 수정 | v1은 **원본 전체 수정만 지원** |

**보완 사항:**

- [B-31] "이번만 수정" 기능은 v2로 이관 (별도 예외 테이블 필요)
- [B-32] soft delete 적용

---

### 4.5 비즈니스5: 홈 통계/분석

#### API 1: 월별 기록 추이 조회

| 항목 | 검토 결과 |
|------|-----------|
| 기능 | 적절함 |

**보완 사항:**

- [B-33] **방향별(SENT/RECEIVED) 분리 집계** 추가 필수 (금액 기준)
- [B-34] 기록이 없는 달은 0건/0원으로 응답 (프론트 차트 안정성)
- [B-35] 월 버킷은 Asia/Seoul 기준 (UTC 변환 주의)

**응답 예시:**

```json
{
  "months": [
    {
      "yearMonth": "2026-01",
      "totalCount": 5,
      "sentCount": 3,
      "receivedCount": 2,
      "sentAmount": 150000,
      "receivedAmount": 200000
    }
  ]
}
```

#### API 2: 기록 유형 분포 조회

| 항목 | 검토 결과 |
|------|-----------|
| 기능 | 적절함 |

**보완 사항:**

- [B-36] 비율 + 건수 + 금액 동시 제공
- [B-37] 코드 테이블 기반 동작 → 프론트 하드코딩 금지

#### API 3: 관계 유형 분포 조회

| 항목 | 검토 결과 |
|------|-----------|
| 기능 | 적절함 |

**보완 사항:**

- [B-38] deleted_yn = 'N' 조건 필수
- [B-39] 코드 미분류(NULL) 데이터는 'OTHER'로 처리

#### API 4: 자주 기록한 관계/이벤트 조회

| 항목 | 검토 결과 |
|------|-----------|
| 기능 | 적절함 |

**보완 사항:**

- [B-40] 동률 처리: record_dt DESC (최근 기록 우선)
- [B-41] 상위 5건 제한
- [B-42] 최근성 가중치는 v2 검토 (v1은 단순 빈도 기준)

---

### 4.6 비즈니스6: 관계 흐름 인사이트

#### API 1: 홈 인사이트 조회

| 항목 | 검토 결과 |
|------|-----------|
| 생성 방식 | v1 규칙 기반(rule-based) 적절 |

**보완 사항:**

- [B-43] 인사이트 템플릿 예시:

| 조건 | 문구 |
|------|------|
| 이번 달 기록이 가장 많은 유형 | "이번 달에는 {유형} 관련 기록이 가장 많았어요" |
| 가장 자주 관리하는 관계 유형 | "가장 자주 관리하는 관계는 {유형}이에요" |
| 최근 기록 트렌드 증가 | "최근 기록이 늘어나고 있어요" |
| 데이터 부족 | "최근 등록된 기록을 바탕으로 흐름을 준비 중이에요" |

- [B-44] 최소 기록 5건 이상일 때만 인사이트 생성, 미만이면 fallback 문구

#### API 2: 인사이트 생성 배치

| 항목 | 검토 결과 |
|------|-----------|
| 서버 | **System (worker-batch)** 수행 |
| 트리거 | 일 1회 + 기록 CUD 이벤트 |

**보완 사항:**

- [B-45] 배치 주기: 매일 03:00 KST
- [B-46] 기록 CUD 이벤트 발생 시 해당 사용자의 스냅샷 invalidate → 다음 조회 시 재생성
- [B-47] MQ 이벤트 구조: `exchange: memon.event / routing_key: record.changed.v1`

---

### 4.7 비즈니스7: 알림/확인 필요 항목

#### API 1: 홈 알림 배지 수 조회

| 항목 | 검토 결과 |
|------|-----------|
| 위치 | 헤더 공통 영역 → **별도 API 적절** |

**보완 사항:**

- [B-48] 읽지 않은 알림 수만 반환 (단일 숫자)
- [B-49] 성능: 단순 COUNT 쿼리 → Redis 캐시 가능

#### API 2: 알림 목록 조회

| 항목 | 검토 결과 |
|------|-----------|
| 기능 | 적절함 |

**보완 사항:**

- [B-50] target_type_cd + target_no로 클릭 시 해당 상세 화면 딥링크 가능
- [B-51] 페이징 필수 (cursor 기반 권고, 시간순 정렬)

#### API 3: 알림 읽음 처리

| 항목 | 검토 결과 |
|------|-----------|
| 기능 | 적절함 |

**보완 사항:**

- [B-52] 개별 읽음 + 전체 읽음 지원
- [B-53] 본인 알림만 처리 가능 (소유권 검증)

#### [추가 고려] 알림 생성 트리거

요구사항에 알림 **생성** 시점이 정의되지 않았다.

| 트리거 | 알림 내용 | 서버 |
|--------|-----------|------|
| 일정 D-day 도달 | "내일은 {관계명}의 {일정명}이에요" | System (worker-batch) |
| 일정 생성/수정 | (알림 불필요, 본인 행위) | - |
| 놓친 일정 | "어제 {관계명}의 {일정명}이 있었어요" | System (worker-batch) |

---

### 4.8 비즈니스8: 빠른 실행 지원

#### API 1: 빠른 실행용 기본 데이터 조회

| 항목 | 검토 결과 |
|------|-----------|
| 기능 | 적절함 |

**보완 사항:**

- [B-54] 코드성 데이터: System (api) 또는 Redis 캐시에서 조회
- [B-55] 최근 사용 관계: 최근 기록 기준 상위 5~10건
- [B-56] 자주 쓰는 유형: 기록 빈도 기준 상위 5건

---

### 4.9 비즈니스9: 사용자 컨텍스트/프로필

#### API 1: 홈 헤더 사용자 정보 조회

| 항목 | 검토 결과 |
|------|-----------|
| 테이블 | tb_user + tb_user_profile 조인 |

**보완 사항:**

- [B-57] 헤더 공통 API로 분리 → 재사용 가능
- [B-58] tb_user_profile이 없는 경우 tb_user.user_nm을 fallback으로 사용

#### API 2: 로그아웃

| 항목 | 검토 결과 |
|------|-----------|
| 위치 | **Gateway 책임** (인증 관련) |

**보완 사항:**

- [B-59] 로그아웃은 Gateway에서 처리 (Application 구현 금지)
- [B-60] Gateway → refresh token 무효화
- [B-61] 멀티 디바이스 정책: v1은 현재 세션만 로그아웃, v2에서 전체 로그아웃 지원

---

### 4.10 비즈니스10: 코드/정책 관리

#### API 1: 공통 코드 조회

| 항목 | 검토 결과 |
|------|-----------|
| 테이블 | tb_code_mngmn + tb_code (기존 체계 활용) |
| 서버 | **System (api profile)** 에서 제공 |

**보완 사항:**

- [B-62] 요구사항의 "tb_code_detail"은 기존 "tb_code"와 동일
- [B-63] 표시 순서(sort_order), 사용 여부(use_yn) 필터 필수
- [B-64] 다국어 확장: v2에서 tb_code에 locale 컬럼 추가 검토

---

### 4.11 비즈니스11: 가족/공유 확장 대비

#### API 1: 대시보드 조회 범위 설정

| 항목 | 검토 결과 |
|------|-----------|
| 우선순위 | v2 이관 |
| 현재 | dashboard_scope_cd = 'SELF' 고정 |

**보완 사항:**

- [B-65] tb_user_dashboard_setting 테이블은 v1에서 생성 (확장 대비)
- [B-66] 공유 관련 테이블(tb_share_group, tb_share_member)은 v2에서 생성

---

## 5. API 설계 방향

### 5.1 서버 역할 매핑

| 비즈니스 | API | 담당 서버 | Profile |
|----------|-----|-----------|---------|
| B1 대시보드 요약 | API 1 | Application | - |
| B1 대시보드 상세 | API 2 | Application | - |
| B2 관계 CRUD | API 1~4 | Application | - |
| B3 행사 CRUD | API 0 (신설) | Application | - |
| B3 기록 CRUD | API 1~4 | Application | - |
| B4 일정 CRUD | API 1~4 | Application | - |
| B5 통계 | API 1~4 | Application | - |
| B6 인사이트 조회 | API 1 | Application | - |
| B6 인사이트 배치 | API 2 | System | worker-batch |
| B7 알림 CRUD | API 1~3 | Application | - |
| B7 알림 생성 배치 | (내부) | System | worker-batch |
| B8 빠른 실행 데이터 | API 1 | Application | - |
| B9 프로필 조회 | API 1 | Application | - |
| B9 로그아웃 | API 2 | **Gateway** | - |
| B10 공통코드 | API 1 | **System** | api |
| B11 공유 설정 | API 1 | Application (v2) | - |

### 5.2 URL 설계

> 규칙: POST only, camelCase, /api/{version}/ prefix

#### 대시보드 (B1)

| URL | 설명 |
|-----|------|
| POST /api/{version}/dashboard/getSummary | 메인 요약 조회 |
| POST /api/{version}/dashboard/getDetailSections | 상세 섹션 일괄 조회 |

#### 관계 (B2)

| URL | 설명 |
|-----|------|
| POST /api/{version}/relationship/addRelationship | 관계 등록 |
| POST /api/{version}/relationship/getRelationshipList | 관계 목록 조회 |
| POST /api/{version}/relationship/getRelationshipDetail | 관계 상세 조회 |
| POST /api/{version}/relationship/updateRelationship | 관계 수정 |
| POST /api/{version}/relationship/deleteRelationship | 관계 삭제 |

#### 행사 (B3-A 신설)

| URL | 설명 |
|-----|------|
| POST /api/{version}/event/addEvent | 행사 등록 |
| POST /api/{version}/event/getEventList | 행사 목록 조회 |
| POST /api/{version}/event/getEventDetail | 행사 상세 조회 |
| POST /api/{version}/event/updateEvent | 행사 수정 |
| POST /api/{version}/event/deleteEvent | 행사 삭제 |

#### 기록 (B3)

| URL | 설명 |
|-----|------|
| POST /api/{version}/record/addRecord | 기록 등록 |
| POST /api/{version}/record/getRecordList | 기록 목록 조회 |
| POST /api/{version}/record/getRecordDetail | 기록 상세 조회 |
| POST /api/{version}/record/updateRecord | 기록 수정 |
| POST /api/{version}/record/deleteRecord | 기록 삭제 |

#### 일정 (B4)

| URL | 설명 |
|-----|------|
| POST /api/{version}/schedule/addSchedule | 일정 등록 |
| POST /api/{version}/schedule/getUpcomingList | 다가오는 일정 조회 |
| POST /api/{version}/schedule/getCalendarList | 캘린더 조회 |
| POST /api/{version}/schedule/updateSchedule | 일정 수정 |
| POST /api/{version}/schedule/deleteSchedule | 일정 삭제 |

#### 통계 (B5)

| URL | 설명 |
|-----|------|
| POST /api/{version}/stat/getMonthlyTrend | 월별 기록 추이 |
| POST /api/{version}/stat/getRecordTypeDistribution | 기록 유형 분포 |
| POST /api/{version}/stat/getRelationshipTypeDistribution | 관계 유형 분포 |
| POST /api/{version}/stat/getTopFrequent | 자주 기록한 관계/이벤트 |

#### 인사이트 (B6)

| URL | 설명 |
|-----|------|
| POST /api/{version}/insight/getInsight | 홈 인사이트 조회 |

#### 알림 (B7)

| URL | 설명 |
|-----|------|
| POST /api/{version}/notification/getBadgeCount | 알림 배지 수 |
| POST /api/{version}/notification/getNotificationList | 알림 목록 |
| POST /api/{version}/notification/markAsRead | 알림 읽음 처리 |

#### 빠른 실행 (B8)

| URL | 설명 |
|-----|------|
| POST /api/{version}/quickAction/getBaseData | 빠른 실행 기본 데이터 |

#### 프로필 (B9)

| URL | 설명 |
|-----|------|
| POST /api/{version}/profile/getUserInfo | 헤더 사용자 정보 |

### 5.3 Application 디렉토리 구조 (신규 도메인)

```
biz/
├── vo/
│   ├── mmhr/
│   │   ├── RelationshipVO
│   │   ├── EventVO
│   │   ├── RecordVO
│   │   ├── RecordAttachmentVO
│   │   ├── ScheduleVO
│   │   ├── UserProfileVO
│   │   └── UserDashboardSettingVO
│   ├── mmauth/
│   │   └── (기존)
│   └── mmpublic/
│       ├── NotificationVO
│       └── DashboardInsightSnapshotVO
│
├── auth/           (기존)
├── relationship/   (신규)
│   ├── controller/
│   ├── service/impl/
│   ├── dao/
│   └── dto/v1/
├── event/          (신규)
│   ├── controller/
│   ├── service/impl/
│   ├── dao/
│   └── dto/v1/
├── record/         (신규)
│   ├── controller/
│   ├── service/impl/
│   ├── dao/
│   └── dto/v1/
├── schedule/       (신규)
│   ├── controller/
│   ├── service/impl/
│   ├── dao/
│   └── dto/v1/
├── dashboard/      (신규)
│   ├── controller/
│   ├── service/impl/
│   ├── dao/
│   └── dto/v1/
├── stat/           (신규)
│   ├── controller/
│   ├── service/impl/
│   ├── dao/
│   └── dto/v1/
├── insight/        (신규)
│   ├── controller/
│   ├── service/impl/
│   ├── dao/
│   └── dto/v1/
├── notification/   (신규)
│   ├── controller/
│   ├── service/impl/
│   ├── dao/
│   └── dto/v1/
└── profile/        (신규)
    ├── controller/
    ├── service/impl/
    ├── dao/
    └── dto/v1/
```

---

## 6. 에러코드 확장 계획

Application 에러코드 형식: `3{비즈1}{비즈2}{시퀀스2}`

| 범위 | 도메인 | 예시 |
|------|--------|------|
| 31XXX | 계정/인증 (기존) | 31001 |
| 32XXX | 보안 (기존) | 32001 |
| 33XXX | 데이터 (기존) | 33001 |
| 34XXX | 관계 관리 (신규) | 34001 관계 미존재, 34002 중복 관계 |
| 35XXX | 기록 관리 (신규) | 35001 기록 미존재, 35002 금액 필수 |
| 36XXX | 일정 관리 (신규) | 36001 일정 미존재 |
| 37XXX | 대시보드/통계 (신규) | 37001 조회 기간 초과 |
| 38XXX | 알림 (신규) | 38001 알림 미존재 |
| 39XXX | 행사 관리 (신규) | 39001 행사 미존재, 39002 관계 필수 |

---

## 7. 기술적 고려사항

### 7.1 캐시 전략

| 대상 | 캐시 위치 | TTL | 무효화 조건 |
|------|-----------|-----|-------------|
| 대시보드 요약 | Redis | 5분 | 기록/관계/일정 CUD |
| 공통 코드 | Redis | 24시간 | 코드 변경 시 |
| 알림 배지 수 | Redis | 1분 | 알림 생성/읽음 시 |
| 인사이트 | DB (snapshot) | 1일 | 배치 갱신 |

### 7.2 성능 최적화

| 항목 | 전략 |
|------|------|
| 대시보드 초기 로딩 | API 1(요약)은 200ms 이내, API 2(상세)는 비동기 로딩 |
| 통계 집계 | 인덱스 기반 실시간 계산 (v1), 사전 집계 테이블 (v2) |
| 기록 목록 | offset 기반 페이징 (v1), cursor 기반 (v2 모바일) |
| 월별 추이 | record_dt 인덱스 활용, 빈 달 채우기는 Application에서 처리 |

### 7.3 보안

| 항목 | 정책 |
|------|------|
| 데이터 접근 | 모든 API에서 user_no 소유권 검증 필수 |
| 타인 데이터 | 소유권 불일치 시 BizException (에러코드로 처리) |
| 첨부파일 | record 소유권 검증 → 파일 접근 허용 |
| 금액 정보 | 본인 데이터이므로 마스킹 불필요 |

### 7.4 타임존

| 항목 | 정책 |
|------|------|
| 서버 기본 | UTC |
| 월/일 집계 | Asia/Seoul 기준 |
| 사용자 설정 | tb_user_profile.timezone (기본 Asia/Seoul) |
| DB 저장 | DATE: timezone 무관, TIMESTAMP: UTC 저장 |

### 7.5 MQ 이벤트 (System 연동)

| 이벤트 | Exchange | Routing Key | Consumer |
|--------|----------|-------------|----------|
| 기록 변경 | memon.event | record.changed.v1 | worker-batch (인사이트 재계산) |
| 일정 알림 | memon.event | schedule.alarm.v1 | worker-batch (알림 생성) |
| 대시보드 무효화 | memon.event | dashboard.invalidate.v1 | worker-batch (캐시 갱신) |

---

## 8. 개발 우선순위

### Phase 1: 핵심 도메인 (Core)

```
순서  작업                        의존성
─────────────────────────────────────────
1     DB 테이블 생성 (DDL)         없음
2     공통 코드 데이터 등록 (DML)   1
3     관계 CRUD (B2)              1, 2
4     행사 CRUD (B3-A 신설)       1, 2, 3
5     기록 CRUD (B3)              1, 2, 3, 4
6     일정 CRUD (B4)              1, 2, 3
```

### Phase 2: 대시보드/통계 (Dashboard)

```
순서  작업                        의존성
─────────────────────────────────────────
7     대시보드 요약 API (B1-1)     3, 5, 6
8     통계 API (B5)               5
9     대시보드 상세 API (B1-2)     7, 8
```

### Phase 3: 부가 기능 (Support)

```
순서  작업                        의존성
─────────────────────────────────────────
10    프로필 API (B9)             1
11    빠른 실행 API (B8)          3, 5
12    인사이트 조회 API (B6-1)    5
13    인사이트 배치 (B6-2)        12 (System)
```

### Phase 4: 알림 (Notification)

```
순서  작업                        의존성
─────────────────────────────────────────
14    알림 테이블/API (B7)         1
15    알림 생성 배치              6, 14 (System)
```

### Phase 5: 확장 (v2)

```
순서  작업                        의존성
─────────────────────────────────────────
16    가족/공유 (B11)             전체
17    관계 태그/그룹              3
18    기록 태그/첨부파일           5
19    음력 완전 지원              6
20    모바일 cursor 페이징        전체
```

---

## 9. 누락/보완 필요 사항 요약

### Critical (전체 확정 완료)

| ID | 항목 | 상태 |
|----|------|------|
| C-01 | 기록 방향(SENT/RECEIVED) 개념 추가 | ✅ **확정** |
| C-02 | 행사(Event) 엔티티 1급 승격 및 CRUD 신설 | ✅ **확정** |
| C-03 | 코드 테이블 이중 정의 → tb_code 통합 | ✅ **확정** |

### Important (v1 개발 중 결정)

| ID | 항목 | 상태 |
|----|------|------|
| B-08 | 관계 중복 검증 기준 (이름+전화번호 vs 이름만) | 정책 필요 |
| B-16 | 유형별 금액 필수/선택 정책 | 정책 필요 |
| B-25 | 음력 지원 여부 및 범위 | 정책 필요 |
| B-27 | 놓친 일정 표시 정책 | 정책 필요 |
| B-59 | 로그아웃 API → Gateway 책임 확인 | 구조 확인 |

### Nice to Have (v2 이관 권고)

| ID | 항목 | 상태 |
|----|------|------|
| B-09 | 관계 등록 시 자동 일정 생성 | v2 |
| B-26 | 음력 생일 자동 양력 변환 일정 | v2 |
| B-31 | 반복 일정 "이번만 수정" | v2 |
| B-42 | 통계 최근성 가중치 | v2 |
| B-61 | 멀티 디바이스 전체 로그아웃 | v2 |
| B-64 | 다국어 코드 | v2 |
| B-66 | 가족/공유 기능 | v2 |

---

## 10. 최종 소견

### 10.1 요구사항 완성도

전체 요구사항은 **홈 대시보드 중심의 사용자 경험**을 잘 설계하고 있으며,
API 단위의 접근 권한, 필수값, 조회 범위가 명확하게 정의되어 있다.

Critical 보완 3건이 모두 확정되어 MEMON의 핵심 비즈니스 가치인
**"주고받은 금전 흐름 추적"**을 구현할 수 있는 구조가 완성되었다.

- ✅ C-01: 기록 방향(SENT/RECEIVED) → tb_record.record_direction_cd 확정
- ✅ C-02: 행사(Event) 1급 엔티티 → tb_event CRUD 신설 확정
- ✅ C-03: 코드 테이블 통합 → tb_code 단일 체계 확정

### 10.2 아키텍처 적합성

요구사항은 기존 아키텍처(Gateway/Application/System)에 잘 부합한다.
- 비즈니스 로직 → Application
- 배치/인사이트 생성 → System (worker-batch)
- 공통 코드 → System (api)
- 인증/로그아웃 → Gateway

서버 역할 경계 위반 없이 구현 가능하다.

### 10.3 데이터 모델 적합성

기존 스키마 정책(scm_mmhr, scm_mmpublic, scm_mmlog)에 신규 테이블을 자연스럽게 배치할 수 있다.
코드 테이블은 기존 tb_code 체계를 활용하여 일관성을 유지해야 한다.

### 10.4 다음 단계

1. ~~**Critical 항목 3건** (C-01, C-02, C-03) 결정~~ → ✅ 전체 확정 완료
2. Phase 1 DDL 작성 (테이블 생성) ← **현재 착수 가능**
3. Phase 1 DML 작성 (공통 코드 데이터)
4. Phase 1 API 개발 착수

---

END OF FILE
