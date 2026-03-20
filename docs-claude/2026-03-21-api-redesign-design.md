# MEMON Backend API Redesign — Design Spec

**Date:** 2026-03-21
**Scope:** 32_application (Business Logic Server) — API 전면 재설계
**Trigger:** Frontend UI 전면 리디자인에 따른 Backend 정합
**방식:** 아키텍처 유지 + Controller/Service/DAO/DTO 재설계, POST Only, v1 직접 수정

---

## 1. 설계 원칙

1. **아키텍처 골격 유지** — Resolver 패턴, 패키지 구조, 네이밍 컨벤션, 생성자 주입, Service 트랜잭션 관리
2. **POST Only** — 모든 엔드포인트 POST 메서드
3. **Frontend 타입 1:1 정합** — 응답 DTO 필드명/구조를 Frontend TypeScript 타입과 일치
4. **`RestResponse<T>`** — 표준 응답 래퍼 유지 (`code`, `message`, `data`)
5. **`PageDto<T>`** — 페이징 응답 유지 (`list`, `totalCount`, `pageNumber`, `pageSize`, `totalPages`)
6. **v1 직접 수정** — v2 도입 없이 기존 v1 엔드포인트 수정 (Frontend 동시 배포)
7. **DTO 네이밍** — 모든 DTO에 `V1` 접미사 필수 (기존 컨벤션: `{Action}{Type}DtoV1`)
8. **Java 타입** — nullable 필드는 `Long`/`Integer` wrapper 사용, non-null은 `long`/`int` primitive 허용 (기존 컨벤션 준수)
9. **DTO 상속 금지** — 기존 코드가 flat class 패턴이므로 DTO 상속(`extends`) 사용하지 않음. 모든 DTO는 독립 flat class로 정의
10. **`relationshipTypeNo` 통일** — Frontend `RelationAmount.relationshipTypeCode`를 `relationshipTypeNo: number`로 변경하여 Backend `relationship_type_no`와 직접 매핑. `String.valueOf()` 변환 불필요

---

## 2. 엔드포인트 전체 매핑

### 2.1 수정 대상 (9건)

| 도메인 | 엔드포인트 | 변경 내용 |
|--------|-----------|-----------|
| Dashboard | `getHomeSummary` | Request에 DateRange 추가, 기간별 집계 |
| Record | `getList` | 응답에 `relationshipName` JOIN |
| Record | `getDetail` | 응답에 `relationshipTypeCode` JOIN |
| Ceremony | `getList` | 응답에 `relationshipName` JOIN |
| Ceremony | `getDetail` | 응답에 `relationshipTypeCode` JOIN |
| Relationship | `getList` | 응답에 집계 필드 5개 추가 |
| Relationship | `getDetail` | 중첩 데이터 포함 (anniversaries, recentRecords, upcomingSchedules) |
| Relationship | `getRelationshipTypeList` | 응답에 `relationshipCount` 집계 |
| Schedule | `getUpcomingList` | 응답에 `dday`, `relationshipName` 추가 |

### 2.2 삭제 대상 (2건)

| 도메인 | 삭제 범위 | 이유 |
|--------|----------|------|
| QuickAction | Controller, Resolver, V1 전체 (`biz/quickaction/`) | Frontend 미사용 |
| Insight | Controller, Resolver, V1 전체 (`biz/insight/`) | 대시보드 리디자인으로 제거 |

### 2.3 유지 대상

| 도메인 | 엔드포인트 |
|--------|-----------|
| Auth | `syncOAuthUser`, `recordLoginHistory` |
| User | `getMyInfo` |
| Record | `getRecentList`, `createRecord`, `updateRecord`, `deleteRecord` |
| Ceremony | `getStatistics`, `createCeremony`, `updateCeremony`, `deleteCeremony` |
| Relationship | `getOptionList`, `createRelationship`, `updateRelationship`, `deleteRelationship`, `createRelationshipType`, `updateRelationshipType`, `deleteRelationshipType` |
| Schedule | `getCalendarList`, `getDetail`, `createSchedule`, `updateSchedule`, `deleteSchedule` |
| Statistics | `getMonthlyTrend`, `getRecordTypeDistribution`, `getRelationshipTypeDistribution`, `getTopFrequent` |
| Notification | `getUnreadCount`, `getList`, `markAsRead`, `markAllAsRead` |
| CommonCode | `getAll`, `getList`, `save` |

