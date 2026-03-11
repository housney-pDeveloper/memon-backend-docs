# CODE_CONVENTION.md

백엔드 전체 모듈(Gateway, Application, System)에 공통 적용되는 코드 컨벤션을 정의한다.

---

## 1. DI 규칙

- Java 21
- 생성자 주입만 허용
- 필드 주입 금지
- Controller에서 Dao 직접 접근 금지
- Service에서만 트랜잭션 관리

---

## 2. URL 규칙

- 모든 REST API는 POST만 허용 (예외: Health Check, SSE)
- camelCase 사용
- hyphen(-), underscore(_) 금지
- "by" 사용 금지

```
O  /getCodeGroupList
X  /getByCodeGroupList
```

---

## 3. 클래스 명명 규칙

| 역할 | Suffix |
|------|--------|
| Controller | XxxController |
| Service (인터페이스) | XxxService |
| Service (구현체) | XxxServiceV1 |
| Dao | XxxDao |
| VO | XxxVO |
| Dto | XxxDto |
| 설정 | XxxConfig |

### 3.1 Bean 충돌 방지

- 동일 Dao 이름 사용 금지
- 패턴: {Domain}{Entity}Dao

```
O  AuthClientDao, ClientDao
X  ClientDao (중복 발생)
```

---

## 4. VO / DTO 정책

### 4.1 Table → VO 매핑

- tb_{entity} → {Entity}VO
- BaseVO 상속
- 위치: biz/vo/{schema} (mmhr / mmauth / mmpublic)

### 4.2 VO → Dto 상속

- Dto는 VO 상속
- 필드 재정의 금지
- SearchDto는 SearchVO 상속

### 4.3 VO 명명 규칙

- tb_ prefix 제거
- CamelCase 변환
- suffix는 반드시 VO

```
tb_client → ClientVO
```

### 4.4 @Alias 규칙

- 모든 VO/DTO에 @Alias 필수
- Alias 값은 클래스명과 동일

---

## 5. Validation 규칙

### 5.1 Validation Group

- OnCreate
- OnUpdate
- OnSearch

위치: common/validation/ValidationGroups.java

### 5.2 작성 원칙

- Validation은 VO에 정의
- Dto는 VO 상속
- Dto에서 필드 재정의 금지

### 5.3 Controller 규칙

- **@Valid 사용 금지**
- **@Validated(Group.class) 사용**

```java
@Validated(OnCreate.class)
```

---

## 6. 응답 정책

### 6.1 통일된 응답 구조

```json
{
  "code": 1,
  "message": "OK",
  "data": {}
}
```

### 6.2 Gateway 응답

- RestResponse<T> 클래스
- success(data), fail(responseCode), fail(code, message)
- Filter에서 오류 응답 생성

### 6.3 Application / System 응답

- ResponseWrapperAdvice: 모든 API는 Object 또는 void 반환 → 자동 RestResponse.success() wrapping
- ExceptionAdvice:
  - BizException → HTTP 200
  - Exception → HTTP 500
  - Validation 오류 → HTTP 200 + 상세 data 포함

---

## 7. 에러 코드 글로벌 정책

### 7.1 기본 규칙

- OK = 1
- FAIL = -1
- 에러 코드는 5자리 숫자

형식: {프로젝트번호}{비즈니스1레벨}{비즈니스2레벨}{시퀀스}

### 7.2 프로젝트 번호

| 프로젝트 | 번호 |
|----------|------|
| Gateway | 1 |
| System | 2 |
| Application | 3 |
| 공통 | 9 |

### 7.3 신규 프로젝트 추가

1. 번호 4-8 중 할당
2. hs-common/ErrorCode.java 주석 추가
3. 프로젝트 CLAUDE.md 업데이트

---

## 8. 계층 책임 원칙

- controller에서 Repository/Dao 직접 접근 금지
- service에서만 트랜잭션 관리
- core / common / biz 역할 혼합 금지
- biz 외부에 비즈니스 로직 작성 금지
- @Transactional은 Service 계층에만 허용

---

END OF FILE
