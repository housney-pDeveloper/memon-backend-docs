# COMMUNICATION_ARCHITECTURE.md

## 1. Client → Gateway → Application

- 외부 URL: /{clientType}/{version}
- 내부 URL: /api/{version}
- clientType은 Gateway에서 제거

---

## 2. Gateway 인증 헤더

- X-Gateway-Verified
- X-Gateway-Secret

Application은 두 헤더 모두 검증한다.

---

## 3. 통일된 응답 구조

{
"code": 1,
"message": "OK",
"data": {}
}

code == 1 → 성공
code != 1 → 실패