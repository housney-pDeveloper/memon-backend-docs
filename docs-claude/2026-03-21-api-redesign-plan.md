# MEMON Backend API Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Frontend UI 리디자인에 맞춰 Backend API를 정합시킨다. 기간 필터 추가, 누락 필드 보완, 불필요 도메인 삭제.

**Architecture:** 기존 Resolver 패턴 + 도메인별 패키지 구조를 유지하면서, DTO 필드 추가와 DAO 쿼리 수정으로 Frontend 타입과 1:1 정합을 달성한다. POST Only, v1 직접 수정.

**Tech Stack:** Java 21, Spring Boot 3.5, MyBatis, PostgreSQL

**Spec:** `01_docs/docs-claude/2026-03-21-api-redesign-design.md`

---

## File Structure

### 신규 생성 파일

| 파일 | 책임 |
|------|------|
| `32_application/.../dashboard/v1/dto/HomeSummaryRequestDtoV1.java` | Dashboard 기간 필터 요청 DTO |
| `32_application/.../dashboard/v1/dto/HostEventDtoV1.java` | 개별 행사 상세 (received 금액, 관계별/항목별 집계) |

### 수정 파일

| 파일 | 변경 내용 |
|------|-----------|
| `32_application/.../dashboard/controller/DashboardController.java` | Request 파라미터 추가 |
| `32_application/.../dashboard/resolver/DashboardService.java` | 메서드 시그니처 변경 |
| `32_application/.../dashboard/v1/service/DashboardServiceV1.java` | 기간 필터 로직 추가 |
| `32_application/.../dashboard/v1/dao/DashboardDaoV1.java` | 기간 파라미터 전달 |
| `32_application/.../dashboard/v1/dto/DashboardRecentRecordDtoV1.java` | `relationshipTypeNo` 추가 |
| `32_application/.../dashboard/v1/dto/HostSummaryDtoV1.java` | `events` 필드 추가 |
| `32_application/.../dashboard/v1/dto/CategoryAmountDtoV1.java` | `percentage` int→Double 변경 |
| `32_application/src/main/resources/mapper/dashboard/v1/DashboardMapperV1.xml` | 날짜 조건, JOIN, 이벤트별 쿼리 추가 |
| `32_application/.../record/v1/dto/RecordDetailResponseDtoV1.java` | `relationshipTypeNo` 추가 |
| `32_application/src/main/resources/mapper/record/v1/RecordMapperV1.xml` | Detail 쿼리에 relationship_type_no 추가 |
| `32_application/.../ceremony/v1/dto/CeremonyDetailResponseDtoV1.java` | `relationshipTypeNo` 추가 (기존 typeName/typeColor 보존) |
| `32_application/src/main/resources/mapper/ceremony/v1/CeremonyMapperV1.xml` | Detail 쿼리에 relationship_type_no 추가 |
| `32_application/.../relationship/v1/dto/RelationshipListResponseDtoV1.java` | `totalGivenAmount`, `totalReceivedAmount`, `memo` 추가 |
| `32_application/src/main/resources/mapper/relationship/v1/RelationshipMapperV1.xml` | List 쿼리에 집계 서브쿼리 추가 |

### 삭제 대상

| 디렉토리 | 파일 수 |
|----------|--------|
| `32_application/.../biz/quickaction/` (전체) | Java 7파일 |
| `32_application/src/main/resources/mapper/quickaction/` (전체) | XML 1파일 |
| `32_application/.../biz/insight/` (전체) | Java 7파일 |
| `32_application/src/main/resources/mapper/insight/` (전체) | XML 1파일 |

---

## Task 1: Dashboard — 기간 필터 + 누락 필드 추가

**Files:**
- Create: `32_application/src/main/java/kr/housney/memon/api/biz/dashboard/v1/dto/HomeSummaryRequestDtoV1.java`
- Create: `32_application/src/main/java/kr/housney/memon/api/biz/dashboard/v1/dto/HostEventDtoV1.java`
- Modify: `32_application/src/main/java/kr/housney/memon/api/biz/dashboard/controller/DashboardController.java`
- Modify: `32_application/src/main/java/kr/housney/memon/api/biz/dashboard/resolver/DashboardService.java`
- Modify: `32_application/src/main/java/kr/housney/memon/api/biz/dashboard/v1/service/DashboardServiceV1.java`
- Modify: `32_application/src/main/java/kr/housney/memon/api/biz/dashboard/v1/dao/DashboardDaoV1.java`
- Modify: `32_application/src/main/java/kr/housney/memon/api/biz/dashboard/v1/dto/DashboardRecentRecordDtoV1.java`
- Modify: `32_application/src/main/java/kr/housney/memon/api/biz/dashboard/v1/dto/HostSummaryDtoV1.java`
- Modify: `32_application/src/main/java/kr/housney/memon/api/biz/dashboard/v1/dto/CategoryAmountDtoV1.java`
- Modify: `32_application/src/main/resources/mapper/dashboard/v1/DashboardMapperV1.xml`