---

## 3. Dashboard API 상세

### 3.1 `POST /api/v1/dashboard/getHomeSummary`

**Request DTO — `HomeSummaryRequestDtoV1`:**

```java
@Alias("HomeSummaryRequestDtoV1")
public class HomeSummaryRequestDtoV1 {
    private String startDate;   // nullable, YYYYMMDD
    private String endDate;     // nullable, YYYYMMDD
}
```

**Response DTO — `HomeSummaryResponseDtoV1`:**

Frontend `HomeSummary` 타입과 1:1 매칭.

```java
@Alias("HomeSummaryResponseDtoV1")
public class HomeSummaryResponseDtoV1 {
    private String userName;
    private int totalRelationshipCount;
    private int totalRecordCount;           // 기간 필터 적용
    private int monthlyRecordCount;         // 이번 달 기록 수
    private int upcomingScheduleCount;      // 향후 일정 수

    private HostSummaryDtoV1 hostSummary;
    private GuestSummaryDtoV1 guestSummary;

    private List<DashboardRecentRecordDtoV1> recentRecords;           // 최근 5건, 기간 필터 무관
    private List<DashboardUpcomingScheduleDtoV1> upcomingSchedules;   // 향후 5건, 기간 필터 무관
}
```

**중첩 DTO (모두 V1 접미사, flat class, wrapper 타입):**

```java
// HostSummaryDtoV1
public class HostSummaryDtoV1 {
    private Integer eventCount;
    private Long totalReceivedAmount;       // 기간 필터 적용
    private List<HostEventDtoV1> events;    // ← 신규 DTO
    private List<RelationAmountDtoV1> receivedByRelation;
    private List<CategoryAmountDtoV1> expenseByCategory;
}

// GuestSummaryDtoV1
public class GuestSummaryDtoV1 {
    private Integer eventCount;
    private Long totalGivenAmount;          // 기간 필터 적용
    private List<RelationAmountDtoV1> givenByRelation;
    private List<CategoryAmountDtoV1> givenByType;
}

// HostEventDtoV1 — 신규 생성
public class HostEventDtoV1 {
    private Long ceremonyNo;
    private String title;
    private String ceremonyTypeCode;
    private String ceremonyDate;
    private Long receivedAmount;
    private List<RelationAmountDtoV1> receivedByRelation;
    private List<CategoryAmountDtoV1> expenseByCategory;
}

// RelationAmountDtoV1
public class RelationAmountDtoV1 {
    private Long relationshipTypeNo;        // relationship_type_no 직접 매핑
    private String label;                   // type_name
    private Long amount;
}

// CategoryAmountDtoV1
public class CategoryAmountDtoV1 {
    private String categoryCode;
    private String label;
    private Long amount;
    private Double percentage;              // int → Double 변경 (Frontend number 타입 정합)
}

// DashboardRecentRecordDtoV1 — Frontend DashboardRecentRecord와 1:1
public class DashboardRecentRecordDtoV1 {
    private Long recordNo;
    private String relationshipName;        // tb_relationship JOIN
    private Long relationshipTypeNo;        // tb_relationship → tb_relationship_type JOIN
    private String recordTypeCode;
    private String recordDate;
    private Long amount;
    private String directionCode;
    private String hostTypeCode;            // "SELF" or "OTHER"
}

// DashboardUpcomingScheduleDtoV1 — Frontend DashboardUpcomingSchedule와 1:1
public class DashboardUpcomingScheduleDtoV1 {
    private Long scheduleNo;
    private String title;
    private String scheduleDate;
    private String scheduleTypeCode;
    private String relationshipName;        // tb_relationship JOIN
    private Integer dday;                   // DB 계산: schedule_date::date - CURRENT_DATE
}
```

