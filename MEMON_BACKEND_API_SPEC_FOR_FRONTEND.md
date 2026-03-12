# MEMON Backend API 명세서 (Frontend 전달용)

> **작성일:** 2026-03-13
> **대상:** Frontend Engineering Lead
> **API Server:** Application Server (비즈니스 로직 전용)
> **현재 API 버전:** v1

---

## 1. 서비스 개요

MEMON은 **경조사/인간관계 기록 관리 서비스**다.
사용자의 인간관계를 등록하고, 경조사 기록(축의금, 조의금 등)을 관리하며, 일정 알림과 통계 분석을 제공한다.

### 1.1 핵심 도메인 관계도

```
[사용자(User)]
    ├── [관계(Relationship)] ─── 사람 간의 관계 등록/관리
    │       ├── [기록(Record)] ─── 경조사 기록 (축의금, 조의금, 선물 등)
    │       └── [일정(Schedule)] ─── 관계에 연결된 일정 관리
    ├── [일정(Schedule)] ─── 관계 없이 독립적인 일정도 가능
    ├── [알림(Notification)] ─── 시스템이 발행하는 알림
    ├── [통계(Statistics)] ─── 기록 데이터 집계/분석
    ├── [인사이트(Insight)] ─── 규칙 기반 맞춤 인사이트
    └── [빠른 실행(QuickAction)] ─── 빠른 등록을 위한 기초 데이터
```

### 1.2 데이터 흐름 요약

- **관계(Relationship)** 가 모든 것의 기준점. 기록과 일정은 관계에 연결된다.
- **기록(Record)** 은 반드시 하나의 관계에 소속된다. (relationshipNo 필수)
- **일정(Schedule)** 은 관계에 연결할 수도 있고, 독립적으로 생성할 수도 있다. (relationshipNo 선택)
- **알림(Notification)** 은 System Server에서 생성되며, Application Server에서는 조회/읽음처리만 한다.
- **통계/인사이트** 는 기록/관계 데이터를 기반으로 실시간 집계한다.

---

## 2. API 공통 규약

### 2.1 요청 규칙

| 항목 | 규칙 |
|------|------|
| HTTP Method | **모든 API는 POST** |
| Content-Type | `application/json` |
| URL 패턴 | `/api/{version}/{domain}/{action}` |
| URL 네이밍 | **camelCase** (하이픈, 언더스코어 사용 금지) |
| 현재 버전 | `v1` (예: `/api/v1/relationship/getList`) |
| 인증 | Gateway에서 처리. Application Server 도달 시 이미 인증 완료 상태 |
| 사용자 식별 | Gateway가 `X-User-No` 헤더로 전달. 프론트에서 별도 전송 불필요 |

### 2.2 응답 규칙

모든 API 응답은 동일한 wrapper 구조로 반환된다:

```json
{
  "code": "20000",
  "message": "성공",
  "data": { ... }
}
```

**에러 응답 예시:**
```json
{
  "code": "35001",
  "message": "관계 정보를 찾을 수 없습니다.",
  "data": null
}
```

### 2.3 공통 데이터 정책

| 정책 | 설명 |
|------|------|
| 삭제 방식 | **Soft Delete** — 모든 도메인은 `deleted_yn` 플래그 처리. 삭제된 데이터는 조회되지 않음 |
| 날짜 형식 | `yyyyMMdd` (예: `20260313`) |
| 일시 형식 | `yyyy-MM-dd HH:mm:ss` |
| 연월 형식 | `yyyyMM` (예: `202603`) |
| 금액 단위 | Long (원 단위) |
| 정렬 기본값 | API별 상이 (각 API 명세 참고) |
| 코드값 관리 | `tb_code` / `tb_code_mngmn` 테이블 기반 공통코드 시스템 |

---

## 3. 에러 코드 일람표

| 코드 | 상수명 | 설명 | 사용 도메인 |
|------|--------|------|------------|
| 20000 | SUCCESS | 성공 | 공통 |
| 35001 | RELATIONSHIP_NOT_FOUND | 관계 정보 없음 | relationship |
| 35002 | RELATIONSHIP_ACCESS_DENIED | 타인 소유 관계 접근 불가 | relationship, schedule, record |
| 36001 | RECORD_NOT_FOUND | 기록 정보 없음 | record |
| 36002 | RECORD_ACCESS_DENIED | 타인 소유 기록 접근 불가 | record |
| 36003 | RECORD_INVALID_RELATIONSHIP | 유효하지 않은 관계에 기록 연결 시도 | record |
| 37001 | SCHEDULE_NOT_FOUND | 일정 정보 없음 | schedule |
| 37002 | SCHEDULE_ACCESS_DENIED | 타인 소유 일정 접근 불가 | schedule |
| 38001 | NOTIFICATION_NOT_FOUND | 알림 정보 없음 | notification |