- [ ] **Step 1: HomeSummaryRequestDtoV1 생성**

```java
package kr.housney.memon.api.biz.dashboard.v1.dto;

import lombok.Getter;
import lombok.Setter;
import org.apache.ibatis.type.Alias;

@Getter @Setter
@Alias("HomeSummaryRequestDtoV1")
public class HomeSummaryRequestDtoV1 {
    private Long userNo;
    private String startDate;   // nullable, YYYYMMDD
    private String endDate;     // nullable, YYYYMMDD
}
```

- [ ] **Step 2: HostEventDtoV1 생성**

```java
package kr.housney.memon.api.biz.dashboard.v1.dto;

import lombok.Getter;
import lombok.Setter;
import org.apache.ibatis.type.Alias;
import java.util.List;

@Getter @Setter
@Alias("HostEventDtoV1")
public class HostEventDtoV1 {
    private Long ceremonyNo;
    private String title;
    private String ceremonyTypeCode;
    private String ceremonyDate;
    private Long receivedAmount;
    private List<RelationAmountDtoV1> receivedByRelation;
    private List<CategoryAmountDtoV1> expenseByCategory;
}
```

- [ ] **Step 3: DashboardRecentRecordDtoV1에 relationshipTypeNo 추가**

기존 필드에 추가:
```java
private Long relationshipTypeNo;  // tb_relationship → tb_relationship_type JOIN
```

- [ ] **Step 4: HostSummaryDtoV1에 events 필드 추가**

기존 필드에 추가:
```java
private List<HostEventDtoV1> events;
```

- [ ] **Step 5: CategoryAmountDtoV1.percentage 타입 변경**

`private Integer percentage;` → `private Double percentage;`

- [ ] **Step 6: DashboardController 수정**

`getHomeSummary` 메서드에 `@RequestBody HomeSummaryRequestDtoV1 request` 파라미터 추가. 기존에 body 없이 호출하던 것을 변경.

- [ ] **Step 7: DashboardService (resolver interface) 시그니처 변경**

`HomeSummaryResponseDtoV1 getHomeSummary(Long userNo)` → `HomeSummaryResponseDtoV1 getHomeSummary(HomeSummaryRequestDtoV1 request)`

- [ ] **Step 8: DashboardServiceV1 로직 수정**

- `getHomeSummary` 메서드에서 request의 `startDate`/`endDate`를 DAO 호출 시 전달
- 금액 집계 쿼리(selectHostTotalReceivedAmount, selectGuestTotalGivenAmount 등)에 날짜 조건 추가
- `recentRecords`, `upcomingSchedules`는 기간 필터 무관하게 기존 로직 유지
- Host events 조회 로직 추가 (ceremonies에서 hostTypeCode='SELF' 조회 → 각 이벤트별 관계/항목 집계)

- [ ] **Step 9: DashboardDaoV1 메서드 시그니처 변경**

기존 `selectHostTotalReceivedAmount(Long userNo)` → `selectHostTotalReceivedAmount(HomeSummaryRequestDtoV1 request)` 등 날짜 파라미터가 필요한 메서드들 변경.

신규 추가:
- `List<HostEventDtoV1> selectHostEvents(HomeSummaryRequestDtoV1 request)`
- `List<RelationAmountDtoV1> selectEventReceivedByRelation(Map params)` (ceremonyNo + userNo)
- `List<CategoryAmountDtoV1> selectEventExpenseByCategory(Map params)` (ceremonyNo + userNo)

- [ ] **Step 10: DashboardMapperV1.xml 쿼리 수정**