**DAO 쿼리 변경:**
- 금액 집계 쿼리에 `WHERE` 절 추가: `AND (#{startDate} IS NULL OR r.record_date >= #{startDate}) AND (#{endDate} IS NULL OR r.record_date <= #{endDate})`
- `recentRecords` 쿼리: `tb_record r LEFT JOIN tb_relationship rel ON r.relationship_no = rel.relationship_no LEFT JOIN tb_relationship_type rt ON rel.relationship_type_no = rt.relationship_type_no`
- `upcomingSchedules` 쿼리: `(s.schedule_date::date - CURRENT_DATE) AS dday`, `LEFT JOIN tb_relationship`

---

## 4. Record API 상세

### 4.1 `POST /api/v1/record/getList`

**Request DTO — `RecordListRequestDtoV1`** (변경 없음):

```java
public class RecordListRequestDtoV1 extends RecordVO {
    private String searchKeyword;
    private String sortBy;
    private String sortOrder;
    private int pageNumber;
    private int pageSize;
}
```

**Response DTO — `RecordListResponseDtoV1`** (수정):

```java
@Alias("RecordListResponseDtoV1")
public class RecordListResponseDtoV1 {
    private Long recordNo;
    private Long relationshipNo;
    private String relationshipName;        // ← 추가 (JOIN)
    private String recordTypeCode;
    private String recordDate;
    private Long amount;
    private String directionCode;
    private String memo;
    private String regDate;
}
```

**DAO 변경:**
```sql
SELECT r.record_no, r.relationship_no, rel.relationship_name,
       r.record_type_code, r.record_date, r.amount,
       r.direction_code, r.memo, r.reg_date
FROM scm_mmhr.tb_record r
LEFT JOIN scm_mmhr.tb_relationship rel
  ON r.relationship_no = rel.relationship_no
WHERE r.user_no = #{userNo}
  AND r.deleted_yn = 'N'
```

### 4.2 `POST /api/v1/record/getDetail`

**Response DTO — `RecordDetailResponseDtoV1`** (수정, flat class):

```java
@Alias("RecordDetailResponseDtoV1")
public class RecordDetailResponseDtoV1 {
    private Long recordNo;
    private Long relationshipNo;
    private String relationshipName;        // JOIN
    private String recordTypeCode;
    private String recordDate;
    private Long amount;
    private String directionCode;
    private String memo;
    private String regDate;
    private Long relationshipTypeNo;        // ← 추가: relationship_type_no 직접 매핑
    private String modDate;
}
```

**DAO 변경:** `LEFT JOIN scm_mmhr.tb_relationship rel ON r.relationship_no = rel.relationship_no LEFT JOIN scm_mmhr.tb_relationship_type rt ON rel.relationship_type_no = rt.relationship_type_no`, `rt.relationship_type_no`

---

## 5. Ceremony API 상세

### 5.1 `POST /api/v1/ceremony/getList`

**Response DTO — `CeremonyListResponseDtoV1`** (수정):

```java
@Alias("CeremonyListResponseDtoV1")
public class CeremonyListResponseDtoV1 {
    private Long ceremonyNo;
    private Long relationshipNo;
    private String relationshipName;        // ← 추가 (JOIN)
    private String ceremonyTypeCode;
    private String title;
    private String ceremonyDate;
    private String location;
    private Long amount;
    private String directionCode;
    private String attendYn;
    private String memo;
    private Long linkedRecordNo;            // ← 추가 (Frontend 정합)
    private String regDate;
}
```

### 5.2 `POST /api/v1/ceremony/getDetail`

**Response DTO — `CeremonyDetailResponseDtoV1`** (수정, flat class):

```java
@Alias("CeremonyDetailResponseDtoV1")
public class CeremonyDetailResponseDtoV1 {
    private Long ceremonyNo;
    private Long relationshipNo;
    private String relationshipName;
    private String ceremonyTypeCode;
    private String title;
    private String ceremonyDate;
    private String location;
    private Long amount;
    private String directionCode;
    private String attendYn;
    private String memo;
    private Long linkedRecordNo;
    private String regDate;
    private Long relationshipTypeNo;        // ← 추가: relationship_type_no 직접 매핑
    private String modDate;
}
```

**DAO 변경:** Record와 동일 패턴 — `tb_relationship` + `tb_relationship_type` JOIN 추가.