---

## 4. 전체 API 목록 (32개)

| # | 도메인 | 엔드포인트 | 기능 | 유형 |
|---|--------|-----------|------|------|
| 1 | auth | `/api/v1/auth/syncOAuthUser` | OAuth 사용자 동기화 | 등록 |
| 2 | auth | `/api/v1/auth/recordLoginHistory` | 로그인 이력 기록 | 등록 |
| 3 | dashboard | `/api/v1/dashboard/getHomeSummary` | 홈 대시보드 요약 | 조회 |
| 4 | user | `/api/v1/user/getMyInfo` | 내 정보 조회 | 조회 |
| 5 | relationship | `/api/v1/relationship/createRelationship` | 관계 등록 | 등록 |
| 6 | relationship | `/api/v1/relationship/getOptionList` | 관계 옵션 목록 (드롭다운용) | 조회 |
| 7 | relationship | `/api/v1/relationship/getList` | 관계 목록 조회 | 조회 |
| 8 | relationship | `/api/v1/relationship/getDetail` | 관계 상세 조회 | 조회 |
| 9 | relationship | `/api/v1/relationship/updateRelationship` | 관계 수정 | 수정 |
| 10 | relationship | `/api/v1/relationship/deleteRelationship` | 관계 삭제 | 삭제 |
| 11 | record | `/api/v1/record/createRecord` | 기록 등록 | 등록 |
| 12 | record | `/api/v1/record/getRecentList` | 최근 기록 목록 | 조회 |
| 13 | record | `/api/v1/record/getList` | 기록 목록 조회 | 조회 |
| 14 | record | `/api/v1/record/getDetail` | 기록 상세 조회 | 조회 |
| 15 | record | `/api/v1/record/updateRecord` | 기록 수정 | 수정 |
| 16 | record | `/api/v1/record/deleteRecord` | 기록 삭제 | 삭제 |
| 17 | schedule | `/api/v1/schedule/getUpcomingList` | 다가오는 일정 목록 | 조회 |
| 18 | schedule | `/api/v1/schedule/createSchedule` | 일정 등록 | 등록 |
| 19 | schedule | `/api/v1/schedule/getDetail` | 일정 상세 조회 | 조회 |
| 20 | schedule | `/api/v1/schedule/getCalendarList` | 캘린더 일정 조회 | 조회 |
| 21 | schedule | `/api/v1/schedule/updateSchedule` | 일정 수정 | 수정 |
| 22 | schedule | `/api/v1/schedule/deleteSchedule` | 일정 삭제 | 삭제 |
| 23 | notification | `/api/v1/notification/getUnreadCount` | 안읽은 알림 수 | 조회 |
| 24 | notification | `/api/v1/notification/getList` | 알림 목록 조회 | 조회 |
| 25 | notification | `/api/v1/notification/markAsRead` | 알림 읽음 처리 | 수정 |
| 26 | notification | `/api/v1/notification/markAllAsRead` | 알림 전체 읽음 처리 | 수정 |
| 27 | statistics | `/api/v1/statistics/getMonthlyTrend` | 월별 기록 추이 | 조회 |
| 28 | statistics | `/api/v1/statistics/getRecordTypeDistribution` | 기록 유형별 분포 | 조회 |
| 29 | statistics | `/api/v1/statistics/getRelationshipTypeDistribution` | 관계 유형별 분포 | 조회 |
| 30 | statistics | `/api/v1/statistics/getTopFrequent` | 자주 기록한 관계/유형 | 조회 |
| 31 | insight | `/api/v1/insight/getHomeInsight` | 홈 인사이트 | 조회 |
| 32 | quickAction | `/api/v1/quickAction/getBaseData` | 빠른 등록 기초 데이터 | 조회 |

---

## 5. 도메인별 API 상세 명세

---

### 5.1 인증 (Auth)

> Gateway 내부 호출 전용. 프론트에서 직접 호출하지 않음.

#### `POST /api/v1/auth/syncOAuthUser`

OAuth 로그인 성공 시 Gateway가 호출. 사용자 정보 동기화.

#### `POST /api/v1/auth/recordLoginHistory`

