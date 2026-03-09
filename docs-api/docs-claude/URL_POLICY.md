# URL_POLICY.md

API URL 작성 규칙을 정의한다.

---

## 1. 기본 원칙

- 이 문서는 프론트엔드 개발자를 위한 API 문서이다
- 모든 API는 Gateway를 통한 접근만 허용
- **문서의 URL은 외부 유입 URL 기준으로 작성 (Gateway 진입점 기준)**

---

## 2. URL 패턴

```
/{clientType}/{version}/{endpoint}
```

| 파라미터 | 값 | 설명 |
|---------|-----|------|
| {clientType} | app, web | 클라이언트 유형 식별자 |
| {version} | v1, v2, ... | API 버전 |
| {endpoint} | 각 API 경로 | Controller의 RequestMapping 경로 (/api 제외) |

---

## 3. URL 예시

### 3-1. 문서 작성 시 URL 표기

- 패턴: `/{clientType}/{version}/{endpoint}`
- 예시:
  - `POST /app/v1/login`
  - `POST /web/v1/login`
  - `POST /web/v1/signup/insClient`

### 3-2. Gateway URL 변환 참고

Gateway가 외부 URL을 내부 URL로 변환함

| 외부 URL (문서 기준) | 내부 URL (Application) |
|---------------------|----------------------|
| `/app/v1/signup/getBizRegNo` | `/api/v1/signup/getBizRegNo` |
| `/web/v1/login` | `/api/v1/login` |

---

## 4. 주의사항

- 문서에는 항상 외부 URL을 기재
- 내부 URL (/api/...)은 문서에 기재하지 않음
- clientType은 호출하는 클라이언트에 따라 명시

---

END OF FILE