---

## 6. Relationship API 상세

### 6.1 `POST /api/v1/relationship/getList`

**Response DTO — `RelationshipListResponseDtoV1`** (수정):

```java
@Alias("RelationshipListResponseDtoV1")
public class RelationshipListResponseDtoV1 {
    private Long relationshipNo;
    private String relationshipName;
    private Long relationshipTypeNo;
    private String typeName;                // ← JOIN (tb_relationship_type)
    private String typeColor;               // ← JOIN (tb_relationship_type)
    private String phoneNo;
    private String memo;
    private String birthday;
    private String regDate;
    private Integer totalRecordCount;       // ← 서브쿼리
    private Long totalGivenAmount;          // ← 서브쿼리 (direction_code='GIVEN')
    private Long totalReceivedAmount;       // ← 서브쿼리 (direction_code='RECEIVED')
    private String lastRecordDate;          // ← 서브쿼리 MAX(record_date)
    private String upcomingScheduleYn;      // ← 서브쿼리 EXISTS (future schedule)
    // tags 기능 전체 제거됨 — 목록/상세 모두 tags 미포함
}
```

**DAO 쿼리:**
```sql
SELECT r.*, rt.type_name, rt.type_color,
  (SELECT COUNT(*) FROM scm_mmhr.tb_record rec
   WHERE rec.relationship_no = r.relationship_no AND rec.deleted_yn = 'N') AS total_record_count,
  (SELECT COALESCE(SUM(rec.amount), 0) FROM scm_mmhr.tb_record rec
   WHERE rec.relationship_no = r.relationship_no AND rec.direction_code = 'GIVEN' AND rec.deleted_yn = 'N') AS total_given_amount,
  (SELECT COALESCE(SUM(rec.amount), 0) FROM scm_mmhr.tb_record rec
   WHERE rec.relationship_no = r.relationship_no AND rec.direction_code = 'RECEIVED' AND rec.deleted_yn = 'N') AS total_received_amount,
  (SELECT MAX(rec.record_date) FROM scm_mmhr.tb_record rec
   WHERE rec.relationship_no = r.relationship_no AND rec.deleted_yn = 'N') AS last_record_date,
  CASE WHEN EXISTS (SELECT 1 FROM scm_mmhr.tb_schedule s
   WHERE s.relationship_no = r.relationship_no AND s.schedule_date >= CURRENT_DATE AND s.deleted_yn = 'N')
   THEN 'Y' ELSE 'N' END AS upcoming_schedule_yn
FROM scm_mmhr.tb_relationship r
LEFT JOIN scm_mmhr.tb_relationship_type rt ON r.relationship_type_no = rt.relationship_type_no
WHERE r.user_no = #{userNo} AND r.deleted_yn = 'N'
```

### 6.2 `POST /api/v1/relationship/getDetail`

**Response DTO — `RelationshipDetailResponseDtoV1`** (수정):

```java
// RelationshipDetailResponseDtoV1 — flat class (상속 금지)
@Alias("RelationshipDetailResponseDtoV1")
public class RelationshipDetailResponseDtoV1 {
    // 기본 필드 (RelationshipListResponseDtoV1과 동일)
    private Long relationshipNo;
    private String relationshipName;
    private Long relationshipTypeNo;
    private String typeName;
    private String typeColor;
    private String phoneNo;
    private String memo;
    private String birthday;
    private String regDate;
    private Integer totalRecordCount;
    private Long totalGivenAmount;
    private Long totalReceivedAmount;
    private String lastRecordDate;
    private String upcomingScheduleYn;
    // 중첩 데이터 (Detail 전용)
    private List<RelationshipAnniversaryDtoV1> anniversaries;
    private List<RelationshipRecentRecordDtoV1> recentRecords;        // 최근 5건
    private List<RelationshipUpcomingScheduleDtoV1> upcomingSchedules; // 향후 5건
}

// RelationshipAnniversaryDtoV1 — Frontend RelationshipAnniversary와 1:1
public class RelationshipAnniversaryDtoV1 {
    private Long anniversaryNo;
    private String anniversaryDate;
    private String title;
    private String memo;
}

// RelationshipRecentRecordDtoV1 — Frontend RelationshipRecentRecord와 1:1
public class RelationshipRecentRecordDtoV1 {
    private Long recordNo;
    private String recordTypeCode;
    private String recordDate;
    private Long amount;
    private String directionCode;
}

// RelationshipUpcomingScheduleDtoV1 — Frontend RelationshipUpcomingSchedule와 1:1
public class RelationshipUpcomingScheduleDtoV1 {
    private Long scheduleNo;
    private String title;
    private String scheduleDate;
    private Integer dday;               // schedule_date::date - CURRENT_DATE
}
```