로그인 이력 기록. Gateway가 호출.

---

### 5.2 대시보드 (Dashboard)

#### `POST /api/v1/dashboard/getHomeSummary`

홈 화면에 표시할 요약 정보 일괄 조회.

**Request:** 없음 (body 불필요)

**Response:**
```json
{
  "code": "20000",
  "data": {
    "totalRelationshipCount": 25,
    "recentRecords": [
      {
        "recordNo": 1,
        "relationshipNm": "김철수",
        "recordTypeCd": "WEDDING",
        "amount": 50000,
        "recordDate": "20260310",
        "directionCd": "GIVEN"
      }
    ],
    "upcomingSchedules": [
      {
        "scheduleNo": 1,
        "title": "김철수 결혼식",
        "scheduleDate": "20260320",
        "dday": 7,
        "relationshipNm": "김철수"
      }
    ],
    "unreadNotificationCount": 3,
    "monthSummary": {
      "givenAmount": 150000,
      "receivedAmount": 100000,
      "givenCount": 3,
      "receivedCount": 2
    }
  }
}
```

---

### 5.3 사용자 (User)

#### `POST /api/v1/user/getMyInfo`

로그인 사용자 본인 정보 조회.

**Request:** 없음

**Response:**
```json
{
  "code": "20000",
  "data": {
    "userNo": 1,
    "userNm": "홍길동",
    "email": "user@example.com",
    "profileImageUrl": "https://...",
    "oauthProvider": "KAKAO"
  }
}
```

---

### 5.4 관계 관리 (Relationship)

#### `POST /api/v1/relationship/createRelationship`

새로운 인간관계 등록.

**Request:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| relationshipNm | String | Y | 관계 대상 이름 |
| relationshipTypeCd | String | Y | 관계유형 코드 (FAMILY, FRIEND, WORK, ETC) |
| phoneNo | String | N | 전화번호 |
| memo | String | N | 메모 |
| importance | Integer | N | 중요도 (1~5) |
| birthday | String | N | 생일 (yyyyMMdd) |
| anniversaryDt | String | N | 기념일 (yyyyMMdd) |

**Response:**
```json
{
  "code": "20000",
  "data": {
    "relationshipNo": 1
  }
}
```

---

#### `POST /api/v1/relationship/getOptionList`

관계 드롭다운/선택 목록 (기록/일정 등록 시 관계 선택에 사용).

**Request:** 없음

**Response:**
```json
{
  "code": "20000",
  "data": [
    {
      "relationshipNo": 1,
      "relationshipNm": "김철수",
      "relationshipTypeCd": "FRIEND"
    }
  ]
}
```

---

#### `POST /api/v1/relationship/getList`

관계 목록 조회 (검색/필터/정렬/페이징).

**Request:**
| 필드 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| searchKeyword | String | N | | 이름/메모 통합 검색 |
| relationshipTypeCd | String | N | | 관계유형 필터 (FAMILY, FRIEND, WORK, ETC) |
| sortBy | String | N | NAME | 정렬 기준: NAME, RECENT_RECORD, IMPORTANCE, REG_DT |
| sortOrder | String | N | ASC | ASC / DESC |
| limit | Integer | N | 20 | 조회 건수 (1~100) |
| offset | Integer | N | 0 | 시작 위치 (0~10000) |

**Response:**
```json
{
  "code": "20000",
  "data": [
    {
      "relationshipNo": 1,
      "relationshipNm": "김철수",
      "relationshipTypeCd": "FRIEND",
      "phoneNo": "010-1234-5678",
      "importance": 4,
      "birthday": "19900101",
      "lastRecordDate": "20260310",
      "upcomingScheduleYn": "Y",
      "totalRecordCount": 5
    }
  ]
}
```

---

#### `POST /api/v1/relationship/getDetail`

관계 상세 정보 + 최근 기록 5건 + 다가오는 일정 5건 조회.

**Request:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| relationshipNo | Long | Y | 관계 번호 |

**Response:**
```json
{
  "code": "20000",
  "data": {
    "relationshipNo": 1,
    "relationshipNm": "김철수",
    "relationshipTypeCd": "FRIEND",
    "phoneNo": "010-1234-5678",
    "memo": "대학 동기",
    "importance": 4,
    "birthday": "19900101",
    "anniversaryDt": null,
    "regDt": "2026-03-01 10:00:00",
    "totalRecordCount": 5,
    "totalGivenAmount": 200000,
    "totalReceivedAmount": 150000,
    "recentRecords": [
      {
        "recordNo": 10,
        "recordTypeCd": "WEDDING",
        "recordDate": "20260310",
        "amount": 50000,
        "directionCd": "GIVEN"
      }
    ],
    "upcomingSchedules": [
      {
        "scheduleNo": 5,
        "title": "김철수 생일",
        "scheduleDate": "20260401",
        "dday": 19
      }
    ]
  }
}
```

