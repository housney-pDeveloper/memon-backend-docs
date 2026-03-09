# VO_DTO_GLOBAL_POLICY.md

## 1. Table → VO

- tb_{entity} → {Entity}VO
- BaseVO 상속
- 위치: application/biz/vo

---

## 2. VO → Dto

- Dto는 VO 상속
- 필드 재정의 금지
- SearchDto는 SearchVO 상속

---

## 3. 중복 Bean 방지

- 동일 Dao 이름 금지
- 패턴: {도메인}{Entity}Dao

---

## 4. @Alias 규칙

- 모든 VO/DTO에 @Alias 필수
- Alias 값은 클래스명과 동일

---

## 5. Validation Group

- OnCreate
- OnUpdate
- OnSearch
- @Validated(Group.class) 사용