**Service 로직:**
1. 기본 정보 조회 (`RelationshipListResponseDtoV1`와 동일 쿼리)
2. `anniversaries` 별도 쿼리 (`tb_relationship_anniversary WHERE relationship_no = ?`)
3. `recentRecords` 별도 쿼리 (`tb_record WHERE relationship_no = ? ORDER BY record_date DESC LIMIT 5`)
4. `upcomingSchedules` 별도 쿼리 (`tb_schedule WHERE relationship_no = ? AND schedule_date >= CURRENT_DATE ORDER BY schedule_date LIMIT 5`)

### 6.3 `POST /api/v1/relationship/getRelationshipTypeList`

**Response DTO — `RelationshipTypeInfoDtoV1`** (수정):

```java
@Alias("RelationshipTypeInfoDtoV1")
public class RelationshipTypeInfoDtoV1 {
    private Long relationshipTypeNo;
    private String typeName;
    private String typeColor;
    private Integer sortOrder;
    private Integer relationshipCount;      // ← 서브쿼리 COUNT
}
```

**DAO 변경:**
```sql
SELECT rt.*,
  (SELECT COUNT(*) FROM scm_mmhr.tb_relationship r
   WHERE r.relationship_type_no = rt.relationship_type_no AND r.deleted_yn = 'N') AS relationship_count
FROM scm_mmhr.tb_relationship_type rt
WHERE rt.user_no = #{userNo}
ORDER BY rt.sort_order
```

---

## 7. Schedule API 상세

### 7.1 `POST /api/v1/schedule/getUpcomingList`

**Response DTO — `UpcomingScheduleResponseDtoV1`** (수정):

```java
@Alias("UpcomingScheduleResponseDtoV1")
public class UpcomingScheduleResponseDtoV1 {
    private Long scheduleNo;
    private String title;
    private String scheduleDate;
    private String scheduleTypeCode;
    private String relationshipName;        // ← JOIN 추가
    private Integer dday;                   // ← DB 계산
}
```

**DAO 변경:**
```sql
SELECT s.schedule_no, s.title, s.schedule_date, s.schedule_type_code,
       rel.relationship_name,
       (s.schedule_date::date - CURRENT_DATE) AS dday
FROM scm_mmhr.tb_schedule s
LEFT JOIN scm_mmhr.tb_relationship rel ON s.relationship_no = rel.relationship_no
WHERE s.user_no = #{userNo}
  AND s.schedule_date >= CURRENT_DATE
  AND s.deleted_yn = 'N'
ORDER BY s.schedule_date ASC
```

### 7.2 `POST /api/v1/schedule/getDetail`

**Response DTO** 기존 필드에 `dday` 추가:

```java
// 기존 ScheduleDetailResponseDtoV1에 추가
private int dday;       // (schedule_date::date - CURRENT_DATE)
private String modDate;
```

---

## 8. 삭제 대상 상세

### 8.1 QuickAction 도메인 전체 삭제

```
32_application/src/main/java/kr/housney/memon/api/biz/quickaction/
├── controller/QuickActionController.java
├── QuickActionServiceResolver.java (또는 resolver/ 하위)
├── QuickActionService.java (resolver interface)
└── v1/
    ├── service/QuickActionServiceV1.java
    ├── dao/QuickActionDaoV1.java
    └── dto/ (모든 DTO 파일: QuickActionBaseDataResponseDtoV1, QuickActionCodeDtoV1, QuickActionRelationshipDtoV1 등)
```