**에러:**
- `35001` — 관계 정보 없음
- `35002` — 타인 소유 관계 접근 불가

---

#### `POST /api/v1/relationship/updateRelationship`

관계 정보 수정. **null이 아닌 필드만 업데이트** (부분 수정 지원).

**Request:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| relationshipNo | Long | Y | 관계 번호 |
| relationshipNm | String | N | 변경할 이름 |
| relationshipTypeCd | String | N | 변경할 관계유형 |
| phoneNo | String | N | 변경할 전화번호 |
| memo | String | N | 변경할 메모 |
| importance | Integer | N | 변경할 중요도 |
| birthday | String | N | 변경할 생일 |
| anniversaryDt | String | N | 변경할 기념일 |

**Response:**
```json
{
  "code": "20000",
  "data": {
    "relationshipNo": 1
  }
}
```

**에러:** `35001`, `35002`

---

#### `POST /api/v1/relationship/deleteRelationship`

관계 삭제 (soft delete). 연결된 기록/일정은 삭제되지 않는다.

> **프론트 처리 필요:** 삭제된 관계에 연결된 기록/일정의 관계명을 "삭제된 관계"로 표시해야 한다.

**Request:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| relationshipNo | Long | Y | 관계 번호 |

**Response:**
```json
{
  "code": "20000",
  "data": {
    "relationshipNo": 1
  }
}
```

**에러:** `35002`

---

### 5.5 기록 관리 (Record)

#### `POST /api/v1/record/createRecord`

경조사 기록 등록.

**Request:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| relationshipNo | Long | Y | 관계 번호 |
| recordTypeCd | String | Y | 기록유형 코드 (WEDDING, FUNERAL, BIRTHDAY 등) |
| recordDate | String | Y | 기록일 (yyyyMMdd) |
| amount | Long | Y | 금액 (원 단위) |
| directionCd | String | N | 방향 (GIVEN: 보냄, RECEIVED: 받음). 기본 GIVEN |
| memo | String | N | 메모 |

**Response:**
```json
{
  "code": "20000",
  "data": {
    "recordNo": 1
  }
}
```

**에러:** `36003` — 유효하지 않은 관계

---

#### `POST /api/v1/record/getRecentList`

최근 기록 목록 (홈/대시보드 위젯용).

**Request:**
| 필드 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| limit | Integer | N | 10 | 조회 건수 (1~100) |
| offset | Integer | N | 0 | 시작 위치 (0~10000) |

---

#### `POST /api/v1/record/getList`

기록 목록 조회 (필터/검색/정렬/페이징).

**Request:**
| 필드 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| relationshipNo | Long | N | | 특정 관계 필터 |
| recordTypeCd | String | N | | 기록유형 필터 |
| directionCd | String | N | | 방향 필터 (GIVEN / RECEIVED) |
| startDate | String | N | | 조회 시작일 (yyyyMMdd) |
| endDate | String | N | | 조회 종료일 (yyyyMMdd) |
| searchKeyword | String | N | | 메모 검색어 |
| sortBy | String | N | RECORD_DATE | 정렬 기준: RECORD_DATE, AMOUNT, REG_DT |
| sortOrder | String | N | DESC | ASC / DESC |
| limit | Integer | N | 20 | 조회 건수 (1~100) |
| offset | Integer | N | 0 | 시작 위치 (0~10000) |

**Response:**
```json
{
  "code": "20000",
  "data": [
    {
      "recordNo": 1,
      "relationshipNo": 1,
      "relationshipNm": "김철수",
      "recordTypeCd": "WEDDING",
      "recordDate": "20260310",
      "amount": 50000,
      "directionCd": "GIVEN",
      "memo": "결혼식 축의금",
      "regDt": "2026-03-10 09:00:00"
    }
  ]
}
```

---

#### `POST /api/v1/record/getDetail`

기록 상세 조회.

**Request:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| recordNo | Long | Y | 기록 번호 |