금액 집계 쿼리에 날짜 조건 추가:
```xml
<if test="startDate != null and startDate != ''">
    AND r.record_date >= #{startDate}
</if>
<if test="endDate != null and endDate != ''">
    AND r.record_date &lt;= #{endDate}
</if>
```

recentRecords 쿼리에 `rel.relationship_type_no` 추가 (tb_relationship_type JOIN):
```xml
LEFT JOIN scm_mmhr.tb_relationship_type rt
  ON rel.relationship_type_no = rt.relationship_type_no
```
SELECT 절에 `rel.relationship_type_no AS relationshipTypeNo` 추가.

Host events 쿼리 추가: `hostTypeCode='SELF'`인 ceremony 조회.

- [ ] **Step 11: 빌드 검증**

```bash
cd 32_application && ./gradlew compileJava
```

- [ ] **Step 12: 커밋**

```bash
git add 32_application/
git commit -m "feat: Dashboard API에 기간 필터 추가 및 Frontend 타입 정합"
```

---

## Task 2: Record — Detail에 relationshipTypeNo 추가

**Files:**
- Modify: `32_application/src/main/java/kr/housney/memon/api/biz/record/v1/dto/RecordDetailResponseDtoV1.java`
- Modify: `32_application/src/main/resources/mapper/record/v1/RecordMapperV1.xml`

- [ ] **Step 1: RecordDetailResponseDtoV1에 relationshipTypeNo 추가**

기존 필드에 추가:
```java
private Long relationshipTypeNo;
```

- [ ] **Step 2: RecordMapperV1.xml selectRecordDetail 쿼리 수정**

SELECT 절에 `rel.relationship_type_no AS relationshipTypeNo` 추가. 기존 `tb_relationship_type` JOIN이 이미 있으면 SELECT만 추가, 없으면 JOIN도 추가.

- [ ] **Step 3: 빌드 검증**

```bash
cd 32_application && ./gradlew compileJava
```

- [ ] **Step 4: 커밋**

```bash
git add 32_application/
git commit -m "feat: Record Detail에 relationshipTypeNo 필드 추가"
```

---

## Task 3: Ceremony — Detail에 relationshipTypeNo 추가

**Files:**
- Modify: `32_application/src/main/java/kr/housney/memon/api/biz/ceremony/v1/dto/CeremonyDetailResponseDtoV1.java`
- Modify: `32_application/src/main/resources/mapper/ceremony/v1/CeremonyMapperV1.xml`

- [ ] **Step 1: CeremonyDetailResponseDtoV1에 relationshipTypeNo 추가**

기존 필드에 추가 (typeName, typeColor 보존):
```java
private Long relationshipTypeNo;
```

- [ ] **Step 2: CeremonyMapperV1.xml selectCeremonyDetail 쿼리 수정**

SELECT 절에 `rel.relationship_type_no AS relationshipTypeNo` 추가. 기존 tb_relationship_type JOIN 활용.

- [ ] **Step 3: 빌드 검증**

```bash
cd 32_application && ./gradlew compileJava
```

- [ ] **Step 4: 커밋**

```bash
git add 32_application/
git commit -m "feat: Ceremony Detail에 relationshipTypeNo 필드 추가"
```

---

## Task 4: Relationship — List에 집계 필드 추가

**Files:**
- Modify: `32_application/src/main/java/kr/housney/memon/api/biz/relationship/v1/dto/RelationshipListResponseDtoV1.java`
- Modify: `32_application/src/main/resources/mapper/relationship/v1/RelationshipMapperV1.xml`

- [ ] **Step 1: RelationshipListResponseDtoV1에 필드 추가**

기존 DTO에 누락 필드 추가:
```java
private Long totalGivenAmount;      // 서브쿼리 (direction_code='GIVEN')
private Long totalReceivedAmount;   // 서브쿼리 (direction_code='RECEIVED')
private String memo;                // tb_relationship.memo 직접 매핑
```

주의: `totalRecordCount`, `lastRecordDate`, `upcomingScheduleYn`은 이미 존재하는지 확인 후 없으면 추가.

- [ ] **Step 2: RelationshipMapperV1.xml selectRelationshipList 쿼리 수정**