MyBatis mapper XML도 함께 삭제: `resources/mapper/quickaction/` 전체

### 8.2 Insight 도메인 전체 삭제

```
32_application/src/main/java/kr/housney/memon/api/biz/insight/
├── controller/InsightController.java
├── InsightServiceResolver.java (또는 resolver/ 하위)
├── InsightService.java (resolver interface)
└── v1/
    ├── service/InsightServiceV1.java
    ├── dao/InsightDaoV1.java
    └── dto/ (모든 DTO 파일: HomeInsightResponseDtoV1 등)
```

MyBatis mapper XML도 함께 삭제: `resources/mapper/insight/` 전체

---

## 9. 영향 범위 정리

### 9.1 수정 파일

| 도메인 | 파일 유형 | 변경 내용 |
|--------|----------|-----------|
| Dashboard | Controller | Request 파라미터 추가 |
| Dashboard | Service | 기간 필터 로직 추가 |
| Dashboard | DAO + mapper XML | WHERE 절 날짜 조건, JOIN 추가 |
| Dashboard | DTO | Request DTO 신규, Response DTO 필드 추가 |
| Record | DAO + mapper XML | `tb_relationship` JOIN 추가 |
| Record | DTO | `relationshipName`, `relationshipTypeCode` 필드 추가 |
| Ceremony | DAO + mapper XML | `tb_relationship` JOIN 추가 |
| Ceremony | DTO | `relationshipName`, `relationshipTypeCode` 필드 추가 |
| Relationship | Service | getDetail에서 중첩 데이터 조합 로직 |
| Relationship | DAO + mapper XML | 집계 서브쿼리 5개, 중첩 데이터 쿼리 4개 |
| Relationship | DTO | 집계 필드 5개, 중첩 DTO 4종 추가 |
| Schedule | DAO + mapper XML | `dday` 계산, `tb_relationship` JOIN |
| Schedule | DTO | `dday`, `relationshipName` 필드 추가 |

### 9.2 삭제 파일

| 도메인 | 삭제 대상 |
|--------|----------|
| QuickAction | Controller, Resolver, Service, DAO, DTO, mapper XML 전체 |
| Insight | Controller, Resolver, Service, DAO, DTO, mapper XML 전체 |

### 9.3 Database 변경

- **테이블 구조 변경 없음** — 기존 스키마로 모든 요구사항 충족
- **DAO 쿼리만 변경** — JOIN, 서브쿼리, 계산 컬럼 추가

---

## 10. Frontend 타입 정합 검증 체크리스트

| Frontend 타입 | Backend Response DTO | 필드 매칭 |
|--------------|---------------------|-----------|
| `HomeSummary` | `HomeSummaryResponseDtoV1` | 전체 필드 1:1 |
| `DashboardRecentRecord` | `DashboardRecentRecordDto` | 전체 필드 1:1 |
| `DashboardUpcomingSchedule` | `DashboardUpcomingScheduleDto` | 전체 필드 1:1 |
| `MemonRecord` | `RecordListResponseDtoV1` | 전체 필드 1:1 |
| `RecordDetail` | `RecordDetailResponseDtoV1` | 전체 필드 1:1 |
| `Ceremony` | `CeremonyListResponseDtoV1` | 전체 필드 1:1 |
| `CeremonyDetail` | `CeremonyDetailResponseDtoV1` | 전체 필드 1:1 |
| `Relationship` | `RelationshipListResponseDtoV1` | 전체 필드 1:1 |
| `RelationshipDetail` | `RelationshipDetailResponseDtoV1` | 전체 필드 1:1 |
| `RelationshipTypeInfo` | `RelationshipTypeInfoDtoV1` | 전체 필드 1:1 |
| `UpcomingSchedule` | `UpcomingScheduleResponseDtoV1` | 전체 필드 1:1 |
| `RelationshipAnniversary` | `RelationshipAnniversaryDto` | 전체 필드 1:1 |
| `RelationshipRecentRecord` | `RelationshipRecentRecordDto` | 전체 필드 1:1 |
| `RelationshipUpcomingSchedule` | `RelationshipUpcomingScheduleDto` | 전체 필드 1:1 |

---