**Response:**
```json
{
  "code": "20000",
  "data": {
    "recordNo": 1,
    "relationshipNo": 1,
    "relationshipNm": "김철수",
    "relationshipTypeCd": "FRIEND",
    "recordTypeCd": "WEDDING",
    "recordDate": "20260310",
    "amount": 50000,
    "directionCd": "GIVEN",
    "memo": "결혼식 축의금",
    "regDt": "2026-03-10 09:00:00",
    "modDt": "2026-03-10 09:00:00"
  }
}
```

**에러:** `36001`, `36002`

---

#### `POST /api/v1/record/updateRecord`

기록 수정. null이 아닌 필드만 업데이트.

**Request:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| recordNo | Long | Y | 기록 번호 |
| relationshipNo | Long | N | 변경할 관계 |
| recordTypeCd | String | N | 변경할 기록유형 |
| recordDate | String | N | 변경할 기록일 |
| amount | Long | N | 변경할 금액 |
| directionCd | String | N | 변경할 방향 |
| memo | String | N | 변경할 메모 |

**Response:**
```json
{
  "code": "20000",
  "data": {
    "recordNo": 1
  }
}
```

**에러:** `36002`, `36003`

---

#### `POST /api/v1/record/deleteRecord`

기록 삭제 (soft delete).

**Request:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| recordNo | Long | Y | 기록 번호 |

**Response:**
```json
{
  "code": "20000",
  "data": {
    "recordNo": 1
  }
}
```

**에러:** `36002`

---

### 5.6 일정 관리 (Schedule)

#### `POST /api/v1/schedule/createSchedule`

일정 등록. 관계에 연결하거나 독립적으로 생성 가능.

**Request:**
| 필드 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| relationshipNo | Long | N | | 관계 연결 (선택) |
| scheduleTypeCd | String | Y | | 일정유형 코드 |
| title | String | Y | | 일정 제목 |
| scheduleDate | String | Y | | 일정일 (yyyyMMdd) |
| memo | String | N | | 메모 |
| repeatYn | String | N | N | 반복 여부 (Y/N) |
| repeatCycleCd | String | N | | 반복 주기 (YEARLY, MONTHLY 등) |
| alarmYn | String | N | Y | 알림 여부 (Y/N) |

**Response:**
```json
{
  "code": "20000",
  "data": {
    "scheduleNo": 1
  }
}
```

**에러:** `35002` — 연결하려는 관계가 타인 소유

---

#### `POST /api/v1/schedule/getUpcomingList`

다가오는 일정 목록 (홈/대시보드 위젯용).

**Request:**
| 필드 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| days | Integer | N | 30 | 조회 기간 (일 단위, 1~365) |
| limit | Integer | N | 10 | 조회 건수 (1~100) |

---

#### `POST /api/v1/schedule/getDetail`

일정 상세 조회.

**Request:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| scheduleNo | Long | Y | 일정 번호 |

**Response:**
```json
{
  "code": "20000",
  "data": {
    "scheduleNo": 1,
    "relationshipNo": 1,
    "relationshipNm": "김철수",
    "scheduleTypeCd": "BIRTHDAY",
    "title": "김철수 생일",
    "scheduleDate": "20260401",
    "memo": "선물 준비하기",
    "repeatYn": "Y",
    "repeatCycleCd": "YEARLY",
    "alarmYn": "Y",
    "dday": 19,
    "regDt": "2026-03-01 10:00:00",
    "modDt": "2026-03-01 10:00:00"
  }
}
```

**에러:** `37001`, `37002`

---

#### `POST /api/v1/schedule/getCalendarList`

월별 캘린더 뷰용 일정 조회.

**Request:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| yearMonth | String | Y | 조회 연월 (yyyyMM, 예: 202603) |
| relationshipNo | Long | N | 특정 관계 필터 |

**Response:**
```json
{
  "code": "20000",
  "data": [
    {
      "scheduleNo": 1,
      "title": "김철수 생일",
      "scheduleDate": "20260401",
      "scheduleTypeCd": "BIRTHDAY",
      "relationshipNm": "김철수",
      "repeatYn": "Y",
      "alarmYn": "Y"
    }
  ]
}
```

---

#### `POST /api/v1/schedule/updateSchedule`

일정 수정. null이 아닌 필드만 업데이트.

**Request:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| scheduleNo | Long | Y | 일정 번호 |
| relationshipNo | Long | N | 변경할 관계 |
| scheduleTypeCd | String | N | 변경할 일정유형 |
| title | String | N | 변경할 제목 |
| scheduleDate | String | N | 변경할 일정일 |
| memo | String | N | 변경할 메모 |
| repeatYn | String | N | 변경할 반복여부 |
| repeatCycleCd | String | N | 변경할 반복주기 |
| alarmYn | String | N | 변경할 알림여부 |