SELECT 절에 집계 서브쿼리 추가:
```sql
, r.memo
, (SELECT COALESCE(SUM(rec.amount), 0) FROM scm_mmhr.tb_record rec
   WHERE rec.relationship_no = r.relationship_no
   AND rec.direction_code = 'GIVEN' AND rec.deleted_yn = 'N') AS total_given_amount
, (SELECT COALESCE(SUM(rec.amount), 0) FROM scm_mmhr.tb_record rec
   WHERE rec.relationship_no = r.relationship_no
   AND rec.direction_code = 'RECEIVED' AND rec.deleted_yn = 'N') AS total_received_amount
```

기존에 `total_record_count`, `last_record_date`, `upcoming_schedule_yn` 서브쿼리가 없으면 함께 추가.

- [ ] **Step 3: 빌드 검증**

```bash
cd 32_application && ./gradlew compileJava
```

- [ ] **Step 4: 커밋**

```bash
git add 32_application/
git commit -m "feat: Relationship List에 금액 집계 필드 추가"
```

---

## Task 5: QuickAction + Insight 도메인 삭제

**Files to delete:**
- `32_application/src/main/java/kr/housney/memon/api/biz/quickaction/` (전체 디렉토리)
- `32_application/src/main/resources/mapper/quickaction/` (전체 디렉토리)
- `32_application/src/main/java/kr/housney/memon/api/biz/insight/` (전체 디렉토리)
- `32_application/src/main/resources/mapper/insight/` (전체 디렉토리)

- [ ] **Step 1: QuickAction 도메인 전체 삭제**

```bash
rm -rf 32_application/src/main/java/kr/housney/memon/api/biz/quickaction
rm -rf 32_application/src/main/resources/mapper/quickaction
```

- [ ] **Step 2: Insight 도메인 전체 삭제**

```bash
rm -rf 32_application/src/main/java/kr/housney/memon/api/biz/insight
rm -rf 32_application/src/main/resources/mapper/insight
```

- [ ] **Step 3: 삭제된 도메인 참조 확인**

다른 파일에서 QuickAction/Insight를 import하는 곳이 있는지 확인하고, 있으면 제거.

```bash
grep -r "quickaction\|QuickAction\|insight\|Insight" 32_application/src/main/java/ --include="*.java" -l
```

- [ ] **Step 4: 빌드 검증**

```bash
cd 32_application && ./gradlew compileJava
```

- [ ] **Step 5: 커밋**

```bash
git add 32_application/
git commit -m "chore: QuickAction, Insight 도메인 전체 삭제 (Frontend 미사용)"
```

---

## Task 6: 전체 빌드 검증 + Frontend 타입 변경 커밋

**Files:**
- Frontend 타입 파일 변경분 (relationshipTypeCode → relationshipTypeNo)

- [ ] **Step 1: Backend 전체 빌드**

```bash
cd 32_application && ./gradlew build
```

- [ ] **Step 2: Frontend 타입 변경 커밋**

Frontend에서 `relationshipTypeCode` → `relationshipTypeNo` 변경된 파일들:
- `21_hs-common/src/types/dashboard.ts`
- `21_hs-common/src/types/record.ts`
- `21_hs-common/src/types/ceremony.ts`
- `21_hs-common/src/types/statistics.ts`
- `21_hs-common/src/types/relationship.ts`
- `31_webadmin/src/widgets/dashboard/RecentRecordsSection.tsx`
- `31_webadmin/src/app/(authenticated)/records/[id]/page.tsx`
- `31_webadmin/src/app/(authenticated)/records/page.tsx`
- `31_webadmin/src/app/(authenticated)/statistics/page.tsx`
- `32_appuser/app/(tabs)/index.tsx`

- [ ] **Step 3: 최종 커밋**

```bash
# Frontend submodules
cd 21_hs-common && git add -A && git commit -m "refactor: relationshipTypeCode를 relationshipTypeNo로 통일"
cd 31_webadmin && git add -A && git commit -m "refactor: relationshipTypeNo 타입 정합"
cd 32_appuser && git add -A && git commit -m "refactor: relationshipTypeNo 타입 정합"
```

---

## 실행 순서

```
Task 1 (Dashboard) ─── 가장 복잡, 독립 수행
Task 2 (Record) ─── 독립 수행
Task 3 (Ceremony) ─── 독립 수행
Task 4 (Relationship) ─── 독립 수행
Task 5 (삭제) ─── 독립 수행
Task 6 (검증+커밋) ─── Task 1-5 완료 후
```

**병렬 가능:** Task 1-5 모두 서로 다른 도메인이므로 병렬 수행 가능.