**Response:**
```json
{
  "code": "20000",
  "data": {
    "scheduleNo": 1
  }
}
```

**에러:** `37002`, `35002`

---

#### `POST /api/v1/schedule/deleteSchedule`

일정 삭제 (soft delete).

**Request:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| scheduleNo | Long | Y | 일정 번호 |

**Response:**
```json
{
  "code": "20000",
  "data": {
    "scheduleNo": 1
  }
}
```

**에러:** `37002`

---

### 5.7 알림 (Notification)

> 알림은 System Server에서 생성된다. 프론트에서는 조회와 읽음 처리만 수행한다.

#### `POST /api/v1/notification/getUnreadCount`

안읽은 알림 수 조회 (헤더 뱃지용).

**Request:** 없음

**Response:**
```json
{
  "code": "20000",
  "data": {
    "unreadCount": 3
  }
}
```

---

#### `POST /api/v1/notification/getList`

알림 목록 조회 (최신순 고정 정렬).

**Request:**
| 필드 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| readYn | String | N | 전체 | 읽음 필터 (Y: 읽은 것만, N: 안읽은 것만, null: 전체) |
| limit | Integer | N | 20 | 조회 건수 (1~100) |
| offset | Integer | N | 0 | 시작 위치 (0~10000) |

**Response:**
```json
{
  "code": "20000",
  "data": [
    {
      "notificationNo": 1,
      "notificationTypeCd": "SCHEDULE_ALARM",
      "title": "일정 알림",
      "content": "내일 김철수 생일이에요!",
      "targetTypeCd": "SCHEDULE",
      "targetNo": 5,
      "readYn": "N",
      "readDt": null,
      "regDt": "2026-03-12 09:00:00"
    }
  ]
}
```

> **프론트 처리:** `targetTypeCd`와 `targetNo`를 조합하여 클릭 시 해당 상세 화면으로 이동 처리 가능 (예: SCHEDULE + scheduleNo → 일정 상세)

---

#### `POST /api/v1/notification/markAsRead`

개별 알림 읽음 처리. 이미 읽은 알림에 다시 호출해도 정상 응답 (멱등성 보장).

**Request:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| notificationNo | Long | Y | 알림 번호 |

**Response:**
```json
{
  "code": "20000",
  "data": {
    "notificationNo": 1
  }
}
```

**에러:** `38001`

---

#### `POST /api/v1/notification/markAllAsRead`

모든 안읽은 알림 일괄 읽음 처리.

**Request:** 없음

**Response:**
```json
{
  "code": "20000",
  "data": {
    "updatedCount": 5
  }
}
```

---

### 5.8 통계 (Statistics)

> 모든 통계 API는 조회 전용. 실시간 집계 기반.

#### `POST /api/v1/statistics/getMonthlyTrend`

월별 기록 추이 (차트용).

**Request:**
| 필드 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| months | Integer | N | 6 | 조회 개월 수 (1~12) |

**Response:**
```json
{
  "code": "20000",
  "data": [
    {
      "yearMonth": "202610",
      "totalCount": 5,
      "totalGivenAmount": 250000,
      "totalReceivedAmount": 100000
    },
    {
      "yearMonth": "202611",
      "totalCount": 3,
      "totalGivenAmount": 150000,
      "totalReceivedAmount": 200000
    }
  ]
}
```

---

#### `POST /api/v1/statistics/getRecordTypeDistribution`

기록 유형별 분포 (파이차트/도넛차트용).

**Request:**
| 필드 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| startDate | String | N | 올해 1월 1일 | 조회 시작일 (yyyyMMdd) |
| endDate | String | N | 오늘 | 조회 종료일 (yyyyMMdd) |

**Response:**
```json
{
  "code": "20000",
  "data": [
    {
      "recordTypeCd": "WEDDING",
      "count": 10,
      "totalAmount": 500000,
      "percentage": 45.5
    },
    {
      "recordTypeCd": "FUNERAL",
      "count": 5,
      "totalAmount": 250000,
      "percentage": 22.7
    }
  ]
}
```

---

#### `POST /api/v1/statistics/getRelationshipTypeDistribution`

관계 유형별 기록 분포 (파이차트/도넛차트용).

**Request:**
| 필드 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| startDate | String | N | 올해 1월 1일 | 조회 시작일 |
| endDate | String | N | 오늘 | 조회 종료일 |

**Response:**
```json
{
  "code": "20000",
  "data": [
    {
      "relationshipTypeCd": "FAMILY",
      "count": 15,
      "totalAmount": 750000,
      "percentage": 55.0
    },
    {
      "relationshipTypeCd": "FRIEND",
      "count": 8,
      "totalAmount": 400000,
      "percentage": 30.0
    }
  ]
}
```

---

#### `POST /api/v1/statistics/getTopFrequent`

자주 기록한 관계 또는 기록유형 랭킹.

**Request:**
| 필드 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| targetType | String | Y | | RELATIONSHIP: 관계별 랭킹 / RECORD_TYPE: 기록유형별 랭킹 |
| limit | Integer | N | 5 | 조회 건수 (1~20) |
| startDate | String | N | | 조회 시작일 |
| endDate | String | N | | 조회 종료일 |

**Response:**
```json
{
  "code": "20000",
  "data": [
    {
      "rank": 1,
      "targetName": "김철수",
      "targetCd": "1",
      "count": 8,
      "totalAmount": 400000
    },
    {
      "rank": 2,
      "targetName": "이영희",
      "targetCd": "2",
      "count": 5,
      "totalAmount": 250000
    }
  ]
}
```

---

### 5.9 인사이트 (Insight)

#### `POST /api/v1/insight/getHomeInsight`

홈 화면에 표시할 맞춤 인사이트 메시지. 규칙 기반으로 자동 생성.

**Request:** 없음

**Response:**
```json
{
  "code": "20000",
  "data": {
    "insightMessage": "김철수님에게 연락할 때가 되었어요",
    "insightTypeCd": "TREND",
    "relatedRelationshipNm": "김철수",
    "generatedDt": "2026-03-13 10:00:00"
  }
}
```

**인사이트 유형 (insightTypeCd):**
| 코드 | 설명 | 예시 메시지 |
|------|------|-----------|
| TREND | 기록 추이 기반 | "이번 달 경조사 지출이 지난달보다 30% 증가했어요" |
| UPCOMING | 다가오는 일정 기반 | "이번 주 김철수님 결혼식이 있어요" |
| BALANCE | 보냄/받음 균형 기반 | "김철수님과의 주고받기 균형을 확인해보세요" |
| NO_CONTACT | 연락 부재 기반 | "김철수님에게 연락할 때가 되었어요" |

---

### 5.10 빠른 실행 (QuickAction)

#### `POST /api/v1/quickAction/getBaseData`

빠른 기록/일정 등록 시 필요한 기초 데이터 일괄 조회.

> **사용 시나리오:** 사용자가 "빠른 등록" 버튼을 눌렀을 때, 관계 목록과 코드 목록을 한 번에 가져온다.

**Request:** 없음

**Response:**
```json
{
  "code": "20000",
  "data": {
    "relationships": [
      {
        "relationshipNo": 1,
        "relationshipNm": "김철수",
        "relationshipTypeCd": "FRIEND"
      }
    ],
    "recordTypeCodes": [
      { "codeCd": "WEDDING", "codeNm": "결혼" },
      { "codeCd": "FUNERAL", "codeNm": "장례" },
      { "codeCd": "BIRTHDAY", "codeNm": "생일" }
    ],
    "scheduleTypeCodes": [
      { "codeCd": "BIRTHDAY", "codeNm": "생일" },
      { "codeCd": "ANNIVERSARY", "codeNm": "기념일" },
      { "codeCd": "WEDDING", "codeNm": "결혼" }
    ]
  }
}
```

---

## 6. 주요 코드 값 참고

### 6.1 관계 유형 (relationshipTypeCd)
| 코드 | 설명 |
|------|------|
| FAMILY | 가족 |
| FRIEND | 친구 |
| WORK | 직장 |
| ETC | 기타 |

### 6.2 기록 유형 (recordTypeCd)
| 코드 | 설명 |
|------|------|
| WEDDING | 결혼 |
| FUNERAL | 장례 |
| BIRTHDAY | 생일 |
| FIRST_BIRTHDAY | 돌잔치 |
| HOUSEWARMING | 집들이 |
| PROMOTION | 승진 |
| ETC | 기타 |

### 6.3 방향 (directionCd)
| 코드 | 설명 |
|------|------|
| GIVEN | 보냄 (내가 준 것) |
| RECEIVED | 받음 (내가 받은 것) |

### 6.4 일정 유형 (scheduleTypeCd)
| 코드 | 설명 |
|------|------|
| BIRTHDAY | 생일 |
| ANNIVERSARY | 기념일 |
| WEDDING | 결혼 |
| FUNERAL | 장례 |
| ETC | 기타 |

### 6.5 반복 주기 (repeatCycleCd)
| 코드 | 설명 |
|------|------|
| YEARLY | 매년 |
| MONTHLY | 매월 |

### 6.6 알림 유형 (notificationTypeCd)
| 코드 | 설명 |
|------|------|
| SCHEDULE_ALARM | 일정 알림 |

### 6.7 알림 대상 유형 (targetTypeCd)
| 코드 | 설명 | 클릭 시 이동 |
|------|------|-------------|
| SCHEDULE | 일정 | 일정 상세 (/schedule/getDetail) |
| RECORD | 기록 | 기록 상세 (/record/getDetail) |
| RELATIONSHIP | 관계 | 관계 상세 (/relationship/getDetail) |

---

## 7. 프론트엔드 구현 시 참고 사항

### 7.1 화면별 API 매핑 가이드

| 화면 | 사용 API |
|------|---------|
| **홈/대시보드** | `dashboard/getHomeSummary` + `insight/getHomeInsight` |
| **관계 목록** | `relationship/getList` |
| **관계 상세** | `relationship/getDetail` |
| **관계 등록/수정** | `relationship/createRelationship` / `updateRelationship` |
| **기록 목록** | `record/getList` |
| **기록 상세** | `record/getDetail` |
| **기록 등록/수정** | `record/createRecord` / `updateRecord` |
| **캘린더** | `schedule/getCalendarList` |
| **일정 상세** | `schedule/getDetail` |
| **일정 등록/수정** | `schedule/createSchedule` / `updateSchedule` |
| **알림 목록** | `notification/getList` |
| **알림 뱃지** | `notification/getUnreadCount` |
| **통계 화면** | `statistics/getMonthlyTrend` + `getRecordTypeDistribution` + `getRelationshipTypeDistribution` + `getTopFrequent` |
| **빠른 등록 모달** | `quickAction/getBaseData` → `record/createRecord` 또는 `schedule/createSchedule` |
| **내 정보** | `user/getMyInfo` |

### 7.2 수정 API 사용법

수정 API는 **부분 수정(Partial Update)** 을 지원한다.
- 변경할 필드만 전송하면 해당 필드만 업데이트된다.
- 전송하지 않은(null인) 필드는 기존 값이 유지된다.
- PK(xxxNo)는 반드시 전송해야 한다.

```json
// 이름만 변경하는 경우
{
  "relationshipNo": 1,
  "relationshipNm": "김철수(변경)"
}

// 이름 + 전화번호 변경
{
  "relationshipNo": 1,
  "relationshipNm": "김철수(변경)",
  "phoneNo": "010-9999-8888"
}
```

### 7.3 페이징 처리

- 모든 목록 API는 `limit` + `offset` 기반 페이징을 사용한다.
- `limit`의 최대값은 100, `offset`의 최대값은 10000이다.
- 서버에서 자동으로 범위를 보정한다 (limit=200 요청 → 100으로 보정).

### 7.4 삭제된 관계 처리

관계가 삭제되어도 연결된 기록/일정 데이터는 유지된다.
기록/일정 조회 시 `relationshipNm`이 null로 올 수 있으므로, 프론트에서 "삭제된 관계"로 대체 표시해야 한다.

### 7.5 보안 관련

- 모든 데이터는 **사용자 본인 것만** 조회/수정/삭제 가능하다.
- 타인 데이터에 접근 시 `ACCESS_DENIED` 에러가 반환된다.
- 프론트에서 별도 인증 토큰 전송은 불필요하다 (Gateway에서 처리).

---

## 8. 향후 확장 예정

| 기능 | 현재 상태 | 비고 |
|------|----------|------|
| 공통코드 관리 API | 미구현 | 현재 DB 직접 관리 |
| 가족 공유 기능 | 미구현 | 향후 확장 예정 |
| 반복 일정 전개 로직 | 1차 구현 | 현재는 원본 일정일 기준으로만 조회. 반복 전개는 2차 |
| 인사이트 고도화 | 1차 구현 | 현재 규칙 기반. 향후 배치 + 사전계산 테이블 확장 |
| 통계 캐싱 | 미적용 | 대량 데이터 시 캐시/배치 전략 도입 예정 |
